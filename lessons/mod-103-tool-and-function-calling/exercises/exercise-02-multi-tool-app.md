# exercise-02: Multi-Tool App

**Estimated effort:** 3 hours

## Objective

Extend the single-tool loop from exercise-01 into a multi-tool app. You will register several tools behind a small dispatcher, watch the model choose between them, exercise `tool_choice` to force and forbid calls, and see how parallel tool use changes the loop. By the end you should be able to read a model's behavior — which tool it picked, when it called several in one turn, when it gave up and answered in prose — and trace it back to a specific tool description or prompt choice.

## Background

This exercise covers material from:

- [Chapter 2 — Anatomy of a Tool Definition](../02-anatomy-of-a-tool-definition.md)
- [Chapter 3 — The Call/Result Loop](../03-call-and-result-loop.md)
- [Chapter 4 — Wiring Python Functions as Tools](../04-wiring-python-functions.md)
- [Chapter 5 — When to Call a Tool vs. Generate](../05-when-to-call-a-tool.md)

## Prerequisites

- Completed exercise-01. Reuse its `get_order_status` tool and the loop skeleton.
- Same SDK and model as exercise-01. Pin `temperature=0`.
- A small spend cap. This exercise should still cost under a dollar.

## The task

Same e-commerce support assistant, now with a wider surface. You will add three tools — a catalog search, a refund creator, and an order canceller — plus a small tool registry that owns the schemas, the functions, and the dispatcher in one place.

You will also probe how the model decides between the tools, and how `tool_choice` lets you override that decision when the application's contract calls for it.

### The tool surface

Implement the following four tools. Use the patterns from chapter 2 for naming, descriptions, and schemas. Keep descriptions tight enough that a careful new engineer could pick the right tool for a given user request.

1. **`get_order_status(order_id)`** — same as exercise-01.

2. **`search_products(query, limit=10)`** — search a small in-memory product table by free text. Use a tiny fake catalog (10–20 items is plenty):

   ```python
   PRODUCTS = [
       {"sku": "BL-MED-001", "name": "Blue T-Shirt, Medium", "in_stock": True,  "price_cents": 1999},
       {"sku": "BL-LG-001",  "name": "Blue T-Shirt, Large",  "in_stock": False, "price_cents": 1999},
       {"sku": "RD-MED-001", "name": "Red T-Shirt, Medium",  "in_stock": True,  "price_cents": 1999},
       {"sku": "HS-001",     "name": "Wireless Headset",     "in_stock": True,  "price_cents": 6999},
       # ... a few more, including at least one item with multiple sizes/colors
   ]
   ```

   Implementation can be naive substring matching against `name`. Return at most `limit` items.

3. **`create_refund(order_id, amount_cents, reason, idempotency_key)`** — write a row into a `REFUNDS = {}` dict keyed by `idempotency_key`. If the key already exists, return the existing refund (idempotent). Otherwise record `{"refund_id": "rf_<uuid4>", "order_id": ..., "amount_cents": ..., "reason": ...}` and return it. `reason` is one of `damaged | wrong_item | not_received | other`.

4. **`cancel_order(order_id)`** — flip the order's status to `"cancelled"` *only if* its current status is `"processing"`. Otherwise return `{"error": "not_cancellable", "current_status": "<status>"}`. (This previews chapter 6's "expected outcome as data" pattern; you do not need to add general error handling yet.)

### The registry

Build a small `ToolRegistry` along the lines of chapter 4:

- `register(definition, fn)`
- `definitions()` — list passed as `tools=[...]` on every call.
- `dispatch(name, args)` — look up the callable and invoke it with `**args`. Raise `UnknownToolError` if the name is unknown.

The loop from exercise-01 should now call `registry.dispatch(block.name, block.input)` instead of hard-coding the function. Replace the `assert block.name == ...` with that lookup; the assert was the placeholder that "tomorrow there will be more tools."

## Tasks

### 1. Wire the four tools through the registry

Refactor exercise-01 to use the registry. Confirm with one quick smoke prompt that `get_order_status` still works through the new dispatcher. Do not change any behavior beyond the indirection.

### 2. Drive the model through a prompt suite

Run each of the following prompts through your loop and capture `results.jsonl` rows of the form:

```
{"prompt": "...", "tool_calls": [{"name": "...", "args": {...}}, ...], "final_text": "..."}
```

Suite:

1. "Is the blue t-shirt in medium in stock?"
2. "I'd like to cancel order ORD-1003."
3. "Where is ORD-1001, and do you have any wireless headsets?"   *(two intents in one prompt)*
4. "I need a refund on ORD-1002. It arrived broken."
5. "Do you do refunds in general?"   *(a meta question — should NOT call a tool)*
6. "Cancel ORD-1001."   *(ORD-1001 already shipped — should hit `not_cancellable`)*
7. "What's the SKU for the red shirt?"

For each, note:

- Which tools the model called (names + args).
- Whether the call was correct, redundant, or wrong.
- For prompt 3, did the model issue both tool calls in parallel in a single assistant turn, or did it serialize them across two round trips? Your loop must handle both.

### 3. Exercise `tool_choice`

Pick **one** prompt from your suite — a good candidate is prompt 4, the refund request — and run it three times:

1. `tool_choice="auto"` (the default).
2. `tool_choice={"type": "tool", "name": "create_refund"}` (Anthropic) or `{"type": "function", "function": {"name": "create_refund"}}` (OpenAI) — force calling refund.
3. `tool_choice={"type": "none"}` — forbid tool calls.

Capture, per run:

- Did the model produce a sensible final reply?
- For the forced run, did the model invent any arguments (especially `amount_cents` and `idempotency_key`) without enough information?
- For the forbidden run, did the model give a useful answer, refuse to act, or hallucinate ("I've issued the refund")?

This is the data behind chapter 5's claim that `tool_choice` is not a substitute for a good description.

### 4. Tighten one description

Pick one tool whose behavior in the suite was off — too eager, missed cases, or got confused with another tool. Edit only the **description** (not the schema, not the prompt) and re-run the affected prompts. Note what changed.

This is the smallest unit of iteration in tool-using apps; it is worth practicing.

### 5. Audit parallel tool use

If your provider/model supports parallel tool calls (most current Anthropic and OpenAI models do), look at prompt 3's run. Capture how many `tool_use` blocks were in the assistant turn. If you see one block, run it again with parallel tool use explicitly enabled. If you see two, run it once with parallel tool use disabled (`disable_parallel_tool_use=True` on Anthropic, `parallel_tool_calls=False` on OpenAI) and observe the difference in step count and latency.

Note which version produced a clearer final reply.

## Starter guidance

A registry sketch that fits the chapter-4 shape:

```python
from typing import Callable, Any

class UnknownToolError(Exception): pass

class ToolRegistry:
    def __init__(self):
        self._tools: list[dict] = []
        self._fns: dict[str, Callable[..., Any]] = {}

    def register(self, definition: dict, fn: Callable[..., Any]) -> None:
        name = definition["name"]
        if name in self._fns:
            raise ValueError(f"tool {name!r} already registered")
        self._tools.append(definition)
        self._fns[name] = fn

    def definitions(self) -> list[dict]:
        return list(self._tools)

    def dispatch(self, name: str, args: dict) -> Any:
        try:
            return self._fns[name](**args)
        except KeyError:
            raise UnknownToolError(name)

registry = ToolRegistry()
registry.register(GET_ORDER_STATUS_TOOL, get_order_status)
registry.register(SEARCH_PRODUCTS_TOOL, search_products)
registry.register(CREATE_REFUND_TOOL, create_refund)
registry.register(CANCEL_ORDER_TOOL, cancel_order)
```

Wire it into the loop from exercise-01 — the only line that changes is `result = registry.dispatch(block.name, block.input)` in place of the hard-coded function call.

Do not over-engineer the registry. It is a list and two dicts.

## Things you should not do in this exercise

- Validate tool arguments with Pydantic, retry on errors, or implement an error budget. Exercise-03 is for that.
- Add a fifth tool to "cover" a confusing case. Fix the description instead.
- Increase `max_steps` past 10. If you need more, you have a tool-design problem.
- Disable parallel tool use globally just because it makes the loop simpler. Note when it changes behavior and pick deliberately.

## Acceptance criteria

You can demonstrate:

- Four tools registered through a single registry with a single dispatcher. Tools are independently testable as plain Python functions (write one or two `pytest` tests for `cancel_order` covering both the success and `not_cancellable` paths).
- A `results.jsonl` with rows for all seven prompts of the suite, including the tool calls (names + args) and the final text.
- A `results_tool_choice.jsonl` (or appended rows) with the three `tool_choice` runs for prompt 4.
- A `NOTES.md` containing:
  - Your before/after for the description you tightened in task 4, with the affected prompts re-run.
  - At least one observation about parallel tool use from task 5.
  - The pinned `MODEL`, `temperature`, and `max_tokens` for the runs.
- A trace for prompt 3 showing whether tools were called in parallel or serially.

## Reflection

Answer briefly in `NOTES.md`:

1. Which tool was the model most likely to over-call (use when it should have generated)? Which was it most likely to under-call (miss when it should have called)? What does that tell you about the *descriptions*?
2. For the `tool_choice="none"` run on the refund prompt, did the model behave honestly (refuse / explain) or dishonestly (claim to act)? What does that say about where you should put guard rails — in the prompt, in the application code, or both?
3. Pick one prompt where the model produced something you would not ship to a user. What is the smallest change (description, schema, prompt, code) that fixes it?

## Stretch goals

- Add a fifth tool, `lookup_order_for_customer(customer_email)`, that returns the most recent order for a customer. Watch how the model now resolves "Where is *my* last order?" (a phrasing it could not handle before). Note any new failure modes.
- Implement a small *audit log*: every tool call captured as `{ts, name, args, result_summary}` written to `audit.jsonl`. Then write a small script that reads the log and prints, per tool, the rate of "errored or returned not_found." This is the data that drives chapter 6 in exercise-03.
- Add a system prompt that names the assistant ("Acme Support Bot") and forbids issuing a refund without first confirming with the user. Re-run prompt 4 (`auto`) — did the model now ask for confirmation before calling `create_refund`? Compare against the run without the guard rail.
