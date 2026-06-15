# From Generation to Action

## Why this matters

So far in this track the model has been a function over tokens: text in, text out. That is enough to summarize, classify, extract, and translate — but it is not enough to *do* anything. The model cannot read your database, hit a price API, send an email, or check whether an order shipped. It can guess, and on the bad day it will guess confidently and wrongly.

Tool calling closes that gap. You hand the model a menu of functions you have implemented in your application, the model decides when and how to call them, and your code runs the function and feeds the result back. The model's job stays the same — pick the next token — but its inputs now include real, fresh data from your system.

This is also the protocol that the rest of this track is built on. `mod-104` uses it for retrieval, `mod-105` turns it into an agent loop, and `mod-106` ships it. Get the mental model right here.

## What "tool calling" actually is

A "tool call" is a structured request the model emits **instead of** a final user-facing reply. Concretely:

- You include a `tools` (or `functions`) parameter in your API call. Each tool has a name, a description, and a JSON Schema for its arguments.
- The model can respond with a normal text turn ("Here is your answer…") **or** with a `tool_use` block that names a tool and supplies arguments matching the schema.
- Your code reads the tool name and arguments, runs the corresponding Python function, and sends the result back to the model in the next turn as a `tool_result`.
- The model uses that result to continue — usually to either call another tool or produce the final reply.

The model never executes anything itself. It produces a structured intent — "call `get_order_status` with `order_id="4421"`" — and **your application** decides whether and how to honor it. That separation is the whole safety story.

If you have already used schema-constrained tool output from `mod-102` chapter 5 as a structured-output transport, the wire format is the same. The difference is that here you actually run code and feed the result back. The protocol is bidirectional.

## A concrete shape

In the Anthropic Messages API, a tool definition looks like this:

```python
get_order_status_tool = {
    "name": "get_order_status",
    "description": "Look up the current status of a customer order by order ID.",
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

You pass it on every turn:

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[get_order_status_tool],
    messages=[{"role": "user", "content": "Where is my order ORD-4421?"}],
)
```

A normal reply comes back as a `text` content block. A tool call comes back as a `tool_use` content block:

```python
# response.content might look like:
# [
#   ToolUseBlock(
#       type="tool_use",
#       id="toolu_abc123",
#       name="get_order_status",
#       input={"order_id": "ORD-4421"},
#   )
# ]
# and response.stop_reason == "tool_use"
```

The model has effectively said: "Before I can answer this, I need you to call `get_order_status('ORD-4421')` and tell me what you got." You do, and the conversation continues. The full loop is chapter 3; the point of this chapter is the *shape* of the interaction.

OpenAI's API uses different field names (`tools` with `"type": "function"`, `tool_calls` on the assistant message, `arguments` as a JSON string instead of a parsed object) but the protocol is identical: structured request → host executes → structured result → continuation.

## Why this is better than "just prompt for it"

The first instinct is usually wrong: "Why not just ask the model for the order status in the prompt?" Three reasons.

**Freshness.** The model's training data has a cutoff. It does not know the status of order `ORD-4421` today. Even if it claims to, that claim is a hallucination dressed up as a fact. A tool call routes through your database, where the answer is authoritative.

**Action.** Some tools have side effects — issue a refund, send an email, write a row. A "please refund order 4421" prompt cannot do that. A `refund_order` tool can, and you control exactly which arguments and which authorizations are required.

**Auditability.** Every tool invocation is a structured record you can log, replay, and gate. "The model said the refund was approved" is a vibe. "The model called `refund_order(order_id='4221', amount_cents=4900)` and the function returned `{ok: True, refund_id: 'rf_…'}`" is an audit trail.

Tool calling is how you keep the model's flexibility (it picks what to do) without its weaknesses (it hallucinates facts and cannot reach your systems).

## The mental model: typed function the model can call

Treat each tool as a small, typed function with three properties:

1. A **name** that describes its effect in your domain (`get_order_status`, not `fn1`).
2. A **schema** that constrains the arguments — the same JSON Schema discipline from `mod-102` chapter 5.
3. A **description** that tells the model *when to use it*. The description is part of the prompt; it is read on every turn.

The model's job is to pick the right tool and produce arguments that fit the schema. Your job is to execute the tool safely and return a result the model can read. Chapter 4 covers wiring the Python side; chapter 2 covers writing the schema; this chapter sets the mental model.

A useful comparison:

| Concern              | Plain generation             | Tool calling                               |
|----------------------|------------------------------|--------------------------------------------|
| Model produces       | text                         | text **or** a structured call              |
| You execute          | nothing                      | your code, on the model's behalf           |
| Source of facts      | model's training + prompt    | your system, fetched live                  |
| Side effects         | none                         | exactly the ones you implement and allow   |
| Auditable            | "what did it say?"           | "which tools did it call with what args?"  |
| Schema discipline    | optional (chapter 5 of 102)  | mandatory                                  |

## What this module covers

The remaining chapters work through the full loop:

- **Chapter 2** — anatomy of a tool definition. Name, description, JSON Schema for parameters, what good tool design looks like.
- **Chapter 3** — the call/result loop. The multi-turn protocol with real code on both providers, including how to spot a tool call, how to run it, and how to send the result back.
- **Chapter 4** — wiring Python functions as tools. Bridging the schema to actual callables, a small tool registry, and dispatch.
- **Chapter 5** — when to call a tool vs. generate. The decision criteria, `tool_choice`, multi-tool selection, and the cost/latency tradeoffs.
- **Chapter 6** — handling tool errors and malformed arguments. Defensive execution, error-as-feedback, retry budgets, the safety story.

By the end you should be able to take a vague task ("answer support questions using our order system") and ship a small, well-wired tool-calling app whose behavior you can predict, audit, and defend.

## Summary

Tool calling lets the model trigger your code instead of guessing. The model emits a structured intent ("call `get_order_status(order_id='ORD-4421')`") and your application decides whether to honor it. That shift turns the model from a text generator into the planner of a small, auditable program. Everything that follows in this module — schemas, loops, error handling — is mechanics around that one idea.
