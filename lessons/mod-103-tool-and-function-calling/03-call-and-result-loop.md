# The Call/Result Loop

## Why this matters

You have a tool defined. The model can request to call it. Now you need to actually run the loop: detect that a tool call was emitted, execute the corresponding code, send the result back, and let the model continue until it produces a final reply. This loop is the operational heart of every tool-using app, including every agent in `mod-105`.

It is also where new code most often breaks. The model can answer in one turn, call one tool, call several tools in parallel, call a tool that errors, refuse, or hit `max_tokens` mid-call. Your loop has to handle all of those without spinning forever or silently dropping a result.

## The protocol, in one diagram

```
user message
   |
   v
[ model.create(tools=[...], messages=[...]) ]
   |
   +-- stop_reason == "end_turn"   -> done; reply to user
   |
   +-- stop_reason == "tool_use"   -> for each tool_use block:
   |                                     execute(tool, args)
   |                                     build tool_result
   |                                 append assistant turn (the tool_use)
   |                                 append user turn (the tool_results)
   |                                 loop
   |
   +-- stop_reason == "max_tokens" -> raise / increase / report
```

Three rules drop out of this picture:

1. **The assistant's tool-use turn is part of the conversation.** You append it back into `messages` verbatim, then append the user turn that carries the `tool_result`s. The model sees both on the next call.
2. **Tool results go in a *user* turn.** Counterintuitive but how the protocol is defined: from the model's perspective, your application is the "user" that supplies tool outputs.
3. **You keep looping until the stop reason is *not* `tool_use`.** Multiple back-to-back tool calls are normal; one user prompt can result in five round trips.

## Walking through one round trip (Anthropic)

A minimal, runnable shape with a single tool:

```python
from anthropic import Anthropic

client = Anthropic()

TOOLS = [{
    "name": "get_order_status",
    "description": "Look up the current status of a customer order by ID.",
    "input_schema": {
        "type": "object",
        "required": ["order_id"],
        "properties": {"order_id": {"type": "string"}},
    },
}]

def get_order_status(order_id: str) -> dict:
    # In real code, hit a database. Here, fake it.
    return {"order_id": order_id, "status": "shipped", "carrier": "UPS"}

messages = [{"role": "user", "content": "Where is order ORD-4421?"}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=TOOLS,
    messages=messages,
)
```

If the model wants to call the tool, `response.stop_reason == "tool_use"` and the content list contains a `tool_use` block (and may also contain a leading `text` block — the model often narrates "Let me check that for you" before the call).

You handle it like this:

```python
# 1. Append the assistant's turn verbatim into the conversation.
messages.append({"role": "assistant", "content": response.content})

# 2. Execute every tool_use block and collect the results.
tool_results = []
for block in response.content:
    if block.type != "tool_use":
        continue
    if block.name == "get_order_status":
        result = get_order_status(**block.input)
        tool_results.append({
            "type": "tool_result",
            "tool_use_id": block.id,
            "content": json.dumps(result),
        })

# 3. Append a user turn that carries the tool_result block(s).
messages.append({"role": "user", "content": tool_results})

# 4. Call again.
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=TOOLS,
    messages=messages,
)
```

The second call lets the model continue with the new information. Most commonly it now produces a final `text` reply ("Your order ORD-4421 shipped via UPS."), and `stop_reason == "end_turn"`. Done.

A few details that matter:

- **`tool_use_id` must match exactly.** It is how the model correlates the result with the request it made. Copy the id from the `tool_use` block directly.
- **`content` is a string.** Even when your function returns a dict, serialize it (`json.dumps(...)`) before sending. The model treats it as text.
- **Pass `tools=` on every turn.** The tools are part of the prompt, not a one-time registration. Drop them and the model forgets they exist.

## The same loop, generically

In production you do not hand-code a branch per tool name. You build the loop once around a dispatcher (chapter 4 covers the dispatcher in detail):

```python
def run_with_tools(client, messages, tools, dispatch, *, max_steps=10):
    for step in range(max_steps):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "max_tokens":
            raise RuntimeError("hit max_tokens mid-loop")

        if response.stop_reason != "tool_use":
            return response  # end_turn or stop_sequence

        messages.append({"role": "assistant", "content": response.content})

        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            payload = dispatch(block.name, block.input)   # returns a JSON-serializable result
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": json.dumps(payload),
            })

        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError(f"exceeded max_steps={max_steps}")
```

The shape is small and worth memorizing:

```
loop:
  call -> if no tool, return; else execute every tool_use, append, repeat
guard with a step cap.
```

The step cap (`max_steps`) is non-negotiable. Without it a misbehaving model can call the same tool forever. Five to ten is a reasonable default for simple apps; agents (`mod-105`) raise it but add additional guards.

## The same loop on OpenAI

OpenAI's protocol is the same; the field names differ.

A tool call comes back on the assistant message as `tool_calls`:

```python
message = response.choices[0].message
if message.tool_calls:
    for call in message.tool_calls:
        name = call.function.name
        args = json.loads(call.function.arguments)   # arguments arrive as a JSON STRING
        result = dispatch(name, args)
        # Append the assistant message verbatim, then a tool message per call:
        messages.append(message)                     # the assistant turn
        messages.append({
            "role": "tool",
            "tool_call_id": call.id,
            "content": json.dumps(result),
        })
```

Two differences worth flagging:

- OpenAI sends `arguments` as a **JSON-encoded string**; Anthropic sends `input` as a **parsed object**. Always parse before dispatching on OpenAI.
- OpenAI's tool result message has `role: "tool"` (not `user`); Anthropic puts the `tool_result` blocks inside a `user`-role turn.

The protocol is the same; only the carrier is different.

## Parallel tool calls

A single assistant turn can contain multiple `tool_use` blocks (Anthropic) or multiple `tool_calls` (OpenAI) when the model decides several independent calls are needed at once — e.g., "look up the order status *and* the customer's address." Two rules:

- **Execute every call** before sending results back. Sending a partial set will confuse the next turn.
- **Send all tool results in a single follow-up turn.** Anthropic expects all `tool_result` blocks for that round in one user message; OpenAI expects one `role: "tool"` message per call, all before the next assistant turn.

The dispatcher above already handles this — the inner `for block in response.content` loop runs every call, and the `tool_results` list is appended once. You may run the calls in parallel with `asyncio.gather` or a thread pool if your tools do I/O; the order of the result list does not matter as long as every `tool_use_id` is matched.

## Stop reasons you must handle

The model can leave the loop in several ways. Each one has a different correct response:

- **`end_turn` / `stop`** — the model finished its reply normally. Return the text.
- **`tool_use` / `tool_calls`** — the model wants tools run. Execute and loop.
- **`max_tokens` / `length`** — the model was cut off mid-output. Treat this as a failure: the partial `tool_use` may be missing arguments, the JSON may be truncated, or a final reply is half-written. Raise, log, and either increase `max_tokens` or shorten the prompt.
- **`stop_sequence`** — the model emitted a sequence you told it to stop on. Usually treated like `end_turn`.
- **`refusal`** (provider-specific) — the model declined. Surface this to the caller; do not silently retry forever.

Always branch explicitly. The default of "if it has tools, run them, else assume text" misses the failure cases.

## Streaming and tools

You can stream the model output while a tool call is being formed; the SDKs surface `input_json_delta` (Anthropic) or `tool_calls[*].function.arguments` deltas (OpenAI) so a UI can show progress. The mechanics are different but the loop is the same — wait for the stream to finish, run the tool, send the result back, stream the next turn.

If you are streaming for the *user-facing* response only, the simplest pattern is to disable streaming during tool round trips and re-enable it for the final reply. The latency cost is small and the code is much simpler.

## What the user sees

The model sometimes emits a leading text block before a tool call: "Let me check on that for you." That text is yours to surface or suppress. A useful UX rule:

- **Surface intermediate text** when the round trip will take noticeable time (a slow API call). It is honest and reads natural.
- **Suppress it** when the round trip is fast (a single DB query) — the user just wants the answer.

Tool results themselves are *not* shown to the user. They are model-facing context. If you want the user to see, e.g., the order status in a structured card, render it from the function's return value in your own code — do not rely on the model to reformat it.

## Common mistakes

- **Forgetting to append the assistant tool-use turn.** The next call has no record of what tool was just requested; the model regenerates a tool call and you loop forever.
- **Not echoing `tool_use_id` exactly.** The model loses track of which result belongs to which call.
- **Returning Python objects instead of strings.** Always `json.dumps(...)` the payload.
- **Sending tool results without first sending the assistant turn that requested them.** The provider will reject the request.
- **No step cap.** Misbehaving prompt + misbehaving model = infinite billing.
- **Treating `tool_use` as "the model errored."** It is the normal happy path. Treat `max_tokens` as the error; tool use is a feature.

## Summary

The call/result loop is small but precise: call → if `tool_use`, execute every block → append the assistant turn → append a user turn carrying `tool_result`s with matching ids → loop, capped. Handle every stop reason explicitly, batch parallel calls into a single follow-up, and always serialize results to strings. Once this loop is in your spine, the rest of the module — wiring functions, multi-tool selection, error handling — is incremental.
