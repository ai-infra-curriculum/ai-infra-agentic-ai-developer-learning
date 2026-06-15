# Parsing and Validating Output at the Boundary

## Why this matters

Model output is **untrusted input** to your application. It looks like prose written by a careful collaborator, but on the wire it is a string from a non-deterministic process that occasionally hallucinates a field, drops a closing brace, decides to wrap your JSON in a code fence, or hits `max_tokens` mid-token. If you `json.loads()` the raw text and pass it straight into a database call, you will eventually crash on the bad day.

The fix is the same boundary you already enforce for user input: parse, validate, and reject — at one well-defined edge of your code, with a typed library and an explicit failure plan.

## The boundary mental model

Think of every LLM call as a network call to a flaky third party. The model returns:

```
raw_text  ->  parsed_object  ->  validated_object  ->  domain object
```

- **raw_text** — the literal string the API gave you. Untrusted.
- **parsed_object** — what you get after `json.loads()` (or equivalent). Syntactically valid, semantically anything.
- **validated_object** — what you get after running it through a schema validator. Conforms to the shape your app expects.
- **domain object** — your application's typed class, with invariants enforced by your own code.

The job of this chapter is the middle two arrows. Skip them and your "domain object" will quietly accept garbage.

## Step 1: parse defensively

Even with a vendor JSON mode, three things go wrong often enough to handle them up front.

**Code fences and prose around the JSON.** When you have not constrained the format, you will routinely get this:

````
Sure! Here's the classification:

```json
{"category": "billing", "urgency": "high"}
```

Let me know if you need anything else.
````

A small helper that extracts the largest JSON object substring before parsing:

```python
import json, re

_JSON_OBJECT = re.compile(r"\{.*\}", re.DOTALL)

def extract_json_object(s: str) -> dict:
    s = s.strip()
    # strip a single ```...``` fence if present
    if s.startswith("```"):
        s = re.sub(r"^```[a-zA-Z]*\n?", "", s)
        s = re.sub(r"\n?```$", "", s).strip()
    # find the outermost {...}
    m = _JSON_OBJECT.search(s)
    if not m:
        raise ValueError("no JSON object found in model output")
    return json.loads(m.group(0))
```

This is a fallback for when you cannot or did not use strict structured output. With schema-constrained generation or tool calling, you can usually `json.loads` the response field directly — but keep the helper around for the cases that slip through.

**Truncation.** If the response was cut off by `max_tokens`, the JSON is incomplete and `json.loads` will raise. Always check the stop reason before you trust the body:

```python
if response.stop_reason == "max_tokens":
    raise OutputTruncated("model hit max_tokens; raise the limit or shorten the schema")
```

**Refusals and empty outputs.** Models sometimes refuse, or return an empty string when the input was bad. Treat that as a validation error, not a crash.

## Step 2: validate with a typed schema

Do not use raw `dict` access in your domain code. Use a schema/validator. Pydantic (Python) and zod (TypeScript) are the standard choices; both let you describe the shape once and get parsing + validation + a typed object.

Pydantic:

```python
from typing import Literal
from pydantic import BaseModel, Field, ValidationError

class Classification(BaseModel):
    category: Literal["billing", "shipping", "returns", "product", "other"]
    urgency:  Literal["low", "normal", "high"]
    summary:  str = Field(max_length=200)

def parse_classification(raw: str) -> Classification:
    obj = extract_json_object(raw)
    return Classification.model_validate(obj)   # raises ValidationError on shape mismatch
```

zod:

```typescript
import { z } from "zod";

const Classification = z.object({
  category: z.enum(["billing", "shipping", "returns", "product", "other"]),
  urgency:  z.enum(["low", "normal", "high"]),
  summary:  z.string().max(200),
});

type Classification = z.infer<typeof Classification>;

function parseClassification(raw: string): Classification {
  const obj = JSON.parse(extractJsonObject(raw));
  return Classification.parse(obj);   // throws ZodError on shape mismatch
}
```

Both libraries also let you generate JSON Schema from the model definition. That is exactly the schema you should hand to your structured-output API (chapter 5). Define the type once, use it for both the prompt and the validator, and the two stay in sync.

## Step 3: have a plan when validation fails

Validation failure is the interesting case. Pick a strategy per call site:

1. **Fail loudly.** Surface the error, log it, return a 4xx/5xx upstream. Right for non-critical paths where you would rather alert than guess.
2. **Retry with the same prompt.** Models are non-deterministic; a single regenerate often succeeds. Cap retries (1–3) and add backoff. Use a fresh `request_id` or seed; do not retry against the exact same cache key.
3. **Retry with the error fed back.** Send a follow-up turn that includes the validation error and asks the model to fix the response. Surprisingly effective. Beware of infinite loops; cap to one or two repair attempts.
4. **Fall back.** Use a deterministic default (`category: "unknown"`), a cheaper or stronger model, or a different code path entirely. Right when "no answer" is worse than "an answer based on a default."

A small repair loop:

```python
def call_with_repair(prompt: str, validator, *, max_repairs: int = 1):
    raw = llm.call(prompt)
    for attempt in range(max_repairs + 1):
        try:
            return validator(raw)
        except ValidationError as err:
            if attempt == max_repairs:
                raise
            raw = llm.call(
                prompt
                + "\n\nThe previous reply failed validation:\n"
                + str(err)
                + "\nReturn a corrected reply that satisfies the schema."
            )
```

Use this sparingly. A repair loop that runs on every call is a sign that your schema or prompt needs work, not that you need a bigger repair budget.

## What to do about the model lying inside a valid shape

Schema validation tells you the JSON has the right *fields*. It cannot tell you the *values* are right. The model can hand you a perfectly typed `Classification` with the wrong category. Three habits help:

- **Constrain values, not just types.** Use `Literal` / `enum` for closed sets. The model literally cannot return a value outside the enum under strict modes, and your validator catches it under non-strict modes.
- **Add business-rule validators.** Pydantic `field_validator` / `model_validator` can enforce rules like "urgency must be high when category is billing-refund." Keep them in the model definition so they run on every call site.
- **Add semantic checks where they matter.** Date in the past for a future-only field, currency outside an allowed set, ID that does not match a known prefix. These are cheap and catch the "valid-but-wrong" case.

## Logging and observability

You will not be able to fix a bad output you cannot inspect. For every LLM call, log:

- The rendered prompt (or a hash + version, if it is large and reused).
- The model ID and sampler settings.
- The raw output **and** the parsed/validated output.
- The validation error, if any.
- Token usage and stop reason from the response.

Two things this enables: you can replay any production call against a new prompt and see the diff, and you can build your golden set (chapter 7) from real, anonymized failures rather than from your imagination.

Be deliberate about PII. The raw prompt and output may contain customer data; treat your LLM log the same way you treat your application log.

## A complete boundary in one place

A small adapter that puts all of the above together:

```python
from pydantic import BaseModel, ValidationError

class Classification(BaseModel):
    category: Literal["billing", "shipping", "returns", "product", "other"]
    urgency:  Literal["low", "normal", "high"]
    summary:  str = Field(max_length=200)

def classify_message(text: str) -> Classification:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=512,
        temperature=0,
        system=CLASSIFY_SYSTEM,                # versioned, see chapter 1
        tools=[CLASSIFY_TOOL],                 # schema-constrained via tool calling
        tool_choice={"type": "tool", "name": "record_classification"},
        messages=[{"role": "user", "content": text}],
    )

    if response.stop_reason == "max_tokens":
        raise OutputTruncated()

    # extract the tool_use block
    block = next(
        (b for b in response.content if b.type == "tool_use" and b.name == "record_classification"),
        None,
    )
    if block is None:
        raise OutputMissingToolCall()

    try:
        return Classification.model_validate(block.input)
    except ValidationError as err:
        log_failure(text, block.input, err)
        raise
```

The function returns either a typed object or an exception. The rest of the application never sees raw model output. That is the whole game.

## Common mistakes

- **Trusting `try/except: pass` around `json.loads`.** Silent failures here mean silent garbage in your database. Always log and raise.
- **Using `dict.get(...)` everywhere downstream.** That style spreads "the value might not be there" across the codebase. Validate once at the boundary; use typed access after.
- **Not checking the stop reason.** Truncated JSON parses *sometimes* if the cut happened on a closing brace; you will get a "shorter than expected" object you did not catch.
- **Retrying forever.** A repair loop without a cap is a billing incident.
- **Validating only in production.** Build the validator into your tests so you fail in CI when the schema and the prompt drift apart.

## Summary

Treat the model output as untrusted input. Parse defensively, validate with a typed library, and choose a deliberate strategy for what happens when validation fails — fail, retry, repair, or fall back. Constrain values, not just types. Log enough to debug and to build your eval set from. Once this boundary is in place, the rest of your application can pretend the model is a typed function.
