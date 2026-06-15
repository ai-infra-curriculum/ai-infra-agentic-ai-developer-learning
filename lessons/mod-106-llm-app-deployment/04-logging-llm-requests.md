# Logging LLM Requests

## Why this matters

A script shows you what is happening because you are watching the terminal. A service shows you what is happening only if you wrote it down. In an LLM app this is not a "we should probably have logs" hygiene point — it is the difference between "we know why yesterday's bill spiked" and "we have a theory." Every meaningful operational question about an LLM service ("which prompt got slow," "which user is abusing the API," "did the new model help our latency," "why was that answer wrong") is answered by reading a stream of structured per-request log lines.

This chapter is about the contents of that line. What goes in. What does *not* go in. How to emit it as JSON so a future log-aggregation tool can index it. And the single hardest question in LLM logging: do you log the prompt?

## One line per request

The aspiration is dead simple: one structured log line per `/chat` request, written when the request finishes. That line is the unit of observation.

A good line looks like this:

```json
{
  "ts": "2025-10-22T19:14:03.812Z",
  "level": "INFO",
  "event": "chat_request",
  "request_id": "9f8c5b4d2a14",
  "session_id": "user-1029",
  "model": "claude-sonnet-4-6",
  "endpoint": "POST /chat",
  "status_code": 200,
  "input_tokens": 412,
  "output_tokens": 187,
  "steps": 3,
  "estimated_cost_usd": 0.0041,
  "latency_ms": 2840,
  "outcome": "ok"
}
```

That's it. A single row. Anyone looking at a stream of these rows can answer:

- **"What was yesterday's bill?"** `sum(estimated_cost_usd)` for the day.
- **"Are we slow?"** `p50/p90/p99(latency_ms)` per hour.
- **"Is any one caller abusive?"** `count by session_id`, ordered descending.
- **"Did the new prompt land?"** Median tokens before and after the deploy.
- **"Why is one specific request slow?"** Find it by `request_id` and read the row.

You will write this format yourself; the JSON shape is the whole point.

## Why structured (and why JSON)

A plain-text log line — `INFO chat ok 2840ms` — is unsearchable past a `grep`. Structured logs are searchable by field with off-the-shelf tools (CloudWatch Insights, Datadog Logs, Grafana Loki, Elasticsearch, even `jq` at the command line on a flat file).

JSON is the lingua franca. Almost every aggregator accepts JSON lines. The contract is one document per line, no embedded newlines:

```jsonl
{"ts":"2025-10-22T19:14:03.812Z","event":"chat_request","request_id":"9f8c","latency_ms":2840,...}
{"ts":"2025-10-22T19:14:04.520Z","event":"chat_request","request_id":"a73e","latency_ms":1180,...}
```

You will see other formats in the wild — logfmt (`ts=... level=INFO event=chat_request ...`), keyvalue, OpenTelemetry attributes — and they are all defensible. JSON is the most portable; pick a different format only if your platform team has already standardized on one.

> The acronym to know is JSONL: JSON Lines. One document per line, no top-level array. <https://jsonlines.org/>

## What fields belong on the line

The set below is the minimum that turns "we have logs" into "we can answer questions." Add fields as you learn what questions you ask repeatedly; remove fields you have never read.

### Identity and routing

- **`ts`** — ISO 8601 timestamp with timezone (`Z` for UTC). Required.
- **`request_id`** — a UUID stamped at the start of the request. Used to correlate this line with traces, downstream logs, and the user's report. Required.
- **`session_id`** — optional, supplied by the caller for multi-turn sessions. Lets you bucket all of one user's traffic.
- **`endpoint`** — `POST /chat` is enough; for future endpoints, the method + path.
- **`status_code`** — the HTTP status returned. Required.

### LLM call

- **`model`** — the model id you actually called. Pin it; you will rotate it more often than you expect.
- **`provider`** — `anthropic` or `openai`. Useful the day you add a second provider.
- **`input_tokens`** — total input token count (the provider returns this). Required.
- **`output_tokens`** — total output token count. Required.
- **`steps`** — for an agent, the number of model calls in the loop. Optional for non-agent endpoints.
- **`estimated_cost_usd`** — derived field; tokens × rate per million. Chapter 5 puts the rate table in code.

### Performance

- **`latency_ms`** — server-side latency: handler entry to handler return. Required.
- **`provider_latency_ms`** — optional but useful: time spent inside `messages.create(...)`. The gap between `latency_ms` and `provider_latency_ms` is your own overhead, and a good early signal that something local is slow.

### Outcome

- **`outcome`** — `ok`, `validation_error`, `provider_error`, `provider_timeout`, `max_steps_exceeded`, `guardrail_tripped`. A small fixed vocabulary. Required.
- **`error_class`** — for failures, a more specific tag (e.g. `provider_429`). Optional.

### What you may *not* want every time

- **`user_ip`** — useful for abuse triage, but PII in many jurisdictions; chapter 4's stretch goal is hashing.
- **`first_tool_called`** / **`last_tool_called`** — for the agent endpoint, the names of the first and last tool the agent used. Cheaper than the full trace and answers "which tool is the agent over-using?"

That's the field set you will write in exercise-03. Treat the list as a starting schema; it will change.

## The prompt-and-PII boundary

The hardest question in LLM logging is: do you log the actual prompt the user sent?

The trade-off is real:

- **Pro.** When you cannot reproduce a bad answer, the prompt is what you need to debug. A log with `input_tokens: 412` and no prompt does not tell you why the agent answered wrong.
- **Con.** Prompts contain PII — emails, addresses, order numbers, occasional credit-card digits — and uploading PII to your log aggregator inherits all the regulatory weight of "we store this." GDPR, CCPA, HIPAA, SOC 2 controls. The bill goes up. The deletion process gets harder.

A useful default for this module: **do not log raw prompts or raw model output by default**. Log a hash, a length, and a truncated preview if the env flag says so:

```python
import hashlib

def prompt_metadata(prompt: str, debug: bool):
    h = hashlib.sha256(prompt.encode("utf-8")).hexdigest()[:16]
    payload = {"prompt_sha256_16": h, "prompt_chars": len(prompt)}
    if debug:
        payload["prompt_preview"] = prompt[:200]
    return payload
```

The hash lets you detect duplicates ("ten people asked the same question") and recognize a specific prompt when a user reports it back to you, without storing the text. The preview is gated on a settings flag (`LOG_PROMPT_PREVIEW=true`) so you can flip it on in dev for debugging and leave it off in production.

A second useful default: **never log secrets**. Add a known-fields filter that drops anything matching `api_key`, `password`, `token`, `secret` from the JSON before it goes out. (Concrete redaction code is a stretch goal; the rule lives at the boundary between your logger and your handler.)

You will see vendors logging full prompts because they want them for evaluation. That is a legitimate use, but it is *its* own decision with *its* own legal review. Treat "log full prompts" as a feature you ship deliberately, not the default.

## Writing the logger

Python's stdlib `logging` is fine; you do not need a dependency. The shape:

```python
# logging_config.py
import json
import logging
import sys

class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "ts":    self.formatTime(record, "%Y-%m-%dT%H:%M:%S.%f"),
            "level": record.levelname,
            "event": record.getMessage(),
        }
        # `extra=...` lands in record.__dict__; anything custom we surface.
        for k, v in record.__dict__.items():
            if k in ("args", "msg", "levelname", "levelno", "pathname",
                     "filename", "module", "exc_info", "exc_text", "stack_info",
                     "lineno", "funcName", "created", "msecs", "relativeCreated",
                     "thread", "threadName", "processName", "process",
                     "name", "message"):
                continue
            payload[k] = v
        return json.dumps(payload, default=str)

def configure_logging(level: str = "INFO") -> None:
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    root = logging.getLogger()
    root.handlers = [handler]
    root.setLevel(level)
```

Usage in the handler:

```python
import logging
import time
import uuid

log = logging.getLogger(__name__)

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings: Settings = Depends(get_settings)) -> ChatResponse:
    request_id = uuid.uuid4().hex[:12]
    started = time.monotonic()
    try:
        result = run_agent(req.question, max_steps=req.max_steps, ...)
    except MaxStepsExceeded:
        log.info("chat_request", extra={
            "request_id": request_id, "session_id": req.session_id,
            "endpoint": "POST /chat", "status_code": 504,
            "outcome": "max_steps_exceeded",
            "latency_ms": int((time.monotonic() - started) * 1000),
        })
        raise
    log.info("chat_request", extra={
        "request_id":         request_id,
        "session_id":         req.session_id,
        "endpoint":           "POST /chat",
        "status_code":        200,
        "model":              settings.model,
        "input_tokens":       result["usage"]["input_tokens"],
        "output_tokens":      result["usage"]["output_tokens"],
        "steps":              result["steps"],
        "estimated_cost_usd": estimate_cost(result["usage"], settings.model),
        "latency_ms":         int((time.monotonic() - started) * 1000),
        "outcome":            "ok",
    })
    return ChatResponse(...)
```

Two observations:

- The success and failure paths emit the *same shape*. Same fields, different `outcome` and `status_code`. Querying the stream is much easier when the shape is invariant.
- The handler is starting to grow a `try/except` block. Push it into middleware in exercise-03 once you've felt the duplication.

## Where logs go

In this module, logs go to **stdout**. That is correct and also the entire production answer:

> A twelve-factor app treats logs as event streams... A twelve-factor app never concerns itself with routing or storage of its output stream.
>
> — <https://12factor.net/logs>

The application writes lines to stdout. The orchestrator (Docker, Kubernetes, systemd) routes those lines to wherever they should go. You do not write to a file. You do not call a logging vendor's SDK from inside your handler. Stdout. JSON. Done.

When the time comes for centralized log aggregation, the platform team mounts a fluent-bit / Vector / Datadog agent sidecar that reads your container's stdout and ships it. Nothing in your code changes.

## Tracing, briefly

A log line is a *point*; a trace is a *span* — a tree of operations with parent/child relationships and timings. You will not stand up tracing in this module. But: the `request_id` field you emit on every log line is the seed of distributed tracing. The day you adopt OpenTelemetry, you'll add `trace_id` and `span_id` next to `request_id` and your existing log queries will still work.

OpenTelemetry has emerging GenAI semantic conventions (linked from `resources.md`) that name attributes like `gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`. If you name your log fields with reasonable approximations to those (`model`, `input_tokens`) you keep a path to adoption without renaming everything later.

## Cost estimation, mechanics only

The `estimated_cost_usd` field deserves a brief note even though chapter 5 owns the budget enforcement:

```python
PRICES_USD_PER_MTOK = {
    # placeholder rates; check provider pricing pages for current values.
    # <!-- needs-research: pull the current per-million input/output rates
    #      for the model id this module targets at exercise time -->
    "claude-sonnet-4-6": {"in": None, "out": None},
}

def estimate_cost(usage: dict, model: str) -> float:
    rates = PRICES_USD_PER_MTOK.get(model)
    if not rates or rates["in"] is None:
        return 0.0
    return (usage["input_tokens"]  * rates["in"]  / 1_000_000
          + usage["output_tokens"] * rates["out"] / 1_000_000)
```

Three things:

1. **Rates change.** Hardcoding them in `prices.py` is the right move *only if* you also write a TODO that says "pull these from the pricing page on every release." A check-in process that re-confirms the rates matters more than the code shape.
2. **Estimation is good enough.** A log line that says `estimated_cost_usd: 0.0041` is not your invoice; it is a signal. Provider invoices reconcile at the end of the month and may differ for cache hits, batch discounts, etc. Make sure the estimate is *within an order of magnitude* and you'll catch the runaway loops it's there to catch.
3. **`<!-- needs-research: ... -->` is the right placeholder when a rate is unknown.** Don't make up numbers; the curriculum convention is that unverified facts are flagged, not invented.

## Log levels and noise

A working rule for this module:

- **`DEBUG`.** The whole agent trace, prompt previews, intermediate tool args. Enabled only in dev.
- **`INFO`.** One line per request (the schema above). Startup line. Background-task completion.
- **`WARNING`.** Recoverable provider errors (a `429` that you retried successfully). Approaching the daily cap (e.g. at 80%).
- **`ERROR`.** Unrecoverable failures. The catch-all `500` handler logs at ERROR.
- **`CRITICAL`.** Used sparingly: process-is-going-to-die-level events. You almost never log at this level in a request handler.

`LOG_LEVEL` is set in `Settings`. Dev defaults to `DEBUG`; prod to `INFO`. Library noise (httpx connection pool, anthropic SDK debug logs) is silenced one-time at startup by setting their loggers explicitly:

```python
logging.getLogger("httpx").setLevel("WARNING")
logging.getLogger("anthropic").setLevel("WARNING")
```

Otherwise your one-line-per-request stream is buried in ten lines of HTTP client chatter per request.

## What a healthy log stream looks like

Tail a healthy service and you should see:

- One `startup` line on boot.
- One `chat_request` line per request, in order.
- Occasional WARNINGs for retried failures.
- ERRORs only when something actually broke.

If you see two lines per request, three lines per request, or — worse — partial lines, that is a bug. The signature of operational pain ten weeks from now is *exactly* the moment you let logging get sloppy in week one.

## A short checklist

Before chapter 5, your service should:

- [ ] Emit one JSON line per `/chat` request.
- [ ] Include `request_id`, `model`, `input_tokens`, `output_tokens`, `latency_ms`, `outcome`, `status_code`, and `estimated_cost_usd`.
- [ ] Never emit a raw API key, raw prompt, or raw model output by default.
- [ ] Set log level from `Settings`.
- [ ] Silence noisy third-party loggers (`httpx`, `anthropic`, `openai`).
- [ ] Write to stdout, not a file.

That is the floor.

## Summary

Observation is what turns an LLM service from a black box into a system you can answer questions about. The unit of observation is one structured JSON line per request, carrying identity, model, token counts, latency, and outcome. The boundary you defend most carefully is prompt content — hash it, count it, optionally preview a few characters, do not store the raw text by default. Write to stdout; the orchestrator routes from there. Cost estimation gives you a heads-up signal long before the provider invoice does, which is exactly the input chapter 5 needs to enforce a daily cap before the bill arrives.
