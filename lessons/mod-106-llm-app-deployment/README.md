# mod-106-llm-app-deployment: Shipping an LLM Application

**Estimated effort:** 10 hours

The five modules before this one taught you to build LLM apps that run on your laptop. This module turns one of them into a service: a FastAPI process you can `curl`, with Pydantic-validated requests, environment-managed secrets, structured logs that include token usage, a guardrail that stops the app before a runaway loop drains your budget, and a Dockerfile so the whole thing runs the same way on another machine. By the end you should be able to hand a teammate a Docker image, a `.env.example`, and a one-line `docker run` and have them poking at your agent in five minutes — not a Jupyter notebook in five hours.

This module is deliberately the smallest "production-shaped" surface you can put around an LLM app. It is not a deployment course. There is no Kubernetes, no autoscaler, no managed model gateway, no PagerDuty. Those belong in the AI Engineer and AI Operator tracks. What lives here is the *bare minimum* a single developer needs to understand before any of that has a hope of working: the request boundary, the secret boundary, the observability boundary, and the run boundary.

## Learning objectives

- Wrap an LLM app behind a simple API (FastAPI) with input validation.
- Manage API keys and other secrets via environment variables.
- Add basic logging and a cost/usage guardrail.
- Containerize and run the app locally.

## Lecture chapters

1. [From Script to Service](01-from-script-to-service.md) — what changes when an LLM app stops being a `__main__` and starts being a request handler: concurrency, statelessness, latency budgets, failure modes.
2. [FastAPI and Input Validation](02-fastapi-and-input-validation.md) — the smallest useful FastAPI app, request/response Pydantic models, `422` validation errors, error mapping, and the seam between your handler and your LLM call.
3. [Secrets and Configuration](03-secrets-and-configuration.md) — twelve-factor config, `.env` files, `pydantic-settings`, what to *never* commit, and what changes between dev, staging, and production without touching code.
4. [Logging LLM Requests](04-logging-llm-requests.md) — structured (JSON) logs, the fields every LLM request log should carry (request id, model, latency, tokens), and the prompt/PII boundary.
5. [Cost and Usage Guardrails](05-cost-and-usage-guardrails.md) — the "$2,500 weekend" failure mode, per-request token caps, in-process daily-spend ceilings, and per-key rate limits.
6. [Containerizing with Docker](06-containerizing-with-docker.md) — a small, multi-stage Python Dockerfile, baking the app and *not* the secrets, healthchecks, and the `docker run` you hand a teammate.
7. [Running Locally, End to End](07-running-locally-end-to-end.md) — smoke tests, what a healthy `curl` looks like, and the gap between "runs on my laptop in Docker" and "runs in production."

## Exercises

Hands-on practice. Solutions live in the paired solutions repo.

- [exercise-01: FastAPI LLM endpoint](exercises/exercise-01-fastapi-llm-endpoint.md) — wrap the agent from `mod-105` (or any LLM call from earlier modules) behind a `POST /chat` endpoint with Pydantic request/response models, a `GET /healthz`, and explicit error mapping.
- [exercise-02: Secrets and config](exercises/exercise-02-secrets-and-config.md) — move every secret and tunable knob out of code, into environment variables loaded by `pydantic-settings`. Prove the app refuses to start when a required secret is missing.
- [exercise-03: Logging and usage guardrail](exercises/exercise-03-logging-and-usage-guardrail.md) — emit one structured JSON log line per request with token usage and latency, then add a per-process daily-spend guardrail that returns `429` when tripped.

## Resources

- [resources.md](resources.md) — primary documentation and standards used by this module.

## How to work through this module

Read each chapter, then do the matching exercise. You will not need new API keys — you'll reuse the provider key from `mod-101`–`mod-105`. You *will* need Docker Desktop (or rootless Docker / Podman / colima) installed and able to build and run a Python image; if you have not used Docker before, the official "Get started" tutorial linked in `resources.md` is the right pre-read.

Pick *one* app from earlier in the track to wrap. The cleanest choices are `mod-105` exercise-02 (the three-tool support agent) or `mod-103` exercise-02 (the multi-tool app). The exercises are written against an "agent in a function" shape, but anything with a `def answer(question: str) -> str` entry point works.

`mod-105` is the hard prerequisite: you are shipping the agent you just built. If your `mod-105` exercise didn't end with a function you can call from another file, refactor before chapter 2 — every later step assumes a clean call site.

## What this module is not

It is worth naming the things this module *does not* teach, so you know where to stop and what to expect in later courses:

- **Kubernetes, autoscaling, blue/green, canary releases.** Operator track.
- **Multi-tenant authentication and tenancy isolation.** A `POST /chat` with an API key check is the ceiling here; SSO, OAuth, per-tenant rate limits, audit logs are AI Engineer / Operator.
- **Distributed tracing, dashboards, SLO definitions.** Chapter 4 emits log lines a tracer or dashboard could later ingest; it does not stand them up.
- **Model gateways (Bedrock, Vertex, Azure OpenAI, LiteLLM proxy).** You ship a single-provider service. Multi-provider routing is its own course.
- **Prompt-injection-aware request validation.** Pydantic stops malformed *shapes*; it does not stop a hostile *prompt*. Defensive prompting and red-teaming are covered separately.
- **CI/CD pipelines, secret managers (Vault, AWS SM), production secret rotation.** Chapter 3 explains the contract; the platform work is Operator.

The goal is one well-behaved service you understand top to bottom: a request comes in, validation rejects bad shapes, secrets come from the environment, every call is logged with its cost, a guardrail trips before the bill does, and the whole thing runs in a Docker container you can hand to anyone.

> Layout note: `labs/` and `quizzes/` are placeholders authored on a later cycle. The chapters and exercises above are the canonical content for this module.
