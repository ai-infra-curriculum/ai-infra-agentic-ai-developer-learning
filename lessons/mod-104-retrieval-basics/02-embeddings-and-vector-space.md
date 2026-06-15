# Embeddings and Vector Space

## Why this matters

Retrieval starts with a single trick: turn text into a vector such that texts about the same thing land near each other. That vector is an *embedding*. Once you have embeddings for every chunk in your corpus and one embedding for the user's question, "find what's relevant" reduces to "find the chunk vectors nearest to the question vector." All of chapter 3 (similarity search) is built on top of this.

You do not need to understand how the embedding model was trained to use one. You do need to understand what an embedding is, which model you picked, the similarity metric you are using, and the cost and dimensionality of the vector you are about to multiply by a million chunks.

## What an embedding is, concretely

An embedding model takes a string and returns a fixed-length vector of floats. Same string in → same vector out (deterministic for a given model and version). Same model, different inputs that mean similar things → vectors close to each other.

```python
from openai import OpenAI
client = OpenAI()

resp = client.embeddings.create(
    model="text-embedding-3-small",
    input="Do you ship to Hawaii?",
)
vec = resp.data[0].embedding  # list[float], length 1536 for this model
```

That vector has no human-readable meaning per coordinate. You do not look at `vec[42]`. You only ever compare two vectors with a similarity function and ask: "are these close?"

Two facts you should keep in mind:

- **Embeddings are tied to a specific model.** A vector from `text-embedding-3-small` cannot be compared to a vector from `voyage-3`. Different geometries, different training, different scales. Reindex when you change models.
- **Embeddings are tied to a specific *version* of the model.** Providers occasionally retrain. The API will tell you the model name; bury it in your metadata so you know which vectors came from which version (chapter 5).

## Provider models and their shape

The market changes; the categories are stable. Three places you will get embeddings:

- **OpenAI** — `text-embedding-3-small` (default cost-effective choice, 1536 dims) and `text-embedding-3-large` (higher quality, 3072 dims). Both accept a `dimensions` parameter to truncate the output vector down for storage savings; the trade-off is documented in the OpenAI guide. <!-- needs-research: confirm latest OpenAI embeddings model lineup at https://platform.openai.com/docs/guides/embeddings as model names and price tables change -->
- **Voyage AI** — Anthropic's recommended embedding partner; `voyage-3`, `voyage-3-large`, and domain-tuned variants. Used when you build on Claude and want a vendor that publishes evaluation results aligned with retrieval use cases. <!-- needs-research: confirm current Voyage model names and Anthropic's "recommended embedding provider" guidance at https://docs.anthropic.com/en/docs/build-with-claude/embeddings -->
- **Cohere** — `embed-english-v3.0`, `embed-multilingual-v3.0`, etc. Notable for first-class support for `input_type` ("search_document" vs "search_query") which slightly boosts retrieval quality when you label inputs correctly. <!-- needs-research: confirm at https://docs.cohere.com/docs/embeddings -->
- **Open-source / local** — Sentence Transformers / BGE / E5 / nomic-embed-text run locally with no API call. Useful when data cannot leave your network or when cost dominates. Quality varies; check the MTEB leaderboard for current standings.

For this module you can pick any of these. The exercises assume OpenAI by default because the API is the most uniform with what you already use, but the code shape is the same for all four.

When you pick a model, write down four numbers and keep them:

| Property               | Why it matters                                                       |
|------------------------|----------------------------------------------------------------------|
| Model name + version   | Your index is locked to this. Changing it means reindexing.          |
| Vector dimension       | Sets storage size and what your index code allocates.                |
| Max input tokens       | Anything over this is silently truncated (or rejected). Chapter 4.   |
| Cost per million tokens | Multiplies by total chunks ingested + every query. Track it.        |

## Similarity metrics

Two vectors, one number that says how close they are. Three metrics show up in practice:

- **Cosine similarity** — `cos(a, b) = a·b / (|a| |b|)`. Range `[-1, 1]`; higher is closer. Insensitive to vector magnitude; only the *direction* matters. This is the default for almost every embedding model in production.
- **Dot product** — `a·b`. Higher is closer. Sensitive to magnitude. Equivalent to cosine *only when both vectors are unit-length (L2-normalized)*. Many providers (including OpenAI's `text-embedding-3-*`) return already-normalized vectors, which means dot product and cosine give identical rankings for free. <!-- needs-research: confirm OpenAI's current statement on default normalization at https://platform.openai.com/docs/guides/embeddings -->
- **Euclidean distance (L2)** — `|a - b|`. Lower is closer. Equivalent ranking to cosine *for L2-normalized vectors* (the standard case). Some libraries default to L2; do not let that surprise you.

In practice you almost always want cosine semantics. If your provider returns normalized vectors you can use dot product directly (faster, no division). If you are not sure, normalize explicitly with `numpy.linalg.norm` and store the unit vectors. The index code in chapter 3 will assume this.

```python
import numpy as np

def cosine(a, b):
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

# If you normalize once, you can skip the divides at query time:
def normalize(v):
    v = np.asarray(v, dtype=np.float32)
    return v / np.linalg.norm(v)
```

## Embedding a query vs. embedding a document

Same model, same call shape, slightly different intent:

```python
doc_vec   = client.embeddings.create(model="text-embedding-3-small",
                                     input="Returns are accepted within 30 days...").data[0].embedding
query_vec = client.embeddings.create(model="text-embedding-3-small",
                                     input="how long do I have to return things").data[0].embedding
score = np.dot(doc_vec, query_vec)  # high if the policy chunk is relevant
```

Some providers (Cohere, Voyage) let you flag the call with an `input_type` of `"search_document"` or `"search_query"` and tune the embedding asymmetrically. It is a small but real win for retrieval. <!-- needs-research: confirm at Cohere and Voyage docs which exact param name and values are current --> If your provider supports it, use it; the only cost is remembering which side you are on.

## Batching, cost, and rate limits

Embedding APIs accept a list of inputs per call. Always batch:

```python
texts = ["chunk one", "chunk two", "chunk three"]
resp = client.embeddings.create(model="text-embedding-3-small", input=texts)
vectors = [item.embedding for item in resp.data]  # same order as `texts`
```

Reasonable batch sizes are typically 100–500 chunks per call, but **read your provider's docs for the current cap on inputs per call and total tokens per call** — these numbers change. Chapter 5 wraps batching, retry, and idempotency around a real ingest job.

Cost is multiplicative: `(corpus_tokens × embedding_price) + (queries_per_day × avg_query_tokens × embedding_price) × days`. For a 50k-chunk corpus and a few thousand queries a day, the ingest cost is the dominant term and you pay it once; the query cost is small but steady. Estimate before you ingest.

## What goes wrong

A handful of failure modes appear in every project sooner or later:

- **Mixed model vectors.** You reindexed half the corpus on a new model and forgot the rest. Searches return nonsense. The fix is to tag every row with the embedding model + version (chapter 5) and refuse to query an index where they disagree.
- **Unnormalized comparisons.** You used dot product on un-normalized vectors and the ranking is wrong. Normalize at ingest, query with dot product, move on.
- **Wrong "input type."** On Cohere/Voyage you forgot to set `input_type=search_query` for queries. Retrieval works, but slightly worse than it should. Cheap to fix; easy to miss.
- **Truncation.** You embedded chunks longer than the model's max input tokens. The provider truncated silently. Your tail content is invisible. Chapter 4's chunk sizes are picked specifically to avoid this.
- **No persistence.** Re-embedding the same chunk every run because you did not save the vectors. Embedding is not free; cache. Chapter 5 again.

Most of these are fixed once and never again. The first one — mixed model vectors — bites teams that "just updated the model" without reindexing; it is silent and corrosive. Tag everything.

## A minimal end-to-end "search" without a real index

To anchor the rest of the module, here is the whole retrieval idea in 20 lines of NumPy, brute-force, no library:

```python
import numpy as np
from openai import OpenAI

client = OpenAI()
MODEL = "text-embedding-3-small"

DOCS = [
    "We accept returns within 30 days of delivery for unused items.",
    "Shipping to Hawaii and Alaska takes 5–7 business days.",
    "All electronics must be returned in original packaging.",
    "Sale items are final and cannot be returned.",
]

def embed(texts):
    resp = client.embeddings.create(model=MODEL, input=texts)
    return np.array([d.embedding for d in resp.data], dtype=np.float32)

doc_vecs = embed(DOCS)              # shape (4, 1536), already normalized
q_vec    = embed(["can I return headphones I opened?"])[0]
scores   = doc_vecs @ q_vec         # cosine == dot for normalized vectors
top      = np.argsort(-scores)[:2]
for i in top:
    print(f"{scores[i]:.3f}  {DOCS[i]}")
```

That is a retriever. It is naive, slow at scale, and good enough to ship the first prototype. Chapter 3 takes the same idea and makes it fast on a million chunks; chapter 4 and 5 take care of where the chunks came from in the first place.

## Summary

An embedding is a fixed-length vector representation of text from a specific model and version. Similar texts produce vectors that are close under cosine similarity (or dot product on normalized vectors). You choose a provider and model up front, tag every vector with which model produced it, normalize once at ingest, and compare with dot product at query time. The rest of retrieval is about *which* vectors to compare against and how to do it fast — the next chapter starts there.
