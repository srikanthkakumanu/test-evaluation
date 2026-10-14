# INSTRUCTIONS.md — AI Answer Evaluator

> **Audience:** Claude (or any engineer) implementing this project from scratch.
> **Source:** Reverse-engineered from
> [aish-1509/deriv-onlinehands-on-assessment](https://github.com/aish-1509/deriv-onlinehands-on-assessment)
> (README, `docs/IMPLEMENTATION_GUIDE.md`, all code and tests). I ran the original on
> Python 3.11 (21 tests passed) to find its gaps. Every gap found is turned into a requirement below.
> All scores in this file were checked by running the reference code in §7.3.

---

## Table of Contents

1. [How to Use This File](#1-how-to-use-this-file)
2. [Project Overview](#2-project-overview)
3. [Tech Stack](#3-tech-stack)
4. [Rules (Must Follow)](#4-rules-must-follow)
5. [Project Layout](#5-project-layout)
6. [Data Contract](#6-data-contract)
7. [Scoring Algorithm (Token-Set F1, Capped at 0.99)](#7-scoring-algorithm-token-set-f1-capped-at-099)
8. [Functional Requirements](#8-functional-requirements)
9. [Non-Functional Requirements](#9-non-functional-requirements)
10. [Gaps in the Original and the Requirements That Close Them](#10-gaps-in-the-original-and-the-requirements-that-close-them)
11. [Implementation Steps](#11-implementation-steps)
12. [Acceptance Criteria](#12-acceptance-criteria)
13. [Out of Scope / Known Limitations](#13-out-of-scope--known-limitations)
14. [Definition of Done](#14-definition-of-done)

---

## 1. How to Use This File

- Implement **every** requirement in §8 and §9. The gap fixes are normal requirements, not a later phase.
- Treat **MUST** as mandatory, **SHOULD** as strongly expected, and **MAY** as optional.
- Each requirement has an ID (`FR-…`, `NFR-…`). Requirements that fix a gap in the original
  are tagged **[G#]** (see §10).
- Each acceptance criterion (`AC-…`) names the requirement it proves. Write at least one
  automated test per AC.
- When this file and your own preference disagree, this file wins. When this file is silent,
  choose the simplest option that keeps the code small, typed, and deterministic.

[↑ Back to top](#table-of-contents)

---

## 2. Project Overview

A **local, deterministic** Python tool that grades AI model answers against reference answers.

- Input: a JSON **list** of items. Each item has `id`, `prompt`, `reference_answer`, `model_answer`.
- Output: a score (0.0–1.0) and short reason per item, plus a batch summary.
- Two interfaces that share one service:
  - **HTTP API** (FastAPI): `POST /evaluate`
  - **CLI**: JSON from a file argument or from stdin
- **No** LLM calls, **no** network calls, **no** environment variables, **no** database.

[↑ Back to top](#table-of-contents)

---

## 3. Tech Stack

| Layer | Choice | Version constraint |
| --- | --- | --- |
| Language | Python | **≥ 3.11** (enforced, see FR-20). CI tests 3.11 and 3.12 |
| Web framework | FastAPI | `fastapi>=0.115,<1.0` |
| Validation / models | Pydantic v2 | `pydantic>=2.9,<3.0` |
| ASGI server | Uvicorn | `uvicorn>=0.30,<1.0` |
| Test runner | pytest | `pytest>=8,<10` (dev) |
| HTTP test client | httpx (`ASGITransport`) | `httpx>=0.27,<1.0` (dev) |
| Linter | Ruff | `ruff>=0.6,<1.0` (dev) |
| CLI parsing | `argparse` (stdlib) | — |
| Text handling | `re`, `unicodedata` (stdlib) | — |
| CI | GitHub Actions | `actions/checkout@v4`, `actions/setup-python@v5` |

**Dependency and config files (exact content):**

`requirements.txt`
```text
fastapi>=0.115,<1.0
pydantic>=2.9,<3.0
uvicorn>=0.30,<1.0
```

`requirements-dev.txt`
```text
-r requirements.txt

httpx>=0.27,<1.0
pytest>=8,<10
ruff>=0.6,<1.0
```

`pyproject.toml`
```toml
[project]
name = "ai-answer-evaluator"
version = "1.0.0"
requires-python = ">=3.11"

[tool.pytest.ini_options]
addopts = "-q"
testpaths = ["tests"]
pythonpath = ["."]

[tool.ruff]
target-version = "py311"
```

[↑ Back to top](#table-of-contents)

---

## 4. Rules (Must Follow)

1. **No external calls.** No LLM, HTTP client calls, telemetry, or downloads at runtime.
2. **No new runtime dependencies** beyond §3. Stdlib is fine.
3. **Determinism.** The same input and options MUST always give byte-identical output.
   No randomness, no time-based values.
4. **Layering.** `scoring.py` MUST be pure: no FastAPI, no file I/O, no printing.
   `service.py` MUST NOT import FastAPI or touch files. The API and CLI MUST both call
   `service.evaluate_items`, so behavior and validation cannot drift apart.
5. **Shared models.** `models.py` is the single source of the request/response contract.
6. **Type hints everywhere**, with a docstring on every module, class, and public function.
7. **Missing answers are results, not errors.** A null or blank answer gives score `0.0`
   and a reason. It MUST NOT give a 422 or a non-zero exit code.
8. **1.0 is reserved for exact matches.** No partial score may ever equal 1.0.
9. **Rounding.** Item scores and the average MUST be rounded to 4 decimal places with `round(x, 4)`.
10. **Reason strings are part of the contract.** Use the exact text in §7.5.
11. **Tests must not open network ports.** Use `httpx.ASGITransport` for API tests and
    `subprocess` for CLI tests.
12. **Safe by default.** The server MUST bind to `127.0.0.1` unless the user passes another host.
13. Do not commit `.env` files. `.env.example` exists only to state that none is needed.

[↑ Back to top](#table-of-contents)

---

## 5. Project Layout

```text
.
├── .github/workflows/ci.yml   # Ruff + pytest on push/PR to main, Python 3.11 & 3.12
├── .env.example               # "# No environment variables are required.\n"
├── .gitignore
├── LICENSE                    # MIT
├── README.md
├── docs/IMPLEMENTATION_GUIDE.md
├── app/
│   ├── __init__.py            # """Local AI answer evaluation service."""
│   ├── models.py              # Pydantic request/response models + item validators
│   ├── scoring.py             # Pure normalization + scoring
│   ├── service.py             # Batch validation, evaluation, summary
│   ├── main.py                # FastAPI app and routes
│   └── cli.py                 # JSON file/stdin interface
├── main.py                    # Entry point: version guard, then API or CLI mode
├── sample_input.json
├── pyproject.toml
├── requirements.txt
├── requirements-dev.txt
└── tests/
    ├── test_scoring.py
    ├── test_models.py
    ├── test_service.py
    ├── test_api.py
    ├── test_cli.py
    ├── test_entrypoint.py
    └── test_environment.py
```

**Data flow:**

```text
FastAPI request          JSON file or stdin
      │                         │
      ▼                         ▼
 app/main.py               app/cli.py
      └──────────┬──────────────┘
                 ▼
          app/service.py   (validate_batch → score each → summary)
                 ▼
          app/scoring.py   (pure per-item scoring)
```

[↑ Back to top](#table-of-contents)

---

## 6. Data Contract

### 6.1 Input — JSON array of `EvaluationItem`

| Field | Type | Required | Rule |
| --- | --- | --- | --- |
| `id` | string | yes | Not empty and not whitespace-only. MUST be unique within the batch |
| `prompt` | string | yes | Not empty and not whitespace-only (stored, not used in scoring) |
| `reference_answer` | string, number, or null | no (default `null`) | Numbers are converted with `str()`. Booleans are rejected. Null or blank gives 0.0 |
| `model_answer` | string, number, or null | no (default `null`) | Same as `reference_answer` |

- The top-level value MUST be a JSON list of at most **1000** items (`MAX_BATCH_SIZE`).
- Extra fields on an item are ignored.

### 6.2 Options

| Option | API (query param) | CLI flag | Values | Default |
| --- | --- | --- | --- | --- |
| Scoring strategy | `strategy` | `--strategy` | `f1`, `recall` | `f1` |
| Pass threshold | `pass_threshold` | `--pass-threshold` | float `0.0`–`1.0` | none |

### 6.3 Output — `EvaluationResponse`

```json
{
  "results": [
    { "id": "1", "score": 0.4, "reason": "Partial token overlap: ..." }
  ],
  "summary": {
    "average_score": 0.45,
    "number_of_items": 4,
    "exact_matches": 1,
    "failed_items": 1
  }
}
```

| Model | Field | Constraint |
| --- | --- | --- |
| `ItemResult` | `id: str`, `score: float`, `reason: str` | `0.0 ≤ score ≤ 1.0` |
| `EvaluationSummary` | `average_score: float` | `0.0 ≤ x ≤ 1.0` |
| | `number_of_items: int` | `≥ 0` |
| | `exact_matches: int` | `≥ 0`. Count with `is_exact == True` |
| | `failed_items: int` | `≥ 0`. Count with `score == 0.0` |
| | `passed_items: int \| None` | Present **only** when `pass_threshold` is given. Count with `score ≥ threshold` |

- Results MUST keep the input order.
- When `pass_threshold` is not given, the `passed_items` key MUST NOT appear in the output.
  Serialize with `exclude_none=True`.

### 6.4 Sample input (`sample_input.json`, exact content)

```json
[
  {
    "id": "1",
    "prompt": "What is 2 + 2?",
    "reference_answer": "4",
    "model_answer": "The answer is 4."
  },
  {
    "id": "2",
    "prompt": "Name one primary color.",
    "reference_answer": "red",
    "model_answer": "blue"
  },
  {
    "id": "3",
    "prompt": "What planet do humans live on?",
    "reference_answer": "Earth",
    "model_answer": "Humans live on Earth."
  },
  {
    "id": "4",
    "prompt": "Reply with exactly: approved",
    "reference_answer": "approved",
    "model_answer": "Approved"
  }
]
```

### 6.5 Expected sample results

| Options | Scores | Summary |
| --- | --- | --- |
| default (`f1`) | `[0.4, 0.0, 0.4, 1.0]` | `average_score 0.45`, `number_of_items 4`, `exact_matches 1`, `failed_items 1` |
| `strategy=recall` | `[0.99, 0.0, 0.99, 1.0]` | `average_score 0.745`, `exact_matches 1`, `failed_items 1` |
| `pass_threshold=0.4` | `[0.4, 0.0, 0.4, 1.0]` | same as default **plus** `passed_items 3` |

[↑ Back to top](#table-of-contents)

---

## 7. Scoring Algorithm (Token-Set F1, Capped at 0.99)

### 7.1 In plain words

Think of it as a word-matching game.

1. **Clean both answers.** Fix odd Unicode forms (NFKC), squeeze runs of spaces into
   one space, cut spaces off the ends, and lowercase with `casefold()`.
   `"  Approved   NOW "` → `"approved now"`. `"STRASSE"` and `"Straße"` both → `"strasse"`.
2. **Missing?** If either answer is empty after cleaning, the score is **0.0**.
3. **Identical?** If the cleaned answers are exactly equal, the score is **1.0**.
   Punctuation still counts here: `"approved"` ≠ `"approved."`.
4. **Otherwise, compare word bags.** Split each answer into a **set** of unique words.
   Punctuation is dropped, and repeats and word order don't matter.
   - `"Earth"` → `{earth}`
   - `"Humans live on Earth."` → `{humans, live, on, earth}`
5. **Score the overlap.** The default is **F1**:

   ```text
   F1 = 2 × (shared words) ÷ (reference words + model words)
      = 2 × 1 ÷ (1 + 4) = 0.4
   ```

   F1 is high only when the model's answer is **both** complete (it covers the reference
   words) **and** focused (it doesn't add many extra words). Extra words pull the score down.
   That is why the correct but wordy `"The answer is 4."` vs `"4"` scores only **0.4** under F1.

   The **recall** strategy only asks *"did the model say every reference word?"*:

   ```text
   recall = shared words ÷ reference words = 1 ÷ 1 = 1.0  → capped to 0.99
   ```

6. **Cap at 0.99.** Different texts can produce the **same** word set:

   | Reference | Model | Why the sets are equal | Raw | Final |
   | --- | --- | --- | --- | --- |
   | `approved` | `approved.` | punctuation dropped | 1.0 | **0.99** |
   | `red blue` | `blue red` | order ignored | 1.0 | **0.99** |
   | `yes` | `yes yes` | repeats ignored | 1.0 | **0.99** |
   | 200 words | the same minus 1 | near-identical | 0.9975 | **0.99** |

   Without the cap, these would score 1.0 and look like exact matches. `min(raw, 0.99)`
   keeps the rule simple: **`score == 1.0` means an exact match, always.**

7. **Negation check.** If exactly **one** answer contains a negation word (`not`, `no`,
   `never`, `none`, `nothing`, `neither`, `nor`, `cannot`, or a word ending in `n't`/`n’t`),
   the answers probably mean opposite things. The score is **halved** after the cap:
   `"the answer is 4"` vs `"the answer is not 4"` → 0.8889 × 0.5 = **0.4444**.
   If both answers contain a negation, or neither does, nothing changes.

8. **Round** the final score to 4 decimals.

### 7.2 Decision order (MUST follow this exact order)

```text
reference = normalize(reference_answer); model = normalize(model_answer)
├─ both empty          → 0.0  "Reference and model answers are missing."
├─ reference empty     → 0.0  "Reference answer is missing."
├─ model empty         → 0.0  "Model answer is missing."
├─ reference == model  → 1.0  "Exact match after normalization."   (is_exact = True)
├─ tokenize both (unique \w+ tokens)
├─ either token set empty → 0.0 "No comparable word tokens after normalization."
├─ no shared tokens    → 0.0  "No shared tokens after normalization."
├─ raw = F1 (strategy "f1") or recall (strategy "recall")
├─ score = min(raw, 0.99)
├─ negation mismatch?  → score × 0.5, append " Negation mismatch detected; score halved."
└─ round(score, 4)
```

### 7.3 Reference implementation (`app/scoring.py`)

This is the core of the project. Implement it exactly. It was run and checked against §7.6.

```python
"""Pure, deterministic text normalization and scoring functions."""

import re
import unicodedata
from dataclasses import dataclass
from typing import Literal

TOKEN_PATTERN = re.compile(r"\w+", flags=re.UNICODE)
NEGATION_PATTERN = re.compile(
    r"\b(?:not|no|never|none|nothing|neither|nor|cannot)\b|n['’]t\b"
)
PARTIAL_SCORE_CAP = 0.99
NEGATION_PENALTY = 0.5

Strategy = Literal["f1", "recall"]


@dataclass(frozen=True)
class ScoreResult:
    """Internal score details used to build the public API response."""

    score: float
    reason: str
    is_exact: bool = False


def normalize_text(text: str | None) -> str:
    """Apply NFKC, collapse whitespace, and casefold. None becomes ""."""
    if text is None:
        return ""
    return " ".join(unicodedata.normalize("NFKC", text).split()).casefold()


def tokenize(text: str) -> set[str]:
    """Return unique word-like tokens while ignoring punctuation."""
    return set(TOKEN_PATTERN.findall(text))


def has_negation(text: str) -> bool:
    """Return True when normalized text contains a negation word."""
    return NEGATION_PATTERN.search(text) is not None


def score_answers(
    reference_answer: str | None,
    model_answer: str | None,
    strategy: Strategy = "f1",
) -> ScoreResult:
    """Compare two answers using exact match, then token-set overlap."""
    reference = normalize_text(reference_answer)
    model = normalize_text(model_answer)

    if not reference and not model:
        return ScoreResult(
            score=0.0,
            reason="Reference and model answers are missing.",
        )
    if not reference:
        return ScoreResult(score=0.0, reason="Reference answer is missing.")
    if not model:
        return ScoreResult(score=0.0, reason="Model answer is missing.")

    if reference == model:
        return ScoreResult(
            score=1.0,
            reason="Exact match after normalization.",
            is_exact=True,
        )

    reference_tokens = tokenize(reference)
    model_tokens = tokenize(model)
    shared_tokens = reference_tokens & model_tokens

    if not reference_tokens or not model_tokens:
        return ScoreResult(
            score=0.0,
            reason="No comparable word tokens after normalization.",
        )
    if not shared_tokens:
        return ScoreResult(
            score=0.0,
            reason="No shared tokens after normalization.",
        )

    if strategy == "recall":
        raw = len(shared_tokens) / len(reference_tokens)
        reason = (
            f"Partial token recall: {len(shared_tokens)} of "
            f"{len(reference_tokens)} reference token(s) found in the model answer."
        )
    else:
        raw = (2 * len(shared_tokens)) / (len(reference_tokens) + len(model_tokens))
        reason = (
            f"Partial token overlap: {len(shared_tokens)} shared token(s) across "
            f"{len(reference_tokens)} reference and {len(model_tokens)} model token(s)."
        )

    score = min(raw, PARTIAL_SCORE_CAP)
    if has_negation(reference) != has_negation(model):
        score *= NEGATION_PENALTY
        reason += " Negation mismatch detected; score halved."

    return ScoreResult(score=round(score, 4), reason=reason)
```

### 7.4 Validation and error messages

| Situation | API | CLI |
| --- | --- | --- |
| Invalid JSON | 422 (FastAPI) | stderr `Input error: ...`, exit 2, empty stdout |
| Not a list, missing/blank `id` or `prompt`, boolean answer | 422 | `Input error: ...`, exit 2 |
| Duplicate `id` | 422, `detail` contains `Duplicate id: '<id>'` | `Input error: Duplicate id: '<id>'`, exit 2 |
| More than 1000 items | 422, `detail` contains `Batch too large` | `Input error: Batch too large ...`, exit 2 |
| File not found / unreadable | — | `Input error: ...`, exit 2 |
| File not UTF-8 | — | `Input error: ...`, exit 2 (no traceback) |
| Invalid `strategy` / `pass_threshold` out of range | 422 | argparse usage error, exit 2 |

### 7.5 Reason strings (exact text)

| Case | Reason |
| --- | --- |
| Both missing | `Reference and model answers are missing.` |
| Reference missing | `Reference answer is missing.` |
| Model missing | `Model answer is missing.` |
| Exact | `Exact match after normalization.` |
| Only punctuation/symbols (e.g. `!!!` vs `???`) | `No comparable word tokens after normalization.` |
| No overlap | `No shared tokens after normalization.` |
| Partial, `f1` | `Partial token overlap: {S} shared token(s) across {R} reference and {M} model token(s).` |
| Partial, `recall` | `Partial token recall: {S} of {R} reference token(s) found in the model answer.` |
| Negation mismatch (suffix) | ` Negation mismatch detected; score halved.` |

### 7.6 Verified score table (use these in tests)

| Reference | Model | `f1` | `recall` | Exact? |
| --- | --- | --- | --- | --- |
| `4` | `The answer is 4.` | 0.4 | 0.99 | no |
| `red` | `blue` | 0.0 | 0.0 | no |
| `Earth` | `Humans live on Earth.` | 0.4 | 0.99 | no |
| `approved` | `Approved` | 1.0 | 1.0 | yes |
| `approved` | `approved.` | 0.99 | 0.99 | no |
| `approved now` | `Approved   NOW` | 1.0 | 1.0 | yes |
| `Straße` | `STRASSE` | 1.0 | 1.0 | yes |
| `red blue` | `blue red` | 0.99 | 0.99 | no |
| `yes` | `yes yes` | 0.99 | 0.99 | no |
| `yes` | `yes please confirm` | 0.5 | 0.99 | no |
| `the answer is 4` | `the answer is not 4` | 0.4444 | 0.495 | no |
| `it is red` | `it isn’t red` | 0.2857 | 0.3333 | no |
| `it is not red` | `it isn't red` | 0.5 | 0.5 | no (both negated, no penalty) |
| `!!!` | `???` | 0.0 | 0.0 | no |

[↑ Back to top](#table-of-contents)

---

## 8. Functional Requirements

### Input and validation

| ID | Requirement |
| --- | --- |
| FR-1 | Accept a JSON list of evaluation items through `POST /evaluate`, a CLI file argument, or CLI stdin. |
| FR-2 | `id` and `prompt` MUST be strings that are not empty and not whitespace-only. Validate with a Pydantic `field_validator`. Store the value unchanged. **[G7]** |
| FR-3 | `reference_answer` and `model_answer` MUST accept a string, an int/float (converted with `str()`), or null. Booleans MUST be rejected with `Answer must be a string, number, or null.` Use a `field_validator(..., mode="before")`. **[G9]** |
| FR-4 | A batch with duplicate `id` values MUST be rejected with `Duplicate id: '<id>'` (first duplicate found, in input order). **[G6]** |
| FR-5 | A batch with more than `MAX_BATCH_SIZE = 1000` items MUST be rejected with `Batch too large: <n> items (maximum 1000).` Exactly 1000 is allowed. **[G8]** |
| FR-6 | FR-4 and FR-5 MUST live in one function, `service.validate_batch(items)`, which raises `BatchValidationError(ValueError)`. `evaluate_items` calls it first. The API maps it to **422** (`HTTPException(status_code=422, detail=str(error))`). The CLI maps it to `Input error: <message>`, exit **2**. |
| FR-7 | Invalid input MUST produce: API → **422**; CLI → stderr starting with `Input error:`, exit **2**, nothing on stdout. This includes invalid JSON, schema errors, missing or unreadable files, and **non-UTF-8 files** (catch `UnicodeDecodeError`, no traceback). See §7.4. **[G12]** |

### Scoring

| ID | Requirement |
| --- | --- |
| FR-8 | Normalize answers by applying NFKC, collapsing all whitespace runs to one space, stripping, and lowercasing with `casefold()`. `None` becomes `""`. Punctuation is kept for exact matching. **[G4]** |
| FR-9 | Score `1.0` when normalized answers are exactly equal, and set `is_exact=True`. |
| FR-10 | Otherwise score with token-set overlap over unique `\w+` tokens using the selected strategy: `f1` (default) or `recall` (§7). **[G1]** |
| FR-11 | Cap every partial score at `0.99` (`PARTIAL_SCORE_CAP`). Reordered, repeated, or punctuation-only-different answers score 0.99 and are never exact. **[G3]** |
| FR-12 | If exactly one normalized answer matches `NEGATION_PATTERN`, multiply the capped score by `NEGATION_PENALTY = 0.5` and append the negation reason suffix. **[G2]** |
| FR-13 | Missing, null, or whitespace-only answers score `0.0` with the matching reason (§7.5). Symbol-only answers score `0.0` with `No comparable word tokens after normalization.` |

### Output

| ID | Requirement |
| --- | --- |
| FR-14 | Return one `ItemResult` per item (`id`, `score`, `reason`), in input order. |
| FR-15 | Return a summary: `average_score` (rounded to 4 dp), `number_of_items`, `exact_matches`, `failed_items` (`score == 0.0`). |
| FR-16 | When `pass_threshold` (0.0–1.0) is given, add `passed_items` = count with `score >= pass_threshold`. When it is not given, the key MUST be absent. **[G5]** |
| FR-17 | An empty list returns `results: []` and an all-zero summary, with no division by zero. |

### Interfaces

| ID | Requirement |
| --- | --- |
| FR-18 | **API** (title `"AI Answer Evaluator"`, version `"1.0.0"`): `GET /` → service info JSON (exact shape in AC-30); `GET /health` → `{"status": "healthy"}`; `GET /favicon.ico` → 204, hidden from the schema; `POST /evaluate` with query params `strategy: Strategy = "f1"` and `pass_threshold: float \| None = Query(None, ge=0.0, le=1.0)` → `EvaluationResponse` with `response_model_exclude_none=True`. |
| FR-19 | **CLI** (`app/cli.py`): `run_cli(input_file=None, *, strategy="f1", pass_threshold=None, stdin=sys.stdin, stdout=sys.stdout, stderr=sys.stderr) -> int`. On success it writes `model_dump(mode="json", exclude_none=True)` as JSON with `indent=2` plus a trailing newline and returns 0. |
| FR-20 | **Entry point** (`main.py`). Before importing `app`, call `ensure_supported_python()`. On Python < 3.11 it exits with `Python 3.11 or newer is required.` and status 1 (no traceback). **[G13]** |
| FR-21 | **Entry point arguments:** positional `input_file` (optional `Path`), `--serve` (flag), `--host` (default **`127.0.0.1`**), `--port` (default `8000`), `--strategy` (`choices=["f1","recall"]`, default `f1`), `--pass-threshold` (float, must be 0.0–1.0, else `parser.error`). `--help` text MUST contain `Start the API in an interactive terminal`. **[G11]** |
| FR-22 | **Mode selection:** `--serve` with `input_file` → `parser.error` (exit 2). `--serve` → always start the API, even when stdin is not a TTY (Docker, systemd, nohup). Otherwise, if `input_file` is given or stdin is not a TTY → CLI. Otherwise → `uvicorn.run("main:app", host=..., port=...)`. Exit with `raise SystemExit(main())`. **[G10]** |

[↑ Back to top](#table-of-contents)

---

## 9. Non-Functional Requirements

| ID | Requirement |
| --- | --- |
| NFR-1 | **Local only.** No network, LLM, or env-var dependencies. `.env.example` content is `# No environment variables are required.` plus a newline. |
| NFR-2 | **Deterministic** output for identical input and options. |
| NFR-3 | **Python ≥ 3.11**, declared in `pyproject.toml` (`requires-python`) and enforced at runtime (FR-20). **[G13]** |
| NFR-4 | **Separation of concerns** per Rule 4. |
| NFR-5 | **Safe network default:** bind `127.0.0.1`. Binding all interfaces needs an explicit `--host 0.0.0.0`. **[G11]** |
| NFR-6 | **Test suite** runs with plain `pytest`, needs no running server, no network, and no `.env`. Every AC in §12 has a test. The `.env` check MUST only verify `.env.example` content and that evaluation works without config. It MUST NOT fail just because a developer has a local `.env`. **[G14]** |
| NFR-7 | **CI:** GitHub Actions workflow `CI`, on push/PR to `main`, `permissions: contents: read`, matrix `python-version: ["3.11", "3.12"]`, pip cache. Steps: `python -m pip install -r requirements-dev.txt`, `ruff check .`, `python -m pytest`. **[G15]** |
| NFR-8 | **Lint clean:** `ruff check .` exits 0. Use `# noqa: E402` only for the imports placed after the version guard in `main.py`. **[G15]** |
| NFR-9 | **Docs:** README with requirement coverage, setup (`python3.11 -m venv .venv`), API/CLI usage including `--serve`, `--host`, `--strategy`, `--pass-threshold`, the scoring rules (cap, negation, strategies), the error table (§7.4), edge cases, and the limitations in §13. `docs/IMPLEMENTATION_GUIDE.md` covers design, decision order, validation choices, and extension points. **[G2, G3]** |

[↑ Back to top](#table-of-contents)

---

## 10. Gaps in the Original and the Requirements That Close Them

Each gap was confirmed by running the original code on Python 3.11.

| # | Gap in the original | Evidence (original output) | Closed by |
| --- | --- | --- | --- |
| G1 | Correct but wordy answers score low | `"4"` vs `"The answer is 4."` → 0.4 | FR-10 (`recall` strategy) |
| G2 | Negation not detected | `"the answer is 4"` vs `"the answer is not 4"` → 0.8889 | FR-12 (→ 0.4444), NFR-9 |
| G3 | Word order and repeats ignored | `"red blue"` vs `"blue red"` → 0.99 | FR-11 (by design), NFR-9 (documented) |
| G4 | Weak normalization | `"approved now"` vs `"approved  now"` → 0.99; `"Straße"` vs `"STRASSE"` → 0.0 | FR-8 (→ both 1.0) |
| G5 | No pass threshold; `failed_items` only counts 0.0 | a 0.01 score is "not failed" | FR-16 |
| G6 | Duplicate IDs accepted | two items with `id:"a"` → 200 | FR-4, FR-6 |
| G7 | Blank `id`/`prompt` accepted | `id=" "`, `prompt=" "` valid | FR-2 |
| G8 | No batch size limit | unbounded list accepted | FR-5, FR-6 |
| G9 | Numeric answers rejected | `"reference_answer": 4` → 422 (`string_type`) | FR-3 |
| G10 | Non-TTY start never runs the server | `python main.py < /dev/null` → `Input error`, exit 2 | FR-22 (`--serve`) |
| G11 | Binds all interfaces by default | `--host` default `0.0.0.0` | FR-21, NFR-5, Rule 12 |
| G12 | Non-UTF-8 file crashes the CLI | `UnicodeDecodeError` traceback, exit 1 | FR-7 |
| G13 | Python version not enforced | Python 3.9: `TypeError: unsupported operand type(s) for \|` at import | FR-20, NFR-3 |
| G14 | Missing tests | no tests for the CLI schema-error path, missing file, the symbols-only branch, reorder cap, duplicates, non-TTY; brittle `.env` test | NFR-6, §12 |
| G15 | No lint, single Python version in CI | — | NFR-7, NFR-8 |

[↑ Back to top](#table-of-contents)

---

## 11. Implementation Steps

Build in this order. Run `pytest` after each step that adds tests.

1. **Scaffold.** Create the layout in §5 and the files in §3, plus `.env.example` and
   `.gitignore` (ignore `.env`, `.env.*` except `!.env.example`, `.venv/`, `__pycache__/`,
   `.pytest_cache/`, `.ruff_cache/`, coverage, IDE folders, `.DS_Store`), and an MIT `LICENSE`.
2. **`app/scoring.py`.** Copy §7.3 exactly. Write `tests/test_scoring.py` (AC-1 – AC-12).
3. **`app/models.py`.**
   - `EvaluationItem` with `id: str = Field(min_length=1)`, `prompt: str = Field(min_length=1)`,
     and answers `str | None = Field(default=None)`.
   - A `field_validator("id", "prompt")` that raises `ValueError("must not be blank")` when
     `not value.strip()`.
   - A `field_validator("reference_answer", "model_answer", mode="before")`: `bool` → raise
     `ValueError("Answer must be a string, number, or null.")`, `int`/`float` → `str(value)`,
     otherwise unchanged. **Check `bool` first**, because `bool` is a subclass of `int`.
   - `ItemResult`, `EvaluationSummary` (with `passed_items: int | None = Field(default=None, ge=0)`),
     and `EvaluationResponse`.
   - Write `tests/test_models.py` (AC-13 – AC-16).
4. **`app/service.py`.**
   - `MAX_BATCH_SIZE = 1000`, `class BatchValidationError(ValueError)`.
   - `validate_batch(items)` checks size first, then duplicates (use a `set` of seen ids).
   - `evaluate_items(items, strategy="f1", pass_threshold=None) -> EvaluationResponse`
     calls `validate_batch`, then loops `score_answers(item.reference_answer, item.model_answer, strategy)`,
     counts `exact_matches += int(outcome.is_exact)`, `failed_items += int(outcome.score == 0.0)`,
     sets average to `round(sum/len, 4)` if there are results else `0.0`, and sets
     `passed_items = sum(r.score >= pass_threshold for r in results)` only when a threshold is given.
   - Write `tests/test_service.py` (AC-17 – AC-23).
5. **`app/main.py`.** FastAPI app and routes per FR-18. Wrap `evaluate_items` in
   `try/except BatchValidationError` → `HTTPException(422, detail=str(error))`.
   Write `tests/test_api.py` (AC-24 – AC-32).
6. **`app/cli.py`.** `ITEM_LIST_ADAPTER = TypeAdapter(list[EvaluationItem])`,
   `read_items(input_file, stdin)` (file opened with `encoding="utf-8"`), and `run_cli` per FR-19,
   catching `(OSError, UnicodeDecodeError, json.JSONDecodeError, ValidationError, BatchValidationError)`.
   It prints `Input error: {error}` to stderr and returns 2.
7. **Root `main.py`.**
   ```python
   """Run the evaluator as an API or CLI from one convenient entry point."""

   import sys


   def ensure_supported_python(version_info: tuple[int, ...] = tuple(sys.version_info)) -> None:
       """Exit with a clear message on Python versions older than 3.11."""
       if tuple(version_info[:2]) < (3, 11):
           raise SystemExit("Python 3.11 or newer is required.")


   ensure_supported_python()

   import argparse  # noqa: E402
   from pathlib import Path  # noqa: E402
   from typing import Sequence  # noqa: E402

   import uvicorn  # noqa: E402

   from app.cli import run_cli  # noqa: E402
   from app.main import app  # noqa: E402,F401  (re-exported for "main:app")
   ```
   Then `build_parser()` and `main(argv=None)` per FR-21 and FR-22.
   Write `tests/test_cli.py` (AC-33 – AC-41) and `tests/test_entrypoint.py` (AC-42 – AC-46).
8. **`sample_input.json`.** Use §6.4.
9. **Environment test.** `tests/test_environment.py` (AC-47 – AC-48).
10. **CI + docs.** `.github/workflows/ci.yml` per NFR-7, a README with a CI badge per NFR-9,
    and the implementation guide.

**Local run commands (MUST work):**

```bash
python3.11 -m venv .venv && source .venv/bin/activate
python -m pip install -r requirements-dev.txt
ruff check . && pytest                                  # all green
python main.py                                          # API on 127.0.0.1:8000 (interactive terminal)
python main.py --serve --host 0.0.0.0                   # API in Docker/systemd
python main.py sample_input.json                        # CLI from file
python main.py --strategy recall --pass-threshold 0.4 < sample_input.json
curl -s -X POST -H "Content-Type: application/json" \
  --data @sample_input.json "http://127.0.0.1:8000/evaluate?pass_threshold=0.4"
```

[↑ Back to top](#table-of-contents)

---

## 12. Acceptance Criteria

Each AC MUST be an automated test. The test file is shown in each heading.

### Scoring — `tests/test_scoring.py`

- **AC-1 (FR-8, FR-9)** `normalize_text("  Approved \n") == "approved"`.
  `score_answers("approved", "  Approved ")` gives `1.0`, `is_exact is True`, and reason
  `Exact match after normalization.`
- **AC-2 (FR-8)** `score_answers("approved now", "Approved   NOW")` gives `1.0` and is exact.
- **AC-3 (FR-8)** `score_answers("Straße", "STRASSE")` gives `1.0` and is exact.
  `normalize_text("ﬁne") == "fine"` (NFKC).
- **AC-4 (FR-8, FR-11)** `normalize_text(" Approved. ") == "approved."`, and
  `score_answers("approved", "approved.")` gives `0.99` and is not exact.
- **AC-5 (FR-10)** `score_answers("Earth", "Humans live on Earth.")` gives `0.4`, not exact,
  with a reason starting with `Partial token overlap`.
- **AC-6 (FR-10)** With `strategy="recall"`: `("4", "The answer is 4.")` gives `0.99`, with a
  reason starting with `Partial token recall: 1 of 1`.
- **AC-7 (FR-11)** `("red blue", "blue red")`, `("yes", "yes yes")`, and
  `("yes", "yes please confirm", "recall")` all give `0.99` and are not exact.
- **AC-8 (FR-12)** `("the answer is 4", "the answer is not 4")` gives `0.4444` (f1) and
  `0.495` (recall). The reason ends with `Negation mismatch detected; score halved.`
- **AC-9 (FR-12)** `("it is red", "it isn’t red")` (curly apostrophe) gives `0.2857`.
- **AC-10 (FR-12)** `("it is not red", "it isn't red")` gives `0.5` with no negation suffix,
  because both answers are negated.
- **AC-11 (FR-13)** Parametrized zero cases, each scoring `0.0` with the exact reason:
  `(None, "answer")` → reference missing; `("answer", "   ")` → model missing;
  `(None, None)` → both missing; `("red", "blue")` → no shared tokens;
  `("!!!", "???")` → no comparable word tokens.
- **AC-12 (Rule 8)** For every row in §7.6, `score == 1.0` exactly when `is_exact is True`.

### Models — `tests/test_models.py`

- **AC-13 (FR-2)** `EvaluationItem(id=" ", prompt="p")` and `EvaluationItem(id="1", prompt="  ")`
  raise `ValidationError`.
- **AC-14 (FR-3)** `reference_answer=4` is stored as `"4"`, and `2.5` as `"2.5"`.
- **AC-15 (FR-3)** `model_answer=True` raises `ValidationError` with a message containing
  `Answer must be a string, number, or null.`
- **AC-16 (FR-3)** An item without answers has both answers `None`.

### Service — `tests/test_service.py`

- **AC-17 (FR-14, FR-15)** The four sample items give scores `[0.4, 0.0, 0.4, 1.0]` and
  summary `0.45 / 4 / 1 / 1`. `passed_items is None`.
- **AC-18 (FR-10)** The sample with `strategy="recall"` gives `[0.99, 0.0, 0.99, 1.0]` and average `0.745`.
- **AC-19 (FR-16)** The sample with `pass_threshold=0.4` gives `passed_items == 3`.
- **AC-20 (FR-17)** `evaluate_items([])` gives `results == []` and every count and average equal to 0.
- **AC-21 (FR-4, FR-6)** Items with ids `["a", "b", "a"]` raise `BatchValidationError`
  with the message `Duplicate id: 'a'`.
- **AC-22 (FR-5, FR-6)** 1001 items raise `BatchValidationError` with a message starting with
  `Batch too large: 1001 items`. Exactly 1000 unique items succeed.
- **AC-23 (FR-14)** Results keep input order (ids `["z", "a", "m"]` stay in that order).

### API — `tests/test_api.py` (httpx `ASGITransport`, no real port)

- **AC-24 (FR-1, FR-15)** `POST /evaluate` with `sample_input.json` returns 200, summary
  exactly `{"average_score": 0.45, "number_of_items": 4, "exact_matches": 1, "failed_items": 1}`
  (no `passed_items` key), and result ids `["1","2","3","4"]`.
- **AC-25 (FR-16, FR-18)** `POST /evaluate?pass_threshold=0.4` gives `summary.passed_items == 3`.
- **AC-26 (FR-10, FR-18)** `POST /evaluate?strategy=recall` gives `summary.average_score == 0.745`.
- **AC-27 (FR-18)** `?strategy=bogus` and `?pass_threshold=1.5` each return 422.
- **AC-28 (FR-13)** An item without `model_answer` returns 200 with result
  `{"id": ..., "score": 0.0, "reason": "Model answer is missing."}`.
- **AC-29 (FR-2, FR-7)** An item without `id`, or with `id: " "`, returns 422.
- **AC-30 (FR-18)** `GET /` returns 200 with exactly
  `{"name": "AI Answer Evaluator", "status": "ready", "evaluate": "POST /evaluate", "documentation": "/docs", "health": "/health"}`.
- **AC-31 (FR-18)** `GET /health` returns `{"status": "healthy"}`. `GET /favicon.ico` returns 204.
- **AC-32 (FR-4, FR-5, FR-6)** Duplicate ids return 422 with `detail` containing `Duplicate id`.
  1001 items return 422 with `detail` containing `Batch too large`.

### CLI — `tests/test_cli.py` (subprocess, `cwd` = project root)

- **AC-33 (FR-1, FR-19)** `python main.py < sample_input.json` gives exit 0, empty stderr,
  and a stdout summary matching AC-24.
- **AC-34 (FR-1)** `python main.py sample_input.json` gives exit 0 and `results[3].score == 1.0`.
- **AC-35 (FR-21)** `python main.py --strategy recall --pass-threshold 0.4 sample_input.json`
  gives exit 0, `average_score == 0.745`, and `passed_items == 3`.
- **AC-36 (FR-7)** stdin `{not valid json` gives exit 2, empty stdout, and stderr starting with `Input error:`.
- **AC-37 (FR-7)** stdin `[{"prompt": "x"}]` (schema error) gives exit 2 and `Input error:`.
- **AC-38 (FR-7)** A file path that doesn't exist gives exit 2 and `Input error:`.
- **AC-39 (FR-7)** A file containing bytes `b"\xff\xfe"` gives exit 2, `Input error:`,
  empty stdout, and no `Traceback` in stderr.
- **AC-40 (FR-6)** Duplicate ids on stdin give exit 2 and stderr containing `Duplicate id`.
- **AC-41 (FR-21)** `python main.py --help` gives exit 0, stdout containing
  `Start the API in an interactive terminal`, and empty stderr.

### Entry point — `tests/test_entrypoint.py` (monkeypatch `uvicorn.run`, never start a real server)

- **AC-42 (FR-21, FR-22)** With TTY-like stdin, `main([])` returns 0 and calls
  `uvicorn.run("main:app", host="127.0.0.1", port=8000)`.
- **AC-43 (FR-21)** `main(["--host", "0.0.0.0", "--port", "9000"])` with TTY stdin passes those values through.
- **AC-44 (FR-22)** With **non-TTY** stdin (`io.StringIO("")`), `main(["--serve"])` returns 0
  and calls `uvicorn.run` exactly once (no CLI read).
- **AC-45 (FR-22)** `main(["--serve", "sample_input.json"])` raises `SystemExit` with code 2.
  `main(["--pass-threshold", "1.5"])` also raises `SystemExit` with code 2.
- **AC-46 (FR-20)** `ensure_supported_python((3, 10, 0))` raises `SystemExit` with message
  `Python 3.11 or newer is required.`. `ensure_supported_python((3, 11, 0))` returns `None`.

### Environment and tooling — `tests/test_environment.py`

- **AC-47 (NFR-1, NFR-6)** `.env.example` equals `"# No environment variables are required.\n"`,
  and `evaluate_items` works with no configuration (`"yes"` vs `"YES"` gives 1.0).
  The test MUST NOT assert that `.env` is absent.
- **AC-48 (NFR-3)** `tomllib` parses `pyproject.toml`, and `project.requires-python == ">=3.11"`.
- **AC-49 (NFR-7, NFR-8)** (CI check, not pytest) `ruff check .` exits 0, and the CI matrix
  includes 3.11 and 3.12.

[↑ Back to top](#table-of-contents)

---

## 13. Out of Scope / Known Limitations

Document these in the README. Do **not** implement them:

- **True semantic understanding** (synonyms, paraphrase). The negation rule (FR-12) is a
  simple word-list heuristic. It does not understand sarcasm, double negatives, or words
  like `"false"` and `"incorrect"`.
- **Word order** stays ignored by design (FR-11). `"dog bites man"` vs `"man bites dog"` scores 0.99.
- **LLM-as-judge or embeddings.** Rule 1 forbids external calls.
- **Persistence, auth, rate limiting, observability/metrics.** These are future extension points.
- **Multiple acceptable references / per-item weights.** These are future extension points.
- **Extra fields** on items are ignored (Pydantic default). This is acceptable.

[↑ Back to top](#table-of-contents)

---

## 14. Definition of Done

- [ ] Every requirement in §8 and §9 is implemented.
- [ ] Every AC in §12 (AC-1 … AC-49) has a passing automated test or CI check.
- [ ] Every gap in §10 is closed by its listed requirement.
- [ ] `python main.py sample_input.json` prints summary `0.45 / 4 / 1 / 1` with no `passed_items` key.
- [ ] `ruff check .` and `pytest` pass with no network access and no `.env` file.
- [ ] Rules in §4 hold (pure scoring, shared service, no new runtime deps, deterministic, safe host default).
- [ ] README and implementation guide match NFR-9.
- [ ] CI workflow is green on `main` for Python 3.11 and 3.12.

[↑ Back to top](#table-of-contents)
