# Streaming Responses

## Why this matters

Non-streaming calls block until the entire reply is generated. For a 1,000-token answer, that can be many seconds of silence before anything happens at all. Streaming flips the experience: the first token arrives in a few hundred milliseconds, and the rest trickle in. The end-to-end latency is *the same*; the perceived latency is night and day. Every chat UI you have used does this for that reason.

## The transport: Server-Sent Events

Both major chat APIs stream with **Server-Sent Events (SSE)** over a single long-lived HTTPS response. The body looks like a sequence of `event:` and `data:` lines separated by blank lines. Each `data:` line is JSON for one event.

```
event: message_start
data: {"type": "message_start", "message": {...}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": "Hello"}}

event: content_block_delta
data: {"type": "content_block_delta", "index": 0, "delta": {"type": "text_delta", "text": " world"}}

event: message_stop
data: {"type": "message_stop"}
```

You almost never parse this by hand. The SDKs expose typed event streams.

## What the events mean

The exact event names differ by provider, but the *shape* is consistent: a start event, a series of incremental delta events, optional usage/metadata, and a stop event.

**Anthropic Messages API** event types you will see:

- `message_start` — message metadata, including the model id.
- `content_block_start` — a new content block begins (text, tool_use, etc.).
- `content_block_delta` — incremental text or partial JSON for the current block.
- `content_block_stop` — that block is complete.
- `message_delta` — top-level updates (stop reason, output token count).
- `message_stop` — generation is done.
- `ping` — keepalive; ignore.

**OpenAI Chat Completions** streams `chat.completion.chunk` objects with `choices[0].delta.content` carrying the incremental text, and a final `[DONE]` sentinel.

In both, the deltas are pure additions. You concatenate them in order to reconstruct the final string.

## Streaming in Python

The Anthropic SDK gives you a context manager that yields typed events and exposes the accumulated message at the end:

```python
from anthropic import Anthropic

client = Anthropic()

with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Explain SSE in two sentences."}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
    final = stream.get_final_message()
    print(f"\n[stop_reason={final.stop_reason}, output_tokens={final.usage.output_tokens}]")
```

The OpenAI SDK exposes `stream=True` on the same endpoint:

```python
from openai import OpenAI

client = OpenAI()
stream = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[{"role": "user", "content": "Explain SSE in two sentences."}],
    stream=True,
)
for chunk in stream:
    piece = chunk.choices[0].delta.content or ""
    print(piece, end="", flush=True)
```

In both, you have to consume the stream to completion — closing early on the client side typically cancels the request, but the partial tokens you have are already billed.

## What streaming buys you

- **Time-to-first-token (TTFT) becomes the UX metric.** Users perceive responsiveness from the first character, not the last. If TTFT is good, the rest can be slow without feeling slow.
- **Progressive rendering.** You can word-wrap, syntax-highlight, and animate as text arrives.
- **Early cancellation.** If the user hits "stop," you close the stream and stop paying for further output tokens.
- **Tool calls can start sooner** in agent loops — you can begin parsing tool arguments before the whole turn is done (advanced; see `mod-103`).

## What streaming costs you

- **Errors arrive mid-stream.** A network blip, a model timeout, or a moderation event can interrupt the response after some tokens have already been delivered. Your client must be ready to display a partial answer and surface the error.
- **You cannot easily set a response Content-Length** or buffer a complete reply for, say, JSON validation. Either validate at the end of the stream, or use prefill + non-streaming for strict structured output.
- **Backpressure.** If your client is slow to read the SSE bytes, the upstream may stall or eventually time out. Read fast and buffer.
- **Proxies and infra.** Some HTTP intermediaries (older load balancers, certain CDNs) buffer responses. If your tokens "arrive all at once," check the proxy chain, not the model.

## Wiring streaming into a backend

The common pattern is: your frontend opens an SSE or WebSocket connection to *your* backend, and your backend opens an SSE connection to the LLM provider, pumping deltas through. Two notes from the field:

- **Do not pass the provider stream through unmodified.** Map it to your own event vocabulary (`token`, `tool_call`, `done`, `error`). You will change providers; you will not want to change clients each time.
- **Track usage on the final event.** Your billing/observability depends on the input/output token counts in the closing message. Don't drop the last event.

A minimal FastAPI relay sketch:

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from anthropic import Anthropic

app = FastAPI()
client = Anthropic()

@app.post("/chat")
def chat(req: dict):
    def gen():
        with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=req["messages"],
        ) as stream:
            for text in stream.text_stream:
                yield f"data: {text}\n\n"
            yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

Production version adds error handling, structured event types, request IDs, and keepalive pings.

## When *not* to stream

Streaming is the default for chat UIs. It is the *wrong* default for:

- **Batch jobs and offline pipelines** — no human watching; you do not benefit from progressive delivery.
- **Server-to-server calls** where the consumer is another service that needs a complete object.
- **Strict structured output** where you cannot use a result until it is fully formed and validated.

For those, take the latency hit and use a non-streaming call. Fewer moving parts.

## Summary

Streaming uses Server-Sent Events to deliver tokens incrementally. It does not lower total latency; it lowers *perceived* latency, which is what users feel. Use it for any human-facing chat experience, but model your own event types over it, plan for mid-stream errors, and skip it for batch and structured-output paths.
