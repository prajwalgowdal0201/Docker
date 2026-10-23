# Chapter 12: Troubleshooting and Cheat Sheet

This chapter covers a debugging method, the debugging toolbox, a catalogue of common errors (message, cause, fix), build debugging, performance tuning and a command cheat sheet.

Commands were checked against Docker Engine 29.x, Compose v5.x and Buildx 0.38. OS-specific behaviour is marked.

## Table of contents

1. [A systematic debugging method](#1-a-systematic-debugging-method)
2. [The debugging toolbox](#2-the-debugging-toolbox)
3. [The error catalogue (40+ errors)](#3-the-error-catalogue)
4. [Debugging builds](#4-debugging-builds)
5. [Performance tuning tips](#5-performance-tuning-tips)
6. [The one-page cheat sheet](#6-the-one-page-cheat-sheet)
7. [Common mistakes about troubleshooting itself](#7-common-mistakes-about-troubleshooting-itself)

---

## 1. A systematic debugging method

Check one layer at a time, from the engine up to the host.

```
 Layer 1: Is the DOCKER ENGINE (daemon) up?             -> docker version / docker info
 Layer 2: Is the IMAGE what I expect?                   -> docker images / docker history / pull again
 Layer 3: Did the CONTAINER start, and why did it stop? -> docker ps -a / docker logs / exit code
 Layer 4: Is the APP inside healthy and configured?     -> docker exec / env / config files
 Layer 5: Can it reach what it needs? (NETWORK / DNS)   -> docker network inspect / netshoot
 Layer 6: Does it have its DATA / permissions?          -> docker volume inspect / ls -l / id
 Layer 7: Does it have enough RESOURCES?                -> docker stats / OOMKilled / df -h
 Layer 8: Is it the HOST / OS / PROXY?                  -> firewall, WSL2, corporate TLS proxy
```

### The seven-step method

1. Read the full error. The last lines usually hold the cause. Copy the exact text.
2. Reproduce with the smallest command (one service, or a plain `docker run`).
3. Identify what changed: image tag, base image, machine, Docker Desktop version, `.env`, code.
4. Gather facts with the standard five commands below.
5. Change one thing, then retest.
6. Check the layers in order.
7. Record the fix in a runbook.

### The "standard five" fact-gathering commands

Replace `NAME` with the container name or ID.

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
docker logs --tail 100 --timestamps NAME
docker inspect NAME --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}} err={{.State.Error}} health={{if .State.Health}}{{.State.Health.Status}}{{end}}'
docker stats --no-stream
docker system df
```

| Command | Meaning |
|---|---|
| `docker ps -a` | Lists all containers; `-a` includes stopped ones. |
| `--format "table ..."` | Go template that picks columns. `{{.Field}}` is a field, `\t` is a tab. |
| `docker logs --tail 100 --timestamps` | Last 100 log lines with timestamps. |
| `docker inspect NAME --format '...'` | Prints JSON details; `--format` extracts exit code, OOM flag, start error and health status. |
| `docker stats --no-stream` | One snapshot of CPU, memory, network and disk I/O. |
| `docker system df` | Disk used by images, containers, volumes and build cache. |

Example output of the first command (illustrative):

```
NAMES      IMAGE            STATUS                      PORTS
web        nginx:1.27       Up 3 minutes                0.0.0.0:8080->80/tcp
api        chat-api:1.0     Exited (1) 2 minutes ago
```

### Reading an exit code

The **exit code** is the number a process returns to its parent when it ends. A code above 128 means "killed by a signal"; the signal number is the code minus 128.

| Exit code | Meaning | Typical cause |
|---|---|---|
| `0` | Finished normally | Short-lived command, or the main process daemonised and returned |
| `1` | Generic application error | Exception in the app; read the logs |
| `2` | Shell builtin misuse or usage error | Bad arguments |
| `125` | `docker run` itself failed | Bad Docker flag, daemon error |
| `126` | Command found but cannot execute | File not executable, wrong format |
| `127` | Command not found | Typo in `CMD`, binary missing, wrong PATH |
| `130` | SIGINT (128 + 2) | Ctrl+C |
| `137` | SIGKILL (128 + 9) | OOM kill, `docker kill`, or stop timeout expired |
| `139` | Segmentation fault (128 + 11) | Native-code crash, wrong-architecture library, bad CUDA install |
| `143` | SIGTERM (128 + 15) | Normal `docker stop` |

---

## 2. The debugging toolbox

### 2.1 `docker logs`

```bash
docker logs NAME                  # everything
docker logs -f NAME               # follow new lines; Ctrl+C stops following
docker logs --tail 50 NAME        # last 50 lines
docker logs --since 10m NAME      # last 10 minutes
docker logs --since 2026-10-23T09:00:00 --until 2026-10-23T09:30:00 NAME
docker logs -t NAME               # prefix timestamps
docker compose logs -f --tail 100 api db   # several Compose services, interleaved
```

- `--since` / `--until` accept durations (`10m`, `2h`) or timestamps.
- Logs come from stdout and stderr. An app that logs to a file shows nothing; log to stdout or run `docker exec NAME cat /path/to/log`.
- Python output can be delayed by buffering. Set `PYTHONUNBUFFERED=1` or run `python -u`.
- Removing a container removes its logs, unless a log driver shipped them elsewhere.

### 2.2 `docker inspect`

Returns everything Docker knows about a container, image, volume or network, as JSON.

```bash
docker inspect NAME                                         # whole JSON
docker inspect NAME --format '{{.State.Status}}'            # running, exited, restarting...
docker inspect NAME --format '{{json .Config.Env}}'         # environment variables
docker inspect NAME --format '{{json .NetworkSettings.Networks}}'   # networks and IPs
docker inspect NAME --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} ({{.Type}}){{"\n"}}{{end}}'
docker inspect NAME --format '{{.HostConfig.RestartPolicy.Name}} restarts={{.RestartCount}}'
docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'
```

- `{{json X}}` prints a value as JSON; `{{range .Mounts}}...{{end}}` loops over mounts.
- In Windows PowerShell, use double quotes around the format and escape inner double quotes, or use Git Bash.

### 2.3 `docker exec`

```bash
docker exec -it NAME sh                 # interactive shell (bash if available)
docker exec NAME env                    # run one command and exit
docker exec -u 0 -it NAME sh            # run as root (UID 0)
docker exec -w /app NAME ls -la         # set the working directory
docker compose exec api sh              # Compose form, uses the service name
```

- `-i` keeps stdin open; `-t` allocates a terminal.
- `exec` works only on a running container. Minimal images (distroless, `scratch`) have no shell and fail with `executable file not found in $PATH`; use `docker debug` or a sidecar.

### 2.4 Running with a different entrypoint

If a container exits immediately, start a new container from the same image with a shell:

```bash
docker run --rm -it --entrypoint sh chat-api:1.0
```

- `--entrypoint sh` overrides the image's `ENTRYPOINT`. Run the start command by hand to see the error.
- `--rm` deletes the container on exit.

Compose: `docker compose run --rm --entrypoint sh api` starts a one-off container with the service's volumes, networks and environment.

### 2.5 `docker events`

```bash
docker events --since 30m --filter type=container
```

- Streams lifecycle events: `create`, `start`, `die`, `oom`, `kill`, `health_status`, `restart`. An `oom` event confirms a memory kill.
- `--filter type=container` limits output to containers (others: `image`, `volume`, `network`).

Illustrative output:

```
2026-10-23T10:15:02.123456789Z container oom 8f3c... (image=chat-api:1.0, name=api)
2026-10-23T10:15:02.234567890Z container die  8f3c... (exitCode=137, image=chat-api:1.0, name=api)
```

### 2.6 netshoot

**netshoot** (`nicolaka/netshoot`) is a community image containing network tools (`curl`, `dig`, `tcpdump`, and others) that production images usually lack. Attach it to the **network namespace** (the container's own network stack: IP, DNS, ports) of the problem container:

```bash
docker run --rm -it --network container:api nicolaka/netshoot
```

- `--network container:api` shares the network stack of container `api`.

| Command inside | Use |
|---|---|
| `curl -v http://db:5432` | Test reachability |
| `dig db` / `nslookup db` | Test DNS |
| `ss -tulpn` | List listening sockets |
| `tcpdump -i eth0 port 8000` | Capture packets |
| `traceroute`, `mtr`, `iperf3` | Path and throughput tests |

To test resolution as another container on a Compose network: `docker run --rm -it --network myproject_default nicolaka/netshoot`, then `nslookup api`.

### 2.7 `docker debug`

`docker debug` opens a shell with extra tools (vim, curl, htop) inside any container or image, including ones with no shell (distroless, `scratch`, stopped containers), without modifying the image.

```bash
docker debug api            # attach to a container named api
docker debug chat-api:1.0   # debug shell for an image
```

- It is a **Docker Desktop** feature: generally available in 4.33 and, per the Desktop release notes, free for all users in 4.49. `docker debug --help` works on the guide author's machine.
- Unverified: no evidence found that it ships with plain Docker Engine on Linux servers. Check Docker's docs.
- Alternatives on servers: a `nicolaka/netshoot` sidecar, the `cdebug` tool, or `kubectl debug` with an ephemeral container.

### 2.8 Debugging without a shell

Share the target's namespaces from a tool container:

```bash
docker run --rm -it --pid container:api --network container:api --cap-add SYS_PTRACE nicolaka/netshoot
```

- `--pid container:api` shares the process namespace, so `ps aux` shows the other container's processes.
- `--cap-add SYS_PTRACE` allows tracers such as `strace` to attach.
- This weakens isolation. Use it only on non-production systems or with authorisation.

Other options when `strace` cannot be installed:

1. **Read `/proc` from the host.** Get the PID with `docker inspect -f '{{.State.Pid}}' NAME`, then run `ls -l /proc/PID/fd` (open files), `cat /proc/PID/status` (memory, threads) and `cat /proc/PID/limits`. On Docker Desktop the host is a VM; enter it with `docker run --rm -it --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh` (privileged).
2. Raise the app's log level (`LOG_LEVEL=debug`).
3. Copy files out with `docker cp`.
4. Build a debug image variant (`FROM your-image` plus `RUN apk add strace curl`). Never ship it.
5. Snapshot a broken container (debugging only): `docker commit NAME debug-snapshot:1`, then `docker run --rm -it --entrypoint sh debug-snapshot:1`.

---

## 3. The error catalogue

Entries are grouped by area. Search for a distinctive phrase from the error.

### Group A: Docker Engine and permissions

#### A1. Cannot connect to the Docker daemon

```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
error during connect: Get "http://%2F%2F.%2Fpipe%2FdockerDesktopLinuxEngine/v1.51/containers/json": open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
```
The first line is Linux; the second is Docker Desktop (example).

**Cause:** The **daemon** (the background service that does the work) is not running, or the CLI points at the wrong place. The `docker` command is only a client.

**Fix:**
- Linux: `sudo systemctl start docker` and `sudo systemctl enable docker`. Check `systemctl status docker` and `journalctl -u docker --since "10 min ago"`. An invalid `/etc/docker/daemon.json` is a common cause. Validate with `dockerd --validate --config-file=/etc/docker/daemon.json`.
- Windows/macOS: start **Docker Desktop** and wait for "Engine running".
- Check the target engine with `docker context ls` (asterisk = active; on Windows normally `desktop-linux`). Switch with `docker context use desktop-linux`.
- Check that `DOCKER_HOST` is not set to a stale value (`echo $DOCKER_HOST`; empty is normal).

#### A2. Permission denied on the Docker socket

```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: ... connect: permission denied
```
**Cause:** On Linux, `/var/run/docker.sock` belongs to root and the `docker` group. The user is in neither.

**Fix:**
```bash
sudo usermod -aG docker $USER     # add yourself to the docker group
newgrp docker                     # apply in this shell (or log out and in)
docker ps                         # test without sudo
```
- `-aG docker` appends (`-a`) the user to the supplementary group (`-G`) `docker`.
- Any member of the `docker` group can become root on the host (a container can mount `/`). Add only trusted users.
- Do not use `chmod 666 /var/run/docker.sock`; it gives every user root-equivalent access. For stricter setups use rootless Docker ([Chapter 10](10-security.md)).

#### A3. "docker: command not found" / "docker-compose: command not found"

**Cause:** Docker is not installed or not on PATH, or a script uses the old standalone `docker-compose` (hyphen). Current Compose is a plugin invoked as `docker compose` (space).

**Fix:** install Docker Engine and the Compose plugin ([Chapter 2](02-installation-and-operating-systems.md)); update scripts to `docker compose`. Test with `docker compose version` (prints `Docker Compose version v5.6.1` on the guide author's machine).

#### A4. Docker Desktop will not start / stuck on "Starting"

**Symptoms:** the whale icon spins indefinitely; "WSL distro terminated abruptly"; "Docker Desktop stopped".

**Causes and fixes, in order:**
1. **WSL 2 outdated or unhealthy (Windows).** In PowerShell run `wsl --update`, `wsl --shutdown`, then start Docker Desktop. Check `wsl -l -v`: `docker-desktop` should be `Running`, version `2`.
2. **Virtualization disabled** in BIOS/UEFI (Intel VT-x / AMD-V / SVM). Task Manager > Performance > CPU should show "Virtualization: Enabled". Enable the Windows features **Virtual Machine Platform** and **Windows Subsystem for Linux**.
3. **Another hypervisor or antivirus interfering.** Test with real-time scanning off, or add exclusions per Docker's docs.
4. **Corrupt state.** Troubleshoot menu (bug icon): **Restart**, then **Clean / Purge data** (deletes local images, containers and volumes; back up first), then **Reset to factory defaults** as a last resort.
5. **Disk full** on the drive holding the WSL virtual disk (`docker_data.vhdx`).
6. **Stale `~/.docker/config.json`** referencing a missing credential helper (`docker-credential-desktop` not found). Fix PATH or remove the `credsStore` line.
7. Collect diagnostics: Troubleshoot menu > Get support / Gather diagnostics.

Current guidance: <https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/>.

#### A5. "driver failed programming external connectivity" (iptables)

```
Error response from daemon: driver failed programming external connectivity on endpoint web (...): (iptables failed: iptables --wait -t nat -A DOCKER ...: iptables: No chain/target/match by that name.
```
**Cause:** The firewall rules Docker manages were flushed (for example by `ufw reload` or a `firewalld` restart).

**Fix:** `sudo systemctl restart docker` recreates the chains. Avoid tools that flush all rules; put custom rules in the `DOCKER-USER` chain. On `nftables` distributions, check that Docker and the firewall use compatible backends (`docker info`, Docker's firewall docs for your version).

### Group B: Ports, names and conflicts

#### B1. Port is already allocated

```
Bind for 0.0.0.0:8080 failed: port is already allocated.
```
Windows variant: `listen tcp 0.0.0.0:8080: bind: Only one usage of each socket address (protocol/network address/port) is normally permitted.`

**Cause:** Another container or a host program already listens on that host port.

**Fix:**
```bash
docker ps --filter "publish=8080"        # which container publishes 8080?
sudo lsof -i :8080                       # Linux/macOS owner (or: sudo ss -ltnp | grep 8080)
netstat -ano | findstr :8080             # Windows: shows the PID
Get-Process -Id <PID>                    # Windows PowerShell
```
- `--filter "publish=8080"` finds containers publishing that port. `lsof` / `ss` / `netstat -ano` find the owning process.
- Stop the other process, or change the host side of the mapping: `-p 8081:80` (host left of the colon, container right).
- Windows: Hyper-V and WSL can reserve port ranges. List them with `netsh interface ipv4 show excludedportrange protocol=tcp` and pick a port outside them.
- Prevention: `${WEB_PORT:-8080}:80` in Compose; publish only needed ports (containers on the same network reach each other without publishing).

#### B2. Container name is already in use

```
Conflict. The container name "/api" is already in use by container "8f3c...". You have to remove (or rename) that container to be able to reuse that name.
```
**Cause:** A container (possibly stopped) with that `--name` exists. Names are unique.

**Fix:** `docker rm api` (`docker rm -f api` if running), choose another name, or use `--rm` for one-off containers. In Compose, avoid hard-coded `container_name:`.

#### B3. "Pool overlaps with other one on this address space"

```
Error response from daemon: invalid pool request: Pool overlaps with other one on this address space
```
**Cause:** A new network's subnet overlaps an existing Docker network or the host/VPN range. Related symptom: containers cannot reach corporate services because Docker's default `172.17.0.0/16` or `192.168.x.x` ranges clash with the VPN.

**Fix:** list subnets (`docker network ls`, `docker network inspect NET --format '{{json .IPAM.Config}}'`), remove unused networks (`docker network prune` removes only networks no container uses), or pick another subnet. To change the defaults, set `default-address-pools` in `daemon.json` ([Chapter 11](11-production-and-enterprise.md#16-docker-daemon-production-configuration-daemonjson)).

### Group C: Disk and memory

#### C1. No space left on device

```
write /var/lib/docker/...: no space left on device
ERROR: failed to solve: ... no space left on device
```
**Cause:** Docker storage (images, containers, volumes, build cache, unrotated container logs) filled the disk. On Docker Desktop it can be the virtual disk's size cap.

**Fix:**
```bash
docker system df                   # where is the space?
docker builder prune -f            # remove unused build cache (safe)
docker image prune -a              # remove images not used by any container (re-pullable)
docker container prune             # remove stopped containers
sudo du -ah /var/lib/docker/containers | sort -rh | head   # large log files (Linux host)
```
- Read [Chapter 11, Section 17](11-production-and-enterprise.md#17-disk-management-and-cleanup-policies) before pruning on a server.
- Do not run `docker volume prune` or `docker system prune --volumes` unless no needed data is in unused volumes. A stopped database's volume looks unused.
- Huge `*-json.log`: configure rotation (`max-size`, `max-file`) and recreate the container. Emergency truncate: `sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log`.
- Docker Desktop (WSL 2): the virtual disk grows but does not shrink automatically. After pruning, reclaim space via Docker Desktop's disk-image settings, or compact the VHDX (`wsl --shutdown`, then `Optimize-VHD` in an admin PowerShell where available).
- Check host disks and inodes with `df -h` and `df -i`. Inodes can run out with millions of small files.
- Prevention: log rotation, a weekly prune of images and build cache, alerting at 80% disk.

#### C2. OOMKilled / exit code 137

**Signs:** `Exited (137)`; `OOMKilled=true` in `docker inspect`; the process vanishes with no stack trace; an `oom` event in `docker events`.

**Cause:** The container exceeded its memory limit (or the host ran out of RAM) and the kernel's **OOM killer** sent SIGKILL (128 + 9 = 137). Exit 137 also results from `docker kill` or a `docker stop` that exceeded its timeout, so confirm with the `OOMKilled` flag.

**Fix:**
```bash
docker inspect NAME --format 'OOM={{.State.OOMKilled}} limit={{.HostConfig.Memory}}'
docker stats --no-stream            # memory usage vs limit
```
- If `OOM=true`, raise the limit (`--memory=2g`, or Compose `deploy.resources.limits.memory: 2G`) or reduce memory use (smaller batches, fewer workers, a smaller or quantised model, streaming).
- Make the runtime respect the limit: JVM `-XX:MaxRAMPercentage=75`, Node `--max-old-space-size`, Gunicorn `--workers`, `OMP_NUM_THREADS` for numeric libraries.
- On Docker Desktop (WSL 2) the whole VM has a memory cap, so a container can die with no container-level limit. See C3.
- A 7B-parameter model in 16-bit needs roughly 14 GB for weights alone; a smaller limit or VM causes an OOM kill during load. Use a quantised model or more memory ([Chapter 8](08-docker-for-ai-and-ml.md)).

#### C3. WSL 2 `vmmem` / `VmmemWSL` using too much memory (Windows)

**Symptom:** Task Manager shows `vmmem` or `Vmmem WSL` using many gigabytes (often half of RAM or more) while Docker is idle.

**Cause:** Docker Desktop runs in a WSL 2 VM. By default WSL 2 may use a large fraction of host memory, and Linux keeps spare memory as file cache and returns it slowly.

**Fix:** create or edit `%UserProfile%\.wslconfig` (`C:\Users\<you>\.wslconfig`):

```ini
[wsl2]
memory=6GB
processors=4
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```
| Setting | Effect |
|---|---|
| `memory=6GB` | Caps the VM's RAM |
| `processors=4` | Limits CPU cores |
| `swap=2GB` | Sets swap size |
| `autoMemoryReclaim=gradual` | Returns idle cache to Windows gradually. Experimental; check `wsl --version` and Microsoft's docs. |

- Apply with `wsl --shutdown` in PowerShell, then restart Docker Desktop.
- Set limits high enough for the heaviest container (see C2).
- Docker Desktop **Settings > Resources** can set limits on some configurations; WSL 2 mode defers to `.wslconfig`.

### Group D: Architecture, images and registries

#### D1. exec format error (architecture mismatch)

```
exec /usr/local/bin/python: exec format error
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```
An older form is `standard_init_linux.go: exec user process caused: exec format error`.

**Cause:** The image was built for a different **CPU architecture** (`amd64` vs `arm64`) than the host. Typical case: built on an Apple Silicon Mac (arm64), run on an amd64 server, or the reverse. Other causes: a script without a shebang line, or Windows line endings (E3).

**Fix:**
```bash
docker image inspect IMAGE --format '{{.Os}}/{{.Architecture}}'      # image platform
docker run --platform linux/amd64 IMAGE                              # force a platform (emulates if needed)
docker buildx build --platform linux/amd64,linux/arm64 -t REG/IMG:TAG --push .   # build both
```
- `--platform linux/amd64` selects or emulates that architecture. Emulation (QEMU, or Rosetta on Apple Silicon) is much slower; use native images in production.
- A **multi-arch image** holds one variant per architecture under one tag, and Docker pulls the matching one (Chapter 11 has a CI example).

#### D2. pull access denied / repository does not exist

```
pull access denied for myapp, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```
**Cause:** A typo in the image name; a private image without login; an image that exists only locally; a wrong registry hostname.

**Fix:** check spelling and registry prefix (`ghcr.io/owner/name`); run `docker login REGISTRY`; for local-only images do not `pull` (in Compose use `build:` or `pull_policy: never`). For private GHCR images use a token with `read:packages`.

#### D3. manifest unknown / not found

```
manifest for myrepo/app:1.4.3 not found: manifest unknown: manifest unknown
no matching manifest for linux/arm64/v8 in the manifest list entries
```
**Cause:** The tag does not exist (typo, never pushed, removed by retention policy), or the image has no variant for the host platform (second message).

**Fix:** list tags in the registry UI or with `docker buildx imagetools inspect REPO:TAG`; fix the tag. For the platform case use `--platform linux/amd64` or ask the publisher for a multi-arch build.

#### D4. toomanyrequests (Docker Hub pull rate limit)

```
toomanyrequests: You have reached your unauthenticated pull rate limit. https://www.docker.com/increase-rate-limit
```
Older wording: "You have reached your pull rate limit. You may increase the limit by authenticating and upgrading."

**Cause:** Docker Hub caps pulls per six hours: 100 for anonymous users (counted per IP, so a shared office or CI IP exhausts it quickly) and 200 for free authenticated accounts, at the time of writing. See [Chapter 11, Section 5.3](11-production-and-enterprise.md#53-docker-hub-rate-limits).

**Fix:** `docker login` (use a personal access token); add credentials to CI; run a pull-through cache or mirror base images to your own registry; wait for the window to reset. Docker documents a test repository, `ratelimitpreview/test`, whose response headers (`ratelimit-limit`, `ratelimit-remaining`) show the remaining allowance; see the Docker Hub usage docs for the token steps.

#### D5. denied / unauthorized when pushing

```
denied: requested access to the resource is denied
unauthorized: authentication required
denied: permission_denied: write_package
```
**Cause:** Not logged in to the registry; the repository name lacks the namespace (`app:1.0` instead of `youruser/app:1.0`); the token lacks write scope (GHCR needs `write:packages`; GitHub Actions needs `permissions: packages: write`); or the account has no access.

**Fix:**
```bash
echo $TOKEN | docker login ghcr.io -u USER --password-stdin
docker tag app:1.0 ghcr.io/user/app:1.0
docker push ghcr.io/user/app:1.0
```
- `--password-stdin` reads the token from stdin, which keeps it out of shell history.
- Tag with the full registry path before pushing.

#### D6. invalid reference format

```
docker: invalid reference format: repository name (MyApp) must be lowercase.
```
**Cause:** Repository names must be lowercase and use only letters, digits, `-`, `_` and `.`. Also caused by stray spaces, a trailing `:` with no tag, or unset variables (`image: myapp:${TAG}` with `TAG` unset gives `myapp:`).

**Fix:** use lowercase; run `docker compose config` to see the substituted value; use `${TAG:-latest}` to default or `${TAG:?TAG is required}` to fail with a message.

### Group E: Running containers

#### E1. Container exits immediately

**Symptom:** `docker run -d myimage` prints an ID; `docker ps` shows nothing; `docker ps -a` shows `Exited (0)` or `Exited (1)`.

**Cause:** A container lives only as long as its **main process** (PID 1). Common reasons:
- The command is a one-shot task.
- The app daemonises into the background (`nginx` without `-g 'daemon off;'`, `service x start`), so PID 1 exits.
- The app crashed on startup (missing variable, database unreachable, bad config).
- Wrong `CMD` (exit 127) or non-executable file (exit 126).
- An interactive program (shell, REPL) needs a terminal; run with `-it`.

**Fix:**
```bash
docker ps -a                     # exit code
docker logs NAME                 # reason
docker run --rm -it --entrypoint sh IMAGE     # inspect interactively
docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'   # what it runs
```
- Run the server in the foreground, for example `CMD ["nginx", "-g", "daemon off;"]`.
- Do not use `tail -f /dev/null` to keep a container alive in production.

#### E2. Container keeps restarting (restart loop)

**Symptom:** `STATUS` alternates between `Restarting (1) 5 seconds ago` and `Up 2 seconds`.

**Cause:** A restart policy (`always`, `unless-stopped`, `on-failure`) with an app that crashes on start. Docker retries with an increasing delay.

**Fix:** read the logs (`docker logs --tail 200 NAME` works between restarts); check `docker inspect NAME --format '{{.RestartCount}} {{.State.ExitCode}}'`; set `restart: "no"` temporarily; fix the root cause (configuration, unreachable dependency, or OOM).

#### E3. CRLF line endings in shell scripts (Windows)

```
exec /entrypoint.sh: no such file or directory
/bin/sh^M: bad interpreter: No such file or directory
standard_init_linux.go: exec user process caused: no such file or directory
```
**Cause:** On Windows, Git (with `core.autocrlf=true`) and many editors save **CRLF** (`\r\n`) line endings. Linux expects **LF** (`\n`). The kernel reads the shebang as `#!/bin/sh\r`, looks for an interpreter named `sh\r`, and reports "no such file or directory" although the file exists.

**Fix:**
1. Convert the file: `dos2unix entrypoint.sh` or `sed -i 's/\r$//' entrypoint.sh`. In VS Code, click `CRLF` in the status bar and choose `LF`.
2. Add a `.gitattributes` file at the repo root, then run `git add --renormalize .`:
   ```
   * text=auto
   *.sh text eol=lf
   Dockerfile text eol=lf
   *.yaml text eol=lf
   ```
   `* text=auto` lets Git normalise text files; `*.sh text eol=lf` forces LF checkouts of shell scripts on every OS.
3. Strip `\r` during the build: `RUN sed -i 's/\r$//' /entrypoint.sh`.
4. Set `end_of_line = lf` in `.editorconfig`.

#### E4. "permission denied" running the entrypoint

```
exec /entrypoint.sh: permission denied
OCI runtime create failed: ... exec: "./start.sh": permission denied: unknown
```
**Cause:** The script is not executable. Files copied from Windows lose the executable bit.

**Fix:** `COPY --chmod=755 entrypoint.sh /entrypoint.sh` (BuildKit) or `RUN chmod +x /entrypoint.sh`; or `git update-index --chmod=+x entrypoint.sh`; or invoke through the interpreter: `CMD ["sh", "/entrypoint.sh"]`.

#### E5. OCI runtime: executable file not found in $PATH

```
OCI runtime create failed: runc create failed: unable to start container process: exec: "python3": executable file not found in $PATH: unknown
```
**Cause:** The command does not exist in the image (slim and alpine images lack tools such as `bash`, `curl`, `python3`), a typo, or a shell feature (`&&`, `$VAR`) used in exec form without a shell.

**Fix:** check available binaries (`docker run --rm IMAGE ls /usr/bin`); install the tool (`apk add --no-cache bash` or `apt-get install -y ...`); use `sh` instead of `bash` on Alpine. Exec form `CMD ["a","b"]` runs no shell; for shell features use `CMD ["sh","-c","cmd1 && cmd2"]`.

#### E6. Container unhealthy

**Symptom:** `STATUS` shows `Up 5 minutes (unhealthy)`, or `docker compose up --wait` fails with `container ... is unhealthy`.

**Cause:** The healthcheck fails: the app is not ready, the check uses the wrong port or path, the check tool (`curl`) is missing from the image, the check runs before startup finishes (no `start_period`), or its timeout is shorter than the response time.

**Fix:**
```bash
docker inspect NAME --format '{{json .State.Health}}' | python -m json.tool
```
- Prints recent health-check attempts with exact output and exit code.
- Run the check by hand: `docker exec NAME <test command>`.
- Add `start_period`, raise `timeout`, or use a tool present in the image (a one-line `python -c` or `wget -q --spider`).
- Plain Docker does not restart unhealthy containers; it only labels them.

#### E7. depends_on did not wait (startup race)

**Symptom:** the API crashes at boot with `connection refused` to the database, then works after a restart.

**Cause:** By default `depends_on: [db]` waits only until the `db` container has **started**, not until PostgreSQL accepts connections.

**Fix:** add a healthcheck to the dependency and wait for it:

```yaml
services:
  db:
    image: postgres:17
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 10
  api:
    build: .
    depends_on:
      db:
        condition: service_healthy
```
- `pg_isready` is Postgres's readiness probe. `condition: service_healthy` starts `api` only after the `db` healthcheck passes.
- The app should also retry database connections with backoff ([Chapter 6](06-docker-compose.md)).

#### E8. `localhost` confusion: connection refused between containers

**Symptoms:** `Connection refused` or `Can't connect to server on 'localhost'` from a container to a running database or API; or the browser cannot reach a service inside a container.

**Cause:** `localhost` means "this network namespace". Inside container A it is container A, not the host and not container B.

```
 Browser on host ---- localhost:8080 ----> published port ---> container "web" (port 80)
 Container api   ---- localhost:5432 ----> api itself  (WRONG: db is another container)
 Container api   ---- db:5432        ----> container "db" (RIGHT: service name on the same network)
```

**Fix:**
- Container to container: use the service name (`postgresql://user:pw@db:5432/app`) on the same user-defined network (Compose does this automatically).
- Host to container: use the published port (`localhost:8080`) and publish it (`-p 8080:80`). The published container port must be the port the app listens on.
- Container to a service on the host: use `host.docker.internal` (built in on Docker Desktop). On Linux Engine add `--add-host=host.docker.internal:host-gateway` (Compose: `extra_hosts: ["host.docker.internal:host-gateway"]`).
- The server must listen on `0.0.0.0` (all interfaces), not `127.0.0.1`, or published ports cannot reach it. Examples: `uvicorn app:app --host 0.0.0.0 --port 8000`; Flask `app.run(host="0.0.0.0")`. A loopback-only listener is the most common cause of "port published but browser shows connection reset / empty reply".

#### E9. Containers cannot resolve DNS or reach the internet

```
Temporary failure in name resolution
Could not resolve host: pypi.org
getaddrinfo ENOTFOUND registry.npmjs.org
dial tcp: lookup registry-1.docker.io: no such host
```
**Cause:** Docker's embedded DNS forwards to the host's resolvers. It fails when the host uses a local stub resolver (`127.0.0.53` on Ubuntu) that containers cannot reach, a VPN or corporate network changes DNS, the firewall blocks outbound DNS (UDP/TCP 53), or the machine is offline.

**Fix:**
```bash
docker run --rm busybox nslookup google.com                  # test from a container
docker run --rm busybox cat /etc/resolv.conf                 # DNS in use
docker run --rm --dns 8.8.8.8 busybox nslookup google.com    # explicit DNS
```
- If explicit DNS works, set it globally in `/etc/docker/daemon.json` as `{ "dns": ["1.1.1.1", "8.8.8.8"] }` (or corporate DNS servers) and restart Docker. In Compose use `dns:` per service.
- Docker Desktop with a VPN: restart Docker Desktop after connecting or disconnecting. Check for MTU problems (F3).
- Between containers: the **default bridge network** has no name-based DNS. Use a user-defined network ([Chapter 5](05-storage-and-networking.md)).

### Group F: Networking and proxies

#### F1. TLS / certificate errors (corporate proxy)

```
x509: certificate signed by unknown authority
curl: (60) SSL certificate problem: unable to get local issuer certificate
pip: SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] ...
```
**Cause:** Many company networks run a TLS-intercepting proxy that re-signs HTTPS traffic with the company's certificate authority (CA). The host OS trusts that CA. Containers keep their own CA list and do not, so HTTPS calls from containers and builds fail. A wrong system clock also causes this.

**Fix, by where it fails:**

1. **Daemon pulling images.** Install the corporate CA for the daemon. On Linux, place it at `/etc/docker/certs.d/<registry-host>/ca.crt` (per registry) or add it to the OS trust store (`sudo cp corp-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`) and restart Docker. On Docker Desktop, importing the CA into the Windows/macOS trust store is typically enough; restart Desktop. Set proxy variables if needed (below).
2. **During `docker build`.** Copy the CA into the image and update the trust store:
   ```dockerfile
   COPY corp-ca.crt /usr/local/share/ca-certificates/corp-ca.crt
   RUN update-ca-certificates
   ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
       SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt \
       NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/corp-ca.crt
   ```
   - `update-ca-certificates` merges the CA into the system bundle (Debian/Ubuntu; install `ca-certificates` first if missing). On Alpine run `apk add ca-certificates` first; on RHEL-family images use `update-ca-trust`.
   - The `ENV` lines point Python `requests`, OpenSSL tools and Node.js at the right bundle, because each runtime locates CAs differently.
   - A golden base image that already contains the company CA avoids repeating this ([Chapter 11](11-production-and-enterprise.md#21-team-practices)).
3. **At runtime.** Mount the CA and set the environment variables above, or bake it into the image.

Do not use `pip install --trusted-host`, `curl -k`, `NODE_TLS_REJECT_UNAUTHORIZED=0` or `insecure-registries` as a fix. They disable verification. Use them only briefly to diagnose, then install the CA.

**Proxy settings.** If outbound traffic must use an HTTP proxy, configure it in three places:

| Where | How |
|---|---|
| Docker Desktop | Settings > Resources > Proxies |
| Linux daemon | systemd drop-in `/etc/systemd/system/docker.service.d/http-proxy.conf` with `[Service]` and `Environment="HTTP_PROXY=http://proxy.example.com:3128"`, `Environment="HTTPS_PROXY=..."`, `Environment="NO_PROXY=localhost,127.0.0.1,.internal.example.com"`; then `sudo systemctl daemon-reload && sudo systemctl restart docker` |
| Containers and builds | `~/.docker/config.json` with `"proxies": {"default": {"httpProxy": "...", "httpsProxy": "...", "noProxy": "..."}}`, or `--build-arg HTTP_PROXY=...` per build |

#### F2. Registry timeouts / TLS handshake timeout

```
Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
net/http: TLS handshake timeout
```
**Cause:** No route to the registry: firewall, proxy not configured, DNS failure, VPN, or a slow network.

**Fix:** test with `curl -I https://registry-1.docker.io/v2/`. A `401 Unauthorized` response is expected: the registry is reachable and wants authentication. Then configure the proxy (F1), try another network, or use a registry mirror.

#### F3. Large downloads hang (MTU problem)

**Symptom:** small requests succeed; large downloads (`apt-get update`, big `pip` wheels, `git clone`) stall or time out, mostly on VPNs or some cloud networks.

**Cause:** The **MTU** (maximum transmission unit, the largest packet size) of the container network (default 1500) exceeds what the VPN or overlay path allows, so large packets are dropped.

**Fix:** set a smaller MTU in `daemon.json`: `{ "mtu": 1400 }` (or per network: `-o com.docker.network.driver.mtu=1400`), restart Docker, retest. If it still fails try 1350, then 1300.

#### F4. Published port not reachable from other machines (or too reachable)

| Case | Causes |
|---|---|
| Not reachable | App listens on `127.0.0.1` in the container (E8); mapping binds loopback (`-p 127.0.0.1:8080:80`); cloud security group or host firewall blocks the port; wrong host port |
| Too reachable | On Linux, published ports can bypass `ufw`. Bind internal services to `127.0.0.1` or do not publish them ([Chapter 11](11-production-and-enterprise.md#7-where-do-you-run-the-containers-deployment-targets)) |

**Check:** `docker port NAME`; `docker ps` (`0.0.0.0:8080->80/tcp` is open on all interfaces, `127.0.0.1:8080->80/tcp` is local only); test from outside with `curl http://SERVER_IP:8080` or a port scanner.

### Group G: Builds and files

#### G1. COPY failed: file not found / excluded by .dockerignore

```
COPY failed: file not found in build context or excluded by .dockerignore: stat app.py: file does not exist
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ...: "/app.py": not found
```
**Cause:** `COPY` paths are relative to the **build context** (the folder passed at the end of `docker build`, usually `.`), not to the Dockerfile. Reasons: the file is outside the context (`COPY ../x` is not allowed), the path is misspelled or has the wrong case (Linux is case-sensitive), or the file is listed in `.dockerignore` and was never sent to the builder.

**Fix:**
```bash
docker build -f docker/Dockerfile -t myapp .      # -f selects the Dockerfile; "." is still the context
cat .dockerignore                                 # is the file excluded?
docker buildx build --progress=plain .            # full output
```
- Check path and case, the context argument, and `.dockerignore`. Common culprits: `*.md`, `build/`, `dist/`, `*.json`, `.env`, or `*` followed by exceptions such as `!app.py`.
- For a Dockerfile in another directory, make the context the repo root and write paths from there, or use named contexts / `additional_contexts` in Compose.

#### G2. Build cache surprises

| Symptom | Cause |
|---|---|
| Change missing from the image | A cached layer was reused because the instruction text is unchanged (`RUN git clone ...`, `RUN curl ... latest`, `RUN apt-get update`) |
| Every build reinstalls all dependencies | An early `COPY . .` before `pip install` / `npm install` invalidates the cache on any file change |

Docker caches each instruction by its text and, for `COPY`/`ADD`, by file contents. When a layer changes, every later layer is rebuilt. Fixes:

1. Order instructions from least to most frequently changing: base image, system packages, dependency manifests, dependency install, then source.
   ```dockerfile
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   ```
2. Use BuildKit cache mounts: `RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt`.
3. Force a fresh build: `docker build --no-cache .` (ignore cache), `docker build --pull .` (also fetch the newest base image), or change a build `ARG` just before the layer to refresh.
4. Put `apt-get update && apt-get install -y ...` in the same `RUN`; pin versions where reproducibility matters.
5. Deleted files still occupy earlier layers: `RUN rm bigfile` in a later layer does not shrink the image. Download, use and delete in one `RUN`, or use a multi-stage build ([Chapter 4](04-dockerfile-deep-dive.md)).

#### G3. A build step fails: "did not complete successfully"

```
ERROR: failed to solve: process "/bin/sh -c pip install -r requirements.txt" did not complete successfully: exit code: 1
```
**Cause:** The command in a `RUN` returned non-zero. The reason is above this line in the output.

**Fix:** re-run with `docker build --progress=plain --no-cache .` ([Section 4](#4-debugging-builds)). Typical causes: network, DNS or proxy failure; a package needing build tools (`gcc`, `libpq-dev`); a version conflict; a package with no wheel for the CPU architecture (arm64); a missing build argument.

#### G4. Cannot find Dockerfile / unable to prepare context

```
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
unable to prepare context: path "/some/path" not found
```
**Cause:** Wrong directory, wrong file name (`dockerfile`, `Dockerfile.dev`), or a missing context argument.

**Fix:** run `ls` to confirm; use `-f path/to/Dockerfile`; in Compose check `build.context` and `build.dockerfile` (relative to the Compose file's directory).

#### G5. Bind mount permission denied

```
PermissionError: [Errno 13] Permission denied: '/data/output.txt'
mkdir: cannot create directory '/app/logs': Permission denied
```
**Cause:** On Linux, file ownership is by numeric user ID (UID). The container runs as, for example, UID 1000 or 10001, while the mounted host folder belongs to another UID or root. Docker Desktop on Windows/macOS usually hides this by translating permissions, so the problem appears on a Linux server. A mounted folder that did not exist is created as root.

**Fix (choose one):**
```bash
id                                                        # host UID and GID
docker run --user "$(id -u):$(id -g)" -v "$PWD/data:/data" IMAGE   # run as the host user
sudo chown -R 10001:10001 ./data                          # give the folder to the container's UID
```
- Compose: `user: "${UID:-1000}:${GID:-1000}"`, or create the folder beforehand with the right owner.
- Prefer named volumes for application data. Docker initialises them with the image's ownership of the mount path, if the directory was `chown`ed in the Dockerfile before first use.
- SELinux (RHEL, Fedora, Rocky): add `:z` (shared label) or `:Z` (private label), for example `-v ./data:/data:z`. `ausearch -m avc -ts recent` confirms SELinux denials.
- Read-only root filesystem (`--read-only`): write to a volume or a `tmpfs` (`--tmpfs /tmp`).

#### G6. Windows path problems in Git Bash (MSYS path mangling)

**Symptoms:** `docker run -v /c/Users/me/data:/data ...` mounts the wrong thing; `docker exec api ls /app` returns `ls: cannot access 'C:/Program Files/Git/app'`; `-w /app` becomes `-w C:/Program Files/Git/app`.

**Cause:** **Git Bash** (MSYS2) rewrites anything that looks like a Unix path into a Windows path before the command runs.

**Fix:**
```bash
MSYS_NO_PATHCONV=1 docker exec api ls /app           # one command
export MSYS_NO_PATHCONV=1                            # whole session (add to ~/.bashrc)
docker exec api sh -c "ls /app"                      # or wrap the path in a quoted sh -c
docker run -v //c/Users/me/data:/data IMAGE          # a double leading slash avoids conversion
```
Or use PowerShell or WSL. In PowerShell write mounts as `-v "${PWD}/data:/data"` (braces and quotes) or use the Compose `./data:/data` form.

#### G7. Host file changes do not appear in the container (hot reload)

**Cause:** On Windows/macOS, file-change notifications may not propagate through bind mounts into the Linux VM, so watchers (webpack, `uvicorn --reload`, nodemon) miss edits. Code copied into the image at build time is a copy, not live.

**Fix:** use a bind mount in development (`./src:/app/src`); keep the project in the WSL 2 Linux filesystem (`\\wsl$\Ubuntu\home\you\project`) instead of `C:\...`; or switch the watcher to polling (`CHOKIDAR_USEPOLLING=true`, `WATCHFILES_FORCE_POLLING=true`, `nodemon -L`). Compose `develop.watch` (`docker compose watch`) syncs or rebuilds on change ([Chapter 6](06-docker-compose.md)).

### Group H: Compose-specific

#### H1. Data missing after `docker compose down -v`

**Symptom:** the database is empty after restarting the stack.

**Cause:** `docker compose down` removes containers and networks but keeps named volumes. `-v` (`--volumes`) also deletes named volumes. Other causes of apparent data loss:
- The project name or folder name changed. Compose prefixes volume names with the project name, so a new name creates a new, empty volume.
- An anonymous volume was used.
- `docker volume prune` or `docker system prune --volumes` deleted an "unused" volume.
- No volume was used, so data lived in the container's writable layer.

**Recovery:** `docker volume ls` (old volumes look like `oldproject_pgdata`); mount the old volume with `external: true` and its full name; restore from backup.

**Prevention:** avoid `down -v`; set `name:` at the top of the Compose file to pin the project name; back up volumes; use external volumes for critical data.

#### H2. "version is obsolete" warning

```
WARN[0000] /path/compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
```
**Cause:** The top-level `version: "3.8"` key belonged to old Compose file formats. The Compose Specification does not use it.

**Fix:** delete the `version:` line. The warning is harmless.

#### H3. YAML syntax errors

```
yaml: line 12: mapping values are not allowed in this context
yaml: line 8: did not find expected key
services.api.ports must be a list
service "api" refers to undefined volume pgdata: invalid compose project
```
**Cause:**
- Indentation errors (YAML uses spaces, never tabs).
- A missing space after a colon (`key:value`).
- Unquoted special values: `yes`/`no`/`on` become booleans, and port mappings like `80:80` can be misread as base-60 numbers by some parsers. Quote them: `"80:80"`.
- A named volume used by a service but not declared in top-level `volumes:`.

**Fix:** run `docker compose config`. It validates the file, prints the result and usually points at the line. Use an editor with YAML linting.

#### H4. Orphan containers warning

```
WARN[0000] Found orphan containers ([myproject-old-service-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
```
**Cause:** A service was removed or renamed in the Compose file, but its old container still exists.

**Fix:** `docker compose up -d --remove-orphans` removes containers of services no longer defined. Do not use it if another Compose file in the same project folder defines other services.

#### H5. Environment variable is blank / `.env` not applied

**Symptoms:** `WARN The "DB_PASSWORD" variable is not set. Defaulting to a blank string.`, or values from `.env` do not reach the container.

**Rules:**
- A `.env` file in the project directory supplies values for `${VARIABLES}` substituted in the Compose file. It does not automatically become environment variables inside containers. Use `env_file:` or `environment:` entries with `${VAR}` to pass values in.
- Shell environment variables override `.env` values.
- In `.env`, avoid spaces around `=`; be careful with quotes and `#` in values.

**Fix:** `docker compose config` shows resolved values; `${VAR:?error message}` fails fast on missing values; `docker compose exec api env` shows what the container received.

#### H6. Service builds alone but fails in Compose (or the reverse)

Usually a difference in context path, environment, network or build args. Compare `docker compose config` output with the manual `docker build` / `docker run` flags.

### Group I: Platform-specific and GPU

#### I1. GPU not visible in the container

```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]
```
**Cause:** The **NVIDIA Container Toolkit** or the host NVIDIA driver is missing or not configured on the Linux host. Docker Desktop on Windows supports GPUs through WSL 2 with a recent NVIDIA Windows driver.

**Fix:** install the NVIDIA driver and the NVIDIA Container Toolkit, then:
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```
- Use a CUDA base image tag that currently exists.
- If `nvidia-smi` works on the host but not in the container, check the toolkit configuration. A CUDA version error means the image's CUDA is newer than the host driver supports.
- Compose syntax and details: [Chapter 8](08-docker-for-ai-and-ml.md).

#### I2. Slow bind mounts and builds on Windows/macOS

**Cause:** File I/O across the Windows/macOS-to-VM boundary is slower than native Linux disk. Projects with `node_modules` or Python virtualenvs on a bind mount are slow.

**Fix:** keep code in the WSL 2 filesystem (Windows) and run Docker commands from there; keep dependency folders in a named volume (`- node_modules:/app/node_modules`); keep the build context small with `.dockerignore`; on macOS enable VirtioFS in Docker Desktop settings.

#### I3. Clock drift / "certificate has expired or is not yet valid"

**Cause:** After a laptop sleeps, the WSL 2 VM clock can lag, so TLS certificates look not yet valid and package signatures fail (`Release file ... is not valid yet`).

**Fix:** `wsl --shutdown` and restart Docker Desktop; or sync the clock (`sudo hwclock -s` in WSL); on servers run NTP (`chrony` or `systemd-timesyncd`).

---

## 4. Debugging builds

A failed build has no container to `exec` into, and BuildKit collapses output by default.

### 4.1 See all the output

```bash
docker build --progress=plain -t myapp .
```
- `--progress=plain` prints a full sequential log. The real error appears above the `ERROR: failed to solve` line.
- Environment variable equivalent: `BUILDKIT_PROGRESS=plain`. Accepted values: `auto`, `plain`, `tty`, `quiet`, `rawjson`.
- Compose: `docker compose build --progress=plain api`.

### 4.2 Remove the cache as a variable

```bash
docker build --no-cache --pull --progress=plain -t myapp .
```
- `--no-cache` ignores all cached layers; `--pull` fetches the newest base image. If the result changes, the cache was involved. Use it for diagnosis only; it is slow.

### 4.3 Stop at a stage: `--target`

```dockerfile
FROM python:3.12-slim AS base
...
FROM base AS builder
RUN pip install --no-cache-dir -r requirements.txt
FROM base AS runtime
COPY --from=builder /usr/local /usr/local
```

```bash
docker build --target builder -t myapp:builder .
docker run --rm -it myapp:builder sh
```
- `--target builder` builds only up to the stage named `builder`, so a shell opens in the exact state before the failing step.

### 4.4 Debugging a failing `RUN` step interactively

Comment out the failing `RUN` and everything after it, build, open a shell and run the command by hand:

```bash
docker build -t myapp:debug .          # failing RUN commented out
docker run --rm -it myapp:debug sh     # run the failing command manually
```
- Alternative: add `RUN ls -la /app; pwd; env` before the failing step to print files and variables at that point.

### 4.5 Check the Dockerfile before building

```bash
docker build --check .
```
- Runs **build checks** (a Dockerfile linter built into BuildKit) without building. It warns about undefined variables, mismatched `FROM ... AS` case, JSON-form `CMD` mistakes and similar issues. **hadolint** is a stricter alternative.

### 4.6 Inspect what was sent to the builder

List the context contents with a throwaway Dockerfile (`Dockerfile.ctx`):

```dockerfile
FROM busybox
COPY . /ctx
RUN find /ctx -maxdepth 2 | sort
```
```bash
docker build --progress=plain --no-cache -f Dockerfile.ctx .
```
- The output lists the files that survived `.dockerignore`. A `*` ignore with `!` exceptions is easy to get wrong.

### 4.7 Look at layers and sizes

```bash
docker history myapp:1.0
docker history --no-trunc --format "{{.Size}}\t{{.CreatedBy}}" myapp:1.0
```
- Lists each layer, its size and the instruction that made it, to find large layers (an `apt` cache, a copied dataset). The third-party **dive** tool (`wagoodman/dive`) shows layer file changes interactively.

### 4.8 Build secrets and private dependencies

Do not pass credentials through `ARG` or `ENV`; they stay in image history. Use BuildKit mounts:

```dockerfile
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```
```bash
docker build --secret id=pip_token,src=./pip_token.txt .
```
- The secret exists only during that `RUN` step, at `/run/secrets/pip_token`, and is not written to a layer.
- For SSH use `RUN --mount=type=ssh` with `docker build --ssh default .` ([Chapter 10](10-security.md)).

### 4.9 Multi-platform build problems

- `exec format error` during a cross-platform build means QEMU emulation is not set up. Register emulators with `docker run --privileged --rm tonistiigi/binfmt --install all`, or use `docker/setup-qemu-action` in CI.
- Python packages without arm64 wheels compile from source (slow or failing). Add build tools (`gcc`, `python3-dev`) or pin versions that have wheels.
- List builders with `docker buildx ls`; create one with `docker buildx create --name mybuilder --use`.

---

## 5. Performance tuning tips

### 5.1 Faster builds

1. **Order layers** by change frequency (dependencies before source).
2. **Use `.dockerignore`** to exclude `.git`, `node_modules`, `__pycache__`, `*.log`, `.venv`, datasets, model weights and `.env`. The whole context is uploaded to the builder on every build.
3. **Use BuildKit cache mounts** for package managers: `--mount=type=cache,target=/root/.cache/pip` (pip), `/var/cache/apt` with `sharing=locked` (apt), `/root/.npm` (npm), `/go/pkg/mod` (Go).
4. **Use multi-stage builds.** They keep compilers out of the final image, and BuildKit builds independent stages in parallel.
5. **Reuse base images and cache.** Use a registry or CI cache backend (`--cache-from` / `--cache-to`) so ephemeral CI runners do not start cold.
6. **Install only what is needed:** `--no-install-recommends` for apt, `--no-cache-dir` for pip in the final image, `npm ci --omit=dev`.
7. **ML images:** put heavy wheels (`torch`) in an early, rarely changing layer; consider a prebuilt base image with CUDA and PyTorch; keep model weights out of the image ([Chapter 8](08-docker-for-ai-and-ml.md)).

### 5.2 Smaller, faster-starting images

- Use slim or distroless bases (`python:3.12-slim`, `gcr.io/distroless/...`). Use `alpine` only when dependencies support musl libc; some Python wheels do not, which can make Alpine builds slower and larger.
- Combine cleanup with install in the same layer; remove package caches.
- Check with `docker images` and `docker history`; use `dive` to find waste.
- Smaller images pull faster, which improves autoscaling and cold-start time.

### 5.3 Faster runtime

| Area | Guidance |
|---|---|
| Bind mounts | Avoid them for hot paths on Windows/macOS; use volumes or the WSL 2 filesystem |
| VM resources | Give the Docker VM enough CPU and memory; undersized VMs cause swapping and OOM kills |
| Architecture | Use the native architecture; emulation is 5-20x slower |
| Limits | Set limits so one container cannot starve others; find the heavy one with `docker stats` |
| PID 1 | Use `init: true` / `--init` to reap zombie processes and forward signals |
| Logging | Rotate logs; avoid debug logging in hot paths; heavy stdout logging can slow an app when the log driver blocks |
| App tuning | Match worker counts to CPU limits (Gunicorn `workers = 2 * cores + 1` is a starting point for the cores the container has, not the host's count); size database connection pools per replica |
| Networking | Use user-defined networks (not `--link`). `"userland-proxy": false` in `daemon.json` uses iptables only; verify the setup after changing it. Use `network_mode: host` (Linux only) only when the loss of isolation is acceptable |
| Storage | Use `overlay2` (default on modern Linux) on ext4/xfs. Put heavy-write data in volumes; copy-on-write in the writable layer is slow for large files and databases |
| Startup | Avoid heavy work at import time; lazy-load models; use readiness probes so traffic waits |
| Triage | `docker stats --no-stream` (CPU/memory), `docker top NAME` (processes), `docker system df` (disk), `docker info \| grep -E "Storage Driver\|Cgroup\|Total Memory\|CPUs"` (on Docker Desktop, "Total Memory" is the VM's allotment and is often the bottleneck) |

---

## 6. The one-page cheat sheet

`NAME` = container name or ID, `IMG` = image, `VOL` = volume, `NET` = network.

### Images

| Command | What it does |
|---|---|
| `docker pull IMG:TAG` | Download an image from a registry |
| `docker images` / `docker image ls` | List local images |
| `docker build -t NAME:TAG .` | Build from the Dockerfile in the current folder (`.` = build context) |
| `docker build -f path/Dockerfile -t NAME:TAG ctx` | Use a specific Dockerfile and context folder |
| `docker build --no-cache --pull .` | Rebuild without cache; refresh the base image |
| `docker build --target STAGE .` | Build up to a named stage |
| `docker build --progress=plain .` | Show full build output |
| `docker tag SRC:TAG REG/NAME:TAG` | Add another name/tag to an image |
| `docker push REG/NAME:TAG` | Upload an image to a registry |
| `docker history IMG` | Show layers and sizes |
| `docker image inspect IMG` | Full JSON (platform, env, cmd, layers) |
| `docker rmi IMG` / `docker image rm IMG` | Delete a local image |
| `docker image prune` / `docker image prune -a` | Delete dangling images / all images not used by a container |
| `docker save -o f.tar IMG` / `docker load -i f.tar` | Export / import images as a tar file |
| `docker scout quickview IMG` | Vulnerability summary (Docker Scout; plan limits apply, check Docker's pricing) |

### Containers

| Command | What it does |
|---|---|
| `docker run IMG` | Create and start a container (foreground) |
| `docker run -d --name NAME IMG` | Run in the background (`-d` detached) with a name |
| `docker run -it --rm IMG sh` | Interactive shell, deleted on exit |
| `docker run -p HOST:CONTAINER IMG` | Publish a port (`-p 8080:80`); `-p 127.0.0.1:8080:80` for local only |
| `docker run -e KEY=value IMG` / `--env-file f` | Set environment variables |
| `docker run -v VOL:/path IMG` | Mount a named volume |
| `docker run -v "$PWD":/app IMG` | Bind-mount a host folder (development) |
| `docker run --restart unless-stopped IMG` | Restart policy |
| `docker run --memory 512m --cpus 1 IMG` | Resource limits |
| `docker run --platform linux/amd64 IMG` | Force architecture |
| `docker run --gpus all IMG` | Give the container NVIDIA GPUs (needs NVIDIA Container Toolkit) |
| `docker run --network NET IMG` | Attach to a network |
| `docker run --entrypoint sh -it IMG` | Replace the entrypoint (debugging) |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker stop NAME` | Graceful stop (SIGTERM, then SIGKILL after 10 s) |
| `docker kill NAME` | Immediate stop (SIGKILL) |
| `docker start NAME` / `docker restart NAME` | Start a stopped container / restart |
| `docker rm NAME` / `docker rm -f NAME` | Remove a stopped / force-remove a running container |
| `docker logs -f --tail 100 NAME` | Follow the last 100 log lines |
| `docker exec -it NAME sh` | Shell in a running container |
| `docker cp NAME:/path ./local` | Copy files between container and host |
| `docker inspect NAME` | Full JSON details |
| `docker stats` | Live CPU/memory/network per container |
| `docker top NAME` | Processes in the container |
| `docker diff NAME` | Files changed in the container |
| `docker port NAME` | Published ports |
| `docker events` | Live engine event stream |
| `docker container prune` | Remove all stopped containers |
| `docker update --memory 1g NAME` | Change limits of a running container (some options) |
| `docker rename OLD NEW` | Rename a container |

### Volumes

| Command | What it does |
|---|---|
| `docker volume create VOL` | Create a named volume |
| `docker volume ls` | List volumes |
| `docker volume inspect VOL` | Details, including mountpoint |
| `docker volume rm VOL` | Delete a volume (data is lost) |
| `docker volume prune` | Delete volumes no container uses (dangerous on servers) |
| `docker run --rm -v VOL:/d alpine tar czf - -C /d . > backup.tgz` | Back up a volume to a tarball on the host |

### Networks

| Command | What it does |
|---|---|
| `docker network ls` | List networks |
| `docker network create NET` | Create a user-defined bridge network (DNS by container name) |
| `docker network inspect NET` | Subnet, attached containers and IPs |
| `docker network connect NET NAME` / `disconnect` | Attach / detach a running container |
| `docker network prune` | Remove unused networks |
| `docker run --rm -it --network container:NAME nicolaka/netshoot` | Network debugging toolkit in a container's namespace |

### Compose

| Command | What it does |
|---|---|
| `docker compose up -d` | Create and start all services in the background |
| `docker compose up -d --build` | Rebuild images first |
| `docker compose up -d --wait` | Wait until services are running/healthy |
| `docker compose up -d --force-recreate SERVICE` | Recreate containers even if unchanged |
| `docker compose up -d --remove-orphans` | Remove containers of services no longer defined |
| `docker compose down` | Stop and remove containers and networks (keeps named volumes) |
| `docker compose down -v` | Also delete named volumes (data loss) |
| `docker compose ps` | Service status |
| `docker compose logs -f SERVICE` | Follow logs |
| `docker compose exec SERVICE sh` | Shell in a running service |
| `docker compose run --rm SERVICE CMD` | One-off container for a service |
| `docker compose build [--no-cache] SERVICE` | Build images |
| `docker compose pull` | Pull newer images |
| `docker compose restart SERVICE` | Restart services |
| `docker compose stop` / `start` | Stop / start without removing |
| `docker compose config` | Show the merged config (best for debugging) |
| `docker compose --profile NAME up -d` | Enable optional profile services |
| `docker compose -f a.yaml -f b.yaml up` | Merge several Compose files |
| `docker compose watch` | Sync/rebuild on file changes (development) |
| `docker compose cp`, `top`, `events`, `images` | Copy files, list processes, stream events, list images |

### Buildx

| Command | What it does |
|---|---|
| `docker buildx ls` | List builders |
| `docker buildx create --name b --use` | Create and select a builder |
| `docker buildx build --platform linux/amd64,linux/arm64 -t REG/IMG:TAG --push .` | Multi-arch build and push |
| `docker buildx build --load -t IMG .` | Build a single-platform image into local Docker |
| `docker buildx build --cache-from type=registry,ref=REG/IMG:cache --cache-to type=registry,ref=REG/IMG:cache,mode=max .` | Remote build cache |
| `docker buildx imagetools inspect REG/IMG:TAG` | Inspect a registry image (platforms, digests) without pulling |
| `docker buildx imagetools create -t REG/IMG:new REG/IMG:old` | Re-tag in the registry without re-uploading |
| `docker buildx du` / `docker builder prune` | Inspect / clean build cache |
| `docker build --check .` | Lint the Dockerfile with BuildKit checks |

### System, cleanup and info

| Command | What it does |
|---|---|
| `docker version` / `docker info` | Versions; daemon configuration and resources |
| `docker system df` | Disk usage by images, containers, volumes, cache |
| `docker system prune` | Remove stopped containers, unused networks, dangling images, build cache |
| `docker system prune -a` | Also remove all unused images (never add `--volumes` without checking) |
| `docker builder prune` | Remove build cache |
| `docker context ls` / `docker context use NAME` | List / switch the engine the CLI talks to |
| `docker login [REG]` / `docker logout` | Authenticate to a registry |
| `docker debug NAME` | Enhanced debug shell (Docker Desktop feature) |

### Formatting and one-liners

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"     # compact container table
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' NAME   # container IP
docker rm -f $(docker ps -aq)                    # removes ALL containers (not for servers)
docker stop $(docker ps -q)                      # stop all running containers
docker images --filter dangling=true             # dangling images
docker ps -a --filter status=exited              # exited containers
docker run --rm -v "$PWD":/work -w /work IMG CMD # run a tool against the current folder
```

---

## 7. Common mistakes about troubleshooting itself

| Mistake | Problem | Fix |
|---|---|---|
| Changing many things at once | Cannot tell which change fixed it | Change one thing per test |
| Reading only the first error line | The first line is often generic | Read the last lines and the lines above `ERROR: failed to solve` |
| Running `docker system prune -a --volumes` first | Can delete needed data | Diagnose before cleaning up |
| Editing a running container | Changes vanish when it is recreated | Fix the Dockerfile or Compose file |
| Using `latest` | Cannot tell what changed | Pin versions |
| Blaming Docker (or the app) without testing | Wrong layer investigated | Run the app outside Docker once |
| Disabling TLS verification permanently | Hides real attacks | Install the proper CA (F1) |
| Skipping `docker compose config` | Misses blank variables and indentation errors | Run it first for any Compose problem |
| Ignoring the host | Full disk, exhausted memory, wrong time, firewall changes | Check host resources and clock |
| Not recording fixes | Same problem costs time again | Keep a runbook (also useful for interviews, see [Chapter 13](13-interview-questions-roadmap-glossary.md)) |

---

Previous: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md) | Next: [Chapter 13: Interview Questions, Roadmap and Glossary](13-interview-questions-roadmap-glossary.md)
