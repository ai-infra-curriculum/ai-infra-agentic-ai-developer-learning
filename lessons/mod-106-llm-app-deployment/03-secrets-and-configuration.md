# Secrets and Configuration

## Why this matters

The first thing that breaks when a script becomes a service is the secret. In the script, `os.environ["ANTHROPIC_API_KEY"]` works because the key is in your shell. In a service that ships to a teammate, a CI runner, or a Docker image, the key needs to travel — but not in the code, not in the image, not in the repo. Every story of a leaked API key on a public GitHub starts with a developer who treated a secret like a configuration value.

This chapter is the secret-and-config contract for the rest of the module: where secrets live (environment variables), how the app reads them (`pydantic-settings`), what is *required* vs. *optional* (the difference between "refuses to start" and "starts with a default"), and what files exist on disk vs. what files do not.

## The twelve-factor rule, briefly

> **Store config in the environment.** The Twelve-Factor App, factor III.

The relevant claim is short: anything that varies between deploys — API keys, hostnames, ports, log levels, feature flags — lives in environment variables, not in the code or in committed config files. The reason is mechanical:

- The same image runs in dev, staging, and prod. Only the env differs.
- Adding a new environment doesn't require a code change or a redeploy.
- Secrets can be injected by a platform (Kubernetes secrets, AWS Parameter Store, Doppler, 1Password) without the app caring.

You will follow this rule in this module not because you have a multi-environment platform but because the moment you do, the only safe shape is the one that was always env-driven from day one.

Reference: <https://12factor.net/config>

## Three categories, named clearly

It pays to name what you are reading from the environment, because the categories have different rules.

| Category               | Examples                                       | Required at startup? | Logged?  | In `.env.example`? |
|------------------------|------------------------------------------------|----------------------|----------|--------------------|
| **Secrets**            | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`          | Usually yes          | Never    | Placeholder only   |
| **Configuration**      | `MODEL`, `MAX_STEPS`, `DAILY_USD_CAP`, `PORT`  | No (have defaults)   | Yes      | Real default       |
| **Environment flags**  | `ENV` (`dev`/`prod`), `DEBUG`, `DOCS_ENABLED`  | No (have defaults)   | Yes      | Real default       |

The point of the table is that "an env var" is not one thing. A secret has stricter rules around how it shows up in logs, in tracebacks, in `repr(settings)`, in the docs page. Mixing them in the same flat dict is fine for code; mixing them in your *head* is how you end up logging an API key.

## The `Settings` object

Centralize. *Do not* sprinkle `os.environ.get("ANTHROPIC_API_KEY")` across the codebase. One object, read at startup, validates types, fails loud, and is imported wherever needed.

```python
# settings.py
from functools import lru_cache
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="forbid",     # unknown env vars are a typo, not a feature
    )

    # --- secrets ---
    anthropic_api_key: SecretStr = Field(alias="ANTHROPIC_API_KEY")

    # --- configuration ---
    model:            str = Field(default="claude-sonnet-4-6",
                                  alias="MODEL")
    max_steps:        int = Field(default=8,  ge=1, le=12, alias="MAX_STEPS")
    daily_usd_cap:    float = Field(default=5.0, ge=0,    alias="DAILY_USD_CAP")
    request_timeout_s: int = Field(default=30, ge=1,      alias="REQUEST_TIMEOUT_S")

    # --- environment ---
    env:           str  = Field(default="dev", alias="ENV")
    log_level:     str  = Field(default="INFO", alias="LOG_LEVEL")
    docs_enabled:  bool = Field(default=True,   alias="DOCS_ENABLED")

@lru_cache
def get_settings() -> Settings:
    return Settings()  # validates at first call; raises on missing/bad
```

Three things in this snippet that are doing real work:

- **`SecretStr`.** A wrapper that hides the value from `repr()` and most logging frames. `print(settings)` shows `anthropic_api_key=SecretStr('**********')`. You retrieve the actual value with `settings.anthropic_api_key.get_secret_value()` — an explicit verb that is grep-able. Use `SecretStr` for *every* secret.
- **`extra="forbid"`.** If somebody types `ANTROPIC_API_KEY` (note the typo) in `.env`, the app refuses to start. The alternative — silent acceptance — is the loudest possible foot-gun.
- **`@lru_cache` on `get_settings`.** The first call validates and either returns a frozen settings object or raises. Every subsequent call returns the same cached instance. Tests can clear the cache (`get_settings.cache_clear()`) to inject overrides.

Reference: pydantic-settings docs — <https://docs.pydantic.dev/latest/concepts/pydantic_settings/>

## Required vs. optional

The single most consequential design choice in `Settings`:

```python
anthropic_api_key: SecretStr = Field(alias="ANTHROPIC_API_KEY")  # required
model:            str = Field(default="claude-sonnet-4-6", ...)   # optional
```

A field with no `default=` is *required*. If the env var is missing at startup, Pydantic raises `ValidationError` and the process dies before it serves a single request. That is the correct behavior — a service that boots fine and only finds out the API key is missing on the first request returns `500` to a real caller and confuses three downstream observers.

A field with a `default=` is *optional* — the env var is allowed to be absent, the default takes over, and the value is logged at startup so you can see what got used.

Decide for each field which it is. A useful heuristic: "would I rather discover this is wrong in one second at startup, or one day at a customer report?" Required is the former; defaults are the latter.

## `.env` files

A `.env` file is a flat list of `KEY=VALUE` lines that `pydantic-settings` loads automatically when the process starts:

```bash
# .env  (local development; NEVER committed)
ANTHROPIC_API_KEY=sk-ant-api03-...real-key...
MODEL=claude-sonnet-4-6
DAILY_USD_CAP=2.0
LOG_LEVEL=DEBUG
ENV=dev
```

It exists for one reason: convenience during local development. You do not want to type `export ANTHROPIC_API_KEY=...` before every shell session, and you do not want to bake the key into your code. `.env` lives next to your code, gets loaded automatically when uvicorn starts, and is read by `pydantic-settings`.

Two rules:

1. **`.env` is never committed.** Add it to `.gitignore` as the first line of work in this module. If you have already committed one, the key is compromised — rotate it, then delete the file from history.
2. **`.env.example` is committed.** It mirrors the structure of `.env` with no real secrets. Anyone cloning the repo runs `cp .env.example .env` and fills in placeholders.

```bash
# .env.example  (committed; lives next to README)
ANTHROPIC_API_KEY=replace-me-with-your-anthropic-key

# Optional — defaults are fine to start
# MODEL=claude-sonnet-4-6
# DAILY_USD_CAP=5.0
# LOG_LEVEL=INFO
# ENV=dev
```

The `.env.example` is a contract. It enumerates every variable the app reads (and only those — `extra="forbid"` keeps the two in sync). A new contributor sees the file, knows what they need, fills in the secret, and the app boots.

`.gitignore` minimum:

```
.env
.env.local
*.env.local
```

This module never asks you to commit a real key. If you do, treat it as a compromise.

## Where secrets actually come from

`.env` is one source. In a real deployment there are usually two or three:

- **Local development.** `.env` file loaded by `pydantic-settings`. Convenient, ephemeral, scoped to your laptop.
- **CI.** Encrypted secrets (`secrets.ANTHROPIC_API_KEY` in GitHub Actions, equivalent in GitLab CI) injected into the runner's env. The CI run sees the same env vars as your dev shell.
- **Production / staging.** A platform secret manager: AWS Parameter Store, Kubernetes Secrets, Vault, Doppler, 1Password, etc. The orchestrator reads from the manager and presents env vars to the container. The container sees no difference from your laptop.

Notice the pattern: every layer presents the secret as an *environment variable* to the running process. The code never changes. You ship one Docker image that reads `ANTHROPIC_API_KEY` from the env, and the platform supplies it. Chapter 6 makes this explicit when we write the Dockerfile.

You will not stand up a secret manager in this module. The contract is the only thing that matters here: the running process reads from the env, the source of those env vars is platform-shaped, and `.env` is the laptop-only convenience.

## Reading the setting from a handler

A `Settings` object is useful everywhere, but the cleanest shape in FastAPI is dependency injection:

```python
from fastapi import Depends, FastAPI
from settings import Settings, get_settings

app = FastAPI()

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings: Settings = Depends(get_settings)) -> ChatResponse:
    api_key = settings.anthropic_api_key.get_secret_value()
    ...
```

Three benefits:

- **Tests inject a fake.** `app.dependency_overrides[get_settings] = lambda: test_settings`. No monkey-patching `os.environ`.
- **The handler is explicit about what configuration it reads.** A grep for `get_settings` finds every consumer.
- **`lru_cache` ensures the validation runs once.** First request validates; subsequent requests get the cached instance.

Settings used by the application layer (your agent loop) can read the same `get_settings()` directly without `Depends`. The dependency wiring is a FastAPI convenience, not a hard rule.

## A startup sanity check

The cheapest "is this configured right" signal you can add is a structured log line at startup:

```python
import logging

@app.on_event("startup")   # FastAPI lifespan APIs also work; both are fine here
def log_settings():
    s = get_settings()
    logging.info("startup", extra={
        "env":           s.env,
        "model":         s.model,
        "max_steps":     s.max_steps,
        "daily_usd_cap": s.daily_usd_cap,
        "log_level":     s.log_level,
        "docs_enabled":  s.docs_enabled,
        "anthropic_api_key_set": bool(s.anthropic_api_key.get_secret_value()),
    })
```

Note the last line: log *whether* the key is set, never the key itself. "API key set: True" is a tremendous time-saver and reveals nothing.

`@app.on_event("startup")` is the simpler hook; FastAPI's newer `lifespan` context manager is the recommended path in current releases. Pick one — both work for this module.

## What changes per environment

A useful list of things that vary between `ENV=dev` and `ENV=prod`:

| Variable           | Dev      | Prod                                           |
|--------------------|----------|------------------------------------------------|
| `LOG_LEVEL`        | `DEBUG`  | `INFO`                                         |
| `DOCS_ENABLED`     | `true`   | `false` (don't expose `/docs` publicly)        |
| `DAILY_USD_CAP`    | `2.0`    | platform-set, often higher                     |
| `MODEL`            | cheaper  | tuned per workload                             |
| `ANTHROPIC_API_KEY`| a dev key| a separate prod key with prod budget alerts    |

The shape is the same on both sides. Only the values differ. That is the property that makes the twelve-factor rule worth following: the same image runs everywhere.

You can branch on `settings.env` for environment-specific behavior, but be sparing — every `if settings.env == "prod"` branch is a path that doesn't get exercised in dev:

```python
app = FastAPI(
    title="support-agent",
    docs_url="/docs" if settings.docs_enabled else None,
    redoc_url="/redoc" if settings.docs_enabled else None,
)
```

This is the right shape: the *config* drives whether docs are mounted. Not `if settings.env == "prod": disable docs`. The flag is named for what it does.

## Things to never put in `.env`

A short list of things `.env` (and config in general) is the wrong home for:

- **Anything that varies *per request*.** User id, session id, request id. Those are request-scoped, not process-scoped. They belong in the request body or a header.
- **Prompt templates and tool descriptions.** They are *code*, not config. Put them in `.py` files under version control and review them like code.
- **A list of allowed users.** That is either request-time auth (which you don't have yet) or a database table. Not an env var.
- **Credentials for other people's systems you don't actually use.** Don't pre-populate `.env.example` with placeholders for AWS, GCP, OpenAI, Anthropic, and Cohere "in case." Add a row when the app actually reads it.

## How leaks happen, briefly

Three ways the secret you carefully placed in `.env` still ends up somewhere it shouldn't:

1. **Committed by accident.** You added it before `.gitignore`, you cloned without `.env.example` and made one named `.env`, you copy-pasted it into a unit test. Mitigate: pre-commit hooks (e.g. `gitleaks`, `trufflehog`), and rotate immediately if a key ever shows up in `git log -p`.
2. **Logged by accident.** A `print(settings)` or a traceback that includes the value. Mitigate: `SecretStr`, never `print()` settings, and a structured log layer (chapter 4) that filters known sensitive fields.
3. **Baked into the image.** A `COPY .env .` in your Dockerfile or a build step that reads the local `.env` and bundles it. Mitigate: chapter 6's `.dockerignore` excludes `.env` and the entrypoint reads from the environment at runtime, never from a file inside the image.

The first one is the most common. The second is the most embarrassing. The third is the most expensive.

## A note on auth (briefly, since you'll be asked)

This module's service is unauthenticated. Anyone with the URL can call `POST /chat`. That is fine for a laptop service behind a firewall; it is not fine for the internet.

The minimum production add is a single static API key compared in a constant-time function (FastAPI's `Security(APIKeyHeader(...))` does this), with the expected key read from `settings.app_api_key: SecretStr`. The pattern is small enough to write in five lines. The reason it is not in this module is that one static shared key is *better than nothing and worse than real auth*; the lesson "don't ship a public endpoint without auth" deserves more than five lines, and that conversation belongs in the AI Engineer track.

If exercise-01 lands and you want to extend, the `Security` recipe in FastAPI's docs is the right starting point. <https://fastapi.tiangolo.com/tutorial/security/api-keys/>

## Summary

Secrets and configuration live in environment variables. `pydantic-settings` reads them into a single typed `Settings` object that validates at startup and fails loud on missing required fields. `SecretStr` keeps secrets out of repr() and tracebacks. `.env` is the laptop-only convenience; `.env.example` is the committed contract. The same image runs in every environment; only the env vars differ. The right amount of code in your application layer that touches `os.environ` directly is zero — settings come from one object, read once, used everywhere.
