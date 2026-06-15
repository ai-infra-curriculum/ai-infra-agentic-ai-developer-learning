# mod-104-retrieval-basics: Retrieval Basics (Embeddings & Simple RAG)

**Estimated effort:** 10 hours

This module turns the tool-calling protocol from `mod-103` into a knowledge protocol: how to put *your documents* in front of the model at query time. You will build the two pipelines every retrieval-augmented system needs — an offline ingest that chunks, embeds, and indexes your corpus, and an online query loop that retrieves the right chunks and assembles a grounded, cited answer. By the end you should be able to take a folder of markdown files and ship a small RAG app whose answers you can defend with a recall@K number.

## Learning objectives

- Generate embeddings and run similarity search.
- Build a simple retrieval-augmented generation (RAG) pipeline end to end.
- Chunk and index documents for retrieval with stable IDs and useful metadata.
- Evaluate retrieval quality at a basic level — recall@K, MRR, answer groundedness, refusal correctness.

## Lecture chapters

1. [From Tools to Knowledge](01-from-tools-to-knowledge.md) — why RAG, what the two pipelines are, how this connects to the rest of the track.
2. [Embeddings and Vector Space](02-embeddings-and-vector-space.md) — what an embedding is, similarity metrics, provider models, normalization.
3. [Similarity Search and Indexes](03-similarity-search-and-indexes.md) — brute-force NumPy at small scale; FAISS, HNSW, IVF, and pgvector at larger scale; top-K and filters.
4. [Chunking Documents](04-chunking-documents.md) — structural, recursive, and token-aware splits; sizes, overlap, and the metadata every chunk needs.
5. [The Indexing Pipeline](05-indexing-pipeline.md) — load → chunk → embed → store, with batching, idempotency, retries, and embedding-model versioning.
6. [The RAG Query Loop](06-the-rag-query-loop.md) — embed the question, retrieve top-K, assemble the prompt, return a cited answer.
7. [Evaluating Retrieval Quality](07-evaluating-retrieval-quality.md) — a small labeled eval set, recall@K, MRR, groundedness, sweeping the knobs, wiring CI.

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: Embeddings and similarity](exercises/exercise-01-embeddings-similarity.md) — embed a small corpus, rank by cosine similarity, probe the metric.
- [exercise-02: A simple RAG pipeline](exercises/exercise-02-simple-rag-pipeline.md) — ingest a markdown corpus to SQLite + NumPy, run the full query loop, return cited answers.
- [exercise-03: Chunking, indexing, and evaluation](exercises/exercise-03-chunking-and-indexing.md) — build a labeled eval set, sweep chunkers and K, pick a winner you can defend.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise. The shape of the work matters here: exercise-01 builds the embedding primitive, exercise-02 ships a working end-to-end app on top of it, and exercise-03 turns "I think it improved" into "recall@5 went from 0.61 to 0.78." The lecture chapters intentionally stay tight because the value lands when you run a real query against a real index and have to explain why the top hit was not the right one.

Bring a paid API key for at least one embeddings provider (OpenAI is the path of least resistance; Voyage, Cohere, and local Sentence Transformers also work), one answer-generation key (Anthropic or OpenAI), Python 3.10+, and a willingness to re-ingest your corpus a few times.

`mod-105` (first agent) wraps the retriever from exercise-02 in a tool the model decides when to call. `mod-106` (deployment) turns the ingest into a scheduled job and the eval set into a regression gate. The cleaner your loop and eval here, the easier those will be.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
