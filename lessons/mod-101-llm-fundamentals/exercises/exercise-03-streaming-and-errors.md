# exercise-03: Streaming and Robust Errors

**Estimated effort:** 3 hours

## Objective

Build the production-shaped version of your single-shot caller from exercise-01: it streams responses, instruments latency, classifies errors correctly, retries the right ones with the right policy, and degrades gracefully when the model is down or rate-limited.

## Background

This exercise covers material from:

- [Chapter 6 — Streaming Responses](../06-streaming-responses.md)
- [Chapter 7 — Estimating Cost and Latency](../07-cost-and-latency.md)
- [Chapter 8 — Errors, Rate Limits, and Retries](../08-errors-rate-limits-and-retries.md)

## Prerequisites

- Completed exercise-01 and exercise-02.
- Same provider account; same SDK.

## Tasks

### 1. Streaming basics

Refactor your single-turn caller from exercise-01 to use the streaming API:

- Print tokens as they arrive (do not wait for the full response).
- Record three timestamps: `t_request_sent`, `t_first_token`, `t_complete`.
- After the stream closes, print:
  - **Time-to-first-token (TTFT)** = `t_first_token - t_request_sent`.
  - **Total wall time** = `t_complete - t_request_sent`.
  - **Output tokens / sec** = `output_tokens / (t_complete - t_first_token)`.
  - The final `stop_reason` and `usage` totals.

### 2. A small "non-streaming vs. streaming" comparison

Run the same prompt three times non-streaming and three times streaming. Record TTFT and total wall time for both modes. In a short note, explain why streaming usually has *lower TTFT* but *similar total time*.

### 3. Error taxonomy

Add a thin layer above the SDK that classifies any exception as one of:

- `BadInputError` — `400`, `404`, `413`, `422`, auth errors. Not retryable.
- `RateLimitError` — `429`. Retryable.
- `TransientServerError` — `5xx` (and Anthropic's `529`). Retryable.
- `TimeoutError` — local or upstream timeout. Retryable.
- `UnknownError` — anything else. Log loudly; do not retry.

For each class, write a small unit-test or manual experiment that triggers it:

- `BadInputError`: send an invalid `model` string. Confirm you classify it correctly. Do *not* retry.
- `RateLimitError`: either burst calls until you hit a `429`, or simulate one (most SDKs let you inject a mock transport; otherwise, set your own per-second cap intentionally low and retry-loop until you see the SDK raise the typed exception).
- `TransientServerError` and `TimeoutError`: simulate by pointing the SDK at a localhost HTTP server that returns `503` or hangs. This avoids spending real tokens.

### 4. Retry with backoff and jitter

Wrap your call in a retry loop with these rules:

- Retry only the retryable classes above.
- Use exponential backoff: `sleep = uniform(0, min(cap, base * 2**attempt))` ("full jitter").
- Honor a server-supplied `Retry-After` header / SDK-exposed retry hint when present — it wins over your computed sleep.
- Cap at 5 attempts and at a 20-second sleep.

Test it against your simulated `503` server. Confirm with logs that the delays grow exponentially with jitter.

### 5. Idempotency and request IDs

For each call, log:

- The provider's request ID from the response headers (or from the SDK exception's `response` attribute on failures).
- A locally-generated `idem_key` (UUID per logical call, not per retry).

Confirm in your retry loop that the same `idem_key` is reused across attempts of the same call but a new `idem_key` is minted for each new logical call.

### 6. Streaming + errors

The harder case: handle errors that arrive *mid-stream*. Pick a way to simulate one (e.g., kill the network briefly, point at a server that emits a few SSE events then closes the connection).

- Display the partial output that did arrive.
- Surface a user-facing message like "the response was cut off."
- Do *not* silently retry a mid-stream failure: the partial output is already billed, and re-issuing the call produces a different reply. Surface and let the user decide.

### 7. Operational dashboard

Add a `--metrics` mode that, after running a batch of prompts from a file, prints a small table:

```
Calls:               20
Successes:           18
Retries used:         4
Failures surfaced:    2
TTFT p50 / p95:    420ms / 1.1s
Total p50 / p95:   1.8s / 4.6s
Input tokens:    14,318
Output tokens:    6,902
```

Keep it simple — Python `statistics` is enough.

## Starter guidance

A useful skeleton for the streaming call with timing:

```python
import time
from anthropic import Anthropic

client = Anthropic(max_retries=0, timeout=30.0)  # we run our own retry loop

def stream_once(messages):
    t0 = time.monotonic()
    t_first = None
    pieces = []
    with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=512,
        messages=messages,
    ) as stream:
        for text in stream.text_stream:
            if t_first is None:
                t_first = time.monotonic()
            pieces.append(text)
            print(text, end="", flush=True)
        final = stream.get_final_message()
    t_end = time.monotonic()
    return {
        "text": "".join(pieces),
        "ttft": t_first - t0 if t_first else None,
        "total": t_end - t0,
        "usage": final.usage,
        "stop_reason": final.stop_reason,
        "request_id": stream.response.headers.get("request-id"),
    }
```

For the simulated upstream, a 30-line `aiohttp` or `flask` server that returns `503` (or hangs forever) is sufficient. You do not need to run a real LLM locally to exercise the failure paths.

For the retry loop, prefer one well-tested helper used everywhere rather than ad-hoc try/except blocks scattered through your code.

## Acceptance criteria

You can demonstrate:

- A streaming call that prints tokens as they arrive and reports TTFT, total wall time, and tokens/sec.
- A non-streaming vs. streaming comparison with measured numbers and a one-paragraph explanation.
- An error classifier that maps SDK exceptions to your typed categories, with a triggered example for each retryable class.
- A retry loop with exponential backoff + full jitter that honors `Retry-After` / SDK retry hints and caps both attempts and sleep duration.
- A stable idempotency key per logical call, threaded through retries; provider request IDs logged on success and failure.
- Mid-stream errors that display partial output, surface a clear user message, and do not silently retry.
- A `--metrics` report over a batch of real calls.

## Reflection

Answer briefly:

1. In what scenarios is silent retry the *wrong* choice, even though the call clearly failed?
2. How does the `Retry-After` header change your backoff math?
3. If you were running this at 1,000 requests/minute and the provider rate-limit was 600 RPM, what changes would you make beyond retries?

## Stretch goals

- Add a **circuit breaker** that, after N consecutive transient failures, stops calling the provider for a cool-off window and immediately fails subsequent calls. Re-probe periodically.
- Build a **multi-provider fallback**: when provider A is unhealthy, automatically retry the call against provider B (e.g., Anthropic ↔ OpenAI). Adapt the message shape per provider in a thin adapter.
- Capture and display the rate-limit headers (`anthropic-ratelimit-*` or `x-ratelimit-*`) on every response. Add a warning when you cross 80% of your token-per-minute budget.
- Pipe streaming output through a real frontend (a small FastAPI server + HTML page using `EventSource`).
