# Chapter 14 — Real-World Case Study: Reading a Production-Style Docker Deployment

Tutorials are tidy; real projects are messy. This chapter dissects a **real multi-service Docker deployment** of an AI-tool server (an LLM-powered service exposing a tool API plus a chat web UI) that was built for an internal engineering team of a few hundred users. It has been **anonymized** — product, company and host names are removed — but the architecture, configuration decisions and mistakes are real.

The goal: teach you to **read someone else's Docker setup critically**, which is exactly what you'll do in a new job, and to see which of the earlier chapters' ideas appear in practice.

> **Portfolio note:** if you build something similar at work, check your employer's policy before publishing any of it. This chapter deliberately contains no internal URLs, credentials or product names. Publish your *own* sample projects (like [`sample-ai-app`](../sample-ai-app/)) rather than company code.

---

## 1. The system in one picture

```
                        Internet / corporate network
                                   │
                         :443 HTTPS │  :80 → redirect to HTTPS
                                   ▼
                         ┌───────────────────┐
                         │   nginx           │  TLS termination, load balancing,
                         │   (1 container)   │  health checks
                         └───────┬───────────┘
              path "/mcp"        │            path "/"
        ┌────────┬────────┬──────┴─┬────────┬────────┐       ┌──────────────┐
        ▼        ▼        ▼        ▼        ▼        ▼       ▼              │
     api-1    api-2    api-3    api-4    api-5    api-6     web-ui  (chat UI)│
      (6 identical replicas of the same image, ~60 users each)              │
        └────────┴────────┴────┬───┴────────┴────────┘                      │
                               ▼                                           │
                         ┌──────────┐   coordination lock + shared         │
                         │  Redis   │◄─ session store + "ready" flags      │
                         └──────────┘                                      │
                                                                           │
   Monitoring (separate containers):  nginx-exporter ─► Prometheus ─► Grafana
```

Containers (11 in total): `nginx`, six `api` replicas, one `web-ui`, `redis`, `nginx-exporter`, `prometheus`, `grafana`. All defined in one Compose file; a second Compose file holds the production variant.

**Concepts from earlier chapters visible here:** reverse proxy and load balancing, Compose multi-service stacks ([Ch 6](06-docker-compose.md)), health checks, resource limits, named volumes ([Ch 5](05-storage-and-networking.md)), TLS termination, Prometheus/Grafana monitoring and a Linux production server ([Ch 11](11-production-and-enterprise.md)).

---

## 2. Reading the Dockerfile

The Dockerfile (simplified):

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*
COPY pyproject.toml README.md ./
COPY src/ src/
COPY certs/corp-ca.pem /tmp/corp-ca.pem
RUN pip install --no-cache-dir '.[prod]' && \
    pip install --no-cache-dir 'uvicorn[standard]' && \
    cat /tmp/corp-ca.pem >> $(python -c "import certifi; print(certifi.where())") && \
    rm /tmp/corp-ca.pem
COPY Resources/ Resources/
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser
EXPOSE 8000 8001
CMD ["python", "-m", "app_server", "--host", "0.0.0.0", "--port", "8000"]
```

### What it does well
* **Slim base image** and `--no-install-recommends` + removing apt lists — keeps size down ([Ch 4](04-dockerfile-deep-dive.md)).
* **`pip install --no-cache-dir`** — no wasted cache layer.
* **Non-root user** (`USER appuser`) — good security practice.
* **One image, two roles.** The same image runs as an API replica *or* the web UI; the Compose file overrides `command:` per service. This is a very common, efficient pattern (one build, one scan, one version).
* **Exec-form `CMD`** — correct signal handling.

### Things worth improving (and why they matter)
| Observation | Why it matters | Better approach |
|---|---|---|
| `COPY src/` happens **before** `pip install` | Any source edit invalidates the install layer → every build reinstalls all dependencies (slow) | Copy dependency manifests first, install, *then* copy source ([Ch 4](04-dockerfile-deep-dive.md) layer caching) |
| A **corporate CA certificate is baked into the image** at build time | Required behind SSL-inspecting proxies, but now the image only works/should only be shared inside that network, and the cert is tied to the build | Mount the CA at runtime, or inject it with a BuildKit secret/bind mount; keep CA handling out of the app image |
| Single-stage build | The image contains pip's build leftovers and `curl` | Multi-stage build ([Ch 4](04-dockerfile-deep-dive.md)) |
| Base tag `python:3.11-slim` (floating) | Security patches arrive silently, builds aren't reproducible | Pin a patch version or digest, rebuild on a schedule |
| `EXPOSE 8000 8001` for one image with two roles | Fine as documentation, but easy to misread | Comment which role uses which port (the file does) |
| Optional data (`Resources/`) copied into the image **and** mounted as a volume at runtime | Two sources of truth; image carries data that may be stale | Choose one: bake it (immutable, versioned) *or* mount it (updatable) |

---

## 3. Reading the Compose file

### 3.1 YAML anchors to avoid copy-paste

```yaml
api-1: &api-base
  build: .
  environment: &api-env
    - SETTING_A=${SETTING_A}
    ...
api-2:
  <<: *api-base
  container_name: api-2
```
`&api-base` *names* a block; `<<: *api-base` *merges* it into another service. Six replicas cost six short stanzas instead of six copies of 60 lines. (Modern alternative: a single service with `deploy.replicas: 6` or `docker compose up --scale api=6` — but fixed `container_name`s are needed here because each replica must be addressable individually, see §4.)

### 3.2 Environment variables with defaults and guards

`${OPENAI_MODEL:-gpt-4o}` (default), `${GRAFANA_PASSWORD}` required in prod. The `.env` file supplies them; a separate `.env.production.example` documents the production values without secrets.

### 3.3 Health checks everywhere

```yaml
healthcheck:
  test: ["CMD", "curl", "-sf", "http://localhost:9091/metrics"]
  interval: 30s
  timeout: 10s
  start_period: 20s
  retries: 3
```
`curl -sf` = silent + fail on HTTP errors. The metrics endpoint is used as a cheap "is the process alive" probe — a pragmatic trick when the main endpoint needs special headers. `start_period` avoids false alarms while the app warms up.

In the production file nginx starts only after every replica is healthy:
```yaml
depends_on:
  api-1: { condition: service_healthy }
  ...
```

### 3.4 Resource limits

```yaml
deploy:
  resources:
    limits:       { cpus: '2.0', memory: 1500M }
    reservations: { cpus: '0.25', memory: 512M }
```
A runaway replica can't starve the others; if it exceeds its memory limit the kernel kills it (exit code 137) and `restart: unless-stopped` brings it back. **Capacity math** from the project's own docs: 6 × 1.5 GB = 9 GB for replicas + ~1.5 GB for the rest + ~2 GB OS ⇒ ~13 GB, so a 16 GB server leaves headroom. *This sizing arithmetic is exactly what you do before choosing a VM.*

### 3.5 Network exposure discipline (a highlight)

* Replicas use **`expose:`** (internal only), never `ports:`; only nginx publishes 80/443.
* In production, Prometheus and Grafana bind to **`127.0.0.1`** (`"127.0.0.1:9090:9090"`) so they're reachable only via an SSH tunnel — not from the network.
* nginx also exposes an **internal-only** port (8080) for container-to-container traffic that doesn't need TLS overhead.

### 3.6 Dev vs production Compose files

| | `docker-compose.yml` (dev) | `docker-compose.prod.yml` |
|---|---|---|
| Ports | 8080 plain + 443 self-signed | 80 + 443, configurable via env |
| Auth | shared service token OK | **per-user tokens only** — no shared secret exists in the stack |
| Grafana | default `admin` password allowed | password **required** from env |
| nginx start | after containers *start* | after every replica is **healthy** |
| Monitoring ports | published | loopback only |

Two files for two environments is the right instinct. (Compose's *override file* mechanism — [Ch 6](06-docker-compose.md) — would reduce the duplication: a base file plus small prod/dev deltas.)

---

## 4. Design decisions worth studying

### 4.1 Why six replicas of a *stateful* service, and how it's coordinated
Each replica holds in-memory caches and long-lived streaming sessions, so it's **stateful** — unlike the "stateless web app" ideal. The team handled this with:
* **Redis as a coordination layer** — a lock so only *one* replica calls an expensive upstream API at startup while the others wait for a "ready" flag; a shared session store so a user's auth survives a replica restart.
* **Later**, they moved to a **stateless request mode**, so *any* replica can serve *any* request and nginx can balance freely. The comments in the Compose file preserve the history ("the old pinning workaround is no longer needed").

**Lesson:** statelessness is what makes horizontal scaling easy. When you can't have it, push shared state into Redis/a database.

### 4.2 Rule of thumb from the docs: "1 replica per ~60–70 concurrent users"
This kind of empirically derived ratio — measured, written down, and used to size the cluster — is what separates a guess from an engineered deployment. You'd validate it with a load test (e.g., k6, Locust) and adjust.

### 4.3 Security model
* Per-user bearer tokens instead of a shared service credential (blast radius of a leak = one user).
* Optional second-factor gateway key header.
* A read-only guard flag for service accounts.
* HTTPS everywhere users connect; TLS certificate options documented (corporate CA preferred, self-signed as a fallback with the trade-offs spelled out, Let's Encrypt only for public domains).
* `chmod 600 .env`; dedicated unprivileged OS user; Docker group membership understood as privileged.

### 4.4 Observability
`nginx-exporter` converts nginx's `stub_status` into Prometheus metrics; each replica exposes a metrics port; Prometheus keeps 30 days of data; Grafana dashboards are provisioned from files (`./grafana/provisioning`) so they're **version-controlled** and reproducible — a mark of mature practice.

### 4.5 Deployment on a plain Linux server
The deploy guide walks through: Ubuntu LTS sizing table → install Docker via the official script → dedicated service user and `/opt/<app>` directory → `rsync`/`git clone` the project → `.env` from a template with generated secrets → TLS certs in `/etc/ssl/<app>` → `docker compose -f docker-compose.prod.yml up -d --build` → optionally a **systemd unit** so the stack starts on boot. This is a realistic, low-ceremony production recipe for an internal tool and a good template for your own small deployments ([Ch 11](11-production-and-enterprise.md)).

---

## 5. A critical review — issues a careful reader would flag

Reviewing real configurations is a skill. These are the kinds of findings an experienced engineer would raise (descriptions are generic so you can apply them anywhere):

1. **Secrets in a local `.env` next to the code.** Acceptable on a locked-down server (`chmod 600`), but it must be `.gitignore`d and `.dockerignore`d (it was) and ideally replaced by a secrets manager (Vault, cloud secret store, Docker/Compose secrets) as the team grows. Rotate any key that has ever sat in a shared folder or chat message.
2. **Default credentials as a fallback** (`${GRAFANA_PASSWORD:-admin}` in the dev file). Convenient, but a default password that reaches production is a classic breach. The production file correctly *requires* a value — keep dev and prod behaviours distinct and make the prod file fail loudly if unset (`${VAR:?error message}`).
3. **A volume mounted at a path under `/root/`** while the container runs as a non-root user. A named volume mounted at `/root/.app_memory.db` would not be writable (or even used) by `appuser`, so the data the author intended to persist may silently live in the ephemeral container layer. **Always verify persistence** by writing data, running `docker compose down && up`, and checking it survived — exactly Experiment 1 in [Chapter 9](09-hands-on-ai-app-walkthrough.md).
4. **A "mounted data" volume plus baked-in copy** (see §2) — two sources of truth.
5. **No log rotation configured** in either Compose file for the default `json-file` driver. Logs grow without bound; add `max-size`/`max-file` or set it globally in `daemon.json` ([Ch 11](11-production-and-enterprise.md)).
6. **Single nginx and single Redis = single points of failure.** For an internal tool that's often an acceptable trade-off; for a customer-facing system you'd add redundancy or use managed services.
7. **Building on the production server (`up -d --build`).** It works, but it means source code lives on the server and every deployment compiles there. The mature pattern is *CI builds and scans a versioned image → registry → server pulls* ([Ch 11](11-production-and-enterprise.md), [Ch 9 Part 9](09-hands-on-ai-app-walkthrough.md)).
8. **Floating image tags** for some auxiliary images vs pinned versions for others (nginx 1.27, Prometheus 2.52.0, Grafana 11.0.0 pinned — good). Be consistent: pin everything.

None of these make the project "bad" — it embodies many best practices (internal-only ports, health checks, limits, monitoring-as-code, documented sizing). The point is that **every real system has a backlog of improvements**, and being able to articulate them is a senior-level skill — it's also great interview material ([Ch 13](13-interview-questions-roadmap-glossary.md)).

---

## 6. If you had to improve it, in priority order

1. Get secrets out of plain files (secret manager or Compose secrets).
2. Move image building into CI with scanning + versioned tags + a registry.
3. Fix the persistence path and add a test that proves data survives restarts.
4. Add log rotation and centralised logs.
5. Convert to a base file + dev/prod override files; use `service_healthy` ordering in both.
6. Restructure the Dockerfile: multi-stage, dependencies-first ordering, CA injection at runtime.
7. Consider moving to an orchestrator (Kubernetes / ECS) *only if* you need autoscaling, self-healing across machines or zero-downtime rollouts — Compose on one VM is a legitimate choice until then.

---

## 7. How to use this as your own learning exercise

1. Re-create a smaller version: nginx + 3 replicas of [`sample-ai-app`](../sample-ai-app/)'s API + Redis. (Hint: nginx needs an `upstream` block listing the replicas, or Docker's DNS round-robin via `--scale`.)
2. Add Prometheus + Grafana with a provisioned datasource and dashboard JSON stored in git.
3. Load-test with k6 or Locust and write down *your own* "users per replica" ratio.
4. Write your own critical review of your finished stack, in the style of §5.

---

### Key takeaways
* A realistic deployment is *many cooperating containers*: proxy, app replicas, shared state, monitoring.
* Expose the minimum to the network; publish only the proxy.
* Health checks + resource limits + restart policies make containers self-managing.
* Stateful services need shared state outside the container; stateless ones scale trivially.
* Monitoring configuration belongs in version control.
* Always **test** persistence, backups and rollbacks — don't assume.
* Being able to critique a setup (§5) is as valuable as being able to create one.
