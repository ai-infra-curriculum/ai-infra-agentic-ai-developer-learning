# Evaluating Retrieval Quality

## Why this matters

You have a retriever, an index, and a query loop. The temptation now is to type a few questions in, decide the answers look "pretty good," and ship. Six weeks later someone tweaks the chunk size, the answers get worse on the questions you did not test, and you have no way to prove it changed.

Evaluation is the difference between "I improved the prompt" and "Recall@5 went from 0.61 to 0.78 on the 80-question eval set." This chapter sets up the smallest version of that loop you can maintain — a few dozen labeled questions, two or three metrics, a script that runs in under a minute and prints a number. The same shape from `mod-102` chapter 7 (golden sets), applied to retrieval.

## Two kinds of "is it good?"

A RAG answer can be wrong for two distinct reasons. Keep them separate; they are debugged separately.

- **Retrieval quality.** Did the relevant chunks make it into the top-K? If not, no prompt-engineering on top can save you. The model is being asked a question without the answer.
- **Answer quality (groundedness).** Given the chunks that *did* come back, did the model produce an answer that is faithful to them, cites them correctly, and refuses when it should?

Bad retrieval + good generation → fluent, wrong answer. Good retrieval + bad generation → the right snippet sitting in context next to a hallucination. Both happen. You measure them separately.

## A small labeled eval set

Build the eval set by hand. 30–80 questions is plenty to get started; 200 is "robust." Each row of the set looks like this:

```jsonl
{"q": "Can I return open headphones?", "answer_doc": "handbook-returns", "answer_chunk_ids": ["handbook-returns#0004"], "expected_answer_snippet": "opened electronics within 14 days"}
{"q": "Do you ship to Alaska?", "answer_doc": "handbook-shipping", "answer_chunk_ids": ["handbook-shipping#0002"], "expected_answer_snippet": "5–7 business days"}
{"q": "What is the warranty on third-party items?", "answer_doc": null, "answer_chunk_ids": [], "expected_answer_snippet": null, "expected_refusal": true}
```

Three things every row should have:

- **A question** the user might really ask. Stretch into the *phrasings users actually use*, not how the policy is written.
- **The correct chunk_id(s).** The chunks that contain the answer. You produce this list by chunking your corpus first, browsing it, and recording which chunks answer which questions. Painful and worth it.
- **A small snippet of the expected answer.** Not the whole answer — just a substring you can look for. Lets you grade "is the answer in the answer" with a substring match instead of an LLM.

Include a *refusal* class — questions whose answer is not in the corpus. Around 10–20% of the set. Without these you cannot measure whether the system over-answers.

This file is precious. Check it in. Treat changes to it like changes to a test suite.

## Retrieval metrics

Two metrics carry most of the weight for a beginning eval. Both are computed entirely on the *retrieved chunk_ids* — no model call needed.

### Recall@K

"Of the chunks I declared correct for this question, what fraction did the retriever pull into its top-K?"

```python
def recall_at_k(retrieved_ids: list[str], correct_ids: set[str], k: int) -> float:
    if not correct_ids:
        return 1.0  # nothing to recall; treat as success
    top = set(retrieved_ids[:k])
    return len(top & correct_ids) / len(correct_ids)
```

Average across the eval set:

```python
mean_recall = sum(recall_at_k(r, c, K) for r, c in dataset) / len(dataset)
```

Recall@K is the single most useful retrieval number. "Did the answer make it into the prompt?" If it did not, generation cannot recover.

### MRR (Mean Reciprocal Rank)

"How high in the ranking is the first correct chunk?"

```python
def reciprocal_rank(retrieved_ids: list[str], correct_ids: set[str]) -> float:
    for i, cid in enumerate(retrieved_ids, start=1):
        if cid in correct_ids:
            return 1.0 / i
    return 0.0
```

MRR rewards getting the right chunk *near the top* of the ranking, not just somewhere in the top-K. Two retrievers can have the same recall@5 — one putting the right chunk at position 1, the other at position 5 — and MRR will separate them. Useful when you are choosing a K (a high-MRR retriever can use a smaller K).

### nDCG (mention, don't implement yet)

Normalized Discounted Cumulative Gain handles the case where chunks have *graded* relevance ("very relevant," "somewhat relevant," "not relevant") instead of binary. For this module's binary labels, recall@K and MRR are enough. Reach for nDCG when you start labelling partial-credit chunks. <!-- needs-research: link to a primary definition such as the Microsoft Learn or scikit-learn `ndcg_score` documentation -->

## Answer-quality metrics

Once retrieval is correct, generation can still go wrong. Three things to measure:

### Answer-substring presence

The cheapest answer evaluator: does the model's response contain the `expected_answer_snippet`?

```python
def answer_contains(answer: str, expected_snippet: str | None) -> bool:
    if expected_snippet is None:
        return True
    return expected_snippet.lower() in answer.lower()
```

This is brittle (paraphrases fail), but it is fast, cheap, and a real signal. Treat it as a lower bound on quality. Failures are usually real failures; passes are usually fine.

### Citation correctness

Did the model cite chunk_ids that actually appeared in the retrieved set? Did it cite *the right* ones?

```python
def cited_ids(answer: str) -> set[str]:
    # crude extractor; tighten for your citation format
    import re
    return set(re.findall(r"\[chunk_id:\s*([\w\-#]+)\]", answer))

def citations_valid(answer: str, retrieved_ids: set[str]) -> bool:
    cited = cited_ids(answer)
    return bool(cited) and cited <= retrieved_ids
```

Hallucinated citations (ids the model invented) are an unambiguous bug. Missing citations on a non-refusal answer are also a bug.

### Refusal correctness

For each question marked `expected_refusal: true`, check that the model actually refused. A simple heuristic on the answer:

```python
REFUSAL_PHRASES = (
    "i do not see that",
    "not in the documents",
    "not in the context",
    "i don't have information",
)

def refused(answer: str) -> bool:
    a = answer.lower()
    return any(p in a for p in REFUSAL_PHRASES)
```

Then:

- **Over-answer rate** = fraction of `expected_refusal` questions where the model gave a confident answer. Should be near zero.
- **Over-refuse rate** = fraction of answerable questions where the model refused. Should also be small. (Tighten the refusal phrase list to avoid false positives.)

### LLM-as-judge (optional, second tier)

Once the cheap metrics are not discriminating, bring in an LLM-as-judge: a separate model call that grades the answer against the expected snippet and the retrieved chunks. `mod-102` chapter 7 has the full recipe. Use it sparingly — it is the most expensive metric and the noisiest.

## A working eval harness

Wire the pieces above into one script:

```python
import json
from statistics import mean

def evaluate(retriever, answerer, dataset, k: int = 5) -> dict:
    rows = []
    for ex in dataset:
        retrieved = retriever(ex["q"], k=k)            # list[dict] with chunk_id
        retrieved_ids = [c["chunk_id"] for c in retrieved]
        correct = set(ex.get("answer_chunk_ids") or [])
        result = answerer(ex["q"], retrieved)          # str
        rows.append({
            "q":                ex["q"],
            "recall_at_k":      recall_at_k(retrieved_ids, correct, k),
            "mrr":              reciprocal_rank(retrieved_ids, correct),
            "answer_ok":        answer_contains(result, ex.get("expected_answer_snippet")),
            "citations_ok":     citations_valid(result, set(retrieved_ids)),
            "refused":          refused(result),
            "expected_refusal": bool(ex.get("expected_refusal")),
        })
    summary = {
        "n":                len(rows),
        "recall_at_k_mean": mean(r["recall_at_k"] for r in rows),
        "mrr_mean":         mean(r["mrr"] for r in rows),
        "answer_ok_rate":   mean(r["answer_ok"]    for r in rows),
        "citations_ok_rate": mean(r["citations_ok"] for r in rows),
        "over_answer_rate": mean(r["refused"] is False and r["expected_refusal"] for r in rows),
        "over_refuse_rate": mean(r["refused"] is True  and not r["expected_refusal"] for r in rows),
    }
    return {"summary": summary, "rows": rows}
```

Run it. Print the summary. Save the rows. The summary goes in your PR description; the rows go in a git-ignored `eval_runs/` directory so you can compare runs after the fact.

A reasonable thing to put in your README:

```
$ python eval.py
n=64  recall@5=0.78  mrr=0.61  answer_ok=0.83  citations_ok=0.91
       over_answer=0.05  over_refuse=0.02
```

That is a number you can ratchet.

## Sweeping the knobs

The whole point of the eval is to make tuning empirical. The knobs you almost certainly want to sweep:

| Knob                | Typical range            | What it usually moves              |
|---------------------|--------------------------|------------------------------------|
| `chunk_size`        | 300, 600, 1200 tokens    | recall@K, MRR                      |
| `overlap`           | 0, 100, 200 chars        | recall@K (boundary-straddling Qs)  |
| `K`                 | 3, 5, 10                 | recall@K (up), prompt cost (up), answer focus (down) |
| Chunker             | structural vs recursive  | recall@K, MRR                      |
| Embedding model     | small vs large           | recall@K, MRR, cost                |
| Score floor (refusal) | 0.0 to 0.5             | over_refuse vs over_answer trade   |
| K-rewrite           | off / 1-shot query rewrite | recall@K (cheap win)              |

Sweep one at a time. Write the configuration into your `manifest.json` from chapter 5 so every run is reproducible. Plot. Pick the row that wins on the metric you care about, not the row you were rooting for.

## Where the small eval breaks down

Honesty: 50 hand-labeled questions is enough to *catch regressions* and *direct tuning*. It is not enough to declare a system "good for production." The known limits:

- **Coverage.** Real users ask things you did not anticipate. Sample real questions from logs once you have any; add them to the set.
- **Distribution shift.** A handbook update changes which questions are answerable. The eval set ages. Re-label on a cadence (monthly is reasonable).
- **Multi-chunk answers.** Some answers genuinely need two or three chunks combined. The recall@K computation above credits any overlap; it does not require *all* correct chunks to appear. Adapt by recording "all of," "any of," or "majority of" semantics per row, depending on the question.
- **Adversarial questions.** Users will eventually ask the system about its prompt, ask it to ignore its instructions, or feed it questions designed to extract data it should not surface. The small eval set will not catch these; track them separately as a "safety" suite.

The small eval is a first instrument. Add labeled real-user queries on top; you will be replacing the seed set within a quarter.

## CI and "did this PR make it worse?"

Wire the eval into CI. The cheapest shape:

- Run on every PR that touches the chunker, the retriever, the index code, or the prompt.
- Cache the index between runs by chunk-size config to keep the wall clock down.
- Print the delta against `main` and fail the build if any of {`recall_at_k`, `answer_ok_rate`, `citations_ok_rate`} drops by more than a small threshold (e.g. 0.03).
- Make the threshold tunable; otherwise legitimate tradeoff PRs become a fight with CI.

The point of CI here is not to enforce a number; it is to make sure changes to the retrieval surface are *seen*. A 0.08 drop in recall@5 is sometimes an intentional tradeoff. A 0.08 drop in recall@5 that nobody mentioned in the PR is a regression.

## Common mistakes

- **Eyeballing.** "It looks good on my three questions" is not a number.
- **Letting the eval set track the system.** If you keep adding the failures the system already gets right, recall@K rises while real quality stays flat. Add cases the system *struggles* with.
- **Conflating retrieval and answer quality.** Optimizing the prompt when retrieval is the bottleneck wastes weeks.
- **Measuring only top-1.** Recall@5 and MRR catch ranking issues that top-1 hides.
- **Skipping refusal cases.** Without them you cannot tell the system from a confident hallucinator.
- **Re-running the eval but not the ingest.** A chunker change requires re-ingesting; otherwise you are evaluating the old index against the new query code.

## Summary

Evaluation turns retrieval from a vibe into a number. Build a small (30–80 question) labeled set that includes refusal cases; measure recall@K and MRR on retrieval, and answer-substring presence + citation validity + over-answer/over-refuse rates on generation. Sweep `chunk_size`, `overlap`, `K`, and chunker choice against the metric you care about. Wire the summary into CI so regressions are seen. The goal is not a perfect number; it is a number that moves when the system does.
