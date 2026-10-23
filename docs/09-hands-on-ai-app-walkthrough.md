# Chapter 9 — Hands-On: Build, Run and Deploy an AI Chat App with Docker

This chapter applies the concepts from Chapters 1–8 to a working application. The code is in [`../sample-ai-app/`](../sample-ai-app/). The stack consists of an nginx reverse proxy, a FastAPI service, Postgres, Redis and a pluggable LLM backend.

```
  HOST (all components run as Docker containers)

  browser/curl ──► nginx ──► api (FastAPI) ──┬──► Postgres  (chat history)
               :8080  reverse   app code     ├──► Redis     (cache, rate limit)
                      proxy                  └──► LLM       (mock by default, or an
                                                            OpenAI-compatible API,
                                                            or Ollama)
```

**Verification status.** Parts 1–8 were executed on Windows 11 with Docker Desktop (Docker Engine 29.9.2, Compose 5.6.1). The outputs shown as "Real output" come from those runs. Part 9 (Linux server deployment) follows the standard procedure and was **not** executed against a real cloud server. Adapt hostnames and IPs.

**Time needed:** about 2–3 hours.

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

| Tool | Purpose | Check with |
|---|---|---|
| Docker Desktop (Windows/macOS) or Docker Engine (Linux) | Runs containers. See [Chapter 2](02-installation-and-operating-systems.md). | `docker --version` |
| Docker Compose v2+ (bundled with Docker Desktop) | Starts multi-container apps from one file. See [Chapter 6](06-docker-compose.md). | `docker compose version` |
| A terminal (PowerShell, Terminal, bash) | Runs commands | — |
| A code editor (for example VS Code) | Reads and edits files | — |

Python is not required on the host; it lives inside the image. Docker must be running: on Windows/macOS, start Docker Desktop and wait for "Engine running". The error `failed to connect to the docker API ... dockerDesktopLinuxEngine` means the engine is not started.

### What the app does

It is a chat API. A request such as `{"session_id": "demo", "message": "What is Docker?"}` goes through these steps:

1. **Rate limit check** in Redis: 30 requests per minute per IP.
2. **Load history**: the last 10 messages of the session from Postgres.
3. **Cache lookup** in Redis for an identical prompt. A hit skips the model call.
4. **LLM call**: the mock model by default, or a real one if configured.
5. **Save** the message and the answer to Postgres, then return the answer.

**Glossary**

- **LLM**: Large Language Model, the AI model (for example GPT or Llama) that generates text.
- **Reverse proxy**: a server (here nginx) in front of the app that forwards incoming requests to it.
- **Rate limiting**: rejecting requests from a client that sends too many in a short time.
- **Cache**: a fast temporary store of earlier results, used to skip repeated work.

---

## Part 2 — Tour of the project, file by file

```
sample-ai-app/
├── app/
│   ├── main.py            ← API endpoints (FastAPI)
│   ├── llm.py             ← model client (mock / OpenAI-compatible)
│   └── config.py          ← settings from environment variables
├── tests/test_llm.py      ← unit tests
├── nginx/nginx.conf       ← reverse proxy config
├── Dockerfile             ← image recipe for the API
├── compose.yaml           ← all containers and their connections
├── compose.override.yaml  ← development settings (applied automatically)
├── compose.prod.yaml      ← production settings (opt-in)
├── requirements.txt       ← Python libraries (pinned versions)
├── .env.example           ← configuration template
└── .dockerignore          ← files not sent to the builder
```

### 2.1 `config.py` — configuration from the environment

```python
class Settings(BaseSettings):
    llm_provider: str = "mock"
    database_url: str = "postgresql://chat:chat@db:5432/chat"
    redis_url: str = "redis://redis:6379/0"
    ...
```

- `pydantic-settings` fills each field from the environment variable with the same name (case-insensitive), for example `LLM_PROVIDER`. The value after `=` is the default.
- One image runs in every environment; only environment variables differ (Twelve-Factor App, rule III: store config in the environment).
- The hostnames `db` and `redis` are Compose service names. Inside a Compose network, each service name is a DNS name that resolves to the container's IP. See [Chapter 5](05-storage-and-networking.md).

### 2.2 `llm.py` — swap models without changing code

```python
def complete(messages):
    if settings.llm_provider == "mock":   return _mock(messages)
    if settings.llm_provider == "openai": return _openai_compatible(messages)
```

The `openai` provider sends `POST {LLM_BASE_URL}/chat/completions`. Hosted services (OpenAI, Groq, Azure OpenAI) and local servers (Ollama, vLLM, llama.cpp) accept this "OpenAI-compatible" format. Switching provider means changing `LLM_BASE_URL`.

### 2.3 `main.py` — the endpoints

| Endpoint | Purpose |
|---|---|
| `GET /health` | Liveness: the process is alive. Returns `{"status":"ok"}`. Used by the Docker `HEALTHCHECK`. |
| `GET /ready` | Readiness: the app can serve traffic. Runs `SELECT 1` on Postgres and `PING` on Redis. Orchestrators such as Kubernetes distinguish the two ([Chapter 11](11-production-and-enterprise.md)). |
| `POST /chat` | The logic described in Part 1. |
| `GET /history/{session_id}` | Returns saved messages. |

- The handlers are plain `def`, not `async def`, because the database, Redis and HTTP libraries are synchronous. FastAPI runs them in a thread pool.
- The `lifespan` function runs once at startup. It opens the Postgres connection pool, creates the table if missing and connects to Redis.

### 2.4 `Dockerfile` — the image recipe (multi-stage)

[`../sample-ai-app/Dockerfile`](../sample-ai-app/Dockerfile) has three stages. A **stage** is a separate build step. Later stages can copy files from earlier ones, and everything else is discarded.

```dockerfile
# syntax=docker/dockerfile:1
```
Selects the current Dockerfile syntax (BuildKit frontend). Keep it as the first line.

**Stage 1 — `builder`: install the libraries**

```dockerfile
FROM python:3.12-slim AS builder
ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1
WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
```

| Instruction | Effect |
|---|---|
| `FROM python:3.12-slim AS builder` | Starts from the official Python 3.12 "slim" image (about 50 MB compressed versus about 350 MB for the full image) and names the stage `builder`. |
| `ENV PIP_DISABLE_PIP_VERSION_CHECK=1 PIP_NO_CACHE_DIR=1` | Disables pip's update notice and its download cache (smaller image). |
| `WORKDIR /build` | Sets the working directory and creates it. |
| `RUN python -m venv /opt/venv` | Creates a virtual environment (an isolated folder of installed libraries). All libraries end up in one folder that the final stage can copy. |
| `ENV PATH=...` | Puts the virtual environment first on `PATH` so `pip` and `python` use it. |

```dockerfile
COPY requirements.txt .
RUN pip install -r requirements.txt
```
Copies only the dependency list, then installs. Docker caches each instruction's result as a **layer**. While `requirements.txt` is unchanged, the install layer is reused even when application code changes. Copying all code first would reinstall every library on each code edit.

**Stage 2 — `test`: run unit tests during the build**

```dockerfile
FROM builder AS test
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt
COPY app/ app/
COPY tests/ tests/
RUN python -m pytest -q
```
Starts from `builder`, adds pytest, copies code and tests, and runs the tests as a build step. A failing test fails `docker build`. The stage is built only with `--target test` (or when CI requests it).

**Stage 3 — `runtime`: the image that ships**

```dockerfile
FROM python:3.12-slim AS runtime
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1 PATH="/opt/venv/bin:$PATH"
COPY --from=builder /opt/venv /opt/venv
RUN useradd --system --uid 10001 --no-create-home appuser
COPY --chown=appuser:appuser app/ app/
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=15s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2).status==200 else 1)"
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

| Instruction | Effect |
|---|---|
| `FROM python:3.12-slim AS runtime` | Fresh slim image with no build leftovers. |
| `PYTHONUNBUFFERED=1` | Python writes logs immediately instead of buffering them. |
| `PYTHONDONTWRITEBYTECODE=1` | Skips `.pyc` files (the filesystem is read-only at runtime). |
| `COPY --from=builder /opt/venv /opt/venv` | Copies only the finished virtual environment from the builder stage. |
| `RUN useradd --system --uid 10001 --no-create-home appuser` | Creates an unprivileged user. |
| `COPY --chown=appuser:appuser app/ app/` | Copies the code, owned by that user. |
| `USER appuser` | All later instructions and the container process run as this user, not root. A compromised app then has no root privileges inside the container ([Chapter 10](10-security.md)). |
| `EXPOSE 8000` | Documentation only. It does not publish the port. |
| `HEALTHCHECK ...` | Docker runs the command inside the container every 15 s (`--interval`). Exit code 0 means healthy, 1 means unhealthy. `--timeout=3s` limits one check. `--start-period=10s` ignores failures during boot. `--retries=3` marks the container `unhealthy` after 3 consecutive failures. Python's `urllib` is used because slim images have no `curl`. |
| `CMD [...]` | Default command in exec form (JSON array). uvicorn runs as PID 1 and receives SIGTERM directly, so `docker stop` shuts it down gracefully instead of waiting 10 s and killing it. `--host 0.0.0.0` listens on all interfaces. `127.0.0.1` inside a container is reachable only from that container. |

### 2.5 `compose.yaml` — the whole stack

[`../sample-ai-app/compose.yaml`](../sample-ai-app/compose.yaml):

```yaml
name: ai-chat
```
The project name. Compose prefixes container, network and volume names with it (`ai-chat-api-1`, `ai-chat_pgdata`). There is no `version:` line because it is obsolete in modern Compose.

**`nginx` service**

```yaml
nginx:
  image: nginx:1.27-alpine
  ports: ["8080:80"]
  volumes: ["./nginx/nginx.conf:/etc/nginx/nginx.conf:ro"]
  depends_on:
    api: { condition: service_healthy }
```

| Key | Meaning |
|---|---|
| `image:` | Use a ready-made image from Docker Hub (`-alpine` is a small variant). |
| `ports: "8080:80"` | Publish: host port 8080 forwards to container port 80. Format is `HOST:CONTAINER`. |
| `volumes: ./nginx/nginx.conf:...:ro` | Bind mount: the host file appears in the container at that path. `:ro` is read-only. This injects config without a custom image. |
| `depends_on ... service_healthy` | Start nginx only after the API health check passes. Plain `depends_on` waits only for the container to start, not to be ready. |

**`api` service**

```yaml
api:
  build: { context: ., target: runtime }
  image: ai-chat-api:local
  environment:
    DATABASE_URL: postgresql://${POSTGRES_USER:-chat}:${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}@db:5432/${POSTGRES_DB:-chat}
```

| Key | Meaning |
|---|---|
| `build.context: .` | Send this folder to the builder. |
| `build.target: runtime` | Stop at the final stage, skipping `test`. |
| `image:` | Name given to the built image. |
| `${VAR:-default}` | Use `$VAR` from the shell or `.env`, or `default` if unset. |
| `${VAR:?message}` | Fail with `message` if `$VAR` is unset. Prevents starting the stack without a password. |
| `@db:5432` | The host is the service name `db`. |

```yaml
  expose: ["8000"]
  depends_on:
    db:    { condition: service_healthy }
    redis: { condition: service_healthy }
  restart: unless-stopped
  read_only: true
  tmpfs: [ /tmp ]
  cap_drop: [ ALL ]
  security_opt: [ "no-new-privileges:true" ]
```

| Key | Meaning |
|---|---|
| `expose` | Port 8000 is reachable by other containers on the network but is not published to the host. In production only nginx reaches the API. The dev override adds a localhost-only publish. |
| `depends_on` | Start the API after Postgres and Redis are healthy. |
| `restart: unless-stopped` | Restart after a crash or a Docker/host restart, unless the container was stopped manually. |
| `read_only: true` | Makes the container root filesystem immutable (verified in Part 6). |
| `tmpfs: [/tmp]` | RAM-backed writable `/tmp`, for programs that need scratch space. |
| `cap_drop: [ALL]` | Removes all Linux capabilities (special privileges such as binding low ports or changing file owners). The app needs none. |
| `no-new-privileges:true` | Blocks privilege escalation inside the container. |

**`db` service**

```yaml
db:
  image: postgres:16-alpine
  environment: { POSTGRES_USER: ..., POSTGRES_PASSWORD: ..., POSTGRES_DB: ... }
  volumes: [ "pgdata:/var/lib/postgresql/data" ]
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER:-chat} -d $${POSTGRES_DB:-chat}"]
```

- The Postgres image creates the user and database from these variables on first start.
- `pgdata:/var/lib/postgresql/data` is a **named volume**: Docker-managed storage that survives container deletion. It is declared under `volumes:` at the bottom of the file. Without it, chat history is lost when the container is removed.
- `$$` is a literal `$`. Compose would otherwise substitute it. The check runs inside the container, where those variables exist.
- `pg_isready` is Postgres's tool for checking that the server accepts connections.

> **Postgres version note.** The mount path above is correct for `postgres:16` (and 17 and earlier). Postgres 18+ images changed their layout and expect the volume at `/var/lib/postgresql`. When changing the image tag to 18 or newer, change the mount path too (see [Chapter 5](05-storage-and-networking.md)) and test that data persists.

**`redis` service.** `command:` overrides the default start command to cap memory at 64 MB and evict least-recently-used keys when full. The health check is `redis-cli ping` (expects `PONG`). Redis has no volume because it is only a cache.

**`ollama` service**

```yaml
ollama:
  image: ollama/ollama:latest
  profiles: ["ollama"]
```
A service with `profiles` is skipped unless started with `--profile ollama`. It is an optional local LLM server (Part 7). The GPU block is commented out; enable it only on a Linux machine with an NVIDIA GPU.

### 2.6 `compose.override.yaml` — development extras

Compose merges `compose.yaml` and `compose.override.yaml` automatically on a plain `docker compose up`. The override:

- publishes the API on `127.0.0.1:8000` and Postgres on `127.0.0.1:5432`. Binding to `127.0.0.1` allows connections only from the local machine.
- overrides the command with `--reload`.
- defines `develop.watch` rules for hot reload (Part 8).

### 2.7 `compose.prod.yaml` — production extras

Used only with `-f compose.yaml -f compose.prod.yaml`. Passing `-f` also disables the automatic dev override. It:

- replaces `build:` with a pre-built versioned `image: ${API_IMAGE}` and `pull_policy: always`.
- adds log rotation (`max-size 10m`, `max-file 5`). Without it, Docker's default `json-file` logs grow until the disk is full.
- sets memory and CPU limits.
- publishes nginx on port 80.

### 2.8 `nginx.conf`

- `resolver 127.0.0.11` with `set $upstream http://api:8000;`: `127.0.0.11` is Docker's embedded DNS server. A variable in `proxy_pass` makes nginx re-resolve the name periodically instead of caching the IP at startup. nginx keeps working when the `api` container is recreated with a new IP.
- `proxy_read_timeout 120s; proxy_buffering off;`: LLM responses are slow and often streamed. The default 60 s timeout and buffering break that.

### 2.9 `.dockerignore`

The Docker client sends the build context (the project folder) to the builder. `.dockerignore` excludes `.git`, `.env`, caches and docs. Secrets such as `.env` must never enter an image. A smaller context also builds faster.

---

## Part 3 — Build the image and run the tests

```bash
cd Docker_Complete_Guide/sample-ai-app
```

### 3.1 Run the tests inside Docker

```bash
docker build --target test --progress=plain -t ai-chat-api:test .
```

| Part | Meaning |
|---|---|
| `docker build` | Build an image from a Dockerfile. |
| `--target test` | Stop at the stage named `test` (runs pytest). |
| `--progress=plain` | Print full text logs instead of the animated view. |
| `-t ai-chat-api:test` | Name the result `repository:tag`. |
| `.` | Build context: the current directory. |

Real output (trimmed):

```
#16 0.770 ..                                                                       [100%]
#16 0.772 2 passed in 0.13s
#16 DONE 0.8s
...
#17 naming to docker.io/library/ai-chat-api:test done
```

The tests run inside the container with the same Python and library versions as production.

A second run finishes almost instantly because every layer comes from cache. After adding a blank line to `tests/test_llm.py`, only the layers after that change re-run and `pip install` stays cached. See [Chapter 4](04-dockerfile-deep-dive.md).

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

Docker Desktop with the containerd image store reports slightly different sizes than the classic store.

```bash
docker history ai-chat-api:local
```

`docker history` lists the layers. The Python base layers are at the bottom and the app's own layers on top, for example `75.1MB COPY /opt/venv /opt/venv` (libraries) and `28.7kB COPY ... app/` (code). The Dockerfile puts the large, rarely changing libraries before the small, frequently changing code for this reason.

---

## Part 4 — Start the whole stack with Compose

### 4.1 Create your `.env`

```bash
cp .env.example .env          # Windows PowerShell: Copy-Item .env.example .env
```

Set `POSTGRES_PASSWORD` in `.env` and keep `LLM_PROVIDER=mock`.

`.env` holds secrets and per-machine settings. `.gitignore` excludes it. The committed `.env.example` documents the variables.

To test the `:?` guard, remove the password line and run `docker compose config`. It fails with `error while interpolating ... Set POSTGRES_PASSWORD in .env`.

### 4.2 Start everything

```bash
docker compose up -d --build
```

| Part | Meaning |
|---|---|
| `docker compose up` | Create and start all services in `compose.yaml`. |
| `-d` | Detached: run in the background. |
| `--build` | Rebuild images first if the Dockerfile or code changed. |

Real output (trimmed):

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

Compose creates the network and volume, starts `db` and `redis`, waits for them to be healthy, starts `api`, waits for it to be healthy, then starts `nginx`. This order comes from the `depends_on ... service_healthy` rules.

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

| PORTS value | Meaning |
|---|---|
| `0.0.0.0:8080->80/tcp` | Published on all interfaces: reachable from other machines. |
| `127.0.0.1:8000->8000/tcp` | Published to localhost only (dev override). |
| `6379/tcp` (no arrow) | Not published: reachable only from containers on the same network. |

---

## Part 5 — Use the app

### 5.1 Health and readiness

```bash
curl http://localhost:8080/ready
```
```
{"status":"ready"}
```
The request path is: host port 8080, nginx container, Docker network, api container. The api container runs `SELECT 1` on `db` and `PING` on `redis`.

### 5.2 The interactive docs

Open <http://localhost:8080/docs>. FastAPI generates a Swagger page for trying each endpoint.

### 5.3 Send a chat message

**macOS / Linux / Git Bash:**
```bash
curl -X POST http://localhost:8080/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id":"demo","message":"What is Docker?"}'
```

**Windows PowerShell.** Windows PowerShell 5.1 strips the double quotes from inline JSON passed to `curl.exe`, which produces a `json_invalid` error. This occurred during testing. Use the native cmdlet:
```powershell
Invoke-RestMethod -Method Post -Uri http://localhost:8080/chat `
  -ContentType "application/json" `
  -Body '{"session_id":"demo","message":"What is Docker?"}'
```
Alternatively, save the JSON in a file and use `curl.exe -d "@body.json"`.

Real response:

```json
{"reply":"[mock model] You said: 'What is Docker?'. This conversation has 1 user message(s). Set LLM_PROVIDER=openai in .env to talk to a real model.","cached":false,"provider":"mock"}
```

Sending the same request again returns `... has 2 user message(s)`: the history comes from Postgres.

### 5.4 View the saved history

```bash
curl http://localhost:8080/history/demo
```
Returns four rows (two user messages and two replies), each with a timestamp.

### 5.5 See the cache work

The cache key is built from the entire prompt (provider, model, history and new message). Within one session the history grows, so the prompt never repeats. Two different sessions that send the same first message produce the same prompt:

```bash
# session "a" says "hi"  → cached: false
# session "b" says "hi"  → cached: true   (model call skipped)
```
In testing, the first call returned `"cached":false` and a later call from another session returned `"cached":true`.

---

## Part 6 — Experiments that teach you how Docker really behaves

### Experiment 1 — Containers are disposable, volumes are not

```bash
docker compose down            # removes containers and network; volumes are kept
docker compose up -d           # recreates everything
curl http://localhost:8080/history/demo
```
Result in testing: still 4 messages. The Postgres data stayed in the `pgdata` volume. Wait about 15 seconds after `up` for the health checks.

```bash
docker compose down -v         # -v also deletes named volumes: data loss
```
The history is deleted. `down -v` deletes the database volume.

The Redis cache is emptied by `down` because Redis has no volume. In testing, a `cached: true` response became `cached: false` after `down` and `up`.

### Experiment 2 — The hardened container

```bash
docker compose exec api sh -c "id; touch /srv/x; touch /tmp/ok && echo tmp-writable"
```

`docker compose exec api ...` runs a command inside the running `api` container.

Real output:

```
uid=10001(appuser) gid=999(appuser) groups=999(appuser)
touch: cannot touch '/srv/x': Read-only file system
tmp-writable
```
Results: the process runs as non-root (`uid=10001`), the root filesystem is read-only (writing to `/srv` fails) and the `/tmp` tmpfs is writable.

### Experiment 3 — Rate limiting

Send 32 requests (PowerShell):

```powershell
1..32 | ForEach-Object { curl.exe -s -o NUL -w "%{http_code} " -X POST http://localhost:8080/chat -H "Content-Type: application/json" -d "@body.json" }
```

| Part | Meaning |
|---|---|
| `-s` | Silent: no progress output. |
| `-o NUL` | Discard the response body (Windows; use `/dev/null` on Linux/macOS). |
| `-w "%{http_code} "` | Print only the HTTP status code. |
| `-d "@body.json"` | Send the file contents as the request body. |

Real result: `200` thirty times, then `429 429` (HTTP 429 is Too Many Requests). The counter is in Redis, keyed by client IP and the current minute. Because the state is external, the limit stays correct with multiple API copies.

### Experiment 4 — Crash recovery (`restart: unless-stopped`)

The restart policy applies to unexpected exits. Docker treats a stop requested through Docker as intentional. Both cases were tested on a real run.

**A) Kill the container through Docker**
```bash
docker kill --signal=SIGKILL ai-chat-api-1
docker inspect --format 'status={{.State.Status}} restarts={{.RestartCount}} exit={{.State.ExitCode}}' ai-chat-api-1
```
Real output: `status=exited restarts=0 exit=137`. The container was not restarted. Exit code 137 is 128 + 9 (SIGKILL). `docker compose kill api` behaves the same way.

**B) Make the main process exit from inside**
```bash
docker compose up -d                                  # bring api back first
docker compose exec -T api sh -c "kill 1"             # PID 1 = uvicorn
docker inspect --format 'status={{.State.Status}} restarts={{.RestartCount}}' ai-chat-api-1
```
Real output: `status=running restarts=1`. Docker detected the process exit and restarted the container.

To test crash recovery, make the process die inside the container; `docker kill` and `docker stop` count as deliberate. `kill 1` worked because uvicorn installs a SIGTERM handler. PID 1 in Linux gets no default signal behaviour, so a process without a handler would ignore the signal. This is a reason to run the server as PID 1 with exec-form `CMD`.

### Experiment 5 — Break a dependency and watch health checks

```bash
docker compose stop redis
curl http://localhost:8080/ready      # 503 "dependency down"
curl http://localhost:8080/health     # still 200
docker compose start redis
```
Real output: `/ready` returns `{"detail":"dependency down: Error -5 connecting to redis:6379. No address associated with hostname."} [503]`. `/health` returns `{"status":"ok"} [200]`. The "no address" text appears because a stopped container's name is removed from Docker's DNS.

`/health` reports that the process is alive, so Docker should not restart it. `/ready` reports that the app cannot serve users, so a load balancer should route traffic elsewhere.

### Experiment 6 — Look inside

```bash
docker compose logs -f api              # follow logs (Ctrl+C to stop)
docker compose exec db psql -U chat -d chat -c "SELECT id, session_id, role, left(content, 40) FROM messages;"
docker compose exec redis redis-cli keys '*'
docker network ls
docker network inspect ai-chat_default
docker volume ls
docker volume inspect ai-chat_pgdata
docker stats --no-stream
```

| Command | Shows |
|---|---|
| `docker compose logs -f api` | Live logs of the `api` service. |
| `psql` (via `exec db`) | Rows in the real chat database. `-U` user, `-d` database, `-c` SQL to run. |
| `redis-cli keys '*'` | Cache and rate-limit keys (`llm:<hash>`, `rl:<ip>:<minute>`). |
| `docker network inspect` | The Compose network and each container's IP on it. |
| `docker volume inspect` | The volume `Mountpoint`. On Docker Desktop this path is inside Docker's Linux VM, not on the host filesystem. |
| `docker stats --no-stream` | One-time CPU and memory snapshot per container. |

---

## Part 7 — Plug in a real AI model

### 7.1 Option A: a hosted API (OpenAI, Groq, Azure OpenAI, ...)

Edit `.env`:

```env
LLM_PROVIDER=openai
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=sk-...your-key...
LLM_MODEL=gpt-4o-mini
```

Groq example (free tier at the time of writing; check current terms and model names):

```env
LLM_BASE_URL=https://api.groq.com/openai/v1
LLM_API_KEY=gsk_...
LLM_MODEL=llama-3.3-70b-versatile
```

Apply the change. Compose detects the changed environment and recreates only the `api` container:

```bash
docker compose up -d
```

**Security notes**

- The API key is a runtime environment variable. It is not in the image (`docker history ai-chat-api:local` shows no key) and not in git (`.env` is ignored).
- Anyone who can run `docker inspect` on the host can read environment variables. For stricter setups use Docker/Compose secrets or a cloud secret manager ([Chapter 10](10-security.md)).
- Set a spending limit on the provider account. The rate limiter is not a budget.
- Revoke and rotate a leaked key immediately. Deleting the commit is not enough.

### 7.2 Option B: run a model locally with Ollama in Docker

**Ollama** is a program that downloads and serves open-source LLMs (Llama, Qwen, Mistral, ...) behind an API.

```bash
docker compose --profile ollama up -d
docker compose exec ollama ollama pull llama3.2:1b
```

| Part | Meaning |
|---|---|
| `--profile ollama` | Activates the optional `ollama` service. |
| `docker compose exec ollama ...` | Runs a command inside the Ollama container. |
| `ollama pull llama3.2:1b` | Downloads the model (about 1.3 GB; check disk space) to `/root/.ollama`, which is the `ollama_models` named volume. Weights persist across container recreation. |

Point the API at it in `.env`:

```env
LLM_PROVIDER=openai
LLM_BASE_URL=http://ollama:11434/v1
LLM_API_KEY=ollama
LLM_MODEL=llama3.2:1b
LLM_TIMEOUT_SECONDS=180
```
`ollama` in the URL is the service name. The key can be any non-empty text; Ollama ignores it. CPU-only inference is slow, so the timeout is longer.

```bash
docker compose --profile ollama up -d      # recreate api with the new env
```
Include `--profile ollama` in later `up` and `down` commands. Without it, Compose ignores the service (for example `down` does not stop it).

**GPU acceleration.** On Linux with an NVIDIA GPU, the NVIDIA driver and the NVIDIA Container Toolkit, uncomment the `deploy.resources.reservations.devices` block in `compose.yaml`. It can work on Windows with WSL2 GPU support. Containers on macOS cannot use the Apple GPU. See [Chapter 8](08-docker-for-ai-and-ml.md).

Switching between mock, OpenAI and local Llama changed only environment variables. The code and image stayed identical.

---

## Part 8 — The developer workflow: hot reload

Rebuilding the image after each edit is slow. Compose watch mode syncs edits into the running container:

```bash
docker compose watch          # or:  docker compose up --watch
```
`compose.override.yaml` (applied automatically in development) contains:

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
Tested: after changing the mock reply text in `app/llm.py`, the watcher printed

```
Watch enabled
Syncing service "api" after 1 changes were detected
 Container ai-chat-api-1 Restarting
service(s) ["api"] restarted
```
and the next `/chat` call returned the new text, within a few seconds and without a manual rebuild.

**Issue found in testing.** The first attempt failed with `Error response from daemon: container rootfs is marked read-only`. `watch` copies files into the container, and Docker refuses that when the root filesystem is read-only. The fix is `read_only: false` in the dev override only. Production (`compose.yaml` + `compose.prod.yaml`) keeps the read-only filesystem.

**Watch versus bind mount.** On Windows/macOS, bind-mounted files cross a virtualization boundary. This is slower, and file-change notifications (`inotify`) are unreliable, so `uvicorn --reload` can miss edits. `sync` copies files reliably. On a Linux development machine, a bind mount also works well.

---

## Part 9 — Deploy to a Linux server (production)

Unverified: this part follows the standard procedure and was not run against a real cloud VM.

### 9.0 The production picture

```
 Laptop            GitHub                      Registry (GHCR)         Linux server (VM)
 ──────            ──────                      ───────────────         ─────────────────
 git push ──────►  Actions CI:                                         docker compose pull
                   1. test (docker build                               docker compose up -d
                      --target test)                                   ┌──────────────────┐
                   2. build image                                      │ nginx → api      │
                   3. scan for vulnerabilities ──► ai-chat-api:1.0.0 ─►│        ↘ db/redis│
                   4. push image                                       └──────────────────┘
```
Principle: do not build on the production server and do not copy source code there. CI builds one immutable, versioned image and the server only pulls and runs it. The workflow is [`../.github/workflows/ci.yml`](../.github/workflows/ci.yml). It follows the pipeline annotated in [Chapter 11](11-production-and-enterprise.md): test, build, scan, push, deploy.

### 9.1 Get a server

Any Linux VM works (AWS EC2, Azure VM, Google Compute Engine, DigitalOcean, Hetzner, a spare machine).

- Recommended: Ubuntu 24.04 LTS, 2 vCPU, 4 GB RAM (more with Ollama).
- Open ports 22 (SSH), 80 and 443 in the cloud firewall.
- Production uses Linux because containers share the host's Linux kernel. On Windows/macOS, Docker runs a hidden Linux VM ([Chapter 2](02-installation-and-operating-systems.md)).

### 9.2 Prepare the server

Connect with `ssh youruser@SERVER_IP` and install Docker Engine from the official apt repository (steps in [Chapter 2](02-installation-and-operating-systems.md)). Then:

```bash
sudo usermod -aG docker $USER      # run docker without sudo (log out and in afterwards)
docker run --rm hello-world        # verify the installation
```
Membership of the `docker` group is effectively root access on the machine ([Chapter 10](10-security.md)). Add only trusted users.

Docker-published ports bypass `ufw` rules because Docker edits `iptables` directly. Publish only what must be public (nginx on 80/443) and keep everything else unpublished or bound to `127.0.0.1`, as `compose.prod.yaml` does.
```bash
sudo ufw allow OpenSSH && sudo ufw allow 80,443/tcp && sudo ufw enable
```

### 9.3 Publish your image to a registry

CI does this automatically. To do it manually:

```bash
# Create a GitHub Personal Access Token (classic) with the write:packages scope, then:
echo YOUR_TOKEN | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
docker build --target runtime -t ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0 .
docker push ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0
```
- `--password-stdin` keeps the token out of shell history and the process list.
- Image name format: `registry/owner/name:tag`. `1.0.0` is an immutable version tag. Do not deploy `latest` to production: it does not identify a version and cannot be rolled back reliably.

### 9.4 Put the config on the server

Only a few small files are needed, not the source code:

```bash
sudo mkdir -p /opt/ai-chat && sudo chown $USER /opt/ai-chat
# from your laptop:
scp compose.yaml compose.prod.yaml youruser@SERVER_IP:/opt/ai-chat/
scp -r nginx youruser@SERVER_IP:/opt/ai-chat/
```
Create `/opt/ai-chat/.env` on the server:

```env
POSTGRES_PASSWORD=<generate a long random one: openssl rand -base64 24>
API_IMAGE=ghcr.io/YOUR_GITHUB_USERNAME/ai-chat-api:1.0.0
LLM_PROVIDER=openai
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=<your key>
LLM_MODEL=gpt-4o-mini
```
```bash
chmod 600 /opt/ai-chat/.env        # readable by the owner only
```
For a private image, run `docker login ghcr.io` on the server with a token that has `read:packages`.

`compose.yaml` still has a `build:` section for `api`, but `compose.prod.yaml` supplies `image:` and `pull_policy: always`. With `docker compose pull` and `up -d` (no `--build`), Compose uses the registry image. The dev override is not applied because `-f` is passed explicitly.

### 9.5 Launch

```bash
cd /opt/ai-chat
docker compose -f compose.yaml -f compose.prod.yaml pull
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose -f compose.yaml -f compose.prod.yaml ps
curl http://localhost/ready
```
To avoid repeating `-f`, add `COMPOSE_FILE=compose.yaml:compose.prod.yaml` to the server's `.env`, or use a shell alias.

Verified locally: `docker compose -f compose.yaml -f compose.prod.yaml config --quiet` returned exit code 0 once `API_IMAGE` was set.

### 9.6 HTTPS (required for any real deployment)

The sample's nginx serves plain HTTP. Production needs TLS (HTTPS) so traffic and credentials are encrypted. Options:

- **Caddy**: a web server that obtains and renews Let's Encrypt certificates automatically. Replace the `nginx` service with Caddy and this `Caddyfile`:
  ```
  chat.yourdomain.com {
      reverse_proxy api:8000
  }
  ```
  This requires a domain whose DNS A-record points at the server's public IP, and ports 80/443 open.
- **nginx + certbot**, **Traefik**, or a **cloud load balancer** (AWS ALB, Azure Front Door, Cloudflare) that terminates TLS.

Details: [Chapter 11](11-production-and-enterprise.md).

---

## Part 10 — Operate it: logs, backups, updates, rollback

```bash
alias dc='docker compose -f compose.yaml -f compose.prod.yaml'   # shortcut used below
```

**Logs**
```bash
dc logs -f --tail 100 api        # last 100 lines, then follow
```
Logs are rotated (10 MB x 5 files per container) by `compose.prod.yaml`.

**Back up the database.** Volumes are not backups: deleting the server or volume loses the data.
```bash
dc exec -T db pg_dump -U chat chat | gzip > backup-$(date +%F).sql.gz
```
- `exec -T` disables the pseudo-terminal so output pipes cleanly.
- `pg_dump` writes the database as SQL text. `gzip` compresses it.
- Copy backups off the server (for example to S3) and test restores periodically:
```bash
gunzip -c backup-2026-10-23.sql.gz | dc exec -T db psql -U chat chat
```
Larger setups usually run the database outside Docker as a managed service (Amazon RDS, Azure Database for PostgreSQL, Cloud SQL) that handles backups and failover. See [Chapter 11](11-production-and-enterprise.md).

**Deploy a new version**
1. Push code. CI builds, tests, scans and pushes `ai-chat-api:1.1.0`.
2. On the server, set `API_IMAGE` in `.env` to `...:1.1.0`.
3. Run `dc pull && dc up -d`. Compose recreates only containers whose image or config changed.

**Roll back.** Set `API_IMAGE` back to `...:1.0.0` and run `dc up -d`. This is fast because every version is an immutable image in the registry.

**Downtime.** With a single `api` container, Compose stops the old container and starts the new one, which leaves a gap of a few seconds. Zero-downtime updates (multiple replicas, rolling updates) need an orchestrator or extra tooling ([Chapter 11](11-production-and-enterprise.md)).

**Housekeeping**
```bash
docker system df          # disk used by images, containers and volumes
docker image prune -f     # remove dangling (untagged) images
```
Do not run `docker system prune --volumes` on a server with data you need.

---

## Part 11 — Clean up

```bash
docker compose --profile ollama down -v     # remove containers, network AND volumes (data loss)
docker rmi ai-chat-api:local ai-chat-api:test
docker image prune -f
```
`down -v` also deletes `pgdata` and `ollama_models` (re-downloadable but large). `docker rmi` removes images by name.

---

## Part 12 — Publish it to GitHub

Checklist before pushing the guide folder as a portfolio repository:

1. Confirm no secrets: `git status` must not list `.env`. Search the tree for `sk-`, `gsk_` and `password`.
2. Run `git init`, `git add .` and `git commit -m "Docker complete guide and sample AI app"`.
3. Create an empty repository on GitHub (no README), then:
   ```bash
   git remote add origin https://github.com/YOUR_USERNAME/docker-complete-guide.git
   git branch -M main
   git push -u origin main
   ```
4. Open the GitHub **Actions** tab and confirm the CI workflow passes.
5. Add a short "how to run it" section ([Chapter 13](13-interview-questions-roadmap-glossary.md) has portfolio advice).

---

## Part 13 — Challenges to go further

| # | Challenge | Notes |
|---|---|---|
| 1 | Add a `/models` endpoint | Rebuild and confirm only the code layer rebuilds. |
| 2 | Stream tokens from `/chat` (Server-Sent Events) | Requires `proxy_buffering off` in nginx. |
| 3 | Add RAG with a vector database (Qdrant, or Postgres with `pgvector`) | New Compose service plus `/ingest`. See [Chapter 8](08-docker-for-ai-and-ml.md). |
| 4 | Add a background worker (RQ or Celery with Redis) | Same image, different `command:`. |
| 5 | Scale with `docker compose up -d --scale api=3` | A fixed published port blocks scaling; use nginx and `expose`. nginx must re-resolve `api` to see all replicas (the `resolver` setup is meant to do this; test it). |
| 6 | Add Prometheus and Grafana | Expose `/metrics` from the API. |
| 7 | Shrink the image (`python:3.12-alpine` or distroless) | Compare size and compatibility; some wheels do not exist for musl. |
| 8 | Scan the image: `docker scout quickview ai-chat-api:local` or `trivy image ai-chat-api:local` | [Chapter 10](10-security.md). |
| 9 | Multi-architecture build: `docker buildx build --platform linux/amd64,linux/arm64 ...` | |
| 10 | Deploy to Kubernetes (Deployment and Service on kind or minikube) | [Chapter 11](11-production-and-enterprise.md). |

---

## Summary

- Multi-stage build: builder, test, runtime (275 MB final image, no build tools).
- Layer caching: dependencies installed before code is copied.
- `pgdata` volume survives `docker compose down`; `down -v` deletes it.
- Containers reach each other by service name (`db`, `redis`, `ollama`).
- Startup order is gated by `depends_on: service_healthy`; `/health` is liveness, `/ready` is readiness.
- Config comes from `.env` and `${VAR:?}`; nothing is baked into the image.
- Hardening: non-root user, read-only filesystem, dropped capabilities.
- Dev and prod differ by override files, `watch`, pinned registry images and log rotation.
- Deploy: build once in CI, scan, push, pull on the server, roll back by tag.

Next: [Chapter 10 — Security](10-security.md) and [Chapter 11 — Production and Enterprise](11-production-and-enterprise.md).
