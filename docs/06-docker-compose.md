# Chapter 6: Docker Compose

Docker Compose defines and runs multi-container applications from one YAML file. This chapter covers the file format, every commonly used service option, environment variable handling, volumes, networks, startup order, healthchecks, profiles, multiple files, secrets, hot reload with `watch`, the command reference, a staged tutorial (one service to a dev/prod stack), and when Compose is enough in production.

**Prerequisites:** [Chapter 4](04-dockerfile-deep-dive.md) (Dockerfiles) and [Chapter 5](05-storage-and-networking.md) (volumes and networks).

---

## What Compose is and why you need it

**Docker Compose** describes an application's services, networks and volumes in a YAML file (conventionally `compose.yaml`) and creates and starts everything with one command.

**YAML** is a data format built from indentation, `key: value` pairs and `- list items`. Indent with spaces, never tabs.

Without Compose, a web app with Postgres and Redis needs several commands in the right order:

```bash
docker network create appnet
docker volume create pgdata
docker run -d --name db --network appnet -v pgdata:/var/lib/postgresql \
  -e POSTGRES_PASSWORD=devonly -e POSTGRES_DB=notes postgres:18
docker run -d --name redis --network appnet redis:7-alpine
docker build -t notes-web .
docker run -d --name web --network appnet -p 8000:8000 \
  -e DB_HOST=db -e REDIS_URL=redis://redis:6379/0 notes-web
```

With Compose, the same setup is a file in Git plus one command:

```bash
docker compose up -d
```

Benefits:

- One version-controlled definition of the stack.
- One command to build, start, stop and delete it.
- A private network per project, with service names as DNS names ([Chapter 5](05-storage-and-networking.md#b2-default-bridge-vs-user-defined-bridge)).
- Reproducible setups across machines and CI.
- Development tools: live reload (`watch`), one-off commands (`run`), aggregated logs.
- Idempotent: running `up` again changes only what changed.

### The Compose application model

| Concept | Meaning |
|---|---|
| **Project** | One deployment of the app. Its name groups and isolates its resources |
| **Service** | A component (web, db, worker). Defined once; runs as one or more identical **containers** |
| **Network** | Connects the containers of services |
| **Volume** | Persistent storage |
| **Config** | Non-sensitive configuration files mounted into containers |
| **Secret** | Sensitive data mounted as files |

A service is the definition; a container is a running instance. `docker compose up --scale web=3` runs three containers of the `web` service.

### Compose V1 vs V2

`docker-compose` (hyphen) is the original Python program, Compose V1, which reached end of life in 2023. Compose V2 is written in Go and ships as a Docker CLI plugin: `docker compose` (space). This chapter uses the space form.

---

## Checking your install

Docker Desktop bundles Compose. On a Linux server with Docker Engine, install the `docker-compose-plugin` package ([Chapter 2](02-installation-and-operating-systems.md)).

```bash
docker compose version
```

Example output (the version used to write this guide):

```
Docker Compose version v5.6.1
```

- Use the `version` subcommand, not `--version`.
- Compose and Docker Engine version numbers differ (this guide was written with Engine 29.9.2 and Compose 5.6.1; older tutorials show 2.x).
- The file format, the **Compose Specification**, is versioned separately and is always the latest; there is no file-format version to choose.
- `docker compose --help` lists all commands (see the [commands reference](#compose-commands-reference)).

---

## Anatomy of a Compose file

### File names

Compose searches the current directory, then parent directories, in this order:

1. `compose.yaml` (preferred)
2. `compose.yml`
3. `docker-compose.yaml` (legacy, supported)
4. `docker-compose.yml` (legacy, supported)

If `compose.yaml` and a legacy name both exist, `compose.yaml` wins. Compose also merges `compose.override.yaml` automatically if present ([overrides](#multiple-compose-files-and-overrides)).

### The `version:` key is obsolete

```yaml
version: "3.8"      # do not write this
services:
  ...
```

According to the Compose Specification, the top-level `version` property is obsolete. It is informative only, Compose ignores it, always validates against the latest schema, and prints a warning if it is present. Remove it. The message `the attribute 'version' is obsolete` means the same.

### A minimal file

```yaml
name: demo

services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
```

| Line | Meaning |
|---|---|
| `name: demo` | Optional **project name**, used as a prefix for container, network and volume names. Defaults to the folder name. |
| `services:` | Required section listing the services. |
| `web:` | The **service name**. It is also the DNS name other services use. |
| `image: nginx:1.27-alpine` | Image to run. |
| `ports:` | Port mappings, `"HOST:CONTAINER"`. |
| `- "8080:80"` | Publish container port 80 on host port 8080. Quote port mappings: YAML can read unquoted `22:22` as a base-60 number. |

### Top-level elements

| Key | Purpose |
|---|---|
| `name` | Project name |
| `services` | The containers (required) |
| `networks` | Named networks |
| `volumes` | Named volumes |
| `configs` | Non-sensitive config files |
| `secrets` | Sensitive files |
| `include` | Pull in other Compose files |
| `x-...` | **Extension fields**: custom reusable fragments, ignored by Compose |

---

## Service options in detail

### `image` and `build`

```yaml
services:
  web:
    image: ghcr.io/your-user/notes-web:1.4.2   # use an existing image (pulled if missing)

  api:
    build: .                                    # build from ./Dockerfile
```

Long form of `build`:

```yaml
  api:
    build:
      context: ./api                  # build context directory (default: .)
      dockerfile: Dockerfile.prod     # Dockerfile, relative to context
      target: runtime                 # multi-stage: build up to this stage
      args:                           # build arguments (ARG in the Dockerfile)
        PYTHON_VERSION: "3.13"
      platforms:                      # multi-platform build targets
        - linux/amd64
        - linux/arm64
      cache_from:
        - type=registry,ref=registry.example.com/me/api:buildcache
      cache_to:
        - type=registry,ref=registry.example.com/me/api:buildcache,mode=max
      secrets:                        # build-time secrets (see Secrets and configs)
        - pip_token
      ssh:
        - default                     # forward the SSH agent
    image: registry.example.com/me/api:1.0     # name/tag for the built image
```

- `context`: directory sent to the builder; `COPY` paths are relative to it ([Chapter 4](04-dockerfile-deep-dive.md#4-build-context-and-dockerignore)).
- `target`: stage of a [multi-stage Dockerfile](04-dockerfile-deep-dive.md#11-multi-stage-builds) to build.
- `args`: values for Dockerfile `ARG`s. Do not put secrets here.
- With both `build` and `image` set, Compose builds and tags the result with the `image` name. One file then serves dev (build locally) and prod (pull the same name from a registry).
- Without `image`, the built image is named `<project>-<service>`.
- `docker compose build` builds; `docker compose up --build` builds then starts. `up` alone does not rebuild unless the image is missing.

**Compose Bake:** Compose can delegate builds to **Docker Bake** (buildx's build orchestrator, for building many images in parallel). Docker's documentation describes enabling it with `COMPOSE_BAKE=true docker compose build`, or permanently with `"plugins": {"compose": {"build": "bake"}}` in `~/.docker/config.json`. Unverified: the documentation checked describes it as opt-in; whether a recent release made it the default is unconfirmed. Check the `docker compose build` output on your version.

### `ports` and `expose`

```yaml
    ports:
      - "8000:8000"                  # host 8000 -> container 8000, all interfaces
      - "127.0.0.1:5432:5432"        # reachable only from this machine
      - "9090:9090/udp"              # UDP
      - target: 80                   # long syntax
        published: "8080"
        protocol: tcp
        host_ip: 127.0.0.1
    expose:
      - "8000"                       # documentation only (like Dockerfile EXPOSE)
```

`ports` publishes to the host ([Chapter 5](05-storage-and-networking.md#b3-port-publishing)). `expose` publishes nothing. Do not publish ports that only other containers need.

### `environment` and `env_file`

```yaml
    environment:
      APP_ENV: production            # map syntax
      LOG_LEVEL: info
      DEBUG:                         # no value: pass through from the shell if set
    # list syntax:
    # environment:
    #   - APP_ENV=production
    env_file:
      - .env.web                     # file of KEY=value lines
      - path: .env.local
        required: false              # do not fail if the file is missing
```

Precedence rules: [next section](#environment-variables-interpolation-and-precedence).

### `volumes`, `networks`

```yaml
    volumes:
      - pgdata:/var/lib/postgresql          # named volume (declared at top level)
      - ./config/app.toml:/etc/app.toml:ro  # bind mount, read-only
      - type: tmpfs                         # long syntax
        target: /tmp
    networks:
      - frontend
      - backend
```

See [Volumes](#volumes) and [Networks](#networks). `depends_on`, `healthcheck`, `restart`, `command` and `entrypoint` have their own sections below.

### Other options

| Option | What it does |
|---|---|
| `container_name: web` | Fixed container name. Avoid: it prevents scaling and clashes across projects. The default is `<project>-<service>-<n>`. |
| `hostname: web` | Container's internal hostname. |
| `working_dir: /app` | Override the image's `WORKDIR`. |
| `user: "10001:10001"` | Run as this UID:GID (overrides image `USER`). |
| `init: true` | Run a small init process as PID 1 that forwards signals and reaps zombies ([Chapter 4](04-dockerfile-deep-dive.md#16-pid-1-signals-tini-and---init)). |
| `stop_signal: SIGINT` | Signal sent by `docker compose stop`/`down` (default `SIGTERM`). |
| `stop_grace_period: 30s` | Wait before `SIGKILL` (default 10 s). |
| `read_only: true` | Read-only root filesystem (pair with `tmpfs: [/tmp]`). |
| `tmpfs: [/tmp]` | RAM-backed scratch directories. |
| `cap_drop: [ALL]` | Drop Linux capabilities the container does not need ([Chapter 10](10-security.md)). |
| `security_opt: ["no-new-privileges:true"]` | Block privilege escalation in the container. |
| `extra_hosts: ["host.docker.internal:host-gateway"]` | Add `/etc/hosts` entries (needed on Linux to reach the host, [Chapter 5](05-storage-and-networking.md#b5-container-to-host-hostdockerinternal)). |
| `dns: [10.0.0.2]` | Custom DNS servers. |
| `labels` | Metadata (also used by Traefik and others for routing). |
| `logging` | Log driver and options, for example `driver: json-file`, `options: {max-size: "10m", max-file: "3"}`. Without a size limit, container logs grow without bound. |
| `platform: linux/amd64` | Force a platform (emulated on mismatching hardware). |
| `pull_policy` | `always`, `never`, `missing` (default), `build`, `daily`, `weekly`, `every_<duration>` such as `every_12h`. |
| `ulimits`, `sysctls`, `shm_size` | Low-level limits and kernel parameters (`shm_size` matters for some ML and browser workloads). |
| `gpus: all` | Request GPU access ([Chapter 8](08-docker-for-ai-and-ml.md)). |
| `scale: 3` / `deploy.replicas: 3` | Number of containers for the service. |

---

## Environment variables: interpolation and precedence

Two separate mechanisms involve environment variables:

1. **Interpolation**: Compose substitutes `${VARIABLES}` inside the YAML file on your machine, before anything is sent to Docker. Values come from the shell and the `.env` file.
2. **Container environment**: variables set inside the running container through `environment:` and `env_file:`.

### 1. Interpolation (`${VAR}` in the YAML)

```yaml
services:
  db:
    image: postgres:${PG_VERSION:-18}
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
```

| Syntax | Meaning |
|---|---|
| `${VAR}` or `$VAR` | Value of `VAR` (empty string plus a warning if unset) |
| `${VAR:-default}` | `default` if `VAR` is unset or empty |
| `${VAR-default}` | `default` only if `VAR` is unset (empty stays empty) |
| `${VAR:?error message}` | Fail with the message if `VAR` is unset or empty |
| `${VAR?error message}` | Fail only if unset |
| `${VAR:+replacement}` | `replacement` if `VAR` is set and non-empty, else empty |
| `$$` | Literal `$` (needed in healthcheck commands and shell snippets) |

Interpolation value sources, highest priority first (per Docker's documentation):

1. Variables in the shell environment (`export PG_VERSION=17` / `$env:PG_VERSION="17"`)
2. The `.env` file (and files given with `--env-file`)
3. Defaults in the YAML (`${VAR:-default}`)

The `.env` file is read from the project directory (where `compose.yaml` lives):

```
# .env
POSTGRES_DB=notes
POSTGRES_USER=notes
POSTGRES_PASSWORD=dev-only-change-me
PG_VERSION=18
```

- `docker compose --env-file ./config/staging.env up` uses an alternative file instead of `.env`.
- Unverified: the exact ranking when `--env-file` and shell variables conflict. The documented rule is "shell beats `.env`". Test with `docker compose config` if it matters.
- `.env` is for interpolation only. Its variables are not passed into containers automatically. Reference them in `environment:` (`DB_USER: ${POSTGRES_USER}`) or use `env_file:`.
- Do not commit `.env` with real secrets. Commit `.env.example` with placeholders and add `.env` to `.gitignore`.
- `COMPOSE_PROJECT_NAME`, `COMPOSE_FILE`, `COMPOSE_PROFILES` and a few others configure Compose itself and can be set in `.env` or the shell.

### 2. Variables inside the container

When a variable is defined in several places, the winner is (highest first, per Docker's documentation):

1. `docker compose run -e VAR=value`
2. `environment:` in the Compose file (after interpolation)
3. `env_file:` files in the Compose file (later files override earlier ones)
4. `ENV` in the image's Dockerfile

### See what Compose resolved

```bash
docker compose config
```

Prints the fully merged and interpolated configuration, validates syntax and shows warnings.

```bash
docker compose config --services        # service names
docker compose config --volumes         # volume names
docker compose config --quiet           # validate only (exit code 0 = valid; for CI)
docker compose config --format json     # machine-readable
docker compose -f compose.yaml -f compose.prod.yaml config    # merged result of several files
```

`docker compose config` prints resolved values, including secrets that came from `.env` or the environment. Do not paste its output into chat or tickets unchecked.

---

## Volumes

Compose volumes use the concepts from [Chapter 5](05-storage-and-networking.md): named volumes, bind mounts, tmpfs.

### Declaring named volumes

```yaml
services:
  db:
    image: postgres:18
    volumes:
      - pgdata:/var/lib/postgresql     # SOURCE:TARGET, source = volume name

volumes:
  pgdata:                              # declare at top level
```

- Compose creates `pgdata` on first `up` as `<project>_pgdata`.
- `docker compose down` removes containers and networks but keeps named volumes. `docker compose down -v` deletes them.
- An empty named volume is populated from the image on first mount.

### Short vs long syntax

```yaml
    volumes:
      # short syntax
      - pgdata:/var/lib/postgresql
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      # long syntax (like docker's --mount)
      - type: volume
        source: pgdata
        target: /var/lib/postgresql
      - type: bind
        source: ./src
        target: /app/src
        read_only: true
      - type: tmpfs
        target: /scratch
        tmpfs:
          size: 67108864          # bytes (64 MiB)
```

Relative bind paths (`./src`) are relative to the Compose file's directory. For bind mounts, `bind: {create_host_path: false}` in the long syntax makes Compose error instead of creating a missing host directory.

### Volume options

```yaml
volumes:
  pgdata:
    name: notes_pgdata               # exact name, no project prefix
  shared_models:
    external: true                   # already exists; Compose must not create or delete it
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.50,rw,nfsvers=4
      device: ":/exports/shared"
```

- `external: true`: managed outside the project. Compose errors if it does not exist and never deletes it with `down -v`. Create it with `docker volume create shared_models`. Use for data that must not be lost or that is shared across projects (for example a model cache).
- `name:`: pins the actual volume name, so a project rename does not produce a new empty volume.
- `driver`/`driver_opts`: volume drivers ([Chapter 5](05-storage-and-networking.md#a6-volume-drivers)).

Common problems: wrong mount path for the image version (Postgres 18 change, [Chapter 5](05-storage-and-networking.md#a2-volumes)); bind-mounting `.` over `/app` hides installed dependencies; renaming the folder or project creates new empty volumes (pin `name:`); `down -v` deletes data.

---

## Networks

By default Compose creates one network per project named `<project>_default` (a user-defined bridge). Every service joins it and is reachable by service name on its container port.

```
 project "notes-app"  ->  network notes-app_default
   web     reaches   db      as  db:5432
   web     reaches   redis   as  redis:6379
```

Use `DB_HOST: db`, never `localhost` ([Chapter 5, section B4](05-storage-and-networking.md#b4-localhost-inside-containers)).

### Custom networks (isolation tiers)

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    networks: [frontend]
    ports: ["80:80"]
  api:
    build: ./api
    networks: [frontend, backend]      # bridges both tiers
  db:
    image: postgres:18
    networks: [backend]                # not reachable from the proxy

networks:
  frontend:
  backend:
    internal: true                     # no outbound internet, no published ports
```

- A service sees only services that share a network with it. Here `proxy` cannot reach `db`.
- `internal: true`: containers on the network cannot reach the outside, and published ports do not work for it. Suited to production databases; do not use it in dev if the DB port must be published to the laptop.
- Once a service defines `networks`, it no longer joins `default` unless `default` is listed.

### Aliases, static IPs, external networks

```yaml
services:
  api:
    networks:
      backend:
        aliases:
          - api.internal            # extra DNS name on this network
        ipv4_address: 172.28.0.10   # static IP (requires the subnet to be defined)
networks:
  backend:
    ipam:
      config:
        - subnet: 172.28.0.0/24
  shared:
    external: true                  # created outside Compose: docker network create shared
    name: shared                    # actual name
```

- Cross-project communication: run `docker network create shared`, mark it `external: true` in both projects and attach the relevant services. They reach each other by service name.
- Avoid static IPs; use names.

### Network mode

```yaml
    network_mode: host           # use the host's network stack (Linux)
    network_mode: "service:vpn"  # share another service's network namespace (sidecar)
```

`network_mode` and `networks` cannot be combined.

---

## depends_on and startup order

`depends_on` sets the order in which services start (and stop, in reverse).

```yaml
services:
  web:
    depends_on:
      - db
      - redis
```

The short form waits only until the dependencies' containers have started, not until the software inside is ready. Postgres needs several seconds after container start to accept connections, so an app connecting immediately fails with `connection refused`.

### Conditions with healthchecks

```yaml
services:
  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully

  db:
    image: postgres:18
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
```

| Condition | Compose waits until the dependency... |
|---|---|
| `service_started` | has started (same as the short syntax) |
| `service_healthy` | passes its `healthcheck` (the dependency must define one) |
| `service_completed_successfully` | has run to completion with exit code 0 (one-shot jobs such as migrations) |

Additional long-syntax fields:

- `restart: true`: restart this service when Compose updates or recreates the dependency (Compose 2.17+).
- `required: false`: only warn if the dependency is unavailable, for example in an inactive [profile](#profiles) (Compose 2.20+).

Limits:

1. `depends_on` applies only at startup. If the database crashes later, Compose does not restart `web`. Use `restart:` policies and make the app retry connections (with backoff).
2. It does not replace application-level resilience.
3. `wait-for-it.sh` scripts from old tutorials are unnecessary with `service_healthy`.

---

## Healthchecks

A Compose healthcheck is the same concept as the Dockerfile `HEALTHCHECK` ([Chapter 4](04-dockerfile-deep-dive.md#511-healthcheck-tell-docker-how-to-test-the-app)), configured per service. It can be added to images you did not build (Postgres, Redis) and overrides the one in your image.

```yaml
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s          # time between checks
      timeout: 3s            # a slower check counts as failed
      retries: 3             # consecutive failures before "unhealthy"
      start_period: 20s      # grace period: failures here do not count
      start_interval: 2s     # check frequency during the start period (Engine 25+)
```

| `test` form | Meaning |
|---|---|
| `["CMD", "pg_isready", "-U", "postgres"]` | Run the program directly (no shell) |
| `["CMD-SHELL", "pg_isready -U postgres \|\| exit 1"]` | Run through the container's shell (for `$VAR`, pipes, `\|\|`) |
| `"pg_isready -U postgres"` (string) | Same as `CMD-SHELL` |
| `["NONE"]` or `disable: true` | Disable a healthcheck baked into the image |

**Escaping `$`:** Compose interpolates `${VAR}` before the container exists. To have the container's shell expand a variable, write `$$POSTGRES_USER` so Compose passes a literal `$POSTGRES_USER`.

| Service | Healthcheck `test` |
|---|---|
| PostgreSQL | `["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]` |
| Redis | `["CMD", "redis-cli", "ping"]` (expects `PONG`) |
| MySQL/MariaDB | `["CMD", "mysqladmin", "ping", "-h", "localhost"]` |
| HTTP service with curl | `["CMD", "curl", "-fsS", "http://localhost:8000/health"]` (`-f` fail on HTTP errors, `-sS` quiet but show errors) |
| HTTP service on slim Python (no curl) | `["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]` |
| nginx (alpine) | `["CMD", "wget", "-qO-", "http://127.0.0.1/"]` (BusyBox wget is present on Alpine) |

View status with `docker compose ps` (the STATUS column shows `(healthy)`).

A failing healthcheck does not restart the container. It changes the displayed status and gates `depends_on: service_healthy`.

---

## Restart policies

```yaml
    restart: unless-stopped
```

| Value | Behaviour |
|---|---|
| `no` (default) | Never restart automatically |
| `always` | Always restart, including at daemon boot, even if stopped manually |
| `unless-stopped` | Like `always`, but a container stopped explicitly (`docker compose stop`) stays stopped across daemon restarts |
| `on-failure[:max-retries]` | Restart only on a non-zero exit code, optionally limited to N attempts (`on-failure:5`) |

Guidance:

- Long-running production services: `unless-stopped`.
- One-shot jobs (migrations): `no` or `on-failure:3`.
- Development: usually `no`, so crashes are visible.
- For restarts to survive a reboot, the Docker service must start at boot (Linux: `sudo systemctl enable docker`; Docker Desktop: "Start Docker Desktop when you log in").
- A crash-looping container restarts with increasing delay; check `docker compose logs`.
- Restart policies react to the process exiting, not to a failed healthcheck.

---

## command and entrypoint

```yaml
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
    entrypoint: ["/entrypoint.sh"]
```

- `command` overrides the image's `CMD`; `entrypoint` overrides `ENTRYPOINT`. The exec-form rules apply ([Chapter 4](04-dockerfile-deep-dive.md#6-cmd-vs-entrypoint-exec-form-vs-shell-form)): use list syntax so the program is PID 1 and receives signals.
- Overriding `entrypoint` also clears the image's `CMD` unless `command` is set.
- To use a variable in the container's shell, call a shell and escape with `$$`: `command: ["sh", "-c", "exec uvicorn app.main:app --port $${PORT:-8000}"]`.
- Multi-line scripts use a YAML block scalar (`|` starts a multi-line literal block):

  ```yaml
  command:
    - sh
    - -c
    - |
      echo "starting"
      exec uvicorn app.main:app --host 0.0.0.0
  ```

---

## Resource limits

Without limits, one container can consume all CPU and RAM on the host (or the whole VM on Docker Desktop).

```yaml
    deploy:
      resources:
        limits:
          cpus: "1.0"          # at most 1 CPU core's worth of time
          memory: 512M         # hard memory cap
        reservations:
          cpus: "0.25"         # guaranteed minimum (scheduling hint)
          memory: 128M
```

- `limits` are enforced through Linux cgroups ([Chapter 1](01-what-is-docker.md)). Exceeding the memory limit gets the process OOM-killed (exit code 137, which is 128 + signal 9); `docker inspect` shows `OOMKilled: true`.
- `cpus` throttles instead of killing.
- `reservations` are minimums for scheduling; on a single host they are mostly advisory.
- `deploy` belongs to the **Deploy Specification**, originally for Swarm. Compose V2 applies `deploy.resources` limits in regular `docker compose up`.

Legacy short forms, still valid:

```yaml
    mem_limit: 512m
    mem_reservation: 128m
    cpus: 1.0
    pids_limit: 200          # cap the number of processes (fork-bomb protection)
```

If both a short form and `deploy` are set, the values must match. Prefer `deploy.resources`.

GPUs (full treatment in [Chapter 8](08-docker-for-ai-and-ml.md)):

```yaml
    gpus: all                               # short form
    # long form:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

Requires an NVIDIA GPU, host drivers and the NVIDIA Container Toolkit. Check live usage with `docker stats` or `docker compose stats`.

---

## Profiles

**Profiles** keep optional services (debug tools, admin UIs, test runners, seed jobs) in the same file without starting them every time.

```yaml
services:
  web:
    build: .                    # no profiles: always started

  adminer:                      # web UI for databases
    image: adminer
    ports: ["8081:8080"]
    profiles: [debug]           # starts only if the "debug" profile is active

  seed:
    build: .
    command: ["python", "-m", "app.seed"]
    profiles: [tools]
```

```bash
docker compose up -d                          # web only
docker compose --profile debug up -d          # web + adminer
docker compose --profile debug --profile tools up -d
COMPOSE_PROFILES=debug,tools docker compose up -d    # same, via environment variable
docker compose run --rm seed                  # naming a service activates its profile
```

- A service without `profiles` always starts; a service with `profiles` starts only when one of them is active.
- Naming a profiled service on the command line (`run seed`, `up adminer`) enables its profile automatically.
- An always-on service that `depends_on` a profiled service fails unless that profile is active or the dependency is marked `required: false`.
- `docker compose --profile "*" up` activates all profiles.

---

## Multiple Compose files and overrides

Keep a base file and layer override files for each environment (dev, test, prod).

### Automatic override: `compose.override.yaml`

If `compose.yaml` and `compose.override.yaml` both exist, plain `docker compose up` merges them (override on top). Convention: the base holds shared settings, the override holds developer conveniences (bind mounts, published ports, debug flags).

### Explicit files with `-f`

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

- `-f` (`--file`) is repeatable; files merge left to right and later files win.
- When `-f` is given, `compose.override.yaml` is not loaded automatically. This is how to run "base + prod" without the dev override.
- Set a default with `COMPOSE_FILE=compose.yaml:compose.prod.yaml` (separator `:` on Linux/macOS, `;` on Windows; override with `COMPOSE_PATH_SEPARATOR`).

### How merging works

| Kind of field | Examples | Merge behaviour |
|---|---|---|
| Scalars | `image`, `restart`, `user` | Later file replaces |
| Mappings | `environment`, `labels`, `build.args` | Merged key by key; later wins on the same key |
| Lists of items | `ports`, `expose`, `dns`, `tmpfs` | Appended |
| `volumes` | | Merged by mount target path; same target in a later file replaces the earlier entry |
| `command`, `entrypoint`, `healthcheck.test` | | Replaced wholesale |

Two YAML tags (Compose 2.24+) override the default:

```yaml
services:
  web:
    ports: !reset []                  # clear ports inherited from the earlier file
    environment: !override            # replace the entire mapping instead of merging
      ONLY_THIS: "yes"
```

- `!reset`: remove the inherited value (`[]` for lists, `null` for scalars).
- `!override`: replace instead of merge.

Verify with `docker compose -f a.yaml -f b.yaml config`.

### Typical layout

```
compose.yaml            # shared base: services, healthchecks, volumes, networks
compose.override.yaml   # dev: auto-loaded; ports, watch, --reload, dev passwords
compose.prod.yaml       # prod: restart policies, limits, secrets, no dev conveniences
compose.test.yaml       # CI: test runner service, ephemeral DB
.env.example            # documented fake values
```

A full example is in the [tutorial](#stage-4-dev-vs-prod-with-override-files).

---

## extends and include

### `extends`: inherit one service definition

```yaml
# common.yaml
services:
  python-service:
    image: python:3.13-slim
    restart: unless-stopped
    environment:
      LOG_LEVEL: info
```

```yaml
# compose.yaml
services:
  api:
    extends:
      file: common.yaml
      service: python-service
    command: ["python", "-m", "api"]
    environment:
      LOG_LEVEL: debug         # overrides the inherited value
```

Per the Compose Specification:

- `service` is required; `file` is optional (omit to extend a service in the same file).
- Merging follows the override rules: mappings merge key by key, sequences combine with inherited items first, scalars from the extending service win.
- Circular references are an error.
- Referenced resources (networks, volumes) are not imported; declare them in the extending file.
- `extends` is not supported by `docker stack deploy` (Swarm).

### `include`: import whole Compose files

```yaml
# compose.yaml
include:
  - ./infra/compose.yaml          # for example a shared database + cache stack
  - path:
      - ./third-party/compose.yaml
      - ./third-party.override.yaml   # per-include override file

services:
  web:
    build: .
    depends_on: [db]              # "db" comes from the included file
```

- Each included file loads as its own Compose model with its own project directory, so relative paths inside it (build contexts, bind mounts, `env_file`) resolve relative to that file. This is the main advantage over `-f` merging and `extends`.
- Included resources are copied into your model, so you can `depends_on` them.
- `include` is recursive.
- Conflicts are errors: a resource defined both in the included file and in yours makes Compose fail. To customize an included service, list an override file under that `include` entry or use `compose.override.yaml`.
- Unverified: Docker's documentation also describes remote sources (Git repositories and OCI artifacts) for `include`; the exact URL syntax is not confirmed here. Check the `include` reference first.

Use `include` when one team owns a shared stack and others include it. Use override files for the same app in different environments.

---

## YAML anchors and extension fields

YAML **anchors** (`&name` defines a block, `*name` references it, `<<:` merges it) combined with Compose **extension fields** (top-level keys starting with `x-`, ignored by Compose) remove copy-paste between similar services.

```yaml
x-python-service: &python-service
  build: .
  restart: unless-stopped
  environment: &python-env
    DB_HOST: db
    REDIS_URL: redis://redis:6379/0
  depends_on:
    db:
      condition: service_healthy

services:
  web:
    <<: *python-service                  # merge all keys from the anchor
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0"]
    ports: ["8000:8000"]

  worker:
    <<: *python-service
    command: ["python", "-m", "app.worker"]
    environment:
      <<: *python-env                    # merge the shared env...
      WORKER_CONCURRENCY: "4"            # ...and add one more
```

- `&python-service` names the block, `*python-service` pastes it, `<<:` copies the anchor's keys into the current mapping (explicit keys win).
- Anchors work within one file only, not across `-f` files. Use `extends`, `include` or overrides across files.
- `docker compose config` shows the expanded result.

---

## Secrets and configs

### Why not environment variables

Environment variables are visible through `docker inspect`, often leak into logs and crash reports, and are inherited by child processes. For sensitive values, deliver them as files mounted read-only (conventionally under `/run/secrets/`) that the app reads at startup.

### Secrets

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password   # official image reads the password from this file
    secrets:
      - db_password

  api:
    build: .
    environment:
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt        # source on the host
```

- Top-level `secrets:` declares each secret's source: `file:` (a path) or `environment:` (name of a shell/`.env` variable).
- Service-level `secrets:` grants access. In the short form, the secret appears as a read-only file at `/run/secrets/<name>`.
- The official Postgres, MySQL and MariaDB images support the `*_FILE` convention (`POSTGRES_PASSWORD_FILE`, ...). Many others do too; check their documentation.
- Add `secrets/` to `.gitignore`. Generate a value with `openssl rand -base64 24 > secrets/db_password.txt` (Git Bash/WSL/macOS/Linux), or in PowerShell `[Convert]::ToBase64String((1..24 | ForEach-Object { Get-Random -Maximum 256 })) | Set-Content -NoNewline secrets/db_password.txt`.
- Long syntax allows `source`, `target`, `uid`, `gid`, `mode` (default `0444`). Per the specification, `uid`, `gid` and `mode` apply only to secrets sourced from `environment`; for `file:` sources they are ignored, so the file's host permissions must let the container user read it.
- Limitation: Compose secrets are a delivery mechanism, not a vault. The source file is plaintext on the host, and anyone who can run `docker exec` can read it. In production use a secrets manager (Vault, cloud secret services) or Swarm/Kubernetes secrets ([Chapter 10](10-security.md), [Chapter 11](11-production-and-enterprise.md)).

### Build secrets

The same `secrets:` definition can feed image builds (`RUN --mount=type=secret`, [Chapter 4](04-dockerfile-deep-dive.md#17-build-secrets-never-bake-secrets-into-layers)):

```yaml
services:
  api:
    build:
      context: .
      secrets:
        - pip_token
secrets:
  pip_token:
    environment: PIP_TOKEN        # read from the shell variable at build time
```

### Configs

**Configs** deliver non-sensitive configuration files (nginx config, app settings), mounted read-only:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf

configs:
  nginx_conf:
    file: ./nginx.conf
```

Compared with a bind mount, configs express intent, work consistently on Swarm/Kubernetes-style platforms and do not depend on the host directory layout. Unverified: recent Compose versions also allow `content:` (inline) or `environment:` as a config source; verify with `docker compose config`.

---

## docker compose watch: hot reload for development

Rebuilding the image on every change is slow. Bind-mounting source is fast but has the cross-OS issues in [Chapter 5](05-storage-and-networking.md#a8-windows-and-wsl-path-pitfalls). **Compose Watch** watches files on the host and pushes changes into the running container (or rebuilds), based on rules under `develop.watch`.

```yaml
services:
  web:
    build: .
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
    ports: ["8000:8000"]
    develop:
      watch:
        - action: sync
          path: ./app
          target: /app/app
          ignore:
            - __pycache__/
        - action: sync+restart
          path: ./config/settings.toml
          target: /app/config/settings.toml
        - action: rebuild
          path: requirements.txt
```

| Action | Effect when a watched file changes | Use for |
|---|---|---|
| `sync` | Copy the changed file into the container at `target` | Source code with a framework that reloads itself (uvicorn `--reload`, Vite, Next.js dev) |
| `rebuild` | Rebuild the image and replace the container (like `up --build <service>`) | Dependency manifests (`requirements.txt`, `package.json`), compiled languages |
| `sync+restart` | Sync the file and restart the container | Config files read only at startup (`nginx.conf`) |

Unverified: newer Compose versions also describe a `sync+exec` action that runs a command after syncing. Check the current "Use Compose Watch" docs.

Attributes (from Docker's documentation):

- `action`: required.
- `path`: required; file or directory to watch, relative to the project directory. Directories are watched recursively. Globs are not supported in `path`.
- `target`: container path for `sync` actions.
- `ignore`: extra patterns to skip, relative to `path`. `.dockerignore` rules also apply.
- `initial_sync`: for `sync+...` actions, ensures files are up to date when the watch session starts.

```bash
docker compose watch              # start services and watch files; logs and sync events are separated
docker compose up --watch         # same, with the regular `up` log output
docker compose watch --prune=false   # keep superseded images after rebuilds (pruned by default)
```

Requirements:

- The image needs the `stat`, `mkdir` and `rmdir` binaries (distroless/scratch images lack them).
- The container user must be able to write to the target path. With a non-root `USER`, copy the app with `COPY --chown=app:app` ([Chapter 4](04-dockerfile-deep-dive.md#15-running-as-a-non-root-user)). Permission errors are the most common reason `watch` syncs nothing.
- For `sync`, the process must reload itself: `uvicorn --reload` (Python), `nodemon`/Vite/Next dev server (Node), `air` (Go), Spring DevTools (Java).
- `watch` is for development only.

---

## Project names

Compose prefixes resource names with the project name:

| Resource | Name pattern | Example |
|---|---|---|
| Container | `<project>-<service>-<index>` | `notes-app-web-1` |
| Network | `<project>_<network>` | `notes-app_default` |
| Volume | `<project>_<volume>` | `notes-app_pgdata` |

Project name resolution, highest priority first (per Docker's documentation):

1. `-p` / `--project-name` flag
2. `COMPOSE_PROJECT_NAME` environment variable
3. `name:` top-level key
4. Name of the directory containing the Compose file

Names are lowercased and limited to letters, digits, `-` and `_`.

- Two copies of the same app on one host need different project names: `docker compose -p myapp-test up -d`.
- Renaming the folder changes the project name, so Compose creates new empty volumes. The old data stays under the old name. Set `name:` at the top of the file.
- `docker compose ls` lists all projects on the machine.

---

## Compose commands reference

Commands operate on the project defined by the Compose file(s) in the current directory (or those given with `-f`/`-p`). Global flags go before the subcommand: `docker compose -f x.yaml -p demo up -d`.

### Global options

| Option | Meaning |
|---|---|
| `-f, --file <path>` | Compose file(s); repeatable |
| `-p, --project-name <name>` | Project name |
| `--env-file <path>` | Alternate env file for interpolation (repeatable) |
| `--profile <name>` | Activate a profile (repeatable) |
| `--project-directory <dir>` | Alternate working directory (defaults to the first Compose file's directory) |
| `--dry-run` | Show what would be done without doing it |
| `--progress <mode>` | `auto`, `tty`, `plain`, `json`, `quiet` |
| `--ansi <never\|always\|auto>` | Control colour/ANSI codes |
| `--parallel <n>` | Max parallelism (-1 = unlimited) |
| `--all-resources` | Include resources not used by any service |
| `--compatibility` | Backward-compatibility mode for V1 conventions (container name separators) |

`docker compose --dry-run up -d` prints the plan and changes nothing.

### Lifecycle commands

#### `docker compose up`

Creates networks and volumes, then creates (or recreates if configuration or image changed) and starts services.

```bash
docker compose up                # foreground: streams all logs; Ctrl+C stops the containers
docker compose up -d             # detached
docker compose up -d --build     # rebuild images first
docker compose up -d web         # only 'web' (and its dependencies)
```

Flags (from `docker compose up --help` on Compose 5.6.1):

| Flag | Meaning |
|---|---|
| `-d, --detach` | Run in the background |
| `--build` | Build images before starting |
| `--no-build` | Never build |
| `--pull always\|missing\|never` | Pull images before running |
| `--force-recreate` | Recreate containers even if nothing changed |
| `--no-recreate` | Do not recreate existing containers |
| `--no-deps` | Do not start dependent services |
| `--wait` | Wait for services to be running/healthy (implies `-d`); useful in CI and scripts |
| `--wait-timeout <s>` | Give up waiting after this many seconds |
| `--scale web=3` | Run N containers of a service |
| `--remove-orphans` | Remove containers for services no longer in the file |
| `-V, --renew-anon-volumes` | Recreate anonymous volumes instead of reusing old ones |
| `--abort-on-container-exit` | Stop everything when any container stops (not with `-d`) |
| `--abort-on-container-failure` | Stop everything if any container exits with an error |
| `--exit-code-from <service>` | Return that service's exit code (implies abort-on-exit); for CI test runs |
| `-w, --watch` | Start and watch files ([watch](#docker-compose-watch-hot-reload-for-development)) |
| `--no-color`, `--no-log-prefix`, `--timestamps` | Log formatting |
| `-t, --timeout <s>` | Shutdown timeout when stopping/recreating |
| `-y, --yes` | Auto-answer prompts |

Steps: read and merge files, interpolate variables, create networks and volumes, build or pull missing images, create and start containers in dependency order while waiting for `depends_on` conditions. Unchanged services are left running; services whose configuration or image changed are recreated.

#### `docker compose down`

Stops and removes the project's containers and networks.

```bash
docker compose down              # remove containers + networks; keep named volumes
docker compose down -v           # also delete named and anonymous volumes: DATA LOSS
docker compose down --rmi local  # also remove images built by this project without a custom tag
docker compose down --remove-orphans
```

| Flag | Meaning |
|---|---|
| `-v, --volumes` | Remove named volumes declared in the file and anonymous volumes attached to containers |
| `--rmi local\|all` | Remove images: `local` = without a custom tag; `all` = every image used by services |
| `--remove-orphans` | Remove containers for services not in the file |
| `-t, --timeout` | Seconds to wait for containers to stop |

> **Warning:** `docker compose down -v` permanently deletes database volumes. Plain `down` keeps data. Use `stop` to pause without removing containers.

#### `start` / `stop` / `restart` / `pause` / `unpause` / `kill` / `rm`

```bash
docker compose stop              # SIGTERM, then SIGKILL after the grace period; containers are kept
docker compose start             # start stopped containers
docker compose restart           # stop + start
docker compose restart web       # one service
docker compose pause web         # freeze processes (cgroup freeze), keeps memory
docker compose unpause web
docker compose kill web          # immediate SIGKILL (-s SIGNAL for another signal)
docker compose rm                # remove stopped containers (-f no prompt, -s stop first, -v remove anonymous volumes)
```

`restart` does not re-read the Compose file or `.env`. After changing configuration run `docker compose up -d`.

### Inspection commands

```bash
docker compose ps                # containers of this project
docker compose ps -a             # include stopped containers
docker compose ps --services     # service names only
docker compose ps --format json
docker compose ps --status running --status exited
```

```bash
docker compose logs              # all services, all history
docker compose logs -f           # follow
docker compose logs -f --tail 100 web     # last 100 lines, then follow, one service
docker compose logs -t web       # with timestamps
docker compose logs --since 10m  # last ten minutes
docker compose logs --no-log-prefix web   # without the "web-1  |" prefix
```

Each log line is prefixed with the container name and coloured per service.

```bash
docker compose top               # processes in each container
docker compose images            # images used by the project's containers
docker compose port web 8000     # host address mapped to web's container port 8000
docker compose ls                # list all Compose projects (-a includes stopped)
docker compose events            # real-time event stream
docker compose volumes           # volumes used by the project
docker compose stats             # live CPU/memory/network per container
```

`docker compose config` renders the merged and interpolated model (see above). Additional flags: `--services`, `--profiles`, `--images`, `--quiet`, `--format json`, `--no-interpolate` (show `${VARS}` unexpanded).

### Running commands

#### `docker compose exec`

Runs a command inside an already-running container of a service.

```bash
docker compose exec web sh                    # interactive shell
docker compose exec db psql -U notes -d notes   # Postgres prompt
docker compose exec -u root web id            # as root
docker compose exec -T web python -c "print(1)"   # no TTY (CI/scripts, piping)
docker compose exec -e FOO=bar web env
docker compose exec --index 2 web sh          # container #2 if scaled
```

Flags: `-d` detached, `-e` env var, `-u` user, `-w` working dir, `-T` disable pseudo-TTY, `--index` pick a replica. In pipelines and CI, `-T` avoids "the input device is not a TTY".

#### `docker compose run`

Starts a new one-off container from a service definition and runs a command in it (tests, migrations, a shell). It also starts the service's dependencies.

```bash
docker compose run --rm web pytest -q          # run tests, delete the container afterwards
docker compose run --rm web python -m app.migrate
docker compose run --rm --no-deps web sh       # do not start db/redis
docker compose run --rm -e DEBUG=1 web sh
docker compose run --service-ports web         # publish the service's ports (off by default for run)
```

`--rm` removes the one-off container when done (otherwise they accumulate as `project-web-run-abc123`). `run` does not publish ports by default; add `--service-ports` if needed.

`exec` enters a running container. `run` launches a new container for a task.

### Build and image commands

```bash
docker compose build                 # build all services with `build:`
docker compose build web             # one service
docker compose build --no-cache      # ignore cache
docker compose build --pull          # re-pull the base image
docker compose build --build-arg PYTHON_VERSION=3.12 web
docker compose pull                  # pull images for services using `image:`
docker compose push                  # push built images to their registry
```

Also `docker compose publish` (publishes the Compose application as an OCI artifact; exists in recent releases; unverified, see `docker compose publish --help`).

### Copying files

```bash
docker compose cp web:/app/output.csv ./output.csv         # container -> host
docker compose cp ./settings.toml web:/app/settings.toml   # host -> container
```

The container side uses `SERVICE:PATH`.

### Other commands (from `docker compose --help` on 5.6.1)

| Command | Purpose |
|---|---|
| `attach` | Attach the terminal to a running container's stdin/stdout |
| `commit` | Create an image from a service container's changes (prefer Dockerfiles) |
| `create` | Create containers without starting them |
| `export` | Export a service container's filesystem as a tarball |
| `scale` | Scale services (`docker compose scale web=3`) |
| `wait` | Block until the selected services stop |
| `bridge` | Convert Compose files to another model |
| `version` | Print the Compose version |

---

## Progressive tutorial

The tutorial builds from one container to a stack with dev and prod configurations. Create a folder `notes-app/`. All passwords are fake dev-only values.

```
notes-app/
  app/
    __init__.py       (empty file)
    main.py
  requirements.txt
  Dockerfile
  .dockerignore
  .env  /  .env.example
  compose.yaml
  compose.override.yaml
  compose.prod.yaml
  secrets/db_password.txt     (prod only; never committed)
```

### Stage 1: a single service

**`requirements.txt`** (pin versions in real projects, [Chapter 4](04-dockerfile-deep-dive.md#13-reproducibility-tags-digests-pinning)):

```
fastapi
uvicorn[standard]
```

**`app/main.py`**

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def root():
    return {"message": "Hello from Compose"}

@app.get("/health")
def health():
    return {"status": "ok"}
```

**`Dockerfile`** (explained in [Chapter 4](04-dockerfile-deep-dive.md#10-ordering-instructions-for-cache-efficiency-before-and-after)):

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim
ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
COPY app/ ./app/
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`compose.yaml`**

```yaml
name: notes-app

services:
  web:
    build: .
    ports:
      - "8000:8000"
```

```bash
docker compose up --build
```

`up` creates and starts everything; `--build` builds the image first. Logs stream in the terminal; stop with Ctrl+C.

Compose creates a network (`notes-app_default`) and a container (`notes-app-web-1`). Test with `curl http://localhost:8000` (returns `{"message":"Hello from Compose"}`) and `docker compose ps`.

Stop with Ctrl+C, then `docker compose down`.

### Stage 2: web + Redis (a hit counter)

`requirements.txt`:

```
fastapi
uvicorn[standard]
redis
```

**`app/main.py`**

```python
import os

import redis
from fastapi import FastAPI

app = FastAPI()
cache = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)


@app.get("/")
def root():
    hits = cache.incr("hits")          # atomic increment stored in Redis
    return {"message": "Hello from Compose", "hits": hits}


@app.get("/health")
def health():
    cache.ping()                       # raises if Redis is unreachable
    return {"status": "ok"}
```

**`compose.yaml`**

```yaml
name: notes-app

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      REDIS_URL: redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
```

| Line | Meaning |
|---|---|
| `REDIS_URL: redis://redis:6379/0` | Hostname is `redis`, the service name, resolved by Docker DNS ([Chapter 5](05-storage-and-networking.md#b2-default-bridge-vs-user-defined-bridge)). `6379` is Redis's container port; `/0` selects database 0. |
| `depends_on: redis: condition: service_healthy` | Start `web` only after Redis's healthcheck passes. |
| `image: redis:7-alpine` | Official image; no `build`. No `ports:` because only `web` needs Redis. |
| `healthcheck.test: ["CMD","redis-cli","ping"]` | Redis answers `PONG` when ready. Checked every 5 s, 3 s timeout, up to 5 failures. |

```bash
docker compose up -d --build
curl http://localhost:8000      # {"message":"Hello from Compose","hits":1}
curl http://localhost:8000      # ... "hits":2
docker compose restart web
curl http://localhost:8000      # ... "hits":3   (count lives in Redis, not in web)
docker compose down
docker compose up -d
curl http://localhost:8000      # "hits":1 again: Redis data lived in the removed container
```

The last step shows [Chapter 5](05-storage-and-networking.md#a1-why-data-disappears-the-containers-writable-layer): without a volume, data is lost with the container. Stage 3 adds volumes.

### Stage 3: web + Postgres + Redis, healthchecks, volumes

```
        host :8000
            |
       +----+-----+      +-----------+
       |   web    |----->|   redis   |  volume: redisdata
       | FastAPI  |      +-----------+
       |          |      +-----------+
       |          |----->|    db     |  volume: pgdata
       +----------+      | Postgres  |
                         +-----------+
     (all on notes-app_default; only web is published)
```

`requirements.txt`:

```
fastapi
uvicorn[standard]
redis
psycopg[binary]
```

**`app/main.py`** (final version used for the rest of the chapter):

```python
import os
from contextlib import asynccontextmanager
from pathlib import Path

import psycopg
import redis
from fastapi import FastAPI


def read_secret(name: str) -> str:
    """Prefer NAME_FILE (a path to a secret file) over NAME (a plain variable)."""
    path = os.environ.get(f"{name}_FILE")
    if path:
        return Path(path).read_text().strip()
    return os.environ[name]


def connect():
    return psycopg.connect(
        host=os.environ["DB_HOST"],
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=read_secret("DB_PASSWORD"),
    )


cache = redis.Redis.from_url(os.environ["REDIS_URL"], decode_responses=True)


@asynccontextmanager
async def lifespan(app: FastAPI):
    # Runs once at startup: create the table if it does not exist.
    with connect() as conn:
        conn.execute(
            "CREATE TABLE IF NOT EXISTS notes ("
            " id serial PRIMARY KEY, body text NOT NULL)"
        )
    yield


app = FastAPI(lifespan=lifespan)


@app.get("/health")
def health():
    cache.ping()
    with connect() as conn:
        conn.execute("SELECT 1")
    return {"status": "ok"}


@app.get("/hits")
def hits():
    return {"hits": cache.incr("hits")}


@app.post("/notes")
def add_note(body: str):
    with connect() as conn:
        row = conn.execute(
            "INSERT INTO notes (body) VALUES (%s) RETURNING id", (body,)
        ).fetchone()
    return {"id": row[0], "body": body}


@app.get("/notes")
def list_notes():
    with connect() as conn:
        rows = conn.execute("SELECT id, body FROM notes ORDER BY id").fetchall()
    return [{"id": r[0], "body": r[1]} for r in rows]
```

`read_secret` lets the same code use a plain variable in dev and a mounted secret file in prod (`DB_PASSWORD_FILE`). The parameterized query (`%s` with a tuple) prevents SQL injection; do not build SQL with f-strings.

**`.env`** (not committed; `.env.example` has the same keys with placeholders):

```
POSTGRES_DB=notes
POSTGRES_USER=notes
POSTGRES_PASSWORD=dev-only-change-me
```

**`compose.yaml`** (single file, dev-friendly):

```yaml
name: notes-app

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      DB_HOST: db
      DB_NAME: ${POSTGRES_DB:-notes}
      DB_USER: ${POSTGRES_USER:-notes}
      DB_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 10s

  db:
    image: postgres:18
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-notes}
      POSTGRES_USER: ${POSTGRES_USER:-notes}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
  redisdata:
```

| Setting | Explanation |
|---|---|
| `${POSTGRES_DB:-notes}` | Interpolation with default: the `.env`/shell value, else `notes`. |
| `${POSTGRES_PASSWORD:?Set ...}` | Required: `docker compose up` fails with that message if unset, instead of starting a database with an empty password. |
| `DB_HOST: db` | The Postgres service name is the hostname. |
| `pgdata:/var/lib/postgresql` | Named volume. The path is correct for Postgres 18+; for 17 and earlier use `/var/lib/postgresql/data` ([Chapter 5](05-storage-and-networking.md#a2-volumes)). |
| `$${POSTGRES_USER}` | `$$` makes the container's shell expand the variable, not Compose. |
| `command: ["redis-server","--appendonly","yes"]` | Enables Redis append-only-file persistence so data in `/data` survives restarts. |
| `redisdata:/data` | The Redis image keeps its data in `/data`. |
| top-level `volumes:` | Declares both volumes (created as `notes-app_pgdata`, `notes-app_redisdata`). |
| `web.healthcheck` | Calls `/health`, which tests Redis and Postgres, so "healthy" means the whole path works. |

```bash
docker compose up -d --build
docker compose ps
```

Wait until all three show `(healthy)`, then:

```bash
curl -X POST "http://localhost:8000/notes?body=first%20note"
# {"id":1,"body":"first note"}
curl http://localhost:8000/notes
# [{"id":1,"body":"first note"}]
curl http://localhost:8000/hits
# {"hits":1}
```

Verify persistence:

```bash
docker compose down            # remove containers and network, keep volumes
docker compose up -d
curl http://localhost:8000/notes     # the note is still there
curl http://localhost:8000/hits      # {"hits":2}
```

Inspect what Compose created:

```bash
docker volume ls --filter name=notes-app
docker network ls --filter name=notes-app
docker compose logs --tail 20 db
```

`docker compose down -v` deletes `pgdata` and `redisdata`.

| Symptom | Cause |
|---|---|
| `required variable POSTGRES_PASSWORD is missing a value` | `.env` not created. The `:?` guard is working. |
| `web` restarts with `connection refused` to db | `condition: service_healthy` omitted, or the app starts before the DB is ready. |
| Postgres refuses to start with a data directory error, or the database is empty after `down`/`up` | The volume mount path does not match the image's major version (Postgres 18 note in [Chapter 5](05-storage-and-networking.md#a2-volumes)). Read the message with `docker compose logs db`. |
| Changing `POSTGRES_PASSWORD` in `.env` has no effect | The password is applied only when the volume is first initialized. Change it with SQL (`ALTER USER`) or recreate the volume (data loss; dev only). |

### Stage 4: dev vs prod with override files

Base holds what is the same everywhere; overrides hold what differs.

**`Dockerfile`** (production-minded version from [Chapter 4](04-dockerfile-deep-dive.md#23-a-full-production-grade-dockerfile-for-fastapi), with a non-root user so `watch` and `read_only` are meaningful):

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.13-slim

ENV PYTHONUNBUFFERED=1 PYTHONDONTWRITEBYTECODE=1

RUN groupadd --system --gid 10001 app \
 && useradd --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

COPY --chown=app:app app/ ./app/
USER 10001:10001
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**`compose.yaml`: shared base** (no ports, no passwords, no dev tools):

```yaml
name: notes-app

services:
  web:
    build: .
    image: ghcr.io/your-user/notes-web:${TAG:-dev}
    environment:
      DB_HOST: db
      DB_NAME: ${POSTGRES_DB:-notes}
      DB_USER: ${POSTGRES_USER:-notes}
      REDIS_URL: redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
      interval: 15s
      timeout: 3s
      retries: 3
      start_period: 10s
    networks: [frontend, backend]

  db:
    image: postgres:18
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-notes}
      POSTGRES_USER: ${POSTGRES_USER:-notes}
    volumes:
      - pgdata:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    networks: [backend]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks: [backend]

volumes:
  pgdata:
  redisdata:

networks:
  frontend:
  backend:
```

With both `build: .` and `image:` set, dev builds and tags the image, and prod pulls the same name from the registry (replace `your-user` with your registry path).

**`compose.override.yaml`: development** (auto-loaded by `docker compose up`):

```yaml
services:
  web:
    ports:
      - "8000:8000"
    environment:
      APP_ENV: development
      DB_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    develop:
      watch:
        - action: sync
          path: ./app
          target: /app/app
        - action: rebuild
          path: requirements.txt

  db:
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
    ports:
      - "127.0.0.1:5432:5432"      # lets a GUI or psql on the laptop connect
```

The override publishes the web port, supplies the dev password from `.env`, swaps the command for `--reload`, defines `watch` rules, and publishes Postgres on loopback only. `environment` is a mapping, so it merges with the base; `command` is replaced; `ports` are appended.

Dev workflow:

```bash
docker compose watch          # build, start, sync ./app edits into the container
# edit app/main.py -> uvicorn reloads inside the container within a second
docker compose run --rm web python -c "print('one-off task')"
docker compose exec db psql -U notes -d notes -c "SELECT * FROM notes;"
```

**`compose.prod.yaml`: production overlay:**

```yaml
services:
  web:
    pull_policy: always              # always fetch the tagged image from the registry
    restart: unless-stopped
    init: true
    ports:
      - "127.0.0.1:8000:8000"        # reachable only from the host's reverse proxy
    environment:
      APP_ENV: production
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    read_only: true                  # immutable root filesystem...
    tmpfs:
      - /tmp                         # ...with a RAM scratch dir
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  db:
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    deploy:
      resources:
        limits:
          memory: 1G
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 256M

secrets:
  db_password:
    file: ./secrets/db_password.txt

networks:
  backend:
    internal: true                   # DB and cache cannot reach the internet
```

| Change | Reason |
|---|---|
| `restart: unless-stopped` | Survive crashes and host reboots |
| `init: true` | Signal handling and zombie reaping ([Chapter 4](04-dockerfile-deep-dive.md#16-pid-1-signals-tini-and---init)) |
| `127.0.0.1:8000:8000` | Only a reverse proxy on the same host (nginx/Caddy, [Chapter 5](05-storage-and-networking.md#b12-reverse-proxies-in-front)) faces the internet |
| No DB port published | The database is reachable only by `web`, on the internal network |
| Secrets via files | No plaintext password in the environment or Compose file |
| `read_only` + `tmpfs` + `cap_drop` + `no-new-privileges` | Container hardening ([Chapter 10](10-security.md)) |
| Resource limits | One service cannot starve the host |
| Log rotation | Prevents logs filling the disk |
| `internal: true` backend | Defense in depth |
| `pull_policy: always` | Deploy exactly what CI published |

Prepare the secret once on the prod host (never in Git; `.gitignore` must contain `secrets/`):

```bash
mkdir -p secrets
openssl rand -base64 24 > secrets/db_password.txt
chmod 644 secrets/db_password.txt     # the non-root container user (UID 10001) must be able to read it
```

The file is bind-mounted, so the container sees the host's permissions. `644` (readable by all host users) is acceptable only on a single-purpose server. Weigh it against your threat model or use a secrets manager.

Run prod (`-f` is given, so the dev override is not loaded):

```bash
export TAG=1.4.2
docker compose -f compose.yaml -f compose.prod.yaml config --quiet     # validate; non-zero exit on error
docker compose -f compose.yaml -f compose.prod.yaml pull
docker compose -f compose.yaml -f compose.prod.yaml up -d --no-build --wait
docker compose -f compose.yaml -f compose.prod.yaml ps
```

| Command | Purpose |
|---|---|
| `config --quiet` | Validate the merged files without printing them |
| `pull` | Fetch `ghcr.io/your-user/notes-web:1.4.2` and the other images |
| `up -d --no-build --wait` | Start in the background, never build on the server, wait until everything is running/healthy (exit status shows success) |

To avoid repeating `-f`, set `COMPOSE_FILE=compose.yaml:compose.prod.yaml` in the shell or CI environment (`;` separator on Windows).

> **First run in prod:** `POSTGRES_PASSWORD_FILE` takes effect only on first initialization of the volume. If dev previously ran against the same `pgdata` volume with `dev-only-change-me`, the old password is still in effect. Use a separate project name or a fresh volume for prod (`-p notes-prod`).

---

## Compose vs Swarm vs Kubernetes

| | **Docker Compose** | **Docker Swarm** | **Kubernetes** |
|---|---|---|---|
| What it is | Define and run multi-container apps, mainly on one host | Docker's built-in clustering mode (multiple hosts) | Standard container orchestration platform (many hosts) |
| Config format | `compose.yaml` | Compose-format files via `docker stack deploy` | YAML manifests (Deployments, Services, Ingress), Helm, Kustomize |
| Multi-host | No (one Docker engine; a Docker context can target a remote one, still one host) | Yes | Yes |
| Self-healing / reschedule on node failure | No (restart policy on the same host only) | Yes | Yes |
| Rolling updates, rollbacks | Basic (recreate); no built-in zero-downtime | Yes | Yes (first-class) |
| Autoscaling | No (manual `--scale`) | Manual scaling | Yes (HPA, cluster autoscaler) |
| Service discovery / load balancing | Docker DNS on one host | Built-in overlay + routing mesh | Services, Ingress/Gateway |
| Secrets/config | Files from the host | Swarm secrets (encrypted at rest in the cluster) | Secrets, ConfigMaps, external secret operators |
| Learning curve | Low | Medium | High |
| Operational cost | Minimal | Low-medium | High (or a managed service) |
| Ecosystem | Large for local/dev | Small, mostly stagnant | Very large |

Swarm mode remains part of Docker Engine and is documented by Docker. Unverified: its current level of investment and roadmap. Check current Docker announcements before choosing Swarm for a new project. Compose files work with Swarm (`docker stack deploy`), but some keys differ (`deploy` applies there, `build` does not).

### When Compose is enough in production

- The app runs on one server (VM) and a short restart during deploys is acceptable.
- Traffic is modest and autoscaling or multi-AZ resilience is not required.
- Internal tools, MVPs, side projects, demos, self-hosted services, single-tenant deployments, build/CI environments.
- The team is small and operational simplicity matters more than cluster features.

A sound Compose production setup includes:

- pinned image tags or digests
- `restart: unless-stopped`
- healthchecks
- resource limits
- log rotation
- named volumes with tested backups ([Chapter 5](05-storage-and-networking.md#a9-backup-and-restore-of-a-volume-tar-method))
- a reverse proxy with TLS
- secrets kept out of Git
- images built and scanned in CI, deployed with `docker compose pull && docker compose up -d`
- the Docker service enabled at boot
- monitoring and alerting
- a host firewall plus a cloud firewall (Docker bypasses ufw, [Chapter 5](05-storage-and-networking.md#b9-firewalls-and-why-docker-bypasses-ufw))

Deploy to a remote host without logging in:

```bash
docker context create prod --docker "host=ssh://deploy@server.example.com"
docker --context prod compose -f compose.yaml -f compose.prod.yaml up -d
```

`docker context` points the local Docker CLI at a remote engine over SSH. Use a dedicated limited deploy user; access to the Docker socket is effectively root on that machine.

### When Compose is not enough

Use an orchestrator (usually Kubernetes, or a managed service such as AWS ECS/Fargate, Azure Container Apps, Google Cloud Run) when you need:

- high availability: automatic survival of a machine failure
- zero-downtime rolling deploys and quick rollbacks as a built-in guarantee
- autoscaling on load
- many services, many teams or multi-tenant platforms, RBAC, quotas, network policies
- multi-host scheduling and bin-packing
- GPU fleets shared between workloads ([Chapter 8](08-docker-for-ai-and-ml.md))
- platform-level auditability and policy enforcement

Compose files already hold service definitions, ports, environment, volumes and healthchecks. Converters exist (`kompose`; `docker compose bridge` is Docker's own converter-style feature), but the output needs hand-tuning. See [Chapter 11](11-production-and-enterprise.md) for the migration path.

---

## Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Warning `the attribute 'version' is obsolete` | `version: "3.8"` at the top | Delete it |
| `connection refused` between services | `localhost` used | Use service names ([Chapter 5](05-storage-and-networking.md#b4-localhost-inside-containers)) |
| `connection refused` at startup | `depends_on` without `condition: service_healthy` | Add a healthcheck and the condition |
| Port parsed as a number | Unquoted `22:22` | Write `"22:22"` |
| Parse errors or wrong structure | Tabs or wrong indentation | Spaces only; check with `docker compose config` |
| Config change not applied | `restart` does not re-read config | `docker compose up -d` |
| Code change not visible | `up` does not rebuild | `--build`, `watch` or `docker compose build` |
| Database data gone | `docker compose down -v` | Omit `-v` |
| New empty volumes after rename | Project or folder name changed | Pin `name:` |
| Secrets in Git | `.env` or `secrets/` committed | Commit `.env.example`; gitignore `.env` and `secrets/` |
| Variable missing in container | `.env` feeds interpolation only | Use `environment:` or `env_file:` |
| Healthcheck variable empty | `$VAR` expanded by Compose | Write `$$VAR` |
| Database reachable externally | `"5432:5432"` published | Do not publish, or bind to `127.0.0.1` |
| Cannot scale; name clashes | `container_name:` set | Remove it |
| Service unreachable after adding networks | `default` network dropped, or `network_mode` combined with `networks` | List `default`; do not combine |
| Unhealthy container not restarted | Healthchecks do not restart containers | Use `restart:` for process exits |
| Disk fills with logs | No log rotation | `logging.options.max-size` |
| Dev settings in production | `compose.override.yaml` auto-loads unless `-f` is given | Name files explicitly or use `COMPOSE_FILE` |
| Secrets visible via `docker inspect` / `docker history` | Secrets in `environment:` or `build.args` | Use `secrets:` |
| No rolling updates | Compose used like Kubernetes on one host | Use an orchestrator |
| `docker-compose: command not found` | V1 command from an old tutorial | Use `docker compose` |

---

## Summary

- Compose turns a multi-container app into one `compose.yaml` and one command. Use `docker compose` (V2) and omit `version:`.
- Services find each other by service name; publish only what outsiders need.
- Use `depends_on` with `condition: service_healthy` and real healthchecks; make apps retry connections.
- Keep data in named volumes. `down` keeps them; `down -v` deletes them.
- Interpolation (`${VAR}` from shell and `.env`) is separate from container environment. Container precedence: `run -e` > `environment` > `env_file` > Dockerfile `ENV`. `docker compose config` shows the result.
- Split environments with override files (`compose.override.yaml` auto-loads; `-f` replaces that); reuse with `extends`, `include` and anchors; make services optional with profiles.
- Deliver sensitive values as secrets (files); harden production services (limits, read-only, dropped capabilities, log rotation).
- `docker compose watch` provides cross-platform hot reload.
- Compose suits single-host production with modest needs; use Kubernetes or a managed platform for HA, autoscaling and multi-host scheduling.

**Next:** [Chapter 7: Who Needs Docker?](07-who-needs-docker.md) and [Chapter 8](08-docker-for-ai-and-ml.md).
