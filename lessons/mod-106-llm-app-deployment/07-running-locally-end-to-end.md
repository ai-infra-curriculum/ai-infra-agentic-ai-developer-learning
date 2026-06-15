# Running Locally, End to End

## Why this matters

The first six chapters are layers. This one is the assembly: a single small repo where the FastAPI app, the Pydantic settings, the JSON log line, the daily guardrail, and the Dockerfile all show up in the same `tree` listing. Each layer was small on purpose; the work this chapter does is making sure they compose without re-implementing themselves at the seams.

This chapter is also the honest moment to draw the line between "runs in Docker on my laptop" and "runs in production." You will not have an autoscaler when you finish. You will have a service you understand end to end. That is the deliverable the rest of the track is built on.

## The repo layout

A working shape for the app you assemble in exercise-01 through exercise-03:

```
support-agent/
├── Dockerfile
├── .dockerignore
├── .env.example
├── .gitignore
├── pyproject.toml
├── README.md
├── src/
│   ├── __init__.py
│   ├── main.py              # FastAPI app, routes, exception handlers
│   ├── settings.py          # pydantic-settings Settings + get_settings()
│   ├── logging_config.py    # JsonFormatter + configure_logging()
│   ├── guardrail.py         # DailySpendGuardrail
│   ├── prices.py            # PRICES_USD_PER_MTOK + estimate_cost()
│   ├── schemas.py           # ChatRequest / ChatResponse / Usage
│   ├── errors.py            # MaxStepsExceeded / ProviderTimeout / GuardrailTripped
│   └── agent.py             # run_agent(...) — ported from mod-105
└── tests/
    ├── test_chat.py
    ├── test_settings.py
    └── test_guardrail.py
```

Eight Python files in `src/`. None of them is more than ~80 lines. The interesting consequence is that you can hand someone the README, the eight files, and the Dockerfile, and they understand the service in an afternoon.

## A full `main.py` you can run

Pulling the threads together — what your final `src/main.py` looks like at the end of exercise-03:

```python
# src/main.py
from __future__ import annotations

import time
import uuid
import logging

from fastapi import FastAPI, Depends, Request
from fastapi.responses import JSONResponse

from .settings        import Settings, get_settings
from .schemas         import ChatRequest, ChatResponse, Usage
from .logging_config  import configure_logging
from .guardrail       import DailySpendGuardrail, get_guardrail
from .errors          import MaxStepsExceeded, ProviderTimeout, GuardrailTripped
from .prices          import estimate_cost
from .agent           import run_agent

settings = get_settings()
configure_logging(settings.log_level)
log = logging.getLogger("app")

app = FastAPI(
    title="support-agent",
    version="0.1.0",
    docs_url="/docs"  if settings.docs_enabled else None,
    redoc_url="/redoc" if settings.docs_enabled else None,
)

@app.on_event("startup")
def on_startup() -> None:
    log.info("startup", extra={
        "env": settings.env, "model": settings.model,
        "max_steps": settings.max_steps,
        "daily_usd_cap": settings.daily_usd_cap,
        "anthropic_api_key_set": bool(settings.anthropic_api_key.get_secret_value()),
    })

@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings: Settings = Depends(get_settings),
         guardrail: DailySpendGuardrail = Depends(get_guardrail)) -> ChatResponse:
    request_id = uuid.uuid4().hex[:12]
    started    = time.monotonic()

    guardrail.check_and_charge(0.0)
    result = run_agent(
        req.question,
        max_steps   = req.max_steps,
        session_id  = req.session_id,
        request_id  = request_id,
        api_key     = settings.anthropic_api_key.get_secret_value(),
        model       = settings.model,
    )
    cost = estimate_cost(result["usage"], settings.model)
    guardrail.check_and_charge(cost)

    log.info("chat_request", extra={
        "request_id":         request_id,
        "session_id":         req.session_id,
        "endpoint":           "POST /chat",
        "status_code":        200,
        "model":              settings.model,
        "input_tokens":       result["usage"]["input_tokens"],
        "output_tokens":      result["usage"]["output_tokens"],
        "steps":              result["steps"],
        "estimated_cost_usd": cost,
        "latency_ms":         int((time.monotonic() - started) * 1000),
        "outcome":            "ok",
    })

    return ChatResponse(
        answer     = result["final_text"],
        session_id = req.session_id or request_id,
        request_id = request_id,
        usage      = Usage(
            input_tokens  = result["usage"]["input_tokens"],
            output_tokens = result["usage"]["output_tokens"],
            steps         = result["steps"],
        ),
    )

# --- exception handlers ---

@app.exception_handler(MaxStepsExceeded)
def handle_max_steps(request: Request, exc: MaxStepsExceeded) -> JSONResponse:
    return JSONResponse(status_code=504, content={
        "detail": "agent did not converge within step cap",
        "error_class": "max_steps_exceeded",
    })

@app.exception_handler(ProviderTimeout)
def handle_provider_timeout(request: Request, exc: ProviderTimeout) -> JSONResponse:
    return JSONResponse(status_code=504, content={
        "detail": "upstream model timed out",
        "error_class": "provider_timeout",
    })

@app.exception_handler(GuardrailTripped)
def handle_guardrail(request: Request, exc: GuardrailTripped) -> JSONResponse:
    return JSONResponse(status_code=429, content={
        "detail": "daily usage cap reached",
        "error_class": "guardrail_tripped",
    })
```

About 90 lines. Every responsibility introduced in chapters 2–5 is one named piece. The handler itself reads top to bottom as: request → guardrail pre-check → call agent → estimate cost → guardrail post-charge → log → respond.

## The five-minute smoke test

Once chapters 2–6 are done and the Dockerfile builds:

```bash
# 1. Build
docker build -t support-agent:0.1 .

# 2. Start
docker run --rm -p 8000:8000 --env-file .env --name agent support-agent:0.1 &
sleep 2

# 3. Health
curl -fsS localhost:8000/healthz
# {"status":"ok"}

# 4. Schema check (the docs page is the contract)
curl -fsS localhost:8000/openapi.json | jq '.paths."/chat".post.requestBody'

# 5. Happy path
curl -fsS -X POST localhost:8000/chat \
    -H 'content-type: application/json' \
    -d '{"question":"What is your return policy?"}'
# {"answer":"...","session_id":"...","request_id":"...","usage":{...}}

# 6. Validation rejection
curl -sS -o /dev/null -w '%{http_code}\n' \
    -X POST localhost:8000/chat \
    -H 'content-type: application/json' \
    -d '{"question":""}'
# 422

# 7. Stop
docker stop agent
```

That is the integration test. If any step is missing a status code or a response shape, you know which chapter to revisit.

A useful extension: tail the container logs and verify exactly one structured line per request:

```bash
docker run --rm -p 8000:8000 --env-file .env support-agent:0.1 \
    | tee logs.jsonl &
# ... make some requests ...
jq -c 'select(.event=="chat_request") | {request_id, outcome, latency_ms, input_tokens, output_tokens}' logs.jsonl
```

If your `jq` selector returns one row per `curl`, the logging chapter landed.

## Forcing the guardrail to fire

The daily cap was set deliberately low in chapter 3 (`DAILY_USD_CAP=2.0` in dev). To see it trip without spending real money:

1. Lower the cap further in `.env`: `DAILY_USD_CAP=0.05`.
2. Restart the container.
3. Make 3–4 requests in a row.
4. The fourth or fifth one returns `429`.

```bash
for i in $(seq 1 6); do
    curl -sS -o /dev/null -w 'req %s -> %{http_code}\n' "$i" \
        -X POST localhost:8000/chat \
        -H 'content-type: application/json' \
        -d '{"question":"hi"}'
done
# req 1 -> 200
# req 2 -> 200
# req 3 -> 200
# req 4 -> 429
# req 5 -> 429
# req 6 -> 429
```

Verify the structured log line for the `429`s carries `outcome: guardrail_tripped` and `level: WARNING`.

## Things you should test by hand at least once

You should not ship without seeing each of these with your own eyes:

- **Startup with a missing required secret.** Comment out `ANTHROPIC_API_KEY` in `.env`, restart. The process refuses to start; the validation error names the missing field. (Chapter 3.)
- **A malformed request.** `{"question": ""}` → `422`. `{}` → `422`. `{"question": "x", "max_steps": 99}` → `422`. (Chapter 2.)
- **An exhausted agent.** Pick a prompt that the agent can't finish in 3 steps; set `max_steps=3` in the request body. The response is `504`, the log has `outcome: max_steps_exceeded`. (Chapter 5.)
- **A `Ctrl-C`.** Stop the container with `docker stop` or `Ctrl-C` and watch uvicorn shut down gracefully — no orphaned requests, no in-flight call halfway through. If it hangs, your `CMD` line is wrong (chapter 6).
- **A second teammate runs it.** Send a coworker the repo URL and `.env.example`. If they can `docker build`, `docker run`, and hit `/chat` without you helping, the module did its job.

## Where this stops and what comes next

You are deliberately at the floor. The gaps from here to a production system are big, named, and not in this module:

- **Persistence.** Your daily counter resets on restart; your `THERMOSTAT_LOG` (or equivalent in-memory state) evaporates. The right home for durable state is a database, behind a connection pool. Operator track.
- **Auth.** The endpoint is open. The right floor is an API-key header verified in constant time; the real ceiling is OAuth/SSO with per-tenant rate limits. AI Engineer / Operator.
- **Multiple workers.** One uvicorn worker per container is fine; multiple workers or multiple containers behind a load balancer need a shared store for any per-process state (guardrail, cache). Operator.
- **Centralized logging.** Stdout is right; the agent that ships those lines to a queryable store is the missing piece. Platform.
- **Tracing.** `request_id` is the seed; OpenTelemetry + a backend (Honeycomb, Datadog, Tempo) is the next mile.
- **Deployment.** Compose for local + a CI job that builds and tags + a target environment that pulls the image is the smallest end-to-end. None of it is in this module.

These are not failures of the module. They are the next courses.

## A short pre-commit checklist

A useful set of questions to answer "yes" to before you call any of this done:

- [ ] `.env` is in `.gitignore`. `git ls-files` does not list it.
- [ ] `.env.example` is in the repo. It mirrors every field in `Settings`.
- [ ] `docker build -t support-agent .` succeeds against a clean checkout.
- [ ] `docker run --env-file .env support-agent` boots, serves `/healthz`, and answers `/chat`.
- [ ] `docker run -e ANTHROPIC_API_KEY="" support-agent` *fails to start* (missing required setting).
- [ ] Every `curl` produces exactly one structured log line.
- [ ] An empty-question `curl` returns `422` without contacting the model.
- [ ] A spammed loop with `DAILY_USD_CAP=0.05` returns `429` once tripped.
- [ ] The image contains no `.env`, no `.git`, no `__pycache__`.
- [ ] The image runs as a non-root user (`docker run support-agent whoami` is not `root`).

If all ten are checked, you have shipped the smallest honest LLM service this module asks you to ship.

## Summary

Every chapter introduced a layer; this one assembled them. The result is a single repository: FastAPI handler, Pydantic schemas, settings, structured logging, guardrail, Dockerfile, `.dockerignore`, `.env.example`. The smoke-test sequence — build, run, hit `/healthz`, hit `/chat`, force a `422`, force a `429` — exercises every layer end to end. What you have is *not* a production system, and it is not pretending to be one. It is the floor that every later course in this track assumes you can produce in an afternoon — and the floor that, when it is missing, makes every conversation about deployment, observability, or cost go in circles.
