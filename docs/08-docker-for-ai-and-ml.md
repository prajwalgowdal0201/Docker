# Chapter 8: Docker for AI and Machine Learning

> **What you will learn in this chapter**
> - Why AI/ML projects are the single best argument for Docker (and why they are also the hardest to containerize well).
> - What an AI application looks like when split into containers: API, model server, vector database, cache, queue, worker, UI.
> - How to pick base images, and how to shrink them (CPU-only wheels, multi-stage builds, uv).
> - How GPUs work inside containers: drivers vs CUDA vs the NVIDIA Container Toolkit, `--gpus all`, CDI, Compose GPU syntax, WSL2 on Windows, and why macOS is different.
> - How to deal with model weights (bake in vs download vs mount) with the trade-offs.
> - Running local LLMs: Ollama, Docker Model Runner, llama.cpp, vLLM, TGI, Triton.
> - A complete RAG stack in Compose, with every term explained.
> - Calling hosted LLM APIs safely, MCP servers in containers, sandboxing agent code execution, Jupyter, training jobs, MLflow, workers, streaming behind nginx, observability, CI/CD, multi-arch builds, and the road from laptop to cloud GPU.

Previous: [Chapter 7: Who Needs Docker?](07-who-needs-docker.md) | Next: [Chapter 9: Hands-on AI App Walkthrough](09-hands-on-ai-app-walkthrough.md)

> **A note on freshness.** AI tooling changes monthly. Where I give image names, tags or flags, I say how I checked them (official docs fetched while writing this chapter, in late 2026) and I flag anything I could not verify. Always glance at the linked official docs before pinning a tag in production. Version numbers in examples are *examples*.

---

## 8.1 Why AI needs Docker

Plain software has dependencies. AI software has **layers of dependencies that must match each other exactly**. Here is what a typical GPU Python AI project depends on, from the hardware upward:

```
+----------------------------------------------------------+
|  Your code (app.py, prompts, business logic)             |
+----------------------------------------------------------+
|  Python packages: transformers, langchain, fastapi, ...  |
+----------------------------------------------------------+
|  Deep-learning framework: PyTorch / TensorFlow / JAX     |   must match
+----------------------------------------------------------+   v
|  CUDA runtime + cuDNN + NCCL  (NVIDIA's GPU libraries)   |   must match
+----------------------------------------------------------+   v
|  NVIDIA kernel driver (installed on the HOST machine)    |   must match
+----------------------------------------------------------+   v
|  GPU hardware (Ampere, Hopper, Blackwell, ...)           |
+----------------------------------------------------------+
|  Python itself (3.10? 3.12? 3.13?) and glibc, OS libs    |
+----------------------------------------------------------+
```

Some plain-English definitions for terms you just met:

* **GPU (Graphics Processing Unit).** A chip with thousands of small cores, originally for graphics, now the workhorse for the matrix maths of neural networks.
* **CUDA.** NVIDIA's programming platform and library set that lets software run maths on NVIDIA GPUs.
* **cuDNN.** NVIDIA's library of optimized deep-learning building blocks (convolutions etc.) used by frameworks.
* **PyTorch / TensorFlow / JAX.** Deep-learning frameworks, the libraries you use to define and run models.
* **Driver.** The software, installed on the host operating system, that lets the OS talk to the GPU.
* **Inference.** Using a trained model to produce outputs (answering a prompt). **Training** is the process of creating or adjusting the model's weights from data.
* **Weights.** The huge arrays of numbers that *are* the model. A 7-billion-parameter model is typically several to ~15 GB depending on numeric precision.

### The problems Docker solves here

| Problem ("dependency hell") | Without Docker | With Docker |
|---|---|---|
| PyTorch 2.x needs a specific CUDA build; your other project needs an older one | Constant reinstalling, broken environments | Each project has its own image; both coexist |
| "It worked on the data scientist's laptop" | Notebook works, production crashes on an import | Same image from laptop to production |
| Python/OS library mismatches (`libssl`, `libGL`, `glibc`) | Debugging system packages on each machine | Libraries are inside the image |
| Reproducing a result from six months ago | Environment is gone | Re-run the pinned image |
| Deploying a model server on a new machine | Hours of driver/CUDA/Python setup | `docker run` (after the one-time driver + toolkit setup) |
| Many teams, many frameworks on shared GPU servers | Conflicts and "who updated the driver?" fights | Isolated containers share one driver |
| Onboarding a new ML engineer | A day of setup | `docker compose up` |
| Scaling inference replicas | Manual server provisioning | Orchestrators schedule identical containers |

### What Docker does *not* solve for AI

Be realistic so you are not surprised:

1. **The driver lives on the host, not in the image.** Docker cannot containerize the kernel driver. You must install an NVIDIA driver on the host (or use a cloud image that includes it).
2. **Images get huge.** GPU libraries are several GB. Plan for it (section 8.4).
3. **Containers do not make GPUs faster or cheaper.** They make environments reproducible.
4. **macOS containers cannot use the Apple GPU** (section 8.6.7).
5. **Model weights are not "code".** Treat them as data with their own distribution strategy (section 8.7).

---

## 8.2 Anatomy of an AI app in containers

A real AI application is almost never one program. It is a small system. Here is the common shape, called a **RAG chatbot** (Retrieval-Augmented Generation; explained fully in section 8.10):

```
                     +-------------------+
   Browser / app --> |   UI (React/      |
                     |   Open WebUI)     |
                     +---------+---------+
                               | HTTPS (via reverse proxy)
                     +---------v---------+       +--------------------+
                     |   API server      | <---> |  Cache (Redis)     |
                     |   (FastAPI)       |       +--------------------+
                     +----+---------+----+
                          |         |            +--------------------+
                          |         +----------> |  Queue (Redis/     |
        embeddings /      |                      |  RabbitMQ)         |
        search            |                      +----------+---------+
                          v                                 |
                +---------+---------+             +---------v---------+
                |  Vector DB        |             |  Worker (Celery/  |
                |  (Qdrant/pgvector)|             |  RQ) - ingestion, |
                +-------------------+             |  batch jobs       |
                          ^                       +-------------------+
                          |
                +---------+---------+      +----------------------------+
                |  Embedding model  |      |  LLM: local (Ollama/vLLM)  |
                |  (local or API)   |      |  OR hosted API (key in     |
                +-------------------+      |  secret, base URL in env)  |
                                           +----------------------------+
```

| Component | Job | Typical image | GPU needed? |
|---|---|---|---|
| **Reverse proxy** | TLS, routing, streaming timeouts | `nginx`, `caddy`, `traefik` | No |
| **UI** | Chat page | `ghcr.io/open-webui/open-webui`, your React build | No |
| **API server** | Your business logic, auth, prompt assembly, calls models and DB | Your image, `python:3.12-slim` base | No (usually) |
| **Model server** | Runs the neural network for inference | `ollama/ollama`, `vllm/vllm-openai`, `llama.cpp` server, Triton | Usually yes |
| **Embedding service** | Turns text into vectors | Same as model server, or a library inside the API | Optional |
| **Vector database** | Stores vectors, finds similar ones | `qdrant/qdrant`, `pgvector/pgvector`, `chromadb/chroma` | No |
| **Cache** | Remember answers, rate limits, sessions | `redis` | No |
| **Queue + worker** | Long jobs: document ingestion, batch scoring, fine-tuning | `redis` + your worker image | Sometimes |
| **Experiment tracker** | Record training runs | MLflow | No |
| **Observability** | Metrics and dashboards | Prometheus, Grafana, Loki | No |

**Design rule.** Keep the **API container thin and CPU-only**, and put the GPU-hungry model in its **own container**. Reasons: the thin API rebuilds in seconds and scales cheaply; the model server rarely changes and is expensive; you can swap models without redeploying your API; and the GPU can be shared by multiple API replicas.

---

## 8.3 Base image choices

The `FROM` line (see [Chapter 4](04-dockerfile-deep-dive.md)) is the biggest decision for image size, build time, and compatibility.

### 8.3.1 The options

| Base image | Typical use | Size ballpark* | Pros | Cons |
|---|---|---|---|---|
| `python:3.12-slim` | API servers, CPU inference, glue code | ~100-200 MB | Small, Debian-based (wheels just work) | You install everything else yourself; no GPU libraries |
| `python:3.12-alpine` | Rarely for ML | smaller | Tiny | Uses musl libc; many ML wheels (numpy, torch) are built for glibc, so builds are slow or fail. **Avoid for ML.** |
| `pytorch/pytorch:<ver>-cuda<ver>-cudnn<ver>-runtime` | GPU inference/training with PyTorch quickly | multiple GB | Preinstalled, tested combo of PyTorch + CUDA + cuDNN | Big; you are locked to their Python/PyTorch combo |
| `nvidia/cuda:<ver>-base-<os>` | Minimal CUDA runtime pieces | ~hundreds of MB | Smallest NVIDIA-supported base | Does not include cuDNN or full libraries |
| `nvidia/cuda:<ver>-runtime-<os>` | Running prebuilt CUDA apps | ~1-3 GB | CUDA runtime libraries | No compiler |
| `nvidia/cuda:<ver>-cudnn-runtime-<os>` | Runtime + cuDNN | larger | Convenient for frameworks that want system cuDNN | |
| `nvidia/cuda:<ver>-devel-<os>` | Compile CUDA code (flash-attention, custom kernels) | ~3-8 GB | Compiler `nvcc`, headers | Huge; use **only in the build stage** |
| `nvcr.io/nvidia/pytorch:<yy.mm>-py3` | NVIDIA's NGC-optimized PyTorch | very large (~10-20+ GB) | Highly tuned, includes many extras | Huge, NGC registry |
| `vllm/vllm-openai`, `ollama/ollama` | Ready-made model servers | large | Zero build | Less customization |

\*Sizes vary with versions and are reported uncompressed or compressed depending on the tool. **Measure with `docker images` or `docker image ls` on your machine** rather than trusting any table, including this one.

### 8.3.2 Reading `nvidia/cuda` tag names

NVIDIA's official `nvidia/cuda` image on Docker Hub uses tags shaped like:

```
nvidia/cuda:<CUDA version>-<flavour>-<OS>

nvidia/cuda:13.4.3-cudnn-devel-ubuntu24.04
              |      |      |       |
              |      |      |       +-- base OS (Ubuntu 22.04 / 24.04 / 26.04 and some others like Rocky, UBI)
              |      |      +---------- devel = compiler + headers; runtime = libs only; base = minimal
              |      +----------------- cudnn = includes cuDNN (also tensorrt variants exist)
              +------------------------ CUDA toolkit version
```

When I checked Docker Hub while writing this (October 2026), the newest tags were CUDA **13.4.3** with `cudnn-devel`, `cudnn-runtime`, `tensorrt-devel` and `tensorrt-runtime` flavours on Ubuntu 22.04/24.04/26.04. The page I could read did not list the plain `base`/`runtime` tags, but they follow the same pattern. Always check <https://hub.docker.com/r/nvidia/cuda/tags> for what exists today, because NVIDIA removes old tags over time (which is a good reason to pin and mirror the images you depend on, see [Chapter 11](11-production-and-enterprise.md)).

| Flavour | Contains | Use for |
|---|---|---|
| `base` | Minimal CUDA runtime pieces (libcudart etc.) | Smallest option; apps that bundle their own CUDA libs |
| `runtime` | Base + CUDA math libraries (cuBLAS, etc.) | Running compiled CUDA apps |
| `cudnn-runtime` | Runtime + cuDNN | Frameworks needing system cuDNN |
| `devel` / `cudnn-devel` | Everything + `nvcc` compiler and headers | **Build stage only** (compile extensions) |

### 8.3.3 A surprising fact: you often do not need an `nvidia/cuda` image at all

The **pip wheels of PyTorch for CUDA bundle their own CUDA runtime libraries** (as separate `nvidia-*` Python packages). That means on a host with a recent enough NVIDIA driver, this works with a plain Python base:

```dockerfile
FROM python:3.12-slim
RUN pip install torch        # default Linux wheel pulls CUDA libs as pip packages
```

You pay in image size (the CUDA libs arrive via pip: gigabytes), but you do not need `nvidia/cuda` or even a CUDA toolkit. You *only* need `nvidia/cuda:*-devel` when you must **compile** something (flash-attention, xformers from source, custom CUDA extensions) or when a vendor documents it as required.

**Rule of thumb:**
* Using prebuilt wheels (most people) -> `python:3.12-slim` (+ GPU passthrough at run time).
* Need a tested all-in-one PyTorch -> `pytorch/pytorch:*-runtime`.
* Compiling CUDA code -> build in `nvidia/cuda:*-devel`, copy results to `-runtime` (multi-stage, section 8.3.5).

### 8.3.4 CPU-only wheels: the cheapest size win

If your container will **never touch a GPU** (an API calling a hosted LLM, a small embedding model on CPU, a classic scikit-learn model), do not install the CUDA build of PyTorch. PyTorch publishes a CPU-only wheel index. Reports from several open-source projects (GitHub issues and a PR in the Roboflow inference project) indicate:

* the CPU-only `torch` wheel is roughly **200 MB** (one report measured 196 MB; another measured 222 MB for the whole CPU-only install across 13 wheels),
* while the default Linux PyPI install pulls in several gigabytes of NVIDIA and Triton packages (reports of about 2.4-2.7 GB of unusable CUDA wheels in CPU deployments),
* with image-level savings reported between roughly 2.5 GB compressed and 4+ GB uncompressed. These figures come from community reports, not PyTorch's official docs, and your savings will differ.

How to do it with pip:

```dockerfile
RUN pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cpu
```

* `pip install torch` installs PyTorch.
* `--no-cache-dir` stops pip from keeping downloaded files inside the image layer (saves space).
* `--index-url https://download.pytorch.org/whl/cpu` tells pip to fetch from PyTorch's CPU-only package index instead of PyPI.

With **uv** (a very fast Python package manager written in Rust), you can configure the PyTorch CPU index in `pyproject.toml`:

```toml
[tool.uv.sources]
torch = { index = "pytorch-cpu" }

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

* `[tool.uv.sources]` says "for the package `torch`, use the index named `pytorch-cpu`".
* `[[tool.uv.index]]` declares that index; `explicit = true` means it is used **only** for packages that explicitly request it (this prevents "dependency confusion", where a malicious package with the same name on another index could sneak in).

(Check the uv documentation, "Using uv with PyTorch", for the current recommended configuration; uv also offers a `--torch-backend` option that I mention only because community sources reported it; I did not verify its exact behavior.)

> **Important.** Dependencies of dependencies can also drag in CUDA wheels (for example, some packages list `torch` as a dependency). After building, verify what you got: `docker run --rm myimage pip list | grep -i nvidia` should print nothing for a CPU image.

### 8.3.5 Multi-stage builds for ML

A **multi-stage build** uses one stage to compile/install (with big tools) and a final slim stage that only copies the results. Concept diagram:

```
Stage 1 "builder"  (FROM python:3.12 ... big, has compilers)
   - install build tools
   - install dependencies into /app/.venv
                       |
                       |  COPY --from=builder /app/.venv  (only the venv crosses over)
                       v
Stage 2 "runtime"  (FROM python:3.12-slim ... small)
   - copy venv + your code
   - run as non-root
```

Full annotated example in the next section.

---

## 8.4 Example 1: a lean CPU AI API image (FastAPI + uv + multi-stage)

Use case: an API that calls a hosted LLM and does lightweight embedding or post-processing on CPU.

Project layout:

```
rag-api/
  pyproject.toml
  uv.lock
  .dockerignore
  Dockerfile
  app/
    main.py
```

`app/main.py` (minimal, for the walk-through):

```python
import os
from fastapi import FastAPI

app = FastAPI()

@app.get("/healthz")
def healthz():
    return {"status": "ok"}

@app.get("/config")
def config():
    # Never return secrets. Return only non-sensitive config for debugging.
    return {"llm_base_url": os.getenv("LLM_BASE_URL", "unset")}
```

`pyproject.toml`:

```toml
[project]
name = "rag-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
  "fastapi>=0.115",
  "uvicorn[standard]>=0.30",
  "httpx>=0.27",
  "openai>=1.40",
  "qdrant-client>=1.9",
]
```

`.dockerignore` (keeps junk and secrets out of the build context; see [Chapter 4](04-dockerfile-deep-dive.md)):

```
.git
.venv
__pycache__
*.pyc
.env
.env.*
*.pem
models/
data/
notebooks/
```

`Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

############ Stage 1: build the virtual environment ############
FROM python:3.12-slim AS builder

COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

ENV UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-install-project

COPY app ./app
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev

############ Stage 2: tiny runtime image ############
FROM python:3.12-slim AS runtime

RUN groupadd --system --gid 10001 app \
 && useradd  --system --uid 10001 --gid app --no-create-home app

WORKDIR /app
COPY --from=builder --chown=app:app /app /app

ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

USER app
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://127.0.0.1:8000/healthz').status==200 else 1)"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Explanation, line by line:

| Line | Meaning |
|---|---|
| `# syntax=docker/dockerfile:1` | Use the current stable Dockerfile frontend (enables `--mount` features). |
| `FROM python:3.12-slim AS builder` | First stage named `builder`. `slim` = Debian with minimal packages. **In production, pin to a digest** (`python:3.12-slim@sha256:...`), see [Chapter 10](10-security.md). |
| `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/` | Copies the `uv` binaries out of uv's official image into ours. This is the documented pattern; **pin a version tag** (e.g. `uv:0.9.x`) instead of `latest` for reproducibility. |
| `ENV UV_COMPILE_BYTECODE=1` | Precompile `.pyc` files at install time so the container starts faster. |
| `UV_LINK_MODE=copy` | Copy files rather than hard-link from the cache (the cache is a separate mount, so links would not work and would warn). |
| `UV_PYTHON_DOWNLOADS=never` | Do not let uv download another Python; use the image's. |
| `WORKDIR /app` | Sets working directory (created if absent). |
| `COPY pyproject.toml uv.lock ./` | Copy **only** dependency manifests first, so Docker's layer cache is reused when only your code changes. |
| `RUN --mount=type=cache,target=/root/.cache/uv uv sync --locked --no-dev --no-install-project` | `--mount=type=cache` gives BuildKit a persistent cache directory that is **not** stored in the image. `uv sync` installs exactly what the lock file says; `--locked` fails if `uv.lock` is out of date (guards reproducibility); `--no-dev` skips dev dependencies; `--no-install-project` installs dependencies only (your code is not copied yet). |
| `COPY app ./app` then second `uv sync` | Now add code and install the project itself. Dependencies layer stays cached when only code changes. |
| `FROM python:3.12-slim AS runtime` | Fresh, small final image: no uv, no build cache. |
| `groupadd`/`useradd --system ... --no-create-home` | Creates a non-root user with fixed UID 10001. Fixed IDs make Kubernetes `runAsUser` and volume permissions predictable. |
| `COPY --from=builder --chown=app:app /app /app` | Brings over the venv and code from stage 1, owned by the new user. |
| `ENV PATH="/app/.venv/bin:$PATH"` | Makes `python`/`uvicorn` resolve to the virtual environment. |
| `PYTHONUNBUFFERED=1` | Print logs immediately (important for `docker logs`). |
| `PYTHONDONTWRITEBYTECODE=1` | Do not write `.pyc` at run time (filesystem may be read-only). |
| `USER app` | Everything after (and the running container) is non-root. |
| `EXPOSE 8000` | Documentation only: says the app listens on 8000. It does not publish anything. |
| `HEALTHCHECK ...` | Docker periodically runs this; failure marks the container `unhealthy`. Uses Python stdlib so no `curl` is needed in the image. |
| `CMD [...]` | Starts uvicorn. `--host 0.0.0.0` is **essential**: inside a container, binding `127.0.0.1` means only the container itself can connect, so published ports would not work. |

Build and run:

```bash
docker build -t rag-api:0.1.0 .
docker run --rm -p 127.0.0.1:8000:8000 -e LLM_BASE_URL=https://api.example.com/v1 rag-api:0.1.0
```

* `-p 127.0.0.1:8000:8000` publishes the container's port 8000 on the host's loopback only.
* `-e LLM_BASE_URL=...` sets an env var (a placeholder URL here).

```bash
curl http://127.0.0.1:8000/healthz
# {"status":"ok"}
```

Check the size and the layers:

```bash
docker images rag-api
docker history rag-api:0.1.0
```

* `docker history` lists each layer and its size, so you can see which instruction added the weight.

---

## 8.5 Example 2: a GPU inference image

Use case: you must run your own PyTorch model on an NVIDIA GPU.

```dockerfile
# syntax=docker/dockerfile:1

FROM python:3.12-slim AS builder
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --locked --no-dev --no-install-project

FROM python:3.12-slim AS runtime
RUN groupadd --system --gid 10001 app && useradd --system --uid 10001 --gid app --create-home app
WORKDIR /app
COPY --from=builder --chown=app:app /app/.venv /app/.venv
COPY --chown=app:app app ./app

ENV PATH="/app/.venv/bin:$PATH" \
    HF_HOME=/models/hf \
    PYTHONUNBUFFERED=1 \
    NVIDIA_VISIBLE_DEVICES=all \
    NVIDIA_DRIVER_CAPABILITIES=compute,utility

USER app
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Things that are new compared with example 1:

| Line | Meaning |
|---|---|
| `torch` is in `pyproject.toml` as a normal dependency | The default Linux wheel brings the CUDA runtime libraries as pip packages. No `nvidia/cuda` base is required. |
| `HF_HOME=/models/hf` | Tells Hugging Face libraries where to cache downloaded models. We will mount a volume at `/models` (section 8.7). |
| `NVIDIA_VISIBLE_DEVICES=all` | A variable read by the NVIDIA runtime to decide which GPUs are visible. Often set by the runtime flags anyway. |
| `NVIDIA_DRIVER_CAPABILITIES=compute,utility` | Which driver features to expose: `compute` for CUDA, `utility` for `nvidia-smi`. (Add `video` if you need NVENC/NVDEC.) |

If you do need to compile CUDA extensions, use a devel stage:

```dockerfile
FROM nvidia/cuda:13.4.3-cudnn-devel-ubuntu24.04 AS build
# ... install python, build the extension wheel ...

FROM nvidia/cuda:13.4.3-cudnn-runtime-ubuntu24.04 AS runtime
# ... copy the built wheel, install it, run as non-root ...
```

(Replace `13.4.3` with a tag that exists *and* that your framework wheels support. Your **framework** chooses the CUDA version: if you install PyTorch built for CUDA 12.8, a 12.8 system toolkit is the safe match for compiling extensions.)

---

## 8.6 GPUs in Docker

This is the topic that confuses beginners most, so we go slowly.

### 8.6.1 Three different things called "CUDA"

| Thing | Where it lives | Who installs it |
|---|---|---|
| **NVIDIA driver** (kernel module + `libcuda.so` user-space driver library) | **On the host** only | You (or the cloud image) |
| **CUDA toolkit / runtime libraries** (cuBLAS, cuDNN, `nvcc`...) | **Inside the image** (from `nvidia/cuda`, from a framework image, or from pip wheels) | Image builder |
| **NVIDIA Container Toolkit** | **On the host**, integrates with Docker | You, once |

The magic: the **NVIDIA Container Toolkit** injects the host driver's libraries and the GPU device files (`/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`) into your container at start time. So the image carries the CUDA *libraries*, the host supplies the *driver*.

```
 Host (Linux or WSL2)                       Container
+----------------------------------+      +-----------------------------+
| NVIDIA kernel driver  (e.g. 5xx) |----->| /dev/nvidia*  (injected)    |
| libcuda.so (driver user-space)   |----->| libcuda.so    (injected)    |
| NVIDIA Container Toolkit         |      | CUDA libs (from the image)  |
| Docker Engine                    |      | PyTorch / your code         |
+----------------------------------+      +-----------------------------+
```

**Compatibility rule (simplified):** the host driver must be new enough for the CUDA version your container's libraries were built against. Newer drivers run older CUDA apps; the reverse fails. `nvidia-smi` on the host prints "CUDA Version: X.Y" in its header, which is the **maximum CUDA version that driver supports** (it does not mean the toolkit is installed). NVIDIA's CUDA compatibility docs describe forward-compatibility packages for data-centre GPUs; read them if you must run newer CUDA on an old driver.

### 8.6.2 Installing the NVIDIA Container Toolkit on Linux (Ubuntu/Debian)

Prerequisite: an NVIDIA driver already installed on the host (check with `nvidia-smi`). The commands below are from NVIDIA's official install guide (<https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>), fetched while writing this chapter:

```bash
sudo apt-get update && sudo apt-get install -y --no-install-recommends \
   ca-certificates curl gnupg2
```

* Installs prerequisites: certificate authorities, `curl` for downloading, `gnupg2` for verifying signing keys.

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

* First half downloads NVIDIA's package-signing **GPG key** and stores it in a keyring file, so `apt` can verify packages are really from NVIDIA.
* Second half downloads the repository definition, rewrites each `deb https://` line to say "only trust packages signed by that key" (`signed-by=...`), and saves it as an apt source.

```bash
sudo apt-get update
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.20.1-1
sudo apt-get install -y \
    nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

* Installs the four toolkit packages at one pinned version (1.20.1-1 was the version in NVIDIA's guide when I read it; **check the page for the current number**). Pinning keeps the four packages consistent.

Then register the NVIDIA runtime with Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

* `nvidia-ctk` is the toolkit's command-line helper. `runtime configure --runtime=docker` edits `/etc/docker/daemon.json` to add an `nvidia` runtime.
* Restart Docker so the daemon rereads its config. (Restarting Docker restarts running containers unless live-restore is enabled; do this in a quiet moment.)
* For **rootless Docker**, NVIDIA's docs say to pass `--config=$HOME/.config/docker/daemon.json` and restart the user service instead.

Verify:

```bash
docker run --rm --gpus all nvidia/cuda:13.4.3-base-ubuntu24.04 nvidia-smi
```

* `--rm` removes the container afterwards.
* `--gpus all` asks Docker to give the container every GPU.
* `nvidia/cuda:...-base-...` is a small image; `nvidia-smi` is the **NVIDIA System Management Interface**, a tool that prints GPU, driver and memory status. (Substitute any existing `nvidia/cuda` base tag whose CUDA version your driver supports. I could not confirm this exact `base` tag exists, since the Hub page I could read listed other flavours.)

Expected output (shape; numbers vary):

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 5xx.xx.xx        Driver Version: 5xx.xx.xx      CUDA Version: 13.x           |
|-----------------------------------------+------------------------+----------------------|
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|   0  NVIDIA GeForce RTX ...         Off | 00000000:01:00.0   On  |                  N/A |
| ...                                                                                     |
+-----------------------------------------------------------------------------------------+
```

If you see your GPU's name, GPU passthrough works.

Selecting GPUs:

| Flag | Meaning |
|---|---|
| `--gpus all` | Every GPU |
| `--gpus 1` | Any one GPU |
| `--gpus '"device=0,2"'` | Specific GPUs by index (note the nested quotes: Docker parses a comma-separated list) |
| `--gpus 'all,capabilities=compute,utility'` | Everything, but only these driver capabilities |
| `-e NVIDIA_VISIBLE_DEVICES=1` (with `--runtime=nvidia`) | Legacy way to select a GPU through the runtime |

### 8.6.3 CDI: the newer way to expose devices

**CDI (Container Device Interface)** is a standard, cross-runtime way to describe devices (GPUs, FPGAs, etc.) in a spec file, so the container engine can inject them without vendor-specific runtime hooks. With the NVIDIA toolkit you generate a spec:

```bash
sudo nvidia-ctk cdi generate --output=/var/run/cdi/nvidia.yaml
nvidia-ctk cdi list
docker run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

* `cdi generate` inspects the host's GPUs and writes a YAML spec describing them.
* `cdi list` shows the device names available, such as `nvidia.com/gpu=0` and `nvidia.com/gpu=all`.
* `--device nvidia.com/gpu=all` asks Docker to inject all GPUs through CDI (requires a Docker Engine version with CDI support enabled).

**Verification status:** NVIDIA's own install guide text that I could read mentions CDI only for Podman and did not show Docker CDI commands. The Docker CDI flow above, and the claim that toolkit 1.18+ can generate CDI specs just-in-time (needing Docker 26.1+), come from third-party write-ups that I could not confirm against NVIDIA's docs. **Treat CDI-with-Docker as "check the NVIDIA toolkit and Docker docs for your versions"**, and use `--gpus all` as the dependable default.

### 8.6.4 GPUs in Compose

Compose uses the generic "device reservations" syntax:

```yaml
services:
  llm:
    image: vllm/vllm-openai:latest
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

* `deploy.resources.reservations.devices` says "this service needs these devices".
* `driver: nvidia` selects the NVIDIA driver/runtime.
* `count: all` (or a number like `1`) picks how many GPUs; alternatively use `device_ids: ["0", "1"]` for specific ones (you cannot combine `count` and `device_ids`).
* `capabilities: [gpu]` is **required**; it declares what you need the device for.

Newer Compose versions also accept a short form, `gpus: all` directly under the service. Since your Compose is v5.x it should work, but if it errors, use the long form above, which is the long-established syntax.

### 8.6.5 GPUs on Windows with WSL2

Docker Desktop on Windows runs Linux containers inside a **WSL2** (Windows Subsystem for Linux 2) virtual machine. NVIDIA GPUs can be passed into WSL2, and so into containers:

1. Install a current **NVIDIA driver for Windows** (the standard Game Ready/Studio/Data Center driver supports WSL2). Do **not** install a Linux GPU driver inside the WSL distribution; the Windows driver provides the GPU to WSL2.
2. Use Docker Desktop with the WSL2 backend.
3. Test: `docker run --rm --gpus all nvidia/cuda:<tag>-base-ubuntu24.04 nvidia-smi`.

Sources I found disagree on whether the NVIDIA Container Toolkit must be installed separately inside WSL2 when using Docker Desktop (Docker's own integration normally handles it, and a 2026 troubleshooting guide says that when you see "could not select device driver nvidia" you should install the toolkit and run `nvidia-ctk runtime configure --runtime=docker`). **Try the `--gpus all` test first; install the toolkit only if it fails.** Also remember: `--gpus` works with `docker run` and Compose services, **not** during `docker build`; GPU-needing steps cannot run in a `RUN` instruction.

If you have no NVIDIA GPU (as on many laptops), nothing is broken; use CPU images, quantized models via Ollama/llama.cpp, Docker Model Runner, or hosted APIs, and do real GPU work in the cloud (section 8.21).

### 8.6.6 AMD and Intel GPUs

Ollama's docs show AMD ROCm support with `ollama/ollama:rocm` and device flags `--device /dev/kfd --device /dev/dri`. The default Ollama image also bundles Vulkan, which is used automatically when the container can reach the GPU devices (disable with `OLLAMA_VULKAN=0`). These are from Ollama's Docker documentation fetched for this chapter. Docker Model Runner lists AMD ROCm and Vulkan support on Linux with Docker Engine.

### 8.6.7 macOS and Apple Silicon: no GPU inside Linux containers

Docker Desktop on a Mac runs Linux containers in a Linux VM. **Apple's GPU (Metal) is not available inside that VM**, so a containerized PyTorch or Ollama will use the CPU only (slow for big models). What people do instead:

| Approach | How it works |
|---|---|
| Run Ollama **natively** on the Mac (outside Docker) and let containers call it at `http://host.docker.internal:11434` | Native Ollama uses Metal; your app still lives in a container |
| **Docker Model Runner** | On Apple Silicon it runs the inference engine as a host process (so it can use the GPU) while you control it with `docker model ...`. Docker's docs list "Apple Silicon on macOS" as a GPU-supported platform |
| Develop on Mac with a **small model or hosted API**, deploy on amd64 + NVIDIA | Use multi-arch images (section 8.19) |

`host.docker.internal` is a special DNS name that Docker Desktop provides so containers can reach services on the host machine. (On Linux Engine you add `--add-host=host.docker.internal:host-gateway`.)

### 8.6.8 Common GPU errors

| Error message | Likely cause | Fix |
|---|---|---|
| `could not select device driver "" with capabilities: [[gpu]]` | NVIDIA toolkit not installed/registered | Install toolkit, `nvidia-ctk runtime configure`, restart Docker |
| `nvidia-smi: command not found` inside container | Image lacks the utility or capabilities do not include `utility` | Use `nvidia/cuda` base or set `NVIDIA_DRIVER_CAPABILITIES=compute,utility` |
| `CUDA error: no kernel image is available for execution on the device` | Framework build does not support your GPU architecture | Use a newer PyTorch/CUDA build (new GPU generations need new builds) |
| `CUDA driver version is insufficient for CUDA runtime version` | Host driver too old for the container's CUDA | Update driver, or use an older CUDA build |
| `RuntimeError: DataLoader worker ... Bus error` / `shared memory` | `/dev/shm` too small (Docker default 64 MB) | `--shm-size=8g` or `--ipc=host` |
| GPU works with `docker run` but not Compose | Missing `capabilities: [gpu]` or wrong YAML nesting | Use the exact snippet above |
| Container sees the GPU but out-of-memory | Model bigger than VRAM | Smaller/quantized model, tensor parallelism, or bigger GPU |

---

## 8.7 Handling model weights

Models are big, change rarely, and are *data*. You have four strategies:

| Strategy | How | Pros | Cons | Use when |
|---|---|---|---|---|
| **A. Bake into the image** | `RUN hf download org/model` (or `COPY`) during build | Self-contained; starts instantly; works offline/air-gapped; version of code + model locked together | Giant images (GBs), slow pushes/pulls, every code change may rebuild/re-upload layers if ordered badly, registry costs | Edge devices, air-gapped, small models (embedding models of ~100-500 MB), serverless where cold start must be fast and a registry is closer than the internet |
| **B. Download at startup** | App fetches the model on first boot into a cache directory | Small images; flexible model choice via env var | Slow first start; needs internet and credentials at runtime; fails if the hub is down; repeated downloads if no persistent cache | Dev, experiments |
| **C. Mount a volume / cache directory** | `-v hf-cache:/models` + `HF_HOME=/models/hf` | Download once, reuse across containers and restarts; images stay small | Cache must be provisioned (shared disk, PVC, network storage) | **Most common for local and server deployments** |
| **D. Model registry / object storage + init step** | A separate "init" container (or init container in Kubernetes) pulls from S3/GCS/MLflow/HF into a shared volume, then the server starts | Clean separation, access control, versioned models | More moving parts | Production with governed models |

### 8.7.1 Cache directories and environment variables

| Variable | What it controls |
|---|---|
| `HF_HOME` | Root of Hugging Face cache (models, datasets, tokens). Default `~/.cache/huggingface` |
| `HF_HUB_CACHE` (older: `TRANSFORMERS_CACHE`) | Where model files specifically go |
| `HF_TOKEN` | Access token for gated/private models (pass as a secret, never bake in) |
| `HF_HUB_OFFLINE=1` | Forbid network access; fail if the model is not already cached (great for production safety) |
| `TORCH_HOME` | Where `torch.hub` stores weights |
| `OLLAMA_MODELS` | Where Ollama stores models (default `/root/.ollama/models` in the official image) |

### 8.7.2 Baking a small model into the image safely (build secrets)

If you must bake in a gated model, the token must not end up in a layer. Use a **build secret** (the BuildKit way, covered in [Chapter 10](10-security.md)):

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim AS model
RUN pip install --no-cache-dir "huggingface_hub[cli]"
RUN --mount=type=secret,id=hf_token \
    HF_TOKEN="$(cat /run/secrets/hf_token)" \
    hf download sentence-transformers/all-MiniLM-L6-v2 --local-dir /models/minilm

FROM python:3.12-slim AS runtime
COPY --from=model /models/minilm /models/minilm
# ... rest of the app ...
```

* `--mount=type=secret,id=hf_token` makes a secret file available at `/run/secrets/hf_token` **only during this one `RUN`**; it is never written into a layer.
* `hf download <repo> --local-dir <dir>` fetches the model files with the Hugging Face CLI. (The CLI command is `hf` in current `huggingface_hub` versions; older ones used `huggingface-cli download`.) `all-MiniLM-L6-v2` is a small, public embedding model, used here as an example; public models do not need a token, so remove the secret mount for them.
* The final stage copies only the model directory. The hub client and token never reach the runtime image.

Build with:

```bash
docker build --secret id=hf_token,env=HF_TOKEN -t embed-api:1 .
```

* `--secret id=hf_token,env=HF_TOKEN` feeds the value of your shell variable `HF_TOKEN` to the build as the secret `hf_token`.

### 8.7.3 Mounting a cache volume (the common pattern)

```bash
docker volume create hf-cache
docker run --rm --gpus all \
  -v hf-cache:/models/hf \
  -e HF_HOME=/models/hf \
  -e HF_TOKEN \
  my-gpu-app:1
```

* `docker volume create hf-cache` makes a named volume managed by Docker.
* `-v hf-cache:/models/hf` mounts it at the path the app uses as its cache.
* `-e HF_TOKEN` (with no `=value`) forwards the variable **from your current shell** without writing it into the command history.

First run downloads (slow); later runs reuse the cache.

### 8.7.4 Init container pattern in Compose

Download once, then start the server:

```yaml
services:
  model-init:
    image: python:3.12-slim
    command: >
      sh -c "pip install --no-cache-dir 'huggingface_hub[cli]' &&
             hf download Qwen/Qwen3-0.6B"
    environment:
      HF_HOME: /models/hf
    volumes:
      - hf-cache:/models/hf
    restart: "no"

  llm:
    image: vllm/vllm-openai:latest
    command: ["--model", "Qwen/Qwen3-0.6B"]
    environment:
      HF_HOME: /models/hf
      HF_HUB_OFFLINE: "1"
    volumes:
      - hf-cache:/models/hf
    depends_on:
      model-init:
        condition: service_completed_successfully

volumes:
  hf-cache:
```

* `restart: "no"` lets the init container exit and stay exited.
* `condition: service_completed_successfully` makes `llm` wait until `model-init` has finished with exit code 0.
* `HF_HUB_OFFLINE: "1"` ensures the server never silently downloads anything at runtime.
* (The `llm` service also needs the GPU reservation block from section 8.6.4; omitted here for brevity. `Qwen/Qwen3-0.6B` is the small model used in vLLM's own Docker documentation example.)

### 8.7.5 Image size and pull-time arithmetic

A very rough planning formula: pulling a 20 GB image over a 1 Gbit/s link (about 100-120 MB/s usable) takes roughly **3-4 minutes** on a good day, longer on a cold node or a slower registry. If your autoscaler starts new GPU nodes during a traffic spike, that pull time is your **cold-start delay**. Strategies: keep weights out of the image (volumes), use registry mirrors/caches near the cluster, pre-pull images onto GPU nodes, use image streaming features where your cloud offers them, and order Dockerfile layers so the huge, rarely-changing layers (libraries, model) come first and the small, frequently-changing layers (your code) come last.

---

## 8.8 Running local LLMs in Docker

**LLM (Large Language Model).** A neural network trained on lots of text that predicts the next token (piece of a word) and so can write, summarize, answer and code. You can use hosted ones (via an API) or run **open-weight** ones yourself. Reasons to self-host: privacy, cost at high volume, offline use, customization, learning.

**Quantization.** Storing the model's weights with fewer bits (for example 4-bit instead of 16-bit) so it needs less memory and runs faster at some quality cost. Formats you will meet: **GGUF** (the file format used by llama.cpp/Ollama), **safetensors** (the standard Hugging Face weights format), **AWQ/GPTQ** (GPU-oriented quantization schemes).

**OpenAI-compatible API.** Most local servers expose endpoints like `POST /v1/chat/completions` that mimic OpenAI's API, so the same client code can talk to a hosted service or your local container just by changing the base URL.

### 8.8.1 Ollama in Docker

**Ollama** is a user-friendly runtime that downloads and serves models (llama.cpp under the hood) with a simple CLI and an HTTP API on port 11434. Commands below are from Ollama's official Docker docs:

CPU only:

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

* `-d` runs in background.
* `-v ollama:/root/.ollama` stores models in a named volume `ollama` (without it, models vanish with the container and you would re-download GBs).
* `-p 11434:11434` publishes the API. (On shared machines prefer `127.0.0.1:11434:11434`, because **the Ollama API has no authentication**.)
* `--name ollama` gives the container a friendly name for later commands.
* `ollama/ollama` is the image.

NVIDIA GPU (after installing the toolkit per 8.6.2):

```bash
docker run -d --gpus=all -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

AMD ROCm:

```bash
docker run -d --device /dev/kfd --device /dev/dri -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama:rocm
```

Run a model:

```bash
docker exec -it ollama ollama run llama3.2
```

* `docker exec -it ollama` runs a command in the running container interactively (`-i` keep stdin open, `-t` allocate a terminal).
* `ollama run llama3.2` downloads the model on first use, then opens a chat prompt. `llama3.2` is one example from Ollama's model library; browse the library for others.

Call the API from the host:

```bash
curl http://localhost:11434/api/generate -d '{"model":"llama3.2","prompt":"Why is the sky blue?","stream":false}'
```

* `/api/generate` is Ollama's native endpoint. `"stream": false` returns one JSON object instead of a stream of chunks.
* Ollama also exposes an OpenAI-compatible endpoint at `/v1/` (for example `http://localhost:11434/v1/chat/completions`).

#### Ollama + Open WebUI in Compose (a self-hosted chat site)

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama:/root/.ollama
    ports:
      - "127.0.0.1:11434:11434"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped

  webui:
    image: ghcr.io/open-webui/open-webui:main
    environment:
      OLLAMA_BASE_URL: http://ollama:11434
    volumes:
      - webui:/app/backend/data
    ports:
      - "127.0.0.1:3000:8080"
    depends_on:
      - ollama
    restart: unless-stopped

volumes:
  ollama:
  webui:
```

* The `ollama` service is as above; delete the whole `deploy:` block on a machine without an NVIDIA GPU.
* `OLLAMA_BASE_URL: http://ollama:11434` tells Open WebUI to reach Ollama by its Compose service name.
* `"127.0.0.1:3000:8080"`: Open WebUI listens on container port 8080; we publish it on host port 3000, loopback only.
* `ghcr.io/open-webui/open-webui:main` is the image name used in the project's docs; `main` is a moving tag, so pin a release tag for anything serious.
* Pull a model once the stack is up: `docker compose exec ollama ollama pull llama3.2`, then open <http://localhost:3000>.

On Windows/macOS without a GPU, expect slow generation for anything bigger than a small model; start with small models (1-4B parameters, quantized).

**Pre-pulling models automatically.** A common trick is an entrypoint wrapper or a one-shot service:

```yaml
  ollama-pull:
    image: ollama/ollama:latest
    depends_on:
      - ollama
    environment:
      OLLAMA_HOST: http://ollama:11434
    entrypoint: ["ollama", "pull", "llama3.2"]
    restart: "no"
```

* The `ollama` CLI inside this container talks to the server at `OLLAMA_HOST` (the CLI is a client, so it does not need a GPU).
* It exits after the pull. (There is a small race if the server is not ready yet; add a retry loop or a healthcheck on `ollama` if you see it.)

### 8.8.2 Docker Model Runner (DMR)

**Docker Model Runner** lets you pull, run and serve AI models through the Docker CLI, storing models as **OCI artifacts** (the same packaging standard as images) in Docker Hub or any compatible registry. From Docker's docs (<https://docs.docker.com/ai/model-runner/>):

* It exposes **OpenAI-compatible and Ollama-compatible** APIs.
* Backends: **llama.cpp** (default, runs everywhere, GGUF), **vLLM** (safetensors, NVIDIA GPUs on Linux x86_64 or Windows via WSL2), and **Diffusers** (image generation, NVIDIA on Linux).
* GPU support listed: NVIDIA CUDA (Windows amd64 with driver 576.57+; Linux with driver 575.57.08+), Apple Silicon on macOS, Qualcomm Adreno on Windows arm64, AMD ROCm and Vulkan on Linux with Docker Engine.
* Models come from Docker Hub (the `ai/` namespace), other OCI registries, or Hugging Face.
* The docs state plainly: **"The Model Runner API is not authenticated."** Do not expose it to a network.

Typical commands (from my knowledge of the `docker model` CLI; the docs page I fetched documented only `docker model configure`, so run `docker model --help` on your machine for the authoritative list):

```bash
docker model pull ai/smollm2
docker model list
docker model run ai/smollm2 "Say hello in one sentence."
```

* `pull` downloads the model (an OCI artifact) into the local model store.
* `list` shows downloaded models.
* `run` runs a one-shot prompt (omit the prompt for an interactive chat).
* `ai/smollm2` is a small model published under Docker's `ai/` namespace (this name also appears in Docker's Compose docs).

Context size can be configured, e.g. `docker model configure --context-size 8192 ai/qwen2.5-coder` (documented example).

**Model Runner with Compose: the `models` top-level element.** Docker Compose (v2.38 or later, so your v5.x qualifies) lets you declare models as dependencies right next to your services. From the docs (<https://docs.docker.com/ai/compose/models-and-compose/>):

```yaml
services:
  app:
    image: my-app
    models:
      - llm
      - embedding-model

models:
  llm:
    model: ai/smollm2
  embedding-model:
    model: ai/all-minilm
```

* Top-level `models:` defines each model; `model:` is the OCI artifact name Compose pulls and runs via Model Runner.
* Under the service, `models:` binds them. Compose then **injects environment variables** named after the model: `LLM_URL`, `LLM_MODEL`, `EMBEDDING_MODEL_URL`, `EMBEDDING_MODEL_MODEL`. Your app reads those to know where to call and which model name to send.
* **Long syntax** lets you pick your own variable names:

```yaml
services:
  app:
    image: my-app
    models:
      llm:
        endpoint_var: AI_MODEL_URL
        model_var: AI_MODEL_NAME

models:
  llm:
    model: ai/smollm2
    context_size: 4096
    runtime_flags:
      - "--a-flag"
```

* `endpoint_var` / `model_var` rename the injected variables.
* `context_size` is the maximum number of tokens the model can consider at once (keep it as small as your use case allows, since bigger values need more memory).
* `runtime_flags` passes raw flags to the inference engine. For embedding models served on `/v1/embeddings` the docs say to add `--embeddings` there.

**Provenance note:** one secondary source claimed the older `provider:` service syntax for models is deprecated; another said it is still supported. The official page I read documents only `models:`, so use that for new work.

Why it is attractive: one `docker compose up` brings up your app **and** its local model without a separate Ollama container, and the same Compose file can be adapted for cloud model backends.

### 8.8.3 llama.cpp server

**llama.cpp** is the C/C++ engine behind many local-LLM tools, running GGUF models efficiently on CPUs and many GPUs. Its project publishes container images with an HTTP server (OpenAI-compatible). Sketch:

```bash
docker run --rm -p 8080:8080 -v "$PWD/models:/models" \
  ghcr.io/ggml-org/llama.cpp:server \
  -m /models/my-model.gguf --host 0.0.0.0 --port 8080
```

* `-v "$PWD/models:/models"` mounts a folder with a `.gguf` file you downloaded.
* After the image name come **arguments to the server**: `-m` model path, `--host 0.0.0.0` listen on all interfaces inside the container, `--port 8080`.
* GPU variants of the image exist (look for tags such as `server-cuda`); **I did not verify current tag names, so check the llama.cpp repository's Docker documentation.**

### 8.8.4 vLLM (high-throughput GPU serving)

**vLLM** is an inference engine focused on throughput (continuous batching and efficient KV-cache memory management, "PagedAttention") for serving many concurrent users on GPUs. Official docs command (<https://docs.vllm.ai/en/stable/deployment/docker/>):

```bash
docker run --runtime nvidia --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HF_TOKEN=$HF_TOKEN" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model Qwen/Qwen3-0.6B
```

| Piece | Meaning |
|---|---|
| `--runtime nvidia --gpus all` | Use the NVIDIA runtime and expose all GPUs |
| `-v ~/.cache/huggingface:/root/.cache/huggingface` | Share your host's Hugging Face cache so weights are downloaded once |
| `--env "HF_TOKEN=$HF_TOKEN"` | Pass your token for gated models (shell expands `$HF_TOKEN`) |
| `-p 8000:8000` | Publish the OpenAI-compatible API |
| `--ipc=host` | Share the host's shared-memory namespace (PyTorch tensor-parallel workers need lots of shared memory). `--shm-size` is the alternative |
| `vllm/vllm-openai:latest` | The OpenAI-compatible server image. Pin a version for production |
| `--model Qwen/Qwen3-0.6B` | An argument to vLLM: which model to load (a small example from vLLM's docs) |

Test it: `curl http://localhost:8000/v1/models`. Then any OpenAI client works with `base_url="http://localhost:8000/v1"`.

(vLLM's docs also describe building the image from source, ARM64 builds for Grace-Hopper/Blackwell platforms, and note that on an x86_64 host you need QEMU handlers registered via a `--privileged` helper container for cross-building. Using `--privileged` for that one-off step is acceptable; never use it for serving.)

### 8.8.5 Hugging Face TGI

**Text Generation Inference (TGI)** is Hugging Face's inference server (`ghcr.io/huggingface/text-generation-inference`). **Status check:** the Hugging Face Inference Endpoints docs say TGI entered **maintenance mode** (as of Dec 11, 2025), accepting only minor bug fixes and docs improvements, and they recommend vLLM or SGLang for new deployments. Some third-party sources claim the repository was archived in March 2026; I could not confirm that from official sources. Existing TGI deployments keep working, but for new projects prefer vLLM (or SGLang).

### 8.8.6 NVIDIA Triton Inference Server

**Triton** is NVIDIA's general-purpose model server: it serves many model types (TensorRT, ONNX, PyTorch, Python backends, and LLM backends including vLLM and TensorRT-LLM) with dynamic batching and multi-model scheduling. Images come from NVIDIA's NGC registry with date-style tags:

```bash
docker run --gpus=all --rm -p 8000:8000 -p 8001:8001 -p 8002:8002 \
  -v "$PWD/model_repository:/models" \
  nvcr.io/nvidia/tritonserver:26.09-py3 \
  tritonserver --model-repository=/models
```

* Tag `26.09-py3` is the YY.MM release naming (the 26.09 release was the newest I found in the project's release notes; variants such as `26.09-vllm-python-py3` exist for LLM backends).
* Ports 8000/8001/8002 are HTTP, gRPC and Prometheus metrics respectively (Triton defaults).
* `-v ...:/models` provides the **model repository**, a directory layout Triton scans: `model_name/1/model.onnx` plus a `config.pbtxt`.
* `tritonserver --model-repository=/models` starts the server.
* NGC images are huge; verify the exact tag on <https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver> (one catalog snapshot I saw was stale, so do not trust a single page).

### 8.8.7 Which one should I choose?

```
                      Do you need to serve MANY concurrent users on NVIDIA GPUs?
                                   |                       |
                                  YES                      NO
                                   |                       |
               vLLM (or SGLang);                  Is it for a dev laptop / quick local use?
               Triton if you need                         |               |
               multi-model/multi-framework              YES              NO (small private service)
                                                          |               |
                                       Ollama or Docker Model Runner    llama.cpp server or Ollama
                                       (Mac: native Ollama / DMR for
                                        Metal)
```

| Tool | Best for | Ease | Throughput | GPU needed |
|---|---|---|---|---|
| Ollama | Local dev, demos, small teams | Very easy | Moderate | Optional |
| Docker Model Runner | Docker-native workflow, Compose `models` | Very easy | Moderate | Optional |
| llama.cpp server | Lightweight, CPU/edge, GGUF control | Medium | Moderate | Optional |
| vLLM | Production serving at scale | Medium | Excellent | Yes (NVIDIA; other hardware has separate builds) |
| TGI | Existing deployments only | Medium | Good | Yes |
| Triton | Mixed model portfolios in enterprise | Harder | Excellent | Usually |

---

## 8.9 Calling hosted LLM APIs (keys and base URLs)

Most production AI apps call a **hosted** model (from a cloud AI provider). The rules are simple and important.

### 8.9.1 Never put API keys in an image

API keys are bearer credentials: anyone holding the key can spend your money. An image is shareable and inspectable (`docker history`, layer extraction). So:

| Do NOT | Why |
|---|---|
| `ENV OPENAI_API_KEY=sk-...` in the Dockerfile | Visible in `docker inspect`, `docker history`, and every layer |
| `ARG API_KEY` then `RUN ... $API_KEY` | `ARG` values are recorded in image metadata/history |
| `COPY .env /app/.env` | The file ends up in a layer forever |
| Commit `.env` to git | Leaks via repo history (see [Chapter 10](10-security.md): rotation is the only real fix) |

Do:

| Do | How |
|---|---|
| **Runtime env var from outside** | `docker run -e API_KEY` (forwards from your shell) or Compose `environment:`/`env_file:` pointing to a git-ignored file |
| **Docker/Compose secrets (files)** | Mounted at `/run/secrets/<name>`; app reads the file |
| **Secret manager in prod** | AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, Kubernetes Secrets (ideally with external secret operators) |

Compose example with a file-based secret:

```yaml
services:
  api:
    image: rag-api:0.1.0
    environment:
      LLM_BASE_URL: https://api.example.com/v1
      LLM_API_KEY_FILE: /run/secrets/llm_api_key
    secrets:
      - llm_api_key

secrets:
  llm_api_key:
    file: ./secrets/llm_api_key.txt
```

* `secrets:` under the service grants it access to the named secret.
* The top-level `secrets:` maps the name to a local file (git-ignore the `secrets/` folder).
* The secret appears **inside the container** as the file `/run/secrets/llm_api_key`, which is a read-only mount of your file (plain Compose bind-mounts the file; Docker Swarm keeps secrets in memory), not an environment variable, so the value does not show in `docker inspect` output.
* The app reads `LLM_API_KEY_FILE` to find the file. Many official images support the `*_FILE` convention; for your own code it is two lines:

```python
import os
from pathlib import Path

def read_secret(name: str) -> str:
    path = os.getenv(f"{name}_FILE")
    if path:
        return Path(path).read_text().strip()
    return os.environ[name]          # fallback for local dev
```

### 8.9.2 The OpenAI-compatible base-URL pattern

Because so many servers (vLLM, Ollama, llama.cpp, Docker Model Runner, LiteLLM and many cloud providers) imitate the OpenAI Chat Completions API, you can write provider-agnostic code:

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.environ["LLM_BASE_URL"],          # https://api.example.com/v1  OR  http://ollama:11434/v1
    api_key=os.environ.get("LLM_API_KEY", "not-needed-locally"),
)

reply = client.chat.completions.create(
    model=os.environ["LLM_MODEL"],                # name that the server understands
    messages=[{"role": "user", "content": "Summarize Docker in one sentence."}],
)
print(reply.choices[0].message.content)
```

* `base_url` decides **where** requests go; switching from a hosted service to a local container is a one-variable change.
* Local servers usually ignore the API key but the client library requires *some* string.
* `model` is whatever name the target server understands (for Ollama, `llama3.2`; for vLLM, the Hugging Face name you launched it with; for hosted services, their model ID; look it up in that provider's docs rather than guessing).
* The official `openai` Python package also reads `OPENAI_BASE_URL` and `OPENAI_API_KEY` from the environment by default, so you can sometimes avoid any code change.

Other providers have their own SDKs and often their own key variable names (for example, Anthropic's SDK reads `ANTHROPIC_API_KEY`). The container-level rule is identical: key in at runtime, never in the image.

### 8.9.3 Network considerations for outbound calls

* Containers can reach the internet by default via NAT, so hosted APIs just work. In locked-down environments you need an egress proxy; pass `HTTPS_PROXY`/`NO_PROXY` env vars.
* Set **timeouts and retries** in your client; LLM calls can take tens of seconds. Do not let a hung upstream hold all your workers.
* Rate limits: add a cache (Redis) and a queue for bursts.
* **LiteLLM** (an open-source proxy/gateway that presents one OpenAI-compatible endpoint in front of many providers) is a popular container to centralize keys, budgets and fallbacks; run it as a service so only it holds provider keys.

---

## 8.10 RAG in Compose: FastAPI + Qdrant + embeddings + LLM

### 8.10.1 Terms, plainly

* **Embedding.** A list of numbers (a *vector*, often hundreds to thousands of numbers long) that represents the *meaning* of a piece of text (or an image). Texts with similar meaning produce vectors that are close together.
* **Embedding model.** A small neural network that converts text to an embedding (for example `all-MiniLM-L6-v2` produces 384 numbers per text; verify the dimension for whichever model you choose, because the vector DB collection must match it).
* **Vector database.** A database built to store embeddings and answer "find the *k* vectors nearest to this one" quickly (using approximate nearest-neighbour indexes such as HNSW). Examples: **Qdrant**, **Chroma**, **Milvus**, **Weaviate**, and **pgvector** (an extension that adds a vector type to PostgreSQL).
* **RAG (Retrieval-Augmented Generation).** A pattern to make an LLM answer from *your* documents without retraining it: (1) **ingest**: split documents into chunks, embed each chunk, store in the vector DB; (2) **retrieve**: embed the user's question, fetch the nearest chunks; (3) **generate**: paste those chunks into the prompt ("use only this context") and ask the LLM.
* **Chunking.** Splitting long documents into passages (a few hundred tokens each) because embeddings and prompts work on limited-size text.

```
INGEST (offline/worker):   documents -> chunk -> embed -> store in vector DB
QUERY (online/API):        question -> embed -> vector DB top-k chunks -> build prompt -> LLM -> answer (+ sources)
```

### 8.10.2 The stack

```yaml
name: rag-demo

services:
  api:
    build: ./rag-api
    environment:
      LLM_BASE_URL: ${LLM_BASE_URL:-http://ollama:11434/v1}
      LLM_MODEL: ${LLM_MODEL:-llama3.2}
      EMBED_MODEL_PATH: /models/minilm
      QDRANT_URL: http://qdrant:6333
      REDIS_URL: redis://redis:6379/0
      LLM_API_KEY_FILE: /run/secrets/llm_api_key
    secrets:
      - llm_api_key
    ports:
      - "127.0.0.1:8000:8000"
    depends_on:
      qdrant:
        condition: service_healthy
      redis:
        condition: service_started
    read_only: true
    tmpfs:
      - /tmp
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
    restart: unless-stopped

  worker:
    build: ./rag-api
    command: ["python", "-m", "app.worker"]
    environment:
      QDRANT_URL: http://qdrant:6333
      REDIS_URL: redis://redis:6379/0
      EMBED_MODEL_PATH: /models/minilm
    depends_on:
      - redis
      - qdrant
    restart: unless-stopped

  qdrant:
    image: qdrant/qdrant:latest
    volumes:
      - qdrant_data:/qdrant/storage
    ports:
      - "127.0.0.1:6333:6333"
    healthcheck:
      test: ["CMD-SHELL", "bash -c ':> /dev/tcp/127.0.0.1/6333' || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--save", "", "--appendonly", "no"]
    restart: unless-stopped

  ollama:
    image: ollama/ollama:latest
    volumes:
      - ollama:/root/.ollama
    restart: unless-stopped

volumes:
  qdrant_data:
  ollama:

secrets:
  llm_api_key:
    file: ./secrets/llm_api_key.txt
```

Explanation of the interesting parts:

| Part | Meaning |
|---|---|
| `${LLM_BASE_URL:-http://ollama:11434/v1}` | Compose variable substitution with a default: use `LLM_BASE_URL` from your environment/`.env` if set; otherwise use the local Ollama container. One switch toggles local vs hosted LLM. |
| `read_only: true` + `tmpfs: [/tmp]` | The container's root filesystem is read-only (attackers and bugs cannot write there), with an in-memory writable `/tmp`. Make sure the app does not need to write elsewhere. (Details in [Chapter 10](10-security.md).) |
| `cap_drop: [ALL]` | Drop every Linux capability (special root-like privileges). A web API needs none. |
| `security_opt: [no-new-privileges:true]` | Processes cannot gain extra privileges via setuid binaries. |
| `worker` reuses the same build with a different `command` | One image, two roles (API and background worker), a very common pattern. |
| `qdrant` image, `/qdrant/storage` volume | Qdrant stores its collections there; REST on 6333 (gRPC on 6334, not published here). |
| `qdrant` healthcheck with `/dev/tcp` | A pure-bash TCP probe so we do not rely on `curl` being in the image (if your Qdrant image tag lacks bash, switch to a different probe or drop the healthcheck; verify against the tag you pin). |
| `redis` with `--save "" --appendonly no` | Run Redis as a pure cache: no persistence. If you use Redis as a job **queue**, enable persistence instead, or you lose queued jobs on restart. |
| No `ports` on `redis`/`ollama` | They are only reachable from inside the Compose network. **Smaller exposed surface is safer.** |
| `EMBED_MODEL_PATH: /models/minilm` | Model baked into the image (see 8.7.2) or provided by a volume. |

Alternatives for the vector store:

| Choice | Image | When |
|---|---|---|
| **Qdrant** | `qdrant/qdrant` | Dedicated, fast, good filtering, easy |
| **pgvector** | `pgvector/pgvector:pg17` (Postgres with the extension) | You already use Postgres; want SQL joins, transactions and one database to back up |
| **Chroma** | `chromadb/chroma` (port 8000) | Prototyping, notebooks |
| **Milvus / Weaviate** | their official images/compose files | Large-scale or advanced features |

*(The `pgvector/pgvector` and `chromadb/chroma` image names are from my knowledge of these projects; confirm tags on Docker Hub before use.)*

With the **Docker Model Runner** you could replace the `ollama` service with a `models:` block (section 8.8.2) and read `LLM_URL`/`LLM_MODEL` injected by Compose.

### 8.10.3 Ingest and query sketch (for the API)

```python
# ingest (worker)
chunks = splitter.split(text)                       # list[str]
vectors = embedder.encode(chunks)                   # shape (n, 384) for MiniLM
qdrant.upsert("docs", points=[
    {"id": i, "vector": v.tolist(), "payload": {"text": c, "source": src}}
    for i, (c, v) in enumerate(zip(chunks, vectors))
])

# query (API)
qv = embedder.encode([question])[0].tolist()
hits = qdrant.search("docs", query_vector=qv, limit=4)
context = "\n\n".join(h.payload["text"] for h in hits)
prompt = f"Answer using only the context.\n\nContext:\n{context}\n\nQuestion: {question}"
```

* `upsert` inserts or updates points (vector + payload metadata).
* `search ... limit=4` returns the 4 nearest chunks.
* Then you send `prompt` to the LLM via the client from 8.9.2. (Exact client method names vary between `qdrant-client` versions; this is a conceptual sketch, and [Chapter 9](09-hands-on-ai-app-walkthrough.md) builds a working version.)

---

## 8.11 MCP servers in containers

**MCP (Model Context Protocol)** is an open protocol that lets AI assistants/agents call external **tools** and read external **resources** through a standard interface. An **MCP server** exposes tools (for example "query this database", "create a GitHub issue", "read this folder"). An **MCP client** is the AI application (an IDE assistant, a desktop chatbot, your agent).

Why containerize MCP servers? They are third-party code that you let an AI drive, often with credentials. A container gives you:

* **Dependency isolation** (no Node/Python version juggling).
* **A boundary** around what it can touch: no host filesystem unless you mount it, restricted network if you choose.
* **Reproducible installs** (pin an image tag or digest).

### 8.11.1 stdio MCP servers via `docker run -i`

Many MCP servers talk over **stdio** (standard input/output): the client launches the server as a child process and exchanges JSON messages through its stdin/stdout. With Docker, the "process" is `docker run -i --rm ...`. A typical client configuration looks like this (illustrative structure; each client documents its own file location):

```json
{
  "mcpServers": {
    "github": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "-e", "GITHUB_PERSONAL_ACCESS_TOKEN",
        "ghcr.io/github/github-mcp-server"
      ],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "<set-in-your-client-not-in-git>" }
    }
  }
}
```

* `command`/`args`: the client runs `docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN <image>`.
* `-i` keeps stdin open (essential for stdio), but **no `-t`** (a TTY corrupts the JSON stream).
* `--rm` cleans up the container when the client disconnects.
* `-e GITHUB_PERSONAL_ACCESS_TOKEN` with no value forwards the variable from the client's `env` block into the container.
* Use a **least-privilege token** (only the scopes the tool needs) because the AI can invoke whatever the token allows.
* The image name here follows the GitHub MCP server project's published container; verify the current image name and flags in its README before use.

For servers that touch files, mount only what is needed and prefer read-only:

```
"args": ["run","-i","--rm","-v","/home/me/project:/workspace:ro","some/mcp-filesystem-server","/workspace"]
```

* `:ro` makes the mount read-only, so the AI cannot modify your files.

### 8.11.2 Docker MCP Toolkit, MCP Gateway and Catalog

Docker ships a **MCP Catalog** (browsable list of containerized MCP servers), an **MCP Toolkit** (a management UI inside Docker Desktop, version 4.62 or later per the docs) and an **MCP Gateway** (a single endpoint that your AI client connects to; it launches and routes to the servers you enabled). From Docker's docs (<https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/>):

* Servers are grouped in **profiles**, named collections you can switch per project.
* Each tool runs in **its own container**, with limits of **1 CPU and 2 GB of memory**.
* Servers have **no host filesystem access by default**; you grant mounts explicitly.
* Official `mcp/` images in the catalog are **built and signed by Docker and include an SBOM**.
* Requests/responses containing secrets are blocked at runtime (per Docs).
* OAuth flows for services like GitHub are handled by the Toolkit, which stores credentials.
* Two CLI commands the page documents: `docker mcp gateway run --profile <name>` and `docker mcp client connect vscode --profile <name>`.

**Security thinking for MCP (important):** a tool server is a *remote-control surface for an LLM*. Prompt injection (malicious text in a web page or document that tells the model to misuse its tools) is a real risk. Containers reduce blast radius, but also use least-privilege tokens, read-only mounts, network restrictions, and human approval for dangerous actions.

---

## 8.12 Agents and sandboxing code execution

An **AI agent** is a loop in which an LLM decides on actions (call a tool, run code, browse), you execute them, and feed results back. Code execution is the most powerful and most dangerous tool: you are running **code written by a model** (which may have been steered by prompt injection or just be wrong). Never run it on your host or in your API container.

### 8.12.1 A hardened, disposable sandbox per execution

```bash
docker run --rm \
  --network none \
  --read-only \
  --tmpfs /tmp:rw,size=64m,noexec \
  --memory 256m --memory-swap 256m \
  --cpus 0.5 \
  --pids-limit 64 \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --user 65534:65534 \
  -v "$PWD/job:/work:ro" \
  python:3.12-slim \
  timeout 10 python /work/script.py
```

| Flag | What it does |
|---|---|
| `--rm` | Delete the container immediately afterward (disposable) |
| `--network none` | No network at all (no data exfiltration, no downloads). Add a restricted network only if essential |
| `--read-only` | Root filesystem is read-only |
| `--tmpfs /tmp:rw,size=64m,noexec` | A small in-memory scratch space; `noexec` forbids running binaries from it |
| `--memory 256m --memory-swap 256m` | Hard memory cap; setting swap equal to memory disables swap use (prevents memory bombs) |
| `--cpus 0.5` | At most half a CPU core |
| `--pids-limit 64` | At most 64 processes/threads (stops fork bombs) |
| `--cap-drop ALL` | No special privileges |
| `--security-opt no-new-privileges:true` | Block privilege escalation through setuid programs |
| `--user 65534:65534` | Run as `nobody` |
| `-v "$PWD/job:/work:ro"` | Provide the code as a read-only mount |
| `timeout 10 python ...` | Kill the script after 10 seconds (wall-clock limit enforced inside) |

A container is **not a perfect boundary**: all containers share the host kernel, so a kernel bug can allow escape. For running truly untrusted code from the public internet, add a stronger layer: a user-space kernel such as **gVisor** (`--runtime=runsc` once installed) or microVMs such as **Firecracker/Kata Containers**, or use a managed code-sandbox service. See [Chapter 10](10-security.md).

### 8.12.2 Driving it from Python (the Docker SDK)

```python
import docker

client = docker.from_env()

def run_untrusted(code: str) -> str:
    out = client.containers.run(
        "python:3.12-slim",
        ["python", "-c", code],
        network_disabled=True,
        mem_limit="256m",
        nano_cpus=500_000_000,      # 0.5 CPU
        pids_limit=64,
        cap_drop=["ALL"],
        security_opt=["no-new-privileges:true"],
        read_only=True,
        user="65534:65534",
        remove=True,
        stdout=True, stderr=True,
    )
    return out.decode()
```

* `docker.from_env()` connects to the local Docker daemon (via the socket).
* Each argument mirrors a `docker run` flag from the table above.
* **Warning:** giving your *API* container access to the Docker socket to do this is dangerous (the socket is root-equivalent, see [Chapter 10](10-security.md)). Safer designs: run the sandbox runner as a **separate, isolated service on a separate host/VM**, or use a rootless Docker daemon dedicated to sandboxes, or a managed sandbox service.

---

## 8.13 Jupyter in Docker and Dev Containers

**Jupyter** notebooks mix code, output and notes; they are the data scientist's workbench. Running them in a container gives a clean, reproducible lab.

The community **Jupyter Docker Stacks** images (hosted on Quay since 2023) bundle common stacks:

```bash
docker run --rm -p 127.0.0.1:8888:8888 \
  -v "$PWD/work:/home/jovyan/work" \
  quay.io/jupyter/scipy-notebook:latest
```

* `quay.io/jupyter/scipy-notebook` is a ready image with Jupyter plus numpy/pandas/scipy/matplotlib. Other flavours exist (`minimal-notebook`, `pytorch-notebook`, `tensorflow-notebook`, `datascience-notebook`). Pin a dated tag for reproducibility.
* The container prints a URL containing a **login token** in its logs; open that URL (it uses `127.0.0.1:8888` because we published to loopback only).
* `-v "$PWD/work:/home/jovyan/work"` mounts a host folder so your notebooks survive; `jovyan` is the image's default user name.
* **Do not publish Jupyter on `0.0.0.0` without authentication on a shared network:** it provides a code-execution shell.

GPU variant: add `--gpus all` and use a CUDA-enabled image (for example the `pytorch-notebook` CUDA variants; check the Jupyter Docker Stacks docs for tag names).

**Dev Containers** (the `devcontainer.json` standard supported by VS Code, JetBrains IDEs and GitHub Codespaces) let your *editor* run inside a container so everyone has identical tools:

```json
{
  "name": "ml-dev",
  "build": { "dockerfile": "Dockerfile" },
  "runArgs": ["--gpus=all", "--shm-size=8g"],
  "mounts": ["source=hf-cache,target=/models/hf,type=volume"],
  "containerEnv": { "HF_HOME": "/models/hf" },
  "customizations": {
    "vscode": { "extensions": ["ms-python.python", "ms-toolsai.jupyter"] }
  },
  "postCreateCommand": "uv sync"
}
```

* `build.dockerfile` builds the dev image.
* `runArgs` passes raw `docker run` flags (GPU and shared memory).
* `mounts` attaches the model-cache volume.
* `customizations.vscode.extensions` installs editor extensions inside the container.
* `postCreateCommand` runs once after creation to install project dependencies.
* The key idea: a new teammate opens the folder, clicks "Reopen in Container", and gets the exact toolchain.

---

## 8.14 Training jobs in containers

Training is a **batch job**: start, run for minutes-to-days, write results, exit. Containers suit this well.

```bash
docker run --rm --gpus all \
  --shm-size=16g \
  -v "$PWD/data:/data:ro" \
  -v "$PWD/runs:/runs" \
  -v hf-cache:/models/hf -e HF_HOME=/models/hf \
  -e MLFLOW_TRACKING_URI=http://mlflow:5000 \
  my-train-image:2026-10 \
  python train.py --data /data --out /runs/exp42 --epochs 10
```

| Piece | Why |
|---|---|
| `--gpus all` | Training needs the GPU(s) |
| `--shm-size=16g` | PyTorch's `DataLoader(num_workers>0)` passes batches between processes via shared memory (`/dev/shm`). Docker's default is only **64 MB**, causing "Bus error"/"unable to write to file </torch_...>" crashes. Set several GB. |
| `--ipc=host` (alternative) | Shares the host's IPC namespace, removing the limit; commonly used for multi-GPU training with NCCL. It reduces isolation, so prefer `--shm-size` unless you need it |
| `-v $PWD/data:/data:ro` | Datasets mounted **read-only**: the job cannot corrupt them, and you avoid copying TBs into an image |
| `-v $PWD/runs:/runs` | **Checkpoints and logs written to a mounted volume**, so they survive container exit or crash |
| `hf-cache` volume | Pretrained weights cached across runs |
| `MLFLOW_TRACKING_URI` | Sends metrics to the tracking server (section 8.15) |

**Checkpointing.** Containers are disposable and GPU nodes (especially spot/preemptible ones) can vanish. Save checkpoints regularly **to the volume**, and make your training script *resume from the latest checkpoint* if one exists. Then "container died" costs minutes, not days.

**Multi-GPU / multi-node.** Single-node multi-GPU: `torchrun --nproc_per_node=4 train.py` inside one container with `--gpus all --ipc=host`. Multi-node needs an orchestrator (Kubernetes with Kubeflow/Volcano/Ray, Slurm with Pyxis/Enroot, or managed services) plus a fast network (InfiniBand/EFA), which is beyond a single `docker run`.

**File ownership gotcha.** Files written to a bind mount by a container running as a different UID will be owned by that UID on the host. Run with `--user "$(id -u):$(id -g)"` for dev, or fix ownership in your pipeline.

---

## 8.15 Experiment tracking: MLflow in Compose

**Experiment tracking** records each training run's parameters, metrics and artifacts (model files) so you can compare runs and reproduce the good one. **MLflow** is a popular open-source tracker.

```yaml
services:
  mlflow-db:
    image: postgres:17
    environment:
      POSTGRES_USER: mlflow
      POSTGRES_PASSWORD: ${MLFLOW_DB_PASSWORD}
      POSTGRES_DB: mlflow
    volumes:
      - mlflow_pg:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow -d mlflow"]
      interval: 5s
      retries: 10

  mlflow:
    image: ghcr.io/mlflow/mlflow:v3.1.0
    command: >
      sh -c "pip install --no-cache-dir psycopg2-binary &&
             mlflow server --host 0.0.0.0 --port 5000
             --backend-store-uri postgresql://mlflow:${MLFLOW_DB_PASSWORD}@mlflow-db:5432/mlflow
             --artifacts-destination /mlartifacts"
    volumes:
      - mlflow_artifacts:/mlartifacts
    ports:
      - "127.0.0.1:5000:5000"
    depends_on:
      mlflow-db:
        condition: service_healthy

volumes:
  mlflow_pg:
  mlflow_artifacts:
```

* **Backend store** (`--backend-store-uri`) holds run metadata (parameters, metrics) in PostgreSQL.
* **Artifact store** (`--artifacts-destination`) holds files (models, plots); here a local volume; in production use S3/GCS/Azure Blob.
* `--host 0.0.0.0` so other containers can reach it; the host port is bound to loopback only.
* `pip install psycopg2-binary` at start is a quick-demo shortcut; in real use build a small image with the driver baked in.
* **The tag `v3.1.0` is an example; I did not verify the current MLflow image tag or its default flags, so check the MLflow docs for the current Docker/server instructions** (recent MLflow versions also added server security settings such as allowed hosts that may require extra flags when accessed through a name other than localhost).
* Your training container logs with `mlflow.log_metric(...)` after setting `MLFLOW_TRACKING_URI=http://mlflow:5000` and attaching to the same Compose network (or the host URL if running outside Compose).

---

## 8.16 Async workers: Celery/RQ + Redis

LLM and ML tasks are slow. If your HTTP API does the work inline, requests time out and your server stalls. The standard fix: the API enqueues a **job** into a **queue**, returns a job ID immediately, and a **worker** process does the work.

```
Client --POST /summarize--> API --enqueue--> Redis (queue) --> Worker (GPU or CPU)
   ^                          |                                     |
   |<--- 202 Accepted {id} ---+                                     v
   +------ GET /jobs/{id} (poll) or websocket/SSE <---- result stored in Redis/DB
```

* **Celery** is the feature-rich Python task queue; **RQ** (Redis Queue) is simpler; **Dramatiq/Arq/Taskiq** are alternatives.
* **Redis** here is the *broker* (message transport) and often the result backend.

```yaml
  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis_data:/data

  worker:
    build: ./rag-api
    command: ["celery", "-A", "app.tasks", "worker", "--loglevel=info", "--concurrency=2"]
    environment:
      CELERY_BROKER_URL: redis://redis:6379/0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

* `--appendonly yes` makes Redis persist its data to disk (so queued jobs survive restarts).
* `--concurrency=2` runs two worker processes. For GPU work keep this low (each process loads the model into VRAM) or use one process with batching.
* Scale workers independently of the API: `docker compose up -d --scale worker=3` (each replica competes for the GPU unless you assign devices carefully, so on one GPU prefer one worker with internal batching).

---

## 8.17 Streaming responses behind nginx (timeouts and buffering)

LLM answers arrive token by token, and apps stream them to the user using **SSE (Server-Sent Events)**, a one-way HTTP stream, or WebSockets. A reverse proxy in front can **break streaming** by (a) buffering the whole response before forwarding and (b) timing out long requests.

```nginx
server {
    listen 80;

    location /api/ {
        proxy_pass http://api:8000;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_cache off;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        chunked_transfer_encoding on;
    }
}
```

| Directive | Why |
|---|---|
| `proxy_pass http://api:8000;` | Forward to the Compose service named `api` |
| `proxy_http_version 1.1;` + `proxy_set_header Connection "";` | Needed for streaming/keep-alive (HTTP/1.0 to upstream would close each response) |
| `proxy_buffering off;` | **The key setting:** nginx forwards chunks immediately instead of collecting them |
| `proxy_cache off;` | Never cache a stream |
| `proxy_read_timeout 300s;` | Default is 60 s; slow first tokens or long generations would otherwise get cut with 504 |
| `chunked_transfer_encoding on;` | Allow streaming without a known length |

Also let the app help: FastAPI `StreamingResponse(..., media_type="text/event-stream")` and sending the response header `X-Accel-Buffering: no` tells nginx per-response not to buffer. Cloud load balancers have their own idle timeouts (for example 60 seconds by default on some); raise them or send periodic heartbeat comments (`: ping`) in the stream.

---

## 8.18 Observability for AI services

You cannot fix what you cannot see. Three pillars:

1. **Logs**, write **structured JSON to stdout**; Docker captures it; a collector ships it.

```python
import json, logging, sys, time

class JsonFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "ts": time.strftime("%Y-%m-%dT%H:%M:%S", time.gmtime(record.created)),
            "level": record.levelname,
            "msg": record.getMessage(),
            "logger": record.name,
        })

h = logging.StreamHandler(sys.stdout)
h.setFormatter(JsonFormatter())
logging.basicConfig(level=logging.INFO, handlers=[h])
```

   * Plain `print` and a JSON formatter both reach `docker logs`. JSON lets Loki/ELK/CloudWatch search by field (for example `request_id`, `latency_ms`, `model`).
   * **Do not log prompts, documents or API keys** if they contain personal or confidential data.
   * Limit local log growth: Compose `logging: {driver: json-file, options: {max-size: "10m", max-file: "3"}}` (otherwise the default json-file driver can fill your disk).

2. **Metrics**, numbers over time. Expose `/metrics` using `prometheus-client`; scrape with **Prometheus**; graph in **Grafana**. Useful AI metrics: request rate, latency (p50/p95/p99), **time to first token**, tokens per second, queue depth, error rate, cache hit rate, GPU utilization and memory. For GPUs, NVIDIA's **DCGM exporter** publishes GPU metrics in Prometheus format (image from NVIDIA's registry; check NVIDIA's docs for the current tag). vLLM and Triton also expose Prometheus metrics endpoints (Triton on port 8002 by default).

3. **Traces**, follow one request across services (OpenTelemetry). LLM-specific tools (Langfuse, OpenLLMetry, Phoenix) record prompts, tool calls and costs; many have official Compose files.

Compose snippet for Prometheus + Grafana:

```yaml
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"

  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "127.0.0.1:3001:3000"
    volumes:
      - grafana_data:/var/lib/grafana
```

`prometheus.yml`:

```yaml
scrape_configs:
  - job_name: rag-api
    scrape_interval: 15s
    static_configs:
      - targets: ["api:8000"]
```

* `scrape_interval` is how often Prometheus pulls `/metrics`.
* `targets` uses the Compose service name and port, as usual.

---

## 8.19 Reproducible builds, CI/CD, and multi-arch

### 8.19.1 Reproducibility checklist for ML images

| Item | How |
|---|---|
| Pin the base image | `FROM python:3.12-slim@sha256:<digest>`: tags move, digests do not |
| Lock Python dependencies | `uv lock` / `uv sync --locked`, or `pip-compile --generate-hashes` + `pip install --require-hashes -r requirements.txt` |
| Pin system packages when it matters | `apt-get install pkg=version` (or accept drift and rebuild regularly) |
| Pin model revisions | `from_pretrained("org/model", revision="<commit-sha>")` so upstream edits do not silently change behaviour |
| Pin the framework/CUDA pair | Choose PyTorch wheel index with explicit CUDA version (for example the `cu128` index) |
| Record provenance | Tag images with the git SHA; generate SBOM/provenance attestations ([Chapter 10](10-security.md)) |
| Seed randomness for tests | Fixed seeds in evaluation (remember GPU kernels can still be nondeterministic) |

### 8.19.2 CI/CD pipeline for an ML image

```
git push -> CI: lint + unit tests (CPU) -> build image (BuildKit cache) -> scan (Trivy/Scout)
          -> push to registry (tag: git SHA) -> integration test (compose up + smoke) -> deploy (staging) -> promote
```

A GitHub Actions sketch:

```yaml
name: build-image
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}/rag-api:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: mode=max
          sbom: true
```

* `permissions` follows least privilege: read the code, write packages.
* `docker/setup-buildx-action` enables BuildKit's advanced features.
* `login-action` signs in to GitHub Container Registry using the automatic `GITHUB_TOKEN`.
* `build-push-action` builds and pushes; the tag is the commit SHA (immutable, traceable).
* `cache-from/to: type=gha` stores layer cache in GitHub's cache so ML dependency layers are not rebuilt each time.
* `provenance: mode=max` and `sbom: true` attach SLSA provenance and an SBOM as attestations (see [Chapter 10](10-security.md)). Note Docker's docs warn that `mode=max` records build-argument values, so never pass secrets as build args.
* Action versions (`@v4`, `@v3`, `@v6`) are examples; pin to commit SHAs in sensitive pipelines.

Standard GitHub-hosted runners have **no GPU**. GPU tests need self-hosted GPU runners or a provider's GPU runners; keep the bulk of your tests CPU-only with small models and use a separate, less frequent GPU smoke test.

### 8.19.3 Multi-architecture images (Apple Silicon dev, amd64 prod)

Apple Silicon Macs are **arm64**; most cloud GPU servers are **amd64** (x86-64), and some new data-centre superchips are arm64 as well. An image built on one architecture does not run natively on the other (it may run slowly via emulation or fail). A **multi-arch image** is one tag that points to per-architecture variants; Docker picks the right one automatically.

```bash
docker buildx create --use --name multi
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/myorg/rag-api:1.0.0 \
  --push .
```

* `buildx create --use` creates and selects a BuildKit builder capable of multi-platform builds.
* `--platform linux/amd64,linux/arm64` builds both variants.
* `--push` pushes the combined manifest directly (multi-arch results cannot be loaded into the classic local image store as a single image).
* Cross-architecture builds use QEMU emulation (slow for compiling) unless you have native builders for each architecture.

ML-specific cautions: some wheels (certain CUDA or niche libraries) exist only for amd64, so the arm64 build may fail or need a different dependency set. A pragmatic split many teams use: **CPU API image = multi-arch**; **GPU model-server image = amd64 only**, because there is no GPU in a Mac container anyway.

Verify what an image supports with `docker buildx imagetools inspect ghcr.io/myorg/rag-api:1.0.0`, which lists each platform and digest.

---

## 8.20 Image-size tricks (summary)

| Trick | Typical effect |
|---|---|
| CPU-only PyTorch wheel for non-GPU services | Community reports: ~2.5 GB or more saved (see 8.3.4) |
| `python:*-slim` instead of full `python:*` | Hundreds of MB |
| Multi-stage build; compilers only in the builder | Hundreds of MB to GBs |
| `--no-cache-dir` for pip / cache mounts for uv | Hundreds of MB of caches kept out of layers |
| Remove apt lists in the same `RUN` | ~tens of MB |
| `.dockerignore` (exclude data, notebooks, `.git`, venvs) | Faster builds, smaller context, fewer accidental leaks |
| Keep model weights out of the image (volumes) | GBs |
| `nvidia/cuda:*-runtime` rather than `*-devel` in the final stage | Several GB |
| Don't install both `tensorflow` and `torch` unless you need both | GBs |
| Check with `docker history` / [dive](https://github.com/wagoodman/dive) | Finds the biggest layers |

Measure, do not guess: `docker images` (size column), `docker history <image>` (per layer), and `docker system df` (total disk usage by images, containers, volumes, and build cache).

---

## 8.21 From laptop to cloud GPU

### 8.21.1 The journey

```
Laptop (CPU, small model, Compose)
   -> same images on a single cloud GPU VM (docker compose up, NVIDIA toolkit installed)
      -> managed container service with GPUs (serverless or cluster)
         -> Kubernetes + GPU operator (multi-node, autoscaling, many teams)
```

### 8.21.2 Options

| Option | What it is | Good for | Watch out |
|---|---|---|---|
| **Single GPU VM** (AWS EC2 GPU instances, GCP Compute Engine with GPUs, Azure NC/ND-series, Lambda Cloud, RunPod pods, Vast.ai, CoreWeave, etc.) | Rent a machine with a GPU, install driver + Docker + toolkit (or use a GPU-ready image), run Compose | Simplest path, experiments, small production | You manage patching, uptime, cost when idle |
| **Provider-managed GPU containers / serverless GPU** (RunPod serverless, Modal, Replicate, Cloud Run with GPUs, Azure Container Apps GPU, AWS SageMaker endpoints) | Give them an image; they run it on GPUs, often scale to zero | Spiky inference, little ops | Cold starts (image pull + model load), per-provider constraints; check the current feature and pricing pages, which change frequently |
| **AWS ECS on EC2 GPU instances / EKS** | Container orchestration on GPU instances | AWS shops | Capacity availability of GPU instances, GPU-optimized AMI |
| **GCP GKE / Cloud Run GPUs** | Kubernetes or serverless containers with GPUs | GCP shops | Region/GPU quotas |
| **Azure AKS / Container Apps** | Same on Azure | Azure shops | Quotas |
| **Kubernetes + NVIDIA GPU Operator** | Operator that automates installing drivers, container toolkit, device plugin, DCGM monitoring on K8s nodes | Multi-team platforms | Operational complexity |

(I name providers as examples of categories; feature sets, GPU availability and prices change quickly and I did not verify any provider's current offering. Check their pricing and docs.)

### 8.21.3 Kubernetes and GPUs in 60 seconds

**Kubernetes (K8s)** is an orchestrator: you declare "run 3 replicas of this image" and it schedules them across machines, restarts failures, and routes traffic ([Chapter 11](11-production-and-enterprise.md)). To use GPUs there:

1. GPU nodes need the NVIDIA driver and container toolkit. The **NVIDIA GPU Operator** automates this.
2. The **NVIDIA device plugin** advertises GPUs as a schedulable resource `nvidia.com/gpu`.
3. Your pod requests it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: llm
spec:
  containers:
    - name: vllm
      image: vllm/vllm-openai:latest
      args: ["--model", "Qwen/Qwen3-0.6B"]
      resources:
        limits:
          nvidia.com/gpu: 1
```

* `limits: nvidia.com/gpu: 1` asks the scheduler for a node with one free GPU and attaches it. (GPUs are requested as whole units by default; sharing techniques such as MIG and time-slicing exist but are advanced.)
* Add a PersistentVolumeClaim for the model cache, a readiness probe that waits for the model to load, and an HPA/KEDA autoscaler keyed on queue depth or request concurrency, not CPU.

### 8.21.4 Cost considerations

| Factor | Advice |
|---|---|
| GPUs are the dominant cost | Turn them off when idle; autoscale to zero where cold starts are tolerable |
| Spot/preemptible instances | 60-90% cheaper (typical range) but can be reclaimed; requires checkpointing and retry logic |
| Right-size the model | Quantized or smaller models can serve the same task for a fraction of the GPU cost |
| Batching | vLLM-style continuous batching serves many users per GPU |
| Hosted API vs self-host | Self-hosting usually only wins at sustained high utilization or with privacy needs. Compute your break-even with real traffic |
| Egress and storage | Large images and model downloads incur data transfer; keep registry and compute in the same region |
| Cache | Cache embeddings and repeated answers (Redis) to avoid paying for the same inference twice |
| Idle containers | Set memory/CPU requests honestly; over-reserving wastes money |
| Registry bills | Delete old image tags; multi-GB images add up |

---

## 8.22 Common mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Baking `OPENAI_API_KEY`/HF tokens into the image | Leaked credentials | Runtime env or secrets; rotate if leaked ([Chapter 10](10-security.md)) |
| Using the CUDA PyTorch wheel in an API that never uses a GPU | +2.5 GB or more | CPU wheel index (8.3.4) |
| Alpine for ML | Build failures, slow | `python:*-slim` |
| `nvidia/cuda:*-devel` as the final image | Multi-GB bloat | Use `runtime` in the final stage or a Python slim base with wheels |
| Forgetting `--gpus all` (or the Compose device reservation) | Silent CPU fallback; mysteriously slow | Run `nvidia-smi` / `torch.cuda.is_available()` at startup and fail loudly |
| Trying to use `--gpus` in `docker build` | Not supported | Do GPU work at run time |
| `/dev/shm` too small | DataLoader crashes | `--shm-size` or `--ipc=host` |
| No volume for model cache | Re-download GBs on every restart | Mount `HF_HOME`/Ollama volume |
| `latest` tags for vLLM/Ollama/Open WebUI in production | A silent upgrade changes behaviour or breaks | Pin versions/digests |
| Exposing Ollama/Model Runner/Jupyter ports to the network | Unauthenticated code/inference access | Bind to `127.0.0.1`, put an authenticated proxy in front |
| Storing training checkpoints inside the container | Lost on exit | Write to a mounted volume |
| Running agent-generated code in the API container | Compromise | Disposable locked-down sandbox, preferably isolated host/VM |
| Binding the app to `127.0.0.1` inside the container | Published port does not work | Bind `0.0.0.0` inside, restrict exposure at the publish step |
| Expecting Mac containers to use the Apple GPU | CPU-only, slow | Native Ollama / Docker Model Runner on Mac, or cloud GPU |
| Streaming "works locally but not behind proxy" | Buffering/timeouts | `proxy_buffering off`, raise timeouts (8.17) |

---

## 8.23 Try it yourself

1. **No-GPU path (works on your Windows machine today).** Run Ollama in CPU mode, pull a small model and chat:
   ```bash
   docker run -d -v ollama:/root/.ollama -p 127.0.0.1:11434:11434 --name ollama ollama/ollama
   docker exec -it ollama ollama run llama3.2
   ```
   Then `docker rm -f ollama` to clean up the container (the `ollama` volume keeps the model; remove with `docker volume rm ollama` only if you want the disk space back).
2. **Docker Model Runner.** Check `docker model --help`; pull and run a small model such as `ai/smollm2`.
3. **Shrink an image.** Build the Example 1 image twice, once with the default `torch` and once with the CPU index, and compare with `docker images`. Write down the difference.
4. **Compose RAG.** Build the stack in 8.10, ingest three text files, and ask questions about them. Swap `LLM_BASE_URL` to a hosted API (with the key in a secret file) and confirm nothing else changes.
5. **Break streaming on purpose.** Put nginx in front of an SSE endpoint with default settings, watch the stream arrive all at once, then fix it with `proxy_buffering off`.
6. **Sandbox.** Run the hardened sandbox command from 8.12.1 against a script that tries to reach the internet and one that allocates 1 GB of memory; observe both fail.
7. **If you have an NVIDIA GPU or a cloud VM:** install the toolkit (8.6.2), run `nvidia-smi` in a container, then serve a small model with vLLM and call `/v1/models`.

---

## 8.24 Chapter summary

* AI stacks are stacks of tightly coupled versions (Python, framework, CUDA, driver). Docker freezes everything except the host driver.
* Split the app: thin CPU API, separate GPU model server, vector DB, cache, queue/worker, UI.
* Choose `python:slim` + wheels by default; use `nvidia/cuda` `devel` only to compile; use CPU-only wheels where no GPU is used.
* GPUs need a host driver + NVIDIA Container Toolkit; use `--gpus all` or Compose device reservations; WSL2 works for Windows; macOS containers cannot use Metal.
* Keep weights out of images unless you have a reason; use cache volumes, init steps and offline mode.
* Ollama and Docker Model Runner for local dev; vLLM (or Triton) for production throughput; TGI is in maintenance mode.
* Keys come from the environment or secrets at runtime, never from the image; use the OpenAI-compatible base-URL pattern for portability.
* Sandbox agent code, containerize MCP servers, bind sensitive ports to loopback, and set `--shm-size`, checkpoints and streaming-friendly proxy settings.
* Pin everything, build multi-arch where it makes sense, and move to the cloud only when you must.

Next: [Chapter 9: Hands-on AI App Walkthrough](09-hands-on-ai-app-walkthrough.md) builds a working AI application end to end. Security for everything in this chapter is covered in [Chapter 10](10-security.md).
