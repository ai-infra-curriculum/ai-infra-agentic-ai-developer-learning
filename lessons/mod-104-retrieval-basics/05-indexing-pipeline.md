# The Indexing Pipeline

## Why this matters

Chapters 2–4 introduced the pieces: embeddings, an index, and chunks. This chapter snaps them together into the *offline* pipeline that turns your corpus into a queryable index. The pipeline runs once when you first set up the system and again every time documents change. It is plumbing, but the plumbing has sharp edges — partial runs, model upgrades, duplicate documents, rate limits — and getting them right now saves a recurring class of bugs later.

Critically: this is not the same as the *query* pipeline (chapter 6). The query pipeline reads the index. This pipeline *builds* it.

## The pipeline shape

The offline pipeline is four steps, repeated per document:

```
load -> chunk -> embed -> store
```

```
[ load document(s) from source ]
        |
        v
[ chunk each document, producing (chunk_id, text, metadata) ]
        |
        v
[ embed the chunk texts in batches ]
        |
        v
[ store (chunk_id, text, metadata, vector) in the index + sidecar store ]
```

Each step has a single failure mode you should plan for. Loading: source moved. Chunking: format changed. Embedding: rate limits and partial batches. Storing: half-written index.

## A minimal, runnable pipeline

A working ingest that walks a directory, chunks markdown by heading, embeds with OpenAI, and stores into a SQLite + NumPy sidecar. Library-free vector store on purpose; you will swap in FAISS or pgvector in chapter 3's territory once you outgrow this.

```python
import hashlib
import json
import sqlite3
from pathlib import Path

import numpy as np
from openai import OpenAI

EMBED_MODEL = "text-embedding-3-small"
EMBED_DIM   = 1536
BATCH       = 100

client = OpenAI()


def stable_chunk_id(doc_id: str, ordinal: int, text: str) -> str:
    h = hashlib.sha1(text.encode("utf-8")).hexdigest()[:10]
    return f"{doc_id}#{ordinal:04d}#{h}"


def open_store(path: str) -> sqlite3.Connection:
    conn = sqlite3.connect(path)
    conn.execute("""
        CREATE TABLE IF NOT EXISTS chunks (
            chunk_id      TEXT PRIMARY KEY,
            document_id   TEXT NOT NULL,
            source_path   TEXT NOT NULL,
            section_title TEXT,
            ordinal       INTEGER NOT NULL,
            text          TEXT NOT NULL,
            embed_model   TEXT NOT NULL,
            embed_dim     INTEGER NOT NULL,
            vector        BLOB NOT NULL
        )
    """)
    conn.commit()
    return conn


def embed_batch(texts: list[str]) -> np.ndarray:
    resp = client.embeddings.create(model=EMBED_MODEL, input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    # OpenAI text-embedding-3-* returns L2-normalized vectors; safe to assume,
    # but normalizing again is cheap and removes one foot-gun.
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)


def ingest_directory(root: Path, store_path: str, chunker) -> None:
    conn = open_store(store_path)
    pending: list[tuple[dict, str]] = []  # (chunk_row, text_to_embed)

    for path in root.rglob("*.md"):
        doc_id = path.relative_to(root).with_suffix("").as_posix()
        text = path.read_text(encoding="utf-8")
        for ordinal, section in enumerate(chunker(text, doc_id)):
            cid = stable_chunk_id(doc_id, ordinal, section["text"])
            row = {
                "chunk_id":      cid,
                "document_id":   doc_id,
                "source_path":   str(path),
                "section_title": section.get("title"),
                "ordinal":       ordinal,
                "text":          section["text"],
                "embed_model":   EMBED_MODEL,
                "embed_dim":     EMBED_DIM,
            }
            pending.append((row, section["text"]))

            if len(pending) >= BATCH:
                flush(conn, pending)
                pending.clear()

    if pending:
        flush(conn, pending)


def flush(conn: sqlite3.Connection, pending: list[tuple[dict, str]]) -> None:
    texts = [t for _, t in pending]
    vecs = embed_batch(texts)
    rows = []
    for (row, _), vec in zip(pending, vecs):
        row["vector"] = vec.tobytes()
        rows.append(row)
    conn.executemany("""
        INSERT OR REPLACE INTO chunks
        (chunk_id, document_id, source_path, section_title, ordinal,
         text, embed_model, embed_dim, vector)
        VALUES
        (:chunk_id, :document_id, :source_path, :section_title, :ordinal,
         :text, :embed_model, :embed_dim, :vector)
    """, rows)
    conn.commit()
```

About 80 lines, end-to-end, no dependencies beyond NumPy and your embedding client. It will ingest tens of thousands of chunks before you notice. The exercises start from a shape like this.

## Stable IDs and idempotency

The pipeline must be safe to re-run on an overlapping set of documents. If you ingest the same handbook twice, you want one row per chunk, not two.

Two patterns combine to give you that:

- **Stable `chunk_id`.** Compose it from things that do not change for the same logical chunk: `document_id`, position in the document, and a hash of the text. `stable_chunk_id` above uses `doc_id + ordinal + sha1(text)`. The hash makes it explicit when the chunk *content* changed even though the position did not.
- **Upsert on the primary key.** `INSERT OR REPLACE` (SQLite), `INSERT … ON CONFLICT … DO UPDATE` (Postgres), `index.add_with_ids` (FAISS with deduplication). A re-ingest overwrites the row instead of duplicating it.

When a document is *removed* from your source, neither of those handles it. Either keep a "this run touched these chunk_ids" set and delete the complement, or attach a `last_seen` timestamp at ingest time and run a cleanup pass for rows older than the latest ingest.

## Batching, rate limits, and retries

Embedding APIs are happiest with batches. The pattern is:

- Build chunk rows greedily into a buffer.
- When the buffer reaches `BATCH` items, embed and flush.
- Drain the remainder at the end.

Two failure modes to plan for:

- **Rate limits.** Providers return 429s when you push too fast. Use exponential backoff with jitter on retry; cap retries at ~5; fail the run loudly if you blow the cap. The provider SDKs ship with retry logic — read its defaults before you wrap your own. <!-- needs-research: confirm current default retry behavior in https://github.com/openai/openai-python and https://github.com/anthropics/anthropic-sdk-python -->
- **Partial batches.** If a batch of 100 fails on item 73, you do not want to redo the first 72. Save successful batches as you go (the `flush(conn, pending)` shape above commits per batch). On rerun, the upsert handles re-encountering the same `chunk_id`s.

A small concurrency note: most providers accept several in-flight requests at once. You can run multiple embedding batches concurrently with `asyncio.gather` and a `Semaphore`, which roughly halves the wall-clock time of a large ingest. Start sequential; add concurrency once the sequential version is correct.

## Embedding model versioning

Tag every row with the model that produced it. Pulled forward from chapter 2: `embed_model = "text-embedding-3-small"`, plus a `embed_version` string if your provider publishes one. Then make two rules permanent:

1. **A query embedding and the chunk embeddings it is compared against must come from the same model.** Enforce this at query time — read the model name from a config and pass it to both the embedding call and the index filter. Cross-model comparisons return junk silently.
2. **Reindexing on a model upgrade is a deliberate operation, not an accident.** When you point at a new model, you mark the corpus as needing a refresh, run the ingest from scratch (or in parallel into a new index), and cut over only when the new index passes the eval from chapter 7.

If you store both old and new vectors in the same table, the `WHERE embed_model = ?` predicate from chapter 3 keeps them apart at query time.

## Persistence: what to save and where

You need three things at query time:

- The **vector** (to compare against the query vector).
- The **chunk text** (to put into the prompt).
- The **metadata** (for citations and filters).

How you store them depends on the index:

| Index choice          | Vector lives in        | Chunk text + metadata live in       |
|-----------------------|------------------------|-------------------------------------|
| Brute-force NumPy     | `.npy` / `.npz` file   | SQLite or JSONL sidecar             |
| FAISS                 | `index.faiss` (binary) | SQLite or JSONL sidecar             |
| pgvector              | `vector` column        | Same Postgres row                   |
| Hosted vector DB      | The service            | Service metadata + (optionally) sidecar mirror |

The "sidecar" pattern (NumPy / FAISS for vectors + SQLite for everything else) is the default for self-hosted, small-to-medium systems. The two must stay in sync: never write the vector file without also writing the metadata row, and vice versa. The cleanest way is a single transaction-style flush — write the SQLite rows and append to the vector file in the same function, with rollback on error.

`pgvector` collapses both into one row and one transaction, which is one reason teams reach for it.

## Incremental ingest

In the real world, documents change one at a time. Re-ingesting everything every time wastes money and time. Three useful patterns:

- **Per-document refresh.** Take a `document_id` and a new file. Delete all existing chunks with that `document_id`. Re-chunk, re-embed, re-store. Cheap and obviously correct.
- **Per-chunk content hash.** Compute `hash(text)` for each chunk during chunking. If a chunk with the same `(document_id, ordinal)` already exists and has the same hash, skip embedding it. Saves API calls when documents grow at the end (very common in handbooks).
- **Source-time-based incremental.** Walk only files with `mtime > last_ingest_time` and run per-document refresh on them. Pair with a cleanup pass for deleted files.

The first is the right default. The second is a nice optimization. The third is what your scheduled job runs at 3am.

## Observability

You will have to debug this pipeline. At minimum, log per run:

- Total documents loaded.
- Total chunks produced (and total characters / tokens).
- Embedding API calls, tokens spent, time spent.
- Batches succeeded vs. retried vs. failed.
- Final row count in the index, with a quick assertion against expectations.

`tqdm` for progress is fine for interactive runs; in production the same numbers should land in structured logs. Track *embedding tokens* specifically — they are the single biggest variable cost of a RAG system.

## A worked output

What you should have at the end of a successful ingest:

- One database (or vector file + sidecar) with N rows.
- Every row tagged with `embed_model`, `embed_version`, and a `created_at`.
- A small `manifest.json` (or row in a `runs` table) capturing what ran: the corpus root, the chunker name + version, the embedding model + version, the row count, the elapsed time. This is the artifact you re-create the run from.
- Sanity-check queries you can paste into a notebook: pick a known chunk, retrieve top-5 for a known question, eyeball that the right chunk is on top.

If you cannot answer "which run of which chunker produced this row," you are flying blind during evaluation. Bake the manifest in early.

## Common mistakes

- **No idempotency.** Re-running the pipeline duplicates rows. Embeddings double. Search returns the same chunk three times.
- **One giant embedding call.** Either it fails because the input is too big, or it goes through and you have nothing to resume from when it breaks halfway.
- **Saving vectors but not metadata (or vice versa).** Half-broken state, debugged with grep.
- **Forgetting to normalize.** Chapter 2's foot-gun comes back at query time as ranking weirdness.
- **No model name tag.** You upgrade the embedding model "just to see," and a week later you cannot tell which rows are stale.
- **Embedding the entire document instead of chunks.** You lost chapter 4. Rerun the chunker.

## Summary

The indexing pipeline is offline plumbing: load → chunk → embed → store, idempotent, batched, retry-aware, and tagged with the embedding model so the query side can refuse to mix vintages. Start with a single function and a SQLite sidecar; swap the vector store for FAISS or pgvector when scale demands it. Bake in versioning and a manifest from the first commit — those two artifacts are what make every later change safe.
