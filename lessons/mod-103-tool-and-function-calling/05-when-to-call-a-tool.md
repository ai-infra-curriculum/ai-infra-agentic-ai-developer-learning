# When to Call a Tool vs. Generate

## Why this matters

Once tools exist, the next failure mode is *overuse*. A model with five tools and a fuzzy prompt will route everything through them: it will look up an order to answer "do you support refunds in principle?", it will search products to answer "what's the difference between blue and navy?", it will call `cancel_order` because the user said "cancel" in a casual sentence. Each unnecessary call adds latency, cost, a chance to error, and — for side-effecting tools — a real-world consequence.

The flip side is *underuse*. A model that should have called `get_order_status` instead guesses ("Your order is probably out for delivery!") and gets it wrong.

This chapter is about the decision. Some of it is the model's job (description and `tool_choice` guide it) and some is yours (you decide which tools exist and how to keep their use proportional to the situation).

## When a tool call is the right move

Reach for a tool when at least one of these is true:

- **The answer depends on data the model cannot have.** Order status, inventory levels, prices, the user's own preferences. The model's parameters did not memorize your database.
- **The answer must reflect a moment in time.** "Is this in stock *right now*" is a tool call. "What does 'in stock' mean" is generation.
- **The work has a side effect.** Sending a message, issuing a refund, creating a calendar event. Side effects must be calls, not narration. A model that says "I have refunded your order" without actually calling the tool is a critical bug, not a feature.
- **A deterministic computation is more reliable than the model's guess.** Currency conversion at today's rate, arithmetic on numbers the model is reading from input, date math beyond a few weeks. Tools beat free-form arithmetic.
- **The user explicitly asked for it.** "Look up my account balance." "Find me three blue shirts under $40." Phrasing that points at a specific capability deserves a specific tool.

## When plain generation is the right move

Skip tools when:

- **The answer is general knowledge or domain reasoning** the model has from training. Policy explanations, definitions, how-to advice that does not require fresh data.
- **The question is about the tools themselves**: "Can you check my order status if I give you the number?" The model should explain and then optionally call — not call with a placeholder.
- **The user wants prose.** A summary, a draft email, a creative reply. There is nothing to look up.
- **The tool call would cost more than the answer is worth.** Calling a slow downstream API to confirm something the user can read in front of them is wasted budget. Use judgment when describing tools.

A useful framing: tools exist to *ground* the model in facts and *enact* its decisions. If you do not need grounding or action, you do not need a tool.

## Steering the model with `tool_choice`

Most providers let you control how aggressively the model uses tools per call. On Anthropic the parameter is `tool_choice`; OpenAI exposes the same field with the same name. Four common values:

- **`auto`** (default) — the model chooses whether to call a tool or reply directly. The right setting for most conversational apps.
- **`any` / `required`** — the model *must* call one of the available tools. Use when the application's contract is "this turn produces a tool call, full stop" (a triage step, a routing step, the structured-output pattern from `mod-102` chapter 5).
- **`tool` (specific name)** — the model must call the named tool. Use for forced structured output and for "I already know what to call, just fill the args."
- **`none`** — the model *must not* call a tool. Use during the final synthesis turn of an agent, or to prevent loops.

A small example showing each:

```python
# Auto (default). The model decides.
client.messages.create(..., tools=TOOLS)

# Force any tool — useful when the app is "one tool per turn" by design.
client.messages.create(..., tools=TOOLS, tool_choice={"type": "any"})

# Force a specific tool — useful for structured output.
client.messages.create(..., tools=TOOLS, tool_choice={"type": "tool", "name": "record_triage"})

# Forbid tools — usually the final wrap-up turn.
client.messages.create(..., tools=TOOLS, tool_choice={"type": "none"})
```

(`tool_choice` exists for OpenAI under the same name with shapes `"auto"`, `"required"`, `"none"`, or `{"type": "function", "function": {"name": "..."}}`.)

`tool_choice` is not a substitute for good descriptions. If your tool description is vague and your `tool_choice` is `auto`, the model will be flaky. If your description is sharp, `auto` will pick correctly most of the time.

## Multiple tools and how the model chooses

When you advertise several tools, the model picks based on the description, the parameter shapes, and the rolling conversation. Two patterns reliably increase correct selection:

1. **One clear purpose per tool.** Resist the urge to put a `mode` flag on a tool that does two things; split it into two tools with separate descriptions. The model picks between tool names more reliably than between enum values.
2. **Anti-overlap descriptions.** If two tools sit close in meaning, say so explicitly. `cancel_order`'s description should mention "Do not use `cancel_order` to issue a partial refund — use `create_refund` for that." The model reads the descriptions before each turn and uses them as routing hints.

You can also order matters at the margin: most providers do not document a strong recency or position bias for tool definitions, but keeping the most common tool near the top of your list is a cheap habit. If a particular tool keeps getting picked wrongly, the description is the first thing to tighten; the order of the `tools` list is the last.

### Parallel tool calls

Most current Anthropic and OpenAI models can emit several `tool_use` blocks (or `tool_calls`) in one assistant turn — for example, looking up an order *and* the customer's address in one shot. The loop from chapter 3 already handles this: execute every block, send all results back in one user turn.

Two implications:

- **A parallel call is not always a parallel intent.** Sometimes the model would have been better served by a sequential plan (look up the order, *then* decide whether to look up the address). If your dispatcher logs sequences and you see redundant parallel calls, sharpen the descriptions to distinguish them.
- **You can disable parallel tool use** via `disable_parallel_tool_use=True` (Anthropic) or `parallel_tool_calls=False` (OpenAI) when you want strict one-call-at-a-time behavior — useful for tutorials, debugging, and tools where ordering matters.

## Cost, latency, and the user

Each tool call is at least one extra model round trip plus the tool's own execution time. For a chat app this is invisible if your tools are fast; for anything user-facing it adds up. A few practical numbers worth carrying around:

- A `tools=[...]` call is *slightly* more expensive than the same call without tools, because the schema sits in the prompt. The schemas count as input tokens, on every turn. Keep them lean.
- A round trip that calls a tool, then calls again with the result, is roughly **twice** the latency of a single call (plus tool execution). A three-step chain is roughly three times.
- Streaming masks the cost of the *final* reply but does not mask the cost of intermediate tool round trips. Users will see a pause between "Let me check on that" and the streamed answer.

The right mitigations:

- Cache stable tool definitions in the prompt (provider features like prompt caching from `mod-101` work here too).
- For high-frequency apps, choose tools the model can reach without a round trip (e.g., precomputed counts) over ones that require live lookups.
- Stream the final user-facing turn but skip streaming during tool round trips — simpler code, no visible degradation.

## Side effects and human-in-the-loop

For side-effecting tools, the question "when to call a tool?" overlaps with "who is allowed to authorize this?" Three common patterns:

- **Read freely, write with confirmation.** The model can call `get_order_status` whenever it wants. Before calling `create_refund`, the application surfaces the proposed call to the user and waits for an explicit "yes." This is `tool_choice="none"` on the confirmation turn, then a forced call once approval is recorded.
- **Soft side effects without confirmation.** A "send_search_results_to_email" tool is mild enough to allow auto. A "delete_account" tool is not.
- **Hard caps.** Even with confirmation, never let the model call `create_refund` for more than a domain-appropriate cap without a separate manual approval. The cap belongs in your function, not in the prompt.

This is the simplest version of agent safety, and it is enough for everything in this module. `mod-105` and `mod-106` go further.

## The decision flow, written down

A short rubric you can pin above the keyboard:

```
1. Does the answer need fresh data from a system I run?       -> tool
2. Does the answer require a side effect?                     -> tool (with confirmation if risky)
3. Could a deterministic function answer this better?         -> tool
4. Is the user explicitly asking for a capability I built?    -> tool
5. Otherwise                                                  -> generate
```

And on your side:

```
1. Is the tool's description sharp enough that a careful new engineer
   would pick the right tool? If not, fix the description before tuning tool_choice.
2. Are two tools close in meaning? Either merge them, or add anti-overlap
   text to both descriptions.
3. Are tool calls happening for things the model could just answer? Tighten
   the description, or remove the tool for the chat path.
```

## Common mistakes

- **Treating `tool_choice="any"` as a fix for "the model didn't call a tool."** Sometimes the right answer is plain generation; forcing a tool produces nonsense arguments.
- **One tool with a `mode` flag instead of N tools.** The model is worse at filling enums than at picking names.
- **Side effects without confirmation.** A model that issues refunds for plausible-sounding messages is a billing incident waiting to happen.
- **Treating each turn in isolation.** The model uses the rolling conversation to decide; if a prior `get_order_status` already returned the answer, it should reuse it, not re-fetch. Make sure your loop appends the result back so the model can see it.
- **Adding tools that overlap with the model's strengths.** A "do_arithmetic" tool for adding small numbers, or a "summarize_text" tool — these are usually a tax on every call without improving accuracy.

## Summary

Tools are for fresh data, side effects, deterministic computation, and explicit user intent — everything else should stay as generation. Steer the decision with sharp descriptions, one purpose per tool, and `tool_choice` when the application's contract requires it. Charge the model for what it touches: tool calls cost latency and money, and side effects can cost more than that. Get the decision right at the prompt and registry layer, and the loop from chapter 3 stays small and well-behaved.
