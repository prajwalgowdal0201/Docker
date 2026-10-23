# Docker Complete Guide — From Zero to Production for AI Developers

A complete, practical Docker course in plain English: what Docker is, how every command works, how to containerize an AI application, and how Docker is used in real production environments. Every technical term is explained, every command is explained, and the guide ends with a runnable AI application you build and deploy yourself.

> **Who this is for:** a developer who understands *why* Docker exists but wants real, job-ready depth — especially for AI/LLM applications.

## How to use this guide

Read the chapters in order and **type the commands yourself**. Chapters 3, 4, 6 and 9 contain labs. If you only have one evening, do Chapter 9 first (it works on its own), then return to the theory.

## Contents

| # | Chapter | You will learn |
|---|---|---|
| 1 | [What is Docker?](docs/01-what-is-docker.md) | Containers vs VMs, images, layers, architecture, Docker Compose overview |
| 2 | [Installation & operating systems](docs/02-installation-and-operating-systems.md) | Windows/macOS/Linux setup, **which OS to use for production** |
| 3 | [Core CLI commands](docs/03-core-cli-commands.md) | Every everyday `docker` command and flag explained |
| 4 | [Dockerfile deep dive](docs/04-dockerfile-deep-dive.md) | Writing efficient, secure images; multi-stage builds; caching |
| 5 | [Storage & networking](docs/05-storage-and-networking.md) | Volumes, bind mounts, networks, DNS, port publishing |
| 6 | [Docker Compose](docs/06-docker-compose.md) | Multi-container apps, health checks, profiles, dev/prod files |
| 7 | [Who needs Docker?](docs/07-who-needs-docker.md) | Project types, **does an Android app need Docker?**, when *not* to use it |
| 8 | [Docker for AI & ML](docs/08-docker-for-ai-and-ml.md) | GPUs, model weights, Ollama, RAG stacks, serving LLMs |
| 9 | [**Hands-on: build & deploy an AI app**](docs/09-hands-on-ai-app-walkthrough.md) | The full workflow, step by step, with real outputs |
| 10 | [Security](docs/10-security.md) | Hardening, secrets, scanning, supply chain |
| 11 | [Production & enterprise](docs/11-production-and-enterprise.md) | CI/CD, registries, Kubernetes/ECS/Cloud Run, logging, monitoring, rollbacks |
| 12 | [Troubleshooting & cheat sheet](docs/12-troubleshooting-and-cheatsheet.md) | Fix the 30+ most common errors; one-page command reference |
| 13 | [Interview Q&A, roadmap, glossary](docs/13-interview-questions-roadmap-glossary.md) | 60+ questions, learning path, portfolio tips, A–Z glossary |
| 14 | [Real-world case study](docs/14-real-world-case-study.md) | Critically reading a real multi-service production deployment |

## The sample project

[`sample-ai-app/`](sample-ai-app/) — a FastAPI AI chat service with Postgres (history), Redis (cache + rate limit), nginx (reverse proxy) and an optional local LLM via Ollama. It demonstrates multi-stage builds, build-time tests, non-root + read-only hardening, health-check-gated startup, dev/prod Compose files and a CI pipeline ([`.github/workflows/ci.yml`](.github/workflows/ci.yml)).

```bash
cd sample-ai-app
cp .env.example .env        # set POSTGRES_PASSWORD
docker compose up --build   # then open http://localhost:8080/docs
```

## Quick answers

* **Which OS for production?** Linux (Ubuntu LTS/Debian/RHEL family). Windows and macOS are for development. See [Chapter 2](docs/02-installation-and-operating-systems.md) and [Chapter 11](docs/11-production-and-enterprise.md).
* **Does an Android app need Docker?** The app itself, no. The build servers, backends, and test infrastructure around it often do. See [Chapter 7](docs/07-who-needs-docker.md).
* **Docker or Docker Compose?** Docker runs one container; Compose describes and runs a whole multi-container application from one file. You almost always use both.

## Verification status

The sample application, its tests, the Compose stack (base, dev override, `watch`, production overlay validation, Ollama profile validation) were executed on Windows 11 with Docker Desktop — Docker Engine 29.9.2 and Compose 5.6.1. Deployment to a remote Linux server and GPU features were **not** executed in the author's environment; those sections are written from official documentation and marked as such.

## License / usage

Educational material. Do not commit real secrets (`.env` is git-ignored).
