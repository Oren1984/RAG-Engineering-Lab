# Testing Strategy

## Current Status

The repository has a **test scaffold** in place (`tests/unit/`, `tests/integration/`,
`tests/e2e/`, each with an `__init__.py`), but no test cases have been implemented yet.
There are no stage smoke-test scripts and no pytest configuration file in the
repository at this time.

Running `pytest` today will collect zero tests. This document describes the
intended layered testing approach so that tests can be added consistently as
the project matures.

---

## Intended Layered Approach

1. **Unit tests** (`tests/unit/`) — pure function and single-component tests
2. **Integration tests** (`tests/integration/`) — multi-component tests (e.g. chunker + embedder, pipeline + vector store)
3. **End-to-end tests** (`tests/e2e/`) — full pipeline tests from a raw document to a CSV evaluation result

**All tests should run in mock mode.** No API keys, no model downloads, and no
network access should be required to run the test suite. `MockEmbeddingProvider`
makes this possible for every layer.

---

## Running Tests

```bash
# Run the full test suite
pytest

# Run with coverage report
pytest --cov=src --cov-report=term-missing

# Run a specific layer
pytest tests/unit/
pytest tests/integration/
pytest tests/e2e/
```

---

## Recommended Test Targets

### Unit Tests (`tests/unit/`)

- `src/evaluation/metrics.py` — each metric function against hand-calculated values:
  - Empty inputs return 0.0
  - `k=0` returns 0.0
  - `k > len(retrieved)` handles gracefully
  - Multi-relevant document sets
  - nDCG deduplication (same `doc_id` across chunks)
- `src/chunking/` — each chunker with:
  - Empty document text
  - Text shorter than `chunk_size` (produces one chunk)
  - `chunk_overlap >= chunk_size` raises `ChunkingError`
  - Whitespace-only text produces no chunks
- `src/ingestion/text_cleaner.py` — whitespace normalisation, PII redaction
- `src/security/query_validation.py` — empty query, query over max length, valid query passes
- `src/embeddings/mock_embedding_provider.py` — dimension, L2 norm, determinism

None of these require API keys — `MockEmbeddingProvider` produces valid vectors
without any network calls.

### Integration Tests (`tests/integration/`)

- `Chunker` + `EmbeddingProvider` — chunks from a real document get valid embeddings
- `EmbeddingProvider` + `VectorStore` — embedded chunks are stored and retrieved
- `RAGPipeline.build_index()` — full ingestion path with a real temporary file
- `RAGPipeline.query()` — returns `List[Chunk]` with `score` and `rank` populated
- `RetrievalEvaluator` + `RAGPipeline` — evaluation completes with no errors

ChromaDB integration tests should use
`tempfile.TemporaryDirectory(ignore_cleanup_errors=True)` to handle the Windows
file-lock behaviour of ChromaDB's persistent client (see
[`known_limitations.md`](known_limitations.md)).

FAISS integration tests should check:
- `ntotal == 0` before add; graceful empty-index search
- Sentinel index `-1` returned by FAISS on insufficient results handled correctly

### End-to-End Tests (`tests/e2e/`)

Reference scenario:

1. Write `doc_001.txt` to a temp directory
2. Build an index with `MockEmbeddingProvider` + `ChromaVectorStore`
3. Load `data/ground_truth/questions.json`
4. Run `RetrievalEvaluator.evaluate(ground_truth, top_k=5)`
5. Assert:
   - `error_count == 0`
   - All metric scores in `[0.0, 1.0]`
   - `recall@5 == 1.0` (all five questions find `doc_001`)
6. Assert `save_results_csv()` writes a 6-line file (header + 5 rows)

---

## Manual Smoke Checks

Until an automated suite exists, use these manual checks to validate a change.

### Docker

```bash
docker compose up --build -d
# Wait for the container to start
curl -s -o /dev/null -w "%{http_code}" http://localhost:8501
# Expected: 200
docker compose down
```

The container runs with the default `.env` settings (`MOCK_MODE=true`), so no
API keys are required.

### Streamlit

1. `streamlit run app/streamlit_app.py`
2. Open [http://localhost:8501](http://localhost:8501)
3. Upload a `.txt` file → click "Build Index" → verify success message
4. Switch to Retrieval Demo → enter a query → verify ranked results appear
5. Switch to Evaluation → click "Run Evaluation" → verify metric scores appear
6. Switch to System Status → verify METRICS counters are non-zero

All steps should complete without entering any API key.

---

## Test Isolation Rules (for future test authors)

1. **Never share state between tests.** Each test should create its own temp
   directory and call `METRICS.reset()` (`src/observability/metrics.py`) in teardown.
2. **Never write to `data/` or `vector_db/` in tests.** Use `tempfile.mkdtemp()`
   or `tempfile.TemporaryDirectory()`.
3. **Never commit `.env` with real API keys.** Tests must pass with the defaults
   in `.env.example`.
4. **Never skip cleanup on Windows.** Use `ignore_cleanup_errors=True` on
   `TemporaryDirectory` for ChromaDB tests.

---

## Suggested Coverage Goals (once tests exist)

| Layer | Target coverage |
|-------|----------------|
| `src/evaluation/metrics.py` | 100% |
| `src/evaluation/ground_truth_loader.py` | 100% |
| `src/evaluation/evaluator.py` | > 90% |
| `src/chunking/` | > 90% |
| `src/embeddings/mock_embedding_provider.py` | 100% |
| `src/security/` | > 85% |
| `src/pipeline/rag_pipeline.py` | > 85% |
| `src/vectorstores/` | > 80% |
