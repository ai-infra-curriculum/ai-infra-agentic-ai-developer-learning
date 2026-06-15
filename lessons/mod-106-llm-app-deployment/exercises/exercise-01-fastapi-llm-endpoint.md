# exercise-01: FastAPI LLM Endpoint

**Estimated effort:** 3 hours

## Objective

Wrap an LLM app from earlier in the track behind a small FastAPI service: a `POST /chat` endpoint with Pydantic request and response models, a `GET /healthz`, and an exception handler that maps known failures to honest HTTP status codes. You will run the service with `fastapi dev`, exercise it with `curl`, and convince yourself the validation layer is doing real work *before* the model is ever called.

By the end you should be able to point at every line of `src/main.py` and say what it does — including why the request body is a Pydantic model rather than a `dict`, and what `response_model=` is buying you.

## Background

This exercise covers material from:

- [Chapter 1 — From Script to Service](../01-from-script-to-service.md)
- [Chapter 2 — FastAPI and Input Validation](../02-fastapi-and-input-validation.md)

## Prerequisites

- One completed app from earlier in the track that exposes a single callable entry point: `mod-105` exercise-02 (`run_agent(question, ...)` returning `{"final_text", "trace", "steps"}`) is the canonical pick. `mod-103` exercise-02 or `mod-104` exercise-02 work too — the contract is "a function that takes a string and returns a dict."
- Python 3.10+ and an API key for one provider with native tool use.
- A small spend cap. This exercise should cost well under a dollar — most of your work is on the HTTP layer, not the model.

## The task

You will scaffold a small repo (`support-agent/`) with a FastAPI service that wraps your existing LLM app. The model call does not change. The shape *around* it does:

1. A `pyproject.toml` and a `src/` layout.
2. A FastAPI app with two routes: `GET /healthz` and `POST /chat`.
3. Pydantic models for request and response.
4. A central place for known application errors and an exception handler per error class.
5. A `README.md` with one `curl` example for each route, including one expected `422`.

The service does **not** need to read secrets from a settings object yet (that's exercise-02). For this exercise you may read `ANTHROPIC_API_KEY` directly from `os.environ` once at startup so you can focus on the HTTP layer.

## Tasks

### 1. Lay the repo out

```
support-agent/
├── pyproject.toml
├── README.md
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── schemas.py
│   ├── errors.py
│   └── agent.py     # ported from mod-105 (or wherever you're starting from)
```

Three rules:

- `agent.py` is a near-copy of your `mod-105` driver, exposed as a callable: `def run_agent(question: str, *, max_steps: int = 8, session_id: str | None = None) -> dict`.
- `main.py` does not import any provider SDK directly. The handler imports `from .agent import run_agent`; the agent imports the SDK.
- No `requirements.txt`. Use `pyproject.toml` with explicit pins for `fastapi`, `pydantic`, the provider SDK, and `uvicorn`.

### 2. Request and response models

In `schemas.py`:

```python
from pydantic import BaseModel, Field
from typing import Annotated

class ChatRequest(BaseModel):
    question:   Annotated[str, Field(min_length=1, max_length=4000)]
    session_id: Annotated[str | None, Field(default=None, max_length=64)]
    max_steps:  Annotated[int, Field(default=8, ge=1, le=12)]

class Usage(BaseModel):
    input_tokens:  int
    output_tokens: int
    steps:         int

class ChatResponse(BaseModel):
    answer:     str
    session_id: str
    request_id: str
    usage:      Usage
```

If your `run_agent(...)` does not currently return `usage`, plumb it through from the provider response (`resp.usage.input_tokens` / `resp.usage.output_tokens` on Anthropic, `resp.usage.prompt_tokens` / `resp.usage.completion_tokens` on OpenAI). For an agent that loops, sum across steps.

### 3. The handlers

In `main.py`:

- `GET /healthz` returns `{"status": "ok"}`. Does not touch the model, the network, or any file. Microseconds.
- `POST /chat` accepts `ChatRequest`, builds a `request_id`, calls `run_agent(...)`, returns a `ChatResponse`.

The handler body should be five named steps:

```python
@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    request_id = uuid.uuid4().hex[:12]
    result = run_agent(req.question,
                       max_steps=req.max_steps,
                       session_id=req.session_id)
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
```

No `try/except`. Failures are handled centrally in task 4.

### 4. Named errors and exception handlers

In `errors.py`:

```python
class MaxStepsExceeded(RuntimeError):  ...
class ProviderTimeout(RuntimeError):   ...
class ProviderBadRequest(RuntimeError): ...
```

Wire your `agent.py` to raise `MaxStepsExceeded` when its existing `RuntimeError("exceeded max_steps=...")` would fire. Wrap the provider call in a `try/except` that maps timeout/connection errors to `ProviderTimeout` and `400 Bad Request` responses to `ProviderBadRequest`.

In `main.py`, register handlers:

| Exception            | Status | Detail                                          |
|----------------------|--------|-------------------------------------------------|
| `MaxStepsExceeded`   | `504`  | `"agent did not converge within step cap"`      |
| `ProviderTimeout`    | `504`  | `"upstream model timed out"`                    |
| `ProviderBadRequest` | `502`  | `"upstream provider rejected the request"`      |
| `Exception` (catchall)| `500` | `"internal server error"`                       |

Each handler returns a `JSONResponse` with `{"detail": str, "error_class": str}`.

### 5. Drive the service with `curl`

Run with `fastapi dev src/main.py`. Hit each of the following and capture the request, response, and status code in `runs.md`:

1. `GET /healthz` — expect `200` and `{"status": "ok"}`.
2. `POST /chat` with a normal question — expect `200` and the response model.
3. `POST /chat` with `{"question": ""}` — expect `422` and a validation error pointing at `question`.
4. `POST /chat` with `{}` — expect `422` and a validation error pointing at the missing field.
5. `POST /chat` with `{"question": "x", "max_steps": 99}` — expect `422`. (Pydantic should reject the `max_steps` value before your handler runs.)
6. `POST /chat` with a wall-of-text question that forces a 5+ step agent and `{"question": "...", "max_steps": 2}` — expect `504` and `error_class: max_steps_exceeded`.
7. `GET /openapi.json` — confirm the schema includes `ChatRequest`, `ChatResponse`, and the validation rules.
8. Open `localhost:8000/docs` in a browser and "Try it out" the `/chat` endpoint.

### 6. Read what you wrote

In `NOTES.md`, write a short paragraph (4–6 sentences) on each:

- **Validation surface.** Which of the six test cases would have crashed your agent loop if Pydantic hadn't rejected them at the boundary? Be concrete — name the line that would have blown up.
- **The handler body.** Yours should be ~5–10 lines. If it grew larger, what did you push down into `agent.py`, and what is still left at the boundary that doesn't belong there?
- **The docs page.** What does `/docs` get right? What is misleading or missing for a teammate trying to drive your service?

## Starter guidance

A scaffold matching chapter 2:

```python
# src/main.py
from __future__ import annotations

import os
import uuid

from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

from .schemas import ChatRequest, ChatResponse, Usage
from .errors  import MaxStepsExceeded, ProviderTimeout, ProviderBadRequest
from .agent   import run_agent

app = FastAPI(title="support-agent", version="0.1.0")

# Quick sanity check at startup; exercise-02 replaces this with Settings.
if not os.environ.get("ANTHROPIC_API_KEY"):
    raise RuntimeError("ANTHROPIC_API_KEY is required")

@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    request_id = uuid.uuid4().hex[:12]
    result = run_agent(req.question,
                       max_steps  = req.max_steps,
                       session_id = req.session_id)
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

@app.exception_handler(MaxStepsExceeded)
def _max_steps(request: Request, exc: MaxStepsExceeded) -> JSONResponse:
    return JSONResponse(status_code=504, content={
        "detail":      "agent did not converge within step cap",
        "error_class": "max_steps_exceeded",
    })

@app.exception_handler(ProviderTimeout)
def _provider_timeout(request: Request, exc: ProviderTimeout) -> JSONResponse:
    return JSONResponse(status_code=504, content={
        "detail":      "upstream model timed out",
        "error_class": "provider_timeout",
    })

@app.exception_handler(ProviderBadRequest)
def _provider_bad_request(request: Request, exc: ProviderBadRequest) -> JSONResponse:
    return JSONResponse(status_code=502, content={
        "detail":      "upstream provider rejected the request",
        "error_class": "provider_bad_request",
    })

@app.exception_handler(Exception)
def _unexpected(request: Request, exc: Exception) -> JSONResponse:
    return JSONResponse(status_code=500, content={
        "detail":      "internal server error",
        "error_class": "unexpected",
    })
```

Run with:

```bash
pip install "fastapi[standard]" pydantic uvicorn anthropic
export ANTHROPIC_API_KEY=sk-ant-...
fastapi dev src/main.py
```

## Things you should not do in this exercise

- Read secrets from a settings object. That is exercise-02. For now, `os.environ` once at startup is fine.
- Add logging beyond `print()`. That is exercise-03.
- Add a guardrail. Also exercise-03.
- Wrap the route handler in `try/except`. Let the exception handlers do their job.
- Add a second endpoint (`/embed`, `/feedback`, `/agent-with-tools`) to "make the API more interesting." One endpoint is enough; the boundary is the lesson.

## Acceptance criteria

You can demonstrate:

- A working `fastapi dev src/main.py` that serves `/healthz` and `/chat`.
- `runs.md` capturing the eight smoke-test requests above with their actual responses and status codes.
- `NOTES.md` containing the three paragraphs from task 6.
- An `openapi.json` that, when piped through `jq`, shows the `ChatRequest` constraints (`min_length`, `max_length`, `ge`, `le`).
- A handler in `main.py` whose `def chat(...)` body is under ~15 lines and contains no `try/except`.

## Reflection

Answer briefly in `NOTES.md`:

1. Pydantic rejected test 5 (`max_steps=99`) before the handler ran. If you had instead validated this inside the handler, name two ways the bug could have surfaced later (in a refactor, in a test, in production) that Pydantic's rejection avoids.
2. Suppose `ChatResponse` had been a plain `dict` returned from the handler. What would `response_model=ChatResponse` have stopped you from accidentally leaking into the wire?
3. The `Exception` catch-all returns `500` with a generic message. Sketch one specific exception (not in your current map) that *should* not be a `500` — name what status code it should be and why.

## Stretch goals

- **Async port.** Switch the agent to `anthropic.AsyncAnthropic` (or `openai.AsyncOpenAI`) and the handler to `async def chat(...)`. Make sure you no longer call any blocking I/O in the handler. Note the latency difference under three concurrent `curl`s.
- **Add `/chat/stream`.** Use Server-Sent Events (`fastapi.responses.StreamingResponse`) to stream incremental tokens to the client as the agent emits them. This is one of the highest-leverage UX changes you can make for an LLM service; it's also where async stops being optional.
- **A static API-key header.** Add `Security(APIKeyHeader(name="X-API-Key"))`, compare with `hmac.compare_digest` against an `APP_API_KEY` env var, return `401` otherwise. Worth doing once to feel the shape; production auth is bigger than this.
- **Property tests.** Use `hypothesis` to generate arbitrary JSON bodies and assert that every response is one of `{200, 422, 504, 502, 500}` — never a 200 with malformed body. This catches a surprising number of validation bugs.
