# Anatomy of a Tool Definition

## Why this matters

A tool definition is a prompt. The model reads the tool name, the description, and the JSON Schema for the parameters on every turn, and uses them to decide whether to call the tool and what to pass. A vague description gets called at random times with random arguments; a sharp one gets called exactly when it should. Tool design *is* prompt design — applied to a small, structured surface.

This chapter is about that surface: what every tool definition has, what each field is for, and what separates a tool the model uses well from one it mishandles.

## The three required pieces

Every tool definition needs three things:

1. **A name.** A short identifier the model emits when it decides to call this tool. Naming a tool `do_thing` is like naming a function `do_thing` in your codebase: the model has nothing to go on.
2. **A description.** A short natural-language explanation of what the tool does and when to use it. This is the part the model actually reads to choose.
3. **An input schema.** A JSON Schema describing the arguments. The model is constrained to produce arguments that fit this schema.

On Anthropic the tool definition looks like:

```python
{
    "name": "get_order_status",
    "description": "Look up the current status of a customer order by order ID. "
                   "Use when the user asks where their order is or whether it has shipped.",
    "input_schema": {
        "type": "object",
        "required": ["order_id"],
        "properties": {
            "order_id": {
                "type": "string",
                "description": "Order identifier, e.g., 'ORD-4421'.",
            },
        },
    },
}
```

OpenAI's shape wraps the same content slightly differently:

```python
{
    "type": "function",
    "function": {
        "name": "get_order_status",
        "description": "Look up the current status of a customer order by order ID. "
                       "Use when the user asks where their order is or whether it has shipped.",
        "parameters": {                      # JSON Schema, same dialect
            "type": "object",
            "required": ["order_id"],
            "properties": {
                "order_id": {"type": "string", "description": "Order identifier, e.g., 'ORD-4421'."},
            },
        },
    },
}
```

The key field names differ — `input_schema` vs. `parameters`, the `"type": "function"` wrapper — but the model sees the same information either way.

## Naming a tool

Two rules carry most of the work.

**Verb-noun, in your domain.** `get_order_status`, `create_refund`, `search_products`, `send_password_reset_email`. Avoid generic verbs (`do`, `handle`, `process`) and avoid jargon that only your team uses (`record_lwm`, `flip_qc`).

**Distinct enough that two tools cannot mean the same thing.** If you have `find_order` and `lookup_order`, the model will pick one at random and the other will sit unused. Name pairs that point at different intents (`get_order_status` vs. `cancel_order`), not synonyms.

Stick to `snake_case` with letters, digits, and underscores. Most providers enforce a regex on names (broadly `^[A-Za-z0-9_-]+$`) and a length cap (commonly 64 characters). Save yourself the gotcha and keep names short.

## Writing the description

The description is where the model decides whether to call this tool *at all*. Three things to put in it:

1. **What the tool does**, in one sentence. ("Look up the current status of a customer order by order ID.")
2. **When to use it**, in one short clause. ("Use when the user asks where their order is or whether it has shipped.")
3. **When *not* to use it**, if there is an obvious confusable. ("Do not use this to cancel an order — use `cancel_order` for that.")

A good test: read all your tool descriptions back-to-back. Could a careful new engineer pick which tool to call for a given user request? If not, the model cannot either.

Two common mistakes:

- **Describing the implementation instead of the behavior.** "Hits the `/v2/orders/{id}` endpoint" is internal trivia the model does not need. "Returns the current status of a customer order" is what the model is choosing between.
- **Forgetting failure modes.** If a tool returns `not_found` for unknown IDs, say so. The model will see the result anyway, but the description sets expectations and reduces panic-rerouting.

Descriptions also belong in the **per-field** schema, not only at the top level. Annotate every parameter that is not obvious from its name.

## Designing the input schema

The input schema is just JSON Schema — the same dialect from `mod-102` chapter 5 — applied to function arguments. Most provider strict modes implement a subset; read your provider's docs for which keywords are supported.

A reasonable set to lean on:

- `type` — `"string"`, `"integer"`, `"number"`, `"boolean"`, `"array"`, `"object"`.
- `enum` — closed sets. Use this aggressively for categories and modes.
- `description` — what the field means. Read by the model.
- `required` — list of property names that must be present.
- `minimum` / `maximum`, `minLength` / `maxLength`, `pattern` — value-level constraints.
- `items` — element schema for arrays.

Apply the same schema-design guidance from `mod-102` chapter 5:

```python
"input_schema": {
    "type": "object",
    "required": ["order_id"],
    "additionalProperties": False,        # forbid the model from inventing fields
    "properties": {
        "order_id": {
            "type": "string",
            "pattern": "^ORD-[0-9]{4,8}$",
            "description": "Order identifier in the form 'ORD-1234'.",
        },
        "include_history": {
            "type": "boolean",
            "description": "If true, include the full state-transition history; default false.",
        },
    },
},
```

Two design moves that pay off:

- **Constrain values, not just types.** A `status` field that is `{"type": "string"}` will be called with `"PENDING"`, `"pending"`, `"in-transit"`, and `"on its way"` by different invocations. `{"type": "string", "enum": ["pending", "shipped", "delivered", "cancelled"]}` will not.
- **Add per-field descriptions for anything not self-evident.** `amount_cents` is fine on its own; `mode` will get random values without an explanation.

### Optional vs. required

If a parameter is optional with a default, mark it optional (omit from `required`) and document the default in the description. Do *not* mark it required and tell the model "leave blank if you don't want one" — the model will produce empty strings or zeros, which downstream code will misread. Required fields should be genuinely required; optional fields should genuinely have a default.

### Avoid catch-all `kwargs`

A `params` field of type `object` with `additionalProperties: true` lets the model invent any keys it wants. It will. Make every parameter explicit, even if you end up with 6 properties. Explicit beats clever.

## Designing the return value

The tool's *return value* is also part of the design, even though it does not appear in the tool definition. The model reads tool results as plain text (or a JSON string of arbitrary length). Keep the return tight:

- Return what the model needs to keep going — usually a small JSON object or a short status line. Do not dump the whole row from your database.
- Use stable field names. `{"status": "shipped", "tracking_number": "1Z…", "delivered_at": null}` is much easier for the model to reason about than ad-hoc prose.
- If the tool has *no* useful payload (a fire-and-forget action), return a brief confirmation like `{"ok": true}`. Empty strings confuse downstream turns.

You will set this contract by what your Python function returns; chapter 4 covers the wiring.

## A worked example: a small tool surface

Here is a credible tool surface for a customer-support assistant:

```python
TOOLS = [
    {
        "name": "get_order_status",
        "description": (
            "Look up the current status of a customer order by ID. "
            "Use when the user asks where their order is or when it will arrive."
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
    },
    {
        "name": "search_products",
        "description": (
            "Search the product catalog by free-text query. "
            "Use when the user asks whether a product is available or what options exist. "
            "Do NOT use this to look up the status of an order they already placed."
        ),
        "input_schema": {
            "type": "object",
            "required": ["query"],
            "additionalProperties": False,
            "properties": {
                "query": {
                    "type": "string",
                    "minLength": 1,
                    "description": "Free-text product search terms.",
                },
                "limit": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 25,
                    "description": "Maximum number of results to return; default 10.",
                },
            },
        },
    },
    {
        "name": "create_refund",
        "description": (
            "Issue a refund against an order. Only call this after the user has confirmed "
            "they want a refund and the amount. Idempotent: passing the same idempotency_key twice "
            "returns the original refund."
        ),
        "input_schema": {
            "type": "object",
            "required": ["order_id", "amount_cents", "idempotency_key"],
            "additionalProperties": False,
            "properties": {
                "order_id":        {"type": "string", "pattern": "^ORD-[0-9]{4,8}$"},
                "amount_cents":    {"type": "integer", "minimum": 1, "description": "Refund amount in cents (USD)."},
                "reason":          {"type": "string", "enum": ["damaged", "wrong_item", "not_received", "other"]},
                "idempotency_key": {"type": "string", "description": "Unique key to make this call idempotent."},
            },
        },
    },
]
```

Notice the patterns: each tool has a single clear purpose, each description names a likely user phrasing, value constraints are tight, and `create_refund` warns about the danger of repeated calls in its description.

## Smell tests

A tool definition is in good shape when you can answer yes to all of these:

- **Could I write a docstring for the Python function from just the tool's description?** If not, the description is too thin.
- **For every parameter, do I know what a valid value looks like without reading my own code?** If not, add `enum`, `pattern`, or a clearer `description`.
- **If I see a model invocation in production with these arguments, can I tell whether the model "understood" the user's intent?** Tight schemas make this true; loose schemas hide it.
- **If a junior engineer added a second tool tomorrow, would the existing descriptions still uniquely route to the right one?** If not, you have name or description overlap to fix.

## Common mistakes

- **A description that is the function name in English.** "Gets the order status." Tells the model nothing about *when* to call it.
- **No enums on closed sets.** Categories, status values, modes — these should never be free-form strings.
- **A schema with optional everything.** If nothing is required, the model will sometimes omit the only field that matters.
- **Sneaky "you can pass anything here" parameters.** Open-ended `data` or `metadata` objects are how arbitrary garbage ends up in your function arguments.
- **One tool that does five things based on a `mode` flag.** Split it into five tools. The model picks the right tool more reliably than it picks the right enum value.
- **Names that overlap with provider-reserved words.** `list`, `search`, `query` are safe as `snake_case` names but conflict semantically with provider features in some SDKs. Be specific (`search_products`, not `search`).

## Summary

A tool definition is a name, a description, and a JSON Schema for the arguments. Name tools after their effect; describe them with enough specificity that the model can pick correctly; constrain arguments aggressively with enums, patterns, and required fields; and keep returns tight. Treat tool design as the same engineering discipline as the rest of your prompt. The next chapter wires this definition into a real call and runs the loop.
