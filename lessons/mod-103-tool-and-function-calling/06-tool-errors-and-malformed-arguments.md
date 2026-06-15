# Handling Tool Errors and Malformed Arguments

## Why this matters

Schemas help, descriptions help, `tool_choice` helps — but in production the model still sends you arguments that miss a required field, hand you a string where an integer was expected, or invoke a refund tool with `amount_cents=-100`. Networks blip. Downstream APIs return 503. A model that gets a Python traceback dumped into its next turn will often retry the same bad call with the same bad arguments.

The previous chapters set up the happy path. This chapter is the unhappy path: how to validate the model's arguments, how to surface tool failures so the model can recover, and how to keep one bad call from melting your loop.

## The four failure modes

Almost every tool failure falls into one of four buckets. Knowing which bucket you are in tells you how to respond.

1. **Malformed arguments.** The model emitted arguments that violate the schema in a way the provider did not catch — extra keys, wrong types, missing required fields, invalid enum values.
2. **Expected business outcomes.** The order does not exist. The customer is not eligible for a refund. The product is out of stock. These are *not* errors in the program; they are facts the function reports.
3. **Transient failures.** The downstream API timed out, returned 503, or hit a rate limit. Retrying might succeed; failing forever is wrong.
4. **Bugs and unexpected exceptions.** Your code raised `KeyError`, a divide-by-zero, an `AttributeError`. The function itself broke.

Each gets a different response. Conflating them — "anything that goes wrong is an error" — produces a loop that is sometimes too aggressive (retries on a real "order not found") and sometimes too tolerant (forgives genuine bugs).

## Step 1: validate arguments at the boundary

The first defense is the same boundary discipline from `mod-102` chapter 6, applied to tool arguments. Treat `block.input` (Anthropic) or `json.loads(call.function.arguments)` (OpenAI) as **untrusted input**: parse, validate, then dispatch.

The simplest pattern reuses the typed args model from chapter 4:

```python
from pydantic import BaseModel, Field, ValidationError

class CreateRefundArgs(BaseModel):
    order_id: str = Field(pattern=r"^ORD-[0-9]{4,8}$")
    amount_cents: int = Field(ge=1, le=100_000)
    reason: Literal["damaged", "wrong_item", "not_received", "other"]
    idempotency_key: str = Field(min_length=8)

    model_config = {"extra": "forbid"}

def dispatch_with_validation(name: str, raw_args: dict, registry: ToolRegistry):
    fn = registry.callable(name)
    args_model = registry.arg_model(name)
    if args_model is None:
        return fn(**raw_args)
    try:
        validated = args_model.model_validate(raw_args)
    except ValidationError as err:
        # NOT an exception to the caller — the model needs to see this.
        return {"error": "invalid_arguments", "detail": err.errors()}
    return fn(validated)
```

Two things are deliberate:

- `ValidationError` is caught at the dispatcher, not raised. The whole point is to feed the failure back to the model so it can correct the call.
- The error payload uses `err.errors()`, which produces structured-enough output for the model to read field-by-field ("amount_cents must be ≥ 1") — much more useful than `str(err)`.

This single validation layer prevents most "the model invented a key" and "the model passed a string where I wanted an int" failures from ever reaching your business logic.

## Step 2: encode expected outcomes as data, not exceptions

A `get_order_status` call for a nonexistent order is not an error — the model asked a sensible question and the system has an answer ("no such order"). Return it as data:

```python
def get_order_status(order_id: str) -> dict:
    row = db.orders.find(order_id)
    if row is None:
        return {"status": "not_found", "order_id": order_id}
    return {"status": row.status, "carrier": row.carrier, "eta": row.eta.isoformat()}
```

The model now sees `{"status": "not_found", "order_id": "ORD-9999"}` as the tool result, which is enough to write a human-readable reply ("I couldn't find an order with that ID — can you double-check the number?"). If you had raised `OrderNotFound`, the dispatcher would have caught it and either retried pointlessly or failed loudly.

Rule of thumb: if the function's outcome is something a user would understand, return it as data. Reserve exceptions for cases where the function itself broke.

## Step 3: convert tool exceptions into tool results

When something *does* go wrong — a downstream 503, an unexpected `KeyError` — you have two choices: bubble the exception out (the whole conversation fails), or convert it into a structured tool result and let the model react. For most apps, the second is what you want:

```python
def dispatch(name: str, raw_args: dict) -> dict:
    try:
        validated = validate(name, raw_args)
    except ValidationError as err:
        return {"error": "invalid_arguments", "detail": err.errors()}

    try:
        return registry.run(name, validated)
    except KnownTransientError as err:
        return {"error": "transient", "detail": str(err)}
    except DownstreamError as err:
        return {"error": "downstream", "detail": str(err)}
    except Exception as err:
        log.exception("tool %s crashed", name)
        return {"error": "internal", "detail": "internal error"}
```

Notes:

- **The error payload itself is sent to the model as a tool result.** That is the trick: failures become context. The provider expects `tool_result` blocks to have an `is_error` flag on Anthropic; OpenAI does not have a dedicated flag, but the convention is the same — return a structured `{"error": ...}` payload.

  ```python
  # Anthropic: mark the result as an error so the model can react.
  tool_results.append({
      "type": "tool_result",
      "tool_use_id": block.id,
      "content": json.dumps(payload),
      "is_error": True,
  })
  ```

- **Distinguish categories.** `invalid_arguments` is the model's fault; `transient` is the network's; `internal` is yours. Use different category names so you can analyze failures later.
- **Do not leak internal details.** The model will repeat whatever you put in `detail`. Avoid raw stack traces and user PII in error payloads.

This pattern lets the model recover gracefully from most failures: it sees the error, reads the detail, and either retries with corrected arguments, calls a different tool, or tells the user it could not complete the action.

## Step 4: cap retries

The model is reasonably good at recovering from `{"error": "invalid_arguments", "detail": ...}` — it usually fixes the arguments on the next turn. It is *not* good at giving up. Without limits, a model that cannot figure out how to fix a call will retry it forever. Two guards:

1. **A step cap on the outer loop.** Already in chapter 3's `run_with_tools(max_steps=10)`. Without this, nothing else saves you.
2. **A per-tool retry budget.** Count how many times the model has called the same tool with the same error category in this conversation and abort early.

A compact per-call counter:

```python
from collections import Counter

class ToolBudget:
    def __init__(self, max_per_tool: int = 3, max_total: int = 12):
        self.per_tool = Counter()
        self.total = 0
        self.max_per_tool = max_per_tool
        self.max_total = max_total

    def charge(self, name: str) -> None:
        self.per_tool[name] += 1
        self.total += 1
        if self.per_tool[name] > self.max_per_tool:
            raise BudgetExceeded(f"too many calls to {name!r}")
        if self.total > self.max_total:
            raise BudgetExceeded("too many tool calls overall")
```

Call `budget.charge(block.name)` before dispatching. Tune `max_per_tool` per tool if needed — search tools may legitimately call more often than refund tools.

## Step 5: a small "error → feedback" example

A complete defensive dispatch step for a refund tool:

```python
def safe_dispatch(block, registry, budget) -> dict:
    budget.charge(block.name)

    args_model = registry.arg_model(block.name)
    try:
        validated = args_model.model_validate(block.input)
    except ValidationError as err:
        return {
            "is_error": True,
            "payload": {"error": "invalid_arguments", "detail": err.errors()},
        }

    try:
        result = registry.run(block.name, validated)
    except RetriableError as err:
        return {
            "is_error": True,
            "payload": {"error": "transient", "detail": str(err)},
        }
    except Exception:
        log.exception("tool %s crashed for input %s", block.name, block.input)
        return {
            "is_error": True,
            "payload": {"error": "internal", "detail": "internal error"},
        }

    return {"is_error": False, "payload": result}
```

And the corresponding `tool_result` build in the loop:

```python
out = safe_dispatch(block, registry, budget)
tool_results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": json.dumps(out["payload"]),
    "is_error": out["is_error"],
})
```

The model sees the failure on the next turn, reads it, and continues. Most of the time it apologizes to the user and offers an alternative; sometimes it corrects an argument and tries again. Either way, the host did not crash and the budget did not blow out.

## A pattern to avoid: silent fallbacks

A tempting shape is "if the tool fails, return a plausible-looking default." Do not. Two reasons:

- The model will use the default as if it were a real result and produce confident wrong answers.
- You will never see the failures because they look like successes in your logs.

Better to return an error payload, let the model react, and surface the failure to the user. "I couldn't reach our order system — can you try again in a minute?" is a better experience than a hallucinated tracking number.

## Logging that is actually useful

For every tool call, log enough to debug later:

- Tool name and the input the model sent (post-validation if available).
- Whether validation passed.
- The result category (success / error category).
- The function's elapsed time.
- The model ID and the step number in the loop.

Two reasons this earns its keep:

- When something looks weird in production, the question is usually "did the model send bad args, or did the function break?" The log answers this immediately.
- The pile of "invalid_arguments" cases is your best signal that a tool description or schema needs sharpening. Bucket them and fix the top one each week.

Respect PII: tool arguments may contain customer data. Apply the same redaction rules as the rest of your application log.

## What good error feedback looks like (and what doesn't)

The model reads tool results as text. The difference between a recoverable failure and a stuck loop is often in the wording of the error payload.

Good — names the problem, names the offending field:

```json
{
  "error": "invalid_arguments",
  "detail": [
    {"field": "amount_cents", "msg": "must be greater than or equal to 1"}
  ]
}
```

Bad — opaque, gives the model nothing to fix:

```json
{"error": "bad input"}
```

Good — distinguishes "you should not retry" from "you can retry":

```json
{"error": "not_eligible", "detail": "refunds are not available for orders older than 90 days"}
```

Bad — vague enough that the model will try `create_refund` again with random tweaks:

```json
{"error": "failed"}
```

The model is, after all, just reading text. Write error payloads the way you would write API responses for a junior engineer: specific, structured, and honest about whether the action is recoverable.

## Common mistakes

- **Letting `ValidationError` bubble out of the dispatcher.** The model never gets a chance to fix the call.
- **Returning `None` from a tool.** `None` serializes to `"null"` and the model treats it like a successful empty answer.
- **Raising `OrderNotFound` instead of returning `{"status": "not_found"}`.** Confuses an expected outcome with a bug.
- **No `is_error` flag (Anthropic).** The model gets the payload but does not know it indicated a failure.
- **Retrying the same call without changing inputs.** If the args were wrong, the model needs to *see* what was wrong; if the network was wrong, your code should retry once silently.
- **No budget.** A misbehaving conversation calls `create_refund` thirteen times and the user notices on their next statement.
- **Stack traces in error payloads.** Internal detail leaks to the model and, via the model, sometimes to the user.

## Summary

Tool errors are part of the protocol, not the exception. Validate arguments at the dispatcher, treat business outcomes as data, convert exceptions into structured tool-result payloads with an `is_error` flag, and cap retries. Write error messages the model can act on — name the field, name the problem, say whether retrying makes sense. With this layer in place, your tool surface tolerates the bad day: malformed calls become corrections, transient failures become apologies, and your loop keeps running.

This is the last chapter of the module. Putting all six together, you can take a tool-shaped task, design a small surface, wire it to Python, run a clean loop, decide when to call and when not to, and survive the failures. The next module (`mod-104`) reuses every piece of this protocol to wire retrieval as a tool. Bring this chapter with you.
