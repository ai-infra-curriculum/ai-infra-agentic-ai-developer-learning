# Resources for mod-106-llm-app-deployment

Primary documentation and standards used by this module. Prefer these over secondary write-ups: they are authoritative and they get updated as the frameworks, runtimes, and provider APIs evolve.

## FastAPI and ASGI

- **FastAPI — official documentation.** The canonical reference for everything in chapters 2 and the early parts of 7: routing, dependencies, request and response models, exception handlers, lifespan events. <https://fastapi.tiangolo.com/>
- **FastAPI — Tutorial: Body, Request Body.** Pydantic request bodies, optional vs. required fields, the `422` validation behavior used throughout chapter 2. <https://fastapi.tiangolo.com/tutorial/body/>
- **FastAPI — Tutorial: Response Model.** What `response_model=` actually does at runtime (filtering, schema generation, type-checking). <https://fastapi.tiangolo.com/tutorial/response-model/>
- **FastAPI — Tutorial: Handling Errors.** `HTTPException`, custom exception handlers, and the pattern chapter 2's error map is built on. <https://fastapi.tiangolo.com/tutorial/handling-errors/>
- **FastAPI — Tutorial: Dependencies.** `Depends(...)`, dependency overrides for tests, and the `Settings` injection shape in chapter 3. <https://fastapi.tiangolo.com/tutorial/dependencies/>
- **FastAPI — Tutorial: Middleware.** Where the "one log line per request" pattern from chapter 4 lives most naturally. <https://fastapi.tiangolo.com/tutorial/middleware/>
- **FastAPI — Async / Concurrency notes.** The sync-vs-async decision in chapter 1 and the stretch goal in exercise-01. <https://fastapi.tiangolo.com/async/>
- **Starlette — Application docs.** FastAPI is built on Starlette; the lower-level docs are useful when you need to understand request/response internals or write a custom middleware. <https://www.starlette.io/>
- **Uvicorn — official documentation.** The ASGI server you use to run the app, including signal handling, worker model, and the flags chapter 6's `CMD` line uses. <https://www.uvicorn.org/>
- **ASGI specification.** The protocol Starlette / FastAPI implement; worth reading once when you start wondering "what is the actual contract between uvicorn and my code." <https://asgi.readthedocs.io/>

## Pydantic and settings

- **Pydantic — Models.** The validation primitives behind every `BaseModel` used in this module. <https://docs.pydantic.dev/latest/concepts/models/>
- **Pydantic — Field constraints.** `min_length`, `max_length`, `ge`, `le`, and the rest of the validators chapter 2's request model relies on. <https://docs.pydantic.dev/latest/concepts/fields/>
- **Pydantic — Validators (`field_validator`, `model_validator`).** For validation that the type system alone can't express. <https://docs.pydantic.dev/latest/concepts/validators/>
- **Pydantic Settings — Settings management.** The full reference for `BaseSettings`, `SettingsConfigDict`, `env_file`, `extra=forbid`, prefix handling, and CLI integration. This is the chapter-3 canon. <https://docs.pydantic.dev/latest/concepts/pydantic_settings/>
- **Pydantic — `SecretStr` and `SecretBytes`.** The two wrappers chapter 3 uses to keep secrets out of `repr()` and most log frames. <https://docs.pydantic.dev/latest/api/types/#pydantic.types.SecretStr>

## Twelve-factor and config hygiene

- **The Twelve-Factor App.** Particularly factors III (Config), XI (Logs), and XII (Admin Processes), which underpin chapters 3, 4, and the operational expectations throughout this module. <https://12factor.net/>
- **Twelve-Factor App — III. Config.** The "store config in the environment" argument used as the chapter-3 baseline. <https://12factor.net/config>
- **Twelve-Factor App — XI. Logs.** The "treat logs as event streams" and "write to stdout" rules behind chapter 4. <https://12factor.net/logs>

## Provider client docs

- **Anthropic — Python SDK.** The client used in the chapter examples; relevant especially for `timeout=`, retry behavior, and the `usage` object on responses. <https://github.com/anthropics/anthropic-sdk-python>
- **Anthropic — Token counting.** The authoritative pre-flight token counter referenced in chapter 5. <https://docs.anthropic.com/en/docs/build-with-claude/token-counting>
- **Anthropic — Errors and rate limits.** Provider error taxonomy you map to HTTP status codes in chapter 2. <https://docs.anthropic.com/en/api/errors>
- **Anthropic — Pricing page.** Where you read the current per-million-token rates for the `prices.py` table in chapter 4 / exercise-03 — always confirm against the live page, never rely on a number copied elsewhere. <https://www.anthropic.com/pricing>
- **OpenAI — Python SDK.** Equivalent client, equivalent `timeout=` and usage fields if your service targets OpenAI. <https://github.com/openai/openai-python>
- **OpenAI — Error codes and rate limits.** <https://platform.openai.com/docs/guides/error-codes>
- **OpenAI — Pricing page.** <https://openai.com/api/pricing/>
- **`tiktoken`.** Local tokenizer for OpenAI models, useful for pre-flight token estimation in chapter 5. <https://github.com/openai/tiktoken>

## Logging and observability

- **Python — `logging` module.** Standard library reference; the chapter-4 `JsonFormatter` is a thin subclass of `logging.Formatter`. <https://docs.python.org/3/library/logging.html>
- **Python — `logging.config`.** When the inline `configure_logging(...)` outgrows being a function; the dict-config / file-config alternatives. <https://docs.python.org/3/library/logging.config.html>
- **JSON Lines specification.** The one-document-per-line contract chapter 4 follows. <https://jsonlines.org/>
- **`python-json-logger`.** A small library that gets you most of `JsonFormatter` for free if you want to skip writing your own. <https://github.com/madzak/python-json-logger>
- **`structlog`.** A widely used structured-logging library for Python with a different (richer) shape than stdlib `logging`. Out of scope for the chapter but a reasonable next step. <https://www.structlog.org/>
- **OpenTelemetry — Semantic conventions for GenAI.** The emerging standard for tracing LLM calls; chapter 4 names a few attributes (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.usage.input_tokens`) you can adopt now to ease later migration. <https://opentelemetry.io/docs/specs/semconv/gen-ai/>
- **OpenTelemetry — Python instrumentation.** When you graduate from log lines to spans. <https://opentelemetry.io/docs/languages/python/>

## Containers

- **Docker — Get started.** The official tutorial; the right pre-read if you have not used Docker before this module. <https://docs.docker.com/get-started/>
- **Docker — Dockerfile reference.** Every directive used in chapter 6 (`FROM`, `COPY`, `RUN`, `ENV`, `ARG`, `USER`, `EXPOSE`, `HEALTHCHECK`, `CMD`, `ENTRYPOINT`). <https://docs.docker.com/engine/reference/builder/>
- **Docker — Best practices for writing Dockerfiles.** Layer caching, ordering, multi-stage builds, `.dockerignore`. The chapter-6 Dockerfile is a direct application. <https://docs.docker.com/develop/develop-images/dockerfile_best-practices/>
- **Docker — Multi-stage builds.** The pattern used to keep the runtime image small. <https://docs.docker.com/build/building/multi-stage/>
- **Docker — Build secrets.** What `--secret` does and why `--build-arg` is the wrong shape for secrets in builds. <https://docs.docker.com/build/building/secrets/>
- **Docker — `.dockerignore` file reference.** What patterns are supported and how exclusions interact with negations. <https://docs.docker.com/engine/reference/builder/#dockerignore-file>
- **Docker — `HEALTHCHECK` reference.** Probe semantics, retry handling, and the difference between "running" and "healthy." <https://docs.docker.com/engine/reference/builder/#healthcheck>
- **Docker Compose — file specification.** The reference for the optional `compose.yaml` from chapter 6 and exercise-03's stretch goal. <https://compose-spec.io/>
- **Python on Docker — official image.** Tag conventions (`slim`, `alpine`, version pins) for the base image used in chapter 6. <https://hub.docker.com/_/python>
- **Python — Containerization best practices (PyPA).** Practical guidance on virtual envs, slim images, and reproducible builds. <https://packaging.python.org/en/latest/discussions/index/>

## Security and secrets

- **OWASP API Security Top 10 (2023).** The shortlist of failure modes any HTTP API should have a plan for; "broken authentication" and "unrestricted resource consumption" are immediate neighbors of this module's scope. <https://owasp.org/API-Security/editions/2023/en/0x11-t10/>
- **OWASP — Logging Cheat Sheet.** What to log and what not to log, including the sections directly relevant to chapter 4's PII boundary. <https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html>
- **OWASP — Top 10 for LLM Applications.** A curated list of LLM-specific risks; "LLM10: Unbounded Consumption" maps onto chapter 5's guardrail rationale. <https://owasp.org/www-project-top-10-for-large-language-model-applications/>
- **gitleaks.** A widely used scanner for committed secrets; useful as a pre-commit hook so the `.gitignore` rule in chapter 3 has a second line of defense. <https://github.com/gitleaks/gitleaks>
- **trufflehog.** An alternative scanner that searches git history for secrets in many formats. <https://github.com/trufflesecurity/trufflehog>

## Testing the service

- **`httpx` — Python HTTP client.** Used by FastAPI's `TestClient` under the hood and a useful manual client for integration tests against your running service. <https://www.python-httpx.org/>
- **FastAPI — Testing.** The `TestClient` pattern, dependency overrides for tests, and the JSON-body assertion style used in exercise-01 and -02 tests. <https://fastapi.tiangolo.com/tutorial/testing/>
- **`pytest` documentation.** The test framework all exercise test files assume. <https://docs.pytest.org/>
- **`hypothesis`.** Property-based testing library used in exercise-01's stretch goal. <https://hypothesis.readthedocs.io/>

## Adjacent topics (recognize-on-sight)

You do not need any of these to complete the module. Read enough of each to know what it is and where it lives in the landscape.

- **LiteLLM — proxy and SDK.** A common open-source LLM gateway / proxy that abstracts multiple provider APIs behind one OpenAI-compatible interface. Where multi-provider routing happens in many real deployments. <https://docs.litellm.ai/>
- **vLLM — API server.** A self-hosted LLM serving stack; FastAPI-shaped on the outside, GPU-shaped on the inside. Useful as a reference when your team starts hosting open-weight models. <https://docs.vllm.ai/>
- **AWS Bedrock — runtime API.** The cloud-managed model gateway analog; reading the API shape is useful even if you stay on the SDKs in this module. <https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html>
- **Google Cloud — Vertex AI Generative Models.** The Google-managed equivalent. <https://cloud.google.com/vertex-ai/docs/generative-ai/learn/overview>
- **Azure OpenAI — REST API.** The Microsoft-managed equivalent. <https://learn.microsoft.com/en-us/azure/ai-services/openai/reference>
- **Model Context Protocol (MCP).** Open protocol for connecting LLMs to external tools, resources, and memory servers. Worth knowing the name; relevant when you outgrow per-app tool wiring. <https://modelcontextprotocol.io/>

## A note on freshness

Web framework versions, provider SDKs, model ids, and pricing pages move. Where this module shows a specific version (`fastapi >= 0.115`), a specific model id (`claude-sonnet-4-6`), or a specific rate, follow the link, confirm against the live page, and snapshot the URL with a date in your code comments. If you cannot verify a rate or version, leave a `<!-- needs-research: ... -->` placeholder — do not guess. The curriculum convention is that unverified facts are flagged loudly, not invented.
