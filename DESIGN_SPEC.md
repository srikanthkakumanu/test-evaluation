# DESIGN_SPEC.md — AI Answer Evaluator (Principal Engineering Design)

> **Status:** Proposed · **Version:** 2.0 design · **Date:** 2026-10-14
> **Companion to:** [`INSTRUCTIONS.md`](INSTRUCTIONS.md). That file says **what** to build
> (requirements, gaps, acceptance criteria). This file says **how** an experienced principal
> engineer would build it: architecture, contracts, decisions, and trade-offs.
> **Scope:** a production-grade **local** tool. It stays deterministic, makes no LLM calls,
> and needs no network.
>
> The scoring engine code in §5 and the property tests in §12 were run before publishing.
> 19 tests passed, including 3 Hypothesis properties at 500 examples each. Benchmark numbers in
> §11 were measured on the prototype.

---

## Table of Contents

1. [Summary, Goals, Non-Goals](#1-summary-goals-non-goals)
2. [Quality Attributes (Ranked)](#2-quality-attributes-ranked)
3. [Architecture](#3-architecture)
4. [Domain Model](#4-domain-model)
5. [Scoring Engine](#5-scoring-engine)
6. [Application Layer (Use Case)](#6-application-layer-use-case)
7. [HTTP Adapter (API v1)](#7-http-adapter-api-v1)
8. [CLI Adapter](#8-cli-adapter)
9. [Configuration](#9-configuration)
10. [Observability and Security](#10-observability-and-security)
11. [Performance and Capacity](#11-performance-and-capacity)
12. [Testing Strategy](#12-testing-strategy)
13. [Build, Packaging, CI/CD, Release](#13-build-packaging-cicd-release)
14. [Architecture Decision Records](#14-architecture-decision-records)
15. [Compatibility and Migration](#15-compatibility-and-migration)
16. [Risks, Trade-offs, Roadmap](#16-risks-trade-offs-roadmap)
17. [Delivery Plan and Acceptance Criteria](#17-delivery-plan-and-acceptance-criteria)

---

## 1. Summary, Goals, Non-Goals

### 1.1 Summary

The original project works, but it is built like an assessment answer. Scoring, I/O, and
mode detection are tangled together. Results can't be traced back to the rules that produced them.
The only quality check is example-based tests. This design keeps the product the same and
changes how it is built:

- A **pure scoring engine**. Every rule is one small stage, and every knob lives in one
  `ScoringPolicy` value with a stable `policy_id`.
- **Thin adapters** (HTTP, CLI) around **one use case**, so behavior cannot drift.
- **Exact arithmetic** with `Fraction` and `Decimal`, so the same input gives byte-identical output on any machine.
- **Explainable results**: a machine-readable `reason_code` and `details`, not just a sentence.
- **Real contracts**: a versioned API, RFC 9457 errors, an OpenAPI snapshot, and CLI exit codes for CI gates.
- **Stronger tests**: property-based invariants, golden files, and contract tests.

### 1.2 Goals

| # | Goal | How we measure it |
| --- | --- | --- |
| G-1 | Same input + same policy → byte-identical output | Determinism property test; golden files |
| G-2 | Every score is explainable and traceable | `reason_code`, `details`, and `meta.policy_id` present on every response |
| G-3 | API and CLI can never disagree | Both call `EvaluateBatch`. The adapter parity test compares their outputs |
| G-4 | Usable as a CI quality gate | `answer-eval evaluate --fail-under 0.8` exits 1 when below the bar |
| G-5 | Fast enough not to think about | 1000 items: engine ≤ 50 ms p95, HTTP end-to-end ≤ 200 ms p95 |
| G-6 | Safe by default | Binds `127.0.0.1`; size limits on body, items, and answers; no answer text in logs |
| G-7 | Easy to extend | A new metric is one function plus `@register_metric`. No other file changes |

### 1.3 Non-Goals

- Semantic understanding, embeddings, or LLM-as-judge (see the extension seam in §16).
- Persistence, multi-tenancy, auth, or a job queue. This is a local tool.
- A UI.

[↑ Back to top](#table-of-contents)

---

## 2. Quality Attributes (Ranked)

When two goals conflict, the higher one wins.

1. **Correctness and determinism.** A grader that gives different answers on different days is worthless.
2. **Explainability.** Users must see *why* an answer got its score.
3. **Simplicity.** Small modules, no frameworks inside the core, and no speculative abstraction.
4. **Extensibility.** New metrics and adjusters plug in without edits to existing code.
5. **Performance.** It matters, but it's already far inside budget (§11).

> **Note:** "Simplicity" is ranked above "extensibility" on purpose. There is **no** repository
> port, no plugin loader, and no DI container until a real need appears (YAGNI).

[↑ Back to top](#table-of-contents)

---

## 3. Architecture

### 3.1 Style: Hexagonal (Ports and Adapters)

```text
              ┌──────────────── adapters (I/O, frameworks) ─────────────────┐
              │                                                             │
  HTTP client ──►  adapters/http   (FastAPI, problem+json, limits, logging) │
  Shell / CI  ──►  adapters/cli    (argparse, JSON/JSONL, exit codes)       │
              │            │                         │                      │
              └────────────┼─────────────────────────┼──────────────────────┘
                           ▼                         ▼
              ┌────────── application ────────────────────────┐
              │  EvaluateBatch use case                       │
              │  batch validation · streaming · summary       │
              └───────────────────────┬───────────────────────┘
                                      ▼
              ┌────────── domain + scoring (pure) ────────────┐
              │  EvaluationItem · ScoringPolicy · ItemScore   │
              │  normalize → tokenize → metric → adjust →     │
              │  quantize                                     │
              └───────────────────────────────────────────────┘
```

**Dependency rule:** arrows point inward only. `domain/` and `scoring/` import nothing from
FastAPI, argparse, pydantic-settings, logging handlers, or the file system. An import-linter
check in CI enforces this (§13).

### 3.2 Package layout (`src` layout)

```text
.
├── pyproject.toml              # hatchling build, deps, tools config, console script
├── uv.lock                     # pinned, reproducible environment
├── Dockerfile
├── main.py                     # compatibility shim → answer_eval.adapters.cli.legacy
├── sample_input.json
├── src/answer_eval/
│   ├── __init__.py             # __version__
│   ├── config.py               # Settings (pydantic-settings)
│   ├── domain/
│   │   ├── models.py           # EvaluationItem, ItemResult, Summary, Report (pydantic)
│   │   └── errors.py           # DomainError hierarchy
│   ├── scoring/
│   │   ├── policy.py           # ScoringPolicy, policy_id
│   │   ├── text.py             # normalize, tokenize, has_negation
│   │   ├── metrics.py          # registry + f1, recall
│   │   ├── numeric.py          # quantize, average (Fraction → Decimal)
│   │   ├── reasons.py          # ReasonCode + reason message templates
│   │   └── engine.py           # score_pair (the pipeline)
│   ├── application/
│   │   ├── evaluate.py         # EvaluateBatch, SummaryAccumulator
│   │   └── fingerprint.py      # canonical input sha256
│   └── adapters/
│       ├── http/
│       │   ├── app.py          # create_app(settings) factory
│       │   ├── routes_v1.py    # /v1/evaluations, /v1/policy
│       │   ├── routes_legacy.py# deprecated POST /evaluate
│       │   ├── problems.py     # RFC 9457 handlers
│       │   └── middleware.py   # request id, body limit, timing log
│       └── cli/
│           ├── main.py         # `answer-eval` entry point (subcommands)
│           ├── io.py           # JSON / JSONL readers and writers
│           └── legacy.py       # old `python main.py` behavior
└── tests/
    ├── unit/  property/  adapters/  contract/  golden/  perf/
```

[↑ Back to top](#table-of-contents)

---

## 4. Domain Model

### 4.1 Types

| Type | Kind | Responsibility |
| --- | --- | --- |
| `EvaluationItem` | Pydantic model (input) | `id`, `prompt`, `reference_answer`, `model_answer`, with validation from INSTRUCTIONS FR-2/FR-3 |
| `ScoringPolicy` | Frozen dataclass (value object) | All scoring knobs, plus `policy_id` |
| `ReasonCode` | `StrEnum` | Machine-readable outcome of one item |
| `ItemScore` | Frozen dataclass | Engine output: `score: Decimal`, `reason_code`, `details` |
| `ItemResult` | Pydantic model (output) | `id`, `score`, `is_exact`, `reason_code`, `reason`, `details` |
| `EvaluationSummary` | Pydantic model | `average_score`, `number_of_items`, `exact_matches`, `failed_items`, `passed_items?` |
| `ReportMeta` | Pydantic model | `tool_version`, `api_version`, `policy_id`, `input_sha256`, `item_count` |
| `EvaluationReport` | Pydantic model | `meta`, `results`, `summary` |
| `DomainError` → `BatchValidationError`, `LimitExceededError` | Exceptions | Carry a stable `code` that adapters map to HTTP/CLI |

### 4.2 Reason codes and messages

`reason` stays the exact human text from INSTRUCTIONS §7.5, for backward compatibility.
`reason_code` is the stable API that machines should use.

| `reason_code` | Score | `reason` (human) |
| --- | --- | --- |
| `both_missing` | 0 | `Reference and model answers are missing.` |
| `reference_missing` | 0 | `Reference answer is missing.` |
| `model_missing` | 0 | `Model answer is missing.` |
| `exact_match` | 1 | `Exact match after normalization.` |
| `no_comparable_tokens` | 0 | `No comparable word tokens after normalization.` |
| `no_shared_tokens` | 0 | `No shared tokens after normalization.` |
| `partial_overlap` | (0, 0.99] | Metric-specific template, plus the negation suffix when applied |

**Invariant (tested):** `score == 1` **if and only if** `reason_code == exact_match`.

### 4.3 Why `Decimal` in the domain and `float` only at the edge

Scores are computed as exact `Fraction`s and rounded once to `Decimal` (4 dp, half-even).
Adapters convert to JSON numbers at the very last step (`float(Decimal("0.4444")) == 0.4444`).
This removes binary float drift, so results are identical on every OS and Python version.

[↑ Back to top](#table-of-contents)

---

## 5. Scoring Engine

### 5.1 Pipeline

```text
 reference, model
       │
       ▼
 normalize  (NFKC → collapse whitespace → casefold)
       │
       ├── empty? ─────────────► both_missing / reference_missing / model_missing (0)
       ├── equal? ─────────────► exact_match (1)
       ▼
 tokenize   (unique \w+ tokens)
       ├── a set is empty? ────► no_comparable_tokens (0)
       ├── nothing shared? ────► no_shared_tokens (0)
       ▼
 metric     (registry: f1 | recall)        → exact Fraction
       ▼
 adjusters  (cap 99/100 → negation × 1/2)  → recorded in details.adjustments
       ▼
 quantize   (Decimal, 4 dp, ROUND_HALF_EVEN)
```

Each stage is a pure function. The order is fixed and documented. Changing the order changes
`policy_id` (see §5.3).

### 5.2 Reference implementation (verified)

Split across `scoring/*.py` in the real package. Shown here as one block. This exact code passed
the table and property tests.

```python
"""Pure, deterministic scoring engine."""

from __future__ import annotations

import hashlib
import json
import re
import unicodedata
from collections.abc import Callable, Mapping
from dataclasses import dataclass, field
from decimal import ROUND_HALF_EVEN, Decimal, localcontext
from enum import StrEnum
from fractions import Fraction

# ---------- policy ----------------------------------------------------------


@dataclass(frozen=True, slots=True)
class ScoringPolicy:
    """Every knob that can change a score. Hashed into policy_id."""

    metric: str = "f1"
    partial_cap: Fraction = Fraction(99, 100)
    negation_penalty: Fraction = Fraction(1, 2)
    decimals: int = 4
    normalizer: str = "nfkc-ws-casefold/v1"
    tokenizer: str = r"unique-\w+/v1"

    @property
    def policy_id(self) -> str:
        """Stable short hash of the canonical policy."""
        canonical = json.dumps(
            {
                "metric": self.metric,
                "partial_cap": str(self.partial_cap),
                "negation_penalty": str(self.negation_penalty),
                "decimals": self.decimals,
                "normalizer": self.normalizer,
                "tokenizer": self.tokenizer,
            },
            sort_keys=True,
            separators=(",", ":"),
        )
        return "pol_" + hashlib.sha256(canonical.encode()).hexdigest()[:12]


# ---------- metric registry -------------------------------------------------

Metric = Callable[[int, int, int], Fraction]  # (shared, reference, model) -> ratio
METRICS: dict[str, Metric] = {}


def register_metric(name: str) -> Callable[[Metric], Metric]:
    """Register an overlap metric under a stable name."""

    def decorator(fn: Metric) -> Metric:
        if name in METRICS:
            raise ValueError(f"metric already registered: {name}")
        METRICS[name] = fn
        return fn

    return decorator


@register_metric("f1")
def f1(shared: int, reference: int, model: int) -> Fraction:
    """Harmonic mean of token precision and recall."""
    return Fraction(2 * shared, reference + model)


@register_metric("recall")
def recall(shared: int, reference: int, model: int) -> Fraction:
    """Share of reference tokens present in the model answer."""
    return Fraction(shared, reference)


# ---------- pipeline stages -------------------------------------------------

TOKEN_PATTERN = re.compile(r"\w+")
NEGATION_PATTERN = re.compile(
    r"\b(?:not|no|never|none|nothing|neither|nor|cannot)\b|n['’]t\b"
)


def normalize(text: str | None) -> str:
    """NFKC, collapse whitespace, casefold. None -> ''."""
    if text is None:
        return ""
    return " ".join(unicodedata.normalize("NFKC", text).split()).casefold()


def tokenize(text: str) -> frozenset[str]:
    """Unique word tokens; punctuation ignored."""
    return frozenset(TOKEN_PATTERN.findall(text))


def has_negation(text: str) -> bool:
    """True when normalized text contains a negation cue."""
    return NEGATION_PATTERN.search(text) is not None


def quantize(value: Fraction, decimals: int) -> Decimal:
    """Exact Fraction -> Decimal rounded half-even. No float drift."""
    with localcontext() as ctx:
        ctx.prec = 50
        exact = Decimal(value.numerator) / Decimal(value.denominator)
        return exact.quantize(Decimal(1).scaleb(-decimals), rounding=ROUND_HALF_EVEN)


# ---------- result ----------------------------------------------------------


class ReasonCode(StrEnum):
    """Machine-readable outcome of scoring one item."""

    BOTH_MISSING = "both_missing"
    REFERENCE_MISSING = "reference_missing"
    MODEL_MISSING = "model_missing"
    EXACT_MATCH = "exact_match"
    NO_COMPARABLE_TOKENS = "no_comparable_tokens"
    NO_SHARED_TOKENS = "no_shared_tokens"
    PARTIAL_OVERLAP = "partial_overlap"


@dataclass(frozen=True, slots=True)
class ItemScore:
    """Score, explanation, and evidence for one item."""

    score: Decimal
    reason_code: ReasonCode
    details: Mapping[str, object] = field(default_factory=dict)

    @property
    def is_exact(self) -> bool:
        return self.reason_code is ReasonCode.EXACT_MATCH


ZERO = Decimal(0)


def score_pair(
    reference_answer: str | None,
    model_answer: str | None,
    policy: ScoringPolicy = ScoringPolicy(),
) -> ItemScore:
    """Run the scoring pipeline for one reference/model pair."""
    reference, model = normalize(reference_answer), normalize(model_answer)

    if not reference and not model:
        return ItemScore(ZERO, ReasonCode.BOTH_MISSING)
    if not reference:
        return ItemScore(ZERO, ReasonCode.REFERENCE_MISSING)
    if not model:
        return ItemScore(ZERO, ReasonCode.MODEL_MISSING)
    if reference == model:
        return ItemScore(Decimal(1), ReasonCode.EXACT_MATCH)

    ref_tokens, model_tokens = tokenize(reference), tokenize(model)
    if not ref_tokens or not model_tokens:
        return ItemScore(ZERO, ReasonCode.NO_COMPARABLE_TOKENS)
    shared = len(ref_tokens & model_tokens)
    if shared == 0:
        return ItemScore(ZERO, ReasonCode.NO_SHARED_TOKENS)

    raw = METRICS[policy.metric](shared, len(ref_tokens), len(model_tokens))
    adjustments: list[str] = []
    value = raw
    if value > policy.partial_cap:
        value = policy.partial_cap
        adjustments.append("capped")
    if has_negation(reference) != has_negation(model):
        value *= policy.negation_penalty
        adjustments.append("negation_mismatch")

    return ItemScore(
        quantize(value, policy.decimals),
        ReasonCode.PARTIAL_OVERLAP,
        {
            "metric": policy.metric,
            "shared_tokens": shared,
            "reference_tokens": len(ref_tokens),
            "model_tokens": len(model_tokens),
            "raw": str(raw),
            "adjustments": adjustments,
        },
    )


def average(scores: list[Decimal], decimals: int = 4) -> Decimal:
    """Exact mean of already-quantized scores, quantized again."""
    if not scores:
        return quantize(Fraction(0), decimals)
    return quantize(Fraction(sum(scores)) / len(scores), decimals)
```

> **Note:** `>` in the cap check matters. `details.adjustments` contains `"capped"` only when
> the raw ratio was really above 0.99. That tells a reader the two answers had **identical word
> sets** but different text (punctuation, order, or repeats).

### 5.3 `policy_id` — why it exists

A score is only meaningful together with the rules that produced it. `policy_id` is a SHA-256
of the canonical policy (metric, cap, penalty, decimals, normalizer version, tokenizer version):

| Policy | `policy_id` |
| --- | --- |
| default (`f1`) | `pol_227dd9143bc3` |
| `metric="recall"` | `pol_672929e45328` |

It appears in every report's `meta`. If you compare two runs with different `policy_id`s, you
are comparing different graders. Any change to text handling MUST bump the `normalizer` or
`tokenizer` version string, which changes the id. This is how scoring changes become visible.

### 5.4 Worked examples (verified)

| Reference | Model | `f1` | `recall` | `adjustments` (f1) |
| --- | --- | --- | --- | --- |
| `4` | `The answer is 4.` | 0.4 | 0.99 | — |
| `Earth` | `Humans live on Earth.` | 0.4 | 0.99 | — |
| `approved` | `Approved` | 1 (exact) | 1 (exact) | — |
| `approved` | `approved.` | 0.99 | 0.99 | `capped` |
| `red blue` | `blue red` | 0.99 | 0.99 | `capped` |
| `Straße` | `STRASSE` | 1 (exact) | 1 (exact) | — |
| `yes` | `yes please confirm` | 0.5 | 0.99 | — |
| `the answer is 4` | `the answer is not 4` | 0.4444 | 0.495 | `negation_mismatch` |
| `it is red` | `it isn’t red` | 0.2857 | 0.3333 | `negation_mismatch` |
| `!!!` | `???` | 0 | 0 | — |

Sample batch: `f1` → `[0.4, 0, 0.4, 1]`, average **0.45**. `recall` → average **0.745**.

### 5.5 Adding a new metric (the extension contract)

```python
from fractions import Fraction

from answer_eval.scoring.metrics import register_metric


@register_metric("precision")
def precision(shared: int, reference: int, model: int) -> Fraction:
    """Share of model tokens that also appear in the reference."""
    return Fraction(shared, model)
```

That's the whole change: one function, plus a golden file and a property-test entry. The cap,
the negation adjuster, rounding, the API enum, and `policy_id` all pick it up automatically.

[↑ Back to top](#table-of-contents)

---

## 6. Application Layer (Use Case)

### 6.1 One code path for batch and stream

Both adapters use **one** `EvaluateBatch`. The batch API is just "stream, then collect". This
makes it impossible for JSON and JSONL (or API and CLI) to compute different summaries.

```python
"""EvaluateBatch use case: validation, streaming scoring, summary."""

from collections.abc import Iterable, Iterator
from dataclasses import dataclass, field
from decimal import Decimal

from answer_eval.domain.errors import BatchValidationError, LimitExceededError
from answer_eval.domain.models import EvaluationItem, EvaluationSummary, ItemResult
from answer_eval.scoring.engine import ScoringPolicy, average, score_pair
from answer_eval.scoring.reasons import render_reason


@dataclass(frozen=True, slots=True)
class EvaluationOptions:
    """Caller-chosen options for one run."""

    policy: ScoringPolicy = ScoringPolicy()
    pass_threshold: Decimal | None = None
    max_items: int | None = 1000


@dataclass(slots=True)
class SummaryAccumulator:
    """Builds the summary incrementally so streaming uses O(1) memory per item."""

    pass_threshold: Decimal | None
    scores: list[Decimal] = field(default_factory=list)
    exact: int = 0
    failed: int = 0
    passed: int = 0

    def add(self, result: ItemResult) -> None:
        score = Decimal(str(result.score))
        self.scores.append(score)
        self.exact += int(result.is_exact)
        self.failed += int(score == 0)
        if self.pass_threshold is not None:
            self.passed += int(score >= self.pass_threshold)

    def finish(self) -> EvaluationSummary:
        return EvaluationSummary(
            average_score=float(average(self.scores)),
            number_of_items=len(self.scores),
            exact_matches=self.exact,
            failed_items=self.failed,
            passed_items=self.passed if self.pass_threshold is not None else None,
        )


class EvaluateBatch:
    """Validate items and score them with one policy."""

    def stream(
        self,
        items: Iterable[EvaluationItem],
        options: EvaluationOptions,
        summary: SummaryAccumulator,
    ) -> Iterator[ItemResult]:
        seen: set[str] = set()
        for index, item in enumerate(items, start=1):
            if options.max_items is not None and index > options.max_items:
                raise LimitExceededError(
                    f"Batch too large: more than {options.max_items} items."
                )
            if item.id in seen:
                raise BatchValidationError(f"Duplicate id: {item.id!r}")
            seen.add(item.id)

            outcome = score_pair(item.reference_answer, item.model_answer, options.policy)
            result = ItemResult(
                id=item.id,
                score=float(outcome.score),
                is_exact=outcome.is_exact,
                reason_code=outcome.reason_code,
                reason=render_reason(outcome),
                details=dict(outcome.details),
            )
            summary.add(result)
            yield result

    def run(
        self, items: list[EvaluationItem], options: EvaluationOptions
    ) -> tuple[list[ItemResult], EvaluationSummary]:
        summary = SummaryAccumulator(options.pass_threshold)
        results = list(self.stream(items, options, summary))
        return results, summary.finish()
```

### 6.2 Input fingerprint

`meta.input_sha256` is the SHA-256 of the canonical JSON of the validated items
(`sort_keys=True`, `separators=(",", ":")`, `ensure_ascii=False`). Two reports with the same
`input_sha256` and `policy_id` MUST be identical. This is the reproducibility contract.

> **Warning:** Never put timestamps or durations in the response body, because that breaks
> byte-identical reruns. Timing goes in logs and in the `Server-Timing` header only.

[↑ Back to top](#table-of-contents)

---

## 7. HTTP Adapter (API v1)

### 7.1 Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/v1/evaluations` | Evaluate a batch (main endpoint) |
| `GET` | `/v1/policy?metric=f1` | Show the policy and its `policy_id` without scoring anything |
| `GET` | `/health` | Liveness: `{"status": "healthy"}` |
| `GET` | `/version` | `{"tool_version": "2.0.0", "api_version": "v1"}` |
| `GET` | `/` | Service info (same shape as the original, plus `"evaluate": "POST /v1/evaluations"`) |
| `POST` | `/evaluate` | **Deprecated** legacy alias: bare list in, original response shape out |

### 7.2 Request and response

```http
POST /v1/evaluations
Content-Type: application/json
```

```json
{
  "options": { "metric": "f1", "pass_threshold": 0.4 },
  "items": [
    { "id": "3", "prompt": "What planet do humans live on?",
      "reference_answer": "Earth", "model_answer": "Humans live on Earth." }
  ]
}
```

```json
{
  "meta": {
    "tool_version": "2.0.0",
    "api_version": "v1",
    "policy_id": "pol_227dd9143bc3",
    "input_sha256": "…64 hex chars…",
    "item_count": 1
  },
  "results": [
    {
      "id": "3",
      "score": 0.4,
      "is_exact": false,
      "reason_code": "partial_overlap",
      "reason": "Partial token overlap: 1 shared token(s) across 1 reference and 4 model token(s).",
      "details": {
        "metric": "f1", "shared_tokens": 1, "reference_tokens": 1,
        "model_tokens": 4, "raw": "2/5", "adjustments": []
      }
    }
  ],
  "summary": {
    "average_score": 0.4, "number_of_items": 1,
    "exact_matches": 0, "failed_items": 0, "passed_items": 1
  }
}
```

**Why an envelope (`{options, items}`)?** A bare list can't carry options without query-string
hacks, and it can't grow. The envelope is the extensible contract. The legacy alias keeps old
callers working.

### 7.3 Errors — RFC 9457 `application/problem+json`

```json
{
  "type": "https://answer-eval.local/problems/duplicate-id",
  "title": "Duplicate item id",
  "status": 422,
  "detail": "Duplicate id: 'a'",
  "instance": "/v1/evaluations",
  "request_id": "6f1c2e0a9b6d4c1e"
}
```

| Problem `type` (suffix) | Status | When |
| --- | --- | --- |
| `validation-error` | 422 | Schema errors. Includes an `errors[]` list of `{loc, msg}` from Pydantic |
| `duplicate-id` | 422 | Two items share an `id` |
| `too-many-items` | 422 | More than `max_items` |
| `answer-too-long` | 422 | Any field over `max_answer_chars` |
| `payload-too-large` | 413 | Body bytes over `max_body_bytes` (checked before parsing) |
| `unsupported-media-type` | 415 | `Content-Type` is not JSON |
| `internal-error` | 500 | Unexpected. Generic `detail`, with full trace in logs only |

All errors are produced by **one** exception-handler module (`problems.py`). Routes never build
error responses by hand.

### 7.4 Route sketch

```python
"""/v1 routes. Thin: translate HTTP <-> use case, nothing else."""

from fastapi import APIRouter, Depends

from answer_eval.adapters.http.deps import get_settings, get_use_case
from answer_eval.adapters.http.schemas import EvaluationRequest, EvaluationResponse
from answer_eval.application.evaluate import EvaluateBatch, EvaluationOptions
from answer_eval.application.fingerprint import input_sha256
from answer_eval.config import Settings

router = APIRouter(prefix="/v1", tags=["evaluation"])


@router.post("/evaluations", response_model=EvaluationResponse, response_model_exclude_none=True)
def create_evaluation(
    request: EvaluationRequest,
    use_case: EvaluateBatch = Depends(get_use_case),
    settings: Settings = Depends(get_settings),
) -> EvaluationResponse:
    """Score a batch and return meta, per-item results, and a summary."""
    options = EvaluationOptions(
        policy=request.options.to_policy(),
        pass_threshold=request.options.pass_threshold_decimal(),
        max_items=settings.max_items,
    )
    results, summary = use_case.run(request.items, options)
    return EvaluationResponse.build(
        policy=options.policy,
        input_sha256=input_sha256(request.items),
        results=results,
        summary=summary,
    )
```

### 7.5 Deprecation of `POST /evaluate`

- Same input (bare list) and same output shape as the original (`id`, `score`, `reason`, plus the summary).
- Responses carry `Deprecation: true` and `Link: </v1/evaluations>; rel="successor-version"`.
- Removal no earlier than the next **major** version.

### 7.6 OpenAPI as a contract

`tests/contract/openapi.v1.json` is a committed snapshot of `app.openapi()`. CI fails on any
difference. A deliberate API change means updating the snapshot in the same PR, so every
contract change is visible in review.

[↑ Back to top](#table-of-contents)

---

## 8. CLI Adapter

### 8.1 Commands

Explicit subcommands replace "guess the mode from the TTY" (that guess was gap G10 in INSTRUCTIONS.md).

```text
answer-eval evaluate [INPUT|-] [--format auto|json|jsonl] [--output FILE]
                     [--metric f1|recall] [--pass-threshold X] [--fail-under X]
                     [--max-items N] [--compact]
answer-eval serve    [--host 127.0.0.1] [--port 8000] [--log-format json|text]
answer-eval policy   [--metric f1|recall]          # print policy JSON + policy_id
answer-eval --version
```

- `INPUT` defaults to `-` (stdin). `--format auto` picks **JSONL** for `*.jsonl` files and
  JSON otherwise.
- **JSON mode:** reads an array and writes the full report (same shape as the API).
- **JSONL mode (streaming):** reads one item per line and writes one result per line as it goes.
  The **last line** is `{"meta": {...}, "summary": {...}}`. Memory use stays flat for any file
  size, except the set of seen ids.
- `--fail-under X` turns the tool into a **CI quality gate**.

### 8.2 Exit codes

| Code | Meaning |
| --- | --- |
| 0 | Success (and the quality gate passed, if one was set) |
| 1 | Quality gate failed: `average_score < --fail-under` (the report is still written) |
| 2 | Usage or input error (bad flags, bad JSON, schema, duplicate id, limits, unreadable or non-UTF-8 file) |
| 70 | Unexpected internal error (`EX_SOFTWARE`). The message goes to stderr, with a traceback only with `--debug` |

Errors go to **stderr** as one line, `answer-eval: error: <code>: <detail>`, where `<code>`
uses the same names as the API problem types (e.g. `duplicate-id`). stdout stays clean, or
holds only the partial JSONL already written.

### 8.3 Compatibility shim

`python main.py …` keeps working. It calls `answer_eval.adapters.cli.legacy.main`, which keeps
the old behavior (file or piped stdin → evaluate; TTY → serve; `--serve` forces serve; host
default `127.0.0.1`) and prints a one-line deprecation hint to stderr.

[↑ Back to top](#table-of-contents)

---

## 9. Configuration

One `Settings` class (`pydantic-settings`), created once and passed in. Nothing reads
`os.environ` anywhere else. Every setting has a safe default, so **no env vars are required**.

| Setting | Env var | Default | Notes |
| --- | --- | --- | --- |
| `host` | `ANSWER_EVAL_HOST` | `127.0.0.1` | Never `0.0.0.0` by default |
| `port` | `ANSWER_EVAL_PORT` | `8000` | |
| `max_items` | `ANSWER_EVAL_MAX_ITEMS` | `1000` | API and JSON mode. JSONL uses `--max-items` |
| `max_body_bytes` | `ANSWER_EVAL_MAX_BODY_BYTES` | `5_242_880` (5 MiB) | Checked before JSON parsing |
| `max_answer_chars` | `ANSWER_EVAL_MAX_ANSWER_CHARS` | `10_000` | Per field. Guards CPU and memory |
| `default_metric` | `ANSWER_EVAL_DEFAULT_METRIC` | `f1` | Must exist in the registry |
| `log_level` | `ANSWER_EVAL_LOG_LEVEL` | `INFO` | |
| `log_format` | `ANSWER_EVAL_LOG_FORMAT` | `json` | `text` for local dev |

```python
"""Typed, validated settings with safe defaults."""

from typing import Literal

from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Runtime configuration. All fields optional; env overrides use ANSWER_EVAL_."""

    model_config = SettingsConfigDict(env_prefix="ANSWER_EVAL_", frozen=True)

    host: str = "127.0.0.1"
    port: int = Field(default=8000, ge=1, le=65535)
    max_items: int = Field(default=1000, ge=1)
    max_body_bytes: int = Field(default=5 * 1024 * 1024, ge=1024)
    max_answer_chars: int = Field(default=10_000, ge=1)
    default_metric: Literal["f1", "recall"] = "f1"
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR"] = "INFO"
    log_format: Literal["json", "text"] = "json"
```

[↑ Back to top](#table-of-contents)

---

## 10. Observability and Security

### 10.1 Logging

- Stdlib `logging` with a small JSON formatter (no extra dependency). One line per request.
- Fields: `event`, `request_id`, `method`, `route`, `status`, `item_count`, `policy_id`,
  `duration_ms`.
- `X-Request-ID` is accepted if valid (≤ 64 chars, `[A-Za-z0-9-]`), otherwise generated. It is
  echoed in the response and in problem bodies.
- `Server-Timing: evaluate;dur=<ms>` is added to API responses.
- **Never logged:** `prompt`, `reference_answer`, `model_answer`. They may contain private data.
  A unit test feeds a known secret string and asserts it never appears in captured logs.

### 10.2 Threat list (local tool, still taken seriously)

| Threat | Mitigation |
| --- | --- |
| Accidental exposure on a network | Bind `127.0.0.1` by default. In Docker, document `-p 127.0.0.1:8000:8000` |
| Memory or CPU exhaustion (huge body, huge answers, huge batch) | `max_body_bytes` (413, checked on a streamed body), `max_answer_chars`, `max_items` |
| ReDoS | Only linear regexes (`\w+`, a fixed alternation). No user-supplied patterns |
| Sensitive data in logs | Redaction rule plus the test in §10.1 |
| Stack traces leaking internals | `internal-error` problem with a generic detail. Traces only in server logs |
| Supply chain | `uv.lock` pinned, `pip-audit` in CI, Dependabot weekly |
| Container breakout impact | Non-root user (uid 10001), read-only root filesystem, slim base image |

[↑ Back to top](#table-of-contents)

---

## 11. Performance and Capacity

- **Complexity:** O(total characters) per item for normalize and tokenize, and O(min(|R|,|M|)) for set
  intersection. The batch is linear in input size.
- **Measured (prototype engine, dev laptop):** 1000 items of 30-word references and 60-word
  model answers. **Median 16.4 ms, p95 18.0 ms** (engine only, `Fraction`/`Decimal` included).
- **Budgets (enforced by `tests/perf`, marked `@pytest.mark.perf`, run in CI nightly):**

  | Scenario | Budget (p95) |
  | --- | --- |
  | Engine, 1000 items (30/60 words) | ≤ 50 ms |
  | HTTP `POST /v1/evaluations`, 1000 items, in-process ASGI | ≤ 200 ms |
  | CLI JSONL, 100k items | Flat memory (< 100 MB RSS) |

- **Why no async or worker pool:** the work is CPU-bound and takes milliseconds. FastAPI runs the
  sync route in its threadpool. Adding concurrency would add complexity for no gain at this scale.

[↑ Back to top](#table-of-contents)

---

## 12. Testing Strategy

### 12.1 Test pyramid

| Layer | Folder | What it proves | Speed |
| --- | --- | --- | --- |
| Unit | `tests/unit/` | Each stage (normalize, tokenize, metrics, adjusters, quantize) and the score table | ms |
| Property | `tests/property/` | Invariants hold for **any** text (Hypothesis) | ~1 s |
| Golden | `tests/golden/` | Full reports for fixed inputs match committed files byte for byte | ms |
| Adapter | `tests/adapters/` | API (httpx `ASGITransport`) and CLI (in-process plus 2 subprocess smoke tests): status codes, problem+json, exit codes | < 2 s |
| Contract | `tests/contract/` | OpenAPI snapshot unchanged; legacy `/evaluate` shape unchanged | ms |
| Parity | `tests/adapters/test_parity.py` | API JSON == CLI JSON == CLI JSONL summary for the same input | ms |
| Perf | `tests/perf/` | Budgets in §11 (nightly) | s |

### 12.2 Property tests (verified)

```python
from decimal import Decimal

from hypothesis import given, settings
from hypothesis import strategies as st

from answer_eval.scoring.engine import ScoringPolicy, score_pair

RECALL = ScoringPolicy(metric="recall")
text = st.one_of(st.none(), st.text(max_size=40))


@settings(max_examples=500)
@given(text, text)
def test_score_in_range_and_one_iff_exact(ref, model):
    for policy in (ScoringPolicy(), RECALL):
        result = score_pair(ref, model, policy)
        assert Decimal(0) <= result.score <= Decimal(1)
        assert (result.score == 1) == result.is_exact


@settings(max_examples=500)
@given(text, text)
def test_f1_symmetric_and_deterministic(ref, model):
    assert score_pair(ref, model).score == score_pair(model, ref).score
    assert score_pair(ref, model) == score_pair(ref, model)


@settings(max_examples=500)
@given(
    st.text(max_size=40),
    st.text(max_size=40),
    st.lists(st.from_regex(r"zq[a-z]{1,6}", fullmatch=True), min_size=1, max_size=3),
)
def test_extra_unrelated_words_never_raise_f1(ref, model, extra):
    before = score_pair(ref, model).score
    after = score_pair(ref, model + " " + " ".join(extra)).score
    if "zq" not in ref.casefold():
        assert after <= before
```

Planned additional properties: normalization is idempotent (`normalize(normalize(x)) == normalize(x)`).
`recall ≥ f1` whenever no adjuster applies **and** the model answer has at least as many
tokens as the reference. JSON output re-parses to the same report.

### 12.3 Quality gates (CI fails if missed)

- `ruff check` and `ruff format --check`
- `mypy --strict src/`
- `lint-imports` (import-linter): `domain` and `scoring` must not import `adapters`, `fastapi`, or `pydantic_settings`
- Coverage: **≥ 95% branch** for `scoring/` and `domain/`, **≥ 90%** overall
- OpenAPI snapshot and golden files unchanged (or updated in the same PR)

[↑ Back to top](#table-of-contents)

---

## 13. Build, Packaging, CI/CD, Release

### 13.1 Dependencies

| Kind | Packages |
| --- | --- |
| Runtime | `fastapi`, `pydantic>=2.9`, `pydantic-settings>=2.4`, `uvicorn` |
| Dev | `pytest`, `pytest-cov`, `httpx`, `hypothesis`, `mypy`, `ruff`, `import-linter`, `pip-audit` |

`pyproject.toml` (hatchling) declares `requires-python = ">=3.11"` and the console script
`answer-eval = "answer_eval.adapters.cli.main:main"`. `uv.lock` pins everything.

### 13.2 CI (GitHub Actions)

| Job | Steps |
| --- | --- |
| `lint` | ruff check and format, mypy --strict, lint-imports |
| `test` (matrix 3.11 / 3.12 / 3.13) | `uv sync --frozen`, then pytest with coverage gates |
| `security` | `pip-audit` |
| `contract` | OpenAPI snapshot and golden diff (part of pytest, also reported separately) |
| `docker` | Build the image and smoke-test `/health` |
| `perf` (nightly) | `pytest -m perf` |

### 13.3 Docker

```dockerfile
FROM python:3.12-slim AS build
COPY --from=ghcr.io/astral-sh/uv:0.5 /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
COPY src ./src
RUN uv sync --frozen --no-dev

FROM python:3.12-slim
RUN useradd --uid 10001 --no-create-home appuser
WORKDIR /app
COPY --from=build /app /app
ENV PATH="/app/.venv/bin:$PATH"
USER 10001
EXPOSE 8000
HEALTHCHECK CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"
CMD ["answer-eval", "serve", "--host", "0.0.0.0"]
```

Inside a container, the server must bind `0.0.0.0` to be reachable at all. Exposure is then
controlled by the publish flag: `docker run -p 127.0.0.1:8000:8000 answer-eval`.

### 13.4 Versioning and release

- **SemVer** for the tool. The API is versioned separately by path (`/v1`).
- **Any change that alters a score** (new normalizer version, cap, penalty, tokenizer) is a
  **minor** bump at minimum. It needs a `CHANGELOG.md` entry under "Scoring changes", a new
  `policy_id`, and updated golden files in the same PR.
- Releases come from tags `vX.Y.Z`. CI builds the wheel and image and attaches the SBOM.

[↑ Back to top](#table-of-contents)

---

## 14. Architecture Decision Records

Short form: **Context → Decision → Consequences.**

| ADR | Title | Context | Decision | Consequences |
| --- | --- | --- | --- | --- |
| 001 | Hexagonal architecture | The original mixed I/O, mode detection, and scoring | Pure core; HTTP and CLI adapters call one use case | + parity guaranteed, easy tests; − a few more modules |
| 002 | Scoring pipeline + metric registry | Metrics and adjusters will grow | Fixed stage order, `@register_metric` | + one-function extension; − the order is a policy decision that must be documented |
| 003 | `ScoringPolicy` + `policy_id` | Scores without their rules aren't comparable | Hash the canonical policy and stamp it on every report | + traceable, reproducible; − must remember to bump versions (enforced by golden tests) |
| 004 | Exact math (`Fraction` → `Decimal` half-even) | Float rounding can differ at edges | Exact ratios, one rounding step | + identical everywhere; − negligible cost (measured) |
| 005 | Structured reasons | Strings are for humans; machines need codes | `reason_code` + `details` + legacy `reason` | + filterable, translatable; − slightly larger payload |
| 006 | Versioned envelope API + RFC 9457 | A bare list can't carry options; ad-hoc errors | `/v1/evaluations` `{options, items}`; problem+json | + evolvable, standard errors; − legacy alias to maintain |
| 007 | Explicit CLI subcommands | TTY guessing broke Docker/systemd (G10) | `evaluate` / `serve` / `policy`; shim for `main.py` | + predictable; − users learn new commands (shim softens this) |
| 008 | JSONL streaming | Large offline runs | Stream through the same use case | + flat memory; − summary only at the end |
| 009 | Settings object, safe defaults | Config was hard-coded; unsafe host | `pydantic-settings`, `127.0.0.1`, limits | + 12-factor ready, still zero-config; − one more dependency |
| 010 | Property + golden + contract tests | Example tests miss edge cases | Hypothesis invariants, byte-exact golden files, OpenAPI snapshot | + catches regressions early; − snapshots need deliberate updates |

[↑ Back to top](#table-of-contents)

---

## 15. Compatibility and Migration

### 15.1 Mapping from INSTRUCTIONS.md

| INSTRUCTIONS.md | Where it lives in this design |
| --- | --- |
| FR-1 inputs (API, file, stdin) | `/v1/evaluations`, legacy `/evaluate`, `answer-eval evaluate`, `main.py` shim |
| FR-2, FR-3 item validation | `domain/models.py` validators |
| FR-4 to FR-6 duplicates, batch limit | `EvaluateBatch.stream` → `BatchValidationError` / `LimitExceededError` → problem+json / exit 2 |
| FR-7 error mapping (incl. UTF-8) | `adapters/http/problems.py`, `adapters/cli/io.py` |
| FR-8 to FR-13 scoring rules | `scoring/` pipeline (§5) |
| FR-14 to FR-17 results and summary | `SummaryAccumulator`, `EvaluationReport` |
| FR-18 API routes | `routes_v1.py`, `routes_legacy.py`, `app.py` |
| FR-19 CLI | `adapters/cli/main.py`, `io.py` |
| FR-20 to FR-22 entry point, host, `--serve` | `answer-eval serve`; `main.py` → `cli/legacy.py` |
| NFR-1 to NFR-9 | §2, §9, §10, §12, §13 |

### 15.2 Behavior differences (deliberate)

| Topic | Original / INSTRUCTIONS | This design | Impact |
| --- | --- | --- | --- |
| Rounding | `round(float, 4)` | `Decimal` half-even on exact `Fraction` | Same on every verified case. May differ only on exact ties that float can't represent |
| Response shape (v1) | `{results, summary}` | `{meta, results, summary}`; results add `is_exact`, `reason_code`, `details` | Additive. The legacy route keeps the old shape |
| Errors (v1) | FastAPI default 422 body | RFC 9457 problem+json | Clients should switch on `type` |
| Mode selection | TTY detection | Subcommands | Shim keeps the old behavior for `python main.py` |

[↑ Back to top](#table-of-contents)

---

## 16. Risks, Trade-offs, Roadmap

### 16.1 Risks

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Negation heuristic false positives (`"no. 5"`, `"no-fly zone"`) | Medium | Score halved wrongly | `details.adjustments` makes it visible. A policy flag can turn it off. Golden cases document it |
| Users read token overlap as "correctness" | High | Wrong conclusions | README and API docs state the limits. `reason_code` and `details` show what was measured |
| Silent scoring drift | Low | Broken comparisons | `policy_id` plus golden files plus changelog rule |
| Over-engineering a small tool | Medium | Slower delivery | YAGNI list in §2. No DI framework, no plugin loader, no persistence |

### 16.2 Roadmap (designed seams, not built now)

1. **Multiple references:** `reference_answers: list[str]`. The score is the max over references,
   and `details.best_reference` gives the index. Fits the pipeline unchanged.
2. **Per-item weights:** `weight` on the item, and a weighted average in `SummaryAccumulator`.
3. **More metrics:** `precision`, character n-gram, numeric-tolerance (e.g. `"4.0"` vs `"4"`),
   each one `@register_metric`.
4. **LLM-as-judge (optional, out of process):** a separate adapter behind a `Judge` port,
   flagged `deterministic=false` in `meta`, never part of the default policy.
5. **Run persistence:** a `ReportStore` port (SQLite adapter) for comparing runs over time,
   keyed by `input_sha256` and `policy_id`.

[↑ Back to top](#table-of-contents)

---

## 17. Delivery Plan and Acceptance Criteria

### 17.1 Milestones

| Milestone | Scope | Exit criteria |
| --- | --- | --- |
| **M1 — Core** | `scoring/`, `domain/`, unit, property, and golden tests | INSTRUCTIONS AC-1 to AC-16 pass. DS-AC-1 to DS-AC-4 pass. Coverage ≥ 95% on the core |
| **M2 — Use case + CLI** | `application/`, `answer-eval evaluate/policy`, JSON/JSONL, shim | INSTRUCTIONS AC-17 to AC-23, AC-33 to AC-46 pass. DS-AC-5 to DS-AC-8 pass |
| **M3 — HTTP** | `/v1`, problem+json, middleware, legacy alias, OpenAPI snapshot | INSTRUCTIONS AC-24 to AC-32 pass. DS-AC-9 to DS-AC-13 pass |
| **M4 — Hardening** | Settings, logging, limits, Docker, CI, perf, docs | INSTRUCTIONS AC-47 to AC-49 pass. DS-AC-14 to DS-AC-18 pass. CI green on 3.11–3.13 |

### 17.2 Design acceptance criteria (in addition to INSTRUCTIONS.md §12)

| ID | Given / When / Then |
| --- | --- |
| DS-AC-1 | Any two strings (Hypothesis): score ∈ [0, 1], and `score == 1` ⇔ `reason_code == "exact_match"` |
| DS-AC-2 | Any two strings: `f1` is symmetric and repeated calls are equal |
| DS-AC-3 | Default policy: `policy_id == "pol_227dd9143bc3"`. `metric="recall"` gives a different id |
| DS-AC-4 | `("approved", "approved.")` gives `details.adjustments == ["capped"]` and `details.raw == "1"` |
| DS-AC-5 | `answer-eval evaluate sample_input.json --fail-under 0.5` exits **1** (average 0.45) and still writes the report. `--fail-under 0.45` exits 0 |
| DS-AC-6 | JSONL input of 4 sample lines gives 4 result lines plus 1 final summary line with `average_score 0.45` |
| DS-AC-7 | The same input through the API, CLI JSON, and CLI JSONL gives identical results and summary (parity test) |
| DS-AC-8 | Running `answer-eval evaluate` twice on the same input gives byte-identical stdout |
| DS-AC-9 | `POST /v1/evaluations` with the sample gives `meta.policy_id`, a 64-hex `meta.input_sha256`, and summary `0.45/4/1/1` |
| DS-AC-10 | Duplicate ids return 422 with `Content-Type: application/problem+json` and a `type` ending in `/duplicate-id` |
| DS-AC-11 | A body over `max_body_bytes` returns **413** `payload-too-large` before JSON parsing |
| DS-AC-12 | `POST /evaluate` (legacy) returns the original shape plus `Deprecation: true` |
| DS-AC-13 | `app.openapi()` equals `tests/contract/openapi.v1.json` |
| DS-AC-14 | With default settings, `answer-eval serve` binds `127.0.0.1:8000` (`uvicorn.run` monkeypatched) |
| DS-AC-15 | A request containing a secret marker string never shows that string in captured logs |
| DS-AC-16 | Every response has `X-Request-ID`. A valid incoming id is echoed back |
| DS-AC-17 | `lint-imports` passes: `scoring`/`domain` do not import adapters or frameworks |
| DS-AC-18 | Perf: engine p95 ≤ 50 ms and HTTP p95 ≤ 200 ms for 1000 items (nightly job) |

[↑ Back to top](#table-of-contents)
