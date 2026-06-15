# exercise-02: A Simple RAG Pipeline

**Estimated effort:** 3 hours

## Objective

Build the smallest end-to-end retrieval-augmented generation system: ingest a small corpus, store chunks + vectors + metadata, run the query loop end-to-end against a real provider, and return answers with citations. You should be able to point at every part of the loop and explain why it is there.

This is the working baseline you will improve in exercise-03.

## Background

This exercise covers material from:

- [Chapter 4 — Chunking Documents](../04-chunking-documents.md)
- [Chapter 5 — The Indexing Pipeline](../05-indexing-pipeline.md)
- [Chapter 6 — The RAG Query Loop](../06-the-rag-query-loop.md)

## Prerequisites

- Completed exercise-01 (you have working embeddings, normalized, and a NumPy dot-product search).
- An API key for one embedding provider and one answer-generation provider (a single Anthropic or OpenAI key covers both for that provider; mixing is also fine).
- Python 3.10+, the provider SDKs, `numpy`, and the standard-library `sqlite3`.
- A small spend cap configured. The full exercise should cost well under a dollar.

## The task

You are building the first version of a support assistant over a small "company handbook" corpus. The corpus is intentionally tiny — five or six markdown files of a few hundred words each — so the focus stays on the loop and not on parsing.

You will:

1. Ingest the corpus into a SQLite + NumPy index: chunked, embedded, with metadata.
2. Implement the RAG query loop end-to-end: embed query → search → assemble prompt → answer.
3. Have the model produce answers with chunk_id citations.
4. Validate citations against the retrieved set.
5. Run the loop against at least five test questions and capture the trace.

Pin both your embedding model and your answer model as module-level constants. Use `temperature=0` for the answer model so results are reproducible.

### The corpus

Create `corpus/` with at least these five files. Use the seed text below or adapt your own; the only requirement is that several distinct topics are present and at least one question's answer requires looking up the right file.

- `corpus/returns.md` — returns policy: time window, condition requirements, electronics special case, final-sale items.
- `corpus/shipping.md` — shipping policy: domestic, Hawaii/Alaska, international, gift options.
- `corpus/payments.md` — accepted methods, when charges appear, refund timeline.
- `corpus/support.md` — contact hours, escalation, languages supported, what kinds of issues route where.
- `corpus/products.md` — three short product descriptions with names, prices, and warranty terms.

Aim for ~300–600 words per file so each file produces 2–5 chunks.

### The exercise corpus structure

Each file should be markdown with at least 2–3 `##` headings. The chunker will use those as section boundaries.

## Tasks

### 1. Build the ingest pipeline

Write `ingest(corpus_dir, db_path)` that does the four-step `load → chunk → embed → store` pipeline from chapter 5:

- **Load.** Walk `corpus_dir/*.md`. Use `path.relative_to(corpus_dir).with_suffix("").as_posix()` as `document_id`.
- **Chunk.** Use a simple markdown heading-based splitter: each `##` (or higher) heading starts a new chunk. If a section is more than ~1500 characters, fall back to a paragraph split. (You do *not* need the full recursive splitter from chapter 4 here; exercise-03 will swap it in.)
- **Embed.** Batch the chunk texts (≤ 100 per call) and embed with your pinned model. Normalize the vectors. Store as `bytes(np.asarray(vec, dtype=np.float32).tobytes())`.
- **Store.** SQLite table with `(chunk_id, document_id, source_path, section_title, ordinal, text, embed_model, vector)`. Use `INSERT OR REPLACE` keyed on `chunk_id` so re-runs are idempotent.

Generate `chunk_id` as `f"{document_id}#{ordinal:04d}"`. (You will harden this with a content hash in exercise-03.)

Run `ingest` and print a one-line summary: number of documents, number of chunks, embed-model name, elapsed seconds. Save the summary to `manifest.json`.

### 2. Build the search function

Write `search(conn, query: str, k: int = 5) -> list[dict]` that:

- Embeds the query with the same model name as the index.
- Loads vectors for rows where `embed_model = <pinned model>`.
- Computes scores via dot product, returns top-K with `chunk_id`, `text`, `section_title`, `source_path`, and `score`.

This is the same brute-force NumPy search from exercise-01, with a SQLite read layered on top. Do **not** introduce a vector library yet.

### 3. Build the RAG query loop

Write `answer(conn, question: str) -> dict` that:

1. Calls `search(conn, question, k=K)`.
2. Builds a user message containing the chunks (with their `chunk_id` and `section_title`) and the question. Use the template from chapter 6.
3. Calls the answer model with `temperature=0` and a system prompt that instructs "answer only from the context" and "cite chunk_ids" and "say so if the answer is not in the documents."
4. Returns `{"question", "chunks", "answer", "stop_reason"}`.

Pin `K = 5` for this exercise. You will sweep K in the next one.

### 4. Validate citations

Extract chunk_ids from the answer (with a small regex, e.g. `\[chunk_id:\s*([\w\-#]+)\]`). Confirm every cited id appears in `[c["chunk_id"] for c in chunks]`. Flag any answer where the model invented an id.

### 5. Drive it with five questions

Run your `answer` function against at least:

- **Direct hit:** "How many days do I have to return an item?"
- **Paraphrase:** "Do you deliver to Hawaii?"
- **Multi-doc:** "If I return a sale item, when will I see the money?"  (touches returns + payments)
- **Specific product:** A question about one of the items in `products.md`.
- **Not in corpus:** "What is your stance on cryptocurrency payments?" — the answer should be the refusal phrase.

Capture per-run into `results.jsonl`:

```jsonl
{"q": "...", "retrieved_ids": ["..."], "cited_ids": ["..."], "citations_valid": true, "answer": "..."}
```

### 6. Look at one trace

Pick the answer you find most interesting (often the multi-doc one or the refusal). Paste into `NOTES.md`:

- The full user message you sent (system + chunks + question).
- The full model reply.
- Which chunks were retrieved, with their scores.

Then write 4–8 sentences on:
- Was the right information in top-K? If not, which chunk should have been there?
- Did the model use *all* retrieved chunks, or only the relevant ones? Did it pull in anything that was retrieved but irrelevant?
- Did the model refuse cleanly on the not-in-corpus question? Did it cite chunks even when refusing?

## Starter guidance

You have most of the pieces already. The new code is the ingest function and the prompt template. A skeleton:

```python
import json, re, sqlite3
import numpy as np
from pathlib import Path
from openai import OpenAI
from anthropic import Anthropic

embed_client  = OpenAI()
answer_client = Anthropic()
EMBED_MODEL   = "text-embedding-3-small"
ANSWER_MODEL  = "claude-sonnet-4-6"
K             = 5

SYSTEM = (
    "You are a support assistant. Answer the user's question using ONLY the "
    "information in the provided context chunks. If the answer is not present in "
    "the context, say 'I do not see that in the documents.' Cite chunks you used "
    "by [chunk_id: ...]."
)

# 1. ingest(corpus_dir, db_path) — open SQLite, walk md files, chunk by heading,
#    embed in batches of 100, INSERT OR REPLACE into chunks(...)
# 2. search(conn, query, k)      — embed query, dot-product against all rows
#                                  filtered by embed_model, return top-K rows
# 3. answer(conn, question)      — search; build prompt; messages.create;
#                                  validate citations; return dict
```

Keep the whole thing under ~250 lines. If you find yourself growing past that, you are probably adding things that belong in exercise-03.

Reuse the SDK setup and config-loading patterns from `mod-103` exercise-01. Do **not** introduce LangChain, LlamaIndex, or any RAG framework — the point is to see the loop.

## Things you should not do in this exercise

- Use a vector library (FAISS, hnswlib, pgvector). NumPy is enough at this scale.
- Use a fancy chunker. Heading-split + paragraph fallback is the assignment.
- Implement query rewriting, hybrid search, or reranking. Stretch only.
- Tune `K`, `chunk_size`, or the prompt to make the demo look better. Capture the unembellished baseline; exercise-03 is where you optimize.
- Use anything other than `temperature=0` on the answer model.

## Acceptance criteria

You can demonstrate:

- A working `ingest(corpus_dir, db_path)` that produces a SQLite file with at least 15 chunks, all tagged with the embed model.
- A working `answer(conn, question)` that returns an answer plus the retrieved chunks for at least the five baseline questions.
- A `results.jsonl` with one entry per baseline question showing retrieved chunk_ids, cited chunk_ids, whether citations were valid, and the answer text.
- A `manifest.json` summarizing the ingest run (corpus root, embed model, chunk count, elapsed seconds).
- A `NOTES.md` containing the trace from step 6 plus your pinned `EMBED_MODEL`, `ANSWER_MODEL`, `K`, and `temperature`.
- The refusal case ("cryptocurrency") returns the refusal phrase, not a fabricated policy.

## Reflection

Answer briefly in `NOTES.md`:

1. What part of the system would break if the embedding model name in `search` did not match the one used during ingest? Describe the failure mode you would expect.
2. Suppose a user asks the same question phrased five slightly different ways. The retrieved chunks shift run-to-run. Where in the pipeline does that variability live, and where could you reduce it?
3. Pick one chunk that got retrieved on a baseline query but was not relevant. Was the problem retrieval, chunking, or both?

## Stretch goals

- Replace the brute-force NumPy search with `faiss.IndexFlatIP` or pgvector. Confirm identical top-K to the NumPy baseline.
- Add a "score floor" short-circuit: if the top hit's score is below a threshold, skip the model call and return the refusal phrase directly. Try thresholds of 0.0, 0.3, and 0.5; note how the over-refuse rate changes on your baseline queries.
- Add structured-output citations (the model returns JSON with `{answer, used_chunks: [...]}`); render a sources panel from the result.
- Wire prompt caching for the system prompt (Anthropic or OpenAI) and measure the latency delta on five consecutive queries.
