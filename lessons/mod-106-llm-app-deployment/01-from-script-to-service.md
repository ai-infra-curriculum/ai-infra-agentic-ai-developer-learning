# From Script to Service

## Why this matters

Everything you built in `mod-101`–`mod-105` ran as a script. You hit a Python file, it called the model, it printed an answer, it exited. The code on disk was the code that ran. The user — you — sat at the terminal and waited.

The moment another person needs to use your app — a teammate, a frontend, a teammate's frontend, a cron job, a customer — that shape stops working. The model call doesn't change. Everything *around* it changes: who invokes it, how many at once, where the secrets live, what happens when a call takes 30 seconds instead of two, what happens when a malformed request lands at 3 a.m.

This module is the smallest set of changes that turn the script into something a second person can use. This chapter draws the boundary between the two worlds so the rest of the chapters have a target.

## The script you have today

A typical end-of-`mod-105` driver looks like:

```python
# run.py
from agent import run_agent

if __name__ == "__main__":
    question = "Where is order ORD-1001?"
    result = run_agent(question)
    print(result["final_text"])
```

Properties of this shape, mostly invisible because they are defaults:

- **One caller.** You. No concurrency story is needed because there is no concurrency.
- **One process per request.** Python starts, the script runs, the script exits. State is born and dies with the process.
- **Secrets in your shell.** Your `ANTHROPIC_API_KEY` lives in `~/.bashrc` or an `export` you typed; the script reads it from `os.environ`.
- **Output to stdout.** Errors blow up the process; success prints text.
- **No deadline.** If the model takes 90 seconds, you wait 90 seconds. Nothing else is waiting on you.
- **No budget.** If your loop spins, you watch the dashboard and Ctrl-C when you notice.

None of those defaults are wrong. They are exactly right for the development loop you are still in. They are also exactly the assumptions that break the first time a second person needs the app.

## The service you want next

A "service" in this module is the smallest thing that does *not* break those assumptions when a second caller arrives:

- **An HTTP endpoint.** `POST /chat` accepts a JSON body, returns a JSON body. Anyone with `curl` can drive it.
- **A long-running process.** The Python interpreter starts once, serves many requests, and stays up between them. State that should outlive a request lives in the process; state that should *not* outlive a request stays in the request handler.
- **Validated input.** Pydantic models reject malformed bodies with a `422` *before* the model is called. Bad input never reaches the LLM.
- **Configured, not coded.** API keys, model ids, spend caps, and feature flags come from environment variables read at startup. You change behavior without touching the code.
- **Observed.** Every request produces a structured log line: who called, what model, how many tokens, how long it took, did it succeed.
- **Bounded.** Per-request token caps and a per-process daily-spend ceiling stop the runaway loop before it stops you.
- **Packaged.** A `Dockerfile` produces an image any machine can run; a teammate runs the same bytes you do.

The vocabulary here — endpoint, validation, configuration, observation, bound, package — is the rest of the module. Each chapter takes one of those words and makes it concrete.

## What stays the same

It is worth being clear that the actual *LLM call* does not change. `client.messages.create(...)` is the same line of code in the script and in the service. The agent loop from `mod-105` is the same loop. The retriever from `mod-104` is the same retriever.

A clean way to picture it:

```
+----------------------------+
|   HTTP layer (FastAPI)     |
|   - parse JSON             |
|   - validate (Pydantic)    |
|   - return JSON / error    |
+-------------+--------------+
              |
              v
+----------------------------+
|   Application code         |
|   - run_agent(question)    |   <-- this is the code you already wrote
|   - search_kb / tools      |
+-------------+--------------+
              |
              v
+----------------------------+
|   Provider call            |
|   - client.messages.create |
+----------------------------+
```

The HTTP layer is *thin*. It is a function-call boundary with a few extra responsibilities (validation, error mapping, logging, headers). It is not where business logic lives. If you find yourself writing prompt construction inside a route handler, push it back down into the application layer; FastAPI is not the place for that.

## What changes when concurrency shows up

The single most surprising property of a service vs. a script is concurrency. A script never has to think about it. A service is *always* either single-threaded with one in-flight request or fielding several at once.

Two things that bite immediately:

1. **Module-level state is shared.** If you keep a dict at module scope — say, `THERMOSTAT_LOG = []` from `mod-105` — every request sees and mutates the same list. That is sometimes what you want (a counter, a cache) and sometimes a bug (a "current user" that leaks between requests). Be deliberate. State that belongs to *one request* lives inside the handler. State that belongs to *the process* lives at module scope.

2. **The model is slow.** A `messages.create` call takes 1–10 seconds, sometimes more. If your server is single-threaded and one request blocks for 8 seconds, a request that arrives in second 2 waits 6 seconds before it even starts. FastAPI's solution is `async def` handlers and `await`ing the provider's async client. You will use the synchronous client for clarity in chapter 2 and switch to async in chapter 7's stretch goal; both work. Just know which one you're running.

You will not write a thread pool or a queue in this module. The default uvicorn worker model — one process, async event loop — is enough for a single-developer service. Multi-worker, multi-process, queue-backed designs are Operator territory.

## What changes when secrets show up

In the script you typed `export ANTHROPIC_API_KEY=sk-ant-...` once and forgot about it. In a service that ships to another machine, three things change:

1. **The secret can't be in the image.** A Docker image built once and shared is a public-ish artifact even when it's private. Bake your key into the image and it's only one accidental `docker push` away from a public registry. Chapter 3 enforces "the image has no secrets; the *running container* has secrets injected at start time."

2. **A missing secret should be a startup failure, not a runtime crash.** If `ANTHROPIC_API_KEY` is unset and your first `POST /chat` is the one that finds out, you're returning a `500` to a user whose request never had a chance. Detect missing config at startup and refuse to serve — `pydantic-settings` makes this a one-liner.

3. **You will have more than one.** API keys, model ids, spend caps, log levels, debug flags. A central `Settings` object is the right shape; scattered `os.environ.get(...)` calls are the wrong one.

Chapter 3 is the whole story. The point here is just that the configuration boundary is real and has a name.

## What changes when observation shows up

In the script you saw what was happening because you were watching the terminal. In a service you don't see anything unless you log it.

The minimum useful observation per request is one structured log line carrying:

- A request id (so a user complaint maps to a row in your log).
- The endpoint and method.
- The model called.
- Input and output token counts.
- The latency (start to finish).
- Whether it succeeded and, if not, what the failure class was.

That single line is enough to answer most production questions: "Why was yesterday's bill so high?" (sum the tokens column). "Did the new prompt slow things down?" (latency median by day). "Is this user being abusive?" (request count by IP).

Chapter 4 walks through the fields, the JSON shape, and — most importantly — what *not* to log. Full prompts and PII are tempting and dangerous; the chapter draws the line.

## What changes when budget shows up

In the script the budget guard is the keyboard: you Ctrl-C when the loop looks wrong. In a service nobody is watching.

Two layers in this module:

1. **Per-request limits.** `max_steps` from `mod-105` is already half of this; the other half is a hard `max_tokens` cap on the model call, plus an upper bound on the request body's input size. A single request cannot cost more than $X.
2. **A process-level daily-spend ceiling.** A simple counter at module scope: total dollars spent today by this process. When it crosses a configured cap, every subsequent request returns `429 Too Many Requests` with a structured body. No human needs to be awake.

This is not a substitute for real platform-level rate limiting and budget alerts. It is the *floor* of safety so a runaway agent doesn't ruin a weekend before the platform layer catches it. Chapter 5 makes it concrete.

## What changes when packaging shows up

In the script `python run.py` is the install instruction. On a teammate's machine that breaks the first time: their Python is a different version, they don't have the right packages, they need to source a venv, the venv was created on macOS and they're on Linux, the `.env` file isn't there. Every "works on my machine" story starts here.

Docker is the smallest fix. A `Dockerfile` declares the exact base image, the exact dependencies, the exact entrypoint. `docker build` produces an artifact. `docker run` starts it. The same bytes run on the laptop, the CI runner, the staging host. Chapter 6 writes the smallest Dockerfile that does this honestly — multi-stage, no `.env` baked in, healthcheck wired up.

You will not deploy the image anywhere. The goal is "runs locally in Docker, can be handed to a teammate." Going further — Compose, registries, Kubernetes — is later.

## A working definition

For the rest of this module, a "service" means:

> A long-running Python process that exposes an HTTP API for an LLM application, reads its configuration and secrets from environment variables, validates and logs every request, enforces per-request and per-process spend limits, and ships as a single Docker image that runs the same way locally as anywhere else.

That is the bar. Nothing in the definition mentions a load balancer, a database, or a CDN. Those exist; they are not in this module.

## A note on FastAPI

FastAPI is one of several reasonable choices (Starlette, Flask, Litestar, Django REST Framework, even raw `uvicorn` + an ASGI app). This module picks FastAPI for three concrete reasons:

1. **Pydantic is first-class.** You already use Pydantic for tool-argument validation in `mod-103` and structured outputs in `mod-102`. The same type ends up validating an HTTP request body in chapter 2. No new vocabulary.
2. **OpenAPI/JSON Schema for free.** Your endpoints render a self-describing schema at `/openapi.json` and an interactive UI at `/docs`. For a teammate to drive your app, they read the docs page and start sending requests.
3. **It is what the ecosystem reaches for.** Most LLM gateway and agent-serving examples in the wild — vLLM's API server, Anthropic/OpenAI cookbook deployment recipes, popular open-source agent frameworks — are FastAPI. Reading other people's code becomes easier.

If you have strong reasons to use Flask, Litestar, or anything else for this exercise, the chapters translate almost line-for-line. Pick what you'll keep using; the boundaries the module enforces are framework-agnostic.

## Summary

A script runs once, prints, and exits. A service is a process that fields many requests, validates them, runs against secrets it didn't write down on disk, logs what happened, refuses to spend more than its budget, and ships as a single image. The model call in the middle is the same call you already wrote. The rest of the module makes the surface around it production-shaped — at a single-developer scale, not a platform-team scale. The next chapter builds the smallest version of that surface in FastAPI.
