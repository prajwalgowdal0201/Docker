# Chapter 3: Core CLI Commands

> **What you will learn in this chapter**
> - How Docker commands are structured (`docker <object> <verb>`) and why there are two ways to type most of them
> - Every important command for images, containers, logs, files, resources, and cleanup, with **every major flag explained** and sample output
> - How `docker run` really works: foreground vs detached, port publishing (`-p host:container`), volumes, environment variables, restart policies, limits, security flags
> - The container **lifecycle** and its states
> - **Exit codes** (0, 1, 125, 126, 127, 130, 137, 139, 143) and how to diagnose them
> - **PID 1 and signals**: why `docker stop` sometimes takes ten seconds
> - `docker inspect` with Go templates, `stats`, `top`, `events`, `diff`
> - `docker system df` and the prune commands (with warnings)
> - `docker login`, `docker context`, a first look at `docker build` / `buildx`, and image scanning tools
> - A progressive **hands-on lab**: run nginx, a Python shell, and a database; inspect, exec, read logs, clean up
>
> **Time to read:** 90 minutes to 2 hours, including the lab. **Prerequisites:** Docker installed ([Chapter 2](02-installation-and-operating-systems.md)) and the concepts in [Chapter 1](01-what-is-docker.md).
>
> **About the sample output.** Hashes, container IDs, timestamps, random container names, and sizes will differ on your machine. The formats come from Docker Engine **29.9.2** (API 1.56), the version used while writing this guide, and the flag lists were checked against that version's `--help` output. Docker 29 introduced a few output changes (notably the `docker images` layout), and I point those out. If something looks slightly different on your version, `docker <command> --help` is authoritative.

---

## Table of contents

1. [How Docker commands are structured](#1-how-docker-commands-are-structured)
2. [Reading your Docker setup: `version` and `info`](#2-reading-your-docker-setup-version-and-info)
3. [Working with images](#3-working-with-images)
4. [`docker run`: creating and starting containers](#4-docker-run-creating-and-starting-containers)
5. [Listing containers: `docker ps`](#5-listing-containers-docker-ps)
6. [Looking inside a container: `logs`, `exec`, `attach`, `cp`](#6-looking-inside-a-container-logs-exec-attach-cp)
7. [Container lifecycle: `stop`, `kill`, `start`, `restart`, `pause`, `rm`](#7-container-lifecycle-stop-kill-start-restart-pause-rm)
8. [Inspecting and monitoring: `inspect`, `stats`, `top`, `events`, `diff`, `port`](#8-inspecting-and-monitoring-inspect-stats-top-events-diff-port)
9. [Exit codes](#9-exit-codes)
10. [Port publishing explained](#10-port-publishing-explained)
11. [Foreground vs detached](#11-foreground-vs-detached)
12. [PID 1 and signals](#12-pid-1-and-signals)
13. [Disk usage and cleanup: `system df` and `prune`](#13-disk-usage-and-cleanup-system-df-and-prune)
14. [Registries: `login`, `logout`, `search`](#14-registries-login-logout-search)
15. [Contexts: talking to other Docker hosts](#15-contexts-talking-to-other-docker-hosts)
16. [Building: `docker build`, `builder`, and `buildx` (intro)](#16-building-docker-build-builder-and-buildx-intro)
17. [Scanning images: `docker scout`, Trivy, and friends](#17-scanning-images-docker-scout-trivy-and-friends)
18. [Other commands and plugins you will see](#18-other-commands-and-plugins-you-will-see)
19. [Hands-on lab](#19-hands-on-lab)
20. [Common mistakes](#20-common-mistakes)
21. [Cheat sheet](#21-cheat-sheet)
22. [Check your understanding](#22-check-your-understanding)

---

## 1. How Docker commands are structured

### 1.1 The grammar: `docker <object> <verb>`

Modern Docker commands follow the pattern:

```
docker  [global options]  <object>  <verb>  [options]  [arguments]
  |            |             |         |        |           |
  |            |             |         |        |           +-- e.g. the container name or image
  |            |             |         |        +-- flags like -f, --all
  |            |             |         +-- what to do:  ls, rm, inspect, prune, create ...
  |            |             +-- what to act on: container, image, volume, network, builder, context...
  |            +-- e.g. --context, --host, --debug
  +-- the CLI program
```

For example, `docker container ls` means "list containers" and `docker image rm nginx` means "remove the image nginx". The same verbs (`ls`, `rm`, `inspect`, `prune`) work consistently across object types, so once you learn the pattern you can guess commands.

### 1.2 The old short forms still work

Docker's early versions had flat commands (`docker ps`, `docker images`, `docker rm`, `docker rmi`). Those still exist as **aliases** (alternative names) for the structured commands, and almost every tutorial uses them:

| Short form (common) | Full form | What it does |
|---|---|---|
| `docker run` | `docker container run` | Create **and** start a container |
| `docker ps` | `docker container ls` (also `docker container ps`) | List containers |
| `docker ps -a` | `docker container ls -a` | List all containers (including stopped) |
| `docker rm` | `docker container rm` | Remove containers |
| `docker start` / `stop` / `restart` | `docker container start` / `stop` / `restart` | Lifecycle |
| `docker logs` | `docker container logs` | Show container output |
| `docker exec` | `docker container exec` | Run a command in a running container |
| `docker cp` | `docker container cp` | Copy files |
| `docker images` | `docker image ls` (also `docker image list`) | List images |
| `docker rmi` | `docker image rm` | Remove images |
| `docker pull` / `push` | `docker image pull` / `push` | Download/upload images |
| `docker build` | `docker image build` (or `docker buildx build`) | Build an image |
| `docker tag` | `docker image tag` | Add a name to an image |
| `docker history` | `docker image history` | Show an image's layers |
| `docker save` / `load` | `docker image save` / `load` | Export/import images as tar files |
| `docker events` | `docker system events` | Stream daemon events |
| `docker inspect` | (works for any object) | Detailed JSON about containers, images, networks, volumes |

**This guide uses the short forms** (they are shorter and everywhere in the wild) and mentions the full form where useful. In scripts and documentation you are writing, either is fine; the structured form is slightly clearer to new readers.

### 1.3 Getting help

```bash
docker --help                  # list all commands
docker run --help              # all flags for 'run' (very long!)
docker container --help        # all container sub-commands
docker image ls --help
```

Everything in `--help` is the ground truth for your installed version.

### 1.4 Global options you will actually use

These come right after `docker` and before the command:

| Option | Meaning |
|---|---|
| `-H, --host <url>` | Connect to a specific daemon (e.g., `-H ssh://user@host`) |
| `-c, --context <name>` | Use a named context ([Section 15](#15-contexts-talking-to-other-docker-hosts)) |
| `-D, --debug` | Verbose debug output |
| `-l, --log-level` | `debug`, `info`, `warn`, `error`, `fatal` |
| `--config <dir>` | Alternative client config directory (default `~/.docker`) |
| `-v, --version` | Print the version (same as `docker --version`) |
| `--tls`, `--tlsverify`, `--tlscacert`, `--tlscert`, `--tlskey` | Encrypted/authenticated daemon connections |

### 1.5 Anatomy of a long command

```bash
docker run -d --name web -p 8080:80 -e MODE=prod --restart unless-stopped nginx:1.27 nginx -g "daemon off;"
#      |   |          |        |           |            |                   |         |
#      |   |          |        |           |            |                   |         +-- COMMAND and its ARGS (override image's default)
#      |   |          |        |           |            |                   +-- IMAGE[:TAG]
#      |   |          |        |           |            +-- OPTIONS (flags) for docker run
#      +---+----------+--------+-----------+---- everything between 'run' and IMAGE is an option
```

**The rule:** after `docker run`, **everything up to the image name is an option for Docker**; **everything after the image name is the command (and arguments) to run inside the container**. Beginners get this wrong constantly: `docker run nginx -d` does *not* detach; it passes `-d` to the container's program.

Single-letter flags can be combined: `-it` = `-i -t`, and `-dit` = `-d -i -t`. Long flags use double dashes and can be written as `--name web` or `--name=web`.

### 1.6 Terminal differences (Bash vs PowerShell vs cmd)

| Task | Bash / WSL / macOS / Linux | PowerShell (Windows) | cmd.exe |
|---|---|---|---|
| Continue a long command on the next line | `\` at line end | `` ` `` (backtick) at line end | `^` at line end |
| Current directory in a path | `"$(pwd)"` or `$PWD` | `${PWD}` | `%cd%` |
| Environment variable in the shell | `$HOME` | `$env:USERPROFILE` | `%USERPROFILE%` |
| Single quotes | Literal text | Literal text | **Not quotes**: use double quotes |

The examples in this guide use Bash line continuation (`\`). **PowerShell users: replace `\` with a backtick, or put the command on one line.**

---

## 2. Reading your Docker setup: `version` and `info`

### 2.1 `docker version`

```bash
docker version
```

Shows the client and the server (daemon) separately:

```
Client:
 Version:           29.9.2
 API version:       1.56
 Go version:        go1.26.10
 Git commit:        a82e528
 Built:             Tue Oct 20 19:44:07 2026
 OS/Arch:           windows/amd64
 Context:           desktop-linux

Server: Docker Desktop 4.96.3 (244472)
 Engine:
  Version:          29.9.2
  API version:      1.56 (minimum version 1.40)
  Go version:       go1.26.10
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          v2.4.2
 runc:
  Version:          1.5.2
 docker-init:
  Version:          0.19.0
```

| Field | Meaning |
|---|---|
| `Client` / `Server` | Two separate programs ([Chapter 1](01-what-is-docker.md#8-docker-architecture-what-runs-when-you-type-docker-run)). If you only see `Client:` plus an error, the daemon is unreachable. |
| `API version` | The version of the REST API between them. `minimum version 1.40` means this daemon still talks to clients using API 1.40 or newer (Docker's 29.x release notes say 29.3.0 lowered the minimum to 1.40). |
| `OS/Arch` | Platform of each. Here the client is Windows but the engine is Linux (in a VM). |
| `Context` | Which Docker endpoint the client is using ([Section 15](#15-contexts-talking-to-other-docker-hosts)). |
| `containerd`, `runc`, `docker-init` | The component versions in the chain. |

Useful variants:

```bash
docker --version                          # one line: Docker version 29.9.2, build a82e528
docker version --format '{{.Server.Version}}'   # just the number; --format uses Go templates
```

### 2.2 `docker info`

```bash
docker info
```

A long report. The most useful lines:

```
Client:
 Version:    29.9.2
 Context:    desktop-linux
 Plugins:
  buildx: Docker Buildx (Docker Inc.)  Version: v0.38.1
  compose: Docker Compose (Docker Inc.)  Version: v5.6.1
  ...
Server:
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 5
 Server Version: 29.9.2
 Storage Driver: overlayfs
 Logging Driver: local
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Runtimes: io.containerd.runc.v2 nvidia runc
 Default Runtime: runc
 Security Options:
  cgroupns
 Kernel Version: 6.18.54.1-microsoft-standard-WSL2
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 22
 Total Memory: 15.34GiB
```

| Field | What it tells you |
|---|---|
| `Plugins` | Installed CLI plugins (`compose`, `buildx`, `scout`, `model`, ...). If `compose` is missing, `docker compose` will not work. |
| `Containers / Images` | Counts, by state. |
| `Storage Driver` | How image layers are stored (`overlayfs` with the containerd image store, `overlay2` on older/classic setups). |
| `Logging Driver` | Default way container logs are stored (`json-file` on plain Linux Engine, `local` on this Docker Desktop). |
| `Cgroup Version` | `2` on modern systems ([Chapter 1](01-what-is-docker.md#52-cgroups-how-much-can-this-process-use)). |
| `Runtimes` | Available OCI runtimes (`runc` default; `nvidia` appears when the NVIDIA toolkit is installed). |
| `Security Options` | Active hardening: `seccomp`, `apparmor`, `cgroupns`, `rootless`... |
| `Kernel Version`, `OSType`, `Architecture` | The *engine's* Linux kernel and platform (this decides which image architecture is pulled by default). |
| `CPUs`, `Total Memory` | Resources visible to the engine (this is what `.wslconfig` or Desktop settings control). |

Pick single fields with a template: `docker info --format '{{.NCPU}} CPUs, {{.MemTotal}} bytes'`.

---

## 3. Working with images

Recall: an **image** is a read-only layered template; a **tag** is a movable label; a **digest** is the immutable hash ([Chapter 1](01-what-is-docker.md#66-the-anatomy-of-an-image-name)).

### 3.1 `docker pull`: download an image

```bash
docker pull nginx:1.27
```

What it does: downloads the image `nginx` at tag `1.27` from Docker Hub (`docker.io/library/nginx:1.27`) into your local image store. Layers you already have are skipped.

Example output:

```
1.27: Pulling from library/nginx
2d429b9e73a6: Pull complete
a1b2c3d4e5f6: Pull complete
...
Digest: sha256:<long hash>
Status: Downloaded newer image for nginx:1.27
docker.io/library/nginx:1.27
```

| Flag | Meaning |
|---|---|
| `-a, --all-tags` | Download **every tag** in the repository (can be huge; rarely what you want) |
| `--platform <os/arch>` | Pull a specific platform variant, e.g. `--platform linux/amd64` ([Chapter 2](02-installation-and-operating-systems.md#35-the---platform-flag)) |
| `-q, --quiet` | Suppress the verbose progress output |

Variants for naming:

```bash
docker pull nginx                              # = docker.io/library/nginx:latest
docker pull python:3.13-slim                   # a specific version tag
docker pull ghcr.io/owner/tool:2.1             # another registry (GitHub)
docker pull nginx@sha256:3f1c...               # by digest: exact content, immutable
docker pull ubuntu:24.04 --platform linux/arm64   # force architecture
```

You usually do not need `docker pull` explicitly: `docker run` pulls automatically if the image is missing. Use `pull` to pre-download, to update a moving tag (`latest`), or to check access.

### 3.2 `docker images` / `docker image ls`: list local images

```bash
docker images
```

**Docker 29 output (containerd image store, the default for fresh installs):**

```
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:1.27           65645c7bb6a0       74.5MB         21.9MB   U
python:3.13-slim     a1b2c3d4e5f6        125MB         41.2MB
```

**Older (classic) layout**, which you will see in most tutorials:

```
REPOSITORY   TAG         IMAGE ID       CREATED        SIZE
nginx        1.27        65645c7bb6a0   2 weeks ago    192MB
python       3.13-slim   a1b2c3d4e5f6   3 weeks ago    125MB
```

| Column (new layout) | Meaning |
|---|---|
| `IMAGE` | `repository:tag` |
| `ID` | Short form of the image ID (a hash of its config) |
| `DISK USAGE` | How much disk the image occupies once unpacked locally |
| `CONTENT SIZE` | Size of the image's compressed content (roughly what is transferred in a pull/push) |
| `EXTRA` | Status marks. **`U`** means the image is **in use** by at least one container (even a stopped one). Such an image cannot be deleted without removing its containers. (I confirmed `U` marks in-use images on this Docker version by cross-checking against `docker system df`. Check `docker image ls --help` if your version differs.) |

| Flag | Meaning |
|---|---|
| `-a, --all` | Show all images, including intermediate and dangling ones (hidden by default) |
| `-q, --quiet` | Print **only image IDs**, one per line (great for scripting: `docker rmi $(docker images -q)`; be careful!) |
| `--digests` | Show the `sha256` digests |
| `--no-trunc` | Do not shorten IDs |
| `-f, --filter <k=v>` | Filter results (see below) |
| `--format <template>` | Custom output (`table`, `json`, or a Go template) |
| `--tree` | **(Experimental)** Show multi-platform images as a tree, with one child row per platform |

Filters:

```bash
docker images --filter dangling=true          # untagged "<none>" leftovers
docker images --filter "reference=python*"    # name pattern
docker images --filter before=nginx:1.27      # created before
docker images --filter since=nginx:1.27       # created after
docker images --filter label=team=ai          # by image label
```

A **dangling image** is an image with no tag (shown as `<none>:<none>` in the classic layout), usually left over when you rebuild an image and the new build takes over the tag. They are safe to prune ([Section 13](#13-disk-usage-and-cleanup-system-df-and-prune)).

Custom formats:

```bash
docker images --format '{{.Repository}}:{{.Tag}}  {{.ID}}  {{.Size}}'
docker images --format json                  # one JSON object per line
```

(Template fields available vary by version; in the Docker 29 layout I confirmed `.Repository`, `.Tag`, `.ID`, `.Size` work, while a field name like `.DiskUsage` does **not** exist as a template field. Use `docker images --format json` to discover the fields on your version.)

**The tree view:**

```bash
docker image ls --tree
```

```
IMAGE                   ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:1.27-alpine       65645c7bb6a0       74.5MB         21.9MB   U
├─ linux/amd64          62223d644fa2       73.6MB           21MB   U
├─ linux/arm64/v8       63ffc0d1f14e           0B             0B
└─ linux/s390x          a697209b4a44           0B             0B
```

This shows that `nginx:1.27-alpine` is a **multi-platform index**; only the `linux/amd64` variant is actually downloaded on this machine (the others show `0B` because their layers were not pulled). It is a great way to *see* the manifest-list concept from Chapter 1.

### 3.3 `docker image inspect`

```bash
docker image inspect nginx:1.27
docker image inspect nginx:1.27 --format '{{.Architecture}} {{.Os}} {{.Id}}'
docker image inspect nginx:1.27 --format '{{json .Config.Env}}'
```

Prints the image's metadata as JSON: architecture, OS, ID, environment variables set in the image, default command (`Cmd`), `Entrypoint`, exposed ports, labels, layer hashes, and size. See [Section 8.1](#81-docker-inspect-and-go-templates) for the template syntax. For example:

```
amd64 linux sha256:65645c7bb6a0...
```

### 3.4 `docker history`: the layers of an image

```bash
docker history nginx:1.27
```

```
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
65645c7bb6a0   2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      2 weeks ago   COPY 30-tune-worker-processes.sh ...            4.62kB    buildkit.dockerfile.v0
<missing>      2 weeks ago   RUN /bin/sh -c set -x && apt-get update ...     63.8MB    buildkit.dockerfile.v0
```

Each row is one Dockerfile instruction (newest at the top). `SIZE` is the data that step added; `0B` for metadata-only instructions like `CMD`, `EXPOSE`. `<missing>` in the IMAGE column is normal for intermediate layers (they are not stored as separate images). **This is how you find out which step makes an image fat.** (The rows above are illustrative; real output varies by image version.)

| Flag | Meaning |
|---|---|
| `--no-trunc` | Show full (untruncated) `CREATED BY` commands |
| `-H, --human` | Human-readable sizes/dates (the default) |
| `-q, --quiet` | Only IDs |
| `--format` | Go template |
| `--platform` | Show history for a specific platform of a multi-platform image |

### 3.5 `docker tag`: give an image another name

```bash
docker tag nginx:1.27 myregistry.example.com/web/nginx:stable
```

Syntax: `docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`. This does **not** copy anything; it adds another name (a pointer) to the same image. You tag before pushing, because the name encodes the destination registry: `docker push` sends the image to whichever registry is in its name.

### 3.6 `docker push`: upload an image

```bash
docker login ghcr.io                                  # first, authenticate
docker tag myapp:1.0 ghcr.io/yourname/myapp:1.0       # name it for the registry
docker push ghcr.io/yourname/myapp:1.0
```

Output shows each layer as `Pushed` or `Layer already exists` and ends with a `digest: sha256:... size: ...` line. Flags: `-a, --all-tags` (push all tags of the repo), `--platform` (push one platform's manifest), `-q, --quiet`.

### 3.7 `docker rmi` / `docker image rm`: delete images

```bash
docker rmi nginx:1.27                 # remove by name:tag
docker image rm 65645c7bb6a0          # remove by ID
docker rmi -f myapp:old               # force
```

Output looks like `Untagged: nginx:1.27` followed by `Deleted: sha256:...`. **Untagged** means that *name* was removed; **Deleted** means the underlying image data is gone too (it happens only when no other name or container still needs it).

| Flag | Meaning |
|---|---|
| `-f, --force` | Force removal even if multiple tags point to it (it still refuses if a *running* container uses it) |
| `--no-prune` | Do not delete untagged parent layers |
| `--platform` | Remove only the given platform variant of a multi-platform image |

Error: `conflict: unable to delete <id> (must be forced) - image is being used by stopped container ...`: remove the container first (`docker rm`), then the image.

### 3.8 `docker save` and `docker load`: move images without a registry

Useful for air-gapped (offline) servers, backups, and sharing without a registry.

```bash
docker save -o myapp.tar myapp:1.0          # write the image to a tar archive
docker save myapp:1.0 | gzip > myapp.tar.gz   # compressed (Bash)
docker load -i myapp.tar                    # on the other machine: import it
docker load < myapp.tar.gz                  # works with gz too
```

| Command/flag | Meaning |
|---|---|
| `docker save -o FILE IMAGE...` | Export one or more images (all layers and tags) to a tar file. Without `-o`, writes to standard output. |
| `docker save --platform linux/amd64` | Save only the named platform(s) (useful to keep multi-arch archives small) |
| `docker load -i FILE` | Import images from the file. Without `-i`, reads standard input. |
| `docker load -q` | Quiet |

### 3.9 `docker export`, `docker import`, `docker commit` (know them, rarely use them)

| Command | What it does | Why you should usually avoid it |
|---|---|---|
| `docker export CONTAINER > fs.tar` | Dumps a container's **filesystem** (flattened, no layers/metadata) | Loses history and metadata. |
| `docker import fs.tar myimage:flat` | Creates an image from such a tar | Same. |
| `docker commit CONTAINER newimage:tag` | **Snapshots a container's changes into a new image** | The result is not reproducible: nobody knows what you did. Write a Dockerfile instead (Chapter 4). Good only for quick experiments. |

### 3.10 `docker search`

```bash
docker search --limit 5 postgres
docker search --filter is-official=true python
```

Searches Docker Hub from the terminal and prints name, description, stars, and an `[OK]` flag for official images. Browsing hub.docker.com is usually easier, and shows tags and docs.

### 3.11 Image cleanup commands

```bash
docker image prune             # remove dangling images (untagged leftovers) -- asks for confirmation
docker image prune -a          # remove ALL images not used by any container
docker image prune -a --filter "until=168h"   # ... only those older than 7 days
```

See the warnings in [Section 13](#13-disk-usage-and-cleanup-system-df-and-prune).

---

## 4. `docker run`: creating and starting containers

`docker run` is the most important command you will learn. It does three things in one:

```
 docker run IMAGE
      |
      +--1. docker pull   (only if the image is not already local)
      +--2. docker create (make a new container: image + writable layer + config)
      +--3. docker start  (start its main process) and, unless -d, attach your terminal to it
```

### 4.1 Syntax

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

- `IMAGE`: what to run, e.g., `nginx:1.27`
- `COMMAND [ARG...]`: optional; **overrides the image's default command** (`CMD`)

```bash
docker run alpine echo "hello from a container"
```

Output: `hello from a container`. Docker created a container from `alpine`, ran `echo ...` inside it, then the container exited because its main process finished.

> **Key idea: a container lives exactly as long as its main process (PID 1).** When that process exits, the container stops. A web server stays up because the server process keeps running; `echo` exits immediately, so the container exits immediately.

### 4.2 The flags you need (grouped)

**Lifecycle and attachment**

| Flag | Meaning |
|---|---|
| `-d, --detach` | Run in the **background** and print the new container's ID. Your terminal stays free. |
| `-i, --interactive` | Keep **standard input (stdin)** open so you can type into the container. |
| `-t, --tty` | Allocate a **pseudo-terminal (TTY)**: gives a proper terminal experience (prompt, colors, line editing). |
| `-it` | Combine: the usual way to get an interactive shell: `docker run -it ubuntu bash`. |
| `--rm` | **Automatically delete the container when it exits** (and its anonymous volumes). Use for throwaway runs; prevents the pile-up of stopped containers. |
| `--name <name>` | Give the container a **name** instead of a random one. Names must be unique among existing containers. Lets you write `docker logs web` instead of an ID. |
| `--restart <policy>` | What to do when it exits (see 4.9). |
| `--init` | Run a tiny init process as PID 1 that forwards signals and reaps zombie processes ([Section 12](#12-pid-1-and-signals)). |
| `--pull <always\|missing\|never>` | When to pull the image. Default `missing`. `always` checks the registry for a newer image every time; `never` uses only local images. |

**Networking and ports**

| Flag | Meaning |
|---|---|
| `-p, --publish <host>:<container>` | **Publish** a container port on the host ([Section 10](#10-port-publishing-explained)). |
| `-P, --publish-all` | Publish **all** `EXPOSE`d ports to random high host ports (see `docker port`). |
| `--network <name>` | Attach to a specific network (`bridge`, `host`, `none`, or a user-defined network). Chapter 5. |
| `--network-alias <name>` | An extra DNS name for the container on a user-defined network. |
| `-h, --hostname <name>` | Set the container's hostname. |
| `--add-host <name>:<ip>` | Add an `/etc/hosts` entry inside the container (e.g., `host.docker.internal:host-gateway` to reach the host on Linux). |
| `--dns <ip>` | Custom DNS server. |
| `--expose <port>` | Document/mark a port as exposed (does **not** publish it). |

**Storage**

| Flag | Meaning |
|---|---|
| `-v, --volume <src>:<dest>[:opts]` | Mount a **bind mount** (host folder) or **named volume**. Short syntax. |
| `--mount type=...,source=...,target=...` | The same, but **explicit** and more readable ([4.7](#47-volumes-and-bind-mounts--v-and---mount)). |
| `--tmpfs <path>` | Mount an in-memory temporary filesystem at the path (RAM-backed, vanishes at exit). |
| `--read-only` | Make the container's root filesystem **read-only** (security hardening). Pair with `--tmpfs /tmp` and volumes for paths that must be writable. |
| `--volumes-from <container>` | Mount all volumes from another container. |
| `-w, --workdir <dir>` | Working directory inside the container for the command. |

**Configuration**

| Flag | Meaning |
|---|---|
| `-e, --env KEY=value` | Set an environment variable (`-e KEY` with no value copies it from your shell). |
| `--env-file <file>` | Read environment variables from a file. |
| `-l, --label key=value` | Attach metadata labels. |
| `--entrypoint <cmd>` | Override the image's `ENTRYPOINT`. |
| `-u, --user <name\|uid[:gid]>` | Run as a specific user (e.g., `1000:1000`). |
| `--platform <os/arch>` | Choose the platform (e.g., `linux/amd64`). |
| `--stop-signal <sig>` / `--stop-timeout <sec>` | Customise how `docker stop` stops it. |
| `--health-cmd`, `--health-interval`, `--health-timeout`, `--health-retries`, `--health-start-period`, `--no-healthcheck` | Define or disable health checks (Chapter 4 and 11). |
| `--log-driver`, `--log-opt` | Choose how this container's logs are stored. |

**Resources**

| Flag | Meaning |
|---|---|
| `-m, --memory <size>` | Hard memory limit (`512m`, `2g`). Exceed it and the kernel's OOM killer terminates the container (exit 137). |
| `--memory-swap <size>` | Memory + swap total limit. Set equal to `--memory` to disable swap. |
| `--memory-reservation <size>` | A soft limit: the target under memory pressure. |
| `--cpus <n>` | How many CPUs' worth of time (e.g., `1.5`). |
| `--cpu-shares <n>` / `-c` | Relative CPU weight when CPUs are contended (default 1024). |
| `--cpuset-cpus <list>` | Pin to specific cores (`0-3`, `0,2`). |
| `--pids-limit <n>` | Max number of processes (defends against fork bombs). |
| `--shm-size <size>` | Size of `/dev/shm` shared memory (default 64 MB; **PyTorch DataLoader workers and browsers often need more**, e.g. `--shm-size=2g`). |
| `--ulimit <type>=<soft>:<hard>` | Resource limits like open files (`nofile=65536:65536`). |
| `--gpus <spec>` | Give the container GPU(s): `--gpus all` ([Chapter 8](08-docker-for-ai-and-ml.md)). |
| `--device <host>:<container>` | Pass a host device (e.g., `/dev/ttyUSB0`). |

**Security (introduced here, detailed in [Chapter 10](10-security.md))**

| Flag | Meaning |
|---|---|
| `--cap-drop <CAP>` / `--cap-add <CAP>` | Remove/add Linux capabilities (`--cap-drop ALL` is a strong baseline). |
| `--security-opt <opt>` | e.g., `no-new-privileges`, custom seccomp/AppArmor profiles. |
| `--privileged` | **Almost all protections removed**: full device access. Avoid; treat as "root on the host". |
| `--userns`, `--pid`, `--ipc`, `--uts` | Share or isolate namespaces (e.g., `--pid=host` removes PID isolation). |

### 4.3 Foreground, detached, and interactive

```bash
docker run nginx                       # FOREGROUND: your terminal shows nginx's logs; Ctrl+C stops it
docker run -d nginx                    # DETACHED: prints a container ID and returns your prompt
docker run -it ubuntu bash             # INTERACTIVE: a shell inside a new Ubuntu container
docker run -d -it ubuntu bash          # detached but with a TTY, so you can attach later
```

Detailed explanation in [Section 11](#11-foreground-vs-detached).

### 4.4 Naming: `--name` and `--rm`

```bash
docker run -d --name web nginx:1.27
docker run --rm alpine echo "one-shot"     # the container is deleted after it exits
```

Without `--name`, Docker invents a name like `vibrant_turing`. With `--name web`, running the same command twice fails:

```
docker: Error response from daemon: Conflict. The container name "/web" is already in use by container "3f4a1b9c2d7e...". You have to remove (or rename) that container to be able to reuse that name.
```

Fix: `docker rm web` (or `docker rm -f web` if it is running), or choose another name.

### 4.5 Publishing ports: `-p` and `-P`

```bash
docker run -d --name web -p 8080:80 nginx:1.27
```

`-p 8080:80` means **host port 8080 forwards to container port 80**. Then `http://localhost:8080` reaches nginx. Full explanation with diagrams in [Section 10](#10-port-publishing-explained). Forms:

| Form | Meaning |
|---|---|
| `-p 8080:80` | Host port 8080, container port 80, on **all** host interfaces (`0.0.0.0`) |
| `-p 127.0.0.1:8080:80` | Only reachable from the host itself (not the network) |
| `-p 8080:80/udp` | UDP instead of the default TCP |
| `-p 80` | Container port 80 mapped to a **random** host port |
| `-p 5000-5010:5000-5010` | A range |
| `-P` | Publish every `EXPOSE`d port to random host ports |

### 4.6 Environment variables: `-e` and `--env-file`

```bash
docker run --rm -e GREETING=hello -e USER_NAME=ada alpine sh -c 'echo "$GREETING, $USER_NAME"'
```

Output: `hello, ada`. (We use `sh -c '...'` so the *container's* shell expands the variables, not your host shell; note the single quotes.)

```bash
export API_KEY=placeholder-not-a-real-key      # Bash; in PowerShell: $env:API_KEY = "placeholder-not-a-real-key"
docker run --rm -e API_KEY alpine sh -c 'echo "key length: ${#API_KEY}"'
```

`-e API_KEY` (no `=value`) **copies the value from your current shell** into the container, so the value does not appear in your command history.

**`--env-file`:** put many variables in a file:

```
# app.env   (lines starting with # are comments)
DATABASE_HOST=db
DATABASE_PORT=5432
LOG_LEVEL=info
```

```bash
docker run --rm --env-file app.env alpine env
```

Rules of the format: one `KEY=value` per line, `#` comment lines, and **no shell processing**: quotes are *not* stripped (so `KEY="abc"` gives a value that includes the quotes), no `export` prefix, no variable substitution. **Never commit real secrets in `.env` files to Git.** Add them to `.gitignore`; Chapter 10 covers proper secret handling.

### 4.7 Volumes and bind mounts: `-v` and `--mount`

Container data is **ephemeral** unless you mount storage. (Full treatment in [Chapter 5](05-storage-and-networking.md).) Three kinds:

| Type | What it is | Example | Use for |
|---|---|---|---|
| **Named volume** | Storage managed by Docker (lives under Docker's data directory), referenced by name | `-v pgdata:/var/lib/postgresql/data` | Databases and any data Docker should own |
| **Bind mount** | A specific **host folder or file** mapped into the container | `-v "$(pwd)":/app` | Live-editing code, sharing config files |
| **tmpfs** | Memory-only filesystem | `--tmpfs /tmp` | Scratch data and secrets that must not touch disk |

Short syntax (`-v`):

```bash
docker run -d -v pgdata:/var/lib/postgresql/data postgres:17        # named volume
docker run --rm -v "$(pwd)":/app -w /app python:3.13-slim ls         # bind mount current folder
docker run --rm -v "$(pwd)"/config.yml:/etc/app/config.yml:ro alpine cat /etc/app/config.yml   # read-only file mount
```

The `:ro` suffix makes the mount **read-only** inside the container (`:rw` is the default).

Explicit syntax (`--mount`), equivalent but clearer:

```bash
docker run -d \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:17

docker run --rm \
  --mount type=bind,source="$(pwd)",target=/app,readonly \
  alpine ls /app

docker run --rm --mount type=tmpfs,target=/scratch alpine df /scratch
```

| `--mount` key | Meaning |
|---|---|
| `type` | `bind`, `volume`, or `tmpfs` |
| `source` (or `src`) | Host path (bind) or volume name (volume); omitted for tmpfs |
| `target` (or `dst`, `destination`) | Path **inside the container** |
| `readonly` (or `ro`) | Make read-only |

**`-v` vs `--mount`:** with `-v`, if the host path of a bind mount does not exist, Docker silently **creates it as an (empty) directory** (a common source of "my file became a folder" bugs). With `--mount type=bind`, a missing source path is an **error**, which is safer. For new work, `--mount` is recommended for readability. `-v` is shorter and ubiquitous.

**Windows paths:** PowerShell `-v ${PWD}:/app`; cmd `-v %cd%:/app`; and for performance keep the folder inside WSL ([Chapter 2](02-installation-and-operating-systems.md#211-the-1-windows-performance-tip-keep-your-project-files-inside-wsl)).

### 4.8 Working directory, user, entrypoint

```bash
docker run --rm -w /tmp alpine pwd                 # -w sets the working directory
# Output: /tmp

docker run --rm alpine id                          # default user in this image: root
# Output: uid=0(root) gid=0(root) groups=0(root),...

docker run --rm -u 1000:1000 alpine id             # run as UID 1000, GID 1000
# Output: uid=1000 gid=1000 groups=1000

docker run --rm --entrypoint /bin/sh alpine -c 'echo overridden entrypoint'
```

`--user` improves security (avoid running as root); **but** the user must have permissions to the files the process needs. **`--entrypoint`** replaces the image's configured entry command (the arguments after the image name become that command's arguments). Chapter 4 explains `ENTRYPOINT` vs `CMD` thoroughly.

### 4.9 Restart policies: `--restart`

By default (`no`) a container that exits stays stopped. For servers you want Docker to bring it back.

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart automatically. |
| `on-failure[:max]` | Restart **only if the exit code is non-zero**, optionally up to `max` attempts (e.g., `on-failure:5`). |
| `always` | Always restart, including when the daemon starts. (If you manually stop it, it stays stopped until the daemon restarts or you start it.) |
| `unless-stopped` | Like `always`, but **does not** restart after a reboot if you had explicitly stopped it. **Usually the best choice for services.** |

```bash
docker run -d --name api --restart unless-stopped -p 8000:8000 myapi:1.0
docker update --restart on-failure:3 api      # change the policy of an existing container
```

A restart policy is **not** a substitute for fixing crashes; a container that crashes immediately will enter a restart loop (status `Restarting`). Check `docker logs`.

### 4.10 Resource limits: memory, CPU, processes

```bash
docker run -d --name limited --memory 512m --memory-swap 512m --cpus 1.5 --pids-limit 200 myapp:1.0
```

| Flag | Meaning in this command |
|---|---|
| `--memory 512m` | Max RAM = 512 MiB. Above this, the kernel kills the container's process (**exit code 137**, `OOMKilled=true`). |
| `--memory-swap 512m` | Memory + swap = 512 MiB, i.e., **no swap** (equal to `--memory`). If you omit it, Docker lets the container use swap equal to the memory limit (swap enabled), subject to host config. |
| `--cpus 1.5` | At most 1.5 cores' worth of CPU time (throttled, not killed). |
| `--pids-limit 200` | No more than 200 processes/threads. |

Check enforcement with `docker stats` ([Section 8.3](#83-docker-stats)). Note: on Docker Desktop, limits apply *within* the VM's total resources ([Chapter 2](02-installation-and-operating-systems.md#210-limit-wsl-2s-resources-with-wslconfig)).

**Update limits on a running container** (no restart needed for most):

```bash
docker update --memory 1g --memory-swap 1g --cpus 2 limited
```

### 4.11 Read-only containers and tmpfs

```bash
docker run --rm --read-only --tmpfs /tmp alpine sh -c 'touch /tmp/ok && echo "wrote /tmp"; touch /nope'
```

Output:

```
wrote /tmp
touch: /nope: Read-only file system
```

`--read-only` makes everything except mounted volumes/tmpfs unwritable, so attackers (or buggy code) cannot modify the container's files. `--tmpfs /tmp` gives programs a writable scratch space in memory.

### 4.12 Platform and GPU

```bash
docker run --rm --platform linux/amd64 alpine uname -m     # x86_64, even on an ARM Mac (emulated)
docker run --rm --gpus all nvidia/cuda:<tag> nvidia-smi    # gives the container all GPUs
docker run --rm --gpus 2 ...                               # any 2 GPUs
docker run --rm --gpus '"device=0,1"' ...                  # specific GPUs by index
```

**`--gpus` requires** an NVIDIA GPU, the NVIDIA driver on the host, and the NVIDIA Container Toolkit (on Linux); Docker Desktop with WSL 2 handles this when your Windows NVIDIA driver is current. The CUDA image tag in the example (`<tag>`) is a placeholder: tags change, so pick a current one from Docker Hub's `nvidia/cuda` page. Full coverage in [Chapter 8](08-docker-for-ai-and-ml.md).

### 4.13 Health checks (a first look)

```bash
docker run -d --name web \
  --health-cmd "curl -fsS http://localhost/ || exit 1" \
  --health-interval 10s --health-timeout 3s --health-retries 3 --health-start-period 5s \
  nginx:1.27
docker ps    # STATUS shows "Up 30 seconds (healthy)" or "(unhealthy)" / "(health: starting)"
```

Docker runs the command periodically *inside* the container; exit code 0 means healthy. (The nginx image may not include `curl`; this is just to illustrate syntax. Chapter 4 shows the Dockerfile `HEALTHCHECK` instruction, the more common way.)

### 4.14 Putting many flags together

```bash
docker run -d \
  --name chatbot-api \
  --restart unless-stopped \
  -p 127.0.0.1:8000:8000 \
  --env-file ./api.env \
  -e LOG_LEVEL=info \
  --mount type=volume,source=model-cache,target=/root/.cache \
  --memory 4g --cpus 2 \
  --read-only --tmpfs /tmp \
  --user 1000:1000 \
  --cap-drop ALL --security-opt no-new-privileges \
  myorg/chatbot-api:1.4.2
```

Reading it top to bottom: runs detached, named `chatbot-api`, restarts unless you stop it, reachable only from the local host on port 8000, gets settings from a file plus one extra variable, stores a model cache in a named volume, is limited to 4 GB / 2 CPUs, has a read-only root filesystem with an in-memory `/tmp`, runs as non-root user 1000, drops all Linux capabilities, and cannot gain new privileges. (In practice, you would put this in a Compose file: [Chapter 6](06-docker-compose.md).)

---

## 5. Listing containers: `docker ps`

```bash
docker ps
```

```
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                  NAMES
3f4a1b9c2d7e   nginx:1.27   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   0.0.0.0:8080->80/tcp   web
```

| Column | Meaning |
|---|---|
| `CONTAINER ID` | Shortened unique ID (the full ID is 64 hex characters). You can use any unique prefix in commands. |
| `IMAGE` | The image it was created from. |
| `COMMAND` | The command running as PID 1 (truncated). |
| `CREATED` | When created. |
| `STATUS` | `Up 2 minutes`, `Exited (0) 5 seconds ago`, `Up 3 minutes (healthy)`, `Restarting (1) 4 seconds ago`, `Paused`... |
| `PORTS` | Port mappings: `0.0.0.0:8080->80/tcp` reads "host port 8080 on all interfaces forwards to container port 80/tcp". A port shown without `->` (e.g., `80/tcp`) is exposed but **not published**. |
| `NAMES` | The container name. |

| Flag | Meaning |
|---|---|
| `-a, --all` | Show **all** containers, including stopped ones (default shows only running) |
| `-q, --quiet` | Only IDs (`docker rm -f $(docker ps -aq)` removes all; dangerous!) |
| `-l, --latest` | Only the most recently created container |
| `-n, --last <N>` | The N most recently created containers (any state) |
| `-s, --size` | Add a `SIZE` column: writable-layer size and (in parentheses) virtual size |
| `--no-trunc` | Do not truncate the command or IDs |
| `-f, --filter` | Filter (below) |
| `--format` | Custom output |

Filters:

```bash
docker ps -a --filter status=exited               # states: created, running, paused, restarting, exited, dead
docker ps -a --filter "exited=0"                  # exited with a specific code
docker ps --filter name=web                       # name contains "web"
docker ps --filter ancestor=nginx:1.27            # created from this image
docker ps --filter publish=8080                   # publishing this port
docker ps --filter health=unhealthy
docker ps --filter label=env=prod
```

Custom formats with Go templates:

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --format '{{.Names}}: {{.Status}}'
docker ps --format json
```

(`\t` is a tab; `table` adds a header row. Common placeholders: `.ID`, `.Image`, `.Command`, `.CreatedAt`, `.RunningFor`, `.Ports`, `.State`, `.Status`, `.Size`, `.Names`, `.Labels`, `.Mounts`, `.Networks`.)

---

## 6. Looking inside a container: `logs`, `exec`, `attach`, `cp`

### 6.1 `docker logs`

```bash
docker logs web
```

Shows everything the container's main process wrote to **stdout and stderr** (standard output and standard error: the two text streams every program has). This is the **first thing to run when a container misbehaves.**

> **Important:** `docker logs` only captures **stdout/stderr**. If your app writes logs only to a file inside the container, `docker logs` shows nothing. Containerized apps should log to stdout/stderr (the "twelve-factor" convention).

| Flag | Meaning |
|---|---|
| `-f, --follow` | **Stream** new log lines live (like `tail -f`). Ctrl+C stops following, not the container. |
| `-n, --tail <N>` | Only the last N lines (`--tail 50`; default `all`) |
| `--since <time>` | Logs since a timestamp or relative time: `--since 10m`, `--since 2026-10-23T09:00:00` |
| `--until <time>` | Logs before a time |
| `-t, --timestamps` | Prefix each line with a timestamp |
| `--details` | Show extra details provided to the logging driver |

```bash
docker logs -f --tail 20 web              # last 20 lines, then stream
docker logs --since 5m -t web             # last 5 minutes with timestamps
docker logs web 2>&1 | grep -i error      # search: 2>&1 merges stderr into the pipe
```

Output example (nginx access log):

```
172.17.0.1 - - [23/Oct/2026:09:15:02 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0" "-"
```

**Logging drivers.** *Where* logs are kept depends on the logging driver: `json-file` (default on plain Linux Engine; unbounded unless you configure rotation), `local` (rotates by default), `syslog`, `journald`, `fluentd`, `awslogs`, `none`. Note that `docker logs` only works with some drivers (`json-file`, `local`, `journald`). Choose per container with `--log-driver` and `--log-opt max-size=10m --log-opt max-file=3`, or globally in `daemon.json` ([Chapter 2](02-installation-and-operating-systems.md#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)).

### 6.2 `docker exec`: run a command in a *running* container

```bash
docker exec web nginx -v                 # run one command and print its output
docker exec -it web bash                 # open an interactive shell inside it
docker exec -it web sh                   # use sh if bash is not installed (e.g., Alpine)
docker exec -u root -w /etc web ls       # as a specific user, in a specific directory
docker exec -e DEBUG=1 web env           # with an extra environment variable
docker exec -d web touch /tmp/started    # run in the background (detached)
```

| Flag | Meaning |
|---|---|
| `-i, --interactive` | Keep stdin open |
| `-t, --tty` | Allocate a terminal |
| `-d, --detach` | Run the command in the background |
| `-u, --user` | Run as this user |
| `-w, --workdir` | Working directory for the command |
| `-e, --env` / `--env-file` | Extra environment variables for this command only |
| `--privileged` | Give extended privileges to the command (avoid) |

**How `exec` differs from `run` and `attach`:**

| Command | What it does |
|---|---|
| `docker run` | Creates a **new** container from an image and starts it |
| `docker exec` | Starts an **additional process** inside an **already running** container (the container's main process is unaffected) |
| `docker attach` | Connects your terminal to the container's **main process** (PID 1) streams |

Exiting a shell started with `exec` (type `exit`) does **not** stop the container, because the container's main process keeps running. That is why `exec` is the safe way to poke around in a service. Troubleshooting a container that has no shell (distroless images)? Docker's `docker debug` plugin and `docker cp` can help ([Section 18](#18-other-commands-and-plugins-you-will-see)). `exec` only works on **running** containers; for a crashed container, use `docker logs`, or start a *new* container from the same image with `--entrypoint sh`.

### 6.3 `docker attach`

```bash
docker attach web
```

Connects your terminal's stdin/stdout/stderr to the container's PID 1 process. Useful for interactive programs started with `-it` and detached with `-d`. **Danger:** Ctrl+C in an attached session sends SIGINT to the main process and can *stop the container*. To **detach without stopping**, press **Ctrl+P then Ctrl+Q** (works only if the container was started with `-it`).

| Flag | Meaning |
|---|---|
| `--no-stdin` | Do not attach stdin (read-only view; safer, as you cannot type into it) |
| `--sig-proxy` | Forward signals you send to the process (default **true**; use `--sig-proxy=false` to prevent Ctrl+C from reaching it) |
| `--detach-keys` | Change the detach key sequence |

For just watching output, `docker logs -f` is almost always better than `attach`.

### 6.4 `docker cp`: copy files between host and container

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf     # container -> host
docker cp ./index.html web:/usr/share/nginx/html/    # host -> container
docker cp web:/var/log/nginx/. ./nginx-logs/         # copy a folder's CONTENTS (note the trailing /.)
```

Syntax: `docker cp SRC DEST`, where one side is `CONTAINER:PATH`. It works on both running and **stopped** containers. A trailing `/.` on a source directory copies the contents rather than the directory itself.

| Flag | Meaning |
|---|---|
| `-a, --archive` | Preserve uid/gid information |
| `-L, --follow-link` | Follow symbolic links in the source |
| `-q, --quiet` | Suppress progress output |

`docker cp` is for debugging and one-off extraction. Don't use it as a deployment method: bake files into the image, or use volumes/bind mounts.

---

## 7. Container lifecycle: `stop`, `kill`, `start`, `restart`, `pause`, `rm`

### 7.1 The lifecycle diagram

```
                       docker create
                            |
                            v
                     +-------------+   docker start    +-------------+
   docker run  ----> |   created   | ----------------> |   running   | <-------+
   (create+start)    +-------------+                   +------+------+         |
                                                              |  ^             |
                         docker pause                         |  | docker      | docker start
                     +----------------------------------------+  | unpause     | (or restart
                     v                                          |  |            | policy)
                +-----------+                                   |  |            |
                |  paused   | ----------------------------------+  |            |
                +-----------+                                      |            |
                                                                    |            |
   process exits  /  docker stop  /  docker kill  /  OOM           |            |
                     +----------------------------------------------+            |
                     v                                                           |
                +-----------+  ---------------------------------------------------+
                |  exited   |
                +-----+-----+
                      | docker rm
                      v
                 (removed: container and its writable layer are gone)

   Other states you may see:  restarting (restart policy is retrying),
                              removing (being deleted), dead (failed removal; needs manual cleanup)
```

| State | Meaning |
|---|---|
| **created** | Container exists, but its process has never been started. |
| **running** | Main process is alive. |
| **paused** | All processes frozen (using the cgroup freezer); memory is retained. |
| **restarting** | A restart policy is waiting/retrying after an exit. |
| **exited** | Main process ended (exit code is recorded). The container, its logs, and writable layer **still exist** until removed. |
| **removing** | Being deleted. |
| **dead** | Removal failed partway; manual intervention needed (`docker rm -f`, or restart the daemon). |

### 7.2 `docker stop`: graceful shutdown

```bash
docker stop web
docker stop -t 30 web          # wait up to 30 s before force-killing
docker stop web db cache       # several at once
```

What happens:

1. Docker sends **SIGTERM** (signal 15, the polite "please shut down") to the container's main process (PID 1). You can change the signal with `--stop-signal` at run time, or `-s` here.
2. It waits up to **10 seconds** by default (`-t`/`--timeout` changes it).
3. If the process is still alive, Docker sends **SIGKILL** (signal 9, which cannot be caught or ignored).

A well-behaved app catches SIGTERM, finishes in-flight requests, closes connections, and exits with code 0 or 143. If your container always takes exactly ten seconds to stop and exits 137, your PID 1 is ignoring SIGTERM ([Section 12](#12-pid-1-and-signals)).

| Flag | Meaning |
|---|---|
| `-t, --timeout <sec>` | Seconds to wait before SIGKILL (default 10) |
| `-s, --signal <sig>` | Signal to send instead of SIGTERM |

### 7.3 `docker kill`: immediate

```bash
docker kill web                 # SIGKILL: instant, no cleanup
docker kill -s HUP web          # send a specific signal (HUP often means "reload config")
docker kill -s USR1 web
```

`kill` defaults to **SIGKILL**: the process cannot clean up (no flushed buffers, no closed connections). Prefer `stop`. Use `kill -s <signal>` to send a particular signal without stopping, e.g., reload nginx with `docker kill -s HUP web`.

### 7.4 `docker start`, `restart`

```bash
docker start web              # start a stopped container (keeps its name, settings, and data in its writable layer)
docker start -a web           # start and ATTACH to its output
docker start -ai mycontainer  # attach and interactive (restart a stopped interactive shell)
docker restart web            # stop then start
docker restart -t 5 web       # wait 5 s before killing during the stop phase
```

`start` flags: `-a, --attach` (attach stdout/stderr and forward signals), `-i, --interactive` (attach stdin). `restart` flags: `-t, --timeout`, `-s, --signal`.

**Important:** `docker start` resumes the *same container* with its existing writable layer. Changing an image tag does not update an existing container: you must **remove and re-create** the container (`docker rm` + `docker run`) to pick up a new image or new `-p`/`-e`/`-v` flags. Those settings are fixed at creation (except a few updatable via `docker update`).

### 7.5 `docker pause` / `unpause`

```bash
docker pause web        # freeze every process in the container (no CPU use; memory stays allocated)
docker unpause web
```

Uses the cgroup *freezer*. Rarely needed; helpful for taking consistent snapshots or temporarily halting a busy container without losing state.

### 7.6 `docker rm`: delete containers

```bash
docker rm web                  # remove a STOPPED container
docker rm -f web               # force: kill (SIGKILL) a running container, then remove it
docker rm -v web               # also remove its anonymous volumes
docker rm web db cache         # several at once
```

| Flag | Meaning |
|---|---|
| `-f, --force` | Remove even if running (uses SIGKILL) |
| `-v, --volumes` | Also delete **anonymous** volumes attached to it (named volumes are *never* deleted by `rm`) |
| `-l, --link` | Remove a legacy link (rare) |

Removing a container deletes its **writable layer**: any data not in a volume is lost. Error `You cannot remove a running container ... Stop the container before attempting removal or force remove`: stop it first or use `-f`.

Bulk cleanup of **all stopped containers**:

```bash
docker container prune          # asks for confirmation, removes every stopped container
```

### 7.7 `docker rename`, `docker wait`, `docker update`, `docker create`

```bash
docker rename web web-old             # rename a container (running or not)
docker wait web                       # BLOCK until the container stops, then print its exit code (great in scripts)
docker create --name later nginx:1.27 # create (but don't start) a container; status "created"
docker start later                    # start it later
docker update --restart unless-stopped --memory 1g web   # change some settings on a live container
```

`docker create` takes the same flags as `docker run`; it only prepares the container.

---

## 8. Inspecting and monitoring: `inspect`, `stats`, `top`, `events`, `diff`, `port`

### 8.1 `docker inspect` and Go templates

`docker inspect` prints **everything** Docker knows about an object (container, image, volume, network...) as a large JSON document.

```bash
docker inspect web
```

An abbreviated sample of what's in the output for a container:

```json
[
  {
    "Id": "3f4a1b9c2d7e...",
    "Name": "/web",
    "State": {
      "Status": "running",
      "Running": true,
      "Paused": false,
      "Restarting": false,
      "OOMKilled": false,
      "Dead": false,
      "Pid": 48213,
      "ExitCode": 0,
      "StartedAt": "2026-10-23T09:13:00.123456789Z",
      "Health": { "Status": "healthy" }
    },
    "Image": "sha256:65645c7bb6a0...",
    "Config": {
      "Hostname": "3f4a1b9c2d7e",
      "Env": ["PATH=/usr/local/sbin:...", "NGINX_VERSION=1.27.x"],
      "Cmd": ["nginx", "-g", "daemon off;"],
      "Image": "nginx:1.27",
      "ExposedPorts": { "80/tcp": {} }
    },
    "HostConfig": {
      "PortBindings": { "80/tcp": [ { "HostIp": "", "HostPort": "8080" } ] },
      "RestartPolicy": { "Name": "unless-stopped" },
      "Memory": 0
    },
    "Mounts": [],
    "NetworkSettings": {
      "Ports": { "80/tcp": [ { "HostIp": "0.0.0.0", "HostPort": "8080" } ] },
      "Networks": { "bridge": { "IPAddress": "172.17.0.2", "Gateway": "172.17.0.1" } }
    }
  }
]
```

Reading JSON dumps is tedious, so use **`--format` (`-f`)** with a **Go template** to pull out exactly what you want.

**What is a Go template?** Docker is written in the Go language, and `--format` accepts Go's templating syntax: text with `{{ ... }}` blocks. Inside, a leading `.` means "the object being inspected," and `.State.Status` means "the `Status` field inside `State`."

```bash
docker inspect --format '{{.State.Status}}' web                       # running
docker inspect --format '{{.State.ExitCode}}' web                     # 0
docker inspect --format '{{.State.OOMKilled}}' web                    # false (was it killed for memory?)
docker inspect --format '{{.Config.Image}}' web                       # nginx:1.27
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' web      # unless-stopped
docker inspect --format '{{.State.Pid}}' web                          # the process's PID on the HOST
docker inspect --format '{{.State.Health.Status}}' web                # healthy (only if a health check exists)
```

Intermediate templates:

```bash
# Print JSON for a sub-section (json function), nicely compact:
docker inspect --format '{{json .Config.Env}}' web
# ["PATH=/usr/local/sbin:...","NGINX_VERSION=1.27.x"]

# The container's IP address on every network it is attached to:
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' web
# 172.17.0.2

# Which host port is mapped to container port 80/tcp?  (index function for keys that contain special characters)
docker inspect --format '{{(index (index .NetworkSettings.Ports "80/tcp") 0).HostPort}}' web
# 8080

# Mount sources:
docker inspect --format '{{range .Mounts}}{{.Type}}: {{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' web
```

| Template syntax | Meaning |
|---|---|
| `{{.Field}}` | Print a field; chain with dots: `.State.Status` |
| `{{json .Field}}` | Print a field as JSON |
| `{{range .List}} ... {{end}}` | Loop over a list or map; inside, `.` is the current item |
| `{{index .Map "key"}}` | Look up a key (needed when the key has `/`, `-`, or similar) |
| `{{"\n"}}` | A literal newline |
| `{{if .Field}} ... {{end}}` | Conditional |

**Quoting by shell** (important on Windows):

| Shell | Write it like this |
|---|---|
| Bash / zsh / PowerShell | `--format '{{.State.Status}}'` (single quotes) |
| cmd.exe | `--format "{{.State.Status}}"` (double quotes; single quotes are not quotes in cmd) |

Other things you can inspect: `docker inspect nginx:1.27` (image), `docker inspect myvolume`, `docker inspect bridge` (network). Use `--type container|image|volume|network` to disambiguate if names clash. `-s, --size` adds container size info.

To pipe into the `jq` tool (a JSON command-line processor): `docker inspect web | jq '.[0].State'`.

### 8.2 `docker top`

```bash
docker top web
```

```
UID     PID     PPID    C   STIME   TTY   TIME       CMD
root    48213   48190   0   09:13   ?     00:00:00   nginx: master process nginx -g daemon off;
101     48247   48213   0   09:13   ?     00:00:00   nginx: worker process
```

Lists processes **inside** the container, but displays the **host's PIDs** (inside, the master is PID 1). Remember: containers are just processes ([Chapter 1](01-what-is-docker.md#5-how-containers-actually-work)). You can pass `ps` options: `docker top web -ef`.

### 8.3 `docker stats`

```bash
docker stats
```

```
CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT    MEM %   NET I/O         BLOCK I/O     PIDS
3f4a1b9c2d7e   web    0.00%   4.512MiB / 15.34GiB  0.03%   1.2kB / 0B      0B / 12.3kB   5
```

A live-updating (refreshes about once per second) table of resource use per running container. Press Ctrl+C to quit.

| Column | Meaning |
|---|---|
| `CPU %` | CPU usage relative to the host's CPUs (can exceed 100% on multi-core: 200% = two full cores) |
| `MEM USAGE / LIMIT` | Memory used / the limit (host total if you set no `--memory`) |
| `MEM %` | Usage as a percentage of the limit |
| `NET I/O` | Network bytes received / sent |
| `BLOCK I/O` | Disk bytes read / written |
| `PIDS` | Number of processes/threads |

| Flag | Meaning |
|---|---|
| `--no-stream` | Print **one** snapshot and exit (use in scripts) |
| `-a, --all` | Include stopped containers |
| `--format` | Custom template |
| `--no-trunc` | Full IDs |

```bash
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
```

### 8.4 `docker events`

```bash
docker events
docker events --filter type=container --filter event=die
docker events --since 30m --until 1m
```

Streams real-time events from the daemon: every container create/start/die/stop, image pull, network connect, etc. Output sample:

```
2026-10-23T09:13:00.101010101Z container create 3f4a1b9c2d7e... (image=nginx:1.27, name=web)
2026-10-23T09:13:00.402020202Z container start 3f4a1b9c2d7e... (image=nginx:1.27, name=web)
2026-10-23T09:20:11.303030303Z container die 3f4a1b9c2d7e... (exitCode=0, image=nginx:1.27, name=web)
```

Flags: `-f, --filter` (e.g., `type=`, `event=`, `container=`, `image=`), `--since`, `--until`, `--format`. Use it to answer "*when* and *why* did my container die?" (the `die` event includes `exitCode`; `oom` events show memory kills).

### 8.5 `docker diff`

```bash
docker diff web
```

```
C /run
C /run/nginx.pid
A /tmp/debug.log
D /usr/share/nginx/html/old.html
```

Shows files changed in the container's **writable layer** relative to the image: `A` = added, `C` = changed, `D` = deleted. This is a powerful way to find *what a running program writes to disk* (so you know what to put on a volume or make writable under `--read-only`).

### 8.6 `docker port`

```bash
docker port web            # all mappings:  80/tcp -> 0.0.0.0:8080
docker port web 80         # just container port 80:  0.0.0.0:8080
```

Handy after `-P` (random ports) to discover which host port you got.

---

## 9. Exit codes

When a container's main process ends, it returns an **exit code** (a number from 0 to 255; by convention **0 = success**, anything else = some kind of failure). You see it in `docker ps -a` (`Exited (137)`), in `docker inspect --format '{{.State.ExitCode}}'`, from `docker wait`, and, for `docker run` in the foreground, in your shell's `$?` (Bash) or `$LASTEXITCODE` (PowerShell).

```bash
docker run --rm alpine sh -c 'exit 3'; echo "exit code: $?"
# exit code: 3
```

### 9.1 The table

| Code | Meaning | Typical cause and what to do |
|---|---|---|
| **0** | Success | Process finished normally. For a *service*, an exit 0 means it quit on its own (check why). |
| **1** | Generic application error | The program crashed/raised an unhandled error. **Read `docker logs`.** |
| **2** | Misuse of a shell built-in or bad CLI arguments | Often "wrong arguments" to a program. |
| **125** | **The `docker run` command itself failed** (the *Docker daemon* reported an error, not your program) | Bad flag, name already in use, port already allocated, invalid mount, cannot start container. Docker's docs: "Exit code 125 indicates that the error is with Docker daemon itself." The error message prints to your terminal. |
| **126** | The command inside the container **cannot be invoked** | File exists but is not executable (missing `chmod +x`), or is a directory, or wrong format. |
| **127** | **Command not found** inside the container | Typo, wrong `CMD`/`ENTRYPOINT`, the program is not installed in this image (e.g., `bash` on Alpine), or the PATH is wrong. |
| **130** | Terminated by **Ctrl+C** (SIGINT, signal 2: 128 + 2) | You interrupted it. |
| **134** | Aborted (SIGABRT, 6: 128 + 6) | Program called `abort()` (assertion failure, C/C++ crash). |
| **137** | **Killed by SIGKILL** (signal 9: 128 + 9) | (a) **Out of memory (OOM killer)**: the container exceeded its memory limit (or the host/VM ran out of RAM); (b) `docker kill` or `docker rm -f`; (c) `docker stop` timed out (process ignored SIGTERM for 10 s). **Check OOM:** `docker inspect --format '{{.State.OOMKilled}}' <name>`. |
| **139** | **Segmentation fault** (SIGSEGV, 11: 128 + 11) | Native-code crash: bad memory access, mismatched binary/library, incompatible architecture under emulation, buggy native extension. |
| **143** | **Terminated by SIGTERM** (signal 15: 128 + 15) | Normal outcome of `docker stop` when the app does not install its own handler (the process died from the signal) or exits with 143 after handling it. Usually fine. |
| **255** | Exit status out of range / unknown | Sometimes `ssh` errors or `exit -1`. |

**The rule of thumb:** codes **above 128** mean "killed by a signal", and the signal number is the **code minus 128**: 137 - 128 = 9 (SIGKILL); 143 - 128 = 15 (SIGTERM); 139 - 128 = 11 (SIGSEGV); 130 - 128 = 2 (SIGINT).

### 9.2 Reproducing the important ones (try these!)

```bash
# 0 and an application-defined code
docker run --rm alpine true;            echo "true   -> $?"   # 0
docker run --rm alpine sh -c 'exit 7';  echo "exit 7 -> $?"   # 7

# 125: Docker itself complains (unknown flag)
docker run --rm --bogus-flag alpine;    echo "bad flag -> $?" # 125, with "unknown flag" error

# 126: not executable (here, a directory)
docker run --rm alpine /etc;            echo "dir -> $?"      # 126

# 127: command not found
docker run --rm alpine nosuchprogram;   echo "missing -> $?"  # 127

# 137 via OOM: allocate 100 MB with a 20 MB limit
docker run --rm -m 20m --memory-swap 20m python:3.13-slim python -c "x = b'x' * (100*1024*1024); print('allocated')"
echo "oom -> $?"                                               # 137

# 137 via explicit kill, then confirm it
docker run -d --name victim alpine sleep 300
docker kill victim
docker inspect --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' victim   # exit=137 oom=false
docker rm victim
```

(PowerShell: replace `$?` with `$LASTEXITCODE`, and `echo "...$?"` with `Write-Host "... $LASTEXITCODE"`.)

For the OOM test, `OOMKilled` will be `true` if you keep the container (omit `--rm` and use `--name`) and inspect it. This distinction tells you *whether memory was the cause* (`OOMKilled=true`) or someone sent SIGKILL (`OOMKilled=false`).

> **On Docker Desktop**, an OOM kill can also come from the **VM** running out of memory, in which case `OOMKilled` may be `false` while the exit code is still 137; check your WSL/Desktop memory settings ([Chapter 2](02-installation-and-operating-systems.md#210-limit-wsl-2s-resources-with-wslconfig)).

---

## 10. Port publishing explained

### 10.1 Why you need it

A container has its own **network namespace**: its own IP address (like `172.17.0.2`) and its own port numbers, on a private internal network. Your browser on the host cannot just reach `172.17.0.2:80` reliably from outside (and from other machines, definitely not). **Publishing** a port tells Docker: "listen on a port on the *host*, and forward connections to a port in the *container*."

```
        YOUR MACHINE (the "host")
  +-------------------------------------------------------------+
  |                                                             |
  |   Browser --> localhost:8080                                |
  |                   |                                         |
  |            host port 8080  (opened by Docker: -p 8080:80)   |
  |                   |  forwards                               |
  |   +---------------v--------------------------------------+  |
  |   |  Container "web"  (own network namespace)            |  |
  |   |   container port 80  <-- nginx listens here          |  |
  |   |   container IP 172.17.0.2 (private)                  |  |
  |   +------------------------------------------------------+  |
  +-------------------------------------------------------------+

   -p  HOST_PORT : CONTAINER_PORT
        8080     :     80
        ^ "the door on the outside"     ^ "where the app listens inside"
```

**The order is always `host:container`.** A mnemonic: it reads left to right "from the outside in".

### 10.2 Examples

```bash
docker run -d --name web -p 8080:80 nginx:1.27
curl http://localhost:8080          # nginx welcome page HTML
```

Two containers cannot publish the **same host port**, but they can both use container port 80:

```bash
docker run -d --name web1 -p 8081:80 nginx:1.27
docker run -d --name web2 -p 8082:80 nginx:1.27
```

Error if the host port is taken:

```
docker: Error response from daemon: ... Bind for 0.0.0.0:8080 failed: port is already allocated.
```

(This is exit code 125, from [Section 9](#9-exit-codes).) Fix: pick a different host port or stop the other container.

### 10.3 Binding to localhost for safety

```bash
docker run -d -p 127.0.0.1:5432:5432 --name db -e POSTGRES_PASSWORD=learn-only postgres:17
```

`127.0.0.1:5432:5432` means: only the local machine can connect to the database. Without the `127.0.0.1:` prefix, Docker listens on `0.0.0.0` (all network interfaces), so anyone on your network (or the internet, on a public server) could try to reach it. **And remember: Docker's published ports bypass `ufw`/`firewalld` rules on Linux** ([Chapter 2](02-installation-and-operating-systems.md#49-firewall-caveat-important)). IPv6 loopback is written `[::1]:8080:80`.

### 10.4 `EXPOSE` vs `-p`

`EXPOSE 80` in a Dockerfile is **documentation only**: it records "this image's app listens on 80". It does **not** open anything to the host. Only `-p` (or `-P`, which publishes all `EXPOSE`d ports to random host ports) actually publishes.

### 10.5 The classic gotcha: listening on `127.0.0.1` *inside* the container

You publish `-p 8000:8000`, but the page will not load. Often the app inside is bound to **`127.0.0.1`** (loopback inside the container's own network namespace), which means "only connections from inside this very container". Docker's forwarded traffic arrives from the outside of that namespace, so it is refused. **Make your app listen on `0.0.0.0`** (all interfaces *inside* the container):

| Framework | Fix |
|---|---|
| FastAPI/uvicorn | `uvicorn main:app --host 0.0.0.0 --port 8000` |
| Flask | `flask run --host=0.0.0.0` |
| Node/Express | `app.listen(3000, '0.0.0.0')` |
| Streamlit | `streamlit run app.py --server.address=0.0.0.0` |
| Jupyter | `jupyter lab --ip=0.0.0.0 --no-browser` |

Binding to `0.0.0.0` *inside* the container is safe **as long as** you control exposure with the host side of `-p` (e.g., `127.0.0.1:8000:8000`). Diagnose by running `docker exec web ss -ltn` (or `netstat -ltn`) to see which address the app listens on.

---

## 11. Foreground vs detached

| Mode | Command | Behavior | Use when |
|---|---|---|---|
| **Foreground** | `docker run nginx` | The container's stdout/stderr stream to your terminal; the terminal is "stuck" until it exits. Ctrl+C sends SIGINT and usually stops the container. | Quick tests, watching startup, short jobs |
| **Detached** | `docker run -d nginx` | Starts in the background and prints the full container ID; you keep your prompt. View output with `docker logs`. | Services (web servers, databases, APIs) |
| **Interactive** | `docker run -it ubuntu bash` | stdin open + terminal: you are "inside" a shell. Exiting the shell ends PID 1 and thus the container. | Exploring, debugging, REPLs |
| **Detached + interactive** | `docker run -dit ubuntu bash` | Background container with a shell that you can `attach` to or `exec` into later | Long-lived shells |

**A container that exits immediately is almost always one whose main process finished.** For example, `docker run -d ubuntu` starts a container whose default command is `bash`, which sees no terminal/stdin and exits right away. Use `docker run -d ubuntu sleep infinity`, or `-dit`, when you need an idle container to poke at.

Foreground-mode extras: `docker run -a stdout alpine echo hi` uses `-a` to attach only certain streams. The signal-forwarding flag `--sig-proxy` (default true) forwards your Ctrl+C to the main process.

---

## 12. PID 1 and signals

### 12.1 What is a signal?

A **signal** is a small message the operating system delivers to a process, e.g., "please terminate" (SIGTERM), "stop right now" (SIGKILL), "user pressed Ctrl+C" (SIGINT), "reload your config" (SIGHUP). A process may *handle* most signals (run its own cleanup code). **SIGKILL and SIGSTOP cannot be handled; the kernel enforces them.**

| Signal | Number | Meaning | Catchable? |
|---|---|---|---|
| SIGHUP | 1 | Hangup / reload config (by convention) | Yes |
| SIGINT | 2 | Interrupt (Ctrl+C) | Yes |
| SIGQUIT | 3 | Quit with core dump (Ctrl+\\); nginx uses it for graceful stop | Yes |
| SIGKILL | 9 | Kill immediately | **No** |
| SIGSEGV | 11 | Segmentation fault (bad memory access) | Yes, rarely handled |
| SIGTERM | 15 | Polite terminate request (**what `docker stop` sends**) | Yes |

### 12.2 Why PID 1 is special

Inside a container, your main process is **PID 1**, and the Linux kernel treats PID 1 of a PID namespace specially:

1. **Signal handling:** for most signals, a PID 1 process has *no default action*. If a normal process receives SIGTERM and has not installed a handler, the kernel kills it. For PID 1, **the signal is simply ignored unless the program explicitly handles it.** So a Python script run as PID 1 with no `signal.signal(SIGTERM, ...)` will ignore `docker stop`'s SIGTERM. Docker waits 10 seconds, then SIGKILLs it: exit code **137**.
2. **Zombie reaping:** when a child process dies, its parent is supposed to collect its exit status ("reap" it); otherwise it lingers as a **zombie** (a dead process entry). On a normal host, PID 1 (init/systemd) adopts and reaps orphaned processes. In a container, **your app is PID 1** and may not know how to do this, so zombies can pile up.

### 12.3 Shell form vs exec form: the usual culprit

How you write the `CMD`/`ENTRYPOINT` in a Dockerfile (Chapter 4) decides who is PID 1:

| Dockerfile | What runs as PID 1 | Signals |
|---|---|---|
| `CMD python app.py` (**shell form**) | `/bin/sh -c "python app.py"`; Python is a *child* | `sh` does **not** forward SIGTERM to its child: `docker stop` hangs 10 s, then SIGKILL |
| `CMD ["python", "app.py"]` (**exec form**) | `python app.py` directly | Python receives SIGTERM directly (and must handle it) |

You can see this yourself:

```bash
docker run -d --name shellform alpine sh -c 'sleep 300'     # sh as PID 1... (alpine's sh may exec sleep directly)
docker top shellform
```

(Some shells optimize a single command with `exec`, so results vary; the general guidance holds.)

### 12.4 Demonstration: slow stop vs fast stop

```bash
# A process that IGNORES SIGTERM as PID 1 (a shell loop with no trap):
docker run -d --name slowstop alpine sh -c 'while true; do sleep 1; done'
time docker stop slowstop           # takes ~10 seconds!
docker inspect --format 'exit={{.State.ExitCode}}' slowstop    # exit=137  (SIGKILL after the timeout)
docker rm slowstop

# The same loop, but with a trap that handles SIGTERM:
docker run -d --name faststop alpine sh -c 'trap "echo bye; exit 0" TERM; while true; do sleep 1 & wait $!; done'
time docker stop faststop           # returns almost immediately
docker inspect --format 'exit={{.State.ExitCode}}' faststop    # exit=0
docker rm faststop
```

(`time` is a Bash/WSL/macOS builtin. In PowerShell use `Measure-Command { docker stop slowstop }`.)

### 12.5 The fixes

1. **Use exec form** in the Dockerfile: `CMD ["python", "app.py"]`.
2. **Handle SIGTERM** in your application (Python: `signal.signal(signal.SIGTERM, handler)`; Node: `process.on('SIGTERM', ...)`; frameworks like uvicorn/gunicorn/nginx already do).
3. **Use an init process:** `docker run --init ...` (or `init: true` in Compose) places Docker's tiny init (`docker-init`, which is `tini`) as PID 1. It forwards signals to your process and reaps zombies. A cheap, safe default for apps that spawn subprocesses.
4. If the app expects a different stop signal (nginx prefers SIGQUIT for graceful stop), set `--stop-signal` or the Dockerfile `STOPSIGNAL`. Give slow apps more time: `--stop-timeout 30` or `docker stop -t 30`.
5. For shell-script entrypoints, end with `exec "$@"` so the real program replaces the shell as PID 1.

---

## 13. Disk usage and cleanup: `system df` and `prune`

Docker accumulates data: images, stopped containers, volumes, build cache, logs. Disks fill up. Learn to look before you delete.

### 13.1 `docker system df`

```bash
docker system df
```

```
TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
Images          5         3         1.2GB     512MB (42%)
Containers      7         2         311.3kB   290kB (93%)
Local Volumes   7         1         2.4GB     2.1GB (87%)
Build Cache     13        0         133.8MB   30.72MB
```

| Column | Meaning |
|---|---|
| `TOTAL` | How many objects exist |
| `ACTIVE` | How many are in use (images used by a container; containers currently running; volumes attached to a container) |
| `SIZE` | Total disk consumed |
| `RECLAIMABLE` | What you could recover by removing unused objects |

`docker system df -v` gives a detailed per-image, per-container, per-volume listing, with a `CONTAINERS` count per image and `UNIQUE SIZE` vs `SHARED SIZE`. Use it to find the biggest items.

### 13.2 The prune family

| Command | Removes | Needs `-a`? |
|---|---|---|
| `docker container prune` | All **stopped** containers | n/a |
| `docker image prune` | **Dangling** images (untagged) | `-a` = all images **not used by any container** |
| `docker volume prune` | **Anonymous** unused volumes (by default) | `-a, --all` = *all* unused volumes, including **named** ones |
| `docker network prune` | Unused user-defined networks | n/a |
| `docker builder prune` | Build cache | `-a` includes more (internal/frontend images) |
| `docker system prune` | Stopped containers + unused networks + dangling images + (build cache) | `-a` = all unused images; `--volumes` = also anonymous volumes |

Common flags: `-f, --force` (do not ask for confirmation), `--filter "until=<duration>"` (only objects older than, e.g., `until=24h`, `until=168h`), `--filter label=...`.

**Example, interactive (shows a warning and asks y/N):**

```
$ docker system prune
WARNING! This will remove:
  - all stopped containers
  - all networks not used by at least one container
  - all dangling images
  - unused build cache

Are you sure you want to continue? [y/N] y
Deleted Containers:
3f4a1b9c2d7e...
Total reclaimed space: 311.3kB
```

### 13.3 Warnings (read these)

> **`docker system prune -a --volumes` (and `docker volume prune -a`) can destroy data you care about.**
> - **Volumes hold your databases.** A volume that no container currently uses is considered "unused," even if it holds your only copy of a PostgreSQL database. Pruning volumes deletes them **permanently with no undo**. In Docker 23+, plain `docker volume prune` and `docker system prune --volumes` only remove *anonymous* volumes, while `-a` on `volume prune` extends it to named volumes. Do not add `-a` to a volume prune unless you have looked at the list first.
> - **"Unused" images are re-downloadable but may not be reproducible** (e.g., an image you built locally from a Dockerfile you no longer have; or a tag that has since been replaced upstream).
> - Pruning **on a shared or production Docker host** can delete things other people need. Prefer `docker system df -v` first, then targeted removals.
> - **`-f` skips the confirmation prompt**, so never combine `-f` with a prune you have not previewed.
> - Stopped containers may contain logs and files you want for debugging. Prune only after you have looked.

**Safer habits:**

```bash
docker system df                                  # 1. look first
docker ps -a                                      # 2. see which containers exist
docker volume ls                                  # 3. see volumes
docker image prune                                # 4. start with the least risky (dangling images)
docker container prune --filter "until=24h"       # 5. remove only stale stopped containers
docker builder prune --filter "until=168h"        # 6. trim old build cache
```

Recovering from disk-full on a build machine usually means `docker builder prune` (build cache is often the biggest consumer) and `docker image prune -a --filter "until=720h"`.

---

## 14. Registries: `login`, `logout`, `search`

### 14.1 `docker login`

```bash
docker login                       # Docker Hub (prompts for username and password/token)
docker login ghcr.io               # GitHub Container Registry
docker login myregistry.example.com
docker logout                      # remove stored credentials for Docker Hub
docker logout ghcr.io
```

| Flag | Meaning |
|---|---|
| `-u, --username` | Username |
| `-p, --password` | Password or token **(discouraged: appears in your shell history and process list)** |
| `--password-stdin` | Read the password/token from standard input (**recommended in scripts**) |

**The safe way:**

```bash
echo "$MY_TOKEN" | docker login ghcr.io -u yourname --password-stdin
```

Output: `Login Succeeded`. What happens: Docker stores credentials in `~/.docker/config.json` (on Windows, `C:\Users\<you>\.docker\config.json`), ideally via a **credential helper** (a program that stores secrets in the OS keychain: Windows Credential Manager, macOS Keychain, Linux `pass`/secretservice) so no plain-text password sits in the file. Check `config.json` for a `credsStore` entry. If it is missing on a Linux server, credentials are stored **base64-encoded (not encrypted)**, so protect that file or configure a helper.

Use **personal access tokens (PATs)** or registry-specific tokens rather than your account password, with the minimum scope (read-only for pulls). Logging in raises your Docker Hub pull-rate limit ([Chapter 1](01-what-is-docker.md#91-docker-hub)). CI systems should use short-lived or scoped tokens stored in the CI's secret store.

### 14.2 `docker search` (again)

See [3.10](#310-docker-search). For private registries you must know the repository names.

---

## 15. Contexts: talking to other Docker hosts

A **Docker context** is a saved, named connection to a Docker daemon (local Docker Desktop, a remote server over SSH, a cloud endpoint). The CLI sends commands to whichever context is **active**.

```bash
docker context ls
```

```
NAME              DESCRIPTION                               DOCKER ENDPOINT                             ERROR
default           Current DOCKER_HOST based configuration   npipe:////./pipe/docker_engine
desktop-linux *   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```

The `*` marks the active context. (On Linux the endpoints are like `unix:///var/run/docker.sock`; on Mac, a Unix socket under your home directory.)

| Command | Meaning |
|---|---|
| `docker context ls` | List contexts |
| `docker context create NAME --docker "host=ssh://user@host"` | Create a context pointing at a remote daemon over SSH |
| `docker context use NAME` | Make it the active context for all later commands |
| `docker context inspect NAME` | Show details |
| `docker context rm NAME` | Delete a context (not the remote server!) |
| `docker --context NAME ps` | Use a context for **one command** only |
| `DOCKER_CONTEXT=NAME` (environment variable) | Select a context for a shell session |

```bash
docker context create staging --docker "host=ssh://deploy@203.0.113.10"
docker --context staging ps                # list containers on the staging server
docker context use staging                 # everything now targets it
docker context use desktop-linux           # back to local (use "default" on Linux servers)
```

Why SSH: no extra open daemon port, key-based authentication, encryption. The remote machine needs Docker Engine, the SSH user must be allowed to run `docker`, and your SSH keys must work non-interactively (`ssh-agent`). A safety habit: **always check which context is active before running destructive commands.** `docker rm -f $(docker ps -aq)` on the wrong context is a memorable mistake. Many people put the context name in their shell prompt.

---

## 16. Building: `docker build`, `builder`, and `buildx` (intro)

Chapter 4 is the deep dive; here is what you need to recognize.

```bash
docker build -t myapp:1.0 .
```

| Piece | Meaning |
|---|---|
| `docker build` | Build an image from a **Dockerfile** |
| `-t myapp:1.0` | **Tag** the result (`name:tag`) |
| `.` | The **build context**: the folder whose contents are sent to the builder (Docker looks for `Dockerfile` here by default) |
| `-f path/Dockerfile` | Use a different Dockerfile |
| `--build-arg KEY=VALUE` | Pass a build-time variable |
| `--no-cache` | Ignore the build cache; rebuild everything |
| `--target stage` | Build only up to a named stage of a multi-stage Dockerfile |
| `--platform linux/amd64,linux/arm64` | Build for specific platforms |

**BuildKit and buildx.** Since Docker Engine 23.0, **BuildKit** is the default build engine for Linux containers. Docker's current docs confirm BuildKit is "the default builder for Docker Desktop and Docker Engine users", with the legacy builder still used for **Windows container** builds. **Buildx** is the CLI plugin that exposes BuildKit's full feature set:

```bash
docker buildx build -t myapp:1.0 .                                    # same idea, full feature set
docker buildx build --platform linux/amd64,linux/arm64 -t myorg/myapp:1.0 --push .   # multi-arch + push
docker buildx ls                                                      # list builder instances
docker builder ls                                                     # same listing via `docker builder`
docker builder prune                                                  # clear build cache
docker buildx bake                                                    # build several targets defined in a file (docker-bake.hcl / compose)
```

Example `docker buildx ls` output seen on the test machine:

```
NAME/NODE           DRIVER/ENDPOINT     STATUS    BUILDKIT   PLATFORMS
default             docker
 \_ default          \_ default         running   v0.34.0    linux/amd64 (+3), linux/arm64, linux/arm (+2), linux/ppc64le, (2 more)
desktop-linux*      docker
 \_ desktop-linux    \_ desktop-linux   running   v0.34.0    linux/amd64 (+3), linux/arm64, ...
```

Reading it: each **builder instance** (with a BuildKit version) lists the platforms it can build for. `docker builder` is a set of commands for managing builds (`docker builder prune`, `docker builder ls`...). `docker buildx` has more (create builders, `bake`, `imagetools`).

`docker init` (a plugin) can generate a starter `Dockerfile`, `.dockerignore`, and `compose.yaml` for your project: a nice way to start.

---

## 17. Scanning images: `docker scout`, Trivy, and friends

Images contain operating-system packages and libraries that may have known vulnerabilities (**CVEs**: Common Vulnerabilities and Exposures, public IDs like CVE-2025-12345). **Image scanners** read an image's contents, match packages against vulnerability databases, and report findings.

| Tool | How | Notes |
|---|---|---|
| **Docker Scout** | `docker scout quickview nginx:1.27`, `docker scout cves nginx:1.27`, `docker scout recommendations nginx:1.27` | Built into Docker's CLI/Desktop (a plugin). Free tier available; some features need a subscription: check Docker's current plans. |
| **Trivy** (Aqua Security, open source) | `trivy image nginx:1.27` | Very popular, fast, scans OS packages and language dependencies, plus IaC and secrets. |
| **Grype** (Anchore, open source) | `grype nginx:1.27` | Another open-source scanner. |
| **Snyk** | `snyk container test nginx:1.27` | Commercial with free tier. |

```bash
docker scout quickview nginx:1.27      # summary counts by severity (critical/high/medium/low)
docker scout cves nginx:1.27           # detailed vulnerability list
trivy image --severity HIGH,CRITICAL nginx:1.27
```

These are introduced here for awareness. Interpreting results, fixing them (smaller base images, updates, multi-stage builds), generating SBOMs, signing images, and gating CI are covered in [Chapter 10](10-security.md). (Whether a particular Scout feature is free or paid changes over time, and I did not verify it. Check Docker's pricing page.)

---

## 18. Other commands and plugins you will see

Run `docker --help` and you will see many entries. The list on the test machine (Docker Desktop 4.96.3) includes these **plugins**, shown by `docker --help` under "Management Commands" (marked with `*`):

| Command | Description from `docker --help` | Covered in |
|---|---|---|
| `docker compose` | Docker Compose | [Chapter 6](06-docker-compose.md) |
| `docker buildx` | Docker Buildx | Chapter 4 |
| `docker init` | Creates Docker-related starter files for your project | Chapter 4 |
| `docker scout` | Docker Scout | Chapter 10 |
| `docker model` | Docker Model Runner (run AI models locally) | [Chapter 8](08-docker-for-ai-and-ml.md) |
| `docker debug` | Get a shell into any image or container (even ones with no shell) | Debugging |
| `docker desktop` | Docker Desktop commands | -- |
| `docker extension` | Manage Docker Desktop extensions | -- |
| `docker ai` | Docker AI Agent: "Ask Gordon" | -- |
| `docker mcp` | Docker MCP Plugin (Model Context Protocol tooling) | Chapter 8 |
| `docker agent` | Docker AI Agent Runner | Chapter 8 |
| `docker offload` | Docker Offload (run builds/containers on remote cloud resources) | -- |
| `docker dhi` | Manage Docker Hardened Images | Chapter 10 |
| `docker pass` | Docker Pass Secrets Manager Plugin (beta) | Chapter 10 |

I list these because they appear in your `docker --help` and I verified the one-line descriptions from the live output. **I have not verified the availability, pricing, or maturity of each** (they are newer and some are marked beta); treat them as "things to look into" and follow Docker's docs for each.

Other built-in management commands: `docker volume` (Chapter 5), `docker network` (Chapter 5), `docker system` (`df`, `prune`, `events`, `info`), `docker manifest` (inspect multi-platform manifests), `docker plugin`, `docker swarm` (Docker's built-in clustering mode; see Chapter 11 for orchestration choices), `docker bake`.

Also worth knowing: `docker run --use-api-socket` (a newer flag listed in `docker run --help`: "Bind mount Docker API socket and required auth"), which gives the container access to the Docker API: a powerful capability that must be treated with the same caution as mounting `/var/run/docker.sock` ([Chapter 10](10-security.md)).

---

## 19. Hands-on lab

Everything here is safe to run: it uses throwaway containers with unique names and cleans up at the end. **Outputs below are examples; IDs, times, and random names will differ.** If a name or port collides with something on your machine, change it. In PowerShell, use backticks instead of `\` for line continuation, or write commands on one line.

### Lab 0: sanity check

```bash
docker version --format '{{.Server.Version}}'
docker ps -a
```

You should see a version number and a (possibly empty) list.

### Lab 1: run nginx, view it in the browser, read its logs

```bash
docker run -d --name lab-web -p 8080:80 nginx:1.27
```

Output: a long container ID like `3f4a1b9c2d7e8f...`. Flags: `-d` detached, `--name lab-web` for easy reference, `-p 8080:80` host 8080 to container 80, then image `nginx:1.27`. The first time, Docker pulls the image (you may see `Unable to find image ... locally` and layer downloads).

```bash
docker ps
```

```
CONTAINER ID   IMAGE        COMMAND                  CREATED         STATUS         PORTS                  NAMES
3f4a1b9c2d7e   nginx:1.27   "/docker-entrypoint.…"   5 seconds ago   Up 4 seconds   0.0.0.0:8080->80/tcp   lab-web
```

Now request the page. Open `http://localhost:8080` in a browser, or:

```bash
curl http://localhost:8080
```

(PowerShell: `curl.exe http://localhost:8080`, or `Invoke-WebRequest http://localhost:8080 | Select-Object -ExpandProperty Content`. Plain `curl` in Windows PowerShell 5.1 is an alias for `Invoke-WebRequest`.)

Expected: the HTML containing `<title>Welcome to nginx!</title>`.

```bash
docker logs lab-web
```

You will see nginx's startup messages and one access-log line for your request, e.g. `... "GET / HTTP/1.1" 200 615 ...`. Try following live:

```bash
docker logs -f --tail 5 lab-web
```

Refresh the browser and watch new lines appear. Press Ctrl+C (this stops *following*, not the container).

**What you learned:** `run -d`, `-p`, `ps`, `logs -f --tail`.

### Lab 2: look inside nginx

```bash
docker exec lab-web nginx -v
```

Output: `nginx version: nginx/1.27.x` (written to stderr, which is why it shows even though it is a "version" command).

```bash
docker exec -it lab-web bash
```

You get a prompt like `root@3f4a1b9c2d7e:/#`. Inside, try:

```bash
ls /usr/share/nginx/html        # the web root: contains index.html
cat /etc/nginx/conf.d/default.conf | head -n 20
hostname                        # equals the container ID prefix (UTS namespace)
ps aux                          # (may say "ps: not found" in slim images; nginx images are Debian-based but minimal)
exit                            # leaves the shell; the container keeps running
```

Verify it is still running: `docker ps`. Now copy a file out and change the page:

```bash
docker cp lab-web:/usr/share/nginx/html/index.html ./index.html
```

Edit `./index.html` (change the heading text to "Hello from my lab!"), then copy it back:

```bash
docker cp ./index.html lab-web:/usr/share/nginx/html/index.html
curl http://localhost:8080
```

Your edited page appears. Then see what changed in the container:

```bash
docker diff lab-web
```

```
C /usr
C /usr/share
C /usr/share/nginx
C /usr/share/nginx/html
C /usr/share/nginx/html/index.html
C /run
A /run/nginx.pid
...
```

`C` = changed, `A` = added. This demonstrates **copy-on-write** from [Chapter 1](01-what-is-docker.md#53-union--overlay-filesystems-what-files-does-this-process-have): your edit lives only in this container's writable layer.

Prove it is ephemeral: remove the container and start a fresh one, and the edit is gone.

```bash
docker rm -f lab-web                     # -f: stop (kill) and remove in one go
docker run -d --name lab-web -p 8080:80 nginx:1.27
curl http://localhost:8080               # back to "Welcome to nginx!"
```

### Lab 3: inspect and monitor

```bash
docker inspect --format 'status={{.State.Status}} pid={{.State.Pid}} image={{.Config.Image}} restart={{.HostConfig.RestartPolicy.Name}}' lab-web
```

Expected shape: `status=running pid=48213 image=nginx:1.27 restart=no` (PID is a host PID).

```bash
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' lab-web
docker top lab-web
docker stats --no-stream
docker port lab-web
```

You get the container's private IP (like `172.17.0.2`), its processes (nginx master + workers), a one-time resource snapshot, and the port mapping `80/tcp -> 0.0.0.0:8080`.

### Lab 4: an interactive Python container

```bash
docker run -it --rm --name lab-py python:3.13-slim python
```

Flags: `-it` interactive terminal, `--rm` delete the container when it exits, `python:3.13-slim` the image; the trailing `python` is the command (it overrides the image default so we get the REPL). You get Python's `>>>` prompt. Try:

```python
>>> import sys, platform, os
>>> sys.version
>>> platform.platform()
>>> os.getpid()
1
>>> exit()
```

`os.getpid()` returning `1` proves that Python is **PID 1** in its own PID namespace. Now run a script from your host **without building an image**, using a bind mount:

```bash
mkdir -p lab-scripts && cd lab-scripts
printf 'import platform\nprint("hello from", platform.machine(), "inside a container")\n' > hello.py
docker run --rm -v "$(pwd)":/app -w /app python:3.13-slim python hello.py
```

Output: `hello from x86_64 inside a container` (or `aarch64` on ARM). Flags: `-v "$(pwd)":/app` mounts the current folder at `/app`; `-w /app` sets the working directory; the command runs `hello.py` from your host folder. (PowerShell: `-v ${PWD}:/app`.) This "mount your code, run it in a clean, consistent environment" pattern is the foundation of containerized development. `cd ..` when done.

### Lab 5: run a database (PostgreSQL) with persistent data

```bash
docker run -d --name lab-db \
  -e POSTGRES_USER=labuser \
  -e POSTGRES_PASSWORD=lab-password-do-not-reuse \
  -e POSTGRES_DB=labdb \
  -p 127.0.0.1:5433:5432 \
  -v lab-pgdata:/var/lib/postgresql/data \
  postgres:17
```

| Flag | Meaning |
|---|---|
| `-e POSTGRES_USER/PASSWORD/DB` | Initial configuration, read by the image's entrypoint on first start. (The password is a **demo value for a throwaway container**; real secrets never go in commands or Git.) |
| `-p 127.0.0.1:5433:5432` | Host port 5433 (localhost only) to Postgres's 5432. Using 5433 avoids clashing with a Postgres you may already have. |
| `-v lab-pgdata:/var/lib/postgresql/data` | A **named volume** `lab-pgdata` (created automatically) holds the database files so they survive container removal. |
| `postgres:17` | Official image, major version 17 (pinned on purpose). |

> **Note on newer Postgres images:** some newer major versions of the official Postgres image changed their default data directory layout, which changes the recommended mount path. This lab pins version 17 and the path above; if you move to a newer major version, read that image's Docker Hub page first. I did not verify the details for versions newer than 17.

Watch it start:

```bash
docker logs -f lab-db
```

Wait for the line `database system is ready to accept connections` (it appears twice: once for the temporary init server and once for the real one). Press Ctrl+C.

Check health from the outside:

```bash
docker exec lab-db pg_isready -U labuser -d labdb
```

Output: `/var/run/postgresql:5432 - accepting connections`.

### Lab 6: exec into the database and prove persistence

```bash
docker exec -it lab-db psql -U labuser -d labdb
```

Inside `psql` (the PostgreSQL command-line client; the prompt looks like `labdb=#`):

```sql
CREATE TABLE notes (id serial PRIMARY KEY, body text);
INSERT INTO notes (body) VALUES ('docker volumes persist data');
SELECT * FROM notes;
\q
```

The `SELECT` prints one row. Now **destroy the container but keep the volume**, then create a new container using the same volume:

```bash
docker rm -f lab-db
docker volume ls                       # lab-pgdata is still there

docker run -d --name lab-db \
  -e POSTGRES_USER=labuser -e POSTGRES_PASSWORD=lab-password-do-not-reuse -e POSTGRES_DB=labdb \
  -p 127.0.0.1:5433:5432 -v lab-pgdata:/var/lib/postgresql/data postgres:17

sleep 5   # (PowerShell: Start-Sleep 5) give it a moment
docker exec lab-db psql -U labuser -d labdb -c "SELECT * FROM notes;"
```

Your row is still there: the new container found existing data in the volume. (This time the log says `PostgreSQL Database directory appears to contain a database; Skipping initialization`: check with `docker logs lab-db | head`.) **That is what volumes are for.** Without `-v`, the data would have vanished with the container.

### Lab 7: inspect resource limits and cause an OOM (exit 137)

```bash
docker run -d --name lab-limited --memory 256m --cpus 0.5 nginx:1.27
docker stats --no-stream lab-limited
```

In the `MEM USAGE / LIMIT` column you will see `.../256MiB`, proving the cgroup limit is applied. Now trigger an out-of-memory kill deliberately:

```bash
docker run --name lab-oom -m 20m --memory-swap 20m python:3.13-slim \
  python -c "x = b'x' * (100*1024*1024); print('allocated')"
echo "exit code: $?"          # PowerShell: $LASTEXITCODE
docker inspect --format 'exit={{.State.ExitCode}} oomkilled={{.State.OOMKilled}}' lab-oom
```

Expected: no `allocated` printed, `exit code: 137`, and `exit=137 oomkilled=true`. The kernel killed the process for exceeding 20 MB. (On some Docker Desktop setups the kill message can differ slightly; the exit code 137 is the reliable signal.)

### Lab 8: exit codes and signals

```bash
docker run --rm alpine sh -c 'exit 3'; echo "code=$?"
docker run --rm alpine nosuchprogram;  echo "code=$?"
docker run --rm --bogus alpine;        echo "code=$?"
```

Expected: `code=3`, `code=127` (with a "not found / executable file not found in $PATH" error from Docker), and `code=125` (with `unknown flag: --bogus`). Now the stop-timing test from [12.4](#124-demonstration-slow-stop-vs-fast-stop):

```bash
docker run -d --name lab-slow alpine sh -c 'while true; do sleep 1; done'
time docker stop lab-slow            # about 10 seconds
docker inspect --format '{{.State.ExitCode}}' lab-slow    # 137
docker rm lab-slow
```

### Lab 9: lifecycle and events

In terminal A:

```bash
docker events --filter type=container
```

In terminal B:

```bash
docker run -d --name lab-life alpine sleep 600
docker pause lab-life
docker unpause lab-life
docker stop lab-life
docker start lab-life
docker rm -f lab-life
```

Watch terminal A print `create`, `start`, `pause`, `unpause`, `kill`, `die`, `stop`, `start`, `kill`, `die`, `destroy`, each with the container name. Press Ctrl+C in terminal A. **This is the lifecycle diagram, live.**

### Lab 10: look at disk usage, then clean up

```bash
docker system df
docker ps -a --filter "name=lab-"
docker volume ls
```

Now remove **only the things you created in this lab** (explicit names, no broad prune):

```bash
docker rm -f lab-web lab-db lab-limited lab-oom 2>/dev/null     # PowerShell: add  2>$null  or just ignore errors
docker volume rm lab-pgdata
docker rmi python:3.13-slim postgres:17 nginx:1.27 alpine       # remove the images pulled in this lab (skip any you want to keep)
rm -rf lab-scripts index.html                                    # PowerShell: Remove-Item -Recurse -Force lab-scripts, index.html
```

Verify nothing is left:

```bash
docker ps -a --filter "name=lab-"
docker volume ls --filter "name=lab-"
```

Both should be empty. If `docker rmi` says an image is "being used by stopped container ..." or "must be forced," look at `docker ps -a` for the container that references it; if it is a container of yours, remove it; if it belongs to something else, **leave that image alone.**

### Lab summary: what you practiced

| Skill | Commands |
|---|---|
| Run services | `run -d --name -p`, `ps`, `logs -f --tail` |
| Get inside | `exec -it`, `cp`, `diff`, `top`, `stats` |
| Configuration | `-e`, `-v` (named volume and bind mount), `-w`, `--rm` |
| Persistence | Named volume survived container deletion |
| Limits and failure | `--memory`, exit code 137, `OOMKilled` |
| Exit codes | 0, 3, 125, 127, 137 |
| Signals | `docker stop` 10 s timeout, PID 1 ignoring SIGTERM |
| Lifecycle | `pause`, `unpause`, `stop`, `start`, `rm`, `events` |
| Cleanup | Explicit `rm`, `volume rm`, `rmi` |

---

## 20. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Putting flags **after** the image name (`docker run nginx -d -p 80:80`) | Flags are passed to the container's program; nothing detaches or publishes | All Docker options go **before** the image name |
| 2 | Wrong port order (`-p 80:8080` when the app listens on 80) | Connection reset / empty reply | `-p HOST:CONTAINER`. The container port is where the app listens |
| 3 | App listens on `127.0.0.1` inside the container | Published port does not respond | Bind to `0.0.0.0` ([10.5](#105-the-classic-gotcha-listening-on-127001-inside-the-container)) |
| 4 | Expecting `docker restart` or `start` to pick up a new image or flags | Old behavior persists | `docker rm -f` and `docker run` again (or use Compose) |
| 5 | Storing data in the container filesystem | Data lost on `docker rm` | Use a named volume ([Lab 6](#lab-6-exec-into-the-database-and-prove-persistence)) |
| 6 | Using `docker run` repeatedly without `--rm` or cleanup | Dozens of stopped containers | `--rm` for throwaway runs; `docker container prune` |
| 7 | `docker run -d ubuntu` and "it exits immediately" | Container shows `Exited (0)` | The default command (bash) has nothing to do. Use `sleep infinity` or `-dit`, or run a real service |
| 8 | Name conflict (`Conflict. The container name ... is already in use`) | `docker run` fails (exit 125) | `docker rm` the old one or pick a new name |
| 9 | `docker exec` on a stopped container | `Error response from daemon: container ... is not running` | `docker start` first (or `docker logs` for crash analysis) |
| 10 | Using `latest` and assuming it never changes | Surprise breaking changes | Pin versions (`nginx:1.27`) or digests |
| 11 | `docker system prune -a --volumes` out of habit | Lost databases | Never run it blind; look first ([13.3](#133-warnings-read-these)) |
| 12 | `-p 5432:5432` for a database on a public server | Database exposed to the internet | Bind to localhost (`127.0.0.1:5432:5432`) or use an internal network |
| 13 | Passing secrets with `-e SECRET=...` and committing scripts | Secrets leak via shell history, `docker inspect`, process lists, git | Use `-e NAME` (copy from environment), secret managers, or Compose/Swarm secrets (Chapter 10). Anyone who can run `docker inspect` can read env vars |
| 14 | Ctrl+C in `docker attach` kills the container | Service stops | Detach with Ctrl+P, Ctrl+Q; prefer `docker logs -f` |
| 15 | Forgetting `-it` for `exec bash` | `the input device is not a TTY` or no prompt | `docker exec -it NAME bash` |
| 16 | `docker exec -it NAME bash` on Alpine | `executable file not found` (no bash) | Use `sh` |
| 17 | Not checking the active **context** | Commands hit a remote server | `docker context ls` before destructive commands |
| 18 | Misreading `docker stop` taking 10 s | PID 1 ignores SIGTERM | Exec-form `CMD`, handle signals, or `--init` ([Section 12](#12-pid-1-and-signals)) |
| 19 | Mounting a non-existent host path with `-v` | Docker creates an empty **directory** with that name | Use `--mount` (errors instead), or verify paths |
| 20 | PowerShell/cmd quoting for `--format` | `template parsing error` | Single quotes in Bash/PowerShell; double quotes in cmd |

---

## 21. Cheat sheet

```text
INFO
  docker version | docker info | docker context ls | docker --help | docker <cmd> --help

IMAGES
  docker pull IMG[:TAG]            docker images [-a] [-q]       docker image ls --tree
  docker history IMG               docker image inspect IMG      docker tag SRC DST
  docker push NAME[:TAG]           docker rmi IMG                docker save -o f.tar IMG
  docker load -i f.tar             docker image prune [-a]       docker search TERM

RUN
  docker run [-d] [-it] [--rm] [--name N] [-p H:C] [-e K=V | --env-file F]
             [-v VOL:/path | --mount type=...] [--network NET] [--restart POLICY]
             [-m 512m] [--cpus 1.5] [-u UID:GID] [--read-only] [-w DIR]
             [--entrypoint X] [--platform P] [--gpus all] [--init] IMAGE [CMD...]

OBSERVE
  docker ps [-a] [-q] [--filter ...] [--format ...]
  docker logs [-f] [--tail N] [--since 10m] [-t] NAME
  docker top NAME | docker stats [--no-stream] | docker events | docker diff NAME
  docker port NAME | docker inspect [--format '{{...}}'] NAME

INTERACT
  docker exec [-it] NAME CMD       docker attach NAME (detach: Ctrl+P Ctrl+Q)
  docker cp NAME:/path ./local     docker cp ./local NAME:/path

LIFECYCLE
  docker create | start | stop [-t N] | restart | kill [-s SIG] | pause | unpause
  docker rm [-f] [-v] NAME         docker rename OLD NEW        docker wait NAME
  docker update --restart P --memory M NAME

CLEANUP (look first!)
  docker system df [-v]
  docker container prune | docker image prune [-a] | docker volume prune | docker network prune
  docker builder prune | docker system prune [-a] [--volumes]   <-- dangerous

REGISTRY / CONTEXT / BUILD
  docker login [REG] --password-stdin | docker logout [REG]
  docker context ls|create|use|rm  | docker --context NAME ...
  docker build -t NAME:TAG . | docker buildx build --platform ... --push .

EXIT CODES
  0 ok | 1 app error | 125 docker error | 126 not executable | 127 not found
  130 SIGINT | 137 SIGKILL/OOM | 139 SIGSEGV | 143 SIGTERM   (>128 = 128 + signal)
```

---

## 22. Check your understanding

1. What is the difference between `docker run` and `docker exec`?
2. What does `-p 9000:3000` do? Which number is the container's port?
3. You run `docker run -d --name api myapp` and then `docker ps` shows nothing. What command do you run first to find out why, and what might `Exited (127)` tell you?
4. `docker stop` always takes ~10 seconds for your service and the exit code is 137. What is happening and what are two fixes?
5. What is the difference between `docker rm` and `docker rmi`?
6. Why does deleting a container lose your data, and how do you keep it?
7. What does `docker inspect --format '{{.State.OOMKilled}}' NAME` tell you?
8. Name three differences between `-v` and `--mount`.
9. You want to see how much disk Docker is using and what is safe to delete. Which commands do you run, in order?
10. Why can `docker system prune -a --volumes` be dangerous?
11. In `docker run -it alpine sh -c "echo hi"`, which part is the image and which is the command?
12. Which exit code means the `docker run` command itself failed (not your program)?

<details>
<summary>Answers</summary>

1. `run` creates a new container from an image and starts it; `exec` runs an extra process inside an already-running container.
2. It forwards host port 9000 to container port 3000 (host:container). The container port is 3000.
3. `docker ps -a` to see the stopped container, then `docker logs api`. Exit 127 means "command not found" inside the container (wrong CMD/ENTRYPOINT, or the program is not installed in the image).
4. PID 1 is ignoring SIGTERM, so Docker waits the full timeout and SIGKILLs it (137). Fixes: exec-form CMD, handle SIGTERM in the app, use `--init`, or use a proper entrypoint script that `exec`s the program.
5. `rm` deletes containers; `rmi` (`image rm`) deletes images.
6. A container's writable layer is deleted with it. Use volumes or bind mounts for data that must persist.
7. Whether the kernel killed the container for exceeding its memory limit.
8. `--mount` is explicit (key=value), errors if a bind-mount source path does not exist (`-v` silently creates a directory), and is more readable; `-v` is shorter and positional.
9. `docker system df` (and `-v`), then `docker ps -a`, `docker volume ls`, then targeted prunes such as `docker image prune` and `docker builder prune`.
10. It removes all unused images and can delete volumes (databases) that are not currently attached to a container, permanently, with no confirmation if `-f` is used.
11. The image is `alpine`; the command is `sh -c "echo hi"`. `-it` is an option before the image.
12. 125.

</details>

---

**Next:** [Chapter 4: Dockerfile Deep Dive](04-dockerfile-deep-dive.md): write your own images, use the build cache well, and build small, secure images.

**Previous:** [Chapter 2: Installation and Operating Systems](02-installation-and-operating-systems.md)
