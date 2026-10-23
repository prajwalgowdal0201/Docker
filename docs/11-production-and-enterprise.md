# Chapter 11: Production and Enterprise Docker

> **What you will learn:** how Docker is *really* used in companies: which operating system servers run, how code travels from your laptop to a running container in front of customers, how images are named and stored, how to build a complete CI/CD pipeline, where to run containers (one VM, Kubernetes, serverless, PaaS), how to deploy without downtime, how to handle secrets, logs, monitoring and backups, and a checklist you can use before going live.

This is the chapter that turns "I can run containers on my laptop" into "I can ship containers that other people depend on." Everything earlier in this guide ([Chapter 4](04-dockerfile-deep-dive.md) on Dockerfiles, [Chapter 5](05-storage-and-networking.md) on storage and networking, [Chapter 6](06-docker-compose.md) on Compose, [Chapter 10](10-security.md) on security) is a building block. Here we assemble the blocks.

A note on facts: this chapter was fact-checked in October 2026. Where a fact could not be confirmed from a primary source, it is marked **(unverified)**. Version numbers of tools change quickly. Always glance at the project's releases page before you copy a version into your own pipeline.

---

## Table of contents

1. [The mindset shift: laptop vs production](#1-the-mindset-shift-laptop-vs-production)
2. [Which operating system runs production Docker?](#2-which-operating-system-runs-production-docker)
3. [The end-to-end enterprise flow](#3-the-end-to-end-enterprise-flow)
4. [Image tagging strategy](#4-image-tagging-strategy)
5. [Container registries](#5-container-registries)
6. [A complete CI/CD pipeline with GitHub Actions](#6-a-complete-cicd-pipeline-with-github-actions)
7. [Where do you run the containers? Deployment targets](#7-where-do-you-run-the-containers-deployment-targets)
8. [Deployment strategies and rollbacks](#8-deployment-strategies-and-rollbacks)
9. [Configuration and secrets in production](#9-configuration-and-secrets-in-production)
10. [Health checks: liveness vs readiness](#10-health-checks-liveness-vs-readiness)
11. [Resource limits and the OOM killer](#11-resource-limits-and-the-oom-killer)
12. [Logging](#12-logging)
13. [Monitoring, alerting and tracing](#13-monitoring-alerting-and-tracing)
14. [Persistence and backups](#14-persistence-and-backups)
15. [Scaling](#15-scaling)
16. [Docker daemon production configuration](#16-docker-daemon-production-configuration-daemonjson)
17. [Disk management and cleanup policies](#17-disk-management-and-cleanup-policies)
18. [Upgrading Docker Engine](#18-upgrading-docker-engine)
19. [Cost and capacity planning](#19-cost-and-capacity-planning)
20. [Compliance and audit](#20-compliance-and-audit)
21. [Team practices](#21-team-practices)
22. [Production readiness checklist](#22-production-readiness-checklist)
23. [Incident runbook outline](#23-incident-runbook-outline)
24. [Reference architectures](#24-reference-architectures)
25. [Common mistakes](#25-common-mistakes)
26. [Summary](#26-summary)

---

## 1. The mindset shift: laptop vs production

On your laptop, a container is a convenience. If it crashes, you type the command again. In production, a container is a **service with promises**: it must start by itself, restart by itself, tell somebody when it is sick, not lose data, not leak secrets, and be replaceable at 3 a.m. by someone who has never seen it before.

Analogy: cooking dinner at home versus running a restaurant. The recipe (your Dockerfile) is the same. But a restaurant also needs a health inspection (security scans), supplier contracts (registries), a fire exit (rollbacks), a kitchen display (monitoring), and shifts so that the kitchen runs when the chef is asleep (automation).

Here is the difference in one table.

| Concern | On your laptop | In production |
|---|---|---|
| Starting | You type `docker run` | A machine boots and the platform starts it for you |
| Crash | You notice and restart | Auto-restart plus an alert |
| Image version | `latest`, whatever you built | A specific, immutable, traceable version |
| Secrets | A `.env` file | A secrets manager, injected at runtime |
| Logs | `docker logs` | Shipped to a central system, rotated, searchable |
| Data | A volume on your SSD | Backed-up, often outside the container platform entirely |
| Updates | Rebuild and rerun | Pipeline builds, scans, signs, deploys, and can roll back |
| Access | You are the only user | Many people, least privilege, audit trail |

The rest of this chapter is, in essence, a tour through every row of that table.

---

## 2. Which operating system runs production Docker?

### 2.1 The short answer: Linux

Containers are a **Linux technology**. A container is just a normal Linux process that has been given a private view of the system using two kernel features:

- **Namespaces** (isolated views of process IDs, network, filesystem mounts, hostnames, users).
- **cgroups** (control groups: limits on how much CPU, memory and disk I/O a process may use).

Because the kernel does the work, Linux containers run **natively** on a Linux server, with almost no overhead. On Windows and macOS, Docker has to run a hidden Linux virtual machine (WSL 2 or a lightweight VM) and your containers live inside that. That is fine for development and awkward for servers.

```
 Linux server (production)                  Windows / macOS laptop (development)
+---------------------------------+        +---------------------------------------+
|  container  container  container|        |  Windows or macOS                     |
|     |          |          |     |        |  +---------------------------------+  |
|  +--v----------v----------v--+  |        |  |  Hidden Linux VM (WSL 2 etc.)   |  |
|  |   Linux kernel (shared)   |  |        |  |  container  container          |  |
|  +---------------------------+  |        |  |  +--------------------------+   |  |
|        Physical or cloud VM     |        |  |  | Linux kernel inside VM   |   |  |
+---------------------------------+        |  |  +--------------------------+   |  |
                                           |  +---------------------------------+  |
                                           +---------------------------------------+
```

### 2.2 Why not Windows or macOS for production?

| OS | Verdict | Why |
|---|---|---|
| **macOS** | Never for production | Apple licensing restricts macOS in data centres, Docker Desktop on Mac is a development tool, and there are no macOS containers. Mac hardware is also not built to be rented by the hour. |
| **Windows Server (Linux containers)** | Not recommended | Linux containers need the same hidden Linux VM trick. You pay a licensing and performance cost for no gain. |
| **Windows Server (Windows containers)** | Niche, legitimate | Use it only if you must run legacy .NET Framework or other Windows-only software in containers. These containers share the Windows kernel, images are huge (multiple gigabytes), and the ecosystem is smaller. |
| **Docker Desktop on a server** | Do not | It is licensed for desktop use (paid for larger companies), has a GUI you do not need on a server, and adds a VM layer. Install **Docker Engine** instead. |

**Rule of thumb:** develop on whatever OS you like, but build and run production on **Linux**, using **Docker Engine** (the headless daemon, no GUI) or a container platform that hides the host from you.

### 2.3 Which Linux?

There are two families: **general-purpose distributions** (you install packages, patch them yourself, and Docker is just one program on the box) and **container-optimized operating systems** (a minimal, locked-down OS whose only job is to run containers).

| Linux | Type | Best for | Notes |
|---|---|---|---|
| **Ubuntu Server LTS** (for example 24.04 LTS) | General purpose | The most common choice for small teams and cloud VMs | "LTS" means Long Term Support: five years of free security updates, with a new LTS every two years. Docker publishes an official apt repository for it. Huge community and tutorials. |
| **Debian** (stable) | General purpose | Teams that want a very stable, small, neutral base | Ubuntu is built on Debian. Slightly slower release cadence, extremely reliable. |
| **RHEL** (Red Hat Enterprise Linux) | General purpose, commercial | Large enterprises that need vendor support contracts and certifications | Red Hat's own container engine is **Podman**. Docker Engine can be installed but is not the "blessed" path on RHEL. |
| **Rocky Linux / AlmaLinux** | General purpose | Free, RHEL-compatible rebuilds for people who want the RHEL ecosystem without the subscription | Same package family (`dnf`). Docker publishes a repo for CentOS-compatible systems. |
| **Amazon Linux** (2023) | General purpose, cloud | EC2 instances on AWS | Tuned and patched by AWS, and Docker is available from its package repository. Do not expect it to behave identically outside AWS. |
| **Container-Optimized OS (COS)** | Container-optimized | Google Cloud (GCE and GKE nodes) | Minimal, read-only root filesystem, automatic updates, Docker/containerd included. You cannot `apt install` things. |
| **Bottlerocket** | Container-optimized | AWS (EKS/ECS nodes) | Open source from AWS. Immutable root, atomic updates, API-driven configuration, no SSH by default. |
| **Flatcar Container Linux** | Container-optimized | Anywhere (bare metal, any cloud) | Community continuation of the original CoreOS Container Linux. Auto-updating, immutable `/usr`. |

**Which one should you pick?**

- Learning or small team with one VM: **Ubuntu Server LTS**. Boring is good.
- Running Kubernetes nodes in the cloud: let the cloud pick (the managed service chooses the node image), or choose Bottlerocket/COS for a smaller attack surface.
- Large company with a Red Hat contract: **RHEL** and probably Podman/OpenShift, with Docker used at the developer and build level.

**Common mistakes (OS choice)**

- Running production on a "cheap" distribution nobody on the team knows how to patch.
- Using an end-of-life OS release (for example, an old Ubuntu that stopped receiving security updates).
- Installing Docker from the distribution's very old built-in package instead of Docker's official repository. Always follow [Chapter 2](02-installation-and-operating-systems.md).
- Using the **kernel's cgroup v1**. Modern distributions default to cgroup v2. Docker Engine 29 deprecated cgroup v1 support (kept until at least 2029 according to the release notes), so prefer distributions that ship cgroup v2.

---

## 3. The end-to-end enterprise flow

Before the details, look at the whole picture. This is how nearly every company that uses containers ships software, from a start-up to a bank.

```
 +-----------------+      +-----------------+      +----------------------------+
 | 1. DEVELOPER    |      | 2. GIT HOST     |      | 3. CI (Continuous          |
 |    LAPTOP       | push | (GitHub/GitLab) |event |    Integration) RUNNER     |
 | code + Dockerfile+----->| pull request,   +----->|                            |
 | docker compose  |      | code review,    |      |  a. checkout code          |
 | tests locally   |      | branch protect  |      |  b. lint + unit tests      |
 +-----------------+      +-----------------+      |  c. docker build (cached)  |
                                                   |  d. SCAN image (Trivy/Scout)|
                                                   |  e. integration tests      |
                                                   |  f. SBOM + SIGN image      |
                                                   +-------------+--------------+
                                                                 | push
                                                                 v
                                                   +----------------------------+
                                                   | 4. CONTAINER REGISTRY      |
                                                   | ghcr.io / ECR / ACR / GAR  |
                                                   | Harbor / Artifactory       |
                                                   | image:1.4.2, :sha-3f9a1c2  |
                                                   +-------------+--------------+
                                                                 | pull by digest
                                                                 v
 +---------------------+   +------------------------------+   +-------------------+
 | 5. CD (Continuous   |   | 6. RUNTIME PLATFORM          |   | 7. OBSERVABILITY  |
 |    Delivery/Deploy) +-->|  - single VM + Compose       |-->|  logs  -> Loki/ELK|
 | pipeline job, or    |   |  - Kubernetes (EKS/AKS/GKE)  |   |  metrics->Prometheus
 | GitOps (Argo CD)    |   |  - ECS Fargate / Cloud Run   |   |  traces -> OTel   |
 | staging -> approval |   |  - Azure Container Apps      |   |  alerts -> pager  |
 | -> production       |   +--------------+---------------+   +---------+---------+
 +----------^----------+                  |                             |
            |                             v                             |
            |                  +---------------------+                  |
            |                  |  USERS / CUSTOMERS  |                  |
            |                  +---------------------+                  |
            |                                                           |
            +---------- 8. ROLLBACK: redeploy previous known-good <-----+
                           image tag/digest when alerts fire
```

Let us define each stage in plain English.

| # | Stage | What happens | Key idea |
|---|---|---|---|
| 1 | **Developer laptop** | You write code, run `docker compose up` to test, run unit tests. | "Works on my machine" becomes "works in the container," which is the same container production uses. |
| 2 | **Git host** | You push a branch and open a **pull request** (a proposal to merge your change). Teammates review it. **Branch protection** forbids merging unless checks pass. | Git is the *single source of truth*. Nothing reaches production except through git. |
| 3 | **CI** (Continuous Integration) | A robot (a "runner") automatically checks out the code, runs tests, builds the image, scans it for vulnerabilities, and signs it. | Humans make mistakes; pipelines are repeatable. |
| 4 | **Registry** | A server that stores images. CI pushes, production pulls. | The registry is the *hand-off point* between building and running. |
| 5 | **CD** (Continuous Delivery/Deployment) | Another automated step that takes an image from the registry and tells the runtime platform to run it. *Delivery* usually has a human approval button; *Deployment* is fully automatic. | You deploy an **image that was already tested**, never rebuild for production. |
| 6 | **Runtime platform** | The thing that actually runs containers 24/7: a VM, Kubernetes, serverless containers. | Choose by team size and needs ([Section 7](#7-where-do-you-run-the-containers-deployment-targets)). |
| 7 | **Observability** | Logs, metrics and traces flow to central tools. Alerts page a human. | You cannot fix what you cannot see. |
| 8 | **Rollback** | If the new version is bad, redeploy the previous image. | Fast rollback matters more than perfect testing. |

**The golden rule: build once, deploy many.** The *exact same image* (identified by its digest, explained next) travels through test, staging and production. Only configuration (environment variables, secrets) changes between environments. If you rebuild per environment, you are not testing what you ship.

---

## 4. Image tagging strategy

A **tag** is a human-friendly label attached to an image in a registry: in `ghcr.io/acme/chat-api:1.4.2`, `ghcr.io` is the registry, `acme/chat-api` is the repository, and `1.4.2` is the tag.

The most important fact about tags: **a tag is a movable sticky note, not a permanent name.** Anyone with push permission can move `1.4.2` to point to a different image tomorrow. What actually identifies an image is its **digest**, a SHA-256 hash of its content. The digest changes if one byte of the image changes, and it can never point to anything else.

```
 Tags (movable labels)                    Digest (permanent fingerprint)

   :latest  ----+
   :1.4     ----+--------------------->   sha256:9f86d081884c7d659a2feaa0c55ad015...
   :1.4.2   ----+                         (identifies exactly one image forever)
   :sha-3f9a1c2 +
```

### 4.1 Why `latest` is dangerous

`latest` is not magic. It is just the tag Docker uses when you do not specify one. It does **not** mean "newest." It means "whatever the publisher last tagged as latest, if they bothered."

Problems when you run `image: myapp:latest` in production:

1. **Unreproducible.** Two servers pulling `latest` an hour apart can run different code.
2. **No rollback target.** What do you roll back *to*? The previous `latest` is gone.
3. **Silent changes.** A restart can pull a new version you never tested.
4. **No audit trail.** Which commit caused last night's outage? Unknown.

Use `latest` for convenience in local development only.

### 4.2 The tags you should create for every build

| Tag type | Example | Purpose | Mutable? |
|---|---|---|---|
| **Git SHA** | `sha-3f9a1c2` | Exactly which commit produced this image. The most important tag for traceability. | No (treat as immutable) |
| **Semantic version** | `1.4.2` | A release humans talk about. Format `MAJOR.MINOR.PATCH`: bump MAJOR for breaking changes, MINOR for new backward-compatible features, PATCH for bug fixes. | No (treat as immutable) |
| **Minor / major alias** | `1.4`, `1` | Convenience: "give me the latest 1.4.x." Moves when a new patch is released. | Yes, by design |
| **Branch** | `main`, `staging` | Points to the newest build of that branch. Fine for dev/staging environments. | Yes, by design |
| **Environment** | `prod` | Marks "this is what production runs." Used in promotion (below). Moves on promotion. | Yes, by design |
| **Date-time** | `2026-10-23-1415` | An alternative when you do not use semver (some internal tools). | No |

**Recommended minimum:** every CI build pushes `sha-<short-commit>`. Every release (git tag `v1.4.2`) additionally pushes `1.4.2` and `1.4`.

### 4.3 Immutable tags

An **immutable tag** is one the registry refuses to overwrite once it is pushed. If you try to push a different image under `1.4.2`, the push fails.

- **Amazon ECR** has a per-repository "tag immutability" setting.
- **Azure Container Registry** lets you lock an image or repository (write/delete locks).
- **Google Artifact Registry** supports immutable tags per repository.
- **Harbor** supports tag immutability rules.
- **GitHub Container Registry and Docker Hub** have no repository-wide immutability switch **(unverified for 2026; check the current docs)**, so you enforce the discipline in your pipeline: never re-push a version or SHA tag.

If your registry supports it, turn it on for version and SHA tags. Leave alias tags (`1.4`, `prod`) mutable by using a separate repository or by choosing a registry that supports patterns for mutable tags.

### 4.4 Pinning by digest

For the strongest guarantee, deploy by digest:

```yaml
services:
  api:
    image: ghcr.io/acme/chat-api@sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

- `@sha256:...` means "exactly this content." The registry will refuse to serve anything else, and Docker verifies the hash after download.
- Humans cannot read digests, so keep the tag as a comment or have your deploy tooling resolve tag to digest. You can write both: `image: ghcr.io/acme/chat-api:1.4.2@sha256:9f86...` (Docker uses the digest and ignores the tag for resolution, but it is readable for humans).

Find a digest:

```bash
docker buildx imagetools inspect ghcr.io/acme/chat-api:1.4.2
```

- `docker buildx imagetools inspect` queries the **registry** (not your local cache) for the image manifest. Works for multi-architecture images.

Example output (shortened):

```
Name:      ghcr.io/acme/chat-api:1.4.2
MediaType: application/vnd.oci.image.index.v1+json
Digest:    sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08

Manifests:
  Name:        ghcr.io/acme/chat-api:1.4.2@sha256:1a2b...
  Platform:    linux/amd64

  Name:        ghcr.io/acme/chat-api:1.4.2@sha256:3c4d...
  Platform:    linux/arm64
```

- The top **Digest** is the digest of the *index* (a list pointing to one image per CPU architecture). Deploying the index digest works on both architectures.

### 4.5 Promotion between environments

**Promotion** means moving the *same image* from one stage to the next without rebuilding.

```
 CI builds & pushes                  gates                        gates
 sha-3f9a1c2 ------> DEV  ------> (tests pass) ---> STAGING ---> (human approval) ---> PROD
   (digest A)       runs A                          runs A                              runs A
                       \________ same digest A in every environment ________/
```

How to promote without rebuilding: add a **new tag** to the *existing* image.

```bash
docker buildx imagetools create \
  --tag ghcr.io/acme/chat-api:staging \
  ghcr.io/acme/chat-api:sha-3f9a1c2
```

- `docker buildx imagetools create` creates a new manifest in the registry **without pulling or re-uploading layers**, so it is fast and preserves the digest.
- `--tag ghcr.io/acme/chat-api:staging` is the new label to add.
- The last argument is the existing image being re-labelled.

Alternative tool: `crane tag ghcr.io/acme/chat-api:sha-3f9a1c2 staging` (from the open-source `crane` utility).

In GitOps setups ([Section 7.4](#74-kubernetes-for-docker-users)), promotion is a git commit that changes the image tag/digest in a manifest file.

### 4.6 A tagging policy you can copy

```
1. Every CI build on any branch:    push  :sha-<7-char-commit>
2. Merge to main:                   also  :main   (dev auto-deploys this)
3. Git tag vX.Y.Z (release):        also  :X.Y.Z  :X.Y  :X     (never overwritten)
4. Promotion to staging/prod:       add   :staging / :prod to the SAME digest
5. Production manifests reference:  :X.Y.Z or @sha256 digest. Never :latest or :main.
6. Retention: keep all release tags forever; expire untagged and old sha- tags after 30-90 days.
```

**Common mistakes (tagging)**

- Deploying `:latest` or `:main`.
- Re-pushing `1.4.2` after a hotfix (breaks everything that cached it). Make `1.4.3`.
- Rebuilding for production instead of promoting.
- Tagging only with version numbers and no SHA, so you cannot map an image to a commit.

---

## 5. Container registries

A **container registry** is a server that stores and serves images (like a library, or like GitHub for images). Pushing = uploading an image; pulling = downloading it. Registries speak a standard HTTP API defined by the **OCI Distribution Specification**, so any Docker client works with any registry. (OCI = Open Container Initiative, the industry body that standardises image and runtime formats.)

### 5.1 Hosted registries compared

| Registry | Address pattern | Who runs it | Good for | Notable features |
|---|---|---|---|---|
| **Docker Hub** | `docker.io/library/nginx`, `username/app` | Docker, Inc. | Public images, official base images | Largest public catalogue, "Docker Official Images" and "Verified Publisher" badges, automated scanning (Docker Scout), pull rate limits (see [5.3](#53-docker-hub-rate-limits)) |
| **GitHub Container Registry (GHCR)** | `ghcr.io/OWNER/IMAGE` | GitHub | Projects already on GitHub | Authenticates with the workflow's built-in `GITHUB_TOKEN`, free for public images, permissions inherit from the repository. GitHub documents container storage and bandwidth as free at the time of writing, but the policy has been described as a "soft" billing state, so treat "free" as current policy rather than a guarantee. |
| **Amazon ECR** (Elastic Container Registry) | `123456789012.dkr.ecr.us-east-1.amazonaws.com/app` | AWS | Workloads on AWS (ECS, EKS, Lambda) | IAM-based permissions, tag immutability, lifecycle policies, cross-region/account replication, basic and enhanced scanning (enhanced uses Amazon Inspector), **pull-through cache** for upstream registries |
| **Azure Container Registry (ACR)** | `myregistry.azurecr.io/app` | Microsoft | Workloads on Azure (AKS, Container Apps) | Entra ID (Azure AD) auth, geo-replication (Premium tier), ACR Tasks (build in the cloud), private endpoints, content trust and signing features, artifact cache |
| **Google Artifact Registry (GAR)** | `us-docker.pkg.dev/PROJECT/REPO/app` | Google Cloud | Workloads on GCP (GKE, Cloud Run) | Stores containers *and* language packages (npm, Maven, Python), IAM, cleanup policies, remote repositories (proxy Docker Hub) |
| **Quay.io** | `quay.io/ORG/app` | Red Hat | Red Hat ecosystems | Built-in scanning, robot accounts, OpenShift integration |
| **GitLab Container Registry** | `registry.gitlab.com/group/project` | GitLab | Projects on GitLab | Built into every GitLab project |

**Google Container Registry (the older `gcr.io` service) was shut down.** Writing images to it became unavailable in March 2025 and the final stage, on 14 October 2025, retired its remaining storage. `gcr.io` hostnames can now be served by Artifact Registry. If you see a tutorial using `gcr.io` with the old service, it is outdated; use Artifact Registry.

### 5.2 Self-hosted registries

Large companies often run their own registry because they want one internal place that all teams use, air-gapped (no internet) operation, caching of external images, and tight control over who can pull what.

| Product | Type | Highlights |
|---|---|---|
| **Harbor** | Open source, **CNCF graduated project** | Role-based access control, project quotas, vulnerability scanning (Trivy built in), image signing (Cosign/Notation), replication between registries, proxy-cache projects, retention rules, robot accounts for CI. The most popular open-source enterprise registry. |
| **JFrog Artifactory** | Commercial (with a limited free edition) | A universal repository: Docker images plus npm, Maven, PyPI, Helm and more, with virtual repositories that combine local, remote (cached upstream) and virtual views. |
| **Sonatype Nexus Repository** | Commercial with a free community edition | Similar universal repository; Docker proxy/hosted/group repository types. |
| **CNCF Distribution (`registry:3`)** | Open source, minimal | The reference implementation of the registry API. No UI, no RBAC, basic auth only. Fine for a lab, not for an enterprise. |

A minimal private registry for experimentation:

```bash
docker run -d -p 127.0.0.1:5000:5000 --name registry --restart=unless-stopped registry:3
```

- `-d` runs in the background. `-p 127.0.0.1:5000:5000` publishes port 5000 but **only on the local machine** (safer than exposing it to the world).
- `registry:3` is the official registry image (version 3 line). If your Docker Hub mirror does not have the `3` tag, check the registry project's current tags.
- `--restart=unless-stopped` restarts it after crashes or reboots.

Then push to it:

```bash
docker tag chat-api:1.0 localhost:5000/chat-api:1.0
docker push localhost:5000/chat-api:1.0
```

- `docker tag` gives your local image a second name that includes the registry address.
- `docker push` uploads it. Output shows layer IDs and `digest: sha256:... size: 1234`.

### 5.3 Docker Hub rate limits

Docker Hub limits how many times you can **pull** images, to control cost.

| Who | Limit (per official docs, seen October 2026) |
|---|---|
| Unauthenticated (anonymous) | **100 pulls per 6 hours**, counted per source IPv4 address (or IPv6 /64 block) |
| Authenticated, free Personal account | **200 pulls per 6 hours** |
| Paid plans (Pro, Team, Business) | Effectively unlimited, subject to fair use |

> **Verification note:** I could confirm these figures through the official documentation's summary and several secondary sources, but I could not load the live page. Docker also announced stricter hourly limits in 2025 and then said it had not enforced them. Always check <https://docs.docker.com/docker-hub/usage/> before designing around a number.

**Why this matters in the real world:** a CI system or a NAT gateway (many machines sharing one public IP) can burn through 100 pulls quickly. The symptom is:

```
toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```

Fixes, from cheapest to most robust:

1. **Authenticate** the pull (`docker login`), so you get the higher limit and, with a paid plan, no limit.
2. **Use a pull-through cache** (below), so each upstream image is downloaded once and served internally afterwards.
3. **Mirror critical base images** into your own registry (`docker buildx imagetools create` can copy them) and reference your copy in Dockerfiles.
4. **Use base images hosted elsewhere** (for example, many Official Images are also mirrored on `public.ecr.aws`).

### 5.4 Pull-through caches

A **pull-through cache** (a.k.a. proxy cache) is a registry that sits between you and an upstream registry. The first time someone pulls `nginx:1.27`, the cache fetches it from Docker Hub and stores a copy. Every later pull is served from the cache.

```
 CI runners / servers ---> internal cache (Harbor / ECR / ACR / GAR / Artifactory) ---> Docker Hub
        pull nginx              first pull: fetch + store                            (hit once)
        pull nginx              later pulls: served locally, no Hub rate limit consumed
```

Benefits: dodges rate limits, faster pulls, works if Docker Hub is briefly down, and gives you a place to scan and approve what comes in.

Options: ECR pull-through cache rules, ACR artifact cache, Artifact Registry remote repositories, Harbor proxy-cache projects, Artifactory/Nexus remote repositories, or the lightweight `registry` image in proxy mode. On a Docker host, configure the mirror in `daemon.json` with `registry-mirrors` ([Section 16](#16-docker-daemon-production-configuration-daemonjson)). Note that `registry-mirrors` only applies to Docker Hub pulls.

### 5.5 Retention and cleanup policies

Registries fill up. Every CI build adds an image; with 50 builds a day you will add thousands of images a year. Define a **retention policy** (a rule that automatically deletes old images):

| Rule | Example |
|---|---|
| Delete **untagged** images older than N days | 7 days |
| Keep only the last N `sha-*` images per repository | last 50 |
| Delete branch tags older than N days | 30 days |
| **Never** delete release tags (`1.4.2`) or anything currently deployed | exempt `^[0-9]+\.[0-9]+\.[0-9]+$` |

Every managed registry has this feature (ECR lifecycle policies, ACR retention/purge tasks, Artifact Registry cleanup policies, Harbor retention rules). Test your policy in "dry run" mode first. Deleting an image that production needs for a rollback is a classic outage.

### 5.6 Signing, SBOMs and attestations

- An **SBOM** (Software Bill of Materials) is a list of everything inside an image (OS packages, language libraries) so you can answer "are we affected by CVE-XXXX?" in seconds.
- **Provenance** is a signed statement saying "this image was built from this commit, by this workflow, at this time."
- **Signing** (for example with **Sigstore Cosign**) attaches a cryptographic signature so the cluster can verify an image came from your pipeline.
- BuildKit can generate SBOM and provenance attestations automatically (`--sbom=true --provenance=mode=max`, shown in the workflow below).
- Docker Content Trust (the older `DOCKER_CONTENT_TRUST` signing system) was **removed from the Docker CLI in Engine 29** according to the official release notes. Use Cosign or Notation for new work.

Security details are in [Chapter 10](10-security.md).

---

## 6. A complete CI/CD pipeline with GitHub Actions

**GitHub Actions** is GitHub's built-in automation service. A **workflow** is a YAML file in `.github/workflows/` that describes jobs; a **job** is a list of **steps** that runs on a **runner** (a temporary virtual machine GitHub gives you, for example `ubuntu-latest`). A step either runs a shell command (`run:`) or uses a reusable **action** (`uses:`).

### 6.1 Actions used, and their versions

Verified from the repositories' release pages in October 2026:

| Action | Purpose | Current major (Oct 2026) |
|---|---|---|
| `actions/checkout` | Downloads your repository into the runner | v7 |
| `docker/setup-qemu-action` | Installs CPU emulators so an x86 runner can build ARM images | v4 |
| `docker/setup-buildx-action` | Sets up **Buildx**, Docker's modern builder (uses the BuildKit engine) | v4 |
| `docker/login-action` | Logs in to a registry | v4 |
| `docker/metadata-action` | Computes image tags and labels from git info | v6 |
| `docker/build-push-action` | Runs the build and optionally pushes | v7 (runs on Node.js 24) |

> **Supply-chain warning, and why this chapter does not use `trivy-action`.** In March 2026 the Trivy project's release pipeline and its GitHub Actions (`aquasecurity/trivy-action`, `aquasecurity/setup-trivy`) were compromised: attackers published a malicious Trivy release and rewrote most version tags to point at credential-stealing code. Anyone referencing a *tag* such as `@0.28.0` could have run attacker code with access to their CI secrets. Lessons: (1) a version tag in `uses:` is a movable sticky note, exactly like an image tag; (2) third-party actions run with your secrets; (3) pin third-party actions to a **full commit SHA that you verified against the upstream repository**, and keep the number of third-party actions small. In the workflow below we run Trivy as a **container image pinned to a version** using plain `docker run`, so no third-party action is involved. For maximum safety you would also pin that image by digest. For the official Docker-maintained actions, major-version tags are widely used; the strictest teams pin those to SHAs too (Dependabot or Renovate can keep the SHAs updated automatically).

### 6.2 Repository layout the pipeline assumes

```
chat-api/
├── Dockerfile
├── .dockerignore
├── compose.prod.yaml        # production Compose file (used on the VM)
├── deploy.sh                # deploy + rollback script (lives on the VM too)
├── src/ ...
└── .github/
    └── workflows/
        └── ci-cd.yaml       # the workflow below
```

### 6.3 The workflow, annotated line by line

Save as `.github/workflows/ci-cd.yaml`.

```yaml
name: ci-cd
```
- `name` is the label shown in the GitHub "Actions" tab.

```yaml
on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
```
- `on` defines **triggers**. This workflow runs when you push to `main`, when you push a git tag shaped like `v1.4.2`, and when someone opens or updates a pull request targeting `main`.
- On pull requests the pipeline builds and scans but does **not** push or deploy (we enforce that with `if:` conditions below).

```yaml
concurrency:
  group: ci-cd-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```
- `concurrency` stops two runs for the same branch from racing each other. `group` names the lane; `cancel-in-progress` cancels an older **pull request** run when a new commit arrives (saves minutes), but never cancels a run on `main` or a tag (a half-finished deploy is dangerous).

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  TRIVY_VERSION: "0.76.1"
```
- `env` at the top defines variables for all jobs. `github.repository` is `OWNER/REPO` (for example `acme/chat-api`), so the image becomes `ghcr.io/acme/chat-api`. GHCR requires **lowercase** repository names; if your owner has capital letters, lowercase it (a step in the job below does this safely).
- `TRIVY_VERSION` pins the scanner version. 0.76.1 was the newest release found at writing time; check <https://github.com/aquasecurity/trivy/releases> and update. Never use a version from the March 2026 compromised range (v0.69.4 was the malicious release).

```yaml
permissions:
  contents: read
```
- `permissions` at workflow level sets the **default** power of the automatic `GITHUB_TOKEN` to read-only. Jobs that need more ask for it explicitly. This is least privilege: a compromised step cannot, for instance, write to your repository.

#### Job 1: build, scan, push

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      id-token: write
    outputs:
      image: ${{ steps.vars.outputs.image }}
      digest: ${{ steps.push.outputs.digest }}
      tag: ${{ steps.vars.outputs.sha_tag }}
```
- `jobs.build` is a job named `build`. `runs-on: ubuntu-latest` picks the runner OS (Ubuntu Linux).
- `timeout-minutes` kills a stuck job instead of burning minutes for six hours.
- `packages: write` allows pushing to GHCR with `GITHUB_TOKEN`. `id-token: write` allows requesting an OIDC token (used by keyless signing or cloud logins; harmless if unused, delete it if you do not need it).
- `outputs` exposes values (the image name, digest and SHA tag) so the **deploy** job can use them.

```yaml
    steps:
      - name: Check out code
        uses: actions/checkout@v7
```
- Downloads your repository into the runner. `@v7` is the version of the action.

```yaml
      - name: Compute lowercase image name and SHA tag
        id: vars
        run: |
          IMAGE="${REGISTRY}/${IMAGE_NAME,,}"
          echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"
          echo "sha_tag=sha-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
```
- `id: vars` lets later steps refer to this step's outputs.
- `${IMAGE_NAME,,}` is a Bash trick that lowercases the string.
- `echo "key=value" >> "$GITHUB_OUTPUT"` is how a step publishes an output. `${GITHUB_SHA::7}` is the first 7 characters of the commit hash.

```yaml
      - name: Set up QEMU (cross-CPU emulation)
        uses: docker/setup-qemu-action@v4
```
- **QEMU** is an emulator. This registers it so the x86 runner can execute ARM binaries during the build, which lets us make `linux/arm64` images. Skip this step if you only build for one architecture.

```yaml
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v4
```
- Creates a Buildx builder using the BuildKit engine, which gives us cache export, multi-platform builds, and attestations.

```yaml
      - name: Extract tags and labels
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ${{ steps.vars.outputs.image }}
          tags: |
            type=sha,format=short
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=raw,value=latest,enable={{is_default_branch}}
```
- `docker/metadata-action` turns git facts into Docker tags and OCI **labels** (key/value metadata baked into the image, such as source URL and commit).
- `images` is the image name without a tag.
- Each `type=` line is a tagging rule:
  - `type=sha,format=short` makes `sha-3f9a1c2`.
  - `type=ref,event=branch` makes `main` when pushing to main.
  - `type=ref,event=pr` makes `pr-123` for pull requests.
  - `type=semver,pattern={{version}}` makes `1.4.2` when the pushed git tag is `v1.4.2`. `{{major}}.{{minor}}` makes `1.4`.
  - `type=raw,value=latest,enable={{is_default_branch}}` adds `latest` only on the default branch. We keep it for human convenience; production manifests must never reference it.

```yaml
      - name: Build image for scanning (amd64, loaded into local Docker)
        uses: docker/build-push-action@v7
        with:
          context: .
          load: true
          push: false
          tags: scan-target:ci
          cache-from: type=gha
          cache-to: type=gha,mode=max
```
- `docker/build-push-action` runs `docker buildx build`. Here we build **one** image (amd64) and `load: true` puts it into the runner's local Docker so we can scan it. (Multi-platform images cannot be loaded into the classic local store, which is why we scan one platform now and push all platforms later.)
- `context: .` means "use the repo root as the build context" (see [Chapter 4](04-dockerfile-deep-dive.md)).
- `push: false` makes sure nothing is uploaded yet.
- `tags: scan-target:ci` is a throwaway local name.
- `cache-from: type=gha` / `cache-to: type=gha,mode=max` use the **GitHub Actions cache** as a store for Docker layer cache between runs. `mode=max` caches intermediate stages as well (important for multi-stage builds), so repeated builds are much faster. The cache is limited in size per repository (10 GB at the time of writing) and old entries are evicted; if you outgrow it, use `type=registry` (a cache stored as an image in your registry) instead.

```yaml
      - name: Scan image with Trivy (fail on HIGH/CRITICAL with fixes available)
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$HOME/.cache/trivy:/root/.cache/" \
            ghcr.io/aquasecurity/trivy:${TRIVY_VERSION} image \
              --exit-code 1 \
              --severity HIGH,CRITICAL \
              --ignore-unfixed \
              --no-progress \
              scan-target:ci
```
- `docker run --rm` runs a one-off container and deletes it afterwards.
- `-v /var/run/docker.sock:/var/run/docker.sock` gives the scanner access to the runner's Docker so it can read the local image `scan-target:ci`. (This socket is root-equivalent power; acceptable on a throwaway CI runner, never on a production server. See [Chapter 10](10-security.md).)
- `-v "$HOME/.cache/trivy:/root/.cache/"` persists the vulnerability database between steps/runs on this runner so it does not redownload every time.
- `ghcr.io/aquasecurity/trivy:${TRIVY_VERSION}` is the official Trivy image, pinned to a version.
- `image` is the Trivy sub-command for scanning container images.
- `--exit-code 1` makes Trivy return a failing exit code if it finds matching vulnerabilities, which fails the job and blocks the pipeline.
- `--severity HIGH,CRITICAL` only fail on high and critical issues (otherwise nearly every image fails on trivial "LOW" items).
- `--ignore-unfixed` ignores vulnerabilities with no available fix; you cannot act on those, and blocking on them stalls teams. Revisit this setting as your maturity grows.
- `--no-progress` disables the animated progress bar so CI logs stay clean.

Example failing output:

```
scan-target:ci (debian 12.8)
============================
Total: 1 (HIGH: 1, CRITICAL: 0)

┌─────────┬────────────────┬──────────┬────────┬───────────────────┬───────────────┐
│ Library │ Vulnerability  │ Severity │ Status │ Installed Version │ Fixed Version │
├─────────┼────────────────┼──────────┼────────┼───────────────────┼───────────────┤
│ libexpat│ CVE-2026-XXXXX │ HIGH     │ fixed  │ 2.5.0-1           │ 2.5.0-1+deb12u2│
└─────────┴────────────────┴──────────┴────────┴───────────────────┴───────────────┘
Error: Process completed with exit code 1.
```

(The CVE number above is a made-up placeholder to show the format.)

```yaml
      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
- `if: github.event_name != 'pull_request'` skips this on pull requests (PRs from forks do not get write permission anyway).
- `docker/login-action` runs `docker login` for you. `username: ${{ github.actor }}` is the user who triggered the run. `password: ${{ secrets.GITHUB_TOKEN }}` is the automatic short-lived token GitHub creates for each run, so you do not store any long-lived registry password.
- `secrets.GITHUB_TOKEN` is never printed in logs (GitHub masks secrets).

```yaml
      - name: Build and push (multi-arch, with SBOM and provenance)
        id: push
        if: github.event_name != 'pull_request'
        uses: docker/build-push-action@v7
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          sbom: true
          provenance: mode=max
```
- Same action, but now `push: true` and `platforms: linux/amd64,linux/arm64` builds for both Intel/AMD and ARM (AWS Graviton, Apple Silicon, Raspberry Pi) and publishes them as a single multi-arch image. Because the layers are in the cache, only the arm64 part has real work to do.
- `tags` and `labels` come from the metadata step (multi-line lists).
- `sbom: true` attaches an SBOM; `provenance: mode=max` attaches a detailed build-provenance statement. These are stored in the registry next to the image. (Side effect: the GHCR web page may show extra entries of platform "unknown/unknown". Those are the attestation manifests, not broken images.)
- `id: push` lets us read this step's `digest` output, which is the exact fingerprint of what was pushed.

```yaml
      - name: Summary
        if: github.event_name != 'pull_request'
        run: |
          echo "### Image pushed" >> "$GITHUB_STEP_SUMMARY"
          echo "- Image: \`${{ steps.vars.outputs.image }}\`" >> "$GITHUB_STEP_SUMMARY"
          echo "- Digest: \`${{ steps.push.outputs.digest }}\`" >> "$GITHUB_STEP_SUMMARY"
          echo "- Tags:" >> "$GITHUB_STEP_SUMMARY"
          echo '${{ steps.meta.outputs.tags }}' | sed 's/^/  - /' >> "$GITHUB_STEP_SUMMARY"
```
- `$GITHUB_STEP_SUMMARY` is a file whose Markdown content is shown on the workflow run page. Handy for humans to see which digest was built.

#### Job 2: deploy over SSH to a Linux VM

```yaml
  deploy:
    needs: build
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    runs-on: ubuntu-latest
    environment: production
    permissions:
      contents: read
```
- `needs: build` means "wait for the build job to succeed."
- `if:` limits deployment to pushes to `main` or to a release tag; never from pull requests.
- `environment: production` ties the job to a GitHub **Environment**. In the repository settings you can add **required reviewers** to that environment, so the job pauses until a human clicks "Approve." That is your "continuous delivery" gate. Environments also scope secrets, so production secrets are only exposed to this job.

```yaml
    steps:
      - name: Configure SSH
        env:
          SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          KNOWN_HOSTS: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
        run: |
          install -m 700 -d ~/.ssh
          printf '%s\n' "$SSH_KEY" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          printf '%s\n' "$KNOWN_HOSTS" > ~/.ssh/known_hosts
```
- `secrets.DEPLOY_SSH_KEY` is a private SSH key you created *only for deployments* and stored in the GitHub environment's secrets. Its public half is in the VM's `~/.ssh/authorized_keys` for a dedicated `deploy` user.
- `secrets.DEPLOY_KNOWN_HOSTS` holds the server's host key line (get it once with `ssh-keyscan -t ed25519 your.server.example`, then **verify** the fingerprint out-of-band). Using a pinned known-hosts value means the job refuses to talk to an impostor server. Avoid `StrictHostKeyChecking=no`; it disables that protection.
- `install -m 700 -d ~/.ssh` creates the folder with correct permissions; SSH refuses keys with loose permissions.
- Using plain `ssh` rather than a third-party "ssh deploy" action means there is no extra code that can read your key.

```yaml
      - name: Deploy
        env:
          HOST: ${{ secrets.DEPLOY_HOST }}
          USER: ${{ secrets.DEPLOY_USER }}
          IMAGE_TAG: ${{ needs.build.outputs.tag }}
        run: |
          ssh -o BatchMode=yes "${USER}@${HOST}" \
            "cd /opt/chat-api && ./deploy.sh '${IMAGE_TAG}'"
```
- Passes the SHA tag produced by the build job to a script on the server. `BatchMode=yes` makes SSH fail instead of waiting for a password prompt.
- The script (next section) pulls the image, restarts the stack, waits until it is healthy, and rolls back if it is not.
- Notice the server needs *no inbound access from GitHub except SSH*, and it only needs a **read-only registry credential** to pull images. If you prefer no SSH at all, use a pull-based tool or GitOps (see [Section 7](#7-where-do-you-run-the-containers-deployment-targets)).

### 6.4 The server side: `compose.prod.yaml` and `deploy.sh`

`/opt/chat-api/compose.prod.yaml`:

```yaml
services:
  api:
    image: ghcr.io/acme/chat-api:${IMAGE_TAG:?IMAGE_TAG must be set}
    restart: unless-stopped
    env_file: /opt/chat-api/secrets/api.env
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "5"
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    ports:
      - "127.0.0.1:8000:8000"
```
- `${IMAGE_TAG:?message}` makes Compose **fail loudly** if the variable is missing instead of silently using an empty tag.
- `restart: unless-stopped` restarts after crashes and reboots, unless you stopped it manually.
- `env_file` loads secret variables from a file only root/deploy can read ([Section 9](#9-configuration-and-secrets-in-production)).
- `healthcheck` tells Docker how to test the app; `--wait` (below) uses it. It uses Python because slim images often lack `curl`.
- `logging` caps log size (see [Section 12](#12-logging)).
- `deploy.resources.limits` caps CPU and memory ([Section 11](#11-resource-limits-and-the-oom-killer)). Modern Compose honours these outside Swarm.
- `ports: "127.0.0.1:8000:8000"` publishes the port **only on localhost**; the reverse proxy (Section 7.1) is the only public entrance.

`/opt/chat-api/deploy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

NEW_TAG="${1:?usage: deploy.sh <image-tag>}"
cd /opt/chat-api
PREV_TAG="$(cat .current_tag 2>/dev/null || echo none)"

echo "Deploying ${NEW_TAG} (previous: ${PREV_TAG})"
export IMAGE_TAG="${NEW_TAG}"

docker compose -f compose.prod.yaml pull api
if docker compose -f compose.prod.yaml up -d --wait --wait-timeout 120; then
  echo "${NEW_TAG}" > .current_tag
  echo "${PREV_TAG}" > .previous_tag
  docker image prune -f --filter "until=168h" >/dev/null
  echo "Deploy OK"
else
  echo "Deploy FAILED, rolling back to ${PREV_TAG}" >&2
  if [ "${PREV_TAG}" != "none" ]; then
    export IMAGE_TAG="${PREV_TAG}"
    docker compose -f compose.prod.yaml up -d --wait --wait-timeout 120
  fi
  exit 1
fi
```
- `set -euo pipefail` makes the script stop on the first error (`-e`), on unset variables (`-u`), and on failures inside pipes (`pipefail`).
- `PREV_TAG` remembers what was running so we can roll back.
- `docker compose pull api` downloads the new image **before** touching the running container, so downtime is just the swap.
- `docker compose up -d --wait --wait-timeout 120` recreates changed services in the background (`-d`) and **waits** (`--wait`) until containers are running and, if they define a healthcheck, healthy. If that does not happen within 120 seconds, the command fails. This is the key to safe automated deploys.
- On failure we start the previous tag again and exit non-zero so the GitHub job turns red.
- `docker image prune -f --filter "until=168h"` removes unused images older than a week so the disk does not fill (only dangling/unused images; running images are untouched).

> Note that this plain-Compose approach has a **few seconds of downtime** while the container is replaced. [Section 8](#8-deployment-strategies-and-rollbacks) shows how to get to zero.

### 6.5 One-time setup checklist for this pipeline

1. Create the VM, install Docker Engine ([Chapter 2](02-installation-and-operating-systems.md)), create a `deploy` user in the `docker` group (remember: membership in the `docker` group is root-equivalent, so use a dedicated key and restrict it; stricter shops use rootless Docker or `command=` restrictions in `authorized_keys`).
2. On the VM: `docker login ghcr.io` using a **fine-grained token with read-only package access** (or make the image public).
3. In GitHub: create the `production` environment, add reviewers, add the secrets `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`.
4. Enable branch protection on `main` requiring the `build` job to pass.
5. First deploy manually with `./deploy.sh sha-xxxxxxx` to confirm it works.

**Common mistakes (CI/CD)**

- Giving the pipeline a personal access token with full power instead of the scoped `GITHUB_TOKEN`.
- No `timeout-minutes` (a hung job burns your CI quota).
- Building the image again at deploy time instead of pulling the one CI built.
- Letting pull requests from forks access secrets.
- Skipping the health-wait so "deploy succeeded" means only "we ran a command."
- Referencing third-party actions by mutable tag without thinking about the supply-chain risk.
- Forgetting that the GHCR package is **private by default**, then wondering why the server gets `denied`.

---

## 7. Where do you run the containers? Deployment targets

There is no single right answer. Choose based on team size, traffic, and how much operations work you are willing to do.

```
  Operational effort ->  LOW ................................................. HIGH
  Control & flexibility -> LOW ............................................... HIGH

  PaaS                Serverless containers        Single VM + Compose     Kubernetes (managed)    Kubernetes (self-run)
  (Fly.io, Render,    (Cloud Run, ECS Fargate,     (you run Docker         (EKS, AKS, GKE)         (kubeadm, bare metal)
   Railway)            Azure Container Apps)        on one server)
```

### 7.0 Decision table

| Situation | Reasonable choice |
|---|---|
| Side project, demo, portfolio app | Single VM + Compose, or a PaaS (Render/Railway/Fly.io) |
| Start-up with 1-5 engineers, one or two services | Cloud Run / ECS Fargate / Azure Container Apps, or a VM + Compose with a managed database |
| Need scale-to-zero (pay nothing when idle) | Cloud Run or Azure Container Apps |
| Dozens of services, many teams, need a common platform | Managed Kubernetes (EKS/AKS/GKE) |
| GPU inference at scale | Kubernetes with GPU node pools, or a managed GPU service ([Chapter 8](08-docker-for-ai-and-ml.md)) |
| Strict data-residency, on-prem, air-gapped | Self-managed Kubernetes or OpenShift |
| Team is already using Swarm and it works | Keep Swarm; plan an exit only if you need features it lacks |
| Regulated workload with required vendor support | Platform from your cloud vendor with support plan |

### 7.1 Single VM + Docker Compose (do it properly)

A single Linux VM running Docker Compose is **good enough for far more projects than people admit**: internal tools, small SaaS products, proofs of concept, and many production systems serving thousands of users. Its weaknesses are that the VM is a single point of failure, scaling is vertical (buy a bigger machine), and you own the patching. If you accept those, here is how to do it *properly* instead of "docker run in a tmux session."

#### The target layout

```
 Internet
    |  80/443 (HTTPS)
    v
+----------------------------------- Linux VM (Ubuntu LTS) ------------------------------------+
|  firewall: allow 22 (from your IP), 80, 443 only                                              |
|                                                                                               |
|   +---------+     internal Docker network "app-net"                                          |
|   |  Caddy  |----> api (127.0.0.1:8000 / internal)  ----> redis                              |
|   | reverse |                                       \---> (managed Postgres outside the VM)  |
|   |  proxy  |  automatic TLS certificates (Let's Encrypt)                                    |
|   +---------+                                                                                 |
|   Docker Engine: json/local log rotation, live-restore, restart policies                      |
|   systemd: starts Docker at boot     cron/systemd timers: backups, cleanup                    |
+-----------------------------------------------------------------------------------------------+
```

#### Step 1: Harden the host

```bash
# Create a non-root user for you and one for deployments
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
```
- `adduser --disabled-password` creates a user that cannot log in with a password (key-only). `usermod -aG docker deploy` appends (`-a`) the user to the `docker` group (`-G`), letting it run Docker without sudo. Remember that this is effectively root access on the machine.

```bash
# SSH: keys only, no root login
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
```
- Edits the SSH server config: disables password logins and root login, then reloads the SSH service. (On some distributions the service is called `sshd`.) **Keep your current session open and test a second login before closing it**, in case you lock yourself out.

#### Step 2: Firewall, and the Docker-bypasses-ufw trap

Ubuntu's friendly firewall front-end is `ufw`:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```
- `default deny incoming` blocks everything inbound unless allowed; `allow outgoing` lets the server reach the internet. The three `allow` lines open SSH, HTTP and HTTPS. `ufw enable` activates it.

> **The trap:** when you publish a port with `-p 8080:80` (or `ports:` in Compose), **Docker inserts its own iptables rules that run before ufw's rules**, so the port can be open to the world even though `ufw status` shows it blocked. Docker's own documentation describes this interaction. Defences: (1) publish internal-only services on loopback, `127.0.0.1:8000:8000`, so they are not reachable externally in the first place; (2) do not publish database or cache ports at all, since containers on the same Compose network reach each other by service name; (3) for rules that must restrict published ports, add them to the `DOCKER-USER` iptables chain (the chain Docker designates for your rules), or use a cloud-provider firewall (security group) in front of the VM, which Docker cannot bypass. Always verify from **outside**, for example with `nmap` from another machine or an online port scanner.

#### Step 3: Reverse proxy and TLS

A **reverse proxy** is a server that sits in front of your apps, receives all public traffic, and forwards requests to the right container. It terminates **TLS** (the encryption behind `https://`), routes by hostname or path, and can add compression, rate limits and headers. Options:

| Proxy | Strength | Best for |
|---|---|---|
| **Caddy** | Obtains and renews Let's Encrypt certificates **automatically** with a 3-line config | Small to medium deployments; the easiest path |
| **Traefik** | Discovers containers automatically from Docker labels; automatic certificates | Many changing services on one host |
| **nginx + certbot** | Ubiquitous, extremely flexible; certificates via the separate `certbot` tool and a renewal timer | Teams that already know nginx |

Caddy example. `Caddyfile`:

```
chat.example.com {
    encode zstd gzip
    reverse_proxy api:8000
}
```
- `chat.example.com { ... }` is a *site block*. When Caddy sees a real domain name it automatically requests and renews a free certificate from Let's Encrypt and redirects HTTP to HTTPS. (The domain's DNS A record must already point to this server, and ports 80/443 must be reachable.)
- `encode zstd gzip` compresses responses.
- `reverse_proxy api:8000` forwards requests to the Compose service named `api` on port 8000. Names resolve through Docker's internal DNS ([Chapter 5](05-storage-and-networking.md)).

Add Caddy to the Compose file:

```yaml
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      api:
        condition: service_healthy

volumes:
  caddy_data:
  caddy_config:
```
- `caddy:2` is the official image on the 2.x major line. Pin a full version (for example `caddy:2.10`) in real deployments. I did not verify the newest 2.x minor version, so check Docker Hub.
- `443:443/udp` allows HTTP/3 (QUIC).
- `./Caddyfile:/etc/caddy/Caddyfile:ro` bind-mounts your config read-only.
- `caddy_data` **must be a persistent volume**: it stores the issued certificates and account keys. Without it Caddy would re-request certificates on every restart and quickly hit Let's Encrypt rate limits.
- `depends_on ... condition: service_healthy` starts Caddy only after `api` reports healthy.

#### Step 4: Start on boot, restart on crash

Docker is normally enabled at boot by the installer. Confirm:

```bash
sudo systemctl enable --now docker
systemctl is-enabled docker
```
- `systemctl enable --now docker` makes the Docker service start at every boot (`enable`) and starts it right now (`--now`). `is-enabled` should print `enabled`.

Containers with `restart: unless-stopped` come back automatically when the daemon starts. If you want the whole stack to be a first-class system service, add a systemd unit `/etc/systemd/system/chat-api.service`:

```ini
[Unit]
Description=chat-api (Docker Compose stack)
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/chat-api
EnvironmentFile=-/opt/chat-api/.current_env
ExecStart=/bin/bash -c 'IMAGE_TAG=$(cat .current_tag) docker compose -f compose.prod.yaml up -d --remove-orphans'
ExecStop=/usr/bin/docker compose -f compose.prod.yaml down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
- `[Unit]` describes ordering: start after Docker and after the network is up.
- `Type=oneshot` plus `RemainAfterExit=yes` means "this command runs once and finishes, but treat the service as active afterward" (Compose runs containers in the background, so there is no long-running process for systemd to watch).
- `ExecStart` brings the stack up with the last deployed tag. `ExecStop` brings it down (note: `down` on stop is optional; many people omit it so a `systemctl restart` does not stop everything for long).
- Enable it with `sudo systemctl daemon-reload && sudo systemctl enable chat-api`.
- With `restart: unless-stopped` already in place, this unit is a belt-and-braces extra, not a requirement.

#### Step 5: Patching the host

**Unattended upgrades** installs security updates automatically on Debian/Ubuntu:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```
- The first command installs the tool; the second enables the daily automatic security updates (`-plow` means "ask only low-priority-and-above questions").

Caveats: a kernel update needs a reboot to take effect; enable `Unattended-Upgrade::Automatic-Reboot` for a quiet window or schedule reboots. If `docker-ce` is upgraded automatically the daemon restarts and (without `live-restore`) so do all containers. Many teams exclude Docker packages from unattended upgrades (`Unattended-Upgrade::Package-Blacklist`) and upgrade Docker deliberately ([Section 18](#18-upgrading-docker-engine)).

Updating **container images** automatically: **Watchtower** (the tool that polled for new image versions and restarted containers) is **no longer a safe default**. The original `containrrr/watchtower` repository was archived (read-only) on 17 December 2025, with its last release at v1.7.1, and the maintainers said they could not vouch for the forks. Forks exist and some are maintained, but I could not verify any of them from primary sources, so treat them as third-party code with Docker socket access (root-equivalent). The better pattern is the pipeline in Section 6: CI builds, scans, and *tells the server* exactly which version to run.

#### Step 6: Backups

Back up **state**, not containers (containers are disposable). See [Section 14](#14-persistence-and-backups). A minimal schedule: nightly database dump plus volume archive, shipped off the VM to object storage, and a tested restore.

#### What "good enough" looks like for single VM

| Requirement | How |
|---|---|
| Survive container crash | `restart: unless-stopped`, healthchecks |
| Survive reboot | Docker enabled at boot |
| Survive VM loss | Infrastructure as code (Terraform) or a documented rebuild, plus off-site backups and managed DB |
| HTTPS | Caddy/Traefik/certbot |
| No downtime deploys | Blue/green with the proxy (Section 8) |
| Patching | Unattended upgrades + scheduled reboots |
| Monitoring | Uptime check from outside + Prometheus/Grafana or a hosted agent |

### 7.2 Docker Swarm: status and when to use it

**Docker Swarm mode** is a clustering feature built into Docker Engine. You join several machines into a **swarm**; some are *managers* (store state, make decisions) and some are *workers*. You describe **services** (a container image plus replicas) and Swarm schedules them across machines, restarts failed tasks, load-balances, and does rolling updates.

```bash
docker swarm init --advertise-addr 10.0.0.5
docker stack deploy -c compose.prod.yaml chat
docker service ls
docker service update --image ghcr.io/acme/chat-api:1.4.3 chat_api
docker service update --rollback chat_api
```
- `docker swarm init --advertise-addr 10.0.0.5` turns this machine into the first manager and tells other nodes which IP to use to reach it.
- `docker stack deploy -c compose.prod.yaml chat` deploys a Compose-format file as a **stack** named `chat`. In Swarm, the `deploy:` section (replicas, `update_config`, `rollback_config`, `placement`) is honoured in full.
- `docker service ls` lists services and replica counts.
- `docker service update --image ...` performs a rolling update; `--rollback` returns to the previous spec.

**Status (as of October 2026).** Swarm is *not* formally deprecated: Docker's documentation carries no end-of-life banner, and Docker Engine 29.x release notes still list Swarm-mode fixes. Mirantis, which acquired Docker's enterprise business in 2019, publicly committed in 2025 to support Swarm in Mirantis Kubernetes Engine 3 through at least 2030 (reported by secondary sources; **(unverified)** from primary text, and one blog reported a Mirantis corporate acquisition that I could not corroborate). However, Swarm is widely regarded as being in **maintenance mode**: security and bug fixes continue, but significant new features are not expected. The ecosystem (tooling, hiring, managed offerings) has moved to Kubernetes.

**When Swarm still makes sense:**

- A small team already comfortable with Compose that needs 2-5 machines, rolling updates and secrets, without Kubernetes complexity.
- Edge or on-prem sites with limited resources.

**When to avoid it for new projects:** if you expect to grow, need autoscaling, a large ecosystem (service mesh, operators, GitOps), or managed offerings, start on Kubernetes or a serverless container platform. Note that `live-restore` is not compatible with Swarm mode.

### 7.3 A reality check before Kubernetes

Kubernetes (often written **K8s**, "K" + 8 letters + "s") is the industry standard container orchestrator, but it brings serious complexity. A fair rule: if you cannot name a concrete reason you need it (many services, many teams, autoscaling, custom networking policy, multi-cloud, a platform team), a managed serverless container service or a single VM may serve you better and cost far less in people-time.

### 7.4 Kubernetes for Docker users

An **orchestrator** is software that runs containers across a *fleet* of machines and continuously makes reality match your description ("I want 3 copies of this container, reachable at this URL"). This "describe the desired state, let the system converge" idea is called **declarative** management.

Kubernetes architecture in one picture:

```
                        +-------------------- CONTROL PLANE (the brain) --------------------+
 you: kubectl apply --> | API server <--> etcd (database of desired state)                  |
                        | scheduler (picks a node)   controller managers (make it so)      |
                        +--------------------------------+----------------------------------+
                                                         |
              +------------------------------------------+-------------------------------+
              |                                          |                               |
        +-----v------ WORKER NODE ----+          +-----v------ WORKER NODE ----+   ...
        | kubelet (node agent)        |          | kubelet                     |
        | container runtime (containerd)|        | container runtime           |
        | [Pod][Pod]   [Pod]          |          | [Pod]   [Pod]               |
        +-----------------------------+          +-----------------------------+
```

Note: Kubernetes removed its built-in Docker integration ("dockershim") in v1.24 and runs containers through **containerd** or CRI-O. Images built with Docker still work perfectly because images follow the OCI standard.

#### Vocabulary mapped from Docker to Kubernetes

| Kubernetes term | Plain-English meaning | Docker/Compose analogy |
|---|---|---|
| **Pod** | The smallest unit: one or more tightly coupled containers sharing a network address and storage. Usually one app container. Pods are disposable. | A container (or a group sharing a network namespace) |
| **Deployment** | Manages a set of identical pods: how many replicas, which image, how to roll out updates and roll back. | A Compose service with `replicas` plus update logic |
| **ReplicaSet** | The object a Deployment uses behind the scenes to keep N pods alive. You rarely touch it. | (internal) |
| **Service** | A stable internal address (and load balancer) in front of a changing set of pods. Pods get new IPs each restart; the Service name does not change. | Compose service name resolved by Docker DNS |
| **Ingress** / **Gateway API** | Rules that route external HTTP(S) traffic to Services by host and path, usually with TLS. | Your reverse proxy (Caddy/nginx) |
| **ConfigMap** | Non-secret configuration, injected as environment variables or files. | `environment:` / `env_file` / config files |
| **Secret** | Sensitive configuration, injected like a ConfigMap. (By default only base64-encoded, **not encrypted**; enable encryption at rest or use an external secrets manager.) | Compose `secrets:` |
| **Namespace** | A virtual partition of the cluster (for example `dev`, `staging`, `team-a`) to separate resources, quotas and permissions. | Compose project name |
| **HPA** (Horizontal Pod Autoscaler) | Automatically changes the replica count based on CPU or custom metrics. | (No equivalent in Compose) |
| **Volume / PersistentVolumeClaim (PVC)** | Storage that outlives a pod; a PVC is a request for storage from the cluster. | Docker volume |
| **StatefulSet** | Like a Deployment but for stateful apps needing stable names and storage. | (No equivalent) |
| **Job / CronJob** | Run-to-completion tasks / scheduled tasks. | `docker run` one-offs, cron |
| **Node** | A machine in the cluster. | A Docker host |
| **kubectl** | The CLI. | `docker` CLI |
| **Helm** | A package manager for Kubernetes manifests (charts). | Compose + templating |

Current state (verified October 2026): Kubernetes **1.37** is the latest stable minor release (released August 2026); 1.36 and 1.35 are still supported; 1.38 is scheduled for December 2026. Kubernetes supports roughly the three most recent minor versions, so clusters need regular upgrades.

> **Ingress news:** the Kubernetes community's popular `ingress-nginx` controller (the one in `kubernetes/ingress-nginx`) was **retired in March 2026**. It receives no further releases or security fixes. The Kubernetes `Ingress` API object still exists (frozen), other controllers (Traefik, Contour, cloud load-balancer controllers, F5's separate NGINX controller) keep working, and the successor API is **Gateway API**. For new clusters, pick a maintained controller or implement Gateway API from the start.

#### A Compose file and its Kubernetes equivalent

Compose (the sort of file you already know):

```yaml
services:
  api:
    image: ghcr.io/acme/chat-api:1.4.2
    environment:
      LOG_LEVEL: info
    env_file: secrets/api.env
    ports:
      - "8000:8000"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
```

Kubernetes equivalent (one file, several objects separated by `---`). Save as `chat-api.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chat
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: chat-api-config
  namespace: chat
data:
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: chat-api-secrets
  namespace: chat
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:CHANGE_ME@db.internal:5432/chat"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chat-api
  namespace: chat
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chat-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: chat-api
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/chat-api:1.4.2
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: chat-api-config
            - secretRef:
                name: chat-api-secrets
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "1"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /ready
              port: 8000
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
---
apiVersion: v1
kind: Service
metadata:
  name: chat-api
  namespace: chat
spec:
  selector:
    app: chat-api
  ports:
    - port: 80
      targetPort: 8000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chat-api
  namespace: chat
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chat-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chat-api
  namespace: chat
spec:
  ingressClassName: traefik
  tls:
    - hosts: [chat.example.com]
      secretName: chat-example-tls
  rules:
    - host: chat.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chat-api
                port:
                  number: 80
```

Explanation of every object:

- `apiVersion` / `kind` / `metadata`: every Kubernetes object states which API group and version it belongs to, what type it is, and its name. `namespace: chat` places it in the `chat` namespace.
- **Namespace**: creates the partition.
- **ConfigMap** `data`: plain key/value configuration. Becomes environment variables through `envFrom`.
- **Secret** `stringData`: lets you write values in plain text in the file (Kubernetes stores them encoded). The value shown is a placeholder. **Never commit real secrets in a manifest**; use Sealed Secrets, SOPS, or External Secrets Operator that pull from a secrets manager.
- **Deployment**:
  - `replicas: 3` means run three identical pods.
  - `selector.matchLabels` and `template.metadata.labels` link the Deployment to its pods by label `app: chat-api`.
  - `strategy: RollingUpdate` with `maxSurge: 1, maxUnavailable: 0` means: during an update, start one extra new pod first and never take an old pod away until a new one is Ready. That is zero-downtime.
  - `containers[].image` is the same image string you use in Docker.
  - `envFrom` injects all ConfigMap and Secret keys as environment variables.
  - `resources.requests` is what the scheduler *reserves* for the pod (used to choose a node: `250m` = 250 millicores = a quarter of a CPU core; `256Mi` = 256 mebibytes). `resources.limits` is the hard cap (exceeding memory gets the container OOM-killed; exceeding CPU gets it throttled).
  - `readinessProbe`: "is this pod ready for traffic?" If it fails, the pod is removed from the Service's endpoints but not restarted.
  - `livenessProbe`: "is this pod alive?" If it fails repeatedly, the kubelet restarts the container.
  - `securityContext`: runs as a non-root user, forbids privilege escalation, makes the filesystem read-only (your app must write only to mounted volumes or `/tmp` via `emptyDir`).
- **Service**: gives a stable virtual IP and DNS name (`chat-api.chat.svc.cluster.local`, or just `chat-api` inside the namespace). `port: 80` is what clients use; `targetPort: 8000` is the container's port.
- **HPA**: keeps average CPU utilisation (relative to the *request*) near 70% by scaling between 3 and 10 replicas. Requires the metrics-server add-on, which most managed clusters provide.
- **Ingress**: tells the ingress controller (here named `traefik` as an example class) to route `https://chat.example.com/` to the Service, using the TLS certificate stored in secret `chat-example-tls` (commonly created by **cert-manager**, an add-on that automates Let's Encrypt certificates).

Apply it and watch:

```bash
kubectl apply -f chat-api.yaml
kubectl -n chat get pods
kubectl -n chat rollout status deployment/chat-api
kubectl -n chat rollout undo deployment/chat-api
```
- `kubectl apply -f` creates or updates everything in the file to match it (declarative).
- `kubectl -n chat get pods` lists pods in namespace `chat`; example output:

```
NAME                        READY   STATUS    RESTARTS   AGE
chat-api-7d9f8c6b5d-4kq2x   1/1     Running   0          2m
chat-api-7d9f8c6b5d-9mzt7   1/1     Running   0          2m
chat-api-7d9f8c6b5d-xk8vw   1/1     Running   0          2m
```
- `rollout status` blocks until the rollout finishes or fails; `rollout undo` rolls back to the previous ReplicaSet.

#### Helm in five minutes

Writing raw YAML for every environment gets repetitive. **Helm** is the package manager for Kubernetes: a **chart** is a folder of templated manifests plus a `values.yaml` of defaults.

```
chat-api-chart/
├── Chart.yaml          # name, version, description
├── values.yaml         # default settings: image tag, replicas, resources
└── templates/
    ├── deployment.yaml # YAML with placeholders like {{ .Values.image.tag }}
    ├── service.yaml
    └── ingress.yaml
```

```bash
helm install chat ./chat-api-chart -n chat --create-namespace \
  --set image.tag=1.4.2
helm upgrade chat ./chat-api-chart -n chat --set image.tag=1.4.3
helm rollback chat 1 -n chat
helm list -n chat
```
- `helm install <release-name> <chart>` renders the templates with values and applies them; the result is a **release**. `--set image.tag=1.4.2` overrides a value; `--create-namespace` creates the namespace if it does not exist.
- `helm upgrade` applies a new version; `helm rollback chat 1` returns to **revision 1**; `helm list` shows releases.
- Most third-party software (Prometheus, Postgres, cert-manager) ships as a Helm chart, so Helm is how you install it.

#### GitOps (Argo CD / Flux)

**GitOps** means the cluster's desired state lives in a git repository, and an agent *inside* the cluster (**Argo CD** or **Flux**, both CNCF projects) continuously pulls that repository and makes the cluster match it. To deploy, you merge a commit that changes the image tag; to roll back, you revert the commit. Benefits: full audit trail in git, no cluster credentials stored in CI, and automatic drift correction (if somebody manually changes the cluster, the agent undoes it).

#### Managed Kubernetes

Running your own control plane is hard. Cloud providers run it for you:

| Service | Cloud | Notes |
|---|---|---|
| **Amazon EKS** | AWS | Control plane managed; worker nodes as managed node groups, Fargate profiles, or Karpenter-managed auto-scaled nodes. |
| **Azure AKS** | Azure | Control plane free tier available; integrates with Entra ID, ACR, Azure Monitor. |
| **Google GKE** | GCP | Offers *Autopilot* mode (Google manages nodes; you pay per pod resource) as well as Standard mode. Often regarded as the most polished managed Kubernetes. |
| **Red Hat OpenShift** | Anywhere, or managed on the clouds | A Kubernetes distribution with a stricter security posture and extra developer tooling. |
| **k3s / k3d / kind / minikube** | Local, edge, labs | Lightweight Kubernetes for learning and testing. |

Even managed Kubernetes leaves you responsible for: cluster upgrades (several times a year), node patching, ingress/Gateway, certificates, secrets, monitoring, cost control, and security policy. Budget for a person or a team.

### 7.5 Serverless containers

"Serverless containers" means you hand the platform an image and it runs it, scales it and bills you, without you managing machines or a cluster.

| Service | Cloud | Model | Scale to zero | Notes |
|---|---|---|---|---|
| **Google Cloud Run** | GCP | Request-driven containers (and jobs); you set min/max instances | Yes | Very simple developer experience. Supports GPUs (NVIDIA L4 reported GA in 2026). Container must listen on the port given in `$PORT`. |
| **AWS ECS on Fargate** | AWS | ECS schedules your tasks on AWS-managed capacity | No (a service keeps at least its desired task count; scaling to zero tasks is possible via autoscaling policies/ scheduled actions but is not request-driven) | The standard "containers without Kubernetes" option on AWS. No native GPU on Fargate: use ECS on EC2 or EKS for GPUs. |
| **AWS App Runner** | AWS | Source/image to HTTPS service | n/a | **Closed to new customers on 30 April 2026**; AWS recommends **ECS Express Mode** for new workloads, and existing customers keep access with no new features. Do not start new projects on it. |
| **ECS Express Mode** | AWS | One API call provisions a Fargate-based ECS service, load balancer and auto scaling | no | AWS's recommended replacement for App Runner. (Newer service: read AWS's current docs before depending on specifics.) |
| **Azure Container Apps** | Azure | Serverless containers built on Kubernetes + KEDA (event-driven autoscaling) + Dapr; consumption and dedicated workload profiles | Yes (consumption plan) | Good for microservices and event-driven jobs. Supports GPU via workload profiles. |

Serverless limitations to know: cold starts when scaling from zero (seconds for heavy images, which makes **image size** matter), request timeouts, limited local disk (use object storage), no privileged containers, limited sidecar flexibility, and for long-running GPU model servers the economics can be worse than reserved instances.

Rough decision: **spiky or low traffic** and simple HTTP services lean towards Cloud Run or Container Apps (pay nothing when idle); **steady traffic** leans towards Fargate or a VM (cheaper per unit when always on). Pricing numbers change frequently and the comparisons I found disagreed with each other, so compute your own estimate with the vendors' current calculators.

### 7.6 PaaS (Platform as a Service)

A **PaaS** is a platform where you push code or an image and it handles everything else: builds, TLS, scaling, logs, databases as add-ons.

| Platform | Style | Good for |
|---|---|---|
| **Fly.io** | Runs your Docker image as lightweight VMs in many regions near users (`fly launch`, `fly deploy`) | Global low-latency apps, simple multi-region |
| **Render** | Git-connected web services and managed Postgres/Redis, Docker or native build | Small teams, predictable pricing, quick start |
| **Railway** | Git-connected deployments with a friendly UI; Docker or auto-detected builds | Prototypes, hobby and early-stage projects |
| **Heroku** | The original PaaS (container registry and `heroku.yml` support) | Existing Heroku users |

Trade-offs: very fast to start and little to operate, but higher cost at scale, less control over networking/security, vendor-specific features (lock-in is mild because you still ship a standard Docker image), and company/pricing changes happen (always check current pricing and free-tier status; I have not verified any specific price or free tier for this chapter).

---

## 8. Deployment strategies and rollbacks

A **deployment strategy** is the plan for replacing the old version with the new one. Goals: no user-visible downtime, a fast way back, and limited blast radius if the new version is bad.

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Recreate** | Stop old, start new | Simplest | Downtime. Fine for dev and internal tools, or when two versions can never run together (some database migrations) |
| **Rolling update** | Replace instances one by one (or in small batches), keeping most of them serving | No downtime, no extra full environment | Old and new versions run side-by-side for a while, so they must be compatible; rollback is another rolling update |
| **Blue/green** | Run the full new stack ("green") beside the old ("blue"), test it, then switch the router to green in one step | Instant switch and instant rollback (switch back) | Needs double capacity during the switch |
| **Canary** | Send a small percentage (1-5%) of traffic to the new version, watch metrics, then gradually increase | Limits damage; real-user testing | Needs traffic-splitting infrastructure and good metrics |
| **Shadow** | Copy real traffic to the new version without returning its responses | Safe load testing | Complex; side effects must be avoided |

```
 ROLLING                          BLUE/GREEN                         CANARY
 [v1][v1][v1][v1]                 router -> [v1 ×4]  (blue, live)    router -> 95% -> [v1 ×4]
 [v2][v1][v1][v1]                 [v2 ×4] (green, idle, tested)              5% -> [v2 ×1]
 [v2][v2][v1][v1]                 router -> [v2 ×4]  (switch!)       metrics OK? -> 25% -> 100%
 [v2][v2][v2][v2]                 [v1 ×4] kept for rollback          metrics bad? -> back to 0%
```

### 8.1 Zero-downtime on a single VM with Compose

Compose by itself recreates a changed container (stop old, start new), which causes a short gap. Two practical remedies:

**Option A: Blue/green behind Caddy.** Run two copies of the service, `api_blue` and `api_green`, only one of which Caddy sends traffic to.

```yaml
services:
  api_blue:
    image: ghcr.io/acme/chat-api:${BLUE_TAG:-none}
    profiles: ["blue"]
    healthcheck: { test: ["CMD","python","-c","import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"], interval: 5s, retries: 10 }
  api_green:
    image: ghcr.io/acme/chat-api:${GREEN_TAG:-none}
    profiles: ["green"]
    healthcheck: { test: ["CMD","python","-c","import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"], interval: 5s, retries: 10 }
```
- `profiles` ([Chapter 6](06-docker-compose.md)) mean a service only starts if you ask for that profile, so you can start the idle colour independently.
- `${BLUE_TAG:-none}` uses `none` when the variable is not set (a placeholder so Compose can parse the file).

Deploy procedure (script outline):

1. Determine the idle colour (say green) and start it with the new tag: `GREEN_TAG=sha-new docker compose --profile green up -d --wait`.
2. Smoke-test it directly (for example `docker compose exec caddy wget -qO- http://api_green:8000/health`).
3. Change the Caddyfile `reverse_proxy api_green:8000` and reload Caddy gracefully: `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`. A reload swaps configuration without dropping in-flight connections.
4. Keep blue running for some minutes. If metrics are bad, switch the Caddyfile back and reload (instant rollback).
5. Stop blue: `docker compose --profile blue stop api_blue`.

**Option B: Scale-out then scale-in.** With a proxy that discovers containers (Traefik) you can run `docker compose up -d --no-deps --scale api=2 --no-recreate api` to add a second instance, wait for health, then remove the old one. A community plugin called **docker-rollout** automates this pattern; I did not verify its current maintenance status, so evaluate it before relying on it.

**Option C: Use an orchestrator.** Kubernetes `RollingUpdate`, Swarm `update_config`, ECS rolling/blue-green deployments (with CodeDeploy), and Cloud Run revisions with traffic splitting all give you this without scripts. For example, Cloud Run lets you route 5% of traffic to a new **revision** and 95% to the old one, then shift gradually.

### 8.2 Requirements for zero-downtime to actually work

- **Graceful shutdown.** When Docker stops a container it sends **SIGTERM** (a polite "please exit" signal), waits 10 seconds by default (`stop_grace_period` / `--stop-timeout`), then sends **SIGKILL** (immediate death). Your app must catch SIGTERM, stop accepting new requests, finish in-flight ones, and exit. This requires the app to be PID 1 or run under a proper init (use `init: true` in Compose or `docker run --init`) and to use the **exec form** of `CMD` ([Chapter 4](04-dockerfile-deep-dive.md)) so signals reach it.
- **Readiness.** Traffic must go only to instances that are ready (Section 10).
- **Backward-compatible database migrations.** During a rolling or blue/green deploy, v1 and v2 both talk to the same database. Use the **expand/contract** pattern: first *add* the new column (compatible with v1), deploy v2 that uses it, and only later *remove* the old column.
- **Idempotent, stateless services** (Section 15).

### 8.3 Rollbacks

A rollback is only as good as your preparation.

| Platform | Command |
|---|---|
| Compose + script | Re-run `deploy.sh` with the previous tag (stored in `.previous_tag`) |
| Swarm | `docker service update --rollback SERVICE` |
| Kubernetes | `kubectl rollout undo deployment/NAME` (or revert the commit in GitOps) |
| Helm | `helm rollback RELEASE REVISION` |
| Cloud Run | Send 100% traffic to the previous revision |
| ECS | Update the service to the previous task definition revision |

Rollback prerequisites:

1. Old images must still exist in the registry (retention policy exempts deployed versions).
2. Configuration must be versioned with the image or compatible with it.
3. Database changes must be reversible or backward-compatible; **you cannot un-drop a column.**
4. Practise: run a rollback drill every quarter.

**Common mistakes (deployments)**

- No health check, so "deployed" just means "started."
- Shipping a destructive database migration in the same release as the code that needs it.
- Running only one replica and calling it highly available.
- Stop timeout too short for the app's shutdown, causing dropped requests.
- Rollback relying on `latest`.

---

## 9. Configuration and secrets in production

### 9.1 The Twelve-Factor App, the useful parts

The **Twelve-Factor App** is a set of guidelines (from the Heroku founders, still widely followed) for building apps that run well on modern platforms. The ones that matter most for containers:

| Factor | Meaning for you |
|---|---|
| **III. Config** | Store configuration in the *environment*, not in the code or image. The same image must run in dev, staging and prod; only the environment differs. |
| **IV. Backing services** | Treat databases, caches and queues as attached resources reached through a URL in config. Swapping a local Postgres for a managed one is a config change. |
| **VI. Processes** | Run as stateless processes; anything that must persist goes to a backing service. |
| **IX. Disposability** | Start fast and shut down gracefully, so instances can be created and destroyed at will. |
| **X. Dev/prod parity** | Keep environments as similar as possible. Containers make this easy. |
| **XI. Logs** | Treat logs as an event stream: write to stdout/stderr and let the platform collect them. |

### 9.2 Environment variables: fine, with caveats

Environment variables are the lingua franca of container config.

```bash
docker run -e LOG_LEVEL=info -e API_BASE_URL=https://api.example.com chat-api:1.4.2
```
- `-e NAME=value` sets a variable inside the container.

Risks when using them for secrets:

- `docker inspect CONTAINER` prints all environment variables to anyone with Docker access.
- Child processes inherit them, and crash dumps or debug endpoints (`/env`, framework error pages) may expose them.
- They can leak into logs if the app prints its config at startup.
- Anything baked in with `ENV` or `ARG` in a Dockerfile is **stored in the image layers forever**, visible with `docker history`. Never put secrets in a Dockerfile.

### 9.3 Docker secrets / file-based secrets

A better pattern for passwords and keys is to mount them as **files**, typically read-only, under `/run/secrets/`.

```yaml
services:
  api:
    image: ghcr.io/acme/chat-api:1.4.2
    secrets:
      - db_password
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```
- The top-level `secrets:` block declares a secret whose content comes from a local file. In the service, `secrets: - db_password` mounts it at `/run/secrets/db_password` (an in-memory-backed read-only file in Swarm; a bind-mounted file in plain Compose).
- `DB_PASSWORD_FILE` follows a widespread convention (used by the official Postgres and MySQL images) where the app reads the *path* from an environment variable and loads the password from that file. The app's code (or entrypoint script) must support reading the file.
- The source file `./secrets/db_password.txt` must be created with tight permissions (`chmod 600`), kept out of git (`.gitignore`), and delivered to the server by your secret tooling.

### 9.4 Secrets managers

For anything serious, secrets live in a dedicated service that provides encryption, access control, rotation and audit logs:

| Tool | Where | Notes |
|---|---|---|
| **AWS Secrets Manager** / **SSM Parameter Store** | AWS | ECS and EKS can inject values into containers at start. Secrets Manager supports automatic rotation. |
| **Azure Key Vault** | Azure | Container Apps and AKS can reference secrets. |
| **Google Secret Manager** | GCP | Cloud Run can mount secrets as env vars or files. |
| **HashiCorp Vault** (or its open-source fork OpenBao) | Anywhere | Dynamic, short-lived credentials (for example a database user that exists for one hour). |
| **SOPS** + age/KMS | Anywhere | Encrypts secret *files* so the encrypted version can be committed to git; decrypt at deploy time. |
| **External Secrets Operator / Secrets Store CSI Driver** | Kubernetes | Syncs secrets from the managers above into pods. |
| **Doppler, 1Password Secrets Automation, Infisical** | SaaS | Developer-friendly managed options. |

Key principles:

- **Least privilege:** each service can read only its own secrets.
- **Rotation:** assume secrets will leak someday; make rotating them routine and automated.
- **No long-lived cloud keys in CI.** Use **OIDC federation**: GitHub Actions requests a short-lived token (`id-token: write` permission) and the cloud trusts it for a specific repository and branch. No access key is stored in GitHub at all.
- **Scan for leaked secrets** in git history with tools like `gitleaks` or `trufflehog`, and enable GitHub's push protection.
- Remember the golden rule: `.env` files and keys never go into git ([Chapter 13](13-interview-questions-roadmap-glossary.md) covers this in the portfolio guide).

### 9.5 Configuration per environment with Compose

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
```
- Multiple `-f` files are merged left to right; later files override earlier ones. `config` prints the **final merged configuration** (with variables substituted) without running anything, an excellent way to review what production will actually get.

**Common mistakes (config/secrets)**

- Baking secrets into images with `ENV`/`ARG`/`COPY .env`.
- Committing `.env` files.
- One shared "admin" secret used by every service.
- Never rotating anything.
- Printing configuration (including secrets) at startup.

---

## 10. Health checks: liveness vs readiness

A **health check** is a small command or HTTP request the platform runs periodically to ask "are you okay?" Without one, the platform only knows whether your process *exists*, not whether it *works* (a deadlocked server is still a running process).

There are three different questions, and Kubernetes gives each its own probe. Even if you do not use Kubernetes, design your app with all three in mind.

| Probe | Question | If it fails | Typical check |
|---|---|---|---|
| **Liveness** | "Are you stuck or broken beyond repair?" | The container is **restarted** | A cheap `/health` endpoint that returns 200 if the process can respond. **Do not check the database here**, or a database outage will restart every app instance and make things worse. |
| **Readiness** | "Can you handle traffic right now?" | The instance is **removed from load balancing** (not restarted) | `/ready` that verifies dependencies (database reachable, cache warmed, migrations done). |
| **Startup** | "Have you finished starting?" | Liveness/readiness are paused until it passes; if it never passes the container is restarted | Used for slow-starting apps (loading a large ML model can take minutes). |

```
 request flow                         health logic
 -------------                        ------------
 Load balancer -> only Ready pods     readiness fails -> pod removed from rotation (no restart)
 kubelet / Docker -> restarts         liveness fails  -> container restarted
```

### Docker's built-in HEALTHCHECK

In a Dockerfile:

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
```
- `--interval=30s`: run the check every 30 seconds. `--timeout=3s`: a check taking longer than this counts as a failure. `--start-period=40s`: failures during the first 40 seconds do not count (grace time for startup). `--retries=3`: the container is marked `unhealthy` after three consecutive failures.
- `CMD ...`: the command; exit code 0 means healthy, 1 means unhealthy.
- `|| exit 1` makes sure any failure becomes exit code 1.

See the state:

```bash
docker ps
```
```
CONTAINER ID   IMAGE              STATUS                    NAMES
a1b2c3d4e5f6   chat-api:1.4.2     Up 2 minutes (healthy)    chat-api
```
- The `STATUS` column shows `(healthy)`, `(unhealthy)`, or `(health: starting)`.

Important: **plain Docker does not restart an unhealthy container.** It only labels it. Compose with `restart:` also restarts only on process exit. Docker Swarm and Kubernetes (and ECS) *do* replace unhealthy tasks. On a single VM, pair the healthcheck with `depends_on: condition: service_healthy`, `docker compose up --wait`, and an external monitor that alerts you. (Some people add a small "autoheal" sidecar that restarts unhealthy containers; it needs the Docker socket, so weigh the risk.)

**Common mistakes (health checks)**

- Healthcheck uses `curl` in an image that does not contain `curl`, so the container is always `unhealthy`.
- Liveness checks the database, producing restart storms during a database outage.
- No `start_period` for slow starters, so the app is killed while still loading.
- A health endpoint that always returns 200 without testing anything meaningful for readiness.

---

## 11. Resource limits and the OOM killer

By default a container may use *as much CPU and memory as the host has*. One misbehaving container can starve everything else on the machine.

```bash
docker run -d --name api \
  --memory=512m --memory-swap=512m \
  --cpus=1.5 \
  --pids-limit=200 \
  chat-api:1.4.2
```
- `--memory=512m` sets a hard memory cap of 512 MiB. If the container tries to use more, the kernel's **OOM killer** (Out Of Memory killer, the kernel component that kills processes when memory runs out) terminates it.
- `--memory-swap=512m` sets total memory plus swap to the same value, meaning **no swap** (swapping makes latency unpredictable).
- `--cpus=1.5` allows at most one and a half CPU cores' worth of time.
- `--pids-limit=200` caps the number of processes/threads, protecting against fork bombs.

In Compose:

```yaml
services:
  api:
    mem_limit: 512m
    cpus: 1.5
    pids_limit: 200
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: 512M
        reservations:
          memory: 256M
```
- `deploy.resources.limits` is the Compose-spec way and is honoured by current `docker compose`. `reservations` is a soft guarantee used for scheduling in orchestrators.

### Recognising an OOM kill

```bash
docker inspect api --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
```
- Prints something like `true 137`. Exit code **137** = 128 + 9 (signal 9, SIGKILL). `OOMKilled=true` confirms the memory limit was the cause. Also look at `dmesg | grep -i oom` on the host. More on this in [Chapter 12](12-troubleshooting-and-cheatsheet.md).

### Sizing guidance

1. Measure first: `docker stats` shows live CPU/memory per container.
2. Set the memory limit comfortably above the observed peak (20-50% headroom). Memory limits are hard; CPU limits throttle.
3. In Kubernetes, set `requests` (what you need normally) and `limits` (maximum). Setting memory request equal to limit gives the most predictable behaviour. Many teams do not set CPU limits (only requests) to avoid unnecessary throttling; this is debated, so understand your platform before choosing.
4. For Python/Java/Node: tune the runtime to the limit (JVM `-XX:MaxRAMPercentage`, Node `--max-old-space-size`, Gunicorn worker counts), or the runtime may believe it has the whole host's memory.
5. For ML inference, model size dictates memory: a model that needs 6 GB of RAM needs a limit above that plus overhead ([Chapter 8](08-docker-for-ai-and-ml.md)).

**Common mistakes:** no limits at all on shared hosts; limit set below real peak (random 137 exits); limits that ignore the runtime's own memory model; forgetting that all containers' limits together must fit the machine.

---

## 12. Logging

### 12.1 The container logging model

Containers should not write log *files*. They should write to **stdout** (standard output) and **stderr** (standard error). Docker captures these two streams and hands them to a **logging driver** (a plug-in that decides where logs go). You read them with `docker logs`.

```
 app prints to stdout/stderr -> Docker daemon captures -> logging driver -> destination
                                                         json-file / local -> files on host
                                                         journald          -> systemd journal
                                                         awslogs, gcplogs, splunk, fluentd, gelf, syslog -> remote services
```

### 12.2 The default driver has a trap

The default driver is **`json-file`**, which writes one JSON line per log message to `/var/lib/docker/containers/<id>/<id>-json.log`. **By default it never rotates.** A chatty container will eventually fill the disk and take the whole host down. This is one of the most common production incidents, and the fix takes four lines.

The supported drivers (per the Docker docs) include `none`, `local`, `json-file`, `syslog`, `journald`, `gelf`, `fluentd`, `awslogs`, `splunk`, `etwlogs` (Windows only), `gcplogs`, plus plug-ins.

| Driver | Where logs go | Notes |
|---|---|---|
| `json-file` | JSON files on the host | Default, **no rotation unless you configure it**. Works with `docker logs`. |
| `local` | Compact binary files on the host | **Rotates by default** (20 MB x 5 files), lower overhead; Docker recommends it for most non-Kubernetes uses. Works with `docker logs`. |
| `journald` | systemd journal | Good if you already centralise the journal. |
| `syslog` | A syslog server | Traditional. |
| `fluentd` / `gelf` | Fluentd/Fluent Bit or Graylog/Logstash | For ELK-style pipelines. |
| `awslogs` | Amazon CloudWatch Logs | Natural on AWS. |
| `gcplogs` | Google Cloud Logging | Natural on GCP. |
| `splunk` | Splunk HTTP Event Collector | Enterprise Splunk shops. |
| `none` | Nowhere | Disables logs (and `docker logs`). |

Caveat: with some remote drivers `docker logs` no longer works for that container (newer Docker Engine versions use "dual logging" so that `docker logs` keeps working, but verify on your version).

### 12.3 Rotation settings you should always set

Per container:

```bash
docker run -d --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 chat-api:1.4.2
```
- `--log-driver json-file` picks the driver. `--log-opt max-size=10m` rotates the file when it reaches 10 megabytes. `--log-opt max-file=3` keeps at most 3 files. So the cap is about 30 MB per container.

Per Compose service:

```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```
- Values must be quoted strings.

For every container on the host, in `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```
- After editing, restart the daemon (`sudo systemctl restart docker`). **The new defaults apply only to containers created afterwards**; existing containers keep their old settings until recreated.

Optional: `"mode": "non-blocking"` with `"max-buffer-size": "4m"` in `log-opts` stops a slow log destination from blocking your application's writes, at the price of dropping messages if the buffer fills. Default mode is `blocking`.

### 12.4 Good logging practice

- **Structured logs** (JSON, one object per line with fields like `level`, `timestamp`, `request_id`) are searchable; free-form text is not.
- Include a **correlation/request ID** passed through all services.
- **Never log secrets or personal data** (tokens, passwords, full prompts that may contain private user data, which matters for AI chat apps).
- Use log levels sensibly; set `LOG_LEVEL=info` in production, `debug` only temporarily.
- Time in UTC.

### 12.5 Centralised logging

On one VM `docker logs` is enough. With several machines you need logs in one place so you can search "everything that happened to request abc123."

| Stack | Pieces | Notes |
|---|---|---|
| **Grafana Loki** | Loki (storage, indexes only labels, cheap) + a collector + Grafana to view | Collector options: **Grafana Alloy** (the current recommended agent), or the Docker log-driver plug-in. The older **Promtail** agent was deprecated in favour of Alloy and reached end of life in 2026 (reported; verify on Grafana's site). |
| **ELK / EFK** | Elasticsearch + Logstash (or Fluentd/Fluent Bit) + Kibana | Powerful full-text search; heavier to run. OpenSearch is the open-source fork. |
| **CloudWatch Logs** | `awslogs` driver or the CloudWatch agent / Fluent Bit | Native on AWS; ECS/Fargate integrates directly. |
| **Cloud Logging** (GCP), **Azure Monitor / Log Analytics** | Built into Cloud Run/GKE and Container Apps/AKS | Zero setup on those platforms. |
| **Datadog, Splunk, New Relic, Elastic Cloud** | Commercial SaaS | Fast to adopt, watch costs: log volume is billed. |

Lightweight collector: **Fluent Bit** runs as a container, reads Docker's log files and ships them anywhere. In Kubernetes, a collector typically runs as a **DaemonSet** (one pod per node) reading `/var/log/containers`.

Retention and cost: logs are the most expensive observability data. Decide retention (for example 14-30 days hot, longer cold or archived for compliance), sample noisy debug logs, and drop health-check request logs.

**Common mistakes (logging):** unrotated `json-file`; logs written to files inside the container (lost on removal, fills the writable layer); logging secrets; no timestamps or IDs; dropping logs on the floor in a crash because the destination was unreachable and the driver was in blocking mode.

---

## 13. Monitoring, alerting and tracing

Logs tell you *what happened*. **Metrics** are numbers over time (CPU, requests per second, error rate, latency) that tell you *how healthy* you are and let you alert before users complain. **Traces** follow one request across services.

```
 containers/host --> exporters --> Prometheus (scrapes & stores) --> Grafana (dashboards)
                                              |
                                              +--> Alertmanager --> email / Slack / PagerDuty
```

### 13.1 The standard open-source stack

| Component | Role |
|---|---|
| **Prometheus** | A time-series database that **pulls** ("scrapes") metrics from HTTP endpoints every few seconds and lets you query them with PromQL. |
| **cAdvisor** (Container Advisor, from Google) | A container exposing per-container CPU, memory, network and disk metrics for every container on the host. |
| **node_exporter** | Exposes host-level metrics (CPU, memory, disk, network) of the VM itself. |
| **Docker Engine metrics endpoint** | Set `"metrics-addr": "127.0.0.1:9323"` in `daemon.json` to expose daemon metrics (if it does not come up on your version, check the Docker docs for any additional flags required). |
| **Grafana** | Dashboards and visualisation; also does alerting. |
| **Alertmanager** | Deduplicates, groups and routes alerts to people. |
| **Blackbox exporter / Uptime Kuma / external pingers** | Checks from outside that your public URL works. |

A starter `compose.monitoring.yaml`:

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    expose: ["8080"]

  node_exporter:
    image: prom/node-exporter:latest
    restart: unless-stopped
    pid: host
    volumes:
      - /:/host:ro,rslave
    command: ["--path.rootfs=/host"]
    expose: ["9100"]

  prometheus:
    image: prom/prometheus:latest
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom_data:/prometheus
    ports: ["127.0.0.1:9090:9090"]

  grafana:
    image: grafana/grafana:latest
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
    ports: ["127.0.0.1:3000:3000"]

volumes:
  prom_data:
  grafana_data:
```
- Every service uses `latest` here **only to keep the example short**; in real use, pin versions (the guidance of Section 4 applies to third-party images too). Image locations (`gcr.io/cadvisor/cadvisor`, `prom/prometheus`, etc.) should be double-checked against each project's docs, since cAdvisor's image location has changed in the past.
- `cadvisor` mounts host directories **read-only** (`:ro`) so it can see containers' resource usage. This reveals a lot about the host; do not expose its port publicly (`expose` keeps the port internal to the Compose network).
- `node_exporter` uses `pid: host` and mounts `/` read-only to see the host's real metrics.
- `prometheus` is bound to `127.0.0.1` so it is not reachable from the internet; reach it through an SSH tunnel or the reverse proxy with authentication.
- Volumes persist the time-series data and Grafana's dashboards.

`prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
  - job_name: node
    static_configs:
      - targets: ["node_exporter:9100"]
  - job_name: api
    metrics_path: /metrics
    static_configs:
      - targets: ["api:8000"]
```
- `scrape_interval: 15s` collects metrics every 15 seconds. Each `job_name` lists the targets (host:port) to scrape; `api` assumes your app exposes `/metrics` (the Python `prometheus-client` and most frameworks make this easy).

### 13.2 What to measure: the golden signals

Google's SRE book defines four **golden signals**:

| Signal | Meaning | Example alert |
|---|---|---|
| **Latency** | How long requests take (watch the 95th/99th percentile, not just the average) | p95 > 800 ms for 5 minutes |
| **Traffic** | Request rate | Sudden drop to zero (outage) or 10x spike |
| **Errors** | Rate of failed requests (5xx) | Error rate > 2% for 5 minutes |
| **Saturation** | How "full" a resource is: CPU, memory, disk, connection pools | Disk > 85%, memory > 90% of limit |

Also alert on: container restart count rising, container `unhealthy`, certificate expiring within 14 days, backup job failed, queue depth growing. For AI apps add: tokens per second, time-to-first-token, GPU utilisation and GPU memory, queue depth, model-load time.

**Alert wisely:** every alert should be *actionable* and *urgent*. Too many noisy alerts cause "alert fatigue" and the real one gets ignored. Page humans for symptoms users feel (errors, latency, downtime); send causes (high CPU) to a chat channel.

### 13.3 Tracing and OpenTelemetry

A **trace** records the path of one request through many services (API -> cache -> model server -> database) as a tree of timed **spans**, so you can see exactly where the 4 seconds went. **OpenTelemetry (OTel)** is the vendor-neutral, CNCF standard for generating and exporting traces, metrics and logs. You add the OTel SDK/auto-instrumentation to your app, send data to an **OTel Collector** (a small service that receives, processes and forwards telemetry), and the collector forwards it to Tempo, Jaeger, Datadog, Honeycomb, or any other backend. Start with traces for the request path and add metrics/logs later. For LLM apps, OTel-based tracing tools also capture prompts, token counts and latencies per call.

### 13.4 Managed alternatives

You can skip running monitoring yourself: Grafana Cloud, Datadog, New Relic, AWS CloudWatch/Container Insights, Azure Monitor, Google Cloud Monitoring. You trade money for time. Small teams often do best with a hosted solution plus a free external uptime monitor.

---

## 14. Persistence and backups

### 14.1 Containers are disposable, data is not

A container's writable layer disappears when the container is removed ([Chapter 5](05-storage-and-networking.md)). Anything you cannot afford to lose must live in a **volume**, a bind mount, or, better, a service outside the container platform.

### 14.2 Why databases often live outside containers in production

Running Postgres in a container is perfectly feasible, and many teams do it for development and tests. In production, many teams prefer a **managed database** (Amazon RDS/Aurora, Azure Database for PostgreSQL, Google Cloud SQL, or hosted providers like Neon, Supabase, Crunchy Bridge) because:

| Concern | Self-run in a container | Managed database |
|---|---|---|
| Backups & point-in-time recovery | You build and test it | Built in |
| High availability / failover | You build it | Checkbox |
| Patching & upgrades | You do it | Provider does it |
| Monitoring | You assemble it | Included |
| Storage resizing | Manual | Automatic |
| Cost | Cheaper in money | Cheaper in people-time |
| Control, extensions, tuning | Total | Limited by provider |

Orchestrators are designed for **stateless** workloads; stateful workloads need care (storage attachment, ordered startup, safe failover). Kubernetes has StatefulSets and operators (CloudNativePG, Zalando Postgres Operator) that do this well, but they require expertise. For a vector database or a cache in an AI app, the same logic applies: managed offering (Pinecone, managed Redis, pgvector on managed Postgres) or a carefully backed-up volume.

**When self-hosting in a container is fine:** small apps where occasional downtime is acceptable, on-prem or air-gapped systems, cost-sensitive projects, and caches (Redis used purely as a cache can be rebuilt).

### 14.3 Backing up what you do run

Follow the **3-2-1 rule**: three copies of your data, on two different kinds of media/systems, one of them off-site.

**Database dump (logical backup):**

```bash
docker compose exec -T db pg_dump -U chat -Fc chat > /backups/chat-$(date +%F).dump
```
- `docker compose exec -T db ...` runs a command inside the running `db` service; `-T` disables the pseudo-terminal so output redirection works in scripts.
- `pg_dump -U chat -Fc chat` dumps database `chat` as user `chat` in **custom format** (`-Fc`: compressed, restorable selectively with `pg_restore`).
- `> /backups/chat-$(date +%F).dump` writes the output to a dated file on the host (`date +%F` prints `2026-10-23`).

Restore:

```bash
docker compose exec -T db pg_restore -U chat -d chat --clean --if-exists < /backups/chat-2026-10-23.dump
```
- `pg_restore ... --clean --if-exists` drops existing objects before recreating them; `< file` feeds the dump to the command's standard input.

**Volume backup (file-level):**

```bash
docker run --rm -v chat_pgdata:/data:ro -v /backups:/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
```
- Starts a temporary Alpine container that mounts the volume `chat_pgdata` read-only at `/data` and the host's `/backups` at `/backup`, then creates a compressed tarball. Note: copying a live database's files can produce an inconsistent backup; for databases prefer `pg_dump` or stop the database briefly, or use filesystem snapshots plus the database's own backup tooling.

**Ship it off the machine:** use `rclone`, `restic` or `borg` to push encrypted backups to object storage (S3, Azure Blob, GCS, Backblaze B2). Schedule with a systemd timer or cron:

```bash
0 2 * * * /opt/chat-api/backup.sh >> /var/log/chat-backup.log 2>&1
```
- A cron line: at 02:00 every day run the backup script and append all output to a log.

Backup rules that save careers:

1. **A backup you have never restored is a hope, not a backup.** Schedule restore tests.
2. Define **RPO** (Recovery Point Objective: how much data you can afford to lose, e.g. 15 minutes) and **RTO** (Recovery Time Objective: how long you can be down, e.g. 1 hour). These drive your design.
3. Encrypt backups and protect the keys separately.
4. Alert if a backup does not run or is suspiciously small.
5. Keep backups in a different account/credentials than production, to survive ransomware or a compromised admin.

Named volumes live under `/var/lib/docker/volumes`. Treat that path as the daemon's private territory; use the commands above rather than copying files by hand.

---

## 15. Scaling

### 15.1 Vertical vs horizontal

| | Vertical scaling ("scale up") | Horizontal scaling ("scale out") |
|---|---|---|
| What | A bigger machine | More copies |
| Limit | The biggest machine you can buy | Practically very high |
| Resilience | Single point of failure | Survives losing a copy |
| Complexity | Almost none | Needs stateless design and a load balancer |

### 15.2 Designing for horizontal scale: statelessness

If any instance can handle any request, you can run N instances behind a **load balancer** (a component that spreads requests across instances). That requires **stateless** services: no user session in process memory, no files written to local disk that other instances need. Store state in a shared backing service (Redis for sessions and caches, Postgres for records, object storage for files).

```
              +--> api #1 --+
 users -> LB -+--> api #2 --+--> Redis (sessions/cache)  +--> Postgres
              +--> api #3 --+
```

**Sticky sessions** (also "session affinity": the load balancer always sends the same user to the same instance) are a caveat. They are a workaround for apps that keep state in memory, but they cause uneven load, make deployments and scale-down harder (users lose their session when "their" instance goes away), and hide design problems. Prefer a shared session store. Exceptions where affinity is genuinely useful: WebSocket connections that must stay on one instance for their lifetime, and some caching tricks (for example, routing a conversation to the replica holding its KV-cache in an LLM serving system).

### 15.3 How to scale on each platform

| Platform | How |
|---|---|
| Compose | `docker compose up -d --scale api=3` (needs the reverse proxy to balance across replicas; do not publish a fixed host port on a service you scale) |
| Swarm | `docker service scale chat_api=5` |
| Kubernetes | Change `replicas`, or let the **HPA** scale on CPU/memory/custom metrics; **cluster autoscaler** or **Karpenter** adds nodes; **KEDA** scales on queue length |
| Cloud Run / Container Apps | Automatic by concurrent requests; set min/max instances |
| ECS | Service auto scaling with target tracking policies |

Watch the other bottleneck: scaling the API from 3 to 30 replicas can overwhelm the database with connections. Use a connection pooler (PgBouncer, RDS Proxy) and limits.

**Scaling AI inference** is different: model servers are heavy (GBs of weights, slow to load, GPU-bound), so you scale with care (pre-pulled images, warm pools, batching, queueing) as discussed in [Chapter 8](08-docker-for-ai-and-ml.md).

---

## 16. Docker daemon production configuration (`daemon.json`)

The **Docker daemon** (`dockerd`) is the background service that does the real work. On Linux you configure it in `/etc/docker/daemon.json` (create it if it does not exist). It must be **valid JSON**, or the daemon will refuse to start.

A reasonable production baseline:

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "5"
  },
  "live-restore": true,
  "default-address-pools": [
    { "base": "10.200.0.0/16", "size": 24 }
  ],
  "registry-mirrors": ["https://mirror.internal.example.com"],
  "data-root": "/var/lib/docker",
  "metrics-addr": "127.0.0.1:9323",
  "default-ulimits": {
    "nofile": { "Name": "nofile", "Hard": 65536, "Soft": 65536 }
  }
}
```

| Key | What it does and why |
|---|---|
| `"log-driver": "local"` | Use the compact `local` driver by default. Or `"json-file"` if you prefer plain-text JSON. |
| `"log-opts"` | Rotation: at most 5 files of 10 MB each per container. **The single most valuable setting here.** Quote the numbers. |
| `"live-restore": true` | Containers **keep running while the Docker daemon is stopped or restarted** (for upgrades or crashes). When the daemon returns it re-attaches. Limits: works for patch-level updates (not major version upgrades), not for Swarm mode, not for Windows containers, and during a long daemon outage logs can fill a small pipe buffer and block the container. You can apply it with a reload (`sudo systemctl reload docker` or `kill -SIGHUP $(pidof dockerd)`). |
| `"default-address-pools"` | The IP ranges Docker uses when it creates new networks. The default pools (172.17.0.0/16 and friends, plus 192.168.x.x ranges) often **collide with corporate VPNs or cloud VPC ranges**, causing mysterious unreachable hosts. Here every new network gets a /24 carved out of 10.200.0.0/16 (254 networks). Choose a range that does not overlap your real networks. Changing it affects only networks created later. |
| `"registry-mirrors"` | Docker Hub pull-through caches to try first (Docker Hub only). Dodges rate limits and speeds up pulls. |
| `"data-root"` | Where Docker stores images, containers and volumes. Default `/var/lib/docker`. Move it to a bigger, faster disk (for example a dedicated volume mounted at `/data/docker`). Moving it requires stopping Docker and copying the data (`rsync -aP /var/lib/docker/ /data/docker/`), then updating this key. |
| `"metrics-addr"` | Exposes Prometheus metrics for the daemon, bound to localhost only. |
| `"default-ulimits"` | Default per-process limits, for example open files (`nofile`), for every container. Some databases and proxies need higher values. |

Apply and verify:

```bash
sudo systemctl restart docker
docker info | grep -E "Logging Driver|Live Restore|Docker Root Dir"
```
- Restart applies the config (changes to log settings only affect new containers). `docker info` prints the daemon's effective settings; example output:

```
 Logging Driver: local
 Docker Root Dir: /var/lib/docker
 Live Restore Enabled: true
```

Validate the JSON before restarting:

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
```
- `dockerd --validate` checks the configuration file for errors without starting the daemon. Prints `configuration OK` on success.

Other settings worth knowing: `"userns-remap"` (map container root to an unprivileged host user), `"no-new-privileges": true` (default for all containers), `"icc": false` (disable unrestricted inter-container communication on the default bridge), `"features": {"containerd-snapshotter": true}` (the containerd image store, which became the default for new installs in Docker Engine 29), and `"insecure-registries"` (avoid; it disables TLS verification). Rootless mode and the full security hardening list are in [Chapter 10](10-security.md).

**Common mistakes:** invalid JSON (trailing comma) that stops the daemon; forgetting that `daemon.json` log options do not retroactively change running containers; address pool overlap with the VPN; editing without a backup of the previous file.

---

## 17. Disk management and cleanup policies

Docker keeps everything until told otherwise: old images, stopped containers, build cache, unused volumes, logs. Disks fill up silently and the first symptom is `no space left on device` in production.

Inspect usage:

```bash
docker system df
```
- Shows how much space images, containers, local volumes and build cache use. Example output:

```
TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
Images          23      6        8.41GB    5.92GB (70%)
Containers      9       6        152.3MB   12.6MB (8%)
Local Volumes   11      4        3.2GB     1.1GB (34%)
Build Cache     87      0        4.7GB     4.7GB
```
- `RECLAIMABLE` is what a prune would free.

Cleanup commands (careful, these delete things; read the explanation before running):

| Command | What it removes | Risk |
|---|---|---|
| `docker image prune` | **Dangling** images (untagged leftovers) | Very low |
| `docker image prune -a --filter "until=168h"` | All images **not used by a container** and older than 7 days | Low if you can re-pull; would delete rollback images that are not currently used |
| `docker container prune` | Stopped containers | Low |
| `docker builder prune --filter "until=72h"` | Build cache older than 3 days | Low (builds get slower once) |
| `docker volume prune` | Volumes not used by any container | **High: this can delete database data** if a stack is temporarily stopped |
| `docker system prune` | Stopped containers, unused networks, dangling images, (with `-a` all unused images), build cache | Medium; **never add `--volumes` on a server with data** |

Policies for production hosts:

1. **Never schedule `docker volume prune` or `docker system prune --volumes` on production.** Volumes of a stopped stack look "unused."
2. Schedule a weekly `docker image prune -a --filter "until=168h"` and `docker builder prune --filter "until=168h"` (a systemd timer or cron), *after* making sure rollback images are in a registry (they can be re-pulled).
3. Put `data-root` on a separate disk so Docker cannot fill the root filesystem.
4. Alert at 80% disk usage.
5. Make sure logs rotate ([Section 12](#12-logging)).
6. On build servers (CI runners you manage), prune aggressively and automatically; use ephemeral runners where possible, since they are discarded after each job.

---

## 18. Upgrading Docker Engine

Docker Engine releases frequently (the local machine used while writing this guide runs 29.9.2). Security fixes matter; run a supported version.

Version notes worth knowing (from the official Engine 29 release notes): 29.0.0 raised the **minimum API version** the daemon accepts to 1.44 (clients and tools built for Docker older than v25 must be updated; some third-party tools like Traefik and Portainer needed updates), made the **containerd image store** the default for new installs, removed Docker Content Trust from the CLI, and deprecated cgroup v1. One secondary source says a later 29.x release relaxed the API floor; I could not confirm that from primary text, so always ask your daemon with `docker version`.

A safe upgrade procedure:

1. **Read the release notes** for breaking changes (API version, storage driver, deprecations).
2. **Test in staging** on the same OS and storage configuration.
3. **Back up** `/etc/docker/daemon.json` and any state; note running containers (`docker ps`).
4. **Upgrade via the package manager** (so you receive the Docker apt/dnf repository's signed packages):
   ```bash
   sudo apt update
   sudo apt install --only-upgrade docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
   - `--only-upgrade` upgrades the named packages and never installs them if absent. The listed packages are the engine, the CLI, the container runtime, and the Buildx and Compose plug-ins.
5. With `live-restore` enabled and a patch-level upgrade, containers keep running. Otherwise expect all containers to restart; schedule a maintenance window or drain traffic first (move the load balancer to another node).
6. **Verify:** `docker version`, `docker info`, `docker ps` (all expected containers up and healthy), smoke-test the app.
7. **Roll out one host at a time** in a fleet; keep one node on the old version until the others look good.

Pin versions in automation. For example, `apt-mark hold docker-ce` prevents surprise upgrades from unattended-upgrades, and then you upgrade on purpose.

**Docker Desktop** (developer machines) is different: it auto-updates, and its settings are managed through the app or admin tools.

---

## 19. Cost and capacity planning

Containers do not make compute free, but they let you pack workloads efficiently.

### 19.1 Where the money goes

| Cost driver | Notes |
|---|---|
| **Compute** | VMs/nodes/Fargate vCPU-seconds. The biggest line. Right-size using real metrics from Section 13 instead of guessing. |
| **GPU** | Often 10-100x CPU cost per hour; idle GPUs burn money. Use autoscaling to zero where practical, spot/preemptible instances for batch work, batching and quantised models for inference. |
| **Storage** | Registry storage, volumes, snapshots, logs. Apply retention policies. |
| **Network egress** | Data leaving the cloud is billed; cross-zone and cross-region traffic too. NAT gateways are a notorious hidden cost for private subnets pulling images. |
| **Managed services** | Load balancers, managed DBs, Kubernetes control plane fees. |
| **Observability** | Log and metric ingestion volume. |
| **People** | Often the largest cost of all: Kubernetes needs engineers. |

### 19.2 Capacity planning method

1. **Measure** the CPU/memory/GPU per instance at typical and peak load (load test with k6, Locust, or `hey`).
2. **Compute** instances needed: `peak_requests_per_second / requests_per_second_per_instance`, then add headroom (30-50%) and redundancy (N+1, so losing one instance does not overload the rest).
3. **Decide** the autoscaling floor/ceiling. The floor protects availability; the ceiling protects your wallet (and your database).
4. **Review monthly**: look at utilisation. If average CPU is 5%, you are overpaying; if memory sits at 95% of limits, you are one spike from an OOM.
5. **Use discounts when stable**: reserved instances/savings plans (commitment) for the steady baseline; spot/preemptible for fault-tolerant work.
6. **Tag and attribute costs** by team/service (labels on cloud resources) so someone owns every line of the bill.

### 19.3 Image size is a cost too

Smaller images mean faster pulls, faster scale-up and cold starts, cheaper registry storage and egress, and a smaller attack surface. Multi-stage builds, slim/distroless bases and `.dockerignore` ([Chapter 4](04-dockerfile-deep-dive.md)) are cost controls as well as hygiene.

---

## 20. Compliance and audit

Regulated environments (finance, healthcare, government, anything handling personal data) ask for evidence. Containers can make this easier, because everything is declared as code.

| Requirement (typical) | How containers/pipelines help |
|---|---|
| **Who changed what, when?** | Everything goes through git and pull requests; pipeline logs record every deploy; cloud audit logs (CloudTrail, Azure Activity Log, Cloud Audit Logs) record API calls |
| **What is running in production?** | Image digests recorded per deployment; registry holds the SBOM and provenance |
| **Are we vulnerable to CVE-X?** | Query SBOMs/scan results across all images; continuous scanning in the registry |
| **Is software from a trusted source?** | Signed images and admission policies that reject unsigned images; approved base images only |
| **Separation of duties** | Required reviewers on environments; developers cannot deploy to production directly |
| **Access control** | RBAC in registry/cluster, SSO, short-lived credentials, MFA |
| **Data protection** | Encryption at rest and in transit, no personal data in logs, backups encrypted |
| **Retention** | Documented log and backup retention policies |
| **Frameworks** | SOC 2, ISO 27001, PCI DSS, HIPAA, GDPR. Each has its own control list, and container-specific benchmarks exist: **CIS Docker Benchmark** and **CIS Kubernetes Benchmark** (audit with tools such as `docker-bench-security`, `kube-bench`) |

Practical audit-readiness checklist: retain pipeline logs; keep an inventory of images and their owners; enforce image signing and scanning gates; document the rollback and incident process; run `docker-bench-security` against hosts and record the results; review access lists quarterly. Detailed security controls are in [Chapter 10](10-security.md).

---

## 21. Team practices

Technology scales poorly without conventions. Mature teams standardise.

### 21.1 Dockerfile standards

A short, enforced team standard (also linted automatically with **hadolint**, a Dockerfile linter):

1. Use an approved base image (see golden images below), pinned to a specific version (and ideally digest).
2. Multi-stage builds; final stage contains only what runs.
3. Run as a **non-root** user (`USER 10001`).
4. `COPY` only what is needed; keep a `.dockerignore`.
5. No secrets in `ENV`, `ARG` or layers; use BuildKit secret mounts (`RUN --mount=type=secret`).
6. Pin package versions where practical; clean package caches in the same `RUN` layer.
7. Add `HEALTHCHECK` (or define probes), `LABEL` with source/commit/owner.
8. Use exec-form `CMD`/`ENTRYPOINT` and handle SIGTERM.
9. One process per container.

### 21.2 Golden base images

A **golden image** (or "paved road" base image) is a company-maintained base that teams must start from. A platform/security team builds it nightly from a trusted upstream (Debian slim, Alpine, Ubuntu, a distroless variant), applies patches, installs corporate CA certificates and proxy config, adds standard users and labels, scans it, signs it, and publishes it as `registry.example.com/base/python:3.12`. Application teams `FROM` it. Benefits: one place to fix a CVE, consistent hardening, and the ability to rebuild the world when a base vulnerability appears. Downsides: the platform team must keep it fresh and fast to respond. Automate rebuilds of downstream images when the base changes (Renovate/Dependabot can open pull requests that bump the base digest).

### 21.3 Internal image catalog

Maintain a list (a docs page, a Backstage catalog, or a registry project) of **approved images**: base images, language runtimes, common infrastructure images (nginx, Redis, Postgres) with the approved versions, owners, support status and update cadence. Block everything else at the registry or admission level.

### 21.4 Policy-as-code (OPA/Conftest and friends)

**Policy as code** means expressing rules as machine-checkable code so they are enforced automatically and consistently, instead of living in a wiki page.

- **OPA (Open Policy Agent)** is a general policy engine; policies are written in a language called **Rego**.
- **Conftest** runs OPA policies against configuration files (Dockerfiles, Compose files, Kubernetes manifests, Terraform) in CI.
- **Kubernetes admission control:** **OPA Gatekeeper** or **Kyverno** reject non-compliant objects at deploy time. Kubernetes' built-in **ValidatingAdmissionPolicy** (using the CEL expression language) covers many simple cases without extra components.

Example Conftest policy that rejects Dockerfiles using `latest` or running as root (save as `policy/dockerfile.rego`):

```rego
package main

import rego.v1

# Deny FROM with :latest or no tag
deny contains msg if {
  some i
  input[i].Cmd == "from"
  val := input[i].Value[0]
  not contains(val, "@sha256:")
  endswith(val, ":latest")
  msg := sprintf("Line %d: do not use the 'latest' tag (%s)", [i, val])
}

# Deny if there is no USER instruction
deny contains msg if {
  not any_user
  msg := "Dockerfile must set a non-root USER"
}

any_user if {
  some i
  input[i].Cmd == "user"
}
```
- `package main` is the namespace Conftest evaluates by default. `import rego.v1` opts into the modern Rego syntax.
- The first rule inspects each parsed Dockerfile instruction (`Cmd == "from"`), takes its first value (the image reference), and denies if it ends with `:latest` and is not pinned to a digest. `msg` is the error shown.
- The second rule denies Dockerfiles that never set `USER`.
- Run: `conftest test Dockerfile --policy policy/` prints `FAIL` lines for violations and exits non-zero, which fails the pipeline.
- (Rego syntax evolves; check the OPA docs for your version if the policy errors.)

Other pipeline gates: **hadolint** (Dockerfile lint), **Trivy/Grype/Docker Scout** (vulnerabilities), **Trivy config / Checkov / KICS** (infrastructure-as-code misconfiguration), **gitleaks** (secrets), **cosign verify** (signatures), **Syft** (SBOM generation).

### 21.5 Other habits

- **Ownership:** every image and service has an owner and an on-call.
- **Renovate/Dependabot** keep base images, actions and dependencies fresh through pull requests.
- **Reproducible builds:** pin versions, avoid `apt-get install` without version awareness for critical packages, and cache deliberately.
- **Documentation and runbooks** live next to the code.
- **Blameless post-incident reviews** feed improvements back into standards.

---

## 22. Production readiness checklist

Copy this into your repository's `docs/` or an issue template. Each item should be a clear yes or no.

**Image and build**
- [ ] Image built by CI from a specific commit; tagged with git SHA and version
- [ ] Production references an immutable tag or a digest, never `latest`
- [ ] Base image pinned, from an approved source, and updated on a schedule
- [ ] Multi-stage build; final image minimal; `.dockerignore` present
- [ ] Runs as non-root; read-only root filesystem where possible; capabilities dropped
- [ ] Image scanned (no unaddressed HIGH/CRITICAL with fixes); SBOM and provenance generated
- [ ] Image signed; deploy verifies signatures (where policy requires)
- [ ] No secrets in image, layers, or repository history

**Runtime configuration**
- [ ] Health checks defined (liveness, readiness, and startup where needed)
- [ ] CPU and memory limits (and requests) set from measured data
- [ ] Restart policy / orchestrator replacement configured
- [ ] Graceful shutdown (SIGTERM handled; stop timeout adequate; `init` used)
- [ ] Configuration via environment/files; secrets from a secrets manager
- [ ] Only the reverse proxy / load balancer is exposed publicly; databases and caches are not
- [ ] Network policies or firewall rules least-privilege; Docker published-port bypass of host firewall checked

**Reliability**
- [ ] At least 2 replicas (or a documented, accepted single-instance risk)
- [ ] Zero-downtime deploy path tested; rollback tested and documented
- [ ] Database migrations backward-compatible; migration process defined
- [ ] Dependencies (DB, cache, third-party APIs) have timeouts, retries with backoff, and failure behaviour decided
- [ ] Backups automated, encrypted, off-site; **restore tested** with measured RTO/RPO
- [ ] Capacity tested at expected peak plus headroom (load test)

**Observability**
- [ ] Logs to stdout/stderr, structured, rotated, shipped centrally, retention defined, no secrets
- [ ] Metrics scraped; dashboards for golden signals; resource saturation visible
- [ ] Alerts: availability, error rate, latency, disk, certificate expiry, backup failure; routed to an on-call
- [ ] External uptime check of the public URL
- [ ] Tracing for multi-service request paths (as needed)

**Host / platform**
- [ ] Supported Linux release; automatic security updates; reboot plan
- [ ] Docker Engine from the official repository, supported version, upgrade plan
- [ ] `daemon.json`: log rotation, live-restore, address pool, data-root decided
- [ ] Disk monitoring and cleanup schedule; Docker data on dedicated disk
- [ ] SSH key-only, no root login, least-privilege users, audit logging
- [ ] Time synchronisation (NTP/chrony) running

**Process**
- [ ] Runbook and on-call rota exist; owners named
- [ ] Rollback drill done in the last quarter
- [ ] Cost estimate and budget alert in place
- [ ] Compliance requirements identified and evidenced (if applicable)
- [ ] Architecture diagram and dependency list up to date

---

## 23. Incident runbook outline

An **incident** is any event that degrades the service for users. A **runbook** is the pre-written set of steps so that a tired person at 3 a.m. can act without thinking. Template:

```
INCIDENT RUNBOOK: <service name>

0. ROLES
   Incident commander (decides, communicates) | Operator (types commands) | Scribe (timeline)

1. DETECT & ACKNOWLEDGE
   - Alert fired / user report. Acknowledge the page. Open an incident channel.
   - Severity: SEV1 (full outage/data loss), SEV2 (major degradation), SEV3 (minor)

2. ASSESS (first 5 minutes, "what changed?")
   - Recent deploys? (pipeline history, `cat /opt/app/.current_tag`)
   - Dashboards: errors? latency? traffic? saturation?
   - Dependencies: database up? cloud provider status page? DNS? certificate?

3. MITIGATE FIRST, DIAGNOSE LATER
   - Bad deploy suspected      -> ROLL BACK (Section 8.3)
   - Overload                  -> scale out, enable rate limiting, shed load
   - Dependency down           -> degrade gracefully, failover
   - Disk full                 -> prune/rotate logs (safe commands only)
   - One bad instance          -> restart / replace it

4. HOST-LEVEL TRIAGE COMMANDS
   docker ps -a                                  # what is running/exited?
   docker logs --tail 200 --timestamps <name>    # recent logs
   docker inspect <name> --format '{{.State.OOMKilled}} {{.State.ExitCode}} {{.State.Health.Status}}'
   docker stats --no-stream                      # CPU / memory right now
   docker system df ; df -h                      # disk
   journalctl -u docker --since "30 min ago"     # daemon logs
   dmesg -T | grep -i -E "oom|killed"            # kernel OOM events
   docker events --since 30m                     # container lifecycle events

5. COMMUNICATE
   - Status page / customer message within agreed time; update every 30 minutes
   - Internal channel: facts, current hypothesis, next action, who owns it

6. RESOLVE & VERIFY
   - Confirm via metrics and a synthetic check, not just "it seems fine"
   - Keep heightened monitoring for a period

7. POST-INCIDENT (within a week, blameless)
   - Timeline, root cause(s), contributing factors, what went well/badly
   - Action items with owners and dates (add alert? add test? fix doc?)
   - Update this runbook
```

Quality markers: the runbook states *how to roll back in one command*; everybody knows where the dashboards and logs are; escalation contacts are current; and you practise (game days).

---

## 24. Reference architectures

### 24.1 Small team (1-10 engineers): pragmatic and cheap

Goal: reliable enough, simple enough for two people to understand, minimal on-call pain.

```
  Developers --push--> GitHub (PR review, branch protection)
                         |
                  GitHub Actions: test -> build (buildx, cache) -> Trivy -> push GHCR
                         |                                  (approval on `production` env)
                         v
                  SSH deploy job (or `docker compose` over a Docker context)
                         |
 Users --> Cloudflare (DNS, CDN, DDoS protection, WAF)
              |
              v
  +------------------ VM #1 (Ubuntu LTS, 2-4 vCPU) ------------------+
  | Caddy (auto TLS) --> api (x2) --> redis (cache only)             |
  | cAdvisor + node_exporter + Prometheus/Grafana (or hosted agent)  |
  | Docker: local log driver w/ rotation, live-restore               |
  +------------------------------+-----------------------------------+
                                 |
                                 v
              Managed Postgres (RDS / Cloud SQL / etc.), automatic backups + PITR
              Object storage (S3-compatible) for files and VM backups (restic)
  Monitoring: external uptime check -> Slack/e-mail; alerts on disk, 5xx, cert expiry
  Infra as code: Terraform for the VM, firewall, DNS (so it can be rebuilt)
```

Characteristics: roughly tens of dollars to low hundreds per month; deploy in minutes; recovery plan = rebuild VM from Terraform + redeploy last good image + managed DB. Single VM is the accepted risk; upgrading to two VMs behind a cloud load balancer removes it. For a low-ops variant, replace the VM with Cloud Run, Azure Container Apps or ECS Fargate, keeping the managed database.

### 24.2 Large enterprise (hundreds of engineers): platform-based

Goal: self-service for teams, strong governance, high availability, auditability.

```
 +----------------------------- SOURCE & BUILD (shared platform) -----------------------------+
 | GitHub Enterprise / GitLab  -> CI (ephemeral runners) -> build (BuildKit, remote cache)      |
 |   -> unit/integration tests -> SAST + dependency scan + secret scan -> image scan            |
 |   -> SBOM + provenance -> Cosign signature -> push to Harbor/ECR/ACR/GAR (immutable tags)    |
 |   Golden base images built nightly by the platform team; internal catalog; Renovate bumps    |
 +---------------------------------------------+------------------------------------------------+
                                               | image promoted by digest
                                               v
 +---------------------------- DEPLOY (GitOps) -----------------------------------------------+
 | Config repo (Helm/Kustomize) --PR--> approvals --> Argo CD / Flux --> clusters                |
 | Progressive delivery (Argo Rollouts / Flagger): canary + automated rollback on SLO breach     |
 +---------------------------------------------+------------------------------------------------+
                                               v
 +-------------------- RUNTIME: multi-account / multi-region cloud (landing zone) -------------+
 |  Region A                                          Region B (DR / active-active)             |
 |  +-------------------------------+                 +-------------------------------+         |
 |  | Managed Kubernetes (EKS/AKS/GKE) clusters per env/team, node pools (CPU, GPU, spot)|      |
 |  |  Gateway API / ingress, service mesh (mTLS), NetworkPolicies, Pod Security Standards|      |
 |  |  Admission: Kyverno/Gatekeeper (signed images only, no root, resource limits)      |       |
 |  |  Autoscaling: HPA + KEDA + Karpenter/Cluster Autoscaler                            |       |
 |  +-------------------------------+                 +-------------------------------+         |
 |  Managed data: Aurora/Cloud SQL/Cosmos, managed Redis/Kafka, object storage w/ replication    |
 |  Secrets: Vault/cloud secrets manager + External Secrets Operator; OIDC workload identity     |
 |  Edge: global load balancer / CDN / WAF / DDoS protection; private connectivity (VPN/Direct)  |
 +---------------------------------------------+------------------------------------------------+
                                               v
 +------------------------------ OBSERVABILITY & GOVERNANCE -----------------------------------+
 |  OpenTelemetry -> Prometheus/Mimir, Loki/ELK, Tempo/Jaeger -> Grafana / Datadog; SLOs; pager  |
 |  Cloud audit logs -> SIEM; CIS benchmarks; cost allocation (FinOps) per team; DR drills       |
 +----------------------------------------------------------------------------------------------+
```

Key differences from the small-team setup: Kubernetes as a shared platform (often run by a platform engineering team that provides "golden paths": templates, pipelines and defaults), GitOps instead of SSH deploys, signed images enforced at admission, multiple environments and regions, formal SLOs (Service Level Objectives: reliability targets such as 99.9% availability) with error budgets, and a security and compliance programme layered over everything.

**Choosing between them:** start with the small-team architecture. Add pieces (second VM, Kubernetes, GitOps, service mesh) only when you can name the pain they solve. Every added piece is something you must operate.

---

## 25. Common mistakes

A consolidated list of the production traps from this chapter:

1. Running `:latest` in production.
2. Rebuilding images per environment instead of promoting one.
3. Unrotated container logs filling the disk.
4. Docker-published ports bypassing the host firewall; exposing databases or caches to the internet.
5. Secrets in images, git, or plain env files in the repository.
6. No health checks; health checks that always pass or that cascade failures.
7. No resource limits, or limits ignoring the runtime's memory behaviour.
8. Backups that were never restored.
9. Only one replica and no tested rollback.
10. Running `docker volume prune` on a production host.
11. Mounting `/var/run/docker.sock` into application containers (full root access to the host).
12. Treating Watchtower-style auto-updates as a deployment strategy; trusting unpinned third-party CI actions.
13. Adopting Kubernetes before there is a problem it solves.
14. No owner, no runbook, no alert routing: dashboards nobody watches.
15. Letting address pools collide with the VPN, then debugging "random" network failures.

---

## 26. Summary

- Production Docker means **Linux** hosts running **Docker Engine** or a managed container platform.
- The enterprise flow is: git -> CI (build, scan, test, sign) -> registry -> CD -> runtime -> observability -> rollback. **Build once, deploy the same digest everywhere.**
- Tag images with git SHA and semantic versions; never deploy `latest`; promote by re-tagging, not rebuilding.
- Registries: Docker Hub (mind the rate limits), GHCR, ECR, ACR, GAR, Harbor/Artifactory/Nexus; use pull-through caches and retention policies.
- A GitHub Actions workflow with Buildx, caching, multi-arch builds, scanning, GHCR push and an SSH deploy with automatic rollback is a solid foundation. Pin and minimise third-party actions after the 2026 Trivy compromise.
- Choose the simplest runtime that meets your needs: PaaS, serverless containers, single VM + Compose (done properly), Swarm (maintenance mode), or Kubernetes (powerful, costly in effort). App Runner is closed to new customers; ingress-nginx is retired; Watchtower is archived.
- Operate it: health checks, resource limits, rotated and centralised logs, metrics and alerts, tested backups, a daemon baseline, disk hygiene, planned upgrades, runbooks.

Next: [Chapter 12](12-troubleshooting-and-cheatsheet.md) teaches you to debug everything that goes wrong, and [Chapter 13](13-interview-questions-roadmap-glossary.md) turns this knowledge into interview answers and a portfolio.

Previous: [Chapter 10: Security](10-security.md) | Back to the start: [Chapter 1](01-what-is-docker.md)

---

### Sources consulted while verifying this chapter (October 2026)

- Docker Hub usage and pull limits: <https://docs.docker.com/docker-hub/usage/pulls/>
- Docker logging configuration: <https://docs.docker.com/engine/logging/configure/>
- Docker live-restore: <https://docs.docker.com/engine/daemon/live-restore/>
- Docker Engine 29 release notes: <https://docs.docker.com/engine/release-notes/29/>
- GitHub release pages for `actions/checkout`, `docker/setup-qemu-action`, `docker/setup-buildx-action`, `docker/login-action`, `docker/metadata-action`, `docker/build-push-action`, and `aquasecurity/trivy`
- Trivy supply-chain advisory: <https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23>
- AWS App Runner availability change: <https://docs.aws.amazon.com/apprunner/latest/dg/apprunner-availability-change.html>
- Google Container Registry transition: <https://docs.cloud.google.com/artifact-registry/docs/transition/transition-from-gcr>
- Watchtower archive notice: <https://github.com/containrrr/watchtower/discussions/2135>
- Ingress NGINX retirement: <https://www.kubernetes.dev/blog/2025/11/12/ingress-nginx-retirement/>
- Kubernetes release information: <https://www.kubernetes.dev/resources/release/>
