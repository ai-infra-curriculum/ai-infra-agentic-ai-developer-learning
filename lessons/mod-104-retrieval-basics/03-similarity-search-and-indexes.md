# Similarity Search and Indexes

## Why this matters

In chapter 2 you embedded a handful of strings and ranked them with a four-line NumPy dot product. That works up to roughly tens of thousands of vectors. After that, "compare the query to every chunk" stops being a free operation: a million 1536-dimensional float32 vectors is ~6 GB of math per query, and you do not want to do that on every request.

Vector indexes exist for exactly this scaling step. The query-side mental model stays the same — embed the question, find the K nearest chunks — but the *how* changes from "scan everything" to "use a data structure that gets to the answer without scanning everything." This chapter is about which structure, when, and the trade-offs they bake in.

## Two kinds of search, one decision

There are exactly two families of vector search:

- **Exact (brute-force) search.** Compare the query against every vector. Guaranteed to return the true top-K. Cost is `O(N · d)` per query.
- **Approximate Nearest Neighbor (ANN) search.** Use an index structure to skip most of the corpus. Returns "almost always the true top-K." Cost is sub-linear; recall is a tunable parameter.

You pick by corpus size:

| Corpus size              | What to use                                          |
|--------------------------|------------------------------------------------------|
| Up to ~10k chunks        | NumPy brute-force in a pickle file. Done.            |
| ~10k – ~1M               | FAISS `IndexFlatIP` or `IndexHNSWFlat`, or pgvector with `ivfflat` / `hnsw`. |
| ~1M – ~100M              | ANN with HNSW or IVF + product quantization. |
| 100M+                    | Sharded ANN, hosted vector DB, hardware acceleration. Out of scope here. |

For this module we work in the first two bands. Most production RAG systems sit there as well. The exercises use brute-force NumPy and one ANN library so you can feel the trade-off.

## Brute-force search in NumPy

The simplest possible "index" is a 2-D array.

```python
import numpy as np

class FlatIndex:
    """Brute-force cosine search on L2-normalized float32 vectors."""

    def __init__(self, dim: int):
        self.dim = dim
        self.vectors: np.ndarray | None = None  # (N, dim)
        self.ids: list[str] = []

    def add(self, ids: list[str], vectors: np.ndarray) -> None:
        assert vectors.shape[1] == self.dim
        v = vectors.astype(np.float32, copy=False)
        self.vectors = v if self.vectors is None else np.vstack([self.vectors, v])
        self.ids.extend(ids)

    def search(self, query: np.ndarray, k: int = 5) -> list[tuple[str, float]]:
        assert self.vectors is not None
        scores = self.vectors @ query.astype(np.float32)   # (N,)
        top = np.argpartition(-scores, kth=min(k, len(scores) - 1))[:k]
        top = top[np.argsort(-scores[top])]
        return [(self.ids[i], float(scores[i])) for i in top]
```

That is enough for any prototype with under ten thousand chunks, and it is what you will use in exercise-01. The numbers stay honest, the code is debuggable, and you can swap it out without changing the rest of the pipeline.

Performance trick: use `np.argpartition` for top-K instead of `np.argsort` of the full array. The partition is `O(N)` for a fixed K; the full sort is `O(N log N)`.

## ANN libraries and indexes

When brute-force gets slow, you swap in an index data structure. Three families dominate.

### HNSW (Hierarchical Navigable Small World)

A multi-layer graph where each layer is a navigable small-world graph. Queries enter at the top, descend to neighbors closer to the query, and bottom out in the K nearest. In practice:

- Fast query latency at high recall.
- Higher memory footprint (you store the graph alongside the vectors).
- Build cost is non-trivial; updates are insertions into a graph.

Implementations: `faiss.IndexHNSWFlat`, `hnswlib`, the `hnsw` index type in pgvector, the HNSW backends inside most managed vector DBs. <!-- needs-research: confirm pgvector HNSW is the recommended default vs ivfflat in current pgvector docs at https://github.com/pgvector/pgvector -->

### IVF (Inverted File)

Cluster the corpus into `nlist` centroids (k-means). At query time, pick the `nprobe` closest centroids and search only their cells. Trade-off:

- Lower memory than HNSW.
- Requires a *training* step on a sample of vectors before adding data.
- Recall depends on `nprobe`; raise it for better recall at the cost of latency.

Implementations: `faiss.IndexIVFFlat`, pgvector's `ivfflat`. <!-- needs-research: confirm IVF is still the second-most-common index after HNSW in recent surveys -->

### Quantization (PQ, SQ)

Compress each vector — for example, replace each 1536-D float32 vector (6 KB) with a sequence of small integer codes (under 100 bytes). Pairs with IVF (`IndexIVFPQ`) or HNSW (`IndexHNSWPQ`). Cuts memory by ~10–100× at the cost of some recall.

You do not need quantization until you actually do not fit in RAM. Skip it for this module.

## FAISS in practice

[FAISS](https://github.com/facebookresearch/faiss) is the de facto open-source ANN library. The API is small:

```python
import faiss
import numpy as np

dim = 1536
index = faiss.IndexFlatIP(dim)         # exact inner product (brute force on a C++ side)
# index = faiss.IndexHNSWFlat(dim, 32) # HNSW with M=32 graph degree

vectors = np.random.randn(50_000, dim).astype(np.float32)
faiss.normalize_L2(vectors)            # IP on unit vectors == cosine
index.add(vectors)

q = np.random.randn(1, dim).astype(np.float32)
faiss.normalize_L2(q)
scores, ids = index.search(q, k=5)     # ids: int row indices; map to your own string IDs
```

A few things to know:

- **IDs are integers row-positions by default.** You almost always want a string ID (e.g., `chunk_id`). Wrap the FAISS index in your own class that keeps a parallel list of string IDs, or use `IndexIDMap` to attach 64-bit ints and map those to strings outside.
- **`IndexFlatIP` is brute force on the C++ side.** It is dramatically faster than NumPy at the same exact-search guarantee. For corpora of a few hundred thousand it may still be your best move.
- **`IndexHNSWFlat` parameters matter.** `M` (graph degree) and `efConstruction` (build effort) trade memory and build time for recall; `efSearch` at query time trades latency for recall. Read the FAISS wiki for the defaults; the exercise tells you exactly which to start with.
- **Saving and loading.** `faiss.write_index(index, path)` and `faiss.read_index(path)` persist a built index. You still need to persist your string-ID map and chunk text separately (chapter 5).

## pgvector and "the database already has it"

If you already run Postgres, the smallest-effort path is `pgvector`: a Postgres extension that adds a `vector` column type and `<->`/`<=>`/`<#>` distance operators, with optional `ivfflat` and `hnsw` indexes.

```sql
CREATE EXTENSION vector;

CREATE TABLE chunks (
    id           TEXT PRIMARY KEY,
    document_id  TEXT NOT NULL,
    chunk_text   TEXT NOT NULL,
    embedding    VECTOR(1536) NOT NULL,
    embed_model  TEXT NOT NULL
);

-- Cosine similarity ordering, top 5:
SELECT id, chunk_text, 1 - (embedding <=> $1) AS score
FROM chunks
WHERE embed_model = 'text-embedding-3-small'
ORDER BY embedding <=> $1
LIMIT 5;

-- Optional ANN index for scale:
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
```

What you get for "free":

- Filters compose naturally: `WHERE tenant_id = $2 AND embed_model = '…' ORDER BY embedding <=> $1` is one query and Postgres figures it out.
- Joins to your application tables work the same way they always do.
- One backup strategy.

The trade-off is that pgvector at very large scale (tens of millions of high-dim vectors) is less tuned than dedicated vector DBs. For the band this module cares about, it is an excellent default. <!-- needs-research: re-check current pgvector size/perf advice at https://github.com/pgvector/pgvector and https://www.postgresql.org/docs/current/ -->

Hosted vector databases — Pinecone, Weaviate, Qdrant, Milvus, Chroma — fill the same role with managed infrastructure, hybrid search baked in, and per-call APIs. The protocol your code presents to the rest of the system is the same: `add(id, vector, metadata)`, `search(vector, k, filter=…) -> list[hit]`. Treat the choice as an infra decision, not an algorithmic one.

## Top-K and `k`

Every search returns the K most similar chunks. K is a tuning knob:

- **K = 1** is rarely right. One chunk gives the model no room to cross-check or quote.
- **K = 3–5** is the common sweet spot for short Q&A on focused corpora.
- **K = 8–20** for broader corpora or when chunks are small.
- Very large K wastes prompt tokens and dilutes signal. The model has to work harder to find the right snippet among the noise.

Chapter 6 lays out the prompt-assembly side; chapter 7 measures whether your K is set right. For now, treat K as something you will sweep during evaluation, not as a fixed law.

## Filters and metadata

A pure vector search is "what is most similar." Real applications usually want "what is most similar *and matches some structural constraint*" — same tenant, same product line, last 90 days, a specific document ID.

Two ways to compose:

- **Pre-filter.** Restrict the candidate set first, then rank by similarity. Cheap when the filter is selective and you have an index on it (Postgres `WHERE tenant_id = …`).
- **Post-filter.** Rank by similarity first, then drop hits that fail the filter. Cheap when the filter rejects only a small fraction; can return fewer than K results if it rejects a lot.

pgvector handles pre-filter naturally with `WHERE`. FAISS supports filter expressions on selected index types but the ergonomics are worse; teams often add filterable metadata in a sidecar dict and post-filter. Hosted vector DBs make filters first-class. <!-- needs-research: confirm current FAISS filter support story at https://github.com/facebookresearch/faiss/wiki -->

The right answer is "pre-filter when you can"; the cheap answer is "post-filter and hope K is large enough." Chapter 5 lays out the metadata schema you will want.

## Recall, latency, and the trade-off

ANN buys speed at the cost of *some* recall — occasionally it misses a true top-K result. You manage this with a measurable number: at index build time and query time, compare ANN results against brute-force on a sample of queries. The metric is `recall@K = |ANN_topK ∩ Brute_topK| / K`, averaged across queries.

Useful operating points to know:

- `recall@K = 1.0` is exact search. If you can afford it, take it.
- `recall@K ≥ 0.95` is what most ANN parameter defaults aim for; usually fine for RAG.
- `recall@K = 0.80` is "you have noticeably tuned for speed." Validate against the eval set from chapter 7.

The relationship between an index's tunable knobs (HNSW's `efSearch`, IVF's `nprobe`) and recall@K is monotonic and easy to plot. When in doubt, plot it.

## Common mistakes

- **Mixing exact and approximate without measuring recall.** You "upgraded to FAISS" and answer quality fell. Recall@K dropped silently. Always measure.
- **Letting FAISS's integer row IDs leak into your application code.** Use your own string `chunk_id` everywhere outside the FAISS call.
- **Picking an HNSW `M` of 4 because the example used 4.** That was the example. Read the FAISS wiki.
- **Confusing dot product with cosine on un-normalized vectors.** Normalize at ingest (chapter 5). Then `IndexFlatIP` is cosine, full stop.
- **Building an ANN index, then ingesting the same vectors back into a brute-force baseline for "checking."** That brute-force baseline is your eval; build it first, keep it forever, and use it as the truth for recall measurement.

## Summary

Similarity search is a single primitive — "find the K nearest chunks to this query vector" — implemented two ways: exactly (brute force, slow at scale) or approximately (HNSW or IVF, fast but recall-tunable). Up to a few tens of thousands of chunks, NumPy is enough; beyond that, FAISS or pgvector is the lowest-friction step up. Wrap whatever you use behind a small `Index.search(query, k, filter)` interface — chapter 5 builds the ingest pipeline that feeds it, chapter 6 builds the query loop that calls it, and chapter 7 measures whether it works.
