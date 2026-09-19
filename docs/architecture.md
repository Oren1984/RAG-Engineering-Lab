# Architecture

## System Overview

The RAG Engineering Lab is structured as a **five-layer pipeline** with a
thin orchestrator (`RAGPipeline`) wiring each layer together. All layers are
defined by abstract base classes in `src/core/contracts.py` and are individually
replaceable without touching other layers.

```
src/
├── core/            # Contracts (ABCs), DTOs, exceptions, config
├── ingestion/       # Document loading and text cleaning
├── chunking/        # Text splitting strategies
├── embeddings/      # Embedding providers
├── vectorstores/    # Vector store backends
├── retrieval/       # Retriever, reranker
├── pipeline/        # RAGPipeline orchestrator
├── evaluation/      # Metrics, ground truth loader, evaluator
├── security/        # Input validation and safety
└── observability/   # Metrics singleton, tracing, health
```

---

## Layer Descriptions

### 1. Ingestion Layer (`src/ingestion/`)

**`DocumentLoader`** reads a file from disk and returns a `Document` dataclass.

```
load_document(file_path) -> Document(doc_id, text, source, file_type, metadata)
```

- `doc_id` is derived from the file stem (filename without extension)
- Supported formats: `.txt` (built-in), `.pdf` (pypdf), `.html` / `.htm` (BeautifulSoup)
- Unsupported formats raise `IngestionError`

**`TextCleaner`** normalises the document text before chunking.

```
clean_text(text, redact_pii=False) -> str
```

- Collapses runs of whitespace
- Strips leading/trailing whitespace
- Optionally redacts PII patterns via `src/security/pii_redaction.py`

---

### 2. Chunking Layer (`src/chunking/`)

Both chunkers implement the `Chunker` ABC: `chunk(document) -> List[Chunk]`.

**`RecursiveChunker`** splits text on progressively finer natural boundaries:

1. Paragraph breaks (`\n\n`)
2. Line breaks (`\n`)
3. Sentence endings (`. `, `! `, `? `)
4. Word spaces (` `)
5. Characters (last resort)

Small pieces are merged into windows of approximately `chunk_size` characters,
with the tail of each window carried over as overlap into the next.

**`FixedChunker`** splits at a fixed character count with configurable overlap.
Useful as a controlled baseline comparison against `RecursiveChunker`.

Both accept `chunk_size` and `chunk_overlap` constructor parameters.
`chunk_overlap` must be strictly less than `chunk_size`.

Each returned `Chunk` carries: `chunk_id`, `doc_id`, `text`, `metadata`.
The `embedding`, `score`, and `rank` fields are `None` until set by later layers.

---

### 3. Embedding Layer (`src/embeddings/`)

All providers implement the `EmbeddingProvider` ABC:

```python
def embed(self, texts: List[str]) -> List[List[float]]: ...
def provider_name(self) -> str: ...     # property
def dimension(self) -> int: ...         # property
```

| Provider | Class | Dimension | Key Required |
|----------|-------|-----------|-------------|
| Mock | `MockEmbeddingProvider` | 384 | No |
| HuggingFace | `HuggingFaceEmbeddingProvider` | model-dependent | No |
| OpenAI | `OpenAIEmbeddingProvider` | model-dependent | Yes (`OPENAI_API_KEY`) |
| Gemini | `GeminiEmbeddingProvider` | model-dependent | Yes (`GEMINI_API_KEY`) |

All providers are lazy-importing — heavy dependencies are only loaded when the
provider is actually instantiated. This keeps startup time fast in mock mode.

#### Why Mock Mode Exists

Real embedding providers have two drawbacks for development:

1. **OpenAI / Gemini** — require API keys and incur per-call cost
2. **HuggingFace sentence-transformers** — require downloading a model (~90 MB+) on first use

`MockEmbeddingProvider` solves both by using a deterministic SHA-256-seeded
NumPy RNG to produce L2-normalised 384-dim vectors. The same text always produces
the same vector, making tests and evaluations reproducible without any external
dependencies.

> **Important:** Mock vectors do not capture semantic similarity. They are intended
> for pipeline correctness testing, not semantic retrieval experiments.

---

### 4. Vector Store Layer (`src/vectorstores/`)

Both stores implement the `VectorStore` ABC:

```python
def add(self, chunks: List[Chunk]) -> None: ...
def search(self, query_embedding: List[float], top_k: int) -> List[Chunk]: ...
def persist(self) -> None: ...
def load(self) -> None: ...
def clear(self) -> None: ...
def store_name(self) -> str: ...    # property
```

**`ChromaVectorStore`** wraps ChromaDB's `PersistentClient`:
- Collection space: `hnsw:space: cosine`
- Score = `1.0 - cosine_distance` (range −1.0 to 1.0)
- `add()` uses `upsert` — re-indexing the same `chunk_id` is idempotent
- Auto-persists to disk; `persist()` is a no-op
- Metadata is flattened to str/int/float/bool before storage (None values dropped)
- `doc_id` is injected into metadata on `add()` and extracted on `search()`

**`FaissVectorStore`** wraps `faiss.IndexFlatIP` (inner product):
- For L2-normalised vectors, inner product equals cosine similarity
- Maintains an in-memory `_sidecar: List[Dict]` for metadata, indexed by FAISS integer position
- `persist()` writes the FAISS index file + JSON sidecar to disk
- `load()` reads both files; graceful no-op if files do not exist
- Handles FAISS sentinel index value `-1` gracefully on search

---

### 5. Retrieval Layer (`src/retrieval/`)

**`Retriever`** is the dense retrieval component:

```
query(query_text, top_k) -> List[Chunk]
```

1. Validates `query_text` and `top_k` via `src/security/query_validation.py`
2. Embeds `query_text` using the same `EmbeddingProvider` instance used at index time
3. Calls `vector_store.search(query_embedding, top_k)`
4. Optionally applies a `Reranker` if one was provided

**Rerankers** implement `rerank(query, chunks) -> List[Chunk]`:

- `PassThroughReranker` — returns chunks unchanged (v1 default)
- `CrossEncoderReranker` — lazy-loads a `CrossEncoder` from sentence-transformers;
  re-scores and re-sorts chunks by descending cross-encoder score

**`HybridRetriever`** is a documented stub — hybrid sparse+dense retrieval is
planned for a future version.

---

### 6. Pipeline Orchestrator (`src/pipeline/`)

`RAGPipeline` wires all layers together and maintains pipeline state:

```python
class RAGPipeline:
    def build_index(self, file_path: str, redact_pii: bool = False) -> Dict: ...
    def query(self, query_text: str, top_k: int) -> List[Chunk]: ...
    def clear(self) -> None: ...
    def mark_ready(self) -> None: ...
```

Key design decisions:

- The **same `EmbeddingProvider` instance** is passed to both `build_index()` and
  the internal `Retriever`, guaranteeing embedding-space consistency
- `_index_built` flag prevents querying before an index is built
- `mark_ready()` allows bypassing the flag for pre-loaded vector stores
- `build_index()` updates `METRICS` after each successful operation
- All exceptions are recorded via `METRICS.record_error()` before re-raising

---

### 7. Evaluation Layer (`src/evaluation/`)

**`GroundTruthLoader`** reads a JSON file and returns `List[GroundTruthEntry]`.

Each entry has `question`, `expected_answer`, and `relevant_doc_ids: List[str]`.
The loader normalises both `relevant_doc_id` (single string) and `relevant_doc_ids`
(list) JSON formats.

**`RetrievalEvaluator`** accepts any object with `query(text, top_k) -> List[Chunk]`:

1. Issues each question against the pipeline
2. Extracts `doc_id` from each returned chunk
3. **Deduplicates doc_ids by first occurrence** before computing metrics —
   without this, the same document appearing as multiple chunks inflates nDCG above 1.0
4. Computes Recall@K, Precision@K, MRR, and nDCG@K per question
5. Returns per-question result dicts and an averaged summary dict

**Metric functions** in `src/evaluation/metrics.py` are pure, side-effect-free
functions with no imports from the rest of the project.

---

### 8. Shared Contracts (`src/core/contracts.py`)

All inter-layer data transfer uses these dataclasses:

| DTO | Fields |
|-----|--------|
| `Document` | `doc_id`, `text`, `source`, `file_type`, `page_number`, `metadata` |
| `Chunk` | `chunk_id`, `doc_id`, `text`, `metadata`, `embedding`, `score`, `rank` |
| `EvaluationQuestion` | `question`, `expected_answer`, `relevant_doc_id` |

ABCs defined here: `EmbeddingProvider`, `VectorStore`, `Chunker`, `Reranker`.

---

### 9. Security Layer (`src/security/`)

| Module | Responsibility |
|--------|---------------|
| `file_validation.py` | Validates upload size and file extension before write |
| `query_validation.py` | Validates query text length and content; raises `ValidationError` |
| `prompt_safety.py` | Logs warnings for potentially unsafe prompt patterns |
| `pii_redaction.py` | Optionally redacts PII patterns from document text |

---

### 10. Observability Layer (`src/observability/`)

| Module | Responsibility |
|--------|---------------|
| `metrics.py` | `METRICS` singleton — counters, latencies, provider/store state |
| `tracing.py` | `timer()` context manager — wall-clock latency measurement |
| `health.py` | Health and readiness check functions |

---

## Provider Pattern

The provider pattern is used throughout the system. Each abstract interface lives
in `src/core/contracts.py`. Concrete implementations are in their respective
`src/<layer>/` packages.

New providers can be added by:
1. Implementing the relevant ABC
2. Adding the provider to the factory function in `app/streamlit_app.py`

No changes to the pipeline, evaluator, or metric code are required.

---

## Streamlit UI Layer (`app/streamlit_app.py`)

The UI is a single-file Streamlit application with four tabs:

| Tab | Purpose |
|-----|---------|
| Ingestion | Upload document, configure chunker, build or clear index |
| Retrieval Demo | Enter a query, run it, inspect ranked results |
| Evaluation | Load ground truth, run `RetrievalEvaluator`, download CSV |
| System Status | Live `METRICS` snapshot, pipeline health indicators |

Sidebar controls configure the embedding provider, vector store, chunking strategy,
chunk size, chunk overlap, top-K, and reranker. A config-change warning prompts the
user to rebuild the index if settings have changed since the last build.

All pipeline state is stored in `st.session_state` so it persists across tab switches.
