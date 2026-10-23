# Chapter 13: Interview Questions, Learning Roadmap, Portfolio Guide and Glossary

This chapter contains 88 interview questions with model answers (beginner, intermediate, advanced, scenario-based, AI-specific), an eight-week learning plan, a list of topics to learn after Docker, a portfolio guide, further resources and an A-Z glossary.

How to use the questions: answer each one aloud in your own words, then compare with the model answer. Reread the matching chapter for any missed point. Run command-related answers once on your own machine.

## Table of contents

1. [Interview questions and model answers](#1-interview-questions-and-model-answers)
   - [Beginner (Q1-Q22)](#beginner-questions)
   - [Intermediate (Q23-Q44)](#intermediate-questions)
   - [Advanced (Q45-Q60)](#advanced-questions)
   - [Scenario-based (Q61-Q73)](#scenario-based-questions)
   - [AI-specific (Q74-Q88)](#ai-specific-questions)
2. [How to answer well in interviews](#2-how-to-answer-well-in-interviews)
3. [Staged learning roadmap (weeks 1-8 and beyond)](#3-staged-learning-roadmap)
4. [What to learn after Docker](#4-what-to-learn-after-docker)
5. [Portfolio guide](#5-portfolio-guide)
6. [Recommended further resources](#6-recommended-further-resources)
7. [Glossary (A-Z)](#7-glossary-a-z)

---

## 1. Interview questions and model answers

### Beginner questions

**Q1. What is Docker, and what problem does it solve?**

Docker packages an application with everything it needs (code, runtime, libraries, system tools, configuration) into an **image** and runs it as an isolated process called a **container**. The container carries its own environment, so it behaves the same on a laptop, a CI server and production. It makes deployments repeatable and lets apps with conflicting dependencies share one machine.

**Q2. What is the difference between a container and a virtual machine?**

A virtual machine (VM) emulates a whole computer, with virtual hardware and a guest operating system with its own kernel, on a **hypervisor**. A container is an isolated **process** that **shares the host's kernel**. Isolation comes from Linux **namespaces** (separate views of processes, network, filesystem) and **cgroups** (resource limits). Containers are lighter (megabytes), start in about a second and run in larger numbers per machine. VMs isolate more strongly and can run different operating systems. A Linux container needs a Linux kernel.

```
 VM:                                  Container:
 [App][App]                           [App][App]
 [Guest OS][Guest OS]                 [libs ][libs ]
 [   Hypervisor    ]                  [ Container engine ]
 [   Host OS/HW    ]                  [   Host OS (shared kernel) ]
```

**Q3. What is the difference between an image and a container?**

An image is a read-only template: filesystem layers plus metadata (default command, environment, exposed ports). A container is a running or stopped instance of an image with a thin writable layer. Analogy: an image is a class, a container is an object. Many containers can start from one image.

**Q4. What is a Dockerfile?**

A text file of instructions (`FROM`, `COPY`, `RUN`, `CMD` and others) that tells Docker how to build an image. Each instruction that changes the filesystem creates a **layer**. `docker build` reads it and produces the image.

**Q5. What is a registry, and what is Docker Hub?**

A registry is a server that stores and distributes images (`push` uploads, `pull` downloads). Docker Hub is the default public registry (`docker.io`) and hosts official images such as `nginx`, `postgres` and `python`. Others: GitHub Container Registry (`ghcr.io`), Amazon ECR, Azure Container Registry, Google Artifact Registry, and self-hosted Harbor.

**Q6. What does `docker run -d -p 8080:80 --name web nginx` do?**

It creates and starts a container from the `nginx` image, pulling it first if it is not local.

| Part | Meaning |
|---|---|
| `-d` | Run detached (in the background) |
| `-p 8080:80` | Publish container port 80 on host port 8080 (host left, container right); `http://localhost:8080` reaches nginx |
| `--name web` | Name the container so `docker logs web` works |

**Q7. What is the difference between `CMD` and `ENTRYPOINT`?**

Both define what runs at container start. `ENTRYPOINT` is the fixed executable. `CMD` provides default arguments (or the default command when there is no entrypoint), and anything typed after the image name in `docker run` replaces it. Example: `ENTRYPOINT ["python", "app.py"]` plus `CMD ["--port", "8000"]` lets users override only the arguments. Replacing the entrypoint needs `--entrypoint`. Prefer the **exec form** (JSON array) so the process receives signals directly.

**Q8. What is the difference between `COPY` and `ADD`?**

`COPY` copies files from the build context into the image. `ADD` also extracts local tar archives and fetches URLs. Use `COPY` unless tar auto-extraction is needed.

**Q9. What is the difference between `EXPOSE` and `-p`?**

`EXPOSE 8000` is documentation of the port the app listens on and opens nothing. `-p HOST:CONTAINER` (or `ports:` in Compose) publishes the port by mapping a host port to the container. `-P` publishes all exposed ports to random host ports.

**Q10. What are the ways to persist data?**

| Type | Description | Use |
|---|---|---|
| Named volume | Storage managed by Docker (under `/var/lib/docker/volumes` on Linux), independent of any container | Databases, application data |
| Bind mount | Maps a host path into the container | Development (live code); ties the container to the host's layout and permissions |
| tmpfs mount | In-memory storage, lost when the container stops | Sensitive or scratch data |

Without any of these, data lives in the writable layer and is lost when the container is removed.

**Q11. What happens when you run `docker run hello-world`?**

The CLI sends the request to the daemon. The daemon checks for the image locally and pulls it from the registry if absent. It creates the container (writable layer, network interface), asks the runtime (containerd, which uses runc) to start the process in new namespaces and cgroups, and connects the output to the terminal. The container exits when the program finishes.

**Q12. What is the difference between `docker stop` and `docker kill`?**

`docker stop` sends **SIGTERM**, waits (10 seconds by default), then sends **SIGKILL** if the process is still running. `docker kill` sends SIGKILL (or a chosen signal) immediately, with no chance for cleanup.

**Q13. Why do image layers matter?**

Each filesystem-changing instruction creates a read-only layer, stored by content hash. Layers are **shared** between images and **cached** during builds. If a layer changes, all later layers rebuild. Order instructions from least to most frequently changing (dependencies before source code).

**Q14. What is `.dockerignore` and why use it?**

A file listing paths excluded from the **build context** (the files sent to the builder). It keeps images small, speeds builds, avoids needless cache invalidation and keeps secrets (`.env`, `.git`, private keys) out of images. Typical entries: `.git`, `node_modules`, `__pycache__`, `*.log`, `.env`.

**Q15. What is the difference between `ENV` and `ARG`?**

`ARG` is a build-time variable (set with `--build-arg`). `ENV` exists at build time and at runtime. Neither is safe for secrets: both are visible in image metadata or history (`docker history`, `docker inspect`). Use BuildKit secret mounts at build time and a secrets manager or mounted files at runtime.

**Q16. What is Docker Compose?**

A tool for defining and running **multi-container** applications in one YAML file (`compose.yaml`): services, networks, volumes, environment, ports and dependencies. `docker compose up -d` starts the stack; `docker compose down` removes it. It suits local development, testing and small deployments.

**Q17. How do containers talk to each other?**

On a **user-defined bridge network** (Compose creates one automatically), containers reach each other by **service name** through Docker's embedded DNS (`http://db:5432`), using the container's own port. Publishing a port is needed only for access from outside. The default bridge network has no name-based DNS.

**Q18. What is the difference between `docker ps` and `docker ps -a`?**

`docker ps` lists running containers. `docker ps -a` also lists stopped ones, which is needed to find a crashed container and read its logs.

**Q19. Why do containers start faster than VMs?**

No operating system boots. A container is a new process in isolated namespaces with cgroup limits on a running kernel, using an image filesystem already on disk.

**Q20. What does the tag `latest` mean?**

Nothing special. It is the default tag used when none is given, not the "newest" image. It is mutable, so using it in production gives unreproducible deployments and prevents reliable rollback. Pin explicit versions (`postgres:17.2`) or digests.

**Q21. How do you see what is going on inside a container?**

`docker logs` (stdout/stderr), `docker exec -it NAME sh` (shell), `docker inspect` (configuration and state), `docker stats` (resource use), `docker top` (processes), `docker events` (lifecycle events). See Chapter 12.

**Q22. Why is Docker widely used by developers?**

Reproducible environments, fast onboarding (clone, `docker compose up`), dependency isolation, consistent CI/CD, a large catalogue of ready-made images, easy local replication of production-like stacks, and portability across laptops, servers and clouds.

---

### Intermediate questions

**Q23. What is a multi-stage build and why use it?**

A Dockerfile with several `FROM` stages. Early stages hold compilers and build tools and produce artifacts. The final stage copies only the artifacts (`COPY --from=builder /app/dist /app`) onto a small runtime base. Benefits: smaller images, smaller attack surface, no build tools or source in production, and build-time secrets stay out of the final image. A single stage can be built for debugging with `--target`.

```dockerfile
FROM node:22 AS build
WORKDIR /src
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:1.27-alpine
COPY --from=build /src/dist /usr/share/nginx/html
```

**Q24. How do you reduce an image's size?**

Use a smaller base (`-slim`, Alpine where compatible, distroless); multi-stage builds; install only what is needed (`--no-install-recommends`, `--no-cache-dir`); clean package caches in the same `RUN` layer; use `.dockerignore`; keep datasets and model weights out; inspect with `docker history` and `dive`.

**Q25. Explain Docker's network drivers.**

| Driver | Description |
|---|---|
| bridge (default) | Private virtual network on the host. Containers on the same user-defined bridge reach each other by name; outbound traffic is NATed; inbound needs published ports |
| host | Container shares the host's network stack: no isolation, no port mapping, best raw performance, Linux only |
| none | No networking except loopback |
| overlay | Multi-host networking for Swarm |
| macvlan | Container gets its own MAC/IP on the physical network and appears as a separate machine on the LAN |

**Q26. Why use a user-defined bridge instead of the default bridge?**

It provides automatic DNS by container name, better isolation (only attached containers can talk) and live connect/disconnect. The default bridge uses IPs only (or legacy `--link`) and places all unrelated containers on one network.

**Q27. How does `depends_on` work, and what is its limitation?**

It controls start **order**. By default Compose waits only until the dependency container has started, not until the service is ready. For readiness, define a `healthcheck` on the dependency and use `depends_on: { db: { condition: service_healthy } }`. The app should also retry connections, because dependencies can restart in production.

**Q28. What is a HEALTHCHECK and what does Docker do with it?**

A command Docker runs periodically inside the container; exit code 0 is healthy, 1 is unhealthy. The status appears in `docker ps` and can gate `depends_on` and `docker compose up --wait`. Plain Docker does not restart unhealthy containers; Swarm, Kubernetes (probes) and ECS replace them.

**Q29. How do you handle secrets with Docker?**

Never in the image (`ENV`, `ARG`, copied files) or in git. At build time use BuildKit secret mounts (`RUN --mount=type=secret`). At runtime use Docker/Compose secrets (files at `/run/secrets/...`), orchestrator secrets (Kubernetes Secrets with encryption at rest, or External Secrets), or a secrets manager (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager). Prefer short-lived credentials, rotate them, and scan history for leaks.

**Q30. How does Docker isolate containers from each other and from the host?**

| Mechanism | Role |
|---|---|
| Namespaces (PID, network, mount, UTS, IPC, user, cgroup) | Each container gets its own view of the system |
| cgroups | Limit resources |
| Capabilities | Split root's powers; Docker drops many by default |
| seccomp | Filters dangerous system calls |
| AppArmor / SELinux | Mandatory access control |

Containers are less isolated than VMs because they share the kernel, and a kernel exploit can escape. Also run as non-root, drop capabilities, use a read-only filesystem and patch.

**Q31. What is the PID 1 problem?**

The first process in a container (PID 1) must reap orphaned **zombie** children and has no default signal handlers (SIGTERM is ignored unless the program installs a handler). An app that ignores SIGTERM makes `docker stop` wait 10 seconds before killing it. Solutions: exec-form `CMD`/`ENTRYPOINT` so the app is PID 1, `exec` the app at the end of shell entrypoint scripts, and use a small init (`docker run --init` / Compose `init: true`, which uses tini).

**Q32. Shell form vs exec form for CMD/ENTRYPOINT?**

Shell form (`CMD python app.py`) runs through `/bin/sh -c`; the shell is PID 1 and may not forward signals, but variable expansion works. Exec form (`CMD ["python","app.py"]`) runs the program directly: correct signal handling, no shell features unless `sh -c` is invoked. Use exec form for long-running services.

**Q33. What restart policies exist?**

`no` (default), `on-failure[:max]`, `always`, `unless-stopped`. `always` restarts a manually stopped container when the daemon restarts; `unless-stopped` honours a manual stop. In production use `unless-stopped` (single host) or let the orchestrator replace containers.

**Q34. How do you limit a container's CPU and memory?**

`--memory`/`--memory-swap`, `--cpus`, `--cpu-shares`, `--pids-limit` in `docker run`; `deploy.resources.limits` (or `mem_limit`, `cpus`) in Compose; `resources.requests/limits` in Kubernetes. Exceeding a memory limit triggers an OOM kill (exit 137); CPU limits throttle.

**Q35. What are logging drivers? What is the default, and what is the risk?**

A logging driver sets where stdout/stderr go: `json-file` (default), `local`, `journald`, `syslog`, `fluentd`, `gelf`, `awslogs`, `gcplogs`, `splunk`, `none`. The default `json-file` does not rotate, so a chatty container can fill the disk. Configure `max-size` and `max-file`, or use `local`, which rotates by default.

**Q36. Tag vs digest?**

A tag is a mutable label. A digest (`sha256:...`) is the immutable content hash of the image manifest. Deploying by digest guarantees the tested image runs. Typical practice: tag with version and git SHA for humans, deploy by digest or immutable tag.

**Q37. What is BuildKit?**

Docker's modern build engine and default builder. It provides parallel execution of independent stages, better caching (cache mounts, cache export to registries), secret and SSH mounts, multi-platform builds, attestations (SBOM, provenance) and build checks. `docker buildx` is its CLI front end.

**Q38. How do you build multi-architecture images?**

`docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.0 --push .` produces a manifest list (OCI image index) with one image per architecture under one tag, and clients pull the matching one. Non-native architectures build through QEMU emulation (slow) or native builders/cross-compilation. Needed for Apple Silicon development with amd64 servers, and for AWS Graviton.

**Q39. Docker vs containerd vs runc vs Podman vs OCI?**

| Term | Role |
|---|---|
| OCI (Open Container Initiative) | Standards for image format and runtime |
| runc | Low-level runtime that creates the container (namespaces, cgroups) from an OCI bundle |
| containerd | Higher-level runtime daemon managing image pulls, storage and lifecycle, and calling runc. Used by Docker Engine and by Kubernetes |
| Docker Engine | Adds CLI, API, build, networking and volumes on top of containerd |
| Podman | Daemonless, rootless-first engine with a Docker-compatible CLI; common on RHEL |

**Q40. What is rootless Docker, and why does it matter?**

The daemon and containers run as an unprivileged user (using user namespaces), so a container escape yields only that user's privileges, not host root. Trade-off: some networking modes, low ports and some storage drivers are limited.

**Q41. Why is mounting `/var/run/docker.sock` into a container dangerous?**

The socket is the Docker API. Anything that can use it can start a privileged container that mounts the host's root filesystem, which is full host control. Avoid it. If a tool needs it (monitoring, some CI), restrict it with a socket proxy exposing a read-only API subset, or use rootless/daemonless builders (rootless BuildKit, Kaniko, Buildah).

**Q42. What are Compose profiles and override files?**

Profiles (`profiles: ["ollama"]`) mark services that start only on request (`docker compose --profile ollama up`), making optional components opt-in. Override files (`compose.override.yaml`, merged automatically, or extra `-f` files) layer environment-specific changes on a base file, for example `compose.yaml` plus `compose.prod.yaml`.

**Q43. How do build-time and run-time configuration differ (twelve-factor)?**

Build-time values are baked into the image (dependencies, compiled assets). Run-time configuration (database URL, feature flags, secrets) comes from the environment so one image runs in every environment. Twelve-factor: store config in the environment, treat backing services as attached resources via URLs, keep processes stateless.

**Q44. What is Docker Swarm, and is it still relevant?**

Swarm mode is Docker's built-in clustering: managers/workers, replicated services, rolling updates, overlay networks and secrets, using a Compose-like file via `docker stack deploy`. It is simpler than Kubernetes. As of 2026 it is not formally deprecated and still gets fixes, but is generally considered to be in maintenance mode. Suitable for small existing setups; usually not chosen for new large projects.

---

### Advanced questions

**Q45. How does the overlay2 storage driver work?**

An image is a stack of read-only layers. **OverlayFS** (a kernel union filesystem) merges `lowerdir` (read-only image layers) and `upperdir` (the container's writable layer), with a `workdir`, into `merged`. Reads fall through to lower layers. Writes use **copy-on-write**: the first modification of a lower-layer file copies it to the upper layer. Deletions are recorded as "whiteout" files. Write-heavy workloads therefore belong in volumes, and deleting a file in a later layer does not shrink the image.

**Q46. What is inside an image? Explain manifest, config and layers.**

- **Manifest:** JSON listing the config blob and the ordered layer blobs, each identified by digest.
- **Config:** JSON with environment, entrypoint, exposed ports, architecture, history and the ordered layer diff IDs.
- **Layers:** compressed tar archives of filesystem changes.

Everything is **content-addressable** (identified by SHA-256), so identical layers are stored once and integrity is verifiable. A multi-arch tag points to an **image index** (manifest list) that references one manifest per platform.

**Q47. How do cgroups v2 enforce limits, and what happens on memory exhaustion?**

Cgroups form a hierarchy of process groups with controllers for CPU, memory, I/O and PIDs. A container's cgroup has `memory.max` (hard limit) and `memory.high` (throttle threshold). When the cgroup exceeds the limit and the kernel cannot reclaim memory, the OOM killer kills a process in it, typically the main process, giving exit 137 and `OOMKilled=true`. CPU limits use bandwidth control (`cpu.max`, a quota per period) and throttle instead of killing. Docker Engine 29 deprecated cgroup v1.

**Q48. Describe the path of a packet to a published port.**

With `-p 8080:80`, Docker adds **iptables/nftables DNAT rules** (the `nat` table's DOCKER chain) that rewrite traffic arriving on host port 8080 to the container's IP on port 80, plus FORWARD chain rules permitting it. Outbound container traffic is **masqueraded** (source NAT) to the host's address. For connections from the host itself, Docker may use a userspace `docker-proxy`. Docker inserts its rules ahead of typical `ufw` rules, so published ports can bypass the host firewall. Bind to `127.0.0.1`, avoid publishing, or use the `DOCKER-USER` chain or an external firewall.

**Q49. How would you secure the container supply chain end to end?**

- Use minimal, pinned, trusted base images (ideally an internal golden image).
- Lint Dockerfiles (hadolint) and enforce policy (OPA/Conftest).
- Scan images and dependencies in CI and continuously in the registry.
- Generate an **SBOM** and **provenance** attestations.
- **Sign** images (Cosign/Sigstore keyless with OIDC) and verify at deploy time (admission control with Kyverno or Gatekeeper).
- Use immutable tags and digests.
- Protect CI: least-privilege tokens, pinned third-party actions, ephemeral runners, no long-lived cloud keys.
- Restrict who can push; scan for leaked secrets; patch and rebuild regularly.

Reported incident: in 2026 a widely used scanner's GitHub Action was compromised through mutable version tags. Pin actions to verified commit SHAs.

**Q50. How do you achieve zero-downtime deployments with containers?**

Run at least two replicas behind a load balancer; use **readiness probes** so traffic reaches only ready instances; use **rolling updates** with surge and no unavailable instances (Kubernetes `maxUnavailable: 0`), **blue/green** or **canary**; handle SIGTERM for **graceful shutdown**; keep database migrations **backward compatible** (expand/contract); roll back automatically on failed health checks or SLO breaches.

**Q51. Compare Docker Compose, Docker Swarm and Kubernetes.**

| Tool | Scope | Notes |
|---|---|---|
| Compose | Single host | Simple YAML; dev/test and small deployments; no self-healing across machines |
| Swarm | Multi-host | Compose-like model, rolling updates, secrets; little new development |
| Kubernetes | Multi-host | Industry-standard orchestrator with a large ecosystem (autoscaling, operators, RBAC, network policies, service meshes, GitOps); steep learning curve and operational cost; managed offerings (EKS/AKS/GKE) |

Choose by team size and needs.

**Q52. Why did Kubernetes remove dockershim, and did that break Docker images?**

Kubernetes talked to Docker through a built-in adapter ("dockershim"). Docker itself wraps containerd and Kubernetes standardised on the **CRI** (Container Runtime Interface), so the shim was extra burden and was removed in v1.24. Clusters use containerd or CRI-O directly. Images did not break because they follow the OCI standard. On a node, `docker` commands no longer show Kubernetes containers; use `crictl`.

**Q53. What capabilities, seccomp and read-only filesystem settings would you apply to harden a container?**

`--cap-drop ALL`, adding back only what is needed (for example `NET_BIND_SERVICE`); `--security-opt no-new-privileges`; Docker's default seccomp profile (or a tighter one); `--read-only` with `tmpfs` for `/tmp`; a non-root user (`USER 10001`); no `--privileged`; no Docker socket or host path mounts; resource limits; user namespaces or rootless mode; a minimal image (distroless) with no shell.

**Q54. What is distroless, and when do you use `scratch`?**

Distroless images contain only the application runtime and dependencies, with no package manager or shell, which reduces attack surface and size. `scratch` is the empty base image, used for statically linked binaries (Go, Rust). The trade-off is harder debugging: use `docker debug`, ephemeral debug containers or debug image variants.

**Q55. How do you make Docker builds fast in CI?**

Order the Dockerfile for caching; use `.dockerignore`; BuildKit cache mounts; multi-stage builds with parallel stages; a **remote cache** (`--cache-to/--cache-from` with `type=gha` or `type=registry`) because runners are ephemeral; a pull-through cache or mirrored base images to avoid Docker Hub rate limits; build only changed services in a monorepo; do amd64 on pull requests and multi-arch on release.

**Q56. How do you build images in CI without exposing the Docker socket?**

Use BuildKit directly (`docker buildx` with the docker-container driver), **rootless BuildKit**, **Kaniko** (userspace builds in a container; check its maintenance status before choosing it for new projects), **Buildah**, or cloud build services. Docker-in-Docker (a privileged `docker:dind` sidecar) works but needs privileged mode, a risk on shared runners.

**Q57. How does Docker handle DNS inside containers?**

Containers on user-defined networks use Docker's **embedded DNS server** at `127.0.0.11`, which resolves container and service names and aliases on that network and forwards other queries to the host's DNS servers. Containers on the default bridge copy the host's `/etc/resolv.conf`. Problems come from host stub resolvers like `127.0.0.53`, VPNs and corporate DNS; fix with `dns` settings in `daemon.json` or Compose.

**Q58. What is `live-restore` and what are its limits?**

A daemon setting that keeps containers running while the daemon is stopped or restarted. Limits: not for major version upgrades, Swarm mode or Windows containers; container logs can block if the daemon is down long enough to fill the log pipe buffer.

**Q59. How would you debug a container that has no shell?**

Use `docker debug` (a Docker Desktop feature that adds a toolbox without modifying the image); attach a sidecar such as `nicolaka/netshoot` sharing the target's network and PID namespaces (`--network container:NAME --pid container:NAME`); copy files out with `docker cp`; inspect from the host via `/proc/<pid>`; read logs and `inspect` data; on Kubernetes use `kubectl debug` with an ephemeral container.

**Q60. What makes a build reproducible, and why does it matter?**

Pin everything that can change: base image by digest, OS packages and language dependencies by exact version and lock file, build tool versions. Avoid `latest`, unpinned `apt-get install` and downloads of moving targets. It matters for security (known contents, rebuildable), debugging and compliance. Bit-for-bit reproducibility also needs normalised timestamps and ordering (BuildKit supports `SOURCE_DATE_EPOCH`).

---

### Scenario-based questions

**Q61. A container exits immediately after `docker run -d`. How do you diagnose it?**

1. `docker ps -a` for the exit code.
2. `docker logs NAME` for the error.
3. Interpret the code: 0 = process finished (one-shot command, or the app backgrounded itself); 1 = application error; 126/127 = permission or command-not-found; 137 = killed (OOM).
4. Check what it runs: `docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'`.
5. Reproduce: `docker run --rm -it --entrypoint sh IMAGE` and run the command by hand.

Typical causes: missing environment variable or unreachable dependency, server running in the background, wrong `CMD`, CRLF line endings in the entrypoint script.

**Q62. The production server's disk is full. What do you do?**

Find the cause with `df -h`, `docker system df` and `du -ah /var/lib/docker/containers | sort -rh | head`. The usual culprit is an unrotated `json-file` log: truncate it (`truncate -s 0 <file>`) and fix rotation (`max-size`, `max-file`, recreate the container). Free space with `docker builder prune` and `docker image prune -a --filter until=168h`. Do not run `docker volume prune` or `system prune --volumes` blindly; a stopped database's volume looks unused. Then prevent recurrence: log rotation in `daemon.json`, scheduled cleanup, Docker data on a separate disk, disk alerts at 80%.

**Q63. "It works on my machine but not in the container." Where do you look?**

Check: missing system libraries or tools in the slim image; OS or CPU architecture; environment variables and config not passed; the app listening on `127.0.0.1` instead of `0.0.0.0`; case-sensitive paths; file permissions and non-root user; `.dockerignore` excluding needed files; files outside the build context; `localhost` pointing at the wrong thing (use service names); time zone and locale; resource limits. Method: run the container interactively, compare `env`, check logs and `docker compose config`, reproduce the failing command step by step.

**Q64. Your image build takes 20 minutes. How do you speed it up?**

Profile with `--progress=plain`. Copy dependency manifests first and install before copying source; add `.dockerignore`; use BuildKit cache mounts; split into parallel stages; use a remote cache in CI; use a prebuilt base image with heavy dependencies (PyTorch, CUDA); avoid emulated cross-architecture builds on every commit; use a faster runner. Measure after each change.

**Q65. Your image is 3 GB. How do you shrink it?**

Use `docker history` and `dive` to find large layers. Typical fixes: slim base; multi-stage build to drop compilers; keep datasets and model weights out of the image; clean caches in the same layer; for ML, CPU-only wheels when no GPU is needed or `-runtime` rather than `-devel` CUDA bases; remove unused dependencies; `--no-install-recommends`.

**Q66. A deployment caused an outage. What is your process?**

Mitigate first: **roll back** to the last known-good image tag or digest. Communicate status. Then find what changed (commit, config, dependency, migration), reproduce in staging, and write a blameless post-incident review with action items (a test, a health/readiness check that would have caught it, canary rollout, better alerting). Rollback requires immutable tags, retained old images and backward-compatible migrations.

**Q67. A developer committed an API key to git. What do you do?**

Treat the key as compromised and **revoke or rotate it immediately**. Removing the file or rewriting history does not un-leak it; clones, caches and bots may already have it. Then check logs for misuse, remove it from history (`git filter-repo` or BFG) and force-push if appropriate, notify affected parties, enable push protection and secret scanning (gitleaks, GitHub secret scanning), move secrets to a manager and add `.env` to `.gitignore`. If the secret was baked into an image, rotate it, rebuild and purge the image from registries.

**Q68. A container's memory keeps growing until it is OOM-killed. What now?**

Confirm with `docker stats` and `docker inspect` (`OOMKilled=true`, exit 137). Distinguish a **leak** (steady growth, no plateau) from a **limit set too low** (plateaus near the limit) and **runtime mis-sizing** (JVM/Node/Python workers sized to host memory). Profile the app (heap dumps, `tracemalloc`, `pprof`, `memory_profiler`), check caches and unbounded queues, worker counts and per-request memory. Fix the leak or raise limits with headroom. Meanwhile add memory alerts and a restart policy; in Kubernetes consider memory requests equal to limits.

**Q69. Docker Hub pulls fail in CI with "toomanyrequests". How do you fix it permanently?**

Short term: authenticate pulls (`docker login` with a token stored as a CI secret). Durable: run a **pull-through cache** or mirror base images into your own registry (ECR/ACR/GAR/Harbor/Artifactory) and reference those in Dockerfiles; cache layers; consider a paid Docker plan if you rely heavily on Hub.

**Q70. Design a CI/CD pipeline for a small team's containerised web app.**

- **Pull request:** lint, unit tests, cached image build, scan, optional integration tests with Compose.
- **Merge to main:** multi-arch build, tag with git SHA (plus semver on releases), generate SBOM/provenance, scan, push.
- **Deploy:** staging automatically; production after manual approval via a protected environment. Deploy by digest or immutable tag, wait for health, roll back automatically on failure.
- **Secrets:** from the CI/cloud secrets manager using OIDC (no long-lived keys).
- Keep the previous version for rollback and notify the team. Chapter 11 has a working workflow.

**Q71. How would you containerise a legacy application with no tests and hand-installed dependencies?**

Discover the runtime environment (OS version, packages, configuration, ports, files written, environment variables) by inspecting the existing server. Write a Dockerfile that reproduces it on an equivalent base OS; get it running first, optimise later. Move state (databases, uploads) to volumes or external services, configuration to environment variables, logs to stdout, and add a health endpoint if possible. Run side by side with the old system, compare behaviour and migrate gradually. Document unknowns as risks.

**Q72. Containers behind a corporate proxy cannot pull images or install packages. What do you check?**

Proxy settings in three places: Docker Desktop/daemon (pulls), `~/.docker/config.json` or build args (builds), and inside containers (environment variables, `NO_PROXY`). x509 errors mean the proxy re-signs TLS, so install the corporate **CA certificate** in the daemon's trust store and in images (`update-ca-certificates`, plus `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`). Do not disable TLS verification as a permanent fix. A golden base image with the CA built in solves it for all teams.

**Q73. Users see intermittent 502 errors during deployments. Why, and how do you fix it?**

Requests reach instances that are starting or stopping. Causes: no readiness probe, app not handling SIGTERM (in-flight requests dropped), load balancer still routing to a terminating instance (deregistration delay shorter than stop timeout), too short a grace period, all replicas replaced at once. Fixes: readiness probes, graceful shutdown with a `preStop` delay / `stop_grace_period`, `maxUnavailable: 0` with surge, connection draining, at least two replicas, and deployment tests under load.

---

### AI-specific questions

**Q74. Why use Docker for machine learning and AI applications?**

ML environments are fragile: Python, CUDA/cuDNN, framework versions, system libraries and drivers must match. A container freezes that stack, giving reproducible training and inference, easy sharing, identical dev/CI/prod behaviour, isolated dependency sets per project and a clean deployment unit for model serving. One Compose file can package the API, vector database, cache and model server.

**Q75. How do you give a container access to a GPU?**

On a Linux host install the **NVIDIA driver** and the **NVIDIA Container Toolkit**, run `nvidia-ctk runtime configure --runtime=docker`, restart Docker, then run with `--gpus all` (or `--gpus device=0`). In Compose use `deploy.resources.reservations.devices` with `driver: nvidia`, `capabilities: [gpu]`. The container does not contain the GPU driver; the toolkit injects host driver libraries. On Windows, Docker Desktop with WSL 2 and a recent NVIDIA Windows driver supports GPUs. Verify with `docker run --rm --gpus all <cuda-image> nvidia-smi`.

**Q76. Explain CUDA compatibility between host driver and container.**

The **host driver** version sets the maximum supported CUDA version. The container carries the CUDA **toolkit/runtime libraries** (from the base image, such as `nvidia/cuda:12.x-runtime`, or a PyTorch wheel with bundled CUDA). The container's CUDA version must be less than or equal to what the driver supports (NVIDIA guarantees compatibility across minor versions within a major series). A newer image gives errors such as "CUDA driver version is insufficient for CUDA runtime version". Update the host driver or use an image built for older CUDA.

**Q77. Where should model weights live: in the image or outside?**

Usually outside. Weights can be tens of gigabytes; baking them in makes the image huge, slow to build, push and pull (hurting autoscaling and cold starts) and ties a model version to a code release. Store them in object storage or a model registry (S3, GCS, Hugging Face Hub, MLflow) and download at startup into a **volume or cache directory** (restarts reuse them), or mount a shared volume or network filesystem. Exceptions: small models, air-gapped environments, or a fully self-contained immutable artifact (then use a dedicated model layer ordered so code changes do not re-push the weights).

**Q78. How do you containerise a FastAPI model-serving app well?**

Multi-stage or slim base; pinned dependencies in an early layer; code copied last; non-root user; production server (`uvicorn`, or `gunicorn` with uvicorn workers) binding `0.0.0.0`; `/health` and `/ready` endpoints (ready only after the model loads); model loaded at startup, not per request; configuration from environment variables; logs to stdout; memory limit above model size; SIGTERM handling; model cache on a volume; a `HEALTHCHECK`; pinned base image; image scanning. Chapter 9 builds one end to end.

**Q79. How would you run a local LLM with Docker for development?**

Run an inference server container such as **Ollama**: persist models in a volume mounted at `/root/.ollama`, expose port 11434, optionally attach a GPU, and point the app at it with an environment variable like `OLLAMA_BASE_URL=http://ollama:11434` on the same Compose network. Make it an optional Compose **profile** so developers without GPUs can use a hosted API. Docker Desktop also offers **Docker Model Runner** for running models locally (a recent feature; check current docs for platform availability).

**Q80. How do you serve LLMs in production on containers?**

Use a purpose-built inference server (vLLM, Hugging Face Text Generation Inference, NVIDIA Triton or TensorRT-LLM, Ollama for simple cases) in a GPU-enabled image on GPU nodes (Kubernetes GPU node pools with the NVIDIA device plugin / GPU Operator, or a managed service). Key concerns:
- GPU memory sizing (model size plus KV cache) and quantisation.
- Request batching.
- Autoscaling on queue depth or GPU utilisation, not just CPU.
- Slow cold starts: pre-pull images, keep warm replicas, cache weights on a shared volume.
- Readiness only after the model loads.
- Observability: tokens/sec, time-to-first-token, queue length.
- Cost: spot instances for batch workloads, scale to zero where acceptable.

**Q81. Why are AI images often huge, and how do you manage that?**

CUDA libraries, cuDNN, PyTorch/TensorFlow and scientific dependencies add several gigabytes. Mitigations: `-runtime` CUDA bases instead of `-devel`; multi-stage builds; CPU-only builds when no GPU is needed; remove build caches; keep weights out; share a company base image with CUDA and the framework preinstalled so only a thin application layer changes; put heavy, rarely changing layers first so registries and nodes cache them.

**Q82. How do you keep API keys for LLM providers secure in a containerised AI app?**

Do not bake them into images or commit `.env` files. Inject at runtime from a secrets manager or orchestrator secret, as a mounted file or environment variable. Use separate keys per environment with spend limits, rotate them, keep them out of logs (and out of prompts echoed to logs), restrict egress where possible, and use OIDC or scoped secrets in CI. Limit the tools and network access the app has to reduce prompt-injection data exfiltration.

**Q83. How do you make ML training or experiments reproducible with containers?**

Pin the base image (digest), framework and library versions via lock files, and the CUDA version. Record the image digest with each experiment (MLflow / Weights & Biases metadata). Set random seeds. Version data (DVC, lakeFS) and code (git SHA baked in as a label). Run training as a container job (Kubernetes Job, Batch, SageMaker, Vertex AI). GPU nondeterminism may still require framework-level determinism flags.

**Q84. How would you run a RAG (retrieval-augmented generation) stack with Docker Compose?**

Services:
- API (FastAPI) that handles queries.
- **Vector store** (pgvector in Postgres, Qdrant, Chroma, Weaviate or Redis with vector search) with a **named volume** so embeddings persist.
- Cache (Redis).
- Embedding/LLM provider (hosted API or local server).
- Ingestion worker that chunks documents and writes embeddings.
- Optional reverse proxy.

Add health checks and `depends_on: service_healthy` so the API starts after the store is ready, model/embedding caches on volumes, secrets via environment or files, and memory limits. In production consider a managed vector database and run ingestion as a scheduled job.

**Q85. How do you scale AI inference differently from a normal web service?**

Web APIs scale on CPU or request rate with small, fast-starting replicas. Inference replicas are heavy (GPU-bound, multi-GB models, slow start), so scale on **GPU utilisation, queue depth or tokens per second**. Tune batching and concurrency, keep a warm minimum, place replicas on GPU nodes with node affinity, share GPUs carefully (MIG/time-slicing) if utilisation is low, consider model-specific routing, and design clients for long-running or streaming responses (timeouts, backpressure, retries).

**Q86. How do you observe and debug an LLM-backed container service?**

Standard signals: logs, metrics (latency percentiles, error rate, saturation), traces. LLM-specific: per-request token counts and cost, time-to-first-token, tokens per second, prompt and response sizes, provider error rates and retries, cache hit rates, GPU utilisation and memory, queue depth. Use OpenTelemetry-based tracing (LLM observability tools build on it) with redaction of prompts and personal data in logs. Add evaluation and guardrail metrics for output quality.

**Q87. What are the security risks specific to AI containers?**

- Large untrusted dependency trees (many Python packages).
- **Pickle**-based model files that can execute code when loaded (prefer safetensors; load only trusted models).
- Runtime model downloads from public hubs (verify sources and checksums).
- Prompt injection leading tools to run commands or reach internal networks (sandbox tool execution in separate unprivileged containers with no secrets and restricted network).
- Data leaks through logs.
- GPU-sharing isolation limits and privileged GPU containers.

Apply standard hardening plus network egress controls and model provenance checks.

**Q88. When would you not use Docker for an AI workload?**

Specialised bare-metal performance tuning (some HPC clusters use Apptainer/Singularity because Docker needs a root daemon), very small prototypes where a virtual environment is enough, notebook-only exploratory work (a container can still help share it), or managed platforms that abstract containers away (hosted fine-tuning or inference APIs). Those platforms use containers internally.

---

## 2. How to answer well in interviews

1. **Structure:** one-sentence definition, two or three sentences on how it works, why it matters or when to use it, and a concrete example from something you built.
2. **STAR format for scenario questions:** Situation, Task, Action, Result. Prepare three stories: a bug you debugged (use Chapter 12), an optimisation with measured before/after numbers (image size or build time), and a design decision with trade-offs (for example Compose rather than Kubernetes).
3. **State limits honestly.** For example: "I have run this on a single VM with Compose and studied Kubernetes concepts, but have not operated a production cluster."
4. **Give trade-offs.** "It depends, and here is on what."
5. **Ask clarifying questions** in design scenarios (traffic, team size, budget, compliance).
6. **State the command and what it does.**
7. **Mention security and operations unprompted:** non-root, secrets, scanning, health checks, rollback, backups.

---

## 3. Staged learning roadmap

Assumed pace: 8-10 hours per week. Each week has reading, a build task and a checkpoint to pass before moving on.

| Week | Theme | Read | Build | Checkpoint |
|---|---|---|---|---|
| **1** | Foundations and installation | [Ch 1](01-what-is-docker.md), [Ch 2](02-installation-and-operating-systems.md), [Ch 3](03-core-cli-commands.md) | Install Docker; run `hello-world`, nginx, Postgres, a Python container; use every Chapter 3 command once | Explain container vs VM vs image; run a container, view logs, exec in, stop and remove it from memory |
| **2** | Dockerfiles | [Ch 4](04-dockerfile-deep-dive.md) | Containerise a small app; write a multi-stage Dockerfile; add `.dockerignore`; cut image size by at least 50% and note before/after | Rebuild after a code change takes under 10 seconds (cache works); runs as non-root; you can explain every line |
| **3** | Storage and networking | [Ch 5](05-storage-and-networking.md) | Run Postgres with a named volume, recreate the container, prove the data survived; build a two-container app on a user-defined network using service names | Explain volume vs bind mount, why `localhost` fails between containers, and what `-p` does |
| **4** | Compose and the sample app | [Ch 6](06-docker-compose.md), [Ch 9](09-hands-on-ai-app-walkthrough.md) | Run the FastAPI + Redis + Postgres + nginx sample app; add healthchecks and `depends_on` conditions; try the Ollama profile; change something and redeploy | Bring the stack up and down from scratch; explain every service and network; recover from a deliberately broken config using logs |
| **5** | CI/CD and registries | [Ch 11 sections 3-6](11-production-and-enterprise.md) | Push the sample app to GHCR from GitHub Actions: Buildx with cache, SHA and version tags, scan, multi-arch push | A merge to `main` produces a scanned image with traceable tags; rollback by redeploying a previous tag |
| **6** | Production server, security | [Ch 10](10-security.md), [Ch 11 sections 7-18](11-production-and-enterprise.md) | Rent a small Linux VM; install Docker; deploy with Compose behind Caddy with HTTPS; add log rotation, firewall, backups, uptime monitor; harden containers | App reachable over HTTPS; everything returns after a VM reboot; a backup restore succeeded |
| **7** | Docker for AI/ML, troubleshooting | [Ch 8](08-docker-for-ai-and-ml.md), [Ch 12](12-troubleshooting-and-cheatsheet.md) | Containerise a model-serving API; try a local LLM via Ollama; run with `--gpus` if available; cause and fix five Chapter 12 errors | Diagnose OOM, port, DNS, permission and architecture errors without searching |
| **8** | Capstone, portfolio, interviews | [Ch 13](13-interview-questions-roadmap-glossary.md), [Ch 14](14-real-world-case-study.md) | Polish the repository (Section 5); write the README; record a demo; answer 30 interview questions aloud | A friend gets the app running in under five minutes from the README alone; you can discuss the project for 5 minutes without notes |

### Weeks 9-16 and beyond (suggested)

| Phase | Focus | Project |
|---|---|---|
| **Weeks 9-12** | Kubernetes fundamentals: pods, deployments, services, ingress/Gateway, config/secrets, probes, HPA, Helm. Use `kind`, `minikube` or `k3d` locally | Port the Compose app to Kubernetes manifests, then a Helm chart; deploy to a local cluster; do a rolling update and rollback |
| **Weeks 13-16** | Infrastructure as code and cloud: Terraform, one cloud (AWS/Azure/GCP), managed Kubernetes or serverless containers, managed databases, IAM basics | Provision the VM/cluster and registry with Terraform; deploy through GitOps (Argo CD or Flux) |
| **Months 5-6** | Observability and reliability: Prometheus, Grafana, Loki, OpenTelemetry, SLOs, alerting, incident practice | Add dashboards, alerts and tracing; run a game day |
| **Ongoing (AI focus)** | MLOps: experiment tracking (MLflow, Weights & Biases), model registries, feature stores, pipelines (Kubeflow, Airflow, Prefect, Dagster), serving (vLLM, KServe, BentoML, Triton), evaluation, GPU scheduling | Serve an open model with vLLM on Kubernetes with autoscaling and monitoring |

### Self-assessment checklist

- [ ] Write a production-quality Dockerfile for a Python or Node app from a blank file
- [ ] Explain image layers and caching, and speed up a slow build
- [ ] Set up a multi-service Compose stack with volumes, networks, health checks and secrets
- [ ] Debug a crashing container using logs, exit codes, `inspect` and `exec`
- [ ] Build and push a multi-arch image from CI with sensible tags
- [ ] Deploy to a Linux VM with HTTPS and rollback
- [ ] Describe how to secure a container image and its supply chain
- [ ] Explain how a Compose file maps to Kubernetes objects
- [ ] Run and serve a model in a container, with or without a GPU

---

## 4. What to learn after Docker

| Next skill | Why | First step |
|---|---|---|
| **Linux fundamentals** (processes, permissions, systemd, networking, `journalctl`, `ss`, `iptables`) | Containers are Linux; production problems end here | Administer a small VM for a month |
| **Git and CI/CD** (GitHub Actions, GitLab CI, branching, pull requests) | The delivery pipeline is how containers reach production | Automate build, test, scan, push for a project |
| **Kubernetes** | Standard platform for containers at scale; the most requested skill next to Docker | Official tutorials, then `kind` and your own manifests; consider CKAD/CKA |
| **A cloud provider** (AWS, Azure or GCP) | Registries, managed Kubernetes, serverless containers, managed databases, IAM | Deploy a container to Cloud Run, ECS Fargate or Azure Container Apps |
| **Terraform / infrastructure as code** | Reproducible infrastructure reviewable in git | Provision a VM, network and registry with code |
| **Observability** (Prometheus, Grafana, Loki, OpenTelemetry) | Running systems need visibility | Add dashboards and alerts to a project |
| **Security** (supply chain, secrets, policy as code, CIS benchmarks) | Increasingly a hiring differentiator | Add scanning, signing and Conftest policies to a pipeline |
| **GitOps** (Argo CD, Flux) | Modern Kubernetes deployment model | Deploy an app to a local cluster via Argo CD |
| **MLOps and AI infrastructure** (MLflow, model registries, vLLM/Triton/KServe, GPU scheduling, vector databases, LLM evaluation and observability) | Turning models into reliable products | Serve a model, version it, monitor it |
| **Alternative tools** (Podman, Buildah, nerdctl, containerd, Dev Containers, Buildpacks) | Organisations choose different tools; concepts transfer | Run the Compose app with Podman |

Certifications are optional; projects matter more. Existing options: Linux Foundation **CKAD** and **CKA** (Kubernetes; exam-only price was $445 at the time of writing), a cloud associate-level certification (AWS Solutions Architect Associate, Azure Administrator, Google Associate Cloud Engineer), and HashiCorp **Terraform Associate**. The **Docker Certified Associate (DCA)** has been handled by Mirantis rather than Docker. Unverified: sources conflict on whether it is still offered; check before planning around it.

---

## 5. Portfolio guide

### 5.1 What this repository should demonstrate

1. Deep understanding of Docker, beyond commands.
2. Ability to build something real (the sample AI app).
3. Production thinking (CI/CD, security, logging, backups).
4. Clear communication.

### 5.2 Recommended repository structure

```
docker-complete-guide/
├── README.md                    <- most important file (see 5.3)
├── LICENSE                      <- e.g. MIT (for docs)
├── .gitignore                   <- includes .env, *.pem, *.key, data/, models/
├── docs/
│   ├── 01-what-is-docker.md
│   ├── ...
│   └── 14-real-world-case-study.md
├── sample-ai-app/
│   ├── README.md                <- how to run this app in 3 commands
│   ├── compose.yaml
│   ├── compose.override.yaml    <- developer conveniences
│   ├── .env.example             <- fake placeholder values only
│   ├── .dockerignore
│   ├── api/
│   │   ├── Dockerfile
│   │   ├── requirements.txt
│   │   └── app/...
│   ├── nginx/
│   │   └── nginx.conf
│   └── scripts/
├── .github/
│   └── workflows/
│       └── ci.yaml              <- builds, lints, scans
├── assets/
│   └── architecture.png         <- diagram, screenshots, demo GIF
└── CONTRIBUTING.md  (optional)
```

### 5.3 README structure

1. **Title and one-line pitch**, for example: "A complete beginner-to-enterprise Docker guide, with a production-style AI chat app (FastAPI, Redis, Postgres, nginx, optional Ollama)."
2. **Badges** (build status, license).
3. **Screenshot or demo GIF** and an **architecture diagram**.
4. **Quick start**, three copy-pastable commands. State the URL to open and what to expect. Test on a clean machine.
   ```bash
   git clone https://github.com/<you>/docker-complete-guide.git
   cd docker-complete-guide/sample-ai-app
   cp .env.example .env && docker compose up -d
   ```
5. **Table of contents** linking the 14 chapters with one-sentence summaries.
6. **What you will learn / who this is for.**
7. **Key design decisions** (why Compose, multi-stage, non-root).
8. **Tech stack** and **prerequisites** (Docker version tested).
9. **Roadmap** (for example "Kubernetes manifests, Terraform for the VM").
10. **License and author contact.**

Tips: lead with outcomes, keep the quick start short, use relative links, do not claim "used in production" unless true, keep images small.

### 5.4 What to show

- CI pipeline running green, with scanning and multi-arch builds.
- Measured improvements, for example "Image size 1.2 GB to 180 MB via multi-stage build and slim base; rebuild time 4 min to 12 s with layer ordering and cache mounts." Use real numbers only.
- Security work: non-root user, pinned versions, scan results, secrets handling.
- Operations work: health checks, log rotation, backups, rollback procedure.
- A short "lessons learned" page.
- A few architecture decision records on trade-offs.

### 5.5 What not to commit

| Never commit | Instead |
|---|---|
| `.env` files with real values | Commit `.env.example` with placeholders (`OPENAI_API_KEY=your-key-here`); add `.env` to `.gitignore` |
| API keys, tokens, passwords, private keys (`*.pem`, `*.key`, SSH keys), cloud credentials, Docker Hub/GHCR tokens | Use secrets managers and CI secrets; rotate anything that leaked |
| `~/.docker/config.json` contents, `kubeconfig` files, Terraform state (`*.tfstate`), `*.tfvars` with secrets | Use remote state and encrypted variables |
| Database dumps or files with real personal data | Use synthetic data |
| Large model weights and datasets | Use Git LFS, object storage or Hugging Face; document the download |
| Build output and caches (`node_modules`, `__pycache__`, `.venv`, `dist/`) | `.gitignore` them |
| Employer or client material (internal hostnames, project names, proprietary code, screenshots of internal systems) | Build demos from public data and your own work |
| Personal data in screenshots (emails, IP addresses, tokens in terminal output) | Review every screenshot before publishing |

Before the first public push, scan the entire history, not only current files:

```bash
gitleaks detect --source . --log-opts="--all"
```

- `gitleaks detect` scans for patterns that look like secrets.
- `--source .` scans the current repository.
- `--log-opts="--all"` includes every branch and commit.
- If it reports anything, rotate the credential first, then rewrite history or start a fresh repository. `trufflehog` is an alternative.
- Also enable GitHub secret scanning and push protection, and consider a local pre-commit hook.

A committed secret must be treated as stolen: revoke it, then clean up. Deleting it in a later commit does not remove it from history.

### 5.6 LinkedIn and resume

**LinkedIn "Featured" section:** pin the repository and add a short post. Template:

> I spent [N weeks] turning "I know what Docker is" into "I can ship containers to production." I wrote a complete Docker guide (14 chapters, from your first `docker run` to CI/CD, security and running AI workloads) and built a containerised AI chat app (FastAPI, Redis, Postgres, nginx, optional local LLM) to prove it. Biggest lessons: [two or three real lessons, such as "layer ordering cut my rebuild time from X to Y"]. Repo: [link]. Feedback welcome.

**Resume entry** (use real numbers; delete any bullet you cannot support):

```
Docker Complete Guide and AI Chat App (personal project)            github.com/<you>/docker-complete-guide
- Authored a 14-chapter open-source guide covering Docker fundamentals, Compose, security hardening,
  CI/CD, production operations and AI/ML containerisation.
- Built a containerised AI chat service (FastAPI, Redis, PostgreSQL, nginx, optional Ollama LLM profile)
  with health checks, non-root images, and multi-stage builds; reduced image size from [X] to [Y].
- Implemented a GitHub Actions pipeline: cached multi-arch builds, vulnerability scanning, signed
  releases and automated rollback-capable deployment to a Linux VM behind a TLS reverse proxy.
- Documented a production-readiness checklist, incident runbook and 40+ troubleshooting scenarios.
```

Skills line: Docker, Docker Compose, Buildx/BuildKit, GitHub Actions, GHCR, Linux, Nginx/Caddy, PostgreSQL, Redis, FastAPI, Trivy, Prometheus/Grafana (list only tools you used).

### 5.7 Talking about it in interviews

- **30-second pitch:** "I built a containerised AI chat app and wrote a full Docker guide around it. The app has an API, a database, a cache and a reverse proxy, runs from one Compose command, and has a CI pipeline that builds, scans and publishes the image. I used it to learn how production container workflows work."
- Be ready to explain any file in the repo. Remove or learn any line you cannot explain.
- Prepare a "what I would change" answer: for example, move Postgres to a managed service, add centralised logging and tracing, use Kubernetes only when team and traffic justify it.
- Prepare three STAR stories (Section 2).
- Describe it as a learning and portfolio project; do not overclaim.
- Prepare AI-angle answers: GPU containers, model storage, serving, LLM app security (Q74-Q88).

### 5.8 Keeping it current

Update dependency and action versions every few months (Dependabot/Renovate), re-run CI, fix broken links, refresh facts that age (rate limits, tool versions), and add a "last verified" date to chapters.

---

## 6. Recommended further resources

Existence and status were checked in October 2026. For books and courses, search the exact title; editions change.

### Official documentation

- **Docker Docs**, <https://docs.docker.com/>: Get Started, Dockerfile reference, Compose reference, Engine, Build (BuildKit), Docker Hub, Docker Desktop troubleshooting. Includes Dockerfile best practices.
- **Compose Specification**, <https://compose-spec.io/>
- **Open Container Initiative**, <https://opencontainers.org/>: image and runtime specifications.
- **Kubernetes Documentation**, <https://kubernetes.io/docs/>
- **The Twelve-Factor App**, <https://12factor.net/>
- **CNCF Landscape**, <https://landscape.cncf.io/>
- **Cloud provider docs:** AWS (ECS, ECR, EKS), Azure (ACR, AKS, Container Apps), Google Cloud (Artifact Registry, GKE, Cloud Run).
- **Tool docs:** Trivy (aquasecurity.github.io/trivy), Sigstore/Cosign (docs.sigstore.dev), OpenTelemetry (opentelemetry.io), Prometheus (prometheus.io), Grafana (grafana.com/docs).
- **Google SRE books**, free at <https://sre.google/books/>: the operations mindset behind Chapter 11.

### Books

- **Docker Deep Dive**, Nigel Poulton. The 2025 edition (Leanpub, last updated May 2025) added a Docker Model Runner chapter and updates on BuildKit, Buildx and Docker Debug. Unverified: no 2026 edition found; check the publisher.
- **The Kubernetes Book**, Nigel Poulton.
- **Kubernetes Up & Running** (O'Reilly), Brendan Burns, Joe Beda, Kelsey Hightower, Lachlan Evenson.
- **Kubernetes in Action** (Manning), Marko Lukša. Check the current edition.
- **Docker in Practice** (Manning), Ian Miell and Aidan Hobson Sayers.
- **Docker: Up & Running** (O'Reilly), Sean P. Kane and Karl Matthias.
- **Learn Docker in a Month of Lunches** (Manning), Elton Stoneman. Beginner-friendly.
- **Container Security** (O'Reilly), Liz Rice. How isolation works in the kernel.
- **Designing Machine Learning Systems** (O'Reilly), Chip Huyen. MLOps.

### Courses and labs

- Docker's official "Get started" guides (docs.docker.com).
- TechWorld with Nana (YouTube): free Docker and Kubernetes tutorials.
- KodeKloud: hands-on courses with browser labs (paid, some free content).
- Killercoda (killercoda.com): free interactive Kubernetes and Docker scenarios.
- iximiuz Labs (labs.iximiuz.com): container and Kubernetes labs.
- Linux Foundation / CNCF training and the CKA/CKAD/CKS exams.
- **Play with Docker** was announced as deprecated and unavailable from 1 March 2026. Do not rely on tutorials that depend on it; use Docker Desktop, a cloud VM or the alternatives above.

### Community

Docker Community Forums (forums.docker.com), r/docker and r/kubernetes, CNCF and Kubernetes Slack, Stack Overflow. The awesome-docker list on GitHub is a curated link collection; check that it is maintained.

### Tools worth knowing

`dive` (layer explorer), `hadolint` (Dockerfile linter), `Trivy`, `Grype` and `Syft` (scanning, SBOM), `Cosign` (signing), `lazydocker` and `ctop` (terminal dashboards), `Portainer` (web UI), `Dockge` (Compose manager), `Renovate` and `Dependabot` (dependency updates), `k9s` (Kubernetes terminal UI), `kind`/`k3d`/`minikube` (local Kubernetes), `Helm`, `Argo CD`, `Flux`.

---

## 7. Glossary (A-Z)

### A

- **ADD**: Dockerfile instruction like `COPY` that also unpacks tar archives and downloads URLs. Prefer `COPY`.
- **Alpine**: tiny Linux distribution (a few MB) used as a base image. Uses musl libc, which can break some Python packages.
- **AppArmor**: Linux security module restricting what a program may do; Docker applies a default profile where supported.
- **ARG**: Dockerfile variable available only while building.
- **Artifact registry**: server storing build outputs (images, packages). Google's product is named Artifact Registry.
- **Attestation**: signed statement about an image, such as provenance or SBOM.
- **Autoscaling**: adding or removing instances automatically based on load.

### B

- **Base image**: the image in `FROM` that yours builds on (for example `python:3.12-slim`).
- **Bind mount**: maps a host file or folder into a container.
- **Blue/green deployment**: run old (blue) and new (green) versions side by side, switch traffic in one step, keep blue for rollback.
- **Bridge network**: Docker's default single-host virtual network; containers get private IPs and NAT outward.
- **BuildKit**: Docker's modern build engine (parallelism, cache mounts, secrets, attestations).
- **Build cache**: stored results of earlier build steps, reused when inputs are unchanged.
- **Build context**: files (usually a folder) sent to the builder by `docker build`.
- **Buildx**: Docker CLI plug-in exposing BuildKit's features, including multi-platform builds.

### C

- **cAdvisor**: Google tool (run as a container) that reports per-container resource metrics.
- **Canary release**: send a small share of traffic to a new version first, increase if healthy.
- **Capabilities (Linux)**: the pieces root's power is divided into; containers get a reduced set.
- **cgroups (control groups)**: kernel feature that limits and accounts for CPU, memory, I/O and process counts of process groups.
- **CI/CD**: Continuous Integration (build and test every change) and Continuous Delivery/Deployment (release automatically).
- **CMD**: Dockerfile instruction giving the default command or arguments.
- **Compose (Docker Compose)**: tool and YAML format for multi-container applications.
- **Compose Specification**: open standard defining the Compose file format.
- **Container**: isolated process (or group) running from an image, sharing the host kernel.
- **containerd**: container runtime daemon managing images and lifecycle; used by Docker and Kubernetes.
- **Container runtime**: software that creates and runs containers (runc, containerd, CRI-O).
- **Control plane (Kubernetes)**: cluster management components (API server, scheduler, controllers, etcd).
- **Copy-on-write (CoW)**: data is shared until modified, then copied; how container filesystems use image layers.
- **Cosign**: Sigstore tool to sign and verify images.
- **CUDA**: NVIDIA's platform and libraries for general computation on GPUs; required by most GPU machine learning.
- **CVE (Common Vulnerabilities and Exposures)**: public identifier for a known security flaw, such as CVE-2024-12345.

### D

- **Daemon (`dockerd`)**: Docker background service that builds images and runs containers; the `docker` command is its client.
- **Dangling image**: image with no tag (`<none>`), usually left by rebuilds.
- **Declarative**: describing the desired end state and letting the system reach it (Compose, Kubernetes).
- **Deployment (Kubernetes)**: object managing identical pods, rollouts and rollbacks.
- **Digest**: SHA-256 hash that permanently identifies image content (`sha256:...`).
- **Distroless**: minimal images with only the app and runtime dependencies, no shell or package manager.
- **DNS (embedded)**: Docker's built-in resolver (127.0.0.11) for name lookup on user-defined networks.
- **Docker Desktop**: desktop app for Windows, macOS and Linux bundling Docker Engine, a Linux VM where needed, a GUI, Kubernetes and extras.
- **Docker Engine**: open-source container engine (daemon, API, CLI) installed on Linux servers.
- **Docker Hub**: Docker's public registry and image catalogue.
- **Docker Scout**: Docker's image analysis service for vulnerabilities and base-image recommendations.
- **Docker socket (`/var/run/docker.sock`)**: file through which clients talk to the daemon; access equals root on the host.
- **Dockerfile**: text recipe for building an image.
- **.dockerignore**: file listing paths excluded from the build context.

### E

- **Entrypoint (`ENTRYPOINT`)**: fixed executable run at container start; `CMD` supplies default arguments.
- **ENV**: Dockerfile instruction setting an environment variable in the image and container.
- **Ephemeral**: short-lived; containers are ephemeral, so keep important data elsewhere.
- **Exit code**: number a process returns when it ends; 0 is success, above 128 means killed by a signal.
- **EXPOSE**: Dockerfile instruction documenting a port; does not publish it.
- **Egress**: outbound network traffic (often billed in the cloud).

### F

- **Fargate (AWS)**: AWS serverless compute for containers.
- **Fluent Bit / Fluentd**: log collectors and forwarders.
- **Flatcar Container Linux**: minimal, auto-updating Linux distribution for running containers.

### G

- **GHCR (GitHub Container Registry)**: GitHub's registry at `ghcr.io`.
- **GitOps**: manage infrastructure and deployments by storing desired state in git and letting an agent apply it.
- **Golden image**: approved, hardened, regularly rebuilt base image teams must use.
- **GPU (Graphics Processing Unit)**: processor with thousands of cores suited to parallel maths in machine learning.
- **Grafana**: dashboard tool for metrics and logs.

### H

- **Harbor**: open-source enterprise container registry (CNCF graduated project).
- **Health check (`HEALTHCHECK`)**: command Docker runs periodically to test whether the container works.
- **Helm**: Kubernetes package manager; packages are charts.
- **Horizontal scaling**: adding more copies of a service instead of a bigger machine.
- **Host network**: mode where the container shares the host's network stack.
- **HPA (Horizontal Pod Autoscaler)**: Kubernetes component that adjusts replica counts from metrics.
- **Hypervisor**: software that creates and runs virtual machines.

### I

- **Image**: read-only template (layers plus metadata) used to create containers.
- **Image index / manifest list**: pointer from one tag to several platform-specific images (multi-architecture tags).
- **Immutable**: cannot change after creation; immutable tags and infrastructure improve reproducibility.
- **Inference**: using a trained model to produce outputs, as opposed to training it.
- **Infrastructure as code (IaC)**: defining servers and networks in code (Terraform, CloudFormation) so they are repeatable and reviewable.
- **Ingress**: Kubernetes rules (or the controller implementing them) routing external HTTP(S) traffic to services; being superseded by the Gateway API.
- **Init process**: tiny PID 1 program (like tini) that forwards signals and reaps zombies; enabled with `--init`.

### J-K

- **JSON-file (logging driver)**: default driver writing logs as JSON files on the host; does not rotate by default.
- **Kaniko**: tool that builds images inside a container without a Docker daemon (check current maintenance status).
- **Kubernetes (K8s)**: leading container orchestration platform.
- **kubectl**: Kubernetes command-line client.
- **kubelet**: agent on each Kubernetes node that starts and monitors pods.
- **KEDA**: Kubernetes Event-Driven Autoscaling; scales workloads on events such as queue length.
- **KV cache**: in LLM serving, stored attention keys/values of previous tokens; consumes GPU memory per request.

### L

- **Label**: key-value metadata on images, containers or Kubernetes objects.
- **Layer**: one read-only filesystem change set in an image, created by a Dockerfile instruction.
- **Liveness probe**: check deciding whether to restart an unresponsive container.
- **Live-restore**: daemon option that keeps containers running while Docker restarts.
- **Load balancer**: component distributing requests across instances.
- **LLM (Large Language Model)**: neural network trained on large text corpora to generate and understand text.
- **Loki**: Grafana's log aggregation system.
- **Logging driver**: plug-in determining where container logs go.

### M

- **Macvlan**: network mode giving containers their own MAC and IP on the physical LAN.
- **Manifest**: JSON document describing an image (config and layers); in Kubernetes, a YAML file describing objects.
- **Metrics**: numeric measurements over time (CPU, request rate, latency).
- **MLOps**: practices and tools for deploying, monitoring and maintaining ML systems in production.
- **Model weights**: learned numerical parameters of a model; often gigabytes.
- **Multi-arch image**: image tag with variants for several CPU architectures.
- **Multi-stage build**: Dockerfile with multiple `FROM` stages so build tools stay out of the final image.

### N

- **Namespace (Linux)**: kernel feature giving a process its own view of a resource (PIDs, network, mounts); basis of container isolation.
- **Namespace (Kubernetes)**: virtual cluster partition for teams, environments and quotas.
- **Network policy**: Kubernetes rules defining which pods may talk to which.
- **Node**: machine (physical or virtual) in a cluster.
- **NVIDIA Container Toolkit**: software that lets containers use NVIDIA GPUs by injecting host driver libraries.

### O

- **OCI (Open Container Initiative)**: standards body for image and runtime formats.
- **Ollama**: tool for running open LLMs locally with a simple API; commonly run as a container.
- **OOM killer**: Linux kernel component that kills processes when memory is exhausted (exit code 137).
- **OPA (Open Policy Agent)**: general-purpose policy engine (Rego); **Conftest** applies it to config files.
- **OpenTelemetry (OTel)**: open standard and toolkit for traces, metrics and logs.
- **Orchestrator**: software that runs and manages containers across machines (Kubernetes, Swarm, ECS).
- **Overlay network**: virtual network spanning multiple hosts.
- **overlay2**: default Docker storage driver on modern Linux, built on OverlayFS.

### P

- **Pod**: Kubernetes' smallest deployable unit: one or more containers sharing network and storage.
- **Podman**: daemonless, rootless-friendly container engine with a Docker-compatible CLI.
- **Port publishing (`-p`)**: mapping a host port to a container port.
- **PID 1**: first process in a container, with special duties for signals and zombies.
- **Privileged container**: container with nearly all host capabilities and device access; avoid.
- **Profile (Compose)**: label that makes a service start only when requested.
- **Prometheus**: open-source monitoring system that scrapes and stores metrics.
- **Provenance**: record of how and where an artifact was built.
- **Pull-through cache**: registry that fetches from an upstream registry on first request and serves copies afterwards.
- **PVC (PersistentVolumeClaim)**: Kubernetes request for storage.

### Q-R

- **QEMU**: emulator that lets one CPU architecture run another's programs; used for cross-platform builds.
- **Quantisation**: reducing numeric precision of model weights (for example 16-bit to 4-bit) to cut memory and speed up inference at some accuracy cost.
- **RAG (Retrieval-Augmented Generation)**: LLM technique that fetches relevant documents and adds them to the prompt.
- **RBAC (Role-Based Access Control)**: permissions granted by roles.
- **Readiness probe**: check deciding whether an instance receives traffic.
- **Registry**: server storing and distributing images.
- **Replica**: one of several identical copies of a service.
- **Repository**: named collection of image versions (tags) in a registry.
- **Restart policy**: rule telling Docker when to restart a container.
- **Reverse proxy**: server that receives client requests and forwards them to backends (nginx, Caddy, Traefik).
- **Rolling update**: replacing instances gradually to avoid downtime.
- **Rootless mode**: running Docker and containers without root privileges on the host.
- **RPO / RTO**: Recovery Point Objective (acceptable data loss) and Recovery Time Objective (acceptable downtime).
- **runc**: low-level runtime that creates containers per the OCI spec.

### S

- **SBOM (Software Bill of Materials)**: list of all components inside an image.
- **Seccomp**: Linux feature filtering which system calls a process may make; Docker applies a default profile.
- **Secret**: sensitive value (password, key, token) that must be protected.
- **SELinux**: Linux mandatory access control system common on Red Hat-family distributions.
- **Semantic versioning (semver)**: `MAJOR.MINOR.PATCH` numbering with defined meaning.
- **Service (Compose)**: one container definition (image, ports, volumes) in a Compose file.
- **Service (Kubernetes)**: stable network endpoint in front of a set of pods.
- **Sidecar**: helper container deployed alongside the main one (logging, proxy, debugging).
- **SIGKILL / SIGTERM**: signals; SIGTERM asks a process to stop gracefully, SIGKILL forces it to stop immediately.
- **Sigstore**: open-source project for signing and verifying software, including keyless signing.
- **SLO / SLI / SLA**: Service Level Objective (target), Indicator (measurement), Agreement (contract).
- **Stateful / stateless**: whether a service keeps data between requests.
- **Sticky sessions**: load balancing that keeps sending a user to the same instance.
- **Swarm (Docker Swarm)**: Docker's built-in clustering and orchestration mode.

### T

- **Tag**: mutable label on an image (`1.4.2`, `latest`).
- **Terraform**: popular infrastructure-as-code tool.
- **tmpfs**: memory-backed temporary filesystem; contents vanish when the container stops.
- **TLS**: Transport Layer Security, the encryption behind HTTPS.
- **Tracing**: following one request through multiple services as timed spans.
- **Traefik**: reverse proxy that configures itself from container labels.
- **Trivy**: open-source vulnerability and misconfiguration scanner.

### U-V

- **Union filesystem**: filesystem presenting several layered directories as one.
- **User namespace**: maps container user IDs to different host IDs, so container root is not host root.
- **Vector database**: database optimised to store embeddings and find similar items (pgvector, Qdrant, Chroma, Weaviate).
- **Embedding**: list of numbers representing the meaning of text or other data, used for similarity search.
- **Vertical scaling**: giving a service a bigger machine.
- **VM (virtual machine)**: emulated computer with its own operating system.
- **Volume**: Docker-managed persistent storage that outlives containers.
- **vLLM**: high-throughput open-source LLM inference server.
- **Vulnerability scanning**: comparing image contents against databases of known flaws.

### W-Z

- **Watchtower**: tool that auto-updated running containers when new images appeared; the original project was archived in December 2025.
- **Workload**: application or job running on a platform.
- **WSL 2 (Windows Subsystem for Linux 2)**: lightweight Linux VM inside Windows, used by Docker Desktop.
- **YAML**: indentation-based data format used by Compose and Kubernetes (spaces only, never tabs).
- **Zero-downtime deployment**: releasing a new version with no period of unavailability.
- **Zombie process**: finished process whose exit status has not been collected by its parent; a PID 1 that does not reap children lets them accumulate.

### Additional terms

- **12-factor app**: methodology for portable, scalable cloud apps (config in environment, stateless processes, logs to stdout).
- **Admission controller**: Kubernetes component that accepts or rejects objects at creation (policy and signature checks).
- **API server**: front door of the Kubernetes control plane.
- **Buildpacks**: tools that turn source code into container images without a Dockerfile (Cloud Native Buildpacks).
- **Chart**: Helm package of Kubernetes templates.
- **CRI (Container Runtime Interface)**: Kubernetes interface for container runtimes.
- **CRI-O**: lightweight container runtime built for Kubernetes.
- **DaemonSet**: Kubernetes object that runs one pod on every node (log collectors, monitoring agents).
- **Dev Container**: standard for defining a containerised development environment (VS Code, GitHub Codespaces).
- **etcd**: key-value database holding Kubernetes' state.
- **Gateway API**: newer Kubernetes API for traffic routing, successor to Ingress.
- **Idempotent**: safe to run repeatedly with the same result.
- **Job / CronJob**: Kubernetes objects for run-to-completion and scheduled tasks.
- **Kustomize**: Kubernetes-native way to customise manifests with overlays.
- **Landing zone**: pre-configured, governed cloud account structure for organisations.
- **Service mesh**: infrastructure (Istio, Linkerd) providing encrypted, observable service-to-service traffic.
- **StatefulSet**: Kubernetes object for stateful apps needing stable identity and storage.
- **Taint / toleration**: Kubernetes mechanism to keep pods off (or allow them onto) certain nodes, commonly used for GPU nodes.
- **Time-to-first-token (TTFT)**: time an LLM service takes to emit the first output token; a key latency metric.

---

Previous: [Chapter 12: Troubleshooting and Cheat Sheet](12-troubleshooting-and-cheatsheet.md) | Next: [Chapter 14: Real-World Case Study](14-real-world-case-study.md) | Start over: [Chapter 1](01-what-is-docker.md)
