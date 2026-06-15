# exercise-03: Tool Error Handling

**Estimated effort:** 2 hours

## Objective

Break your tools on purpose. Watch the model react. Add the validation + error-as-feedback layer from chapter 6 so malformed arguments and downstream failures become recoverable feedback instead of crashes. By the end you should be able to point at the four failure modes — malformed arguments, expected business outcomes, transient failures, internal bugs — in your own logs and explain how each one was handled.

This is the exercise that turns the exercise-02 prototype into something you would be slightly less embarrassed to ship.

## Background

This exercise covers material from:

- [Chapter 4 — Wiring Python Functions as Tools](../04-wiring-python-functions.md)
- [Chapter 5 — When to Call a Tool vs. Generate](../05-when-to-call-a-tool.md)
- [Chapter 6 — Handling Tool Errors and Malformed Arguments](../06-tool-errors-and-malformed-arguments.md)

## Prerequisites

- Completed exercise-02. You have the multi-tool registry, the dispatcher, and the prompt suite.
- Pydantic 2 installed (`pip install pydantic`). zod works equivalently if you are working in TypeScript.
- The same provider and model as the previous two exercises. Pin `temperature=0`.

## The task

You will harden the four-tool surface from exercise-02 along three axes:

1. **Argument validation at the boundary.** Every tool gets a Pydantic args model. The dispatcher validates first, then dispatches. Validation failures become structured tool results, not raised exceptions.
2. **Error-as-feedback.** Business-rule failures (`not_cancellable`, `not_eligible_for_refund`), transient simulated failures (a flaky search), and unexpected exceptions are all converted into structured payloads with an `is_error` flag and category. The model continues from the failure.
3. **Retry budgets.** A `ToolBudget` caps the number of calls per tool and total calls per conversation. Exhaustion raises out of the loop with a clear error.

Then you will run a set of *adversarial* prompts designed to trigger each failure mode and confirm the loop recovers gracefully.

## Tasks

### 1. Add typed args models

For each of the four tools from exercise-02, define a Pydantic model that mirrors the JSON Schema:

```python
from typing import Literal
from pydantic import BaseModel, Field

class CreateRefundArgs(BaseModel):
    model_config = {"extra": "forbid"}
    order_id:        str = Field(pattern=r"^ORD-[0-9]{4,8}$")
    amount_cents:    int = Field(ge=1, le=100_000)
    reason:          Literal["damaged", "wrong_item", "not_received", "other"]
    idempotency_key: str = Field(min_length=8)
```

Update the registry so each tool registration includes the args model:

```python
registry.register(CREATE_REFUND_TOOL, create_refund, args_model=CreateRefundArgs)
```

Make the dispatcher validate first and call the function with the validated, typed args:

```python
def dispatch(self, name: str, raw_args: dict) -> Any:
    fn = self._fns[name]
    model = self._arg_models.get(name)
    if model is None:
        return fn(**raw_args)
    validated = model.model_validate(raw_args)   # raises ValidationError
    return fn(**validated.model_dump())
```

For consistency, keep the underlying functions accepting plain keyword arguments. The typed model is for validation, not for changing function signatures.

(Optional refactor: have the function accept the args model directly. Either works; pick one and stay consistent.)

### 2. Wrap the dispatcher with `safe_dispatch`

Wrap the dispatcher to convert errors into structured tool results, per chapter 6. The function returns `{"is_error": bool, "payload": ...}`:

```python
from pydantic import ValidationError

def safe_dispatch(name: str, raw_args: dict, registry, budget) -> dict:
    budget.charge(name)
    try:
        result = registry.dispatch(name, raw_args)
        return {"is_error": False, "payload": result}
    except ValidationError as err:
        return {"is_error": True, "payload": {"error": "invalid_arguments", "detail": err.errors()}}
    except RetriableError as err:
        return {"is_error": True, "payload": {"error": "transient", "detail": str(err)}}
    except Exception:
        log.exception("tool %s crashed", name)
        return {"is_error": True, "payload": {"error": "internal", "detail": "internal error"}}
```

In the loop, build the tool result using both fields:

```python
out = safe_dispatch(block.name, block.input, registry, budget)
tool_results.append({
    "type": "tool_result",
    "tool_use_id": block.id,
    "content": json.dumps(out["payload"]),
    "is_error": out["is_error"],     # Anthropic-specific; OpenAI just uses the payload
})
```

For OpenAI, the `is_error` flag is not part of the protocol; rely on the `error` key in the payload to signal failure.

### 3. Add a retry budget

```python
from collections import Counter

class BudgetExceeded(Exception): pass

class ToolBudget:
    def __init__(self, max_per_tool: int = 3, max_total: int = 10):
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

Create a fresh budget per conversation and pass it into `safe_dispatch`. When `BudgetExceeded` raises out of the loop, return a final user-facing message such as "I wasn't able to complete this request — please reach out again or contact support." Do not swallow it silently.

### 4. Inject failures on purpose

Modify two of your tools to simulate failures, gated by an env var or a flag so you can keep the exercise-02 happy path working:

- **`search_products`**: when `FLAKE_SEARCH=1`, fail on the first call of every conversation with a `RetriableError("upstream unavailable")` and succeed afterward.
- **`create_refund`**: when called for an order whose status is `"cancelled"`, return `{"error": "not_eligible", "detail": "cannot refund a cancelled order"}` as **data** (not raised). When called for an order whose status is `"delivered"`, succeed normally.

These cover three of the four failure buckets:

- Business outcome → `not_eligible` (data).
- Transient → simulated flaky search (raised, converted to `{"error": "transient", ...}`).
- Internal → a real Python bug; you do not need to inject one, but if your tool ever raises an unexpected `KeyError` during the suite, that fourth bucket is exercised for free.

### 5. The adversarial prompt suite

Run the following prompts with the hardened loop. Capture per-run rows into `results.jsonl`:

```
{"prompt": "...", "tool_calls": [{"name": "...", "args": {...}, "is_error": bool, "category": "..."}, ...], "final_text": "...", "budget_exhausted": bool}
```

Prompts:

1. **Malformed args probe**: "Refund order 1004 for fifty bucks because it's damaged." *(no `ORD-` prefix, missing idempotency key, `amount_cents` not provided as integer — the model should produce one or more invalid-arg failures and recover.)*
2. **Repair loop probe**: "Refund order ORD-2004 for $5 because it's damaged." *(unknown order ID; the underlying refund function should treat that as `not_eligible`. Watch whether the model retries or gives up.)*
3. **Transient failure**: with `FLAKE_SEARCH=1`, ask "Is the wireless headset in stock?" The first `search_products` call should fail; the model should retry and succeed on the second attempt.
4. **Budget exhaustion**: "Search for a blue shirt. If you don't find one, try a red shirt. If you don't find that either, try a green shirt. Then a yellow shirt." *(this is a contrived prompt to force the search budget to exhaust. With `max_per_tool=3`, the fourth search should raise `BudgetExceeded`.)*
5. **Cancellation guard**: "Cancel ORD-1001, then refund it." *(ORD-1001 is `shipped` — cancel should return `not_cancellable`; refund should still be processable if your `create_refund` allows it. Note how the model sequences this.)*
6. **Polite no-op**: "Hi, just wanted to check in. How are you doing today?" *(no tool should be called; the budget should stay at zero.)*

### 6. Inspect the audit log

Print or write to `audit.jsonl` one entry per tool call:

```
{"prompt_index": int, "step": int, "name": "...", "args": {...}, "is_error": bool, "error_category": "..." | null, "elapsed_ms": int}
```

In `NOTES.md`, summarize per tool: total calls, error rate, breakdown by `error_category`. This is the dataset chapter 6 promised would tell you which descriptions or schemas need work.

## Things you should not do in this exercise

- Catch `ValidationError` inside individual tool functions. Validation lives at the dispatcher.
- Swallow `BudgetExceeded` to let the conversation continue. The whole point of the budget is to fail loudly.
- Silently coerce types. If the model sends `"amount_cents": "5000"` (a string), let Pydantic complain; the error feedback is the lesson.
- Stuff stack traces or raw exception messages into the `detail` field. Keep error payloads model-readable and free of internals.

## Acceptance criteria

You can demonstrate:

- Every tool has a Pydantic args model registered with the registry. Calling the dispatcher with invalid args returns a structured error rather than raising out.
- `safe_dispatch` distinguishes at least three error categories (`invalid_arguments`, `transient`, `internal`) and includes a business-rule failure (`not_eligible` or similar) returned as data.
- `ToolBudget` enforces per-tool and total caps. Prompt 4 reliably triggers `BudgetExceeded` with `max_per_tool=3`, and the conversation surfaces a clean final message instead of crashing.
- A `results.jsonl` with rows for all six adversarial prompts, showing per-call `is_error` and `category` where applicable.
- An `audit.jsonl` plus a paragraph in `NOTES.md` summarizing error rates per tool.
- A `NOTES.md` paragraph for at least two prompts where the model *recovered* from a tool error — describe the failure payload the model saw and what it did with it.

## Reflection

Answer briefly in `NOTES.md`:

1. For prompt 1 (malformed args), how many self-corrections did the model attempt before producing a clean call? Did it ever loop on the same error? If so, what would you change in the error payload to stop that?
2. Did the model ever produce a final reply that *claimed* a tool succeeded when it had actually returned an error? If yes, that is a critical bug — what change to the loop, prompt, or error payload prevents it?
3. The Anthropic protocol has a dedicated `is_error: true` field on tool results; OpenAI does not. Did you notice any behavioral difference between providers when the same error payload was sent? (If you only ran one provider, hypothesize.)

## Stretch goals

- Add `pytest` tests for `safe_dispatch` that cover all three error categories without involving the model: pass deliberately bad args, raise a transient inside a mock tool, and raise an unexpected exception. Assert on the returned `is_error` and payload shape.
- Implement a small *summary* tool the model can call to produce a final reply with a structured form (`{"action_taken": ..., "follow_up": ...}`) instead of free text. Use `tool_choice` to force it on the last turn after a side effect. This previews the "wrap-up turn" pattern used in `mod-105`.
- Add a *partial-failure* test: in a single assistant turn with multiple parallel tool calls, make one succeed and one fail. Confirm your loop returns both `tool_result`s in the same follow-up turn, with `is_error` set correctly on each. This is the bug most loops have until you write the test for it.
