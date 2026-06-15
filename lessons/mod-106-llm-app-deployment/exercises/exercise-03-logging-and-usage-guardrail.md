# exercise-03: Logging and Usage Guardrail

**Estimated effort:** 3 hours

## Objective

Take the service from exercise-02 (configured, validated, but quiet) and make it observable and bounded. You will emit exactly one structured JSON log line per `/chat` request with the fields you need to answer real operational questions, then add a process-level daily-spend guardrail that returns `429` once the cap is reached. Finally, you will wrap the whole thing in a Dockerfile and prove a teammate can run it with one `docker run`.

By the end you have the shippable shape: a tiny LLM service that logs what it does, refuses to spend more than its budget, and ships as a single image.

## Background

This exercise covers material from:

- [Chapter 4 — Logging LLM Requests](../04-logging-llm-requests.md)
- [Chapter 5 — Cost and Usage Guardrails](../05-cost-and-usage-guardrails.md)
- [Chapter 6 — Containerizing with Docker](../06-containerizing-with-docker.md)
- [Chapter 7 — Running Locally, End to End](../07-running-locally-end-to-end.md)

## Prerequisites

- A completed exercise-02: FastAPI service with `Settings`, `SecretStr`, and `.env`/`.env.example`.
- Docker Desktop, Podman, or colima installed and able to build and run a Python image. Verify with `docker run --rm hello-world`.
- A real API key in `.env`.

## The task

Three additions to the service from exercise-02:

1. A `logging_config.py` with a JSON formatter and a `configure_logging(level)` helper. One log line per `/chat` request, with a fixed schema.
2. A `guardrail.py` with a `DailySpendGuardrail` and a `prices.py` table that estimates cost from token counts. The handler pre-checks and post-charges.
3. A multi-stage `Dockerfile`, a `.dockerignore`, and a working `docker run` you can hand to someone else.

## Tasks

### 1. Structured JSON logging

In `src/logging_config.py`, write the `JsonFormatter` from chapter 4 (or a `python-json-logger` equivalent if you prefer the dependency). The formatter:

- Emits one JSON document per line, no embedded newlines.
- Includes `ts`, `level`, `event` on every line.
- Includes everything passed via `extra={...}`.
- Drops known internal fields from `LogRecord` (don't ship `pathname`, `funcName`, `args`, etc., unless you mean to).

Call `configure_logging(settings.log_level)` at app import time (above `app = FastAPI(...)`). Silence library noise:

```python
logging.getLogger("httpx").setLevel("WARNING")
logging.getLogger("anthropic").setLevel("WARNING")
```

### 2. The per-request log line

In the `/chat` handler, emit exactly one `INFO` log line per request, on the success path and on every failure path. The schema (from chapter 4):

```json
{
  "ts":          "2025-10-22T19:14:03.812Z",
  "level":       "INFO",
  "event":       "chat_request",
  "request_id":  "9f8c5b4d2a14",
  "session_id":  null,
  "endpoint":    "POST /chat",
  "status_code": 200,
  "model":       "claude-sonnet-4-6",
  "input_tokens":  412,
  "output_tokens": 187,
  "steps":         3,
  "estimated_cost_usd": 0.0041,
  "latency_ms":    2840,
  "outcome":       "ok"
}
```

The failure paths emit the same schema with the appropriate `status_code`, `outcome`, and an `error_class` field. Outcomes you must cover: `ok`, `validation_error`, `max_steps_exceeded`, `provider_timeout`, `provider_bad_request`, `guardrail_tripped`.

The cleanest way to land "exactly one line per request, including validation errors" is FastAPI middleware:

```python
@app.middleware("http")
async def log_requests(request: Request, call_next):
    request_id = uuid.uuid4().hex[:12]
    request.state.request_id = request_id
    request.state.start      = time.monotonic()

    response = await call_next(request)

    log.info("chat_request", extra={
        "request_id":  request_id,
        "endpoint":    f"{request.method} {request.url.path}",
        "status_code": response.status_code,
        "latency_ms":  int((time.monotonic() - request.state.start) * 1000),
        # outcome + token counts come from request.state set by the handler
        **getattr(request.state, "extra_log", {}),
    })
    return response
```

The handler stashes its specific fields onto `request.state.extra_log` and the middleware emits the row. You can ship without middleware — the handler can emit directly — but then a `422` from Pydantic produces no log line (it never reaches your handler). The middleware approach gets you "exactly one line per request" uniformly.

### 3. Prompt and PII handling

Implement the chapter-4 prompt-metadata helper:

```python
def prompt_metadata(prompt: str, preview: bool) -> dict:
    h = hashlib.sha256(prompt.encode("utf-8")).hexdigest()[:16]
    payload = {"prompt_sha256_16": h, "prompt_chars": len(prompt)}
    if preview:
        payload["prompt_preview"] = prompt[:200]
    return payload
```

Add `LOG_PROMPT_PREVIEW: bool = Field(default=False, ...)` to `Settings`. The handler calls `prompt_metadata(req.question, settings.log_prompt_preview)` and merges into `request.state.extra_log`. In your `.env` for dev you set it `true`; in production-ish runs you leave it `false`.

Do **not** log the full prompt or the full model output by default. Hash + length + 200-char preview is the right floor.

### 4. The guardrail

In `src/guardrail.py`, write `DailySpendGuardrail` from chapter 5: a dataclass with `cap_usd`, `spend_usd`, `day`, a `Lock`, and `check_and_charge(estimated_cost_usd)` that raises `GuardrailTripped`. Module-level `@lru_cache` factory `get_guardrail()` reads `settings.daily_usd_cap`.

In `src/prices.py`, define the rate table:

```python
PRICES_USD_PER_MTOK = {
    # <!-- needs-research: pull the current Anthropic / OpenAI per-million
    #      input and output rates for the model id this module targets;
    #      cite the pricing page URL and the date in code comments -->
    "claude-sonnet-4-6": {"in": None, "out": None},
}

def estimate_cost(usage: dict, model: str) -> float:
    rates = PRICES_USD_PER_MTOK.get(model)
    if not rates or rates["in"] is None:
        return 0.0
    return (usage["input_tokens"]  * rates["in"]  / 1_000_000
          + usage["output_tokens"] * rates["out"] / 1_000_000)
```

Look up the actual rate when you write the exercise. Pin it in code with a comment that names the pricing page URL and the date you copied it. Do not invent a number.

### 5. Wire the guardrail into the handler

```python
@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings:  Settings              = Depends(get_settings),
         guardrail: DailySpendGuardrail   = Depends(get_guardrail)) -> ChatResponse:
    guardrail.check_and_charge(0.0)           # pre-check (raises if tripped)

    result = run_agent(req.question, max_steps=req.max_steps,
                       api_key=settings.anthropic_api_key.get_secret_value(),
                       model=settings.model)

    cost = estimate_cost(result["usage"], settings.model)
    guardrail.check_and_charge(cost)          # post-charge

    # Stash for the middleware log line:
    request.state.extra_log = {
        "model":              settings.model,
        "input_tokens":       result["usage"]["input_tokens"],
        "output_tokens":      result["usage"]["output_tokens"],
        "steps":              result["steps"],
        "estimated_cost_usd": cost,
        "outcome":            "ok",
        **prompt_metadata(req.question, settings.log_prompt_preview),
    }
    return ChatResponse(...)
```

Register a `GuardrailTripped` exception handler that returns `429` with a body containing `detail` and `error_class: "guardrail_tripped"`. The middleware emits the log line with `outcome: guardrail_tripped` and `level: WARNING` (you'll need to either log inside the exception handler or have the middleware look at the response shape).

### 6. The Dockerfile

Write the chapter-6 multi-stage Dockerfile. Required properties:

- Two stages (`build`, `runtime`).
- Uses `python:3.12-slim` (or pinned equivalent).
- Final image runs as a non-root user.
- `HEALTHCHECK` hits `/healthz`.
- `CMD` runs uvicorn as PID 1.
- `PYTHONUNBUFFERED=1` so log lines reach stdout immediately.

Write a `.dockerignore` that excludes `.env`, `.git`, `__pycache__/`, `tests/`, `.venv/`, your editor's metadata.

### 7. The runbook

Update your `README.md` with the two-line teammate instructions:

```bash
cp .env.example .env       # fill in ANTHROPIC_API_KEY
docker build -t support-agent:0.1 .
docker run --rm -p 8000:8000 --env-file .env support-agent:0.1
curl localhost:8000/healthz
curl -X POST localhost:8000/chat -H 'content-type: application/json' \
     -d '{"question":"hi"}'
```

### 8. The smoke test

Capture each in `runs.md`:

1. **Build.** `docker build -t support-agent:0.1 .` succeeds.
2. **Image size.** `docker images support-agent` — confirm under ~250 MB.
3. **Non-root.** `docker run --rm support-agent:0.1 whoami` prints `app`, not `root`.
4. **Healthz.** `docker run -p 8000:8000 --env-file .env support-agent:0.1` then `curl localhost:8000/healthz` returns `200`.
5. **Happy path.** A `POST /chat` returns the response model.
6. **Validation log.** A `POST /chat` with `{"question":""}` returns `422` *and* the container emits exactly one structured JSON line for it.
7. **Guardrail trip.** Set `DAILY_USD_CAP=0.001` in `.env`, restart, hit `/chat` twice. First returns `200`; second returns `429` with `error_class: guardrail_tripped`.
8. **Restart resets the guardrail.** Stop the container, start it again, hit `/chat` — the counter is back to zero. (Document this; it's the in-memory tradeoff from chapter 5.)
9. **No secret in the image.** `docker history support-agent:0.1 --no-trunc | grep -i anthropic` returns nothing. `docker inspect support-agent:0.1` env section does not contain the key. `docker run support-agent:0.1 env | grep ANTHROPIC` only shows the value because you passed `--env-file .env`; remove the flag and the key is absent.
10. **The log stream is parseable.** `docker run ... 2>&1 | jq -c '. | select(.event=="chat_request")'` runs without errors and prints one row per request.

### 9. Read what you wrote

In `NOTES.md`, short paragraphs (4–8 sentences) on each:

- **The fields you logged.** Pretend you are the on-call engineer at 2 a.m. responding to "the bill spiked yesterday." Walk through the queries you'd run against your log lines to identify the cause. Are there fields you'd add? Fields you'd drop?
- **The prompt-and-PII decision.** You opted to hash + length + optional 200-char preview. If a product manager pushed for full prompts in logs to enable evals, what would you ask them to agree to before you flipped that switch?
- **The guardrail tradeoff.** The counter is in-memory, single process. Trace through three failure modes (process restart, multi-worker deploy, distributed denial-of-funds attack) where it falls short. For each, what's the smallest next step you'd take in a real deployment?

## Starter guidance

A minimal middleware that handles "one line per request" including validation errors:

```python
@app.middleware("http")
async def log_request(request: Request, call_next):
    request_id = uuid.uuid4().hex[:12]
    request.state.request_id = request_id
    started = time.monotonic()
    response = await call_next(request)

    fields = {
        "request_id":  request_id,
        "endpoint":    f"{request.method} {request.url.path}",
        "status_code": response.status_code,
        "latency_ms":  int((time.monotonic() - started) * 1000),
        **getattr(request.state, "extra_log", {}),
    }
    fields.setdefault("outcome",
        "ok" if response.status_code < 400 else
        "validation_error" if response.status_code == 422 else
        "error",
    )
    log.info("chat_request", extra=fields)
    return response
```

If you find the middleware getting tangled with the exception handler routing, simplify by emitting the log line *inside* each exception handler and *inside* the handler success path, and skip the middleware entirely. Either shape is acceptable; the goal is exactly one line per request.

## Things you should not do in this exercise

- Log the full prompt by default. Hash + length + optional preview is the floor.
- Log the API key — anywhere. Not in startup, not in tracebacks, not in `repr()`. `SecretStr` plus a redaction filter handles 95%; review the remaining 5% by hand.
- Use a third-party logging vendor's SDK (Datadog, Honeycomb, Sentry) inside your handler. Stdout JSON is the contract; the platform routes from there.
- Persist the guardrail to disk to "make it survive restarts." That's a different shape (and a different exercise). The in-memory floor is intentional.
- Set `DAILY_USD_CAP` to a huge number to skip the trip test. Forcing the trip is the point.
- Run as root in the container.
- `COPY .env .` in the Dockerfile. Ever.

## Acceptance criteria

You can demonstrate:

- One structured JSON log line per `/chat` request, covering at minimum `request_id`, `model`, `input_tokens`, `output_tokens`, `latency_ms`, `outcome`, `status_code`, and `estimated_cost_usd`.
- A working `DailySpendGuardrail` that returns `429` when tripped and emits a log line with `outcome: guardrail_tripped`.
- A `Dockerfile` that builds an image under ~250 MB, runs as non-root, and includes a `HEALTHCHECK`.
- A `.dockerignore` that demonstrably excludes `.env`, `.git`, `__pycache__`, and tests.
- `runs.md` capturing the 10 smoke-test steps with actual outputs.
- `NOTES.md` with the three reading-paragraphs from task 9.
- A teammate-runnable repo: cloning + `cp .env.example .env` + filling in the key + `docker build` + `docker run` lands them at a working service in under five minutes.

## Reflection

Answer briefly in `NOTES.md`:

1. The `estimated_cost_usd` you log is an estimate, not your invoice. Sketch what would have to be true for the estimate to be off by 2×. What would you do operationally if you saw such a divergence?
2. Suppose the `/chat` handler logs the input prompt only when `LOG_PROMPT_PREVIEW=true`. A teammate flips it on in production for "one quick debug." Walk through the cleanup steps once they realize what they've done. What's the smallest engineering change that would prevent the next person doing this?
3. The guardrail's `Lock` matters for async / multi-request safety. If you deleted the lock and ran with 20 concurrent `curl`s against `DAILY_USD_CAP=0.10`, what specifically would go wrong, and would the user notice?
4. Look at your Dockerfile. Which line, if changed accidentally, would silently break secret isolation? (There is at least one; identify it and add a comment so the next person doesn't trip over it.)

## Stretch goals

- **80% soft warning.** Extend `DailySpendGuardrail` to log a `WARNING` once when `spend_usd` crosses 80% of `cap_usd`. Add a once-per-day flag so it doesn't log every subsequent request.
- **Per-key rate limit.** Add `slowapi` (or write a small token-bucket) for per-IP rate limiting. Capture a row when it returns `429` with `error_class: rate_limited`.
- **Async port.** Move to `anthropic.AsyncAnthropic`, `async def chat(...)`, and `await asyncio.wait_for(...)` for the request timeout. Run the smoke test with 5 concurrent `curl`s and compare latency.
- **A `compose.yaml`.** Add a Docker Compose file that brings the service up with `docker compose up`. Add a second service (a `curl`-based smoke-tester or a tiny `redis` you don't use yet) to feel the shape.
- **Image-supply-chain hygiene.** Run `docker scout` (or `trivy image support-agent:0.1`) against your image and address any HIGH-severity findings. Note what changed.
- **Persisted guardrail.** Replace the in-memory counter with a SQLite-backed one (`sqlite3.connect(":memory:")` for tests, a path on disk for the container). Note what changes in the failure-mode list — and what new failure modes appear (race conditions on the file, locking on read).
