# Chapter 7: Who Needs Docker? A Decision Framework

> **What you will learn in this chapter**
> - How to decide, for any project, whether Docker is worth it (and when it is not).
> - A table mapping common project types to the value Docker gives them and the typical setup.
> - A direct, honest answer to "Does my Android app need Docker?" (spoiler: the app no, the world around it often yes).
> - Two complete worked examples: a Docker-based Android CI build image, and a Compose stack for an IoT/RFID-style backend (API + database + MQTT broker + dashboards).
> - How Docker compares with virtual environments, virtual machines, serverless, buildpacks and Nix.
> - When containers are the wrong tool, what they really cost you, and an adoption checklist you can run on a team.

Previous: [Chapter 6: Docker Compose](06-docker-compose.md) | Next: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md)

---

## 7.1 The one-sentence test

Before the long version, here is the shortest useful rule:

> **Docker is worth it when "it runs on my machine" is not good enough.**
> That means: someone else (a teammate, a CI server, a production server, a customer) must run the same thing, **or** you need many different things to coexist on one computer without fighting each other.

If you are the only person who will ever run a small script on your own laptop, Docker is probably overkill. If anyone else, anywhere else, ever has to run it, Docker starts paying for itself.

Think of Docker as a **shipping container for software**. A shipping container does not care whether it holds bananas or laptops; cranes, ships and trucks all handle it the same way. Docker does the same for programs: whatever is inside (Python, Java, Node, a database), the outside looks the same, and any machine with Docker can "lift" it. Shipping containers are a bad idea if you are just carrying one bag across the street. That is the essence of this chapter: **a standard box is great at scale and annoying at tiny scale.**

---

## 7.2 The decision flowchart

Use this flowchart the next time somebody asks "should we Dockerize this?"

```
                      START: I have a piece of software
                                    |
                                    v
            +-----------------------------------------------+
            | Will it run anywhere other than my own laptop? |
            | (teammates, CI, a server, a customer, a cloud) |
            +-----------------------------------------------+
                 | NO                                  | YES
                 v                                     v
      +------------------------+        +---------------------------------+
      | Is it a one-off script  |        | Does it need a server-side      |
      | with few dependencies?  |        | runtime (Python, Java, Node,    |
      +------------------------+        | a DB, a broker, system libs)?   |
         | YES        | NO                +---------------------------------+
         v            v                      | NO                  | YES
   Skip Docker.   Consider Docker            v                     v
   Use a venv /   for the *dependencies*  Static files only?   +----------------------+
   system Python  (e.g. a DB) only.       -> CDN / static      | Does the target      |
                                          hosting. No Docker.  | platform ALREADY run |
                                                               | containers or can    |
                                                               | you pick one?        |
                                                               +----------------------+
                                                                  | NO          | YES
                                                                  v             v
                                                        Serverless / PaaS /   DOCKER IS A
                                                        a VM with systemd may GOOD FIT.
                                                        be simpler. Compare   Start with a
                                                        (section 7.9).        Dockerfile +
                                                                              Compose.
```

A second, more qualitative check. Count how many of these are true for your project:

| # | Question | If "yes" |
|---|----------|----------|
| 1 | Does setup take more than 15 minutes of "install X, then Y, then configure Z"? | Docker (or devcontainer) saves real time |
| 2 | Have you ever heard "works on my machine"? | Docker helps directly |
| 3 | Do you need more than one service (app + database + cache)? | Compose helps directly |
| 4 | Do you run on a cloud, Kubernetes, or any container platform? | You almost certainly need an image |
| 5 | Do you need multiple versions of the same tool side by side (Postgres 14 and 17)? | Containers isolate them |
| 6 | Do you onboard new people regularly? | One-command setup is valuable |
| 7 | Do you need to run untrusted or third-party code safely-ish? | Containers add a boundary (not a perfect one, see [Chapter 10](10-security.md)) |
| 8 | Must you reproduce exactly the same build months later? | Pinned images give you that |

**Scoring.** 0-1 yes: probably skip it. 2-3 yes: use it for the parts that hurt (usually the database and the CI). 4+ yes: containerize the whole thing.

---

## 7.3 Project types that benefit from Docker

Here is the big table. Read the "Docker value" column as "what problem does the container solve here?" and "Typical setup" as what you would actually build.

| Project type | Docker value | Typical setup |
|---|---|---|
| **Web backends / REST APIs** (Python FastAPI/Django, Node/Express, Java/Spring, Go, .NET) | Same runtime on laptop, CI and prod; easy horizontal scaling; no "wrong Python version" | One `Dockerfile` (multi-stage, non-root), a Compose file with API + Postgres + Redis for local dev, image pushed to a registry and run on ECS/Cloud Run/Kubernetes |
| **Microservices** | Each service owns its dependencies; polyglot (Go + Python + Node) works; independent deploys | One image per service, Compose for local, Kubernetes (or ECS) for production, a shared network and a gateway/reverse proxy |
| **Data / ETL / ML pipelines** | Pin Python, system libs and drivers; schedule batch jobs reproducibly | Batch container per step, orchestrator (Airflow/Prefect/Dagster) launching containers, mounted data volumes |
| **AI / LLM applications** | Tames CUDA/PyTorch version hell; ships model servers; reproducible deploys | API container + model server + vector DB + cache; GPU passthrough. Whole [Chapter 8](08-docker-for-ai-and-ml.md) is about this |
| **Databases for dev/test** | Throw-away Postgres/MySQL/Mongo/Redis in seconds; any version; clean state per test run | `docker run postgres:17` or a Compose service; Testcontainers in test suites |
| **CI/CD runners and build agents** | The build environment itself becomes versioned code; clean per job; no "snowflake" build servers | Jobs run inside an image (`container:` in GitHub Actions, `image:` in GitLab CI, Docker agents in Jenkins); builds that *produce* images with BuildKit |
| **Legacy application packaging** | Freeze an old app (old Java, old PHP, old libs) so it keeps running on modern hosts without the rewrite | Dockerfile reproducing the old environment; pin an old base image; wrap in a container and put a reverse proxy in front |
| **Dev-environment standardization** | Everyone gets the same compilers, linters, tools; onboarding in minutes | Dev Containers (`.devcontainer/devcontainer.json`) used by VS Code / JetBrains / GitHub Codespaces |
| **IoT / edge gateways** | Update a fleet of small Linux boxes by swapping images; isolate apps; roll back | ARM images (multi-arch), Compose or a lightweight orchestrator (k3s, balena, Docker on Raspberry Pi/industrial PCs), MQTT broker + local DB + processing service |
| **Game servers** (Minecraft, Valheim, CS, custom) | One command to start a server; many servers on one host; easy backup of world data via volumes | Community images, ports published to the internet, a named volume for saves |
| **Self-hosted tools** (Nextcloud, Jenkins, SonarQube, Nexus, Grafana, Home Assistant) | Install without polluting your host; trivial upgrades and rollbacks | Official image + volume + Compose |
| **Scientific / research code** | Reproduce a paper's environment years later | Image with pinned versions, plus the data mounted in |
| **Security research & sandboxes** | Run suspicious tools or untrusted code in a disposable box | Short-lived container, no network or limited network, read-only filesystem |
| **Training / teaching** | "Install this one image and everything works" | Pre-built image with tools and course materials |
| **Software distribution to customers** | Ship your product as an image; customer needs only Docker | Image on a registry (public or private), a Compose file or Helm chart as the "installer" |

### Why each of these works (in plain English)

* **Isolation.** Each container has its own filesystem view, its own process list and its own network. Two apps that need conflicting versions of a library never see each other's files.
* **Reproducibility.** An *image* (the frozen, read-only template a container starts from) captures everything except the kernel. If you pin versions, the same image runs identically for years.
* **Portability.** Any Linux machine with a container runtime can run it: your laptop, a coworker's Mac, a cloud VM, a Raspberry Pi (if the CPU architecture matches or you built multi-arch images).
* **Density.** Containers are lightweight (they share the host's kernel; they do not each boot a full operating system), so you can run dozens on a laptop.
* **Declarative infrastructure.** A Dockerfile and Compose file are text, so you can review them in pull requests and version them in git. The environment becomes code.

---

## 7.4 Project types that do not need Docker much

Being honest builds trust, so here is where Docker adds little or nothing.

| Project type | Why Docker is not a big deal here | What to use instead |
|---|---|---|
| **Static websites** (HTML/CSS/JS, Hugo, Jekyll, Astro static output) | Output is just files. Hosting them is solved better by CDNs. | GitHub Pages, Netlify, Cloudflare Pages, S3 + CloudFront. (Docker may still help to *build* the site reproducibly in CI.) |
| **Simple one-off scripts** (rename files, quick data fix) | Nobody else runs it; dependencies are small | `python -m venv`, `pipx`, or `uv run script.py` |
| **Desktop GUI applications** (Windows/macOS native apps, Electron apps for end users) | Containers are built for server-side Linux processes; GUIs and windowing systems are painful in them. Users do not have Docker. | Native installers (MSI, DMG, AppImage). (Docker can still help your *build* agent.) |
| **Serverless-only apps** (AWS Lambda, Cloud Functions, Vercel/Netlify functions with zip deployments) | The platform owns the runtime. You upload code, not an image | The platform's tooling (SAM, Serverless Framework, `vercel deploy`). Note: Lambda *can* run container images, which you choose only if you need big dependencies, up to 10 GB |
| **Mobile apps themselves** (Android APK/AAB, iOS IPA) | They run on a phone, not in a container | Android Studio / Gradle / Xcode (see section 7.5) |
| **Tiny hobby projects with one dependency** | Setup cost of Docker exceeds the problem | A package manager and a README |
| **Apps tightly bound to hardware that containers cannot easily reach** (USB-attached medical devices on Windows, GPU-accelerated desktop apps, some proprietary drivers) | Hardware access via containers is possible on Linux (`--device`) but awkward on Windows/macOS | Native install; or a thin gateway service in a container on a Linux box next to the device |
| **Windows-only .NET Framework desktop software** | Windows containers exist, but are a niche and heavy | Normal Windows deployment |

> **Analogy.** You do not need a shipping container to carry a lunchbox. But you do need one the moment you are shipping 10,000 lunchboxes across an ocean.

---

## 7.5 "Does an Android app need Docker?" (the direct answer)

This section is written for you, so let us be exact. Split the question into pieces.

### 7.5.1 The Android app itself: **No**

An Android app is compiled into an **APK** (Android Package) or **AAB** (Android App Bundle) and runs on a phone or tablet using the **Android Runtime (ART)**. It does not run in a Docker container on the device. You develop it in **Android Studio**, build with **Gradle** (the build tool), and ship it through Google Play or a device-management system.

```
Your laptop                         Phone / tablet
+-----------------------+           +--------------------------+
| Android Studio        |  APK/AAB  | Android OS               |
| Gradle build          | --------> |   ART runtime            |
| (no Docker needed)    |           |     Your app             |
+-----------------------+           +--------------------------+
```

There is no step in the normal Android developer workflow where Docker is *required*. Many Android developers go their entire careers without touching it.

### 7.5.2 Docker around Android: **Often yes**

Docker enters the picture when you look at the *systems* around the app. Here are the seven common places, from most to least common.

#### (1) Reproducible CI build images

Your CI server (GitHub Actions, GitLab CI, Jenkins, Bitbucket) has to build your app. That requires the right JDK, the Android SDK, platform and build-tools versions, NDK (if you have native code), Gradle, licences accepted, and so on. Without Docker you either rely on whatever the hosted runner has (which changes over time) or you maintain a snowflake Jenkins machine that nobody dares reinstall.

With Docker you describe the build environment in a `Dockerfile`, build it once, and every build, from every developer machine and every CI run, uses **the same image**. You can even reproduce a CI failure locally by running the exact same image. See the full example in section 7.6.

#### (2) Backend services the app talks to

Most real apps talk to a server: REST/GraphQL APIs, authentication, push notification fan-out, file storage, analytics, feature flags. Those services are *classic* Docker territory:

* a REST API in Kotlin (Ktor/Spring), Node, Python or Go;
* auth servers such as Keycloak or Authentik;
* push notification servers (your own FCM-sending service);
* Firebase alternatives you can self-host: **Supabase**, **Appwrite**, **PocketBase**, **Parse Server**;
* storage with MinIO (S3-compatible), databases such as Postgres, caches such as Redis.

During development you start the whole backend locally with `docker compose up`, point the emulator at `http://10.0.2.2:8080` (the special address the Android emulator uses for "the host computer's localhost"), and develop against a realistic server without a cloud account.

#### (3) Running the Android emulator in containers (CI only, Linux only)

You *can* run an Android emulator inside a container for automated UI tests, but with big caveats:

* It needs hardware virtualization, i.e. **KVM** (Kernel-based Virtual Machine, Linux's built-in hypervisor). The container must be given `/dev/kvm`.
* So it works on **Linux hosts with KVM** (bare-metal Linux or cloud VMs that allow nested virtualization). On Docker Desktop for Windows/macOS it generally does not give you accelerated emulation; on those systems use the emulator from Android Studio directly.
* Community images exist, for example `budtmo/docker-android` (emulator plus noVNC web viewer and Appium support). The project's own docs and community guides say it needs Linux with KVM and recommend several GB of RAM per instance; I could not verify exact current tag names or requirements from the project's own site while writing this, so read its README before relying on it. Google also publishes **android-emulator-container-scripts** to build emulator containers.
* Many teams skip all this and use **Firebase Test Lab**, **AWS Device Farm**, **BrowserStack** or GitHub Actions runner features (GitHub's Linux runners can use KVM for emulators if you enable it; check their current docs).

Typical docker run shape (illustrative, see the image's README for the real flags):

```bash
docker run -d --device /dev/kvm -p 6080:6080 -e EMULATOR_DEVICE="Samsung Galaxy S10" -e WEB_VNC=true --name android-emu budtmo/docker-android:emulator_14.0
```

* `-d` runs it in the background ("detached").
* `--device /dev/kvm` hands the host's KVM device file to the container so the emulator can use hardware acceleration.
* `-p 6080:6080` publishes the noVNC web viewer so you can watch the emulator at `http://localhost:6080`.
* `-e ...` passes configuration as environment variables.
* The tag (`emulator_14.0`) selects an Android version; tag names change over time, so check the repository.

#### (4) Developer tooling servers: Sonar, Artifactory, Nexus, Jenkins

The infrastructure that supports Android teams is overwhelmingly packaged as containers:

| Tool | Purpose | Image (verify tag before use) |
|---|---|---|
| **SonarQube** | Code quality and static analysis | `sonarqube` (official image) |
| **Sonatype Nexus Repository** | Host Maven/Gradle artifacts (your own AARs) | `sonatype/nexus3` |
| **JFrog Artifactory** | Same, enterprise flavour | `releases-docker.jfrog.io/jfrog/artifactory-oss` |
| **Jenkins** | CI server | `jenkins/jenkins:lts` |
| **GitLab / Gitea** | Source hosting | `gitlab/gitlab-ce`, `gitea/gitea` |
| **Gradle Build Cache Node** | Share build cache across the team to cut build times | `gradle/build-cache-node` |

Running these in containers means a one-command install, trivial upgrades (pull a new tag), and easy backup (copy the volume).

#### (5) Test mocks and fakes

Instead of waiting for the real backend, mock it:

* **WireMock** (`wiremock/wiremock`) lets you declare "when the app calls `GET /api/tags/123`, return this JSON."
* **MockServer** (`mockserver/mockserver`) does similar, plus verification of requests.
* **Testcontainers** (a library for JVM, Python, Node, Go, .NET) starts real containers (Postgres, Kafka, a mock) from inside your integration tests, then throws them away.

Mini example of a WireMock stub used by instrumentation tests:

```bash
docker run -d --name wiremock -p 8080:8080 -v "$PWD/mocks:/home/wiremock" wiremock/wiremock:latest
```

* `-p 8080:8080` exposes the mock on host port 8080, reachable from the Android emulator at `http://10.0.2.2:8080`.
* `-v "$PWD/mocks:/home/wiremock"` mounts your local `mocks/` folder (containing `mappings/` and `__files/`) so you can edit stubs without rebuilding anything.
* (In real projects, pin a version instead of `latest`; see [Chapter 10](10-security.md).)

#### (6) Edge, IoT and RFID-flavoured systems

If your Android app is a handheld that talks to RFID readers, barcode scanners or sensors, the *data* eventually has to go somewhere. That "somewhere" is typically a small Linux server, an edge gateway (an industrial PC or Raspberry-Pi-class device near the readers) and a cloud backend. This is where Docker shines:

```
 Handheld (Android app)          Edge gateway (Linux box)                    Cloud / Data center
+----------------------+       +-----------------------------------+       +----------------------+
| RFID SDK, scanning   | MQTT/ | Mosquitto (MQTT broker)           | MQTT/ | Postgres / Timescale |
| UI, local cache      | HTTPS | Node-RED (flows/transform)        | HTTPS | Location/inventory   |
+----------------------+ ----> | InfluxDB (time-series)            | ----> | API + dashboards     |
                               | Grafana (local dashboards)        |       +----------------------+
                               | Your location/inventory service   |
                               +-----------------------------------+
                                  every box above = one container
```

* **MQTT** is a tiny publish/subscribe messaging protocol popular in IoT. Devices *publish* messages to *topics* (like `rfid/gate1/tags`) and other programs *subscribe* to topics. A **broker** such as **Mosquitto** relays messages.
* **InfluxDB** is a **time-series database** (optimized for "value at time T" data such as tag read counts per second).
* **Grafana** draws dashboards from databases.
* **Node-RED** is a visual, flow-based tool to wire data sources together (MQTT in, transform, HTTP/DB out).
* A **location or inventory backend** is your own service that turns raw reads into "this pallet is in zone B."

Docker's advantages at the edge: update by pulling a new image; roll back by starting the old tag; the same Compose file runs on your laptop and on the gateway; and ARM64 (multi-arch) images let one build serve a Raspberry-Pi-class device and an x86 server. Section 7.7 has a full Compose example.

#### (7) Build tooling and release pipelines

Beyond builds: running **lint, unit tests, dependency scans and signing** inside a controlled image; **fastlane** (a release automation tool) in a container; **bundletool** to produce device-specific APKs. Same benefit: reproducibility.

### 7.5.3 Summary table for the Android developer

| Question | Answer |
|---|---|
| Do I need Docker to build and run my Android app on my phone? | **No** |
| Do I need Docker to ship to Google Play? | **No** |
| Is it useful for my CI build agents? | **Yes, very** |
| Is it useful for the backend my app talks to? | **Yes, very** |
| Is it useful for running Sonar / Nexus / Jenkins? | **Yes** |
| Is it useful for mocking APIs in tests? | **Yes** |
| Is it useful for running the emulator? | **Only on Linux CI with KVM**; otherwise use Android Studio's emulator or a device farm |
| Is it useful for IoT/RFID gateways and data platforms? | **Yes, very** |

---

## 7.6 Worked example: a Dockerfile-based Android CI build image

Goal: an image that contains JDK 17, the Android command-line SDK tools, and a pinned platform + build-tools, so that `./gradlew assembleRelease` behaves identically everywhere.

> **Tip.** Gradle itself is usually downloaded by the Gradle *wrapper* (`gradlew`) in your repo, so you do not need to install Gradle in the image. The image provides the JDK and the Android SDK.

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

Line by line:

| Line | What it does |
|---|---|
| `# syntax=docker/dockerfile:1` | Tells BuildKit (Docker's modern build engine) to use the latest stable Dockerfile syntax. |
| `FROM eclipse-temurin:17-jdk-jammy` | Starts from an Official Image containing JDK 17 (Eclipse Temurin is a free OpenJDK build) on Ubuntu 22.04 ("jammy"). Android Gradle Plugin 8.x requires JDK 17; adjust to match your project. |
| `ARG ...` | Build-time variables with defaults. Version numbers are **examples**: check the Android developer site for the current command-line tools number and installed API levels, and match `compileSdk` in your project. |
| `ENV ANDROID_HOME=...` | Environment variables that persist in the image so Gradle and `sdkmanager` find the SDK. |
| `ENV PATH=...` | Adds SDK tools to the shell's search path so you can type `sdkmanager` directly. |
| `RUN apt-get update && apt-get install ... && rm -rf /var/lib/apt/lists/*` | Installs download/unzip tools **and** deletes the package index in the *same layer* so it does not bloat the image (see [Chapter 4](04-dockerfile-deep-dive.md) on layers). `--no-install-recommends` skips optional extras. |
| `curl -fsSL -o ... URL` | Downloads Google's command-line tools zip. `-f` fails on HTTP errors, `-s` silences progress, `-S` still shows errors, `-L` follows redirects. |
| `unzip` / `mv ... latest` | The zip contains a folder `cmdline-tools/`; `sdkmanager` insists on the layout `cmdline-tools/latest/bin`, so we rename it. |
| `yes \| sdkmanager --licenses` | Automatically answers "y" to every licence prompt. By running this you are accepting the Android SDK licences, so read them. |
| `sdkmanager "platform-tools" "platforms;..." "build-tools;..."` | Installs exactly the pieces you need, and only those, to keep the image smaller. |
| `useradd ... USER builder` | Creates and switches to a **non-root** user. Many CI systems and Gradle itself behave better, and it is safer. |
| `WORKDIR /workspace` | Sets the directory the source code will be mounted into. |
| `CMD [...]` | Default command if none is given. Harmless, prints versions. |

Build and use it:

```bash
docker build -t myorg/android-ci:2026-10 android-ci/
```

* `-t myorg/android-ci:2026-10` names (tags) the image. Using a date or version tag, not `latest`, makes builds reproducible.
* `android-ci/` is the build context (the folder containing the Dockerfile).

```bash
docker run --rm \
  -v "$PWD":/workspace \
  -v gradle-cache:/home/builder/.gradle \
  myorg/android-ci:2026-10 \
  ./gradlew assembleDebug --no-daemon
```

* `--rm` deletes the container when it exits (the image stays).
* `-v "$PWD":/workspace` mounts your project into the container. On Windows PowerShell use `${PWD}` instead of `"$PWD"`.
* `-v gradle-cache:/home/builder/.gradle` uses a **named volume** to keep Gradle's downloaded dependencies between runs; without it every build re-downloads the internet. (Volumes are covered in [Chapter 5](05-storage-and-networking.md).)
* `./gradlew assembleDebug --no-daemon` is the build; `--no-daemon` avoids leaving a background Gradle process in a throw-away container.

Expected tail of the output:

```
BUILD SUCCESSFUL in 2m 31s
48 actionable tasks: 48 executed
```

Using it in **GitHub Actions** (the job runs inside your image):

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

* `runs-on: ubuntu-latest` is the VM that hosts the job.
* `container.image` makes every step run inside your image rather than directly on the VM.
* `actions/checkout@v4` fetches your source. (For maximum safety in CI pin actions to a full commit SHA; see [Chapter 10](10-security.md) on the 2026 Trivy action compromise.)

**Do not put your signing keystore or its passwords in this image.** Inject them at job time from the CI secret store (see [Chapter 10](10-security.md), section on secrets). Anything baked into an image can be extracted by anybody who can pull the image.

---

## 7.7 Worked example: IoT/RFID-style backend in Compose

We will build the stack from the diagram: an **API** service, **PostgreSQL**, an **MQTT broker** (Mosquitto), **InfluxDB**, and **Grafana**. This is a development/gateway-class setup; for production see [Chapter 11](11-production-and-enterprise.md).

Folder layout:

```
iot-stack/
  compose.yaml
  .env                      (NOT committed; holds passwords)
  mosquitto/
    mosquitto.conf
  api/
    Dockerfile
    (your service code)
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

* `listener 1883` makes Mosquitto listen on the standard MQTT port. Since Mosquitto 2.0 it only listens on localhost unless you declare a listener, so this line is required for other containers to connect.
* `allow_anonymous true` lets clients connect without a password. **Fine for a laptop demo, wrong for anything networked**: use a password file or TLS in real deployments.
* `persistence true` plus `persistence_location` makes the broker save queued messages to disk.

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

`.env` (create it yourself; never commit it; values below are placeholders):

```
POSTGRES_PASSWORD=change-me-locally
INFLUX_PASSWORD=change-me-too-8chars+
GRAFANA_PASSWORD=change-me-as-well
```

Now the explanation, piece by piece.

| Piece | Meaning |
|---|---|
| `name: rfid-edge` | Project name; Compose prefixes networks/volumes with it (`rfid-edge_pgdata`). |
| `services:` | The list of containers that make up the app. |
| `api.build: ./api` | Build this service's image from `./api/Dockerfile` instead of pulling it. |
| `ports: "127.0.0.1:8080:8080"` | Publish container port 8080 on the host, **but only on the loopback interface** so other machines on the network cannot reach it. Omitting `127.0.0.1:` would publish on all interfaces. |
| `environment:` (api) | Env vars the service reads. `${POSTGRES_PASSWORD}` is substituted from `.env` at `docker compose up` time. |
| `db`, `mqtt`, `influx` as hostnames | Compose creates a private network where every service is reachable **by its service name**. So the API connects to `db:5432`, not `localhost`. (This is the #1 beginner confusion; `localhost` inside a container means the container itself.) |
| `depends_on ... condition: service_healthy` | Start the API only after Postgres reports healthy (see the healthcheck). Plain `depends_on` only waits for *start*, not *readiness*. |
| `image: postgres:17` | Use the official PostgreSQL 17 image. Pinning a major version avoids surprise upgrades (a Postgres data directory cannot be opened by a different major version without migration). |
| `volumes: pgdata:/var/lib/postgresql/data` | A **named volume** so the database survives `docker compose down`. Without it, you lose data when the container is removed. (Check the Postgres image docs if you move to a newer major version; the recommended data-path layout changed in recent versions.) |
| `healthcheck.test` | Command Docker runs every 5s inside the container; exit code 0 means healthy. `pg_isready` is Postgres' own readiness probe. |
| `restart: unless-stopped` | Restart after crashes or reboots unless you explicitly stopped it. Essential for edge gateways. |
| Mosquitto `:ro` mount | Config is mounted **read-only** (`:ro`) so the container cannot alter it. |
| `DOCKER_INFLUXDB_INIT_*` | Special variables of the official InfluxDB 2 image that perform first-run setup (admin user, organization, first bucket). They are ignored after the data volume is initialized. |
| `GF_SECURITY_ADMIN_PASSWORD` | Grafana's convention: variables starting with `GF_` override config settings. |
| `volumes:` (top-level) | Declares the named volumes Compose will create and manage. |

Run it:

```bash
docker compose up -d
docker compose ps
```

* `up -d` creates networks, volumes and containers, then returns your terminal (`-d` = detached).
* `ps` lists the running services.

Example output of `docker compose ps` (trimmed):

```
NAME                  IMAGE                       STATUS                    PORTS
rfid-edge-api-1       rfid-edge-api               Up 20 seconds             127.0.0.1:8080->8080/tcp
rfid-edge-db-1        postgres:17                 Up 26 seconds (healthy)   5432/tcp
rfid-edge-grafana-1   grafana/grafana-oss:latest  Up 20 seconds             127.0.0.1:3000->3000/tcp
rfid-edge-influx-1    influxdb:2                  Up 26 seconds             8086/tcp
rfid-edge-mqtt-1      eclipse-mosquitto:2         Up 26 seconds             0.0.0.0:1883->1883/tcp
```

Quick smoke test of MQTT (uses a throw-away container with the Mosquitto client tools, on the same Compose network):

```bash
docker compose exec mqtt mosquitto_pub -t rfid/gate1/tags -m '{"epc":"E2801160600002054F5B0001","rssi":-52}'
docker compose exec mqtt mosquitto_sub -t 'rfid/#' -C 1
```

* `exec mqtt` runs a command inside the already-running `mqtt` service container.
* `mosquitto_pub -t <topic> -m <message>` publishes one message. The EPC above is a made-up example.
* `mosquitto_sub -t 'rfid/#' -C 1` subscribes to every topic under `rfid/` (`#` is the MQTT multi-level wildcard) and exits after 1 message (`-C 1`). Run the subscriber first in another terminal to see the message arrive.

Tear down (keeps data):

```bash
docker compose down
```

Tear down **and delete data volumes** (destructive; only do it when you mean it):

```bash
docker compose down -v
```

---

## 7.8 Docker vs the alternatives

"Docker or not" is rarely a pure either/or. These are the main alternatives and how they compare.

| Approach | What it gives you | What it lacks | Pick it when |
|---|---|---|---|
| **Language virtual environments** (`venv`, `uv`, `nvm`, `rbenv`, `sdkman`) | Per-project language packages; zero overhead | Cannot isolate system libraries, databases, OS packages, or ports. Not a deployment artifact. | Pure-language projects with no system dependencies, for local development |
| **Virtual machines** (VirtualBox, VMware, Hyper-V, EC2 instances) | Full OS isolation, different kernels, strongest boundary | Heavy (GBs, minutes to boot), harder to version, slow to distribute | Need a different OS/kernel, strong isolation (multi-tenant), legacy OS software |
| **Docker containers** | Light, fast, portable images, rich ecosystem, shares host kernel | Shared kernel (weaker isolation than a VM), Linux-centric, learning curve | Default choice for services and pipelines |
| **Serverless functions** (Lambda, Cloud Functions) | No servers to manage, scale to zero, pay per call | Execution time and memory limits, cold starts, vendor lock-in, awkward for long-running or stateful work | Event-driven glue, spiky traffic, small APIs |
| **PaaS** (Heroku, Render, Fly.io, Railway, Cloud Run, App Runner) | `git push` or image push and it runs | Less control; can get pricey | Small teams wanting less ops. Many PaaS platforms *run Docker images* anyway |
| **Buildpacks** (Cloud Native Buildpacks, Heroku-style) | Build an image from source with no Dockerfile; maintained by the platform; automatic security patches of base layers | Less customizable; another tool to learn | You have standard apps (Spring Boot, Node) and want zero Dockerfile maintenance (`pack build`; Spring Boot's `bootBuildImage`) |
| **Nix / NixOS / Guix** | Truly bit-for-bit reproducible builds and environments; per-project shells (`nix develop`) without a daemon | Steep learning curve; unfamiliar to most teams | You value reproducibility above all and have time to learn it. Nix can also *produce* Docker images |
| **Config management** (Ansible, Chef, Puppet) on VMs | Mature way to configure long-lived servers | Servers drift; slower, mutable infrastructure | Existing fleets of VMs; Docker and Ansible often combine (Ansible installs Docker and deploys Compose files) |
| **Podman / containerd / Kubernetes** | Other container tools; Podman is daemonless and rootless by default | Different ergonomics | Same *image format* (OCI), so skills transfer; Docker is the friendliest entry point |

> **Important.** Images are an open standard (the **OCI**, Open Container Initiative, image spec). Learning Dockerfiles is not lock-in: Podman, Kubernetes, ECS, Cloud Run, Lambda (container images) and others all run them.

---

## 7.9 When containers are the wrong tool

Be cautious, and consider a different design, in these situations.

1. **Stateful, performance-critical databases in production** without a plan. Containers can run databases well, but you must manage volumes, backups, I/O performance and upgrades. Many teams use a managed database (RDS, Cloud SQL, Atlas) in production and containers only in dev/test.
2. **GUI desktop apps for end users.** You can forward X11 or use VNC/noVNC, but it is clunky. Package natively.
3. **Hard real-time or hardware-timing-critical code.** Containers share a kernel and a scheduler; you can pin CPUs and give real-time priority on Linux, but it is delicate. Embedded control loops stay on the metal (or an RTOS).
4. **Software that needs a different kernel or kernel modules** (some network appliances, custom filesystems). Use a VM.
5. **Strong multi-tenant isolation of hostile code.** A container is not a security boundary as strong as a VM ([Chapter 10](10-security.md)). Use microVMs (Firecracker, gVisor, Kata Containers) or VMs for hostile workloads.
6. **Windows-native or macOS-native software.** Linux containers cannot run them.
7. **When nobody on the team can operate it.** Containers in production mean image registries, scanning, orchestration and monitoring. If you have one app and one server, a systemd service might be the more maintainable choice.
8. **Pet projects where time-to-first-run matters more than reproducibility.**

---

## 7.10 What Docker really costs (cost/complexity trade-offs)

| Cost | Details | Mitigation |
|---|---|---|
| **Learning curve** | Images, layers, networking, volumes, Compose, registries | This guide; start with Compose only |
| **Build time** | Image builds can be slow (especially ML images) | Layer caching, BuildKit cache mounts, CI build cache ([Chapter 4](04-dockerfile-deep-dive.md)) |
| **Disk space** | Images and volumes accumulate; ML images are many GB | Regular `docker system df` / pruning policy (be careful: pruning deletes things); slim bases |
| **Debugging indirection** | Another layer between you and the process; logs, shells, networking are via Docker | `docker logs`, `docker exec`, `docker inspect` ([Chapter 3](03-core-cli-commands.md), [Chapter 12](12-troubleshooting-and-cheatsheet.md)) |
| **Docker Desktop on Windows/macOS** | Runs a hidden Linux VM; file sharing across the boundary is slower; uses RAM | Keep source inside the Linux filesystem (WSL2) when possible; set resource limits |
| **Security responsibility** | You own the base image's patches, secrets, and the runtime flags | [Chapter 10](10-security.md) |
| **Licensing** | Docker Desktop requires a paid subscription for larger companies (check Docker's current terms: historically businesses above a size/revenue threshold). Docker Engine on Linux, Podman, Rancher Desktop, Colima are alternatives | Read the current terms; consider alternatives for company machines |
| **Operational overhead in production** | Orchestration (Kubernetes) is its own skill set | Use a managed service (ECS/Fargate, Cloud Run) until you need more |
| **Cloud cost** | An idle container still uses memory; many small containers can cost more than one big process | Right-size limits ([Chapter 11](11-production-and-enterprise.md)) |

A fair rule of thumb: Docker **adds** a few hours of setup and a modest ongoing maintenance load, and it **removes** an open-ended stream of "works on my machine" problems. For teams larger than one person, or projects that live more than a few months, that trade is almost always positive.

---

## 7.11 Adoption checklist

Use this when introducing Docker to a project or team. Work top to bottom.

**Phase 0: Decide**
- [ ] I ran the flowchart in section 7.2 and the answer was "yes" for at least the dev database or the CI.
- [ ] The team agrees on which container runtime to use (Docker Engine, Docker Desktop, Podman, Colima) and has checked licensing.

**Phase 1: Local development**
- [ ] A `Dockerfile` exists, builds cleanly with `docker build`, and uses a pinned, trusted base image.
- [ ] A `.dockerignore` file excludes `.git`, build output, local env files and secrets.
- [ ] A `compose.yaml` starts everything with one command (`docker compose up`).
- [ ] Secrets come from a git-ignored `.env` file or a secret store, never from the Dockerfile.
- [ ] The README documents: install Docker, `docker compose up`, where to open the app.
- [ ] Data that must persist is in a named volume; everyone knows `down -v` deletes it.

**Phase 2: CI**
- [ ] CI builds the image on every pull request.
- [ ] Image is tagged with the git commit SHA (and a semantic version on release), not only `latest`.
- [ ] Layer caching is configured so builds are fast.
- [ ] Tests run inside the same image that will ship.
- [ ] A vulnerability scan runs (Trivy, Scout or Grype) and fails the build on critical findings you can fix.
- [ ] Actions/plugins used in CI are pinned to immutable versions.

**Phase 3: Delivery**
- [ ] A private registry (GHCR, ECR, GCR/Artifact Registry, ACR, Docker Hub private repos) holds the images.
- [ ] Deployments reference images by version tag or digest.
- [ ] Containers run as non-root with sensible resource limits and health checks.
- [ ] Logs go to stdout/stderr and are collected centrally.
- [ ] There is a rollback plan: redeploy the previous tag.

**Phase 4: Operate**
- [ ] Base images are rebuilt on a schedule to pick up security patches.
- [ ] Someone owns image hygiene (pruning old tags in the registry, rotating tokens).
- [ ] Backups of volumes/databases are tested by restoring.
- [ ] Monitoring and alerting cover container restarts and resource usage.

---

## 7.12 Common mistakes

| Mistake | Why it hurts | Better |
|---|---|---|
| "Dockerizing" a project that has no portability or dependency problem | Added complexity with no payoff | Use the flowchart; start with just the database |
| Using `localhost` between containers | `localhost` inside a container is that container itself | Use the service name (`db`, `mqtt`) on a Compose network |
| Baking secrets, signing keystores or API keys into images | Anyone with the image can extract them | Inject at runtime; see [Chapter 10](10-security.md) |
| Using `latest` for everything | Builds change under your feet | Pin tags; better, pin digests for production |
| Running databases in containers without volumes | Data disappears with the container | Named volumes + backups |
| Expecting the Android emulator to run well in Docker Desktop on Windows/macOS | No hardware acceleration | Run the emulator natively or use cloud device farms |
| Publishing ports to the whole network (`-p 1883:1883`) by accident | Exposes services (and Docker bypasses host firewalls like ufw, see Chapter 10) | Bind to `127.0.0.1` unless remote access is intended |
| One giant container running five services | Loses all the benefits (independent restarts, scaling) | One main process per container; use Compose |
| Treating containers as VMs ("I will SSH in and fix it") | Changes vanish on recreate; drift | Change the Dockerfile, rebuild, redeploy |

---

## 7.13 Try it yourself

1. **Score your own project.** Pick any project you have and answer the 8 questions in section 7.2. Decide: skip, partial, or full.
2. **Disposable database.** Run a Postgres 17 container, connect with `psql`, create a table, then delete the container and observe the data is gone (no volume). Then repeat with a named volume and observe the data survive.

   ```bash
   docker run -d --name pg-test -e POSTGRES_PASSWORD=devonly -p 127.0.0.1:5432:5432 postgres:17
   docker exec -it pg-test psql -U postgres -c "create table t(x int); insert into t values (1); select * from t;"
   docker rm -f pg-test
   ```

   * `-e POSTGRES_PASSWORD=devonly` is required by the image to set the superuser password (a throw-away dev value).
   * `docker exec -it ... psql` opens a SQL command inside the container.
   * `docker rm -f` forcibly removes **this test container only**.
3. **Build the Android CI image** from section 7.6 against your own project and compare its build time and output to your normal build.
4. **Stand up the IoT stack** from section 7.7. Publish an MQTT message, then open Grafana at `http://localhost:3000` (user `admin`, password from your `.env`) and add InfluxDB as a data source (URL `http://influx:8086`, because Grafana is inside the Compose network).
5. **Write down** three things in your current work that would be easier if the environment were code.

---

## 7.14 Chapter summary

* Docker is worth it when someone else or somewhere else has to run your software, or when many services must coexist.
* It is a poor fit for static sites, tiny scripts, desktop GUI apps, serverless-only designs and mobile apps themselves.
* An **Android app does not need Docker**, but the CI that builds it, the backend it talks to, the tooling servers (Sonar, Nexus, Jenkins), API mocks, and IoT/edge/RFID data platforms (MQTT, InfluxDB, Grafana, Node-RED) are all excellent Docker candidates.
* Docker coexists with venvs, VMs, serverless, buildpacks and Nix; images are an open (OCI) standard, so the skill is portable.
* Containers cost learning time, disk, and security responsibility; use the adoption checklist to introduce them in stages.

Next: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md), where containers go from "nice to have" to "practically mandatory."
