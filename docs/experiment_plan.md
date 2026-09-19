# Experiment Plan

This document describes how to design and run controlled retrieval experiments
using the RAG Engineering Lab.

---

## Experimental Variables

The pipeline has four independently configurable dimensions:

| Dimension | Options | Config key |
|-----------|---------|-----------|
| Chunking strategy | `recursive`, `fixed` | `CHUNK_STRATEGY` |
| Chunk size | integer (characters) | `CHUNK_SIZE` |
| Chunk overlap | integer (characters) | `CHUNK_OVERLAP` |
| Embedding provider | `mock`, `huggingface`, `openai`, `gemini` | `EMBEDDING_PROVIDER` |
| Vector store | `chroma`, `faiss` | `VECTOR_STORE` |
| Top-K | integer | `TOP_K` |

A controlled experiment holds all but one variable constant, rebuilds the index,
and re-runs evaluation. Results are compared using the four IR metrics.

---

## Comparing Chunking Strategies

### Hypothesis

`RecursiveChunker` preserves natural text boundaries (paragraphs, sentences),
which may improve retrieval quality compared to `FixedChunker` when answers
span natural linguistic units.

### Setup

Use the same document, same embedding provider, same vector store, same `TOP_K`.
Vary only the chunking strategy and optionally `chunk_size` / `chunk_overlap`.

### Recommended baseline

| Parameter | Value |
|-----------|-------|
| `CHUNK_SIZE` | 500 |
| `CHUNK_OVERLAP` | 100 (20% of chunk_size) |
| `TOP_K` | 5 |
| `EMBEDDING_PROVIDER` | `huggingface` (for semantic results) |
| `VECTOR_STORE` | `chroma` |

### Metric to watch

- **nDCG@K** — sensitive to rank position; favours configurations that place
  the relevant chunk near the top
- **Recall@K** — checks whether the document is found at all within top-K

### Note on chunk overlap

The recommended overlap baseline is **10–20 % of `chunk_size`**. For example:

| `chunk_size` | Recommended `chunk_overlap` |
|-------------|---------------------------|
| 300 | 30–60 |
| 500 | 50–100 |
| 1000 | 100–200 |

Overlap larger than 20 % increases index size without proportional recall gain.

---

## Comparing Embedding Providers

### Hypothesis

Semantic embedding providers (HuggingFace, OpenAI, Gemini) produce vectors that
capture meaning, leading to higher MRR and nDCG compared to mock embeddings.

### Setup

Hold chunking strategy, chunk size, vector store, and `TOP_K` constant.
Vary only `EMBEDDING_PROVIDER`.

> **Important:** You must rebuild the index for each provider change. The
> `EmbeddingProvider` used at index time must match the one used at query time.
> `RAGPipeline` enforces this by using the same instance for both.

### Provider comparison matrix

| Provider | Semantic? | Cost | Setup |
|----------|-----------|------|-------|
| `mock` | No | Free | None |
| `huggingface` | Yes | Free | `pip install sentence-transformers` + first-use download |
| `openai` | Yes | Per API call | `OPENAI_API_KEY` in `.env` |
| `gemini` | Yes | Per API call | `GEMINI_API_KEY` in `.env` |

### Expected results

With the current five-question ground truth set (all questions about `doc_001`):

- `mock`: Recall@5 should be 1.0 (plumbing works), but MRR and nDCG will be low
- `huggingface`: MRR and nDCG should be meaningfully higher than mock
- `openai` / `gemini`: comparable to HuggingFace; differences depend on model choice

---

## Comparing ChromaDB vs FAISS

### Hypothesis

For correctly normalised vectors, ChromaDB cosine and FAISS inner-product search
should return nearly identical ranked results. Any differences reflect implementation
details (tie-breaking, floating-point precision, HNSW approximate search in Chroma).

### Setup

Hold all other variables constant. Vary only `VECTOR_STORE`.

### Key implementation differences

| Property | ChromaDB | FAISS |
|----------|----------|-------|
| Search algorithm | HNSW (approximate) | Flat (exact) |
| Similarity measure | Cosine (`1 - distance`) | Inner product |
| Persistence | Automatic | Requires `persist()` |
| Metadata storage | Built-in | JSON sidecar file |
| Available without GPU | Yes | Yes (faiss-cpu) |

For small corpora (< 10,000 chunks), the results should be effectively identical.
FAISS exact search may be preferred when reproducibility is critical.

---

## Ground Truth Questions

The default ground truth file is `data/ground_truth/questions.json`.

### Current content (5 questions)

All five questions are about `doc_001` — the sample document covering chunking
strategies, embedding providers, vector stores, and mock mode:

| # | Question | Relevant doc |
|---|----------|-------------|
| 1 | What is the recommended chunk overlap baseline? | `doc_001` |
| 2 | What chunking strategies does the system support? | `doc_001` |
| 3 | Which embedding providers are available? | `doc_001` |
| 4 | What vector stores are supported? | `doc_001` |
| 5 | How does mock mode work? | `doc_001` |

### Format

```json
[
  {
    "question": "...",
    "expected_answer": "...",
    "relevant_doc_id": "doc_001"
  }
]
```

The loader also accepts `"relevant_doc_ids": ["doc_001", "doc_002"]` for
multi-document relevance.

### Extending the ground truth

To add new questions:
1. Add a document to `data/raw/` and build an index
2. Add entries to `questions.json` with the correct `relevant_doc_id`
   (matching the file stem of the indexed document)
3. Re-run the evaluation

---

## Evaluation CSV Output

`RetrievalEvaluator.save_results_csv()` writes to `results/evaluation_results.csv`.

### Column schema

| Column | Type | Description |
|--------|------|-------------|
| `question` | str | The query issued |
| `expected_answer` | str | Reference answer (informational) |
| `relevant_doc_ids` | str | Pipe-separated relevant doc IDs |
| `retrieved_doc_ids` | str | Pipe-separated retrieved doc IDs |
| `retrieved_chunk_ids` | str | Pipe-separated retrieved chunk IDs |
| `recall_at_k` | float | Recall@K score |
| `precision_at_k` | float | Precision@K score |
| `mrr` | float | MRR score |
| `ndcg_at_k` | float | nDCG@K score |
| `top_k` | int | K used for this evaluation |
| `error` | str / null | Error message if retrieval failed |

List fields are serialised as pipe-separated strings for spreadsheet compatibility.

### Comparing runs

To compare two configurations, save results with different filenames and use
pandas or a spreadsheet to compare the score columns:

```python
import pandas as pd
df1 = pd.read_csv('results/run_recursive_hf.csv')
df2 = pd.read_csv('results/run_fixed_hf.csv')
comparison = df1[['question', 'recall_at_k', 'mrr', 'ndcg_at_k']].copy()
comparison['ndcg_fixed'] = df2['ndcg_at_k']
print(comparison)
```
