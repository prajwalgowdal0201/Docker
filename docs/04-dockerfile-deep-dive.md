# Chapter 4: The Dockerfile Deep Dive

> **What you will learn in this chapter**
>
> - What a Dockerfile really is and how `docker build` turns it into an image
> - Every Dockerfile instruction, with every flag explained
> - How layers and the build cache work (and how to make your builds 10x faster)
> - Build context, `.dockerignore`, and why they matter
> - Multi-stage builds in Python, Node.js, Go and Java
> - How to choose a base image (Debian vs slim vs Alpine vs distroless vs scratch)
> - Reproducibility, image size, non-root users, PID 1 and signals
> - Build secrets done right (and the proof of what happens when you do them wrong)
> - BuildKit, buildx, multi-platform builds, CI cache export, `docker init`, and linting
> - A complete, annotated, production-grade Dockerfile for a FastAPI app

**Prerequisites:** [Chapter 1](01-what-is-docker.md) (what images and containers are) and [Chapter 3](03-core-cli-commands.md) (the basic CLI). If you can run `docker run`, `docker images` and `docker ps`, you are ready.

**Where this fits:** The Dockerfile is the single most important file in the Docker world. Everything else in this guide (Compose in [Chapter 6](06-docker-compose.md), security in [Chapter 10](10-security.md), production in [Chapter 11](11-production-and-enterprise.md)) builds on the images you produce here. Time spent mastering the Dockerfile pays back more than any other Docker skill.

---

## Table of contents

1. [What a Dockerfile is](#1-what-a-dockerfile-is)
2. [Your first Dockerfile, line by line](#2-your-first-dockerfile-line-by-line)
3. [Layers and the build cache](#3-layers-and-the-build-cache)
4. [Build context and `.dockerignore`](#4-build-context-and-dockerignore)
5. [The instruction reference](#5-the-instruction-reference)
6. [CMD vs ENTRYPOINT, exec form vs shell form](#6-cmd-vs-entrypoint-exec-form-vs-shell-form)
7. [ARG vs ENV](#7-arg-vs-env)
8. [Heredocs](#8-heredocs)
9. [BuildKit power features: `RUN --mount` and friends](#9-buildkit-power-features-run---mount-and-friends)
10. [Ordering instructions for cache efficiency (before and after)](#10-ordering-instructions-for-cache-efficiency-before-and-after)
11. [Multi-stage builds](#11-multi-stage-builds)
12. [Choosing a base image](#12-choosing-a-base-image)
13. [Reproducibility: tags, digests, pinning](#13-reproducibility-tags-digests-pinning)
14. [Making images small](#14-making-images-small)
15. [Running as a non-root user](#15-running-as-a-non-root-user)
16. [PID 1, signals, tini and `--init`](#16-pid-1-signals-tini-and---init)
17. [Build secrets: never bake secrets into layers](#17-build-secrets-never-bake-secrets-into-layers)
18. [BuildKit and buildx](#18-buildkit-and-buildx)
19. [Multi-platform builds (amd64 and arm64)](#19-multi-platform-builds-amd64-and-arm64)
20. [Exporting the build cache for CI](#20-exporting-the-build-cache-for-ci)
21. [`docker init`: generate a starter setup](#21-docker-init-generate-a-starter-setup)
22. [Linting and inspecting images](#22-linting-and-inspecting-images)
23. [A full production-grade Dockerfile for FastAPI](#23-a-full-production-grade-dockerfile-for-fastapi)
24. [Common mistakes](#24-common-mistakes)
25. [Exercises](#25-exercises)
26. [Summary](#26-summary)

---

## 1. What a Dockerfile is

A **Dockerfile** is a plain text file (literally named `Dockerfile`, no extension) containing a list of instructions that tell Docker how to build an **image**. An image, as you saw in [Chapter 1](01-what-is-docker.md), is a read-only package holding a filesystem plus some metadata (which command to run, which user, which environment variables).

The best analogy is a **recipe**:

| Cooking | Docker |
|---|---|
| Recipe card | Dockerfile |
| Following the recipe | `docker build` |
| A frozen meal, ready to ship | Image |
| Heating and serving the meal | `docker run` (creates a container) |

A Dockerfile is **declarative-ish and sequential**: Docker reads it top to bottom, executing each instruction, and each instruction that changes the filesystem produces a new **layer** (more on that in section 3).

### The build pipeline

```
 Dockerfile  +  build context (your files)
      |
      v
 +-------------+      +----------------------+
 | docker build| ---> | BuildKit (the engine |
 |  (client)   |      |  that does the work) |
 +-------------+      +----------+-----------+
                                 |
          reads FROM, runs RUN, copies COPY, ... one step at a time
                                 |
                                 v
                  +-----------------------------+
                  |  IMAGE = stack of layers    |
                  |  + config (CMD, ENV, USER)  |
                  +-----------------------------+
                                 |
                  docker run  ---+--->  container
```

Key vocabulary introduced here:

- **BuildKit**: the modern build engine inside Docker. It builds steps in parallel when it can, caches aggressively, and supports advanced features like secrets and cache mounts. It has been the default builder since Docker Engine 23.0, so you get it automatically.
- **Build context**: the set of files you send to the builder (usually the current directory). `COPY` can only see files inside the context.
- **Layer**: one filesystem "delta" (a set of added/changed/deleted files) produced by one instruction.
- **Registry**: a server that stores images (Docker Hub, GitHub Container Registry, AWS ECR, etc.).

---

## 2. Your first Dockerfile, line by line

Let's build a tiny Python web app so every instruction has something real to act on. Create an empty folder and these three files.

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

Note `0.0.0.0`: inside a container, a server must listen on **all interfaces**, not just `127.0.0.1`, otherwise nothing outside the container can reach it (see [Chapter 5](05-storage-and-networking.md#b4-the-1-beginner-bug-localhost-confusion)).

**`Dockerfile`**

```dockerfile
FROM python:3.13-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

Explanation, line by line:

| Line | What it does |
|---|---|
| `FROM python:3.13-slim` | Start from an existing image: the official Python 3.13 image, "slim" variant (a trimmed-down Debian with Python installed). Every Dockerfile starts with `FROM`. You never build from nothing; you build on top of something. |
| `WORKDIR /app` | Set the "current directory" for all following instructions, and create `/app` if it doesn't exist. Like `mkdir -p /app && cd /app`, but it persists. |
| `COPY app.py .` | Copy `app.py` from your build context (host) into the image. The `.` means "the current WORKDIR", so the file lands at `/app/app.py`. |
| `EXPOSE 8000` | Documentation only: "this app listens on port 8000". It does **not** open or publish anything (see [Chapter 5](05-storage-and-networking.md#exposing-vs-publishing)). |
| `CMD ["python", "app.py"]` | The default command to run when a container starts from this image. |

**Build it:**

```bash
docker build -t hello-py:1.0 .
```

- `docker build`: build an image from a Dockerfile.
- `-t hello-py:1.0`: **tag** the result with the name `hello-py` and version `1.0` (format `name:tag`). Without `-t` the image only has an ID.
- `.`: the **build context**, the directory whose contents are sent to the builder. Docker also looks for `./Dockerfile` here by default.

Example output (BuildKit's progress style; yours will differ in timings and IDs):

```
[+] Building 6.8s (8/8) FINISHED                              docker:desktop-linux
 => [internal] load build definition from Dockerfile                          0.0s
 => => transferring dockerfile: 118B                                          0.0s
 => [internal] load metadata for docker.io/library/python:3.13-slim           1.2s
 => [internal] load .dockerignore                                             0.0s
 => [1/3] FROM docker.io/library/python:3.13-slim@sha256:ab12...              4.9s
 => [internal] load build context                                             0.0s
 => => transferring context: 361B                                             0.0s
 => [2/3] WORKDIR /app                                                        0.1s
 => [3/3] COPY app.py .                                                       0.0s
 => exporting to image                                                        0.1s
 => => naming to docker.io/library/hello-py:1.0                               0.0s
```

**Run it:**

```bash
docker run --rm -p 8000:8000 hello-py:1.0
```

- `--rm`: delete the container automatically when it stops (keeps your machine tidy).
- `-p 8000:8000`: **publish** container port 8000 on host port 8000 (format `HOST:CONTAINER`).

In another terminal: `curl http://localhost:8000` prints `Hello from inside a container!`. Press Ctrl+C to stop.

---

## 3. Layers and the build cache

This is the most important concept in the chapter. If you only deeply understand one thing, make it this.

### What is a layer?

Each instruction that changes the filesystem (`RUN`, `COPY`, `ADD`) creates a **layer**: a compressed archive of the files that were added, changed, or deleted by that step. An image is an ordered stack of layers. Instructions like `ENV`, `CMD`, `EXPOSE`, `LABEL` only change the image's metadata (they add no files, so they are effectively "empty" layers).

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
 
   Running container adds one thin WRITABLE layer on top:
 +-----------------------------------------+
 | Container layer (your runtime changes)  |
 +-----------------------------------------+
```

Layers are stacked using a **union filesystem** (on modern Docker, `overlayfs`): a trick that presents several directories layered on top of each other as one merged directory. If a file exists in a higher layer, it hides the same file in a lower layer.

Layers are **content-addressed**: each is identified by a SHA-256 hash of its content. This gives you two superpowers:

1. **Sharing**: ten images built on the same `python:3.13-slim` store those base layers on disk only once.
2. **Caching**: if a layer's inputs have not changed, Docker reuses the previous result instead of re-running the step.

### Deleting files in a later layer does NOT shrink the image

Layers only ever add. If layer 2 downloads a 500 MB file and layer 3 runs `rm` on it, layer 3 records a "whiteout" marker saying "this file is deleted", but layer 2 still physically contains the 500 MB. The image is still 500 MB bigger. This is why you see Dockerfiles that download, use and delete **within a single `RUN`**:

```dockerfile
# BAD: the tarball lives forever in layer 2
RUN curl -O https://example.com/big.tar.gz
RUN tar xzf big.tar.gz && rm big.tar.gz

# GOOD: download, extract and delete in ONE layer
RUN curl -O https://example.com/big.tar.gz \
 && tar xzf big.tar.gz \
 && rm big.tar.gz
```

(Multi-stage builds, section 11, are an even cleaner answer.)

### How the build cache decides to reuse a layer

For each instruction, BuildKit computes a **cache key** and asks "have I executed exactly this before, on exactly this parent layer?" The rules:

| Instruction | The cache key is based on |
|---|---|
| `RUN` | The **text of the command** (and the parent layer). Docker does **not** re-run it just because the outside world changed. `RUN apt-get update` is cached forever until you edit the line or an earlier layer changes. |
| `COPY` / `ADD` | A **checksum of the content** of the files being copied (and their metadata like permissions). File modification time is ignored; only content/metadata changes matter. |
| `ADD <url>` | The URL (and, for HTTP sources, validators like ETag/Last-Modified where available). For reproducibility use `--checksum`. |
| `ENV`, `ARG`, `LABEL`, `USER`, ... | The instruction text and value. Changing an `ARG` value invalidates the cache from the first instruction that **uses** it. |

And the crucial **cascade rule**:

> **Once one layer is invalidated (cache miss), every layer after it is rebuilt too.**

Why? Because each layer is built on top of the previous one; if the parent changed, the child can't be assumed valid.

```
 Step 1  FROM python:3.13-slim        CACHED
 Step 2  WORKDIR /app                 CACHED
 Step 3  COPY requirements.txt .      CACHED   (file unchanged)
 Step 4  RUN pip install ...          CACHED
 Step 5  COPY . .                     MISS     (you edited main.py)
 Step 6  RUN python -m compileall .   REBUILT  (cascade)
 Step 7  CMD [...]                    REBUILT  (cascade)
```

So the golden rule of Dockerfile ordering is:

> **Put the things that change least at the top and the things that change most at the bottom.**

Base image and system packages change rarely; dependency lists change occasionally; your source code changes constantly. We'll prove the benefit with a timed before/after example in [section 10](#10-ordering-instructions-for-cache-efficiency-before-and-after).

### Cache gotcha: stale `apt-get update`

Because `RUN` is cached by its text, this is a classic bug:

```dockerfile
RUN apt-get update
# Months later, the cached update above is stale -> 404 on packages
RUN apt-get install -y curl
```

Always **combine them in one `RUN`** so they are cached (and invalidated) together:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*
```

Flags explained: `-y` answers "yes" to prompts (builds are non-interactive); `--no-install-recommends` skips optional "recommended" packages that bloat the image; the final `rm -rf /var/lib/apt/lists/*` deletes the package index downloaded by `apt-get update` (it's useless at runtime and would sit in the layer).

### Forcing a rebuild

```bash
docker build --no-cache -t hello-py:1.0 .     # ignore the cache entirely
docker build --pull -t hello-py:1.0 .         # always check for a newer base image
docker builder prune                           # delete the build cache (frees disk space)
```

- `--no-cache`: don't use any cached layers; rebuild everything.
- `--pull`: before building, re-check the registry for a newer version of the `FROM` image. Without it, Docker reuses a locally cached base image even if the tag has since moved.
- `docker builder prune`: removes cached build data. Safe, but your next build will be slow. (Do not run it on a shared build server mid-day.)

---

## 4. Build context and `.dockerignore`

When you run `docker build .`, the final `.` is the **build context**. The client packages the files in that directory and sends them to the builder. Only files in the context can be referenced by `COPY` and `ADD`. (BuildKit is smart and transfers only the files the Dockerfile actually needs, but ignoring things up front is still important for speed and safety.)

Common beginner disasters:

- Running `docker build` from your home directory (`~`), which tries to send gigabytes.
- A `node_modules/` or `.venv/` folder (hundreds of MB, and platform-specific!) being sent and then copied into the image by `COPY . .`.
- `.git/` (the whole history, sometimes with secrets) ending up inside the image.
- `.env` files with credentials ending up inside the image.

The fix is a **`.dockerignore`** file placed next to the Dockerfile (in the context root). It uses patterns similar to `.gitignore`, and files matching it are excluded from the context entirely.

**A solid starter `.dockerignore`:**

```gitignore
# Version control
.git
.gitignore

# Secrets and local config - NEVER let these into an image
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

# Build outputs, editor files, OS junk
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

# Docs and tests (if you don't need them at runtime)
docs/
tests/
*.md
```

Pattern rules worth knowing:

| Pattern | Meaning |
|---|---|
| `node_modules` | Ignore `node_modules` in the context **root** only. |
| `**/node_modules` | Ignore `node_modules` at any depth (important in monorepos). |
| `*.log` | Any file ending in `.log` in the root. |
| `**/*.log` | Any `.log` file anywhere. |
| `!README.md` | **Exception**: re-include a file excluded by an earlier pattern. The **last matching line wins**, so exceptions must come after the rule they override. |
| `# comment` | A comment. |

> **Tip: allowlist style.** For maximum safety, ignore everything and re-include only what you need:
>
> ```gitignore
> *
> !pyproject.toml
> !requirements.txt
> !app/
> ```
>
> This way a new file you add later (a stray `.env`) is excluded by default.

> **Tip: per-Dockerfile ignore files.** If you have `Dockerfile.dev`, BuildKit will use `Dockerfile.dev.dockerignore` if it exists, otherwise it falls back to `.dockerignore`.

> **Warning:** `.dockerignore` excludes files from the *context*. If you write `.dockerignore` with `Dockerfile` in it, `COPY Dockerfile .` won't work, though the build itself still uses your Dockerfile (Docker handles that specially). Ignoring it is fine and common.

---

## 5. The instruction reference

Here is every instruction, in the order you would normally meet them. For each: what it does, syntax, flags, and gotchas.

### 5.0 The `syntax` directive (first line, optional but recommended)

```dockerfile
# syntax=docker/dockerfile:1
```

This is a **parser directive**: a special comment that must be the very first line (before any blank line or other comment). It tells BuildKit which **Dockerfile frontend** (the component that parses the Dockerfile) to use. `docker/dockerfile:1` means "download the latest stable 1.x frontend from Docker Hub before building", so you get new features (heredocs, `--link`, `--parents`, build checks, ...) even if your local Docker is older. The official Dockerfile reference recommends this exact line.

Other parser directives: `# escape=\`` (changes the escape character, mostly useful for Windows containers) and `# check=skip=...;error=true` (configures build checks, Dockerfile 1.8+).

Trade-off: the syntax image is fetched from the network, so fully offline or locked-down builds may prefer to pin a specific version (e.g. `docker/dockerfile:1.10`) or mirror it internally. Modern Docker bundles a recent frontend, so omitting the line still works for most features; adding it makes your intent explicit and your build independent of the local Docker version.

### 5.1 `FROM`: choose the starting point

```dockerfile
FROM [--platform=<platform>] <image>[:<tag>|@<digest>] [AS <name>]
```

- `<image>:<tag>`: e.g. `python:3.13-slim`. If you omit the tag, it defaults to `latest`, which is almost never what you want in production (see [section 13](#13-reproducibility-tags-digests-pinning)).
- `@sha256:...`: pin to an exact, immutable image **digest**.
- `AS <name>`: names this **build stage** so later stages can copy from it ([multi-stage builds](#11-multi-stage-builds)).
- `--platform=...`: force a platform for this stage (e.g. `--platform=$BUILDPLATFORM` for cross-compilation, see [section 19](#19-multi-platform-builds-amd64-and-arm64)).
- `FROM scratch`: the special empty image (no OS at all), used for fully static binaries.

A Dockerfile can have **multiple** `FROM` lines; each starts a new stage. The final stage becomes the image you ship.

### 5.2 `ARG`: build-time variables

```dockerfile
ARG PYTHON_VERSION=3.13
FROM python:${PYTHON_VERSION}-slim
```

`ARG` defines a variable available **only while building** (not in the running container). You can set it at build time: `docker build --build-arg PYTHON_VERSION=3.12 .`. An `ARG` declared **before** the first `FROM` can be used in `FROM` lines but is not visible inside stages unless you re-declare it (`ARG PYTHON_VERSION` again inside the stage). Full comparison with `ENV` in [section 7](#7-arg-vs-env).

BuildKit also provides automatic platform ARGs (`TARGETPLATFORM`, `TARGETOS`, `TARGETARCH`, `BUILDPLATFORM`, ...), which you must re-declare inside a stage to use.

### 5.3 `ENV`: environment variables baked into the image

```dockerfile
ENV APP_ENV=production \
    PORT=8000
```

Sets environment variables that exist at build time (for later instructions) **and** at runtime (inside every container). Always use the `KEY=value` form; the older `ENV KEY value` form is discouraged and may be removed in a future release. Values are visible to anyone who can inspect the image (`docker inspect`, `docker history`), so **never put secrets in `ENV`**.

### 5.4 `WORKDIR`: set the working directory

```dockerfile
WORKDIR /app
```

Sets the directory for `RUN`, `CMD`, `ENTRYPOINT`, `COPY` and `ADD` that follow, and creates it if missing. Always use absolute paths. Prefer `WORKDIR` over `RUN cd /app && ...` (which only affects that one `RUN`) and over `RUN mkdir`. You can use it several times; relative paths accumulate (`WORKDIR /a` then `WORKDIR b` = `/a/b`). It can use `ENV` variables.

### 5.5 `COPY` vs `ADD`: put files into the image

```dockerfile
COPY [--chown=<user>:<group>] [--chmod=<perms>] [--from=<stage|image|context>] [--link] [--parents] [--exclude=<pattern>] <src>... <dest>
ADD  [--chown=...] [--chmod=...] [--checksum=<hash>] [--keep-git-dir] [--unpack] [--link] [--exclude=...] <src>... <dest>
```

| | `COPY` | `ADD` |
|---|---|---|
| Copies local files/folders from the build context | Yes | Yes |
| Copies from another build stage or image (`--from`) | Yes | No |
| Downloads from an HTTP(S) URL | No | Yes |
| Clones a Git repository | No | Yes |
| **Auto-extracts** a local `.tar`, `.tar.gz`, etc. into the destination | No | **Yes** (a surprise!) |
| Predictability | High | Lower |

**Rule of thumb: use `COPY` always; use `ADD` only when you specifically want tar auto-extraction or a remote source.** `ADD`'s "magic" has caused many surprises (a tarball you only wanted to copy gets silently unpacked). For downloads, `RUN curl -fsSL ... | ...` or the `ADD --checksum=sha256:... https://...` form (which verifies integrity) are both better than a bare `ADD <url>`.

Flags explained:

- **`--chown=user:group`**: sets the owner of the copied files. By default files are copied as root (UID 0), which is a problem when you later run as a non-root user. Use `COPY --chown=app:app . .` Numeric IDs work too (`--chown=10001:10001`) and are safer when the user does not exist yet in that stage.
- **`--chmod=755`**: sets permissions during the copy (octal, or symbolic like `a+x` in newer syntax versions). Saves a separate `RUN chmod` layer (which would duplicate the file's size).
- **`--from=<name>`**: copy from an earlier stage (`--from=builder`), from any image (`--from=nginx:alpine`), or from an additional named build context. This is the heart of multi-stage builds.
- **`--link`** (Dockerfile syntax 1.4+): makes the copied files an **independent layer** that does not depend on the previous layers' content. Benefits: if the base image or earlier layers change, the `COPY --link` layers are still reusable from cache, and layers can be pushed/pulled in parallel and re-based onto a new base image without re-copying. Caveat: with `--link`, you can't rely on directories created by earlier instructions in the same stage (it behaves as if copying into an empty filesystem and then merging), and file ownership/permissions of parent directories are not inherited. Great for the final stage of multi-stage builds.
- **`--parents`** (syntax 1.20+): keeps the parent directory structure of the source path. `COPY --parents ./src/a/b.py /app/` places the file at `/app/src/a/b.py` instead of `/app/b.py`. Handy with wildcards: `COPY --parents **/package.json ./`.
- **`--exclude=<pattern>`** (syntax 1.19+): skip files matching a pattern for this `COPY` only (e.g. `COPY --exclude=*.test.js . /app/`).

Other behaviours to remember:

- `COPY src/ dest/` copies the **contents** of `src`, not the folder itself. To copy a folder *as* a folder: `COPY src/ dest/src/`.
- Always end a directory destination with `/` to make it unambiguous.
- Wildcards work in sources: `COPY *.txt /docs/`.
- `COPY . .` copies the whole context (minus `.dockerignore`d files). Do this **late** in the Dockerfile (cache!).

### 5.6 `RUN`: execute commands at build time

```dockerfile
RUN <command>                          # shell form: runs via /bin/sh -c "<command>"
RUN ["executable", "arg1", "arg2"]     # exec form: runs directly, no shell
RUN [--mount=...] [--network=...] [--security=...] <command>
```

`RUN` executes the command in a temporary container, then commits the filesystem changes as a new layer. Used for installing packages, compiling, creating users, etc.

Practical rules:

1. **Chain related commands with `&&`** and clean up in the same instruction (see the `apt-get` example above). Use `\` for line continuation, with `&&` at the start of continuation lines so diffs are readable.
2. **Fail loudly**. In shell form, `a | b` only reports the exit code of `b`, so a failure in `a` is hidden. Use `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` (section 5.15) when you pipe, or avoid pipes.
3. Use `set -eux` style in long scripts (`-e` exit on error, `-u` error on undefined variable, `-x` echo commands).
4. Never run `apt-get upgrade`/`dist-upgrade` casually; it makes builds non-reproducible. Pick a newer base image instead.
5. The `--mount` flag unlocks cache, secret, ssh and bind mounts ([section 9](#9-buildkit-power-features-run---mount-and-friends)). `--network=none` can run a step with no network access (hardening/verifying you aren't downloading things unexpectedly); `--network=host` needs a special entitlement.

### 5.7 `CMD` and `ENTRYPOINT`: what runs when the container starts

These are so important and so confusing that they get their own section: [section 6](#6-cmd-vs-entrypoint-exec-form-vs-shell-form).

### 5.8 `EXPOSE`: document the port

```dockerfile
EXPOSE 8000
EXPOSE 8000/tcp 53/udp
```

Purely **documentation/metadata**: it tells humans and tools which port the app listens on. It does not publish the port to the host. You still need `-p` / `ports:` at run time. `docker run -P` (capital P) publishes all `EXPOSE`d ports to random host ports.

### 5.9 `USER`: switch to a non-root user

```dockerfile
USER 10001:10001
USER app
```

All following `RUN`, `CMD` and `ENTRYPOINT` run as this user, and the container's main process runs as this user. This is one of the most important security settings; see [section 15](#15-running-as-a-non-root-user).

### 5.10 `VOLUME`: declare a mount point

```dockerfile
VOLUME /var/lib/postgresql/data
```

Tells Docker "data at this path should live in a volume". If you run the container without mounting anything there, Docker creates an **anonymous volume** automatically. Gotchas:

- Any changes to that path **in later Dockerfile instructions are discarded** (the build treats it as a volume after the `VOLUME` line).
- It creates anonymous volumes that pile up unnoticed.
- It can't be undone in a child image.

For **your own** application images, it's usually best **not** to use `VOLUME` and instead mount named volumes explicitly at run time or in Compose ([Chapter 5](05-storage-and-networking.md)). You'll see `VOLUME` in database images because they are *guaranteeing* data survives.

### 5.11 `HEALTHCHECK`: tell Docker how to test the app

```dockerfile
HEALTHCHECK [--interval=30s] [--timeout=30s] [--start-period=0s] [--start-interval=5s] [--retries=3] \
  CMD <command>
HEALTHCHECK NONE
```

Docker periodically runs the command **inside the container**. Exit code `0` = healthy, `1` = unhealthy (`2` is reserved; don't use it). Options and defaults (as of the current Dockerfile reference):

| Option | Default | Meaning |
|---|---|---|
| `--interval` | `30s` | Time between checks. |
| `--timeout` | `30s` | If a check takes longer, it counts as failed. |
| `--start-period` | `0s` | Grace time for slow-starting apps: failures during this window **don't** count toward `retries` (a success ends the window early). |
| `--start-interval` | `5s` | Check frequency **during** the start period (needs Docker Engine 25.0+). |
| `--retries` | `3` | Consecutive failures before status becomes `unhealthy`. |

States: `starting` -> `healthy` or `unhealthy`. See it in `docker ps` (the STATUS column shows `(healthy)`) and `docker inspect --format '{{json .State.Health}}' <container>`. Only the last `HEALTHCHECK` in a Dockerfile counts. `HEALTHCHECK NONE` disables a check inherited from the base image.

Important truth: **plain Docker does not restart an unhealthy container.** The status is information that orchestrators (Compose `depends_on: condition: service_healthy`, Swarm, Kubernetes-like systems, load balancers) can act on. Kubernetes ignores the Dockerfile HEALTHCHECK and uses its own probes ([Chapter 11](11-production-and-enterprise.md)).

Slim images don't have `curl`. Use what you have:

```dockerfile
# Python (stdlib only, no curl needed):
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
```

If the URL returns a non-2xx status, `urlopen` raises an exception, Python exits with code 1, and Docker marks the check failed.

### 5.12 `LABEL`: attach metadata

```dockerfile
LABEL org.opencontainers.image.title="my-api" \
      org.opencontainers.image.description="Example FastAPI service" \
      org.opencontainers.image.source="https://github.com/you/my-api" \
      org.opencontainers.image.licenses="MIT"
```

Key-value metadata stored in the image, readable with `docker inspect`. Use the standard **OCI annotation keys** (`org.opencontainers.image.*`) so registries and tools (GitHub Container Registry links the image to your repo via `...image.source`) understand them. `MAINTAINER` is deprecated; use `org.opencontainers.image.authors`.

### 5.13 `STOPSIGNAL`: how to ask the app to stop

```dockerfile
STOPSIGNAL SIGINT
```

`docker stop` sends a signal to PID 1, waits (10 seconds by default), then sends `SIGKILL`. The default signal is `SIGTERM`. Some programs expect something else (nginx's graceful shutdown uses `SIGQUIT`; some Node/Python frameworks handle `SIGINT`). Set `STOPSIGNAL` to match what your app handles gracefully. You can also override at run time with `--stop-signal`, and the grace period with `docker stop -t <seconds>`.

### 5.14 `ONBUILD`: triggers for child images

```dockerfile
ONBUILD COPY . /app
ONBUILD RUN pip install -r /app/requirements.txt
```

Registers an instruction that runs **later, when another image uses this one as its base** (`FROM`). Historically used for language "stack" base images. It is considered a surprising, hard-to-debug feature (a `FROM` line secretly does things!). Know it exists; avoid it in your own images. Cannot chain `ONBUILD ONBUILD` or trigger `FROM`.

### 5.15 `SHELL`: change the default shell

```dockerfile
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
```

Changes the shell used by shell-form `RUN`, `CMD` and `ENTRYPOINT`. The default is `["/bin/sh", "-c"]` on Linux. The common reason to change it: enabling `pipefail`, so `RUN curl ... | tar ...` fails if `curl` fails. (`bash` must exist in the image; `slim` Debian images have it, Alpine does not by default.)

### 5.16 `MAINTAINER` (deprecated)

Do not use; use a `LABEL` instead.

### 5.17 Instruction cheat table

| Instruction | Layer-producing? | Runtime effect? | One-line purpose |
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

Every `RUN`, `CMD` and `ENTRYPOINT` can be written two ways:

```dockerfile
CMD ["python", "app.py"]
CMD python app.py
```

- The first line is **exec form**: a JSON array. Docker runs the program directly.
- The second line is **shell form**: a plain string. Docker runs `/bin/sh -c "python app.py"`.

(These are two alternatives; a real Dockerfile would contain only one `CMD`.)

| | Exec form `["a","b"]` | Shell form `a b` |
|---|---|---|
| How it runs | Docker starts `a` directly as PID 1 | Docker starts `/bin/sh -c "a b"`; `sh` is PID 1 and `a` is its child |
| Variable expansion (`$HOME`) | **No** (no shell to expand it) | Yes |
| Signals (`docker stop`) | Delivered to your program | Delivered to `sh`, which usually does **not** forward them, so your app never shuts down gracefully and gets `SIGKILL`ed after 10 s |
| Requires a shell in the image | No (works in distroless/scratch) | Yes |
| Syntax | Must be valid JSON: **double quotes only** | Free text |

**Recommendation: use exec form for `CMD` and `ENTRYPOINT`.** The shell-form signal problem is the number one cause of "my container takes exactly 10 seconds to stop". (More on this in [section 16](#16-pid-1-signals-tini-and---init).)

If you need variable expansion in exec form, call the shell yourself and use `exec`:

```dockerfile
CMD ["sh", "-c", "exec uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

`exec` replaces the shell process with uvicorn so uvicorn becomes PID 1 and receives signals.

> Common JSON mistake: `CMD ['python', 'app.py']` (single quotes) is not valid JSON, so Docker silently falls back to shell form, treating the whole text as a string. Always use double quotes.

### What are CMD and ENTRYPOINT?

- **`ENTRYPOINT`** = the executable that **always** runs. Think: the "program" of the container.
- **`CMD`** = the **default arguments** (or default command if there is no ENTRYPOINT). Easy for users to override.

Analogy: `ENTRYPOINT` is a coffee machine; `CMD` is the default drink selected on the dial. You can turn the dial (override CMD by passing arguments to `docker run`), but it's still the coffee machine. To replace the machine itself you need `--entrypoint`.

```bash
docker run myimage                  # uses ENTRYPOINT + default CMD
docker run myimage --verbose        # replaces CMD with "--verbose"; ENTRYPOINT unchanged
docker run --entrypoint sh -it myimage   # replaces ENTRYPOINT (e.g. to get a debug shell)
```

### Example: a "CLI tool" image

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

What actually gets executed, depending on which instructions you wrote:

| | **No ENTRYPOINT** | **ENTRYPOINT exec_e p1_e** (shell form) | **ENTRYPOINT ["exec_e", "p1_e"]** (exec form) |
|---|---|---|---|
| **No CMD** | *error: nothing to run* | `/bin/sh -c exec_e p1_e` | `exec_e p1_e` |
| **CMD ["exec_c", "p1_c"]** (exec form) | `exec_c p1_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e exec_c p1_c` |
| **CMD exec_c p1_c** (shell form) | `/bin/sh -c exec_c p1_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e /bin/sh -c exec_c p1_c` |
| **CMD ["p1_c", "p2_c"]** (as default params) | `p1_c p2_c` | `/bin/sh -c exec_e p1_e` (CMD ignored) | `exec_e p1_e p1_c p2_c` |

Read it like this:

1. If `ENTRYPOINT` is **shell form**, it completely ignores `CMD` and any `docker run` arguments, and it cannot receive signals properly. Avoid.
2. If `ENTRYPOINT` is **exec form**, `CMD` (exec form) is **appended** as arguments. This is the pattern for "tool" images.
3. If there is **no** `ENTRYPOINT`, `CMD` is the whole command.
4. If both exist, **always write both in exec form**. Mixing a shell-form CMD with an exec-form ENTRYPOINT produces the strange `exec_e p1_e /bin/sh -c ...` result in the last-but-one row.

### Which should you choose?

| Situation | Use |
|---|---|
| A web service/app | `CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]` (simple; easy to override for debugging) |
| A CLI tool packaged as an image | `ENTRYPOINT ["tool"]` + `CMD ["--help"]` |
| You need setup logic then your app | An `ENTRYPOINT ["/entrypoint.sh"]` script that ends with `exec "$@"` (see below) |

### The entrypoint-script pattern

```bash
#!/bin/sh
# entrypoint.sh
set -e
# ... do setup: wait for a file, run migrations, render a config ...
echo "Starting with args: $@"
exec "$@"        # replace this shell with the real command so it becomes PID 1
```

```dockerfile
COPY --chmod=755 entrypoint.sh /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Explanation: the entrypoint runs first; `"$@"` expands to "all arguments", which is the `CMD`; `exec` hands control over to it. If you forget `exec`, your script's shell stays PID 1 and eats signals.

> **Windows warning:** if `entrypoint.sh` has Windows line endings (CRLF), the kernel looks for the interpreter `/bin/sh\r`, which doesn't exist, and you get a baffling `exec ... no such file or directory`. Configure git (`* text=auto eol=lf` in `.gitattributes`) for shell scripts.

---

## 7. ARG vs ENV

Both set variables, but their **lifetimes** differ:

| | `ARG` | `ENV` |
|---|---|---|
| Available during build | Yes (from the line it's declared) | Yes (from the line it's declared) |
| Available in the running container | **No** | **Yes** |
| Set from the command line | `docker build --build-arg NAME=value` | `docker run -e NAME=value` (overrides at run time) |
| Stored in image metadata/history | Its **name and value can appear in `docker history`** if used in a command | Yes, visible in `docker inspect` |
| Typical use | Versions, feature toggles for the build, platform | Runtime config defaults (`PORT`, `APP_ENV`) |
| Safe for secrets? | **No** | **No** |

If an `ARG` and an `ENV` have the same name, the `ENV` wins.

Pattern to turn a build argument into a runtime variable:

```dockerfile
ARG APP_VERSION=dev
ENV APP_VERSION=${APP_VERSION}
```

Scope rules:

```dockerfile
# Global ARG: usable ONLY in FROM lines
ARG BASE=3.13
FROM python:${BASE}-slim AS build
# Re-declare it to use it inside this stage
ARG BASE
RUN echo "building on python ${BASE}"
```

Neither `ARG` nor `ENV` is a place for secrets. Secrets get their own section ([17](#17-build-secrets-never-bake-secrets-into-layers)).

---

## 8. Heredocs

With the modern Dockerfile syntax (1.4+; the `# syntax=docker/dockerfile:1` line guarantees it) you can write multi-line scripts and create small files inline using **here-documents** (a shell concept: text between a start marker and an end marker is fed as input).

### Multi-line `RUN`

Instead of this `&&`/`\` chain:

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

You can write:

```dockerfile
RUN <<EOF
set -eux
apt-get update
apt-get install -y --no-install-recommends curl ca-certificates
rm -rf /var/lib/apt/lists/*
EOF
```

Explanation: `<<EOF` starts the heredoc; everything until a line that is exactly `EOF` is a script run by the default shell as **one layer**. `set -e` makes the script stop at the first failure (`-u` and `-x` as described earlier). It reads like a normal shell script, no trailing backslashes.

You can pick another interpreter with a shebang:

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

This creates `/etc/app/config.ini` with that content, with no separate file in your repo. Use `<<"EOF"` (quoted) if you want `$VARIABLES` in the body kept literally; unquoted heredocs expand build-time variables.

---

## 9. BuildKit power features: `RUN --mount` and friends

BuildKit lets a `RUN` step attach special temporary mounts that **never end up in the final image**. There are five types you should know.

### 9.1 `--mount=type=cache`: persistent caches for package managers

Problem: every time `requirements.txt` changes, `pip install` re-downloads *all* packages, even the unchanged ones, because the layer is rebuilt from scratch. A **cache mount** is a directory that persists **between builds** (kept by BuildKit, outside any image layer) so downloaded packages can be reused.

```dockerfile
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

- `type=cache`: choose the cache mount type.
- `target=/root/.cache/pip`: where it appears inside the build container (pip's download cache).
- Extra options: `id=` (name a cache so it can be shared between different Dockerfiles), `sharing=shared|private|locked` (what happens when two builds use it at once; `locked` makes the second wait), `uid=`/`gid=`/`mode=` (ownership of the mount), `from=`/`source=` (initial content from another stage).

Because the cache is not part of the layer, your image stays small, and an `--no-cache-dir` flag is not needed (and should be omitted, or it defeats the cache).

Other common targets:

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

**apt requires one extra trick** because Debian images ship a config that deletes downloaded `.deb` files automatically:

```dockerfile
RUN rm -f /etc/apt/apt.conf.d/docker-clean \
 && echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get install -y --no-install-recommends libpq5
```

`sharing=locked` is used because apt cannot run twice concurrently on the same directories. (This is an optimization for *developer* build speed; for CI with a cold runner you need cache export, see [section 20](#20-exporting-the-build-cache-for-ci).)

### 9.2 `--mount=type=secret`: use a secret without storing it

Covered in depth in [section 17](#17-build-secrets-never-bake-secrets-into-layers). Preview:

```dockerfile
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```

The secret appears as a file at `/run/secrets/<id>` **only during that RUN** and leaves no trace in any layer.

### 9.3 `--mount=type=ssh`: use your SSH keys without copying them

```dockerfile
RUN --mount=type=ssh \
    mkdir -p ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts \
 && git clone git@github.com:yourorg/private-lib.git /src/private-lib
```

```bash
docker build --ssh default .
```

- `--ssh default` forwards your local **SSH agent** (the background program that holds your unlocked keys) into the build. The private key file itself is never copied into the image. Make sure the agent is running (`ssh-add -l` should list a key).
- `ssh-keyscan` adds the host's public key to `known_hosts` so SSH doesn't prompt "are you sure you want to continue connecting?" (an interactive question a build can't answer). For stricter security, pin the host key fingerprint instead of scanning blindly.

### 9.4 `--mount=type=bind`: read files from the context (or another stage) without copying

```dockerfile
RUN --mount=type=bind,source=requirements.txt,target=/tmp/requirements.txt \
    pip install -r /tmp/requirements.txt
```

A bind mount makes a file or folder from the build context (or another stage via `from=`) visible during this one `RUN`, read-only by default. Nothing is copied into a layer. It's a neat way to use `requirements.txt` for installing without a separate `COPY` layer, and the cache is still correctly invalidated when the file changes. Add `rw` to allow writes (changes are discarded after the step).

### 9.5 `--mount=type=tmpfs`

```dockerfile
RUN --mount=type=tmpfs,target=/tmp,size=512m ./build-script.sh
```

A RAM-backed scratch directory that vanishes after the step. Useful for large temporary build files that you don't want in a layer.

### 9.6 Other `RUN` flags you may meet

- `--network=none|default|host`: control network access for that step (`host` needs the `network.host` entitlement).
- `--security=sandbox|insecure`: `insecure` (needs the `security.insecure` entitlement, enabled at build time) is for special cases like Docker-in-Docker builds.
- `--device=...`: request CDI devices (e.g. GPUs) during build; needs BuildKit 0.20+ and the `device` entitlement.

---

## 10. Ordering instructions for cache efficiency (before and after)

Let's make section 3's advice concrete with a FastAPI project.

Project layout:

```
myapi/
  Dockerfile
  requirements.txt      (fastapi, uvicorn, ... ~30 packages)
  app/
    main.py
    ...
```

### BEFORE: the naive Dockerfile

```dockerfile
FROM python:3.13-slim
WORKDIR /app
# Copies EVERYTHING, including source code
COPY . .
# This step depends on the line above!
RUN pip install -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

(A Dockerfile gotcha worth knowing: comments must be on their **own line**. A `#` in the middle of an instruction line is *not* a comment, it becomes part of the arguments. That is why the comments above sit on separate lines.)

What happens when you change one line in `app/main.py`:

```
 Step 3  COPY . .                      MISS  (main.py content changed)
 Step 4  RUN pip install ...           REBUILT: re-downloads and installs ALL packages (~60 s)
```

Every code edit costs a full dependency install. Over a day of development that is dozens of minutes lost.

### AFTER: dependencies first, code last

```dockerfile
FROM python:3.13-slim
WORKDIR /app

# 1) Copy ONLY the dependency list. Its content changes rarely.
COPY requirements.txt .

# 2) Install dependencies. Cached as long as requirements.txt is unchanged.
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# 3) Copy the source code LAST. It changes constantly.
COPY app/ ./app/

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

When you edit `app/main.py`:

```
 Step 1  FROM python:3.13-slim          CACHED
 Step 2  WORKDIR /app                   CACHED
 Step 3  COPY requirements.txt .        CACHED  (file unchanged)
 Step 4  RUN pip install ...            CACHED  <- the expensive step is skipped
 Step 5  COPY app/ ./app/               MISS    (your edit)
 Step 6  CMD ...                        REBUILT (metadata, instant)
```

Typical timing (illustrative numbers; yours vary with network and package set):

| Scenario | Before | After |
|---|---|---|
| Cold build, nothing cached | ~60 s | ~60 s |
| Edit one source file, rebuild | ~60 s | ~1-2 s |
| Add one new dependency, rebuild | ~60 s | ~10 s (with cache mount; unchanged wheels reused) |

**The pattern generalizes to every language.** Copy the dependency manifest(s) (`package.json` + lockfile, `go.mod` + `go.sum`, `pom.xml`, `pyproject.toml` + `uv.lock`) first, install, **then** copy source.

### Try it yourself

1. Create the BEFORE Dockerfile and a `requirements.txt` containing `fastapi` and `uvicorn[standard]`.
2. Run `docker build -t order-test .` and note the time.
3. Add a comment to a Python file and rebuild: notice `pip install` runs again.
4. Switch to the AFTER Dockerfile, build once, edit a Python file, rebuild: notice `CACHED` next to the pip step.

(Clean up afterwards with `docker rmi order-test`, only on images you created.)

---

## 11. Multi-stage builds

### The problem

To *build* software you need compilers, headers, package managers, test tools. To *run* it you usually need only the result. If you build in a single stage, your production image carries all that build baggage: bigger (slower to pull, more disk), and more attack surface (more software to have vulnerabilities, and tools an attacker can use).

### The solution

A **multi-stage build** uses several `FROM` lines in one Dockerfile. Each is a separate stage with its own filesystem. You build in a "fat" stage and then **copy only the artifacts** into a small clean final stage. Everything else is thrown away.

```
 +------------------------------+        +-----------------------------+
 | Stage 1: "builder"           |        | Stage 2: "runtime" (FINAL)  |
 | python:3.13 (big)            |        | python:3.13-slim (small)    |
 | + compilers, headers         | COPY   |                             |
 | + pip install -> /opt/venv   | -----> | /opt/venv  (just the venv)  |
 | + tests, caches ...          | --from | + your app code             |
 +------------------------------+        +-----------------------------+
      (discarded after build)                  (this is the image you ship)
```

Useful commands/flags:

- `FROM ... AS name`: name a stage.
- `COPY --from=name /path /dest`: pull files from a stage.
- `docker build --target name .`: stop at a specific stage (e.g. build only the `test` stage, or the `dev` stage). The default is the last stage.
- BuildKit only builds stages that the target actually depends on, and builds independent stages **in parallel**.

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

Explanation of the new ideas:

- `python -m venv /opt/venv`: creates a **virtual environment** (a self-contained folder of installed Python packages) at a fixed path. Because the whole thing is in one folder, `COPY --from=builder /opt/venv /opt/venv` moves all dependencies in one instruction.
- `ENV PATH="/opt/venv/bin:$PATH"`: puts the venv's `bin` first on the search path, which is the same as "activating" it; `pip` and `python` now refer to the venv's copies.
- `PYTHONUNBUFFERED=1`: makes Python print logs immediately instead of buffering them, so `docker logs` shows output in real time.
- The venv works in the second stage because both stages use the **same Python version, same OS, same path**. That is the crucial requirement for this pattern.
- When is this worth it? When you need compilers to build wheels (e.g., `psycopg2`, `cryptography` from source) in the builder stage but not at runtime. The builder can be the full `python:3.13` or `slim` plus `apt-get install build-essential`.
- Modern alternative: [uv](https://docs.astral.sh/uv/) (a fast Python package manager). Copy its binary with `COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv` and use `uv sync --frozen` with a cache mount `/root/.cache/uv`. The pattern (builder installs into a venv, runtime copies it) is identical. Pin the uv image tag/digest in real projects.

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

Explanation:

- `npm ci` ("clean install") installs exactly what the lockfile says and fails if `package.json` and the lockfile disagree. Always use it in builds instead of `npm install`.
- Three stages: `deps` (install), `build` (compile the React/Vue/etc. app into static files), `runtime` (nginx serving those files). The final image contains **no Node.js at all**, only nginx and a few MB of HTML/JS/CSS.
- Pick tags that exist and are supported at the time you read this; Node LTS versions rotate, so check [nodejs.org](https://nodejs.org) and pin accordingly.

For a **Node server** (Express, NestJS, Next.js standalone), the final stage would be `node:24-slim` with `npm ci --omit=dev` (production dependencies only) plus the compiled output, running as the built-in `node` user (`USER node`) with `CMD ["node", "dist/server.js"]`.

### 11.3 Go (the classic: compile, then ship a ~10 MB image)

Go compiles to a single, self-contained binary, which makes it ideal for tiny images.

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

Explanation:

- `CGO_ENABLED=0`: disable cgo (Go's bridge to C libraries) so the binary is **statically linked**, with no dependency on any shared library like glibc. A static binary runs on `scratch` or distroless/static.
- `-trimpath`: removes local file-system paths from the binary (smaller, more reproducible, doesn't leak your directory layout).
- `-ldflags="-s -w"`: strips the symbol table and DWARF debug info (`-s` and `-w`), shrinking the binary by roughly a quarter to a third.
- `gcr.io/distroless/static-debian12:nonroot`: Google's distroless image, which contains only CA certificates, timezone data, a minimal `/etc/passwd`, and a pre-made **non-root user (UID 65532)**. About 2 MB. No shell, no package manager. Check the distroless project page for the current Debian-version tags.
- The final image is typically 10-20 MB depending on your app.
- Alternative: `FROM scratch` (a totally empty image), but then you must copy CA certificates yourself if the program makes HTTPS calls (`COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`), and set `USER 65532:65532` yourself.

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

Explanation:

- **JDK** (Java Development Kit) includes the compiler; **JRE** (Java Runtime Environment) only runs programs. The JRE image is considerably smaller.
- `dependency:go-offline` downloads all Maven dependencies in a cacheable step before the source is copied (same ordering idea as everywhere).
- `-DskipTests`: skip tests in the image build (run them in CI as a separate step or a `test` stage).
- `-XX:MaxRAMPercentage=75.0`: tells the JVM to size its heap as a percentage of the **container's** memory limit instead of a fixed number. Modern JVMs are container-aware, but this option makes the heap target explicit and leaves room for non-heap memory.
- Advanced: `jlink` can build a trimmed custom runtime containing only the Java modules your app uses.
- If your build produces more than one jar (e.g. a `-plain.jar`), make the `COPY` pattern specific, otherwise the glob may match multiple files and fail.

### 11.5 Extra multi-stage tricks

**A test stage** that runs only when you ask:

```dockerfile
FROM builder AS test
COPY tests/ ./tests/
RUN pytest -q
```

`docker build --target test .` runs the tests; a normal `docker build .` (targeting the last stage) skips them entirely.

**Copying from any image** without a stage (grab one binary from a published image):

```dockerfile
COPY --from=busybox:1.37 /bin/wget /usr/local/bin/wget
```

**Named build contexts** (`docker build --build-context tools=./tools .` then `COPY --from=tools ...`) let you pull in a second directory, or even `docker-image://...` / `https://...git` sources, in the same build.

---

## 12. Choosing a base image

The `FROM` line is the single biggest decision for size, security, compatibility, and build time.

### The options

| Base type | Example | Approx. size* | Pros | Cons |
|---|---|---|---|---|
| **Full Debian-based language image** | `python:3.13` | ~1 GB | Has compilers and many dev libraries; almost everything builds | Huge, many packages = many CVEs |
| **Slim (Debian)** | `python:3.13-slim` | ~120-150 MB | Good balance; uses **glibc** (the standard C library), so binary packages (wheels) just work | Fewer tools; no compilers (add `build-essential` in a builder stage) |
| **Alpine** | `python:3.13-alpine` | ~50-60 MB | Tiny, own package manager `apk` | Uses **musl** libc, not glibc: compatibility pitfalls (below) |
| **Distroless** | `gcr.io/distroless/python3-debian12` or `.../static-debian12` | ~2 MB (static) to ~50 MB (python) | No shell, no package manager: tiny attack surface | Hard to debug; no `sh`, so no shell-form commands or `docker exec ... sh` |
| **scratch** | `FROM scratch` | 0 MB | Empty. Perfect for static binaries | Literally nothing: no certs, no users, no timezone, no shell |
| **OS image** | `debian:stable-slim`, `ubuntu:24.04` | ~75-80 MB | Familiar; you install what you need | You maintain the runtime install yourself |

*Sizes are rough, uncompressed, and change with each release. Check with `docker images` after pulling.

### Why Alpine is tempting and often a trap (for Python especially)

Alpine uses **musl** instead of **glibc**. The consequences:

1. **Python wheels**: Most pre-compiled Python packages ("manylinux wheels") are built for glibc. Newer packages also publish "musllinux" wheels, but many (notably big scientific/ML packages, and various less-popular ones) don't. When no musl wheel exists, `pip` compiles from source, which needs a compiler and headers, making the build slow (minutes instead of seconds) and the image large again, defeating the point. Frameworks like PyTorch, TensorFlow and many CUDA-related packages are not targeted at Alpine ([Chapter 8](08-docker-for-ai-and-ml.md) goes deep on this).
2. **Subtle behavioural differences**: DNS resolution quirks have existed historically, differing `locale` behaviour, a different `iconv`, thread stack sizes, and occasional performance differences in memory allocation. Bugs that only show up in production on Alpine are hard to diagnose.
3. **Tools**: `sh` is BusyBox `ash`, not bash; common utilities are slimmed-down variants with different flags.
4. **No `apt`**; packages come from `apk`, with different names.

Alpine is *fine* for Go (static binaries), simple Node apps, nginx, Redis and other things that ship an official Alpine variant and don't need native extensions. For Python and anything with native dependencies, **prefer `-slim`**.

### Distroless: when and how

Distroless images (from Google) include only what a language runtime needs, no shell or package manager. Benefits: fewer CVEs and nothing for an attacker to use post-exploit. Costs: debugging. Remedies: use the `:debug` tag variants (which add a BusyBox shell) while troubleshooting, or `docker debug` / ephemeral debug containers ([Chapter 12](12-troubleshooting-and-cheatsheet.md)). Because there is no shell, **every** `CMD`/`ENTRYPOINT`/`HEALTHCHECK` must be in exec form, and `RUN` steps can't happen in the final stage; do them in the builder.

### Decision guide

```
 Compiled language that produces a static binary (Go, Rust)?
        -> distroless/static (or scratch)
 Python / Node / Ruby app with normal dependencies?
        -> -slim  (python:3.13-slim, node:24-slim)
 Need ML/GPU libs?
        -> an NVIDIA CUDA or framework-provided image (Debian/Ubuntu-based), see Chapter 8
 Java?
        -> eclipse-temurin:<ver>-jre (or a distroless java image)
 Known-tiny, official Alpine variant, no native deps?
        -> alpine is fine
 Not sure?
        -> -slim.
```

---

## 13. Reproducibility: tags, digests, pinning

**Reproducible** means: building the same Dockerfile today and in six months yields functionally the same image. Many things can break that:

1. **Moving tags.** `python:3.13-slim` is a *tag*, a mutable pointer. Today it points to Python 3.13.x on Debian version A; after the next patch release or Debian point release, it points to a different image. `latest` is the worst: it can jump major versions.
2. **Unpinned packages.** `apt-get install curl` installs whatever version is current. `pip install fastapi` likewise.
3. **Network sources** that change underneath you.

### Levels of pinning

| Level | Example | Trade-off |
|---|---|---|
| Floating | `python:slim`, `python:latest` | Never in production. |
| Major.minor | `python:3.13-slim` | Gets security patches automatically; occasionally a patch release breaks something. Common choice. |
| Exact tag | `python:3.13.7-slim-bookworm` | More stable; you must bump it manually (Dependabot/Renovate automate this). |
| **Digest** | `python:3.13-slim@sha256:<64 hex characters>` | Immutable. The exact bytes. Best for reproducibility and supply-chain integrity. |

A **digest** is the SHA-256 hash of the image manifest (the file describing the image's layers). Content-addressed, so it cannot change without the hash changing. Get it:

```bash
docker buildx imagetools inspect python:3.13-slim
```

Example output (abbreviated):

```
Name:      docker.io/library/python:3.13-slim
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:0123abcd...ef     <- this is the multi-arch index digest
Manifests:
  Name:      docker.io/library/python:3.13-slim@sha256:aaaa...
  Platform:  linux/amd64
  ...
```

Then:

```dockerfile
FROM python:3.13-slim@sha256:0123abcd...ef
```

When both a tag and a digest are present, Docker uses the **digest**; the tag is then just a human-readable comment. Pin the **index digest** (the one at the top) so the pin works on both amd64 and arm64 machines.

The catch: a pinned digest never receives security updates by itself. You need automation: **Dependabot** or **Renovate** will open pull requests when the tag moves to a new digest. Pinning without automated updates equals slowly accumulating vulnerabilities.

Pin your application dependencies, too: `requirements.txt` with exact versions (ideally with hashes: `pip install --require-hashes`, or a lockfile from `uv`/`pip-tools`/`poetry`), `package-lock.json` with `npm ci`, `go.sum`, Maven/Gradle lockfiles.

---

## 14. Making images small

Smaller images pull faster (CI, deploys, autoscaling), cost less to store, and contain fewer vulnerabilities. Ordered by impact:

1. **Smaller base** (section 12): `slim`, distroless or scratch rather than the full image.
2. **Multi-stage builds** (section 11): leave compilers and build caches behind.
3. **`.dockerignore`** (section 4): don't copy junk in with `COPY . .`.
4. **Clean up in the same layer** that created the mess:
   - apt: `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*`
   - pip: use a cache mount (preferred) or `--no-cache-dir`
   - npm: `npm ci --omit=dev`, `npm cache clean --force`
5. **Install only runtime dependencies** in the final stage (`--omit=dev`, separate `requirements.txt` vs `requirements-dev.txt`).
6. **Avoid `chown -R` / `chmod -R` in a separate `RUN`**: that duplicates every file into a new layer. Use `COPY --chown`/`--chmod` instead.
7. **Strip binaries** (`-ldflags="-s -w"` in Go; `strip` for C builds).
8. **Don't ship build tools or tests** to production.
9. **ML specifics**: choose CPU-only wheels when you have no GPU (PyTorch's CPU wheel is hundreds of MB smaller than the CUDA one); don't bake multi-GB model weights into the image unless you must ([Chapter 8](08-docker-for-ai-and-ml.md)).

### Measuring

```bash
docker images my-api
docker history my-api:1.0
docker history --no-trunc --human my-api:1.0
```

Flags: `docker history` lists each layer, the instruction that created it, and its size; `--no-trunc` shows full (non-truncated) commands; `--human` (default true) prints sizes like `54MB`.

Example output:

```
IMAGE          CREATED         CREATED BY                                      SIZE
a1b2c3d4e5f6   2 minutes ago   CMD ["uvicorn" "app.main:app" "--host" "0...."   0B
<missing>      2 minutes ago   COPY app/ ./app/ # buildkit                      18.4kB
<missing>      3 minutes ago   RUN /bin/sh -c pip install -r requirements...    94.7MB
<missing>      3 minutes ago   COPY requirements.txt . # buildkit               89B
<missing>      3 minutes ago   WORKDIR /app                                     0B
<missing>      2 weeks ago     CMD ["python3"]                                  0B
...
```

(`<missing>` simply means the layer ID isn't stored locally as a separate image; it's normal for BuildKit-built images and is not an error.)

Free tools to explore layers: **dive** (interactive layer explorer, `docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock wagoodman/dive <image>`; mounting the Docker socket gives the container control of your Docker, so only run tools you trust) and **Docker Scout** (`docker scout quickview <image>`, size/CVE summaries).

---

## 15. Running as a non-root user

By default, the process in a container runs as **root** (UID 0) *inside* the container. Container isolation limits the damage, but a root process inside a container that gets compromised, combined with a kernel or misconfiguration bug ("container escape"), becomes root on the host. Running as an unprivileged user is cheap defense in depth, and many platforms (Kubernetes with `runAsNonRoot`, OpenShift, security scanners) require it. Details in [Chapter 10](10-security.md).

### How to do it (Debian/Ubuntu-based images)

```dockerfile
RUN groupadd --system --gid 10001 app \
 && useradd  --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app app/ ./app/

USER 10001:10001
```

Explanation:

- `groupadd --system --gid 10001 app`: creates a **system group** named `app` with group ID 10001. System accounts are meant for services, not people (no login, low-numbered ID range by default, but an explicit high ID avoids collisions).
- `useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app`: creates the user `app` with UID 10001, in group `app`, no home directory, and a shell that refuses logins.
- A high numeric UID (10001) avoids clashing with host UIDs and makes policy checks easy.
- `COPY --chown=app:app`: files are owned by the app user (instead of root), so the app can read them (and write, if needed).
- `USER 10001:10001`: use the **numeric** UID:GID. Kubernetes' `runAsNonRoot` check can only verify a numeric UID; a name like `app` can't be verified without looking into `/etc/passwd`.

**Alpine** uses `addgroup -S -g 10001 app && adduser -S -u 10001 -G app app` (BusyBox syntax). **Distroless** provides a `nonroot` tag (UID 65532). Many official images already include a user: `node` images have user `node` (UID 1000); `nginx` has `nginx`; the `postgres`/`redis` images drop privileges at start in their own entrypoints.

### Typical problems and fixes

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` writing to `/app/data` | Directory owned by root | `COPY --chown`, or `RUN mkdir -p /app/data && chown app:app /app/data` before `USER` |
| `bind: permission denied` on port 80 | Non-root users cannot bind ports below 1024 on many systems | Listen on a high port (8000/8080) and map it with `-p 80:8000` |
| Can't write to a mounted volume | Volume content owned by another UID | Match UIDs, see [Chapter 5](05-storage-and-networking.md#a7-permissions-and-uid-problems) |
| `pip install` fails after `USER` | Trying to write to system site-packages | Install as root in the builder stage; switch to `USER` only at the end |
| Package manager commands fail | They need root | Put `USER` **after** all root-requiring `RUN` steps |

Order matters: **do all root-needing setup first, put `USER` near the end**. And a corollary for `docker exec`: it uses the image's `USER`, so you'll have to add `-u 0` to get a root shell when debugging.

---

## 16. PID 1, signals, tini and `--init`

### What is PID 1 and why does it matter?

On Linux every process has a process ID (PID). Inside a container, the first process started (your `CMD`/`ENTRYPOINT`) gets **PID 1** in the container's own PID namespace. PID 1 is special:

1. **Signal handling.** The kernel doesn't apply the *default* actions to PID 1 for signals like `SIGTERM` and `SIGINT`: if the process hasn't installed handlers for them, they're **ignored**. Many apps never installed a handler because as a normal process they didn't need to (the default action was "terminate"). So `docker stop` sends `SIGTERM`, PID 1 ignores it, Docker waits 10 seconds, then sends `SIGKILL`. Result: slow stops and **no graceful shutdown** (in-flight requests dropped, database connections not closed, files not flushed).
2. **Zombie reaping.** When a child process finishes, it stays as a "zombie" entry until its parent reads its exit status. Orphaned processes get re-parented to PID 1, and PID 1 is expected to reap them. If your app doesn't (most don't), zombies accumulate. Mostly harmless in small numbers; a problem for apps that spawn many subprocesses (headless browsers, build tools, shells).

### Symptoms you've hit this problem

- `docker stop my-container` takes exactly 10 seconds.
- Ctrl+C in `docker run` (without `-it`) does nothing.
- Shell-form `CMD` (the shell is PID 1 and doesn't forward signals).
- A wrapper script runs your app without `exec`.

### Fixes

1. **Use exec form** (`CMD ["python", "app.py"]`) and `exec` in scripts, so your app is PID 1.
2. **Make sure your app handles `SIGTERM`.** Uvicorn, Gunicorn, Node's `http` (with a handler: `process.on('SIGTERM', ...)`), Go (`signal.Notify`), and Java (shutdown hooks; Spring Boot does this) do it, if configured properly.
3. **Add a tiny init process** that runs as PID 1, forwards signals to your app, and reaps zombies. The standard one is **tini**.

**Option A: `docker run --init`** (Docker injects tini for you):

```bash
docker run --init --rm -p 8000:8000 my-api:1.0
```

`--init` runs the container with Docker's bundled init (`docker-init`, a build of tini) as PID 1, and your command as its child. In Compose: `init: true` under the service.

**Option B: bake tini into the image:**

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends tini && rm -rf /var/lib/apt/lists/*
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
```

Explanation: `tini --` means "run everything after `--` as the child". On Alpine, `apk add --no-cache tini` installs it at `/sbin/tini`. Baking it in is better when you can't control how the image will be started (e.g., a platform that doesn't pass `--init`). Kubernetes does not offer a `--init` flag, so images there either include an init or rely on the app handling signals properly.

**Tuning the stop behaviour:**

```bash
docker stop -t 30 my-container          # allow 30 s of grace before SIGKILL
docker run --stop-signal SIGINT ...      # or STOPSIGNAL in the Dockerfile
```

### Test it

```bash
docker run -d --name sigtest my-api:1.0
time docker stop sigtest
```

A well-behaved container stops in under ~1-2 seconds. Anything near 10 seconds means signals aren't reaching your app. (Then clean up your own test container: `docker rm sigtest`.)

---

## 17. Build secrets: never bake secrets into layers

### The rule

> **Anything you put in a layer is permanent and readable by anyone who can pull the image.**

This includes `ENV`, `ARG` used in a command, `COPY`'d `.env` files and keys, and files you `rm` in a later layer (the earlier layer still has them). Container registries are not secret stores. Images get pushed to Docker Hub accidentally, shared with contractors, and cached on CI machines.

### Proof: how an `ARG` secret leaks

A common *wrong* approach:

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

The "secret" is right there in plain text in the image's history. Docker may also print a warning during build (`SecretsUsedInArgOrEnv`) when it spots names like `TOKEN`, `PASSWORD` or `SECRET` in `ARG`/`ENV` (a build check available in recent Dockerfile frontends). The same thing leaks via `ENV`, and via `docker inspect`.

The "delete it afterwards" approach also fails:

```dockerfile
COPY id_rsa /root/.ssh/id_rsa
RUN git clone git@github.com:org/private.git
# Too late: the COPY layer above still contains the key
RUN rm /root/.ssh/id_rsa
```

Anyone can extract it: `docker save leaky -o leaky.tar` then open the layer tarballs.

### The right way: secret mounts

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
COPY requirements.txt .
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```

Build with:

```bash
# from a file
docker build --secret id=pip_token,src=./pip_token.txt -t safe .

# or from an environment variable on your machine
export PIP_TOKEN=...   # your real token, set in your shell, never committed
docker build --secret id=pip_token,env=PIP_TOKEN -t safe .
```

Explanation:

- `--secret id=pip_token,src=./pip_token.txt`: gives the build a secret named `pip_token` read from a local file.
- `--secret id=pip_token,env=PIP_TOKEN`: reads it from an environment variable.
- `--mount=type=secret,id=pip_token`: inside the `RUN`, the secret is available as a **file** at `/run/secrets/pip_token` (change with `target=`), a tmpfs-style mount that exists only for that step.
- In Dockerfile syntax 1.10+ you can also expose it as an environment variable **for that step only**: `RUN --mount=type=secret,id=pip_token,env=PIP_TOKEN pip install ...`.
- Nothing is written to any layer, `docker history` shows only `RUN --mount=type=secret,id=pip_token ...`, and the secret isn't in the cache key.

Verify with `docker history --no-trunc safe`: the token text is nowhere. (In Compose, use `build.secrets`, see [Chapter 6](06-docker-compose.md#secrets-and-configs).)

### Multi-stage as a second line of defence

If a secret really must be present in an early stage (for example, a private package registry config), do the secret-dependent work in a **builder stage** and copy only the results forward. The final image never contains that stage's layers. But a secret mount is still better, because builder layers can still be pushed or cached by CI.

### Runtime secrets are a different topic

Secrets the application needs **at runtime** (database passwords, API keys) must not be in the image either. Provide them when the container starts: environment variables (acceptable for simple cases), mounted secret files (better), or a secrets manager. See [Chapter 6](06-docker-compose.md#secrets-and-configs) and [Chapter 10](10-security.md).

### If you leak a secret

Treat it as compromised: **rotate (revoke and re-issue) it immediately.** Deleting the image tag from the registry does not un-leak it (someone may have pulled it, and Git history or CI logs may hold it too).

---

## 18. BuildKit and buildx

### BuildKit

**BuildKit** is Docker's modern build engine. It has been the default in Docker Desktop and in Docker Engine since version 23.0, replacing the old "classic builder". What it gives you:

- Parallel execution of independent stages
- A smarter, more precise cache (and cache import/export)
- Only the needed parts of the context are transferred
- `RUN --mount` (cache/secret/ssh/bind/tmpfs)
- Heredocs, `COPY --link`, build checks (linting), and multi-platform builds
- Attestations: SBOMs (a Software Bill of Materials, a list of everything inside the image) and provenance (a signed record of how/where it was built)

### buildx

**`docker buildx`** is the CLI plugin that exposes the full BuildKit feature set. In current Docker, `docker build` is effectively `docker buildx build` with the default builder, so the commands share almost all flags. Use `buildx` explicitly when you need multiple builders, multi-platform, exports, or cache backends.

```bash
docker buildx version          # check it's installed, e.g. "github.com/docker/buildx v0.38.1 ..."
docker buildx ls               # list builder instances
docker buildx inspect --bootstrap
```

On a recent Docker Desktop you'll see something like:

```
NAME/NODE           DRIVER/ENDPOINT   STATUS    BUILDKIT   PLATFORMS
default             docker
 \_ default         \_ default        running   v0.34.0    linux/amd64 (+3), linux/arm64, linux/arm (+2), ...
desktop-linux*      docker
 \_ desktop-linux   \_ desktop-linux  running   v0.34.0    linux/amd64 (+3), linux/arm64, linux/arm (+2), ...
```

(The `*` marks the currently selected builder.)

### Drivers

A buildx **builder** runs with a **driver**:

| Driver | What it is | Use for |
|---|---|---|
| `docker` | BuildKit built into the Docker daemon | Everyday local builds. Limited: can't export some cache types or build multi-platform images unless the containerd image store is enabled. |
| `docker-container` | BuildKit running in its own container | Multi-platform builds, full cache export, CI. Most flexible. |
| `kubernetes` | BuildKit pods in a cluster | Scalable shared builders. |
| `remote` | Connect to an existing BuildKit daemon | Dedicated build servers. |

Create a container-driver builder:

```bash
docker buildx create --name mybuilder --driver docker-container --use
docker buildx inspect --bootstrap
```

- `create --name mybuilder`: name the new builder.
- `--driver docker-container`: run BuildKit in a container (Docker pulls the `moby/buildkit` image).
- `--use`: make it the current builder.
- `inspect --bootstrap`: start it now and print details.

Switch back with `docker buildx use default` and remove with `docker buildx rm mybuilder` (only builders you created).

### Output destinations (`--load`, `--push`, `--output`)

With the container driver, the build result stays inside the builder unless you tell it where to go:

| Flag | Meaning |
|---|---|
| `--load` | Load the single-platform image into your local Docker image store (`docker images`). |
| `--push` | Push to the registry named in the tag. |
| `--output type=local,dest=./out` | Write the **filesystem** of the image to a local folder (handy for extracting build artifacts). |
| `--output type=oci,dest=image.tar` | Write an OCI-format tarball. |

### Build checks (linting) and `--print`

```bash
docker build --check .          # run Dockerfile checks without building
docker buildx build --call=check .    # equivalent call syntax in newer buildx
```

Build checks (Dockerfile frontend 1.8+) warn about things like undefined variables, `FROM` with `latest`, `JSONArgsRecommended` (use exec form), `SecretsUsedInArgOrEnv`, etc. You can silence or promote them with a `# check=skip=RuleName;error=true` directive at the top of the file.

### Attestations: SBOM and provenance

```bash
docker buildx build --sbom=true --provenance=mode=max -t registry.example.com/me/app:1.0 --push .
```

- `--sbom=true`: generate an SBOM and attach it to the image in the registry.
- `--provenance=mode=max`: attach detailed build provenance (mode `max` includes more detail, like build arguments, so don't put secrets in build args!). BuildKit may add minimal provenance by default in some setups.

These are increasingly required in enterprise supply-chain policies ([Chapter 10](10-security.md), [Chapter 11](11-production-and-enterprise.md)).

---

## 19. Multi-platform builds (amd64 and arm64)

### Why you need it

Machines have different CPU architectures:

- **amd64** (also called x86_64): most servers, Intel/AMD PCs.
- **arm64** (also called aarch64): Apple Silicon Macs (M-series), AWS Graviton, Raspberry Pi 4/5, many phones and edge devices.

A container image contains compiled binaries for one architecture. An amd64 image on an arm64 Mac runs only via emulation (slow, sometimes broken), and vice versa. Modern teams build **multi-platform images**: one tag (`my-api:1.0`) that is actually an **index** (a "manifest list") pointing to a separate image per platform; the Docker client automatically pulls the right one.

```
 my-api:1.0  (image index)
    |-- linux/amd64  -> layers A
    |-- linux/arm64  -> layers B
```

### Building

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t youruser/my-api:1.0 --push .
```

- `--platform linux/amd64,linux/arm64`: build for both targets.
- `-t youruser/my-api:1.0`: the tag, including your registry namespace.
- `--push`: multi-platform results can't be loaded into the classic local store, so they are pushed straight to the registry. (If Docker Desktop uses the **containerd image store**, which is the default on new installs in recent versions, `--load` can hold multi-platform images locally. Check `docker info` for the "Storage Driver": `overlayfs` indicates containerd store.)

Inspect the result: `docker buildx imagetools inspect youruser/my-api:1.0`.

### How is the foreign platform built?

Two mechanisms:

1. **Emulation via QEMU** (a CPU emulator): the builder runs arm64 `RUN` commands on your amd64 CPU through emulation. Zero Dockerfile changes, but 5-20x slower for compile-heavy steps. Docker Desktop has QEMU/binfmt set up. On a plain Linux server install it once: `docker run --privileged --rm tonistiigi/binfmt --install all` (`--privileged` gives that container broad host access; it registers emulators with the kernel; run only trusted images).
2. **Native builders / cross-compilation**: add a real arm64 node to the builder, or cross-compile (below).

### Cross-compilation pattern (fast, no emulation)

BuildKit sets automatic build arguments: `BUILDPLATFORM` (the machine running the build), `TARGETPLATFORM` (what you're building for), `TARGETOS`, `TARGETARCH`. For compiled languages, run the compiler on the *native* platform and tell it to emit the target architecture:

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

- `FROM --platform=$BUILDPLATFORM`: run the builder stage natively (fast), regardless of the target.
- `ARG TARGETOS` / `ARG TARGETARCH`: BuildKit fills them in; you must declare them to use them.
- `GOOS`/`GOARCH`: Go's built-in cross-compile switches.
- The final stage is built for the **target** platform, but it only copies a file, so no emulation is needed.

### Pitfalls

- **Python/Node packages with native code**: need arm64 wheels or compile under emulation (slow). Check availability.
- **Base images** must publish your target platforms (official images mostly do).
- **GPU/CUDA images** are usually amd64 only (Jetson/ARM variants exist separately, [Chapter 8](08-docker-for-ai-and-ml.md)).
- **"exec format error"** at runtime = you're running an image built for the wrong architecture. Fix with the right image or `--platform` on `docker run`.
- Running a foreign-architecture image deliberately: `docker run --platform linux/amd64 ...` (works through emulation on Apple Silicon).

---

## 20. Exporting the build cache for CI

On your laptop the build cache sits on disk between builds. On **CI** (GitHub Actions, GitLab CI, Jenkins agents...) each job often starts on a fresh machine with an **empty cache**, so every build is a cold build. The fix is to **export the cache** to somewhere persistent and **import** it on the next run.

```bash
docker buildx build \
  --cache-from type=registry,ref=registry.example.com/me/my-api:buildcache \
  --cache-to   type=registry,ref=registry.example.com/me/my-api:buildcache,mode=max \
  -t registry.example.com/me/my-api:1.0 --push .
```

- `--cache-from`: where to **read** previously exported cache.
- `--cache-to`: where to **write** cache after the build.
- `type=registry`: store the cache as a special image tag (`:buildcache`) in your container registry. Works anywhere.
- `mode=max`: export cache for **all** layers of **all** stages. The default `mode=min` exports only layers of the final image, which misses the expensive builder-stage layers, so you almost always want `max` for multi-stage builds.

Cache backends:

| Backend | Syntax | Notes |
|---|---|---|
| Registry | `type=registry,ref=<image>:<tag>` | Universal; uses registry storage. |
| GitHub Actions cache | `type=gha` | Convenient in GitHub Actions; limited storage per repo. |
| Local directory | `type=local,src=/tmp/cache` / `dest=/tmp/cache` | For CI systems that persist a directory between runs. |
| Inline | `type=inline` (for `--cache-to`) | Embeds cache metadata in the image itself; only `mode=min`. |
| S3 / Azure Blob | `type=s3`, `type=azblob` | Cloud object storage. |

Important: exporting cache requires a builder with the `docker-container` driver (or another non-default driver), not the plain `docker` driver. Example GitHub Actions snippet (the `docker/build-push-action` and `docker/setup-buildx-action` actions are maintained by Docker):

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

(Check the actions' README for the currently recommended major versions.) More CI/CD detail in [Chapter 11](11-production-and-enterprise.md).

---

## 21. `docker init`: generate a starter setup

`docker init` is an interactive command that inspects your project and generates good-practice starter files:

```bash
cd my-project
docker init
```

It asks a few questions (which application platform, version, port, run command) and writes:

| File | Purpose |
|---|---|
| `Dockerfile` | A well-structured Dockerfile for your stack (multi-stage where appropriate, non-root user, cache mounts) |
| `compose.yaml` | A Compose file to run it (`docker compose up --build`) |
| `.dockerignore` | A sensible ignore list |
| `README.Docker.md` | Short instructions for building/running/deploying |

Supported templates (per Docker's reference): ASP.NET Core, Go, Java (Maven), Node, PHP with Apache, Python, Rust, and "Other". If any of the target files already exist, it warns you and offers to overwrite **all** of them, so commit or back up first because overwritten files are not recoverable.

Treat the output as an excellent **first draft**: read every line (you now can), then adapt it. It's also an easy way to learn the current idioms for a stack you don't know well.

---

## 22. Linting and inspecting images

### hadolint

**hadolint** ("Haskell Dockerfile Linter") checks your Dockerfile against best-practice rules and also runs **ShellCheck** on your `RUN` scripts.

```bash
# macOS/Linux/Git Bash:
docker run --rm -i hadolint/hadolint < Dockerfile

# Windows PowerShell (no "<" redirection operator):
Get-Content Dockerfile | docker run --rm -i hadolint/hadolint
```

- `--rm`: remove the throwaway container after it runs.
- `-i`: keep standard input open so the Dockerfile text can be piped in.
- `hadolint/hadolint`: the image.

Example output:

```
-:1 DL3007 warning: Using latest is prone to errors if the image will ever update. Pin the version explicitly to a release tag
-:5 DL3008 warning: Pin versions in apt get install. Instead of `apt-get install <package>` use `apt-get install <package>=<version>`
-:9 DL3025 warning: Use arguments JSON notation for CMD and ENTRYPOINT arguments
```

Common rules (verify exact wording with `hadolint --help` or its wiki):

| Rule | Meaning |
|---|---|
| DL3006 / DL3007 | Tag your base image / don't use `latest` |
| DL3008 | Pin apt package versions (often ignored in practice; see below) |
| DL3013 | Pin pip package versions |
| DL3002 | Last `USER` should not be root |
| DL3025 | Use JSON (exec) notation for `CMD`/`ENTRYPOINT` |
| DL3059 | Multiple consecutive `RUN` instructions; consider consolidating |
| DL4006 | Set `SHELL -o pipefail` before `RUN` with a pipe |

You can ignore rules you disagree with in a `.hadolint.yaml` (`ignored: [DL3008]`) or inline with `# hadolint ignore=DL3008`. Add hadolint to CI so Dockerfile mistakes fail pull requests.

### Other tools worth knowing

| Tool | What for |
|---|---|
| `docker build --check` | Built-in BuildKit checks |
| `docker scout` | Vulnerability and base-image recommendations (Docker's service; free tier available) |
| Trivy, Grype | Open-source vulnerability scanners ([Chapter 10](10-security.md)) |
| dive | Browse layers and find wasted space |
| `docker history`, `docker inspect` | Built-in layer and config introspection |

---

## 23. A full production-grade Dockerfile for FastAPI

Here's everything from this chapter in one annotated file. Assume this project layout:

```
my-api/
  Dockerfile
  .dockerignore
  requirements.txt          # pinned runtime dependencies
  app/
    __init__.py
    main.py                 # FastAPI app with a GET /health endpoint
```

`app/main.py` (minimal):

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/")
def root():
    return {"message": "Hello from a production-grade container"}
```

`requirements.txt`:

```
fastapi==0.115.0
uvicorn[standard]==0.30.6
```

(Use current pinned versions in your project; these numbers are just examples. Generate a fully pinned list with `pip freeze` or a lockfile tool.)

**`Dockerfile`:**

```dockerfile
# syntax=docker/dockerfile:1

# ---------------------------------------------------------------------------
# Global build argument: lets you change the Python version without editing
# the file:  docker build --build-arg PYTHON_VERSION=3.12 .
# For maximum reproducibility, replace the tag with a digest:
#   FROM python:${PYTHON_VERSION}-slim@sha256:<digest> AS base
# ---------------------------------------------------------------------------
ARG PYTHON_VERSION=3.13

# ======================= Stage 1: base (shared settings) =====================
FROM python:${PYTHON_VERSION}-slim AS base

# PYTHONDONTWRITEBYTECODE: don't write .pyc files (smaller image, no clutter)
# PYTHONUNBUFFERED:        flush stdout/stderr immediately so `docker logs` is live
# PIP_DISABLE_PIP_VERSION_CHECK: skip the network call that checks for new pip
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# ======================= Stage 2: builder (fat, discarded) ===================
FROM base AS builder

# Compilers are only needed if some dependency must be built from source.
# They live in THIS stage only and never reach the final image.
RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    rm -f /etc/apt/apt.conf.d/docker-clean \
 && apt-get update \
 && apt-get install -y --no-install-recommends build-essential

# A virtualenv at a fixed path makes it trivially copyable to the final stage.
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Bind-mount requirements.txt (no COPY layer needed) and keep pip's download
# cache between builds. The layer is rebuilt only when requirements.txt changes.
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

# Bring in ONLY the installed dependencies from the builder.
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copy application code LAST (it changes most often) and set the owner in the
# same step (no slow, size-doubling `chown -R` layer).
COPY --chown=app:app app/ ./app/

# Drop privileges. Use the numeric ID so orchestrators can verify non-root.
USER 10001:10001

# Documentation only. Publishing the port is done with -p / ports:.
EXPOSE 8000

# The container is healthy when /health answers. Uses only the Python standard
# library because slim images do not ship curl.
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
  CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]

# Ask for a graceful shutdown on `docker stop` (this is also the default).
STOPSIGNAL SIGTERM

# Exec form: uvicorn is PID 1 and receives SIGTERM directly.
# 0.0.0.0 so the server is reachable from outside the container.
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`.dockerignore`:** use the starter from section 4.

**Build and run it:**

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
- `docker ps` STATUS goes from `(health: starting)` to `(healthy)` within roughly 30 seconds.
- `docker exec my-api id` prints `uid=10001(app) gid=10001(app) groups=10001(app)`: proof you're not root.

Clean up your test container with `docker stop my-api && docker rm my-api`.

### Why each decision was made (cheat sheet)

| Decision | Reason |
|---|---|
| `# syntax=` first line | Pin frontend features independent of local Docker |
| Shared `base` stage | Environment variables defined once |
| `builder` with `build-essential` | Compile native deps without shipping the compiler |
| venv at `/opt/venv` | One folder = one `COPY --from` |
| Cache mounts | Fast rebuilds, small image |
| Bind mount for `requirements.txt` | No extra layer; cache invalidates correctly |
| Numeric non-root `USER` | Security; verifiable by orchestrators |
| `COPY --chown` | No duplicated layer |
| Code copied last | Cache efficiency |
| `HEALTHCHECK` without curl | Works in slim, no extra package |
| Exec-form `CMD` | Signals reach uvicorn |
| `--init` recommended at run | Zombie reaping, signal safety net |
| No secrets anywhere | See section 17 |

### Variations

- **Gunicorn + Uvicorn workers** for multi-process: `CMD ["gunicorn", "app.main:app", "-k", "uvicorn.workers.UvicornWorker", "-w", "4", "-b", "0.0.0.0:8000"]`. In container orchestration, many teams prefer **one worker per container and scale with replicas**, which fits autoscaling and memory limits better; choose based on your platform.
- **`--proxy-headers`** and `--forwarded-allow-ips` when behind a reverse proxy ([Chapter 5](05-storage-and-networking.md#b12-reverse-proxies-in-front)).
- **Read-only root filesystem** at run time: `docker run --read-only --tmpfs /tmp ...` ([Chapter 10](10-security.md)).

---

## 24. Common mistakes

1. **`COPY . .` before installing dependencies.** Every code edit reinstalls everything. (Section 10.)
2. **Using `latest` or no tag.** Builds change under you. (Section 13.)
3. **Running as root.** (Section 15.)
4. **Secrets in `ENV`, `ARG`, or `COPY`'d files**, even if "deleted later". (Section 17.)
5. **No `.dockerignore`.** Huge contexts; `.git`, `.env` and `node_modules` inside the image. (Section 4.)
6. **Shell-form `CMD`/`ENTRYPOINT`.** No signal handling, 10 s stops. (Sections 6, 16.)
7. **Single quotes in JSON array form** (`CMD ['a']`), silently becoming shell form.
8. **`RUN apt-get update` on its own line**, creating stale-cache bugs; or forgetting `rm -rf /var/lib/apt/lists/*`. (Section 3.)
9. **Deleting files in a later layer** expecting a smaller image. (Section 3.)
10. **`chown -R` / `chmod -R` as separate `RUN`** duplicating all files. Use `COPY --chown/--chmod`.
11. **Forgetting `exec` in entrypoint scripts.** (Section 6.)
12. **Windows line endings (CRLF) in shell scripts** causing "no such file or directory" on a file that clearly exists. (Section 6.)
13. **Server bound to `127.0.0.1`** inside the container: unreachable from outside. Bind to `0.0.0.0`.
14. **Using `ADD` where `COPY` would do**, and surprise tar extraction.
15. **Installing dev tools, tests and docs in the production image.** Use multi-stage.
16. **Alpine for Python/ML** with a heavy dependency tree. (Section 12.)
17. **One giant `RUN` for unrelated steps**, or dozens of tiny `RUN` lines. Group *related* commands that share cleanup or caching behaviour.
18. **Baking multi-gigabyte model weights/datasets into images** without thinking. ([Chapter 8](08-docker-for-ai-and-ml.md))
19. **Ignoring `docker history`/image size** until deploys get painfully slow.
20. **Using `VOLUME` in your own application Dockerfile**, then wondering where the data went and why the anonymous volumes keep multiplying.
21. **Assuming `EXPOSE` publishes a port.** It doesn't.
22. **`HEALTHCHECK` using `curl` in an image that doesn't have it**: the container becomes permanently unhealthy.

---

## 25. Exercises

**Exercise 1: Read a Dockerfile.** For each line of the section 23 Dockerfile, explain in your own words what it does and what would break if you deleted it. Then check yourself against the cheat sheet.

**Exercise 2: Feel the cache.** Build the BEFORE and AFTER Dockerfiles from section 10 with a real `requirements.txt` of ~10 packages. Record the time of (a) a cold build, (b) a rebuild after editing a source file, (c) a rebuild after adding one package to `requirements.txt`.

**Exercise 3: Shrink it.** Build a trivial Go "hello world" HTTP server (a single `main.go`) three ways: `FROM golang:1.25` single-stage; multi-stage onto `alpine`; multi-stage onto `distroless/static`. Compare `docker images` sizes. Expected pattern: ~800 MB+ vs ~15 MB vs ~10 MB (illustrative; check your own numbers).

**Exercise 4: Catch the leak.** Build the leaky `ARG` Dockerfile from section 17 using a clearly fake token, inspect it with `docker history --no-trunc`, then fix it with a secret mount and show that the token no longer appears. Remove your test images afterwards (`docker rmi leaky safe`, images you created).

**Exercise 5: Signals.** Build two images that run a trivial Python HTTP server: one with shell-form CMD, one with exec-form CMD. Time `docker stop` on each. Then add `--init` to the shell-form one and compare.

**Exercise 6: Entrypoint script.** Write an `entrypoint.sh` that prints "starting ..." and then `exec "$@"`. Wire it with `ENTRYPOINT` and `CMD`. Override `CMD` from `docker run` and confirm the script still runs.

**Exercise 7: Multi-arch.** Create a `docker-container` builder and build any small image with `--platform linux/amd64,linux/arm64` into a local OCI tarball (`--output type=oci,dest=multi.tar`) to avoid needing a registry. Inspect the tarball's `index.json`.

**Exercise 8: Lint it.** Run hadolint on a sloppy Dockerfile (use `latest`, root, shell-form CMD, `apt-get install` without cleanup) and fix every warning. Then run `docker build --check .` and compare the findings.

**Exercise 9: `docker init`.** In a small Python or Node project run `docker init`, read the output files, and write down three things the generated Dockerfile does that you did not do before.

---

## 26. Summary

- A Dockerfile is a recipe; each file-changing instruction creates an immutable, content-addressed **layer**; the build **cache** reuses unchanged layers and invalidates everything after the first change. **Order: least-changing first, code last.**
- Use `COPY` (not `ADD`), exec-form `CMD`/`ENTRYPOINT`, `WORKDIR`, a numeric non-root `USER`, a `HEALTHCHECK`, and a `.dockerignore`.
- **Multi-stage builds** separate "build" from "run" and are the main tool for small, secure images.
- Prefer `-slim` for Python/Node, distroless/scratch for static binaries, and be careful with Alpine's musl.
- Pin base images (tag at minimum, digest ideally) and automate updates.
- **BuildKit** features (`RUN --mount` cache/secret/ssh/bind, heredocs, `COPY --link`, `--check`, buildx multi-platform and cache export) make builds fast, safe and portable.
- **Never put secrets in layers.** Use `--mount=type=secret`.
- Make sure PID 1 handles signals: exec form, `exec` in scripts, `--init` or tini.

**Next:** [Chapter 5: Storage and Networking](05-storage-and-networking.md): what happens to data when a container dies, and how containers talk to each other and the outside world.
