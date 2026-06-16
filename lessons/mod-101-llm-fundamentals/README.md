# mod-101-llm-fundamentals: LLM Fundamentals for Application Developers

**Estimated effort:** 10 hours

This module gives you a working, application-developer's mental model of LLM APIs: what a call actually is, what each knob does, what it costs, how to handle failures, and how to estimate before you build. It is the foundation every later module — prompt engineering, tool calling, retrieval, agents — assumes you already have.

## Learning objectives

- Call LLM APIs and reason about tokens, context windows, temperature, and model selection.
- Understand the chat/messages format, the roles (system / user / assistant), and streaming responses.
- Estimate the cost and latency of LLM calls before and during production.
- Handle API errors, rate limits, and retries robustly.

## Lecture chapters

1. [What an LLM API Actually Is](01-what-is-an-llm-api.md) — the mental model, request shape, and why "stateless" matters.
2. [The Chat/Messages Format and Roles](02-chat-messages-and-roles.md) — system vs. user vs. assistant, content blocks, prefill, common mistakes.
3. [Tokens and Context Windows](03-tokens-and-context-windows.md) — what a token is, how to count it, and how to budget the window.
4. [Sampling, Temperature, and Determinism](04-sampling-temperature-and-determinism.md) — temperature, top_p, stop sequences, and the limits of reproducibility.
5. [Choosing a Model](05-choosing-a-model.md) — capability vs. cost vs. latency, tiers, routing, pinning model IDs.
6. [Streaming Responses](06-streaming-responses.md) — SSE, event shapes, perceived latency, when not to stream.
7. [Estimating Cost and Latency](07-cost-and-latency.md) — the cost formula, prompt caching, TTFT vs. throughput, what to instrument.
8. [Errors, Rate Limits, and Retries](08-errors-rate-limits-and-retries.md) — a taxonomy of failures, backoff with jitter, idempotency, timeouts.

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: First LLM API calls](exercises/exercise-01-first-llm-api-calls.md) — call the API, parse the response, build a tiny multi-turn loop.
- [exercise-02: Tokens, context, and cost](exercises/exercise-02-tokens-context-cost.md) — count tokens, write a context budget, project cost from real usage.
- [exercise-03: Streaming and robust errors](exercises/exercise-03-streaming-and-errors.md) — stream a response, instrument latency, handle rate limits and retries.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise immediately while the concepts are fresh. The chapters are short by design — the learning happens when you write code against a real provider and watch your token counts, latency numbers, and error responses come back. Bring a paid API key for at least one provider (Anthropic and/or OpenAI) and a terminal.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
