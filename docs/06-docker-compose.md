# Chapter 6: Docker Compose

> **What you will learn in this chapter**
>
> - What Docker Compose is, why it exists, and how to check your install
> - The anatomy of a `compose.yaml` (file names, why the `version:` key is obsolete)
> - Every important service option: `build`, `image`, `ports`, `environment`, `env_file`, `volumes`, `networks`, `depends_on`, `healthcheck`, `restart`, `command`, resource limits
> - How `.env` interpolation and environment variable precedence really work
> - Profiles, multiple files, override files, `extends`, `include`, YAML anchors
> - Secrets and configs, and `docker compose watch` for hot reload
> - Every day-to-day command (`up`, `down`, `ps`, `logs`, `exec`, `run`, `build`, `config`, ...) and the dangerous ones (`down -v`)
> - A progressive tutorial: one service, then web + Redis, then web + Postgres + Redis with healthchecks and volumes, then separate dev and prod setups
> - Compose vs Swarm vs Kubernetes: when Compose is enough in production, and when it isn't

**Prerequisites:** [Chapter 4](04-dockerfile-deep-dive.md) (Dockerfiles) and [Chapter 5](05-storage-and-networking.md) (volumes and networks). Compose is mostly a friendly YAML wrapper around concepts you already know.

**Where this fits:** In real projects you almost never run a single container. A web app needs a database, a cache, maybe a worker and a proxy. Typing five long `docker run` commands is miserable and error-prone. Compose lets you write the whole setup **once, in a file you commit to Git**, and run it with one command. It is also the bridge to the AI chapters ([Chapter 8](08-docker-for-ai-and-ml.md) and [Chapter 9](09-hands-on-ai-app-walkthrough.md)), where an app + vector database + model server is the norm.

---

## Table of contents

1. [What Compose is and why you need it](#what-compose-is-and-why-you-need-it)
2. [Checking your install](#checking-your-install)
3. [Anatomy of a Compose file](#anatomy-of-a-compose-file)
4. [Service options in detail](#service-options-in-detail)
5. [Environment variables: interpolation and precedence](#environment-variables-interpolation-and-precedence)
6. [Volumes](#volumes)
7. [Networks](#networks)
8. [depends_on and startup order](#depends_on-and-startup-order)
9. [Healthchecks](#healthchecks)
10. [Restart policies](#restart-policies)
11. [command and entrypoint](#command-and-entrypoint)
12. [Resource limits](#resource-limits)
13. [Profiles](#profiles)
14. [Multiple Compose files and overrides](#multiple-compose-files-and-overrides)
15. [extends and include](#extends-and-include)
16. [YAML anchors and extension fields](#yaml-anchors-and-extension-fields)
17. [Secrets and configs](#secrets-and-configs)
18. [docker compose watch: hot reload for development](#docker-compose-watch-hot-reload-for-development)
19. [Project names](#project-names)
20. [Compose commands reference](#compose-commands-reference)
21. [Progressive tutorial](#progressive-tutorial)
22. [Compose vs Swarm vs Kubernetes](#compose-vs-swarm-vs-kubernetes)
23. [Common mistakes](#common-mistakes)
24. [Exercises](#exercises)
25. [Summary](#summary)

---

## What Compose is and why you need it

**Docker Compose** is a tool for defining and running **multi-container applications**. You describe your application's services, networks and volumes in a YAML file (conventionally `compose.yaml`), and Compose creates and starts everything with one command.

**YAML** (the format of the file) is a human-friendly data format built from indentation, `key: value` pairs and `- list items`. Indentation uses **spaces, never tabs**.

### The problem it solves

Without Compose, running a web app + Postgres + Redis looks like this:

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

You must remember all of it, run it in the right order, copy it to teammates (through chat messages, probably), and tear it down in reverse. With Compose, that is a file in Git plus:

```bash
docker compose up -d
```

Analogy: `docker run` is cooking one dish by hand; Compose is the **restaurant's menu and kitchen workflow** that produces the whole meal consistently every time.

### What Compose gives you

- **One source of truth**: the stack is documented and version-controlled.
- **One command** to build, start, stop and delete the whole stack.
- **Automatic networking**: a private network per project, with service names as DNS names ([Chapter 5](05-storage-and-networking.md#b2-default-bridge-vs-user-defined-bridge)).
- **Reproducibility** between teammates, laptops and CI.
- **Dev ergonomics**: live reload (`watch`), one-off commands (`run`), log aggregation.
- **Idempotent**: running `up` again only changes what changed.

### The Compose application model

Per Docker's documentation, a Compose application is built from these pieces:

| Concept | Meaning |
|---|---|
| **Project** | One deployment of your app. Has a name that groups and isolates its resources |
| **Service** | A computing component (web, db, worker). Defined once; can run as 1 or many identical **containers** |
| **Network** | Connects services' containers |
| **Volume** | Persistent storage |
| **Config** | Non-sensitive configuration files mounted into containers |
| **Secret** | Sensitive data mounted as files |

**Service vs container:** a *service* is the definition ("the web service uses this image with these settings"); a *container* is a running instance of it. `docker compose up --scale web=3` runs 3 containers of the one `web` service.

### Compose V1 vs V2 (a historical note)

Old tutorials use `docker-compose` (with a hyphen), the original Python program, now called Compose V1, which reached end of life in 2023. Modern Compose is **Compose V2**, written in Go and shipped as a Docker CLI plugin: **`docker compose`** (with a space). Everything in this chapter uses the space form. If a tutorial says `docker-compose up`, just type `docker compose up`.

---

## Checking your install

Docker Desktop (Windows, macOS, Linux) bundles Compose. On a Linux server with Docker Engine you install the `docker-compose-plugin` package (see [Chapter 2](02-installation-and-operating-systems.md)).

```bash
docker compose version
```

Example output (this guide was written with this version):

```
Docker Compose version v5.6.1
```

- Use `docker compose version` (a subcommand), not `--version`.
- Compose's version numbers do not match Docker Engine's (the machine used to write this guide has Docker Engine 29.9.2 and Compose 5.6.1; older tutorials will show 2.x). The file format (the **Compose Specification**) is versioned independently and is "always latest": there is no file-format version to choose.

See every command available to you with `docker compose --help` (the full list is in the [commands reference](#compose-commands-reference) below).

---

## Anatomy of a Compose file

### File names

Compose looks in the current directory (and then parent directories) for, in this preference order:

1. `compose.yaml` **(preferred)**
2. `compose.yml`
3. `docker-compose.yaml` (legacy, still supported)
4. `docker-compose.yml` (legacy, still supported)

If both `compose.yaml` and a legacy name exist, `compose.yaml` wins. Use **`compose.yaml`** for new projects. Compose also automatically merges a file named **`compose.override.yaml`** if it's present (section on [overrides](#multiple-compose-files-and-overrides)).

### The `version:` key is obsolete

Old files start with:

```yaml
version: "3.8"      # <-- do NOT write this anymore
services:
  ...
```

According to the Compose Specification documentation, the top-level `version` property is **obsolete**: it's kept only for backward compatibility and is merely informative. Compose ignores it, always validates against the most recent schema, and prints a **warning** if you include it. So: leave it out. (Old "version 2 vs 3" differences don't apply to Compose V2; you just write the Specification.) If you see `the attribute 'version' is obsolete` in your terminal, delete that line.

### A minimal file

```yaml
name: demo

services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
```

Line by line:

| Line | Meaning |
|---|---|
| `name: demo` | (Optional) the **project name**, used to prefix container, network and volume names. If omitted, Compose uses the folder name. |
| `services:` | Top-level section listing the containers-to-be. Required. |
| `web:` | The **service name** (your choice). It's also the DNS name other services use to reach it. |
| `image: nginx:1.27-alpine` | Which image to run. |
| `ports:` | A list of port mappings, `"HOST:CONTAINER"`. |
| `- "8080:80"` | Publish container port 80 on host port 8080. **Quote** port mappings: YAML can misread unquoted `22:22` as a base-60 number (a real YAML quirk). |

### The top-level elements

A Compose file can contain these top-level keys:

| Key | Purpose |
|---|---|
| `name` | Project name |
| `services` | The containers (required) |
| `networks` | Named networks |
| `volumes` | Named volumes |
| `configs` | Non-sensitive config files |
| `secrets` | Sensitive files |
| `include` | Pull in other Compose files |
| `x-...` | **Extension fields**: your own reusable fragments, ignored by Compose |

---

## Service options in detail

Each entry under `services:` accepts many options. Here are the ones you'll use constantly, each explained.

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
      context: ./api                  # the build context directory (default: .)
      dockerfile: Dockerfile.prod     # which Dockerfile (relative to context)
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
      secrets:                        # build-time secrets (see Secrets section)
        - pip_token
      ssh:
        - default                     # forward your SSH agent
    image: registry.example.com/me/api:1.0     # name/tag for the built image
```

- `context`: the directory sent to the builder; Dockerfile `COPY` paths are relative to it ([Chapter 4](04-dockerfile-deep-dive.md#4-build-context-and-dockerignore)).
- `target`: which stage of a [multi-stage Dockerfile](04-dockerfile-deep-dive.md#11-multi-stage-builds) to build.
- `args`: values for Dockerfile `ARG`s. Never put secrets here.
- When **both** `build` and `image` are set, Compose builds and **tags the result with the `image` name**. That's how one file serves dev (build locally) and prod (pull that same name from a registry).
- Without `image`, the built image gets a generated name like `<project>-<service>`.
- Commands: `docker compose build` builds; `docker compose up --build` builds then starts. Compose does *not* rebuild automatically when you merely run `up` (unless the image is missing), a classic source of "my code change isn't showing up".

**Compose Bake (verified status):** Compose can delegate builds to **Docker Bake** (buildx's build orchestrator, good for building many images in parallel). Docker's documentation describes enabling it with `COMPOSE_BAKE=true docker compose build`, or permanently via `"plugins": {"compose": {"build": "bake"}}` in `~/.docker/config.json`. The documentation page I checked describes this as opt-in; I could not confirm whether any recent release made it the default, so check the output of `docker compose build` on your version.

### `ports` and `expose`

```yaml
    ports:
      - "8000:8000"                  # host 8000 -> container 8000, all interfaces
      - "127.0.0.1:5432:5432"        # only reachable from this machine
      - "9090:9090/udp"              # UDP
      - target: 80                   # long syntax
        published: "8080"
        protocol: tcp
        host_ip: 127.0.0.1
    expose:
      - "8000"                       # documentation only (like Dockerfile EXPOSE)
```

`ports` publishes to the host ([Chapter 5](05-storage-and-networking.md#b3-port-publishing)). `expose` publishes nothing; other services on the network can reach any listening port anyway. **Don't publish what only other containers need.**

### `environment` and `env_file`

```yaml
    environment:
      APP_ENV: production            # map syntax
      LOG_LEVEL: info
      DEBUG:                         # no value: pass through from your shell if set
    # or the list syntax:
    # environment:
    #   - APP_ENV=production
    env_file:
      - .env.web                     # a file of KEY=value lines
      - path: .env.local
        required: false              # don't fail if the file is missing
```

Full precedence rules in the [next section](#environment-variables-interpolation-and-precedence).

### `volumes` (service level)

```yaml
    volumes:
      - pgdata:/var/lib/postgresql          # named volume (declared at top level)
      - ./config/app.toml:/etc/app.toml:ro  # bind mount, read-only
      - type: tmpfs                         # long syntax
        target: /tmp
```

Covered in detail in [Volumes](#volumes).

### `networks`

```yaml
    networks:
      - frontend
      - backend
```

See [Networks](#networks).

### `depends_on`, `healthcheck`, `restart`, `command`, `entrypoint`

Each has its own section below.

### Other options you'll meet

| Option | What it does |
|---|---|
| `container_name: web` | Fixed container name. **Avoid**: it prevents scaling and clashes across projects. Compose's default (`<project>-<service>-<n>`) is better. |
| `hostname: web` | The container's internal hostname. |
| `working_dir: /app` | Override the image's `WORKDIR`. |
| `user: "10001:10001"` | Run as this UID:GID (override image `USER`). |
| `init: true` | Run a tiny init process as PID 1 that forwards signals and reaps zombies ([Chapter 4](04-dockerfile-deep-dive.md#16-pid-1-signals-tini-and---init)). |
| `stop_signal: SIGINT` | Signal sent by `docker compose stop`/`down` (default `SIGTERM`). |
| `stop_grace_period: 30s` | Wait this long before `SIGKILL` (default 10 s). |
| `read_only: true` | Read-only root filesystem (pair with `tmpfs: [/tmp]`). |
| `tmpfs: [/tmp]` | RAM-backed scratch directories. |
| `cap_drop: [ALL]` | Drop Linux capabilities (privilege slices) the container doesn't need ([Chapter 10](10-security.md)). |
| `security_opt: ["no-new-privileges:true"]` | Block privilege escalation inside the container. |
| `extra_hosts: ["host.docker.internal:host-gateway"]` | Add `/etc/hosts` entries (needed on Linux to reach the host, [Chapter 5](05-storage-and-networking.md#b5-container-to-host-hostdockerinternal)). |
| `dns: [10.0.0.2]` | Custom DNS servers. |
| `labels` | Metadata (also used by Traefik etc. for routing). |
| `logging` | Log driver/options, e.g. `driver: json-file`, `options: {max-size: "10m", max-file: "3"}`; **without a size limit, container logs grow without bound**. |
| `platform: linux/amd64` | Force a platform (emulated on mismatching hardware). |
| `pull_policy` | `always`, `never`, `missing` (default), `build`, `daily`, `weekly`, `every_<duration>` such as `every_12h`. |
| `ulimits`, `sysctls`, `shm_size` | Low-level limits and kernel params (`shm_size` matters for some ML/browser workloads). |
| `gpus: all` | Request GPU access ([Chapter 8](08-docker-for-ai-and-ml.md)). |
| `scale: 3` / `deploy.replicas: 3` | Number of containers for the service. |

---

## Environment variables: interpolation and precedence

There are **two different mechanisms** that both involve "environment variables" and beginners mix them up constantly:

1. **Interpolation**: Compose substitutes `${VARIABLES}` **inside the YAML file itself**, on your machine, **before** anything is sent to Docker. The values come from your shell and the `.env` file.
2. **Container environment**: the variables set **inside the running container**, via `environment:` and `env_file:`.

### 1. Interpolation (`${VAR}` in the YAML)

```yaml
services:
  db:
    image: postgres:${PG_VERSION:-18}
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?Set POSTGRES_PASSWORD in .env}
```

Supported forms:

| Syntax | Meaning |
|---|---|
| `${VAR}` or `$VAR` | The value of `VAR` (empty string, plus a warning, if unset) |
| `${VAR:-default}` | `default` if `VAR` is **unset or empty** |
| `${VAR-default}` | `default` only if `VAR` is **unset** (an empty value stays empty) |
| `${VAR:?error message}` | **Fail with the message** if `VAR` is unset or empty (great for required settings) |
| `${VAR?error message}` | Fail only if unset |
| `${VAR:+replacement}` | `replacement` if `VAR` is set and non-empty, else empty |
| `$$` | A literal `$` (escape; needed in healthcheck commands and shell snippets) |

**Where interpolation values come from**, highest priority first (per Docker's documentation, the shell value beats the `.env` file):

1. Variables in your **shell environment** (`export PG_VERSION=17` / `$env:PG_VERSION="17"`)
2. The `.env` file (and any file(s) given with `--env-file`)
3. Defaults written in the YAML (`${VAR:-default}`)

The **`.env` file** is automatically read from the **project directory** (where your `compose.yaml` lives). Format:

```
# .env  (comments allowed)
POSTGRES_DB=notes
POSTGRES_USER=notes
POSTGRES_PASSWORD=dev-only-change-me
PG_VERSION=18
```

- `docker compose --env-file ./config/staging.env up` uses an alternative file *instead of* the default `.env`.
- I could not verify from the documentation I consulted the exact ranking when `--env-file` and shell variables conflict; the documented rule is simply "shell beats `.env`". Test with `docker compose config` (below) if it matters.
- The `.env` file is for **interpolation only**. Its variables are **not** automatically passed into containers. To get them in, reference them in `environment:` (`DB_USER: ${POSTGRES_USER}`) or use `env_file:`.
- **Never commit `.env` with real secrets.** Commit a `.env.example` with fake placeholder values and add `.env` to `.gitignore`.
- The special variables `COMPOSE_PROJECT_NAME`, `COMPOSE_FILE`, `COMPOSE_PROFILES` and a few others configure Compose itself and can be set in `.env` or the shell.

### 2. Variables inside the container

Set with `environment:` and/or `env_file:`. When the same variable is defined in several places, the winner (highest first, per Docker's documentation):

1. `docker compose run -e VAR=value` on the command line
2. `environment:` in the Compose file (after interpolation)
3. `env_file:` files in the Compose file
4. `ENV` in the image's Dockerfile (only applies if nothing above sets it)

Shortcut to remember: **command line > compose `environment` > compose `env_file` > Dockerfile `ENV`.**

(If you have multiple `env_file` entries, later files override earlier ones.)

### See what Compose actually resolved

```bash
docker compose config
```

Prints the **fully merged, interpolated** configuration, exactly what Compose will use. It's the single best debugging tool: it also validates syntax and shows warnings.

Useful variations:

```bash
docker compose config --services        # just the service names
docker compose config --volumes         # just volume names
docker compose config --quiet           # validate only, no output (exit code 0 = valid; good for CI)
docker compose config --format json     # machine-readable
docker compose -f compose.yaml -f compose.prod.yaml config    # see the merged result of several files
```

> **Caution:** `docker compose config` prints resolved values, **including secrets that came from `.env`/environment**. Don't paste its output into chat or tickets without checking.

---

## Volumes

Compose volumes use the same concepts as [Chapter 5](05-storage-and-networking.md): named volumes, bind mounts, tmpfs.

### Declaring and using named volumes

```yaml
services:
  db:
    image: postgres:18
    volumes:
      - pgdata:/var/lib/postgresql     # SOURCE:TARGET, source = volume name

volumes:
  pgdata:                              # declare it at the top level
```

- If `pgdata` is declared at the top level and used by a service, Compose creates it on first `up` as **`<project>_pgdata`** (project name prefix!).
- `docker compose down` removes containers and networks but **keeps** named volumes. `docker compose down -v` deletes them: data loss.
- An empty named volume is pre-populated from the image on first mount.

### Short vs long syntax

```yaml
    volumes:
      # short syntax
      - pgdata:/var/lib/postgresql
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      # long syntax (explicit, like docker's --mount)
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

Relative bind paths (`./src`) are relative to the Compose file's directory (unlike plain `docker run`, where you must use absolute paths or `$(pwd)`). For bind mounts, the long syntax `bind: {create_host_path: false}` makes Compose error instead of creating a missing host directory.

### Volume options

```yaml
volumes:
  pgdata:
    name: notes_pgdata               # use an exact name, no project prefix
  shared_models:
    external: true                   # already exists; Compose must NOT create or delete it
  nfs_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.50,rw,nfsvers=4
      device: ":/exports/shared"
```

- `external: true`: the volume is managed outside this project. Compose errors if it doesn't exist and never deletes it with `down -v`. Create it yourself with `docker volume create shared_models`. Good for data you **must not** lose (or share across projects, for example a large model cache).
- `name:`: pins the actual volume name, protecting you from accidentally getting a *new empty volume* when the project name changes.
- `driver`/`driver_opts`: volume drivers ([Chapter 5](05-storage-and-networking.md#a6-volume-drivers)).

### Common volume mistakes in Compose

- Wrong mount path for the image (e.g., the Postgres 18 path change described in [Chapter 5](05-storage-and-networking.md#a2-volumes)).
- Bind-mounting `.` over `/app` and hiding the dependencies installed in the image.
- Changing the project name (or folder name) and finding an empty database: the old data is in `oldname_pgdata`, still there, untouched. Fix by pinning `name:` or setting `name:` at the top of the file.
- Running `docker compose down -v` casually.

---

## Networks

By default, Compose creates **one network per project** named `<project>_default` (a user-defined bridge). Every service joins it and is reachable **by its service name** on its container port. You need to configure nothing:

```
 project "notes-app"  ->  network notes-app_default
   web     reaches   db      as  db:5432
   web     reaches   redis   as  redis:6379
```

This is why you write `DB_HOST: db` (the service name) and never `localhost` ([Chapter 5, section B4](05-storage-and-networking.md#b4-the-1-beginner-bug-localhost-confusion)).

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

- A service only sees services that share at least one network with it. Here `proxy` cannot reach `db`.
- `internal: true`: containers on this network can't reach the outside world (and published ports don't work for it). Good for databases in production, but don't use it in dev if you need to publish the DB port to your laptop.
- Note: the moment you define networks on a service, it **no longer joins `default`** unless you list `default` too.

### Aliases, static IPs, external networks

```yaml
services:
  api:
    networks:
      backend:
        aliases:
          - api.internal            # extra DNS name on this network
        ipv4_address: 172.28.0.10   # static IP (needs the network's subnet defined)
networks:
  backend:
    ipam:
      config:
        - subnet: 172.28.0.0/24
  shared:
    external: true                  # created outside Compose: docker network create shared
    name: shared                    # actual name
```

- **Cross-project communication:** create `docker network create shared`, then mark it `external: true` in both projects' files and attach the relevant services. They can then reach each other by service name.
- Avoid static IPs unless you have a real reason; names are the point.

### Network mode

```yaml
    network_mode: host         # use the host's network stack (Linux)
    network_mode: "service:vpn"  # share another service's network namespace (like a sidecar)
```

`network_mode` and `networks` can't be combined.

---

## depends_on and startup order

`depends_on` tells Compose in what **order** to start services (and to stop them in reverse).

```yaml
services:
  web:
    depends_on:
      - db
      - redis
```

### The crucial catch

The short form above waits only until the dependencies' containers are **started**, not until the software inside is **ready**. Postgres takes several seconds after its container starts to be able to accept connections. If your app tries to connect immediately, it crashes with `connection refused`. This is the most common Compose beginner failure.

### The right way: conditions + healthchecks

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

The `condition` values (from the Compose Specification):

| Condition | Compose waits until the dependency... |
|---|---|
| `service_started` | has started (same as the short syntax) |
| `service_healthy` | passes its `healthcheck` (so the dependency **must define one**) |
| `service_completed_successfully` | has **run to completion with exit code 0** (for one-shot jobs like database migrations or "init" tasks) |

Extra long-syntax fields:

- `restart: true`: when Compose updates/recreates the dependency, it also restarts this service (Compose 2.17+).
- `required: false`: if the dependency isn't available (for example it's in an inactive [profile](#profiles)), only warn instead of failing (Compose 2.20+).

### Limits (be honest about them)

1. `depends_on` only matters at **startup** (`docker compose up`). If the database crashes at 3 a.m., Compose doesn't restart your web service. Use `restart:` policies and make your app **retry its connections**.
2. It is not a substitute for application-level resilience. Production apps should tolerate dependencies being briefly unavailable (retry with backoff).
3. In old tutorials you'll see `wait-for-it.sh` scripts: unnecessary today if you use `service_healthy`.

---

## Healthchecks

A healthcheck in Compose is the same concept as the Dockerfile `HEALTHCHECK` ([Chapter 4](04-dockerfile-deep-dive.md#511-healthcheck-tell-docker-how-to-test-the-app)), but configured per service so you can add one to images you didn't build (Postgres, Redis) and override the one baked into your image.

```yaml
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s          # how often to run the check
      timeout: 3s            # a check slower than this counts as failed
      retries: 3             # consecutive failures before "unhealthy"
      start_period: 20s      # grace period: failures here don't count
      start_interval: 2s     # check frequency during the start period (Engine 25+)
```

The `test` field forms:

| Form | Meaning |
|---|---|
| `["CMD", "pg_isready", "-U", "postgres"]` | Run the program directly (no shell) |
| `["CMD-SHELL", "pg_isready -U postgres \|\| exit 1"]` | Run through the container's shell (use when you need `$VAR`, pipes or `\|\|`) |
| `"pg_isready -U postgres"` (string) | Same as `CMD-SHELL` |
| `["NONE"]` or `disable: true` | Turn off a healthcheck baked into the image |

**Escaping `$`:** Compose interpolates `${VAR}` in the YAML *before* the container exists. If you want the **container's** shell to expand a variable (`$POSTGRES_USER` as seen inside the container), write **`$$POSTGRES_USER`** so Compose passes a literal `$POSTGRES_USER` through. That's why the examples use `$${POSTGRES_USER}`.

Ready-made checks:

| Service | Healthcheck `test` |
|---|---|
| PostgreSQL | `["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]` |
| Redis | `["CMD", "redis-cli", "ping"]` (expects `PONG`) |
| MySQL/MariaDB | `["CMD", "mysqladmin", "ping", "-h", "localhost"]` |
| HTTP service with curl | `["CMD", "curl", "-fsS", "http://localhost:8000/health"]` (`-f` fail on HTTP errors, `-sS` quiet but show errors) |
| HTTP service on slim Python (no curl) | `["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]` |
| nginx (alpine) | `["CMD", "wget", "-qO-", "http://127.0.0.1/"]` (BusyBox wget is present on Alpine) |

See the result:

```bash
docker compose ps
```

```
NAME               IMAGE             COMMAND                  SERVICE   CREATED          STATUS                    PORTS
notes-app-db-1     postgres:18       "docker-entrypoint.s..."  db        1 minute ago     Up 1 minute (healthy)     5432/tcp
notes-app-redis-1  redis:7-alpine    "docker-entrypoint.s..."  redis     1 minute ago     Up 1 minute (healthy)     6379/tcp
notes-app-web-1    notes-web:dev     "uvicorn app.main:ap..."  web       1 minute ago     Up 50 seconds (healthy)   0.0.0.0:8000->8000/tcp
```

Remember: **a failing healthcheck does not restart the container** in plain Docker or Compose. It only changes the displayed status and gates `depends_on: service_healthy` (and orchestrators/load balancers that read it).

---

## Restart policies

What should Docker do when a container **exits** or the Docker daemon (or machine) restarts?

```yaml
    restart: unless-stopped
```

| Value | Behaviour |
|---|---|
| `no` (default) | Never restart automatically |
| `always` | Always restart; also starts at daemon boot, even if you stopped it manually (it restarts after a daemon restart) |
| `unless-stopped` | Like `always`, but if **you** stopped it explicitly (`docker compose stop`), it stays stopped across daemon restarts |
| `on-failure[:max-retries]` | Restart only if the exit code is non-zero, optionally limited to N attempts (e.g. `on-failure:5`) |

Guidance:

- Long-running services in production: `unless-stopped`.
- One-shot jobs (migrations): `no` or `on-failure:3`.
- Development: usually `no`, so a crash is visible and doesn't loop silently.
- For restarts to survive a **machine reboot**, the Docker service itself must start at boot (on Linux: `sudo systemctl enable docker`; Docker Desktop: "Start Docker Desktop when you log in").
- A crash-looping container restarts with increasing delay; check `docker compose logs`.
- Restart policies are triggered by the **process exiting**, not by a failed healthcheck.

---

## command and entrypoint

```yaml
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--reload"]
    entrypoint: ["/entrypoint.sh"]
```

- `command` overrides the image's `CMD`; `entrypoint` overrides its `ENTRYPOINT`. The same exec-form vs shell-form rules apply ([Chapter 4](04-dockerfile-deep-dive.md#6-cmd-vs-entrypoint-exec-form-vs-shell-form)): prefer list syntax so the program is PID 1 and receives signals.
- Overriding `entrypoint` also **clears the image's `CMD`** unless you set `command` too.
- To use a variable *inside the container's shell*, call a shell: `command: ["sh", "-c", "exec uvicorn app.main:app --port $${PORT:-8000}"]` (note `$$` to avoid Compose interpolating it first).
- Multi-line shell scripts with YAML block scalars:

  ```yaml
  command:
    - sh
    - -c
    - |
      echo "starting"
      exec uvicorn app.main:app --host 0.0.0.0
  ```
  (`|` starts a multi-line literal block in YAML.)

---

## Resource limits

Without limits, one runaway container can consume all your CPU and RAM and take down the host (or, on Docker Desktop, the whole VM).

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

- **`limits`** are enforced via Linux cgroups ([Chapter 1](01-what-is-docker.md)). Exceeding the **memory** limit gets the process **OOM-killed** (exit code **137**, which is 128 + signal 9). `docker inspect` shows `OOMKilled: true`.
- **`cpus`** throttles rather than kills.
- **`reservations`** are minimums used for placement/scheduling; on a single host they're mostly advisory.
- The `deploy` section is part of the **Deploy Specification**. It was originally for Swarm, but Compose V2 applies `deploy.resources` limits to regular `docker compose up` too.

**Legacy/short forms** still valid in the Compose Specification:

```yaml
    mem_limit: 512m
    mem_reservation: 128m
    cpus: 1.0
    pids_limit: 200          # cap the number of processes (fork-bomb protection)
```

If you set both a short form and the `deploy` form, the values **must match**, or Compose will complain; pick one style. I recommend `deploy.resources` for forward compatibility.

**GPUs** (preview; full treatment in [Chapter 8](08-docker-for-ai-and-ml.md)):

```yaml
    gpus: all                               # short form
    # or the long form:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

Needs an NVIDIA GPU, drivers on the host, and the NVIDIA Container Toolkit.

Check live usage: `docker stats` (or `docker compose stats`).

---

## Profiles

**Profiles** let you keep optional services (debug tools, admin UIs, test runners, seed jobs) in the same file without starting them every time.

```yaml
services:
  web:
    build: .                    # no profiles: always started

  adminer:                      # a web UI for databases
    image: adminer
    ports: ["8081:8080"]
    profiles: [debug]           # only starts if the "debug" profile is active

  seed:
    build: .
    command: ["python", "-m", "app.seed"]
    profiles: [tools]
```

```bash
docker compose up -d                          # starts web only
docker compose --profile debug up -d          # web + adminer
docker compose --profile debug --profile tools up -d
COMPOSE_PROFILES=debug,tools docker compose up -d    # same, via environment variable
docker compose run --rm seed                  # explicitly naming a service auto-activates its profile
```

- A service with **no** `profiles` always starts; a service with `profiles` starts only when one of them is active.
- Naming a profiled service explicitly on the command line (like `run seed` or `up adminer`) enables its profile automatically.
- Be careful with dependencies: an always-on service that `depends_on` a profiled service will fail unless that profile is active or you mark the dependency `required: false`.
- `docker compose --profile "*" up` activates all profiles.

---

## Multiple Compose files and overrides

Real projects have different needs per environment (dev, test, prod). Rather than copy-pasting whole files, you keep a **base** file and layer **override** files over it.

### Automatic override: `compose.override.yaml`

If both `compose.yaml` and `compose.override.yaml` exist and you run plain `docker compose up`, Compose **automatically merges both** (override on top). Convention: the base file holds what's shared, and the override file holds **developer conveniences** (bind mounts, published ports, debug flags).

### Explicit files with `-f`

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d
```

- `-f` (`--file`) is repeatable; files merge **left to right**, later files win.
- **Important:** when you pass `-f` yourself, `compose.override.yaml` is **not** loaded automatically; only the files you listed are. This is how you run "base + prod" without the dev override.
- Set a default with an environment variable (often in `.env`): `COMPOSE_FILE=compose.yaml:compose.prod.yaml` (separator is `:` on Linux/macOS and `;` on Windows; `COMPOSE_PATH_SEPARATOR` can override it).

### How merging works

Compose merges **by element type** (this is the part to remember):

| Kind of field | Examples | Merge behaviour |
|---|---|---|
| Scalars (single values) | `image`, `restart`, `user` | The later file **replaces** |
| Mappings (key: value) | `environment`, `labels`, `build.args` | **Merged key by key**; later wins on the same key |
| Lists that are sets of items | `ports`, `expose`, `dns`, `tmpfs` | **Appended** (items from both files) |
| `volumes` | | Merged by **mount target path**; same target in a later file replaces the earlier entry |
| `command`, `entrypoint`, `healthcheck.test` | | **Replaced** wholesale, not appended |

Two special YAML tags (Compose 2.24+) give you control when the default isn't what you want:

```yaml
services:
  web:
    ports: !reset []                  # clear ports inherited from the earlier file
    environment: !override            # replace the entire mapping instead of merging
      ONLY_THIS: "yes"
```

- `!reset`: remove the inherited value (use `[]` for lists, `null` for scalars).
- `!override`: replace instead of merge.

**Always verify** with `docker compose -f a.yaml -f b.yaml config`: it prints the merged result so you see exactly what you'll get.

### A typical layout

```
compose.yaml            # shared base: services, healthchecks, volumes, networks
compose.override.yaml   # dev: auto-loaded; ports, watch, --reload, dev passwords
compose.prod.yaml       # prod: restart policies, limits, secrets, no dev conveniences
compose.test.yaml       # CI: test runner service, ephemeral DB
.env.example            # documented fake values
```

The full worked example is in the [tutorial](#stage-4-dev-vs-prod-with-override-files).

---

## extends and include

Two ways to reuse Compose definitions across files.

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

- `service` is required; `file` is optional (omit it to extend a service in the same file).
- It merges like override files: **mappings merge key by key, sequences combine with the inherited items first, scalars from the extending service win.**
- Circular references are an error.
- Resources the service references (networks, volumes, ...) are **not imported automatically**; you must declare them in the extending file.
- `extends` is not supported by `docker stack deploy` (Swarm).

### `include`: import whole Compose files as sub-applications

```yaml
# compose.yaml
include:
  - ./infra/compose.yaml          # e.g. a shared database + cache stack
  - path:
      - ./third-party/compose.yaml
      - ./third-party.override.yaml   # a per-include override file

services:
  web:
    build: .
    depends_on: [db]              # "db" comes from the included file
```

- Each included file is loaded **as its own Compose model with its own project directory**, so relative paths inside it (build contexts, bind mounts, `env_file`s) resolve relative to *that file*. This is the main advantage over `-f` merging (and over `extends`), where relative paths get confusing.
- After loading, the included resources are copied into your model, so you can `depends_on` them.
- `include` is **recursive** (an included file can include more).
- **Conflicts are errors:** if an included file defines a resource that clashes with one in your file, Compose fails. To customize an included service, either list an override file under that `include` entry (as shown) or use your global `compose.override.yaml`.
- Docker's documentation also describes remote sources (Git repositories and OCI artifacts) for `include`; I haven't verified the exact URL syntax for each, so check the `include` reference before using remote sources.

Use `include` for "team A owns the shared infra stack, team B includes it"; use override files for "same app, different environment".

---

## YAML anchors and extension fields

When several services share many settings, avoid copy-paste with YAML **anchors** (`&name` defines a reusable block, `*name` references it, `<<:` merges it) combined with Compose **extension fields** (top-level keys starting with `x-`, ignored by Compose).

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

- `x-python-service:` is a top-level extension field: Compose ignores its name but YAML anchors still work inside the file.
- `&python-service` names the block; `*python-service` pastes it; `<<:` is YAML's "merge key" that copies the anchor's keys into the current mapping (keys you write explicitly win).
- Anchors work **within one file only** (YAML feature), not across `-f` files; use `extends`/`include`/overrides for cross-file reuse.
- `docker compose config` shows the expanded result.

---

## Secrets and configs

### Why not just environment variables?

Environment variables are visible through `docker inspect`, often leak into logs and crash reports, and are inherited by every child process. For sensitive values (database passwords, API keys, TLS keys), the recommended pattern is to deliver them as **files** mounted read-only into the container (conventionally under `/run/secrets/`), which the app reads at startup.

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
    file: ./secrets/db_password.txt        # source of the secret on the host
```

- Top-level `secrets:` declares where each secret comes from: `file:` (a path) or `environment:` (the name of a variable in your shell/`.env`).
- Service-level `secrets:` grants the service access. In the short form shown, the secret appears as a read-only file at **`/run/secrets/<name>`**.
- The official Postgres, MySQL and MariaDB images support the `*_FILE` convention (`POSTGRES_PASSWORD_FILE`, ...) precisely for this. Many other images do too; check their documentation.
- Create the file without committing it: add `secrets/` to `.gitignore`. Generate a random value, for example `openssl rand -base64 24 > secrets/db_password.txt` (Git Bash/WSL/macOS/Linux), or in PowerShell `[Convert]::ToBase64String((1..24 | ForEach-Object { Get-Random -Maximum 256 })) | Set-Content -NoNewline secrets/db_password.txt`.
- Long syntax allows `source`, `target`, `uid`, `gid`, `mode` (default `0444`). Per the specification, `uid`, `gid` and `mode` apply only to secrets sourced from `environment`; for `file:` sources they are silently ignored, which means the file's **host permissions** must allow the container's user to read it.
- **Honest limitation:** Compose secrets are a *delivery mechanism*, not a vault. The source file is still plaintext on the host, and anyone who can run `docker exec` can read it. In production, back it with a real secrets manager (Vault, cloud secret services) or Swarm/Kubernetes secrets ([Chapter 10](10-security.md), [Chapter 11](11-production-and-enterprise.md)).

### Build secrets

The same `secrets:` top-level definition can feed **image builds** (`RUN --mount=type=secret`, [Chapter 4](04-dockerfile-deep-dive.md#17-build-secrets-never-bake-secrets-into-layers)):

```yaml
services:
  api:
    build:
      context: .
      secrets:
        - pip_token
secrets:
  pip_token:
    environment: PIP_TOKEN        # read from your shell variable at build time
```

### Configs

**Configs** are the same mechanism for **non-sensitive** configuration files (an nginx config, an app settings file). They're mounted read-only into the container:

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

Why use configs instead of a bind mount? They express intent ("this is configuration"), work consistently with Swarm/Kubernetes-style platforms, and don't depend on the service having the host directory layout. (Recent Compose versions also allow `content:` to define a config inline in the YAML, or `environment:` as the source; verify support in your version with `docker compose config`.)

---

## docker compose watch: hot reload for development

Rebuilding the image on every code change is slow. Bind-mounting your source is fast but has the cross-OS issues from [Chapter 5](05-storage-and-networking.md#a8-windows-and-wsl-path-pitfalls) (slow file systems, missing change events, ownership). **Compose Watch** is a third way: Compose watches your files on the host and **pushes changes into the running container** (or rebuilds) automatically, based on rules you write under `develop.watch`.

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

The three documented actions:

| Action | What happens when a watched file changes | Use for |
|---|---|---|
| `sync` | The changed file is **copied into the container** at `target` | Source code with a framework that hot-reloads itself (uvicorn `--reload`, Vite, Next.js dev) |
| `rebuild` | The image is **rebuilt** and the container replaced (like `up --build <service>`) | Dependency manifests (`requirements.txt`, `package.json`), compiled languages |
| `sync+restart` | Sync the file **and restart the container** | Config files read only at startup (e.g. `nginx.conf`) |

(Newer Compose versions also describe a `sync+exec` action that runs a command after syncing. I did not verify its details; check the current "Use Compose Watch" docs if you need it.)

Attributes (from Docker's documentation):

- `action`: required, one of the above.
- `path`: required; a file or directory to watch, **relative to the project directory**. Directories are watched recursively. Glob patterns are not supported in `path`.
- `target`: where inside the container the path maps to (for `sync` actions).
- `ignore`: extra patterns to skip; they're **relative to `path`**, and your `.dockerignore` rules apply as well.
- `initial_sync`: for `sync+...` actions, makes sure files are up to date when the watch session starts.

Run it:

```bash
docker compose watch              # start services, watch files; logs and sync events are separated
docker compose up --watch         # same idea, with the regular `up` log output
docker compose watch --prune=false   # keep superseded images after rebuilds (they're pruned by default)
```

Prerequisites and gotchas:

- The container image needs the `stat`, `mkdir` and `rmdir` binaries (distroless/scratch images lack them).
- The container's user must be able to **write to the target path**. If your Dockerfile runs as a non-root user, copy the app with `COPY --chown=app:app` so that user owns the files ([Chapter 4](04-dockerfile-deep-dive.md#15-running-as-a-non-root-user)). Permission errors here are the top cause of "watch syncs nothing".
- For `sync` to be useful the process must **reload itself**: use `uvicorn --reload` (Python), `nodemon`/Vite/Next dev server (Node), `air` (Go), Spring DevTools (Java).
- `watch` doesn't replace production images; it's strictly a dev feature.

---

## Project names

Compose groups everything under a **project name** and prefixes resource names with it:

| Resource | Name pattern | Example |
|---|---|---|
| Container | `<project>-<service>-<index>` | `notes-app-web-1` |
| Network | `<project>_<network>` | `notes-app_default` |
| Volume | `<project>_<volume>` | `notes-app_pgdata` |

Project name resolution, highest priority first (per Docker's documentation):

1. `-p` / `--project-name` flag
2. `COMPOSE_PROJECT_NAME` environment variable
3. `name:` top-level key in the Compose file
4. The directory name containing the Compose file

(Names are lowercased and restricted to letters, digits, `-` and `_`.)

Why it matters:

- **Two copies of the same app** (staging and test on one host) need different project names: `docker compose -p myapp-test up -d`.
- **Renaming the folder silently changes the project name**, so Compose creates *new, empty volumes* and "loses" your database (the old data is still there under the old name). Setting `name:` at the top of the file avoids this.
- `docker compose ls` shows all projects Compose knows about on this machine.

---

## Compose commands reference

All commands operate on the project defined by the Compose file(s) in the current directory (or those given with `-f`/`-p`). Global flags go **before** the subcommand: `docker compose -f x.yaml -p demo up -d`.

### Global options (from `docker compose --help`)

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
| `--compatibility` | Backward-compatibility mode for old V1 conventions (container name separators etc.) |

`--dry-run` is great for learning: `docker compose --dry-run up -d` prints the plan and changes nothing.

### Lifecycle commands

#### `docker compose up`

Creates (if needed), (re)creates (if configuration or image changed) and starts the services, creating networks and volumes first.

```bash
docker compose up                # foreground: streams all logs; Ctrl+C stops the containers
docker compose up -d             # detached: starts in the background
docker compose up -d --build     # rebuild images first
docker compose up -d web         # only 'web' (and what it depends on)
```

Useful flags (all from `docker compose up --help` on Compose 5.6.1):

| Flag | Meaning |
|---|---|
| `-d, --detach` | Run in the background |
| `--build` | Build images before starting |
| `--no-build` | Never build, even if policy says to |
| `--pull always\|missing\|never` | Pull images before running |
| `--force-recreate` | Recreate containers even if nothing changed |
| `--no-recreate` | Don't recreate existing containers |
| `--no-deps` | Don't start linked/dependent services |
| `--wait` | Wait for services to be running/**healthy** (implies `-d`); perfect in CI and scripts |
| `--wait-timeout <s>` | Give up waiting after this many seconds |
| `--scale web=3` | Run N containers of a service |
| `--remove-orphans` | Remove containers for services no longer in the file |
| `-V, --renew-anon-volumes` | Recreate anonymous volumes instead of reusing old ones |
| `--abort-on-container-exit` | Stop everything when any container stops (not with `-d`) |
| `--abort-on-container-failure` | Stop everything if any container exits with an error |
| `--exit-code-from <service>` | Return that service's exit code (implies abort-on-exit); used for CI test runs |
| `-w, --watch` | Start and watch files (see [watch](#docker-compose-watch-hot-reload-for-development)) |
| `--no-color`, `--no-log-prefix`, `--timestamps` | Log formatting |
| `-t, --timeout <s>` | Shutdown timeout when stopping/recreating |
| `-y, --yes` | Auto-answer prompts (non-interactive) |

**What `up` does, step by step:** reads and merges your files -> interpolates variables -> creates the network(s) and volumes -> builds missing images (when `build` is set and no image exists, or `--build`) / pulls missing ones -> creates and starts containers in dependency order, waiting for `depends_on` conditions. It's idempotent: run it again and unchanged services are left alone ("Running"); services whose configuration (or image) changed are **recreated**.

#### `docker compose down`

Stops and removes the **containers and networks** of the project.

```bash
docker compose down              # remove containers + networks; KEEP named volumes
docker compose down -v           # ALSO delete named volumes + anonymous volumes. DATA LOSS!
docker compose down --rmi local  # also remove images built by this project without a custom tag
docker compose down --remove-orphans
```

| Flag | Meaning |
|---|---|
| `-v, --volumes` | Remove named volumes declared in the file **and** anonymous volumes attached to containers |
| `--rmi local\|all` | Remove images: `local` = ones without a custom tag; `all` = every image used by services |
| `--remove-orphans` | Remove containers for services not in the file |
| `-t, --timeout` | Seconds to wait for containers to stop |

> **Warning (the big one):** `docker compose down -v` **permanently deletes your database volume**. Never type `-v` from muscle memory in an environment with data you care about. Plain `down` is safe for data. Use `stop` if you only want to pause without deleting containers.

#### `docker compose start` / `stop` / `restart` / `pause` / `unpause` / `kill`

```bash
docker compose stop              # stop containers (SIGTERM, then SIGKILL after the grace period); keep them
docker compose start             # start stopped containers again
docker compose restart           # restart containers (stop + start)
docker compose restart web       # one service
docker compose pause web         # freeze processes (SIGSTOP-like cgroup freeze), keeps memory
docker compose unpause web
docker compose kill web          # immediate SIGKILL (-s SIGNAL to send another signal)
docker compose rm                # remove STOPPED containers (-f no prompt, -s stop first, -v remove anonymous volumes)
```

Important: **`restart` does not re-read your Compose file or `.env`**. After changing configuration run `docker compose up -d` so Compose recreates the affected containers.

### Inspection commands

#### `docker compose ps`

```bash
docker compose ps                # containers of this project
docker compose ps -a             # include stopped containers
docker compose ps --services     # just service names
docker compose ps --format json  # JSON output
docker compose ps --status running --status exited
```

#### `docker compose logs`

```bash
docker compose logs              # all services, all history
docker compose logs -f           # follow (stream) new lines
docker compose logs -f --tail 100 web     # last 100 lines, then follow, one service
docker compose logs -t web       # with timestamps
docker compose logs --since 10m  # last ten minutes
docker compose logs --no-log-prefix web   # without the "web-1  |" prefix
```

Each line is prefixed with the container name and coloured per service, so interleaved output from many services stays readable.

#### `docker compose top`, `images`, `port`, `ls`, `events`, `volumes`, `stats`

```bash
docker compose top               # processes running inside each container
docker compose images            # images used by the project's containers
docker compose port web 8000     # which host address maps to web's container port 8000
docker compose ls                # list all Compose projects (-a includes stopped)
docker compose events            # real-time event stream
docker compose volumes           # list volumes used by the project
docker compose stats             # live CPU/memory/network per container (like docker stats)
```

#### `docker compose config`

Covered earlier; renders the merged/interpolated model. Also `--services`, `--profiles`, `--images`, `--quiet`, `--format json`, and `--no-interpolate` (show `${VARS}` unexpanded).

### Running commands

#### `docker compose exec`

Run a command **inside an already-running container** of a service.

```bash
docker compose exec web sh                    # interactive shell
docker compose exec db psql -U notes -d notes   # open a Postgres prompt
docker compose exec -u root web id            # as root
docker compose exec -T web python -c "print(1)"   # no TTY (needed in CI/scripts and for piping)
docker compose exec -e FOO=bar web env
docker compose exec --index 2 web sh          # container #2 if scaled
```

Flags: `-d` detached, `-e` env var, `-u` user, `-w` working dir, `-T` disable pseudo-TTY allocation, `--index` pick a replica. (In a TTY, `exec` allocates one by default; in pipelines/CI you need `-T` or you get "the input device is not a TTY".)

#### `docker compose run`

Start a **new, one-off container** from a service's definition and run a command in it (your tests, a migration, a shell). It also starts the service's dependencies.

```bash
docker compose run --rm web pytest -q          # run tests, delete the container afterwards
docker compose run --rm web python -m app.migrate
docker compose run --rm --no-deps web sh       # don't start db/redis
docker compose run --rm -e DEBUG=1 web sh
docker compose run --service-ports web         # publish the service's ports (off by default for run!)
```

`--rm` removes the one-off container when done (otherwise they pile up as `project-web-run-abc123`). `run` does **not** publish ports by default (so it doesn't conflict with the already-running service); add `--service-ports` if you need them.

**`exec` vs `run`:** `exec` = enter a container that's already running. `run` = launch a fresh container for a task.

### Build and image commands

```bash
docker compose build                 # build all services that have `build:`
docker compose build web             # one service
docker compose build --no-cache      # ignore cache
docker compose build --pull          # re-pull the base image
docker compose build --build-arg PYTHON_VERSION=3.12 web
docker compose pull                  # pull images for services that use `image:`
docker compose push                  # push built images to their registry
```

Also `docker compose images`, and `docker compose publish` (publish the Compose application itself as an OCI artifact; exists in recent releases; I have not verified its details, see `docker compose publish --help`).

### Copying files

```bash
docker compose cp web:/app/output.csv ./output.csv     # container -> host
docker compose cp ./settings.toml web:/app/settings.toml   # host -> container
```

Format `SERVICE:PATH` for the container side. Useful for pulling logs, dumps or generated artifacts out without volumes.

### Other commands (from `docker compose --help` on 5.6.1)

| Command | Purpose |
|---|---|
| `attach` | Attach your terminal to a running container's stdin/stdout |
| `commit` | Create an image from a service container's changes (rarely a good idea; prefer Dockerfiles) |
| `create` | Create containers without starting them |
| `export` | Export a service container's filesystem as a tarball |
| `scale` | Scale services (e.g. `docker compose scale web=3`) |
| `wait` | Block until the (selected) services stop |
| `bridge` | Convert Compose files to another model (experimental-style management command) |
| `version` | Print the Compose version |

### Cheat sheet

```bash
docker compose up -d --build        # build + start everything in background
docker compose ps                   # what's running?
docker compose logs -f --tail 50 web   # watch one service
docker compose exec web sh          # shell into it
docker compose run --rm web pytest  # one-off task
docker compose config               # what will Compose actually do?
docker compose restart web          # restart (does NOT re-read config)
docker compose up -d                # apply config changes
docker compose stop                 # pause the stack, keep everything
docker compose down                 # remove containers/networks, keep data
docker compose down -v              # remove everything INCLUDING DATA
```

---

## Progressive tutorial

We'll build up from one container to a realistic stack with dev and prod configurations. Create a folder, `notes-app/`. Everything here is a throwaway learning setup. All passwords are fake dev-only values.

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

**`requirements.txt`**

```
fastapi
uvicorn[standard]
```

(In real projects pin versions, [Chapter 4](04-dockerfile-deep-dive.md#13-reproducibility-tags-digests-pinning).)

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

**`Dockerfile`**

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

(Explained line by line in [Chapter 4](04-dockerfile-deep-dive.md#10-ordering-instructions-for-cache-efficiency-before-and-after).)

**`compose.yaml`**

```yaml
name: notes-app

services:
  web:
    build: .
    ports:
      - "8000:8000"
```

Run it:

```bash
docker compose up --build
```

- `up`: create and start everything.
- `--build`: build the image first (it doesn't exist yet).
- Attached mode: logs stream to your terminal; stop with Ctrl+C.

Output (abbreviated):

```
[+] Building 12.3s (10/10) FINISHED
[+] Running 2/2
 ✔ Network notes-app_default  Created
 ✔ Container notes-app-web-1  Created
Attaching to web-1
web-1  | INFO:     Started server process [1]
web-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

Notice what Compose created for free: a **network** (`notes-app_default`) and a **container** (`notes-app-web-1`). In another terminal:

```bash
curl http://localhost:8000
# {"message":"Hello from Compose"}
docker compose ps
```

Stop with Ctrl+C, then `docker compose down` to remove the container and network.

### Stage 2: web + Redis (a hit counter)

Add the Redis client: `requirements.txt` becomes

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

What's new, line by line:

| Line | Meaning |
|---|---|
| `REDIS_URL: redis://redis:6379/0` | The hostname is **`redis`, the service name**. Docker's DNS on the project network resolves it ([Chapter 5](05-storage-and-networking.md#b2-default-bridge-vs-user-defined-bridge)). Never `localhost`. `6379` is Redis's container port; `/0` selects Redis database 0. |
| `depends_on: redis: condition: service_healthy` | Don't start `web` until Redis's healthcheck passes. |
| `image: redis:7-alpine` | An official image. No `build` needed. No `ports:` either: only `web` needs to reach Redis, so nothing is published. |
| `healthcheck.test: ["CMD","redis-cli","ping"]` | Redis answers `PONG` when ready. Check every 5 s, 3 s timeout, up to 5 failures. |

```bash
docker compose up -d --build
curl http://localhost:8000      # {"message":"Hello from Compose","hits":1}
curl http://localhost:8000      # ... "hits":2
docker compose restart web
curl http://localhost:8000      # ... "hits":3   (the count lives in Redis, not in web)
docker compose down
docker compose up -d
curl http://localhost:8000      # "hits":1 again!  Redis data lived in the removed container.
```

That last line demonstrates [Chapter 5](05-storage-and-networking.md#a1-why-data-disappears-the-containers-writable-layer): without a volume, data dies with the container. Stage 3 fixes that.

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

**`requirements.txt`**

```
fastapi
uvicorn[standard]
redis
psycopg[binary]
```

**`app/main.py`** (the full version we'll keep):

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
    # Runs once at startup: create the table if it doesn't exist yet.
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

Notes: `read_secret` is the pattern that lets the same code work with a plain variable in dev and a mounted secret file in prod (`DB_PASSWORD_FILE`). The parameterized query (`%s` with a tuple) prevents SQL injection; never build SQL with f-strings.

**`.env`** (not committed; `.env.example` has the same keys with placeholder values):

```
POSTGRES_DB=notes
POSTGRES_USER=notes
POSTGRES_PASSWORD=dev-only-change-me
```

**`compose.yaml`** (stage 3, single file, dev-friendly):

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

New ideas, explained:

| Setting | Explanation |
|---|---|
| `${POSTGRES_DB:-notes}` | Interpolation with a default: use the `.env`/shell value, else `notes`. |
| `${POSTGRES_PASSWORD:?Set ...}` | **Required**: `docker compose up` fails with that message if it's unset, instead of starting a database with an empty password. |
| `DB_HOST: db` | The Postgres **service name** is the hostname. |
| `pgdata:/var/lib/postgresql` | Named volume. The path is correct for **Postgres 18+**; for 17 and earlier use `/var/lib/postgresql/data` ([Chapter 5](05-storage-and-networking.md#a2-volumes)). |
| `$${POSTGRES_USER}` | Double `$$` so the **container's shell** expands the variable, not Compose. |
| `command: ["redis-server","--appendonly","yes"]` | Turns on Redis's append-only-file persistence so data in `/data` survives restarts. |
| `redisdata:/data` | The Redis image keeps its data in `/data`. |
| top-level `volumes:` | Declares both named volumes (created as `notes-app_pgdata`, `notes-app_redisdata`). |
| `web.healthcheck` | Overrides/defines the web service's own check. It calls `/health`, which in turn tests both Redis and Postgres, so "healthy" means the whole path works. |

Run and test:

```bash
docker compose up -d --build
docker compose ps
```

Wait until all three show `(healthy)`. Then:

```bash
curl -X POST "http://localhost:8000/notes?body=first%20note"
# {"id":1,"body":"first note"}
curl http://localhost:8000/notes
# [{"id":1,"body":"first note"}]
curl http://localhost:8000/hits
# {"hits":1}
```

**Prove persistence:**

```bash
docker compose down            # remove containers and network, KEEP volumes
docker compose up -d
curl http://localhost:8000/notes     # the note is still there
curl http://localhost:8000/hits      # {"hits":2}
```

**Look at what Compose created:**

```bash
docker volume ls --filter name=notes-app
docker network ls --filter name=notes-app
docker compose logs --tail 20 db
```

To start fresh deliberately (and understand the cost): `docker compose down -v` deletes `pgdata` and `redisdata`.

Common stage 3 failure modes:

| Symptom | Cause |
|---|---|
| `required variable POSTGRES_PASSWORD is missing a value` | You haven't created `.env`. That's the `:?` guard working. |
| web restarts with `connection refused` to db | You omitted `condition: service_healthy`, or the app starts before DB is ready. |
| Postgres refuses to start with a complaint about the data directory, or the database is empty after `down`/`up` | The volume mount path doesn't match the image's major version (see the Postgres 18 note in [Chapter 5](05-storage-and-networking.md#a2-volumes)). Read the exact message with `docker compose logs db`. |
| Changing `POSTGRES_PASSWORD` in `.env` has no effect | The password is only applied when the volume is **first initialized**. Change it with SQL (`ALTER USER`) or recreate the volume (data loss in dev only). |

### Stage 4: dev vs prod with override files

Now we split by environment, using the principle: **base = what is the same everywhere; override = what differs.**

**`Dockerfile`** (a production-minded version, from [Chapter 4](04-dockerfile-deep-dive.md#23-a-full-production-grade-dockerfile-for-fastapi), with non-root user so `watch` and `read_only` are meaningful):

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

**`compose.yaml`: the shared base** (no ports, no passwords, no dev tools):

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

Both `build: .` and `image:` are set, so in dev Compose builds and tags the image, and in prod the same name is pulled from the registry (replace `your-user` with your own registry path).

**`compose.override.yaml`: development** (auto-loaded by plain `docker compose up`):

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
      - "127.0.0.1:5432:5432"      # lets you use a GUI/psql from your laptop
```

What the override does: publishes the web port; supplies the dev password through `.env`; swaps the command for `--reload`; defines `watch` rules; publishes Postgres **on loopback only** so your DB client can connect but the network can't. Because `environment` is a mapping, it **merges** with the base's (base keys stay, new keys are added); `command` is **replaced**; `ports` are **appended**.

Dev workflow:

```bash
docker compose watch          # builds, starts, and syncs ./app edits into the container live
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
    internal: true                   # DB and cache can't reach the internet
```

What changes in prod and why:

| Change | Reason |
|---|---|
| `restart: unless-stopped` | Survive crashes and host reboots |
| `init: true` | Proper signal handling and zombie reaping ([Chapter 4](04-dockerfile-deep-dive.md#16-pid-1-signals-tini-and---init)) |
| `127.0.0.1:8000:8000` | Only a reverse proxy on the same host (nginx/Caddy, [Chapter 5](05-storage-and-networking.md#b12-reverse-proxies-in-front)) is exposed to the internet |
| no DB port published | The database is reachable only by `web`, on the internal network |
| secrets via files | No plaintext password in the environment or in the Compose file |
| `read_only` + `tmpfs` + `cap_drop` + `no-new-privileges` | Container hardening ([Chapter 10](10-security.md)) |
| resource limits | One service can't starve the host |
| log rotation | Prevents unbounded log files filling the disk |
| `internal: true` backend | Defense in depth |
| `pull_policy: always` | Deploy exactly what CI published |

Prepare the secret (once, on the prod host; never in Git; `.gitignore` must contain `secrets/`):

```bash
mkdir -p secrets
openssl rand -base64 24 > secrets/db_password.txt
chmod 644 secrets/db_password.txt     # the non-root container user (UID 10001) must be able to read it
```

(`644` readable by all users on the host is acceptable only on a single-purpose server; because the file is bind-mounted, the container sees the host's permissions. Weigh this against your threat model or move to a proper secrets manager.)

Run prod (note: `-f` given, so the dev override is **not** loaded):

```bash
export TAG=1.4.2
docker compose -f compose.yaml -f compose.prod.yaml config --quiet     # validate; non-zero exit on error
docker compose -f compose.yaml -f compose.prod.yaml pull
docker compose -f compose.yaml -f compose.prod.yaml up -d --no-build --wait
docker compose -f compose.yaml -f compose.prod.yaml ps
```

- `config --quiet`: validate the merged files without printing them.
- `pull`: fetch `ghcr.io/your-user/notes-web:1.4.2` and the other images.
- `up -d --no-build --wait`: start in the background, **never** build on the server, and block until everything is running/healthy (exit status tells your deploy script if it worked).

To avoid typing `-f` twice every time, put `COMPOSE_FILE=compose.yaml:compose.prod.yaml` in the shell/CI environment (use `;` as separator on Windows).

> **Note on the first run in prod:** the database starts with `POSTGRES_PASSWORD_FILE` only on **first initialization of the volume**. If you previously ran dev with `dev-only-change-me` in the *same* `pgdata` volume, the old password is still the real one. Use a separate project name or fresh volume for prod (`-p notes-prod`).

### What you just learned, in one view

```
 Stage 1  one service             build, ports
 Stage 2  two services            service-name DNS, depends_on, healthcheck
 Stage 3  stateful stack          named volumes, .env interpolation, required vars
 Stage 4  environments            override files, watch, secrets, limits, networks, hardening
```

---

## Compose vs Swarm vs Kubernetes

| | **Docker Compose** | **Docker Swarm** | **Kubernetes** |
|---|---|---|---|
| What it is | Define/run multi-container apps, mainly on **one host** | Docker's built-in **clustering** mode (multiple hosts) | The industry-standard **container orchestration platform** (many hosts) |
| Config format | `compose.yaml` | Compose-format files via `docker stack deploy` | YAML manifests (Deployments, Services, Ingress...), Helm, Kustomize |
| Multi-host | No (one Docker engine; you can target a remote one with a Docker context, still just one host) | Yes | Yes |
| Self-healing / reschedule on node failure | No (restart policy on the same host only) | Yes | Yes |
| Rolling updates, rollbacks | Basic (recreate); no built-in zero-downtime | Yes | Yes (first-class) |
| Autoscaling | No (manual `--scale`) | Manual scaling | Yes (HPA, cluster autoscaler) |
| Service discovery / load balancing | Docker DNS on one host | Built-in overlay + routing mesh | Services, Ingress/Gateway |
| Secrets/config | Files from the host | Swarm secrets (encrypted at rest in the cluster) | Secrets, ConfigMaps, external secret operators |
| Learning curve | Low | Medium | High |
| Operational cost | Minimal | Low-medium | High (or pay for a managed service) |
| Ecosystem | Huge for local/dev | Small and mostly stagnant | Enormous |

Notes on status: Swarm mode remains part of Docker Engine and is documented by Docker; however, I did not verify its current level of investment/roadmap, and the industry's gravity has clearly moved to Kubernetes, so check current Docker announcements before betting a new project on Swarm. Compose files can also be used with Swarm (`docker stack deploy`), but some keys differ in meaning (`deploy` section applies there, `build` doesn't).

### When Compose is enough in production

Compose is a perfectly **legitimate production tool** when:

- Your app runs on **one server** (a VM) and a short restart during deploys is acceptable
- Traffic is modest, and you don't need autoscaling or multi-AZ resilience
- It's an internal tool, a startup MVP, a side project, a demo, a self-hosted service, a single-tenant deployment per customer, or a build/CI environment
- Your team is small and operational simplicity beats cluster features

A sound Compose-in-production setup includes: pinned image tags/digests; `restart: unless-stopped`; healthchecks; resource limits; log rotation; named volumes with **tested backups** ([Chapter 5](05-storage-and-networking.md#a9-backup-and-restore-of-a-volume-tar-method)); a reverse proxy with TLS; secrets not in Git; images built and scanned in CI, then deployed by `docker compose pull && docker compose up -d`; the Docker service enabled at boot; monitoring and alerting; a host firewall plus a cloud firewall (remember Docker bypasses ufw, [Chapter 5](05-storage-and-networking.md#b9-firewalls-and-why-docker-bypasses-ufw)).

Deploying to a remote host from your laptop or CI without logging in:

```bash
docker context create prod --docker "host=ssh://deploy@server.example.com"
docker --context prod compose -f compose.yaml -f compose.prod.yaml up -d
```

(`docker context` points your local Docker CLI at a remote engine over SSH. Use a dedicated limited deploy user, noting that access to the Docker socket is effectively root on that machine.)

### When Compose is not enough

Move to an orchestrator (usually Kubernetes, or a managed container service such as AWS ECS/Fargate, Azure Container Apps, Google Cloud Run) when you need:

- **High availability**: survive a machine failure automatically
- **Zero-downtime rolling deploys and quick rollbacks** as a built-in guarantee
- **Autoscaling** on load
- **Many services / many teams / multi-tenant** platform needs, RBAC, quotas, network policies
- **Multi-host scheduling** and bin-packing
- **GPU fleets** shared between workloads ([Chapter 8](08-docker-for-ai-and-ml.md))
- Compliance requiring auditability and policy enforcement at the platform level

Your Compose files are a **great stepping stone**: you already have the service definitions, ports, environment, volumes and healthchecks. Tools exist to convert them (for example `kompose`; the `docker compose bridge` command is Docker's own converter-style feature), but expect to hand-tune the result. See [Chapter 11](11-production-and-enterprise.md) for the migration path.

A realistic career path: **Compose for development and CI everywhere**, and Compose-or-Kubernetes for production depending on scale.

---

## Common mistakes

1. **Writing `version: "3.8"`** at the top. It's obsolete and triggers a warning; delete it.
2. **Using `localhost` between services.** Use service names. ([Chapter 5](05-storage-and-networking.md#b4-the-1-beginner-bug-localhost-confusion))
3. **`depends_on` without `condition: service_healthy`**, then fighting "connection refused" at startup.
4. **Not quoting ports** (`22:22`). Write `"22:22"`.
5. **Tabs in YAML.** Use spaces only. Use a YAML-aware editor.
6. **Wrong indentation** that silently changes meaning (a key landing under the wrong parent). Run `docker compose config` to see the parsed result.
7. **Expecting `docker compose restart` to apply config changes.** Use `docker compose up -d`.
8. **Expecting `docker compose up` to rebuild after code changes.** Use `--build`, `watch`, or `docker compose build`.
9. **`docker compose down -v` on real data.** It deletes named volumes.
10. **Changing the project/folder name** and "losing" data (new empty volumes). Pin `name:`.
11. **Committing `.env` with real secrets.** Commit `.env.example`; gitignore `.env` and `secrets/`.
12. **Thinking `.env` variables reach containers automatically.** They only feed interpolation. Use `environment:`/`env_file:`.
13. **Forgetting `$$`** in healthchecks and shell commands, so Compose expands `$VAR` too early (usually to an empty string).
14. **Publishing database ports** (`"5432:5432"`) in production. Don't, or bind to `127.0.0.1`.
15. **`container_name:`** everywhere, which breaks scaling and causes name clashes.
16. **Mixing `network_mode` with `networks`**, or defining `networks` on a service and forgetting it dropped off `default`.
17. **Assuming healthchecks restart containers.** They don't; `restart:` handles process exits.
18. **No log rotation** (`logging.options.max-size`) filling the disk.
19. **Bind-mounting the source over `/app` in dev and masking installed dependencies**, or running as root creating root-owned files on the host.
20. **Relying on `compose.override.yaml` in production.** It is auto-loaded unless you pass `-f`; prod commands must name files explicitly (or use `COMPOSE_FILE`).
21. **Postgres data path mismatch** across major versions (17 vs 18 mount path).
22. **Putting secrets in `environment:`/`build.args`** where `docker inspect`/`docker history` expose them.
23. **Using Compose "as Kubernetes"**: relying on `deploy.replicas`, `update_config` etc. and expecting rolling updates on a single host.
24. **Running `docker-compose` (V1)** commands from old tutorials. Type `docker compose`.

---

## Exercises

**Exercise 1: Reproduce stage 1 to 3.** Build the notes app yourself from scratch. Prove persistence by adding a note, running `docker compose down`, `up -d`, and fetching it.

**Exercise 2: Break it on purpose.** Remove `condition: service_healthy` from the web service and run `docker compose up -d` from a clean state (`docker compose down` first, no `-v`). Watch `docker compose logs web` for connection errors. Put the condition back.

**Exercise 3: Precedence lab.** Create a service printing `env`. Set `FOO` in four places: Dockerfile `ENV`, `env_file`, `environment`, and `docker compose run -e`. Remove them one at a time from the top and confirm the order given in the chapter. Then set `PG_VERSION` in both your shell and `.env` and use `docker compose config` to see which interpolation wins.

**Exercise 4: Profiles.** Add an `adminer` service under a `debug` profile. Verify that `docker compose up -d` does not start it, `--profile debug` does, and `docker compose run --rm adminer true` auto-enables it.

**Exercise 5: Merge rules.** Create `a.yaml` with `ports: ["8000:8000"]` and `environment: {A: "1"}` and `b.yaml` with `ports: ["9000:8000"]` and `environment: {B: "2"}`. Run `docker compose -f a.yaml -f b.yaml config`. Predict first, then verify. Try `ports: !reset []` in `b.yaml`.

**Exercise 6: Watch.** Enable `docker compose watch` for the web service. Edit the greeting in `main.py` and observe the reload in the logs and with `curl`. Then add a package to `requirements.txt` and watch the rebuild action fire.

**Exercise 7: Secrets.** Move the DB password to a secret file as in `compose.prod.yaml`. Confirm with `docker compose exec web ls -l /run/secrets` and that `docker inspect` of the web container does **not** show the password in its environment.

**Exercise 8: Limits.** Give a container `memory: 64M` and run a Python process that allocates 200 MB. Observe the OOM kill (`docker inspect --format '{{.State.OOMKilled}} {{.State.ExitCode}}' <container>` shows `true 137`).

**Exercise 9: Compose to production.** Write the deploy script (using `pull`, `up -d --wait`, and a `config --quiet` validation step) you'd run in CI. What would you need to add for zero-downtime? (Hint: a reverse proxy, two replicas of web, and a rolling restart script; or an orchestrator.)

**Exercise 10: Dry run.** Use `docker compose --dry-run up -d` and `--dry-run down` to preview actions on a stack, and describe what each line in the output means.

---

## Summary

- **Compose** turns a multi-container app into one declarative `compose.yaml` and one command. Use `docker compose` (V2). Leave out the obsolete `version:` key.
- Services on the project network find each other **by service name**; publish only what outsiders need.
- Use **`depends_on` with `condition: service_healthy`** plus real healthchecks, and make apps retry anyway.
- Keep data in **named volumes**; `down` keeps them, **`down -v` deletes them**.
- Interpolation (`${VAR}`, from shell and `.env`) is separate from container environment (`environment:`/`env_file:`). Container precedence: `run -e` > `environment` > `env_file` > Dockerfile `ENV`. Use `docker compose config` to see the truth.
- Split environments with **override files** (`compose.override.yaml` auto-loads; `-f` replaces that behaviour); reuse with `extends`, `include`, anchors; optional services via **profiles**.
- Deliver sensitive values as **secrets/files**, harden prod services (limits, read-only, dropped capabilities, log rotation).
- **`docker compose watch`** gives fast, cross-platform hot reload.
- Compose is a solid single-host production tool for modest needs; reach for Kubernetes or a managed platform when you need HA, autoscaling and multi-host scheduling.

**Next:** [Chapter 7: Who Needs Docker?](07-who-needs-docker.md) puts all of this in the context of real roles and teams, and [Chapter 8](08-docker-for-ai-and-ml.md) applies it to AI and ML workloads.
