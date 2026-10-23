# Chapter 11: Production and Enterprise Docker

This chapter covers running Docker in production: host operating systems, the path from code to a running container, image tagging, registries, a CI/CD pipeline, deployment targets, deployment strategies, configuration and secrets, health checks, limits, logging, monitoring, backups, scaling, daemon configuration, upgrades, cost, compliance, team practices, a readiness checklist and an incident runbook.

It builds on [Chapter 4](04-dockerfile-deep-dive.md) (Dockerfiles), [Chapter 5](05-storage-and-networking.md) (storage and networking), [Chapter 6](06-docker-compose.md) (Compose) and [Chapter 10](10-security.md) (security).

> **Verification note.** Facts were checked in October 2026. Anything not confirmed from a primary source is marked **(unverified)**. Tool versions change quickly; check each project's releases page before copying a version into a pipeline.

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

In production, a container is a service that must start by itself, restart by itself, report when it is unhealthy, keep its data, protect its secrets and be replaceable by someone unfamiliar with it.

| Concern | Laptop | Production |
|---|---|---|
| Starting | `docker run` | The platform starts it on boot |
| Crash | Restart manually | Auto-restart plus alert |
| Image version | `latest`, whatever was built | Specific, immutable, traceable version |
| Secrets | `.env` file | Secrets manager, injected at runtime |
| Logs | `docker logs` | Shipped centrally, rotated, searchable |
| Data | Volume on local disk | Backed up, often outside the container platform |
| Updates | Rebuild and rerun | Pipeline builds, scans, signs, deploys, can roll back |
| Access | One user | Many users, least privilege, audit trail |

---

## 2. Which operating system runs production Docker?

### 2.1 The short answer: Linux

Containers are a Linux technology. A container is a Linux process isolated by two kernel features:

- **Namespaces**: isolated views of process IDs, network, mounts, hostnames and users.
- **cgroups** (control groups): limits on CPU, memory and disk I/O.

Linux containers run natively on a Linux server with almost no overhead. On Windows and macOS, Docker runs a hidden Linux VM (WSL 2 or a lightweight VM) and containers run inside it. This suits development, not servers.

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

| OS | Verdict | Reason |
|---|---|---|
| macOS | Never for production | Apple licensing restricts macOS in data centres, Docker Desktop for Mac is a development tool, and macOS containers do not exist |
| Windows Server (Linux containers) | Not recommended | Needs the same hidden Linux VM. Adds licensing and performance cost for no gain |
| Windows Server (Windows containers) | Niche | Use only for legacy .NET Framework or other Windows-only software. These containers share the Windows kernel, images are multiple gigabytes, and the ecosystem is smaller |
| Docker Desktop on a server | Do not | Licensed for desktop use (paid for larger companies), has an unneeded GUI, adds a VM layer. Install **Docker Engine** instead |

Develop on any OS. Build and run production on Linux with **Docker Engine** (the headless daemon, no GUI) or a container platform that hides the host.

### 2.3 Which Linux?

Two families exist: **general-purpose distributions** (you install and patch packages; Docker is one program) and **container-optimized operating systems** (a minimal, locked-down OS that only runs containers).

| Linux | Type | Best for | Notes |
|---|---|---|---|
| **Ubuntu Server LTS** (for example 24.04 LTS) | General purpose | Small teams, cloud VMs | LTS = Long Term Support: five years of free security updates, new LTS every two years. Docker publishes an official apt repository |
| **Debian** (stable) | General purpose | Stable, small, neutral base | Ubuntu is built on Debian. Slower release cadence |
| **RHEL** | General purpose, commercial | Enterprises needing vendor support and certifications | Red Hat's container engine is **Podman**. Docker Engine can be installed but is not the supported path |
| **Rocky Linux / AlmaLinux** | General purpose | Free RHEL-compatible rebuilds | Same package family (`dnf`). Docker publishes a CentOS-compatible repository |
| **Amazon Linux** (2023) | General purpose, cloud | EC2 | Tuned and patched by AWS. Docker is in its package repository. Behaviour may differ outside AWS |
| **Container-Optimized OS (COS)** | Container-optimized | Google Cloud (GCE, GKE nodes) | Read-only root filesystem, automatic updates, Docker/containerd included. No `apt install` |
| **Bottlerocket** | Container-optimized | AWS (EKS/ECS nodes) | Open source from AWS. Immutable root, atomic updates, API-driven configuration, no SSH by default |
| **Flatcar Container Linux** | Container-optimized | Bare metal, any cloud | Community continuation of CoreOS Container Linux. Auto-updating, immutable `/usr` |

Selection:

- Learning, or a small team with one VM: Ubuntu Server LTS.
- Kubernetes nodes in the cloud: the managed service's node image, or Bottlerocket/COS for a smaller attack surface.
- Large company with a Red Hat contract: RHEL, probably Podman/OpenShift, with Docker at developer and build level.

| Mistake | Fix |
|---|---|
| Production on a distribution nobody on the team can patch | Use a distribution the team supports |
| End-of-life OS release | Move to a release that still receives security updates |
| Docker from the distribution's old built-in package | Use Docker's official repository ([Chapter 2](02-installation-and-operating-systems.md)) |
| cgroup v1 | Use cgroup v2 (modern default). Docker Engine 29 deprecated cgroup v1 (kept until at least 2029 per the release notes) |

---

## 3. The end-to-end enterprise flow

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

| # | Stage | What happens |
|---|---|---|
| 1 | Developer laptop | Write code, run `docker compose up`, run unit tests. The container tested locally is the one production uses |
| 2 | Git host | Push a branch and open a **pull request** (a proposal to merge). Teammates review. **Branch protection** blocks merging unless checks pass. Git is the single source of truth |
| 3 | CI (Continuous Integration) | A **runner** checks out code, runs tests, builds the image, scans it and signs it |
| 4 | Registry | Stores images. CI pushes, production pulls |
| 5 | CD (Continuous Delivery/Deployment) | Takes an image from the registry and tells the runtime to run it. *Delivery* usually has a human approval step; *Deployment* is fully automatic. Deploy an image that was already tested; never rebuild for production |
| 6 | Runtime platform | Runs containers continuously: VM, Kubernetes, serverless containers ([Section 7](#7-where-do-you-run-the-containers-deployment-targets)) |
| 7 | Observability | Logs, metrics and traces go to central tools. Alerts page a human |
| 8 | Rollback | Redeploy the previous image if the new version is bad |

**Build once, deploy many.** The same image, identified by digest, moves through test, staging and production. Only configuration (environment variables, secrets) changes between environments.

---

## 4. Image tagging strategy

A **tag** is a label on an image in a registry. In `ghcr.io/acme/chat-api:1.4.2`, `ghcr.io` is the registry, `acme/chat-api` is the repository and `1.4.2` is the tag.

A tag is movable: anyone with push permission can point `1.4.2` at a different image. A **digest** is the SHA-256 hash of the image content and cannot point to anything else.

```
 Tags (movable labels)                    Digest (permanent fingerprint)

   :latest  ----+
   :1.4     ----+--------------------->   sha256:9f86d081884c7d659a2feaa0c55ad015...
   :1.4.2   ----+                         (identifies exactly one image forever)
   :sha-3f9a1c2 +
```

### 4.1 Why `latest` is dangerous

`latest` is the tag Docker uses when none is given. It does not mean "newest". Problems with `image: myapp:latest` in production:

1. Unreproducible: two servers pulling an hour apart can run different code.
2. No rollback target: the previous `latest` is gone.
3. Silent changes: a restart can pull an untested version.
4. No audit trail: the commit behind an outage is unknown.

Use `latest` only in local development.

### 4.2 The tags to create for every build

| Tag type | Example | Purpose | Mutable? |
|---|---|---|---|
| **Git SHA** | `sha-3f9a1c2` | Identifies the commit that produced the image. Most important for traceability | No |
| **Semantic version** | `1.4.2` | A release. `MAJOR.MINOR.PATCH`: MAJOR for breaking changes, MINOR for backward-compatible features, PATCH for fixes | No |
| **Minor / major alias** | `1.4`, `1` | Latest 1.4.x. Moves with each patch release | Yes |
| **Branch** | `main`, `staging` | Newest build of the branch. For dev/staging | Yes |
| **Environment** | `prod` | Marks what production runs. Moves on promotion | Yes |
| **Date-time** | `2026-10-23-1415` | Alternative when semver is not used | No |

Minimum: every CI build pushes `sha-<short-commit>`. Every release (git tag `v1.4.2`) also pushes `1.4.2` and `1.4`.

### 4.3 Immutable tags

An **immutable tag** cannot be overwritten once pushed; a push of a different image under the same tag fails.

- **Amazon ECR**: per-repository tag immutability setting.
- **Azure Container Registry**: write/delete locks on an image or repository.
- **Google Artifact Registry**: immutable tags per repository.
- **Harbor**: tag immutability rules.
- **GitHub Container Registry and Docker Hub**: no repository-wide immutability switch **(unverified for 2026; check current docs)**. Enforce in the pipeline by never re-pushing a version or SHA tag.

Enable immutability for version and SHA tags where supported. Keep alias tags (`1.4`, `prod`) mutable, using a separate repository or a registry that supports patterns for mutable tags.

### 4.4 Pinning by digest

```yaml
services:
  api:
    image: ghcr.io/acme/chat-api@sha256:9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
```

- `@sha256:...`: exactly this content. Docker verifies the hash after download.
- Both forms can be combined for readability: `image: ghcr.io/acme/chat-api:1.4.2@sha256:9f86...`. Docker resolves by digest and ignores the tag.

Find a digest:

```bash
docker buildx imagetools inspect ghcr.io/acme/chat-api:1.4.2
```

- Queries the registry (not the local cache) for the manifest. Works for multi-architecture images.

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

- The top `Digest` is the **index** digest (a list pointing to one image per CPU architecture). Deploying the index digest works on both architectures.

### 4.5 Promotion between environments

**Promotion** moves the same image to the next stage without rebuilding.

```
 CI builds & pushes                  gates                        gates
 sha-3f9a1c2 ------> DEV  ------> (tests pass) ---> STAGING ---> (human approval) ---> PROD
   (digest A)       runs A                          runs A                              runs A
                       \________ same digest A in every environment ________/
```

Add a new tag to the existing image:

```bash
docker buildx imagetools create \
  --tag ghcr.io/acme/chat-api:staging \
  ghcr.io/acme/chat-api:sha-3f9a1c2
```

- `docker buildx imagetools create`: creates a new manifest in the registry without pulling or re-uploading layers. Fast, and preserves the digest.
- `--tag`: the new label.
- Last argument: the existing image.

Alternative: `crane tag ghcr.io/acme/chat-api:sha-3f9a1c2 staging` (open-source `crane` utility).

In GitOps setups ([Section 7.4](#74-kubernetes-for-docker-users)), promotion is a git commit that changes the image tag or digest in a manifest.

### 4.6 A tagging policy

```
1. Every CI build on any branch:    push  :sha-<7-char-commit>
2. Merge to main:                   also  :main   (dev auto-deploys this)
3. Git tag vX.Y.Z (release):        also  :X.Y.Z  :X.Y  :X     (never overwritten)
4. Promotion to staging/prod:       add   :staging / :prod to the SAME digest
5. Production manifests reference:  :X.Y.Z or @sha256 digest. Never :latest or :main.
6. Retention: keep all release tags forever; expire untagged and old sha- tags after 30-90 days.
```

| Mistake | Fix |
|---|---|
| Deploying `:latest` or `:main` | Deploy a version tag or digest |
| Re-pushing `1.4.2` after a hotfix | Publish `1.4.3` |
| Rebuilding for production | Promote the tested image |
| Version tags only, no SHA | Also push `sha-<commit>` |

---

## 5. Container registries

A **container registry** is a server that stores and serves images. Pushing uploads an image; pulling downloads it. Registries use the HTTP API defined by the **OCI Distribution Specification** (OCI = Open Container Initiative, the body that standardises image and runtime formats), so any Docker client works with any registry.

### 5.1 Hosted registries compared

| Registry | Address pattern | Operator | Good for | Notable features |
|---|---|---|---|---|
| **Docker Hub** | `docker.io/library/nginx`, `username/app` | Docker, Inc. | Public images, official base images | Largest public catalogue, "Docker Official Images" and "Verified Publisher" badges, Docker Scout scanning, pull rate limits ([5.3](#53-docker-hub-rate-limits)) |
| **GitHub Container Registry (GHCR)** | `ghcr.io/OWNER/IMAGE` | GitHub | Projects on GitHub | Authenticates with the workflow's `GITHUB_TOKEN`, free for public images, permissions inherit from the repository. GitHub documents storage and bandwidth as free at the time of writing, described as a "soft" billing state; treat free as current policy, not a guarantee |
| **Amazon ECR** | `123456789012.dkr.ecr.us-east-1.amazonaws.com/app` | AWS | AWS workloads (ECS, EKS, Lambda) | IAM permissions, tag immutability, lifecycle policies, replication, basic and enhanced scanning (Amazon Inspector), **pull-through cache** |
| **Azure Container Registry (ACR)** | `myregistry.azurecr.io/app` | Microsoft | Azure workloads (AKS, Container Apps) | Entra ID auth, geo-replication (Premium), ACR Tasks, private endpoints, content trust and signing features, artifact cache |
| **Google Artifact Registry (GAR)** | `us-docker.pkg.dev/PROJECT/REPO/app` | Google Cloud | GCP workloads (GKE, Cloud Run) | Containers and language packages (npm, Maven, Python), IAM, cleanup policies, remote repositories (proxy Docker Hub) |
| **Quay.io** | `quay.io/ORG/app` | Red Hat | Red Hat ecosystems | Built-in scanning, robot accounts, OpenShift integration |
| **GitLab Container Registry** | `registry.gitlab.com/group/project` | GitLab | GitLab projects | Built into every project |

Google Container Registry (`gcr.io`) was shut down: writes became unavailable in March 2025, and the final stage on 14 October 2025 retired remaining storage. `gcr.io` hostnames can now be served by Artifact Registry. Tutorials using the old service are outdated.

### 5.2 Self-hosted registries

Reasons: one internal registry for all teams, air-gapped operation, caching of external images, control over who can pull what.

| Product | Type | Highlights |
|---|---|---|
| **Harbor** | Open source, CNCF graduated | RBAC, project quotas, scanning (Trivy built in), signing (Cosign/Notation), replication, proxy-cache projects, retention rules, robot accounts. Most popular open-source enterprise registry |
| **JFrog Artifactory** | Commercial (limited free edition) | Universal repository (Docker, npm, Maven, PyPI, Helm) with local, remote and virtual repositories |
| **Sonatype Nexus Repository** | Commercial with free community edition | Universal repository; Docker proxy/hosted/group types |
| **CNCF Distribution (`registry:3`)** | Open source, minimal | Reference implementation of the registry API. No UI, no RBAC, basic auth only. For labs |

Minimal private registry for experiments:

```bash
docker run -d -p 127.0.0.1:5000:5000 --name registry --restart=unless-stopped registry:3
```

- `-d`: run in the background.
- `-p 127.0.0.1:5000:5000`: publish port 5000 on the local machine only.
- `registry:3`: official registry image (version 3 line). If the `3` tag is missing from your mirror, check the registry project's tags.
- `--restart=unless-stopped`: restart after crashes or reboots.

Push to it:

```bash
docker tag chat-api:1.0 localhost:5000/chat-api:1.0
docker push localhost:5000/chat-api:1.0
```

- `docker tag`: adds a second name that includes the registry address.
- `docker push`: uploads. Output lists layer IDs and `digest: sha256:... size: 1234`.

### 5.3 Docker Hub rate limits

| Who | Limit (per official docs, seen October 2026) |
|---|---|
| Unauthenticated | **100 pulls per 6 hours**, per source IPv4 address (or IPv6 /64 block) |
| Authenticated, free Personal account | **200 pulls per 6 hours** |
| Paid plans (Pro, Team, Business) | Effectively unlimited, subject to fair use |

> Unverified: figures confirmed through the documentation's summary and secondary sources; the live page could not be loaded. Docker announced stricter hourly limits in 2025, then said it had not enforced them. Check <https://docs.docker.com/docker-hub/usage/>.

CI systems and NAT gateways (many machines on one public IP) exhaust 100 pulls quickly. Symptom:

```
toomanyrequests: You have reached your pull rate limit. You may increase the limit by authenticating and upgrading: https://www.docker.com/increase-rate-limit
```

Fixes, cheapest first:

1. **Authenticate** pulls (`docker login`).
2. **Use a pull-through cache** (5.4).
3. **Mirror critical base images** into your own registry (`docker buildx imagetools create` can copy them) and reference the copy in Dockerfiles.
4. **Use base images hosted elsewhere** (many Official Images are also on `public.ecr.aws`).

### 5.4 Pull-through caches

A **pull-through cache** (proxy cache) is a registry between clients and an upstream registry. The first pull of `nginx:1.27` fetches from Docker Hub and stores a copy; later pulls are served from the cache.

```
 CI runners / servers ---> internal cache (Harbor / ECR / ACR / GAR / Artifactory) ---> Docker Hub
        pull nginx              first pull: fetch + store                            (hit once)
        pull nginx              later pulls: served locally, no Hub rate limit consumed
```

Benefits: avoids rate limits, faster pulls, works during a brief Docker Hub outage, and gives a place to scan and approve incoming images.

Options: ECR pull-through cache rules, ACR artifact cache, Artifact Registry remote repositories, Harbor proxy-cache projects, Artifactory/Nexus remote repositories, or the `registry` image in proxy mode. On a Docker host, set `registry-mirrors` in `daemon.json` ([Section 16](#16-docker-daemon-production-configuration-daemonjson)). `registry-mirrors` applies only to Docker Hub pulls.

### 5.5 Retention and cleanup policies

A **retention policy** automatically deletes old images. Every CI build adds an image, so registries fill up.

| Rule | Example |
|---|---|
| Delete untagged images older than N days | 7 days |
| Keep only the last N `sha-*` images per repository | last 50 |
| Delete branch tags older than N days | 30 days |
| Never delete release tags or anything currently deployed | exempt `^[0-9]+\.[0-9]+\.[0-9]+$` |

Managed registries provide this (ECR lifecycle policies, ACR retention/purge tasks, Artifact Registry cleanup policies, Harbor retention rules). Test in dry-run mode first. Deleting an image needed for rollback causes outages.

### 5.6 Signing, SBOMs and attestations

- **SBOM** (Software Bill of Materials): a list of everything inside an image, used to answer "are we affected by CVE-XXXX?".
- **Provenance**: a signed statement that the image was built from a given commit, by a given workflow, at a given time.
- **Signing** (for example **Sigstore Cosign**): a cryptographic signature that lets a cluster verify the image came from your pipeline.
- BuildKit generates SBOM and provenance attestations (`--sbom=true --provenance=mode=max`, used in the workflow below).
- Docker Content Trust (`DOCKER_CONTENT_TRUST`) was **removed from the Docker CLI in Engine 29** per the official release notes. Use Cosign or Notation for new work.

Details: [Chapter 10](10-security.md).

---

## 6. A complete CI/CD pipeline with GitHub Actions

**GitHub Actions** is GitHub's automation service. A **workflow** is a YAML file in `.github/workflows/` containing **jobs**. A job is a list of **steps** that run on a **runner** (a temporary virtual machine, for example `ubuntu-latest`). A step either runs a shell command (`run:`) or uses a reusable **action** (`uses:`).

### 6.1 Actions used, and their versions

Verified from release pages in October 2026:

| Action | Purpose | Current major |
|---|---|---|
| `actions/checkout` | Downloads the repository into the runner | v7 |
| `docker/setup-qemu-action` | Installs CPU emulators so an x86 runner can build ARM images | v4 |
| `docker/setup-buildx-action` | Sets up **Buildx**, Docker's builder (BuildKit engine) | v4 |
| `docker/login-action` | Logs in to a registry | v4 |
| `docker/metadata-action` | Computes image tags and labels from git info | v6 |
| `docker/build-push-action` | Builds and optionally pushes | v7 (Node.js 24) |

> **Supply-chain warning.** In March 2026 the Trivy project's release pipeline and GitHub Actions (`aquasecurity/trivy-action`, `aquasecurity/setup-trivy`) were compromised. Attackers published a malicious Trivy release and rewrote most version tags to point at credential-stealing code. Anyone referencing a tag such as `@0.28.0` could have run attacker code with access to CI secrets.
>
> - A version tag in `uses:` is movable, like an image tag.
> - Third-party actions run with your secrets.
> - Pin third-party actions to a full commit SHA verified against the upstream repository, and keep their number small.
>
> The workflow below runs Trivy as a version-pinned container with `docker run`, so no third-party action is involved. For more safety, pin that image by digest. Major-version tags are widely used for Docker's own actions; strict teams pin those to SHAs too (Dependabot or Renovate can keep SHAs updated).

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
- `name`: label shown in the GitHub Actions tab.

```yaml
on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  pull_request:
    branches: [main]
```
- `on`: triggers. Runs on pushes to `main`, pushes of tags like `v1.4.2`, and pull requests targeting `main`.
- Pull requests build and scan but do not push or deploy (enforced by `if:` conditions below).

```yaml
concurrency:
  group: ci-cd-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}
```
- `concurrency`: prevents concurrent runs for the same ref. `group` names the lane. `cancel-in-progress` cancels an older pull-request run when a new commit arrives, but never cancels runs on `main` or tags (a half-finished deploy is dangerous).

```yaml
env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  TRIVY_VERSION: "0.76.1"
```
- `env`: variables for all jobs. `github.repository` is `OWNER/REPO` (for example `acme/chat-api`). GHCR requires lowercase names; a step below lowercases it.
- `TRIVY_VERSION`: pins the scanner. 0.76.1 was the newest release found at writing time; check <https://github.com/aquasecurity/trivy/releases>. Never use v0.69.4 (the malicious release).

```yaml
permissions:
  contents: read
```
- Sets the default `GITHUB_TOKEN` permissions to read-only. Jobs request more explicitly (least privilege).

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
- `runs-on: ubuntu-latest`: runner OS.
- `timeout-minutes`: kills a stuck job.
- `packages: write`: allows pushing to GHCR with `GITHUB_TOKEN`.
- `id-token: write`: allows requesting an OIDC token (keyless signing, cloud logins). Remove it if unused.
- `outputs`: values (image name, digest, SHA tag) that the deploy job uses.

```yaml
    steps:
      - name: Check out code
        uses: actions/checkout@v7
```
- Downloads the repository.

```yaml
      - name: Compute lowercase image name and SHA tag
        id: vars
        run: |
          IMAGE="${REGISTRY}/${IMAGE_NAME,,}"
          echo "image=${IMAGE}" >> "$GITHUB_OUTPUT"
          echo "sha_tag=sha-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
```
- `id: vars`: lets later steps read this step's outputs.
- `${IMAGE_NAME,,}`: Bash lowercase.
- `echo "key=value" >> "$GITHUB_OUTPUT"`: publishes a step output. `${GITHUB_SHA::7}` is the first 7 characters of the commit hash.

```yaml
      - name: Set up QEMU (cross-CPU emulation)
        uses: docker/setup-qemu-action@v4
```
- **QEMU** is an emulator. Registers it so the x86 runner can run ARM binaries during the build (`linux/arm64` images). Skip when building one architecture.

```yaml
      - name: Set up Buildx
        uses: docker/setup-buildx-action@v4
```
- Creates a Buildx builder (BuildKit) that provides cache export, multi-platform builds and attestations.

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
- Converts git facts into Docker tags and OCI **labels** (key/value metadata in the image, such as source URL and commit).
- `images`: image name without a tag.
- Tagging rules:
  - `type=sha,format=short`: `sha-3f9a1c2`.
  - `type=ref,event=branch`: `main` on pushes to main.
  - `type=ref,event=pr`: `pr-123` for pull requests.
  - `type=semver,pattern={{version}}`: `1.4.2` when the git tag is `v1.4.2`. `{{major}}.{{minor}}` gives `1.4`.
  - `type=raw,value=latest,enable={{is_default_branch}}`: `latest` on the default branch only. Production manifests must never reference it.

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
- Builds one amd64 image and `load: true` puts it in the runner's local Docker for scanning. Multi-platform images cannot be loaded into the classic local store, so one platform is scanned now and all platforms are pushed later.
- `context: .`: repo root as the build context ([Chapter 4](04-dockerfile-deep-dive.md)).
- `push: false`: nothing uploaded.
- `tags: scan-target:ci`: throwaway local name.
- `cache-from: type=gha` / `cache-to: type=gha,mode=max`: use the GitHub Actions cache for layer cache between runs. `mode=max` also caches intermediate stages (important for multi-stage builds). The cache is limited per repository (10 GB at time of writing) and old entries are evicted. For more space use `type=registry` (cache stored as an image in your registry).

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

| Argument | Meaning |
|---|---|
| `docker run --rm` | One-off container, deleted afterwards |
| `-v /var/run/docker.sock:/var/run/docker.sock` | Gives the scanner the runner's Docker so it can read `scan-target:ci`. The socket is root-equivalent; acceptable on a throwaway CI runner, never on a production server ([Chapter 10](10-security.md)) |
| `-v "$HOME/.cache/trivy:/root/.cache/"` | Persists the vulnerability database on this runner |
| `ghcr.io/aquasecurity/trivy:${TRIVY_VERSION}` | Official Trivy image, version-pinned |
| `image` | Trivy sub-command for container images |
| `--exit-code 1` | Fails the job if matching vulnerabilities are found |
| `--severity HIGH,CRITICAL` | Fails only on high and critical issues |
| `--ignore-unfixed` | Ignores vulnerabilities with no available fix. Revisit as maturity grows |
| `--no-progress` | Disables the progress bar to keep logs clean |

Example failing output (the CVE number is a placeholder):

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

```yaml
      - name: Log in to GHCR
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v4
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```
- `if:` skips the step on pull requests (fork PRs get no write permission anyway).
- `username: ${{ github.actor }}`: the user who triggered the run.
- `password: ${{ secrets.GITHUB_TOKEN }}`: the short-lived token GitHub creates for each run. No long-lived registry password is stored. GitHub masks secrets in logs.

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
- `push: true` and `platforms: linux/amd64,linux/arm64`: build for Intel/AMD and ARM (AWS Graviton, Apple Silicon, Raspberry Pi) and publish one multi-arch image. With a warm cache, only the arm64 part does real work.
- `tags` and `labels`: from the metadata step.
- `sbom: true`, `provenance: mode=max`: attach an SBOM and detailed provenance, stored in the registry. The GHCR page may show extra "unknown/unknown" platform entries; these are attestation manifests.
- `id: push`: exposes this step's `digest` output (the fingerprint of what was pushed).

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
- `$GITHUB_STEP_SUMMARY`: a file whose Markdown is shown on the run page.

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
- `needs: build`: waits for the build job.
- `if:`: deploys only on pushes to `main` or a release tag, never from pull requests.
- `environment: production`: ties the job to a GitHub **Environment**. Adding **required reviewers** to the environment pauses the job until a human approves (the continuous-delivery gate). Environments also scope secrets, so production secrets reach only this job.

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
- `secrets.DEPLOY_SSH_KEY`: a private key created only for deployments, stored in the environment's secrets. Its public half is in `~/.ssh/authorized_keys` of a dedicated `deploy` user on the VM.
- `secrets.DEPLOY_KNOWN_HOSTS`: the server's host key line. Get it once with `ssh-keyscan -t ed25519 your.server.example` and verify the fingerprint out-of-band. A pinned value makes the job refuse an impostor server. Do not use `StrictHostKeyChecking=no`.
- `install -m 700 -d ~/.ssh`: creates the folder with correct permissions; SSH rejects keys with loose permissions.
- Plain `ssh` is used instead of a third-party deploy action so no extra code can read the key.

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
- Passes the SHA tag from the build job to the server script. `BatchMode=yes` makes SSH fail instead of prompting for a password.
- The script (6.4) pulls the image, restarts the stack, waits for health, and rolls back on failure.
- The server needs no inbound access from GitHub except SSH, and only a read-only registry credential. To avoid SSH, use a pull-based tool or GitOps ([Section 7](#7-where-do-you-run-the-containers-deployment-targets)).

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
- `${IMAGE_TAG:?message}`: Compose fails if the variable is missing instead of using an empty tag.
- `restart: unless-stopped`: restarts after crashes and reboots unless stopped manually.
- `env_file`: loads secret variables from a file readable only by root/deploy ([Section 9](#9-configuration-and-secrets-in-production)).
- `healthcheck`: used by `--wait` below. Uses Python because slim images often lack `curl`.
- `logging`: caps log size ([Section 12](#12-logging)).
- `deploy.resources.limits`: caps CPU and memory ([Section 11](#11-resource-limits-and-the-oom-killer)). Modern Compose honours these outside Swarm.
- `ports: "127.0.0.1:8000:8000"`: publishes on localhost only. The reverse proxy (7.1) is the only public entrance.

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
- `set -euo pipefail`: stop on errors (`-e`), unset variables (`-u`) and failures inside pipes (`pipefail`).
- `PREV_TAG`: remembers what was running, for rollback.
- `docker compose pull api`: downloads the new image before touching the running container, so downtime is only the swap.
- `up -d --wait --wait-timeout 120`: recreates changed services in the background (`-d`) and waits (`--wait`) until containers are running and, if a healthcheck exists, healthy. Fails after 120 seconds.
- On failure the previous tag is started again and the script exits non-zero, so the GitHub job fails.
- `docker image prune -f --filter "until=168h"`: removes unused images older than a week. Running images are untouched.

Plain Compose has a few seconds of downtime while the container is replaced. [Section 8](#8-deployment-strategies-and-rollbacks) covers zero-downtime options.

### 6.5 One-time setup checklist for this pipeline

1. Create the VM, install Docker Engine ([Chapter 2](02-installation-and-operating-systems.md)), and create a `deploy` user in the `docker` group. Membership in `docker` is root-equivalent: use a dedicated key and restrict it (stricter shops use rootless Docker or `command=` restrictions in `authorized_keys`).
2. On the VM, run `docker login ghcr.io` with a fine-grained token that has read-only package access (or make the image public).
3. In GitHub, create the `production` environment, add reviewers, and add the secrets `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_SSH_KEY`, `DEPLOY_KNOWN_HOSTS`.
4. Enable branch protection on `main` requiring the `build` job.
5. Run the first deploy manually with `./deploy.sh sha-xxxxxxx`.

| Mistake | Fix |
|---|---|
| Personal access token with full power in the pipeline | Use the scoped `GITHUB_TOKEN` |
| No `timeout-minutes` | Set it on every job |
| Rebuilding the image at deploy time | Pull the image CI built |
| Fork pull requests with access to secrets | Keep secrets out of PR workflows; use environments |
| No health wait, so "deploy succeeded" means only "command ran" | Use `--wait` with a healthcheck |
| Third-party actions pinned by mutable tag | Pin to commit SHAs |
| GHCR package private by default; server gets `denied` | Log in with a read-token or make the package public |

---

## 7. Where do you run the containers? Deployment targets

Choose by team size, traffic and how much operations work is acceptable.

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
| Start-up, 1-5 engineers, one or two services | Cloud Run / ECS Fargate / Azure Container Apps, or VM + Compose with a managed database |
| Scale-to-zero needed | Cloud Run or Azure Container Apps |
| Dozens of services, many teams, common platform | Managed Kubernetes (EKS/AKS/GKE) |
| GPU inference at scale | Kubernetes with GPU node pools, or a managed GPU service ([Chapter 8](08-docker-for-ai-and-ml.md)) |
| Strict data residency, on-prem, air-gapped | Self-managed Kubernetes or OpenShift |
| Already using Swarm and it works | Keep Swarm; plan an exit only if features are missing |
| Regulated workload needing vendor support | Cloud vendor platform with a support plan |

### 7.1 Single VM + Docker Compose (do it properly)

A single Linux VM with Compose suits internal tools, small SaaS products, proofs of concept and many production systems. Limits: the VM is a single point of failure, scaling is vertical, and you own patching.

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
# Create a non-root user for deployments
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
```
- `adduser --disabled-password`: user without password login (key-only).
- `usermod -aG docker deploy`: appends (`-a`) the user to the `docker` group (`-G`), allowing Docker without sudo. This is effectively root access.

```bash
# SSH: keys only, no root login
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl reload ssh
```
- Disables password and root logins, then reloads SSH (the service is `sshd` on some distributions). Keep the current session open and test a second login before closing it.

#### Step 2: Firewall, and the Docker-bypasses-ufw trap

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```
- `default deny incoming`: blocks inbound traffic unless allowed. `allow outgoing`: permits outbound. The `allow` lines open SSH, HTTP and HTTPS. `ufw enable` activates the firewall.

> **Docker bypasses ufw.** Published ports (`-p 8080:80`, or `ports:` in Compose) get Docker iptables rules that run before ufw's, so the port can be open even though `ufw status` shows it blocked. Defences:
> 1. Publish internal-only services on loopback (`127.0.0.1:8000:8000`).
> 2. Do not publish database or cache ports; containers on one Compose network reach each other by service name.
> 3. Restrict published ports with rules in the `DOCKER-USER` iptables chain, or use a cloud firewall (security group) in front of the VM.
>
> Verify from outside, for example with `nmap` from another machine.

#### Step 3: Reverse proxy and TLS

A **reverse proxy** receives public traffic and forwards requests to the right container. It terminates **TLS** (the encryption behind `https://`), routes by hostname or path, and can add compression, rate limits and headers.

| Proxy | Strength | Best for |
|---|---|---|
| **Caddy** | Obtains and renews Let's Encrypt certificates automatically with a 3-line config | Small to medium deployments |
| **Traefik** | Discovers containers from Docker labels; automatic certificates | Many changing services on one host |
| **nginx + certbot** | Ubiquitous and flexible; certificates via `certbot` and a renewal timer | Teams that know nginx |

`Caddyfile`:

```
chat.example.com {
    encode zstd gzip
    reverse_proxy api:8000
}
```
- `chat.example.com { ... }`: a site block. For a real domain, Caddy requests and renews a free Let's Encrypt certificate and redirects HTTP to HTTPS. The DNS A record must point to the server and ports 80/443 must be reachable.
- `encode zstd gzip`: compresses responses.
- `reverse_proxy api:8000`: forwards to the Compose service `api` on port 8000 (Docker internal DNS, [Chapter 5](05-storage-and-networking.md)).

Compose addition:

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
- `caddy:2`: official image, 2.x line. Pin a full version (for example `caddy:2.10`) in real deployments. Unverified: the newest 2.x minor; check Docker Hub.
- `443:443/udp`: HTTP/3 (QUIC).
- `./Caddyfile:/etc/caddy/Caddyfile:ro`: read-only bind mount of the config.
- `caddy_data`: must be a persistent volume. It stores certificates and account keys; without it Caddy re-requests certificates on restart and hits Let's Encrypt rate limits.
- `depends_on ... service_healthy`: starts Caddy after `api` is healthy.

#### Step 4: Start on boot, restart on crash

```bash
sudo systemctl enable --now docker
systemctl is-enabled docker
```
- `enable --now`: starts Docker at every boot (`enable`) and immediately (`--now`). `is-enabled` should print `enabled`.

Containers with `restart: unless-stopped` return when the daemon starts. To make the stack a system service, add `/etc/systemd/system/chat-api.service`:

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
- `[Unit]`: start after Docker and the network.
- `Type=oneshot` with `RemainAfterExit=yes`: the command runs once and the service stays active (Compose runs containers in the background, so systemd has no process to watch).
- `ExecStart`: brings the stack up with the last deployed tag. `ExecStop`: brings it down (optional; many omit it so `systemctl restart` does not stop everything).
- Enable: `sudo systemctl daemon-reload && sudo systemctl enable chat-api`.
- With `restart: unless-stopped` already set, this unit is optional.

#### Step 5: Patching the host

**Unattended upgrades** installs security updates automatically on Debian/Ubuntu:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```
- The first command installs the tool. The second enables daily automatic security updates (`-plow`: ask only low-priority-and-above questions).

Caveats:
- Kernel updates need a reboot. Enable `Unattended-Upgrade::Automatic-Reboot` for a quiet window, or schedule reboots.
- An automatic `docker-ce` upgrade restarts the daemon and, without `live-restore`, all containers. Many teams exclude Docker packages (`Unattended-Upgrade::Package-Blacklist`) and upgrade deliberately ([Section 18](#18-upgrading-docker-engine)).

**Watchtower** (polls for new image versions and restarts containers) is no longer a safe default. The `containrrr/watchtower` repository was archived on 17 December 2025 (last release v1.7.1), and the maintainers said they could not vouch for forks. Unverified: no fork could be checked against primary sources. Treat forks as third-party code with Docker socket access (root-equivalent). Prefer the pipeline in Section 6, where CI builds, scans and tells the server which version to run.

#### Step 6: Backups

Back up state, not containers. See [Section 14](#14-persistence-and-backups). Minimum: nightly database dump plus volume archive, shipped off the VM to object storage, with a tested restore.

#### What "good enough" looks like for a single VM

| Requirement | How |
|---|---|
| Survive container crash | `restart: unless-stopped`, healthchecks |
| Survive reboot | Docker enabled at boot |
| Survive VM loss | Infrastructure as code (Terraform) or a documented rebuild, off-site backups, managed DB |
| HTTPS | Caddy/Traefik/certbot |
| No-downtime deploys | Blue/green with the proxy (Section 8) |
| Patching | Unattended upgrades and scheduled reboots |
| Monitoring | External uptime check plus Prometheus/Grafana or a hosted agent |

### 7.2 Docker Swarm: status and when to use it

**Docker Swarm mode** is a clustering feature in Docker Engine. Machines join a **swarm** as *managers* (store state, make decisions) or *workers*. **Services** (an image plus replicas) are scheduled across machines with restarts, load balancing and rolling updates.

```bash
docker swarm init --advertise-addr 10.0.0.5
docker stack deploy -c compose.prod.yaml chat
docker service ls
docker service update --image ghcr.io/acme/chat-api:1.4.3 chat_api
docker service update --rollback chat_api
```
- `docker swarm init --advertise-addr 10.0.0.5`: makes this machine the first manager and sets the IP other nodes use to reach it.
- `docker stack deploy -c compose.prod.yaml chat`: deploys a Compose-format file as a **stack** named `chat`. In Swarm the `deploy:` section (replicas, `update_config`, `rollback_config`, `placement`) is fully honoured.
- `docker service ls`: lists services and replica counts.
- `docker service update --image ...`: rolling update. `--rollback` returns to the previous spec.

**Status (October 2026).** Swarm is not formally deprecated: Docker's docs have no end-of-life banner and Engine 29.x release notes still list Swarm fixes. Mirantis (which acquired Docker's enterprise business in 2019) reportedly committed in 2025 to support Swarm in Mirantis Kubernetes Engine 3 through at least 2030. Unverified: secondary sources only, and one blog reported a Mirantis corporate acquisition that could not be corroborated. Swarm is widely regarded as being in maintenance mode: security and bug fixes continue, significant new features are not expected, and the ecosystem has moved to Kubernetes.

Use Swarm when:
- A small team comfortable with Compose needs 2-5 machines, rolling updates and secrets without Kubernetes complexity.
- Edge or on-prem sites have limited resources.

Avoid it for new projects that may need autoscaling, a large ecosystem (service mesh, operators, GitOps) or managed offerings. `live-restore` is not compatible with Swarm mode.

### 7.3 A reality check before Kubernetes

Kubernetes (**K8s**) is the standard container orchestrator and brings significant complexity. Without a concrete reason (many services, many teams, autoscaling, network policy, multi-cloud, a platform team), a managed serverless container service or a single VM costs less in people-time.

### 7.4 Kubernetes for Docker users

An **orchestrator** runs containers across a fleet of machines and keeps reality matching a description ("3 copies of this container, reachable at this URL"). Describing desired state and letting the system converge is **declarative** management.

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

Kubernetes removed its built-in Docker integration (dockershim) in v1.24 and runs containers through **containerd** or CRI-O. Docker-built images still work because images follow the OCI standard.

#### Vocabulary mapped from Docker to Kubernetes

| Kubernetes term | Meaning | Docker/Compose analogy |
|---|---|---|
| **Pod** | Smallest unit: one or more tightly coupled containers sharing a network address and storage. Usually one app container. Disposable | A container (or group sharing a network namespace) |
| **Deployment** | Manages identical pods: replica count, image, rollout and rollback | Compose service with `replicas` plus update logic |
| **ReplicaSet** | Used by a Deployment to keep N pods alive. Rarely touched directly | (internal) |
| **Service** | Stable internal address and load balancer in front of changing pods | Compose service name resolved by Docker DNS |
| **Ingress** / **Gateway API** | Rules that route external HTTP(S) traffic to Services by host and path, usually with TLS | Reverse proxy (Caddy/nginx) |
| **ConfigMap** | Non-secret configuration as environment variables or files | `environment:` / `env_file` / config files |
| **Secret** | Sensitive configuration, injected like a ConfigMap. By default only base64-encoded, **not encrypted**; enable encryption at rest or use an external secrets manager | Compose `secrets:` |
| **Namespace** | Virtual partition of the cluster for resources, quotas and permissions | Compose project name |
| **HPA** (Horizontal Pod Autoscaler) | Changes replica count based on CPU or custom metrics | None |
| **Volume / PersistentVolumeClaim (PVC)** | Storage that outlives a pod; a PVC requests storage from the cluster | Docker volume |
| **StatefulSet** | Like a Deployment for stateful apps needing stable names and storage | None |
| **Job / CronJob** | Run-to-completion tasks / scheduled tasks | `docker run` one-offs, cron |
| **Node** | A machine in the cluster | A Docker host |
| **kubectl** | The CLI | `docker` CLI |
| **Helm** | Package manager for Kubernetes manifests (charts) | Compose + templating |

Current state (verified October 2026): Kubernetes **1.37** is the latest stable minor (released August 2026). 1.36 and 1.35 are still supported. 1.38 is scheduled for December 2026. Roughly the three most recent minors are supported, so clusters need regular upgrades.

> **Ingress news.** The community `ingress-nginx` controller (`kubernetes/ingress-nginx`) was **retired in March 2026** and receives no releases or security fixes. The `Ingress` API object still exists (frozen). Other controllers (Traefik, Contour, cloud load-balancer controllers, F5's NGINX controller) keep working. The successor API is **Gateway API**. For new clusters, use a maintained controller or Gateway API.

#### A Compose file and its Kubernetes equivalent

Compose:

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

Kubernetes equivalent (several objects separated by `---`). Save as `chat-api.yaml`:

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

| Object | Notes |
|---|---|
| `apiVersion` / `kind` / `metadata` | Every object states its API group and version, type and name. `namespace: chat` places it in the `chat` namespace |
| Namespace | Creates the partition |
| ConfigMap `data` | Plain key/value configuration, exposed as environment variables through `envFrom` |
| Secret `stringData` | Values are written in plain text in the file; Kubernetes stores them encoded. The value shown is a placeholder. Never commit real secrets in a manifest; use Sealed Secrets, SOPS or External Secrets Operator |
| Deployment `replicas: 3` | Three identical pods |
| `selector.matchLabels` and `template.metadata.labels` | Link the Deployment to its pods by label `app: chat-api` |
| `strategy: RollingUpdate`, `maxSurge: 1`, `maxUnavailable: 0` | During an update, start one extra new pod first and remove an old pod only when a new one is Ready (zero downtime) |
| `envFrom` | Injects all ConfigMap and Secret keys as environment variables |
| `resources.requests` | What the scheduler reserves for node selection. `250m` = 250 millicores (a quarter core); `256Mi` = 256 mebibytes |
| `resources.limits` | Hard cap. Exceeding memory gets the container OOM-killed; exceeding CPU throttles it |
| `readinessProbe` | If it fails, the pod is removed from the Service endpoints but not restarted |
| `livenessProbe` | If it fails repeatedly, the kubelet restarts the container |
| `securityContext` | Non-root, no privilege escalation, read-only filesystem (the app must write only to mounted volumes or `emptyDir` `/tmp`) |
| Service | Stable virtual IP and DNS name (`chat-api.chat.svc.cluster.local`, or `chat-api` inside the namespace). `port: 80` is the client port; `targetPort: 8000` is the container port |
| HPA | Keeps average CPU utilisation (relative to the request) near 70% by scaling between 3 and 10 replicas. Needs the metrics-server add-on, which most managed clusters provide |
| Ingress | Routes `https://chat.example.com/` to the Service through the ingress controller (`traefik` is an example class), using the TLS certificate in secret `chat-example-tls` (commonly created by **cert-manager**, which automates Let's Encrypt certificates) |

Apply and watch:

```bash
kubectl apply -f chat-api.yaml
kubectl -n chat get pods
kubectl -n chat rollout status deployment/chat-api
kubectl -n chat rollout undo deployment/chat-api
```
- `kubectl apply -f`: creates or updates everything in the file to match it.
- `kubectl -n chat get pods`: lists pods in namespace `chat`.
- `rollout status`: blocks until the rollout finishes or fails. `rollout undo`: rolls back to the previous ReplicaSet.

Example `get pods` output:

```
NAME                        READY   STATUS    RESTARTS   AGE
chat-api-7d9f8c6b5d-4kq2x   1/1     Running   0          2m
chat-api-7d9f8c6b5d-9mzt7   1/1     Running   0          2m
chat-api-7d9f8c6b5d-xk8vw   1/1     Running   0          2m
```

#### Helm in five minutes

**Helm** is the package manager for Kubernetes. A **chart** is a folder of templated manifests plus a `values.yaml` of defaults.

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
- `helm install <release-name> <chart>`: renders templates with values and applies them. The result is a **release**. `--set image.tag=1.4.2` overrides a value. `--create-namespace` creates the namespace if missing.
- `helm upgrade`: applies a new version. `helm rollback chat 1`: returns to revision 1. `helm list`: shows releases.
- Most third-party software (Prometheus, Postgres, cert-manager) ships as a Helm chart.

#### GitOps (Argo CD / Flux)

**GitOps** keeps the cluster's desired state in a git repository. An agent inside the cluster (**Argo CD** or **Flux**, both CNCF projects) pulls the repository and makes the cluster match it. To deploy, merge a commit that changes the image tag. To roll back, revert the commit. Benefits: audit trail in git, no cluster credentials in CI, and drift correction (manual cluster changes are undone).

#### Managed Kubernetes

| Service | Cloud | Notes |
|---|---|---|
| **Amazon EKS** | AWS | Managed control plane. Workers as managed node groups, Fargate profiles or Karpenter-managed nodes |
| **Azure AKS** | Azure | Free control-plane tier available. Integrates with Entra ID, ACR, Azure Monitor |
| **Google GKE** | GCP | *Autopilot* mode (Google manages nodes; billing per pod resource) and Standard mode |
| **Red Hat OpenShift** | Anywhere, or managed on clouds | Kubernetes distribution with a stricter security posture and extra developer tooling |
| **k3s / k3d / kind / minikube** | Local, edge, labs | Lightweight Kubernetes for learning and testing |

Managed Kubernetes still leaves cluster upgrades (several per year), node patching, ingress/Gateway, certificates, secrets, monitoring, cost control and security policy to you.

### 7.5 Serverless containers

Serverless containers: the platform runs, scales and bills an image without the user managing machines or a cluster.

| Service | Cloud | Model | Scale to zero | Notes |
|---|---|---|---|---|
| **Google Cloud Run** | GCP | Request-driven containers (and jobs); min/max instances | Yes | Simple developer experience. GPUs supported (NVIDIA L4 reported GA in 2026). Container must listen on `$PORT` |
| **AWS ECS on Fargate** | AWS | ECS schedules tasks on AWS-managed capacity | No (service keeps its desired task count; zero tasks possible via autoscaling or scheduled actions, not request-driven) | Standard "containers without Kubernetes" on AWS. No native GPU; use ECS on EC2 or EKS |
| **AWS App Runner** | AWS | Source/image to HTTPS service | n/a | **Closed to new customers on 30 April 2026**. AWS recommends **ECS Express Mode**. Existing customers keep access with no new features |
| **ECS Express Mode** | AWS | One API call provisions a Fargate-based ECS service, load balancer and auto scaling | No | AWS's replacement for App Runner. Newer service: read current AWS docs |
| **Azure Container Apps** | Azure | Built on Kubernetes + KEDA (event-driven autoscaling) + Dapr; consumption and dedicated workload profiles | Yes (consumption plan) | Microservices and event-driven jobs. GPU via workload profiles |

Limitations: cold starts when scaling from zero (image size matters), request timeouts, limited local disk (use object storage), no privileged containers, limited sidecar flexibility, and possibly worse economics for long-running GPU model servers than reserved instances.

Spiky or low traffic favours Cloud Run or Container Apps (nothing paid when idle). Steady traffic favours Fargate or a VM. Pricing changes and comparisons found disagreed; use the vendors' current calculators.

### 7.6 PaaS (Platform as a Service)

A **PaaS** takes code or an image and handles builds, TLS, scaling, logs and database add-ons.

| Platform | Style | Good for |
|---|---|---|
| **Fly.io** | Runs Docker images as lightweight VMs in many regions (`fly launch`, `fly deploy`) | Global low-latency apps, simple multi-region |
| **Render** | Git-connected web services, managed Postgres/Redis, Docker or native build | Small teams, predictable pricing |
| **Railway** | Git-connected deployments, UI-driven; Docker or auto-detected builds | Prototypes, early-stage projects |
| **Heroku** | Original PaaS (container registry and `heroku.yml` support) | Existing Heroku users |

Trade-offs: fast start and little operations work, but higher cost at scale, less control over networking and security, and vendor-specific features (lock-in is mild because the artifact is a standard Docker image). Unverified: no specific price or free tier was checked for this chapter.

---

## 8. Deployment strategies and rollbacks

A **deployment strategy** is the plan for replacing the old version with the new one. Goals: no user-visible downtime, a fast way back, and limited impact if the new version is bad.

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Recreate** | Stop old, start new | Simplest | Downtime. Fine for dev, internal tools, or when two versions cannot coexist (some database migrations) |
| **Rolling update** | Replace instances one by one or in small batches | No downtime, no extra environment | Old and new run side by side, so they must be compatible. Rollback is another rolling update |
| **Blue/green** | Run the full new stack ("green") beside the old ("blue"), test it, switch the router in one step | Instant switch and rollback | Needs double capacity during the switch |
| **Canary** | Send 1-5% of traffic to the new version, watch metrics, increase gradually | Limits damage; real-user testing | Needs traffic splitting and good metrics |
| **Shadow** | Copy real traffic to the new version without returning its responses | Safe load testing | Complex; side effects must be avoided |

```
 ROLLING                          BLUE/GREEN                         CANARY
 [v1][v1][v1][v1]                 router -> [v1 ×4]  (blue, live)    router -> 95% -> [v1 ×4]
 [v2][v1][v1][v1]                 [v2 ×4] (green, idle, tested)              5% -> [v2 ×1]
 [v2][v2][v1][v1]                 router -> [v2 ×4]  (switch!)       metrics OK? -> 25% -> 100%
 [v2][v2][v2][v2]                 [v1 ×4] kept for rollback          metrics bad? -> back to 0%
```

### 8.1 Zero-downtime on a single VM with Compose

Compose recreates a changed container (stop old, start new), which causes a short gap.

**Option A: Blue/green behind Caddy.** Run `api_blue` and `api_green`; Caddy sends traffic to one.

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
- `profiles` ([Chapter 6](06-docker-compose.md)): a service starts only when its profile is requested, so the idle colour can be started independently.
- `${BLUE_TAG:-none}`: uses `none` when unset, so Compose can parse the file.

Procedure:

1. Start the idle colour (green) with the new tag: `GREEN_TAG=sha-new docker compose --profile green up -d --wait`.
2. Smoke-test it directly, for example `docker compose exec caddy wget -qO- http://api_green:8000/health`.
3. Change the Caddyfile to `reverse_proxy api_green:8000` and reload: `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile`. A reload swaps configuration without dropping in-flight connections.
4. Keep blue running for some minutes. If metrics are bad, switch the Caddyfile back and reload (instant rollback).
5. Stop blue: `docker compose --profile blue stop api_blue`.

**Option B: Scale out, then scale in.** With a proxy that discovers containers (Traefik), run `docker compose up -d --no-deps --scale api=2 --no-recreate api` to add a second instance, wait for health, then remove the old one. The community plugin **docker-rollout** automates this. Unverified: its current maintenance status; evaluate before relying on it.

**Option C: Use an orchestrator.** Kubernetes `RollingUpdate`, Swarm `update_config`, ECS rolling/blue-green deployments (CodeDeploy) and Cloud Run revisions with traffic splitting provide this without scripts. For example, Cloud Run can route 5% of traffic to a new **revision** and 95% to the old one, then shift gradually.

### 8.2 Requirements for zero-downtime to actually work

- **Graceful shutdown.** Docker sends **SIGTERM**, waits 10 seconds by default (`stop_grace_period` / `--stop-timeout`), then sends **SIGKILL**. The app must catch SIGTERM, stop accepting requests, finish in-flight ones and exit. It must be PID 1 or run under an init (`init: true` in Compose, `docker run --init`) and use the exec form of `CMD` ([Chapter 4](04-dockerfile-deep-dive.md)) so signals reach it.
- **Readiness.** Traffic goes only to ready instances (Section 10).
- **Backward-compatible database migrations.** During rolling or blue/green deploys v1 and v2 share the database. Use **expand/contract**: add the new column (compatible with v1), deploy v2, remove the old column later.
- **Idempotent, stateless services** (Section 15).

### 8.3 Rollbacks

| Platform | Command |
|---|---|
| Compose + script | Re-run `deploy.sh` with the previous tag (in `.previous_tag`) |
| Swarm | `docker service update --rollback SERVICE` |
| Kubernetes | `kubectl rollout undo deployment/NAME` (or revert the commit in GitOps) |
| Helm | `helm rollback RELEASE REVISION` |
| Cloud Run | Send 100% of traffic to the previous revision |
| ECS | Update the service to the previous task definition revision |

Prerequisites:

1. Old images still exist in the registry (retention policy exempts deployed versions).
2. Configuration is versioned with the image or compatible with it.
3. Database changes are reversible or backward-compatible. A dropped column cannot be restored.
4. A rollback drill is run every quarter.

| Mistake | Fix |
|---|---|
| No health check, so "deployed" means "started" | Add health checks and wait for them |
| Destructive migration in the same release as the code needing it | Use expand/contract |
| One replica called highly available | Run at least two, or accept the risk explicitly |
| Stop timeout shorter than shutdown time | Raise `stop_grace_period` |
| Rollback relying on `latest` | Roll back to a specific tag or digest |

---

## 9. Configuration and secrets in production

### 9.1 The Twelve-Factor App, the useful parts

The **Twelve-Factor App** is a set of guidelines (from Heroku's founders) for apps that run well on modern platforms. The relevant factors:

| Factor | Meaning |
|---|---|
| **III. Config** | Store configuration in the environment, not in code or the image. The same image runs in dev, staging and prod |
| **IV. Backing services** | Treat databases, caches and queues as attached resources reached through a URL in config |
| **VI. Processes** | Run stateless processes; persistent data goes to a backing service |
| **IX. Disposability** | Start fast and shut down gracefully |
| **X. Dev/prod parity** | Keep environments as similar as possible |
| **XI. Logs** | Treat logs as an event stream: write to stdout/stderr and let the platform collect them |

### 9.2 Environment variables

```bash
docker run -e LOG_LEVEL=info -e API_BASE_URL=https://api.example.com chat-api:1.4.2
```
- `-e NAME=value`: sets a variable inside the container.

Risks when used for secrets:

- `docker inspect CONTAINER` prints all environment variables to anyone with Docker access.
- Child processes inherit them; crash dumps and debug endpoints (`/env`, framework error pages) may expose them.
- They can leak into logs if the app prints its config at startup.
- `ENV` or `ARG` in a Dockerfile is stored in image layers permanently and visible with `docker history`. Never put secrets in a Dockerfile.

### 9.3 Docker secrets / file-based secrets

Mount secrets as read-only files under `/run/secrets/`.

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
- Top-level `secrets:` declares a secret whose content comes from a local file. `secrets: - db_password` in the service mounts it at `/run/secrets/db_password` (in-memory read-only file in Swarm; bind-mounted file in plain Compose).
- `DB_PASSWORD_FILE`: a convention (used by the official Postgres and MySQL images) where the app reads the path from an environment variable and loads the password from that file. The app or entrypoint must support it.
- Create `./secrets/db_password.txt` with `chmod 600`, keep it out of git, and deliver it with secret tooling.

### 9.4 Secrets managers

| Tool | Where | Notes |
|---|---|---|
| **AWS Secrets Manager** / **SSM Parameter Store** | AWS | ECS and EKS can inject values at start. Secrets Manager supports automatic rotation |
| **Azure Key Vault** | Azure | Container Apps and AKS can reference secrets |
| **Google Secret Manager** | GCP | Cloud Run can mount secrets as env vars or files |
| **HashiCorp Vault** (or open-source fork OpenBao) | Anywhere | Dynamic short-lived credentials (for example a database user valid for one hour) |
| **SOPS** + age/KMS | Anywhere | Encrypts secret files so the encrypted version can be committed; decrypt at deploy time |
| **External Secrets Operator / Secrets Store CSI Driver** | Kubernetes | Syncs secrets from the managers above into pods |
| **Doppler, 1Password Secrets Automation, Infisical** | SaaS | Managed options |

Principles:

- **Least privilege**: each service reads only its own secrets.
- **Rotation**: make rotation routine and automated.
- **No long-lived cloud keys in CI.** Use **OIDC federation**: GitHub Actions requests a short-lived token (`id-token: write`) that the cloud trusts for a specific repository and branch.
- **Scan for leaked secrets** in git history with `gitleaks` or `trufflehog`, and enable GitHub push protection.
- `.env` files and keys never go into git ([Chapter 13](13-interview-questions-roadmap-glossary.md) covers this in the portfolio guide).

### 9.5 Configuration per environment with Compose

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
```
- Multiple `-f` files merge left to right; later files override earlier ones. `config` prints the final merged configuration with variables substituted, without running anything.

| Mistake | Fix |
|---|---|
| Secrets baked in with `ENV`/`ARG`/`COPY .env` | Build secrets and runtime secret files |
| Committed `.env` files | `.gitignore`, push protection |
| One shared admin secret for all services | Per-service secrets |
| Never rotating | Automate rotation |
| Printing configuration at startup | Redact secrets in logs |

---

## 10. Health checks: liveness vs readiness

A **health check** is a command or HTTP request the platform runs periodically to test whether the app works. Without one, the platform only knows whether the process exists.

| Probe | Question | If it fails | Typical check |
|---|---|---|---|
| **Liveness** | Is the app stuck or broken? | Container is **restarted** | A cheap `/health` endpoint returning 200 if the process can respond. Do not check the database here, or a database outage restarts every instance |
| **Readiness** | Can the app handle traffic now? | Instance is **removed from load balancing** (not restarted) | `/ready` verifies dependencies (database reachable, cache warmed, migrations done) |
| **Startup** | Has the app finished starting? | Liveness/readiness are paused until it passes; if it never passes, the container is restarted | Slow-starting apps (loading a large ML model can take minutes) |

```
 request flow                         health logic
 -------------                        ------------
 Load balancer -> only Ready pods     readiness fails -> pod removed from rotation (no restart)
 kubelet / Docker -> restarts         liveness fails  -> container restarted
```

### Docker's built-in HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
```

| Option | Meaning |
|---|---|
| `--interval=30s` | Run the check every 30 seconds |
| `--timeout=3s` | A check slower than this counts as a failure |
| `--start-period=40s` | Failures during the first 40 seconds do not count |
| `--retries=3` | Mark `unhealthy` after three consecutive failures |
| `CMD ...` | The check command. Exit code 0 is healthy, 1 is unhealthy |
| `\|\| exit 1` | Converts any failure to exit code 1 |

```bash
docker ps
```
```
CONTAINER ID   IMAGE              STATUS                    NAMES
a1b2c3d4e5f6   chat-api:1.4.2     Up 2 minutes (healthy)    chat-api
```
- `STATUS` shows `(healthy)`, `(unhealthy)` or `(health: starting)`.

Plain Docker does not restart an unhealthy container; it only labels it. Compose `restart:` restarts only on process exit. Swarm, Kubernetes and ECS replace unhealthy tasks. On a single VM, combine the healthcheck with `depends_on: condition: service_healthy`, `docker compose up --wait` and an external monitor. An "autoheal" sidecar can restart unhealthy containers but needs the Docker socket, so weigh the risk.

| Mistake | Fix |
|---|---|
| Healthcheck uses `curl` in an image without it (always `unhealthy`) | Use a tool present in the image (for example Python) |
| Liveness checks the database (restart storms during a DB outage) | Check dependencies in readiness only |
| No `start_period` for slow starters | Set `start_period` |
| Endpoint always returns 200 | Test something meaningful for readiness |

---

## 11. Resource limits and the OOM killer

By default a container can use all CPU and memory on the host.

```bash
docker run -d --name api \
  --memory=512m --memory-swap=512m \
  --cpus=1.5 \
  --pids-limit=200 \
  chat-api:1.4.2
```
- `--memory=512m`: hard memory cap (512 MiB). Beyond it the kernel's **OOM killer** (the component that kills processes when memory runs out) terminates the container.
- `--memory-swap=512m`: memory plus swap equal to memory, so no swap.
- `--cpus=1.5`: at most one and a half cores of CPU time.
- `--pids-limit=200`: caps processes/threads (fork-bomb protection).

Compose:

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
- `deploy.resources.limits`: Compose-spec form, honoured by current `docker compose`. `reservations` is a soft guarantee used for scheduling in orchestrators.

### Recognising an OOM kill

```bash
docker inspect api --format '{{.State.OOMKilled}} {{.State.ExitCode}}'
```
- Prints for example `true 137`. Exit code **137** = 128 + 9 (SIGKILL). `OOMKilled=true` confirms the memory limit. Also check `dmesg | grep -i oom` on the host ([Chapter 12](12-troubleshooting-and-cheatsheet.md)).

### Sizing guidance

1. Measure first with `docker stats`.
2. Set the memory limit 20-50% above observed peak. Memory limits are hard; CPU limits throttle.
3. In Kubernetes set `requests` (normal need) and `limits` (maximum). Memory request equal to limit is most predictable. Many teams set CPU requests only, to avoid throttling; this is debated.
4. Tune the runtime to the limit (JVM `-XX:MaxRAMPercentage`, Node `--max-old-space-size`, Gunicorn worker counts), or it may assume the whole host's memory.
5. For ML inference, the limit must exceed model size plus overhead ([Chapter 8](08-docker-for-ai-and-ml.md)).

| Mistake | Fix |
|---|---|
| No limits on shared hosts | Set memory, CPU and PID limits |
| Limit below real peak (random exit 137) | Size from measurements |
| Ignoring the runtime's memory model | Tune the runtime to the limit |
| Total limits exceed the machine | Make the sum fit the host |

---

## 12. Logging

### 12.1 The container logging model

Containers write to **stdout** and **stderr**, not log files. Docker captures both streams and passes them to a **logging driver** (a plug-in that decides where logs go). `docker logs` reads them.

```
 app prints to stdout/stderr -> Docker daemon captures -> logging driver -> destination
                                                         json-file / local -> files on host
                                                         journald          -> systemd journal
                                                         awslogs, gcplogs, splunk, fluentd, gelf, syslog -> remote services
```

### 12.2 The default driver has a trap

The default driver, **`json-file`**, writes one JSON line per message to `/var/lib/docker/containers/<id>/<id>-json.log` and **never rotates by default**. A chatty container can fill the disk and take down the host. Drivers listed in the Docker docs: `none`, `local`, `json-file`, `syslog`, `journald`, `gelf`, `fluentd`, `awslogs`, `splunk`, `etwlogs` (Windows only), `gcplogs`, plus plug-ins.

| Driver | Destination | Notes |
|---|---|---|
| `json-file` | JSON files on the host | Default. No rotation unless configured. Works with `docker logs` |
| `local` | Compact binary files on the host | Rotates by default (20 MB x 5 files), lower overhead. Docker recommends it for most non-Kubernetes uses. Works with `docker logs` |
| `journald` | systemd journal | If the journal is already centralised |
| `syslog` | A syslog server | Traditional |
| `fluentd` / `gelf` | Fluentd/Fluent Bit or Graylog/Logstash | ELK-style pipelines |
| `awslogs` | Amazon CloudWatch Logs | AWS |
| `gcplogs` | Google Cloud Logging | GCP |
| `splunk` | Splunk HTTP Event Collector | Splunk shops |
| `none` | Nowhere | Disables logs and `docker logs` |

With some remote drivers `docker logs` stops working for that container. Newer Engine versions use "dual logging" so it keeps working; verify on your version.

### 12.3 Rotation settings you should always set

Per container:

```bash
docker run -d --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 chat-api:1.4.2
```
- `--log-driver json-file`: selects the driver.
- `--log-opt max-size=10m`: rotates at 10 MB.
- `--log-opt max-file=3`: keeps at most 3 files (about 30 MB per container).

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
- Restart the daemon (`sudo systemctl restart docker`). The new defaults apply only to containers created afterwards.

Optional: `"mode": "non-blocking"` with `"max-buffer-size": "4m"` in `log-opts` prevents a slow log destination from blocking application writes, at the cost of dropped messages if the buffer fills. The default mode is `blocking`.

### 12.4 Good logging practice

- Use structured logs (JSON, one object per line with `level`, `timestamp`, `request_id`).
- Pass a correlation/request ID through all services.
- Never log secrets or personal data (tokens, passwords, full prompts that may hold private data in AI chat apps).
- Use `LOG_LEVEL=info` in production; `debug` only temporarily.
- Use UTC.

### 12.5 Centralised logging

On one VM `docker logs` is enough. With several machines, centralise logs.

| Stack | Pieces | Notes |
|---|---|---|
| **Grafana Loki** | Loki (indexes only labels, cheap) + collector + Grafana | Collectors: **Grafana Alloy** (current recommended agent) or the Docker log-driver plug-in. **Promtail** was deprecated in favour of Alloy and reached end of life in 2026 (reported; verify on Grafana's site) |
| **ELK / EFK** | Elasticsearch + Logstash (or Fluentd/Fluent Bit) + Kibana | Full-text search; heavier. OpenSearch is the open-source fork |
| **CloudWatch Logs** | `awslogs` driver, CloudWatch agent or Fluent Bit | Native on AWS; ECS/Fargate integrates directly |
| **Cloud Logging** (GCP), **Azure Monitor / Log Analytics** | Built into Cloud Run/GKE and Container Apps/AKS | No setup on those platforms |
| **Datadog, Splunk, New Relic, Elastic Cloud** | Commercial SaaS | Fast to adopt; log volume is billed |

**Fluent Bit** is a lightweight collector that runs as a container, reads Docker's log files and ships them anywhere. In Kubernetes a collector typically runs as a **DaemonSet** (one pod per node) reading `/var/log/containers`.

Logs are the most expensive observability data. Set retention (for example 14-30 days hot, longer archived for compliance), sample noisy debug logs and drop health-check request logs.

| Mistake | Fix |
|---|---|
| Unrotated `json-file` | Set `max-size` and `max-file`, or use `local` |
| Log files inside the container | Write to stdout/stderr |
| Logging secrets | Redact |
| No timestamps or IDs | Structured logs with request IDs |
| Lost logs in a crash because the driver was blocking on an unreachable destination | Consider `non-blocking` mode |

---

## 13. Monitoring, alerting and tracing

**Metrics** are numbers over time (CPU, requests per second, error rate, latency). **Traces** follow one request across services. Logs record what happened; metrics show current health and drive alerts.

```
 containers/host --> exporters --> Prometheus (scrapes & stores) --> Grafana (dashboards)
                                              |
                                              +--> Alertmanager --> email / Slack / PagerDuty
```

### 13.1 The standard open-source stack

| Component | Role |
|---|---|
| **Prometheus** | Time-series database that **scrapes** metrics from HTTP endpoints every few seconds; queried with PromQL |
| **cAdvisor** (from Google) | Container exposing per-container CPU, memory, network and disk metrics |
| **node_exporter** | Host-level metrics (CPU, memory, disk, network) |
| **Docker Engine metrics endpoint** | Set `"metrics-addr": "127.0.0.1:9323"` in `daemon.json`. If it does not come up on your version, check the Docker docs for extra flags |
| **Grafana** | Dashboards and alerting |
| **Alertmanager** | Deduplicates, groups and routes alerts |
| **Blackbox exporter / Uptime Kuma / external pingers** | Check from outside that the public URL works |

`compose.monitoring.yaml`:

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
- `latest` is used only to keep the example short. Pin versions in real use (Section 4 applies to third-party images). Verify image locations (`gcr.io/cadvisor/cadvisor`, `prom/prometheus`) against each project's docs; cAdvisor's location has changed before.
- `cadvisor` mounts host directories read-only (`:ro`) to see container resource usage. Do not expose its port publicly; `expose` keeps it internal to the Compose network.
- `node_exporter` uses `pid: host` and a read-only `/` mount to read real host metrics.
- `prometheus` and `grafana` bind to `127.0.0.1`. Reach them through an SSH tunnel or an authenticated reverse proxy.
- Volumes persist time-series data and Grafana dashboards.

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
- `scrape_interval: 15s`: collect every 15 seconds. Each `job_name` lists targets (host:port). `api` assumes the app exposes `/metrics` (the Python `prometheus-client` and most frameworks support this).

### 13.2 What to measure: the golden signals

Google's SRE book defines four **golden signals**:

| Signal | Meaning | Example alert |
|---|---|---|
| **Latency** | Request duration (watch p95/p99, not only the average) | p95 > 800 ms for 5 minutes |
| **Traffic** | Request rate | Drop to zero (outage) or 10x spike |
| **Errors** | Rate of failed requests (5xx) | Error rate > 2% for 5 minutes |
| **Saturation** | How full a resource is: CPU, memory, disk, connection pools | Disk > 85%, memory > 90% of limit |

Also alert on: rising container restart count, `unhealthy` containers, certificates expiring within 14 days, failed backup jobs, growing queue depth. For AI apps add tokens per second, time-to-first-token, GPU utilisation and memory, queue depth and model-load time.

Every alert should be actionable and urgent; noisy alerts cause alert fatigue. Page humans for symptoms users feel (errors, latency, downtime). Send causes (high CPU) to a chat channel.

### 13.3 Tracing and OpenTelemetry

A **trace** records one request's path through services (API, cache, model server, database) as a tree of timed **spans**. **OpenTelemetry (OTel)** is the vendor-neutral CNCF standard for generating and exporting traces, metrics and logs. Add the OTel SDK or auto-instrumentation to the app, send data to an **OTel Collector** (receives, processes and forwards telemetry), which forwards to Tempo, Jaeger, Datadog, Honeycomb or another backend. Start with traces for the request path, then add metrics and logs. For LLM apps, OTel-based tools also capture prompts, token counts and per-call latency.

### 13.4 Managed alternatives

Grafana Cloud, Datadog, New Relic, AWS CloudWatch/Container Insights, Azure Monitor, Google Cloud Monitoring. These trade money for time. Small teams often use a hosted solution plus a free external uptime monitor.

---

## 14. Persistence and backups

### 14.1 Containers are disposable, data is not

A container's writable layer disappears when the container is removed ([Chapter 5](05-storage-and-networking.md)). Data that must survive belongs in a **volume**, a bind mount or, better, a service outside the container platform.

### 14.2 Why databases often live outside containers in production

Postgres in a container works and is common for development and tests. In production many teams use a **managed database** (Amazon RDS/Aurora, Azure Database for PostgreSQL, Google Cloud SQL, or Neon, Supabase, Crunchy Bridge).

| Concern | Self-run in a container | Managed database |
|---|---|---|
| Backups and point-in-time recovery | Build and test it | Built in |
| High availability / failover | Build it | Option |
| Patching and upgrades | You | Provider |
| Monitoring | Assemble it | Included |
| Storage resizing | Manual | Automatic |
| Cost | Cheaper in money | Cheaper in people-time |
| Control, extensions, tuning | Total | Limited by provider |

Orchestrators are designed for stateless workloads. Stateful workloads need storage attachment, ordered startup and safe failover. Kubernetes StatefulSets and operators (CloudNativePG, Zalando Postgres Operator) handle this but need expertise. For vector databases and caches in AI apps, use a managed offering (Pinecone, managed Redis, pgvector on managed Postgres) or a carefully backed-up volume.

Self-hosting in a container is acceptable for small apps that tolerate occasional downtime, on-prem or air-gapped systems, cost-sensitive projects, and caches (Redis used only as a cache can be rebuilt).

### 14.3 Backing up what you do run

**3-2-1 rule**: three copies of the data, on two kinds of media/systems, one off-site.

**Database dump (logical backup):**

```bash
docker compose exec -T db pg_dump -U chat -Fc chat > /backups/chat-$(date +%F).dump
```
- `docker compose exec -T db ...`: runs a command in the running `db` service. `-T` disables the pseudo-terminal so output redirection works in scripts.
- `pg_dump -U chat -Fc chat`: dumps database `chat` as user `chat` in **custom format** (`-Fc`: compressed, selectively restorable with `pg_restore`).
- `> /backups/chat-$(date +%F).dump`: writes to a dated file on the host (`date +%F` prints `2026-10-23`).

Restore:

```bash
docker compose exec -T db pg_restore -U chat -d chat --clean --if-exists < /backups/chat-2026-10-23.dump
```
- `--clean --if-exists`: drops existing objects before recreating them. `< file` feeds the dump to standard input.

**Volume backup (file-level):**

```bash
docker run --rm -v chat_pgdata:/data:ro -v /backups:/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tar.gz -C /data .
```
- Starts a temporary Alpine container with volume `chat_pgdata` read-only at `/data` and host `/backups` at `/backup`, and creates a compressed tarball.
- Copying a live database's files can produce an inconsistent backup. For databases use `pg_dump`, stop the database briefly, or use filesystem snapshots plus the database's backup tooling.

**Ship it off the machine** with `rclone`, `restic` or `borg` to object storage (S3, Azure Blob, GCS, Backblaze B2), encrypted. Schedule with a systemd timer or cron:

```bash
0 2 * * * /opt/chat-api/backup.sh >> /var/log/chat-backup.log 2>&1
```
- Cron line: run the backup script at 02:00 daily and append output to a log.

Rules:

1. A backup that has never been restored is unverified. Schedule restore tests.
2. Define **RPO** (Recovery Point Objective: acceptable data loss, for example 15 minutes) and **RTO** (Recovery Time Objective: acceptable downtime, for example 1 hour).
3. Encrypt backups and protect keys separately.
4. Alert if a backup does not run or is unusually small.
5. Keep backups under different credentials than production, to survive ransomware or a compromised admin.

Named volumes live under `/var/lib/docker/volumes`. Treat that path as private to the daemon; use the commands above.

---

## 15. Scaling

### 15.1 Vertical vs horizontal

| | Vertical ("scale up") | Horizontal ("scale out") |
|---|---|---|
| What | A bigger machine | More copies |
| Limit | Largest available machine | Very high |
| Resilience | Single point of failure | Survives losing a copy |
| Complexity | Almost none | Needs stateless design and a load balancer |

### 15.2 Designing for horizontal scale: statelessness

If any instance can serve any request, N instances can run behind a **load balancer** (spreads requests across instances). Services must be **stateless**: no user session in process memory, no local files needed by other instances. Keep state in shared services (Redis for sessions and caches, Postgres for records, object storage for files).

```
              +--> api #1 --+
 users -> LB -+--> api #2 --+--> Redis (sessions/cache)  +--> Postgres
              +--> api #3 --+
```

**Sticky sessions** (session affinity: the load balancer always sends a user to the same instance) work around in-memory state but cause uneven load, make deployments and scale-down harder and hide design problems. Prefer a shared session store. Legitimate uses: WebSocket connections that must stay on one instance, and cache-locality tricks (for example routing a conversation to the replica holding its KV-cache in LLM serving).

### 15.3 How to scale on each platform

| Platform | How |
|---|---|
| Compose | `docker compose up -d --scale api=3` (needs a reverse proxy to balance; do not publish a fixed host port on a scaled service) |
| Swarm | `docker service scale chat_api=5` |
| Kubernetes | Change `replicas`, or use the **HPA** (CPU/memory/custom metrics); **cluster autoscaler** or **Karpenter** adds nodes; **KEDA** scales on queue length |
| Cloud Run / Container Apps | Automatic by concurrent requests; set min/max instances |
| ECS | Service auto scaling with target tracking policies |

Scaling the API from 3 to 30 replicas can overwhelm the database with connections. Use a connection pooler (PgBouncer, RDS Proxy) and limits.

AI inference scales differently: model servers are heavy (GBs of weights, slow to load, GPU-bound). Use pre-pulled images, warm pools, batching and queueing ([Chapter 8](08-docker-for-ai-and-ml.md)).

---

## 16. Docker daemon production configuration (`daemon.json`)

The **Docker daemon** (`dockerd`) is the background service that does the work. On Linux it is configured in `/etc/docker/daemon.json` (create it if missing). The file must be valid JSON or the daemon will not start.

Production baseline:

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

| Key | Effect |
|---|---|
| `"log-driver": "local"` | Default to the compact `local` driver. Or `"json-file"` for plain-text JSON |
| `"log-opts"` | Rotation: at most 5 files of 10 MB per container. The most valuable setting here. Quote the numbers |
| `"live-restore": true` | Containers keep running while the daemon is stopped or restarted. Re-attaches when the daemon returns. Limits: patch-level updates only (not major versions), not Swarm mode, not Windows containers, and during a long outage logs can fill a small pipe buffer and block the container. Apply with a reload (`sudo systemctl reload docker` or `kill -SIGHUP $(pidof dockerd)`) |
| `"default-address-pools"` | IP ranges for new networks. Default pools (172.17.0.0/16 and others, plus 192.168.x.x) often collide with corporate VPNs or cloud VPC ranges, causing unreachable hosts. Here each new network gets a /24 from 10.200.0.0/16 (254 networks). Choose a range that does not overlap real networks. Affects only networks created later |
| `"registry-mirrors"` | Docker Hub pull-through caches tried first (Docker Hub only) |
| `"data-root"` | Where images, containers and volumes are stored (default `/var/lib/docker`). Move to a larger, faster disk (for example mounted at `/data/docker`): stop Docker, `rsync -aP /var/lib/docker/ /data/docker/`, update this key |
| `"metrics-addr"` | Prometheus metrics for the daemon, localhost only |
| `"default-ulimits"` | Default per-process limits (for example `nofile`) for every container |

Apply and verify:

```bash
sudo systemctl restart docker
docker info | grep -E "Logging Driver|Live Restore|Docker Root Dir"
```
- Restart applies the config (log settings affect only new containers). `docker info` prints effective settings:

```
 Logging Driver: local
 Docker Root Dir: /var/lib/docker
 Live Restore Enabled: true
```

Validate before restarting:

```bash
sudo dockerd --validate --config-file=/etc/docker/daemon.json
```
- Checks the file without starting the daemon. Prints `configuration OK` on success.

Other settings: `"userns-remap"` (map container root to an unprivileged host user), `"no-new-privileges": true` (default for all containers), `"icc": false` (disable unrestricted inter-container communication on the default bridge), `"features": {"containerd-snapshotter": true}` (containerd image store, default for new installs in Engine 29), `"insecure-registries"` (avoid; disables TLS verification). Rootless mode and the full hardening list: [Chapter 10](10-security.md).

| Mistake | Fix |
|---|---|
| Invalid JSON (trailing comma) stops the daemon | Run `dockerd --validate` first |
| Expecting log options to change running containers | Recreate containers |
| Address pool overlaps the VPN | Choose a non-overlapping range |
| Editing without a backup | Copy the old file first |

---

## 17. Disk management and cleanup policies

Docker keeps old images, stopped containers, build cache, unused volumes and logs until told otherwise. The first symptom of a full disk is `no space left on device` in production.

```bash
docker system df
```
- Shows space used by images, containers, local volumes and build cache. `RECLAIMABLE` is what a prune would free.

```
TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
Images          23      6        8.41GB    5.92GB (70%)
Containers      9       6        152.3MB   12.6MB (8%)
Local Volumes   11      4        3.2GB     1.1GB (34%)
Build Cache     87      0        4.7GB     4.7GB
```

| Command | Removes | Risk |
|---|---|---|
| `docker image prune` | **Dangling** images (untagged leftovers) | Very low |
| `docker image prune -a --filter "until=168h"` | All images not used by a container and older than 7 days | Low if images can be re-pulled; would delete unused rollback images |
| `docker container prune` | Stopped containers | Low |
| `docker builder prune --filter "until=72h"` | Build cache older than 3 days | Low (builds slow down once) |
| `docker volume prune` | Volumes not used by any container | **High: can delete database data** if a stack is temporarily stopped |
| `docker system prune` | Stopped containers, unused networks, dangling images, build cache (`-a` adds all unused images) | Medium. Never add `--volumes` on a server with data |

Policies for production hosts:

1. Never schedule `docker volume prune` or `docker system prune --volumes`. Volumes of a stopped stack look unused.
2. Schedule a weekly `docker image prune -a --filter "until=168h"` and `docker builder prune --filter "until=168h"` (systemd timer or cron), after confirming rollback images are in a registry.
3. Put `data-root` on a separate disk.
4. Alert at 80% disk usage.
5. Rotate logs ([Section 12](#12-logging)).
6. On CI runners you manage, prune aggressively and automatically. Prefer ephemeral runners.

---

## 18. Upgrading Docker Engine

Docker Engine releases frequently (the machine used for this guide runs 29.9.2). Run a supported version for security fixes.

Engine 29 notes (official release notes): 29.0.0 raised the **minimum API version** the daemon accepts to 1.44 (clients and tools built for Docker older than v25 must be updated; some tools such as Traefik and Portainer needed updates), made the **containerd image store** the default for new installs, removed Docker Content Trust from the CLI, and deprecated cgroup v1. Unverified: one secondary source says a later 29.x release relaxed the API floor. Check your daemon with `docker version`.

Upgrade procedure:

1. Read the release notes for breaking changes (API version, storage driver, deprecations).
2. Test in staging on the same OS and storage configuration.
3. Back up `/etc/docker/daemon.json` and any state; note running containers (`docker ps`).
4. Upgrade through the package manager (signed packages from Docker's apt/dnf repository):
   ```bash
   sudo apt update
   sudo apt install --only-upgrade docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
   ```
   - `--only-upgrade`: upgrades the named packages and never installs them if absent. The packages are the engine, CLI, container runtime, and the Buildx and Compose plug-ins.
5. With `live-restore` and a patch-level upgrade, containers keep running. Otherwise all containers restart: schedule a maintenance window or drain traffic first.
6. Verify: `docker version`, `docker info`, `docker ps` (all containers up and healthy), smoke-test the app.
7. Roll out one host at a time; keep one node on the old version until the others look good.

Pin versions in automation: `apt-mark hold docker-ce` prevents surprise upgrades from unattended-upgrades; upgrade deliberately.

**Docker Desktop** (developer machines) auto-updates and is managed through the app or admin tools.

---

## 19. Cost and capacity planning

### 19.1 Where the money goes

| Cost driver | Notes |
|---|---|
| **Compute** | VMs, nodes, Fargate vCPU-seconds. Largest line. Right-size using metrics from Section 13 |
| **GPU** | Often 10-100x CPU cost per hour; idle GPUs waste money. Autoscale to zero where practical, use spot/preemptible instances for batch, batching and quantised models for inference |
| **Storage** | Registry storage, volumes, snapshots, logs. Apply retention policies |
| **Network egress** | Data leaving the cloud, cross-zone and cross-region traffic. NAT gateways pulling images for private subnets are a common hidden cost |
| **Managed services** | Load balancers, managed DBs, Kubernetes control plane fees |
| **Observability** | Log and metric ingestion volume |
| **People** | Often the largest cost; Kubernetes needs engineers |

### 19.2 Capacity planning method

1. **Measure** CPU/memory/GPU per instance at typical and peak load (load test with k6, Locust or `hey`).
2. **Compute** instances needed: `peak_requests_per_second / requests_per_second_per_instance`, plus 30-50% headroom and redundancy (N+1).
3. **Set** autoscaling floor and ceiling. The floor protects availability; the ceiling protects budget and the database.
4. **Review monthly.** Average CPU of 5% means overpaying; memory at 95% of limits is one spike from an OOM.
5. **Use discounts** for stable load: reserved instances or savings plans for the baseline, spot/preemptible for fault-tolerant work.
6. **Tag and attribute costs** by team and service.

### 19.3 Image size is a cost too

Smaller images give faster pulls, faster scale-up and cold starts, cheaper registry storage and egress, and a smaller attack surface. Multi-stage builds, slim/distroless bases and `.dockerignore` ([Chapter 4](04-dockerfile-deep-dive.md)) control cost as well as hygiene.

---

## 20. Compliance and audit

Regulated environments (finance, healthcare, government, personal data) require evidence. Declaring everything as code makes this easier.

| Requirement | How containers and pipelines help |
|---|---|
| Who changed what, when? | Git and pull requests; pipeline logs record every deploy; cloud audit logs (CloudTrail, Azure Activity Log, Cloud Audit Logs) record API calls |
| What is running in production? | Image digests recorded per deployment; registry holds SBOM and provenance |
| Are we vulnerable to CVE-X? | Query SBOMs and scan results across images; continuous registry scanning |
| Is software from a trusted source? | Signed images, admission policies rejecting unsigned images, approved base images only |
| Separation of duties | Required reviewers on environments; no direct developer deploys to production |
| Access control | RBAC in registry/cluster, SSO, short-lived credentials, MFA |
| Data protection | Encryption at rest and in transit, no personal data in logs, encrypted backups |
| Retention | Documented log and backup retention policies |
| Frameworks | SOC 2, ISO 27001, PCI DSS, HIPAA, GDPR. Each has its own controls. Container benchmarks: **CIS Docker Benchmark** and **CIS Kubernetes Benchmark** (audit with `docker-bench-security`, `kube-bench`) |

Audit-readiness checklist: retain pipeline logs; keep an inventory of images and owners; enforce signing and scanning gates; document rollback and incident processes; run `docker-bench-security` on hosts and record results; review access lists quarterly. Controls: [Chapter 10](10-security.md).

---

## 21. Team practices

### 21.1 Dockerfile standards

An enforced team standard, linted with **hadolint** (a Dockerfile linter):

1. Use an approved base image (21.2), pinned to a version and ideally a digest.
2. Multi-stage builds; the final stage contains only what runs.
3. Run as a non-root user (`USER 10001`).
4. `COPY` only what is needed; keep a `.dockerignore`.
5. No secrets in `ENV`, `ARG` or layers; use BuildKit secret mounts (`RUN --mount=type=secret`).
6. Pin package versions where practical; clean package caches in the same `RUN` layer.
7. Add `HEALTHCHECK` (or define probes) and `LABEL` with source/commit/owner.
8. Use exec-form `CMD`/`ENTRYPOINT` and handle SIGTERM.
9. One process per container.

### 21.2 Golden base images

A **golden image** (paved-road base image) is a company-maintained base that teams must start from. A platform or security team builds it nightly from a trusted upstream (Debian slim, Alpine, Ubuntu, distroless), applies patches, adds corporate CA certificates and proxy config, standard users and labels, scans and signs it, and publishes it as `registry.example.com/base/python:3.12`. Application teams use it in `FROM`.

- Benefits: one place to fix a CVE, consistent hardening, ability to rebuild everything when a base vulnerability appears.
- Cost: the platform team must keep it fresh and respond quickly.
- Automate downstream rebuilds when the base changes (Renovate/Dependabot can open pull requests that bump the base digest).

### 21.3 Internal image catalog

Maintain a list (docs page, Backstage catalog or registry project) of approved images: base images, language runtimes and common infrastructure images (nginx, Redis, Postgres), with approved versions, owners, support status and update cadence. Block everything else at the registry or admission level.

### 21.4 Policy-as-code (OPA/Conftest and friends)

**Policy as code** expresses rules as machine-checkable code so they are enforced automatically.

- **OPA (Open Policy Agent)**: a general policy engine; policies are written in **Rego**.
- **Conftest**: runs OPA policies against configuration files (Dockerfiles, Compose files, Kubernetes manifests, Terraform) in CI.
- **Kubernetes admission control**: **OPA Gatekeeper** or **Kyverno** reject non-compliant objects at deploy time. Kubernetes' built-in **ValidatingAdmissionPolicy** (CEL expression language) covers many simple cases.

Conftest policy rejecting Dockerfiles that use `latest` or never set `USER` (save as `policy/dockerfile.rego`):

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
- `package main`: the namespace Conftest evaluates by default. `import rego.v1`: modern Rego syntax.
- First rule: for each Dockerfile instruction with `Cmd == "from"`, takes the image reference and denies it if it ends with `:latest` and has no digest. `msg` is the error shown.
- Second rule: denies Dockerfiles that never set `USER`.
- Run `conftest test Dockerfile --policy policy/`. It prints `FAIL` lines and exits non-zero on violations, failing the pipeline.
- Rego syntax evolves; check the OPA docs for your version if the policy errors.

Other pipeline gates: **hadolint** (Dockerfile lint), **Trivy/Grype/Docker Scout** (vulnerabilities), **Trivy config / Checkov / KICS** (infrastructure-as-code misconfiguration), **gitleaks** (secrets), **cosign verify** (signatures), **Syft** (SBOM generation).

### 21.5 Other habits

- Every image and service has an owner and an on-call.
- Renovate/Dependabot keep base images, actions and dependencies current through pull requests.
- Reproducible builds: pin versions, be version-aware with `apt-get install` for critical packages, cache deliberately.
- Documentation and runbooks live next to the code.
- Blameless post-incident reviews feed back into standards.

---

## 22. Production readiness checklist

Each item is a yes or no.

**Image and build**
- [ ] Image built by CI from a specific commit; tagged with git SHA and version
- [ ] Production references an immutable tag or a digest, never `latest`
- [ ] Base image pinned, from an approved source, updated on a schedule
- [ ] Multi-stage build; minimal final image; `.dockerignore` present
- [ ] Runs as non-root; read-only root filesystem where possible; capabilities dropped
- [ ] Image scanned (no unaddressed HIGH/CRITICAL with fixes); SBOM and provenance generated
- [ ] Image signed; deploy verifies signatures (where policy requires)
- [ ] No secrets in image, layers or repository history

**Runtime configuration**
- [ ] Health checks defined (liveness, readiness, startup where needed)
- [ ] CPU and memory limits (and requests) set from measured data
- [ ] Restart policy or orchestrator replacement configured
- [ ] Graceful shutdown (SIGTERM handled; stop timeout adequate; `init` used)
- [ ] Configuration via environment/files; secrets from a secrets manager
- [ ] Only the reverse proxy / load balancer is public; databases and caches are not
- [ ] Network policies or firewall rules least-privilege; Docker published-port bypass of host firewall checked

**Reliability**
- [ ] At least 2 replicas (or a documented, accepted single-instance risk)
- [ ] Zero-downtime deploy path tested; rollback tested and documented
- [ ] Database migrations backward-compatible; migration process defined
- [ ] Dependencies (DB, cache, third-party APIs) have timeouts, retries with backoff and defined failure behaviour
- [ ] Backups automated, encrypted, off-site; restore tested with measured RTO/RPO
- [ ] Capacity tested at expected peak plus headroom

**Observability**
- [ ] Logs to stdout/stderr, structured, rotated, shipped centrally, retention defined, no secrets
- [ ] Metrics scraped; dashboards for golden signals; saturation visible
- [ ] Alerts for availability, error rate, latency, disk, certificate expiry, backup failure; routed to on-call
- [ ] External uptime check of the public URL
- [ ] Tracing for multi-service request paths (as needed)

**Host / platform**
- [ ] Supported Linux release; automatic security updates; reboot plan
- [ ] Docker Engine from the official repository, supported version, upgrade plan
- [ ] `daemon.json`: log rotation, live-restore, address pool, data-root decided
- [ ] Disk monitoring and cleanup schedule; Docker data on a dedicated disk
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

An **incident** is any event that degrades the service for users. A **runbook** is a pre-written set of steps. Template:

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

A good runbook states how to roll back in one command, lists where dashboards and logs are, has current escalation contacts, and is rehearsed (game days).

---

## 24. Reference architectures

### 24.1 Small team (1-10 engineers): pragmatic and cheap

Goal: reliable enough, simple enough for two people to understand, minimal on-call load.

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

Characteristics: roughly tens of dollars to low hundreds per month; deploys in minutes; recovery is rebuild the VM from Terraform, redeploy the last good image, reconnect the managed DB. The single VM is an accepted risk; two VMs behind a cloud load balancer remove it. For a low-operations variant, replace the VM with Cloud Run, Azure Container Apps or ECS Fargate and keep the managed database.

### 24.2 Large enterprise (hundreds of engineers): platform-based

Goal: team self-service, strong governance, high availability, auditability.

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

Differences from the small-team setup: Kubernetes as a shared platform (often run by a platform engineering team providing "golden paths": templates, pipelines, defaults), GitOps instead of SSH deploys, signed images enforced at admission, multiple environments and regions, formal SLOs (Service Level Objectives: reliability targets such as 99.9% availability) with error budgets, and a security and compliance programme across everything.

Start with the small-team architecture. Add components (second VM, Kubernetes, GitOps, service mesh) only for a specific problem; each is something to operate.

---

## 25. Common mistakes

1. Running `:latest` in production.
2. Rebuilding images per environment instead of promoting one.
3. Unrotated container logs filling the disk.
4. Docker-published ports bypassing the host firewall; exposing databases or caches to the internet.
5. Secrets in images, git or plain env files in the repository.
6. No health checks, or checks that always pass or cascade failures.
7. No resource limits, or limits ignoring the runtime's memory behaviour.
8. Backups that were never restored.
9. One replica and no tested rollback.
10. Running `docker volume prune` on a production host.
11. Mounting `/var/run/docker.sock` into application containers (root access to the host).
12. Using Watchtower-style auto-updates as a deployment strategy; trusting unpinned third-party CI actions.
13. Adopting Kubernetes before there is a problem it solves.
14. No owner, no runbook, no alert routing.
15. Address pools colliding with the VPN.

---

## 26. Summary

- Production Docker runs on **Linux** hosts with **Docker Engine** or a managed container platform.
- Flow: git, CI (build, scan, test, sign), registry, CD, runtime, observability, rollback. **Build once, deploy the same digest everywhere.**
- Tag with git SHA and semantic versions. Never deploy `latest`. Promote by re-tagging.
- Registries: Docker Hub (rate limits), GHCR, ECR, ACR, GAR, Harbor/Artifactory/Nexus. Use pull-through caches and retention policies.
- A GitHub Actions workflow with Buildx, caching, multi-arch builds, scanning, GHCR push and SSH deploy with automatic rollback is a workable base. Pin and minimise third-party actions after the 2026 Trivy compromise.
- Choose the simplest runtime that fits: PaaS, serverless containers, single VM + Compose, Swarm (maintenance mode) or Kubernetes. App Runner is closed to new customers, ingress-nginx is retired and Watchtower is archived.
- Operate it with health checks, resource limits, rotated and central logs, metrics and alerts, tested backups, a daemon baseline, disk hygiene, planned upgrades and runbooks.

Next: [Chapter 12](12-troubleshooting-and-cheatsheet.md) covers debugging, and [Chapter 13](13-interview-questions-roadmap-glossary.md) covers interview answers and a portfolio.

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
