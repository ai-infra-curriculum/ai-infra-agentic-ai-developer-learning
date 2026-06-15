# exercise-03: Output Validation and a Small Eval Loop

**Estimated effort:** 2 hours

## Objective

Take the structured-output endpoint from exercise-02 and put a real boundary around it: typed validation with Pydantic (or zod), a small repair loop with a hard cap, structured logging, and a 20-case golden-set eval that runs in CI and gates prompt changes on a real number.

## Background

This exercise covers material from:

- [Chapter 5 — Strategies for Reliable Structured Output](../05-structured-output-strategies.md)
- [Chapter 6 — Parsing and Validating Output at the Boundary](../06-parsing-and-validating-output.md)
- [Chapter 7 — Iterating on Prompts with Simple Evals](../07-iterating-with-evals.md)

## Prerequisites

- Completed exercise-02 (you have at least one working structured-output strategy and a CSV of results).
- `pip install pydantic pyyaml` (or `npm install zod yaml` if you're working in TypeScript).
- The same provider/API key from exercise-02.

## Tasks

### 1. Define the schema in one place

Define `TriageResult` as a Pydantic model. Reuse exactly the fields from exercise-02 (`category`, `urgency`, `summary`, `language`, `needs_human_review`).

Requirements:

- `category` and `urgency` are `Literal[...]` types over the allowed values.
- `summary` is constrained to `<= 140` characters.
- `language` is constrained to a 2-letter lowercase code (regex pattern is fine).
- Add at least one `field_validator` or `model_validator` that enforces a *business* rule, not a type rule. Example: if `category == "billing"` and the message contains "twice"/"double-charged", `urgency` must be `"high"`. Pick a rule and document why.
- Add a method `TriageResult.json_schema_for_llm()` that returns the JSON Schema you pass to the structured-output API. Use the same definition for both the validator and the prompt.

### 2. Build the boundary function

Write `triage(message: str) -> TriageResult` that:

- Calls the LLM with your chosen structured-output strategy from exercise-02 (schema-constrained or tool-calling preferred).
- Checks the response `stop_reason` and raises a typed `OutputTruncated` exception if it indicates `max_tokens`.
- Extracts the structured payload (parsed JSON object or `tool_use.input`) and validates it with `TriageResult.model_validate(...)`.
- On `ValidationError`, raises a typed `OutputInvalid` exception that carries the raw payload and the validation error message.
- Logs the prompt version, model ID, sampler settings, token usage, raw payload, and parsed object on **every** call (success and failure). A single-line structured log (one JSON object per call) is enough.

### 3. Add a repair loop — with a cap

Wrap the boundary function with a `triage_with_repair(message, *, max_repairs=1)` that:

- Catches `OutputInvalid`.
- Sends a second call that includes the original message *and* the validation error, asking the model to fix it.
- Tries at most `max_repairs` repair attempts. After the cap, re-raises.
- Logs each repair attempt (which try, what the validation error was, what the new payload was).

Demonstrate it works on an input you can construct to deliberately fail validation under your business rule (for example, a "double-charged" billing message that you tweak to ask the model for `urgency: "normal"`).

### 4. Build the golden set

Create `evals/triage.golden.yaml` with 20 cases. Required composition:

- At least 2 cases per category (12 cases total across the 6 categories).
- 1 case per urgency level beyond what category coverage gives you.
- 2 multilingual cases (one Spanish, one any other language).
- 1 empty-input case.
- 1 instruction-injection case (from exercise-02).
- 1 case explicitly designed to trigger your business-rule validator.
- 2 "happy path" cases that should never regress.

Each case must include:

```yaml
- id: <slug>
  input: <message>
  expected:
    category: <value>
    urgency: <value>
    language: <iso>
    needs_human_review: <bool>
  # `summary` is freeform; assert on length, not content.
  notes: <one-line provenance, e.g. "from real ticket #1234, scrubbed">
```

Freeze the file. Once committed, do not edit `expected` values to make a failing run pass.

### 5. Write the eval runner

Write `run_eval.py` that:

- Loads the golden set.
- Calls `triage(case["input"])` for each case (one run at `temperature=0`; three runs at higher temperatures, reported as a pass rate).
- For each case, computes a pass/fail:
  - All expected fields match.
  - `summary` parses, is `<= 140` chars, and is non-empty (when the input is non-empty).
  - No `OutputInvalid` or `OutputTruncated` was raised.
- Prints a per-case line and a summary line: `XX/20 passed (prompt_version=v3, model=...)`.
- Returns a non-zero exit code if any case fails. (This is what makes it usable in CI.)
- Writes a `results-<timestamp>.json` artifact with the raw payloads.

### 6. Use the eval to gate a prompt change

Make one deliberate change to the prompt — add an example, tighten a constraint, swap models, or change the temperature — and run the eval before and after.

In `NOTES.md`, record:

- The before and after scores.
- Any case that regressed (passed before, failed after) and your interpretation.
- Whether you would ship the change.
- If the change *didn't* improve the score: keep the old prompt, don't talk yourself into a placebo.

### 7. Wire it into CI

Add a CI job (GitHub Actions YAML in `.github/workflows/eval.yml` is fine; a `Makefile` target is also acceptable) that:

- Runs `python run_eval.py` against the committed prompt and schema.
- Uses a model environment variable so the same script can run against a cheap model in CI and a workhorse model locally.
- Fails the build if `run_eval.py` exits non-zero.

A skeleton:

```yaml
# .github/workflows/eval.yml
name: triage-eval
on: [pull_request]
jobs:
  eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - run: python run_eval.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          MODEL: claude-haiku-4-5-20251001   # or whatever cheap tier you choose
```

You can demonstrate the workflow locally by `make eval` running the same script; you don't need a live GitHub Actions run for the exercise to count.

## Starter guidance

The Pydantic skeleton:

```python
from typing import Literal
from pydantic import BaseModel, Field, model_validator

class TriageResult(BaseModel):
    category: Literal["billing", "shipping", "returns", "product", "other", "unknown"]
    urgency:  Literal["low", "normal", "high"]
    summary:  str = Field(min_length=1, max_length=140)
    language: str = Field(pattern=r"^[a-z]{2}$")
    needs_human_review: bool

    @model_validator(mode="after")
    def billing_double_charge_is_high(self):
        # Document the rule next to it. If the rule is wrong, the eval will tell you.
        ...
        return self

    @classmethod
    def json_schema_for_llm(cls) -> dict:
        schema = cls.model_json_schema()
        # Strip / tweak as needed for your provider's strict mode.
        return schema
```

Repair-call shape (Anthropic, schema-via-tool flavor):

```python
def repair_prompt(message: str, raw: dict, err: ValidationError) -> str:
    return (
        "Your previous output failed validation:\n"
        f"{err}\n"
        "Previous output:\n"
        f"{raw}\n"
        "Return a corrected response that satisfies the schema."
    )
```

## Acceptance criteria

You can demonstrate:

- A `TriageResult` Pydantic model with `Literal` enums, length/regex constraints, **and** at least one business-rule validator.
- A `triage()` function that always returns either a typed `TriageResult` or a typed exception — never a raw dict — and logs every call.
- A `triage_with_repair()` wrapper with a hard cap on repair attempts and visible logs of each attempt.
- A 20-case `triage.golden.yaml` covering the required composition above, frozen and checked in.
- A `run_eval.py` that prints per-case pass/fail and a summary, writes a JSON artifact, and exits non-zero on any failure.
- A before/after eval result in `NOTES.md` for one prompt change, with the decision to ship or not, justified by the numbers.
- A CI workflow file or `make` target that runs the eval automatically.

## Reflection

Answer briefly in `NOTES.md`:

1. Did your repair loop ever succeed on a case that the first call failed? If so, what did the model fix? If not, what does that say about your prompt or schema?
2. What is one validation that *cannot* be a Pydantic rule and would need an LLM-as-judge or a separate downstream check? Why?
3. If you ran this eval against a brand-new model snapshot tomorrow, which case do you predict would regress first, and why?

## Stretch goals

- Add an LLM-as-judge evaluator for `summary` quality (1–5 score against a short rubric) and report mean score per prompt version. Use a *different* model as the judge to avoid same-model bias.
- Add cost reporting to the eval output: total input tokens, total output tokens, projected USD given current pricing.
- Replay one week of real production logs (or a synthetic stand-in) through `triage()` and add the failures to your golden set. Note which cases were not anticipated by your hand-curated set.
- Add a "previously passing → now failing" diff to the eval runner, so a regression's case IDs are surfaced explicitly in the CI failure message.
