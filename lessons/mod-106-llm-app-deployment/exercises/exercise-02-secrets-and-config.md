# exercise-02: Secrets and Config

**Estimated effort:** 2 hours

## Objective

Take the service from exercise-01 and move every secret and tunable knob out of code, into environment variables read by a single `Settings` object. You will prove three things: the app refuses to start when a required secret is missing, the app survives a missing optional setting by using a sensible default, and no secret ever shows up in a `repr()`, a startup log line, or a traceback.

By the end you should have a `.env.example` your teammates can fill in and a service that runs identically against a real `.env`, a CI environment, or whatever a future platform team injects — without a single line of code changing.

## Background

This exercise covers material from:

- [Chapter 3 — Secrets and Configuration](../03-secrets-and-configuration.md)

## Prerequisites

- A completed exercise-01 (a working FastAPI service with `/healthz` and `/chat`).
- `pip install pydantic-settings`.
- A real API key in your shell (your dev key, same one you've used through `mod-101`–`mod-105`).

## The task

Two pieces:

1. A `src/settings.py` with a `Settings` class that declares every environment variable the app reads — typed, validated, and labeled as either secret, configuration, or environment flag.
2. A wiring pass through your service so every place that previously read `os.environ` now reads from `Settings` via `Depends(get_settings)`.

Then you will demonstrate the failure modes: missing-required, present-optional, typo-detected, secret-never-leaked.

## Tasks

### 1. Build `Settings`

In `src/settings.py`, write a `pydantic-settings` model that covers, at minimum:

```python
from functools import lru_cache
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="forbid",
    )

    # --- secrets ---
    anthropic_api_key: SecretStr = Field(alias="ANTHROPIC_API_KEY")

    # --- configuration ---
    model:             str   = Field(default="claude-sonnet-4-6", alias="MODEL")
    max_steps:         int   = Field(default=8, ge=1, le=12, alias="MAX_STEPS")
    request_timeout_s: int   = Field(default=30, ge=1, le=120, alias="REQUEST_TIMEOUT_S")

    # --- environment ---
    env:          str  = Field(default="dev",  alias="ENV")
    log_level:    str  = Field(default="INFO", alias="LOG_LEVEL")
    docs_enabled: bool = Field(default=True,   alias="DOCS_ENABLED")

@lru_cache
def get_settings() -> Settings:
    return Settings()
```

Three details to get right:

- **`SecretStr` on every secret.** The key field is `SecretStr`, not `str`.
- **`extra="forbid"`.** Mistyped env vars (`ANTROPIC_API_KEY`) should refuse to load.
- **`@lru_cache`.** First call validates; subsequent calls return the cached instance. Tests clear the cache with `get_settings.cache_clear()`.

You will add `DAILY_USD_CAP` in exercise-03; do not pre-add it here unless you want to.

### 2. Wire `Settings` through the app

Replace every `os.environ.get(...)` call with a setting read.

In `main.py`:

```python
from fastapi import Depends
from .settings import Settings, get_settings

settings = get_settings()  # validates at import; fails the process if invalid

app = FastAPI(
    title="support-agent",
    docs_url="/docs"  if settings.docs_enabled else None,
    redoc_url="/redoc" if settings.docs_enabled else None,
)

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings: Settings = Depends(get_settings)) -> ChatResponse:
    api_key = settings.anthropic_api_key.get_secret_value()
    result = run_agent(req.question, ..., api_key=api_key, model=settings.model)
    ...
```

Propagate as needed into `agent.py`. The agent constructs the provider client with the explicit `api_key=` argument rather than relying on the SDK's env-var fallback. This makes it explicit which key is used and lets a test inject a different one.

### 3. The `.env` files

Create both:

- `.env` — your local file with the real key. **Add to `.gitignore` first**, before you ever populate it.
- `.env.example` — committed, with placeholder values:

```bash
# Required
ANTHROPIC_API_KEY=replace-me

# Optional (these defaults are fine)
# MODEL=claude-sonnet-4-6
# MAX_STEPS=8
# REQUEST_TIMEOUT_S=30
# LOG_LEVEL=INFO
# ENV=dev
# DOCS_ENABLED=true
```

Verify `.gitignore` blocks `.env`:

```bash
git check-ignore -v .env
# .gitignore:1:.env       .env
```

If `git check-ignore` returns nothing, your `.gitignore` is wrong; fix it before proceeding.

### 4. The startup log line

Emit one line at startup that proves the settings were read:

```python
@app.on_event("startup")
def on_startup():
    s = get_settings()
    print({
        "event":                  "startup",
        "env":                    s.env,
        "model":                  s.model,
        "max_steps":              s.max_steps,
        "request_timeout_s":      s.request_timeout_s,
        "log_level":              s.log_level,
        "docs_enabled":           s.docs_enabled,
        "anthropic_api_key_set":  bool(s.anthropic_api_key.get_secret_value()),
    })
```

Note the last field: a boolean, not the value. This is the line the next maintainer will read at 2 a.m. when the app boots without a key; it tells them what they need to know without telling them anything they shouldn't.

(You will replace `print()` with proper JSON logging in exercise-03; for now, a print is enough.)

### 5. The failure modes

Capture each of the following in `runs.md` with the command, the exit status, and a one-line note:

1. **Required-secret-present.** `.env` has a valid `ANTHROPIC_API_KEY`. Start the app. Confirm boot. Hit `/chat`. Confirm a real answer.
2. **Required-secret-missing.** Remove `ANTHROPIC_API_KEY` from `.env`. Start the app. *Expected:* `ValidationError` at startup, the process exits non-zero. The error message mentions `ANTHROPIC_API_KEY`.
3. **Required-secret-empty-string.** `ANTHROPIC_API_KEY=` (empty). Start the app. *Expected:* the field is now an empty `SecretStr`. The startup line shows `anthropic_api_key_set: False`. The first `/chat` call returns `502` (provider rejects the empty auth).
4. **Optional-missing-uses-default.** Remove every line *except* `ANTHROPIC_API_KEY` from `.env`. Start the app. *Expected:* boot succeeds, the startup line shows the defaults.
5. **Typo detected.** Add `ANTROPIC_API_KEY=sk-...` (note the typo) to `.env`. *Expected:* `extra="forbid"` triggers a `ValidationError` at startup.
6. **Override at run time.** Without changing `.env`, run `LOG_LEVEL=DEBUG fastapi dev src/main.py`. *Expected:* the startup line shows `log_level: DEBUG` — env-var precedence over `.env`.
7. **Secret never leaks.** Cause the app to print a settings object somewhere: `print(get_settings())`. *Expected:* the output shows `anthropic_api_key=SecretStr('**********')` — never the actual key.

For each, capture in `runs.md`:

```
[2] Required-secret-missing
$ ANTHROPIC_API_KEY= fastapi dev src/main.py
... ValidationError: 1 validation error for Settings
... anthropic_api_key
...   Field required
exit: 1   # process refused to start, as intended
```

### 6. Tests

Write a small `tests/test_settings.py`:

```python
import pytest
from pydantic import ValidationError
from src.settings import Settings, get_settings

def test_required_field_missing(monkeypatch):
    monkeypatch.delenv("ANTHROPIC_API_KEY", raising=False)
    get_settings.cache_clear()
    with pytest.raises(ValidationError):
        Settings(_env_file=None)  # don't read .env in this test

def test_secret_str_redacts():
    s = Settings(ANTHROPIC_API_KEY="sk-test-key-not-real")
    assert "sk-test-key-not-real" not in repr(s)
    assert s.anthropic_api_key.get_secret_value() == "sk-test-key-not-real"

def test_typo_rejected():
    with pytest.raises(ValidationError):
        Settings(ANTHROPIC_API_KEY="x", ANTROPIC_API_KEY="oops")
```

Run with `pytest -v`. Three tests, all green.

### 7. Read what you wrote

In `NOTES.md`, short paragraphs (3–6 sentences) on each:

- **Required vs. optional.** Walk through your `Settings` field by field. Why is each field required or optional? Did any of your choices feel arbitrary, and what is the consequence if you guessed wrong?
- **The `extra="forbid"` test.** When the typo refused to load (case 5), what would have happened if `extra="allow"` was the default? Trace what the bug would have looked like in production.
- **The `SecretStr` redaction.** What is the difference between "the secret isn't in `repr()`" and "the secret isn't logged anywhere"? Name one more code path where you'd want to verify the key doesn't slip through.

## Starter guidance

Add `pydantic-settings` to your `pyproject.toml`:

```toml
[project]
dependencies = [
    "fastapi[standard]>=0.115",
    "pydantic-settings>=2.6",
    "anthropic>=0.39",
    "uvicorn>=0.30",
]
```

For the dependency injection shape:

```python
from fastapi import Depends
from .settings import Settings, get_settings

@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest,
         settings: Settings = Depends(get_settings)) -> ChatResponse:
    ...
```

In tests, override:

```python
from src.main import app
from src.settings import get_settings, Settings

def fake_settings():
    return Settings(ANTHROPIC_API_KEY="sk-test")

app.dependency_overrides[get_settings] = fake_settings
# ... run tests ...
app.dependency_overrides.clear()
```

## Things you should not do in this exercise

- Read secrets from anywhere other than environment variables (yaml config, a Vault SDK, a remote config service). Env vars are the contract; sources are the platform's problem.
- Pre-populate `.env.example` with placeholder values for providers you don't use. Add a row when the code reads it.
- Hardcode the model name in the agent code with a fallback to `os.environ`. Either it's in `Settings` or it's not configurable; pick one.
- Disable `extra="forbid"` because a third-party library set an env var. Either rename your config to avoid the collision or use a setting-specific prefix.

## Acceptance criteria

You can demonstrate:

- A `Settings` class that validates every field at startup and uses `SecretStr` for every secret.
- A `.env` in `.gitignore` and a `.env.example` checked in.
- `runs.md` with all seven failure-mode captures, including the exact command and observed behavior.
- A `tests/test_settings.py` with at least three passing tests covering missing-required, redacted-repr, and typo-rejected.
- `NOTES.md` containing the three paragraphs from task 7.
- A grep of your source confirming `os.environ` is no longer called anywhere outside `settings.py` and any test helpers.

## Reflection

Answer briefly in `NOTES.md`:

1. The `_env_file=None` trick in the test stops Pydantic from reading the actual `.env`. Why is that necessary? What would have happened in CI if you skipped it?
2. The `extra="forbid"` policy means a new optional setting from a future feature *also* refuses to load until you add it to `Settings`. Is that a feature or a bug? When would you flip it to `allow`?
3. You have one secret today. Sketch the diff for adding a second secret (e.g. an OpenAI key as a fallback provider). What changes in `Settings`, `.env.example`, the agent code, and the tests?
4. Suppose you wanted *some* settings reloaded at runtime (a hot-reload of `LOG_LEVEL` without a restart) and *others* frozen at startup (`MODEL`). Where in the codebase would you draw that line?

## Stretch goals

- **A static `APP_API_KEY` header.** Add a `SecretStr` field, wire `Security(APIKeyHeader(...))` against it with `hmac.compare_digest`, return `401` otherwise. The smallest credible auth surface.
- **Env-specific settings classes.** Split into `DevSettings`, `ProdSettings`, etc., with `Settings = {"dev": DevSettings, "prod": ProdSettings}[os.environ["ENV"]]()`. Useful when defaults *should* differ per environment.
- **Settings introspection endpoint.** `GET /_settings` returns the non-secret fields as JSON, gated on `ENV != "prod"`. A debugging convenience; intentionally rude to ship enabled.
- **Lint for `os.environ`.** Add a small `ruff` rule (or a pre-commit grep) that fails the build if `os.environ` is referenced anywhere outside `settings.py`. This is how the rule actually stays a rule.
