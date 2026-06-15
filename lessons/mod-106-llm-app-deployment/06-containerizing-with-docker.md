# Containerizing with Docker

## Why this matters

A FastAPI app that runs on your laptop runs because of a hundred invisible defaults: your Python version, your venv, your installed packages, your environment variables. The first time a teammate runs it, half of those are different. Their Python is 3.13, you wrote against 3.10. They forgot to source the venv. Their `pip install` pulled a different `httpx` than yours. The `.env` file isn't there. They report "it doesn't work" and you spend an afternoon diagnosing a config divergence that never existed on your machine.

Docker is the smallest cure. A `Dockerfile` declares — explicitly — the base image, the dependencies, the source files, the entrypoint. `docker build` produces an artifact. `docker run` starts it. Same bytes on every machine. The same image runs in CI, on a teammate's laptop, in staging, in production. None of the modules in this track is a deployment course, but every later module assumes you can hand someone a container that runs.

This chapter writes the smallest honest Dockerfile for the app you built in chapters 2–5: multi-stage, no secrets baked in, a healthcheck that doesn't lie, an entrypoint that respects environment variables.

## What you'll produce

By the end of this chapter and exercise-03, you have:

- A `Dockerfile` that builds a small Python image (well under 200 MB).
- A `.dockerignore` that excludes `.env`, `__pycache__`, tests, and other things that don't belong in a published image.
- A `docker build -t support-agent .` that completes in seconds after the first run.
- A `docker run -p 8000:8000 --env-file .env support-agent` that starts the service and serves requests.
- A `HEALTHCHECK` directive so the container reports unhealthy when the app is wedged.

That is it. No registry push. No Compose file. No Kubernetes. The goal is "a teammate runs the image."

## Two Docker concepts, fast

If you have used Docker before, skip this. Otherwise, the minimum:

- **An image** is a layered, read-only filesystem plus metadata (env vars, entrypoint, exposed ports). Built from a `Dockerfile`. Identified by a SHA or a tag.
- **A container** is a running instance of an image. The image is the recipe; the container is the cake.

Two operations matter:

```bash
docker build -t support-agent:0.1 .    # produce an image from the Dockerfile in $(pwd)
docker run --rm -p 8000:8000 \         # start a container from that image
           --env-file .env \
           support-agent:0.1
```

`-t` tags the image with a name. `-p 8000:8000` maps host port → container port. `--env-file .env` injects environment variables from `.env` into the container's environment without baking them into the image. `--rm` cleans up the container when it exits.

## The single-stage Dockerfile (don't ship this)

The shape that almost works:

```dockerfile
# DON'T SHIP THIS. See the multi-stage version below.
FROM python:3.12-slim

WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir .
COPY src/ src/

EXPOSE 8000
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

What this does right:

- `python:3.12-slim` is a small official base image with a working Python interpreter.
- `WORKDIR` sets the working directory; everything that follows is relative to it.
- `COPY pyproject.toml .` then `RUN pip install` *before* `COPY src/` — copying the dependency manifest first means a rebuild that doesn't change dependencies skips the install step. Layer caching is the single biggest reason your builds will go from 60 seconds to 3.
- `EXPOSE 8000` documents the port (it doesn't open one; `-p` does that).
- `CMD ["uvicorn", ...]` is the entrypoint, run as PID 1.

What's missing or wrong:

- A build toolchain (`gcc`, headers) may have ended up in the image because a transitive dep needed to compile. The image is bigger than it needs to be.
- The container runs as root. If something compromises the process, it owns the filesystem.
- There's no healthcheck — `docker ps` shows the container as "running" even when it's hung.
- There's no upper bound on `uvicorn` workers, no `--proxy-headers`, no `--forwarded-allow-ips`.

The multi-stage version below fixes the first three.

## The multi-stage Dockerfile

```dockerfile
# ---------- stage 1: build dependencies in a fat image ----------
FROM python:3.12-slim AS build
ENV PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Build deps for any wheels we have to compile.
RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy only the dependency manifest first, install into a venv we'll copy out.
COPY pyproject.toml ./
RUN python -m venv /opt/venv \
 && /opt/venv/bin/pip install --upgrade pip \
 && /opt/venv/bin/pip install .

# ---------- stage 2: runtime image ----------
FROM python:3.12-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PATH="/opt/venv/bin:$PATH"

# Create a non-root user.
RUN groupadd --system app && useradd --system --gid app --home /app app

WORKDIR /app

# Copy the venv from the build stage. No compilers in the runtime image.
COPY --from=build /opt/venv /opt/venv

# Copy app source last so source edits don't bust the dep layer.
COPY --chown=app:app src/ ./src/

USER app

EXPOSE 8000

# Lightweight Python-only healthcheck (no need to install curl).
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request,sys; \
sys.exit(0 if urllib.request.urlopen('http://localhost:8000/healthz',timeout=2).status==200 else 1)"

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

What each piece is doing:

- **`AS build` / `AS runtime`.** Two named stages. The runtime image copies from build but ships none of build's tools (`build-essential` is left behind). Final image size drops by tens of megabytes.
- **`PYTHONDONTWRITEBYTECODE=1`.** Don't write `.pyc` files. Smaller image; container filesystems don't benefit from the bytecode cache.
- **`PYTHONUNBUFFERED=1`.** Python writes log lines to stdout immediately instead of buffering them. Required for any container that ships logs by tailing stdout. Without this, `docker logs` shows nothing until the buffer flushes.
- **The venv.** We install into `/opt/venv` in the build stage and copy `/opt/venv` into the runtime stage. No `pip` in the runtime image. The `PATH` change makes `python`, `uvicorn`, etc. resolve to the venv.
- **`useradd --system app` and `USER app`.** The process runs as a non-root user named `app`. The container's filesystem is owned by root; the app user owns only the `WORKDIR`. This is one of the smallest, highest-leverage hardening steps you can take in a Dockerfile.
- **`HEALTHCHECK`.** Docker probes the container every 30 seconds. If the probe fails three times in a row, the container is marked unhealthy. `docker ps` shows the state; an orchestrator can decide to replace the container. The probe hits `/healthz`, which (per chapter 2) returns instantly without touching the model.
- **`CMD ["uvicorn", ...]`.** The exec form (with the square brackets) runs the command as PID 1 directly, so signals from `docker stop` reach the process and uvicorn shuts down cleanly instead of being killed.

## The `.dockerignore`

Equally important and often forgotten. Without one, every file in `$(pwd)` is shipped to the build context, including your `.env`, your `.git`, your `__pycache__/`, your local virtualenv, the model cache you accidentally downloaded.

```
# .dockerignore
.env
.env.*
!.env.example

.git
.gitignore
.gitattributes

__pycache__/
*.pyc
.pytest_cache/
.mypy_cache/
.ruff_cache/

.venv/
venv/

.coverage
htmlcov/

tests/
notebooks/
docs/
*.md

.DS_Store
.vscode/
.idea/
```

Two things to be deliberate about:

- **`.env` is excluded.** This is the single most important line. A `COPY . .` that includes `.env` bakes your API key into the image — and an image is meant to be shared. Excluding `.env` is non-negotiable.
- **`!.env.example` un-excludes the example file.** You *do* want the example in the image so the container is self-describing.

A useful sanity check: `docker build --no-cache .` and look at the "Sending build context to Docker daemon" line. If your build context is 500 MB, you have a `.dockerignore` problem.

## Where secrets go

Recall the chapter-3 rule: secrets are not in the code, not in the image, only in the *running container*'s environment. Three ways to inject:

- **`--env`.** One variable at a time. `docker run -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY ...`. Brittle but fine for ad hoc.
- **`--env-file`.** Read a `KEY=VALUE` file. `docker run --env-file .env ...`. This is the standard local-dev path.
- **Orchestrator secrets.** Kubernetes Secrets, ECS task env, Compose's `env_file` directive. Nothing in your code or Dockerfile changes; the orchestrator presents env vars to the container.

What you *don't* do: `COPY .env .` in the Dockerfile. The `.dockerignore` above prevents the most common version of this accident.

A concrete teammate-friendly run:

```bash
docker build -t support-agent:0.1 .
cp .env.example .env       # teammate fills in their key
docker run --rm -p 8000:8000 --env-file .env support-agent:0.1
curl -s localhost:8000/healthz
curl -s -X POST localhost:8000/chat \
    -H 'content-type: application/json' \
    -d '{"question":"hi"}'
```

That is the README section for your repo.

## Build-time vs. runtime, named

A frequent source of confusion: `ARG` is set at build time, `ENV` is set at runtime.

```dockerfile
ARG APP_VERSION=dev          # passed via --build-arg APP_VERSION=0.1.0
ENV APP_VERSION=$APP_VERSION # baked into the image; visible at runtime
```

Use `ARG` for build-time facts (version string, base image tag); use `ENV` for runtime defaults (log level, model id). Secrets are *neither* — they are injected per run.

> An anti-pattern that bites people: `docker build --build-arg ANTHROPIC_API_KEY=...`. The ARG is now in the image's history layers, even if you don't reference it in a final ENV. Build-arg secrets are not secrets. <https://docs.docker.com/build/building/secrets/> explains the right shape (`--secret` mounts) when you do need a secret at build time (e.g., to download a private wheel). You will not need that in this module.

## Image hygiene, briefly

A short checklist that pays for itself in every later module:

- **Pin the base image.** `python:3.12-slim` works; `python:3.12.4-slim-bookworm` is better when you can stomach the rebuild. Wildcards bite when the base bumps a vulnerable transitive overnight.
- **Pin your dependencies.** `pyproject.toml` with version pins, or a lockfile. A rebuild three months from now should produce the same image, not a slightly newer one.
- **Don't run as root.** Already covered above; still worth restating because the default is root and the line that fixes it is two short lines.
- **Don't expose `/docs` in prod.** Already covered in chapter 3 via the `DOCS_ENABLED` flag; in the image, set the default to true and let the deployment env flip it off.
- **One process per container.** No `tini`, no init systems, no supervisord. uvicorn is PID 1; `docker stop` sends SIGTERM; uvicorn catches it.
- **Keep the image small.** Multi-stage; `--no-install-recommends`; `rm -rf /var/lib/apt/lists/*`. Worth ten minutes; saves you minutes per CI run forever.

## Compose, briefly

`docker compose` is the next step up from `docker run` when you need more than one process (your app + a Redis + a Postgres). You will not need it in this module. The shape is familiar:

```yaml
# compose.yaml (optional; included for reference)
services:
  app:
    build: .
    ports: ["8000:8000"]
    env_file: [.env]
    environment:
      LOG_LEVEL: INFO
```

`docker compose up` builds and runs. If you reach for Compose in this module, it should be because you added a second service in a stretch goal — not because it's a Docker upgrade.

## What this chapter is not

- **Not a registry push.** `docker push <registry>/support-agent:0.1` is one line; the registry, auth, image scanning, signing, and SBOM are not in scope.
- **Not Kubernetes.** No `Deployment`, no `Service`, no `Ingress`. Operator track.
- **Not autoscaling.** The image runs one process. Scaling is whatever the orchestrator does to copies of this image; *the image itself does not change*.
- **Not signing or scanning.** Cosign, Trivy, Grype, Snyk — all real and all out of scope. Image-supply-chain security is its own course.

## A short checklist

Before exercise-03 closes:

- [ ] Multi-stage Dockerfile that produces a runtime image under ~200 MB.
- [ ] Build runs without errors against a clean checkout.
- [ ] Runtime image runs as a non-root user.
- [ ] `HEALTHCHECK` against `/healthz`.
- [ ] `.dockerignore` excludes `.env`, `__pycache__`, `.git`, `tests/`, and your local venv.
- [ ] `docker run --env-file .env -p 8000:8000 <tag>` boots the app and serves a `curl`.
- [ ] No secret values referenced in the Dockerfile or build args.

That is the floor. Chapter 7 ties the whole thing together.

## Summary

Docker is the smallest cure for "works on my machine." A multi-stage Dockerfile bakes the app and the dependencies; a `.dockerignore` keeps `.env` and `.git` out; a runtime user is non-root; a healthcheck hits the cheap `/healthz` endpoint without touching the model. Secrets are injected at run time via `--env-file`, never baked in. The result is a single artifact you can hand to a teammate with one `docker run` and have them poking at your service in minutes — which is the prerequisite for every later course in this track.
