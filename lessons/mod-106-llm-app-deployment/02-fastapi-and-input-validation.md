# FastAPI and Input Validation

## Why this matters

The HTTP layer is the place where the outside world meets your LLM app. Two things happen at that boundary that did not happen in your script:

1. **You don't trust the input.** Anyone with `curl` can send a body of any shape. Without a validator, garbage propagates inward until the model call crashes, the agent loop wedges on a missing field, or — worse — the model dutifully tries to answer "null" because you forgot to check.
2. **You owe the caller a response.** A script that crashes prints a stack trace. A service that crashes returns `500 Internal Server Error` to a user who has no idea what to do with it. Every failure mode needs a status code, a body, and a log line.

This chapter writes the smallest FastAPI app that does both: a `POST /chat` endpoint with a Pydantic-validated request and response, a `GET /healthz`, and an exception handler that maps the failure modes of your LLM call onto sensible HTTP responses.

## The smallest useful app

A 30-line `main.py` that gets us moving:

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="support-agent", version="0.1.0")

class ChatRequest(BaseModel):
    question: str

class ChatResponse(BaseModel):
    answer: str

@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    # Stand-in for the real call. Chapter 7 wires the agent in.
    return ChatResponse(answer=f"You asked: {req.question}")
```

Run it:

```bash
pip install "fastapi[standard]"
fastapi dev main.py    # or: uvicorn main:app --reload
```

Hit it:

```bash
curl -s localhost:8000/healthz
# {"status":"ok"}

curl -s -X POST localhost:8000/chat \
    -H "content-type: application/json" \
    -d '{"question":"hi"}'
# {"answer":"You asked: hi"}
```

That is the whole shape. Three responsibilities sit at the boundary: declare a request model, declare a response model, write a handler that goes from one to the other. Everything in the rest of this chapter is making each of those three lines stricter, more honest, or more observable.

## Pydantic request models, in practice

The `ChatRequest` above is the bare minimum. A more realistic one for an LLM endpoint:

```python
from pydantic import BaseModel, Field
from typing import Annotated

class ChatRequest(BaseModel):
    question: Annotated[str, Field(min_length=1, max_length=4000)]
    session_id: Annotated[str | None, Field(default=None, max_length=64)]
    max_steps: Annotated[int, Field(default=8, ge=1, le=12)]
```

What each line is buying:

- `min_length=1` rejects `{"question": ""}` with a `422` before the model is called. Empty prompts are a surprisingly common bug source.
- `max_length=4000` enforces an upper bound on input size at the HTTP boundary. Without it a 2 MB body is a free path to a huge token bill. The number is per-app; it should be small enough that no real user hits it and large enough that no real user is rejected. Start low and raise on feedback.
- `session_id` is optional but bounded; an attacker who supplies a 200 KB session id is asking you to use it as a cache key.
- `max_steps` is exposed but clamped to the range your agent loop will actually accept. The handler does not need to defensively re-clamp; Pydantic guarantees a value in `[1, 12]`.

When a caller sends a request that violates any of these, FastAPI returns `422 Unprocessable Entity` with a body that points at the offending field:

```bash
curl -s -X POST localhost:8000/chat \
    -H "content-type: application/json" \
    -d '{"question":""}'
# {"detail":[{"type":"string_too_short","loc":["body","question"],
#             "msg":"String should have at least 1 character", ...}]}
```

You did not write a single line of validation code. The model declaration *is* the validator. Spend your validation effort on naming the constraints that matter — *what* the input is allowed to be — and let the framework enforce them.

> The fields that show up here (`question`, `session_id`, `max_steps`) are the smallest set that lets the rest of the module's chapters work without backtracking. Your real app will add fields (user id, language, tools to allow, model override). Add them as Pydantic fields. Resist the urge to accept `**kwargs` or a free-form `dict`; the moment you do, every downstream component has to re-validate.

## Pydantic response models, in practice

Declaring `response_model=ChatResponse` is doing more than typing. FastAPI:

- Serializes whatever your handler returns through `ChatResponse`, dropping fields you didn't declare. If your `run_agent(...)` happens to return `{"final_text": ..., "trace": ..., "debug": ...}`, only `answer` survives on the wire. You don't accidentally leak `trace` to production.
- Errors at startup if your handler's return type doesn't match the declared model — a refactor that drops a field surfaces immediately.
- Emits a JSON Schema entry into `/openapi.json` so a client can codegen against it.

A realistic response carries a small amount of metadata:

```python
class ChatResponse(BaseModel):
    answer: str
    session_id: str
    request_id: str
    usage: "Usage"

class Usage(BaseModel):
    input_tokens: int
    output_tokens: int
    steps: int
```

`Usage` is the same data you log in chapter 4 — surfacing it on the response lets the caller debug their own request without grepping your server logs. (You can hide it behind a `?debug=true` query if you don't want it in the contract.)

## The interactive docs page

FastAPI mounts `/docs` (Swagger UI) and `/redoc` (ReDoc) by default. They render the `/openapi.json` schema your Pydantic models generated. Open `localhost:8000/docs` and you have a self-describing API:

- See every route and its method.
- Read each request and response schema.
- Click "Try it out", fill the form, see the actual response.

This is the closest thing you get to "documentation that can't lie." If your handler signature changes and you forget to update the README, the docs page is still right. Lean on it during exercise-01.

(For a production service you typically *disable* `/docs` in non-development environments — chapter 3 covers the config switch. The information that is wholesome for a teammate is also useful for someone enumerating your endpoints.)

## Health checks

A health check is a tiny endpoint that returns "I'm alive" to anyone asking — typically a load balancer, an orchestrator, or a teammate poking at a Docker container to see if startup actually finished.

```python
@app.get("/healthz")
def healthz() -> dict[str, str]:
    return {"status": "ok"}
```

The single biggest mistake here is making the health check do work. *Do not* call the model in `/healthz`. Do not hit the database. Do not load a large file. The health check is supposed to answer "is this process up" in microseconds. If it depends on a downstream system you don't control, every blip downstream takes you down too — and the failure cascade has nothing to do with the actual health of your code.

A separate `/readyz` (for "am I ready to *serve*," typically meaning startup config and one-time setup completed) is conventional but optional at this scale. Chapter 6 returns to healthchecks when we wire one into the Dockerfile.

## Error handling, deliberately

In a script, an exception ends the script. In a service, an exception ends *the request*, and the framework returns *something* — by default, a `500 Internal Server Error` with a generic body.

FastAPI gives you a layered approach. From cheapest to richest:

### 1. Raise `HTTPException` from inside the handler.

For things you can detect synchronously:

```python
from fastapi import HTTPException

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    if req.question.lower() in {"ping", "test"}:
        raise HTTPException(status_code=400, detail="ping not allowed")
    ...
```

FastAPI turns the exception into `{"detail": "ping not allowed"}` with a `400` status code. Use this for any precondition you can check before — or right after — the request body lands.

### 2. Register an exception handler for application errors.

Your agent loop's known failure modes deserve named status codes. From `mod-105` chapter 3:

```python
class MaxStepsExceeded(RuntimeError): ...
class ProviderTimeout(RuntimeError): ...
class GuardrailTripped(RuntimeError): ...   # chapter 5
```

Map them centrally:

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(MaxStepsExceeded)
def handle_max_steps(request: Request, exc: MaxStepsExceeded):
    return JSONResponse(status_code=504, content={
        "detail": "agent did not converge within step cap",
        "error_class": "max_steps_exceeded",
    })

@app.exception_handler(ProviderTimeout)
def handle_timeout(request: Request, exc: ProviderTimeout):
    return JSONResponse(status_code=504, content={
        "detail": "upstream model timed out",
        "error_class": "provider_timeout",
    })

@app.exception_handler(GuardrailTripped)
def handle_budget(request: Request, exc: GuardrailTripped):
    return JSONResponse(status_code=429, content={
        "detail": "daily usage cap reached",
        "error_class": "guardrail_tripped",
    })
```

Each handler owns one decision: what status code, what error class. The handlers stay shallow because the *raising* lives in the application code — your route handler does not have a fistful of `try/except` blocks.

A rough working map between failure mode and HTTP status:

| Failure                          | Status | Why                                                |
|----------------------------------|--------|----------------------------------------------------|
| Malformed body (Pydantic)        | `422`  | Caller's fault, can be fixed by caller             |
| Missing required header / auth   | `401`  | Caller didn't authenticate                         |
| Authenticated but not allowed    | `403`  | Caller authenticated, lacks permission             |
| Validation passed, business rule | `400`  | "ping not allowed" etc.; caller's mistake          |
| Provider 4xx (e.g. bad arg)      | `502`  | Your handler made a bad upstream request           |
| Provider 5xx                     | `502`  | Upstream is down                                   |
| Provider timeout                 | `504`  | Upstream took too long                             |
| Agent exceeded `max_steps`       | `504`  | Gateway-style "deadline hit"                       |
| Internal guardrail (daily cap)   | `429`  | Caller is asking too much (or many callers are)    |
| Unhandled exception              | `500`  | Programmer error; fix it and ship                  |

Use these as a starting point, not gospel; the only rule that matters is that two requests with the same failure should always get the same status code, and that code should be derivable from a log entry without reading the call site.

### 3. The catch-all.

Anything you didn't anticipate becomes `500`. Make the body honest but blank:

```python
@app.exception_handler(Exception)
def handle_unexpected(request: Request, exc: Exception):
    # Log the traceback — chapter 4 — then return a vague body.
    return JSONResponse(status_code=500, content={
        "detail": "internal server error",
        "error_class": "unexpected",
    })
```

Two reasons not to include the traceback in the body:

- The traceback may leak file paths, dependency versions, or worse — your prompt template.
- It's noise the caller can't act on. Your logs are where the traceback belongs.

## The call site: keep the handler thin

A common smell is letting the route handler become "the agent." Resist. The handler does five things:

1. Accept a Pydantic-validated request.
2. Stamp a `request_id`.
3. Call into the application layer (`run_agent(...)`).
4. Build a Pydantic response.
5. Return.

Anything else — prompt construction, tool dispatch, retries, logging, cost accounting — lives one layer down. The chapter-7 final shape looks like:

```python
@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest) -> ChatResponse:
    request_id = uuid.uuid4().hex
    result = run_agent(req.question, max_steps=req.max_steps,
                       session_id=req.session_id, request_id=request_id)
    return ChatResponse(
        answer=result["final_text"],
        session_id=req.session_id or request_id,
        request_id=request_id,
        usage=Usage(
            input_tokens=result["usage"]["input_tokens"],
            output_tokens=result["usage"]["output_tokens"],
            steps=result["steps"],
        ),
    )
```

That is the whole handler. Five named operations, no `try/except` (errors are handled centrally), no business logic. When the next chapter adds settings and the chapter after that adds logging, the handler grows by a few lines, not a few sections.

## Sync vs. async, briefly

FastAPI accepts both `def` and `async def` handlers. The difference matters because your `messages.create(...)` call may take seconds.

- **Sync handler.** FastAPI runs your function in a threadpool. Multiple requests can be in flight, but each one occupies a thread for its whole lifetime. Easy to reason about; what you'll write in exercise-01.
- **Async handler.** Your function `await`s an async client (`anthropic.AsyncAnthropic`, `openai.AsyncOpenAI`). One event-loop thread handles many in-flight requests. Better throughput at the cost of one new concept (you can't `time.sleep` in an async handler).

For a single-developer service, both are fine. Pick sync for chapter-2 to chapter-6; chapter-7's stretch goal is the async port. *Do not mix*: a sync function calling `asyncio.run(...)` inside a route handler is the wrong shape and will eat the event loop. (FastAPI's docs go through the rules; the resources page has the link.)

## A word on request size and rate limits

Two safeguards belong at the framework level rather than inside your handler:

- **Body size limits.** A 2 MB JSON body is rejected before it reaches your handler if your reverse proxy or the framework caps it. uvicorn doesn't enforce a default body size cap; in production you'd put a reverse proxy in front. For this module, the Pydantic `max_length` on `question` is the meaningful guard.
- **Rate limiting.** "More than N requests per minute from one caller" is the kind of policy that belongs in a middleware (e.g. `slowapi`) or — in production — a platform layer. You will not add rate limiting in this module beyond the per-process daily cap in chapter 5. Naming it here so you know it's missing.

## What you have at the end of this chapter

A FastAPI app that:

- Exposes `GET /healthz` and `POST /chat`.
- Validates request bodies with a Pydantic model that enforces field lengths and ranges.
- Returns responses serialized through a Pydantic model that won't accidentally leak fields you forgot to redact.
- Renders self-describing docs at `/docs`.
- Maps three named application errors onto sensible HTTP status codes; catches the rest as `500`.
- Has a route handler whose body is five operations long.

That is the floor. Chapter 3 makes the secrets honest. Chapter 4 makes the log line carry the right fields. Chapter 5 adds the guardrail. Chapter 6 packages the whole thing.

## Summary

The HTTP boundary has two jobs: keep bad input out and turn known failures into honest responses. Pydantic request and response models do the first; centralized exception handlers do the second. FastAPI is the smallest framework that gives you both with very little glue, and its Pydantic-first shape is the same Pydantic you already use for tool arguments and structured outputs in earlier modules. The route handler stays thin — five operations — and everything substantive happens in the layer below.
