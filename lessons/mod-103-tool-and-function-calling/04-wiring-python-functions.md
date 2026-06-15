# Wiring Python Functions as Tools

## Why this matters

A tool definition advertises a capability. A Python function provides it. The bridge between them — schema goes to the model, callable lives in your code, dispatcher routes calls — is where most "it works in the demo, breaks in production" bugs are born. Get this layer tidy and adding the seventh tool is as cheap as adding the first.

This chapter is the engineering side: how to keep the JSON Schema and the Python signature from drifting apart, how to route an incoming tool call to the right function, and how to make the whole surface easy to test without the model in the loop.

## The two-sided contract

Every tool has two declarations of the same thing:

```
JSON Schema (sent to the model)         Python signature (called by your code)
{                                       def get_order_status(order_id: str) -> dict:
  "name": "get_order_status",               ...
  "input_schema": {
    "type": "object",
    "required": ["order_id"],
    "properties": {"order_id": {"type": "string"}}
  }
}
```

The two sides must agree on names, types, and which arguments are required. When they drift, you get one of three failure modes:

- Schema says `customer_id` but the function takes `customer`. `TypeError: unexpected keyword argument`.
- Schema says `amount` is a string; function expects an int. Either an exception, or — worse — a silent truncation when Python coerces.
- Function adds a new required argument; schema still says it is optional. Model omits it; runtime crashes.

You want exactly one source of truth for each tool's contract.

## Pattern 1: hand-written schema next to the function

The simplest pattern is to keep the schema and the function physically next to each other, with a clear naming convention:

```python
def get_order_status(order_id: str) -> dict:
    """Look up the current status of a customer order."""
    return db.orders.get_status(order_id)

GET_ORDER_STATUS_TOOL = {
    "name": "get_order_status",
    "description": "Look up the current status of a customer order by ID. "
                   "Use when the user asks where their order is.",
    "input_schema": {
        "type": "object",
        "required": ["order_id"],
        "additionalProperties": False,
        "properties": {
            "order_id": {"type": "string", "pattern": "^ORD-[0-9]{4,8}$"},
        },
    },
}
```

Pros: explicit, easy to read, no magic. The schema is exactly what the model sees.

Cons: you maintain the schema by hand. Type signatures and schema can drift; add a `field_validator` or a unit test to catch it.

This is the right pattern when you have 3–7 tools and a stable API surface.

## Pattern 2: derive the schema from a typed model

Once you have more than a handful of tools, hand-maintaining schemas grows old. Use Pydantic (Python) or zod (TypeScript) as the single source of truth for both the argument *type* and the schema:

```python
from pydantic import BaseModel, Field

class GetOrderStatusArgs(BaseModel):
    order_id: str = Field(pattern=r"^ORD-[0-9]{4,8}$", description="Order ID, e.g., 'ORD-4421'.")

    model_config = {"extra": "forbid"}    # equivalent to additionalProperties: false

def get_order_status(args: GetOrderStatusArgs) -> dict:
    return db.orders.get_status(args.order_id)

GET_ORDER_STATUS_TOOL = {
    "name": "get_order_status",
    "description": "Look up the current status of a customer order by ID.",
    "input_schema": GetOrderStatusArgs.model_json_schema(),
}
```

Pros: one type, two uses. Schema and signature cannot drift because they are *the same definition*. You can also call `GetOrderStatusArgs.model_validate(payload)` at the dispatch layer to validate the model's arguments before invoking the function (chapter 6 leans on this).

Cons: the schema Pydantic emits often needs small post-processing for strict provider modes (drop `title` fields, ensure `additionalProperties: false`, flatten `$defs` for nested models). Add a small `pydantic_to_tool_schema(...)` helper if you adopt this approach.

This is the pattern most production codebases settle on. The exercises in this module are small enough that you can use either pattern; pick by taste.

## Pattern 3: decorator-style registration

A common ergonomic shape: a decorator that registers a Python function as a tool, derives the schema from its signature/docstring, and adds it to a registry:

```python
@tool(name="get_order_status",
      description="Look up the current status of a customer order by ID.")
def get_order_status(order_id: str = Field(pattern=r"^ORD-[0-9]{4,8}$")) -> dict:
    return db.orders.get_status(order_id)
```

Pros: minimum boilerplate, one place to add a tool.

Cons: the magic costs you legibility — you must know what your `@tool` decorator infers and how it converts type hints into JSON Schema. When something goes wrong you debug the helper, not your tool.

If you reach for this, write or audit the decorator yourself. Do not import "magic tool registration" from a library whose internals you have not read; the model output debug session you will run is much harder when an opaque layer sits between the function and the schema. Libraries like Pydantic AI, LangChain's `@tool`, and Anthropic's "function calling" examples implement this pattern — useful references whether you adopt or roll your own.

## A small tool registry

Whichever pattern you pick, every tool-using app needs the same three things at runtime:

1. The list of tool *definitions* sent to the model.
2. A way to look up the Python *callable* for a given tool name.
3. A *dispatcher* that takes a name + arguments and returns a result.

A minimal registry that covers all three:

```python
from typing import Callable, Any

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
            fn = self._fns[name]
        except KeyError:
            raise UnknownToolError(name)
        return fn(**args)
```

Usage:

```python
registry = ToolRegistry()
registry.register(GET_ORDER_STATUS_TOOL, get_order_status)
registry.register(SEARCH_PRODUCTS_TOOL, search_products)
registry.register(CREATE_REFUND_TOOL, create_refund)

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=registry.definitions(),
    messages=messages,
)

# ... when a tool_use block arrives:
result = registry.dispatch(block.name, block.input)
```

The registry is small on purpose. You do not need a tool framework to do this work; the loop from chapter 3 + this registry is enough for any app this module produces.

## Argument handling: `**kwargs` vs. positional

`fn(**args)` is convenient but it relies on Python keyword-matching. Two pitfalls:

- **Extra keys raise `TypeError`.** If the model invents a parameter — perhaps you removed one from the schema but a cached tool turn still references it — the dispatch crashes. With `additionalProperties: false` in the schema this should not happen, but defense in depth helps; chapter 6 shows how to validate-then-call instead of call-then-pray.
- **Type coercion is the model's job, not yours.** The schema promises a type. If the model returns `"order_id": 4421` (an int) for a `string` field, you might want to coerce silently — or you might want to crash and fix the schema. Pick a policy. Most production code chooses to validate strictly via a Pydantic model and fail loudly on mismatch.

A defensible dispatcher uses a Pydantic args model when one exists:

```python
def dispatch(self, name: str, args: dict) -> Any:
    fn = self._fns[name]
    schema_model = self._arg_models.get(name)
    if schema_model is not None:
        parsed = schema_model.model_validate(args)   # raises ValidationError on bad input
        return fn(parsed)                            # function takes the typed model
    return fn(**args)
```

This is exactly the "boundary" pattern from `mod-102` chapter 6, applied to tool arguments. Treat the model's `input` as untrusted input; validate it; then run your business logic against a typed value.

## What the function should return

A few rules that make the model's job easier downstream:

- **Return JSON-serializable data.** Dicts, lists, strings, numbers, bools, `None`. If your code wants to return a `datetime`, convert to ISO 8601 first.
- **Keep it short.** The result becomes prompt context on the next turn. Returning a 50-row database dump for a "did this order ship?" question wastes tokens and slows the model. Filter to the fields the model needs.
- **Use stable field names.** Renaming `tracking_no` to `tracking_number` between two calls confuses the model's continuation. Decide once.
- **Encode failure as data, not exceptions.** Chapter 6 makes this explicit, but the rule starts here: a not-found order should return `{"status": "not_found"}`, not raise. Exceptions are for *your* bugs; outcomes are for the model.

A small helper to enforce these defaults:

```python
import json
from datetime import datetime, date

def jsonable(value):
    if isinstance(value, (datetime, date)):
        return value.isoformat()
    if hasattr(value, "model_dump"):
        return value.model_dump()
    return value

def serialize_result(payload) -> str:
    return json.dumps(payload, default=jsonable)
```

Use this when building the `tool_result` content blocks in the loop.

## Side effects, idempotency, and safety

Tool calls can mutate the world. Three habits keep that survivable:

- **Mark which tools have side effects.** A boolean in your registry, or a naming convention (`get_*` reads, `create_*` / `cancel_*` / `send_*` writes). Use it in tests and audits.
- **Make every write idempotent.** Take an `idempotency_key` parameter on writes and treat repeats as no-ops. Models will occasionally call the same tool twice in a single turn, or re-issue after a network blip; idempotency makes that recoverable.
- **Authorize at the function, not the schema.** A tool's schema says the model *may* request a refund; your function decides whether *this* user is allowed to receive one. Authorization rules belong in code, not in prompt text.

Side effects are also where "tool" stops looking like "function" and starts looking like "endpoint." Treat each side-effecting tool the way you would treat an unauthenticated HTTP endpoint: explicit, narrow, idempotent, logged.

## Testing tools without the model

A side benefit of the registry pattern: every tool is a plain Python function you can unit-test directly. Test the function, then test the dispatcher, then add an integration test that exercises the loop with a recorded or mocked model. Do not skip the unit tests — bugs in `get_order_status` look exactly like model failures from outside.

A small smoke test:

```python
def test_get_order_status_known_order():
    assert get_order_status(order_id="ORD-4421") == {
        "order_id": "ORD-4421",
        "status": "shipped",
        "carrier": "UPS",
    }

def test_dispatch_unknown_tool_raises():
    registry = ToolRegistry()
    with pytest.raises(UnknownToolError):
        registry.dispatch("does_not_exist", {})
```

For the model layer, recording a real conversation and replaying it (e.g., with VCR or a fixture file of recorded JSON responses) is more useful than a unit test that mocks the API client. The interesting bugs live in the loop, not in the SDK.

## Common mistakes

- **Defining the schema once and the function arguments differently, then chasing the bug.** Use one source of truth (Pattern 2) or a unit test that asserts the JSON Schema agrees with the function signature.
- **`fn(**args)` with no validation.** Works until the model invents a key.
- **Raising for "the order doesn't exist."** That is a normal outcome, not a bug. Return data.
- **Returning a giant payload.** The model will read it on every subsequent turn. Trim before returning.
- **One tool dispatcher with five hard-coded `if name == "..."` branches.** Fine for two tools; painful at six. Move to a registry early.

## Summary

A tool definition advertises a callable; a Python function implements it; a registry routes calls between them. Pick a pattern — hand-written schema next to the function, or a typed args model that produces the schema — and stick to it. Validate arguments at the boundary, return JSON-serializable data the model can keep reading, and make every side-effecting call idempotent. With that layer in place, the call/result loop from chapter 3 plugs straight in and adding tools becomes routine.
