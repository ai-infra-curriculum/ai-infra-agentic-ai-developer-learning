# exercise-01: Embeddings and Similarity

**Estimated effort:** 2 hours

## Objective

Build the smallest possible end-to-end "semantic search": embed a handful of documents, embed a query, rank by cosine similarity, and print the top-K. By the end you should be able to point at each line and explain what the vector represents, why you normalized it, and which similarity function you used.

This exercise establishes the primitive that exercises 2 and 3 will scale up. Keep it tiny on purpose.

## Background

This exercise covers material from:

- [Chapter 1 — From Tools to Knowledge](../01-from-tools-to-knowledge.md)
- [Chapter 2 — Embeddings and Vector Space](../02-embeddings-and-vector-space.md)
- [Chapter 3 — Similarity Search and Indexes](../03-similarity-search-and-indexes.md)

## Prerequisites

- Completed `mod-103` exercise-01 (you have a working call to one provider).
- An API key for one provider (OpenAI is the path of least resistance for embeddings; Voyage, Cohere, or a local Sentence Transformers model are all fine).
- Python 3.10+ with the provider SDK installed and either `numpy` or your math library of choice.
- A small spend cap configured. This exercise should cost cents.

## The task

You are going to build a 30–60 line script that:

1. Embeds a small corpus of ~10 short documents.
2. Embeds a user query.
3. Computes cosine similarity between the query and every document.
4. Prints the top-3 hits with their scores.

You will then run a handful of queries that probe the difference between semantic and keyword matching, and write up what you saw.

Pin your embedding model name and the dimension as module-level constants. You will reuse them in the next exercises.

### The mini corpus

Use the seed below or write your own. The key property is that several documents talk about overlapping ideas without sharing words.

```python
DOCS = [
    "We accept returns of unused items within 30 days of delivery.",
    "All electronics must be returned in their original packaging.",
    "Sale items marked 'final' cannot be returned or exchanged.",
    "Standard shipping to the contiguous US takes 2–4 business days.",
    "Orders to Hawaii and Alaska usually arrive in 5–7 business days.",
    "International orders are sent via UPS Worldwide; tracking is included.",
    "Gift wrap is available at checkout for an additional $4.99 per item.",
    "Our customer service team is available Monday–Friday, 9am–6pm Pacific.",
    "We do not currently offer phone support; please use the contact form.",
    "Refunds appear on the original payment method within 5–10 business days.",
]
```

(Use these or your own; just keep at least one cluster of "returns," one of "shipping," and one of "contact" so semantic neighbors are obvious.)

## Tasks

### 1. Embed the corpus

Write `embed_texts(texts: list[str]) -> np.ndarray` that:

- Calls the embedding API once with the full list (one batch).
- Returns an `(N, dim)` `float32` array.
- Normalizes the rows to unit length so dot product equals cosine.

Pin the model in a constant: `EMBED_MODEL = "text-embedding-3-small"` (or your chosen alternative). Print the model name and the resulting shape at the top of your script.

### 2. Search

Write `search(query: str, doc_vecs, docs, k=3) -> list[tuple[float, str]]` that:

- Embeds the query with the same model.
- Computes scores via `doc_vecs @ q_vec`.
- Returns the top-K documents with scores, ordered descending.

You may use `np.argpartition` for top-K or plain `argsort`. Both are fine at this scale; reach for `argpartition` once N is large.

### 3. Run five queries

At minimum, run:

- **Direct hit**: "how long do I have to return something?"
- **Paraphrase**: "do you ship to non-mainland states?"
- **Out-of-vocab**: "can I call you?"
- **Mostly-unrelated**: "what is your favorite color?"
- **Specific phrase**: "refund timeline"

Print the top-3 chunks and scores per query. Capture the output into `results.txt` (or paste into `NOTES.md`).

### 4. Observe and write up

In `NOTES.md`, write 3–6 sentences per question:

- **Semantic vs. keyword.** For the paraphrase query ("non-mainland states"), did the Hawaii/Alaska doc come back? Did any document containing the literal word "ship" rank higher than the Hawaii/Alaska doc? Note the score gap.
- **Wrong-but-plausible.** For the "phone" query, the corpus has both "we do not currently offer phone support" and "customer service is available Monday–Friday". Which ranked higher? Is the user better served by either?
- **Off-topic.** For the unrelated query, what was the top score? Compare it to the top score for a real query. Is there a score gap you could threshold on?

### 5. (Required) Probe the metric

Add one experiment: compute cosine using a manual `dot(a,b) / (norm(a)*norm(b))` *without* pre-normalizing, and confirm you get the same ranking as the normalized-dot-product version for at least three of your queries. Print the two top-3 lists side by side. This catches the most common foot-gun in chapter 2.

## Starter guidance

A scaffold that fits the shape above (OpenAI flavor):

```python
import numpy as np
from openai import OpenAI

client = OpenAI()
EMBED_MODEL = "text-embedding-3-small"

DOCS = [...]  # see corpus above

def embed_texts(texts: list[str]) -> np.ndarray:
    resp = client.embeddings.create(model=EMBED_MODEL, input=texts)
    vecs = np.array([d.embedding for d in resp.data], dtype=np.float32)
    norms = np.linalg.norm(vecs, axis=1, keepdims=True)
    return vecs / np.clip(norms, 1e-12, None)

def search(query: str, doc_vecs: np.ndarray, docs: list[str], k: int = 3):
    q = embed_texts([query])[0]
    scores = doc_vecs @ q
    top = np.argsort(-scores)[:k]
    return [(float(scores[i]), docs[i]) for i in top]

if __name__ == "__main__":
    doc_vecs = embed_texts(DOCS)
    print(f"Loaded {len(DOCS)} docs into shape {doc_vecs.shape}")
    queries = [
        "how long do I have to return something?",
        "do you ship to non-mainland states?",
        "can I call you?",
        "what is your favorite color?",
        "refund timeline",
    ]
    for q in queries:
        print(f"\n> {q}")
        for score, doc in search(q, doc_vecs, DOCS, k=3):
            print(f"  {score:.3f}  {doc}")
```

Do **not** add a vector index library, a chunker, or any prompt-engineering yet. Those come in the next exercises.

## Things you should not do in this exercise

- Add FAISS, hnswlib, or pgvector. The whole point is to see the primitive.
- Chunk documents (they are short enough to embed whole).
- Send anything to an answer model. There is no `messages.create` call in this exercise.
- Fine-tune the corpus to make hits look better. Note what surprised you instead.
- Use cosine via `sklearn.metrics.pairwise.cosine_similarity` without also verifying you get the same ranking from your own dot-product code. The point is to understand the metric, not to hide it behind an import.

## Acceptance criteria

You can demonstrate:

- A working script that prints top-3 results with scores for at least five queries.
- The `EMBED_MODEL` and `dim` are pinned and printed at the top.
- Vectors are normalized; dot-product and explicit-cosine rankings match for at least three queries.
- A `NOTES.md` that includes:
  - Your `EMBED_MODEL` and dimension.
  - One paragraph each on at least three of the five questions in step 4.
  - The top score for the "off-topic" query and the top score for a real query, side by side.

## Reflection

Answer briefly in `NOTES.md`:

1. If you swapped to `text-embedding-3-large` (or another model in the same family), what would you have to re-do? What would *not* need to change?
2. Pick the worst hit you saw on a real query. What is the chunk that *should* have ranked higher, and why might the model have scored your top hit higher than that one?
3. The semantic-search idea here is identical to what a vector database does at scale. Name two things this script does *not* do that production systems must.

## Stretch goals

- Re-run the five queries with `text-embedding-3-small` *and* one other model (large, Voyage, Cohere, or a local Sentence Transformers model). Note where rankings differ.
- Implement the same script with `faiss.IndexFlatIP` and confirm identical top-K against the NumPy version for every query.
- Add a deliberate "hard negative" doc to the corpus — a sentence that shares vocabulary with a query but does not answer it (e.g., "We ship over a million orders per year.") — and re-run. Did it bump the right answer off the top?
