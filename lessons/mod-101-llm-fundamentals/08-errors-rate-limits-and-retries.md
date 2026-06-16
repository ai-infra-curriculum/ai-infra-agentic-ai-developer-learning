# Errors, Rate Limits, and Retries

## Why this matters

The LLM API will, eventually, fail every way an HTTPS API can fail: bad auth, bad request, rate limit, server overload, network drop, timeout. The difference between a robust LLM app and a fragile one is almost entirely how it handles these. Worse, retries get expensive fast — every retry is another billed request. Get the policy right or your bill *and* your reliability suffer.

## A short taxonomy of failures

Status codes follow HTTP conventions, with provider-specific flavors.

- **`400 Bad Request`** — your request is malformed: invalid model ID, message schema violation, token limit exceeded, oversized image, conflicting parameters. **Do not retry.** Fix the request.
- **`401 Unauthorized` / `403 Forbidden`** — wrong API key, missing org permission, wrong region. **Do not retry.** Fix credentials or routing.
- **`404 Not Found`** — usually a typo'd model ID or deprecated endpoint. **Do not retry.**
- **`413 Payload Too Large`** — your request body is over the provider's limit. Trim and retry once.
- **`422 Unprocessable Entity`** — semantically wrong payload (e.g., violates a per-request constraint). **Do not retry.**
- **`429 Too Many Requests`** — rate limit. **Retry** with backoff, honoring `Retry-After`.
- **`500 / 502 / 503 / 504`** — transient server-side errors. **Retry** with backoff.
- **`529 Overloaded` (Anthropic)** — the model fleet is overloaded. **Retry** with backoff.
- **Client-side `TimeoutError` / `ConnectionError`** — network or local timeout. **Retry** with backoff. Confirm the request was *not* completed (idempotency).

The provider's SDK usually exposes these as typed exceptions — `RateLimitError`, `APIStatusError`, `APIConnectionError`, `APITimeoutError`, etc. Catch by type, not by status code parsing.

## Retry policy: exponential backoff with jitter

The defacto policy looks like this:

```python
import random
import time

def retry_with_backoff(call, *, max_attempts=5, base=0.5, cap=20.0):
    for attempt in range(max_attempts):
        try:
            return call()
        except RetryableError as e:
            if attempt + 1 == max_attempts:
                raise
            # Respect Retry-After if the server set one
            sleep_s = getattr(e, "retry_after", None)
            if sleep_s is None:
                # Exponential backoff with full jitter
                sleep_s = random.uniform(0, min(cap, base * (2 ** attempt)))
            time.sleep(sleep_s)
```

Two important details:

- **Honor `Retry-After`.** Both providers set it on `429`s. Override your computed backoff with it. Ignoring it means your spike comes back at exactly the wrong time.
- **Full jitter.** Without jitter, every client in your fleet retries at the same moment after a rate-limit event and you create a thundering herd. Spread retries randomly across the interval.

Cap the number of attempts (5 is a reasonable default) and cap the backoff (20–60s for interactive use). Better to surface a clean error to the user than to keep paying for retries on a permanently degraded backend.

## The SDKs already retry — know the default

Both major SDKs ship with automatic retries on transient errors. You usually do not need to write the loop above; you need to *understand* what the SDK is doing and tune it.

```python
from anthropic import Anthropic

client = Anthropic(max_retries=3)        # default ~2; tune per workload
client.with_options(timeout=30.0).messages.create(...)
```

```python
from openai import OpenAI

client = OpenAI(max_retries=3, timeout=30.0)
client.chat.completions.create(...)
```

What the SDK retries (rough cut, may evolve):

- Connection errors, timeouts.
- `408`, `409`, `429`, `5xx`.

What it does *not* retry: `400`, `401`, `403`, `404`, `422`. Those are your fault, not theirs.

For streaming calls, the SDK's auto-retry covers establishing the connection. If the stream dies *mid-response*, you generally need to handle that yourself.

## Idempotency and double-charging

A retry is only safe if the original call did not also succeed. If the request *did* get through and the server only failed to send the response, retrying will re-run the generation and you will be billed twice.

Mitigations:

- **Idempotency keys.** Some providers accept an `Idempotency-Key` header on POSTs. The server treats matching keys as the same logical request and replays the prior response if it exists. Use a UUID per logical user-visible call, not per retry.
- **Treat unknown failures as "maybe-succeeded."** For non-idempotent downstream effects (sending an email composed by the model, writing to a DB), log enough request context to deduplicate at the application layer.
- **Surface request IDs.** Both providers return a request ID in response headers (`request-id`, `x-request-id`). Log it on success and on error. It is the support ticket.

## Rate limits in shape

Rate limits are typically multi-axis. You can hit any of them:

- **Requests per minute (RPM)** — too many calls.
- **Tokens per minute (TPM)** — too many tokens in *or* out.
- **Input tokens per minute / output tokens per minute** — separately metered on some providers.
- **Concurrent requests** — too many in-flight at once.

Providers expose your current state via response headers (e.g., OpenAI's `x-ratelimit-remaining-requests`, `x-ratelimit-remaining-tokens`, `x-ratelimit-reset-*`). Anthropic exposes similar headers (`anthropic-ratelimit-*`). Surface them in your metrics.

When you are consistently bumping a limit, the long-term answers are:

- **Request a limit increase.** Both providers have a process.
- **Smooth your traffic.** A token bucket on your side prevents your own bursts.
- **Split workloads** across different keys/tiers (interactive vs. batch).
- **Use multiple regions/providers** as failover.

## Timeouts

Set explicit timeouts on every call. The default in many SDKs is generous — fine for batch, dangerous for interactive paths where a hung connection ties up a worker.

Three timeouts to think about:

1. **Connect timeout** — how long to wait for the TCP/TLS handshake. A few seconds is plenty.
2. **Read / streaming timeout** — how long between bytes. For streaming, this should be a small multiple of the expected gap between tokens.
3. **Overall request timeout** — a hard ceiling on total wall time. Calibrate to your latency target with headroom.

If a request times out from your side, *the work upstream may still complete*. Track that in your accounting.

## A robust call pattern

Putting it together:

```python
from anthropic import Anthropic, APIStatusError, APITimeoutError, RateLimitError

client = Anthropic(max_retries=3, timeout=30.0)

def safe_complete(messages):
    try:
        return client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=messages,
        )
    except RateLimitError:
        # Already retried by the SDK; we ran out. Degrade gracefully.
        raise UserVisibleError("We're rate-limited right now. Try again in a moment.")
    except APITimeoutError:
        raise UserVisibleError("The model took too long to respond. Try again.")
    except APIStatusError as e:
        # Includes 4xx and 5xx that survived retries
        log.error("LLM error", status=e.status_code, request_id=e.response.headers.get("request-id"))
        raise UserVisibleError("Something went wrong with the model.")
```

This is a starting point. Real systems add a circuit breaker (stop hammering an obviously dead backend), a fallback model or provider, and structured metrics on each branch.

## Things that look like errors but are not

A few "errors" you'll see that are actually the model behaving as designed:

- **`stop_reason: "max_tokens"`** — not an error; you asked for a short reply and you got one. Decide whether to continue.
- **`stop_reason: "refusal"` / safety-stopped output** — model declined. Surface gracefully; do not retry hoping for a different answer.
- **Empty content with a tool_use stop reason** — the model wants you to run a tool. Expected in agent loops (`mod-103`).

Inspect the response *fields*, not just the absence of an exception.

## Summary

Failures fall into "your fault" (don't retry, fix it) and "transient" (retry with exponential backoff and jitter, respecting `Retry-After`). Set explicit timeouts, log request IDs, and use idempotency keys where you can. Use the SDK's built-in retry policy and *tune* it; do not build a parallel one. And remember that every retry is another billed request — robustness is not free, so it has to be designed.
