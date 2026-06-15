# exercise-01: First Function Calling

**Estimated effort:** 3 hours

## Objective

Build the smallest end-to-end tool-using app: one Python function, one tool definition, and the full call/result loop. By the end you should be able to point at every part of the protocol — the assistant's `tool_use` block, the `tool_result` you sent back, the model's continuation — and explain why it is there.

This exercise establishes the loop you will extend in the next two exercises. Resist any urge to skip ahead to multi-tool or error handling; the goal here is to get the bones right.

## Background

This exercise covers material from:

- [Chapter 1 — From Generation to Action](../01-from-generation-to-action.md)
- [Chapter 2 — Anatomy of a Tool Definition](../02-anatomy-of-a-tool-definition.md)
- [Chapter 3 — The Call/Result Loop](../03-call-and-result-loop.md)
- [Chapter 4 — Wiring Python Functions as Tools](../04-wiring-python-functions.md)

## Prerequisites

- Completed `mod-102` exercise-02 (you have a working call to one provider and have used tool calling as a transport for structured output).
- An API key for one provider (Anthropic *or* OpenAI is fine — both support tool calling and the code patterns are nearly identical).
- Python 3.10+ and the provider SDK installed (`anthropic` and/or `openai`).
- A small spend cap configured. This exercise should cost cents.

## The task

You are building the first step of an e-commerce support assistant. The assistant must be able to look up the status of a customer order on demand.

You will:

1. Implement a single Python function, `get_order_status(order_id)`, against a small fake "database" you ship with the exercise.
2. Define a corresponding tool the model can call.
3. Run the full call/result loop until the model returns a final text answer.
4. Test with at least three inputs that exercise different branches (a normal order, an order in transit, an order ID that does not exist).

Pin your model ID, `temperature=0`, and `max_tokens` consistently across runs.

### The fake order database

Use a small in-memory dict so this exercise has nothing to do with infra:

```python
ORDERS = {
    "ORD-1001": {"status": "shipped",   "carrier": "UPS",   "tracking": "1Z123ABC", "eta": "2025-11-08"},
    "ORD-1002": {"status": "delivered", "carrier": "USPS",  "tracking": "9400111", "delivered_at": "2025-10-31"},
    "ORD-1003": {"status": "processing"},
    "ORD-1004": {"status": "shipped",   "carrier": "FedEx", "tracking": "7849FX",  "eta": "2025-11-12"},
}
```

(Use these IDs or your own; just keep at least one of each broad status.)

### The Python function

```python
def get_order_status(order_id: str) -> dict:
    record = ORDERS.get(order_id)
    if record is None:
        return {"status": "not_found", "order_id": order_id}
    return {"order_id": order_id, **record}
```

This signature is what your tool schema must describe. Note that "not found" is encoded as a normal result, not an exception — this is the pattern from chapter 4 you will be expected to reuse in exercise-03.

## Tasks

### 1. Define the tool

Write the tool definition for your provider. Include:

- A clear `name` in `snake_case`.
- A `description` that names *when* to use the tool, not just what it does.
- An `input_schema` (Anthropic) or `parameters` (OpenAI) that requires `order_id`, constrains it to a string, and ideally uses a `pattern` (`^ORD-[0-9]{4,8}$`) to reject obviously wrong IDs at the schema level.

Hold this definition in a module-level constant; you will reuse it in the next two exercises.

### 2. Build the call/result loop

Write a `run_with_tools(messages, *, max_steps=5)` function that:

1. Calls the model with the tool advertised.
2. Inspects the `stop_reason`:
   - `end_turn` / `stop`: return the assistant's text content.
   - `tool_use` / `tool_calls`: continue.
   - `max_tokens` / `length`: raise an explicit error. Do not "best-effort" past truncation.
3. On a tool-use turn, for every `tool_use` block:
   - Confirms the tool name matches the one you registered. Raise if it does not (this should never happen with one tool; the assertion will catch the bug when you add tools in exercise-02).
   - Calls `get_order_status(**args)`.
   - Builds a `tool_result` (Anthropic) or a `role: "tool"` message (OpenAI) with the function's return value serialized as JSON.
4. Appends the assistant's tool-use turn back into `messages` verbatim, then appends a single user turn (Anthropic) or one tool message per call (OpenAI) carrying the results, then loops.
5. Stops after `max_steps` iterations and raises. This guard is non-negotiable; do not remove it.

This is the exact shape from chapter 3. Keep it under ~60 lines.

### 3. Drive it with three prompts

Run your loop against at least these three prompts:

- **Hit the happy path**: "Hi, can you check on order ORD-1001 for me?"
- **A delivered order**: "Has ORD-1002 arrived yet?"
- **An unknown ID**: "Where is order ORD-9999?"

Capture, per run, into a small `results.jsonl`:

```
{"prompt": "...", "steps": 2, "tool_calls": ["get_order_status"], "final_text": "..."}
```

You will want this file as a baseline when exercise-02 adds tools.

### 4. Observe (don't refactor) something surprising

Write a short note in `NOTES.md` describing one behavior you did not expect. Examples worth recording:

- The model emitted a leading "Let me check that for you" text block *before* the tool call. Was that helpful to the user, or noise?
- The model called the tool with the order ID quoted differently than the user wrote it ("ORD-1001" vs. "ord-1001").
- The model handed you a `tool_use` turn even when the question did not strictly need a lookup.

Do not fix anything yet — exercise-02 and exercise-03 are when you tighten the surface. This step trains the eye.

## Starter guidance

A skeleton, Anthropic flavor, that fits the shape above:

```python
import json
from anthropic import Anthropic

client = Anthropic()
MODEL = "claude-sonnet-4-6"

GET_ORDER_STATUS_TOOL = {
    "name": "get_order_status",
    "description": (
        "Look up the current status of a customer order by ID. "
        "Use when the user asks where their order is, when it will arrive, or whether it shipped."
    ),
    "input_schema": {
        "type": "object",
        "required": ["order_id"],
        "additionalProperties": False,
        "properties": {
            "order_id": {
                "type": "string",
                "pattern": "^ORD-[0-9]{4,8}$",
                "description": "Order identifier in the form 'ORD-1234'.",
            },
        },
    },
}

def get_order_status(order_id: str) -> dict:
    record = ORDERS.get(order_id)
    if record is None:
        return {"status": "not_found", "order_id": order_id}
    return {"order_id": order_id, **record}

def run_with_tools(messages, *, max_steps=5):
    for step in range(max_steps):
        response = client.messages.create(
            model=MODEL,
            max_tokens=1024,
            temperature=0,
            tools=[GET_ORDER_STATUS_TOOL],
            messages=messages,
        )
        if response.stop_reason == "max_tokens":
            raise RuntimeError("hit max_tokens")
        if response.stop_reason != "tool_use":
            return response

        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            assert block.name == "get_order_status", f"unexpected tool: {block.name!r}"
            result = get_order_status(**block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(result),
            })

        messages.append({"role": "user", "content": tool_results})
    raise RuntimeError(f"exceeded max_steps={max_steps}")
```

Reuse the SDK setup and config-loading patterns from earlier modules. Do **not** add Pydantic validation, retry budgets, or multi-tool dispatch yet — those come in the next exercises.

## Things you should not do in this exercise

- Define more than one tool. Exercise-02 will add them.
- Validate `block.input` with Pydantic. Exercise-03 will add it.
- Retry on errors of any kind.
- Suppress, mask, or hide the model's leading text blocks. Look at them.
- Increase `max_steps` to silence a misbehaving loop. If you hit it, that is the bug.

## Acceptance criteria

You can demonstrate:

- A working `run_with_tools` that successfully exits via `end_turn` for all three baseline prompts.
- One commit (or one file) containing the tool definition, the Python function, and the loop. Under ~120 lines total.
- A `results.jsonl` with one entry per baseline prompt showing the number of loop steps and the tool names called.
- A `NOTES.md` with at least one observation from step 4 plus your pinned `MODEL`, `temperature`, and `max_tokens`.
- A trace (printed or logged) of one round trip showing: the `tool_use` block, the `tool_result` you built, and the final text. Paste it into `NOTES.md`.

## Reflection

Answer briefly in `NOTES.md`:

1. What would happen if you forgot to append the assistant `tool_use` turn before the `tool_result` turn? Describe the failure mode you would expect.
2. Why does the protocol put the `tool_result` inside a `user` turn instead of an `assistant` turn?
3. Pick one wording in your `description` field and explain what it would take to delete it. Which phrase, if removed, would most damage the model's ability to choose this tool correctly?

## Stretch goals

- Port the same loop to the other provider (Anthropic → OpenAI or vice versa). Note where the shape differs: `tool_use` vs. `tool_calls`, `input` (parsed) vs. `arguments` (JSON string), `tool_result` in a user message vs. a `role: "tool"` message.
- Add a `--max-steps` CLI flag and a `--trace` flag that prints every loop step. Useful infrastructure for the next exercise.
- Re-run the baseline at `temperature=0.7` three times per prompt. Does the model ever decide not to call the tool? Note any cases where it answered from "memory" instead of fetching.
