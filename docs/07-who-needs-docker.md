# Chapter 7: Who Needs Docker? A Decision Framework

This chapter gives criteria for deciding whether a project benefits from Docker. It maps common project types to the value Docker provides, answers whether an Android app needs Docker, and includes two worked examples (an Android CI build image and a Compose stack for an IoT/RFID backend). It also compares Docker with alternatives and lists costs and an adoption checklist.

Previous: [Chapter 6: Docker Compose](06-docker-compose.md) | Next: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md)

---

## 7.1 The one-sentence test

Docker is worth using when "it runs on my machine" is not enough:

- Someone else (a teammate, a CI server, a production server, a customer) must run the same software, **or**
- Several different tools must coexist on one computer without conflicting.

A script that only one person runs on one laptop usually does not need Docker.

Analogy: a shipping container holds any cargo and is handled the same way by every crane, ship and truck. Docker does the same for programs. A standard container is efficient at scale and unnecessary overhead for a single small item.

---

## 7.2 The decision flowchart

```
                      START: a piece of software
                                    |
                                    v
            +------------------------------------------------+
            | Will it run anywhere other than my own laptop? |
            | (teammates, CI, a server, a customer, a cloud) |
            +------------------------------------------------+
                 | NO                                  | YES
                 v                                     v
      +------------------------+        +---------------------------------+
      | One-off script with    |        | Does it need a server-side      |
      | few dependencies?      |        | runtime (Python, Java, Node,    |
      +------------------------+        | a DB, a broker, system libs)?   |
         | YES        | NO              +---------------------------------+
         v            v                      | NO                  | YES
   Skip Docker.   Docker for the             v                     v
   Use a venv /   dependencies only      Static files only?   +----------------------+
   system Python  (e.g. a database).     -> CDN / static      | Does the target      |
                                         hosting. No Docker.  | platform already run |
                                                              | containers, or can   |
                                                              | you choose one?      |
                                                              +----------------------+
                                                                 | NO          | YES
                                                                 v             v
                                                       Serverless / PaaS /   Docker is a
                                                       a VM with systemd may good fit.
                                                       be simpler            Start with a
                                                       (section 7.8).        Dockerfile +
                                                                             Compose.
```

Qualitative check. Count the "yes" answers:

| # | Question | If "yes" |
|---|----------|----------|
| 1 | Does setup take more than 15 minutes of installing and configuring tools? | Docker (or a devcontainer) saves time |
| 2 | Has "works on my machine" come up? | Docker addresses it directly |
| 3 | Does it need more than one service (app + database + cache)? | Compose helps |
| 4 | Does it run on a cloud, Kubernetes, or another container platform? | An image is required |
| 5 | Are multiple versions of the same tool needed side by side (Postgres 14 and 17)? | Containers isolate them |
| 6 | Are new people onboarded regularly? | One-command setup is valuable |
| 7 | Is untrusted or third-party code run? | Containers add a boundary, not a perfect one ([Chapter 10](10-security.md)) |
| 8 | Must a build be reproducible months later? | Pinned images provide that |

Scoring:

| Yes answers | Recommendation |
|---|---|
| 0-1 | Probably skip Docker |
| 2-3 | Use it for the parts that cause problems (usually the database and CI) |
| 4+ | Containerize the whole project |

---

## 7.3 Project types that benefit from Docker

| Project type | Docker value | Typical setup |
|---|---|---|
| **Web backends / REST APIs** (FastAPI, Django, Express, Spring, Go, .NET) | Same runtime on laptop, CI and production; easy horizontal scaling | Multi-stage, non-root `Dockerfile`; Compose with API + Postgres + Redis for local dev; image pushed to a registry and run on ECS, Cloud Run or Kubernetes |
| **Microservices** | Each service owns its dependencies; mixed languages; independent deploys | One image per service; Compose locally; Kubernetes or ECS in production; shared network and reverse proxy |
| **Data / ETL / ML pipelines** | Pinned Python, system libraries and drivers; reproducible batch jobs | One batch container per step; orchestrator (Airflow, Prefect, Dagster) launching containers; mounted data volumes |
| **AI / LLM applications** | Manages CUDA/PyTorch version conflicts; ships model servers | API container + model server + vector DB + cache; GPU passthrough ([Chapter 8](08-docker-for-ai-and-ml.md)) |
| **Databases for dev/test** | Disposable Postgres/MySQL/Mongo/Redis in seconds; any version; clean state per test run | `docker run postgres:17` or a Compose service; Testcontainers in test suites |
| **CI/CD runners and build agents** | The build environment is versioned code; clean per job | Jobs run inside an image (`container:` in GitHub Actions, `image:` in GitLab CI, Docker agents in Jenkins); BuildKit builds that produce images |
| **Legacy application packaging** | Keeps old apps (old Java, PHP, libraries) running on modern hosts without a rewrite | Dockerfile reproducing the old environment; pinned old base image; reverse proxy in front |
| **Dev-environment standardization** | Same compilers, linters and tools for everyone | Dev Containers (`.devcontainer/devcontainer.json`) in VS Code, JetBrains or GitHub Codespaces |
| **IoT / edge gateways** | Update a fleet of small Linux devices by swapping images; roll back | Multi-arch (ARM) images; Compose or a lightweight orchestrator (k3s, balena); MQTT broker + local DB + processing service |
| **Game servers** | One command per server; many servers per host; world data in volumes | Community images; published ports; a named volume for saves |
| **Self-hosted tools** (Nextcloud, Jenkins, SonarQube, Nexus, Grafana, Home Assistant) | No changes to the host; simple upgrades and rollbacks | Official image + volume + Compose |
| **Scientific / research code** | Reproduces an environment years later | Image with pinned versions; data mounted in |
| **Security research and sandboxes** | Runs untrusted code in a disposable environment | Short-lived container, no or limited network, read-only filesystem |
| **Training / teaching** | One image contains tools and course material | Pre-built image |
| **Software distribution to customers** | Product ships as an image; the customer needs only Docker | Image in a public or private registry; Compose file or Helm chart as the installer |

Why these work:

- **Isolation**: each container has its own filesystem view, process list and network, so apps with conflicting library versions do not interfere.
- **Reproducibility**: an **image** (the read-only template a container starts from) contains everything except the kernel.
- **Portability**: any Linux machine with a container runtime can run it, provided the CPU architecture matches or a multi-arch image exists.
- **Density**: containers share the host kernel instead of booting an operating system.
- **Declarative infrastructure**: Dockerfiles and Compose files are text that can be reviewed and versioned in git.

---

## 7.4 Project types that do not need Docker much

| Project type | Reason | Alternative |
|---|---|---|
| **Static websites** (HTML/CSS/JS, Hugo, Jekyll, Astro static output) | Output is files; CDNs host them better | GitHub Pages, Netlify, Cloudflare Pages, S3 + CloudFront. Docker can still build the site reproducibly in CI |
| **One-off scripts** | One user, few dependencies | `python -m venv`, `pipx`, `uv run script.py` |
| **Desktop GUI applications** (native or Electron apps for end users) | Containers target server-side Linux processes; GUIs are awkward; end users do not have Docker | Native installers (MSI, DMG, AppImage). Docker can still help the build agent |
| **Serverless-only apps** (Lambda, Cloud Functions, Vercel/Netlify functions with zip deployments) | The platform owns the runtime; code is uploaded, not an image | Platform tooling (SAM, Serverless Framework, `vercel deploy`). Lambda can run container images (up to 10 GB) when dependencies are large |
| **Mobile apps themselves** (APK/AAB, IPA) | They run on a phone, not in a container | Android Studio / Gradle / Xcode (section 7.5) |
| **Tiny hobby projects with one dependency** | Docker setup costs more than the problem | A package manager and a README |
| **Apps tied to hardware containers cannot easily reach** (USB medical devices on Windows, GPU desktop apps, proprietary drivers) | `--device` works on Linux but is awkward on Windows/macOS | Native install, or a thin gateway container on a Linux box next to the device |
| **Windows-only .NET Framework desktop software** | Windows containers exist but are niche and heavy | Normal Windows deployment |

---

## 7.5 "Does an Android app need Docker?" (the direct answer)

### 7.5.1 The Android app itself: no

An Android app is compiled into an **APK** (Android Package) or **AAB** (Android App Bundle) and runs on a device using the **Android Runtime (ART)**. It is developed in **Android Studio**, built with **Gradle** (the build tool), and shipped through Google Play or a device-management system. No step in the normal workflow requires Docker.

```
Laptop                              Phone / tablet
+-----------------------+           +--------------------------+
| Android Studio        |  APK/AAB  | Android OS               |
| Gradle build          | --------> |   ART runtime            |
| (no Docker needed)    |           |     Your app             |
+-----------------------+           +--------------------------+
```

### 7.5.2 Docker around Android: often yes

Docker applies to the systems around the app. The seven common uses, from most to least common:

#### (1) Reproducible CI build images

A CI server (GitHub Actions, GitLab CI, Jenkins, Bitbucket) needs the right JDK, Android SDK, platform and build-tools versions, NDK, Gradle and accepted licences. Hosted runners change over time, and hand-maintained build machines drift. A `Dockerfile` pins the environment: every developer machine and CI run uses the same image, and a CI failure can be reproduced locally. Full example: section 7.6.

#### (2) Backend services the app talks to

Typical components: REST/GraphQL API (Ktor, Spring, Node, Python, Go), auth servers (Keycloak, Authentik), push notification services, self-hosted Firebase alternatives (**Supabase**, **Appwrite**, **PocketBase**, **Parse Server**), and storage (MinIO, Postgres, Redis).

For development, run the backend with `docker compose up` and point the emulator at `http://10.0.2.2:8080`. `10.0.2.2` is the address the Android emulator uses for the host computer's `localhost`.

#### (3) Running the Android emulator in containers (CI only, Linux only)

An emulator can run in a container for automated UI tests, with these constraints:

- It needs hardware virtualization: **KVM** (Kernel-based Virtual Machine, the Linux built-in hypervisor). The container needs `/dev/kvm`.
- It works on Linux hosts with KVM (bare metal, or cloud VMs with nested virtualization). Docker Desktop on Windows/macOS generally does not provide accelerated emulation; use the Android Studio emulator there.
- Community image: `budtmo/docker-android` (emulator, noVNC web viewer, Appium support). Google publishes **android-emulator-container-scripts** to build emulator containers.
- Unverified: current tag names and requirements of `budtmo/docker-android` could not be checked against the project's own site. Read its README before use.
- Alternatives: **Firebase Test Lab**, **AWS Device Farm**, **BrowserStack**, or GitHub Actions runner KVM support (check GitHub's current docs).

Illustrative command (see the image's README for real flags):

```bash
docker run -d --device /dev/kvm -p 6080:6080 -e EMULATOR_DEVICE="Samsung Galaxy S10" -e WEB_VNC=true --name android-emu budtmo/docker-android:emulator_14.0
```

| Part | Meaning |
|---|---|
| `-d` | Run in the background (detached) |
| `--device /dev/kvm` | Pass the host KVM device into the container for hardware acceleration |
| `-p 6080:6080` | Publish the noVNC web viewer at `http://localhost:6080` |
| `-e ...` | Set configuration through environment variables |
| `emulator_14.0` | Tag selecting the Android version; tags change, check the repository |

#### (4) Developer tooling servers

| Tool | Purpose | Image (verify tag before use) |
|---|---|---|
| **SonarQube** | Code quality and static analysis | `sonarqube` (official image) |
| **Sonatype Nexus Repository** | Hosts Maven/Gradle artifacts (for example your own AARs) | `sonatype/nexus3` |
| **JFrog Artifactory** | Same, enterprise edition | `releases-docker.jfrog.io/jfrog/artifactory-oss` |
| **Jenkins** | CI server | `jenkins/jenkins:lts` |
| **GitLab / Gitea** | Source hosting | `gitlab/gitlab-ce`, `gitea/gitea` |
| **Gradle Build Cache Node** | Shared build cache for the team | `gradle/build-cache-node` |

Containers give one-command installs, upgrades by pulling a new tag, and backup by copying the volume.

#### (5) Test mocks and fakes

- **WireMock** (`wiremock/wiremock`): declares responses, for example "when the app calls `GET /api/tags/123`, return this JSON."
- **MockServer** (`mockserver/mockserver`): similar, with request verification.
- **Testcontainers**: a library (JVM, Python, Node, Go, .NET) that starts real containers (Postgres, Kafka, a mock) from integration tests and removes them afterwards.

```bash
docker run -d --name wiremock -p 8080:8080 -v "$PWD/mocks:/home/wiremock" wiremock/wiremock:latest
```

| Part | Meaning |
|---|---|
| `-p 8080:8080` | Expose the mock on host port 8080; the emulator reaches it at `http://10.0.2.2:8080` |
| `-v "$PWD/mocks:/home/wiremock"` | Mount the local `mocks/` folder (containing `mappings/` and `__files/`) so stubs can be edited without a rebuild |

In real projects, pin a version instead of `latest` ([Chapter 10](10-security.md)).

#### (6) Edge, IoT and RFID systems

An Android handheld that talks to RFID readers, barcode scanners or sensors sends its data to a Linux server, an edge gateway (an industrial PC or Raspberry-Pi-class device near the readers) and a cloud backend.

```
 Handheld (Android app)          Edge gateway (Linux box)                    Cloud / Data center
+----------------------+       +-----------------------------------+       +----------------------+
| RFID SDK, scanning   | MQTT/ | Mosquitto (MQTT broker)           | MQTT/ | Postgres / Timescale |
| UI, local cache      | HTTPS | Node-RED (flows/transform)        | HTTPS | Location/inventory   |
+----------------------+ ----> | InfluxDB (time-series)            | ----> | API + dashboards     |
                               | Grafana (local dashboards)        |       +----------------------+
                               | Location/inventory service        |
                               +-----------------------------------+
                                  each component = one container
```

- **MQTT**: a lightweight publish/subscribe messaging protocol used in IoT. Devices publish messages to *topics* (for example `rfid/gate1/tags`); other programs subscribe to topics.
- **Broker**: the server that relays MQTT messages (for example **Mosquitto**).
- **InfluxDB**: a **time-series database**, optimized for "value at time T" data such as tag reads per second.
- **Grafana**: draws dashboards from databases.
- **Node-RED**: a visual flow-based tool that connects data sources (MQTT in, transform, HTTP/DB out).
- **Location or inventory backend**: a custom service that turns raw reads into results such as "this pallet is in zone B."

Advantages at the edge: update by pulling a new image, roll back by starting the old tag, run the same Compose file on a laptop and on the gateway, and build multi-arch (ARM64 and x86) images from one build. Full Compose example: section 7.7.

#### (7) Build tooling and release pipelines

Run lint, unit tests, dependency scans and signing inside a controlled image. Run **fastlane** (release automation) or **bundletool** (device-specific APKs) in a container for the same reproducibility.

### 7.5.3 Summary table for the Android developer

| Question | Answer |
|---|---|
| Docker needed to build and run the app on a phone? | No |
| Docker needed to ship to Google Play? | No |
| Useful for CI build agents? | Yes |
| Useful for the backend the app talks to? | Yes |
| Useful for running Sonar / Nexus / Jenkins? | Yes |
| Useful for mocking APIs in tests? | Yes |
| Useful for running the emulator? | Only on Linux CI with KVM; otherwise use Android Studio's emulator or a device farm |
| Useful for IoT/RFID gateways and data platforms? | Yes |

---

## 7.6 Worked example: a Dockerfile-based Android CI build image

Goal: an image with JDK 17, the Android command-line SDK tools, and a pinned platform and build-tools, so `./gradlew assembleRelease` behaves the same everywhere.

The Gradle wrapper (`gradlew`) in the repository downloads Gradle itself, so the image needs only the JDK and the Android SDK.

`android-ci/Dockerfile`:

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:17-jdk-jammy

ARG CMDLINE_TOOLS_VERSION=11076708
ARG ANDROID_PLATFORM=android-35
ARG ANDROID_BUILD_TOOLS=35.0.0

ENV ANDROID_HOME=/opt/android-sdk
ENV ANDROID_SDK_ROOT=/opt/android-sdk
ENV PATH="${PATH}:${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools"

RUN apt-get update \
 && apt-get install -y --no-install-recommends curl unzip git ca-certificates \
 && rm -rf /var/lib/apt/lists/*

RUN mkdir -p ${ANDROID_HOME}/cmdline-tools \
 && curl -fsSL -o /tmp/cmdline-tools.zip \
      https://dl.google.com/android/repository/commandlinetools-linux-${CMDLINE_TOOLS_VERSION}_latest.zip \
 && unzip -q /tmp/cmdline-tools.zip -d ${ANDROID_HOME}/cmdline-tools \
 && mv ${ANDROID_HOME}/cmdline-tools/cmdline-tools ${ANDROID_HOME}/cmdline-tools/latest \
 && rm /tmp/cmdline-tools.zip

RUN yes | sdkmanager --licenses > /dev/null \
 && sdkmanager "platform-tools" "platforms;${ANDROID_PLATFORM}" "build-tools;${ANDROID_BUILD_TOOLS}"

RUN useradd --create-home --uid 1000 builder \
 && chown -R builder:builder ${ANDROID_HOME}
USER builder
WORKDIR /workspace

CMD ["./gradlew", "--version"]
```

| Line | What it does |
|---|---|
| `# syntax=docker/dockerfile:1` | Selects the latest stable Dockerfile syntax for BuildKit (Docker's build engine). |
| `FROM eclipse-temurin:17-jdk-jammy` | Base image with JDK 17 (Eclipse Temurin, a free OpenJDK build) on Ubuntu 22.04 ("jammy"). Android Gradle Plugin 8.x requires JDK 17; adjust to the project. |
| `ARG ...` | Build-time variables with defaults. The version numbers are **examples**: check the Android developer site for the current command-line tools number and API levels, and match the project's `compileSdk`. |
| `ENV ANDROID_HOME=...` | Environment variables stored in the image so Gradle and `sdkmanager` find the SDK. |
| `ENV PATH=...` | Adds the SDK tools to the shell search path. |
| `RUN apt-get ... && rm -rf /var/lib/apt/lists/*` | Installs download tools and deletes the package index in the same layer so the image stays small ([Chapter 4](04-dockerfile-deep-dive.md) covers layers). `--no-install-recommends` skips optional packages. |
| `curl -fsSL -o ... URL` | Downloads the command-line tools zip. `-f` fails on HTTP errors, `-s` hides progress, `-S` still shows errors, `-L` follows redirects. |
| `unzip` / `mv ... latest` | The zip contains `cmdline-tools/`; `sdkmanager` requires the layout `cmdline-tools/latest/bin`, so the folder is renamed. |
| `yes \| sdkmanager --licenses` | Answers "y" to every licence prompt. Running it accepts the Android SDK licences; read them first. |
| `sdkmanager "platform-tools" "platforms;..." "build-tools;..."` | Installs only the required SDK components. |
| `useradd ... USER builder` | Creates and switches to a non-root user (safer; Gradle behaves better). |
| `WORKDIR /workspace` | Directory where the source code is mounted. |
| `CMD [...]` | Default command; prints versions. |

Build the image:

```bash
docker build -t myorg/android-ci:2026-10 android-ci/
```

- `-t myorg/android-ci:2026-10`: names (tags) the image. A date or version tag instead of `latest` keeps builds reproducible.
- `android-ci/`: the build context (the folder containing the Dockerfile).

Run a build:

```bash
docker run --rm \
  -v "$PWD":/workspace \
  -v gradle-cache:/home/builder/.gradle \
  myorg/android-ci:2026-10 \
  ./gradlew assembleDebug --no-daemon
```

- `--rm`: deletes the container on exit; the image remains.
- `-v "$PWD":/workspace`: mounts the project into the container. In Windows PowerShell use `${PWD}` instead of `"$PWD"`.
- `-v gradle-cache:/home/builder/.gradle`: a **named volume** that keeps downloaded Gradle dependencies between runs ([Chapter 5](05-storage-and-networking.md)).
- `./gradlew assembleDebug --no-daemon`: runs the build; `--no-daemon` avoids leaving a background Gradle process in a throw-away container.

Expected tail of the output (illustrative):

```
BUILD SUCCESSFUL in 2m 31s
48 actionable tasks: 48 executed
```

Use in **GitHub Actions** (the job runs inside the image):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/myorg/android-ci:2026-10
    steps:
      - uses: actions/checkout@v4
      - run: ./gradlew assembleRelease
```

- `runs-on: ubuntu-latest`: the VM that hosts the job.
- `container.image`: runs every step inside the image instead of directly on the VM.
- `actions/checkout@v4`: fetches the source. For safety, pin actions to a full commit SHA (see [Chapter 10](10-security.md) on the 2026 Trivy action compromise).

Do not put the signing keystore or its passwords in the image. Inject them at job time from the CI secret store ([Chapter 10](10-security.md), secrets section). Anything baked into an image can be extracted by anyone who can pull it.

---

## 7.7 Worked example: IoT/RFID-style backend in Compose

The stack from the diagram in 7.5.2: an **API** service, **PostgreSQL**, an **MQTT broker** (Mosquitto), **InfluxDB** and **Grafana**. This is a development or gateway-class setup; for production see [Chapter 11](11-production-and-enterprise.md).

Folder layout:

```
iot-stack/
  compose.yaml
  .env                      (not committed; holds passwords)
  mosquitto/
    mosquitto.conf
  api/
    Dockerfile
    (service code)
  grafana/
    provisioning/...        (optional dashboards as code)
```

`mosquitto/mosquitto.conf`:

```
listener 1883
allow_anonymous true
persistence true
persistence_location /mosquitto/data/
```

- `listener 1883`: listens on the standard MQTT port. Since Mosquitto 2.0 the broker listens only on localhost unless a listener is declared, so this line is required for other containers to connect.
- `allow_anonymous true`: allows connections without a password. Acceptable for a local demo only; use a password file or TLS on any network.
- `persistence true` with `persistence_location`: saves queued messages to disk.

`compose.yaml`:

```yaml
name: rfid-edge

services:
  api:
    build: ./api
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      DATABASE_URL: postgresql://rfid:${POSTGRES_PASSWORD}@db:5432/rfid
      MQTT_HOST: mqtt
      INFLUX_URL: http://influx:8086
    depends_on:
      db:
        condition: service_healthy
      mqtt:
        condition: service_started
    restart: unless-stopped

  db:
    image: postgres:17
    environment:
      POSTGRES_USER: rfid
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: rfid
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U rfid -d rfid"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped

  mqtt:
    image: eclipse-mosquitto:2
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/mosquitto.conf:/mosquitto/config/mosquitto.conf:ro
      - mqttdata:/mosquitto/data
    restart: unless-stopped

  influx:
    image: influxdb:2
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: ${INFLUX_PASSWORD}
      DOCKER_INFLUXDB_INIT_ORG: myorg
      DOCKER_INFLUXDB_INIT_BUCKET: tagreads
    volumes:
      - influxdata:/var/lib/influxdb2
    restart: unless-stopped

  grafana:
    image: grafana/grafana-oss:latest
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafanadata:/var/lib/grafana
    depends_on:
      - influx
    restart: unless-stopped

volumes:
  pgdata:
  mqttdata:
  influxdata:
  grafanadata:
```

`.env` (create it locally; never commit it; values are placeholders):

```
POSTGRES_PASSWORD=change-me-locally
INFLUX_PASSWORD=change-me-too-8chars+
GRAFANA_PASSWORD=change-me-as-well
```

| Piece | Meaning |
|---|---|
| `name: rfid-edge` | Project name; Compose prefixes networks and volumes with it (`rfid-edge_pgdata`). |
| `services:` | The containers that make up the app. |
| `api.build: ./api` | Builds the image from `./api/Dockerfile` instead of pulling it. |
| `ports: "127.0.0.1:8080:8080"` | Publishes container port 8080 on the host loopback interface only. Without `127.0.0.1:`, the port is published on all interfaces. |
| `environment:` (api) | Variables the service reads. `${POSTGRES_PASSWORD}` is substituted from `.env` at `docker compose up`. |
| `db`, `mqtt`, `influx` as hostnames | Compose creates a private network where each service is reachable by its service name. The API connects to `db:5432`, not `localhost`, because `localhost` inside a container is the container itself. |
| `depends_on ... condition: service_healthy` | Starts the API only after Postgres reports healthy. Plain `depends_on` waits for start, not readiness. |
| `image: postgres:17` | Official PostgreSQL 17 image. Pinning the major version avoids surprise upgrades; a data directory cannot be opened by a different major version without migration. |
| `volumes: pgdata:/var/lib/postgresql/data` | Named volume that keeps the database across `docker compose down`. Check the Postgres image docs when moving to a newer major version; the recommended data-path layout changed in recent versions. |
| `healthcheck.test` | Command run every 5 seconds inside the container; exit code 0 means healthy. `pg_isready` is Postgres' readiness probe. |
| `restart: unless-stopped` | Restarts after crashes or reboots unless explicitly stopped. Needed on edge gateways. |
| Mosquitto `:ro` mount | Mounts the config read-only. |
| `DOCKER_INFLUXDB_INIT_*` | Variables of the official InfluxDB 2 image that perform first-run setup (admin user, organization, first bucket). Ignored after the data volume is initialized. |
| `GF_SECURITY_ADMIN_PASSWORD` | Grafana variables starting with `GF_` override config settings. |
| `volumes:` (top level) | Declares the named volumes Compose creates and manages. |

Start the stack:

```bash
docker compose up -d
docker compose ps
```

- `up -d`: creates networks, volumes and containers, then returns the terminal (`-d` = detached).
- `ps`: lists the running services.

Example output of `docker compose ps` (illustrative, trimmed):

```
NAME                  IMAGE                       STATUS                    PORTS
rfid-edge-api-1       rfid-edge-api               Up 20 seconds             127.0.0.1:8080->8080/tcp
rfid-edge-db-1        postgres:17                 Up 26 seconds (healthy)   5432/tcp
rfid-edge-grafana-1   grafana/grafana-oss:latest  Up 20 seconds             127.0.0.1:3000->3000/tcp
rfid-edge-influx-1    influxdb:2                  Up 26 seconds             8086/tcp
rfid-edge-mqtt-1      eclipse-mosquitto:2         Up 26 seconds             0.0.0.0:1883->1883/tcp
```

MQTT smoke test, using the Mosquitto client tools inside the running broker container:

```bash
docker compose exec mqtt mosquitto_pub -t rfid/gate1/tags -m '{"epc":"E2801160600002054F5B0001","rssi":-52}'
docker compose exec mqtt mosquitto_sub -t 'rfid/#' -C 1
```

- `exec mqtt`: runs a command inside the running `mqtt` service container.
- `mosquitto_pub -t <topic> -m <message>`: publishes one message. The EPC is a made-up example.
- `mosquitto_sub -t 'rfid/#' -C 1`: subscribes to every topic under `rfid/` (`#` is the MQTT multi-level wildcard) and exits after one message (`-C 1`). Run the subscriber first, in a second terminal.

Stop the stack and keep data:

```bash
docker compose down
```

Stop the stack and delete the data volumes (destructive):

```bash
docker compose down -v
```

---

## 7.8 Docker vs the alternatives

| Approach | Provides | Lacks | Use when |
|---|---|---|---|
| **Language virtual environments** (`venv`, `uv`, `nvm`, `rbenv`, `sdkman`) | Per-project language packages; no overhead | Cannot isolate system libraries, databases, OS packages or ports; not a deployment artifact | Pure-language projects without system dependencies, local development |
| **Virtual machines** (VirtualBox, VMware, Hyper-V, EC2) | Full OS isolation, separate kernels, strongest boundary | Large (GBs), slow boot, harder to version and distribute | A different OS or kernel is needed, strong multi-tenant isolation, legacy OS software |
| **Docker containers** | Light, fast, portable images; large ecosystem; shared host kernel | Shared kernel (weaker isolation than a VM), Linux-centric, learning curve | Default choice for services and pipelines |
| **Serverless functions** (Lambda, Cloud Functions) | No servers to manage, scale to zero, pay per call | Time and memory limits, cold starts, vendor lock-in, poor fit for long-running or stateful work | Event-driven glue, spiky traffic, small APIs |
| **PaaS** (Heroku, Render, Fly.io, Railway, Cloud Run, App Runner) | Deploy by `git push` or image push | Less control; can be expensive | Small teams wanting less operations work. Many PaaS platforms run Docker images |
| **Buildpacks** (Cloud Native Buildpacks, Heroku-style) | Build an image from source without a Dockerfile; base layers patched by the platform | Less customizable; another tool to learn | Standard apps (Spring Boot, Node) with no Dockerfile maintenance (`pack build`, Spring Boot `bootBuildImage`) |
| **Nix / NixOS / Guix** | Bit-for-bit reproducible builds; per-project shells (`nix develop`) without a daemon | Steep learning curve | Reproducibility is the top priority. Nix can also produce Docker images |
| **Config management** (Ansible, Chef, Puppet) on VMs | Mature configuration of long-lived servers | Servers drift; mutable infrastructure | Existing VM fleets. Ansible and Docker combine (Ansible installs Docker and deploys Compose files) |
| **Podman / containerd / Kubernetes** | Other container tools; Podman is daemonless and rootless by default | Different ergonomics | They use the same OCI image format, so skills transfer |

Images follow an open standard, the **OCI** (Open Container Initiative) image spec. Podman, Kubernetes, ECS, Cloud Run, Lambda (container images) and others run them, so learning Dockerfiles is not vendor lock-in.

---

## 7.9 When containers are the wrong tool

| Situation | Reason | Alternative |
|---|---|---|
| Stateful, performance-critical production databases without a plan | Volumes, backups, I/O performance and upgrades must be managed | Managed database (RDS, Cloud SQL, Atlas) in production; containers in dev/test |
| GUI desktop apps for end users | X11 forwarding or VNC/noVNC is clumsy | Native packaging |
| Hard real-time or hardware-timing-critical code | Containers share the kernel and scheduler; CPU pinning and real-time priority are possible on Linux but delicate | Bare metal or an RTOS |
| Software needing a different kernel or kernel modules | Containers use the host kernel | VM |
| Strong multi-tenant isolation of hostile code | A container is a weaker boundary than a VM ([Chapter 10](10-security.md)) | microVMs (Firecracker, gVisor, Kata Containers) or VMs |
| Windows-native or macOS-native software | Linux containers cannot run it | Native deployment |
| Nobody on the team can operate it | Production containers need registries, scanning, orchestration and monitoring | A systemd service for one app on one server |
| Pet projects where time to first run matters more than reproducibility | Docker setup overhead | Run directly |

---

## 7.10 What Docker really costs (cost/complexity trade-offs)

| Cost | Details | Mitigation |
|---|---|---|
| **Learning curve** | Images, layers, networking, volumes, Compose, registries | Start with Compose only |
| **Build time** | Image builds can be slow, especially ML images | Layer caching, BuildKit cache mounts, CI build cache ([Chapter 4](04-dockerfile-deep-dive.md)) |
| **Disk space** | Images and volumes accumulate; ML images are many GB | `docker system df`, a pruning policy (pruning deletes data), slim base images |
| **Debugging indirection** | Logs, shells and networking go through Docker | `docker logs`, `docker exec`, `docker inspect` ([Chapter 3](03-core-cli-commands.md), [Chapter 12](12-troubleshooting-and-cheatsheet.md)) |
| **Docker Desktop on Windows/macOS** | Runs a hidden Linux VM; file sharing across the boundary is slower; uses RAM | Keep source in the Linux filesystem (WSL2) when possible; set resource limits |
| **Security responsibility** | Base image patches, secrets and runtime flags are the owner's job | [Chapter 10](10-security.md) |
| **Licensing** | Docker Desktop requires a paid subscription for larger companies (check Docker's current terms; historically a size/revenue threshold). Alternatives: Docker Engine on Linux, Podman, Rancher Desktop, Colima | Read the current terms; consider alternatives for company machines |
| **Production operations** | Orchestration (Kubernetes) is a separate skill set | Managed service (ECS/Fargate, Cloud Run) until more is needed |
| **Cloud cost** | An idle container still uses memory; many small containers can cost more than one process | Right-size limits ([Chapter 11](11-production-and-enterprise.md)) |

Docker adds setup time and ongoing maintenance. It removes recurring "works on my machine" problems. For teams larger than one person or projects lasting more than a few months, the trade is usually positive.

---

## 7.11 Adoption checklist

**Phase 0: Decide**
- [ ] The flowchart in section 7.2 gave "yes" for at least the dev database or the CI.
- [ ] The team agreed on a container runtime (Docker Engine, Docker Desktop, Podman, Colima) and checked licensing.

**Phase 1: Local development**
- [ ] A `Dockerfile` builds cleanly with `docker build` and uses a pinned, trusted base image.
- [ ] A `.dockerignore` excludes `.git`, build output, local env files and secrets.
- [ ] A `compose.yaml` starts everything with `docker compose up`.
- [ ] Secrets come from a git-ignored `.env` file or a secret store, never from the Dockerfile.
- [ ] The README documents installing Docker, running `docker compose up`, and where to open the app.
- [ ] Persistent data is in a named volume; the team knows `down -v` deletes it.

**Phase 2: CI**
- [ ] CI builds the image on every pull request.
- [ ] Images are tagged with the git commit SHA (and a semantic version on release), not only `latest`.
- [ ] Layer caching is configured.
- [ ] Tests run inside the image that ships.
- [ ] A vulnerability scan (Trivy, Scout or Grype) runs and fails the build on critical findings that have fixes.
- [ ] Actions and plugins used in CI are pinned to immutable versions.

**Phase 3: Delivery**
- [ ] A private registry (GHCR, ECR, GCR/Artifact Registry, ACR, Docker Hub private repos) holds the images.
- [ ] Deployments reference images by version tag or digest.
- [ ] Containers run as non-root with resource limits and health checks.
- [ ] Logs go to stdout/stderr and are collected centrally.
- [ ] A rollback plan exists: redeploy the previous tag.

**Phase 4: Operate**
- [ ] Base images are rebuilt on a schedule to pick up security patches.
- [ ] Someone owns image hygiene (pruning old registry tags, rotating tokens).
- [ ] Volume and database backups are tested by restoring.
- [ ] Monitoring and alerting cover container restarts and resource usage.

---

## 7.12 Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Added complexity with no benefit | Docker applied to a project with no portability or dependency problem | Use the flowchart; start with just the database |
| Connection refused between containers | `localhost` inside a container is that container | Use the service name (`db`, `mqtt`) on a Compose network |
| Secrets extracted from an image | Secrets, signing keystores or API keys baked into the image | Inject at runtime ([Chapter 10](10-security.md)) |
| Builds change unexpectedly | `latest` used for everything | Pin tags; pin digests for production |
| Database data lost | Database container without a volume | Named volumes and backups |
| Emulator is slow or does not start | No hardware acceleration in Docker Desktop on Windows/macOS | Run the emulator natively or use a cloud device farm |
| Services reachable from the network | Ports published to all interfaces (`-p 1883:1883`); Docker bypasses host firewalls such as ufw ([Chapter 10](10-security.md)) | Bind to `127.0.0.1` unless remote access is intended |
| Cannot restart or scale services independently | One container running several services | One main process per container; use Compose |
| Changes vanish when the container is recreated | Container treated as a VM (SSH in and fix) | Change the Dockerfile, rebuild, redeploy |

---

## 7.13 Try it yourself

1. **Score a project.** Answer the 8 questions in section 7.2 for a real project.
2. **Disposable database.** Run Postgres 17, create a table, delete the container, and confirm the data is gone. Repeat with a named volume and confirm the data persists.

   ```bash
   docker run -d --name pg-test -e POSTGRES_PASSWORD=devonly -p 127.0.0.1:5432:5432 postgres:17
   docker exec -it pg-test psql -U postgres -c "create table t(x int); insert into t values (1); select * from t;"
   docker rm -f pg-test
   ```

   - `-e POSTGRES_PASSWORD=devonly`: required by the image to set the superuser password (throw-away dev value).
   - `docker exec -it ... psql`: runs a SQL command inside the container.
   - `docker rm -f`: force-removes this test container only.
3. **Android CI image.** Build the image from section 7.6 against a real project and compare with the normal build.
4. **IoT stack.** Start the stack from section 7.7, publish an MQTT message, open Grafana at `http://localhost:3000` (user `admin`, password from `.env`) and add InfluxDB as a data source with URL `http://influx:8086` (Grafana runs inside the Compose network).

---

## 7.14 Chapter summary

- Use Docker when someone else or somewhere else must run the software, or when several services must coexist.
- Skip it for static sites, tiny scripts, desktop GUI apps, serverless-only designs and mobile apps themselves.
- An Android app does not need Docker. Its CI, backend, tooling servers, API mocks and IoT/edge/RFID data platforms are good candidates.
- Images follow the open OCI standard, so the skill is portable. Introduce Docker in stages using the checklist.

Next: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md)
