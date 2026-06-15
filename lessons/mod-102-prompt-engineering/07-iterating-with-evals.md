# Iterating on Prompts with Simple Evals

## Why this matters

Without evals, "I improved the prompt" is a feeling. With evals, it is a measurement. A small, deliberately-curated set of inputs with expected outputs — your **golden set** — turns prompt engineering from vibes into something closer to test-driven development. You do not need a research lab. A spreadsheet, a thirty-line runner, and the discipline to use them is enough to keep an LLM application from quietly regressing.

This chapter is the iteration loop that the rest of the module hangs on. Every move you have learned — instructions, few-shot, decomposition, structured output, validation — is more useful when you can measure whether it helped.

## What a "simple eval" is

An eval is a function that takes a model's output for an input and returns a score (or a pass/fail). It does *not* have to be sophisticated. Useful eval shapes, in roughly increasing cost:

- **Exact match** — does the output equal the expected value? Cheap, brittle, perfect for classification.
- **Regex / substring** — does the output contain a required phrase, code identifier, or pattern?
- **Schema check** — does the output parse and validate against your schema? (You already wrote this in chapter 6.)
- **Field-level comparison** — does each required field equal (or set-equal) the expected value? Standard for extraction.
- **Functional check** — does the generated code compile, the SQL parse, the JSON validate against a schema, the unit test pass?
- **Metric-based** — embedding similarity, BLEU/ROUGE/BERTScore for text overlap. Useful for summaries but noisy.
- **LLM-as-judge** — ask a model to grade an answer against a rubric. Useful when there is no exact answer, but it has its own biases and costs.

Default to the cheapest evaluator that can distinguish "good" from "bad" on your task. A field-level check on a structured output beats an LLM judge in almost every way: it is cheaper, faster, deterministic, and you can read its code.

## Build a golden set, deliberately

A golden set is 20–200 inputs with expected outputs (or expected properties) that you trust. Properties of a good one:

- **Hand-picked, not random.** Half from real traffic where the model has succeeded, half from where it has failed. Random samples bury the failures.
- **Covers the label distribution.** For a classifier, include all classes — especially the rare ones.
- **Includes edge cases.** Empty input, near-duplicate input, the multilingual one, the one with hostile prompt-injection content, the one your last bug report was about.
- **Frozen.** Once a case is in the golden set, you do not edit its expected output unless the *requirements* changed. If you edit the expected output to match a regression, you have a placebo.
- **Versioned in the repo.** A YAML or JSONL file alongside your prompts. Diff in code review.

A reasonable shape:

```yaml
# evals/classify.golden.yaml
- id: refund-damaged-001
  input: "Please refund order #4421, the box arrived crushed."
  expected:
    category: returns
    urgency: normal
- id: billing-double-charge-001
  input: "I was charged twice for one order, please help."
  expected:
    category: billing
    urgency: high
- id: empty-001
  input: ""
  expected:
    category: unknown
    urgency: low
```

Start with twenty cases. Twenty is enough to catch most categorical regressions and not enough to be precious about.

## A thirty-line runner

Once the schema and validator from chapter 6 exist, a runnable eval is small:

```python
import json
import yaml
from pathlib import Path
from collections import Counter

def run_eval(golden_path: Path, classify_fn):
    cases = yaml.safe_load(golden_path.read_text())
    results = []
    for case in cases:
        try:
            actual = classify_fn(case["input"]).model_dump()
            ok = all(actual.get(k) == v for k, v in case["expected"].items())
            err = None
        except Exception as e:
            actual, ok, err = None, False, str(e)
        results.append({"id": case["id"], "ok": ok, "actual": actual, "error": err})

    passed = sum(r["ok"] for r in results)
    print(f"{passed}/{len(results)} passed")
    by_id = {r["id"]: r for r in results}
    for r in results:
        if not r["ok"]:
            print(f"  FAIL {r['id']}: {r['error'] or r['actual']}")
    return results
```

Wire this into your test suite or run it from the command line. Two runs of this on a 20-case set tell you more about a prompt change than reading model outputs by hand for an hour.

## The iteration loop

The boring, productive loop:

1. Run the eval on the current prompt. Record the score.
2. Make **one** change — a clearer constraint, an extra few-shot example, a different temperature.
3. Run the eval again.
4. If the score went up and no previously-passing case broke, commit the change.
5. If a previously-passing case broke, that is a regression you would have shipped. Inspect it.

A few practical refinements:

- **Pin the model.** Same `model`, same `temperature`, same prompt version. If you change the model and the prompt in the same commit, the eval delta is meaningless.
- **Run each case more than once** at the temperatures you actually use. At `temperature=0`, one run is usually enough; above that, run 3–5 times and report pass rate or majority vote.
- **Track per-case state across runs.** A spreadsheet column per prompt version is the cheapest way to see what each change cost you.
- **Eyeball failures.** The score tells you the count; the model output tells you the cause. Always read the actual failures, not just the summary.

## LLM-as-judge: when and how to use it

For tasks without a single correct answer (summaries, code reviews, freeform replies), an LLM judge can rate output against a rubric:

```text
SYSTEM:
You are grading replies from a support assistant.

For each (customer message, assistant reply) pair, score the reply on:
- helpful (1-5): does it answer the question?
- accurate (1-5): are factual claims correct based on the message?
- tone   (1-5): is the tone appropriate for a support reply?

Reply with JSON: {"helpful": int, "accurate": int, "tone": int, "notes": str}

USER:
Message: <...>
Reply:   <...>
```

Things to keep in mind:

- **Use a different model (or at least different prompt) for the judge.** Same-model bias is real: a model rates its own outputs more generously.
- **Calibrate.** Score 10–20 cases by hand and compare to the judge's scores. If they diverge, fix the rubric.
- **Make rubrics specific.** "Is this good?" is useless. "Is the answer supported verbatim by a sentence in the source?" is testable.
- **Pair with cheaper checks.** An LLM judge for tone alongside a schema validator for shape gives you a lot at moderate cost.

LLM judges are a tool of last resort for things you cannot deterministically check. Reach for exact-match, regex, schema, or field comparison first.

## Regression testing in CI

Once your eval runs in under a minute and costs less than a cup of coffee, it belongs in CI:

- Run it on every PR that touches a prompt, a schema, a tool definition, or the model selection.
- Fail the build on any regression (a case that previously passed and now fails).
- Allow the score to move within a band you set, so a single retry at temperature > 0 does not flap.
- Record the model ID and prompt hash in the CI artifact so you can attribute future regressions.

Two failure modes to watch:

- **Cost explosion.** A 200-case eval that hits a frontier model on every PR will get expensive. Use a workhorse-tier model in CI and run a small "frontier sanity check" only on release branches.
- **Flaky tests.** If your eval flakes at the temperature you use in production, that is a real flake your users will see. Do not "fix" it by adding retries to the test; fix the prompt or pin temperature lower.

## Treating prompts like model versions

Once you have evals, prompts behave a lot like model versions:

- Each prompt has a version string. Store it in the repo.
- A change in prompt is a change in behavior; it gets a PR, a review, and a score.
- Log which prompt version produced each production output, so an old logged failure can be replayed against a new prompt.
- Don't roll forward without verification. If the eval says the new prompt is worse on a class you care about, the eval wins.

This is the rest of the engineering loop chapter 1 sketched. The remainder of the track — tool use, retrieval, agents, deployment — assumes you can answer "did that change help?" with a number, not a guess.

## Common mistakes

- **Evals built from synthetic inputs only.** They miss the weird stuff your real users send. Seed from production traffic as soon as you have any.
- **Evaluating only the final stage of a decomposed pipeline.** Errors compound; each stage needs its own eval, even if it is tiny.
- **Editing the golden answer to match a regression.** That is not "the new model is better"; that is silently lowering the bar.
- **No baseline.** "We got 80% on the eval" is meaningless without "the old prompt got 64%" and "random guessing gets 20%."
- **Mistaking the eval for the product.** A high eval score on your golden set does not prove user happiness. Treat the eval as the floor, not the ceiling.

## Summary

Evals turn prompt engineering from feel into measurement. Build a small, hand-picked, frozen golden set; write the cheapest evaluator that distinguishes good from bad; run it on every change; gate CI on regressions. Pin the model and the prompt version, change one thing at a time, and look at the actual failures, not just the score. This loop is the discipline that lets every other technique in this module pay off.
