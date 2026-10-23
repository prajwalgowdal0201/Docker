# Chapter 4: The Dockerfile Deep Dive

This chapter covers the Dockerfile: every instruction and flag, layers and the build cache, build context, multi-stage builds, base image choice, reproducibility, non-root users, PID 1 and signals, build secrets, BuildKit and buildx, and a complete production Dockerfile for a FastAPI app.

**Prerequisites:** [Chapter 1](01-what-is-docker.md) (images and containers) and [Chapter 3](03-core-cli-commands.md) (basic CLI: `docker run`, `docker images`, `docker ps`).

---

## 1. What a Dockerfile is

A **Dockerfile** is a text file named `Dockerfile` (no extension) that lists the instructions Docker follows to build an **image**. An image is a read-only package containing a filesystem plus metadata (default command, user, environment variables).

Docker reads the file top to bottom. Each instruction that changes the filesystem produces a **layer** ([section 3](#3-layers-and-the-build-cache)).

### The build pipeline

```
 Dockerfile  +  build context (your files)
      |
      v
 +-------------+      +----------------------+
 | docker build| ---> | BuildKit (the build  |
 |  (client)   |      |  engine)             |
 +-------------+      +----------+-----------+
                                 |
          runs FROM, RUN, COPY, ... one step at a time
                                 |
                                 v
                  +-----------------------------+
                  |  IMAGE = stack of layers    |
                  |  + config (CMD, ENV, USER)  |
                  +-----------------------------+
                                 |
                  docker run  ---+--->  container
```

Terms:

- **BuildKit**: the build engine inside Docker (default since Docker Engine 23.0). It runs independent steps in parallel and supports secrets and cache mounts.
- **Build context**: the files sent to the builder (usually the current directory). `COPY` can only read files inside the context.
- **Layer**: one filesystem delta (added, changed or deleted files) produced by one instruction.
- **Registry**: a server that stores images (Docker Hub, GitHub Container Registry, AWS ECR).

---

## 2. Your first Dockerfile, line by line

Create an empty folder with these two files.

**`app.py`**

```python
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Hello from inside a container!\n")

HTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
```

A server in a container must listen on `0.0.0.0` (all interfaces), not `127.0.0.1`. Otherwise nothing outside the container can reach it (see [Chapter 5](05-storage-and-networking.md#b4-localhost-inside-containers)).

**`Dockerfile`**

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

| Line | What it does |
|---|---|
| `FROM python:3.13-slim` | Starts from the official Python 3.13 image, "slim" variant (trimmed Debian with Python). Every Dockerfile starts with `FROM`. |
| `WORKDIR /app` | Sets the working directory for following instructions and creates `/app` if missing. Persists across instructions. |
| `COPY app.py .` | Copies `app.py` from the build context into the image. `.` is the current `WORKDIR`, so the file lands at `/app/app.py`. |
| `EXPOSE 8000` | Documents that the app listens on port 8000. It does not open or publish anything (see [Chapter 5](05-storage-and-networking.md#exposing-vs-publishing)). |
| `CMD ["python", "app.py"]` | Default command when a container starts. |

**Build:**

```bash
docker build -t hello-py:1.0 .
```

| Part | Meaning |
|---|---|
| `docker build` | Build an image from a Dockerfile. |
| `-t hello-py:1.0` | **Tag** the image as name `hello-py`, version `1.0` (`name:tag`). Without `-t` the image only has an ID. |
| `.` | The build context. Docker looks for `./Dockerfile` here by default. |

**Run:**

```bash
docker run --rm -p 8000:8000 hello-py:1.0
```

| Flag | Meaning |
|---|---|
| `--rm` | Delete the container when it stops. |
| `-p 8000:8000` | **Publish** container port 8000 on host port 8000 (`HOST:CONTAINER`). |

`curl http://localhost:8000` prints `Hello from inside a container!`. Stop with Ctrl+C.

---

## 3. Layers and the build cache

### What is a layer?

Each instruction that changes the filesystem (`RUN`, `COPY`, `ADD`) creates a **layer**: a compressed archive of the files added, changed or deleted by that step. An image is an ordered stack of layers. `ENV`, `CMD`, `EXPOSE` and `LABEL` change only metadata and add no files.

```
   IMAGE hello-py:1.0
 +-----------------------------------------+
 | Layer 4: COPY app.py .      (+361 B)    |  <- your code
 +-----------------------------------------+
 | Layer 3: WORKDIR /app       (+0 B)      |
 +-----------------------------------------+
 | Layer 2: python install     (~40 MB)    |  <- from python:3.13-slim
 +-----------------------------------------+
 | Layer 1: Debian base files  (~75 MB)    |  <- from python:3.13-slim
 +-----------------------------------------+
        (all layers READ-ONLY)

   A running container adds one thin WRITABLE layer on top:
 +-----------------------------------------+
 | Container layer (runtime changes)       |
 +-----------------------------------------+
```

Layers are combined with a **union filesystem** (`overlayfs` on modern Docker), which presents stacked directories as one merged directory. A file in a higher layer hides the same file in a lower layer.

Layers are **content-addressed**: each is identified by a SHA-256 hash of its content. This enables:

1. **Sharing**: images built on the same base store the base layers once.
2. **Caching**: if a layer's inputs are unchanged, Docker reuses the previous result.

### Deleting files in a later layer does not shrink the image

Layers only add. If layer 2 downloads a 500 MB file and layer 3 runs `rm` on it, layer 3 records a "whiteout" marker, but layer 2 still contains the 500 MB. Download, use and delete within a single `RUN`:

```dockerfile
# Bad: the tarball stays in layer 2
RUN curl -O https://example.com/big.tar.gz
RUN tar xzf big.tar.gz && rm big.tar.gz

# Good: one layer
RUN curl -O https://example.com/big.tar.gz \
 && tar xzf big.tar.gz \
 && rm big.tar.gz
```

Multi-stage builds ([section 11](#11-multi-stage-builds)) are a cleaner alternative.

### How the build cache decides to reuse a layer

For each instruction, BuildKit computes a **cache key** and checks whether it already executed the same instruction on the same parent layer.

| Instruction | Cache key based on |
|---|---|
| `RUN` | The command text and the parent layer. Docker does not re-run it because the outside world changed. `RUN apt-get update` stays cached until the line or an earlier layer changes. |
| `COPY` / `ADD` | A checksum of the copied files' content and metadata (such as permissions). Modification time is ignored. |
| `ADD <url>` | The URL (and validators such as ETag/Last-Modified where available). Use `--checksum` for reproducibility. |
| `ENV`, `ARG`, `LABEL`, `USER`, ... | The instruction text and value. Changing an `ARG` value invalidates the cache from the first instruction that uses it. |

**Cascade rule:** once one layer is invalidated, every layer after it is rebuilt, because each layer is built on the previous one.

```
 Step 1  FROM python:3.13-slim        CACHED
 Step 2  WORKDIR /app                 CACHED
 Step 3  COPY requirements.txt .      CACHED   (file unchanged)
 Step 4  RUN pip install ...          CACHED
 Step 5  COPY . .                     MISS     (main.py edited)
 Step 6  RUN python -m compileall .   REBUILT  (cascade)
 Step 7  CMD [...]                    REBUILT  (cascade)
```

**Ordering rule:** put what changes least at the top and what changes most at the bottom. Base image and system packages change rarely, dependency lists occasionally, source code constantly. A timed example is in [section 10](#10-ordering-instructions-for-cache-efficiency-before-and-after).

### Stale `apt-get update`

`RUN` is cached by its text, so a separate `apt-get update` line can go stale:

```dockerfile
RUN apt-get update
# Months later the cached update is stale -> 404 on packages
RUN apt-get install -y curl
```

Combine both in one `RUN`:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

| Flag | Meaning |
|---|---|
| `-y` | Answer "yes" to prompts (builds are non-interactive). |
| `--no-install-recommends` | Skip optional recommended packages that enlarge the image. |
| `rm -rf /var/lib/apt/lists/*` | Delete the package index downloaded by `apt-get update`; it is not needed at runtime. |

### Forcing a rebuild

```bash
docker build --no-cache -t hello-py:1.0 .     # ignore the cache
docker build --pull -t hello-py:1.0 .         # check for a newer base image
docker builder prune                           # delete the build cache
```

| Command / flag | Meaning |
|---|---|
| `--no-cache` | Use no cached layers; rebuild everything. |
| `--pull` | Re-check the registry for a newer `FROM` image. Without it, Docker reuses a local base image even if the tag has moved. |
| `docker builder prune` | Remove cached build data (frees disk; the next build is slow). |

---

## 4. Build context and `.dockerignore`

In `docker build .`, the final `.` is the **build context**. The client sends the files in that directory to the builder. Only files in the context can be referenced by `COPY` and `ADD`. BuildKit transfers only the files the Dockerfile needs, but ignoring files up front is still important for speed and safety.

Typical problems:

- Running `docker build` from the home directory, which sends gigabytes.
- `node_modules/` or `.venv/` (large and platform-specific) copied into the image by `COPY . .`.
- `.git/` (full history, sometimes with secrets) ending up in the image.
- `.env` files with credentials ending up in the image.

A **`.dockerignore`** file in the context root excludes files from the context. It uses `.gitignore`-like patterns.

**Starter `.dockerignore`:**

```gitignore
# Version control
.git
.gitignore

# Secrets and local config - never let these into an image
.env
.env.*
*.pem
*.key
secrets/

# Python
__pycache__/
*.pyc
.venv/
venv/
.pytest_cache/
.mypy_cache/
.ruff_cache/

# Node
node_modules/
npm-debug.log

# Build outputs, editor files, OS files
dist/
build/
.idea/
.vscode/
.DS_Store
Thumbs.db

# Docker files themselves (not needed inside the image)
Dockerfile*
compose*.yaml
docker-compose*.yml
.dockerignore

# Docs and tests (if not needed at runtime)
docs/
tests/
*.md
```

Pattern rules:

| Pattern | Meaning |
|---|---|
| `node_modules` | Ignore `node_modules` in the context root only. |
| `**/node_modules` | Ignore `node_modules` at any depth (monorepos). |
| `*.log` | Any `.log` file in the root. |
| `**/*.log` | Any `.log` file anywhere. |
| `!README.md` | Exception: re-include a file excluded earlier. The last matching line wins, so exceptions must come after the rule they override. |
| `# comment` | Comment. |

**Allowlist style.** Ignore everything and re-include only what is needed, so new files (such as a stray `.env`) are excluded by default:

```gitignore
*
!pyproject.toml
!requirements.txt
!app/
```

**Per-Dockerfile ignore files.** For `Dockerfile.dev`, BuildKit uses `Dockerfile.dev.dockerignore` if it exists, otherwise `.dockerignore`.

---

## 5. The instruction reference

### 5.0 The `syntax` directive (first line, optional but recommended)

```dockerfile
# syntax=docker/dockerfile:1
```

A **parser directive**: a special comment that must be the first line (before any blank line or other comment). It selects the **Dockerfile frontend** (the component that parses the Dockerfile). `docker/dockerfile:1` downloads the latest stable 1.x frontend before building, so newer features (heredocs, `--link`, `--parents`, build checks) work even with an older local Docker. The official Dockerfile reference recommends this line.

Other directives: `# escape=\`` (changes the escape character, mainly for Windows containers) and `# check=skip=...;error=true` (configures build checks, Dockerfile 1.8+).

Offline or locked-down builds can pin a version (for example `docker/dockerfile:1.10`) or mirror the frontend image internally. Omitting the line still works for most features with a recent Docker.

### 5.1 `FROM`: choose the starting point

```dockerfile
FROM [--platform=<platform>] <image>[:<tag>|@<digest>] [AS <name>]
```

| Part | Meaning |
|---|---|
| `<image>:<tag>` | For example `python:3.13-slim`. Without a tag, Docker uses `latest`, which is rarely appropriate for production ([section 13](#13-reproducibility-tags-digests-pinning)). |
| `@sha256:...` | Pin an exact, immutable image **digest**. |
| `AS <name>` | Name this **build stage** so later stages can copy from it ([section 11](#11-multi-stage-builds)). |
| `--platform=...` | Force a platform for the stage, for example `$BUILDPLATFORM` for cross-compilation ([section 19](#19-multi-platform-builds-amd64-and-arm64)). |
| `FROM scratch` | The empty image (no OS), used for static binaries. |

A Dockerfile can have multiple `FROM` lines. Each starts a new stage, and the final stage becomes the shipped image.

### 5.2 `ARG`: build-time variables

```dockerfile
ARG PYTHON_VERSION=3.13
FROM python:${PYTHON_VERSION}-slim
```

`ARG` defines a variable available only during the build, not in the running container. Set it with `docker build --build-arg PYTHON_VERSION=3.12 .`. An `ARG` declared before the first `FROM` can be used in `FROM` lines but is not visible inside stages unless re-declared. Comparison with `ENV`: [section 7](#7-arg-vs-env).

BuildKit provides automatic platform ARGs (`TARGETPLATFORM`, `TARGETOS`, `TARGETARCH`, `BUILDPLATFORM`, ...). Re-declare them inside a stage to use them.

### 5.3 `ENV`: environment variables baked into the image

```dockerfile
ENV APP_ENV=production \
    PORT=8000
```

Sets variables available at build time (for later instructions) and at runtime (in every container). Use the `KEY=value` form; the older `ENV KEY value` form is discouraged and may be removed. Values are visible through `docker inspect` and `docker history`, so do not put secrets in `ENV`.

### 5.4 `WORKDIR`: set the working directory

```dockerfile
WORKDIR /app
```

Sets the directory for the following `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD`, and creates it if missing.

- Use absolute paths.
- Prefer `WORKDIR` to `RUN cd /app && ...` (which affects only that `RUN`) and to `RUN mkdir`.
- Relative paths accumulate: `WORKDIR /a` then `WORKDIR b` gives `/a/b`.
- It can use `ENV` variables.

### 5.5 `COPY` vs `ADD`: put files into the image

```dockerfile
COPY [--chown=<user>:<group>] [--chmod=<perms>] [--from=<stage|image|context>] [--link] [--parents] [--exclude=<pattern>] <src>... <dest>
ADD  [--chown=...] [--chmod=...] [--checksum=<hash>] [--keep-git-dir] [--unpack] [--link] [--exclude=...] <src>... <dest>
```

| | `COPY` | `ADD` |
|---|---|---|
| Copies local files/folders from the build context | Yes | Yes |
| Copies from another stage or image (`--from`) | Yes | No |
| Downloads from an HTTP(S) URL | No | Yes |
| Clones a Git repository | No | Yes |
| Auto-extracts a local `.tar`, `.tar.gz`, etc. | No | Yes |
| Predictability | High | Lower |

Use `COPY` by default. Use `ADD` only for tar auto-extraction or a remote source. For downloads, `RUN curl -fsSL ... | ...` or `ADD --checksum=sha256:... https://...` (verifies integrity) is better than a bare `ADD <url>`.

| Flag | Meaning |
|---|---|
| `--chown=user:group` | Set the owner of copied files. Default is root (UID 0), which is a problem when running as non-root later. Example: `COPY --chown=app:app . .`. Numeric IDs (`--chown=10001:10001`) also work and are safer when the user does not exist yet in that stage. |
| `--chmod=755` | Set permissions during the copy (octal, or symbolic such as `a+x` in newer syntax versions). Avoids a separate `RUN chmod` layer that would duplicate the file size. |
| `--from=<name>` | Copy from an earlier stage (`--from=builder`), any image (`--from=nginx:alpine`), or a named build context. Basis of multi-stage builds. |
| `--link` | (syntax 1.4+) Makes the copied files an independent layer that stays cached when the base or earlier layers change, and can be rebased onto a new base without re-copying. Caveat: directories created by earlier instructions and their ownership/permissions are not inherited. Suited to the final stage of multi-stage builds. |
| `--parents` | (syntax 1.20+) Keep the parent directory structure of the source path. `COPY --parents ./src/a/b.py /app/` gives `/app/src/a/b.py`. Useful with wildcards: `COPY --parents **/package.json ./`. |
| `--exclude=<pattern>` | (syntax 1.19+) Skip files matching a pattern for this `COPY` only, for example `COPY --exclude=*.test.js . /app/`. |

Other behaviour:

- `COPY src/ dest/` copies the contents of `src`, not the folder. To keep the folder: `COPY src/ dest/src/`.
- End directory destinations with `/`.
- Wildcards work in sources: `COPY *.txt /docs/`.
- `COPY . .` copies the whole context minus ignored files. Place it late in the Dockerfile (cache).

### 5.6 `RUN`: execute commands at build time

```dockerfile
RUN <command>                          # shell form: runs via /bin/sh -c "<command>"
RUN ["executable", "arg1", "arg2"]     # exec form: runs directly, no shell
RUN [--mount=...] [--network=...] [--security=...] <command>
```

`RUN` executes the command in a temporary container and commits the filesystem changes as a new layer.

Rules:

1. Chain related commands with `&&` and clean up in the same instruction. Use `\` for continuation, with `&&` at the start of continuation lines.
2. In shell form, `a | b` reports only the exit code of `b`. Use `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` ([5.15](#515-shell-change-the-default-shell)) when piping.
3. In long scripts use `set -eux` (`-e` exit on error, `-u` error on undefined variable, `-x` echo commands).
4. Avoid `apt-get upgrade` / `dist-upgrade`; it makes builds non-reproducible. Use a newer base image instead.
5. `--mount` provides cache, secret, ssh and bind mounts ([section 9](#9-buildkit-power-features-run---mount-and-friends)). `--network=none` runs a step without network access. `--network=host` needs a special entitlement.

### 5.7 `CMD` and `ENTRYPOINT`

See [section 6](#6-cmd-vs-entrypoint-exec-form-vs-shell-form).

### 5.8 `EXPOSE`: document the port

```dockerfile
EXPOSE 8000
EXPOSE 8000/tcp 53/udp
```

Metadata only: documents which port the app listens on. It does not publish the port; use `-p` or `ports:` at run time. `docker run -P` (capital P) publishes all exposed ports to random host ports.

### 5.9 `USER`: switch to a non-root user

```dockerfile
USER 10001:10001
USER app
```

All following `RUN`, `CMD` and `ENTRYPOINT` run as this user, as does the container's main process. See [section 15](#15-running-as-a-non-root-user).

### 5.10 `VOLUME`: declare a mount point

```dockerfile
VOLUME /var/lib/postgresql/data
```

Declares that data at this path lives in a volume. If nothing is mounted there at run time, Docker creates an **anonymous volume**. Caveats:

- Changes to that path in later Dockerfile instructions are discarded.
- Anonymous volumes accumulate unnoticed.
- The declaration cannot be undone in a child image.

In application images, usually do not use `VOLUME`; mount named volumes explicitly at run time or in Compose ([Chapter 5](05-storage-and-networking.md)). Database images use it to guarantee data survives.

### 5.11 `HEALTHCHECK`: tell Docker how to test the app

```dockerfile
HEALTHCHECK [--interval=30s] [--timeout=30s] [--start-period=0s] [--start-interval=5s] [--retries=3] \
  CMD <command>
HEALTHCHECK NONE
```

Docker periodically runs the command inside the container. Exit code `0` is healthy, `1` is unhealthy (`2` is reserved; do not use it).

| Option | Default | Meaning |
|---|---|---|
| `--interval` | `30s` | Time between checks. |
| `--timeout` | `30s` | A check taking longer counts as failed. |
| `--start-period` | `0s` | Grace time for slow-starting apps. Failures in this window do not count toward `retries`; a success ends the window early. |
| `--start-interval` | `5s` | Check frequency during the start period (Docker Engine 25.0+). |
| `--retries` | `3` | Consecutive failures before the status becomes `unhealthy`. |

States: `starting`, then `healthy` or `unhealthy`. View with `docker ps` (STATUS column) or `docker inspect --format '{{json .State.Health}}' <container>`. Only the last `HEALTHCHECK` in a Dockerfile counts. `HEALTHCHECK NONE` disables a check inherited from the base image.

Plain Docker does not restart an unhealthy container. The status is information for orchestrators and tools (Compose `depends_on: condition: service_healthy`, Swarm, load balancers). Kubernetes ignores the Dockerfile `HEALTHCHECK` and uses its own probes ([Chapter 11](11-production-and-enterprise.md)).

Slim images do not include `curl`. Python example using only the standard library:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
```

A non-2xx response makes `urlopen` raise an exception, Python exits with code 1, and the check fails.

### 5.12 `LABEL`: attach metadata

```dockerfile
LABEL org.opencontainers.image.title="my-api" \
      org.opencontainers.image.description="Example FastAPI service" \
      org.opencontainers.image.source="https://github.com/you/my-api" \
      org.opencontainers.image.licenses="MIT"
```

Key-value metadata readable with `docker inspect`. Use the standard OCI annotation keys (`org.opencontainers.image.*`) so registries and tools understand them (GitHub Container Registry links the image to a repo through `...image.source`). `MAINTAINER` is deprecated; use `org.opencontainers.image.authors`.

### 5.13 `STOPSIGNAL`: how to ask the app to stop

```dockerfile
STOPSIGNAL SIGINT
```

`docker stop` sends a signal to PID 1, waits (10 seconds by default), then sends `SIGKILL`. The default signal is `SIGTERM`. Some programs expect another (nginx graceful shutdown uses `SIGQUIT`). Override at run time with `--stop-signal`, and the grace period with `docker stop -t <seconds>`.

### 5.14 `ONBUILD`: triggers for child images

```dockerfile
ONBUILD COPY . /app
ONBUILD RUN pip install -r /app/requirements.txt
```

Registers an instruction that runs later, when another image uses this one as its base. It makes a `FROM` line trigger hidden actions, so it is hard to debug; avoid it in your own images. `ONBUILD ONBUILD` and `ONBUILD FROM` are not allowed.

### 5.15 `SHELL`: change the default shell

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

Changes the shell for shell-form `RUN`, `CMD` and `ENTRYPOINT`. The Linux default is `["/bin/sh", "-c"]`. The usual reason is `pipefail`, so `RUN curl ... | tar ...` fails when `curl` fails. `bash` must exist in the image (slim Debian has it; Alpine does not by default).

### 5.17 Instruction cheat table

| Instruction | Layer-producing? | Runtime effect? | Purpose |
|---|---|---|---|
| `FROM` | starts a stage | - | Choose base image |
| `ARG` | no | no (build only) | Build-time variable |
| `ENV` | metadata | yes | Environment variable in image |
| `WORKDIR` | tiny | yes (cwd) | Set/create working directory |
| `COPY` / `ADD` | yes | - | Put files in image |
| `RUN` | yes | - | Execute at build time |
| `CMD` | metadata | yes | Default command/args |
| `ENTRYPOINT` | metadata | yes | Fixed executable |
| `EXPOSE` | metadata | docs only | Document port |
| `USER` | metadata | yes | Run as this user |
| `VOLUME` | metadata | yes | Declare mount point |
| `HEALTHCHECK` | metadata | yes | Health probe |
| `LABEL` | metadata | inspect only | Metadata |
| `STOPSIGNAL` | metadata | yes | Signal for `docker stop` |
| `SHELL` | metadata | build only | Default shell |
| `ONBUILD` | metadata | later builds | Trigger in child build |

---

## 6. CMD vs ENTRYPOINT, exec form vs shell form

### Two syntaxes: exec form and shell form

`RUN`, `CMD` and `ENTRYPOINT` can each be written two ways:

```dockerfile
CMD ["python", "app.py"]
CMD python app.py
```

These are alternatives; a Dockerfile has one `CMD`.

- **Exec form** (first line): a JSON array. Docker runs the program directly.
- **Shell form** (second line): a plain string. Docker runs `/bin/sh -c "python app.py"`.

| | Exec form `["a","b"]` | Shell form `a b` |
|---|---|---|
| How it runs | `a` starts directly as PID 1 | `/bin/sh -c "a b"` starts; `sh` is PID 1 and `a` is its child |
| Variable expansion (`$HOME`) | No (no shell) | Yes |
| Signals (`docker stop`) | Delivered to the program | Delivered to `sh`, which usually does not forward them; the app does not shut down gracefully and is killed with `SIGKILL` after 10 s |
| Needs a shell in the image | No (works in distroless/scratch) | Yes |
| Syntax | Valid JSON, double quotes only | Free text |

Use exec form for `CMD` and `ENTRYPOINT` ([section 16](#16-pid-1-signals-tini-and---init)).

For variable expansion in exec form, call the shell and use `exec`:

```dockerfile
CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

`exec` replaces the shell process with uvicorn, so uvicorn is PID 1 and receives signals.

`CMD ['python', 'app.py']` (single quotes) is not valid JSON, so Docker silently treats it as shell form. Use double quotes.

### What are CMD and ENTRYPOINT?

- **`ENTRYPOINT`**: the executable that always runs.
- **`CMD`**: the default arguments (or the default command when there is no `ENTRYPOINT`). Users can override it.

Analogy: `ENTRYPOINT` is the machine; `CMD` is the default setting on its dial.

```bash
docker run myimage                  # ENTRYPOINT + default CMD
docker run myimage --verbose        # replaces CMD with "--verbose"; ENTRYPOINT unchanged
docker run --entrypoint sh -it myimage   # replaces ENTRYPOINT (for example a debug shell)
```

### Example: a CLI tool image

```dockerfile
FROM python:3.13-slim
COPY greet.py /greet.py
ENTRYPOINT ["python", "/greet.py"]
CMD ["world"]
```

```bash
docker run greet            # runs: python /greet.py world
docker run greet Alice      # runs: python /greet.py Alice
```

### The interaction table

What gets executed for each combination:

| | **No ENTRYPOINT** | **ENTRYPOINT exec_e p1_e** (shell form) | **ENTRYPOINT ["exec_e", "p1_e"]** (exec form) |
|---|---|---|---|
| **No CMD** | *error: nothing to run* | `/bin/sh -c exec_e p1_e` | `exec_e p1_e` |
| **CMD ["exec_c", "p1_c"]** (exec form) | `exec_c p1_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e exec_c p1_c` |
| **CMD exec_c p1_c** (shell form) | `/bin/sh -c exec_c p1_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e /bin/sh -c exec_c p1_c` |
| **CMD ["p1_c", "p2_c"]** (default params) | `p1_c p2_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e p1_c p2_c` |

Rules:

1. A shell-form `ENTRYPOINT` ignores `CMD` and `docker run` arguments and cannot receive signals properly. Avoid it.
2. With an exec-form `ENTRYPOINT`, an exec-form `CMD` is appended as arguments. This is the pattern for tool images.
3. With no `ENTRYPOINT`, `CMD` is the whole command.
4. When both exist, write both in exec form. A shell-form `CMD` with an exec-form `ENTRYPOINT` produces `exec_e p1_e /bin/sh -c ...`.

### Which to choose

| Situation | Use |
|---|---|
| Web service/app | `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]` (easy to override for debugging) |
| CLI tool packaged as an image | `ENTRYPOINT ["tool"]` + `CMD ["--help"]` |
| Setup logic before the app | `ENTRYPOINT ["/entrypoint.sh"]` ending with `exec "$@"` |

### The entrypoint-script pattern

```bash
#!/bin/sh
# entrypoint.sh
set -e
# ... setup: wait for a file, run migrations, render a config ...
echo "Starting with args: $@"
exec "$@"        # replace this shell with the real command so it becomes PID 1
```

```dockerfile
COPY --chmod=755 entrypoint.sh /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

The entrypoint runs first. `"$@"` expands to all arguments (the `CMD`), and `exec` hands control to it. Without `exec`, the script's shell stays PID 1 and does not forward signals.

**Windows line endings.** If `entrypoint.sh` has CRLF line endings, the kernel looks for the interpreter `/bin/sh\r`, which does not exist, and the error is `exec ... no such file or directory`. Add `* text=auto eol=lf` to `.gitattributes` for shell scripts.

---

## 7. ARG vs ENV

| | `ARG` | `ENV` |
|---|---|---|
| Available during build | Yes (from the declaring line) | Yes (from the declaring line) |
| Available in the running container | No | Yes |
| Set from the command line | `docker build --build-arg NAME=value` | `docker run -e NAME=value` (overrides at run time) |
| Visible in image metadata | Name and value can appear in `docker history` if used in a command | Yes, in `docker inspect` |
| Typical use | Versions, build toggles, platform | Runtime config defaults (`PORT`, `APP_ENV`) |
| Safe for secrets | No | No |

If an `ARG` and an `ENV` share a name, the `ENV` wins.

Turn a build argument into a runtime variable:

```dockerfile
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
```

Scope:

```dockerfile
# Global ARG: usable only in FROM lines
ARG BASE=3.13
FROM python:${BASE}-slim AS build
# Re-declare it to use it inside this stage
ARG BASE
RUN echo "building on python ${BASE}"
```

For secrets see [section 17](#17-build-secrets-never-bake-secrets-into-layers).

---

## 8. Heredocs

Dockerfile syntax 1.4+ supports **here-documents** (text between a start marker and an end marker is passed as input) for multi-line scripts and inline files. The `# syntax=docker/dockerfile:1` line guarantees support.

### Multi-line `RUN`

Instead of a `&&`/`\` chain:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

write:

```dockerfile
RUN <<EOF
set -eux
apt-get update
apt-get install -y --no-install-recommends curl ca-certificates
rm -rf /var/lib/apt/lists/*
EOF
```

`<<EOF` starts the heredoc. Everything up to a line that is exactly `EOF` runs as a script in the default shell, as one layer. `set -e` stops at the first failure.

A shebang selects another interpreter:

```dockerfile
RUN <<EOF
#!/usr/bin/env python3
import sys
print("Python", sys.version)
EOF
```

### Creating files inline with `COPY`

```dockerfile
COPY <<EOF /etc/app/config.ini
[server]
port = 8000
EOF
```

This creates `/etc/app/config.ini` with that content. Use `<<"EOF"` (quoted) to keep `$VARIABLES` literal; an unquoted heredoc expands build-time variables.

---

## 9. BuildKit power features: `RUN --mount` and friends

BuildKit lets a `RUN` step attach temporary mounts that never end up in the final image. There are five types.

### 9.1 `--mount=type=cache`: persistent caches for package managers

When `requirements.txt` changes, `pip install` re-downloads all packages because the layer is rebuilt from scratch. A **cache mount** is a directory kept by BuildKit between builds, outside any image layer, so downloaded packages are reused.

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

| Option | Meaning |
|---|---|
| `type=cache` | Cache mount type. |
| `target=/root/.cache/pip` | Path inside the build container (pip's download cache). |
| `id=` | Name the cache so different Dockerfiles can share it. |
| `sharing=shared\|private\|locked` | Behaviour when two builds use the cache at once; `locked` makes the second wait. |
| `uid=` / `gid=` / `mode=` | Ownership and permissions of the mount. |
| `from=` / `source=` | Initial content from another stage. |

The cache is not part of the layer, so the image stays small. Do not add `--no-cache-dir`; it defeats the cache.

Common cache targets:

| Tool | Cache `target` |
|---|---|
| pip | `/root/.cache/pip` |
| uv | `/root/.cache/uv` |
| npm | `/root/.npm` |
| yarn | `/usr/local/share/.cache/yarn` |
| pnpm | `/root/.local/share/pnpm/store` (or your configured store) |
| Go modules | `/go/pkg/mod` and `/root/.cache/go-build` |
| Maven | `/root/.m2` |
| Gradle | `/root/.gradle` |
| apt | `/var/cache/apt` and `/var/lib/apt` |

apt needs an extra step because Debian images ship a config that deletes downloaded `.deb` files:

```dockerfile
RUN rm -f /etc/apt/apt.conf.d/docker-clean \
 && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq5
```

`sharing=locked` is used because apt cannot run twice concurrently on the same directories. Cache mounts speed up local builds. On CI with a cold runner, export the cache ([section 20](#20-exporting-the-build-cache-for-ci)).

### 9.2 `--mount=type=secret`: use a secret without storing it

Details in [section 17](#17-build-secrets-never-bake-secrets-into-layers).

```dockerfile
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```

The secret appears as the file `/run/secrets/<id>` only during that `RUN` and leaves no trace in any layer.

### 9.3 `--mount=type=ssh`: use SSH keys without copying them

```dockerfile
RUN --mount=type=ssh \
    mkdir -p ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts \
 && git clone git@github.com:yourorg/private-lib.git /src/private-lib
```

```bash
docker build --ssh default .
```

- `--ssh default` forwards the local **SSH agent** (the background program holding unlocked keys) into the build. The private key file is never copied into the image. `ssh-add -l` should list a key.
- `ssh-keyscan` adds the host's public key to `known_hosts` so SSH does not prompt for confirmation (a build cannot answer prompts). For stricter security, pin the host key fingerprint instead of scanning blindly.

### 9.4 `--mount=type=bind`: read files from the context or another stage without copying

```dockerfile
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

A bind mount makes a file or folder from the build context (or another stage via `from=`) visible during this `RUN`, read-only by default. Nothing is copied into a layer, and the cache still invalidates when the file changes. Add `rw` to allow writes (discarded after the step).

### 9.5 `--mount=type=tmpfs`

```dockerfile
RUN --mount=type=tmpfs,target=/tmp,size=512m ./build-script.sh
```

A RAM-backed scratch directory that disappears after the step. Use it for large temporary build files that should not enter a layer.

### 9.6 Other `RUN` flags

| Flag | Meaning |
|---|---|
| `--network=none\|default\|host` | Control network access for the step (`host` needs the `network.host` entitlement). |
| `--security=sandbox\|insecure` | `insecure` (needs the `security.insecure` entitlement) is for special cases such as Docker-in-Docker builds. |
| `--device=...` | Request CDI devices (for example GPUs) during build; needs BuildKit 0.20+ and the `device` entitlement. |

---

## 10. Ordering instructions for cache efficiency (before and after)

Project layout:

```
myapi/
  Dockerfile
  requirements.txt      (fastapi, uvicorn, ... ~30 packages)
  app/
    main.py
    ...
```

### Before: dependencies installed after copying all code

```dockerfile
FROM python:3.13-slim
WORKDIR /app
# Copies everything, including source code
COPY . .
# This step depends on the line above
RUN pip install -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

Comments must be on their own line. A `#` in the middle of an instruction line becomes part of the arguments.

After changing one line in `app/main.py`:

```
 Step 3  COPY . .                      MISS  (main.py content changed)
 Step 4  RUN pip install ...           REBUILT: reinstalls all packages (~60 s)
```

### After: dependencies first, code last

```dockerfile
FROM python:3.13-slim
WORKDIR /app

# 1) Copy only the dependency list. It changes rarely.
COPY requirements.txt .

# 2) Install dependencies. Cached while requirements.txt is unchanged.
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# 3) Copy the source code last. It changes constantly.
COPY app/ ./app/

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

After editing `app/main.py`, only the final `COPY` and later steps rebuild; the `pip install` step shows `CACHED`.

Illustrative timings (vary with network and package set):

| Scenario | Before | After |
|---|---|---|
| Cold build, nothing cached | ~60 s | ~60 s |
| Edit one source file, rebuild | ~60 s | ~1-2 s |
| Add one new dependency, rebuild | ~60 s | ~10 s (cache mount reuses unchanged wheels) |

The pattern applies to every language: copy the dependency manifests first (`package.json` + lockfile, `go.mod` + `go.sum`, `pom.xml`, `pyproject.toml` + `uv.lock`), install, then copy source.

---

## 11. Multi-stage builds

Building software needs compilers, headers, package managers and test tools. Running it usually needs only the result. A single-stage build ships all the build tooling, which makes the image larger and adds attack surface.

A **multi-stage build** uses several `FROM` lines in one Dockerfile. Each is a separate stage with its own filesystem. Build in a large stage and copy only the artifacts into a small final stage. Everything else is discarded.

```
 +------------------------------+        +-----------------------------+
 | Stage 1: "builder"           |        | Stage 2: "runtime" (FINAL)  |
 | python:3.13 (big)            |        | python:3.13-slim (small)    |
 | + compilers, headers         | COPY   |                             |
 | + pip install -> /opt/venv   | -----> | /opt/venv  (just the venv)  |
 | + tests, caches ...          | --from | + your app code             |
 +------------------------------+        +-----------------------------+
      (discarded after build)                  (the image you ship)
```

| Command / flag | Meaning |
|---|---|
| `FROM ... AS name` | Name a stage. |
| `COPY --from=name /path /dest` | Copy files from a stage. |
| `docker build --target name .` | Stop at a specific stage (for example `test` or `dev`). Default is the last stage. |

BuildKit builds only the stages the target depends on, and builds independent stages in parallel.

### 11.1 Python (virtualenv copy)

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.13-slim AS builder
WORKDIR /build
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

FROM python:3.13-slim AS runtime
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1
WORKDIR /app
COPY app/ ./app/
USER 10001
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- `python -m venv /opt/venv`: creates a **virtual environment** (a self-contained folder of installed Python packages) at a fixed path, so one `COPY --from=builder /opt/venv /opt/venv` moves all dependencies.
- `ENV PATH="/opt/venv/bin:$PATH"`: puts the venv's `bin` first on the search path (equivalent to activating it).
- `PYTHONUNBUFFERED=1`: prints logs immediately, so `docker logs` shows output in real time.
- Both stages must use the same Python version, OS and path for the venv to work.
- Use this when the builder needs compilers to build wheels (for example `psycopg2`, `cryptography` from source) that the runtime does not.
- Alternative: [uv](https://docs.astral.sh/uv/), a fast Python package manager. `COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv`, then `uv sync --frozen` with a cache mount at `/root/.cache/uv`. Pin the uv image tag or digest.

### 11.2 Node.js (build a frontend, serve static files with nginx)

```dockerfile
# syntax=docker/dockerfile:1

FROM node:24-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci

FROM node:24-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# Produces /app/dist
RUN npm run build

FROM nginx:1.27-alpine AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

- `npm ci` ("clean install") installs exactly what the lockfile specifies and fails if `package.json` and the lockfile disagree. Use it instead of `npm install` in builds.
- Stages: `deps` (install), `build` (compile to static files), `runtime` (nginx). The final image contains no Node.js.
- Unverified: tag availability. Node LTS versions rotate; check [nodejs.org](https://nodejs.org) and pin.

For a Node server (Express, NestJS, Next.js standalone), the final stage is `node:24-slim` with `npm ci --omit=dev` (production dependencies only) plus the compiled output, running as the built-in `node` user (`USER node`) with `CMD ["node", "dist/server.js"]`.

### 11.3 Go (compile, then ship a ~10 MB image)

Go compiles to a single self-contained binary, which suits tiny images.

```dockerfile
# syntax=docker/dockerfile:1

FROM golang:1.25 AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

| Item | Meaning |
|---|---|
| `CGO_ENABLED=0` | Disable cgo (Go's bridge to C libraries) so the binary is statically linked with no shared-library dependency such as glibc. Static binaries run on `scratch` or distroless/static. |
| `-trimpath` | Remove local file-system paths from the binary (smaller, more reproducible, no directory layout leak). |
| `-ldflags="-s -w"` | Strip the symbol table (`-s`) and DWARF debug info (`-w`); shrinks the binary by roughly a quarter to a third. |
| `gcr.io/distroless/static-debian12:nonroot` | Google's distroless image: CA certificates, timezone data, a minimal `/etc/passwd`, and a pre-made non-root user (UID 65532). About 2 MB. No shell or package manager. Check the distroless project page for current Debian-version tags. |

- Typical final size is 10-20 MB.
- Alternative: `FROM scratch`. Copy CA certificates for HTTPS (`COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`) and set `USER 65532:65532`.

### 11.4 Java (JDK to build, JRE to run)

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:21-jdk AS build
WORKDIR /src
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -q -B dependency:go-offline
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw -q -B package -DskipTests

FROM eclipse-temurin:21-jre AS runtime
RUN useradd --system --uid 10001 --no-create-home appuser
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
USER 10001
EXPOSE 8080
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

- **JDK** (Java Development Kit) includes the compiler; **JRE** (Java Runtime Environment) only runs programs and is much smaller.
- `dependency:go-offline`: downloads Maven dependencies in a cacheable step before the source is copied.
- `-DskipTests`: skips tests in the image build; run them in CI or a `test` stage.
- `-XX:MaxRAMPercentage=75.0`: sizes the JVM heap as a percentage of the container's memory limit and leaves room for non-heap memory.
- `jlink` can build a trimmed custom runtime. If the build produces several jars (for example `-plain.jar`), make the `COPY` pattern specific.

### 11.5 Extra multi-stage techniques

**A test stage** that runs only on request:

```dockerfile
FROM builder AS test
COPY tests/ ./tests/
RUN pytest -q
```

`docker build --target test .` runs the tests. A normal `docker build .` (last stage) skips them.

**Copying from any image** without a stage:

```dockerfile
COPY --from=busybox:1.37 /bin/wget /usr/local/bin/wget
```

**Named build contexts**: `docker build --build-context tools=./tools .` then `COPY --from=tools ...` pulls in a second directory, or `docker-image://...` / `https://...git` sources, in the same build.

---

## 12. Choosing a base image

The `FROM` line has the largest effect on size, security, compatibility and build time.

### The options

| Base type | Example | Approx. size* | Pros | Cons |
|---|---|---|---|---|
| Full Debian-based language image | `python:3.13` | ~1 GB | Compilers and many dev libraries; almost everything builds | Large; many packages mean many CVEs |
| Slim (Debian) | `python:3.13-slim` | ~120-150 MB | Good balance; uses **glibc** (the standard C library), so binary wheels work | Fewer tools; no compilers (add `build-essential` in a builder stage) |
| Alpine | `python:3.13-alpine` | ~50-60 MB | Small; package manager `apk` | Uses **musl** libc, not glibc: compatibility issues (below) |
| Distroless | `gcr.io/distroless/python3-debian12` or `.../static-debian12` | ~2 MB (static) to ~50 MB (python) | No shell or package manager: small attack surface | Hard to debug; no `sh`, so no shell-form commands or `docker exec ... sh` |
| scratch | `FROM scratch` | 0 MB | Empty; suits static binaries | No certs, users, timezone or shell |
| OS image | `debian:stable-slim`, `ubuntu:24.04` | ~75-80 MB | Familiar; install what you need | You maintain the runtime install |

*Sizes are rough, uncompressed and change with each release. Check with `docker images`.

### Alpine and Python

Alpine uses **musl** instead of **glibc**:

1. **Python wheels.** Most pre-compiled wheels ("manylinux") target glibc. Some packages publish "musllinux" wheels, but many do not (notably large scientific/ML packages). Without one, `pip` compiles from source, which needs a compiler, slows the build from seconds to minutes and enlarges the image. PyTorch, TensorFlow and many CUDA packages do not target Alpine ([Chapter 8](08-docker-for-ai-and-ml.md)).
2. **Behavioural differences.** Historical DNS quirks, different `locale`, `iconv` and thread stack sizes, occasional allocator performance differences.
3. **Tools.** `sh` is BusyBox `ash`, not bash; utilities have different flags. Packages come from `apk`, not `apt`.

Alpine suits Go (static binaries), simple Node apps, nginx, Redis and other images with an official Alpine variant and no native extensions. For Python and native dependencies, use `-slim`.

### Distroless

Distroless images (from Google) contain only what a language runtime needs: fewer CVEs and no shell for an attacker. Debugging options: the `:debug` tag variants (add a BusyBox shell), `docker debug` or ephemeral debug containers ([Chapter 12](12-troubleshooting-and-cheatsheet.md)). With no shell, every `CMD`, `ENTRYPOINT` and `HEALTHCHECK` must be in exec form, and `RUN` steps cannot run in the final stage.

### Decision guide

```
 Static-binary language (Go, Rust)?
        -> distroless/static (or scratch)
 Python / Node / Ruby app with normal dependencies?
        -> -slim  (python:3.13-slim, node:24-slim)
 ML/GPU libraries?
        -> NVIDIA CUDA or framework-provided image (Debian/Ubuntu-based), see Chapter 8
 Java?
        -> eclipse-temurin:<ver>-jre (or a distroless java image)
 Official Alpine variant, no native dependencies?
        -> alpine is acceptable
 Unsure?
        -> -slim
```

---

## 13. Reproducibility: tags, digests, pinning

A **reproducible** build produces a functionally identical image whenever it runs. Common breakages:

1. **Moving tags.** `python:3.13-slim` is a mutable pointer. After the next patch or Debian point release it points to a different image. `latest` can jump major versions.
2. **Unpinned packages.** `apt-get install curl` and `pip install fastapi` install whatever is current.
3. **Network sources** that change.

### Levels of pinning

| Level | Example | Trade-off |
|---|---|---|
| Floating | `python:slim`, `python:latest` | Do not use in production. |
| Major.minor | `python:3.13-slim` | Receives security patches automatically; a patch release can occasionally break something. Common choice. |
| Exact tag | `python:3.13.7-slim-bookworm` | More stable; bump manually (Dependabot/Renovate can automate). |
| Digest | `python:3.13-slim@sha256:<64 hex characters>` | Immutable; exact bytes. Best for reproducibility and supply-chain integrity. |

A **digest** is the SHA-256 hash of the image manifest (the file describing the image's layers). It cannot change without the hash changing. Get it:

```bash
docker buildx imagetools inspect python:3.13-slim
```

Then:

```dockerfile
FROM python:3.13-slim@sha256:0123abcd...ef
```

When a tag and a digest are both present, Docker uses the digest and the tag is only a label. Pin the index digest (the top one) so the pin works on amd64 and arm64.

A pinned digest never receives security updates by itself. Use **Dependabot** or **Renovate** to open pull requests when the tag moves to a new digest. Pinning without automated updates accumulates vulnerabilities.

Pin application dependencies as well: `requirements.txt` with exact versions (ideally with hashes: `pip install --require-hashes`, or a lockfile from `uv`/`pip-tools`/`poetry`), `package-lock.json` with `npm ci`, `go.sum`, Maven/Gradle lockfiles.

---

## 14. Making images small

Smaller images pull faster, cost less to store and contain fewer vulnerabilities. In order of impact:

1. **Smaller base** ([section 12](#12-choosing-a-base-image)): `slim`, distroless or scratch.
2. **Multi-stage builds** ([section 11](#11-multi-stage-builds)): leave compilers and build caches behind.
3. **`.dockerignore`** ([section 4](#4-build-context-and-dockerignore)): keep junk out of `COPY . .`.
4. **Clean up in the same layer** that created the files:
   - apt: `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*`
   - pip: cache mount (preferred) or `--no-cache-dir`
   - npm: `npm ci --omit=dev`, `npm cache clean --force`
5. **Install only runtime dependencies** in the final stage (`--omit=dev`, separate `requirements.txt` and `requirements-dev.txt`).
6. **Avoid `chown -R` / `chmod -R` in a separate `RUN`**; it duplicates every file in a new layer. Use `COPY --chown` / `--chmod`.
7. **Strip binaries** (`-ldflags="-s -w"` in Go; `strip` for C builds).
8. **Do not ship build tools or tests.**
9. **ML**: use CPU-only wheels when there is no GPU (PyTorch's CPU wheel is hundreds of MB smaller than the CUDA one). Do not bake multi-GB model weights into the image unless required ([Chapter 8](08-docker-for-ai-and-ml.md)).

### Measuring

```bash
docker images my-api
docker history my-api:1.0
docker history --no-trunc --human my-api:1.0
```

`docker history` lists each layer, the instruction that created it and its size. `--no-trunc` shows full commands. `--human` (default true) prints sizes like `54MB`.

`<missing>` means the layer ID is not stored locally as a separate image. It is normal for BuildKit-built images.

Tools for exploring layers:

- **dive**: interactive layer explorer. `docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive <image>`. Mounting the Docker socket gives the container control of Docker; run only trusted tools.
- **Docker Scout**: `docker scout quickview <image>` (size and CVE summary).

---

## 15. Running as a non-root user

By default the container process runs as root (UID 0) inside the container. A compromised root process combined with a kernel or configuration bug ("container escape") becomes root on the host. Running as an unprivileged user is low-cost defense in depth, and Kubernetes (`runAsNonRoot`), OpenShift and security scanners often require it. Details: [Chapter 10](10-security.md).

### Debian/Ubuntu-based images

```dockerfile
RUN groupadd --system --gid 10001 app \
 && useradd  --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app app/ ./app/

USER 10001:10001
```

| Part | Meaning |
|---|---|
| `groupadd --system --gid 10001 app` | Create a system group `app` with GID 10001. System accounts are for services. An explicit high ID avoids collisions. |
| `useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app` | Create user `app` with UID 10001 in group `app`, no home directory, and a shell that refuses logins. |
| `COPY --chown=app:app` | Files are owned by the app user instead of root. |
| `USER 10001:10001` | Use the numeric UID:GID. Kubernetes `runAsNonRoot` can verify only a numeric UID; a name like `app` requires reading `/etc/passwd`. |

A high numeric UID (10001) avoids clashes with host UIDs and simplifies policy checks.

Alpine: `addgroup -S -g 10001 app && adduser -S -u 10001 -G app app` (BusyBox syntax). Distroless: use the `nonroot` tag (UID 65532). Some official images include a user: `node` has `node` (UID 1000), `nginx` has `nginx`; `postgres` and `redis` drop privileges in their own entrypoints.

### Typical problems

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` writing to `/app/data` | Directory owned by root | `COPY --chown`, or `RUN mkdir -p /app/data && chown app:app /app/data` before `USER` |
| `bind: permission denied` on port 80 | Non-root users cannot bind ports below 1024 on many systems | Listen on a high port (8000/8080) and map it with `-p 80:8000` |
| Cannot write to a mounted volume | Volume content owned by another UID | Match UIDs, see [Chapter 5](05-storage-and-networking.md#a7-permissions-and-uid-problems) |
| `pip install` fails after `USER` | Writing to system site-packages | Install as root in the builder stage; switch to `USER` at the end |
| Package manager commands fail | They need root | Put `USER` after all root-requiring `RUN` steps |

Do all root-requiring setup first and put `USER` near the end. `docker exec` uses the image's `USER`; add `-u 0` for a root shell when debugging.

---

## 16. PID 1, signals, tini and `--init`

### PID 1

Every Linux process has a process ID (PID). The first process in a container (your `CMD`/`ENTRYPOINT`) gets **PID 1** in the container's own PID namespace. PID 1 behaves differently from other processes:

1. **Signal handling.** The kernel does not apply default actions for signals such as `SIGTERM` and `SIGINT` to PID 1. If the process has not installed handlers, the signals are ignored. `docker stop` sends `SIGTERM`, PID 1 ignores it, Docker waits 10 seconds, then sends `SIGKILL`. The result is slow stops and no graceful shutdown (dropped requests, unclosed database connections, unflushed files).
2. **Zombie reaping.** A finished child stays as a "zombie" entry until its parent reads its exit status. Orphaned processes are re-parented to PID 1, which is expected to reap them. Most apps do not, so zombies accumulate. This is harmless in small numbers but a problem for apps that spawn many subprocesses (headless browsers, build tools, shells).

### Symptoms

- `docker stop` takes exactly 10 seconds.
- Ctrl+C in `docker run` (without `-it`) does nothing.
- Shell-form `CMD`, or a wrapper script that starts the app without `exec`.

### Fixes

1. Use exec form (`CMD ["python", "app.py"]`) and `exec` in scripts, so the app is PID 1.
2. Make sure the app handles `SIGTERM`. Uvicorn, Gunicorn, Node `http` (with `process.on('SIGTERM', ...)`), Go (`signal.Notify`) and Java (shutdown hooks; Spring Boot does this) can, if configured.
3. Add a small init process as PID 1 that forwards signals and reaps zombies. The standard one is **tini**.

**Option A: `docker run --init`** (Docker injects tini):

```bash
docker run --init --rm -p 8000:8000 my-api:1.0
```

`--init` runs Docker's bundled init (`docker-init`, a build of tini) as PID 1 with the command as its child. In Compose: `init: true` under the service.

**Option B: install tini in the image:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

`tini --` runs everything after `--` as the child. On Alpine, `apk add --no-cache tini` installs it at `/sbin/tini`. Use this when you cannot control how the image is started (for example a platform that does not pass `--init`). Kubernetes has no `--init` flag, so images there include an init or rely on the app handling signals.

**Tuning stop behaviour:**

```bash
docker stop -t 30 my-container          # 30 s grace before SIGKILL
docker run --stop-signal SIGINT ...      # or STOPSIGNAL in the Dockerfile
```

### Test

```bash
docker run -d --name sigtest my-api:1.0
time docker stop sigtest
docker rm sigtest
```

A correct container stops in under about 1-2 seconds. A time near 10 seconds means signals are not reaching the app.

---

## 17. Build secrets: never bake secrets into layers

Anything in a layer is permanent and readable by anyone who can pull the image. This includes `ENV`, `ARG` used in a command, copied `.env` files and keys, and files removed in a later layer (the earlier layer still holds them). Registries are not secret stores.

### How an `ARG` secret leaks

```dockerfile
FROM python:3.13-slim
ARG PIP_TOKEN
RUN pip install --index-url "https://__token__:${PIP_TOKEN}@pypi.example.com/simple" mypackage
```

```bash
docker build --build-arg PIP_TOKEN=FAKE-DEMO-TOKEN-123 -t leaky .
docker history --no-trunc leaky
```

Illustrative output (trimmed):

```
IMAGE          CREATED BY
sha256:aaa...  RUN |1 PIP_TOKEN=FAKE-DEMO-TOKEN-123 /bin/sh -c pip install --index-url "https://__token__:FAKE-DEMO-TOKEN-123@pypi.example.com/simple" mypackage # buildkit
sha256:bbb...  ARG PIP_TOKEN
```

The token is in plain text in the image history. Recent Dockerfile frontends may print the build check warning `SecretsUsedInArgOrEnv` for names such as `TOKEN`, `PASSWORD` or `SECRET`. The same leak occurs through `ENV` and `docker inspect`.

Deleting the secret afterwards also fails:

```dockerfile
COPY id_rsa /root/.ssh/id_rsa
RUN git clone git@github.com:org/private.git
# Too late: the COPY layer still contains the key
RUN rm /root/.ssh/id_rsa
```

Extract it with `docker save leaky -o leaky.tar` and open the layer tarballs.

### Correct approach: secret mounts

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
COPY requirements.txt .
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```

```bash
# from a file
docker build --secret id=pip_token,src=./pip_token.txt -t safe .

# or from an environment variable on the build machine
export PIP_TOKEN=...   # real token set in your shell, never committed
docker build --secret id=pip_token,env=PIP_TOKEN -t safe .
```

| Part | Meaning |
|---|---|
| `--secret id=pip_token,src=./pip_token.txt` | Provide a secret named `pip_token` from a local file. |
| `--secret id=pip_token,env=PIP_TOKEN` | Provide it from an environment variable. |
| `--mount=type=secret,id=pip_token` | Inside the `RUN`, the secret is a file at `/run/secrets/pip_token` (change with `target=`), a tmpfs-style mount that exists only for that step. |
| `env=PIP_TOKEN` on the mount | (Dockerfile syntax 1.10+) Expose the secret as an environment variable for that step only: `RUN --mount=type=secret,id=pip_token,env=PIP_TOKEN pip install ...`. |

Nothing is written to any layer. `docker history` shows only `RUN --mount=type=secret,id=pip_token ...`, and the secret is not in the cache key. Verify with `docker history --no-trunc safe`. In Compose use `build.secrets` ([Chapter 6](06-docker-compose.md#secrets-and-configs)).

### Multi-stage as a second line of defence

If a secret must exist in an early stage (for example private registry config), do the secret-dependent work in a builder stage and copy only the results forward. The final image does not contain that stage's layers. A secret mount is still preferable, because builder layers can be pushed or cached by CI.

### Runtime secrets

Secrets needed at runtime (database passwords, API keys) must not be in the image. Provide them at container start: environment variables (acceptable for simple cases), mounted secret files (better), or a secrets manager. See [Chapter 6](06-docker-compose.md#secrets-and-configs) and [Chapter 10](10-security.md).

### If a secret leaks

Treat it as compromised and rotate (revoke and re-issue) it immediately. Deleting the image tag from the registry does not undo the leak: the image may have been pulled, and Git history or CI logs may also hold it.

---

## 18. BuildKit and buildx

### BuildKit

**BuildKit** is Docker's build engine, the default since Docker Engine 23.0. It provides parallel stages, a precise cache with import/export, `RUN --mount`, heredocs, `COPY --link`, build checks, multi-platform builds, and attestations: SBOMs (Software Bill of Materials, a list of everything inside the image) and provenance (a signed record of how and where the image was built).

### buildx

**`docker buildx`** is the CLI plugin that exposes the full BuildKit feature set. In current Docker, `docker build` is effectively `docker buildx build` with the default builder, so they share almost all flags. Use `buildx` explicitly for multiple builders, multi-platform builds, exports and cache backends.

```bash
docker buildx version          # check installation, e.g. "github.com/docker/buildx v0.38.1 ..."
docker buildx ls               # list builder instances
docker buildx inspect --bootstrap
```

### Drivers

A buildx **builder** runs with a **driver**:

| Driver | What it is | Use for |
|---|---|---|
| `docker` | BuildKit built into the Docker daemon | Everyday local builds. Limited: cannot export some cache types or build multi-platform images unless the containerd image store is enabled. |
| `docker-container` | BuildKit in its own container | Multi-platform builds, full cache export, CI. Most flexible. |
| `kubernetes` | BuildKit pods in a cluster | Scalable shared builders. |
| `remote` | Connect to an existing BuildKit daemon | Dedicated build servers. |

Create a container-driver builder:

```bash
docker buildx create --name mybuilder --driver docker-container --use
docker buildx inspect --bootstrap
```

| Part | Meaning |
|---|---|
| `create --name mybuilder` | Name the new builder. |
| `--driver docker-container` | Run BuildKit in a container (Docker pulls `moby/buildkit`). |
| `--use` | Make it the current builder. |
| `inspect --bootstrap` | Start it now and print details. |

Switch back with `docker buildx use default`. Remove a builder you created with `docker buildx rm mybuilder`.

### Output destinations (`--load`, `--push`, `--output`)

With the container driver, the result stays inside the builder unless an output is specified:

| Flag | Meaning |
|---|---|
| `--load` | Load the single-platform image into the local Docker image store. |
| `--push` | Push to the registry named in the tag. |
| `--output type=local,dest=./out` | Write the image filesystem to a local folder (useful for extracting build artifacts). |
| `--output type=oci,dest=image.tar` | Write an OCI-format tarball. |

### Build checks (linting)

```bash
docker build --check .                # run Dockerfile checks without building
docker buildx build --call=check .    # equivalent call syntax in newer buildx
```

Build checks (Dockerfile frontend 1.8+) warn about undefined variables, `FROM` with `latest`, `JSONArgsRecommended` (use exec form), `SecretsUsedInArgOrEnv` and others. Silence or promote a rule with `# check=skip=RuleName;error=true` at the top of the file.

### Attestations: SBOM and provenance

```bash
docker buildx build --sbom=true --provenance=mode=max -t registry.example.com/me/app:1.0 --push .
```

| Flag | Meaning |
|---|---|
| `--sbom=true` | Generate an SBOM and attach it to the image in the registry. |
| `--provenance=mode=max` | Attach detailed build provenance. `max` includes more detail such as build arguments, so do not put secrets in build args. BuildKit may add minimal provenance by default in some setups. |

Enterprise supply-chain policies increasingly require these ([Chapter 10](10-security.md), [Chapter 11](11-production-and-enterprise.md)).

---

## 19. Multi-platform builds (amd64 and arm64)

An image contains compiled binaries for one CPU architecture:

- **amd64** (x86_64): most servers, Intel/AMD PCs.
- **arm64** (aarch64): Apple Silicon Macs, AWS Graviton, Raspberry Pi 4/5, many edge devices.

An amd64 image on an arm64 machine runs only through emulation (slow, sometimes broken), and the reverse is also true. A **multi-platform image** is one tag (`my-api:1.0`) that is an **index** (manifest list) pointing to one image per platform. The Docker client pulls the matching one.

```
 my-api:1.0  (image index)
    |-- linux/amd64  -> layers A
    |-- linux/arm64  -> layers B
```

### Building

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t youruser/my-api:1.0 --push .
```

| Flag | Meaning |
|---|---|
| `--platform linux/amd64,linux/arm64` | Build for both targets. |
| `-t youruser/my-api:1.0` | Tag including the registry namespace. |
| `--push` | Multi-platform results cannot be loaded into the classic local store, so they are pushed to the registry. With the **containerd image store** (default on new Docker Desktop installs in recent versions), `--load` can hold multi-platform images locally. `docker info` showing Storage Driver `overlayfs` indicates the containerd store. |

Inspect the result: `docker buildx imagetools inspect youruser/my-api:1.0`.

### How the foreign platform is built

1. **Emulation with QEMU** (a CPU emulator): the builder runs arm64 `RUN` commands on an amd64 CPU. No Dockerfile changes, but 5-20x slower for compile-heavy steps. Docker Desktop has QEMU/binfmt configured. On a plain Linux server install it once: `docker run --privileged --rm tonistiigi/binfmt --install all`. `--privileged` gives that container broad host access and registers emulators with the kernel; run only trusted images.
2. **Native builders or cross-compilation**: add a native arm64 node to the builder, or cross-compile (below).

### Cross-compilation pattern (fast, no emulation)

BuildKit sets automatic build arguments: `BUILDPLATFORM` (the machine running the build), `TARGETPLATFORM` (the platform being built for), `TARGETOS`, `TARGETARCH`. For compiled languages, run the compiler on the native platform and emit the target architecture:

```dockerfile
# syntax=docker/dockerfile:1
FROM --platform=$BUILDPLATFORM golang:1.25 AS build
ARG TARGETOS
ARG TARGETARCH
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH go build -o /out/server ./cmd/server

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/server /server
ENTRYPOINT ["/server"]
```

| Part | Meaning |
|---|---|
| `FROM --platform=$BUILDPLATFORM` | Run the builder stage natively regardless of target. |
| `ARG TARGETOS` / `ARG TARGETARCH` | Filled in by BuildKit; must be declared to use them. |
| `GOOS` / `GOARCH` | Go's cross-compile switches. |

The final stage is built for the target platform but only copies a file, so no emulation is needed.

### Pitfalls

- Python/Node packages with native code need arm64 wheels or compile under emulation (slow).
- Base images must publish the target platforms (official images mostly do).
- GPU/CUDA images are usually amd64 only (Jetson/ARM variants exist separately, [Chapter 8](08-docker-for-ai-and-ml.md)).
- `exec format error` at runtime means the image was built for a different architecture. Use the correct image or `--platform` on `docker run`.
- Run a foreign-architecture image deliberately with `docker run --platform linux/amd64 ...` (uses emulation on Apple Silicon).

---

## 20. Exporting the build cache for CI

Locally, the build cache stays on disk between builds. CI jobs (GitHub Actions, GitLab CI, Jenkins agents) often start on a fresh machine with an empty cache, so every build is cold. **Export** the cache to persistent storage and **import** it on the next run.

```bash
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/me/my-api:buildcache \
  --cache-to   type=registry,ref=registry.example.com/me/my-api:buildcache,mode=max \
  -t registry.example.com/me/my-api:1.0 --push .
```

| Part | Meaning |
|---|---|
| `--cache-from` | Where to read previously exported cache. |
| `--cache-to` | Where to write cache after the build. |
| `type=registry` | Store the cache as a special image tag (`:buildcache`) in the registry. Works anywhere. |
| `mode=max` | Export cache for all layers of all stages. Default `mode=min` exports only the final image's layers and misses builder-stage layers, so use `max` for multi-stage builds. |

Cache backends:

| Backend | Syntax | Notes |
|---|---|---|
| Registry | `type=registry,ref=<image>:<tag>` | Universal; uses registry storage. |
| GitHub Actions cache | `type=gha` | Convenient in GitHub Actions; limited storage per repo. |
| Local directory | `type=local,src=/tmp/cache` / `dest=/tmp/cache` | For CI systems that persist a directory between runs. |
| Inline | `type=inline` (for `--cache-to`) | Embeds cache metadata in the image; only `mode=min`. |
| S3 / Azure Blob | `type=s3`, `type=azblob` | Cloud object storage. |

Cache export requires a builder with the `docker-container` driver (or another non-default driver), not the plain `docker` driver. GitHub Actions example (`docker/build-push-action` and `docker/setup-buildx-action` are maintained by Docker):

```yaml
- uses: docker/setup-buildx-action@v3
- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

Check the actions' README for current recommended major versions. More CI/CD detail: [Chapter 11](11-production-and-enterprise.md).

---

## 21. `docker init`: generate a starter setup

```bash
cd my-project
docker init
```

`docker init` asks a few questions (platform, version, port, run command) and writes:

| File | Purpose |
|---|---|
| `Dockerfile` | Dockerfile for the stack (multi-stage where appropriate, non-root user, cache mounts) |
| `compose.yaml` | Compose file to run it (`docker compose up --build`) |
| `.dockerignore` | Ignore list |
| `README.Docker.md` | Build, run and deploy instructions |

Supported templates (per Docker's reference): ASP.NET Core, Go, Java (Maven), Node, PHP with Apache, Python, Rust, and "Other". If any target file exists, it warns and offers to overwrite all of them. Overwritten files are not recoverable, so commit or back up first.

Treat the output as a first draft: review and adapt it.

---

## 22. Linting and inspecting images

### hadolint

**hadolint** checks a Dockerfile against best-practice rules and runs **ShellCheck** on `RUN` scripts.

```bash
# macOS/Linux/Git Bash:
docker run --rm -i hadolint/hadolint < Dockerfile

# Windows PowerShell (no "<" redirection operator):
Get-Content Dockerfile | docker run --rm -i hadolint/hadolint
```

| Part | Meaning |
|---|---|
| `--rm` | Remove the container after it runs. |
| `-i` | Keep stdin open so the Dockerfile text can be piped in. |
| `hadolint/hadolint` | The image. |

Common rules (unverified wording; check `hadolint --help` or its wiki):

| Rule | Meaning |
|---|---|
| DL3006 / DL3007 | Tag the base image / do not use `latest` |
| DL3008 | Pin apt package versions (often ignored in practice) |
| DL3013 | Pin pip package versions |
| DL3002 | Last `USER` should not be root |
| DL3025 | Use JSON (exec) notation for `CMD`/`ENTRYPOINT` |
| DL3059 | Multiple consecutive `RUN` instructions; consider consolidating |
| DL4006 | Set `SHELL -o pipefail` before `RUN` with a pipe |

Ignore rules in `.hadolint.yaml` (`ignored: [DL3008]`) or inline with `# hadolint ignore=DL3008`. Run hadolint in CI so Dockerfile mistakes fail pull requests.

---

## 23. A full production-grade Dockerfile for FastAPI

Project layout:

```
my-api/
  Dockerfile
  .dockerignore
  requirements.txt          # pinned runtime dependencies
  app/
    __init__.py
    main.py                 # FastAPI app with GET / and GET /health (see Chapter 6)
```

`requirements.txt` pins `fastapi` and `uvicorn[standard]` (example versions; generate a full pinned list with `pip freeze` or a lockfile tool).

**`Dockerfile`:**

```dockerfile
# syntax=docker/dockerfile:1

# ---------------------------------------------------------------------------
# Global build argument: change the Python version without editing the file:
#   docker build --build-arg PYTHON_VERSION=3.12 .
# For maximum reproducibility, replace the tag with a digest:
#   FROM python:${PYTHON_VERSION}-slim@sha256:<digest> AS base
# ---------------------------------------------------------------------------
ARG PYTHON_VERSION=3.13

# ======================= Stage 1: base (shared settings) =====================
FROM python:${PYTHON_VERSION}-slim AS base

# PYTHONDONTWRITEBYTECODE: do not write .pyc files (smaller image)
# PYTHONUNBUFFERED:        flush stdout/stderr immediately so `docker logs` is live
# PIP_DISABLE_PIP_VERSION_CHECK: skip the network call that checks for new pip
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# ======================= Stage 2: builder (large, discarded) =================
FROM base AS builder

# Compilers are needed only if a dependency must be built from source.
# They stay in this stage and never reach the final image.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    rm -f /etc/apt/apt.conf.d/docker-clean \
 && apt-get update \
 && apt-get install -y --no-install-recommends build-essential

# A virtualenv at a fixed path is easy to copy to the final stage.
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Bind-mount requirements.txt (no COPY layer) and keep pip's download cache
# between builds. The layer is rebuilt only when requirements.txt changes.
RUN --mount=type=cache,target=/root/.cache/pip \
    --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt

# ======================= Stage 3: runtime (small, shipped) ===================
FROM base AS runtime

# OCI standard labels: registries and scanners understand these.
LABEL org.opencontainers.image.title="my-api" \
      org.opencontainers.image.description="Example FastAPI service" \
      org.opencontainers.image.source="https://github.com/your-user/my-api" \
      org.opencontainers.image.licenses="MIT"

# Create an unprivileged user with a fixed numeric ID.
RUN groupadd --system --gid 10001 app \
 && useradd  --system --uid 10001 --gid app --no-create-home \
             --shell /usr/sbin/nologin app

# Bring in only the installed dependencies from the builder.
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copy application code last (it changes most often) and set the owner in the
# same step (avoids a size-doubling `chown -R` layer).
COPY --chown=app:app app/ ./app/

# Drop privileges. Use the numeric ID so orchestrators can verify non-root.
USER 10001:10001

# Documentation only. Publish the port with -p / ports:.
EXPOSE 8000

# Healthy when /health answers. Uses only the Python standard library because
# slim images do not ship curl.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]

# Graceful shutdown signal for `docker stop` (also the default).
STOPSIGNAL SIGTERM

# Exec form: uvicorn is PID 1 and receives SIGTERM directly.
# 0.0.0.0 makes the server reachable from outside the container.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`.dockerignore`:** use the starter from [section 4](#4-build-context-and-dockerignore).

**Build and run:**

```bash
docker build -t my-api:1.0 .
docker run -d --name my-api --init -p 8000:8000 my-api:1.0
curl http://localhost:8000/health
docker ps
docker logs my-api
docker exec my-api id
```

Expected results:

- `curl` prints `{"status":"ok"}`.
- `docker ps` STATUS changes from `(health: starting)` to `(healthy)` within about 30 seconds.
- `docker exec my-api id` prints `uid=10001(app) gid=10001(app) groups=10001(app)`, confirming the process is not root.

Clean up: `docker stop my-api && docker rm my-api`.

### Variations

- **Gunicorn with Uvicorn workers** for multi-process: `CMD ["gunicorn", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "-b", "0.0.0.0:8000"]`. In orchestrated environments many teams run one worker per container and scale with replicas, which fits autoscaling and memory limits; choose per platform.
- **`--proxy-headers`** and `--forwarded-allow-ips` when behind a reverse proxy ([Chapter 5](05-storage-and-networking.md#b12-reverse-proxies-in-front)).
- **Read-only root filesystem** at run time: `docker run --read-only --tmpfs /tmp ...` ([Chapter 10](10-security.md)).

---

## 24. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Every code edit reinstalls dependencies | `COPY . .` before installing dependencies | Copy the manifest first, install, then copy source ([section 10](#10-ordering-instructions-for-cache-efficiency-before-and-after)) |
| Builds change over time | `latest` or no tag | Pin tag or digest ([section 13](#13-reproducibility-tags-digests-pinning)) |
| Container runs as root | No `USER` | Numeric non-root `USER` ([section 15](#15-running-as-a-non-root-user)) |
| Secret visible in history or layers | Secret in `ENV`, `ARG` or a copied file, even if deleted later | `--mount=type=secret` ([section 17](#17-build-secrets-never-bake-secrets-into-layers)) |
| `.git`, `.env`, `node_modules` in the image | No `.dockerignore` | Add one ([section 4](#4-build-context-and-dockerignore)) |
| `docker stop` takes 10 s | Shell-form `CMD`/`ENTRYPOINT`, or script without `exec` | Exec form and `exec` ([6](#6-cmd-vs-entrypoint-exec-form-vs-shell-form), [16](#16-pid-1-signals-tini-and---init)) |
| `CMD ['a']` behaves like shell form | Single quotes make invalid JSON | Double quotes |
| `404` on `apt-get install` after months | `apt-get update` on its own `RUN` | One `RUN` for update and install ([section 3](#3-layers-and-the-build-cache)) |
| Image stays large after `rm` or `chown -R` | Deletion or ownership change in a later layer | Same-layer cleanup, multi-stage, `COPY --chown` |
| `no such file or directory` for an existing script | CRLF line endings | LF endings ([section 6](#6-cmd-vs-entrypoint-exec-form-vs-shell-form)) |
| Server unreachable from outside | Bound to `127.0.0.1` | Bind to `0.0.0.0` |
| Tarball unpacked unexpectedly | `ADD` instead of `COPY` | `COPY` |
| Dev tools and tests in the image | Single-stage build | Multi-stage |
| Slow builds, large Python/ML image | Alpine with heavy dependencies | `-slim` ([section 12](#12-choosing-a-base-image)) |
| Slow pulls and deploys | Multi-GB model weights baked in | See [Chapter 8](08-docker-for-ai-and-ml.md) |
| Data lost; anonymous volumes multiply | `VOLUME` in an application Dockerfile | Mount named volumes at run time ([5.10](#510-volume-declare-a-mount-point)) |
| Port not reachable | `EXPOSE` assumed to publish | `-p` / `ports:` |
| Container permanently `unhealthy` | `HEALTHCHECK` uses `curl` not present in the image | Use a tool the image has ([5.11](#511-healthcheck-tell-docker-how-to-test-the-app)) |

---

## 25. Summary

- Each file-changing instruction creates an immutable layer; the cache invalidates everything after the first changed layer. Order instructions from least to most frequently changing.
- Use `COPY`, exec-form `CMD`/`ENTRYPOINT`, a numeric non-root `USER`, a `HEALTHCHECK` and a `.dockerignore`.
- Use multi-stage builds, `-slim` bases (distroless/scratch for static binaries), and pinned base images.
- Never put secrets in layers; use `--mount=type=secret`.
- Make PID 1 handle signals: exec form, `exec` in scripts, `--init` or tini.

**Next:** [Chapter 5: Storage and Networking](05-storage-and-networking.md).
