# The RAG Query Loop

## Why this matters

With chunks ingested and an index built (chapter 5), you have the offline half of RAG. This chapter is the online half: the per-request loop that takes a user's question, fetches the relevant chunks, and asks the model to answer using them. The loop is short — five steps and roughly forty lines of code — but every step has a design decision that affects answer quality.

Most "the model is hallucinating" complaints in a RAG system trace to this loop, not to the model. Fix the loop first.

## The loop, end to end

```
user question
   |
   v
[ 1. embed the question (same model as the index) ]
   |
   v
[ 2. search the index for top-K chunks ]
   |
   v
[ 3. assemble a prompt: system + retrieved context + question ]
   |
   v
[ 4. call the model ]
   |
   v
[ 5. return the answer (with citations) ]
```

A minimal implementation, using the brute-force index from chapter 3 and the SQLite sidecar from chapter 5:

```python
import json
import numpy as np
import sqlite3
from anthropic import Anthropic
from openai import OpenAI

embed_client  = OpenAI()
answer_client = Anthropic()

EMBED_MODEL  = "text-embedding-3-small"
ANSWER_MODEL = "claude-sonnet-4-6"
K            = 5

SYSTEM = """You are a support assistant. Answer the user's question using ONLY the
information in the provided context chunks. If the answer is not present in the
context, say "I do not see that in the documents." Cite chunks you used by their
[chunk_id]."""


def embed_query(text: str) -> np.ndarray:
    resp = embed_client.embeddings.create(model=EMBED_MODEL, input=[text])
    v = np.array(resp.data[0].embedding, dtype=np.float32)
    return v / np.linalg.norm(v)


def search(conn: sqlite3.Connection, query_vec: np.ndarray, k: int) -> list[dict]:
    rows = conn.execute("""
        SELECT chunk_id, text, section_title, source_path, vector
        FROM chunks
        WHERE embed_model = ?
    """, (EMBED_MODEL,)).fetchall()
    vecs = np.stack([np.frombuffer(r[4], dtype=np.float32) for r in rows])
    scores = vecs @ query_vec
    top = np.argpartition(-scores, kth=min(k, len(scores) - 1))[:k]
    top = top[np.argsort(-scores[top])]
    return [
        {
            "chunk_id":      rows[i][0],
            "text":          rows[i][1],
            "section_title": rows[i][2],
            "source_path":   rows[i][3],
            "score":         float(scores[i]),
        }
        for i in top
    ]


def build_user_message(question: str, chunks: list[dict]) -> str:
    context_blocks = []
    for c in chunks:
        title = c["section_title"] or c["source_path"]
        context_blocks.append(f"[chunk_id: {c['chunk_id']}] {title}\n{c['text']}")
    context = "\n\n---\n\n".join(context_blocks)
    return f"Context:\n{context}\n\nQuestion: {question}"


def answer(conn: sqlite3.Connection, question: str) -> dict:
    q_vec  = embed_query(question)
    chunks = search(conn, q_vec, K)
    resp = answer_client.messages.create(
        model=ANSWER_MODEL,
        max_tokens=1024,
        system=SYSTEM,
        messages=[{"role": "user", "content": build_user_message(question, chunks)}],
    )
    return {
        "question":   question,
        "chunks":     chunks,
        "answer":     "".join(b.text for b in resp.content if b.type == "text"),
        "stop_reason": resp.stop_reason,
    }
```

That is the whole loop. Exercise-02 expects you to ship something at roughly this size. The rest of this chapter is the design choices each line bakes in.

## Embedding the query

Two rules, both from chapter 2:

- **Same model as the index.** If your chunks were embedded with `text-embedding-3-small`, embed the query with `text-embedding-3-small`. Read the model name from a config; do not type it twice.
- **Same normalization.** If you normalized at ingest, normalize the query. Otherwise dot-product scores are scaled wrong and your top-K is subtly off.

If your provider supports `input_type="search_query"` (Cohere, Voyage), set it here. Mark chunks at ingest with `input_type="search_document"`. Free retrieval quality bump.

One query embedding per user question is cheap, but it is not free. Cache by question hash if you see exact repeats — common for "starter prompts" and demo decks.

## Search

You already have the index. The only decisions here are *K* and *filters*.

- **K.** Start at 5 for prose corpora, 3 for FAQ-style content. Sweep with the eval set in chapter 7; do not pick by intuition.
- **Filters.** Always filter on `embed_model` (anti-foot-gun). Optionally filter on tenant, language, date, doc type — whatever your application needs to scope retrieval.

If recall is bad, two cheap things to try before changing the index:

- **Query rewriting.** Use the model to expand a short question into a richer query (or several variants) before embedding. Embed each variant, retrieve, merge top-K. Costs one extra model call per request and recovers a lot of "user typed three words" cases.
- **Hybrid search.** Run a keyword search (BM25, Postgres `tsvector`, SQLite FTS) in parallel with the vector search and merge results by score. Especially helpful when the corpus has exact-match anchors (product codes, error messages, function names) that semantic search dilutes. <!-- needs-research: confirm current reciprocal-rank-fusion is the standard merge; cite a primary source -->

Hybrid search is conventionally what production systems graduate to once pure vector search is not enough. It is out of scope for this module's required content; the exercise stretch goals point at it.

## Assembling the prompt

The system message has one job: tell the model to answer from the context and to refuse otherwise. The user message has one job: present the context and the question in a shape the model can use.

A prompt that works for almost any RAG setup:

```
[system]
You are a support assistant. Answer the user's question using ONLY the
information in the provided context chunks. If the answer is not present
in the context, say "I do not see that in the documents." Cite the chunks
you used by their [chunk_id].

[user]
Context:
[chunk_id: handbook-returns#0004] Returning opened electronics
We accept returns of opened electronics within 14 days...

---

[chunk_id: handbook-shipping#0002] Shipping to Hawaii and Alaska
Orders to Hawaii and Alaska...

Question: Can I return open headphones?
```

Five design choices in that template:

- **Context comes before the question.** Models perform better when the question is the most recent thing in the user turn. <!-- needs-research: cite Anthropic / OpenAI prompt-engineering guides recommending question-last for retrieval -->
- **Chunks are clearly delimited.** A horizontal rule or XML tags (`<chunk id="…">…</chunk>`) work; the model needs to see where one chunk ends and another begins.
- **Each chunk carries an id and a title.** The id lets the model cite. The title gives a human-readable handle. The score does not need to be in the prompt — it is not for the model.
- **The "use only the context" instruction is explicit.** Without it the model will happily fill in from training data and you have lost the entire point of retrieval.
- **The refusal clause is explicit.** "If the answer is not present, say so." This is the difference between "I do not see that in the documents" and a confident hallucination.

Anthropic recommends wrapping retrieved chunks in XML tags for clarity; OpenAI examples often use markdown horizontal rules. Either works. Be consistent within a project.

## Citations

A RAG answer without citations is a vibes-based answer. Two ways to wire them:

- **Inline.** Ask the model to write `[chunk_id]` after each claim. Simple, and you can verify by checking each chunk_id appears in the retrieved set.
- **Structured.** Have the model emit JSON with `{answer, used_chunks: [chunk_id, …]}`. Cleaner for downstream rendering; pairs with `mod-102` chapter 5's structured-output strategies.

You can render structured citations into a "Sources" panel in your UI, with links built from the chunks' `source_url` metadata. The user sees both the answer and exactly which paragraphs of which docs it came from. This is the user-facing payoff of all the metadata work in chapter 4.

A small validator catches the failure where the model invents a chunk_id: keep a set of `retrieved_ids = {c["chunk_id"] for c in chunks}`; if the model cites an id not in that set, surface it as a problem. Chapter 7 evaluates this directly.

## Handling "I don't know"

The single highest-leverage upgrade you can make over a naive RAG loop is to let the model say "I don't know" cleanly. Two parts to it:

- **Explicit refusal instruction in the system prompt.** Already shown above.
- **Score floor.** If the top-K scores are all below some threshold — say, top hit has cosine < 0.4 — short-circuit and return "I do not see that in the documents" without calling the model. You save the model call and you avoid tempting the model with irrelevant context.

The right threshold is corpus-dependent. Calibrate on the eval set: look at the score distribution of "answerable" vs. "unanswerable" queries and pick a threshold where the two distributions separate cleanly.

## Streaming and UX

For interactive use, stream the answer back to the user — the wall-clock cost of "wait for the full reply, then render" is much higher than the cost of token-by-token rendering. Both Anthropic's and OpenAI's SDKs support streaming with a one-line change to the call.

Render the *sources panel* before the answer starts streaming. Retrieval results are available the moment search returns; the user appreciates seeing "here is what I looked at" while the model is composing. This also gives them a way to spot the wrong-doc case before they read a wrong answer.

## Latency budget

A reasonable RAG request is on the order of:

| Step                         | Typical latency |
|------------------------------|-----------------|
| Embed query (1 call)         | 50–200 ms       |
| Index search (in-memory)     | 1–20 ms         |
| Index search (pgvector)      | 5–100 ms        |
| Model call (answer)          | 0.5–5 s         |

Total: 1–5 seconds, dominated by the model call. The places to push:

- Use a smaller / faster model when quality allows (Haiku tier for simple Q&A).
- Cache query embeddings for repeats.
- If you have prompt-caching available (Anthropic, OpenAI), cache the system prompt — it is identical across requests.

Latency rarely fails a RAG system the way *recall* does. Get the answer right first, optimize latency after.

## A note on the agent variant

You will see RAG implemented as "the model decides when to call a `search` tool." That is the agent shape from `mod-105`. It has real advantages — the model can search again with a sharper query, or skip search when the question doesn't need it. It also has more failure modes: tool-call loops, wrong queries, no retrieval when retrieval was needed.

For this module we use the *non-agent* shape: retrieve on every turn, always. It is simpler to reason about, cheaper, faster, and the right baseline. The exercises ship the non-agent loop; `mod-105` retrofits the agent version on top.

## Common mistakes

- **Mismatched embed model between ingest and query.** Silent. Always wrong. Tag and check.
- **No "answer only from context" instruction.** The model uses training data; you have built a slower, more expensive chatbot.
- **No refusal instruction.** Hallucinations show up exactly when retrieval whiffed.
- **Citations as text the model is asked to invent.** It will sometimes invent ids that look right. Validate them against the retrieved set.
- **K too large.** The model spends tokens reading irrelevant context; signal-to-noise drops.
- **K too small.** The right chunk did not make it into top-K. Recall@K from chapter 7 catches this.
- **Re-running the embedding for the corpus inside the query path.** A `not-found-in-cache` bug; you will notice when your bill spikes and your latency triples. The query path embeds *the query* only.

## Summary

The RAG query loop is five steps: embed the question, search the index for top-K chunks, assemble a prompt with explicit "answer from context only" and "cite chunk_ids" instructions, call the model, return the answer with sources. The defaults that matter are: same embedding model on both sides, normalize, K = 3–5, explicit refusal clause, structured citations checked against the retrieved set. The next chapter measures whether all of this actually works.
