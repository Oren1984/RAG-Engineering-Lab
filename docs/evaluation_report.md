# Evaluation Report

## Purpose

This document describes the retrieval evaluation design for the RAG Retrieval
Evaluation Lab, defines the metrics used, and presents a baseline evaluation
run using mock embeddings.

---

## Evaluation Design

### What is being evaluated

**Retrieval quality** — given a natural-language question, does the system return
chunks from the correct source document within the top-K results?

The unit of relevance is the **document** (`doc_id`), not the individual chunk.
A question is considered answered if at least one chunk from the relevant document
appears in the ranked result list at rank ≤ K.

### What is not being evaluated in v1

- **Answer quality** — there is no LLM generation; expected answers in the ground
  truth are informational only
- **Semantic relevance ranking** — mock embeddings do not produce semantically
  meaningful rankings (see Limitations section)
- **Cross-document relevance** — the current ground truth has single-document
  relevance per question

---

## Ground Truth

| Property | Value |
|----------|-------|
| File | `data/ground_truth/questions.json` |
| Questions | 5 |
| Relevant document | `doc_001` (all five questions) |
| Format | JSON array; `relevant_doc_id` (str) or `relevant_doc_ids` (list) |

The ground truth file can be extended by adding entries following the same schema.
The `GroundTruthLoader` handles both single-string and list formats automatically.

---

## Metric Definitions

All metrics are computed over a **deduplicated** ranked list of doc_ids. When the
same document appears as multiple chunks in the results, only its **first occurrence**
is counted — this prevents inflating DCG above IDCG when a single document dominates
the top-K positions.

---

### Recall @ K

```
Recall@K = |{relevant docs} ∩ {top-K retrieved docs}| / |{relevant docs}|
```

- Returns 0.0 for empty inputs or K ≤ 0
- For single-relevant-document queries, equivalent to **Hit@K**
- Range: [0.0, 1.0]

**Interpretation:** What fraction of the relevant documents did we find in the top K?

---

### Precision @ K

```
Precision@K = |{relevant docs} ∩ {top-K retrieved docs}| / K
```

- Denominator is always K, not the number of results returned
- Returns 0.0 for empty inputs or K ≤ 0
- Range: [0.0, 1.0]

**Interpretation:** Of the K results we returned, what fraction were relevant?

---

### MRR — Mean Reciprocal Rank

```
MRR = 1 / rank_of_first_relevant_result
```

- Returns 0.0 if no relevant result is found in the list
- Range: (0.0, 1.0] — perfect score is 1.0 (first result is relevant)
- Computed over the full retrieved list, not truncated at K

**Interpretation:** How early does the first correct answer appear in the ranking?

---

### nDCG @ K — Normalised Discounted Cumulative Gain

```
DCG  = Σ ( rel_i / log₂(i+1) )    for i in 1..K   where rel_i ∈ {0, 1}
IDCG = Σ ( 1   / log₂(i+1) )     for i in 1..min(|relevant|, K)
nDCG = DCG / IDCG
```

- Returns 0.0 for empty inputs or K ≤ 0
- Range: [0.0, 1.0] — 1.0 means the ranking matches the ideal ordering
- Rewards placing relevant documents at higher ranks

**Interpretation:** How close is the actual ranking to the ideal ranking?

---

## Baseline Evaluation Results (Mock Mode)

The following results are from the Stage 8 smoke test: a single document (`doc_001`)
indexed with `MockEmbeddingProvider`, `RecursiveChunker(size=300, overlap=50)`,
and `ChromaVectorStore`, evaluated at K=5.

| Metric | Score |
|--------|-------|
| avg_recall@5 | 1.0000 |
| avg_precision@5 | 0.2000 |
| avg_MRR | 1.0000 |
| avg_nDCG@5 | 1.0000 |
| error_count | 0 |

### Interpretation of mock-mode baseline

- **Recall@5 = 1.0** — all five questions retrieve at least one chunk from `doc_001`
  within the top 5. This confirms the end-to-end pipeline is functioning correctly.
- **Precision@5 = 0.2** — at K=5 with one relevant document, the theoretical
  maximum is 1/5 = 0.2 (one relevant result out of five). This is the expected value.
- **MRR = 1.0 and nDCG@5 = 1.0** — `doc_001` appears at rank 1 for all five questions.
  This is a property of the deterministic mock hash, not semantic relevance.

> **Note:** These results reflect the behaviour of mock embeddings on a single-document
> corpus. They confirm pipeline correctness but do not constitute a meaningful
> evaluation of semantic retrieval quality.

---

## Limitations of This Evaluation

### Mock embeddings are not semantic

Mock vectors are seeded from text hashes, not trained on language. Retrieval
rankings reflect hash proximity, not meaning. A query about "chunking strategies"
and a chunk about "embedding providers" may score identically.

**Consequence:** MRR and nDCG scores with mock embeddings cannot be used to
compare chunking strategies or other configuration changes in a meaningful way.
Only Recall@K (as a binary hit check) is interpretable.

### Single-document corpus

The current ground truth evaluates against a single document (`doc_001`). This
limits the ability to measure selectivity — the system cannot distinguish between
retrieving the right document and retrieving any document.

### Expected_answer is informational only

The `expected_answer` field in each ground truth entry is not used in scoring.
There is no generative component in v1 to evaluate answer quality.

---

## Extending the Evaluation

### Using a real embedding provider

Replace `MockEmbeddingProvider` with `HuggingFaceEmbeddingProvider` to obtain
semantically meaningful rankings. See [`docs/experiment_plan.md`](experiment_plan.md)
for the provider comparison procedure.

### Adding more documents and questions

1. Add documents to `data/raw/` and index them
2. Add ground truth entries with the correct `relevant_doc_id` values
3. Re-run `RetrievalEvaluator` — no code changes required

### Multi-document evaluation

The evaluator supports `relevant_doc_ids: ["doc_001", "doc_002"]` for queries
that should be answered by any of multiple documents. The metric functions handle
multi-relevant sets correctly.

### Saving and comparing results

```python
# Run two configurations and compare
evaluator.save_results_csv(results_recursive, path='results/recursive_hf.csv')
evaluator.save_results_csv(results_fixed,     path='results/fixed_hf.csv')
```

See [`docs/experiment_plan.md`](experiment_plan.md) for a comparison workflow.
