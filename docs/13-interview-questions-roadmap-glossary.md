# Chapter 13: Interview Questions, Learning Roadmap, Portfolio Guide and Glossary

> **What you will learn:** 75+ interview questions with model answers (beginner to advanced, scenario-based, and AI-specific), an eight-week learning plan with checkpoints and projects, what to learn after Docker, exactly how to present this repository on GitHub, LinkedIn and your resume, a verified list of further resources, and an A-Z glossary of 150+ terms in plain English.

This is the chapter you will come back to the most. The first part helps you *prove* what you learned. The second part tells you what to do next. The last part is a reference you can search when a word in a job description or a colleague's sentence is new to you.

How to use the interview section: do **not** memorise the answers word for word. Interviewers can tell. Read each question, answer it out loud in your own words, then compare with the model answer. If you missed a point, reread the matching chapter. Where an answer is about commands, run the commands once on your own machine.

---

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

Docker is a platform for packaging an application together with everything it needs to run (code, runtime, libraries, system tools, configuration) into a standard unit called an **image**, and running that image as an isolated process called a **container**. It solves "it works on my machine": the container carries its own environment, so it behaves the same on a laptop, a CI server and a production server. It also makes deployments repeatable, lets many apps with conflicting dependencies share one machine, and starts in seconds instead of minutes.

**Q2. What is the difference between a container and a virtual machine?**

A virtual machine (VM) emulates a whole computer: virtual hardware plus a complete guest operating system with its own kernel, running on a **hypervisor**. A container is just an isolated **process** on the host that **shares the host's kernel**; isolation comes from Linux **namespaces** (separate views of processes, network, filesystem) and **cgroups** (resource limits). Consequences: containers are much lighter (megabytes instead of gigabytes), start in about a second, and you can run far more of them per machine. VMs give stronger isolation (separate kernels) and can run different operating systems. Containers can only run workloads that fit the host kernel type (Linux containers need a Linux kernel).

```
 VM:                                  Container:
 [App][App]                           [App][App]
 [Guest OS][Guest OS]                 [libs ][libs ]
 [   Hypervisor    ]                  [ Container engine ]
 [   Host OS/HW    ]                  [   Host OS (shared kernel) ]
```

**Q3. What is the difference between an image and a container?**

An image is a read-only template: the filesystem layers plus metadata (default command, environment, exposed ports). A container is a running (or stopped) instance of an image with a thin writable layer on top. Analogy: an image is a class (or a recipe, or an installer); a container is an object created from it (or a dish cooked from the recipe). You can start many containers from one image.

**Q4. What is a Dockerfile?**

A text file of instructions (`FROM`, `COPY`, `RUN`, `CMD` and others) that tells Docker how to build an image, step by step. Each instruction that changes the filesystem creates a **layer**. `docker build` reads it and produces the image.

**Q5. What is a registry, and what is Docker Hub?**

A registry is a server that stores and distributes images. You `push` images to it and `pull` them from it. Docker Hub is the default public registry (`docker.io`), home to official images such as `nginx`, `postgres` and `python`. Others include GitHub Container Registry (`ghcr.io`), Amazon ECR, Azure Container Registry, Google Artifact Registry and self-hosted ones like Harbor.

**Q6. What does `docker run -d -p 8080:80 --name web nginx` do?**

It creates and starts a container from the `nginx` image (pulling it first if it is not local). `-d` runs it **detached** (in the background). `-p 8080:80` **publishes** container port 80 on host port 8080, so `http://localhost:8080` reaches nginx (host port on the left, container port on the right). `--name web` names the container, so you can say `docker logs web` instead of using an ID.

**Q7. What is the difference between `CMD` and `ENTRYPOINT`?**

Both define what runs when the container starts. `ENTRYPOINT` is the fixed executable; `CMD` provides **default arguments** (or the default command if there is no entrypoint) that are easily replaced by whatever you type after the image name in `docker run`. The common pattern: `ENTRYPOINT ["python", "app.py"]` plus `CMD ["--port", "8000"]` lets users override only the arguments. To replace the entrypoint itself you need `--entrypoint`. Prefer the **exec form** (JSON array) so the process receives signals directly.

**Q8. What is the difference between `COPY` and `ADD`?**

`COPY` copies files from the build context into the image. `ADD` does that too, but also extracts local tar archives and can fetch URLs. Because that extra behaviour is surprising, best practice is to use `COPY` unless you specifically need tar auto-extraction.

**Q9. What is the difference between `EXPOSE` and `-p`?**

`EXPOSE 8000` in a Dockerfile is **documentation**: it records which port the app listens on. It does not open anything to the outside. `-p HOST:CONTAINER` (or `ports:` in Compose) actually **publishes** the port by mapping a host port to the container. (`-P` publishes all exposed ports to random host ports.)

**Q10. What are the ways to persist data, and how do they differ?**

- **Named volume:** storage managed by Docker (under `/var/lib/docker/volumes` on Linux), independent of any container's life. Best for databases and application data.
- **Bind mount:** maps a specific host path into the container. Great for development (live code), but ties the container to the host's layout and permissions.
- **tmpfs mount:** in-memory storage that disappears when the container stops; useful for sensitive or scratch data.

Without any of these, data lives in the container's writable layer and is **lost when the container is removed**.

**Q11. What happens when you run `docker run hello-world`?**

The Docker CLI sends the request to the Docker daemon. The daemon checks for the image locally; if absent it **pulls** it from the registry (Docker Hub by default). It then creates a container (a writable layer, network interface, and so on), asks the container runtime (containerd, which uses runc) to start the process inside new namespaces and cgroups, connects the output to your terminal, and the container exits when the program finishes.

**Q12. What is the difference between `docker stop` and `docker kill`?**

`docker stop` sends **SIGTERM** (a polite request to shut down), waits (10 seconds by default), then sends **SIGKILL** if the process is still running. `docker kill` sends SIGKILL (or another signal you choose) immediately with no chance for cleanup. Use `stop` normally so the app can finish requests and flush data.

**Q13. Why do image layers matter?**

Each filesystem-changing Dockerfile instruction creates a read-only layer, stored by content hash. Layers are **shared** between images (saving disk and bandwidth) and **cached** during builds (unchanged layers are reused). If a layer changes, all later layers are rebuilt. So you order instructions from least to most frequently changing (install dependencies before copying source code) to make builds fast.

**Q14. What is `.dockerignore` and why use it?**

A file listing paths excluded from the **build context** (the files sent to the builder). It keeps images small, speeds up builds, avoids invalidating the cache by irrelevant changes, and prevents leaking secrets (`.env`, `.git`, private keys) into images. Typical entries: `.git`, `node_modules`, `__pycache__`, `*.log`, `.env`.

**Q15. What is the difference between `ENV` and `ARG`?**

`ARG` is a **build-time** variable (set with `--build-arg`), available only while building. `ENV` sets an environment variable that exists at build time **and** at runtime in the container. Neither is a safe place for secrets: both can be seen in image metadata or history (`docker history`, `docker inspect`). Use BuildKit secret mounts for build-time secrets and a secrets manager or mounted files at runtime.

**Q16. What is Docker Compose?**

A tool for defining and running **multi-container** applications in a single YAML file (`compose.yaml`): services, networks, volumes, environment, ports and dependencies. One command (`docker compose up -d`) starts the entire stack; `docker compose down` removes it. It is ideal for local development, testing and small deployments.

**Q17. How do containers talk to each other?**

On a **user-defined bridge network** (which Compose creates automatically), containers can reach each other by **service name** through Docker's embedded DNS (`http://db:5432`). They use the container's own port; publishing a port to the host is only needed for access from outside. On the *default* bridge network there is no name-based DNS.

**Q18. What is the difference between `docker ps` and `docker ps -a`?**

`docker ps` lists only running containers; `docker ps -a` lists all containers, including stopped (exited) ones. You need `-a` to find a crashed container and read its logs.

**Q19. Why do containers start so much faster than VMs?**

No operating system boots. A container is just a new process started with isolated namespaces and cgroup limits on an already-running kernel, using an image filesystem that is already on disk. The only work is setting up the filesystem view and starting the process.

**Q20. What does the tag `latest` mean?**

Nothing special: it is simply the default tag used when you do not specify one. It does not mean "newest"; it is whatever the publisher last tagged `latest`. It is mutable, so using it in production gives unreproducible deployments and makes rollbacks impossible. Pin explicit versions (`postgres:17.2`) or digests.

**Q21. How do you see what is going on inside a container?**

`docker logs` (stdout/stderr), `docker exec -it NAME sh` (a shell inside), `docker inspect` (configuration and state), `docker stats` (resource use), `docker top` (processes), and `docker events` (lifecycle events). Chapter 12 covers them in detail.

**Q22. Why is Docker so widely used by developers?**

Reproducible environments ("works on my machine" disappears), fast onboarding (clone, `docker compose up`), isolation of dependencies, consistent CI/CD, an enormous catalogue of ready-made images, easy local replication of production-like stacks, and portability across laptops, servers and clouds.

---

### Intermediate questions

**Q23. What is a multi-stage build and why use it?**

A Dockerfile with several `FROM` stages. Early stages contain compilers and build tools and produce artifacts; the final stage copies only the artifacts (`COPY --from=builder /app/dist /app`) onto a small runtime base. Benefits: much smaller images, smaller attack surface, no build tools or source code in production, and secrets used during the build never reach the final image. It also allows building a specific stage for debugging with `--target`.

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

Use a smaller base (`-slim`, Alpine where compatible, or distroless); multi-stage builds; install only what you need (`--no-install-recommends`, `--no-cache-dir`); clean package caches in the *same* `RUN` layer; use `.dockerignore`; avoid copying datasets and model weights into the image; combine related commands to avoid leaving deleted files in earlier layers; inspect with `docker history` and `dive`.

**Q25. Explain Docker's network drivers.**

- **bridge** (default): a private virtual network on the host; containers on the same user-defined bridge reach each other by name; outbound traffic is NATed; inbound needs published ports.
- **host:** the container shares the host's network stack (no isolation, no port mapping, best raw performance, Linux only).
- **none:** no networking except loopback.
- **overlay:** multi-host networking for Swarm, spanning several Docker hosts.
- **macvlan:** gives the container its own MAC/IP on the physical network, so it looks like a separate machine on the LAN.

**Q26. Why use a user-defined bridge instead of the default bridge?**

User-defined bridges provide automatic DNS resolution by container name, better isolation (only containers attached to that network can talk), and the ability to connect/disconnect containers on the fly. The default bridge uses IPs only (or legacy `--link`) and puts every unrelated container on one shared network.

**Q27. How does `depends_on` work, and what is its limitation?**

It controls start **order**. By default Compose waits only until the dependency container has *started*, not until the service inside is *ready*. To wait for readiness define a `healthcheck` on the dependency and use `depends_on: { db: { condition: service_healthy } }`. The application should also retry connections, because dependencies can restart at any time in production.

**Q28. What is a HEALTHCHECK and what does Docker do with it?**

A command Docker runs periodically inside the container; exit code 0 = healthy, 1 = unhealthy. The status appears in `docker ps` and can gate `depends_on` and `docker compose up --wait`. Plain Docker does **not** restart unhealthy containers; Swarm, Kubernetes (via probes) and ECS do replace them.

**Q29. How do you handle secrets with Docker?**

Never in the image (`ENV`, `ARG`, copied files) or in git. At **build time** use BuildKit secret mounts (`RUN --mount=type=secret`). At **runtime** use Docker/Compose secrets (files mounted at `/run/secrets/...`), orchestrator secrets (Kubernetes Secrets with encryption at rest or External Secrets), or a secrets manager (Vault, AWS Secrets Manager, Azure Key Vault, GCP Secret Manager) injected at start. Prefer short-lived credentials, rotate regularly, and scan history for leaks.

**Q30. How does Docker isolate containers from each other and from the host?**

Linux **namespaces** (PID, network, mount, UTS, IPC, user, cgroup) give each container its own view of the system; **cgroups** limit resources; **capabilities** split root's powers (Docker drops many by default); **seccomp** filters dangerous system calls; **AppArmor/SELinux** add mandatory access control; the container's root filesystem is separate. Containers are still **not** as isolated as VMs, because they share the kernel; a kernel exploit can escape, which is why you also run as non-root, drop capabilities, keep a read-only filesystem and patch.

**Q31. What is the PID 1 problem?**

The first process in a container (PID 1) has special duties in Linux: it must reap orphaned **zombie** child processes, and it does not get default signal handlers (a SIGTERM is ignored unless the program installs a handler). A shell script or app that ignores SIGTERM makes `docker stop` hang for 10 seconds, then get killed. Solutions: use exec-form `CMD`/`ENTRYPOINT` so your app is PID 1 and handles signals, `exec` your app at the end of shell entrypoint scripts, and use a tiny init (`docker run --init` / Compose `init: true`, which uses tini).

**Q32. Shell form vs exec form for CMD/ENTRYPOINT?**

Shell form (`CMD python app.py`) runs through `/bin/sh -c`, so the shell is PID 1 and may not forward signals; it supports variable expansion. Exec form (`CMD ["python","app.py"]`) runs the program directly with no shell: correct signal handling, but no shell features unless you invoke `sh -c` explicitly. Use exec form for long-running services.

**Q33. What restart policies exist?**

`no` (default), `on-failure[:max]`, `always`, and `unless-stopped`. `always` restarts even if you stopped it manually once the daemon restarts; `unless-stopped` honours a manual stop. In production use `unless-stopped` (single host) or let the orchestrator handle replacement.

**Q34. How do you limit a container's CPU and memory?**

`--memory`/`--memory-swap`, `--cpus`, `--cpu-shares`, `--pids-limit` in `docker run`; `deploy.resources.limits` (or `mem_limit`, `cpus`) in Compose; `resources.requests/limits` in Kubernetes. Memory limits are hard (exceeding it triggers an OOM kill, exit 137); CPU limits throttle.

**Q35. What are logging drivers? What is the default, and what is the risk?**

A logging driver decides where a container's stdout/stderr go: `json-file` (default), `local`, `journald`, `syslog`, `fluentd`, `gelf`, `awslogs`, `gcplogs`, `splunk`, `none`. The default `json-file` **does not rotate**, so a chatty container can fill the disk; configure `max-size` and `max-file` (or use `local`, which rotates by default).

**Q36. Tag vs digest?**

A tag is a mutable human label; a digest (`sha256:...`) is the immutable content hash of the image manifest. Deploying by digest guarantees you run exactly what you tested. Typical practice: tag with a version and git SHA for humans, deploy by digest or immutable tag.

**Q37. What is BuildKit?**

Docker's modern build engine (the default builder). Advantages: parallel execution of independent stages, better caching (including cache mounts and exporting cache to registries), secret and SSH mounts, multi-platform builds, attestations (SBOM and provenance), and build checks. `docker buildx` is its CLI front end.

**Q38. How do you build multi-architecture images?**

With Buildx: `docker buildx build --platform linux/amd64,linux/arm64 -t repo/app:1.0 --push .`. It produces a manifest list (OCI image index) with one image per architecture under one tag; clients pull the right one automatically. Non-native architectures are built via QEMU emulation (slow) or native builders/cross-compilation. Needed for Apple Silicon development with amd64 servers, and for AWS Graviton.

**Q39. Docker vs containerd vs runc vs Podman vs OCI?**

- **OCI** (Open Container Initiative): standards for image format and runtime.
- **runc:** low-level runtime that actually creates the container (namespaces, cgroups) from an OCI bundle.
- **containerd:** a higher-level runtime daemon that manages image pulls, storage and container lifecycle, calling runc. Docker Engine uses it; Kubernetes can use it directly.
- **Docker Engine:** adds the developer experience (CLI, API, build, networking, volumes) on top of containerd.
- **Podman:** a daemonless, rootless-first container engine with a Docker-compatible CLI; common on RHEL.

**Q40. What is rootless Docker/containers, and why does it matter?**

Running the Docker daemon and containers as an unprivileged user (using user namespaces), so even if a container escapes, the attacker has only that user's privileges, not host root. Trade-offs: some features (certain networking modes, low ports, some storage drivers) are limited.

**Q41. Why is mounting `/var/run/docker.sock` into a container dangerous?**

The socket is the Docker API. Anything that can talk to it can start a privileged container that mounts the host's root filesystem, which is full root control of the host. Mounting it into an application (or CI) container hands that application root-equivalent power over the host. Avoid it; if a tool truly needs it (monitoring, some CI), restrict it with a socket proxy exposing a read-only API subset, or use rootless/daemonless alternatives (BuildKit rootless, Kaniko, Buildah).

**Q42. What are Compose profiles and override files?**

Profiles (`profiles: ["ollama"]`) mark services that start only when requested (`docker compose --profile ollama up`), making optional components such as a local LLM server or debugging tools opt-in. Override files (`compose.override.yaml` automatically merged, or extra `-f` files) layer environment-specific changes on a base file: for example `compose.yaml` plus `compose.prod.yaml`.

**Q43. How does build-time vs run-time configuration differ, and what is the twelve-factor guidance?**

Build-time values are baked into the image (dependencies, compiled assets); run-time configuration (database URL, feature flags, secrets) should come from the environment so the same image runs in every environment. Twelve-factor says store config in the environment, treat backing services as attached resources via URLs, and keep processes stateless.

**Q44. What is Docker Swarm, and is it still relevant?**

Swarm mode is Docker's built-in clustering and orchestration: managers/workers, services with replicas, rolling updates, overlay networks, and secrets, using a Compose-like file via `docker stack deploy`. It is simpler than Kubernetes. As of 2026 it is not formally deprecated and still receives fixes, but is generally considered to be in maintenance mode with little new feature development; the ecosystem has moved to Kubernetes. Fine for small, existing setups; usually not the choice for new large projects.

---

### Advanced questions

**Q45. How does the overlay2 storage driver work?**

An image is a stack of read-only layers. **OverlayFS** (a union filesystem in the Linux kernel) merges multiple directories: `lowerdir` (the read-only image layers), `upperdir` (the container's writable layer), a `workdir`, and presents the merged result as `merged`. Reads fall through to the lower layers; writes use **copy-on-write**: the first time a file from a lower layer is modified, it is copied up to the upper layer. Deletions are recorded with "whiteout" files. This is why large write-heavy workloads belong in volumes, and why deleting a file in a later layer does not shrink the image.

**Q46. What is inside an image? Explain manifest, config and layers.**

An image consists of a **manifest** (JSON listing the config blob and the ordered layer blobs, each identified by digest), a **config** (JSON with environment, entrypoint, exposed ports, architecture, history, and the ordered list of layer diff IDs), and the **layers** (compressed tar archives of filesystem changes). Everything is **content-addressable**: the identifier is the SHA-256 of the content, so identical layers are stored once and integrity is verifiable. A multi-arch tag points to an **image index** (manifest list) that references one manifest per platform.

**Q47. How do cgroups v2 enforce limits, and what happens on memory exhaustion?**

Cgroups form a hierarchy of process groups with controllers for CPU, memory, I/O and PIDs. For memory, the container's cgroup has `memory.max` (hard limit) and `memory.high` (throttle threshold). When processes in the cgroup exceed the limit and the kernel cannot reclaim memory, the OOM killer kills a process in that cgroup, typically the main process, so the container exits with 137 and `OOMKilled=true`. CPU limits use bandwidth control (`cpu.max`, a quota per period) that throttles rather than kills. Docker Engine 29 deprecated cgroup v1.

**Q48. Describe the path of a packet to a published port.**

With `-p 8080:80`, Docker adds **iptables/nftables DNAT rules** (in the `nat` table's DOCKER chain) that rewrite traffic arriving at the host on port 8080 to the container's IP on port 80, plus rules in the FORWARD chain permitting it. Outbound traffic from containers is **masqueraded** (source NAT) to the host's address. For connections from the host itself (localhost) Docker may use a userspace `docker-proxy`. Because Docker inserts its rules ahead of typical `ufw` rules, published ports can bypass the host firewall, so bind to `127.0.0.1`, avoid publishing, or use the `DOCKER-USER` chain or an external firewall.

**Q49. How would you secure the container supply chain end to end?**

Use minimal, pinned, trusted base images (ideally an internal golden image); lint Dockerfiles (hadolint) and enforce policy (OPA/Conftest); scan images and dependencies in CI and continuously in the registry; generate an **SBOM** and **provenance** attestations; **sign** images (Cosign/Sigstore keyless with OIDC) and verify signatures at deploy time (admission control with Kyverno or Gatekeeper); use immutable tags and digests; protect the CI system (least-privilege tokens, pinned third-party actions, ephemeral runners, no long-lived cloud keys); restrict who can push; scan for leaked secrets; patch and rebuild regularly. Real-world reminder: in 2026 a widely used scanner's GitHub Action was compromised through mutable version tags, which is why you pin actions to verified commit SHAs.

**Q50. How do you achieve zero-downtime deployments with containers?**

Run at least two replicas behind a load balancer; use **readiness probes** so traffic only goes to ready instances; do **rolling updates** with surge capacity and no unavailable instances (Kubernetes `maxUnavailable: 0`), or **blue/green** (switch the router between two full stacks), or **canary**; handle SIGTERM for **graceful shutdown** (stop accepting new requests, finish in-flight ones); keep database migrations **backward compatible** (expand/contract); and have an automatic **rollback** on failed health checks or SLO breaches.

**Q51. Compare Docker Compose, Docker Swarm and Kubernetes.**

Compose: single host, simple YAML, best for dev/test and small deployments, no self-healing across machines. Swarm: multi-host clustering with a Compose-like model; simple, with rolling updates and secrets, little new development. Kubernetes: the industry-standard orchestrator with a huge ecosystem (autoscaling, operators, RBAC, network policies, service meshes, GitOps), steep learning curve and operational cost, available as managed services (EKS/AKS/GKE). Choose by team size and needs, not fashion.

**Q52. Why did Kubernetes remove dockershim, and did that break Docker images?**

Kubernetes originally talked to Docker through a built-in adapter ("dockershim"). Since Docker itself wraps containerd, and Kubernetes standardised on the **CRI** (Container Runtime Interface), maintaining the shim was extra burden; it was removed in v1.24 and clusters use containerd or CRI-O directly. Images did not break: they follow the OCI standard, so images built with Docker run everywhere. What changed is that `docker` commands on a node no longer show Kubernetes containers (use `crictl`).

**Q53. What Linux capabilities, seccomp and read-only filesystem settings would you apply to harden a container?**

`--cap-drop ALL` then add back only what is needed (for example `NET_BIND_SERVICE`); `--security-opt no-new-privileges`; keep Docker's default seccomp profile (or a tighter custom one); `--read-only` with `tmpfs` for `/tmp`; run as a non-root user (`USER 10001`); no `--privileged`; avoid mounting the Docker socket and host paths; set resource limits; use user namespaces or rootless mode; minimal image (distroless) with no shell.

**Q54. What is distroless, and when do you use `scratch`?**

Distroless images (from Google and others) contain only the application runtime and its dependencies, with no package manager or shell, shrinking the attack surface and size. `scratch` is the empty base image, used for statically linked binaries (Go, Rust). Trade-off: harder to debug (no shell): use `docker debug`, ephemeral debug containers, or debug variants of the image.

**Q55. How do you make Docker builds fast in CI?**

Order the Dockerfile for caching; use `.dockerignore`; BuildKit cache mounts for package managers; multi-stage builds with parallelisable stages; **remote cache** (`--cache-to/--cache-from` with `type=gha` or `type=registry`) because CI runners are ephemeral; a pull-through cache or mirrored base images to avoid pulling from Docker Hub each time and hitting rate limits; build only changed services in a monorepo; avoid multi-arch emulation on every PR (do amd64 on PRs, multi-arch on release).

**Q56. What are the ways to build images inside CI without exposing the Docker socket?**

Use BuildKit directly (`docker buildx` with the docker-container driver, which runs BuildKit in its own container), **rootless BuildKit**, **Kaniko** (builds in userspace inside a container; check its maintenance status before choosing it for new projects), **Buildah**, or cloud build services. "Docker-in-Docker" (a privileged `docker:dind` sidecar) works but needs privileged mode, which is a security risk on shared runners.

**Q57. How does Docker handle DNS inside containers?**

Containers on user-defined networks use Docker's **embedded DNS server** at `127.0.0.11`, which resolves container and service names (and aliases) on that network and forwards other queries to the host's configured DNS servers. Containers on the default bridge copy the host's `/etc/resolv.conf`. Problems arise with host stub resolvers like `127.0.0.53`, VPNs and corporate DNS; fix with `dns` settings in `daemon.json` or Compose.

**Q58. What is `live-restore` and what are its limits?**

A daemon setting that keeps containers running while the Docker daemon is stopped or restarted, so daemon upgrades or crashes do not stop workloads. Limits: not for major version upgrades, not for Swarm mode, not for Windows containers, and container logs can block if the daemon is down long enough to fill the log pipe buffer.

**Q59. Explain how you would debug a container that has no shell.**

Use `docker debug` (a Docker Desktop feature that injects a toolbox without modifying the image), attach a sidecar such as `nicolaka/netshoot` sharing the target's network (and PID) namespace (`--network container:NAME --pid container:NAME`), copy files out with `docker cp`, inspect from the host via `/proc/<pid>`, read logs and `inspect` data, or in Kubernetes use `kubectl debug` with an ephemeral container.

**Q60. What makes a build reproducible, and why does it matter?**

Pinning everything that can change: base image by digest, OS packages and language dependencies by exact versions and lock files, build tool versions, and avoiding `latest`, unpinned `apt-get install`, and network fetches of moving targets. It matters for security (you know exactly what is inside, and can rebuild it), debugging (same inputs produce the same image) and compliance. Bit-for-bit reproducibility additionally needs normalised timestamps and ordering (BuildKit supports `SOURCE_DATE_EPOCH`).

---

### Scenario-based questions

**Q61. A container exits immediately after `docker run -d`. How do you diagnose it?**

1. `docker ps -a` for the exit code. 2. `docker logs NAME` for the error. 3. Interpret the code: 0 means the process finished (a one-shot command, or the app daemonised/backgrounded and PID 1 ended), 1 an application error, 126/127 a permission or command-not-found problem, 137 killed (OOM). 4. Check what it runs: `docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'`. 5. Reproduce interactively: `docker run --rm -it --entrypoint sh IMAGE` and run the command by hand. Typical root causes: missing environment variable or unreachable dependency, server running in the background instead of foreground, wrong `CMD`, CRLF line endings in the entrypoint script.

**Q62. The production server's disk is full. What do you do?**

First stabilise: find the cause with `df -h`, `docker system df` and `du -ah /var/lib/docker/containers | sort -rh | head`. The usual culprit is an unrotated `json-file` log; truncate it (`truncate -s 0 <file>`) and fix rotation (`max-size`, `max-file`, then recreate the container). Free space safely with `docker builder prune` and `docker image prune -a --filter until=168h` (images can be re-pulled). Do **not** run `docker volume prune` or `system prune --volumes` blindly, since stopped databases' volumes look unused. Then prevent recurrence: log rotation in `daemon.json`, scheduled cleanup, Docker data on a separate disk, disk alerts at 80%.

**Q63. "It works on my machine but not in the container." Where do you look?**

Differences between environments: missing system libraries or tools in the slim image; different OS/CPU architecture; environment variables and config not passed in; the app listening on `127.0.0.1` rather than `0.0.0.0`; paths and case-sensitivity (Linux is case-sensitive); file permissions and user (non-root); `.dockerignore` excluding needed files; relying on files outside the build context; localhost pointing at the wrong thing (use service names); time zone/locale; resource limits. Method: run the container interactively, compare `env`, check logs and `docker compose config`, and reproduce the failing command step by step.

**Q64. Your image build takes 20 minutes. How do you speed it up?**

Profile with `--progress=plain` to find slow steps. Fix cache usage by copying dependency manifests first and installing before copying the rest of the source; add `.dockerignore` to shrink the context; use BuildKit cache mounts for pip/npm/apt; split into multi-stage with parallel stages; use a remote cache in CI; use a pre-built base image containing heavy dependencies (PyTorch, CUDA); avoid emulated cross-architecture builds on every commit; use a faster, larger runner. Measure after each change.

**Q65. Your image is 3 GB. How do you shrink it?**

`docker history` and `dive` to find the large layers. Typical wins: switch to a slim base; multi-stage build to drop compilers; do not bake datasets or model weights into the image (mount or download at runtime); clean caches in the same layer; for ML, choose CPU-only wheels when no GPU is needed, or use `-runtime` rather than `-devel` CUDA bases; remove unused dependencies; use `--no-install-recommends`.

**Q66. A deployment caused an outage. What is your process?**

Mitigate first: **roll back** to the last known-good image tag/digest (one command), then diagnose. Communicate status. Afterwards, find what changed (commit, config, dependency, database migration), reproduce in staging, and write a blameless post-incident review with action items (add a test, add a health/readiness check that would have caught it, add canary rollout, improve alerting). Mention prerequisites that make rollback possible: immutable tags, retained old images, backward-compatible migrations.

**Q67. A developer committed an API key to git. What do you do?**

Treat the key as compromised. **Revoke/rotate it immediately** (removing the file or rewriting history does not un-leak it, because copies may exist in clones and caches and bots scrape public repos within minutes). Then check logs for misuse, remove it from history (`git filter-repo` or BFG) and force-push if appropriate, notify affected parties, enable push protection and secret scanning (gitleaks, GitHub secret scanning), move secrets to a manager, and add `.env` to `.gitignore`. If the secret was baked into an image, rotate it and rebuild and purge that image from registries.

**Q68. A container's memory keeps growing until it gets OOM-killed. What now?**

Confirm with `docker stats` and `docker inspect` (`OOMKilled=true`, exit 137). Distinguish a **leak** (steadily growing, never plateaus) from a **limit set too low** (plateaus near the limit) from **runtime mis-sizing** (JVM/Node/Python workers sized to host memory). Profile the app (heap dumps, `tracemalloc`, `pprof`, `memory_profiler`), look at caches and unbounded queues/lists, check worker counts and per-request memory, then fix the leak or raise limits with headroom. Meanwhile, add memory alerts and a restart policy so users are less affected; in Kubernetes consider memory requests = limits.

**Q69. Docker Hub pulls fail in CI with "toomanyrequests". How do you fix it permanently?**

Short-term: authenticate pulls (`docker login` with a token stored as a CI secret). Durable: run a **pull-through cache** or mirror frequently used base images into your own registry (ECR/ACR/GAR/Harbor/Artifactory) and reference those in Dockerfiles, so CI never depends on Docker Hub limits or availability; cache layers; consider a paid Docker plan if you rely heavily on Hub.

**Q70. Design a CI/CD pipeline for a small team's containerised web app.**

On pull request: lint, unit tests, build the image (cached), scan it, optionally run integration tests with Compose. On merge to main: build multi-arch, tag with git SHA (plus semver on releases), generate SBOM/provenance, scan, push to the registry. Deploy to staging automatically; to production after a manual approval via a protected environment. Deploy by digest/immutable tag, health-wait, auto-rollback on failure. Secrets from the CI/cloud secrets manager using OIDC (no long-lived keys). Notify the team and keep the previous version for rollback. Chapter 11 has a full working workflow.

**Q71. How would you containerise a legacy application with no tests and hand-installed dependencies?**

Start by discovering the runtime environment (OS version, packages, configuration, ports, files it writes, environment variables) by inspecting the existing server. Write a Dockerfile that reproduces it using an equivalent base OS; get it running first, optimise later. Separate state (databases, uploaded files) into volumes or external services; move configuration to environment variables; send logs to stdout (or tail log files in a minimal way as a stopgap); add a health endpoint if possible. Run side by side with the old system, compare behaviour, and migrate gradually. Document unknowns as risks.

**Q72. Containers behind a corporate proxy cannot pull images or install packages. What do you check?**

Proxy settings in three places: Docker Desktop/daemon (for pulls), `~/.docker/config.json` or build args (for builds), and inside containers (environment variables, `NO_PROXY`). If you see x509 errors the proxy is re-signing TLS, so the corporate **CA certificate** must be installed in the daemon's trust store and in the images (`update-ca-certificates`, plus `REQUESTS_CA_BUNDLE`, `NODE_EXTRA_CA_CERTS`). Do not disable TLS verification as the permanent fix. A golden base image with the CA built in solves it for everyone.

**Q73. Users see intermittent 502 errors during deployments. Why, and how do you fix it?**

Requests are reaching instances that are starting or stopping. Causes: no readiness probe (traffic sent before ready), app not handling SIGTERM (in-flight requests dropped), the load balancer still routing to a terminating instance (deregistration delay shorter than the stop timeout), too short a stop grace period, or all replicas being replaced at once. Fixes: readiness probes, graceful shutdown with a `preStop` delay/`stop_grace_period`, `maxUnavailable: 0` with surge, connection draining, at least two replicas, and tests that deploy under load.

---

### AI-specific questions

**Q74. Why use Docker for machine learning and AI applications?**

ML environments are notoriously fragile: Python version, CUDA/cuDNN versions, framework versions, system libraries and drivers must all match. A container freezes that stack, giving reproducible training and inference, easy sharing ("here is the image"), identical dev/CI/prod behaviour, isolated dependency sets for different projects (one needs PyTorch 2.x with CUDA 12, another needs an old TensorFlow), and a clean deployment unit for model serving on Kubernetes or serverless platforms. Containers also package the whole app: API, vector database, cache and model server in one Compose file.

**Q75. How do you give a container access to a GPU?**

On a Linux host install the **NVIDIA driver** and the **NVIDIA Container Toolkit**, configure the Docker runtime (`nvidia-ctk runtime configure --runtime=docker`), restart Docker, then run with `--gpus all` (or `--gpus device=0`). In Compose use a `deploy.resources.reservations.devices` entry with `driver: nvidia`, `capabilities: [gpu]`. The container does **not** contain the GPU driver; the toolkit injects the driver libraries from the host. On Windows, Docker Desktop with WSL 2 and a recent NVIDIA Windows driver supports GPUs. Verify with `docker run --rm --gpus all <cuda-image> nvidia-smi`.

**Q76. Explain CUDA compatibility between host driver and container.**

The **host driver** version determines the maximum CUDA version supported. The container carries the CUDA **toolkit/runtime libraries** (via the base image, such as `nvidia/cuda:12.x-runtime` or a PyTorch wheel with bundled CUDA). The container's CUDA version must be less than or equal to what the host driver supports (with NVIDIA's compatibility guarantees across minor versions in a major series). If the image is newer than the driver supports, you get errors like "CUDA driver version is insufficient for CUDA runtime version". Fix by updating the host driver or using an image built for an older CUDA.

**Q77. Where should model weights live: in the image or outside?**

Usually outside. Weights can be tens of gigabytes; baking them into the image makes it huge, slow to build, push and pull (hurting autoscaling and cold starts), and ties a model version to a code release. Better: store them in object storage or a model registry (S3, GCS, Hugging Face Hub, MLflow) and either download at startup into a **volume or cache directory** (so restarts reuse them), or mount a shared volume or network filesystem. Exceptions: small models, air-gapped environments, or when you want a fully self-contained immutable artifact (then use a dedicated model image layer ordered so code changes do not re-push the weights).

**Q78. How do you containerise a FastAPI model-serving app well?**

Multi-stage or slim base; install pinned dependencies in an early layer; copy code last; run as non-root; run with a production server (`uvicorn`/`gunicorn` with uvicorn workers) binding `0.0.0.0`; expose `/health` and `/ready` (ready only after the model is loaded); load the model at startup (not per request); read configuration from environment variables; log to stdout; set memory limits above model size; handle SIGTERM for graceful shutdown; mount model cache as a volume; add a `HEALTHCHECK`; pin the base image; scan the image. Chapter 9 builds one end to end.

**Q79. How would you run a local LLM with Docker for development?**

Run an inference server container such as **Ollama** (pull models into a persistent volume mounted at `/root/.ollama`, expose port 11434, optionally give it a GPU) and point the application at it via an environment variable like `OLLAMA_BASE_URL=http://ollama:11434` on the same Compose network. Make it an optional Compose **profile** so developers without GPUs can skip it and use a hosted API instead. Docker Desktop also offers a **Docker Model Runner** feature for running models locally (a recent Docker Desktop feature, so check the current docs for availability on your platform).

**Q80. How do you serve LLMs in production on containers?**

Use a purpose-built inference server (vLLM, Hugging Face Text Generation Inference, NVIDIA Triton or TensorRT-LLM, Ollama for simpler cases) in a GPU-enabled image, on GPU nodes (Kubernetes GPU node pools with the NVIDIA device plugin / GPU Operator, or a managed service). Key concerns: GPU memory sizing (model size plus KV-cache), quantisation, request batching, autoscaling on queue depth or GPU utilisation (not just CPU), slow cold starts (large images and weights: pre-pull images, keep warm replicas, cache weights on a shared volume), readiness only after the model loads, observability (tokens/sec, time-to-first-token, queue length), and cost control (spot instances for batch workloads, scale to zero where acceptable).

**Q81. Why are AI images often huge, and how do you manage that?**

CUDA libraries, cuDNN, PyTorch/TensorFlow and scientific dependencies commonly add several gigabytes. Mitigations: use `-runtime` CUDA bases instead of `-devel`; multi-stage builds; install CPU-only builds when no GPU is needed; remove build caches; keep weights out; share a company base image with CUDA and the framework pre-installed so only your thin application layer changes; ensure layer order puts heavy, rarely changing layers first so registries and nodes can cache them.

**Q82. How do you keep API keys for LLM providers secure in a containerised AI app?**

Never bake them into images or commit `.env` files. Inject at runtime from a secrets manager or orchestrator secret; mount as a file or environment variable; use separate keys per environment with spend limits; rotate them; keep them out of logs (and out of prompts echoed to logs); restrict egress where possible; and for CI use OIDC or scoped secrets. Also protect against prompt-injection data exfiltration by limiting what tools and network access the app has.

**Q83. How do you make ML training or experiments reproducible with containers?**

Pin the base image (digest), framework and library versions via lock files, the CUDA version, and record the image digest with each experiment (in MLflow/Weights & Biases metadata); set random seeds; version data (DVC, lakeFS) and code (git SHA baked in as a label); run training as a container job (Kubernetes Job, Batch, SageMaker, Vertex AI) so the same image runs everywhere. Note that GPU nondeterminism may still require framework-level determinism flags.

**Q84. How would you run a RAG (retrieval-augmented generation) stack with Docker Compose?**

Services: the API (FastAPI) that handles queries; a **vector store** (pgvector in Postgres, Qdrant, Chroma, Weaviate or Redis with vector search) with a **named volume** so embeddings persist; a cache (Redis); an embedding/LLM provider (hosted API or a local server); an ingestion worker container that chunks documents and writes embeddings; optionally a reverse proxy. Health checks and `depends_on: service_healthy` so the API starts after the store is ready; model/embedding caches on volumes; secrets via environment or files; limits on memory. In production consider a managed vector database and run ingestion as a scheduled job.

**Q85. How do you scale AI inference differently from a normal web service?**

Web APIs scale on CPU or request rate with fast-starting, small replicas. Inference replicas are heavy (GPU-bound, multi-GB models, slow start), so you scale on **GPU utilisation, queue depth or tokens per second**; use batching and concurrency tuning on the server; keep a warm minimum; place replicas with node affinity to GPU nodes; share GPUs carefully (MIG/time-slicing) if utilisation is low; consider model-specific routing; and design the client for long-running or streaming responses (timeouts, backpressure, retries).

**Q86. How do you observe and debug an LLM-backed container service?**

Standard: logs, metrics (latency percentiles, error rate, saturation), traces. LLM-specific: per-request token counts and cost, time-to-first-token, tokens per second, prompt/response sizes, provider error rates and retries, cache hit rates, GPU utilisation and memory, queue depth. Use OpenTelemetry-based tracing (LLM observability tools build on it), with careful redaction of prompts and personal data in logs. Add evaluation and guardrail metrics for output quality.

**Q87. What are the security risks specific to AI containers?**

Large untrusted dependency trees (many Python packages), **pickle**-based model files that can execute code when loaded (prefer safetensors, load only trusted models), downloading models at runtime from public hubs (verify sources and checksums), prompt injection leading tools to run commands or reach internal networks (sandbox tool execution in separate, unprivileged containers with no secrets and restricted network), leaking data through logs, GPU-sharing isolation limits, and privileged GPU containers. Apply the standard hardening plus network egress controls and model provenance checks.

**Q88. When would you not use Docker for an AI workload?**

Highly specialised bare-metal performance tuning (some HPC clusters use Apptainer/Singularity because Docker needs a root daemon), very small prototypes where a virtual environment is enough, notebook-only exploratory work (though a container can still help share it), or managed platforms that abstract containers away (hosted fine-tuning/inference APIs). Even then, understanding containers helps because those platforms use them internally.

---

## 2. How to answer well in interviews

1. **Structure:** definition (one sentence), how it works (two or three sentences), why it matters or when you would use it, and a concrete example from something you actually built.
2. **Use the STAR format for scenario questions:** *Situation, Task, Action, Result.* Prepare three stories from this project: a bug you debugged (use Chapter 12 as your source), an optimisation you made (image size or build time, with before and after numbers you actually measured), and a design decision with trade-offs (why Compose rather than Kubernetes).
3. **Be honest about limits.** "I have run this on a single VM with Compose; I have studied Kubernetes concepts and built a manifest, but I have not operated a production cluster" is far stronger than bluffing.
4. **Know the trade-offs.** Senior interviewers want "it depends, and here is what it depends on."
5. **Ask clarifying questions** in design scenarios (traffic, team size, budget, compliance).
6. **Say the command and what it does**, not just the command.
7. **Show security and operations awareness:** non-root, secrets, scanning, health checks, rollback, backups. Mentioning them unprompted signals production experience.

---

## 3. Staged learning roadmap

Assumption: about 8-10 hours per week. Each week has a goal, what to read, what to build, and a **checkpoint** (a test you must pass before moving on). If you have more time, compress it; if less, stretch it. Do the hands-on parts, since reading alone will not stick.

| Week | Theme | Read | Build | Checkpoint |
|---|---|---|---|---|
| **1** | Foundations and installation | [Ch 1](01-what-is-docker.md), [Ch 2](02-installation-and-operating-systems.md), [Ch 3](03-core-cli-commands.md) | Install Docker, run `hello-world`, nginx, Postgres, a Python container; use every command in Chapter 3 at least once | Explain container vs VM vs image in your own words; run a container, view logs, exec into it, stop and remove it, from memory |
| **2** | Dockerfiles | [Ch 4](04-dockerfile-deep-dive.md) | Containerise a small app of your own; write a multi-stage Dockerfile; add `.dockerignore`; reduce the image size by at least 50% and note the before/after | Your image builds in under 10 seconds on a rebuild after a code change (cache works); runs as non-root; you can explain every line |
| **3** | Storage and networking | [Ch 5](05-storage-and-networking.md) | Run Postgres with a named volume, destroy and recreate the container, prove the data survived; build a two-container app on a user-defined network using service names | You can explain volume vs bind mount, why `localhost` fails between containers, and what `-p` does |
| **4** | Compose and the sample app | [Ch 6](06-docker-compose.md), [Ch 9](09-hands-on-ai-app-walkthrough.md) | Run the FastAPI + Redis + Postgres + nginx sample app; add healthchecks and `depends_on` conditions; try the Ollama profile; change something and redeploy | Bring the whole stack up and down from scratch; explain every service and its network; recover from a deliberately broken config using logs |
| **5** | CI/CD and registries | [Ch 11 sections 3-6](11-production-and-enterprise.md) | Push the sample app to GHCR from GitHub Actions: build with Buildx and cache, tag with SHA and version, scan, push multi-arch | A merge to `main` produces a scanned, signed-off image with traceable tags; you can roll back by redeploying a previous tag |
| **6** | Production on a real server, security | [Ch 10](10-security.md), [Ch 11 sections 7-18](11-production-and-enterprise.md) | Rent a small Linux VM; install Docker; deploy the app with Compose behind Caddy with HTTPS; add log rotation, firewall, backups and an uptime monitor; harden the containers | Your app is reachable on HTTPS; reboot the VM and everything returns on its own; you restored a backup successfully |
| **7** | Docker for AI/ML and troubleshooting | [Ch 8](08-docker-for-ai-and-ml.md), [Ch 12](12-troubleshooting-and-cheatsheet.md) | Containerise a model-serving API; try a local LLM via Ollama; if you have a GPU, run it with `--gpus`; deliberately cause five errors from Chapter 12 and fix them | You can diagnose OOM, port, DNS, permission and architecture errors without searching |
| **8** | Capstone, portfolio and interviews | [Ch 13](13-interview-questions-roadmap-glossary.md), [Ch 14](14-real-world-case-study.md) | Polish the repository (Section 5 below); write the README; record a short demo; answer 30 interview questions out loud | A friend can clone the repo and get the app running in under five minutes by reading only your README; you can talk for 5 minutes about the project without notes |

### Weeks 9-16 and beyond (suggested)

| Phase | Focus | Project |
|---|---|---|
| **Weeks 9-12** | Kubernetes fundamentals: pods, deployments, services, ingress/Gateway, config/secrets, probes, HPA, Helm. Use `kind`, `minikube` or `k3d` locally. | Port your Compose app to Kubernetes manifests, then a Helm chart; deploy to a local cluster; do a rolling update and a rollback |
| **Weeks 13-16** | Infrastructure as code and cloud: Terraform, one cloud (AWS/Azure/GCP), managed Kubernetes or serverless containers, managed databases, IAM basics | Provision the VM/cluster and registry with Terraform; deploy through GitOps (Argo CD or Flux) |
| **Months 5-6** | Observability and reliability: Prometheus, Grafana, Loki, OpenTelemetry, SLOs, alerting, incident practice | Add dashboards, alerts and tracing to the project; run a game day |
| **Ongoing (AI focus)** | MLOps: experiment tracking (MLflow, Weights & Biases), model registries, feature stores, pipelines (Kubeflow, Airflow, Prefect, Dagster), model serving (vLLM, KServe, BentoML, Triton), evaluation, GPU scheduling | Serve an open model with vLLM in Kubernetes with autoscaling and monitoring |

### Self-assessment: can you do these without help?

- [ ] Write a production-quality Dockerfile for a Python or Node app from a blank file
- [ ] Explain image layers and caching, and speed up a slow build
- [ ] Set up a multi-service Compose stack with volumes, networks, health checks and secrets
- [ ] Debug a crashing container using logs, exit codes, `inspect` and `exec`
- [ ] Build and push a multi-arch image from CI with sensible tags
- [ ] Deploy to a Linux VM with HTTPS and rollback
- [ ] Describe how you would secure a container image and its supply chain
- [ ] Explain how a Compose file maps to Kubernetes objects
- [ ] Run and serve a model in a container, with or without a GPU

---

## 4. What to learn after Docker

Docker is the foundation, not the destination. Here is the usual order, with the reason for each.

| Next skill | Why | First step |
|---|---|---|
| **Linux fundamentals** (processes, permissions, systemd, networking, `journalctl`, `ss`, `iptables`) | Containers are Linux; every production problem eventually lands here | Run a small VM and administer it for a month |
| **Git and CI/CD** (GitHub Actions, GitLab CI, branching strategies, pull requests) | The delivery pipeline is how containers reach production | Automate build, test, scan, push for a project |
| **Kubernetes** | The standard platform for running containers at scale; the most requested skill next to Docker | Official Kubernetes tutorials, then `kind` and your own manifests; consider the CKAD/CKA certifications |
| **A cloud provider** (AWS, Azure or GCP) | Registries, managed Kubernetes, serverless containers, managed databases, IAM | Deploy a container to Cloud Run, ECS Fargate or Azure Container Apps |
| **Terraform / infrastructure as code** | Reproducible infrastructure, reviewable in git | Provision your VM, network and registry with code |
| **Observability** (Prometheus, Grafana, Loki, OpenTelemetry) | You cannot run what you cannot see | Add dashboards and alerts to your project |
| **Security** (supply chain, secrets, policy as code, CIS benchmarks) | Increasingly a hiring differentiator | Add scanning, signing and Conftest policies to your pipeline |
| **GitOps** (Argo CD, Flux) | Modern deployment model for Kubernetes | Deploy your app to a local cluster via Argo CD |
| **MLOps and AI infrastructure** (MLflow, model registries, vLLM/Triton/KServe, GPU scheduling, vector databases, LLM evaluation and observability) | Your target role: turning models into reliable products | Serve a model, version it, monitor it |
| **Alternative tools** (Podman, Buildah, nerdctl, containerd, Dev Containers, Buildpacks) | Different organisations choose different tools; the concepts transfer | Run your Compose app with Podman |

Certifications are optional; real projects matter more. If you want one that exists and is respected: the Linux Foundation's **CKAD** and **CKA** (Kubernetes; listed exam-only price was $445 at the time of writing), a cloud provider associate-level certification (AWS Solutions Architect Associate, Azure Administrator, Google Associate Cloud Engineer), and HashiCorp's **Terraform Associate**. The **Docker Certified Associate (DCA)** has been handled by Mirantis rather than Docker, and sources conflict on whether it is still actively offered, so verify availability before planning around it.

---

## 5. Portfolio guide

This repository is a career asset. A hiring manager spends about two minutes on it. Make those two minutes count.

### 5.1 What this repository should demonstrate

1. **You understand Docker deeply**, not just commands (the chapters show it).
2. **You can build something real** (the sample AI app).
3. **You think about production** (CI/CD, security, logging, backups).
4. **You communicate clearly**, which is rare and valuable.

### 5.2 Recommended repository structure

```
docker-complete-guide/
├── README.md                    <- the most important file (see 5.3)
├── LICENSE                      <- e.g. MIT (for docs) so people can reuse it
├── .gitignore                   <- includes .env, *.pem, *.key, data/, models/
├── docs/
│   ├── 01-what-is-docker.md
│   ├── ...
│   └── 14-real-world-case-study.md
├── sample-ai-app/
│   ├── README.md                <- how to run THIS app in 3 commands
│   ├── compose.yaml
│   ├── compose.override.yaml    <- developer conveniences
│   ├── .env.example             <- fake placeholder values ONLY
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
│       └── ci.yaml              <- builds, lints, scans: shows the green badge
├── assets/
│   └── architecture.png         <- diagram, screenshots, demo GIF
└── CONTRIBUTING.md  (optional)
```

### 5.3 The README that gets read

Structure (copy this order):

1. **Title and one-line pitch:** "A complete beginner-to-enterprise Docker guide, with a production-style AI chat app (FastAPI, Redis, Postgres, nginx, optional Ollama)."
2. **Badges** (build status, license). They signal a live pipeline.
3. **A screenshot or short demo GIF** of the app running, and an **architecture diagram**.
4. **Quick start:** three commands, copy-pastable:
   ```bash
   git clone https://github.com/<you>/docker-complete-guide.git
   cd docker-complete-guide/sample-ai-app
   cp .env.example .env && docker compose up -d
   ```
   Say what URL to open and what they should see. Test this on a clean machine.
5. **Table of contents** linking to the 14 chapters, each with a one-sentence summary.
6. **What you will learn / who this is for.**
7. **Key design decisions** (why Compose, why multi-stage, why non-root): this is where you show judgement.
8. **Tech stack** and **prerequisites** (Docker version tested).
9. **Roadmap/what's next** (for example "Kubernetes manifests, Terraform for the VM").
10. **License and author contact** (LinkedIn).

README tips: lead with outcomes not features; keep the quick start short; use relative links; keep it honest (do not claim "used in production" if it is not); keep images small; make the first screen impressive.

### 5.4 What to show and highlight

- **The CI pipeline running green**, with scanning and multi-arch builds.
- **Measured improvements:** "Image size 1.2 GB to 180 MB via multi-stage build and slim base; rebuild time from 4 min to 12 s with layer ordering and cache mounts." Use real numbers from your own experiments; never invent metrics.
- **Security work:** non-root user, pinned versions, scanning results, secrets handled properly.
- **Operations work:** health checks, log rotation, backups, rollback procedure.
- **A short "lessons learned / mistakes I made" page**: it shows growth and honesty.
- **Architecture decision records:** a few short notes on trade-offs.

### 5.5 What NOT to commit (critical)

Treat these as an automatic rejection by a careful reviewer and a genuine security risk:

| Never commit | Why / what to do instead |
|---|---|
| `.env` files with real values | Commit `.env.example` with fake placeholders (`OPENAI_API_KEY=your-key-here`) and add `.env` to `.gitignore` |
| API keys, tokens, passwords, private keys (`*.pem`, `*.key`, SSH keys), cloud credentials, Docker Hub/GHCR tokens | Use secrets managers and CI secrets; rotate anything that ever leaked |
| `~/.docker/config.json` contents, `kubeconfig` files, Terraform state (`*.tfstate`) and `*.tfvars` with secrets | They contain credentials; use remote state and encrypted variables |
| Database dumps or files with real personal data | Use synthetic sample data |
| Large model weights and datasets | Put them in Git LFS, object storage, or Hugging Face; document how to download |
| Build output and caches (`node_modules`, `__pycache__`, `.venv`, `dist/`) | `.gitignore` them |
| Anything confidential from an employer or client (internal hostnames, project names, proprietary code, screenshots showing internal systems) | Build your demos from scratch with public data and your own work only |
| Personal data in screenshots (emails, IP addresses, tokens in terminal output) | Review every screenshot before publishing |

**Safety net:** before the first public push, run a secret scanner over the *entire history*, not only the current files:

```bash
gitleaks detect --source . --log-opts="--all"
```

- `gitleaks detect` scans for patterns that look like secrets; `--source .` scans the current repository; `--log-opts="--all"` includes every branch and commit in history. If it reports anything, **rotate the credential first**, then rewrite history or start a fresh repository. (`trufflehog` is an alternative.) Also enable **GitHub secret scanning and push protection** in repository settings, and consider a local pre-commit hook.

If you ever do commit a secret: assume it is stolen, revoke it immediately, then clean up. Deleting it in a later commit does not remove it from history.

### 5.6 How to present it on LinkedIn and your resume

**LinkedIn "Featured" section:** pin the repository link. Add a post with a short story, not just a link. Template:

> I spent [N weeks] turning "I know what Docker is" into "I can ship containers to production." I wrote a complete Docker guide (14 chapters, from your first `docker run` to CI/CD, security and running AI workloads) and built a containerised AI chat app (FastAPI, Redis, Postgres, nginx, optional local LLM) to prove it. Biggest lessons: [two or three real lessons, such as "layer ordering cut my rebuild time from X to Y" or "an unrotated log file taught me why daemon.json matters"]. Repo: [link]. Feedback welcome.

**Resume entry (use your real numbers; delete any bullet you cannot back up):**

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

Skills line: Docker, Docker Compose, Buildx/BuildKit, GitHub Actions, GHCR, Linux, Nginx/Caddy, PostgreSQL, Redis, FastAPI, Trivy, Prometheus/Grafana (only list tools you truly used).

### 5.7 How to talk about it in interviews

- **The 30-second pitch:** "I built a containerised AI chat app and wrote a full Docker guide around it. The app has an API, a database, a cache and a reverse proxy, runs from one Compose command, and has a CI pipeline that builds, scans and publishes the image. I used it to learn how production container workflows work."
- **Be ready to go deep on any file in the repo.** If you cannot explain a line, remove it or learn it.
- **Prepare a "what I would change" answer:** "In production I would move Postgres to a managed service, add centralised logging and tracing, and use Kubernetes only once the team and traffic justified it." This shows judgement.
- **Prepare your three STAR stories** (Section 2).
- **Do not overclaim.** Describe it as a learning and portfolio project. Honest scope plus real depth beats inflated claims.
- **Be ready for AI-angle questions**: GPU containers, model storage, serving, security of LLM apps (Q74-Q88).

### 5.8 Keep it alive

A repository with recent activity looks maintained. Update dependency and action versions every few months (Dependabot/Renovate), re-run the CI, fix broken links, and refresh the facts that age (rate limits, tool versions). Add a "last verified" date at the top of chapters. Respond to issues politely.

---

## 6. Recommended further resources

I checked the existence and current status of the following in October 2026. URLs are given where they are stable official addresses; for books and courses search the exact title. Editions change, so look for the newest one.

### Official documentation (free, authoritative)

- **Docker Docs**, <https://docs.docker.com/>: Get Started guides, Dockerfile reference, Compose reference, Engine, Build (BuildKit), Docker Hub, Docker Desktop troubleshooting.
- **Dockerfile best practices** in the Docker Build documentation.
- **Compose Specification**, <https://compose-spec.io/>: the open specification behind Compose files.
- **Open Container Initiative**, <https://opencontainers.org/>: image and runtime specifications.
- **Kubernetes Documentation**, <https://kubernetes.io/docs/>: concepts and interactive tutorials.
- **The Twelve-Factor App**, <https://12factor.net/>: short and still relevant.
- **CNCF Landscape**, <https://landscape.cncf.io/>: map of the cloud-native ecosystem.
- **Cloud provider docs:** AWS (ECS, ECR, EKS), Azure (ACR, AKS, Container Apps), Google Cloud (Artifact Registry, GKE, Cloud Run).
- **Trivy documentation** (aquasecurity.github.io/trivy), **Sigstore/Cosign** (docs.sigstore.dev), **OpenTelemetry** (opentelemetry.io), **Prometheus** (prometheus.io), **Grafana** (grafana.com/docs).
- **Google's Site Reliability Engineering books**, free at <https://sre.google/books/>: for the operations mindset behind Chapter 11.

### Books

- **Docker Deep Dive** by Nigel Poulton: updated yearly; the 2025 edition (published on Leanpub, last updated May 2025) added a Docker Model Runner chapter plus updates on BuildKit, Buildx and Docker Debug. A very good next read after this guide. (A 2026 edition was not found at the time of writing; check the publisher.)
- **The Kubernetes Book** by Nigel Poulton: gentle, regularly updated.
- **Kubernetes Up & Running** (O'Reilly) by Brendan Burns, Joe Beda, Kelsey Hightower and Lachlan Evenson.
- **Kubernetes in Action** (Manning) by Marko Lukša: deep and thorough; check the current edition.
- **Docker in Practice** (Manning) by Ian Miell and Aidan Hobson Sayers: problem-and-solution style.
- **Docker: Up & Running** (O'Reilly) by Sean P. Kane and Karl Matthias.
- **Learn Docker in a Month of Lunches** (Manning) by Elton Stoneman: very beginner-friendly.
- **Container Security** (O'Reilly) by Liz Rice: how isolation really works in the kernel; excellent for interviews.
- **Designing Machine Learning Systems** (O'Reilly) by Chip Huyen: for the MLOps side.

### Courses and labs

- **Docker's official "Get started" guides** (free, at docs.docker.com).
- **TechWorld with Nana** on YouTube: full free Docker and Kubernetes tutorials.
- **KodeKloud**: hands-on Docker and Kubernetes courses with browser-based labs (paid, with some free content).
- **Killercoda** (killercoda.com): free interactive Kubernetes and Docker scenarios in the browser.
- **iximiuz Labs** (labs.iximiuz.com): hands-on container and Kubernetes labs; named by Docker's community as an alternative to Play with Docker.
- **Linux Foundation / CNCF training**: courses and the CKA/CKAD/CKS certification exams.
- **Note:** **Play with Docker** (the free browser sandbox) was announced as deprecated and unavailable from 1 March 2026, so do not rely on tutorials that depend on it. Use Docker Desktop locally, a cheap cloud VM, or the alternatives above.

### Community

- Docker Community Forums (forums.docker.com), r/docker and r/kubernetes on Reddit, the CNCF and Kubernetes Slack workspaces, and Stack Overflow for specific errors. The awesome-docker list on GitHub is a curated link collection (check it is still maintained when you visit).

### Tools worth knowing by name

`dive` (image layer explorer), `hadolint` (Dockerfile linter), `Trivy`, `Grype` and `Syft` (scanning and SBOM), `Cosign` (signing), `lazydocker` and `ctop` (terminal dashboards), `Portainer` (web UI), `Dockge` (Compose manager), `Renovate` and `Dependabot` (dependency updates), `k9s` (Kubernetes terminal UI), `kind`/`k3d`/`minikube` (local Kubernetes), `Helm`, `Argo CD`, `Flux`.

---

## 7. Glossary (A-Z)

Plain-English definitions. Terms appear alphabetically; related terms are cross-referenced.

### A

- **ADD:** a Dockerfile instruction like `COPY` that can also unpack tar archives and download URLs. Prefer `COPY`.
- **Alpine:** a tiny Linux distribution (a few MB) popular as a container base image. Uses musl libc, which can cause compatibility surprises with some Python packages.
- **AppArmor:** a Linux security module that restricts what a program may do (files, network, capabilities); Docker applies a default profile on systems that support it.
- **ARG:** a Dockerfile variable available only while building the image.
- **Artifact registry:** a server that stores build outputs (container images, packages). Google's product is named Artifact Registry.
- **Attestation:** a signed statement about an image, such as where it was built (provenance) or what it contains (SBOM).
- **Autoscaling:** automatically adding or removing instances based on load.

### B

- **Base image:** the image named in `FROM` that your image builds on (for example `python:3.12-slim`).
- **Bind mount:** mapping a specific host file or folder into a container.
- **Blue/green deployment:** running old (blue) and new (green) versions side by side and switching traffic in one step, keeping blue for instant rollback.
- **Bridge network:** Docker's default virtual network on a single host; containers on it get private IPs and NAT to the outside.
- **BuildKit:** Docker's modern image build engine, with parallelism, cache mounts, secrets and attestations.
- **Build cache:** stored results of earlier build steps, reused when inputs are unchanged.
- **Build context:** the set of files (usually a folder) sent to the builder when you run `docker build`.
- **Buildx:** the Docker CLI plug-in that exposes BuildKit's full features, including multi-platform builds.

### C

- **cAdvisor:** a Google tool (run as a container) that reports per-container resource usage as metrics.
- **Canary release:** sending a small share of traffic to a new version first and increasing it if healthy.
- **Capabilities (Linux):** the pieces root's power is divided into (bind low ports, change network settings, and so on); containers get a reduced set.
- **cgroups (control groups):** Linux kernel feature that limits and accounts for CPU, memory, I/O and process counts of groups of processes. Docker uses it for resource limits.
- **CI/CD:** Continuous Integration (automatically build and test every change) and Continuous Delivery/Deployment (automatically release it).
- **CMD:** Dockerfile instruction giving the default command or arguments for a container.
- **Compose (Docker Compose):** tool and YAML format to define and run multi-container applications.
- **Compose Specification:** the open standard that defines the Compose file format.
- **Container:** an isolated process (or group) running from an image, sharing the host kernel.
- **containerd:** a container runtime daemon that manages images and container lifecycle; used by Docker and Kubernetes.
- **Container runtime:** software that actually creates and runs containers (runc, containerd, CRI-O).
- **Control plane (Kubernetes):** the cluster's "brain" components (API server, scheduler, controllers, etcd).
- **Copy-on-write (CoW):** a technique where data is shared until modified, and only then copied; how container filesystems use image layers efficiently.
- **Cosign:** a Sigstore tool to sign and verify container images.
- **CUDA:** NVIDIA's platform and libraries for running general computation on GPUs; required by most GPU machine learning.
- **CVE (Common Vulnerabilities and Exposures):** a public identifier for a known security flaw, such as CVE-2024-12345.

### D

- **Daemon (`dockerd`):** the Docker background service that builds images and runs containers; the `docker` command is its client.
- **Dangling image:** an image with no tag (shown as `<none>`), usually a leftover of rebuilds.
- **Declarative:** describing the desired end state and letting the system work out how to reach it (Compose, Kubernetes).
- **Deployment (Kubernetes):** an object managing a set of identical pods, rollouts and rollbacks.
- **Digest:** the SHA-256 hash that uniquely and permanently identifies image content (`sha256:...`).
- **Distroless:** minimal images containing only the app and runtime dependencies, with no shell or package manager.
- **DNS (embedded):** Docker's built-in name resolver (127.0.0.11) that lets containers find each other by name on user-defined networks.
- **Docker Desktop:** the desktop application for Windows, macOS and Linux that bundles Docker Engine, a Linux VM where needed, a GUI, Kubernetes and extras.
- **Docker Engine:** the open-source container engine (daemon, API, CLI) that you install on Linux servers.
- **Docker Hub:** Docker's public registry and image catalogue.
- **Docker Scout:** Docker's image analysis service for vulnerabilities and base-image recommendations.
- **Docker socket (`/var/run/docker.sock`):** the file through which clients talk to the daemon; access equals root on the host.
- **Dockerfile:** the text recipe for building an image.
- **.dockerignore:** a file listing paths excluded from the build context.

### E

- **Entrypoint (`ENTRYPOINT`):** the fixed executable a container runs at start; `CMD` supplies default arguments.
- **ENV:** Dockerfile instruction setting an environment variable in the image and container.
- **Ephemeral:** short-lived; containers are ephemeral, so data you care about must be stored elsewhere.
- **Exit code:** the number a process returns when it ends; 0 is success, above 128 means killed by a signal.
- **EXPOSE:** Dockerfile instruction documenting the port an app listens on; does not publish it.
- **Egress:** outbound network traffic (often billed in cloud environments).

### F

- **Fargate (AWS):** AWS's serverless compute for containers; you do not manage servers.
- **Fluent Bit / Fluentd:** log collectors and forwarders.
- **Flatcar Container Linux:** a minimal, auto-updating Linux distribution for running containers.

### G

- **GHCR (GitHub Container Registry):** GitHub's registry at `ghcr.io`.
- **GitOps:** managing infrastructure and deployments by storing desired state in git and letting an agent apply it.
- **Golden image:** an approved, hardened, regularly rebuilt base image that teams must use.
- **GPU (Graphics Processing Unit):** a processor with thousands of cores suited to the parallel maths of machine learning.
- **Grafana:** a dashboard and visualisation tool for metrics and logs.

### H

- **Harbor:** an open-source enterprise container registry (CNCF graduated project).
- **Health check (`HEALTHCHECK`):** a command Docker runs periodically to determine whether the container is working.
- **Helm:** the package manager for Kubernetes; packages are called charts.
- **Horizontal scaling:** adding more copies of a service rather than a bigger machine.
- **Host network:** a mode where the container shares the host's network stack.
- **HPA (Horizontal Pod Autoscaler):** the Kubernetes component that adjusts replica counts based on metrics.
- **Hypervisor:** software that creates and runs virtual machines.

### I

- **Image:** a read-only template (layers plus metadata) used to create containers.
- **Image index / manifest list:** a pointer from one tag to several platform-specific images, enabling multi-architecture tags.
- **Immutable:** cannot be changed after creation; immutable tags and infrastructure improve reproducibility.
- **Inference:** using a trained model to produce outputs (predictions, text) as opposed to training it.
- **Infrastructure as code (IaC):** defining servers and networks in code files (Terraform, CloudFormation) so they are repeatable and reviewable.
- **Ingress:** Kubernetes rules (or the controller implementing them) routing external HTTP(S) traffic to services; being superseded by the Gateway API.
- **Init process:** a tiny PID 1 program (like tini) that forwards signals and reaps zombies; enabled with `--init`.

### J-K

- **JSON-file (logging driver):** the default driver that writes logs as JSON files on the host; does not rotate by default.
- **Kaniko:** a tool that builds images inside a container without a Docker daemon (check its current maintenance status).
- **Kubernetes (K8s):** the leading container orchestration platform.
- **kubectl:** the Kubernetes command-line client.
- **kubelet:** the agent on each Kubernetes node that starts and monitors pods.
- **KEDA:** Kubernetes Event-Driven Autoscaling; scales workloads on events such as queue length.
- **KV cache:** in LLM serving, the stored attention keys/values of previous tokens, which consume GPU memory per request.

### L

- **Label:** key-value metadata attached to images, containers or Kubernetes objects.
- **Layer:** one read-only filesystem change set in an image, created by a Dockerfile instruction.
- **Liveness probe:** a check that decides whether to restart an unresponsive container.
- **Live-restore:** a daemon option that keeps containers running while Docker restarts.
- **Load balancer:** a component distributing requests across multiple instances.
- **LLM (Large Language Model):** a neural network trained on vast text corpora to generate and understand text.
- **Loki:** Grafana's log aggregation system.
- **Logging driver:** the plug-in determining where container logs are sent.

### M

- **Macvlan:** a network mode giving containers their own MAC and IP addresses on the physical LAN.
- **Manifest:** JSON document describing an image (its config and layers); in Kubernetes, also a YAML file describing objects.
- **Metrics:** numeric measurements over time (CPU, request rate, latency).
- **MLOps:** practices and tools for deploying, monitoring and maintaining machine learning systems in production.
- **Model weights:** the learned numerical parameters of a model; often gigabytes in size.
- **Multi-arch image:** an image tag that offers variants for several CPU architectures.
- **Multi-stage build:** a Dockerfile with multiple `FROM` stages so build tools stay out of the final image.

### N

- **Namespace (Linux):** a kernel feature giving a process its own view of a system resource (PIDs, network, mounts); the basis of container isolation.
- **Namespace (Kubernetes):** a virtual partition of a cluster used to separate teams, environments and quotas.
- **Network policy:** Kubernetes rules defining which pods may talk to which.
- **Node:** a machine (physical or virtual) in a cluster.
- **NVIDIA Container Toolkit:** software that lets containers use NVIDIA GPUs by injecting host driver libraries.

### O

- **OCI (Open Container Initiative):** the standards body defining the image and runtime formats.
- **Ollama:** a tool for running open LLMs locally with a simple API; commonly run as a container.
- **OOM killer:** the Linux kernel component that kills processes when memory is exhausted, giving exit code 137.
- **OPA (Open Policy Agent):** a general-purpose policy engine (policies written in Rego); **Conftest** applies it to config files.
- **OpenTelemetry (OTel):** an open standard and toolkit for traces, metrics and logs.
- **Orchestrator:** software that runs and manages containers across many machines (Kubernetes, Swarm, ECS).
- **Overlay network:** a virtual network spanning multiple hosts.
- **overlay2:** the default Docker storage driver on modern Linux, built on OverlayFS.

### P

- **Pod:** Kubernetes' smallest deployable unit: one or more containers sharing network and storage.
- **Podman:** a daemonless, rootless-friendly container engine with a Docker-compatible CLI.
- **Port publishing (`-p`):** mapping a host port to a container port.
- **PID 1:** the first process in a container, with special responsibilities for signals and zombies.
- **Privileged container:** a container with nearly all of the host's capabilities and device access; avoid.
- **Profile (Compose):** a label that makes a service start only when requested.
- **Prometheus:** an open-source monitoring system that scrapes and stores metrics.
- **Provenance:** a record of how and where an artifact was built.
- **Pull-through cache:** a registry that fetches from an upstream registry on first request and serves copies afterwards.
- **PVC (PersistentVolumeClaim):** a Kubernetes request for storage.

### Q-R

- **QEMU:** an emulator that lets one CPU architecture run another's programs; used for cross-platform builds.
- **Quantisation:** reducing the numeric precision of model weights (for example 16-bit to 4-bit) to cut memory and speed up inference at some accuracy cost.
- **RAG (Retrieval-Augmented Generation):** an LLM technique that fetches relevant documents and includes them in the prompt.
- **RBAC (Role-Based Access Control):** permissions granted by roles.
- **Readiness probe:** a check deciding whether an instance should receive traffic.
- **Registry:** a server storing and distributing images.
- **Replica:** one of several identical copies of a service.
- **Repository:** a named collection of image versions (tags) in a registry.
- **Restart policy:** rule telling Docker when to restart a container.
- **Reverse proxy:** a server that receives client requests and forwards them to backend services (nginx, Caddy, Traefik).
- **Rolling update:** replacing instances gradually to avoid downtime.
- **Rootless mode:** running Docker and containers without root privileges on the host.
- **RPO / RTO:** Recovery Point Objective (how much data you can lose) and Recovery Time Objective (how long you can be down).
- **runc:** the low-level runtime that creates containers according to the OCI spec.

### S

- **SBOM (Software Bill of Materials):** a list of all components inside an image.
- **Seccomp:** a Linux feature filtering which system calls a process may make; Docker applies a default profile.
- **Secret:** a sensitive value (password, key, token) that must be protected.
- **SELinux:** a Linux mandatory access control system common on Red Hat-family distributions.
- **Semantic versioning (semver):** `MAJOR.MINOR.PATCH` version numbering with defined meaning.
- **Service (Compose):** one container definition (image, ports, volumes) in a Compose file.
- **Service (Kubernetes):** a stable network endpoint in front of a set of pods.
- **Sidecar:** a helper container deployed alongside the main one (logging, proxy, debugging).
- **SIGKILL / SIGTERM:** signals: SIGTERM asks a process to stop gracefully; SIGKILL forces it to stop immediately.
- **Sigstore:** an open-source project for signing and verifying software, including keyless signing.
- **SLO / SLI / SLA:** Service Level Objective (target), Indicator (measurement) and Agreement (contract) for reliability.
- **Stateful / stateless:** whether a service keeps data between requests (stateful) or not (stateless).
- **Sticky sessions:** load balancing that keeps sending a user to the same instance.
- **Swarm (Docker Swarm):** Docker's built-in clustering and orchestration mode.

### T

- **Tag:** a mutable label on an image (`1.4.2`, `latest`).
- **Terraform:** a popular infrastructure-as-code tool.
- **tmpfs:** a memory-backed temporary filesystem; contents vanish when the container stops.
- **TLS:** Transport Layer Security, the encryption behind HTTPS.
- **Tracing:** following one request through multiple services as timed spans.
- **Traefik:** a reverse proxy that configures itself from container labels.
- **Trivy:** an open-source vulnerability and misconfiguration scanner.

### U-V

- **Union filesystem:** a filesystem that presents several layered directories as one.
- **User namespace:** maps container user IDs to different host IDs, so container root is not host root.
- **Vector database:** a database optimised to store embeddings and find similar items (pgvector, Qdrant, Chroma, Weaviate).
- **Embedding:** a list of numbers representing the meaning of text or other data, used for similarity search.
- **Vertical scaling:** giving a service a bigger machine.
- **VM (virtual machine):** an emulated computer with its own operating system.
- **Volume:** Docker-managed persistent storage that outlives containers.
- **vLLM:** a high-throughput open-source LLM inference server.
- **Vulnerability scanning:** comparing image contents against databases of known flaws.

### W-Z

- **Watchtower:** a tool that auto-updated running containers when new images appeared; the original project was archived in December 2025.
- **Workload:** an application or job running on a platform.
- **WSL 2 (Windows Subsystem for Linux 2):** a lightweight Linux VM inside Windows, used by Docker Desktop.
- **YAML:** the indentation-based data format used by Compose and Kubernetes (spaces only, never tabs).
- **Zero-downtime deployment:** releasing a new version without any period when the service is unavailable.
- **Zombie process:** a finished process whose exit status has not been collected by its parent; a PID 1 that does not reap children lets them accumulate.

### Additional terms you will meet

- **12-factor app:** a methodology for building portable, scalable cloud apps (config in environment, stateless processes, logs to stdout).
- **Admission controller:** a Kubernetes component that accepts or rejects objects at creation (used for policy and signature checks).
- **API server:** the front door of the Kubernetes control plane.
- **Buildpacks:** tools that turn source code into container images without a Dockerfile (Cloud Native Buildpacks).
- **Chart:** a Helm package of Kubernetes templates.
- **CRI (Container Runtime Interface):** the Kubernetes interface for container runtimes.
- **CRI-O:** a lightweight container runtime built specifically for Kubernetes.
- **DaemonSet:** a Kubernetes object that runs one pod on every node (log collectors, monitoring agents).
- **Dev Container:** a standard for defining a containerised development environment (used by VS Code and GitHub Codespaces).
- **etcd:** the key-value database holding Kubernetes' state.
- **Gateway API:** the newer Kubernetes API for traffic routing, successor to Ingress.
- **Idempotent:** safe to run repeatedly with the same result.
- **Job / CronJob:** Kubernetes objects for run-to-completion and scheduled tasks.
- **Kustomize:** a Kubernetes-native way to customise manifests with overlays.
- **Landing zone:** a pre-configured, governed cloud account structure for organisations.
- **Service mesh:** infrastructure (Istio, Linkerd) providing encrypted, observable service-to-service traffic.
- **StatefulSet:** a Kubernetes object for stateful apps needing stable identity and storage.
- **Taint / toleration:** Kubernetes mechanism to keep pods off (or allow them onto) certain nodes, commonly used for GPU nodes.
- **Time-to-first-token (TTFT):** how long an LLM service takes to emit the first output token; a key latency metric.

---

You have reached the end of the practical guide. The best next step is not another chapter: it is to **build something, ship it, break it, and fix it**. Then write down what you learned. That written record, plus a working repository, is your portfolio.

Previous: [Chapter 12: Troubleshooting and Cheat Sheet](12-troubleshooting-and-cheatsheet.md) | Next: [Chapter 14: Real-World Case Study](14-real-world-case-study.md) | Start over: [Chapter 1](01-what-is-docker.md)
