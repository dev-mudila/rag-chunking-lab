# Architecture & Reading Guide

A map of the codebase for anyone reading it for the first time. The whole lab is
small (~1000 lines of Python) and built around a single pipeline:

```
PDF  →  chunk  →  embed  →  store  →  retrieve  →  generate
```

Read the code in the order that data flows through it (bottom-up). Every higher
layer just composes the layers below it.

---

## The data contracts (learn these first)

Three dictionary shapes flow through the entire system. Pin these down and
everything else reads easily.

**1. Page** — produced by `shared/loader.py`, consumed by every chunker:

```python
{"page_content": str, "metadata": {"source": str, "page": int}}
```

**2. Chunk** — produced by every chunker, consumed by every store:

```python
{"text": str, "metadata": {"source": ..., "chunker": ..., "section"?: ..., "oversized"?: ...}}
```

**3. Search hit** — produced by every store's `search()`:

```python
{"text": str, "metadata": dict, "score": float}
```

> ⚠️ **Score caveat:** for FAISS and Qdrant a higher score = more similar
> (cosine/inner product). For Chroma, `score` is a *distance* (lower = more
> similar). The eval metrics don't use scores, but keep this in mind if you
> ever rank or threshold on them.

---

## Reading order

### Phase 1 — Orientation (skim)
1. **[README.md](README.md)** — the map and the runnable commands.
2. **[RAG_LESSONS.md](RAG_LESSONS.md)** — the teaching narrative; *why* each piece exists.

### Phase 2 — Foundation (`shared/`)
3. **[shared/loader.py](shared/loader.py)** (19 lines) — PDF → list of **Page** dicts. Defines data contract #1.
4. **[shared/embedder.py](shared/embedder.py)** (24 lines) — text → normalized vectors via `all-mpnet-base-v2`. Lazy singleton model; `normalize_embeddings=True` is why inner product == cosine downstream.

### Phase 3 — Chunking (the heart of the lab)
Every chunker exposes the same `chunk(pages, chunk_size=800, chunk_overlap=80)`
signature and returns **Chunk** dicts.

5. **[chunking/recursive.py](chunking/recursive.py)** — the canonical chunker. Tries separators `\n\n → \n → " " → char`, strictly respects `chunk_size`. Start here.
6. **[chunking/character.py](chunking/character.py)** — simplest; splits on one separator and *ships oversized chunks as-is* (tags them `oversized: True`). Contrast with recursive.
7. **[chunking/section_wise.py](chunking/section_wise.py)** — regex header detection; joins pages per-paper so sections survive page breaks; adds `section` to metadata.
8. **[chunking/semantic.py](chunking/semantic.py)** — splits on embedding-similarity drops (boundary where similarity < mean − 1·std). The "smart" but slow one (embeds during chunking).
9. **[chunking/compare.py](chunking/compare.py)** — runnable: `python chunking/compare.py`.

### Phase 4 — Vector stores (`vectordb/`)
Every store exposes the same interface — learn it once from FAISS:
`add(chunks, embeddings)`, `search(query_embedding, k=5) -> [hit]`, `.count`.

10. **[vectordb/faiss_store.py](vectordb/faiss_store.py)** (47 lines) — `IndexFlatIP`, brute-force exact search. The simplest store; learn the interface here.
11. **[vectordb/qdrant_store.py](vectordb/qdrant_store.py)** — HNSW, in-memory by default; adds metadata filtering (`section_filter`, `source_filter`).
12. **[vectordb/chroma_store.py](vectordb/chroma_store.py)** — HNSW via hnswlib; embedded DB; note the distance-as-score behavior.
13. **[vectordb/compare.py](vectordb/compare.py)** — runnable: `python vectordb/compare.py`.

### Phase 5 — Evaluation (`eval/`)
14. **[eval/golden_set.json](eval/golden_set.json)** — ground truth: `{question, evidence, paper}` records.
15. **[eval/metrics.py](eval/metrics.py)** — `recall_at_k` (is the evidence string in any top-k chunk?) and `reciprocal_rank` (1/rank of first hit). Substring match, case-insensitive.
16. **[eval/run_eval.py](eval/run_eval.py)** — runs every chunker × every store against the golden set.

### Phase 6 — Putting it together
17. **[rag/pipeline.py](rag/pipeline.py)** — end-to-end CLI. `build_pipeline()` composes loader → chunker → embedder → store; `ask()` does retrieve → build prompt → (optional) LLM call.
18. **[app.py](app.py)** — read **last**. Gradio UI (6 tabs) wrapping everything above. Once you know the pieces, this is just wiring.

---

## How the layers compose

```
                 ┌─────────────────────────────────────────────┐
 app.py  ───────▶│  rag/pipeline.py · eval/run_eval.py          │   (orchestration)
                 └───────────────┬─────────────────────────────┘
                                 │
        ┌────────────────┬───────┴────────┬──────────────────┐
        ▼                ▼                 ▼                  ▼
   chunking/*       vectordb/*         eval/metrics      eval/golden_set.json
   (Page→Chunk)     (Chunk→hit)        (hit→score)       (ground truth)
        │                │
        └──────┬─────────┘
               ▼
         shared/loader.py   (PDF → Page)
         shared/embedder.py (text → vector)
```

**The thesis of the lab:** retrieval accuracy (Recall@5, MRR) is decided by the
**chunker**, while the **vector DB** decides operational characteristics
(latency, filtering, persistence) — recall is ~identical across stores on the
same chunks.
