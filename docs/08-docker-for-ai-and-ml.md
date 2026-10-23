# Chapter 8: Docker for AI and Machine Learning

This chapter covers containerizing AI and ML software. It describes the container layout of an AI application, base image selection and size reduction, GPU access, model weight handling, local LLM servers, a RAG stack in Compose, hosted LLM APIs, MCP servers, sandboxing, Jupyter, training, workers, streaming, observability, CI/CD, multi-arch builds and cloud GPU deployment.

Previous: [Chapter 7: Who Needs Docker?](07-who-needs-docker.md) | Next: [Chapter 9: Hands-on AI App Walkthrough](09-hands-on-ai-app-walkthrough.md)

**Freshness.** AI tooling changes quickly. Image names, tags and flags in this chapter were checked against official docs fetched in late 2026. Items that could not be verified are marked "Unverified". Check the linked official docs before pinning a tag in production. Version numbers in examples are examples.

---

## 8.1 Why AI needs Docker

AI software has layers of dependencies that must match each other. A typical GPU Python project depends on the following, from the hardware upward:

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

Terms:

- **GPU (Graphics Processing Unit)**: a chip with thousands of small cores, used for the matrix maths of neural networks.
- **CUDA**: NVIDIA's platform and library set for running maths on NVIDIA GPUs.
- **cuDNN**: NVIDIA's library of optimized deep-learning operations (convolutions and others), used by frameworks.
- **PyTorch / TensorFlow / JAX**: deep-learning frameworks used to define and run models.
- **Driver**: software on the host operating system that lets the OS use the GPU.
- **Inference**: using a trained model to produce outputs (for example answering a prompt).
- **Training**: creating or adjusting a model's weights from data.
- **Weights**: the arrays of numbers that make up the model. A 7-billion-parameter model is typically several GB to about 15 GB depending on numeric precision.

### The problems Docker solves here

| Problem | Without Docker | With Docker |
|---|---|---|
| PyTorch 2.x needs a specific CUDA build; another project needs an older one | Constant reinstalling, broken environments | Each project has its own image; both coexist |
| Works on the data scientist's laptop, fails in production | Production crashes on an import | Same image from laptop to production |
| Python/OS library mismatches (`libssl`, `libGL`, `glibc`) | System packages debugged per machine | Libraries are inside the image |
| Reproducing a result from six months ago | Environment is gone | Re-run the pinned image |
| Deploying a model server on a new machine | Hours of driver/CUDA/Python setup | `docker run` (after one-time driver and toolkit setup) |
| Many teams and frameworks on shared GPU servers | Conflicts over driver updates | Isolated containers share one driver |
| Onboarding a new ML engineer | A day of setup | `docker compose up` |
| Scaling inference replicas | Manual server provisioning | Orchestrators schedule identical containers |

### What Docker does not solve for AI

1. **The driver lives on the host, not in the image.** Install an NVIDIA driver on the host (or use a cloud image that includes it).
2. **Images are large.** GPU libraries are several GB (section 8.4).
3. **Containers do not make GPUs faster or cheaper.** They make environments reproducible.
4. **macOS containers cannot use the Apple GPU** (section 8.6.7).
5. **Model weights are data, not code.** They need their own distribution strategy (section 8.7).

---

## 8.2 Anatomy of an AI app in containers

A real AI application is a small system of several containers. The common shape is a **RAG chatbot** (Retrieval-Augmented Generation, explained in section 8.10):

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
| **UI** | Chat page | `ghcr.io/open-webui/open-webui`, a custom React build | No |
| **API server** | Business logic, auth, prompt assembly, calls to models and DB | Custom image on `python:3.12-slim` | Usually no |
| **Model server** | Runs the neural network for inference | `ollama/ollama`, `vllm/vllm-openai`, `llama.cpp` server, Triton | Usually yes |
| **Embedding service** | Turns text into vectors | Same as model server, or a library in the API | Optional |
| **Vector database** | Stores vectors, finds similar ones | `qdrant/qdrant`, `pgvector/pgvector`, `chromadb/chroma` | No |
| **Cache** | Answers, rate limits, sessions | `redis` | No |
| **Queue + worker** | Long jobs: document ingestion, batch scoring, fine-tuning | `redis` + worker image | Sometimes |
| **Experiment tracker** | Records training runs | MLflow | No |
| **Observability** | Metrics and dashboards | Prometheus, Grafana, Loki | No |

Design rule: keep the API container thin and CPU-only, and run the GPU-dependent model in its own container.

- The thin API rebuilds quickly and scales cheaply.
- The model server changes rarely and is expensive.
- Models can be swapped without redeploying the API.
- Several API replicas can share one GPU.

---

## 8.3 Base image choices

The `FROM` line ([Chapter 4](04-dockerfile-deep-dive.md)) has the largest effect on image size, build time and compatibility.

### 8.3.1 The options

| Base image | Typical use | Size* | Notes |
|---|---|---|---|
| `python:3.12-slim` | API servers, CPU inference, glue code | ~100-200 MB | Small; Debian-based, so wheels work. No GPU libraries |
| `python:3.12-alpine` | Rarely for ML | Smaller | Uses musl libc; many ML wheels (numpy, torch) target glibc, so builds are slow or fail. Avoid for ML |
| `pytorch/pytorch:<ver>-cuda<ver>-cudnn<ver>-runtime` | GPU inference/training with PyTorch | Multiple GB | Tested PyTorch + CUDA + cuDNN combination; fixed Python/PyTorch versions |
| `nvidia/cuda:<ver>-base-<os>` | Minimal CUDA runtime | Hundreds of MB | Smallest NVIDIA-supported base; no cuDNN |
| `nvidia/cuda:<ver>-runtime-<os>` | Prebuilt CUDA apps | ~1-3 GB | CUDA runtime libraries; no compiler |
| `nvidia/cuda:<ver>-cudnn-runtime-<os>` | Runtime + cuDNN | Larger | For frameworks that use system cuDNN |
| `nvidia/cuda:<ver>-devel-<os>` | Compiling CUDA code (flash-attention, custom kernels) | ~3-8 GB | Has `nvcc` and headers; use only in the build stage |
| `nvcr.io/nvidia/pytorch:<yy.mm>-py3` | NVIDIA NGC-optimized PyTorch | ~10-20+ GB | Tuned, many extras; NGC registry |
| `vllm/vllm-openai`, `ollama/ollama` | Ready-made model servers | Large | No build needed; less customization |

\*Sizes vary with versions and tooling. Measure with `docker images` on the target machine.

### 8.3.2 Reading `nvidia/cuda` tag names

```
nvidia/cuda:<CUDA version>-<flavour>-<OS>

nvidia/cuda:13.4.3-cudnn-devel-ubuntu24.04
              |      |      |       |
              |      |      |       +-- base OS (Ubuntu 22.04 / 24.04 / 26.04, some others like Rocky, UBI)
              |      |      +---------- devel = compiler + headers; runtime = libs only; base = minimal
              |      +----------------- cudnn = includes cuDNN (tensorrt variants also exist)
              +------------------------ CUDA toolkit version
```

Checked on Docker Hub in October 2026: the newest tags were CUDA **13.4.3** with `cudnn-devel`, `cudnn-runtime`, `tensorrt-devel` and `tensorrt-runtime` flavours on Ubuntu 22.04/24.04/26.04. Unverified: the plain `base`/`runtime` tags were not listed on the page that could be read; they follow the same pattern. Check <https://hub.docker.com/r/nvidia/cuda/tags> for current tags. NVIDIA removes old tags over time, so pin and mirror images you depend on ([Chapter 11](11-production-and-enterprise.md)).

### 8.3.3 A surprising fact: you often do not need an `nvidia/cuda` image at all

PyTorch pip wheels for CUDA bundle their own CUDA runtime libraries as separate `nvidia-*` Python packages. On a host with a recent enough NVIDIA driver, a plain Python base works:

```dockerfile
FROM python:3.12-slim
RUN pip install torch        # default Linux wheel pulls CUDA libs as pip packages
```

The cost is image size (several GB of CUDA libraries from pip). `nvidia/cuda:*-devel` is needed only to compile code (flash-attention, xformers from source, custom CUDA extensions) or when a vendor requires it.

| Situation | Base |
|---|---|
| Prebuilt wheels (most cases) | `python:3.12-slim` plus GPU passthrough at run time |
| Tested all-in-one PyTorch | `pytorch/pytorch:*-runtime` |
| Compiling CUDA code | Build in `nvidia/cuda:*-devel`, copy results to `-runtime` (multi-stage, section 8.3.5) |

### 8.3.4 CPU-only wheels: the cheapest size win

For a container that never uses a GPU (an API calling a hosted LLM, a small CPU embedding model, a scikit-learn model), install the CPU-only build of PyTorch from PyTorch's CPU wheel index. Community reports (GitHub issues and a PR in the Roboflow inference project) indicate:

- The CPU-only `torch` wheel is about 200 MB (one report: 196 MB; another: 222 MB for the whole CPU-only install across 13 wheels).
- The default Linux PyPI install pulls several GB of NVIDIA and Triton packages (reports of about 2.4-2.7 GB of unused CUDA wheels in CPU deployments).
- Reported image savings range from about 2.5 GB compressed to 4+ GB uncompressed.

These figures come from community reports, not PyTorch's official docs. Actual savings differ.

With pip:

```dockerfile
RUN pip install --no-cache-dir torch --index-url https://download.pytorch.org/whl/cpu
```

| Part | Meaning |
|---|---|
| `pip install torch` | Installs PyTorch |
| `--no-cache-dir` | Does not keep downloaded files in the image layer |
| `--index-url https://download.pytorch.org/whl/cpu` | Fetches from PyTorch's CPU-only index instead of PyPI |

With **uv** (a fast Python package manager written in Rust), configure the CPU index in `pyproject.toml`:

```toml
[tool.uv.sources]
torch = { index = "pytorch-cpu" }

[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true
```

- `[tool.uv.sources]`: uses the index named `pytorch-cpu` for the package `torch`.
- `[[tool.uv.index]]`: declares that index. `explicit = true` restricts it to packages that request it, which prevents dependency confusion (a malicious same-name package on another index).

Unverified: uv also offers a `--torch-backend` option reported by community sources; its exact behavior was not verified. See the uv documentation ("Using uv with PyTorch") for the current recommended configuration.

Dependencies of dependencies can also pull CUDA wheels (some packages depend on `torch`). Verify after building: `docker run --rm myimage pip list | grep -i nvidia` should print nothing for a CPU image.

### 8.3.5 Multi-stage builds for ML

A **multi-stage build** uses one stage to compile and install (with large tools) and a final slim stage that copies only the results.

```
Stage 1 "builder"  (FROM python:3.12 ... large, has compilers)
   - install build tools
   - install dependencies into /app/.venv
                       |
                       |  COPY --from=builder /app/.venv  (only the venv crosses over)
                       v
Stage 2 "runtime"  (FROM python:3.12-slim ... small)
   - copy venv + code
   - run as non-root
```

A full example follows in section 8.4.

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

`app/main.py` (minimal):

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

`.dockerignore` (keeps junk and secrets out of the build context; [Chapter 4](04-dockerfile-deep-dive.md)):

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

| Line | Meaning |
|---|---|
| `# syntax=docker/dockerfile:1` | Current stable Dockerfile frontend (enables `--mount`). |
| `FROM python:3.12-slim AS builder` | First stage. In production, pin to a digest (`python:3.12-slim@sha256:...`), see [Chapter 10](10-security.md). |
| `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/` | Copies the `uv` binaries from uv's official image (documented pattern). Pin a version tag (for example `uv:0.9.x`) instead of `latest`. |
| `UV_COMPILE_BYTECODE=1` | Precompiles `.pyc` files for faster startup. |
| `UV_LINK_MODE=copy` | Copies files instead of hard-linking from the cache (the cache is a separate mount, so links would fail with a warning). |
| `UV_PYTHON_DOWNLOADS=never` | Uses the image's Python; uv does not download another. |
| `COPY pyproject.toml uv.lock ./` | Copies only the dependency manifests first, so the layer cache survives code-only changes. |
| `RUN --mount=type=cache,target=/root/.cache/uv uv sync --locked --no-dev --no-install-project` | `--mount=type=cache`: BuildKit cache directory that is not stored in the image. `--locked`: fails if `uv.lock` is out of date. `--no-dev`: skips dev dependencies. `--no-install-project`: installs dependencies only, before the code is copied. |
| `COPY app ./app` then second `uv sync` | Installs the project itself; the dependency layer stays cached. |
| `FROM python:3.12-slim AS runtime` | Fresh final image: no uv, no build cache. |
| `groupadd` / `useradd --system --no-create-home` | Non-root user with fixed UID 10001, so Kubernetes `runAsUser` and volume permissions are predictable. |
| `COPY --from=builder --chown=app:app /app /app` | Copies the venv and code from stage 1, owned by the new user. |
| `ENV PATH="/app/.venv/bin:$PATH"` | Resolves `python` and `uvicorn` to the virtual environment. |
| `PYTHONUNBUFFERED=1` / `PYTHONDONTWRITEBYTECODE=1` | Immediate log output for `docker logs`; no `.pyc` writes at run time (the filesystem may be read-only). |
| `USER app` | Subsequent instructions and the running container are non-root. |
| `EXPOSE 8000` | Documentation only; does not publish the port. |
| `HEALTHCHECK ...` | Failure marks the container `unhealthy`. Uses the Python standard library, so no `curl` is needed. |
| `CMD [...]` | Starts uvicorn. `--host 0.0.0.0` is required: binding `127.0.0.1` inside a container makes the app unreachable through published ports. |

Build and run:

```bash
docker build -t rag-api:0.1.0 .
docker run --rm -p 127.0.0.1:8000:8000 -e LLM_BASE_URL=https://api.example.com/v1 rag-api:0.1.0
```

- `-p 127.0.0.1:8000:8000`: publishes container port 8000 on the host loopback only.
- `-e LLM_BASE_URL=...`: sets an environment variable (placeholder URL).

```bash
curl http://127.0.0.1:8000/healthz
# {"status":"ok"}
```

Check size and layers:

```bash
docker images rag-api
docker history rag-api:0.1.0
```

- `docker history` lists each layer and its size, showing which instruction added the weight.

---

## 8.5 Example 2: a GPU inference image

Use case: running a custom PyTorch model on an NVIDIA GPU.

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

Differences from example 1:

| Line | Meaning |
|---|---|
| `torch` as a normal dependency in `pyproject.toml` | The default Linux wheel brings the CUDA runtime libraries as pip packages. No `nvidia/cuda` base is required. |
| `HF_HOME=/models/hf` | Cache location for Hugging Face downloads. A volume is mounted at `/models` (section 8.7). |
| `NVIDIA_VISIBLE_DEVICES=all` | Read by the NVIDIA runtime to decide which GPUs are visible. Often set by runtime flags anyway. |
| `NVIDIA_DRIVER_CAPABILITIES=compute,utility` | Driver features to expose: `compute` for CUDA, `utility` for `nvidia-smi`. Add `video` for NVENC/NVDEC. |

To compile CUDA extensions, use a devel stage:

```dockerfile
FROM nvidia/cuda:13.4.3-cudnn-devel-ubuntu24.04 AS build
# ... install python, build the extension wheel ...

FROM nvidia/cuda:13.4.3-cudnn-runtime-ubuntu24.04 AS runtime
# ... copy the built wheel, install it, run as non-root ...
```

Replace `13.4.3` with a tag that exists and that the framework wheels support. The framework determines the CUDA version: for PyTorch built for CUDA 12.8, a 12.8 system toolkit is the safe match for compiling extensions.

---

## 8.6 GPUs in Docker

### 8.6.1 Three different things called "CUDA"

| Thing | Where it lives | Who installs it |
|---|---|---|
| **NVIDIA driver** (kernel module + `libcuda.so` user-space driver library) | On the host only | The user (or the cloud image) |
| **CUDA toolkit / runtime libraries** (cuBLAS, cuDNN, `nvcc`, ...) | Inside the image (from `nvidia/cuda`, a framework image, or pip wheels) | Image builder |
| **NVIDIA Container Toolkit** | On the host, integrates with Docker | The user, once |

At container start, the NVIDIA Container Toolkit injects the host driver's libraries and the GPU device files (`/dev/nvidia0`, `/dev/nvidiactl`, `/dev/nvidia-uvm`) into the container. The image carries the CUDA libraries; the host supplies the driver.

```
 Host (Linux or WSL2)                       Container
+----------------------------------+      +-----------------------------+
| NVIDIA kernel driver  (e.g. 5xx) |----->| /dev/nvidia*  (injected)    |
| libcuda.so (driver user-space)   |----->| libcuda.so    (injected)    |
| NVIDIA Container Toolkit         |      | CUDA libs (from the image)  |
| Docker Engine                    |      | PyTorch / your code         |
+----------------------------------+      +-----------------------------+
```

Compatibility rule (simplified): the host driver must be new enough for the CUDA version the container's libraries were built against. Newer drivers run older CUDA apps; the reverse fails. The "CUDA Version: X.Y" in the `nvidia-smi` header is the maximum CUDA version the driver supports; it does not mean a toolkit is installed. NVIDIA's CUDA compatibility docs describe forward-compatibility packages for data-centre GPUs, for running newer CUDA on an old driver.

### 8.6.2 Installing the NVIDIA Container Toolkit on Linux (Ubuntu/Debian)

Prerequisite: an NVIDIA driver installed on the host (check with `nvidia-smi`). The commands are from NVIDIA's official install guide (<https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>), fetched while writing this chapter.

Install prerequisites:

```bash
sudo apt-get update && sudo apt-get install -y --no-install-recommends \
   ca-certificates curl gnupg2
```

- Installs certificate authorities, `curl` for downloads, and `gnupg2` for verifying signing keys.

Add the NVIDIA repository:

```bash
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
```

- First half: downloads NVIDIA's package-signing GPG key into a keyring file so `apt` can verify packages.
- Second half: downloads the repository definition, rewrites each `deb https://` line to `signed-by=...` (trust only packages signed by that key), and saves it as an apt source.

Install the toolkit:

```bash
sudo apt-get update
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.20.1-1
sudo apt-get install -y \
    nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
    libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

- Installs the four toolkit packages at one pinned version. `1.20.1-1` was the version in NVIDIA's guide when read; check the page for the current number.

Register the NVIDIA runtime with Docker:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

- `nvidia-ctk`: the toolkit's command-line helper. `runtime configure --runtime=docker` edits `/etc/docker/daemon.json` to add an `nvidia` runtime.
- The Docker restart makes the daemon reread its config. It restarts running containers unless live-restore is enabled.
- For rootless Docker, NVIDIA's docs say to pass `--config=$HOME/.config/docker/daemon.json` and restart the user service instead.

Verify:

```bash
docker run --rm --gpus all nvidia/cuda:13.4.3-base-ubuntu24.04 nvidia-smi
```

- `--rm`: removes the container afterwards.
- `--gpus all`: gives the container every GPU.
- `nvidia-smi`: the NVIDIA System Management Interface; prints GPU, driver and memory status.
- Unverified: this exact `base` tag was not confirmed to exist. Substitute any existing `nvidia/cuda` base tag whose CUDA version the driver supports.

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

If the GPU name appears, GPU passthrough works.

Selecting GPUs:

| Flag | Meaning |
|---|---|
| `--gpus all` | Every GPU |
| `--gpus 1` | Any one GPU |
| `--gpus '"device=0,2"'` | Specific GPUs by index (nested quotes because Docker parses a comma-separated list) |
| `--gpus 'all,capabilities=compute,utility'` | All GPUs, only these driver capabilities |
| `-e NVIDIA_VISIBLE_DEVICES=1` (with `--runtime=nvidia`) | Legacy way to select a GPU through the runtime |

### 8.6.3 CDI: the newer way to expose devices

**CDI (Container Device Interface)** is a cross-runtime standard for describing devices (GPUs, FPGAs, etc.) in a spec file, so the container engine can inject them without vendor-specific runtime hooks. With the NVIDIA toolkit:

```bash
sudo nvidia-ctk cdi generate --output=/var/run/cdi/nvidia.yaml
nvidia-ctk cdi list
docker run --rm --device nvidia.com/gpu=all ubuntu nvidia-smi
```

- `cdi generate`: inspects the host GPUs and writes a YAML spec describing them.
- `cdi list`: shows available device names, such as `nvidia.com/gpu=0` and `nvidia.com/gpu=all`.
- `--device nvidia.com/gpu=all`: injects all GPUs through CDI (requires a Docker Engine version with CDI support enabled).

Unverified: NVIDIA's install guide text that could be read mentions CDI only for Podman and shows no Docker CDI commands. The Docker CDI flow above, and the claim that toolkit 1.18+ can generate CDI specs just-in-time (needing Docker 26.1+), come from third-party write-ups not confirmed against NVIDIA's docs. Check the NVIDIA toolkit and Docker docs for your versions. Use `--gpus all` as the dependable default.

### 8.6.4 GPUs in Compose

Compose uses device reservations:

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

| Key | Meaning |
|---|---|
| `deploy.resources.reservations.devices` | Devices the service needs |
| `driver: nvidia` | Selects the NVIDIA driver/runtime |
| `count: all` (or a number such as `1`) | Number of GPUs. Alternatively `device_ids: ["0", "1"]` for specific GPUs; `count` and `device_ids` cannot be combined |
| `capabilities: [gpu]` | Required; declares what the device is used for |

Newer Compose versions also accept a short form, `gpus: all`, directly under the service. If it errors, use the long form above, which is the long-established syntax.

### 8.6.5 GPUs on Windows with WSL2

Docker Desktop on Windows runs Linux containers inside a **WSL2** (Windows Subsystem for Linux 2) virtual machine. NVIDIA GPUs can be passed into WSL2 and so into containers:

1. Install a current **NVIDIA driver for Windows** (the standard Game Ready/Studio/Data Center driver supports WSL2). Do not install a Linux GPU driver inside the WSL distribution; the Windows driver provides the GPU to WSL2.
2. Use Docker Desktop with the WSL2 backend.
3. Test: `docker run --rm --gpus all nvidia/cuda:<tag>-base-ubuntu24.04 nvidia-smi`.

Unverified: sources disagree on whether the NVIDIA Container Toolkit must be installed separately inside WSL2 with Docker Desktop. Docker's integration normally handles it. A 2026 troubleshooting guide says that on "could not select device driver nvidia", install the toolkit and run `nvidia-ctk runtime configure --runtime=docker`. Run the `--gpus all` test first and install the toolkit only if it fails.

`--gpus` works with `docker run` and Compose services, not during `docker build`. GPU-dependent steps cannot run in a `RUN` instruction.

Without an NVIDIA GPU, use CPU images, quantized models via Ollama/llama.cpp, Docker Model Runner, or hosted APIs, and do GPU work in the cloud (section 8.21).

### 8.6.6 AMD and Intel GPUs

Ollama's docs show AMD ROCm support with `ollama/ollama:rocm` and the device flags `--device /dev/kfd --device /dev/dri`. The default Ollama image also bundles Vulkan, used automatically when the container can reach the GPU devices (disable with `OLLAMA_VULKAN=0`). These details come from Ollama's Docker documentation fetched for this chapter. Docker Model Runner lists AMD ROCm and Vulkan support on Linux with Docker Engine.

### 8.6.7 macOS and Apple Silicon: no GPU inside Linux containers

Docker Desktop on a Mac runs Linux containers in a Linux VM. Apple's GPU (Metal) is not available inside that VM, so containerized PyTorch or Ollama uses the CPU only. Alternatives:

| Approach | How it works |
|---|---|
| Run Ollama natively on the Mac (outside Docker); containers call it at `http://host.docker.internal:11434` | Native Ollama uses Metal; the app stays in a container |
| **Docker Model Runner** | On Apple Silicon it runs the inference engine as a host process (so it can use the GPU), controlled with `docker model ...`. Docker's docs list "Apple Silicon on macOS" as a GPU-supported platform |
| Develop on Mac with a small model or hosted API, deploy on amd64 + NVIDIA | Use multi-arch images (section 8.19) |

**`host.docker.internal`**: a DNS name provided by Docker Desktop so containers can reach services on the host. On Linux Engine, add `--add-host=host.docker.internal:host-gateway`.

### 8.6.8 Common GPU errors

| Symptom | Cause | Fix |
|---|---|---|
| `could not select device driver "" with capabilities: [[gpu]]` | NVIDIA toolkit not installed or registered | Install the toolkit, run `nvidia-ctk runtime configure`, restart Docker |
| `nvidia-smi: command not found` in container | Image lacks the utility, or capabilities exclude `utility` | Use an `nvidia/cuda` base or set `NVIDIA_DRIVER_CAPABILITIES=compute,utility` |
| `CUDA error: no kernel image is available for execution on the device` | Framework build does not support the GPU architecture | Use a newer PyTorch/CUDA build (new GPU generations need new builds) |
| `CUDA driver version is insufficient for CUDA runtime version` | Host driver too old for the container's CUDA | Update the driver, or use an older CUDA build |
| `RuntimeError: DataLoader worker ... Bus error` / shared memory | `/dev/shm` too small (Docker default 64 MB) | `--shm-size=8g` or `--ipc=host` |
| GPU works with `docker run` but not Compose | Missing `capabilities: [gpu]` or wrong YAML nesting | Use the snippet in 8.6.4 |
| Container sees the GPU but runs out of memory | Model larger than VRAM | Smaller or quantized model, tensor parallelism, or a larger GPU |

---

## 8.7 Handling model weights

Models are large, change rarely, and are data. Four strategies:

| Strategy | How | Pros | Cons | Use when |
|---|---|---|---|---|
| **A. Bake into the image** | `RUN hf download org/model` (or `COPY`) during build | Self-contained; instant start; works offline/air-gapped; code and model versions locked together | Very large images, slow push/pull, layers re-uploaded if ordered badly, registry cost | Edge devices, air-gapped, small models (embedding models of ~100-500 MB), serverless with fast cold start where the registry is closer than the internet |
| **B. Download at startup** | App fetches the model on first boot into a cache directory | Small images; model chosen by env var | Slow first start; needs internet and credentials at runtime; fails if the hub is down; repeated downloads without a persistent cache | Dev, experiments |
| **C. Mount a volume / cache directory** | `-v hf-cache:/models` + `HF_HOME=/models/hf` | Download once, reuse across containers and restarts; small images | Cache must be provisioned (shared disk, PVC, network storage) | Most common for local and server deployments |
| **D. Model registry / object storage + init step** | A separate init container pulls from S3/GCS/MLflow/HF into a shared volume, then the server starts | Clean separation, access control, versioned models | More moving parts | Production with governed models |

### 8.7.1 Cache directories and environment variables

| Variable | Controls |
|---|---|
| `HF_HOME` | Root of the Hugging Face cache (models, datasets, tokens). Default `~/.cache/huggingface` |
| `HF_HUB_CACHE` (older: `TRANSFORMERS_CACHE`) | Where model files go |
| `HF_TOKEN` | Access token for gated/private models (pass as a secret, never bake in) |
| `HF_HUB_OFFLINE=1` | Forbids network access; fails if the model is not cached (safer for production) |
| `TORCH_HOME` | Where `torch.hub` stores weights |
| `OLLAMA_MODELS` | Where Ollama stores models (default `/root/.ollama/models` in the official image) |

### 8.7.2 Baking a small model into the image safely (build secrets)

A token for a gated model must not end up in a layer. Use a **build secret** (BuildKit; [Chapter 10](10-security.md)):

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

- `--mount=type=secret,id=hf_token`: makes a secret file available at `/run/secrets/hf_token` during this `RUN` only; it is never written to a layer.
- `hf download <repo> --local-dir <dir>`: fetches model files with the Hugging Face CLI (`hf` in current `huggingface_hub`; older versions used `huggingface-cli download`). `all-MiniLM-L6-v2` is a small public embedding model; public models need no token, so the secret mount can be removed for them.
- The final stage copies only the model directory. The hub client and token do not reach the runtime image.

Build:

```bash
docker build --secret id=hf_token,env=HF_TOKEN -t embed-api:1 .
```

- `--secret id=hf_token,env=HF_TOKEN`: passes the shell variable `HF_TOKEN` to the build as the secret `hf_token`.

### 8.7.3 Mounting a cache volume (the common pattern)

```bash
docker volume create hf-cache
docker run --rm --gpus all \
  -v hf-cache:/models/hf \
  -e HF_HOME=/models/hf \
  -e HF_TOKEN \
  my-gpu-app:1
```

- `docker volume create hf-cache`: creates a named volume managed by Docker.
- `-v hf-cache:/models/hf`: mounts it at the app's cache path.
- `-e HF_TOKEN` (no `=value`): forwards the variable from the current shell without writing it into the command line.

The first run downloads the model; later runs reuse the cache.

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

- `restart: "no"`: the init container exits and stays exited.
- `condition: service_completed_successfully`: `llm` waits until `model-init` exits with code 0.
- `HF_HUB_OFFLINE: "1"`: the server never downloads at runtime.
- The `llm` service also needs the GPU reservation block from 8.6.4 (omitted here). `Qwen/Qwen3-0.6B` is the small model used in vLLM's Docker documentation example.

### 8.7.5 Image size and pull-time arithmetic

Rough estimate: pulling a 20 GB image over a 1 Gbit/s link (about 100-120 MB/s usable) takes about 3-4 minutes, longer on a cold node or slow registry. When an autoscaler starts new GPU nodes, this pull time is the cold-start delay. Mitigations:

- Keep weights out of the image (volumes).
- Use registry mirrors or caches near the cluster.
- Pre-pull images onto GPU nodes.
- Use image-streaming features where the cloud provides them.
- Order Dockerfile layers so large, rarely changing layers (libraries, model) come first and small, frequently changing layers (code) come last.

---

## 8.8 Running local LLMs in Docker

Terms:

- **LLM (Large Language Model)**: a neural network trained on large amounts of text that predicts the next token (piece of a word), and so can write, summarize, answer and generate code. Hosted LLMs are used through an API. **Open-weight** LLMs can be run locally. Reasons to self-host: privacy, cost at high volume, offline use, customization.
- **Quantization**: storing weights with fewer bits (for example 4-bit instead of 16-bit) to reduce memory use and increase speed at some quality cost. Common formats: **GGUF** (llama.cpp/Ollama), **safetensors** (standard Hugging Face weights format), **AWQ/GPTQ** (GPU-oriented schemes).
- **OpenAI-compatible API**: endpoints such as `POST /v1/chat/completions` that mimic OpenAI's API, so the same client code works with a hosted service or a local container by changing the base URL.

### 8.8.1 Ollama in Docker

**Ollama** is a runtime that downloads and serves models (llama.cpp underneath) with a CLI and an HTTP API on port 11434. Commands are from Ollama's official Docker docs.

CPU only:

```bash
docker run -d -v ollama:/root/.ollama -p 11434:11434 --name ollama ollama/ollama
```

| Part | Meaning |
|---|---|
| `-d` | Run in the background |
| `-v ollama:/root/.ollama` | Store models in the named volume `ollama`; without it, models are lost with the container |
| `-p 11434:11434` | Publish the API. On shared machines use `127.0.0.1:11434:11434`; the Ollama API has no authentication |
| `--name ollama` | Container name for later commands |
| `ollama/ollama` | Image |

NVIDIA GPU (after installing the toolkit, 8.6.2):

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

- `docker exec -it ollama`: runs a command in the running container interactively (`-i` keeps stdin open, `-t` allocates a terminal).
- `ollama run llama3.2`: downloads the model on first use, then opens a chat prompt. `llama3.2` is one example from Ollama's model library.

Call the API from the host:

```bash
curl http://localhost:11434/api/generate -d '{"model":"llama3.2","prompt":"Why is the sky blue?","stream":false}'
```

- `/api/generate`: Ollama's native endpoint. `"stream": false` returns one JSON object instead of a stream of chunks.
- Ollama also exposes an OpenAI-compatible endpoint at `/v1/` (for example `http://localhost:11434/v1/chat/completions`).

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

- Remove the `deploy:` block on a machine without an NVIDIA GPU.
- `OLLAMA_BASE_URL: http://ollama:11434`: Open WebUI reaches Ollama by its Compose service name.
- `"127.0.0.1:3000:8080"`: Open WebUI listens on container port 8080; it is published on host port 3000, loopback only.
- `ghcr.io/open-webui/open-webui:main` is the image name used in the project's docs. `main` is a moving tag; pin a release tag for serious use.
- After the stack is up, pull a model with `docker compose exec ollama ollama pull llama3.2`, then open <http://localhost:3000>.

Without a GPU, generation is slow for anything larger than a small model. Start with small quantized models (1-4B parameters).

Pre-pulling models with a one-shot service:

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

- The `ollama` CLI in this container is a client that talks to the server at `OLLAMA_HOST`; it needs no GPU.
- The container exits after the pull. If the server is not ready yet, the pull can fail; add a retry loop or a healthcheck on `ollama`.

### 8.8.2 Docker Model Runner (DMR)

**Docker Model Runner** pulls, runs and serves AI models through the Docker CLI. Models are stored as **OCI artifacts** (the same packaging standard as images) in Docker Hub or any compatible registry. From Docker's docs (<https://docs.docker.com/ai/model-runner/>):

- It exposes OpenAI-compatible and Ollama-compatible APIs.
- Backends: **llama.cpp** (default, runs everywhere, GGUF), **vLLM** (safetensors; NVIDIA GPUs on Linux x86_64 or Windows via WSL2), **Diffusers** (image generation; NVIDIA on Linux).
- GPU support listed: NVIDIA CUDA (Windows amd64 with driver 576.57+; Linux with driver 575.57.08+), Apple Silicon on macOS, Qualcomm Adreno on Windows arm64, AMD ROCm and Vulkan on Linux with Docker Engine.
- Models come from Docker Hub (the `ai/` namespace), other OCI registries, or Hugging Face.
- The docs state: "The Model Runner API is not authenticated." Do not expose it to a network.

Typical commands. Unverified: the docs page fetched documented only `docker model configure`; the commands below come from general knowledge of the `docker model` CLI. Run `docker model --help` for the authoritative list.

```bash
docker model pull ai/smollm2
docker model list
docker model run ai/smollm2 "Say hello in one sentence."
```

| Command | Meaning |
|---|---|
| `pull` | Downloads the model (an OCI artifact) into the local model store |
| `list` | Shows downloaded models |
| `run` | Runs a one-shot prompt (omit the prompt for interactive chat) |
| `ai/smollm2` | A small model under Docker's `ai/` namespace (also appears in Docker's Compose docs) |

Context size is configured with, for example, `docker model configure --context-size 8192 ai/qwen2.5-coder` (documented example).

**Compose `models` top-level element.** Docker Compose v2.38 or later (v5.x qualifies) can declare models as dependencies next to services (<https://docs.docker.com/ai/compose/models-and-compose/>):

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

- Top-level `models:` defines each model. `model:` is the OCI artifact name Compose pulls and runs through Model Runner.
- `models:` under a service binds them. Compose injects environment variables named after the model: `LLM_URL`, `LLM_MODEL`, `EMBEDDING_MODEL_URL`, `EMBEDDING_MODEL_MODEL`. The app reads them to find the endpoint and model name.
- Long syntax sets custom variable names:

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

| Key | Meaning |
|---|---|
| `endpoint_var` / `model_var` | Rename the injected variables |
| `context_size` | Maximum number of tokens the model considers at once. Keep it as small as the use case allows; larger values need more memory |
| `runtime_flags` | Raw flags for the inference engine. For embedding models served on `/v1/embeddings`, the docs say to add `--embeddings` |

Unverified: one secondary source claimed the older `provider:` service syntax for models is deprecated; another said it is still supported. The official page documents only `models:`; use that for new work.

With `models:`, one `docker compose up` starts the app and its local model without a separate Ollama container.

### 8.8.3 llama.cpp server

**llama.cpp** is the C/C++ engine behind many local-LLM tools. It runs GGUF models on CPUs and many GPUs. The project publishes container images with an OpenAI-compatible HTTP server:

```bash
docker run --rm -p 8080:8080 -v "$PWD/models:/models" \
  ghcr.io/ggml-org/llama.cpp:server \
  -m /models/my-model.gguf --host 0.0.0.0 --port 8080
```

- `-v "$PWD/models:/models"`: mounts a folder containing a downloaded `.gguf` file.
- Arguments after the image name go to the server: `-m` model path, `--host 0.0.0.0` listen on all interfaces inside the container, `--port 8080`.
- GPU image variants exist (tags such as `server-cuda`). Unverified: current tag names. Check the llama.cpp repository's Docker documentation.

### 8.8.4 vLLM (high-throughput GPU serving)

**vLLM** is an inference engine focused on throughput (continuous batching and efficient KV-cache memory management, "PagedAttention") for serving many concurrent users on GPUs. Command from the official docs (<https://docs.vllm.ai/en/stable/deployment/docker/>):

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
| `-v ~/.cache/huggingface:/root/.cache/huggingface` | Share the host's Hugging Face cache so weights download once |
| `--env "HF_TOKEN=$HF_TOKEN"` | Pass the token for gated models (the shell expands `$HF_TOKEN`) |
| `-p 8000:8000` | Publish the OpenAI-compatible API |
| `--ipc=host` | Share the host's shared-memory namespace (PyTorch tensor-parallel workers need a lot of shared memory). `--shm-size` is the alternative |
| `vllm/vllm-openai:latest` | OpenAI-compatible server image. Pin a version for production |
| `--model Qwen/Qwen3-0.6B` | Argument to vLLM: the model to load (small example from vLLM's docs) |

Test with `curl http://localhost:8000/v1/models`. Any OpenAI client works with `base_url="http://localhost:8000/v1"`.

vLLM's docs also describe building the image from source and ARM64 builds for Grace-Hopper/Blackwell platforms. On an x86_64 host, cross-building needs QEMU handlers registered by a `--privileged` helper container. `--privileged` is acceptable for that one-off step; never use it for serving.

### 8.8.5 Hugging Face TGI

**Text Generation Inference (TGI)** is Hugging Face's inference server (`ghcr.io/huggingface/text-generation-inference`). The Hugging Face Inference Endpoints docs say TGI entered **maintenance mode** (as of Dec 11, 2025): only minor bug fixes and docs improvements, with vLLM or SGLang recommended for new deployments. Unverified: some third-party sources claim the repository was archived in March 2026; this was not confirmed from official sources. Existing TGI deployments keep working; prefer vLLM (or SGLang) for new projects.

### 8.8.6 NVIDIA Triton Inference Server

**Triton** is NVIDIA's general-purpose model server. It serves many model types (TensorRT, ONNX, PyTorch, Python backends, and LLM backends including vLLM and TensorRT-LLM) with dynamic batching and multi-model scheduling. Images come from NVIDIA's NGC registry with date-style tags:

```bash
docker run --gpus=all --rm -p 8000:8000 -p 8001:8001 -p 8002:8002 \
  -v "$PWD/model_repository:/models" \
  nvcr.io/nvidia/tritonserver:26.09-py3 \
  tritonserver --model-repository=/models
```

- Tag `26.09-py3` uses the YY.MM release naming. 26.09 was the newest release found in the project's release notes. Variants such as `26.09-vllm-python-py3` exist for LLM backends.
- Ports 8000, 8001 and 8002 are HTTP, gRPC and Prometheus metrics (Triton defaults).
- `-v ...:/models`: provides the **model repository**, a directory layout Triton scans: `model_name/1/model.onnx` plus a `config.pbtxt`.
- `tritonserver --model-repository=/models`: starts the server.
- NGC images are very large. Unverified: verify the exact tag on <https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver>; one catalog snapshot seen was stale.

### 8.8.7 Which one should I choose?

```
                      Serving MANY concurrent users on NVIDIA GPUs?
                                   |                       |
                                  YES                      NO
                                   |                       |
               vLLM (or SGLang);                  Dev laptop / quick local use?
               Triton if multi-model /                    |               |
               multi-framework needed                    YES              NO (small private service)
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

Most production AI apps call a hosted model from a cloud AI provider.

### 8.9.1 Never put API keys in an image

An API key is a bearer credential: anyone holding it can spend the owner's money. Images are shareable and inspectable (`docker history`, layer extraction).

| Do not | Why |
|---|---|
| `ENV OPENAI_API_KEY=sk-...` in the Dockerfile | Visible in `docker inspect`, `docker history` and every layer |
| `ARG API_KEY` then `RUN ... $API_KEY` | `ARG` values are recorded in image metadata/history |
| `COPY .env /app/.env` | The file stays in a layer permanently |
| Commit `.env` to git | Leaks through repo history ([Chapter 10](10-security.md): rotation is the only real fix) |

| Do | How |
|---|---|
| Runtime env var from outside | `docker run -e API_KEY` (forwards from the shell), or Compose `environment:` / `env_file:` pointing to a git-ignored file |
| Docker/Compose secrets (files) | Mounted at `/run/secrets/<name>`; the app reads the file |
| Secret manager in production | AWS Secrets Manager, GCP Secret Manager, Azure Key Vault, HashiCorp Vault, Kubernetes Secrets (ideally with external secret operators) |

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

- `secrets:` under the service grants access to the named secret.
- Top-level `secrets:` maps the name to a local file (git-ignore the `secrets/` folder).
- The secret appears in the container as the file `/run/secrets/llm_api_key`. Plain Compose bind-mounts the file read-only; Docker Swarm keeps secrets in memory. It is not an environment variable, so the value does not appear in `docker inspect` output.
- The app reads `LLM_API_KEY_FILE` to find the file. Many official images support this `*_FILE` convention. In custom code:

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

Many servers (vLLM, Ollama, llama.cpp, Docker Model Runner, LiteLLM and many cloud providers) imitate the OpenAI Chat Completions API, so provider-agnostic code is possible:

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

- `base_url` decides where requests go. Switching from a hosted service to a local container is a one-variable change.
- Local servers usually ignore the API key, but the client library requires some string.
- `model` is the name the target server understands: for Ollama `llama3.2`; for vLLM the Hugging Face name it was launched with; for hosted services the provider's model ID (look it up in the provider's docs).
- The official `openai` Python package also reads `OPENAI_BASE_URL` and `OPENAI_API_KEY` from the environment, so code changes can sometimes be avoided.

Other providers have their own SDKs and often their own key variable names (for example, Anthropic's SDK reads `ANTHROPIC_API_KEY`). The container rule is the same: key in at runtime, never in the image.

### 8.9.3 Network considerations for outbound calls

- Containers reach the internet through NAT by default, so hosted APIs work. In locked-down environments use an egress proxy and pass `HTTPS_PROXY` / `NO_PROXY`.
- Set timeouts and retries in the client. LLM calls can take tens of seconds; a hung upstream must not hold all workers.
- For rate limits, add a cache (Redis) and a queue for bursts.
- **LiteLLM** is an open-source proxy/gateway that presents one OpenAI-compatible endpoint in front of many providers. Run it as a service so that only it holds provider keys, budgets and fallbacks.

---

## 8.10 RAG in Compose: FastAPI + Qdrant + embeddings + LLM

### 8.10.1 Terms, plainly

- **Embedding**: a list of numbers (a *vector*, often hundreds to thousands long) that represents the meaning of a piece of text (or an image). Texts with similar meaning produce vectors that are close together.
- **Embedding model**: a small neural network that converts text to an embedding. For example `all-MiniLM-L6-v2` produces 384 numbers per text. Verify the dimension of the chosen model; the vector DB collection must match it.
- **Vector database**: a database that stores embeddings and quickly finds the *k* vectors nearest to a given one, using approximate nearest-neighbour indexes such as HNSW. Examples: **Qdrant**, **Chroma**, **Milvus**, **Weaviate**, **pgvector** (a PostgreSQL extension that adds a vector type).
- **RAG (Retrieval-Augmented Generation)**: a pattern that makes an LLM answer from your own documents without retraining:
  1. **Ingest**: split documents into chunks, embed each chunk, store in the vector DB.
  2. **Retrieve**: embed the user's question, fetch the nearest chunks.
  3. **Generate**: put those chunks into the prompt ("use only this context") and ask the LLM.
- **Chunking**: splitting long documents into passages (a few hundred tokens each), because embeddings and prompts work on limited-size text.

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

| Part | Meaning |
|---|---|
| `${LLM_BASE_URL:-http://ollama:11434/v1}` | Compose substitution with a default: use `LLM_BASE_URL` from the environment or `.env` if set, otherwise the local Ollama container. One variable switches between local and hosted LLM. |
| `read_only: true` + `tmpfs: [/tmp]` | Read-only root filesystem with an in-memory writable `/tmp`. The app must not need to write elsewhere. See [Chapter 10](10-security.md). |
| `cap_drop: [ALL]` | Drops every Linux capability (root-like privilege). A web API needs none. |
| `security_opt: [no-new-privileges:true]` | Processes cannot gain privileges through setuid binaries. |
| `worker` reuses the same build with a different `command` | One image, two roles (API and background worker). |
| `qdrant` image, `/qdrant/storage` volume | Qdrant stores collections there. REST on 6333; gRPC on 6334 is not published here. |
| `qdrant` healthcheck with `/dev/tcp` | Pure-bash TCP probe that does not need `curl` in the image. If the pinned Qdrant tag lacks bash, use a different probe or drop the healthcheck; verify against the pinned tag. |
| `redis` with `--save "" --appendonly no` | Redis as a pure cache, no persistence. If Redis is a job queue, enable persistence or queued jobs are lost on restart. |
| No `ports` on `redis` / `ollama` | Reachable only inside the Compose network, which reduces the exposed surface. |
| `EMBED_MODEL_PATH: /models/minilm` | Model baked into the image (8.7.2) or provided by a volume. |

Alternatives for the vector store:

| Choice | Image | When |
|---|---|---|
| **Qdrant** | `qdrant/qdrant` | Dedicated, fast, good filtering |
| **pgvector** | `pgvector/pgvector:pg17` (Postgres with the extension) | Postgres already in use; SQL joins, transactions, one database to back up |
| **Chroma** | `chromadb/chroma` (port 8000) | Prototyping, notebooks |
| **Milvus / Weaviate** | Their official images/compose files | Large scale or advanced features |

Unverified: the `pgvector/pgvector` and `chromadb/chroma` image names come from general knowledge of these projects. Confirm tags on Docker Hub before use.

With **Docker Model Runner**, the `ollama` service can be replaced by a `models:` block (8.8.2); read `LLM_URL` / `LLM_MODEL` injected by Compose.

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

- `upsert`: inserts or updates points (vector + payload metadata).
- `search ... limit=4`: returns the 4 nearest chunks.
- The `prompt` is then sent to the LLM with the client from 8.9.2.
- Exact client method names vary between `qdrant-client` versions; this is a conceptual sketch. [Chapter 9](09-hands-on-ai-app-walkthrough.md) builds a working version.

---

## 8.11 MCP servers in containers

**MCP (Model Context Protocol)** is an open protocol that lets AI assistants and agents call external **tools** and read external **resources** through a standard interface. An **MCP server** exposes tools (for example "query this database", "create a GitHub issue", "read this folder"). An **MCP client** is the AI application (an IDE assistant, a desktop chatbot, an agent).

MCP servers are third-party code that an AI drives, often with credentials. A container provides:

- Dependency isolation (no Node/Python version conflicts).
- A boundary around what the server can touch: no host filesystem unless mounted, restricted network if configured.
- Reproducible installs (pinned image tag or digest).

### 8.11.1 stdio MCP servers via `docker run -i`

Many MCP servers use **stdio** (standard input/output): the client launches the server as a child process and exchanges JSON messages through its stdin/stdout. With Docker, the process is `docker run -i --rm ...`. Illustrative client configuration (each client documents its own file location):

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

- `command` / `args`: the client runs `docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN <image>`.
- `-i`: keeps stdin open (required for stdio). Do not use `-t`; a TTY corrupts the JSON stream.
- `--rm`: removes the container when the client disconnects.
- `-e GITHUB_PERSONAL_ACCESS_TOKEN` (no value): forwards the variable from the client's `env` block into the container.
- Use a least-privilege token (only the scopes the tool needs), because the AI can invoke anything the token allows.
- Unverified: the image name follows the GitHub MCP server project's published container. Verify the current image name and flags in its README.

For servers that access files, mount only what is needed, read-only where possible:

```
"args": ["run","-i","--rm","-v","/home/me/project:/workspace:ro","some/mcp-filesystem-server","/workspace"]
```

- `:ro`: read-only mount, so the AI cannot modify the files.

### 8.11.2 Docker MCP Toolkit, MCP Gateway and Catalog

Docker provides an **MCP Catalog** (a list of containerized MCP servers), an **MCP Toolkit** (a management UI in Docker Desktop, version 4.62 or later per the docs) and an **MCP Gateway** (a single endpoint the AI client connects to; it launches and routes to the enabled servers). From Docker's docs (<https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/>):

- Servers are grouped in **profiles**, named collections that can be switched per project.
- Each tool runs in its own container, limited to 1 CPU and 2 GB of memory.
- Servers have no host filesystem access by default; mounts are granted explicitly.
- Official `mcp/` images in the catalog are built and signed by Docker and include an SBOM.
- Requests and responses containing secrets are blocked at runtime (per the docs).
- The Toolkit handles OAuth flows for services such as GitHub and stores credentials.
- Two documented CLI commands: `docker mcp gateway run --profile <name>` and `docker mcp client connect vscode --profile <name>`.

Security: an MCP tool server is a remote-control surface for an LLM. Prompt injection (malicious text in a web page or document that instructs the model to misuse its tools) is a real risk. Containers reduce the blast radius. Also use least-privilege tokens, read-only mounts, network restrictions, and human approval for dangerous actions.

---

## 8.12 Agents and sandboxing code execution

An **AI agent** is a loop in which an LLM decides on actions (call a tool, run code, browse), the application executes them, and the results are fed back. Code execution is the most dangerous tool: the code is written by a model and may be wrong or steered by prompt injection. Never run it on the host or in the API container.

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
| `--rm` | Deletes the container afterward |
| `--network none` | No network (no exfiltration, no downloads). Add a restricted network only if essential |
| `--read-only` | Read-only root filesystem |
| `--tmpfs /tmp:rw,size=64m,noexec` | Small in-memory scratch space; `noexec` forbids running binaries from it |
| `--memory 256m --memory-swap 256m` | Hard memory cap; swap equal to memory disables swap (prevents memory bombs) |
| `--cpus 0.5` | At most half a CPU core |
| `--pids-limit 64` | At most 64 processes/threads (stops fork bombs) |
| `--cap-drop ALL` | No special privileges |
| `--security-opt no-new-privileges:true` | Blocks privilege escalation through setuid programs |
| `--user 65534:65534` | Runs as `nobody` |
| `-v "$PWD/job:/work:ro"` | Provides the code as a read-only mount |
| `timeout 10 python ...` | Kills the script after 10 seconds (wall-clock limit enforced inside) |

A container is not a perfect boundary: all containers share the host kernel, so a kernel bug can allow escape. For untrusted code from the public internet, add a stronger layer: a user-space kernel such as **gVisor** (`--runtime=runsc` once installed), microVMs such as **Firecracker** or **Kata Containers**, or a managed code-sandbox service. See [Chapter 10](10-security.md).

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

- `docker.from_env()`: connects to the local Docker daemon through the socket.
- Each argument mirrors a `docker run` flag from the table above.
- Warning: giving the API container access to the Docker socket is dangerous; the socket is root-equivalent ([Chapter 10](10-security.md)). Safer designs: run the sandbox runner as a separate, isolated service on a separate host or VM, use a rootless Docker daemon dedicated to sandboxes, or use a managed sandbox service.

---

## 8.13 Jupyter in Docker and Dev Containers

**Jupyter** notebooks mix code, output and notes. Running them in a container gives a reproducible environment. The community **Jupyter Docker Stacks** images (hosted on Quay since 2023) bundle common stacks:

```bash
docker run --rm -p 127.0.0.1:8888:8888 \
  -v "$PWD/work:/home/jovyan/work" \
  quay.io/jupyter/scipy-notebook:latest
```

- `quay.io/jupyter/scipy-notebook`: Jupyter plus numpy, pandas, scipy and matplotlib. Other flavours: `minimal-notebook`, `pytorch-notebook`, `tensorflow-notebook`, `datascience-notebook`. Pin a dated tag for reproducibility.
- The container logs a URL containing a login token. Open that URL; it uses `127.0.0.1:8888` because the port is published to loopback only.
- `-v "$PWD/work:/home/jovyan/work"`: mounts a host folder so notebooks persist. `jovyan` is the image's default user.
- Do not publish Jupyter on `0.0.0.0` without authentication on a shared network; it provides a code-execution shell.

For GPU use, add `--gpus all` and use a CUDA-enabled image (for example the `pytorch-notebook` CUDA variants; see the Jupyter Docker Stacks docs for tag names).

**Dev Containers** (the `devcontainer.json` standard supported by VS Code, JetBrains IDEs and GitHub Codespaces) run the editor environment inside a container so everyone has the same tools:

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

| Key | Meaning |
|---|---|
| `build.dockerfile` | Builds the dev image |
| `runArgs` | Raw `docker run` flags (GPU and shared memory) |
| `mounts` | Attaches the model-cache volume |
| `customizations.vscode.extensions` | Installs editor extensions inside the container |
| `postCreateCommand` | Runs once after creation to install project dependencies |

A new teammate opens the folder, selects "Reopen in Container", and gets the same toolchain.

---

## 8.14 Training jobs in containers

Training is a batch job: start, run for minutes to days, write results, exit. Containers suit this.

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

| Piece | Purpose |
|---|---|
| `--gpus all` | Training needs the GPU(s) |
| `--shm-size=16g` | PyTorch `DataLoader(num_workers>0)` passes batches between processes through shared memory (`/dev/shm`). Docker's default is 64 MB, which causes "Bus error" / "unable to write to file </torch_...>" crashes. Set several GB. |
| `--ipc=host` (alternative) | Shares the host IPC namespace, removing the limit; common for multi-GPU training with NCCL. Reduces isolation, so prefer `--shm-size` unless needed |
| `-v $PWD/data:/data:ro` | Datasets mounted read-only: the job cannot corrupt them, and large datasets are not copied into an image |
| `-v $PWD/runs:/runs` | Checkpoints and logs written to a mounted volume, so they survive container exit or crash |
| `hf-cache` volume | Caches pretrained weights across runs |
| `MLFLOW_TRACKING_URI` | Sends metrics to the tracking server (8.15) |

- **Checkpointing**: containers are disposable and GPU nodes (especially spot/preemptible) can disappear. Save checkpoints regularly to the volume, and make the training script resume from the latest checkpoint if one exists.
- **Multi-GPU / multi-node**: single-node multi-GPU uses `torchrun --nproc_per_node=4 train.py` inside one container with `--gpus all --ipc=host`. Multi-node needs an orchestrator (Kubernetes with Kubeflow/Volcano/Ray, Slurm with Pyxis/Enroot, or managed services) and a fast network (InfiniBand/EFA).
- **File ownership**: files written to a bind mount by a container running as a different UID are owned by that UID on the host. Use `--user "$(id -u):$(id -g)"` for dev, or fix ownership in the pipeline.

---

## 8.15 Experiment tracking: MLflow in Compose

**Experiment tracking** records each training run's parameters, metrics and artifacts (model files) so runs can be compared and reproduced. **MLflow** is a popular open-source tracker.

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

- **Backend store** (`--backend-store-uri`): holds run metadata (parameters, metrics) in PostgreSQL.
- **Artifact store** (`--artifacts-destination`): holds files (models, plots). Here a local volume; use S3/GCS/Azure Blob in production.
- `--host 0.0.0.0`: lets other containers reach the server. The host port is bound to loopback only.
- `pip install psycopg2-binary` at start is a demo shortcut. For real use, build an image with the driver included.
- Unverified: the tag `v3.1.0` is an example; the current MLflow image tag and default flags were not verified. Check the MLflow docs for current Docker/server instructions. Recent MLflow versions added server security settings (such as allowed hosts) that may need extra flags when accessed through a name other than localhost.
- The training container logs with `mlflow.log_metric(...)` after setting `MLFLOW_TRACKING_URI=http://mlflow:5000` and joining the same Compose network (or using the host URL if running outside Compose).

---

## 8.16 Async workers: Celery/RQ + Redis

LLM and ML tasks are slow. If the HTTP API does the work inline, requests time out and the server stalls. The usual pattern: the API enqueues a **job** into a **queue**, returns a job ID immediately, and a **worker** process does the work.

```
Client --POST /summarize--> API --enqueue--> Redis (queue) --> Worker (GPU or CPU)
   ^                          |                                     |
   |<--- 202 Accepted {id} ---+                                     v
   +------ GET /jobs/{id} (poll) or websocket/SSE <---- result stored in Redis/DB
```

- **Celery**: a feature-rich Python task queue. **RQ** (Redis Queue) is simpler. Dramatiq, Arq and Taskiq are alternatives.
- **Redis** here is the *broker* (message transport) and often the result backend.

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

- `--appendonly yes`: Redis persists data to disk, so queued jobs survive restarts.
- `--concurrency=2`: runs two worker processes. For GPU work keep this low (each process loads the model into VRAM), or use one process with batching.
- Scale workers independently of the API with `docker compose up -d --scale worker=3`. Replicas compete for the GPU unless devices are assigned carefully; on one GPU, prefer one worker with internal batching.

---

## 8.17 Streaming responses behind nginx (timeouts and buffering)

LLM answers arrive token by token. Apps stream them with **SSE (Server-Sent Events)**, a one-way HTTP stream, or with WebSockets. A reverse proxy can break streaming by buffering the whole response before forwarding it, and by timing out long requests.

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

| Directive | Purpose |
|---|---|
| `proxy_pass http://api:8000;` | Forwards to the Compose service named `api` |
| `proxy_http_version 1.1;` + `proxy_set_header Connection "";` | Required for streaming and keep-alive (HTTP/1.0 to the upstream closes each response) |
| `proxy_buffering off;` | Key setting: nginx forwards chunks immediately instead of collecting them |
| `proxy_cache off;` | Never caches a stream |
| `proxy_read_timeout 300s;` | Default is 60 s; slow first tokens or long generations would otherwise be cut with 504 |
| `chunked_transfer_encoding on;` | Allows streaming without a known length |

The app can also help: FastAPI `StreamingResponse(..., media_type="text/event-stream")`, and the response header `X-Accel-Buffering: no` tells nginx not to buffer that response. Cloud load balancers have their own idle timeouts (for example 60 seconds by default on some); raise them or send periodic heartbeat comments (`: ping`) in the stream.

---

## 8.18 Observability for AI services

Three signals:

**1. Logs.** Write structured JSON to stdout. Docker captures it and a collector ships it.

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

- Both `print` and a JSON formatter reach `docker logs`. JSON lets Loki, ELK or CloudWatch search by field (for example `request_id`, `latency_ms`, `model`).
- Do not log prompts, documents or API keys that contain personal or confidential data.
- Limit local log growth with Compose `logging: {driver: json-file, options: {max-size: "10m", max-file: "3"}}`. Otherwise the default json-file driver can fill the disk.

**2. Metrics.** Numbers over time. Expose `/metrics` with `prometheus-client`, scrape it with **Prometheus**, graph it in **Grafana**. Useful AI metrics: request rate, latency (p50/p95/p99), time to first token, tokens per second, queue depth, error rate, cache hit rate, GPU utilization and memory. NVIDIA's **DCGM exporter** publishes GPU metrics in Prometheus format (image from NVIDIA's registry; check NVIDIA's docs for the current tag). vLLM and Triton also expose Prometheus metrics endpoints (Triton on port 8002 by default).

**3. Traces.** Follow one request across services (OpenTelemetry). LLM-specific tools (Langfuse, OpenLLMetry, Phoenix) record prompts, tool calls and costs; many have official Compose files.

Prometheus + Grafana in Compose:

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

- `scrape_interval`: how often Prometheus pulls `/metrics`.
- `targets`: Compose service name and port.

---

## 8.19 Reproducible builds, CI/CD, and multi-arch

### 8.19.1 Reproducibility checklist for ML images

| Item | How |
|---|---|
| Pin the base image | `FROM python:3.12-slim@sha256:<digest>`: tags move, digests do not |
| Lock Python dependencies | `uv lock` / `uv sync --locked`, or `pip-compile --generate-hashes` + `pip install --require-hashes -r requirements.txt` |
| Pin system packages when it matters | `apt-get install pkg=version` (or accept drift and rebuild regularly) |
| Pin model revisions | `from_pretrained("org/model", revision="<commit-sha>")` so upstream edits do not change behaviour |
| Pin the framework/CUDA pair | Choose the PyTorch wheel index with an explicit CUDA version (for example the `cu128` index) |
| Record provenance | Tag images with the git SHA; generate SBOM/provenance attestations ([Chapter 10](10-security.md)) |
| Seed randomness for tests | Fixed seeds in evaluation (GPU kernels can still be nondeterministic) |

### 8.19.2 CI/CD pipeline for an ML image

```
git push -> CI: lint + unit tests (CPU) -> build image (BuildKit cache) -> scan (Trivy/Scout)
          -> push to registry (tag: git SHA) -> integration test (compose up + smoke) -> deploy (staging) -> promote
```

GitHub Actions sketch:

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

| Key | Meaning |
|---|---|
| `permissions` | Least privilege: read the code, write packages |
| `docker/setup-buildx-action` | Enables BuildKit's advanced features |
| `login-action` | Signs in to GitHub Container Registry with the automatic `GITHUB_TOKEN` |
| `build-push-action` | Builds and pushes; the tag is the commit SHA (immutable, traceable) |
| `cache-from/to: type=gha` | Stores layer cache in GitHub's cache so ML dependency layers are not rebuilt each time |
| `provenance: mode=max`, `sbom: true` | Attach SLSA provenance and an SBOM as attestations ([Chapter 10](10-security.md)). Docker's docs warn that `mode=max` records build-argument values, so never pass secrets as build args |

Action versions (`@v4`, `@v3`, `@v6`) are examples; pin to commit SHAs in sensitive pipelines.

Standard GitHub-hosted runners have no GPU. GPU tests need self-hosted GPU runners or a provider's GPU runners. Keep most tests CPU-only with small models and run a separate, less frequent GPU smoke test.

### 8.19.3 Multi-architecture images (Apple Silicon dev, amd64 prod)

Apple Silicon Macs are **arm64**. Most cloud GPU servers are **amd64** (x86-64), and some newer data-centre superchips are arm64. An image built for one architecture does not run natively on the other (it may run slowly through emulation or fail). A **multi-arch image** is one tag that points to per-architecture variants; Docker picks the right one automatically.

```bash
docker buildx create --use --name multi
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/myorg/rag-api:1.0.0 \
  --push .
```

- `buildx create --use`: creates and selects a BuildKit builder that can do multi-platform builds.
- `--platform linux/amd64,linux/arm64`: builds both variants.
- `--push`: pushes the combined manifest directly (a multi-arch result cannot be loaded into the classic local image store as a single image).
- Cross-architecture builds use QEMU emulation (slow for compiling) unless native builders exist for each architecture.

ML cautions: some wheels (certain CUDA or niche libraries) exist only for amd64, so the arm64 build may fail or need a different dependency set. Common split: CPU API image is multi-arch; GPU model-server image is amd64 only (a Mac container has no GPU anyway).

Verify supported platforms with `docker buildx imagetools inspect ghcr.io/myorg/rag-api:1.0.0`, which lists each platform and digest.

---

## 8.20 Image-size tricks (summary)

| Trick | Typical effect |
|---|---|
| CPU-only PyTorch wheel for non-GPU services | ~2.5 GB or more (community reports, 8.3.4) |
| `python:*-slim` instead of full `python:*` | Hundreds of MB |
| Multi-stage build; compilers only in the builder | Hundreds of MB to GBs |
| `--no-cache-dir` (pip) / cache mounts (uv); apt lists removed in the same `RUN` | Caches kept out of layers |
| `.dockerignore` | Smaller context, fewer accidental leaks |
| Model weights in volumes, not the image | GBs |
| `nvidia/cuda:*-runtime` instead of `*-devel` in the final stage | Several GB |

Measure with `docker images`, `docker history <image>`, `docker system df`, or [dive](https://github.com/wagoodman/dive).

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
| **Single GPU VM** (AWS EC2 GPU instances, GCP Compute Engine with GPUs, Azure NC/ND-series, Lambda Cloud, RunPod pods, Vast.ai, CoreWeave) | Rent a GPU machine, install driver + Docker + toolkit (or use a GPU-ready image), run Compose | Experiments, small production | Patching, uptime, idle cost |
| **Provider-managed / serverless GPU** (RunPod serverless, Modal, Replicate, Cloud Run with GPUs, Azure Container Apps GPU, AWS SageMaker endpoints) | Provide an image; the provider runs it on GPUs, often scaling to zero | Spiky inference, little ops | Cold starts (image pull + model load), per-provider constraints |
| **ECS on EC2 GPU / EKS, GKE, AKS** | Container orchestration on GPU instances | Teams on that cloud | GPU capacity, quotas |
| **Kubernetes + NVIDIA GPU Operator** | Automates drivers, container toolkit, device plugin and DCGM monitoring on K8s nodes | Multi-team platforms | Operational complexity |

Unverified: providers are named as examples of categories. Feature sets, GPU availability and prices change quickly and no provider's current offering was verified.

### 8.21.3 Kubernetes and GPUs in 60 seconds

**Kubernetes (K8s)** is an orchestrator: it takes a declaration such as "run 3 replicas of this image", schedules the replicas across machines, restarts failures and routes traffic ([Chapter 11](11-production-and-enterprise.md)). To use GPUs:

1. GPU nodes need the NVIDIA driver and container toolkit. The **NVIDIA GPU Operator** automates this.
2. The **NVIDIA device plugin** advertises GPUs as a schedulable resource, `nvidia.com/gpu`.
3. The pod requests the resource:

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

- `limits: nvidia.com/gpu: 1`: asks the scheduler for a node with one free GPU and attaches it. GPUs are requested as whole units by default; sharing techniques such as MIG and time-slicing exist but are advanced.
- Add a PersistentVolumeClaim for the model cache, a readiness probe that waits for the model to load, and an HPA/KEDA autoscaler keyed on queue depth or request concurrency, not CPU.

### 8.21.4 Cost considerations

| Factor | Guidance |
|---|---|
| GPUs are the dominant cost | Turn them off when idle; autoscale to zero where cold starts are tolerable |
| Spot/preemptible instances | 60-90% cheaper (typical range) but can be reclaimed; needs checkpointing and retries |
| Right-size the model | Quantized or smaller models can serve the same task for much less |
| Batching | Continuous batching (vLLM) serves many users per GPU |
| Hosted API vs self-host | Self-hosting usually wins only at sustained high utilization or with privacy needs; compute break-even with real traffic |
| Egress and storage | Keep registry and compute in the same region; delete old image tags |
| Cache | Cache embeddings and repeated answers (Redis) |

---

## 8.22 Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Leaked credentials | `OPENAI_API_KEY` / HF tokens baked into the image | Runtime env or secrets; rotate if leaked ([Chapter 10](10-security.md)) |
| Image 2.5 GB or more larger than needed | CUDA PyTorch wheel in an API that never uses a GPU | CPU wheel index (8.3.4) |
| Build failures, slow builds | Alpine for ML | `python:*-slim` |
| Multi-GB bloat | `nvidia/cuda:*-devel` as the final image | Use `runtime` in the final stage, or a Python slim base with wheels |
| Silent CPU fallback; slow | Missing `--gpus all` (or Compose device reservation) | Run `nvidia-smi` / `torch.cuda.is_available()` at startup and fail loudly |
| GPU not available during build | `--gpus` in `docker build` is not supported | Do GPU work at run time |
| DataLoader crashes | `/dev/shm` too small | `--shm-size` or `--ipc=host` |
| GBs re-downloaded on every restart | No volume for the model cache | Mount `HF_HOME` / the Ollama volume |
| Silent upgrade changes behaviour or breaks | `latest` tags for vLLM/Ollama/Open WebUI in production | Pin versions/digests |
| Unauthenticated code or inference access | Ollama/Model Runner/Jupyter ports exposed to the network | Bind to `127.0.0.1`; put an authenticated proxy in front |
| Checkpoints lost on exit | Training checkpoints stored inside the container | Write to a mounted volume |
| Host or API compromise | Agent-generated code run in the API container | Disposable locked-down sandbox, preferably on an isolated host/VM |
| Published port does not work | App bound to `127.0.0.1` inside the container | Bind `0.0.0.0` inside; restrict exposure at the publish step |
| CPU-only, slow on Mac | Expecting containers to use the Apple GPU | Native Ollama / Docker Model Runner on Mac, or a cloud GPU |
| Streaming works locally but not behind a proxy | Buffering/timeouts | `proxy_buffering off`, raise timeouts (8.17) |

---

## 8.23 Try it yourself

1. **No-GPU path.** Run Ollama in CPU mode, pull a small model and chat:
   ```bash
   docker run -d -v ollama:/root/.ollama -p 127.0.0.1:11434:11434 --name ollama ollama/ollama
   docker exec -it ollama ollama run llama3.2
   ```
   Clean up with `docker rm -f ollama`. The `ollama` volume keeps the model; `docker volume rm ollama` reclaims the disk space.
2. **Docker Model Runner.** Check `docker model --help`; pull and run `ai/smollm2`.
3. **Shrink an image.** Build the Example 1 image with the default `torch` and with the CPU index; compare sizes with `docker images`.
4. **Compose RAG.** Build the stack in 8.10, ingest three text files, ask questions, then switch `LLM_BASE_URL` to a hosted API (key in a secret file).
5. **Streaming.** Put nginx with default settings in front of an SSE endpoint, observe buffering, then fix it with `proxy_buffering off`.
6. **Sandbox.** Run the 8.12.1 command against a script that reaches for the internet and one that allocates 1 GB. Both should fail.
7. **With a GPU.** Install the toolkit (8.6.2), run `nvidia-smi` in a container, then serve a small model with vLLM and call `/v1/models`.

---

## 8.24 Chapter summary

- Split the app: thin CPU API, separate GPU model server, vector DB, cache, queue/worker, UI.
- Default to `python:slim` plus wheels. Use `nvidia/cuda` `devel` only to compile, and CPU-only wheels where no GPU is used.
- GPUs need a host driver and the NVIDIA Container Toolkit (`--gpus all` or Compose device reservations). macOS containers cannot use Metal.
- Keep weights out of images; use cache volumes, init steps and offline mode.
- Ollama and Docker Model Runner suit local dev; vLLM or Triton suit production. TGI is in maintenance mode.
- Keys come from the environment or secrets at runtime, never from the image.
- Sandbox agent code, bind sensitive ports to loopback, pin versions, and set `--shm-size`, checkpoints and proxy streaming settings.

Next: [Chapter 9: Hands-on AI App Walkthrough](09-hands-on-ai-app-walkthrough.md). Security for this chapter's topics is covered in [Chapter 10](10-security.md).
