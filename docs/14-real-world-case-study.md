# Chapter 14 — Real-World Case Study: Reading a Production-Style Docker Deployment

This chapter reviews an anonymised multi-service Docker deployment of an AI-tool server: an LLM-powered service that exposes a tool API plus a chat web UI, built for an internal engineering team of a few hundred users. Product, company and host names are removed. The architecture, configuration decisions and issues are real.

The purpose is to practise reading an existing Docker setup critically and to see where the concepts from earlier chapters appear in practice.

> **Note:** check your employer's policy before publishing any work code. This chapter contains no internal URLs, credentials or product names. Publish your own sample projects (such as [`sample-ai-app`](../sample-ai-app/)) instead.

---

## 1. The system in one picture

```
                  Internet / corporate network
                              │
                  :443 HTTPS  │  :80 → redirect to HTTPS
                              ▼
                    ┌───────────────────┐
                    │ nginx (1 container)│  TLS termination, load balancing,
                    └─────────┬─────────┘  health checks
          path "/mcp"         │          path "/"
   ┌───────┬───────┬───────┬──┴────┬───────┬───────┐     ┌────────────┐
   ▼       ▼       ▼       ▼       ▼       ▼       ▼     ▼            │
 api-1   api-2   api-3   api-4   api-5   api-6         web-ui (chat)  │
 (6 identical replicas of one image, ~60 users each)                   │
   └───────┴───────┴───┬───┴───────┴───────┘                           │
                       ▼                                               │
                 ┌──────────┐  coordination lock, shared session store,│
                 │  Redis   │  "ready" flags                           │
                 └──────────┘                                          │

 Monitoring: nginx-exporter ─► Prometheus ─► Grafana
```

Eleven containers: `nginx`, six `api` replicas, `web-ui`, `redis`, `nginx-exporter`, `prometheus`, `grafana`. One Compose file defines them. A second Compose file holds the production variant.

Concepts from earlier chapters present here: reverse proxy and load balancing, multi-service Compose ([Ch 6](06-docker-compose.md)), health checks, resource limits, named volumes ([Ch 5](05-storage-and-networking.md)), TLS termination, Prometheus/Grafana monitoring and a Linux production server ([Ch 11](11-production-and-enterprise.md)).

---

## 2. Reading the Dockerfile

Simplified:

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

- Slim base image, `--no-install-recommends` and removed apt lists keep the size down ([Ch 4](04-dockerfile-deep-dive.md)).
- `pip install --no-cache-dir` avoids a wasted cache layer.
- `USER appuser` runs as non-root.
- One image serves two roles (API replica or web UI). Compose overrides `command:` per service. One build, one scan, one version.
- Exec-form `CMD` gives correct signal handling.

### Things worth improving

| Observation | Why it matters | Better approach |
|---|---|---|
| `COPY src/` happens before `pip install` | Any source edit invalidates the install layer, so every build reinstalls all dependencies. | Copy dependency manifests, install, then copy source ([Ch 4](04-dockerfile-deep-dive.md)). |
| A corporate CA certificate is baked into the image | Needed behind SSL-inspecting proxies, but the image is then tied to that network and the certificate. | Mount the CA at runtime or inject it with a BuildKit secret or bind mount. |
| Single-stage build | The image keeps pip build leftovers and `curl`. | Multi-stage build ([Ch 4](04-dockerfile-deep-dive.md)). |
| Floating base tag `python:3.11-slim` | Patches arrive silently and builds are not reproducible. | Pin a patch version or digest and rebuild on a schedule. |
| `EXPOSE 8000 8001` for one image with two roles | Easy to misread. | Comment which role uses which port (the file does). |
| `Resources/` is copied into the image and also mounted as a volume | Two sources of truth; the image copy may be stale. | Bake it (immutable, versioned) or mount it (updatable), not both. |

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
`&api-base` names a block. `<<: *api-base` merges it into another service. Six replicas need six short stanzas instead of six copies of 60 lines. A single service with `deploy.replicas: 6` or `docker compose up --scale api=6` is the modern alternative, but this setup needs fixed `container_name`s because each replica must be addressable individually (see §4.1).

### 3.2 Environment variables with defaults and guards

`${OPENAI_MODEL:-gpt-4o}` sets a default. `${GRAFANA_PASSWORD}` is required in production. The `.env` file supplies values. `.env.production.example` documents the production values without secrets.

### 3.3 Health checks

```yaml
healthcheck:
  test: ["CMD", "curl", "-sf", "http://localhost:9091/metrics"]
  interval: 30s
  timeout: 10s
  start_period: 20s
  retries: 3
```

| Part | Meaning |
|---|---|
| `curl -sf` | `-s` silent, `-f` fail on HTTP errors (non-zero exit code). |
| `/metrics` endpoint | Used as a cheap "process is alive" probe because the main endpoint needs special headers. |
| `start_period: 20s` | Failures during warm-up are not counted. |

In the production file, nginx starts only after every replica is healthy:
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
A replica cannot starve the others. When it exceeds its memory limit, the kernel kills it (exit code 137) and `restart: unless-stopped` restarts it.

Capacity arithmetic from the project docs: 6 x 1.5 GB = 9 GB for replicas, plus about 1.5 GB for the other services, plus about 2 GB for the OS, gives about 13 GB. A 16 GB server leaves headroom.

### 3.5 Network exposure

- Replicas use `expose:` (internal only), never `ports:`. Only nginx publishes 80/443.
- In production, Prometheus and Grafana bind to `127.0.0.1` (`"127.0.0.1:9090:9090"`), so they are reachable only through an SSH tunnel.
- nginx also exposes an internal-only port (8080) for container-to-container traffic that does not need TLS.

### 3.6 Dev vs production Compose files

| | `docker-compose.yml` (dev) | `docker-compose.prod.yml` |
|---|---|---|
| Ports | 8080 plain + 443 self-signed | 80 + 443, configurable via env |
| Auth | shared service token allowed | per-user tokens only; no shared secret in the stack |
| Grafana | default `admin` password allowed | password required from env |
| nginx start | after containers start | after every replica is healthy |
| Monitoring ports | published | loopback only |

Compose's override-file mechanism ([Ch 6](06-docker-compose.md)) would reduce duplication: one base file plus small dev and prod deltas.

---

## 4. Design decisions worth studying

### 4.1 Six replicas of a stateful service

Each replica holds in-memory caches and long-lived streaming sessions, so it is stateful. Coordination:

- **Redis as coordination layer.** A lock lets one replica call an expensive upstream API at startup while the others wait for a "ready" flag. A shared session store keeps a user's auth valid across a replica restart.
- **Later change to stateless request mode.** Any replica can serve any request, so nginx can balance freely. Comments in the Compose file keep the history ("the old pinning workaround is no longer needed").

Statelessness makes horizontal scaling simple. Where it is not possible, shared state goes in Redis or a database.

### 4.2 Sizing ratio: 1 replica per ~60–70 concurrent users

The ratio was measured, documented and used to size the deployment. Validate such a ratio with a load test (for example k6 or Locust).

### 4.3 Security model

- Per-user bearer tokens instead of a shared service credential (a leak affects one user).
- Optional second-factor gateway key header.
- A read-only guard flag for service accounts.
- HTTPS for all user connections. Certificate options are documented: corporate CA preferred, self-signed as a fallback with trade-offs listed, Let's Encrypt only for public domains.
- `chmod 600 .env`, a dedicated unprivileged OS user, and Docker group membership treated as privileged.

### 4.4 Observability

- `nginx-exporter` converts nginx `stub_status` into Prometheus metrics.
- Each replica exposes a metrics port.
- Prometheus keeps 30 days of data.
- Grafana dashboards are provisioned from files (`./grafana/provisioning`), so they are version-controlled and reproducible.

### 4.5 Deployment on a plain Linux server

Steps in the deploy guide:

1. Choose an Ubuntu LTS size from the sizing table.
2. Install Docker with the official script.
3. Create a dedicated service user and an `/opt/<app>` directory.
4. Copy the project with `rsync` or `git clone`.
5. Create `.env` from a template with generated secrets.
6. Place TLS certificates in `/etc/ssl/<app>`.
7. Run `docker compose -f docker-compose.prod.yml up -d --build`.
8. Optionally add a systemd unit so the stack starts on boot.

This is a low-ceremony production recipe for an internal tool ([Ch 11](11-production-and-enterprise.md)).

---

## 5. A critical review — issues to flag

| # | Finding | Explanation | Recommended fix |
|---|---|---|---|
| 1 | Secrets in a local `.env` next to the code | Acceptable on a locked-down server (`chmod 600`) if `.gitignore`d and `.dockerignore`d (it was). Does not scale with team size. | Use a secrets manager (Vault, cloud secret store) or Docker/Compose secrets. Rotate any key that was in a shared folder or chat message. |
| 2 | Default credential fallback (`${GRAFANA_PASSWORD:-admin}` in the dev file) | A default password that reaches production is a breach risk. The production file correctly requires a value. | Keep dev and prod behaviour distinct. Make prod fail when unset: `${VAR:?error message}`. |
| 3 | Volume mounted under `/root/` while the container runs as non-root | A named volume at `/root/.app_memory.db` is not writable by `appuser`, so intended persistent data may live in the ephemeral container layer. | Mount at a path the non-root user owns. Verify persistence: write data, run `docker compose down && up`, check it survived (Experiment 1 in [Chapter 9](09-hands-on-ai-app-walkthrough.md)). |
| 4 | Mounted data volume plus baked-in copy | Two sources of truth (see §2). | Choose one. |
| 5 | No log rotation in either Compose file | The default `json-file` driver grows without bound. | Set `max-size` and `max-file`, or configure globally in `daemon.json` ([Ch 11](11-production-and-enterprise.md)). |
| 6 | Single nginx and single Redis | Single points of failure. Often acceptable for an internal tool. | Add redundancy or use managed services for customer-facing systems. |
| 7 | Building on the production server (`up -d --build`) | Source code lives on the server and every deploy compiles there. | CI builds and scans a versioned image, pushes to a registry, and the server pulls ([Ch 11](11-production-and-enterprise.md), [Ch 9 Part 9](09-hands-on-ai-app-walkthrough.md)). |
| 8 | Floating tags on some auxiliary images | Other images are pinned (nginx 1.27, Prometheus 2.52.0, Grafana 11.0.0). | Pin all images. |

The project follows many good practices: internal-only ports, health checks, limits, monitoring as code and documented sizing. Every real system has a backlog of improvements, and listing them is a useful review skill ([Ch 13](13-interview-questions-roadmap-glossary.md)).

---

## 6. Improvement order

1. Move secrets out of plain files (secret manager or Compose secrets).
2. Build images in CI with scanning, versioned tags and a registry.
3. Fix the persistence path and add a test that proves data survives restarts.
4. Add log rotation and centralised logs.
5. Convert to a base file plus dev/prod override files, with `service_healthy` ordering in both.
6. Restructure the Dockerfile: multi-stage, dependencies first, CA injection at runtime.
7. Move to an orchestrator (Kubernetes, ECS) only if autoscaling, self-healing across machines or zero-downtime rollouts are needed. Compose on one VM is a valid choice until then.

---

## 7. Exercise

1. Build a smaller version: nginx, 3 replicas of the [`sample-ai-app`](../sample-ai-app/) API, and Redis. nginx needs an `upstream` block listing the replicas, or Docker DNS round-robin via `--scale`.
2. Add Prometheus and Grafana with a provisioned datasource and dashboard JSON stored in git.
3. Load-test with k6 or Locust and record your own users-per-replica ratio.
4. Write a critical review of the finished stack in the format of §5.

---

## Summary

- A realistic deployment has many cooperating containers: proxy, app replicas, shared state, monitoring.
- Publish only the proxy; expose everything else internally.
- Health checks, resource limits and restart policies make containers self-managing.
- Stateful services need shared state outside the container.
- Keep monitoring configuration in version control.
- Test persistence, backups and rollbacks.
