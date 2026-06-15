# exercise-03: Chunking, Indexing, and Evaluation

**Estimated effort:** 3 hours

## Objective

Take the baseline pipeline from exercise-02, build a real evaluation harness, then improve the system by sweeping chunking and retrieval knobs against a small labeled set. By the end you should be able to defend each setting with a number.

This exercise is where retrieval engineering becomes empirical. You will not change the loop shape; you will change the chunker and the K, re-ingest, and re-measure.

## Background

This exercise covers material from:

- [Chapter 4 — Chunking Documents](../04-chunking-documents.md)
- [Chapter 5 — The Indexing Pipeline](../05-indexing-pipeline.md)
- [Chapter 7 — Evaluating Retrieval Quality](../07-evaluating-retrieval-quality.md)

## Prerequisites

- Completed exercise-02 (you have an ingest function, a search function, and a query loop that returns cited answers).
- The same provider keys, Python environment, and `corpus/` directory.
- A small spend cap configured.

## The task

You will:

1. Hand-label a small evaluation set against the exercise-02 corpus.
2. Build a recall@K + MRR + answer-substring eval harness.
3. Generalize the chunker so you can swap strategies through configuration.
4. Run a sweep across at least three chunkers and three values of K, capturing all metrics per configuration.
5. Pick the best configuration on a metric you defend, and lock it in as the new baseline.

You may also extend the corpus slightly (a couple of additional markdown files) if it helps you cover more interesting question types — but treat the corpus as fixed once you start labeling.

## Tasks

### 1. Build the eval set

Create `eval/dataset.jsonl` with at least 30 rows. Each row:

```jsonl
{"q": "...", "answer_doc": "<document_id>", "answer_chunk_ids": ["..."], "expected_answer_snippet": "...", "expected_refusal": false}
```

Coverage requirements:

- At least 5 questions whose answer requires combining information from two documents.
- At least 5 paraphrase questions (the right doc uses different vocabulary than the question).
- At least 5 `expected_refusal: true` questions whose answer is *not* in the corpus.
- At least 5 "direct hit" questions whose answer is in a single chunk.

To populate `answer_chunk_ids`, you need an existing ingested index. Ingest with the baseline chunker first, browse the chunks (a 20-line "list chunks" script suffices), then record the right `chunk_id`(s) per question. This is tedious and important; the eval is only as good as its labels.

Sanity-check yourself: open a few rows and confirm that the chunks you recorded actually contain the snippet you wrote down. Mislabeled rows poison every comparison downstream.

> The `chunk_id` for a given chunk is stable across re-ingests **only if** you tie it to content, not to position. Use the stable-id pattern from chapter 5: `f"{document_id}#{ordinal:04d}#{sha1(text)[:10]}"`. If you only used ordinal in exercise-02, switch to the content-hash form before labeling — otherwise any chunker change invalidates your labels.

### 2. Generalize the chunker

Refactor your ingest so the chunker is a swappable strategy:

```python
def ingest(corpus_dir, db_path, chunker, chunker_name: str): ...
```

Implement three chunkers:

- `chunk_by_heading(text, doc_id)` — your exercise-02 chunker (sections by markdown heading).
- `chunk_recursive(text, doc_id, max_chars=800, overlap=100)` — the recursive character splitter from chapter 4.
- `chunk_fixed_tokens(text, doc_id, max_tokens=400, overlap=40)` — token-aware splitting using `tiktoken` (or your provider's tokenizer).

Each chunker yields `{"title": str | None, "text": str}` records. The ingest function takes care of `chunk_id`, embedding, and storage.

Tag every chunk row with the `chunker_name` it came from so you can keep multiple variants in the same database and filter at query time. (Use a `chunker` column.)

### 3. Build the eval harness

Write `evaluate(conn, dataset, *, chunker_name, k) -> dict` that, for each row in the dataset:

- Searches the index with `WHERE embed_model = ? AND chunker = ?`, returning top-K.
- Runs the answer loop.
- Computes per-row metrics:
  - `recall_at_k(retrieved_ids, correct_ids, k)`
  - `reciprocal_rank(retrieved_ids, correct_ids)`
  - `answer_contains(answer, expected_snippet)` (case-insensitive substring)
  - `citations_valid(answer, set(retrieved_ids))`
  - `refused(answer)` (substring against a short list of refusal phrases)
- Aggregates:
  - mean recall@K
  - mean MRR
  - `answer_ok_rate` (excluding refusal rows)
  - `citations_ok_rate` (excluding refusal rows)
  - `over_answer_rate` (refusal rows where the model answered confidently)
  - `over_refuse_rate` (non-refusal rows where the model refused)

The harness should accept any `(chunker_name, k)` pair and return the same shape. Implementations are in chapter 7; copying them is fine.

### 4. Run a sweep

Run `evaluate` over the cross-product of:

- `chunker_name ∈ {"heading", "recursive_800_100", "tokens_400_40"}`
- `k ∈ {3, 5, 10}`

That is 9 runs. For each run, write a JSON record to `eval_runs/sweep.jsonl`:

```jsonl
{"chunker": "heading", "k": 5, "n": 32, "recall_at_k_mean": 0.74, "mrr_mean": 0.58, "answer_ok_rate": 0.81, "citations_ok_rate": 0.91, "over_answer_rate": 0.0, "over_refuse_rate": 0.06, "embed_model": "text-embedding-3-small", "answer_model": "claude-sonnet-4-6"}
```

You may need to re-ingest once per chunker (three re-ingests total). Keep all variants in the same SQLite file — they have different `chunker` values, so they coexist without collision.

Render a small table in `NOTES.md`:

| chunker | K | recall@K | MRR | answer_ok | citations_ok | over_refuse | over_answer |
|---|---|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... | ... | ... |

### 5. Pick a winner and defend it

In `NOTES.md`:

- Pick the row you would ship. State the metric you optimized for and why (e.g., "recall@K because most of my failures are not-in-context").
- Identify the single biggest tradeoff your choice makes. (Higher recall@K usually means higher K, which means a larger prompt and slower wall-clock.)
- Identify one row that was *close* to your winner and explain the call.
- Identify one question your winner still gets wrong. State whether the failure is a retrieval problem (right chunk not in top-K) or an answer problem (right chunk in top-K, wrong answer anyway).

### 6. Lock the baseline

Pin the winning configuration in your code (`CHUNKER = "..."`, `K = ...`). Make the eval harness re-runnable as `python eval.py` and commit its output for the winning config so a future run can diff against it. This is the regression-gate seed from chapter 7.

## Starter guidance

You already have most of the pieces. The new code:

- The three chunkers (~30 lines each).
- The metrics functions from chapter 7 (~20 lines total).
- A `sweep.py` that loops the cross product.

Keep `eval/dataset.jsonl` in source control. Keep `eval_runs/` git-ignored except for the locked baseline (`eval_runs/baseline.json`).

A skeleton for the chunker registry:

```python
CHUNKERS = {
    "heading":             chunk_by_heading,
    "recursive_800_100":   lambda text, doc_id: chunk_recursive(text, doc_id, max_chars=800, overlap=100),
    "tokens_400_40":       lambda text, doc_id: chunk_fixed_tokens(text, doc_id, max_tokens=400, overlap=40),
}
```

The sweep loop:

```python
for chunker_name in CHUNKERS:
    ingest(CORPUS, DB, CHUNKERS[chunker_name], chunker_name)  # idempotent for that chunker_name
    for k in (3, 5, 10):
        result = evaluate(conn, dataset, chunker_name=chunker_name, k=k)
        with open("eval_runs/sweep.jsonl", "a") as f:
            f.write(json.dumps({"chunker": chunker_name, "k": k, **result["summary"]}) + "\n")
```

## Things you should not do in this exercise

- Add an ANN library (FAISS, HNSW). Brute-force is fine at the corpus size you have and removes recall as a confound while you sweep chunkers.
- Add hybrid (keyword + vector) search. Stretch only.
- Change the answer model or embedding model mid-sweep. Pin them.
- Adjust the eval set after seeing the metrics — that is circular.
- Optimize for a metric you cannot defend ("recall@K went up by 0.02 so I shipped it" even though it tanked over_answer_rate).
- Switch to an LLM-as-judge before the cheap metrics stop discriminating.

## Acceptance criteria

You can demonstrate:

- `eval/dataset.jsonl` with at least 30 rows covering direct-hit, paraphrase, multi-doc, and refusal classes.
- Three chunker implementations and a switchable ingest.
- A sweep that ran 9 configurations and wrote `eval_runs/sweep.jsonl`.
- A summary table in `NOTES.md` and a defensible pick.
- The locked-in baseline configuration in `eval_runs/baseline.json` with all summary metrics.
- One paragraph each on (a) the tradeoff your pick makes, (b) one question the winner still gets wrong, and (c) what you would label next to grow the eval set.

## Reflection

Answer briefly in `NOTES.md`:

1. If `recall@K` is high but `answer_ok_rate` is low, where is the bug — chunking, retrieval, the prompt, or the model? How would you diagnose it?
2. The eval set has 30+ questions and you are confident the system is "better" than the exercise-02 baseline. Name two ways this confidence could still be wrong. (Hint: the eval set itself is the system being measured against.)
3. Suppose a teammate proposes raising K from your winner's value to 10 on the grounds that "more context never hurts." Use one of your sweep rows to push back.

## Stretch goals

- Add a `chunker_size` chunker that takes `max_chars` and `overlap` as parameters and sweep `{300, 600, 1200} × {0, 100, 200}` (9 more rows). Plot the heatmap and identify the local optimum.
- Add an LLM-as-judge metric (chapter 7) — a separate model call that grades each answer against the expected snippet on a 0–2 scale. Compare it to your substring-match metric on the same rows; note disagreements.
- Add query rewriting: before searching, call the model to expand the user's question into 2–3 reformulations. Embed each, retrieve, union top-K, dedupe. Re-run the eval and compare.
- Wire the eval into a `make test-retrieval` target. Have it fail if recall@K or answer_ok_rate drops by more than 0.03 vs. `eval_runs/baseline.json`. This is the regression gate from chapter 7.
