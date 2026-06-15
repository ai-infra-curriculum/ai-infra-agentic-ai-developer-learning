# What an LLM API Actually Is

## Why this matters

You are about to spend the next several modules treating an LLM as a black box you call over HTTPS. Before that becomes muscle memory, take a beat to be precise about what is on the other side of the wire. Almost every "weird" behavior you will hit downstream — non-determinism, latency spikes, mysterious context cutoffs, 429s — has its roots here.

## The mental model

A large language model is a function. Conceptually:

```
generate(prompt, params) -> text
```

The model takes a sequence of tokens (a numeric encoding of text) and produces, one token at a time, a probability distribution over what should come next. A sampler picks a token from that distribution, appends it to the input, and the model runs again. It stops when it emits a special end-of-turn token, when it hits a server-side stop sequence, or when it reaches the response length cap you set.

That is the whole interface. Everything else — chat formats, tool calls, JSON modes, streaming — is a thin protocol layered on top of that loop.

A "chat" API call therefore does roughly this on the server:

1. Serialize the conversation (system prompt + alternating user/assistant turns) into a single token stream.
2. Run the model autoregressively to produce response tokens.
3. Either buffer the result and return it (non-streaming), or push tokens to you as they are produced (streaming).
4. Bill you separately for input tokens (what you sent) and output tokens (what was generated).

## What is in a request

Across providers the shape of a chat-style request is remarkably consistent. Anthropic's Messages API, for example, takes a JSON body like this:

```json
{
  "model": "claude-sonnet-4-6",
  "max_tokens": 1024,
  "system": "You are a precise technical assistant.",
  "messages": [
    { "role": "user", "content": "What is an LLM API?" }
  ]
}
```

OpenAI's Chat Completions endpoint takes the same idea with `messages` carrying a `system` role inline, but the parameters cluster the same way:

- **Which model** to run (`model`).
- **What context** to condition on (`system`, `messages`).
- **How to sample** (`temperature`, `top_p`, `max_tokens`, `stop_sequences`).
- **Optional shaping** (tools, response format, streaming).

The provider's job is to take that JSON and run the function above. Your job — and the rest of this module — is to learn to describe the call accurately, predict its cost and latency, and handle the ways it can fail.

## Calling the API in code

Both major Python SDKs are thin wrappers around the same HTTPS calls. Authentication is by API key on a header.

```python
# Anthropic
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "In one sentence: what is an LLM?"}],
)

print(response.content[0].text)
```

```python
# OpenAI
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])

response = client.chat.completions.create(
    model="gpt-4.1-mini",
    messages=[{"role": "user", "content": "In one sentence: what is an LLM?"}],
)

print(response.choices[0].message.content)
```

There is no magic. The SDK builds the JSON, signs it, sends it, and parses the JSON back into a typed object. You could write the same call with `requests` or `fetch` and it would work identically — and sometimes you should, when you need lower-level control over streaming, headers, or retries.

## What "stateless" means and why you will get burned

LLM endpoints are **stateless**. The server has no memory of your last call. If you want a conversation, *you* hold the conversation history and resend it on every request. The model sees only what is in the current request body.

This has three consequences you will feel repeatedly:

1. **Conversation cost grows quadratically with turn count** if you naively append every message. Turn N pays for re-sending turns 1..N-1 as input tokens. Plan for it.
2. **State changes (user identity, time, retrieved documents) must be re-injected** every turn. If you forget, the model will confidently use the stale version it last saw.
3. **There is no server-side "session"** to clean up. Reasoning about a single request in isolation is almost always the right unit of analysis.

Some providers offer caching, file references, or "assistants" that paper over this, but the underlying API is still stateless. When something looks like memory, somebody is still resending tokens.

## Where this module is going

The rest of the chapters tighten the loop:

- Chapter 2 nails down the chat/messages format and what each role is *for*.
- Chapter 3 makes tokens and context windows quantitative.
- Chapter 4 turns the knobs (temperature, top_p, stops) so you understand sampling.
- Chapter 5 is a decision framework for picking a model.
- Chapter 6 covers streaming — how token-by-token delivery actually arrives.
- Chapter 7 turns the meter on: cost and latency budgets you can defend.
- Chapter 8 handles the failure cases: retries, rate limits, and timeouts.

## Summary

An LLM API is a stateless HTTPS endpoint that runs a token-by-token text generator. You give it context, sampling parameters, and a budget; it gives you back tokens, billing metadata, and a stop reason. Everything fancy you will build sits on top of that loop — including the agents that motivate this whole track.
