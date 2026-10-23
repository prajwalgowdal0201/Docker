# Chapter 3: Core CLI Commands

This chapter covers the Docker commands used daily: images, containers, logs, files, resources, and cleanup. Each command is followed by its main flags. It also covers `docker run` in detail, the container lifecycle, exit codes, PID 1 and signals, `docker inspect`, cleanup, registries, contexts, a first look at builds, and a hands-on lab.

**Prerequisites:** Docker installed ([Chapter 2](02-installation-and-operating-systems.md)) and the concepts in [Chapter 1](01-what-is-docker.md).

**About the sample output.** Hashes, container IDs, timestamps, random names, and sizes differ per machine. Formats come from Docker Engine 29.9.2 (API 1.56), and flag lists were checked against that version's `--help` output. Docker 29 changed some output (notably the `docker images` layout), noted where relevant. `docker <command> --help` is authoritative for the installed version.

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

---

## 1. How Docker commands are structured

### 1.1 The grammar: `docker <object> <verb>`

```
docker  [global options]  <object>  <verb>  [options]  [arguments]
  |            |             |         |        |           |
  |            |             |         |        |           +-- e.g. the container name or image
  |            |             |         |        +-- flags like -f, --all
  |            |             |         +-- action: ls, rm, inspect, prune, create ...
  |            |             +-- target: container, image, volume, network, builder, context...
  |            +-- e.g. --context, --host, --debug
  +-- the CLI program
```

`docker container ls` lists containers. `docker image rm nginx` removes the image `nginx`. The verbs (`ls`, `rm`, `inspect`, `prune`) behave the same across object types.

### 1.2 The old short forms still work

Early Docker versions had flat commands (`docker ps`, `docker images`, `docker rm`, `docker rmi`). They remain as **aliases** (alternative names) for the structured commands and appear in most tutorials.

| Short form | Full form | What it does |
|---|---|---|
| `docker run` | `docker container run` | Create and start a container |
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

This guide uses the short forms.

### 1.3 Getting help

```bash
docker --help                  # list all commands
docker run --help              # all flags for 'run'
docker container --help        # all container sub-commands
docker image ls --help
```

`--help` output is authoritative for the installed version.

### 1.4 Global options

Global options go after `docker` and before the command.

| Option | Meaning |
|---|---|
| `-H, --host <url>` | Connect to a specific daemon (for example `-H ssh://user@host`) |
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
#      |   |          |        |           |            |                   |         +-- COMMAND and ARGS (override the image default)
#      |   |          |        |           |            |                   +-- IMAGE[:TAG]
#      |   |          |        |           |            +-- OPTIONS (flags) for docker run
#      +---+----------+--------+-----------+---- everything between 'run' and IMAGE is an option
```

After `docker run`, everything up to the image name is an option for Docker. Everything after the image name is the command and arguments to run inside the container. `docker run nginx -d` does not detach: it passes `-d` to the container's program.

Single-letter flags combine: `-it` = `-i -t`, `-dit` = `-d -i -t`. Long flags use double dashes: `--name web` or `--name=web`.

### 1.6 Terminal differences (Bash vs PowerShell vs cmd)

| Task | Bash / WSL / macOS / Linux | PowerShell (Windows) | cmd.exe |
|---|---|---|---|
| Continue a command on the next line | `\` at line end | `` ` `` (backtick) at line end | `^` at line end |
| Current directory in a path | `"$(pwd)"` or `$PWD` | `${PWD}` | `%cd%` |
| Environment variable in the shell | `$HOME` | `$env:USERPROFILE` | `%USERPROFILE%` |
| Single quotes | Literal text | Literal text | Not quotes. Use double quotes. |

Examples in this guide use Bash line continuation (`\`). In PowerShell, replace `\` with a backtick or use one line.

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
| `Client` / `Server` | Two separate programs ([Chapter 1](01-what-is-docker.md#8-docker-architecture-what-runs-when-you-type-docker-run)). `Client:` plus an error means the daemon is unreachable. |
| `API version` | REST API version between them. `minimum version 1.40` means the daemon still serves clients using API 1.40 or newer (Docker's 29.x release notes say 29.3.0 lowered the minimum to 1.40). |
| `OS/Arch` | Platform of each. Here the client is Windows and the engine is Linux (in a VM). |
| `Context` | The Docker endpoint the client uses ([Section 15](#15-contexts-talking-to-other-docker-hosts)). |
| `containerd`, `runc`, `docker-init` | Component versions in the chain. |

Variants:

```bash
docker --version                          # one line: Docker version 29.9.2, build a82e528
docker version --format '{{.Server.Version}}'   # just the number; --format uses Go templates
```

### 2.2 `docker info`

```bash
docker info
```

Prints a long report. Key lines:

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

| Field | Meaning |
|---|---|
| `Plugins` | Installed CLI plugins (`compose`, `buildx`, `scout`, `model`, ...). Without `compose`, `docker compose` fails. |
| `Containers / Images` | Counts by state. |
| `Storage Driver` | How image layers are stored (`overlayfs` with the containerd image store, `overlay2` on older/classic setups). |
| `Logging Driver` | Default log storage (`json-file` on plain Linux Engine, `local` on this Docker Desktop). |
| `Cgroup Version` | `2` on modern systems ([Chapter 1](01-what-is-docker.md#52-cgroups-how-much-can-this-process-use)). |
| `Runtimes` | Available OCI runtimes (`runc` default; `nvidia` appears when the NVIDIA toolkit is installed). |
| `Security Options` | Active hardening: `seccomp`, `apparmor`, `cgroupns`, `rootless`. |
| `Kernel Version`, `OSType`, `Architecture` | The engine's Linux kernel and platform (determines the default image architecture). |
| `CPUs`, `Total Memory` | Resources visible to the engine (controlled by `.wslconfig` or Desktop settings). |

Select single fields with a template: `docker info --format '{{.NCPU}} CPUs, {{.MemTotal}} bytes'`.

---

## 3. Working with images

An **image** is a read-only layered template, a **tag** is a movable label, and a **digest** is the immutable hash ([Chapter 1](01-what-is-docker.md#66-the-anatomy-of-an-image-name)).

### 3.1 `docker pull`: download an image

```bash
docker pull nginx:1.27
```

Downloads image `nginx` at tag `1.27` from Docker Hub (`docker.io/library/nginx:1.27`) into the local image store. Layers already present are skipped.

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
| `-a, --all-tags` | Download every tag in the repository (can be huge) |
| `--platform <os/arch>` | Pull a specific platform variant, for example `--platform linux/amd64` ([Chapter 2](02-installation-and-operating-systems.md#35-the---platform-flag)) |
| `-q, --quiet` | Suppress progress output |

Naming variants:

```bash
docker pull nginx                              # = docker.io/library/nginx:latest
docker pull python:3.13-slim                   # a specific version tag
docker pull ghcr.io/owner/tool:2.1             # another registry (GitHub)
docker pull nginx@sha256:3f1c...               # by digest: exact content, immutable
docker pull ubuntu:24.04 --platform linux/arm64   # force architecture
```

`docker run` pulls automatically when the image is missing. Use `pull` to pre-download, to update a moving tag such as `latest`, or to check access.

### 3.2 `docker images` / `docker image ls`: list local images

```bash
docker images
```

Docker 29 output (containerd image store, the default for fresh installs):

```
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
nginx:1.27           65645c7bb6a0       74.5MB         21.9MB   U
python:3.13-slim     a1b2c3d4e5f6        125MB         41.2MB
```

Older (classic) layout, common in tutorials:

```
REPOSITORY   TAG         IMAGE ID       CREATED        SIZE
nginx        1.27        65645c7bb6a0   2 weeks ago    192MB
python       3.13-slim   a1b2c3d4e5f6   3 weeks ago    125MB
```

| Column (new layout) | Meaning |
|---|---|
| `IMAGE` | `repository:tag` |
| `ID` | Short image ID (a hash of its config) |
| `DISK USAGE` | Disk occupied once unpacked locally |
| `CONTENT SIZE` | Compressed content size (roughly what a pull/push transfers) |
| `EXTRA` | Status marks. `U` means the image is in use by at least one container (even a stopped one) and cannot be deleted without removing its containers. `U` was confirmed on this Docker version by cross-checking `docker system df`. Check `docker image ls --help` if the version differs. |

| Flag | Meaning |
|---|---|
| `-a, --all` | Show all images, including intermediate and dangling ones (hidden by default) |
| `-q, --quiet` | Print only image IDs, one per line (for scripting: `docker rmi $(docker images -q)`; be careful) |
| `--digests` | Show `sha256` digests |
| `--no-trunc` | Do not shorten IDs |
| `-f, --filter <k=v>` | Filter results |
| `--format <template>` | Custom output (`table`, `json`, or a Go template) |
| `--tree` | (Experimental) Show multi-platform images as a tree, one child row per platform |

Filters:

```bash
docker images --filter dangling=true          # untagged "<none>" leftovers
docker images --filter "reference=python*"    # name pattern
docker images --filter before=nginx:1.27      # created before
docker images --filter since=nginx:1.27       # created after
docker images --filter label=team=ai          # by image label
```

A **dangling image** has no tag (`<none>:<none>` in the classic layout). It usually remains after a rebuild moves the tag to the new image. Dangling images are safe to prune ([Section 13](#13-disk-usage-and-cleanup-system-df-and-prune)).

Custom formats:

```bash
docker images --format '{{.Repository}}:{{.Tag}}  {{.ID}}  {{.Size}}'
docker images --format json                  # one JSON object per line
```

Template fields vary by version. In the Docker 29 layout, `.Repository`, `.Tag`, `.ID`, `.Size` work (confirmed) and `.DiskUsage` does not exist. Use `docker images --format json` to discover fields.

Tree view:

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

`nginx:1.27-alpine` is a multi-platform index. Only `linux/amd64` is downloaded here. The other platforms show `0B` because their layers were not pulled. This shows the manifest-list concept from Chapter 1.

### 3.3 `docker image inspect`

```bash
docker image inspect nginx:1.27
docker image inspect nginx:1.27 --format '{{.Architecture}} {{.Os}} {{.Id}}'
docker image inspect nginx:1.27 --format '{{json .Config.Env}}'
```

Prints image metadata as JSON: architecture, OS, ID, image environment variables, default command (`Cmd`), `Entrypoint`, exposed ports, labels, layer hashes, size. Template syntax is in [Section 8.1](#81-docker-inspect-and-go-templates). Example output of the second command:

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

Each row is one Dockerfile instruction (newest first). `SIZE` is the data the step added (`0B` for metadata-only instructions such as `CMD`, `EXPOSE`). `<missing>` in the IMAGE column is normal for intermediate layers. Use this to find which step makes an image large. The rows above are illustrative.

| Flag | Meaning |
|---|---|
| `--no-trunc` | Show full `CREATED BY` commands |
| `-H, --human` | Human-readable sizes/dates (default) |
| `-q, --quiet` | Only IDs |
| `--format` | Go template |
| `--platform` | History for one platform of a multi-platform image |

### 3.5 `docker tag`: give an image another name

```bash
docker tag nginx:1.27 myregistry.example.com/web/nginx:stable
```

Syntax: `docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`. This copies nothing. It adds another name (a pointer) to the same image. Tag before pushing, because the name encodes the destination registry.

### 3.6 `docker push`: upload an image

```bash
docker login ghcr.io                                  # authenticate first
docker tag myapp:1.0 ghcr.io/yourname/myapp:1.0       # name it for the registry
docker push ghcr.io/yourname/myapp:1.0
```

Output lists each layer as `Pushed` or `Layer already exists`, then a `digest: sha256:... size: ...` line. Flags: `-a, --all-tags` (push all tags), `--platform` (push one platform's manifest), `-q, --quiet`.

### 3.7 `docker rmi` / `docker image rm`: delete images

```bash
docker rmi nginx:1.27                 # remove by name:tag
docker image rm 65645c7bb6a0          # remove by ID
docker rmi -f myapp:old               # force
```

Output: `Untagged: nginx:1.27`, then `Deleted: sha256:...`. **Untagged** means that name was removed. **Deleted** means the image data is gone too (only when no other name or container needs it).

| Flag | Meaning |
|---|---|
| `-f, --force` | Force removal even if several tags point to it (still refuses if a running container uses it) |
| `--no-prune` | Do not delete untagged parent layers |
| `--platform` | Remove only the given platform variant of a multi-platform image |

Error `conflict: unable to delete <id> (must be forced) - image is being used by stopped container ...`: remove the container first (`docker rm`), then the image.

### 3.8 `docker save` and `docker load`: move images without a registry

Use for air-gapped (offline) servers, backups, and sharing without a registry.

```bash
docker save -o myapp.tar myapp:1.0          # write the image to a tar archive
docker save myapp:1.0 | gzip > myapp.tar.gz   # compressed (Bash)
docker load -i myapp.tar                    # on the other machine: import it
docker load < myapp.tar.gz                  # works with gz too
```

| Command/flag | Meaning |
|---|---|
| `docker save -o FILE IMAGE...` | Export one or more images (all layers and tags) to a tar file. Without `-o`, writes to stdout. |
| `docker save --platform linux/amd64` | Save only the named platform(s) |
| `docker load -i FILE` | Import images from the file. Without `-i`, reads stdin. |
| `docker load -q` | Quiet |

### 3.9 `docker export`, `docker import`, `docker commit`

| Command | What it does | Why to avoid it |
|---|---|---|
| `docker export CONTAINER > fs.tar` | Dumps a container's filesystem (flattened, no layers or metadata) | Loses history and metadata |
| `docker import fs.tar myimage:flat` | Creates an image from such a tar | Same |
| `docker commit CONTAINER newimage:tag` | Snapshots a container's changes into a new image | Not reproducible. Write a Dockerfile instead (Chapter 4). Acceptable for quick experiments. |

### 3.10 `docker search`

```bash
docker search --limit 5 postgres
docker search --filter is-official=true python
```

Searches Docker Hub and prints name, description, stars, and `[OK]` for official images. hub.docker.com also shows tags and docs.

### 3.11 Image cleanup commands

```bash
docker image prune             # remove dangling images (asks for confirmation)
docker image prune -a          # remove ALL images not used by any container
docker image prune -a --filter "until=168h"   # ... only those older than 7 days
```

See the warnings in [Section 13](#13-disk-usage-and-cleanup-system-df-and-prune).

---

## 4. `docker run`: creating and starting containers

`docker run` performs three steps:

```
 docker run IMAGE
      |
      +--1. docker pull   (only if the image is not already local)
      +--2. docker create (make a new container: image + writable layer + config)
      +--3. docker start  (start its main process) and, unless -d, attach the terminal to it
```

### 4.1 Syntax

```
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

- `IMAGE`: what to run, for example `nginx:1.27`.
- `COMMAND [ARG...]`: optional; overrides the image's default command (`CMD`).

```bash
docker run alpine echo "hello from a container"
```

Output: `hello from a container`. Docker created a container from `alpine`, ran `echo`, and the container exited because its main process finished.

A container lives exactly as long as its main process (PID 1). A web server stays up because the server process keeps running. `echo` exits immediately, so its container exits immediately.

### 4.2 The flags you need (grouped)

**Lifecycle and attachment**

| Flag | Meaning |
|---|---|
| `-d, --detach` | Run in the background and print the container ID. |
| `-i, --interactive` | Keep standard input (stdin) open. |
| `-t, --tty` | Allocate a pseudo-terminal (TTY): prompt, colors, line editing. |
| `-it` | Combined; the usual way to get an interactive shell: `docker run -it ubuntu bash`. |
| `--rm` | Delete the container (and its anonymous volumes) when it exits. Use for throwaway runs. |
| `--name <name>` | Set a name instead of a random one. Names must be unique among existing containers. Allows `docker logs web` instead of an ID. |
| `--restart <policy>` | Behavior when it exits (see 4.9). |
| `--init` | Run a small init process as PID 1 that forwards signals and reaps zombie processes ([Section 12](#12-pid-1-and-signals)). |
| `--pull <always\|missing\|never>` | When to pull the image. Default `missing`. `always` checks the registry every time. `never` uses only local images. |

**Networking and ports**

| Flag | Meaning |
|---|---|
| `-p, --publish <host>:<container>` | Publish a container port on the host ([Section 10](#10-port-publishing-explained)). |
| `-P, --publish-all` | Publish all `EXPOSE`d ports to random high host ports (see `docker port`). |
| `--network <name>` | Attach to a network (`bridge`, `host`, `none`, or a user-defined network). Chapter 5. |
| `--network-alias <name>` | Extra DNS name on a user-defined network. |
| `-h, --hostname <name>` | Set the hostname. |
| `--add-host <name>:<ip>` | Add an `/etc/hosts` entry (for example `host.docker.internal:host-gateway` to reach the host on Linux). |
| `--dns <ip>` | Custom DNS server. |
| `--expose <port>` | Mark a port as exposed (does not publish it). |

**Storage**

| Flag | Meaning |
|---|---|
| `-v, --volume <src>:<dest>[:opts]` | Mount a bind mount (host folder) or named volume. Short syntax. |
| `--mount type=...,source=...,target=...` | Same, explicit and more readable ([4.7](#47-volumes-and-bind-mounts--v-and---mount)). |
| `--tmpfs <path>` | Mount a RAM-backed temporary filesystem (gone at exit). |
| `--read-only` | Make the root filesystem read-only (hardening). Pair with `--tmpfs /tmp` and volumes for writable paths. |
| `--volumes-from <container>` | Mount all volumes from another container. |
| `-w, --workdir <dir>` | Working directory inside the container. |

**Configuration**

| Flag | Meaning |
|---|---|
| `-e, --env KEY=value` | Set an environment variable (`-e KEY` with no value copies it from the shell). |
| `--env-file <file>` | Read environment variables from a file. |
| `-l, --label key=value` | Attach metadata labels. |
| `--entrypoint <cmd>` | Override the image's `ENTRYPOINT`. |
| `-u, --user <name\|uid[:gid]>` | Run as a specific user (for example `1000:1000`). |
| `--platform <os/arch>` | Choose the platform (for example `linux/amd64`). |
| `--stop-signal <sig>` / `--stop-timeout <sec>` | Customise how `docker stop` stops it. |
| `--health-cmd`, `--health-interval`, `--health-timeout`, `--health-retries`, `--health-start-period`, `--no-healthcheck` | Define or disable health checks (Chapters 4 and 11). |
| `--log-driver`, `--log-opt` | Choose how this container's logs are stored. |

**Resources**

| Flag | Meaning |
|---|---|
| `-m, --memory <size>` | Hard memory limit (`512m`, `2g`). Exceeding it triggers the kernel OOM killer (exit 137). |
| `--memory-swap <size>` | Memory + swap total. Equal to `--memory` disables swap. |
| `--memory-reservation <size>` | Soft limit: the target under memory pressure. |
| `--cpus <n>` | CPUs' worth of time (for example `1.5`). |
| `--cpu-shares <n>` / `-c` | Relative CPU weight under contention (default 1024). |
| `--cpuset-cpus <list>` | Pin to specific cores (`0-3`, `0,2`). |
| `--pids-limit <n>` | Maximum number of processes (defends against fork bombs). |
| `--shm-size <size>` | Size of `/dev/shm` (default 64 MB). PyTorch DataLoader workers and browsers often need more, for example `--shm-size=2g`. |
| `--ulimit <type>=<soft>:<hard>` | Resource limits such as open files (`nofile=65536:65536`). |
| `--gpus <spec>` | Give the container GPU(s): `--gpus all` ([Chapter 8](08-docker-for-ai-and-ml.md)). |
| `--device <host>:<container>` | Pass a host device (for example `/dev/ttyUSB0`). |

**Security** (details in [Chapter 10](10-security.md))

| Flag | Meaning |
|---|---|
| `--cap-drop <CAP>` / `--cap-add <CAP>` | Remove/add Linux capabilities (`--cap-drop ALL` is a strong baseline). |
| `--security-opt <opt>` | For example `no-new-privileges`, custom seccomp/AppArmor profiles. |
| `--privileged` | Removes almost all protections and gives full device access. Avoid. Equivalent to root on the host. |
| `--userns`, `--pid`, `--ipc`, `--uts` | Share or isolate namespaces (for example `--pid=host` removes PID isolation). |

### 4.3 Foreground, detached, and interactive

```bash
docker run nginx                       # FOREGROUND: terminal shows nginx's logs; Ctrl+C stops it
docker run -d nginx                    # DETACHED: prints a container ID and returns the prompt
docker run -it ubuntu bash             # INTERACTIVE: a shell inside a new Ubuntu container
docker run -d -it ubuntu bash          # detached with a TTY, so it can be attached later
```

Details in [Section 11](#11-foreground-vs-detached).

### 4.4 Naming: `--name` and `--rm`

```bash
docker run -d --name web nginx:1.27
docker run --rm alpine echo "one-shot"     # the container is deleted after it exits
```

Without `--name`, Docker generates a name such as `vibrant_turing`. Running the `--name web` command twice fails:

```
docker: Error response from daemon: Conflict. The container name "/web" is already in use by container "3f4a1b9c2d7e...". You have to remove (or rename) that container to be able to reuse that name.
```

Fix: `docker rm web` (or `docker rm -f web` if running), or choose another name.

### 4.5 Publishing ports: `-p` and `-P`

```bash
docker run -d --name web -p 8080:80 nginx:1.27
```

`-p 8080:80` forwards host port 8080 to container port 80. `http://localhost:8080` then reaches nginx. See [Section 10](#10-port-publishing-explained).

| Form | Meaning |
|---|---|
| `-p 8080:80` | Host port 8080 to container port 80, on all host interfaces (`0.0.0.0`) |
| `-p 127.0.0.1:8080:80` | Reachable only from the host itself |
| `-p 8080:80/udp` | UDP instead of the default TCP |
| `-p 80` | Container port 80 mapped to a random host port |
| `-p 5000-5010:5000-5010` | A range |
| `-P` | Every `EXPOSE`d port to random host ports |

### 4.6 Environment variables: `-e` and `--env-file`

```bash
docker run --rm -e GREETING=hello -e USER_NAME=ada alpine sh -c 'echo "$GREETING, $USER_NAME"'
```

Output: `hello, ada`. `sh -c '...'` with single quotes makes the container's shell expand the variables, not the host shell.

```bash
export API_KEY=placeholder-not-a-real-key      # Bash; PowerShell: $env:API_KEY = "placeholder-not-a-real-key"
docker run --rm -e API_KEY alpine sh -c 'echo "key length: ${#API_KEY}"'
```

`-e API_KEY` (no `=value`) copies the value from the current shell, so it does not appear in command history.

`--env-file` reads many variables from a file:

```
# app.env   (lines starting with # are comments)
DATABASE_HOST=db
DATABASE_PORT=5432
LOG_LEVEL=info
```

```bash
docker run --rm --env-file app.env alpine env
```

Format rules: one `KEY=value` per line, `#` comment lines, and no shell processing. Quotes are not stripped (`KEY="abc"` includes the quotes), there is no `export` prefix, and there is no variable substitution. Do not commit real secrets in `.env` files. Add them to `.gitignore`. Chapter 10 covers secrets.

### 4.7 Volumes and bind mounts: `-v` and `--mount`

Container data is ephemeral unless storage is mounted ([Chapter 5](05-storage-and-networking.md)).

| Type | What it is | Example | Use for |
|---|---|---|---|
| **Named volume** | Storage managed by Docker, referenced by name | `-v pgdata:/var/lib/postgresql/data` | Databases and data Docker should own |
| **Bind mount** | A host folder or file mapped into the container | `-v "$(pwd)":/app` | Live-editing code, sharing config files |
| **tmpfs** | Memory-only filesystem | `--tmpfs /tmp` | Scratch data and secrets that must not touch disk |

Short syntax (`-v`):

```bash
docker run -d -v pgdata:/var/lib/postgresql/data postgres:17        # named volume
docker run --rm -v "$(pwd)":/app -w /app python:3.13-slim ls         # bind mount current folder
docker run --rm -v "$(pwd)"/config.yml:/etc/app/config.yml:ro alpine cat /etc/app/config.yml   # read-only file mount
```

`:ro` makes the mount read-only inside the container (`:rw` is the default).

Explicit syntax (`--mount`):

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
| `target` (or `dst`, `destination`) | Path inside the container |
| `readonly` (or `ro`) | Make read-only |

`-v` vs `--mount`: with `-v`, a missing bind-mount host path is silently created as an empty directory (a common cause of "my file became a folder"). With `--mount type=bind`, a missing source is an error. `--mount` is recommended for readability. `-v` is shorter and common.

Windows paths: PowerShell `-v ${PWD}:/app`; cmd `-v %cd%:/app`. For performance keep the folder inside WSL ([Chapter 2](02-installation-and-operating-systems.md#211-the-1-windows-performance-tip-keep-your-project-files-inside-wsl)).

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

`--user` improves security (avoids root), but the user needs permission on the files the process uses. `--entrypoint` replaces the image's entry command; the arguments after the image name become that command's arguments. Chapter 4 explains `ENTRYPOINT` vs `CMD`.

### 4.9 Restart policies: `--restart`

By default (`no`) an exited container stays stopped.

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart automatically. |
| `on-failure[:max]` | Restart only if the exit code is non-zero, optionally up to `max` attempts (for example `on-failure:5`). |
| `always` | Always restart, including when the daemon starts. A manually stopped container stays stopped until the daemon restarts or it is started again. |
| `unless-stopped` | Like `always`, but does not restart after a reboot if it was explicitly stopped. Usually the best choice for services. |

```bash
docker run -d --name api --restart unless-stopped -p 8000:8000 myapi:1.0
docker update --restart on-failure:3 api      # change the policy of an existing container
```

A restart policy does not fix crashes. A container that crashes immediately enters a restart loop (status `Restarting`). Check `docker logs`.

### 4.10 Resource limits: memory, CPU, processes

```bash
docker run -d --name limited --memory 512m --memory-swap 512m --cpus 1.5 --pids-limit 200 myapp:1.0
```

| Flag | Meaning in this command |
|---|---|
| `--memory 512m` | Max RAM 512 MiB. Above this the kernel kills the container's process (exit code 137, `OOMKilled=true`). |
| `--memory-swap 512m` | Memory + swap = 512 MiB, so no swap. If omitted, Docker allows swap equal to the memory limit, subject to host config. |
| `--cpus 1.5` | At most 1.5 cores' worth of CPU time (throttled, not killed). |
| `--pids-limit 200` | At most 200 processes/threads. |

Check enforcement with `docker stats` ([Section 8.3](#83-docker-stats)). On Docker Desktop, limits apply within the VM's total resources ([Chapter 2](02-installation-and-operating-systems.md#210-limit-wsl-2s-resources-with-wslconfig)).

Update limits on a running container (most need no restart):

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

`--read-only` makes everything except mounted volumes/tmpfs unwritable, so malicious or buggy code cannot modify container files. `--tmpfs /tmp` provides writable in-memory scratch space.

### 4.12 Platform and GPU

```bash
docker run --rm --platform linux/amd64 alpine uname -m     # x86_64, even on an ARM Mac (emulated)
docker run --rm --gpus all nvidia/cuda:<tag> nvidia-smi    # all GPUs
docker run --rm --gpus 2 ...                               # any 2 GPUs
docker run --rm --gpus '"device=0,1"' ...                  # specific GPUs by index
```

`--gpus` requires an NVIDIA GPU, the NVIDIA driver on the host, and (on Linux) the NVIDIA Container Toolkit. Docker Desktop with WSL 2 handles this when the Windows NVIDIA driver is current. `<tag>` is a placeholder: CUDA image tags change, so pick a current one from Docker Hub's `nvidia/cuda` page. See [Chapter 8](08-docker-for-ai-and-ml.md).

### 4.13 Health checks (a first look)

```bash
docker run -d --name web \
  --health-cmd "curl -fsS http://localhost/ || exit 1" \
  --health-interval 10s --health-timeout 3s --health-retries 3 --health-start-period 5s \
  nginx:1.27
docker ps    # STATUS shows "Up 30 seconds (healthy)" or "(unhealthy)" / "(health: starting)"
```

Docker runs the command periodically inside the container. Exit code 0 means healthy. The nginx image may not include `curl`; the example only shows syntax. Chapter 4 covers the Dockerfile `HEALTHCHECK` instruction, the more common method.

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

The command runs detached, restarts unless stopped, publishes only on localhost, loads settings from a file, stores a model cache in a named volume, limits memory and CPU, uses a read-only root filesystem with in-memory `/tmp`, runs as a non-root user, and drops all capabilities.

In practice this belongs in a Compose file ([Chapter 6](06-docker-compose.md)).

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
| `CONTAINER ID` | Shortened unique ID (the full ID is 64 hex characters). Any unique prefix works in commands. |
| `IMAGE` | The image it was created from. |
| `COMMAND` | The command running as PID 1 (truncated). |
| `CREATED` | Creation time. |
| `STATUS` | `Up 2 minutes`, `Exited (0) 5 seconds ago`, `Up 3 minutes (healthy)`, `Restarting (1) 4 seconds ago`, `Paused`, ... |
| `PORTS` | Port mappings. `0.0.0.0:8080->80/tcp` means host port 8080 on all interfaces forwards to container port 80/tcp. A port without `->` (for example `80/tcp`) is exposed but not published. |
| `NAMES` | The container name. |

| Flag | Meaning |
|---|---|
| `-a, --all` | Show all containers, including stopped (default: running only) |
| `-q, --quiet` | Only IDs (`docker rm -f $(docker ps -aq)` removes all containers; dangerous) |
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

Custom formats (Go templates):

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --format '{{.Names}}: {{.Status}}'
docker ps --format json
```

`\t` is a tab and `table` adds a header row. Common placeholders: `.ID`, `.Image`, `.Command`, `.CreatedAt`, `.RunningFor`, `.Ports`, `.State`, `.Status`, `.Size`, `.Names`, `.Labels`, `.Mounts`, `.Networks`.

---

## 6. Looking inside a container: `logs`, `exec`, `attach`, `cp`

### 6.1 `docker logs`

```bash
docker logs web
```

Shows everything the container's main process wrote to stdout and stderr (standard output and standard error, the two text streams every program has). This is the first command to run when a container misbehaves.

`docker logs` captures only stdout/stderr. If an app writes logs only to a file inside the container, `docker logs` shows nothing. Containerized apps should log to stdout/stderr (the "twelve-factor" convention).

| Flag | Meaning |
|---|---|
| `-f, --follow` | Stream new lines live (like `tail -f`). Ctrl+C stops following, not the container. |
| `-n, --tail <N>` | Only the last N lines (default `all`) |
| `--since <time>` | Logs since a timestamp or relative time: `--since 10m`, `--since 2026-10-23T09:00:00` |
| `--until <time>` | Logs before a time |
| `-t, --timestamps` | Prefix each line with a timestamp |
| `--details` | Show extra details provided to the logging driver |

```bash
docker logs -f --tail 20 web              # last 20 lines, then stream
docker logs --since 5m -t web             # last 5 minutes with timestamps
docker logs web 2>&1 | grep -i error      # search: 2>&1 merges stderr into the pipe
```

Example (nginx access log):

```
172.17.0.1 - - [23/Oct/2026:09:15:02 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/8.5.0" "-"
```

**Logging drivers** decide where logs are kept: `json-file` (default on plain Linux Engine; unbounded unless rotation is configured), `local` (rotates by default), `syslog`, `journald`, `fluentd`, `awslogs`, `none`. `docker logs` works only with some drivers (`json-file`, `local`, `journald`). Choose per container with `--log-driver` and `--log-opt max-size=10m --log-opt max-file=3`, or globally in `daemon.json` ([Chapter 2](02-installation-and-operating-systems.md#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)).

### 6.2 `docker exec`: run a command in a running container

```bash
docker exec web nginx -v                 # run one command and print its output
docker exec -it web bash                 # open an interactive shell inside it
docker exec -it web sh                   # use sh if bash is not installed (for example Alpine)
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
| `--privileged` | Extended privileges for the command (avoid) |

| Command | What it does |
|---|---|
| `docker run` | Creates a new container from an image and starts it |
| `docker exec` | Starts an additional process inside an already running container (the main process is unaffected) |
| `docker attach` | Connects the terminal to the container's main process (PID 1) streams |

Exiting a shell started with `exec` does not stop the container, because the main process keeps running. `exec` works only on running containers. For a crashed container, use `docker logs`, or start a new container from the same image with `--entrypoint sh`. For containers without a shell (distroless images), `docker debug` and `docker cp` can help ([Section 18](#18-other-commands-and-plugins-you-will-see)).

### 6.3 `docker attach`

```bash
docker attach web
```

Connects the terminal's stdin/stdout/stderr to the container's PID 1 process. Useful for interactive programs started with `-it` and detached with `-d`. Ctrl+C in an attached session sends SIGINT to the main process and can stop the container. To detach without stopping, press Ctrl+P then Ctrl+Q (only if the container was started with `-it`).

| Flag | Meaning |
|---|---|
| `--no-stdin` | Do not attach stdin (read-only view) |
| `--sig-proxy` | Forward signals to the process (default true; `--sig-proxy=false` stops Ctrl+C from reaching it) |
| `--detach-keys` | Change the detach key sequence |

To watch output, `docker logs -f` is usually better than `attach`.

### 6.4 `docker cp`: copy files between host and container

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf     # container -> host
docker cp ./index.html web:/usr/share/nginx/html/    # host -> container
docker cp web:/var/log/nginx/. ./nginx-logs/         # copy a folder's CONTENTS (note the trailing /.)
```

Syntax: `docker cp SRC DEST`, where one side is `CONTAINER:PATH`. It works on running and stopped containers. A trailing `/.` on a source directory copies its contents rather than the directory.

| Flag | Meaning |
|---|---|
| `-a, --archive` | Preserve uid/gid information |
| `-L, --follow-link` | Follow symbolic links in the source |
| `-q, --quiet` | Suppress progress output |

Use `docker cp` for debugging and one-off extraction, not deployment. Bake files into the image or use volumes/bind mounts.

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

   Other states:  restarting (restart policy is retrying),
                  removing (being deleted), dead (failed removal; needs manual cleanup)
```

| State | Meaning |
|---|---|
| **created** | Container exists; its process has never started. |
| **running** | Main process is alive. |
| **paused** | All processes frozen (cgroup freezer); memory retained. |
| **restarting** | A restart policy is waiting/retrying after an exit. |
| **exited** | Main process ended (exit code recorded). The container, its logs, and writable layer still exist until removed. |
| **removing** | Being deleted. |
| **dead** | Removal failed partway; needs manual intervention (`docker rm -f`, or restart the daemon). |

### 7.2 `docker stop`: graceful shutdown

```bash
docker stop web
docker stop -t 30 web          # wait up to 30 s before force-killing
docker stop web db cache       # several at once
```

Sequence:

1. Docker sends **SIGTERM** (signal 15, a request to shut down) to the main process (PID 1). Change the signal with `--stop-signal` at run time, or `-s` here.
2. It waits up to 10 seconds by default (`-t`/`--timeout` changes it).
3. If the process is still alive, Docker sends **SIGKILL** (signal 9, which cannot be caught or ignored).

A well-behaved app catches SIGTERM, finishes in-flight requests, closes connections, and exits with code 0 or 143. A container that always takes exactly ten seconds to stop and exits 137 has a PID 1 that ignores SIGTERM ([Section 12](#12-pid-1-and-signals)).

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

`kill` defaults to SIGKILL: no cleanup, no flushed buffers, no closed connections. Prefer `stop`. `kill -s <signal>` sends a particular signal without stopping, for example `docker kill -s HUP web` to reload nginx.

### 7.4 `docker start`, `restart`

```bash
docker start web              # start a stopped container (keeps its name, settings, and writable-layer data)
docker start -a web           # start and ATTACH to its output
docker start -ai mycontainer  # attach and interactive (restart a stopped interactive shell)
docker restart web            # stop then start
docker restart -t 5 web       # wait 5 s before killing during the stop phase
```

`start` flags: `-a, --attach` (attach stdout/stderr and forward signals), `-i, --interactive` (attach stdin). `restart` flags: `-t, --timeout`, `-s, --signal`.

`docker start` resumes the same container with its existing writable layer. Changing an image tag does not update an existing container. To pick up a new image or new `-p`/`-e`/`-v` flags, remove and re-create the container (`docker rm` + `docker run`). These settings are fixed at creation, except a few changeable with `docker update`.

### 7.5 `docker pause` / `unpause`

```bash
docker pause web        # freeze every process (no CPU use; memory stays allocated)
docker unpause web
```

Uses the cgroup freezer. Rarely needed. Useful for consistent snapshots or temporarily halting a busy container without losing state.

### 7.6 `docker rm`: delete containers

```bash
docker rm web                  # remove a STOPPED container
docker rm -f web               # force: SIGKILL a running container, then remove it
docker rm -v web               # also remove its anonymous volumes
docker rm web db cache         # several at once
```

| Flag | Meaning |
|---|---|
| `-f, --force` | Remove even if running (uses SIGKILL) |
| `-v, --volumes` | Also delete anonymous volumes attached to it (named volumes are never deleted by `rm`) |
| `-l, --link` | Remove a legacy link (rare) |

Removing a container deletes its writable layer: data not in a volume is lost. Error `You cannot remove a running container ... Stop the container before attempting removal or force remove`: stop it first or use `-f`.

Remove all stopped containers:

```bash
docker container prune          # asks for confirmation, removes every stopped container
```

### 7.7 `docker rename`, `docker wait`, `docker update`, `docker create`

```bash
docker rename web web-old             # rename a container (running or not)
docker wait web                       # block until the container stops, then print its exit code (useful in scripts)
docker create --name later nginx:1.27 # create (but do not start) a container; status "created"
docker start later                    # start it later
docker update --restart unless-stopped --memory 1g web   # change some settings on a live container
```

`docker create` takes the same flags as `docker run` and only prepares the container.

---

## 8. Inspecting and monitoring: `inspect`, `stats`, `top`, `events`, `diff`, `port`

### 8.1 `docker inspect` and Go templates

`docker inspect` prints everything Docker knows about an object (container, image, volume, network) as JSON.

```bash
docker inspect web
```

Abbreviated container output:

```json
[
  {
    "Id": "3f4a1b9c2d7e...",
    "Name": "/web",
    "State": { "Status": "running", "OOMKilled": false, "Pid": 48213, "ExitCode": 0,
               "Health": { "Status": "healthy" } },
    "Config": { "Env": ["PATH=...", "NGINX_VERSION=1.27.x"], "Cmd": ["nginx", "-g", "daemon off;"],
                "Image": "nginx:1.27" },
    "HostConfig": { "PortBindings": { "80/tcp": [ { "HostPort": "8080" } ] },
                    "RestartPolicy": { "Name": "unless-stopped" } },
    "Mounts": [],
    "NetworkSettings": { "Networks": { "bridge": { "IPAddress": "172.17.0.2" } } }
  }
]
```

`--format` (`-f`) with a **Go template** extracts specific fields. Docker is written in Go, and `--format` accepts Go's templating syntax: text with `{{ ... }}` blocks. A leading `.` means the inspected object, and `.State.Status` means the `Status` field inside `State`.

```bash
docker inspect --format '{{.State.Status}}' web                       # running
docker inspect --format '{{.State.ExitCode}}' web                     # 0
docker inspect --format '{{.State.OOMKilled}}' web                    # false (killed for memory?)
docker inspect --format '{{.Config.Image}}' web                       # nginx:1.27
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' web      # unless-stopped
docker inspect --format '{{.State.Pid}}' web                          # the process's PID on the HOST
docker inspect --format '{{.State.Health.Status}}' web                # healthy (only if a health check exists)
```

Intermediate templates:

```bash
# Print a sub-section as compact JSON:
docker inspect --format '{{json .Config.Env}}' web
# ["PATH=/usr/local/sbin:...","NGINX_VERSION=1.27.x"]

# The container's IP address on every attached network:
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' web
# 172.17.0.2

# Host port mapped to container port 80/tcp (index for keys with special characters):
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

Quoting by shell (relevant on Windows):

| Shell | Syntax |
|---|---|
| Bash / zsh / PowerShell | `--format '{{.State.Status}}'` (single quotes) |
| cmd.exe | `--format "{{.State.Status}}"` (double quotes; single quotes are not quotes in cmd) |

Other objects: `docker inspect nginx:1.27` (image), `docker inspect myvolume`, `docker inspect bridge` (network). `--type container|image|volume|network` disambiguates name clashes. `-s, --size` adds container size info. To process with `jq` (a JSON command-line processor): `docker inspect web | jq '.[0].State'`.

### 8.2 `docker top`

```bash
docker top web
```

```
UID     PID     PPID    C   STIME   TTY   TIME       CMD
root    48213   48190   0   09:13   ?     00:00:00   nginx: master process nginx -g daemon off;
101     48247   48213   0   09:13   ?     00:00:00   nginx: worker process
```

Lists processes inside the container, showing the host's PIDs (inside, the master is PID 1). Containers are processes ([Chapter 1](01-what-is-docker.md#5-how-containers-actually-work)). `ps` options are accepted: `docker top web -ef`.

### 8.3 `docker stats`

```bash
docker stats
```

```
CONTAINER ID   NAME   CPU %   MEM USAGE / LIMIT    MEM %   NET I/O         BLOCK I/O     PIDS
3f4a1b9c2d7e   web    0.00%   4.512MiB / 15.34GiB  0.03%   1.2kB / 0B      0B / 12.3kB   5
```

A live table (refreshes about once per second) of resource use per running container. Ctrl+C quits.

| Column | Meaning |
|---|---|
| `CPU %` | CPU use relative to the host's CPUs (can exceed 100% on multi-core; 200% = two full cores) |
| `MEM USAGE / LIMIT` | Memory used / limit (host total if no `--memory`) |
| `MEM %` | Usage as a percentage of the limit |
| `NET I/O` | Network bytes received / sent |
| `BLOCK I/O` | Disk bytes read / written |
| `PIDS` | Number of processes/threads |

| Flag | Meaning |
|---|---|
| `--no-stream` | Print one snapshot and exit (for scripts) |
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

Streams real-time daemon events: container create/start/die/stop, image pull, network connect, and more.

```
2026-10-23T09:13:00.101010101Z container create 3f4a1b9c2d7e... (image=nginx:1.27, name=web)
2026-10-23T09:13:00.402020202Z container start 3f4a1b9c2d7e... (image=nginx:1.27, name=web)
2026-10-23T09:20:11.303030303Z container die 3f4a1b9c2d7e... (exitCode=0, image=nginx:1.27, name=web)
```

Flags: `-f, --filter` (for example `type=`, `event=`, `container=`, `image=`), `--since`, `--until`, `--format`. Use it to find when and why a container died: the `die` event includes `exitCode`, and `oom` events show memory kills.

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

Shows files changed in the container's writable layer relative to the image: `A` added, `C` changed, `D` deleted. Use it to find what a running program writes to disk (what belongs on a volume or must be writable under `--read-only`).

### 8.6 `docker port`

```bash
docker port web            # all mappings:  80/tcp -> 0.0.0.0:8080
docker port web 80         # just container port 80:  0.0.0.0:8080
```

Useful after `-P` (random ports) to find the assigned host port.

---

## 9. Exit codes

When a container's main process ends it returns an **exit code**, a number from 0 to 255. By convention 0 means success and anything else means failure. The code appears in `docker ps -a` (`Exited (137)`), in `docker inspect --format '{{.State.ExitCode}}'`, from `docker wait`, and for a foreground `docker run`, in the shell's `$?` (Bash) or `$LASTEXITCODE` (PowerShell).

```bash
docker run --rm alpine sh -c 'exit 3'; echo "exit code: $?"
# exit code: 3
```

### 9.1 The table

| Code | Meaning | Typical cause and action |
|---|---|---|
| **0** | Success | Process finished normally. For a service, exit 0 means it quit on its own (check why). |
| **1** | Generic application error | Crash or unhandled error. Read `docker logs`. |
| **2** | Misuse of a shell built-in or bad CLI arguments | Often wrong arguments to a program. |
| **125** | The `docker run` command itself failed (daemon error, not the program) | Bad flag, name in use, port already allocated, invalid mount, container cannot start. Docker's docs: "Exit code 125 indicates that the error is with Docker daemon itself." The error prints to the terminal. |
| **126** | The command inside the container cannot be invoked | File exists but is not executable (missing `chmod +x`), is a directory, or has the wrong format. |
| **127** | Command not found inside the container | Typo, wrong `CMD`/`ENTRYPOINT`, program not installed in this image (for example `bash` on Alpine), or wrong PATH. |
| **130** | Terminated by Ctrl+C (SIGINT, signal 2: 128 + 2) | Interrupted by the user. |
| **134** | Aborted (SIGABRT, 6: 128 + 6) | Program called `abort()` (assertion failure, C/C++ crash). |
| **137** | Killed by SIGKILL (signal 9: 128 + 9) | (a) Out of memory (OOM killer): the container exceeded its memory limit, or the host/VM ran out of RAM. (b) `docker kill` or `docker rm -f`. (c) `docker stop` timed out because the process ignored SIGTERM for 10 s. Check OOM: `docker inspect --format '{{.State.OOMKilled}}' <name>`. |
| **139** | Segmentation fault (SIGSEGV, 11: 128 + 11) | Native-code crash: bad memory access, mismatched binary/library, incompatible architecture under emulation, buggy native extension. |
| **143** | Terminated by SIGTERM (signal 15: 128 + 15) | Normal outcome of `docker stop` when the app has no handler (the signal killed the process) or exits 143 after handling it. Usually fine. |
| **255** | Exit status out of range / unknown | Sometimes `ssh` errors or `exit -1`. |

Codes above 128 mean "killed by a signal". The signal number is the code minus 128: 137 - 128 = 9 (SIGKILL), 143 - 128 = 15 (SIGTERM), 139 - 128 = 11 (SIGSEGV), 130 - 128 = 2 (SIGINT).

### 9.2 Reproducing the important ones

```bash
docker run --rm alpine sh -c 'exit 7';  echo "exit 7 -> $?"   # 7 (application-defined)
docker run --rm --bogus-flag alpine;    echo "bad flag -> $?" # 125: Docker itself rejects the flag
docker run --rm alpine /etc;            echo "dir -> $?"      # 126: not executable (a directory)
docker run --rm alpine nosuchprogram;   echo "missing -> $?"  # 127: command not found
docker run --rm -m 20m --memory-swap 20m python:3.13-slim python -c "x = b'x' * (100*1024*1024)"
echo "oom -> $?"                                               # 137: OOM kill (20 MB limit, 100 MB allocation)
docker run -d --name victim alpine sleep 300 && docker kill victim
docker inspect --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' victim   # exit=137 oom=false
docker rm victim
```

PowerShell: use `$LASTEXITCODE` instead of `$?`. For the OOM test, keep the container (omit `--rm`, add `--name`) and inspect it. `OOMKilled=true` means memory was the cause. `OOMKilled=false` means something sent SIGKILL.

On Docker Desktop, an OOM kill can also come from the VM running out of memory. `OOMKilled` may then be `false` while the exit code is still 137. Check the WSL/Desktop memory settings ([Chapter 2](02-installation-and-operating-systems.md#210-limit-wsl-2s-resources-with-wslconfig)).

---

## 10. Port publishing explained

### 10.1 Why it is needed

A container has its own network namespace: its own IP address (for example `172.17.0.2`) and its own port numbers on a private internal network. Other machines cannot reach that address. **Publishing** a port makes Docker listen on a port on the host and forward connections to a port in the container.

```
        HOST MACHINE
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
        (outside)     (where the app listens inside)
```

The order is always `host:container`.

### 10.2 Examples

```bash
docker run -d --name web -p 8080:80 nginx:1.27
curl http://localhost:8080          # nginx welcome page HTML
```

Two containers cannot publish the same host port, but both can use container port 80:

```bash
docker run -d --name web1 -p 8081:80 nginx:1.27
docker run -d --name web2 -p 8082:80 nginx:1.27
```

Error when the host port is taken:

```
docker: Error response from daemon: ... Bind for 0.0.0.0:8080 failed: port is already allocated.
```

This is exit code 125 ([Section 9](#9-exit-codes)). Fix: choose a different host port or stop the other container.

### 10.3 Binding to localhost for safety

```bash
docker run -d -p 127.0.0.1:5432:5432 --name db -e POSTGRES_PASSWORD=learn-only postgres:17
```

`127.0.0.1:5432:5432` allows connections only from the local machine. Without the `127.0.0.1:` prefix, Docker listens on `0.0.0.0` (all interfaces), so anyone on the network (or the internet, on a public server) can try to connect. Published ports also bypass `ufw`/`firewalld` rules on Linux ([Chapter 2](02-installation-and-operating-systems.md#49-firewall-caveat-important)). IPv6 loopback is written `[::1]:8080:80`.

### 10.4 `EXPOSE` vs `-p`

`EXPOSE 80` in a Dockerfile is documentation only. It records that the app listens on port 80 and opens nothing on the host. Only `-p` (or `-P`, which publishes all `EXPOSE`d ports to random host ports) publishes a port.

### 10.5 The classic gotcha: listening on `127.0.0.1` inside the container

With `-p 8000:8000` published but the page not loading, the app is often bound to `127.0.0.1` inside the container. That address accepts only connections from inside the same container. Docker's forwarded traffic arrives from outside that namespace and is refused. Bind the app to `0.0.0.0` (all interfaces inside the container):

| Framework | Fix |
|---|---|
| FastAPI/uvicorn | `uvicorn main:app --host 0.0.0.0 --port 8000` |
| Flask | `flask run --host=0.0.0.0` |
| Node/Express | `app.listen(3000, '0.0.0.0')` |
| Streamlit | `streamlit run app.py --server.address=0.0.0.0` |
| Jupyter | `jupyter lab --ip=0.0.0.0 --no-browser` |

Binding to `0.0.0.0` inside the container is safe when exposure is controlled by the host side of `-p` (for example `127.0.0.1:8000:8000`). To see which address the app listens on: `docker exec web ss -ltn` (or `netstat -ltn`).

---

## 11. Foreground vs detached

| Mode | Command | Behavior | Use when |
|---|---|---|---|
| **Foreground** | `docker run nginx` | stdout/stderr stream to the terminal, which is blocked until the container exits. Ctrl+C sends SIGINT and usually stops the container. | Quick tests, watching startup, short jobs |
| **Detached** | `docker run -d nginx` | Starts in the background and prints the full container ID. View output with `docker logs`. | Services (web servers, databases, APIs) |
| **Interactive** | `docker run -it ubuntu bash` | stdin open plus a terminal: a shell inside the container. Exiting the shell ends PID 1 and the container. | Exploring, debugging, REPLs |
| **Detached + interactive** | `docker run -dit ubuntu bash` | Background container with a shell that can be attached to or `exec`'d into later | Long-lived shells |

A container that exits immediately usually has a main process that finished. For example, `docker run -d ubuntu` runs `bash`, which sees no terminal/stdin and exits. Use `docker run -d ubuntu sleep infinity` or `-dit` for an idle container.

`docker run -a stdout alpine echo hi` uses `-a` to attach only certain streams. `--sig-proxy` (default true) forwards Ctrl+C to the main process.

---

## 12. PID 1 and signals

### 12.1 What is a signal?

A **signal** is a small message the OS delivers to a process: "terminate" (SIGTERM), "stop immediately" (SIGKILL), "Ctrl+C pressed" (SIGINT), "reload config" (SIGHUP). A process can handle most signals with its own cleanup code. SIGKILL and SIGSTOP cannot be handled; the kernel enforces them.

| Signal | Number | Meaning | Catchable |
|---|---|---|---|
| SIGHUP | 1 | Hangup / reload config (by convention) | Yes |
| SIGINT | 2 | Interrupt (Ctrl+C) | Yes |
| SIGQUIT | 3 | Quit with core dump (Ctrl+\\); nginx uses it for graceful stop | Yes |
| SIGKILL | 9 | Kill immediately | No |
| SIGSEGV | 11 | Segmentation fault (bad memory access) | Yes, rarely handled |
| SIGTERM | 15 | Terminate request (sent by `docker stop`) | Yes |

### 12.2 Why PID 1 is special

Inside a container the main process is PID 1, and the Linux kernel treats PID 1 of a PID namespace specially:

1. **Signal handling:** for most signals PID 1 has no default action. A normal process without a SIGTERM handler is killed by the kernel. PID 1 ignores the signal unless the program handles it explicitly. A Python script running as PID 1 without `signal.signal(SIGTERM, ...)` ignores `docker stop`'s SIGTERM. Docker waits 10 seconds, then sends SIGKILL: exit code 137.
2. **Zombie reaping:** when a child process dies, its parent must collect its exit status ("reap" it). Otherwise it remains a **zombie** (a dead process entry). On a normal host, PID 1 (init/systemd) adopts and reaps orphans. In a container the app is PID 1 and may not do this, so zombies can accumulate.

### 12.3 Shell form vs exec form

The form of `CMD`/`ENTRYPOINT` in a Dockerfile (Chapter 4) decides which process is PID 1:

| Dockerfile | PID 1 | Signals |
|---|---|---|
| `CMD python app.py` (shell form) | `/bin/sh -c "python app.py"`; Python is a child | `sh` does not forward SIGTERM to its child: `docker stop` waits 10 s, then SIGKILL |
| `CMD ["python", "app.py"]` (exec form) | `python app.py` directly | Python receives SIGTERM directly (and must handle it) |

Check with:

```bash
docker run -d --name shellform alpine sh -c 'sleep 300'     # sh as PID 1 (alpine's sh may exec sleep directly)
docker top shellform
```

Some shells optimize a single command with `exec`, so results vary. The guidance holds in general.

### 12.4 Demonstration: slow stop vs fast stop

```bash
# A process that IGNORES SIGTERM as PID 1 (a shell loop with no trap):
docker run -d --name slowstop alpine sh -c 'while true; do sleep 1; done'
time docker stop slowstop           # takes ~10 seconds
docker inspect --format 'exit={{.State.ExitCode}}' slowstop    # exit=137  (SIGKILL after the timeout)
docker rm slowstop

# The same loop, with a trap that handles SIGTERM:
docker run -d --name faststop alpine sh -c 'trap "echo bye; exit 0" TERM; while true; do sleep 1 & wait $!; done'
time docker stop faststop           # returns almost immediately
docker inspect --format 'exit={{.State.ExitCode}}' faststop    # exit=0
docker rm faststop
```

`time` is a Bash/WSL/macOS builtin. In PowerShell use `Measure-Command { docker stop slowstop }`.

### 12.5 The fixes

1. Use exec form in the Dockerfile: `CMD ["python", "app.py"]`.
2. Handle SIGTERM in the application (Python: `signal.signal(signal.SIGTERM, handler)`; Node: `process.on('SIGTERM', ...)`). uvicorn, gunicorn, and nginx already do.
3. Use an init process: `docker run --init ...` (or `init: true` in Compose) makes Docker's small init (`docker-init`, which is `tini`) PID 1. It forwards signals and reaps zombies. A cheap, safe default for apps that spawn subprocesses.
4. If the app expects a different stop signal (nginx prefers SIGQUIT for graceful stop), set `--stop-signal` or the Dockerfile `STOPSIGNAL`. Give slow apps more time: `--stop-timeout 30` or `docker stop -t 30`.
5. In shell-script entrypoints, end with `exec "$@"` so the real program replaces the shell as PID 1.

---

## 13. Disk usage and cleanup: `system df` and `prune`

Docker accumulates images, stopped containers, volumes, build cache, and logs. Inspect before deleting.

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
| `TOTAL` | Number of objects |
| `ACTIVE` | Objects in use (images used by a container; running containers; volumes attached to a container) |
| `SIZE` | Total disk consumed |
| `RECLAIMABLE` | Space recoverable by removing unused objects |

`docker system df -v` lists each image, container, and volume, with a `CONTAINERS` count per image and `UNIQUE SIZE` vs `SHARED SIZE`. Use it to find the largest items.

### 13.2 The prune family

| Command | Removes | `-a` effect |
|---|---|---|
| `docker container prune` | All stopped containers | n/a |
| `docker image prune` | Dangling (untagged) images | `-a` = all images not used by any container |
| `docker volume prune` | Anonymous unused volumes (by default) | `-a, --all` = all unused volumes, including named ones |
| `docker network prune` | Unused user-defined networks | n/a |
| `docker builder prune` | Build cache | `-a` includes more (internal/frontend images) |
| `docker system prune` | Stopped containers + unused networks + dangling images + build cache | `-a` = all unused images; `--volumes` = also anonymous volumes |

Common flags: `-f, --force` (no confirmation), `--filter "until=<duration>"` (only objects older than, for example, `until=24h`, `until=168h`), `--filter label=...`.

Without `-f`, each prune prints a warning listing what it removes and asks `[y/N]`.

### 13.3 Warnings (read these)

`docker system prune -a --volumes` and `docker volume prune -a` can destroy data.

- **Volumes hold databases.** A volume no container currently uses counts as "unused", even when it holds the only copy of a PostgreSQL database. Pruning volumes deletes them permanently. In Docker 23+, plain `docker volume prune` and `docker system prune --volumes` remove only anonymous volumes. `-a` on `volume prune` extends it to named volumes. Do not add `-a` without reviewing the list.
- "Unused" images can be downloaded again but may not be reproducible (for example an image built locally from a Dockerfile that no longer exists, or a tag since replaced upstream).
- Pruning on a shared or production host can delete things others need. Run `docker system df -v` first, then remove selectively.
- `-f` skips the confirmation prompt. Do not combine `-f` with a prune that has not been previewed.
- Stopped containers may hold logs and files needed for debugging. Look before pruning.

Safer sequence:

```bash
docker system df                                  # 1. look first
docker ps -a                                      # 2. see which containers exist
docker volume ls                                  # 3. see volumes
docker image prune                                # 4. start with the least risky (dangling images)
docker container prune --filter "until=24h"       # 5. remove only stale stopped containers
docker builder prune --filter "until=168h"        # 6. trim old build cache
```

Recovering from a full disk on a build machine usually means `docker builder prune` (build cache is often the largest consumer) and `docker image prune -a --filter "until=720h"`.

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
| `-p, --password` | Password or token. Discouraged: it appears in shell history and the process list. |
| `--password-stdin` | Read the password/token from stdin. Recommended in scripts. |

Safe form:

```bash
echo "$MY_TOKEN" | docker login ghcr.io -u yourname --password-stdin
```

Output: `Login Succeeded`. Docker stores credentials in `~/.docker/config.json` (Windows: `C:\Users\<you>\.docker\config.json`), ideally through a **credential helper**: a program that stores secrets in the OS keychain (Windows Credential Manager, macOS Keychain, Linux `pass`/secretservice) so no plain-text password sits in the file. Look for a `credsStore` entry in `config.json`. Without one (for example on a Linux server), credentials are stored base64-encoded, not encrypted. Protect that file or configure a helper.

Use personal access tokens (PATs) or registry-specific tokens instead of the account password, with minimum scope (read-only for pulls). Logging in raises the Docker Hub pull-rate limit ([Chapter 1](01-what-is-docker.md#91-docker-hub)). CI systems should use short-lived or scoped tokens from the CI's secret store.

### 14.2 `docker search` (again)

See [3.10](#310-docker-search). Private registries require knowing repository names.

---

## 15. Contexts: talking to other Docker hosts

A **Docker context** is a saved, named connection to a Docker daemon (local Docker Desktop, a remote server over SSH, a cloud endpoint). The CLI sends commands to the active context.

```bash
docker context ls
```

```
NAME              DESCRIPTION                               DOCKER ENDPOINT                             ERROR
default           Current DOCKER_HOST based configuration   npipe:////./pipe/docker_engine
desktop-linux *   Docker Desktop                            npipe:////./pipe/dockerDesktopLinuxEngine
```

`*` marks the active context. On Linux the endpoint looks like `unix:///var/run/docker.sock`. On Mac it is a Unix socket under the home directory.

| Command | Meaning |
|---|---|
| `docker context ls` | List contexts |
| `docker context create NAME --docker "host=ssh://user@host"` | Create a context for a remote daemon over SSH |
| `docker context use NAME` | Make it the active context for later commands |
| `docker context inspect NAME` | Show details |
| `docker context rm NAME` | Delete a context (not the remote server) |
| `docker --context NAME ps` | Use a context for one command only |
| `DOCKER_CONTEXT=NAME` (environment variable) | Select a context for a shell session |

```bash
docker context create staging --docker "host=ssh://deploy@203.0.113.10"
docker --context staging ps                # list containers on the staging server
docker context use staging                 # everything now targets it
docker context use desktop-linux           # back to local (use "default" on Linux servers)
```

SSH needs no extra open daemon port and provides key-based authentication and encryption. The remote machine needs Docker Engine, the SSH user must be allowed to run `docker`, and SSH keys must work non-interactively (`ssh-agent`). Check the active context before destructive commands: `docker rm -f $(docker ps -aq)` on the wrong context removes the wrong containers. Some users show the context name in the shell prompt.

---

## 16. Building: `docker build`, `builder`, and `buildx` (intro)

Chapter 4 covers builds in depth.

```bash
docker build -t myapp:1.0 .
```

| Piece | Meaning |
|---|---|
| `docker build` | Build an image from a Dockerfile |
| `-t myapp:1.0` | Tag the result (`name:tag`) |
| `.` | The build context: the folder sent to the builder (Docker looks for `Dockerfile` here by default) |
| `-f path/Dockerfile` | Use a different Dockerfile |
| `--build-arg KEY=VALUE` | Pass a build-time variable |
| `--no-cache` | Ignore the build cache; rebuild everything |
| `--target stage` | Build only up to a named stage of a multi-stage Dockerfile |
| `--platform linux/amd64,linux/arm64` | Build for specific platforms |

**BuildKit and buildx.** Since Docker Engine 23.0, **BuildKit** is the default build engine for Linux containers. Docker's current docs describe BuildKit as "the default builder for Docker Desktop and Docker Engine users", with the legacy builder still used for Windows container builds. **Buildx** is the CLI plugin that exposes BuildKit's full feature set:

```bash
docker buildx build -t myapp:1.0 .                                    # same idea, full feature set
docker buildx build --platform linux/amd64,linux/arm64 -t myorg/myapp:1.0 --push .   # multi-arch + push
docker buildx ls                                                      # list builder instances
docker builder ls                                                     # same listing via `docker builder`
docker builder prune                                                  # clear build cache
docker buildx bake                                                    # build several targets defined in a file (docker-bake.hcl / compose)
```

`docker buildx ls` on the test machine:

```
NAME/NODE           DRIVER/ENDPOINT     STATUS    BUILDKIT   PLATFORMS
default             docker
 \_ default          \_ default         running   v0.34.0    linux/amd64 (+3), linux/arm64, linux/arm (+2), linux/ppc64le, (2 more)
desktop-linux*      docker
 \_ desktop-linux    \_ desktop-linux   running   v0.34.0    linux/amd64 (+3), linux/arm64, ...
```

Each builder instance (with its BuildKit version) lists the platforms it can build for. `docker builder` manages builds (`docker builder prune`, `docker builder ls`). `docker buildx` adds more (create builders, `bake`, `imagetools`).

`docker init` (a plugin) generates a starter `Dockerfile`, `.dockerignore`, and `compose.yaml` for a project.

---

## 17. Scanning images: `docker scout`, Trivy, and friends

Images contain OS packages and libraries that may have known vulnerabilities (**CVEs**: Common Vulnerabilities and Exposures, public IDs such as CVE-2025-12345). **Image scanners** read an image's contents, match packages against vulnerability databases, and report findings.

| Tool | Usage | Notes |
|---|---|---|
| **Docker Scout** | `docker scout quickview nginx:1.27`, `docker scout cves nginx:1.27`, `docker scout recommendations nginx:1.27` | Plugin built into Docker's CLI/Desktop. Free tier available; some features need a subscription. Check Docker's current plans. |
| **Trivy** (Aqua Security, open source) | `trivy image nginx:1.27` | Fast; scans OS packages and language dependencies, plus IaC and secrets. |
| **Grype** (Anchore, open source) | `grype nginx:1.27` | Open-source scanner. |
| **Snyk** | `snyk container test nginx:1.27` | Commercial with free tier. |

```bash
docker scout quickview nginx:1.27      # summary counts by severity (critical/high/medium/low)
docker scout cves nginx:1.27           # detailed vulnerability list
trivy image --severity HIGH,CRITICAL nginx:1.27
```

Interpreting results, fixes (smaller base images, updates, multi-stage builds), SBOMs, image signing, and CI gating are in [Chapter 10](10-security.md). Unverified: whether a particular Scout feature is free or paid. Check Docker's pricing page.

---

## 18. Other commands and plugins you will see

`docker --help` on the test machine (Docker Desktop 4.96.3) lists these plugins under "Management Commands" (marked `*`):

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

The one-line descriptions come from live output. Unverified: availability, pricing, and maturity of each (they are newer and some are beta). Follow Docker's docs for each.

Other built-in management commands: `docker volume` (Chapter 5), `docker network` (Chapter 5), `docker system` (`df`, `prune`, `events`, `info`), `docker manifest` (inspect multi-platform manifests), `docker plugin`, `docker swarm` (Docker's built-in clustering mode; see Chapter 11), `docker bake`.

`docker run --use-api-socket` (listed in `docker run --help`: "Bind mount Docker API socket and required auth") gives the container access to the Docker API. Treat it with the same caution as mounting `/var/run/docker.sock` ([Chapter 10](10-security.md)).

---

## 19. Hands-on lab

All steps use throwaway containers with unique names and clean up at the end. Outputs are examples; IDs, times, and random names differ. If a name or port collides, change it. In PowerShell, use backticks instead of `\` for line continuation, or write commands on one line.

### Lab 1: run nginx, view it, read its logs

```bash
docker run -d --name lab-web -p 8080:80 nginx:1.27
docker ps
curl http://localhost:8080
docker logs -f --tail 5 lab-web
```

- `docker run -d --name lab-web -p 8080:80 nginx:1.27`: start nginx detached, named `lab-web`, host port 8080 to container port 80. The first run pulls the image.
- `docker ps`: shows `lab-web` with `0.0.0.0:8080->80/tcp`.
- `curl http://localhost:8080`: returns HTML containing `<title>Welcome to nginx!</title>`. In PowerShell use `curl.exe` (plain `curl` in Windows PowerShell 5.1 is an alias for `Invoke-WebRequest`).
- `docker logs -f --tail 5 lab-web`: shows nginx startup lines and access-log lines for each request. Ctrl+C stops following, not the container.

### Lab 2: look inside nginx and prove copy-on-write

```bash
docker exec lab-web nginx -v                 # nginx version: nginx/1.27.x (written to stderr)
docker exec -it lab-web bash                 # interactive shell; type "exit" to leave (the container keeps running)
docker cp lab-web:/usr/share/nginx/html/index.html ./index.html
# edit ./index.html (change the heading text), then copy it back
docker cp ./index.html lab-web:/usr/share/nginx/html/index.html
curl http://localhost:8080                   # shows the edited page
docker diff lab-web                          # C /usr/share/nginx/html/index.html, A /run/nginx.pid, ...
docker rm -f lab-web                         # -f: stop (kill) and remove in one step
docker run -d --name lab-web -p 8080:80 nginx:1.27
curl http://localhost:8080                   # back to "Welcome to nginx!"
```

The edit exists only in the first container's writable layer ([Chapter 1](01-what-is-docker.md#53-union--overlay-filesystems-what-files-does-this-process-have)). `ps` may be missing inside minimal images.

### Lab 3: PostgreSQL with persistent data

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
| `-e POSTGRES_USER/PASSWORD/DB` | Initial configuration, read by the image's entrypoint on first start. The password is a demo value. Real secrets do not belong in commands or Git. |
| `-p 127.0.0.1:5433:5432` | Host port 5433 (localhost only) to Postgres's 5432. 5433 avoids clashing with an existing Postgres. |
| `-v lab-pgdata:/var/lib/postgresql/data` | Named volume (created automatically) that holds the database files. |
| `postgres:17` | Official image, major version 17 (pinned). |

Note: some newer major versions of the official Postgres image changed their default data directory layout, which changes the recommended mount path. Unverified: details for versions newer than 17. Read the image's Docker Hub page before upgrading.

```bash
docker logs -f lab-db                          # wait for "database system is ready to accept connections" (appears twice), then Ctrl+C
docker exec lab-db pg_isready -U labuser -d labdb      # /var/run/postgresql:5432 - accepting connections
docker exec -it lab-db psql -U labuser -d labdb        # psql prompt: labdb=#
```

In `psql`:

```sql
CREATE TABLE notes (id serial PRIMARY KEY, body text);
INSERT INTO notes (body) VALUES ('docker volumes persist data');
SELECT * FROM notes;
\q
```

Remove the container, keep the volume, and start a new container on the same volume:

```bash
docker rm -f lab-db
docker volume ls                       # lab-pgdata is still there
docker run -d --name lab-db \
  -e POSTGRES_USER=labuser -e POSTGRES_PASSWORD=lab-password-do-not-reuse -e POSTGRES_DB=labdb \
  -p 127.0.0.1:5433:5432 -v lab-pgdata:/var/lib/postgresql/data postgres:17
sleep 5   # PowerShell: Start-Sleep 5
docker exec lab-db psql -U labuser -d labdb -c "SELECT * FROM notes;"
```

The row is still present, and the log says `PostgreSQL Database directory appears to contain a database; Skipping initialization`. Without `-v`, the data would be lost with the container.

### Lab 4: resource limits, OOM, exit codes, and stop timing

```bash
docker run -d --name lab-limited --memory 256m --cpus 0.5 nginx:1.27
docker stats --no-stream lab-limited          # MEM USAGE / LIMIT shows .../256MiB

docker run --name lab-oom -m 20m --memory-swap 20m python:3.13-slim \
  python -c "x = b'x' * (100*1024*1024); print('allocated')"
echo "exit code: $?"          # PowerShell: $LASTEXITCODE
docker inspect --format 'exit={{.State.ExitCode}} oomkilled={{.State.OOMKilled}}' lab-oom

docker run --rm alpine sh -c 'exit 3'; echo "code=$?"        # 3
docker run --rm alpine nosuchprogram;  echo "code=$?"        # 127
docker run --rm --bogus alpine;        echo "code=$?"        # 125, "unknown flag: --bogus"

docker run -d --name lab-slow alpine sh -c 'while true; do sleep 1; done'
time docker stop lab-slow                                     # about 10 seconds
docker inspect --format '{{.State.ExitCode}}' lab-slow        # 137
docker rm lab-slow
```

Expected for the OOM test: no `allocated` output, `exit code: 137`, `exit=137 oomkilled=true`. On some Docker Desktop setups the kill message differs slightly. Exit code 137 is the reliable signal. The last block shows PID 1 ignoring SIGTERM ([12.4](#124-demonstration-slow-stop-vs-fast-stop)).

### Lab 5: disk usage and cleanup

```bash
docker system df
docker ps -a --filter "name=lab-"
docker volume ls
docker rm -f lab-web lab-db lab-limited lab-oom 2>/dev/null     # PowerShell: add 2>$null or ignore errors
docker volume rm lab-pgdata
docker rmi python:3.13-slim postgres:17 nginx:1.27 alpine       # images pulled in this lab (skip any to keep)
rm -rf lab-scripts index.html                                    # PowerShell: Remove-Item -Recurse -Force lab-scripts, index.html
docker ps -a --filter "name=lab-"                                # should be empty
docker volume ls --filter "name=lab-"                            # should be empty
```

Only the lab's own objects are removed (explicit names, no broad prune). If `docker rmi` reports an image "being used by stopped container ..." or "must be forced", find the container with `docker ps -a`. Remove it if it is yours. Leave the image if it belongs to something else.

---

## 20. Common mistakes

| # | Mistake | Symptom | Fix |
|---|---|---|---|
| 1 | Flags after the image name (`docker run nginx -d -p 80:80`) | Flags go to the container's program; nothing detaches or publishes | Put all Docker options before the image name |
| 2 | Wrong port order (`-p 80:8080` when the app listens on 80) | Connection reset / empty reply | `-p HOST:CONTAINER`; the container port is where the app listens |
| 3 | App listens on `127.0.0.1` inside the container | Published port does not respond | Bind to `0.0.0.0` ([10.5](#105-the-classic-gotcha-listening-on-127001-inside-the-container)) |
| 4 | Expecting `docker restart` or `start` to pick up a new image or flags | Old behavior persists | `docker rm -f` and `docker run` again (or use Compose) |
| 5 | Data stored in the container filesystem | Data lost on `docker rm` | Use a named volume ([Lab 3](#lab-3-postgresql-with-persistent-data)) |
| 8 | Name conflict (`Conflict. The container name ... is already in use`) | `docker run` fails (exit 125) | `docker rm` the old container or choose a new name |
| 9 | `docker exec` on a stopped container | `Error response from daemon: container ... is not running` | `docker start` first (or `docker logs` for crash analysis) |
| 11 | `docker system prune -a --volumes` out of habit | Lost databases | Never run blind; review first ([13.3](#133-warnings-read-these)) |
| 12 | `-p 5432:5432` for a database on a public server | Database exposed to the internet | Bind to localhost (`127.0.0.1:5432:5432`) or use an internal network |
| 13 | Secrets passed with `-e SECRET=...` in committed scripts | Leaks via shell history, `docker inspect`, process lists, Git | Use `-e NAME` (copy from environment), secret managers, or Compose/Swarm secrets (Chapter 10). Anyone who can run `docker inspect` can read env vars |
| 18 | `docker stop` takes 10 s | PID 1 ignores SIGTERM | Exec-form `CMD`, handle signals, or `--init` ([Section 12](#12-pid-1-and-signals)) |
| 19 | `-v` with a non-existent host path | Docker creates an empty directory with that name | Use `--mount` (errors instead) or verify paths |

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

**Next:** [Chapter 4: Dockerfile Deep Dive](04-dockerfile-deep-dive.md)

**Previous:** [Chapter 2: Installation and Operating Systems](02-installation-and-operating-systems.md)
