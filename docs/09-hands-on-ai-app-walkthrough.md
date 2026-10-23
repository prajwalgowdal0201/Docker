# Chapter 9 — Hands-On: Build, Run and Deploy an AI Chat App with Docker

This is the chapter where everything clicks. Chapters 1–8 explained the concepts; here you **do** them, step by step, with a real application.

**What you will build and run** (code is in [`../sample-ai-app/`](../sample-ai-app/)):

```
                      YOUR LAPTOP / SERVER  (everything below runs in Docker containers)
  ┌─────────────────────────────────────────────────────────────────────────────┐
  │                                                                             │
  │  browser / curl ──► nginx ──────► api (FastAPI, Python) ──┬──► Postgres     │
  │                  :8080  reverse     your AI app code      │    saves chat   │
  │                         proxy                             │    history      │
  │                                                           ├──► Redis        │
  │                                                           │    cache + rate │
  │                                                           │    limiting     │
  │                                                           └──► LLM          │
  │                                                                mock (default)│
  │                                                                or OpenAI-API │
  │                                                                or Ollama     │
  └─────────────────────────────────────────────────────────────────────────────┘
```

**What was verified.** Parts 1–7 below were executed on Windows 11 with Docker Desktop (Docker Engine 29.9.2, Compose 5.6.1), and the outputs shown are real. Part 9 (deploying to a Linux server) follows the standard procedure but was not executed against a real cloud server while writing this guide — treat it as a tested-in-principle recipe and expect to adapt hostnames and IPs.

**Time needed:** about 2–3 hours if you read every explanation and try the experiments.

---

## Table of contents

1. [Prerequisites and what the app does](#part-1--prerequisites-and-what-the-app-does)
2. [Tour of the project, file by file](#part-2--tour-of-the-project-file-by-file)
3. [Build the image and run the tests](#part-3--build-the-image-and-run-the-tests)
4. [Start the whole stack with Compose](#part-4--start-the-whole-stack-with-compose)
5. [Use the app](#part-5--use-the-app)
6. [Experiments that teach you how Docker really behaves](#part-6--experiments-that-teach-you-how-docker-really-behaves)
7. [Plug in a real AI model (hosted API or local Ollama)](#part-7--plug-in-a-real-ai-model)
8. [The developer workflow: hot reload](#part-8--the-developer-workflow-hot-reload)
9. [Deploy to a Linux server (production)](#part-9--deploy-to-a-linux-server-production)
10. [Operate it: logs, backups, updates, rollback](#part-10--operate-it-logs-backups-updates-rollback)
11. [Clean up](#part-11--clean-up)
12. [Publish it to GitHub as your portfolio project](#part-12--publish-it-to-github)
13. [Challenges to go further](#part-13--challenges-to-go-further)

---

## Part 1 — Prerequisites and what the app does

### What you need installed

| Tool | Why | Check with |
|---|---|---|
| Docker Desktop (Windows/macOS) or Docker Engine (Linux) | Runs containers. See [Chapter 2](02-installation-and-operating-systems.md). | `docker --version` |
| Docker Compose v2+ (bundled with Docker Desktop) | Starts multi-container apps from one file. See [Chapter 6](06-docker-compose.md). | `docker compose version` |
| A terminal (PowerShell, Terminal, bash) | To type commands | — |
| A code editor (VS Code) | To read/edit files | — |

**You do NOT need Python installed.** That is one of Docker's big benefits: Python and every library live inside the image. Your machine stays clean.

**Make sure Docker is running.** On Windows/macOS, start *Docker Desktop* and wait until it says "Engine running". If you see an error like `failed to connect to the docker API ... dockerDesktopLinuxEngine`, the engine isn't started yet.

### What the app does

It's a chat API. You send `{"session_id": "demo", "message": "What is Docker?"}`; it:

1. **Checks a rate limit** in Redis (30 requests/minute per IP) — protects you from runaway cost on a paid LLM API.
2. **Loads the last 10 messages** of that session from Postgres, so the model remembers the conversation.
3. **Looks in the Redis cache** for an identical prompt. If found, it skips the (slow, possibly paid) model call.
4. **Calls the LLM** (the mock by default; a real one if configured).
5. **Saves** your message and the answer to Postgres, and returns the answer.

This is the same skeleton as real-world AI products: *API + database + cache + model provider + reverse proxy*.

> **Term: LLM** — Large Language Model, the AI model (like GPT or Llama) that generates text.
> **Term: Reverse proxy** — a server (here nginx) that sits in front of your app and forwards incoming requests to it. It is the single public door; the app itself is hidden behind it.
> **Term: Rate limiting** — refusing requests from a client that sends too many too fast.
> **Term: Cache** — a fast temporary store of previous results so repeated work can be skipped.

---

## Part 2 — Tour of the project, file by file

```
sample-ai-app/
├── app/
│   ├── main.py            ← the API endpoints (FastAPI)
│   ├── llm.py             ← talks to the model (mock / OpenAI-compatible)
│   └── config.py          ← reads settings from environment variables
├── tests/test_llm.py      ← unit tests
├── nginx/nginx.conf       ← reverse proxy config
├── Dockerfile             ← recipe to build the API image
├── compose.yaml           ← describes ALL containers and how they connect
├── compose.override.yaml  ← extra settings for development (auto-applied)
├── compose.prod.yaml      ← extra settings for production (opt-in)
├── requirements.txt       ← Python libraries (pinned versions)
├── .env.example           ← configuration template
└── .dockerignore          ← files NOT sent to the Docker builder
```

### 2.1 `config.py` — configuration from the environment

```python
class Settings(BaseSettings):
    llm_provider: str = "mock"
    database_url: str = "postgresql://chat:chat@db:5432/chat"
    redis_url: str = "redis://redis:6379/0"
    ...
```

`pydantic-settings` automatically fills each field from an environment variable of the same name (case-insensitive): `LLM_PROVIDER`, `DATABASE_URL`, etc. The value after `=` is the default if the variable isn't set.

**Why this matters for Docker:** you build **one image** and run it everywhere. Dev, staging and production differ only by environment variables — never by editing code or rebuilding. This is rule III of the "Twelve-Factor App" methodology (*store config in the environment*).

Notice the hostnames `db` and `redis`. Inside a Compose network, **each service name is a DNS name** that resolves to that container's IP address. So `db` means "the container named db" — you never hardcode IP addresses. (More in [Chapter 5](05-storage-and-networking.md).)

### 2.2 `llm.py` — swap models without changing code

```python
def complete(messages):
    if settings.llm_provider == "mock":   return _mock(messages)
    if settings.llm_provider == "openai": return _openai_compatible(messages)
```

The `openai` provider sends `POST {LLM_BASE_URL}/chat/completions`. Hosted services (OpenAI, Groq, Azure OpenAI), local servers (Ollama, vLLM, llama.cpp) all speak this same "OpenAI-compatible" HTTP format, so **changing `LLM_BASE_URL` is all it takes to switch providers**. That is a pattern you'll use constantly as an AI developer.

### 2.3 `main.py` — the endpoints

* `GET /health` — "is the process alive?" Returns `{"status":"ok"}`. Docker's `HEALTHCHECK` calls this. **Liveness.**
* `GET /ready` — "can I actually serve traffic?" It runs `SELECT 1` on Postgres and `PING` on Redis. **Readiness.** (Orchestrators like Kubernetes distinguish the two — [Chapter 11](11-production-and-enterprise.md).)
* `POST /chat` — the logic described in Part 1.
* `GET /history/{session_id}` — returns saved messages.

The functions are plain `def` (not `async def`) because the database/Redis/HTTP libraries used are synchronous; FastAPI runs such functions in a thread pool, so the server stays responsive.

The `lifespan` function runs once at startup: it opens the Postgres connection pool, creates the table if missing, connects to Redis.

### 2.4 `Dockerfile` — the image recipe (multi-stage)

Open [`../sample-ai-app/Dockerfile`](../sample-ai-app/Dockerfile). It has **three stages**. A *stage* is a mini-build; later stages can copy files out of earlier ones, and everything else is thrown away.

```dockerfile
# syntax=docker/dockerfile:1
```
Opts in to the current Dockerfile syntax (BuildKit frontend). Always keep this first line.

**Stage 1 — `builder`: install the libraries**

```dockerfile
FROM python:3.12-slim AS builder
```
Start from the official Python 3.12 image, "slim" variant (Debian without extras; ~50 MB compressed instead of ~350 MB). `AS builder` names this stage.

```dockerfile
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1
```
Sets environment variables: don't nag about pip updates; don't keep pip's download cache (smaller image).

```dockerfile
WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
```
`WORKDIR` sets the working directory (creating it). We create a Python *virtual environment* (an isolated folder of installed libraries) at `/opt/venv` and put it first on `PATH` so `pip`/`python` use it. We do this so the whole set of libraries is **one folder we can copy** into the final stage.

```dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt
```
**The most important caching trick in Docker.** We copy *only* the dependency list first and install. Docker caches each instruction's result (a *layer*). As long as `requirements.txt` is unchanged, this slow step is reused from cache even when your code changes. If we had copied all the code first, every code edit would reinstall every library.

**Stage 2 — `test`: run unit tests during the build**

```dockerfile
FROM builder AS test
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt
COPY app/ app/
COPY tests/ tests/
RUN python -m pytest -q
```
Starts from the builder stage, adds pytest, copies code and tests, and **runs the tests as a build step**. If a test fails, `docker build` fails, so a broken image can never be produced. This stage is only built when asked with `--target test` (or when CI requests it); the normal runtime build skips it.

**Stage 3 — `runtime`: the small image that ships**

```dockerfile
FROM python:3.12-slim AS runtime
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH="/opt/venv/bin:$PATH"
```
A fresh slim image. `PYTHONUNBUFFERED=1` makes Python print logs immediately (otherwise they can be delayed and you'd think the container is silent). `PYTHONDONTWRITEBYTECODE=1` skips `.pyc` files (we have a read-only filesystem anyway).

```dockerfile
COPY --from=builder /opt/venv /opt/venv
```
Copy the finished virtualenv out of the builder stage. Nothing else from the builder (no build cache, no temp files) comes along.

```dockerfile
RUN useradd --system --uid 10001 --no-create-home appuser
COPY --chown=appuser:appuser app/ app/
USER appuser
```
Create an unprivileged user, copy the code owned by it, and switch to it. **From here on, the container does not run as root.** If an attacker exploits your app, they get a powerless user instead of root inside the container. (Why that matters: [Chapter 10](10-security.md).)

```dockerfile
EXPOSE 8000
```
Documentation only: "this app listens on 8000". It doesn't publish anything by itself.

```dockerfile
HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2).status==200 else 1)"
```
Every 15 seconds Docker runs this command *inside* the container. Exit code 0 = healthy, 1 = unhealthy. `--start-period=10s` gives the app time to boot before failures count; 3 consecutive failures mark the container `unhealthy`. We use Python's built-in `urllib` because slim images have no `curl`.

```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```
The default command. **Exec form** (a JSON array) means uvicorn itself becomes process #1 in the container and receives stop signals (SIGTERM) directly, so `docker stop` shuts it down gracefully instead of waiting 10 seconds and killing it. `--host 0.0.0.0` means "listen on all network interfaces" — required, because `127.0.0.1` inside a container is reachable only from that container itself.

### 2.5 `compose.yaml` — the whole stack

Open [`../sample-ai-app/compose.yaml`](../sample-ai-app/compose.yaml). Key ideas, in order:

```yaml
name: ai-chat
```
The *project name*. Compose prefixes container, network and volume names with it (`ai-chat-api-1`, `ai-chat_pgdata`). There is **no `version:` line**; it's obsolete in modern Compose.

**`nginx` service**

```yaml
nginx:
  image: nginx:1.27-alpine
  ports: ["8080:80"]
  volumes: ["./nginx/nginx.conf:/etc/nginx/nginx.conf:ro"]
  depends_on:
    api: { condition: service_healthy }
```
* `image:` — use a ready-made image from Docker Hub (the `-alpine` variant is tiny).
* `ports: "8080:80"` — publish: traffic arriving at **host** port 8080 is forwarded to **container** port 80. Format is always `HOST:CONTAINER`.
* `volumes: ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro` — a *bind mount*: your file on disk appears inside the container at that path. `:ro` = read-only. This is how you inject config without building a custom image.
* `depends_on … service_healthy` — don't even start nginx until the API's `HEALTHCHECK` passes. (Plain `depends_on` only waits for the container to *start*, not to be *ready* — a classic beginner trap.)

**`api` service**

```yaml
api:
  build: { context: ., target: runtime }
  image: ai-chat-api:local
  environment:
    DATABASE_URL: postgresql://${POSTGRES_USER:-chat}:${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}@db:5432/${POSTGRES_DB:-chat}
```
* `build:` — build from the local Dockerfile (`context: .` = send this folder to the builder; `target: runtime` = stop at the final stage, skipping the test stage).
* `image:` — name given to the built image.
* `${VAR:-default}` — Compose substitution: use `$VAR` from your shell/`.env`, or `default` if unset.
* `${VAR:?message}` — **fail with `message` if `$VAR` is unset.** We use this for the password so you can't accidentally start the stack with a missing secret.
* Notice `@db:5432`: the host is the service name `db`.

```yaml
  expose: ["8000"]
```
Makes port 8000 reachable by *other containers* on the network but **not published to the host**. Only nginx can talk to the API (in production). The dev override adds a localhost-only publish for convenience.

```yaml
  depends_on:
    db:    { condition: service_healthy }
    redis: { condition: service_healthy }
  restart: unless-stopped
```
The API waits for healthy DB and Redis. `restart: unless-stopped` — if the container crashes, or the machine/Docker restarts, bring it back (unless *you* stopped it).

```yaml
  read_only: true
  tmpfs: [ /tmp ]
  cap_drop: [ ALL ]
  security_opt: [ "no-new-privileges:true" ]
```
**Security hardening.** `read_only` makes the container's root filesystem immutable (attackers can't drop files; we verify this in Part 6). `tmpfs /tmp` gives a RAM-backed scratch folder because some programs need a writable `/tmp`. `cap_drop: ALL` removes every special Linux privilege (a *capability*) such as binding low ports or changing file owners — our app needs none. `no-new-privileges` blocks privilege escalation tricks.

**`db` service**

```yaml
db:
  image: postgres:16-alpine
  environment: { POSTGRES_USER: ..., POSTGRES_PASSWORD: ..., POSTGRES_DB: ... }
  volumes: [ "pgdata:/var/lib/postgresql/data" ]
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER:-chat} -d $${POSTGRES_DB:-chat}"]
```
* The Postgres image creates the user/database from those environment variables on first start.
* `pgdata:/var/lib/postgresql/data` — a **named volume**. Docker manages the storage; it survives container deletion. Declared at the bottom under `volumes:`. **Without this, all chat history would vanish when the container is removed.**
* `$$` — a literal `$` (Compose would otherwise try to substitute it). The check runs *inside* the container, where those variables exist. `pg_isready` is Postgres's own "are you accepting connections?" tool.

> **Version trap if you upgrade Postgres:** the mount path above is correct for `postgres:16` (and 17 and earlier). Postgres **18+** images changed their layout and expect the volume at `/var/lib/postgresql` instead. If you change the image tag to 18 or newer, change the mount path too (details in [Chapter 5](05-storage-and-networking.md)), otherwise your data may not persist the way you expect. Always test persistence after changing a database image version.

**`redis` service** — same idea; `command:` overrides the default start command to cap memory at 64 MB and evict least-recently-used keys when full (appropriate for a cache). Health check: `redis-cli ping` (expects `PONG`). Redis has no volume on purpose: it's a cache, losing it is fine.

**`ollama` service**

```yaml
ollama:
  image: ollama/ollama:latest
  profiles: ["ollama"]
```
`profiles` means this service is **skipped by default** and only starts if you pass `--profile ollama`. It's an optional local LLM server (Part 7). The GPU block is commented out — enable it only on a Linux/NVIDIA machine.

### 2.6 `compose.override.yaml` — development extras

Compose automatically merges `compose.yaml` + `compose.override.yaml` when you run a plain `docker compose up`. It publishes the API on `127.0.0.1:8000` and Postgres on `127.0.0.1:5432` (so you can poke at them from your host), overrides the command with `--reload`, and defines `develop.watch` rules for hot reload (Part 8). Binding to `127.0.0.1` means *only your own machine* can connect — not other computers on the network.

### 2.7 `compose.prod.yaml` — production extras

Used only when you explicitly pass `-f compose.yaml -f compose.prod.yaml` (which also *disables* the automatic dev override). It swaps `build:` for a pre-built versioned `image:` (`${API_IMAGE}`), forces `pull_policy: always`, adds **log rotation** (`max-size 10m`, `max-file 5` — without this, Docker's default `json-file` logs can grow until the disk is full), sets memory/CPU limits, and publishes nginx on port 80.

### 2.8 `nginx.conf`

Two details worth understanding:

* `resolver 127.0.0.11` + `set $upstream http://api:8000;` — 127.0.0.11 is Docker's embedded DNS server. Using a *variable* in `proxy_pass` forces nginx to re-resolve the name periodically instead of caching the IP at startup, so nginx keeps working if the `api` container is recreated and gets a new IP.
* `proxy_read_timeout 120s; proxy_buffering off;` — LLM responses are slow and often streamed; default timeouts (60s) and buffering break that.

### 2.9 `.dockerignore`

Before building, the Docker client packages the *build context* (your folder) and sends it to the builder. `.dockerignore` lists what to leave out: `.git`, `.env` (**secrets must never enter an image**), caches, docs. Smaller context = faster builds and no accidental leaks.

---

## Part 3 — Build the image and run the tests

Open a terminal in the project folder:

```bash
cd Docker_Complete_Guide/sample-ai-app
```

### 3.1 Run the tests inside Docker

```bash
docker build --target test --progress=plain -t ai-chat-api:test .
```

* `docker build` — build an image from a Dockerfile.
* `--target test` — stop at the stage named `test` (which runs pytest).
* `--progress=plain` — print full text logs instead of the compact animated view (better for learning/CI).
* `-t ai-chat-api:test` — tag (name) the result `repository:tag`.
* `.` — the build context: this directory.

Real output (trimmed):

```
#16 0.770 ..                                                                       [100%]
#16 0.772 2 passed in 0.13s
#16 DONE 0.8s
...
#17 naming to docker.io/library/ai-chat-api:test done
```

`2 passed` — your tests ran *inside* the container with the exact Python and library versions that will run in production. No "works on my machine".

Run it a second time: it finishes almost instantly because every layer comes from **cache**. Now edit `tests/test_llm.py` (add a blank line) and rebuild: only the layers *after* that change re-run; the slow `pip install` stays cached. That's the layer cache from [Chapter 4](04-dockerfile-deep-dive.md) in action.

### 3.2 Build the runtime image

```bash
docker build --target runtime -t ai-chat-api:local .
docker images ai-chat-api
```

Real output:

```
REPOSITORY    TAG     IMAGE ID       CREATED          SIZE
ai-chat-api   local   ...            ...              275MB
```

(Docker Desktop with the containerd image store reports slightly different numbers than the classic store; don't worry about the exact digits.)

See which layers make up the image:

```bash
docker history ai-chat-api:local
```

You'll see the Python base layers at the bottom and our own instructions at the top, e.g. `75.1MB COPY /opt/venv /opt/venv` (our libraries) and a tiny `28.7kB COPY ... app/` (our code). Notice how small our code layer is and how large the libraries layer is — that's *why* the Dockerfile orders them with libraries first.

---

## Part 4 — Start the whole stack with Compose

### 4.1 Create your `.env`

```bash
cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env
```

Open `.env` and change `POSTGRES_PASSWORD` to any password you like. Leave `LLM_PROVIDER=mock` for now.

> **Why a separate `.env`?** It holds secrets and per-machine settings. The `.gitignore` excludes it so you never push it to GitHub. `.env.example` (committed) documents which variables exist.
>
> **Try the safety net:** delete the password line and run `docker compose config`. You get `error while interpolating ... Set POSTGRES_PASSWORD in .env` — the `:?` guard in action.

### 4.2 Start everything

```bash
docker compose up -d --build
```

* `docker compose up` — create and start all services in `compose.yaml`.
* `-d` — *detached*: run in the background and give your terminal back.
* `--build` — rebuild images first if the Dockerfile or code changed.

Real output (trimmed) — read the order, it tells the story:

```
 Volume ai-chat_pgdata Created
 Network ai-chat_default Created
 Container ai-chat-redis-1 Created
 Container ai-chat-db-1 Created
 Container ai-chat-api-1 Created
 Container ai-chat-nginx-1 Created
 Container ai-chat-db-1 Started
 Container ai-chat-redis-1 Started
 Container ai-chat-db-1 Healthy
 Container ai-chat-redis-1 Healthy
 Container ai-chat-api-1 Started
 Container ai-chat-api-1 Healthy
 Container ai-chat-nginx-1 Started
```

Compose created a **network** and a **volume**, started `db` and `redis` first, **waited for them to become healthy**, then started `api`, waited for *it* to be healthy, and only then started `nginx`. That ordering is the `depends_on … service_healthy` rules.

### 4.3 Check status

```bash
docker compose ps
```

```
NAME              IMAGE                STATUS                    PORTS
ai-chat-api-1     ai-chat-api:local    Up 6 seconds (healthy)    127.0.0.1:8000->8000/tcp
ai-chat-db-1      postgres:16-alpine   Up 12 seconds (healthy)   127.0.0.1:5432->5432/tcp
ai-chat-nginx-1   nginx:1.27-alpine    Up ...                    0.0.0.0:8080->80/tcp
ai-chat-redis-1   redis:7-alpine       Up 12 seconds (healthy)   6379/tcp
```

Read the PORTS column carefully — it's a great way to understand networking:

* `0.0.0.0:8080->80/tcp` — nginx is published on **all interfaces**: anyone who can reach your machine can use it.
* `127.0.0.1:8000->8000/tcp` — API published to **localhost only** (from the dev override).
* `6379/tcp` with no arrow — Redis is **not published at all**: only containers on the same network can reach it. That's the safest default for databases and caches.

---

## Part 5 — Use the app

### 5.1 Health and readiness

```bash
curl http://localhost:8080/ready
```
```
{"status":"ready"}
```
This request traveled: your terminal → host port 8080 → nginx container → (Docker network) → api container → which ran `SELECT 1` on `db` and `PING` on `redis`.

### 5.2 The interactive docs

Open <http://localhost:8080/docs> in a browser. FastAPI generates a Swagger page where you can try each endpoint with a button — handy for demos.

### 5.3 Send a chat message

**macOS / Linux / Git Bash:**
```bash
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id":"demo","message":"What is Docker?"}'
```

**Windows PowerShell** (important gotcha: Windows PowerShell 5.1 strips the double quotes from inline JSON passed to `curl.exe`, producing a confusing `json_invalid` error — this really happened while testing this guide). Use PowerShell's native command instead:
```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:8080/chat `
  -ContentType "application/json" `
  -Body '{"session_id":"demo","message":"What is Docker?"}'
```
(or save the JSON in a file and use `curl.exe -d "@body.json"`).

Real response:

```json
{"reply":"[mock model] You said: 'What is Docker?'. This conversation has 1 user message(s). Set LLM_PROVIDER=openai in .env to talk to a real model.","cached":false,"provider":"mock"}
```

Send the same request again: `... has 2 user message(s)` — the app remembered the first message from Postgres.

### 5.4 View the saved history

```bash
curl http://localhost:8080/history/demo
```
Returns four rows now (two user messages + two replies), each with a timestamp.

### 5.5 See the cache work

The cache key is built from the *entire prompt* (provider, model, history + new message). Inside one session the history grows, so the prompt never repeats. But a **different session** that sends the same first message produces the same prompt:

```bash
# session "a" says "hi"  → cached: false
# session "b" says "hi"  → cached: true   (the model call is skipped)
```
We tested exactly this: the first call returned `"cached":false`, a later call from another session returned `"cached":true`. In a real product this saves money and latency for FAQ-style questions.

---

## Part 6 — Experiments that teach you how Docker really behaves

Do these. Each one cements a concept.

### Experiment 1 — Containers are disposable, volumes are not

```bash
docker compose down            # stop and REMOVE containers + network (volumes are kept)
docker compose up -d           # recreate everything
curl http://localhost:8080/history/demo
```
Result in our test: still **4 messages**. The containers were destroyed and recreated, but the Postgres data lived in the `pgdata` volume. (Wait ~15 seconds after `up` for the health checks to pass before calling.)

Now the destructive version — **only do this when you want to lose data:**

```bash
docker compose down -v         # -v also deletes the named volumes
```
History is gone. Remember: `down -v` is the "delete my database" switch.

Notice also that **the Redis cache was emptied by `down`** (it has no volume). In our test, a "cached: true" response became "cached: false" right after a `down`/`up`. That's expected for a cache.

### Experiment 2 — The hardened container

```bash
docker compose exec api sh -c "id; touch /srv/x; touch /tmp/ok && echo tmp-writable"
```

* `docker compose exec api …` — run a command *inside* the running `api` container.

Real output:

```
uid=10001(appuser) gid=999(appuser) groups=999(appuser)
touch: cannot touch '/srv/x': Read-only file system
tmp-writable
```
Three proofs: it runs as **non-root** (`uid=10001`), the **filesystem is read-only** (writing to `/srv` fails), but `/tmp` (tmpfs) works. If an attacker got code execution in your API, they couldn't install tools or modify your code.

### Experiment 3 — Rate limiting

Send 32 requests quickly (PowerShell example):

```powershell
1..32 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " -X POST http://localhost:8080/chat -H "Content-Type: application/json" -d "@body.json" }
```
Real result: `200` thirty times, then `429 429` (HTTP 429 = Too Many Requests). The counter lives in Redis, keyed by client IP and the current minute. Because the counter is in Redis (not in app memory), it would stay correct even if you ran **ten copies** of the API — the key advantage of external shared state.

### Experiment 4 — Crash recovery (`restart: unless-stopped`) and a surprise

Docker's restart policy is meant for **unexpected** exits. Docker treats a stop that *you* request through Docker as intentional. We tested both on a real run:

```bash
# A) Kill the container from outside, via Docker
docker kill --signal=SIGKILL ai-chat-api-1
docker inspect --format 'status={{.State.Status}} restarts={{.RestartCount}} exit={{.State.ExitCode}}' ai-chat-api-1
```
Real output: `status=exited restarts=0 exit=137` — it was **not** restarted. (`137` = 128 + 9: the process died from signal 9, SIGKILL.) Because you asked Docker to kill it, Docker assumes you meant it. `docker compose kill api` behaves the same way.

```bash
# B) Make the main process exit from the inside (what a real crash looks like to Docker)
docker compose up -d                                  # bring api back first
docker compose exec -T api sh -c "kill 1"             # PID 1 = uvicorn; ask it to terminate
docker inspect --format 'status={{.State.Status}} restarts={{.RestartCount}}' ai-chat-api-1
```
Real output: `status=running restarts=1` — Docker noticed the process exit by itself and **restarted the container automatically**.

**Lesson:** to test crash recovery, make the process die *inside* the container; `docker kill/stop` are treated as deliberate. Also note `exec ... kill 1` worked here because uvicorn installs a handler for SIGTERM; a process that doesn't (PID 1 gets no default signal behaviour in Linux) would ignore it — one more reason to run a proper server as PID 1 using the exec-form `CMD`.

### Experiment 5 — Break a dependency and watch health checks

```bash
docker compose stop redis
curl http://localhost:8080/ready      # now returns 503 "dependency down"
curl http://localhost:8080/health     # still 200 — the process itself is alive
docker compose start redis
```
Real output: `/ready` → `{"detail":"dependency down: Error -5 connecting to redis:6379. No address associated with hostname."} [503]` and `/health` → `{"status":"ok"} [200]`. (The "no address" wording appears because a stopped container's name is removed from Docker's DNS.)
This demonstrates **liveness vs readiness**: `/health` says "the process is up" (so Docker should *not* restart it), while `/ready` says "I can't serve users right now" (a load balancer should route traffic elsewhere).

### Experiment 6 — Look inside

```bash
docker compose logs -f api              # follow logs (Ctrl+C to stop following)
docker compose exec db psql -U chat -d chat -c "SELECT id, session_id, role, left(content, 40) FROM messages;"
docker compose exec redis redis-cli keys '*'
docker network ls                        # find ai-chat_default
docker network inspect ai-chat_default   # see each container's IP address
docker volume ls
docker volume inspect ai-chat_pgdata     # where the data physically lives
docker stats --no-stream                 # CPU/memory per container
```
* `psql` is Postgres's command-line client; you're querying the real database that holds your chat history.
* `redis-cli keys '*'` lists cache and rate-limit keys (`llm:<hash>`, `rl:<ip>:<minute>`).
* `docker network inspect` shows the private network Compose created and every container's IP on it.
* `docker volume inspect` shows the `Mountpoint` — on Docker Desktop this path is inside Docker's hidden Linux VM, not your Windows/macOS filesystem.

---

## Part 7 — Plug in a real AI model

### 7.1 Option A: a hosted API (OpenAI, Groq, Azure OpenAI, …)

Edit `.env`:

```env
LLM_PROVIDER=openai
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=sk-...your-key...
LLM_MODEL=gpt-4o-mini
```

For Groq (has a free tier at the time of writing; check current terms and model names):

```env
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_API_KEY=gsk_...
LLM_MODEL=llama-3.3-70b-versatile
```

Apply the change — Compose notices the environment changed and **recreates only the `api` container**:

```bash
docker compose up -d
```

> **Security reminders**
> * The API key is passed as a *runtime environment variable*. It is **not** in the image (check: `docker history ai-chat-api:local` shows no key) and **not** in git (`.env` is ignored).
> * Environment variables are visible to anyone who can run `docker inspect` on the host. For stricter setups use Docker/Compose *secrets* or a cloud secret manager ([Chapter 10](10-security.md)).
> * Put a spending limit on your provider account. Our rate limiter helps but isn't a budget.
> * If a key ever leaks (committed to git, pasted in a screenshot), **revoke and rotate it immediately** — deleting the commit is not enough.

### 7.2 Option B: run a model locally with Ollama in Docker

**Ollama** is a program that downloads and serves open-source LLMs (Llama, Qwen, Mistral, …) behind an API. Running it as a container means no installation on your machine.

```bash
docker compose --profile ollama up -d
```
`--profile ollama` activates the optional service. Then download a small model into the container (≈1.3 GB for the 1B model; **models are large — check disk space**):

```bash
docker compose exec ollama ollama pull llama3.2:1b
```
* `docker compose exec ollama …` runs a command inside the Ollama container.
* `ollama pull llama3.2:1b` downloads model weights to `/root/.ollama`, which is the **`ollama_models` named volume** — so they survive container recreation and you only download once.

Point the API at it in `.env`:

```env
LLM_PROVIDER=openai
LLM_BASE_URL=http://ollama:11434/v1
LLM_API_KEY=ollama
LLM_MODEL=llama3.2:1b
LLM_TIMEOUT_SECONDS=180
```
`http://ollama:11434` — again the *service name* as hostname. The key can be any non-empty text; Ollama ignores it. CPU-only inference is slow (the first answer may take many seconds, hence the longer timeout).

```bash
docker compose --profile ollama up -d      # recreate api with the new env
```
> **Always include `--profile ollama`** in later `up`/`down` commands, otherwise Compose ignores that service (it won't be stopped by `down`, for example).

**GPU acceleration:** on a **Linux** machine with an NVIDIA GPU, the NVIDIA driver and the NVIDIA Container Toolkit installed, uncomment the `deploy.resources.reservations.devices` block in `compose.yaml`. On Windows with WSL2 GPU support it can work too; on macOS Docker containers cannot use the Apple GPU. Details in [Chapter 8](08-docker-for-ai-and-ml.md).

> **Design lesson:** the *only* thing that changed between "mock", "OpenAI" and "local Llama" was three environment variables. The application code and the image are identical. That is the payoff of externalized configuration plus containers.

---

## Part 8 — The developer workflow: hot reload

Rebuilding the image after every code edit is slow. Compose has a *watch mode* that syncs your edits into the running container:

```bash
docker compose watch          # or:  docker compose up --watch
```
`compose.override.yaml` (auto-applied in development) contains:

```yaml
read_only: false
develop:
  watch:
    - action: sync+restart     # copy changed files into the container, then restart it
      path: ./app
      target: /srv/app
    - action: rebuild          # a changed requirements.txt needs a new image
      path: requirements.txt
```
We tested this: after changing the mock reply text in `app/llm.py` the watcher printed

```
Watch enabled
Syncing service "api" after 1 changes were detected
 Container ai-chat-api-1 Restarting
service(s) ["api"] restarted
```
and the next `/chat` call returned the new text — a few seconds, no manual rebuild.

**A lesson we hit for real:** the first attempt failed with `Error response from daemon: container rootfs is marked read-only`. `watch` copies files *into* the container, and Docker refuses that when the root filesystem is read-only (our production hardening). The fix is the `read_only: false` line in the **dev override only**. Production (`compose.yaml` + `compose.prod.yaml`) keeps the read-only filesystem. This "harden by default, relax in dev" pattern is common.

Why `watch` instead of bind-mounting your source folder? On Windows/macOS, bind-mounted files cross a virtualization boundary: it's slower and file-change notifications (`inotify`) are unreliable, so `uvicorn --reload` can miss edits. `sync` copies files reliably. On a Linux dev machine a bind mount also works well.

---

## Part 9 — Deploy to a Linux server (production)

> This part follows the standard recipe. Each command is explained, but it was not run against a real cloud VM while this guide was written.

### 9.0 The production picture

```
 Your laptop                 GitHub                       Registry (GHCR)          Linux server (VM)
 ───────────                 ──────                       ───────────────          ─────────────────
 git push ───────────────►  Actions CI:                                            docker compose pull
                             1. run tests (docker build                            docker compose up -d
                                --target test)                                     ┌──────────────────┐
                             2. build image                                        │ nginx → api      │
                             3. scan for vulnerabilities  ──► ai-chat-api:1.0.0 ─► │        ↘ db/redis│
                             4. push image                                         └──────────────────┘
```
Key principle: **you never build on the production server and never copy source code there.** CI builds one immutable, versioned image; the server only *pulls and runs* it. The same bytes you tested are the bytes that run. The workflow for this lives at [`../.github/workflows/ci.yml`](../.github/workflows/ci.yml) and uses the same ideas as the fully annotated pipeline in [Chapter 11](11-production-and-enterprise.md): test, build, scan, push, deploy.

### 9.1 Get a server

Any Linux VM works: AWS EC2, Azure VM, Google Compute Engine, DigitalOcean, Hetzner, or a spare machine. Recommended for this app: **Ubuntu 24.04 LTS, 2 vCPU, 4 GB RAM** (more if you run Ollama), with ports **22 (SSH), 80 and 443** open in the cloud firewall. Production runs on **Linux** because containers share the host's Linux kernel; on Windows/macOS Docker has to run a hidden Linux VM, which adds overhead and complexity ([Chapter 2](02-installation-and-operating-systems.md)).

### 9.2 Prepare the server

SSH in (`ssh youruser@SERVER_IP`) and install Docker Engine using the official apt repository — the exact, current steps are in [Chapter 2](02-installation-and-operating-systems.md). Then:

```bash
sudo usermod -aG docker $USER      # let your user run docker without sudo (log out/in after)
docker run --rm hello-world        # verify the installation
```
Note: membership of the `docker` group is effectively root access on that machine ([Chapter 10](10-security.md)); only add trusted users.

Firewall (Ubuntu's `ufw`) — **caution:** Docker-published ports bypass `ufw` rules because Docker edits `iptables` directly. Safest approach: publish only what must be public (nginx on 80/443) and keep everything else unpublished or bound to `127.0.0.1`. That's exactly how `compose.prod.yaml` is arranged.
```bash
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw enable
```

### 9.3 Publish your image to a registry

CI does this for you (see the workflow). To do it manually from your laptop the first time:

```bash
# Create a GitHub Personal Access Token (classic) with the write:packages scope, then:
echo YOUR_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
docker build --target runtime -t ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0 .
docker push ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0
```
* `--password-stdin` keeps the token out of your shell history and process list.
* The image name format is `registry/owner/name:tag`. `1.0.0` is an **immutable version tag** — never deploy `latest` to production, because you can't tell what it points to or roll back reliably.

### 9.4 Put the config on the server

Only a few small files are needed — *not* the source code:

```bash
sudo mkdir -p /opt/ai-chat && sudo chown $USER /opt/ai-chat
# from your laptop:
scp compose.yaml compose.prod.yaml youruser@SERVER_IP:/opt/ai-chat/
scp -r nginx youruser@SERVER_IP:/opt/ai-chat/
```
On the server, create `/opt/ai-chat/.env`:

```env
POSTGRES_PASSWORD=<generate a long random one: openssl rand -base64 24>
API_IMAGE=ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0
LLM_PROVIDER=openai
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=<your key>
LLM_MODEL=gpt-4o-mini
```
```bash
chmod 600 /opt/ai-chat/.env        # only your user can read it
```
If the image is private, run `docker login ghcr.io` on the server with a token that has `read:packages`.

> **Production note:** `compose.yaml` still contains a `build:` section for `api`, but `compose.prod.yaml` supplies `image:` and `pull_policy: always`; with `docker compose pull` + `up -d` (no `--build`) Compose uses the registry image. Also remember the dev override is **not** applied because you pass `-f` explicitly.

### 9.5 Launch

```bash
cd /opt/ai-chat
docker compose -f compose.yaml -f compose.prod.yaml pull
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose -f compose.yaml -f compose.prod.yaml ps
curl http://localhost/ready
```
To avoid retyping `-f` flags every time, add `COMPOSE_FILE=compose.yaml:compose.prod.yaml` to the server's `.env` (Compose reads it), or create a shell alias.

I validated that this combined configuration parses cleanly (`docker compose -f compose.yaml -f compose.prod.yaml config --quiet` returned exit code 0 once `API_IMAGE` was set).

### 9.6 HTTPS (required for any real deployment)

The sample's nginx listens on plain HTTP so the tutorial stays simple. In production you need TLS (HTTPS) so API calls and any credentials are encrypted. Easiest options:

* **Caddy** — a web server that obtains and renews free Let's Encrypt certificates automatically. You can replace the `nginx` service with Caddy and a two-line `Caddyfile`:
  ```
  chat.yourdomain.com {
      reverse_proxy api:8000
  }
  ```
  (You need a domain name whose DNS A-record points at your server's public IP, and ports 80/443 open.)
* **nginx + certbot**, **Traefik**, or a **cloud load balancer** (AWS ALB, Azure Front Door, Cloudflare) terminating TLS for you.

Full treatment, including certificates and renewals: [Chapter 11](11-production-and-enterprise.md).

---

## Part 10 — Operate it: logs, backups, updates, rollback

```bash
alias dc='docker compose -f compose.yaml -f compose.prod.yaml'   # shortcut used below
```

**Logs**
```bash
dc logs -f --tail 100 api        # last 100 lines, then follow
```
Logs are rotated (10 MB × 5 files per container) thanks to `compose.prod.yaml`.

**Backup the database** (volumes are *not* backups — a deleted server or volume loses everything):
```bash
dc exec -T db pg_dump -U chat chat | gzip > backup-$(date +%F).sql.gz
```
* `exec -T` disables the pseudo-terminal so output pipes cleanly.
* `pg_dump` writes the database as SQL text; `gzip` compresses it. **Copy backups off the server** (object storage such as S3) and *test restoring* them periodically:
```bash
gunzip -c backup-2026-10-23.sql.gz | dc exec -T db psql -U chat chat
```
(In larger production setups the database usually lives *outside* Docker as a managed service — Amazon RDS, Azure Database for PostgreSQL, Cloud SQL — which handles backups and failover for you. See Chapter 11 for the reasoning.)

**Deploy a new version**
1. Push code → CI builds/tests/scans and pushes `ai-chat-api:1.1.0`.
2. On the server, change `API_IMAGE` in `.env` to `...:1.1.0`.
3. `dc pull && dc up -d` — Compose recreates **only** containers whose image/config changed.

**Roll back** — set `API_IMAGE` back to `...:1.0.0` and run `dc up -d` again. This takes seconds *because* every version is an immutable image sitting in the registry.

**Heads-up on downtime:** with a single `api` container Compose stops the old one and starts the new one, so there's a few-second gap. Zero-downtime strategies (multiple replicas, rolling updates) need an orchestrator or extra tooling — see [Chapter 11](11-production-and-enterprise.md).

**Housekeeping**
```bash
docker system df          # how much disk images/containers/volumes use
docker image prune -f     # remove dangling (untagged) images
```
Never run `docker system prune --volumes` on a server with data you care about.

---

## Part 11 — Clean up

```bash
docker compose --profile ollama down -v     # remove containers, network AND volumes (data loss!)
docker rmi ai-chat-api:local ai-chat-api:test
docker image prune -f
```
Explanation: `down -v` also deletes `pgdata` and `ollama_models` (the latter is re-downloadable but large). `docker rmi` removes images by name. Keep your `.env` out of version control.

---

## Part 12 — Publish it to GitHub

This guide folder is designed to be pushed as a portfolio repository.

```
Docker_Complete_Guide/
├── README.md
├── docs/              ← the 14 chapters
├── sample-ai-app/     ← the runnable project
└── .github/workflows/ci.yml
```

Checklist before pushing:

1. **No secrets.** Run `git status` and confirm `.env` is *not* listed (the `.gitignore` excludes it). Search your tree for `sk-`, `gsk_`, `password` to be sure.
2. `git init`, `git add .`, `git commit -m "Docker complete guide and sample AI app"`.
3. Create an empty repo on GitHub (no README), then:
   ```bash
   git remote add origin https://github.com/YOUR_USERNAME/docker-complete-guide.git
   git branch -M main
   git push -u origin main
   ```
4. GitHub → **Actions** tab: watch the CI workflow run. Fix anything red; a green badge in your README looks great.
5. Add a short "What I learned / how to run it" section and a screenshot or terminal recording (see the portfolio advice in [Chapter 13](13-interview-questions-roadmap-glossary.md)).

---

## Part 13 — Challenges to go further

Do these to move from "followed a tutorial" to "can do it myself". Roughly ordered by difficulty.

1. **Add a `/models` endpoint** that returns the configured provider and model. Rebuild, observe layer caching (only the code layer rebuilds).
2. **Add streaming:** make `/chat` stream tokens (Server-Sent Events) from the provider. Notice why `proxy_buffering off` in nginx matters.
3. **Add a RAG feature:** run a vector database (Qdrant or Postgres with the `pgvector` extension) as another Compose service, add `/ingest` and retrieval into the prompt ([Chapter 8](08-docker-for-ai-and-ml.md) explains the concepts).
4. **Add a background worker** (RQ or Celery + Redis) in the same image with a different `command:`; learn that one image can run as multiple services.
5. **Scale the API:** run `docker compose up -d --scale api=3`. It will fail to publish a fixed port if you had one — fix it by relying on nginx and `expose`. Observe that rate limiting and caching still work because state is in Redis/Postgres. (Hint: nginx must re-resolve `api` to see all replicas; our `resolver` setup helps, but test it.)
6. **Add monitoring:** add Prometheus + Grafana services and expose `/metrics` from the API.
7. **Shrink the image:** try `python:3.12-alpine` or a distroless base and compare size *and* compatibility (some wheels don't exist for musl).
8. **Scan it:** `docker scout quickview ai-chat-api:local` or `trivy image ai-chat-api:local`; fix what you can ([Chapter 10](10-security.md)).
9. **Multi-arch build:** `docker buildx build --platform linux/amd64,linux/arm64 ...` so the same image runs on an Apple-Silicon Mac and an Intel/AMD server.
10. **Translate to Kubernetes:** write a Deployment + Service for the API and run it on a local cluster (kind or minikube) — the natural next step after Compose ([Chapter 11](11-production-and-enterprise.md)).

---

## Summary: what you just learned by doing

| Concept | Where you used it |
|---|---|
| Image vs container | Built `ai-chat-api:local`, ran 4 containers |
| Layers & caching | Dockerfile ordering; fast rebuilds |
| Multi-stage builds | builder → test → runtime; 275 MB final image with no build tools |
| Volumes | `pgdata` survived `docker compose down` |
| Networking & DNS | Containers call each other by service name (`db`, `redis`, `ollama`) |
| Port publishing | `0.0.0.0:8080` vs `127.0.0.1:8000` vs unpublished Redis |
| Health checks | `depends_on: service_healthy`, `/health` vs `/ready` |
| Config & secrets | `.env`, `${VAR:?}`, nothing baked into the image |
| Hardening | non-root, read-only FS, dropped capabilities |
| Profiles | Optional Ollama service |
| Dev vs prod | override files, `watch`, pinned registry images, log rotation |
| CI/CD & deploy | Build once, scan, push, pull on server, roll back by tag |

Next: [Chapter 10 — Security](10-security.md) to harden what you built, and [Chapter 11 — Production and Enterprise](11-production-and-enterprise.md) for the scaling, orchestration and operations story.
