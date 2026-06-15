# Decomposition, Chain-of-Thought, and Multi-Step Prompts

## Why this matters

For hard tasks, the question is not "what is the right prompt?" — it is "should this be one prompt at all?" Decomposition is the move where you stop wrestling with a single giant call and break the job into smaller, easier calls. Together with the related "ask the model to think step by step" pattern (chain-of-thought, or CoT), it is the most effective tool you have for tasks that require reasoning, planning, or multiple passes over the same input.

It is also the gateway to the agent loops you will build in `mod-105`. An agent is, in large part, a structured way of letting the model decompose its own work.

## Two related ideas

People often blur these together; keep them separate:

- **Chain-of-thought (CoT)** — keep the work in a **single model call**, but ask the model to produce reasoning before the final answer. The model writes its own scratchpad, then commits.
- **Decomposition** — split the task across **multiple model calls** (or multiple turns), each handling a subproblem with a focused prompt.

CoT is a prompting move. Decomposition is an application-architecture move. You will often use both.

## Chain-of-thought, briefly

The classic finding (Wei et al., 2022, "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models") is that prompting a model with "let's think step by step" — or with a few-shot example that shows step-by-step reasoning — substantially improves performance on multi-step problems.

Modern instruction-tuned chat models often do step-by-step reasoning by default for hard problems, and several providers now expose explicit **"thinking"** or **"reasoning"** modes that allocate hidden reasoning tokens before the final reply. Practical rules:

- **Ask for reasoning when accuracy matters more than latency or cost.** Reasoning is paid for in output tokens; non-trivial CoT can multiply the output bill.
- **Do not show the user the reasoning.** Either ask the model to emit reasoning in a dedicated section you discard, or use the provider's hidden-thinking mode. Users want answers, not a tour of the model's stream of consciousness.
- **Do not ask for reasoning when you are doing strict structured output.** A "think then JSON" prompt routinely produces prose-then-JSON, which then fails to parse. Either separate the steps into two calls, use a structured-output mode that suppresses prose (chapter 5), or put the reasoning into a dedicated JSON field whose contents you discard.
- **Beware of placebo effects.** "Let's think step by step" on tasks the model already nails zero-shot just costs you tokens. Use evals (chapter 7) to measure whether reasoning actually moves the score.

A common chain-of-thought shape:

```text
SYSTEM:
You are a careful arithmetic word-problem solver.

For each problem:
1. In a section titled "Reasoning:", show your work step by step.
2. Then, on a new line, write "Answer: <number>" and nothing else.

USER:
Anna has 3 baskets with 12 apples each. She gives away 1/4 of the apples
to her brother. How many apples does she have left?
```

For programmatic use you would then parse out the final `Answer:` line and ignore the reasoning. Even better: use a structured-output mode (chapter 5) to put reasoning in a `reasoning` JSON field and the answer in an `answer` field.

## When to decompose into multiple calls

The smell test for decomposition: when you find yourself adding "and also...", "but make sure to...", and "if that, then..." to a single prompt, you are probably looking at two prompts.

Concrete signals:

- The task has **clearly separable subtasks** (e.g., "extract the entities, *then* answer the question about them"). Each subtask is easier in isolation than the bundle.
- The intermediate result is **useful by itself** (you want to log it, show it, cache it, or re-use it).
- The task **requires different settings** per stage — e.g., a creative draft (`temperature=0.7`) followed by a structured extraction (`temperature=0`).
- The single-call output is **hard to validate** because failure could be in any of several stages. Splitting lets you assert at each step.

A worked example. "Given this support email, classify it, draft a reply, and produce JSON we can store" as one prompt is brittle. As three calls:

```
classify(email)          -> { category, urgency }            (cheap, low T)
draft_reply(email, cls)  -> reply_text                       (mid-tier model)
to_record(email, cls, r) -> { ticket_id, category, body }    (deterministic glue)
```

Each call has one job. Each output is something your validator can check. If the reply is bad, you know which stage produced it. The third call is often plain Python — you do not need the LLM for it.

## When *not* to decompose

Decomposition is not free. Each extra call adds:

- A round-trip of latency.
- A re-pay of input tokens (system prompt, examples, retrieved context that has to be repeated).
- Another opportunity for things to fail or drift.

So:

- **If the task is small and the model handles it zero-shot, leave it alone.**
- **If the subtasks share a lot of context**, prefer chain-of-thought in one call to avoid re-paying for the context every time.
- **If latency is the binding constraint**, prefer one call. Two sequential calls double the user-facing wait.

A useful mental rule: decompose for *correctness and validatability*; consolidate for *latency and cost*.

## Patterns you will reuse

A handful of decomposition shapes show up over and over:

- **Plan, then execute.** First call produces a numbered plan (or a list of items to process). Second call (or a loop of calls) executes against the plan.
- **Map / reduce.** First call processes each chunk of a large input. Second call aggregates the per-chunk outputs. Standard pattern for long-document summarization or analysis.
- **Draft, then refine.** First call produces a rough draft. A second call critiques or rewrites it against rules ("make it shorter", "match this style", "remove anything not supported by the source").
- **Extract, then answer.** First call pulls the relevant facts from a long source into a small intermediate object. Second call answers using only that object. Reduces hallucination because the answer call sees only what was extracted.
- **Self-check.** First call produces an answer. Second call grades the answer against a checklist and either accepts it or asks for a revision. Cheap version of an evaluator loop (chapter 7).

You will see these reappear when we build the first agent in `mod-105`. The agent loop is essentially "plan, then execute" running until a stop condition.

## Prefer code over more LLM calls when you can

A subtle but important point: not every step needs to be an LLM call. Once an upstream call has produced structured output, downstream work that is deterministic — formatting a date, picking the highest-priority item, dispatching to a function — should be plain code. LLM calls in the middle of a pipeline cost latency and money and add a non-deterministic layer to logic that did not need one.

A useful rule: **LLMs at the boundaries, code in the middle.** Let the model do the parts that need fuzzy language understanding (interpretation, generation, judgment), and let your code do the parts that are crisply defined.

## A worked decomposition

Single-call attempt:

```text
You are a legal assistant.
Given the contract below:
- summarize it in 3 bullets,
- list every party,
- list every termination clause with its section number,
- and return JSON.

<contract text>
```

This prompt is doing four jobs at once and will be hard to validate. A cleaner decomposition:

```python
# 1. cheap classification: is this even a contract we can parse?
kind = classify(text, model="cheap")          # {"kind": "service|nda|other|unknown"}
if kind["kind"] == "unknown":
    return refuse()

# 2. extraction: facts only, no prose
facts = extract_facts(text, model="workhorse")  # {"parties": [...], "termination_clauses": [...]}

# 3. summary: short, prose, separate call
summary = summarize(text, model="workhorse")    # str

# 4. assemble (plain Python, no LLM)
return {
    "summary": summary,
    "parties": facts["parties"],
    "termination_clauses": facts["termination_clauses"],
}
```

Now each step is testable on its own, can use a different model tier, and can be cached independently.

## Common mistakes

- **Asking for reasoning *and* strict JSON in the same call.** The reasoning tends to leak out as prose around the JSON. Split, or use a structured field for reasoning.
- **Forgetting that each step re-pays for context.** If you decompose a single 50K-token document review into five calls, you may pay for the document five times. Either feed each step a pre-extracted subset, or use a single call with chain-of-thought.
- **Decomposing tasks the model already does well.** Adds latency and cost for no benefit. Measure first.
- **Using the LLM as glue.** "Take this JSON and rewrite the dates as YYYY-MM-DD" is a regex, not a model call.
- **No evaluation per stage.** If you decompose into four calls and only evaluate the final answer, you have just made debugging four times harder. Add cheap assertions at each step.

## Summary

Chain-of-thought asks the model to reason before answering, in one call. Decomposition splits the task across multiple calls (or LLM-and-code hybrids). Use CoT when accuracy beats latency and the model is doing reasoning; use decomposition when subtasks are separable, individually validatable, or want different settings. Do not decompose what the model handles in one shot, and do not use an LLM call where a few lines of code would do.
