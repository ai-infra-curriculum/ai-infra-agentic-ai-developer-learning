# exercise-01: Prompt Patterns

**Estimated effort:** 2 hours

## Objective

Apply the core prompting patterns — instruction, role, few-shot examples, and decomposition — to a single task and watch how each move changes the output. By the end you will have four versions of a prompt for the same problem, a short notes file explaining what each change bought you, and a clear sense of which patterns are worth their tokens for *your* task.

## Background

This exercise covers material from:

- [Chapter 1 — Prompting as Engineering](../01-prompting-as-engineering.md)
- [Chapter 2 — Anatomy of an Effective Prompt](../02-anatomy-of-an-effective-prompt.md)
- [Chapter 3 — Few-Shot Prompting](../03-few-shot-prompting.md)
- [Chapter 4 — Decomposition, Chain-of-Thought, and Multi-Step Prompts](../04-decomposition-and-reasoning.md)

## Prerequisites

- Completed `mod-101-llm-fundamentals` exercise-01 (you can make a chat call and parse the response).
- An API key for one provider (Anthropic or OpenAI is fine). Set a small spend cap; this exercise should cost cents.
- Python 3.10+ or Node 20+, and the relevant SDK installed.

## The task

You are building a triage step for an e-commerce support inbox. Given a raw customer message, the model must produce:

- A **category**, one of: `billing`, `shipping`, `returns`, `product`, `other`, `unknown`.
- An **urgency**, one of: `low`, `normal`, `high`.
- A one-sentence **summary** (≤ 140 characters) suitable for a ticket title.

Use the same dataset for every version of the prompt so changes are attributable. A reasonable corpus is in `inputs.txt` below — copy it into your repo, or write your own 8–12 messages that include the obvious cases plus a few edge cases (empty input, multilingual content, an ambiguous one, an angry one, a polite one with no actual issue).

```text
# inputs.txt — one message per line, blank lines preserved
Please refund order #4421, the box arrived crushed.
Hi, when will my order ship? It's been a week.
I was charged twice for the same purchase yesterday — please fix this asap.
Do you carry the blue color in size medium?
The website keeps logging me out. I'm trying to check out.

The replacement headset still doesn't work after the firmware update.
hola, mi pedido nunca llegó y necesito el reembolso ya
Hi! Just wanted to say I love the new packaging.
Where is my order!!!!!!
```

For each prompt version you build, run it against every message and collect the results in a simple table (CSV is fine):

```
input,version,category,urgency,summary,latency_ms,stop_reason
```

Use `temperature=0` and pin the model ID for all runs.

## Tasks

### 1. v0 — bare instruction (zero-shot)

A one-line prompt. No role, no format, no examples.

- System: empty or a one-liner.
- User: "Classify the following message and produce category, urgency, summary: <message>".

Run all inputs through it. Note for each input:

- Did the output parse cleanly?
- Did the categories use the exact label set you specified?
- Was the urgency consistent (e.g., did the model invent `critical`)?

### 2. v1 — role + structured instructions + format

Rewrite using the anatomy from chapter 2: role, task, constraints, named label sets, explicit JSON format. No examples yet.

- Put the full label/format spec in the **system** message.
- Put only the input in the **user** message.
- Specify what to do with ambiguous or empty input (return `category: "unknown"`).
- Specify the output as a single JSON object and forbid prose around it.

Re-run all inputs. Compare against v0:

- Which inputs are now classified differently?
- Which inputs still fail to parse (if any)?
- Did the summary length get more consistent?

### 3. v2 — add few-shot examples

Add 2–4 worked input/output pairs to the system prompt, picked from chapter 3's guidance:

- One canonical example per common category.
- One borderline / edge-case example (the ambiguous one, the multilingual one, or the empty one).
- Same JSON output format as v1, exactly.

Run all inputs again. Specifically note:

- The case(s) you added an example for — did the output stabilize?
- Any case that *regressed* (worked in v1, broken in v2). Few-shot can over-fit; this is the most common surprise.

### 4. v3 — decomposition

Now split the task into two model calls (chapter 4):

1. `classify(text) -> {category, urgency}` — strict, `temperature=0`, JSON-only.
2. `summarize(text) -> str` — short, prose, `temperature=0.2`, capped at 140 characters.

Then assemble the final record in plain Python.

Run all inputs again. Compare against v2:

- Is each call easier to reason about?
- Is the total latency higher? By how much?
- Are there cases where a stage produces an obviously wrong result that the other stage now hides or amplifies?

### 5. Pick a winner — and say why

In `NOTES.md`, recommend one of v0–v3 for production. The recommendation must reference *evidence* from your runs (counts of failures, examples of regressions, latency numbers), not preference.

Bonus: identify one case where the "wrong" version was actually better, and explain what that says about the limits of the patterns.

## Starter guidance

A reasonable layout:

```
exercise-01/
  prompts/
    v0_bare.txt
    v1_role_format.txt
    v2_few_shot.txt
    v3_classify.txt
    v3_summarize.txt
  inputs.txt
  run.py
  results.csv
  NOTES.md
```

`run.py` should accept `--version v0|v1|v2|v3`, load the relevant prompt(s), iterate over `inputs.txt`, call the API, parse the result with `json.loads` (after stripping any code fences), and append a row to `results.csv`. Reuse the multi-turn skeleton from `mod-101` exercise-01.

Things you should *not* do in this exercise:

- Build a validation library yet — that is exercise-03.
- Use vendor JSON mode or structured outputs yet — that is exercise-02. For now, ask for JSON in the prompt and parse defensively.
- Add retries or repair loops.

## Acceptance criteria

You can demonstrate:

- Four prompt versions (v0–v3) checked into the repo as separate files.
- A `results.csv` with one row per (input, version) pair, including the parsed fields, latency, and stop reason.
- A `NOTES.md` that, for each version step, identifies at least one input whose output changed and explains *why* the prompt change caused it.
- A reasoned recommendation for which version to ship, grounded in the data you collected.
- All runs used the same model ID and `temperature=0`, both recorded in `NOTES.md`.

## Reflection

Answer briefly in `NOTES.md`:

1. Where did few-shot examples *hurt* the output? What change to the example set would fix that?
2. Did decomposition (v3) actually improve correctness for any input, or only improve maintainability? Be honest.
3. Which prompt block earned the most per-token (system, user, examples)?

## Stretch goals

- Add a fifth version (v4) that uses chain-of-thought: ask the model to "reason: <one paragraph>" in a dedicated JSON field, then write `category`/`urgency`/`summary`. Compare against v2.
- Run v2 at `temperature=0.7` three times per input and report variability per field. Connect the result back to the "deterministic sampling" discussion in `mod-101` chapter 4.
- Render each version's prompt with a small templating step (Jinja2 or string `format`) so the schema lives in one place and is interpolated into the prompt. This previews the prompt-as-code idea from chapter 1.
