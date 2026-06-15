# The Chat/Messages Format and Roles

## Why this matters

Most LLM bugs you will ship are not "the model is wrong." They are "the model was sent the wrong messages." A clean mental model of the chat format — what each role means, what the server stitches together, what content blocks even are — pays back for the entire track. Tool calling, retrieval, and agent loops are all just disciplined uses of this same structure.

## The shape of a conversation

A chat-style request is a list of messages, each with a `role` and `content`. Roles are a fixed, small vocabulary:

- **`system`** — instructions from the developer: persona, constraints, formatting rules, safety policy. The user never authors this. It conditions everything that follows.
- **`user`** — the human's turn. Whatever your application surfaces from end-user input (or from a calling service) goes here.
- **`assistant`** — the model's prior turns. You replay these so the model can see what it has already said.
- **`tool`** (or provider-specific equivalent) — the result of a function/tool call the model previously requested. Covered in detail in `mod-103`.

The messages array is **ordered**. The server concatenates the roles into a single prompt using a chat template the model was trained on. Reorder the messages and you can get a different answer, even if no individual message changed.

## Vendor placement of the system prompt

Two patterns dominate:

- **Anthropic Messages API** treats the system prompt as a *top-level parameter*, not a message. The `messages` array starts with `user`.

  ```json
  {
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "system": "You are a precise technical assistant. Refuse non-technical requests.",
    "messages": [
      { "role": "user",      "content": "What's the capital of France?" },
      { "role": "assistant", "content": "Paris." },
      { "role": "user",      "content": "And of Spain?" }
    ]
  }
  ```

- **OpenAI Chat Completions** treats the system prompt as the first message with `role: "system"`.

  ```json
  {
    "model": "gpt-4.1-mini",
    "messages": [
      { "role": "system",    "content": "You are a precise technical assistant." },
      { "role": "user",      "content": "What's the capital of France?" },
      { "role": "assistant", "content": "Paris." },
      { "role": "user",      "content": "And of Spain?" }
    ]
  }
  ```

Same semantics, different shape. Both APIs require a strict pattern after the system prompt: `user` first, then strictly alternating `user`/`assistant`. Two `user` messages in a row, or starting with `assistant`, is a validation error.

## What each role is for

Treat the roles as channels with different powers:

- **System** carries *instructions you want the model to honor across the whole conversation*. Be explicit about persona ("you are X"), policy ("never reveal Y"), and format ("respond as a single JSON object"). Keep it stable across turns — re-rendering it every turn invalidates the prefix cache (more in chapter 7).
- **User** carries *the current request*. Resist the urge to dump instructions into user messages. If a rule must hold every turn, it belongs in `system`.
- **Assistant** is *what the model said last*. You almost never hand-author these. The two exceptions are: (a) when you are showing the model a worked example (few-shot), and (b) when you want to *seed* the next response by prefilling the first few characters (e.g., to force JSON to start with `{`).

A clean rule of thumb: **system = policy; user = task; assistant = history**. If you find yourself muddling those, your messages array will rot.

## Content blocks, not just strings

A message's `content` is more than text. In modern APIs it is a list of typed blocks:

```json
{
  "role": "user",
  "content": [
    { "type": "text",  "text": "Describe this chart." },
    { "type": "image", "source": { "type": "base64", "media_type": "image/png", "data": "..." } }
  ]
}
```

Common block types you will meet:

- `text` — plain text.
- `image` — a vision-capable model receives image bytes or a URL.
- `tool_use` (assistant) and `tool_result` (user) — the protocol for function calling, covered in `mod-103`.
- `document` / `file` — provider-specific blocks that attach PDFs or other files.

If you only need a single text block, both Anthropic and OpenAI accept a shorthand `"content": "..."` string. The list form is what the wire actually wants.

## Multi-turn assembled from scratch

Because the API is stateless, **you** are the source of truth for the conversation. A typical loop looks like:

```python
from anthropic import Anthropic

client = Anthropic()
history = []  # list of {"role": "...", "content": "..."}

def turn(user_text: str) -> str:
    history.append({"role": "user", "content": user_text})
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="You are a precise technical assistant.",
        messages=history,
    )
    assistant_text = response.content[0].text
    history.append({"role": "assistant", "content": assistant_text})
    return assistant_text
```

That tiny snippet is the kernel of every chatbot you will build in this track. Get it right and you can iterate. Get it wrong (forget to append the assistant turn, swap roles, include two `user` turns in a row) and you will see the model "forget" things or refuse to respond at all.

## Prefill: a small but useful trick

Anthropic and a handful of others let you seed the assistant's reply by appending an `assistant` message with partial text as the last item in `messages`. The model continues from there.

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=512,
    system="Respond with JSON only.",
    messages=[
        {"role": "user", "content": "Give me a user object for Ada Lovelace."},
        {"role": "assistant", "content": "{"},
    ],
)
# response continues from "{"
```

Used sparingly, prefill is the cheapest, most reliable way to force structured output. We will lean on it again in `mod-102`.

## Common mistakes

- **Double `user` turns** — appending the user message without also appending the prior assistant turn. The API will reject this.
- **Stale system prompt** — re-rendering the system prompt every turn with a new timestamp or session ID. Costs you the prefix cache and gives you nothing.
- **Instructions buried in user content** — a fragile pattern. Lift them to `system`.
- **Implicit "the model remembers"** — it does not. Re-send what matters.
- **Mixing vendors' message shapes** — Anthropic does *not* accept a `system` message in `messages`; OpenAI *requires* it there. Wrap each provider in its own adapter.

## Summary

A chat conversation is an ordered list of role-tagged content blocks. System carries policy; user carries the task; assistant carries history (and occasionally a prefill). You assemble the whole thing on every call because the API is stateless. Getting this layout right is the single highest-leverage skill in this module — every later topic assumes it.
