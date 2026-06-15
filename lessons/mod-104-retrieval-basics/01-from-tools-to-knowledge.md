# From Tools to Knowledge

## Why this matters

In `mod-103` you taught the model to *act*: it can call `get_order_status` and know where ORD-1001 is right now. But ask the same model "what is our return policy for opened electronics?" and you are back to guessing. The model knows generic e-commerce returns from training; it does not know *your* policy unless you put your policy in front of it.

You have two ways to do that. You can paste the policy into every prompt and pay for those tokens every turn — fine for a few paragraphs, broken once you have hundreds of pages. Or you can fetch the *relevant* paragraph from your docs at query time and put only that into the prompt. That second pattern is **retrieval-augmented generation** (RAG), and it is the topic of this module.

This is also where most "LLM apps over our docs" projects live in production. Chatbots over a knowledge base, support copilots over a ticket archive, code assistants over a private codebase — almost all of them are RAG underneath.

## What RAG actually is

A RAG system is three steps wrapped around the model call you already know:

```
user question
   |
   v
[ embed the question ]
   |
   v
[ search a vector index for the K most similar chunks ]
   |
   v
[ assemble a prompt: system + retrieved chunks + question ]
   |
   v
[ model.create(messages=...) ]
   |
   v
answer (ideally with citations back to the chunks)
```

Behind the scenes, before any user shows up, you have already done an *offline* ingest pass:

```
docs (markdown, pdfs, html, tickets, ...)
   |
   v
[ chunk into ~200–800 token pieces with metadata ]
   |
   v
[ embed each chunk -> vector ]
   |
   v
[ store (vector, chunk text, metadata) in an index ]
```

Two pipelines, both small. The offline pipeline turns your corpus into a searchable vector index. The online pipeline turns a user question into an embedding, fetches the closest chunks, and stuffs them into a prompt. Get those two right and you have 80% of the systems shipping today.

## Why not just paste the whole corpus into the prompt?

The "stuff everything in context" instinct is reasonable and it does work for a small corpus. It breaks down on four axes:

- **Cost.** Input tokens are billed every turn. A 200-page handbook is roughly 100k tokens. At even a fraction of a cent per thousand input tokens, you are paying that fraction *per request*, forever.
- **Latency.** Long prompts take longer to process. A 100k-token prompt is measurably slower than a 5k-token one even when the model accepts both.
- **Attention.** Models attend to the whole context, but they do not attend *evenly*. Information buried in the middle of a long context is empirically harder for models to use than information near the start or end. <!-- needs-research: cite the "lost in the middle" paper or current provider guidance on long-context retrieval performance -->
- **Freshness and access.** Some of what the user wants is not in any static doc — it is in a ticketing system, a wiki that updates daily, a database. You want a query-time fetch, not a baked-in copy.

RAG addresses all four: you pay tokens only for the chunks you actually need, the prompt stays small and fast, the retrieved chunks land near the question where the model is most likely to use them, and the index can be refreshed without redeploying the prompt.

Long-context models do not make RAG obsolete; they raise the ceiling on how much you *can* stuff and lower the urgency of being clever about it. For anything that does not fit comfortably *and cheaply*, retrieval is still the right pattern.

## Why not just keyword search?

You can search documents with classical full-text search — BM25, SQLite FTS, Postgres `tsvector`, Elasticsearch. For some queries that is the right answer. It returns documents that contain the user's words.

The problem is the user does not always type the words that appear in the doc. They ask "do you ship to Hawaii?" and the policy uses "non-contiguous US states." They ask "can I return open headphones?" and the policy says "opened electronics, see clause 3." Keyword search misses both. Semantic search — built on embeddings — handles both because "Hawaii" and "non-contiguous US states" live near each other in vector space.

Production systems often combine both into a *hybrid* search (BM25 + vectors). For this module we focus on the embeddings half. You can layer keyword search on top later; the protocol does not change.

## How this connects to the rest of the track

- **`mod-103` (tools).** Retrieval is just a tool the model calls. You can wrap your retriever in a `search_kb` tool and let the model decide when to use it (next module). For this one, we run retrieval *every* turn — the simpler shape — so you can focus on embeddings, indexing, and evaluation without an agent loop on top.
- **`mod-105` (first agent).** Once retrieval is a tool, an agent can decide to retrieve, reflect on what it got, and retrieve again with a sharper query. The retriever you build here will plug straight in.
- **`mod-106` (deployment).** The offline ingest pipeline becomes a job you schedule. The online retrieval becomes a service you deploy. The eval set you write here becomes the regression gate.

## The mental model: two pipelines, one shared index

Memorize this picture:

| Pipeline    | When it runs           | Inputs              | Output                                |
|-------------|------------------------|---------------------|---------------------------------------|
| Ingest      | offline, on doc change | raw documents       | rows of `(vector, chunk, metadata)`   |
| Query       | online, per request    | user question       | top-K chunks → prompt → answer        |
| Shared      | both                   | the same embedding model and the same vector index |

Get the ingest pipeline wrong and the query pipeline has no way to recover — bad chunks, missing metadata, the wrong embedding model. Get the query pipeline wrong and a perfect index still returns confused answers. Both have to be solid, and they have to agree on the embedding model and the chunking convention.

## What this module covers

- **Chapter 2 — Embeddings and vector space.** What an embedding is, similarity metrics (cosine, dot, L2), which provider models exist, and how to call them.
- **Chapter 3 — Similarity search and indexes.** Brute-force NumPy for small corpora; FAISS, HNSW, and pgvector for everything else; top-K and filtering.
- **Chapter 4 — Chunking documents.** Token vs. character splits, structural chunking, overlap, and the metadata every chunk needs.
- **Chapter 5 — The indexing pipeline.** Load → chunk → embed → store, end-to-end, with batching, idempotency, and embedding-model versioning.
- **Chapter 6 — The RAG query loop.** Embed the question, retrieve top-K, build the prompt, return a cited answer.
- **Chapter 7 — Evaluating retrieval quality.** Recall@K, MRR, and groundedness on a small labeled set you can actually maintain.

By the end you will have an offline ingest pipeline, an online RAG loop, and an eval harness that turns "I tweaked the chunker" into a number.

## Summary

RAG retrieves the right chunks of your corpus at query time and stuffs them into the prompt so the model can answer from your data instead of from its training set. It is two small pipelines — offline ingest and online query — sharing one vector index and one embedding model. The rest of this module is the mechanics: how to make embeddings, how to search them quickly, how to chop documents into chunks worth retrieving, how to assemble the prompt, and how to tell whether any of it is working.
