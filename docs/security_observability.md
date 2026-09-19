# Security and Observability

## Security Overview

The security layer in `src/security/` validates inputs at system boundaries —
file uploads, query text, and document text — before they enter the core pipeline.
All validation raises typed exceptions from `src/core/exceptions.py`.

No secrets are hardcoded anywhere in the codebase. API keys are read exclusively
from environment variables.

---

## File Validation (`src/security/file_validation.py`)

**Purpose:** Prevent unsafe or oversized files from entering the ingestion pipeline.

**Checks performed:**
- File size must not exceed `MAX_UPLOAD_MB` (default: 10 MB, configurable via `.env`)
- File extension must be in the allowed set: `.txt`, `.pdf`, `.html`, `.htm`
- File must be readable before processing begins

**Error type:** `ValidationError` (subclass of `RAGLabError`)

**Usage in the Streamlit UI:**
- File validation runs inside `_save_upload()` in `app/streamlit_app.py`
- Files that fail validation are rejected before write and a user-visible error is shown
- No partial files are written to `data/raw/`

---

## Query Validation (`src/security/query_validation.py`)

**Purpose:** Prevent empty, excessively long, or malformed query text from being
embedded or logged.

**`validate_query(query_text)`:**
- Strips leading/trailing whitespace
- Raises `ValidationError` if the result is empty
- Raises `ValidationError` if length exceeds the configured maximum

**`validate_top_k(top_k)`:**
- Raises `ValidationError` if `top_k < 1`
- Raises `ValidationError` if `top_k` exceeds the configured maximum

**Error type:** `ValidationError`

---

## Prompt Safety (`src/security/prompt_safety.py`)

**Purpose:** Detect and log potentially unsafe patterns in query text before embedding.

This module performs lightweight pattern matching and **logs a warning** for
flagged inputs. It does not block the query — it is an advisory layer, not a
hard gate.

Examples of flagged patterns include prompt-injection attempts and jailbreak prefixes.
Warnings are written to the application logger (`src/core/logging_config.py`).

**No user-visible blocking occurs** — the warning is for operator awareness only.

---

## PII Redaction (`src/security/pii_redaction.py`)

**Purpose:** Optionally remove personally identifiable information from document
text before chunking and storage.

**Patterns redacted (configurable):**
- Email addresses
- Phone numbers (common formats)
- Names (basic pattern matching)

**How to enable:**

In the pipeline:
```python
pipeline.build_index(file_path, redact_pii=True)
```

In the Streamlit UI:
- A "Redact PII" toggle is available in the Ingestion tab sidebar.

**Limitations:**
- Pattern-based redaction is not exhaustive. It does not replace a dedicated
  PII detection service for sensitive production use cases.
- Redaction is applied to the document text before chunking; it cannot be
  undone once the index is built.

---

## No Secrets in Code

All API keys and sensitive configuration values are managed through environment
variables. The `.env.example` file documents all available variables with safe
defaults:

```
OPENAI_API_KEY=        # Leave empty to stay in mock mode
GEMINI_API_KEY=        # Leave empty to stay in mock mode
MOCK_MODE=true         # Safe default
```

The `.env` file is excluded from version control via `.gitignore`. Tests
never require real API keys — the full test suite passes with the defaults
from `.env.example`.

---

## Observability Overview

The observability layer in `src/observability/` provides lightweight, zero-dependency
runtime monitoring without requiring any external service.

---

## METRICS Singleton (`src/observability/metrics.py`)

`METRICS` is a module-level singleton of the `_Metrics` dataclass. It is updated
by `RAGPipeline` after each successful operation and read by the Streamlit System
Status tab.

**Counters tracked:**

| Counter | Updated by |
|---------|-----------|
| `documents_loaded_total` | `build_index()` after successful ingestion |
| `chunks_created_total` | `build_index()` after successful chunking |
| `query_count_total` | `query()` after each successful retrieval |
| `error_count_total` | `record_error()` on any pipeline exception |

**Latencies tracked (seconds, last recorded value):**

| Latency | Measured by |
|---------|------------|
| `embedding_latency_seconds` | `timer()` around `embed()` in `build_index()` |
| `retrieval_latency_seconds` | `timer()` around `retriever.query()` |
| `rerank_latency_seconds` | Available for reranker timing |

**State tracked:**

| Field | Value |
|-------|-------|
| `selected_embedding_provider` | Provider name set after `build_index()` |
| `selected_vector_store` | Store name set after `build_index()` |
| `last_error` | Most recent error message string |

**API:**

```python
from src.observability.metrics import METRICS

METRICS.to_dict()        # Returns a plain dict snapshot
METRICS.record_error(msg)  # Increments error_count, sets last_error
METRICS.reset()          # Resets all fields to initial values (use in test teardown)
```

---

## Latency Tracing (`src/observability/tracing.py`)

`timer()` is a context manager that measures wall-clock elapsed time:

```python
from src.observability.tracing import timer

with timer() as t:
    result = some_operation()

elapsed = t["elapsed"]   # float, seconds
```

Used in `RAGPipeline` to measure embedding and retrieval latencies independently.

---

## Health Checks (`src/observability/health.py`)

Health check functions return structured status dicts indicating whether
each pipeline component is in a ready state:

- Index built flag
- Selected embedding provider name
- Selected vector store name
- Error count since last reset

These are surfaced in the **System Status** tab of the Streamlit UI, allowing
operators to verify the pipeline state at a glance without inspecting logs.

---

## Docker Container Observability

The Dockerfile runs the application as a **non-root user** (`appuser`) and
exposes port 8501.

For basic container health monitoring:

```bash
# Check if Streamlit is responding
curl -s -o /dev/null -w "%{http_code}" http://localhost:8501
```

Streamlit's built-in health endpoint is available at `/_stcore/health`
on the container port.
