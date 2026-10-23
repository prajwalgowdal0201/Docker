# Chapter 12: Troubleshooting and Cheat Sheet

> **What you will learn:** a repeatable method for debugging anything in Docker, the 40+ most common errors with their exact messages, causes and fixes, the debugging toolbox (logs, inspect, exec, events, netshoot, `docker debug`), how to debug broken builds, performance tuning tips, and a one-page command cheat sheet you can print.

Everyone who uses Docker hits the same twenty or thirty errors. The difference between a beginner and an expert is not that the expert never sees them. It is that the expert has a *method*, recognises the error text instantly, and knows where to look. This chapter gives you both the method and the catalogue.

Throughout this chapter, commands were checked against Docker Engine 29.x, Compose v5.x and Buildx 0.38 (the versions on the machine this guide was written on). Where something is OS-specific, it says so.

---

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

When something breaks, resist the urge to try random fixes. Walk down this ladder. Each rung narrows the problem to one layer.

```
 Layer 1: Is the DOCKER ENGINE (daemon) up?            -> docker version / docker info
 Layer 2: Is the IMAGE what I think it is?             -> docker images / docker history / pull again
 Layer 3: Did the CONTAINER start, and why did it stop? -> docker ps -a / docker logs / exit code
 Layer 4: Is the APP inside healthy and configured?     -> docker exec / env / config files
 Layer 5: Can it reach what it needs? (NETWORK / DNS)   -> docker network inspect / netshoot
 Layer 6: Does it have its DATA / permissions?          -> docker volume inspect / ls -l / id
 Layer 7: Does it have enough RESOURCES?                -> docker stats / OOMKilled / df -h
 Layer 8: Is it the HOST / OS / PROXY?                  -> firewall, WSL2, corporate TLS proxy
```

### The seven-step method

1. **Read the error message completely, then read it again.** The last 3 lines of output are usually the actual cause; the first lines are context. Copy the exact text; you will need it for searching.
2. **Reproduce it with the smallest possible command.** If a 12-service Compose stack fails, can you reproduce the failure with just one service? `docker compose up api` or a plain `docker run`.
3. **Ask "what changed?"** New image tag? New base image? Different machine? Updated Docker Desktop? Edited `.env`? Pulled new code? Most bugs follow a change.
4. **Gather facts before theories.** Run the standard five (below).
5. **Form one hypothesis, change one thing, retest.** Changing three things at once teaches you nothing.
6. **Check the layers from the bottom up** (the ladder above): engine, image, container, app, network, data, resources, host.
7. **Write down the fix.** Add it to your own notes or runbook. Next time it takes ten seconds.

### The "standard five" fact-gathering commands

Run these first for almost any container problem (replace `NAME` with the container name or ID):

```bash
docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
docker logs --tail 100 --timestamps NAME
docker inspect NAME --format 'exit={{.State.ExitCode}} oom={{.State.OOMKilled}} err={{.State.Error}} health={{if .State.Health}}{{.State.Health.Status}}{{end}}'
docker stats --no-stream
docker system df
```

What each does:

- `docker ps -a --format "table ..."`: lists **all** containers (`-a` includes stopped ones) in a compact table. The `--format` flag uses Go template syntax (`{{.Field}}`) to choose columns; `\t` is a tab so columns line up. Example output:
  ```
  NAMES      IMAGE            STATUS                      PORTS
  web        nginx:1.27       Up 3 minutes                0.0.0.0:8080->80/tcp
  api        chat-api:1.0     Exited (1) 2 minutes ago
  ```
- `docker logs --tail 100 --timestamps NAME`: prints the last 100 log lines with timestamps. `--tail 100` avoids dumping a huge log.
- `docker inspect NAME --format '...'`: `inspect` prints a container's full JSON details; `--format` extracts only the fields you want. Here: the exit code, whether the kernel OOM-killed it, any start error, and its health status.
- `docker stats --no-stream`: prints a one-time snapshot of CPU, memory, network and disk I/O for each running container instead of a live-updating display.
- `docker system df`: shows disk used by images, containers, volumes and build cache.

### Reading an exit code

When a container stops, its main process leaves an **exit code** (a number reported to the parent). Knowing the common ones solves half of all problems:

| Exit code | Meaning | Typical cause |
|---|---|---|
| `0` | Success, finished normally | The command was short-lived (a script that ended), or the main process daemonised and returned |
| `1` | Generic application error | Your app threw an exception; read the logs |
| `2` | Misuse of shell builtin or app-specific usage error | Bad arguments |
| `125` | `docker run` itself failed | Bad Docker flag, daemon error |
| `126` | Command found but cannot execute | File not executable (`chmod +x`), wrong format |
| `127` | Command not found | Typo in `CMD`, binary missing from image, wrong PATH |
| `130` | Terminated by Ctrl+C (SIGINT = 2, so 128 + 2) | You interrupted it |
| `137` | Killed by SIGKILL (128 + 9) | **Out of memory (OOM kill)**, or `docker kill`, or a stop timeout expired |
| `139` | Segmentation fault (128 + 11) | Native-code crash; wrong architecture library; bad CUDA install |
| `143` | Terminated by SIGTERM (128 + 15) | Normal `docker stop`; app handled the signal and exited |

Rule: **exit codes above 128 mean "killed by a signal," and the signal number is the code minus 128.**

---

## 2. The debugging toolbox

### 2.1 `docker logs`: what did the app say?

```bash
docker logs NAME                  # everything
docker logs -f NAME               # follow (like tail -f); Ctrl+C to stop following
docker logs --tail 50 NAME        # last 50 lines
docker logs --since 10m NAME      # only the last 10 minutes
docker logs --since 2026-10-23T09:00:00 --until 2026-10-23T09:30:00 NAME
docker logs -t NAME               # prefix timestamps
docker compose logs -f --tail 100 api db   # Compose: several services, interleaved
```

- `-f` / `--follow` streams new lines as they appear.
- `--since` / `--until` accept durations (`10m`, `2h`) or timestamps.
- Logs are captured from the container's stdout/stderr. If your app writes to a **file** inside the container, `docker logs` shows nothing; either reconfigure the app to log to stdout or read the file with `docker exec NAME cat /path/to/log`.
- Python prints can appear late because of output buffering. Set `PYTHONUNBUFFERED=1` (or run `python -u`).
- If the container has been *removed*, its logs are gone (unless a log driver shipped them elsewhere).

### 2.2 `docker inspect`: the full truth

`docker inspect` returns everything Docker knows about an object (container, image, volume, network) as JSON.

```bash
docker inspect NAME                                         # the whole JSON
docker inspect NAME --format '{{.State.Status}}'            # running, exited, restarting...
docker inspect NAME --format '{{json .Config.Env}}'         # environment variables
docker inspect NAME --format '{{json .NetworkSettings.Networks}}'   # networks and IPs
docker inspect NAME --format '{{range .Mounts}}{{.Source}} -> {{.Destination}} ({{.Type}}){{"\n"}}{{end}}'
docker inspect NAME --format '{{.HostConfig.RestartPolicy.Name}} restarts={{.RestartCount}}'
docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'
```

- `{{json X}}` prints a value as JSON (readable for lists and maps).
- `{{range .Mounts}}...{{end}}` loops over every mount and prints source and destination, which is great for "why is my file not there?"
- On Windows PowerShell, use double quotes around the format and escape inner double quotes, or use Git Bash where single quotes work as shown.

### 2.3 `docker exec`: get inside a running container

```bash
docker exec -it NAME sh                 # interactive shell (use bash if the image has it)
docker exec NAME env                    # run one command, print result, exit
docker exec NAME cat /etc/resolv.conf   # check DNS configuration
docker exec -u 0 -it NAME sh            # become root (user ID 0) for the session
docker exec -w /app NAME ls -la         # run in a specific working directory
docker compose exec api sh              # Compose version (uses the service name)
```

- `-i` keeps standard input open and `-t` allocates a terminal; together (`-it`) they give you an interactive session.
- `exec` only works on a **running** container. If it keeps crashing, see `--entrypoint` below.
- Minimal images (distroless, `scratch`) have no shell, so `exec ... sh` fails with `executable file not found in $PATH`. Use `docker debug` or a sidecar debug container (below).

### 2.4 Running a container with a different entrypoint

If a container exits immediately, you cannot `exec` into it. Start a **new** container from the same image but replace the startup command with a shell:

```bash
docker run --rm -it --entrypoint sh chat-api:1.0
```

- `--entrypoint sh` overrides the image's `ENTRYPOINT` (the thing that normally starts your app). With `-it` you land in a shell in the same environment, where you can run the app's start command by hand and see the error live.
- `--rm` deletes the container when you exit.

For Compose services: `docker compose run --rm --entrypoint sh api`. (`run` starts a one-off container for the service with all its volumes, networks and environment.)

### 2.5 `docker events`: what is the engine doing?

```bash
docker events --since 30m --filter type=container
```

- Streams lifecycle events: `create`, `start`, `die`, `oom`, `kill`, `health_status`, `restart`. If a container is flapping, `events` shows the exact order. An `oom` event proves a memory kill. Example:
  ```
  2026-10-23T10:15:02.123456789Z container oom 8f3c... (image=chat-api:1.0, name=api)
  2026-10-23T10:15:02.234567890Z container die  8f3c... (exitCode=137, image=chat-api:1.0, name=api)
  ```
- `--filter type=container` limits to containers (others: `image`, `volume`, `network`).

### 2.6 Other inspection commands

| Command | What it tells you |
|---|---|
| `docker top NAME` | Processes running inside the container (as seen from the host) |
| `docker diff NAME` | Files added (`A`), changed (`C`) or deleted (`D`) in the container's writable layer. Great for finding unexpected writes. |
| `docker port NAME` | The published port mappings |
| `docker cp NAME:/path/in/container ./local` | Copy files out of (or into) a container, even a stopped one. Perfect for retrieving a crash log or config. |
| `docker history IMAGE` | The layers of an image and the commands that created them, with sizes |
| `docker network inspect NET` | Which containers are on a network, their IPs, subnet |
| `docker volume inspect VOL` | Where a volume is and who uses it |
| `docker info` | Daemon details: storage driver, cgroup version, logging driver, runtimes, proxies, root dir |
| `docker version` | Client and server versions (also confirms the daemon is reachable) |

### 2.7 netshoot: a network debugging Swiss-army knife

Most production images are tiny and contain no `curl`, `ping`, `dig` or `tcpdump`. **netshoot** (`nicolaka/netshoot`) is a community image packed with network tools. Attach it to the *network namespace* of the problem container so it sees exactly what that container sees:

```bash
docker run --rm -it --network container:api nicolaka/netshoot
```

- `--network container:api` means "share the network stack of container `api`": same IP address, same DNS configuration, same ports. Inside, you can run:
  - `curl -v http://db:5432` to test reachability,
  - `dig db` or `nslookup db` to test DNS,
  - `ss -tulpn` to list listening sockets,
  - `tcpdump -i eth0 port 8000` to capture packets,
  - `traceroute`, `mtr`, `iperf3` and more.
- You can also join a Compose network: `docker run --rm -it --network myproject_default nicolaka/netshoot`, then `nslookup api` to test name resolution as any container on that network would.

### 2.8 `docker debug`

`docker debug` gives you an **enhanced shell with extra tools (vim, curl, htop and more) inside any container or image, even ones with no shell** (distroless, `scratch`, or stopped containers), without modifying the original image.

```bash
docker debug api
docker debug chat-api:1.0
```

- The first form attaches to a container named `api`; the second opens a debug shell for an image.
- It ships as a **Docker Desktop** feature. It reached general availability in Docker Desktop 4.33 and, according to the Desktop release notes, became free for all users in 4.49. On this guide's author's machine `docker debug --help` works. It is **not part of plain Docker Engine on a Linux server** (I found no evidence that it has shipped for Engine; check Docker's docs). On servers use the open-source alternatives: a `nicolaka/netshoot` sidecar as above, the `cdebug` tool, or Kubernetes' `kubectl debug` with an ephemeral container.

### 2.9 Debugging without a shell: share namespaces

```bash
docker run --rm -it --pid container:api --network container:api --cap-add SYS_PTRACE nicolaka/netshoot
```

- `--pid container:api` shares the process namespace so you can see (`ps aux`) and trace the other container's processes. `--cap-add SYS_PTRACE` grants the permission to attach tracers such as `strace`. Only use on non-production or with proper authorisation; it deliberately weakens isolation.

### 2.10 "strace-less" approaches

You often cannot install `strace` in a locked-down image. Alternatives:

1. **Look at `/proc` from the host.** Find the container's PID (`docker inspect -f '{{.State.Pid}}' NAME`), then on the Linux host: `ls -l /proc/PID/fd` (open files), `cat /proc/PID/status` (memory, threads), `cat /proc/PID/limits`. (On Docker Desktop the "host" is a VM; use `docker run --rm -it --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sh` to enter it; this is powerful, use carefully.)
2. **Run a debug sidecar** sharing the PID namespace as shown above.
3. **Turn up the app's own debug logging** (`LOG_LEVEL=debug`).
4. **Copy files out** with `docker cp` and inspect locally.
5. **Make a debug image variant**: `FROM your-image` plus `RUN apk add strace curl`, never shipped to production.
6. **`docker commit`** a *broken* container to a new image so you can poke at its filesystem state later: `docker commit NAME debug-snapshot:1` then `docker run --rm -it --entrypoint sh debug-snapshot:1`. (For debugging only.)

### 2.11 Compose-specific helpers

| Command | Purpose |
|---|---|
| `docker compose config` | Print the final, merged, variable-substituted configuration. Catches indentation errors, missing variables, wrong file merges. |
| `docker compose ps -a` | Status of every service container (including exited) |
| `docker compose logs -f SERVICE` | Follow logs |
| `docker compose up --build --force-recreate SERVICE` | Rebuild and recreate one service |
| `docker compose events` | Event stream for the project |
| `docker compose run --rm --entrypoint sh SERVICE` | One-off debug shell with the service's config |
| `docker compose top` | Processes per service |

---

## 3. The error catalogue

Each entry has the **exact message** (or its core), the **cause**, the **fix** and a **prevention tip**. They are grouped by area. Use your browser's search (Ctrl+F) with a distinctive phrase of the error you see.

### Group A: Docker Engine and permissions

#### A1. Cannot connect to the Docker daemon

**Linux message:**
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```
**Windows/macOS (Docker Desktop) message (example):**
```
error during connect: Get "http://%2F%2F.%2Fpipe%2FdockerDesktopLinuxEngine/v1.51/containers/json": open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.
```
**Cause:** The Docker **daemon** (the background service that does the real work) is not running, or the CLI is pointing at the wrong place. The `docker` command you type is only a *client*; without the daemon it has nobody to talk to.

**Fix:**
- Linux: `sudo systemctl start docker` and `sudo systemctl enable docker` (also start at boot). Check with `systemctl status docker` and `journalctl -u docker --since "10 min ago"` for why it failed (often invalid `/etc/docker/daemon.json`).
- Windows/macOS: start **Docker Desktop** and wait until the whale icon says "Engine running."
- Check which engine your CLI targets: `docker context ls`. The asterisk marks the active context; on Windows you normally want `desktop-linux`. Switch with `docker context use desktop-linux`.
- Check that the `DOCKER_HOST` environment variable is not set to something stale: `echo $DOCKER_HOST` (empty is normal).

**Prevention:** enable Docker at boot; validate `daemon.json` before restarting (`dockerd --validate --config-file=/etc/docker/daemon.json`).

#### A2. Permission denied on the Docker socket

**Message:**
```
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.51/containers/json": dial unix /var/run/docker.sock: connect: permission denied
```
**Cause:** On Linux, `/var/run/docker.sock` belongs to root and the `docker` group. Your user is neither.

**Fix:**
```bash
sudo usermod -aG docker $USER     # add yourself to the docker group
newgrp docker                     # apply the group in the current shell (or log out and in)
docker ps                         # test without sudo
```
- `usermod -aG docker $USER` appends (`-a`) your user to the supplementary group (`-G`) named `docker`.
- **Security warning:** anyone in the `docker` group can become root on the host (a container can mount `/`). Only add trusted users. Do not "fix" this with `chmod 666 /var/run/docker.sock`; that gives every user and every compromised program root-equivalent power. For stricter setups use **rootless Docker** ([Chapter 10](10-security.md)).

#### A3. "docker: command not found" / "docker-compose: command not found"

**Cause:** Docker is not installed or not on PATH; or you are using the old standalone `docker-compose` (with a hyphen) which is no longer installed by default. Current Compose is a **plugin** invoked as `docker compose` (with a space).

**Fix:** install Docker Engine plus the Compose plugin ([Chapter 2](02-installation-and-operating-systems.md)); use `docker compose ...`. If an old script uses `docker-compose`, change it or create a small wrapper. Test with `docker compose version` (this guide's machine prints `Docker Compose version v5.6.1`).

#### A4. Docker Desktop will not start / stuck on "Starting"

**Typical symptoms:** whale icon spins forever, error dialog "Docker Desktop - WSL distro terminated abruptly" or "Docker Desktop stopped," or the engine never reaches "running."

**Common causes and fixes (try in order):**
1. **WSL 2 is outdated or unhealthy** (Windows). In PowerShell: `wsl --update`, then `wsl --shutdown`, then start Docker Desktop again. Check `wsl -l -v`: the `docker-desktop` distro should be `Running` and version `2`.
2. **Virtualization is disabled** in BIOS/UEFI (Intel VT-x / AMD-V / SVM). Check Task Manager > Performance > CPU: "Virtualization: Enabled." Also ensure the Windows features **Virtual Machine Platform** and **Windows Subsystem for Linux** are enabled.
3. **Another hypervisor or antivirus** interfering. Temporarily test with antivirus real-time scanning off, or add exclusions per Docker's docs.
4. **Corrupt state:** use Docker Desktop's Troubleshoot menu (the bug icon): **Restart**, then **Clean / Purge data** (this deletes your local images, containers and volumes, so back up first) or **Reset to factory defaults** as a last resort.
5. **Disk full** on the drive holding the WSL virtual disk (`docker_data.vhdx`).
6. **Stale `~/.docker/config.json`** referencing a missing credential helper (error mentions `docker-credential-desktop` not found): fix PATH or remove the `credsStore` line.
7. Collect diagnostics: Troubleshoot menu > **Get support / Gather diagnostics**.

Use the official troubleshooting page (<https://docs.docker.com/desktop/troubleshoot-and-support/troubleshoot/>) for current guidance.

#### A5. "Error response from daemon: driver failed programming external connectivity ..." (iptables)

**Message (core):**
```
Error response from daemon: driver failed programming external connectivity on endpoint web (...): (iptables failed: iptables --wait -t nat -A DOCKER ...: iptables: No chain/target/match by that name.
```
**Cause:** The Linux firewall rules Docker manages were flushed or disturbed (for example by `ufw reload`, `firewalld` restarts, or another tool) so Docker's chains vanished.

**Fix:** `sudo systemctl restart docker` recreates its iptables chains. Prevent by not running tools that flush all rules; use Docker's `DOCKER-USER` chain for custom rules. Note that on newer distributions using `nftables`, make sure Docker and the firewall use compatible backends (check `docker info` and Docker's firewall documentation for your version).

### Group B: Ports, names and conflicts

#### B1. Port is already allocated

**Message:**
```
docker: Error response from daemon: driver failed programming external connectivity on endpoint web (a1b2...): Bind for 0.0.0.0:8080 failed: port is already allocated.
```
Windows variant: `Ports are not available: exposing port TCP 0.0.0.0:8080 -> 0.0.0.0:0: listen tcp 0.0.0.0:8080: bind: Only one usage of each socket address (protocol/network address/port) is normally permitted.`

**Cause:** Something else (another container, or a program on the host) already listens on that host port. Two things cannot bind the same port on the same address.

**Fix:**
```bash
docker ps --filter "publish=8080"        # which container publishes 8080?
# Linux/macOS: who owns it on the host?
sudo lsof -i :8080        # or: sudo ss -ltnp | grep 8080
# Windows PowerShell:
netstat -ano | findstr :8080
Get-Process -Id <PID>
```
- `--filter "publish=8080"` finds containers publishing that port.
- `lsof -i :8080` / `ss -ltnp` lists the owning process on Linux/macOS; `netstat -ano` shows the process ID (PID) on Windows.
- Then either stop the other thing, or **change the host side** of the mapping: `-p 8081:80` (host port 8081 forwards to container port 80). Left of the colon = host, right = container.
- Windows specific: Hyper-V / WSL can reserve ranges of ports ("excluded port ranges"); see them with `netsh interface ipv4 show excludedportrange protocol=tcp`. Pick a port outside those ranges.

**Prevention:** use `${WEB_PORT:-8080}:80` in Compose so the port is configurable; do not publish ports you do not need (containers on the same network talk to each other without publishing).

#### B2. Container name is already in use

**Message:**
```
docker: Error response from daemon: Conflict. The container name "/api" is already in use by container "8f3c...". You have to remove (or rename) that container to be able to reuse that name.
```
**Cause:** A container (maybe stopped) with that `--name` still exists. Container names must be unique.

**Fix:** `docker rm api` (or `docker rm -f api` if running), or choose another name, or use `--rm` so one-off containers vanish when they exit. In Compose, avoid hard-coding `container_name:`; Compose generates unique names per project.

#### B3. "Pool overlaps with other one on this address space"

**Message:**
```
Error response from daemon: Pool overlaps with other one on this address space
```
or `failed to create network ...: Error response from daemon: invalid pool request: Pool overlaps with other one on this address space`

**Cause:** You (or Compose) tried to create a network with a subnet that overlaps an existing Docker network, or the host's own network/VPN range. Related symptom: containers cannot reach corporate services because Docker's default `172.17.0.0/16` or `192.168.x.x` ranges clash with the VPN.

**Fix:** list networks and subnets (`docker network ls`, `docker network inspect NET --format '{{json .IPAM.Config}}'`), remove unused ones (`docker network prune` only removes networks no container uses), or pick a different subnet. To change the defaults for all future networks, set `default-address-pools` in `daemon.json` ([Chapter 11](11-production-and-enterprise.md#16-docker-daemon-production-configuration-daemonjson)).

### Group C: Disk and memory

#### C1. No space left on device

**Message:**
```
write /var/lib/docker/...: no space left on device
```
or during build: `ERROR: failed to solve: ... no space left on device`

**Cause:** Docker's storage (images, containers, volumes, build cache, **unrotated container logs**) filled the disk. On Docker Desktop it may be the virtual disk's size cap, not your physical disk.

**Fix:**
```bash
docker system df                   # where is the space?
docker builder prune -f            # remove unused build cache (safe)
docker image prune -a              # remove all images not used by any container (re-pullable)
docker container prune             # remove stopped containers
# Find giant log files (Linux host):
sudo du -ah /var/lib/docker/containers | sort -rh | head
```
- Read [Chapter 11, Section 17](11-production-and-enterprise.md#17-disk-management-and-cleanup-policies) before pruning on a server. **Do not run `docker volume prune` or `docker system prune --volumes` unless you are certain no needed data is in unused volumes**; a stopped database's volume looks "unused."
- A single container with a giant `*-json.log`: configure rotation (`max-size`, `max-file`) and recreate the container; as an emergency, truncate the file: `sudo truncate -s 0 /var/lib/docker/containers/<id>/<id>-json.log`.
- Docker Desktop (WSL 2): the virtual disk grows but does not shrink automatically. After pruning, reclaim space with Docker Desktop's disk-image settings or, on WSL, compact the VHDX (`wsl --shutdown`, then `Optimize-VHD` in an admin PowerShell where available).
- Check the host's own disks and inodes: `df -h` and `df -i` (inodes can run out with millions of tiny files even if bytes remain).

**Prevention:** log rotation, a weekly prune of images/build cache, monitoring at 80% disk.

#### C2. OOMKilled / exit code 137

**Signs:** container exits with `Exited (137)`; `docker inspect` shows `OOMKilled=true`; the process just vanishes with no stack trace; `docker events` shows an `oom` event.

**Cause:** the container exceeded its memory limit (or the host ran out of RAM) and the kernel's **OOM killer** terminated it with SIGKILL (128 + 9 = 137). Note: exit 137 can also come from a manual `docker kill` or from `docker stop` taking longer than its timeout, so confirm with the `OOMKilled` flag.

**Fix:**
```bash
docker inspect NAME --format 'OOM={{.State.OOMKilled}} limit={{.HostConfig.Memory}}'
docker stats --no-stream            # observe memory usage vs limit
```
- If `OOM=true`: raise the limit (`--memory=2g`, or Compose `deploy.resources.limits.memory: 2G`) **or** reduce the app's memory use (smaller batch sizes, fewer worker processes, a smaller or quantised model, streaming instead of loading whole files).
- Make the runtime respect the limit (JVM `-XX:MaxRAMPercentage=75`, Node `--max-old-space-size`, Gunicorn `--workers` count, `OMP_NUM_THREADS` for numeric libraries).
- On Docker Desktop (WSL 2) the **whole VM** has a memory cap, so the container may die even with no container-level limit. See C3 below for `.wslconfig`.
- Loading large AI models: a 7B-parameter model in 16-bit needs roughly 14 GB for weights alone; if your container limit or VM is smaller, it will be OOM-killed during load. Use a quantised model or give the VM more memory ([Chapter 8](08-docker-for-ai-and-ml.md)).

#### C3. WSL 2 `vmmem` / `VmmemWSL` eating memory (Windows)

**Symptom:** Task Manager shows a process named `vmmem` or `Vmmem WSL` using many gigabytes (often half your RAM or more) even when Docker is idle; the laptop becomes slow.

**Cause:** Docker Desktop on Windows runs inside a WSL 2 lightweight VM. By default WSL 2 may use up to a large fraction of host memory, and Linux uses spare memory as **file cache** which it is slow to give back.

**Fix:** create or edit `%UserProfile%\.wslconfig` (that is, `C:\Users\<you>\.wslconfig`):

```ini
[wsl2]
memory=6GB
processors=4
swap=2GB

[experimental]
autoMemoryReclaim=gradual
```
- `memory=6GB` hard-caps the VM's RAM; `processors=4` limits CPU cores; `swap=2GB` sets swap size.
- `autoMemoryReclaim=gradual` asks WSL to hand idle cache back to Windows gradually (an experimental setting available in recent WSL versions; check `wsl --version` and Microsoft's docs).
- Then apply: in PowerShell run `wsl --shutdown`, then restart Docker Desktop. Choose limits generously enough for your heaviest container (see C2) but low enough for Windows to breathe.
- Docker Desktop's own **Settings > Resources** page can also set limits on some configurations (WSL 2 mode defers to `.wslconfig`).

### Group D: Architecture, images and registries

#### D1. exec format error (architecture mismatch)

**Message:**
```
exec /usr/local/bin/python: exec format error
```
or `standard_init_linux.go: exec user process caused: exec format error`, and often the earlier warning:
```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```
**Cause:** The image was built for a different **CPU architecture** (Intel/AMD `amd64` versus ARM `arm64`) than the machine running it. Typical case: you build on an Apple Silicon Mac (arm64) and run on an amd64 cloud server, or the reverse. Another cause: a shell script without a shebang line, or Windows line endings (see E3).

**Fix:**
```bash
docker image inspect IMAGE --format '{{.Os}}/{{.Architecture}}'      # what is this image?
docker run --platform linux/amd64 IMAGE                              # force a platform (uses emulation if needed)
docker buildx build --platform linux/amd64,linux/arm64 -t REG/IMG:TAG --push .   # build for both
```
- `--platform linux/amd64` selects (or emulates) that architecture. Emulation (QEMU, or Rosetta on Apple Silicon) works but is **much slower**; use native images in production.
- A **multi-arch image** contains a variant per architecture under one tag, and Docker picks the right one automatically. Build with `buildx` as shown (Chapter 11 has a CI example).

#### D2. pull access denied / repository does not exist

**Message:**
```
Error response from daemon: pull access denied for myapp, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```
**Cause (any of):** a typo in the image name; the image is private and you are not logged in; the image only exists locally (you never pushed it) and you are trying to pull it from a server; wrong registry hostname.

**Fix:** check the spelling and registry prefix (`ghcr.io/owner/name`); `docker login REGISTRY`; if the image is local-only, do not use `pull`, and in Compose use `build:` or `pull_policy: never`. For private GHCR images log in with a token that has `read:packages`.

#### D3. manifest unknown / not found

**Message:**
```
Error response from daemon: manifest for myrepo/app:1.4.3 not found: manifest unknown: manifest unknown
```
or `no matching manifest for linux/arm64/v8 in the manifest list entries`

**Cause:** that **tag does not exist** (typo, never pushed, deleted by a retention policy), or the image has no variant for your platform (second message).

**Fix:** list real tags in the registry's web UI or with `docker buildx imagetools inspect REPO:TAG`; fix the tag; for the platform case use `--platform linux/amd64` or ask the publisher for a multi-arch build.

#### D4. toomanyrequests (Docker Hub pull rate limit)

**Message:**
```
toomanyrequests: You have reached your unauthenticated pull rate limit. https://www.docker.com/increase-rate-limit
```
(older wording: "You have reached your pull rate limit. You may increase the limit by authenticating and upgrading.")

**Cause:** Docker Hub caps pulls per six hours: **100 for anonymous users** (counted per IP, so a shared office or CI IP exhausts it quickly) and 200 for free authenticated accounts, at the time of writing (see [Chapter 11, Section 5.3](11-production-and-enterprise.md#53-docker-hub-rate-limits) for caveats).

**Fix:** `docker login` (use a **personal access token** rather than your password); add credentials to CI; run a pull-through cache or mirror base images to your own registry; wait for the window to reset. Docker documents a special test repository (`ratelimitpreview/test`) whose response headers (`ratelimit-limit`, `ratelimit-remaining`) show your remaining allowance; see the Docker Hub usage docs for the exact token-fetching steps.

#### D5. denied / unauthorized when pushing

**Messages:**
```
denied: requested access to the resource is denied
unauthorized: authentication required
denied: permission_denied: write_package
```
**Cause:** not logged in to that registry, the repository name does not include your namespace (you tried to push `app:1.0` instead of `youruser/app:1.0`), the token lacks write scope (GHCR needs `write:packages`; in GitHub Actions the job needs `permissions: packages: write`), or your account is not allowed on that repository.

**Fix:** `docker login ghcr.io -u USER --password-stdin` (pipe the token in: `echo $TOKEN | docker login ghcr.io -u USER --password-stdin`; `--password-stdin` avoids leaving the token in your shell history), tag with the full registry path (`docker tag app:1.0 ghcr.io/user/app:1.0`), then push.

#### D6. invalid reference format

**Message:**
```
docker: invalid reference format: repository name (MyApp) must be lowercase.
```
or plain `invalid reference format`.

**Cause:** Image names must be lowercase (the repository part), with only letters, digits, `-`, `_` and `.`. Also happens with stray spaces, a trailing `:` with no tag, or unexpanded variables (`image: myapp:${TAG}` with `TAG` unset gives `myapp:`).

**Fix:** lowercase everything; check `docker compose config` to see the substituted value; use `${TAG:-latest}` or `${TAG:?TAG is required}` to default or fail loudly.

### Group E: Running containers

#### E1. Container exits immediately

**Symptom:** `docker run -d myimage` prints an ID; `docker ps` shows nothing; `docker ps -a` shows `Exited (0)` or `Exited (1)` seconds later.

**Cause:** A container lives only as long as its **main process** (PID 1). If that process finishes or crashes, the container stops. Frequent reasons:
- The command is a one-shot task (a script that ends).
- The app daemonises/forks into the background (`nginx` without `-g 'daemon off;'`, `service x start`), so PID 1 exits while the real work runs in a child that dies with the container.
- The app crashed on startup (missing environment variable, cannot connect to the database, bad config).
- Wrong `CMD` (exit 127) or file not executable (exit 126).
- An interactive program (a shell, Python REPL) needs a terminal: run with `-it`.

**Fix:**
```bash
docker ps -a                     # see the exit code
docker logs NAME                 # see why
docker run --rm -it --entrypoint sh IMAGE     # poke around
docker inspect IMAGE --format '{{json .Config.Cmd}} {{json .Config.Entrypoint}}'   # what does it run?
```
- Run the server in the **foreground** (nginx: `CMD ["nginx", "-g", "daemon off;"]`; Python/Node servers run in the foreground already).
- Do not use `tail -f /dev/null` hacks to keep containers alive in production; fix the real process.

#### E2. Container keeps restarting (restart loop)

**Symptom:** `STATUS` flaps between `Restarting (1) 5 seconds ago` and `Up 2 seconds`.

**Cause:** `restart: always`/`unless-stopped`/`on-failure` plus an app that crashes on start. Docker keeps trying with an increasing delay.

**Fix:** read the logs *of the crashing instance* (`docker logs --tail 200 NAME` works while it is between restarts); check `docker inspect NAME --format '{{.RestartCount}} {{.State.ExitCode}}'`; temporarily set `restart: "no"` so it stays stopped and you can examine it; fix the root cause (usually configuration, a dependency not reachable, or OOM).

#### E3. CRLF line endings in shell scripts (Windows)

**Messages:**
```
exec /entrypoint.sh: no such file or directory
/bin/sh^M: bad interpreter: No such file or directory
standard_init_linux.go: exec user process caused: no such file or directory
': not found     (garbled errors with a carriage return)
```
**Cause:** On Windows, Git (with `core.autocrlf=true`) and many editors save text files with **CRLF** line endings (`\r\n`), while Linux expects **LF** (`\n`). The kernel reads the shebang line as `#!/bin/sh\r`, looks for an interpreter literally named `sh\r`, and fails with the confusing "no such file or directory" even though the file clearly exists.

**Fix (pick the lasting one):**
1. Convert the file: `dos2unix entrypoint.sh` or `sed -i 's/\r$//' entrypoint.sh`; in VS Code click `CRLF` in the status bar and choose `LF`.
2. Tell Git to keep LF for these files. Add a `.gitattributes` file at the repo root:
   ```
   * text=auto
   *.sh text eol=lf
   Dockerfile text eol=lf
   *.yaml text eol=lf
   ```
   - `* text=auto` lets Git normalise text files; `*.sh text eol=lf` forces LF checkouts of shell scripts on every OS. Then re-normalise: `git add --renormalize .`.
3. As a belt-and-braces measure, strip `\r` during the build: `RUN sed -i 's/\r$//' /entrypoint.sh`.
4. Set your editor/`.editorconfig` to `end_of_line = lf`.

#### E4. Wrong interpreter / "permission denied" running the entrypoint

**Messages:**
```
exec /entrypoint.sh: permission denied
OCI runtime create failed: ... exec: "./start.sh": permission denied: unknown
```
**Cause:** the script is not marked executable. Files copied from Windows lose the executable bit.

**Fix:** `COPY --chmod=755 entrypoint.sh /entrypoint.sh` (BuildKit) or `RUN chmod +x /entrypoint.sh`; or in Git: `git update-index --chmod=+x entrypoint.sh`. Or invoke through the interpreter: `CMD ["sh", "/entrypoint.sh"]`.

#### E5. OCI runtime: executable file not found in $PATH

**Message:**
```
docker: Error response from daemon: failed to create task for container: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: exec: "python3": executable file not found in $PATH: unknown
```
**Cause:** The command you gave does not exist in that image (slim/alpine images lack many tools: `bash`, `curl`, `python3`), or a typo, or you used a shell feature (`&&`, `$VAR`) in exec form without a shell.

**Fix:** check what is available (`docker run --rm IMAGE ls /usr/bin`), install it (`apk add --no-cache bash` / `apt-get install -y ...`), use `sh` instead of `bash` on Alpine, or correct the path. Exec form `CMD ["a","b"]` runs no shell: for shell features use `CMD ["sh","-c","cmd1 && cmd2"]`.

#### E6. Container unhealthy

**Symptom:** `STATUS` shows `Up 5 minutes (unhealthy)`, or `docker compose up --wait` fails with `container ... is unhealthy`.

**Cause:** The healthcheck command is failing: the app is not actually ready, the check hits the wrong port/path, the check tool (`curl`) does not exist in the image, the check runs *before* startup finished (no `start_period`), or the check's timeout is shorter than the response time.

**Fix:**
```bash
docker inspect NAME --format '{{json .State.Health}}' | python -m json.tool
```
- That prints the last several health-check attempts with their **exact output and exit code**. This is the fastest way to see *why* it fails.
- Run the same command by hand: `docker exec NAME <the test command>`.
- Add `start_period`, raise `timeout`, use a tool present in the image (a one-line `python -c` or `wget -q --spider`), and remember **plain Docker does not restart unhealthy containers**; it only labels them.

#### E7. depends_on did not wait (race condition on startup)

**Symptom:** the API crashes at boot with `connection refused` to the database, then works after a restart.

**Cause:** By default `depends_on: [db]` only waits until the `db` container has **started**, not until PostgreSQL is **ready to accept connections** (database startup takes seconds).

**Fix:** define a healthcheck on the dependency and wait for health:

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
- `pg_isready` is Postgres's own readiness probe command. `condition: service_healthy` makes Compose start `api` only after `db`'s healthcheck passes. Also make the application itself **retry** database connections with backoff, because in production the database can restart at any time ([Chapter 6](06-docker-compose.md)).

#### E8. "localhost" confusion: connection refused between containers

**Symptoms:** `Connection refused` / `Can't connect to server on 'localhost'` from inside a container to a database or API that is clearly running; or your browser cannot reach a service that listens inside a container.

**Cause:** `localhost` always means **"this same network namespace"**. Inside container A, `localhost` is container A, not your laptop and not container B. The same word means different machines in different places.

```
 Browser on host ---- localhost:8080 ----> published port ---> container "web" (port 80)
 Container api   ---- localhost:5432 ----> api itself!  (WRONG: db is another container)
 Container api   ---- db:5432        ----> container "db" (RIGHT: use the service name on the same network)
```

**Fix:**
- Container-to-container: use the **service name** as hostname (`postgresql://user:pw@db:5432/app`), and ensure both are on the same user-defined network (Compose does this automatically).
- Host reaching container: use the **published** port (`localhost:8080`) and make sure you published it (`-p 8080:80`).
- Container reaching a service on the **host**: use `host.docker.internal` (built in on Docker Desktop). On Linux Engine add `--add-host=host.docker.internal:host-gateway` (Compose: `extra_hosts: ["host.docker.internal:host-gateway"]`).
- Server inside the container must listen on `0.0.0.0` (all interfaces), **not** `127.0.0.1`, or published ports will not reach it. For example `uvicorn app:app --host 0.0.0.0 --port 8000`; Flask `app.run(host="0.0.0.0")`. This is the most common cause of "I published the port but the browser shows connection reset/empty reply."
- The container port you publish must be the port the app really listens on.

#### E9. Running containers can't resolve DNS / reach the internet

**Messages:**
```
Temporary failure in name resolution
Could not resolve host: pypi.org
getaddrinfo ENOTFOUND registry.npmjs.org
dial tcp: lookup registry-1.docker.io: no such host
```
**Cause:** Docker's embedded DNS forwards to the host's resolvers. Problems arise when: the host uses a local stub resolver address (`127.0.0.53` on Ubuntu) that cannot be reached from containers; a VPN or corporate network changes DNS; the firewall blocks outbound DNS (UDP/TCP 53); or the machine is simply offline.

**Fix:**
```bash
docker run --rm busybox nslookup google.com          # test from a container
docker run --rm busybox cat /etc/resolv.conf         # which DNS does it use?
docker run --rm --dns 8.8.8.8 busybox nslookup google.com   # does explicit DNS work?
```
- If an explicit DNS server works, set it globally in `/etc/docker/daemon.json`: `{ "dns": ["1.1.1.1", "8.8.8.8"] }` (or your corporate DNS servers) and restart Docker. In Compose use `dns:` per service.
- On Docker Desktop with a VPN: restart Docker Desktop after connecting or disconnecting the VPN; check Desktop's network settings. Also look for MTU problems (see F3).
- Between containers: containers on the **default bridge network** do not get name-based DNS; use a **user-defined network** (Compose creates one automatically) ([Chapter 5](05-storage-and-networking.md)).

### Group F: Networking and proxies

#### F1. TLS / certificate errors (corporate proxy)

**Messages:**
```
x509: certificate signed by unknown authority
Get "https://registry-1.docker.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority
curl: (60) SSL certificate problem: unable to get local issuer certificate
pip: SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] ...
```
**Cause:** Many company networks run a **TLS-intercepting proxy** (a "man-in-the-middle" security gateway) that re-signs HTTPS traffic with the company's own certificate authority (CA). Your laptop's operating system trusts that CA, but **containers have their own list of trusted CAs** and do not, so every HTTPS call from inside a container (or during a build) fails verification. Another cause is a wrong system clock (certificates look expired or not-yet-valid).

**Fix (three places, depending on where it fails):**

1. **Docker daemon pulling images** (`docker pull` fails): install the corporate CA for the daemon. On Linux put the CA file at `/etc/docker/certs.d/<registry-host>/ca.crt` (per-registry) or add it to the OS trust store (`sudo cp corp-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`) and restart Docker. On Docker Desktop, importing the CA into the Windows/macOS trust store is typically enough, then restart Desktop. Also set proxy variables if needed (below).
2. **During `docker build`** (`pip install`, `apt-get`, `npm install` fail): copy the CA into the image and update the trust store:
   ```dockerfile
   COPY corp-ca.crt /usr/local/share/ca-certificates/corp-ca.crt
   RUN update-ca-certificates
   ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt \
       SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt \
       NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/corp-ca.crt
   ```
   - `update-ca-certificates` (Debian/Ubuntu images; install the `ca-certificates` package first if it is missing) merges your CA into the system bundle. The three `ENV` lines point Python `requests`, generic OpenSSL tools and Node.js at the right bundle, because each runtime has its own habits. On Alpine the command is the same after `apk add ca-certificates`; on RHEL-family images use `update-ca-trust`. This is where a **golden base image** that already contains the company CA pays off ([Chapter 11](11-production-and-enterprise.md#21-team-practices)).
3. **At runtime inside the container:** mount the CA and set the environment variables above, or bake it into the image.

Never "fix" this with `pip install --trusted-host`, `curl -k`, `NODE_TLS_REJECT_UNAUTHORIZED=0` or `insecure-registries` in production. Those disable verification and hide real attacks. Use them only briefly to diagnose, then add the proper CA.

**Proxy settings:** if outbound traffic must go through an HTTP proxy, configure it in three separate places:
- Docker Desktop: Settings > Resources > Proxies.
- Linux daemon: a systemd drop-in `/etc/systemd/system/docker.service.d/http-proxy.conf` with `[Service]` and `Environment="HTTP_PROXY=http://proxy.example.com:3128"`, `Environment="HTTPS_PROXY=..."`, `Environment="NO_PROXY=localhost,127.0.0.1,.internal.example.com"`, then `sudo systemctl daemon-reload && sudo systemctl restart docker`.
- Containers and builds: Docker can pass proxy settings automatically via `~/.docker/config.json` (`"proxies": {"default": {"httpProxy": "...", "httpsProxy": "...", "noProxy": "..."}}`), or per build with `--build-arg HTTP_PROXY=...`.

#### F2. Registry timeouts / TLS handshake timeout

**Messages:**
```
Get "https://registry-1.docker.io/v2/": net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
net/http: TLS handshake timeout
```
**Cause:** no route to the registry: firewall, proxy not configured, DNS failure, VPN, or slow/flaky network.

**Fix:** test from the host (`curl -I https://registry-1.docker.io/v2/` should return `401 Unauthorized`, which is a *good* sign: it means the registry is reachable and wants authentication); configure proxy (F1); try another network; use a registry mirror.

#### F3. Containers work, but large downloads hang (MTU problem)

**Symptom:** small requests succeed, big downloads (`apt-get update`, `pip install` of large wheels, `git clone`) stall or time out, mostly on VPNs or some cloud networks.

**Cause:** the **MTU** (maximum transmission unit: the largest packet size) of the container network (default 1500) is larger than what the VPN/overlay path allows, so big packets are dropped.

**Fix:** set a smaller MTU in `daemon.json`: `{ "mtu": 1400 }` (or per network with `-o com.docker.network.driver.mtu=1400`), restart Docker, retest. Try 1400, then lower (1350, 1300) if still failing.

#### F4. Published port not reachable from other machines (or *too* reachable)

**Not reachable:** the app listens on `127.0.0.1` inside the container (see E8); the mapping binds to loopback (`-p 127.0.0.1:8080:80` is deliberate); a cloud security group or host firewall blocks the port; you used the wrong host port.
**Too reachable:** on Linux, published ports can bypass `ufw`; bind internal services to `127.0.0.1` or don't publish them at all ([Chapter 11](11-production-and-enterprise.md#7-where-do-you-run-the-containers-deployment-targets)).

**Check:** `docker port NAME`, `docker ps` (look at the `PORTS` column: `0.0.0.0:8080->80/tcp` is open to all interfaces; `127.0.0.1:8080->80/tcp` is local only), and test from outside with `curl http://SERVER_IP:8080` or an external port scanner.

### Group G: Builds and files

#### G1. COPY failed: file not found / excluded by .dockerignore

**Messages:**
```
COPY failed: file not found in build context or excluded by .dockerignore: stat app.py: file does not exist
ERROR: failed to solve: failed to compute cache key: failed to calculate checksum of ref ...: "/app.py": not found
```
**Cause:** The path in `COPY` is relative to the **build context** (the folder you pass at the end of `docker build`, usually `.`), not to the Dockerfile. Reasons for failure: the file is outside the context (you cannot `COPY ../x`), the path is misspelled or has the wrong case (Linux is case-sensitive: `App.py` vs `app.py`), or the file is listed in **`.dockerignore`** (so it was never sent to the builder).

**Fix:**
```bash
docker build -f docker/Dockerfile -t myapp .      # -f picks the Dockerfile; the final "." is still the context
cat .dockerignore                                 # is the file excluded?
docker buildx build --progress=plain .            # shows what is happening
```
- Check the exact path/case; check the context argument; check `.dockerignore` (common culprits: `*.md`, `build/`, `dist/`, `*.json`, `.env`, or `*` followed by exceptions `!app.py`).
- Different Dockerfile location: either make the context the repo root and write paths from there, or use named contexts/`additional_contexts` in Compose.

#### G2. Build cache surprises

**Symptoms (opposite problems):**
- *"My change is not in the image"*: Docker reused a cached layer and did not notice your change, for instance after `RUN git clone ...`, `RUN curl ... latest`, or `RUN apt-get update` (the instruction text is unchanged, so the cached result is reused, possibly stale).
- *"Every build reinstalls all dependencies"*: an early `COPY . .` before `pip install`/`npm install` invalidates the cache whenever any file changes.

**Cause and fix:** Docker caches each Dockerfile instruction by its text and, for `COPY`/`ADD`, by the file contents. **When a layer changes, every layer after it is rebuilt.** So:
1. Order instructions from least to most frequently changing: base image, system packages, **copy dependency manifests** (`requirements.txt`, `package.json`), install dependencies, **then** `COPY` the source.
   ```dockerfile
   COPY requirements.txt .
   RUN pip install --no-cache-dir -r requirements.txt
   COPY . .
   ```
2. Use BuildKit cache mounts to speed up package managers without bloating the image: `RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt`.
3. Force a fresh build when you suspect staleness: `docker build --no-cache .` (ignore the cache entirely), or `docker build --pull .` (also fetch the newest base image), or change a build `ARG` just before the layer you want to refresh.
4. Combine `apt-get update && apt-get install -y ...` in the **same** `RUN`, and pin versions where reproducibility matters.
5. Deleted files still occupy layers: `RUN rm bigfile` in a later layer does not shrink the image. Do download-use-delete in one `RUN`, or use a multi-stage build ([Chapter 4](04-dockerfile-deep-dive.md)).

#### G3. A build step fails: "did not complete successfully"

**Message:**
```
ERROR: failed to solve: process "/bin/sh -c pip install -r requirements.txt" did not complete successfully: exit code: 1
```
**Cause:** the command inside a `RUN` returned non-zero. Docker's message does not show *why*; the reason is **above** that line in the output.

**Fix:** re-run with plain progress output so nothing is collapsed: `docker build --progress=plain --no-cache .` (more in [Section 4](#4-debugging-builds)). Typical causes: network/DNS/proxy failure inside the build, a package needing build tools (`gcc`, `libpq-dev`), a version conflict, a package that has no wheel for your CPU architecture (arm64), or a missing build argument.

#### G4. Cannot find Dockerfile / unable to prepare context

**Messages:**
```
ERROR: failed to solve: failed to read dockerfile: open Dockerfile: no such file or directory
unable to prepare context: path "/some/path" not found
ERROR: resolve : lstat ...: no such file or directory
```
**Cause:** wrong directory, wrong file name (`dockerfile`, `Dockerfile.dev`), or you forgot the context argument.

**Fix:** `ls` to confirm; use `-f path/to/Dockerfile`; in Compose check `build.context` and `build.dockerfile` paths (relative to the Compose file's directory).

#### G5. Bind mount permission denied

**Messages:**
```
PermissionError: [Errno 13] Permission denied: '/data/output.txt'
mkdir: cannot create directory '/app/logs': Permission denied
```
**Cause:** On Linux, file ownership is by **numeric user ID (UID)**. Your container runs as, say, UID 1000 or 10001, while the host folder you mounted is owned by another UID (or root), so the container user cannot write. (Docker Desktop on Windows/macOS usually hides this by translating permissions; the problem shows up when you move to a Linux server.) If the mounted folder did not exist, Docker creates it as **root**.

**Fix (choose one):**
```bash
id                                                        # your UID and GID on the host
docker run --user "$(id -u):$(id -g)" -v "$PWD/data:/data" IMAGE   # run as you
sudo chown -R 10001:10001 ./data                          # make the host folder owned by the container's UID
```
- Compose: `user: "${UID:-1000}:${GID:-1000}"`, or create the folder beforehand with the right owner.
- Prefer **named volumes** for application data: Docker initialises them with the image's ownership for the mount path (if you `chown` the directory in the Dockerfile before the volume is first used).
- **SELinux** systems (RHEL, Fedora, Rocky): you may need the relabel flag, `-v ./data:/data:z` (shared label) or `:Z` (private label), otherwise access is denied even with correct ownership. `ausearch -m avc -ts recent` confirms SELinux denials.
- Read-only root filesystems (`--read-only`): write to a mounted volume or a `tmpfs` (`--tmpfs /tmp`).

#### G6. Windows path problems in Git Bash (MSYS path mangling)

**Symptoms:** `docker run -v /c/Users/me/data:/data ...` mounts the wrong thing; a command like `docker exec api ls /app` returns `ls: cannot access 'C:/Program Files/Git/app'`; flags like `-w /app` become `-w C:/Program Files/Git/app`.

**Cause:** **Git Bash** (MSYS2) automatically rewrites anything that looks like a Unix path into a Windows path before the command starts, which breaks container paths.

**Fix:** disable the conversion for that command or session:
```bash
MSYS_NO_PATHCONV=1 docker exec api ls /app           # one command
export MSYS_NO_PATHCONV=1                            # whole session (add to ~/.bashrc)
docker exec api sh -c "ls /app"                      # or wrap the path inside a quoted sh -c
docker run -v //c/Users/me/data:/data IMAGE          # a double leading slash also avoids conversion
```
Or use PowerShell/WSL where this does not happen. In PowerShell, write mounts as `-v "${PWD}/data:/data"` (note `${PWD}` with braces and quotes) or use the Compose `./data:/data` form.

#### G7. File changes on the host do not show up in the container (hot reload)

**Cause:** On Windows/macOS file-change *notifications* may not propagate through bind mounts from the Windows filesystem into the Linux VM, so watchers (webpack, uvicorn `--reload`, nodemon) never notice edits. Also, code copied into the image at build time is a *copy*, not live.

**Fix:** use a bind mount in development (`./src:/app/src`); keep the project in the **WSL 2 Linux filesystem** (`\\wsl$\Ubuntu\home\you\project`) rather than `C:\...`, which is far faster and notifies properly; or switch the watcher to polling (`CHOKIDAR_USEPOLLING=true`, `WATCHFILES_FORCE_POLLING=true`, `nodemon -L`). Compose's `develop.watch` (`docker compose watch`) syncs or rebuilds on change ([Chapter 6](06-docker-compose.md)).

### Group H: Compose-specific

#### H1. Data missing after `docker compose down -v`

**Symptom:** the database is empty after you "restarted" the stack.

**Cause:** `docker compose down` removes containers and networks but **keeps named volumes**. Adding **`-v`** (`--volumes`) also **deletes the named volumes**, and that is where your data lived. Other ways to "lose" data: you changed the project name or folder name (Compose prefixes volume names with the project name, so a new project name means a *new, empty* volume); you used an anonymous volume; `docker volume prune`/`docker system prune --volumes` deleted an "unused" volume; you ran the container without any volume so data lived in the container's writable layer.

**Fix and recovery:** `docker volume ls` (is the old volume still there? names look like `oldproject_pgdata`); mount the old one explicitly with `external: true` and its full name; restore from backup. **Prevention:** never use `down -v` casually; set `name:` at the top of the Compose file to pin the project name; back up volumes; label important volumes; use external volumes for critical data.

#### H2. "version is obsolete" warning

**Message:**
```
WARN[0000] /path/compose.yaml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion
```
**Cause:** The top-level `version: "3.8"` key belonged to the old Compose file formats. The modern Compose Specification does not use it.

**Fix:** delete the `version:` line. Harmless, but remove the noise.

#### H3. YAML syntax errors

**Messages:**
```
yaml: line 12: mapping values are not allowed in this context
yaml: line 8: did not find expected key
services.api.ports must be a list
service "api" refers to undefined volume pgdata: invalid compose project
```
**Cause:** indentation errors (YAML uses **spaces, never tabs**), a missing space after a colon (`key:value` instead of `key: value`), unquoted special values (`yes`/`no`/`on` become booleans; port mappings like `80:80` in some YAML parsers can be misread as base-60 numbers, so **quote them**: `"80:80"`), or a named volume used in a service but not declared under the top-level `volumes:`.

**Fix:** run `docker compose config`; it validates the file and prints the final result, usually pointing at the exact line. Use an editor with YAML linting.

#### H4. Orphan containers warning

**Message:**
```
WARN[0000] Found orphan containers ([myproject-old-service-1]) for this project. If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
```
**Cause:** you removed/renamed a service in the Compose file but its old container still exists.

**Fix:** `docker compose up -d --remove-orphans` (removes containers of services no longer defined). Do not use it if a different Compose file in the same project folder legitimately defines other services.

#### H5. Environment variable is blank / `.env` not applied

**Symptoms:** `WARN The "DB_PASSWORD" variable is not set. Defaulting to a blank string.` or values from `.env` not reaching the container.

**Cause and notes:**
- A `.env` file in the **project directory** supplies values for `${VARIABLES}` **substituted in the Compose file itself**. It does **not** automatically become environment variables inside containers. To pass a file's contents into the container use `env_file:` or list `environment:` entries with `${VAR}`.
- Shell environment variables override `.env` values.
- Inside `.env`, avoid spaces around `=` and be careful with quotes and `#` characters in values.

**Fix:** run `docker compose config` to see the resolved values; use `${VAR:?error message}` to fail fast on missing values; use `docker compose exec api env` to see what the container actually received.

#### H6. Service builds fine alone but fails in Compose (or vice versa)

Usually a difference in **context path**, **environment**, **network** or **build args**. Compare `docker compose config` output with your manual `docker build`/`docker run` flags.

### Group I: Platform-specific and GPU

#### I1. GPU not visible in the container

**Message:**
```
docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]
```
**Cause:** The **NVIDIA Container Toolkit** is not installed/configured on the Linux host, or the host NVIDIA driver is missing. (Docker Desktop on Windows supports GPUs through WSL 2 with a recent NVIDIA Windows driver.)

**Fix:** install the NVIDIA driver, install the NVIDIA Container Toolkit, run `sudo nvidia-ctk runtime configure --runtime=docker` and `sudo systemctl restart docker`, then test: `docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi` (pick a current CUDA base image tag that exists). Details and Compose syntax in [Chapter 8](08-docker-for-ai-and-ml.md). If `nvidia-smi` works on the host but not in the container, check the toolkit configuration; if the container shows a CUDA version error, the image's CUDA version is newer than your host driver supports.

#### I2. Slow bind mounts and builds on Windows/macOS

**Cause:** file I/O across the Windows/macOS-to-VM boundary is slow compared to native Linux disk. Projects with `node_modules` or Python virtualenvs mounted from the host crawl.

**Fix:** keep the code inside the WSL 2 filesystem (Windows) and run Docker commands from there; keep dependency folders in a named volume (`- node_modules:/app/node_modules`) instead of the bind mount; use `.dockerignore` so the build context stays small; on macOS use the VirtioFS file sharing option in Docker Desktop settings.

#### I3. Clock drift / "certificate has expired or is not yet valid"

**Cause:** After a laptop sleeps, the WSL 2 VM clock can lag behind, so TLS certificates appear not-yet-valid and package manager signatures fail (`Release file ... is not valid yet`).

**Fix:** `wsl --shutdown` and restart Docker Desktop; or sync the clock (`sudo hwclock -s` in WSL, or restart the VM); on servers run NTP (`chrony` or `systemd-timesyncd`).

---

## 4. Debugging builds

Builds fail differently from running containers: there is no container to `exec` into, and BuildKit hides output by default.

### 4.1 See all the output

```bash
docker build --progress=plain -t myapp .
```
- `--progress=plain` replaces the animated, collapsed progress view with a full, sequential log of every command's output. When a step fails, the real error is printed right above the `ERROR: failed to solve` line. Environment variable equivalent: `BUILDKIT_PROGRESS=plain`. (Accepted values: `auto`, `plain`, `tty`, `quiet`, `rawjson`.)

With Compose: `docker compose build --progress=plain api`.

### 4.2 Remove the cache as a variable

```bash
docker build --no-cache --pull --progress=plain -t myapp .
```
- `--no-cache` ignores all cached layers; `--pull` fetches the newest version of the base image. If the build now works (or now fails), the cache was involved. It is slow, so use it for diagnosis only.

### 4.3 Stop at a stage: `--target`

For multi-stage Dockerfiles you can build and inspect an intermediate stage:

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
- `--target builder` builds only up to the stage named `builder` and tags the result, so you can open a shell **inside the exact state** just before the failing step and run commands interactively.

### 4.4 Debugging a failing `RUN` step interactively

Classic trick: temporarily comment out the failing `RUN` and everything after it, build, run a shell in that image, and execute the failing command manually:

```bash
docker build -t myapp:debug .          # with the failing RUN commented out
docker run --rm -it myapp:debug sh     # then type the failing command by hand and read the error
```
- Equivalent: add `RUN false || true` or `RUN ls -la /app; pwd; env` (print state) before the failing step to see what files and variables exist at that moment.

### 4.5 Check the Dockerfile before building

```bash
docker build --check .
```
- Runs **build checks** (a built-in Dockerfile linter in BuildKit) that warn about issues such as undefined variables, a `FROM` with case-mismatched `AS`, JSON-form `CMD` mistakes, and `latest` tags, without performing the build. Also look at **hadolint** for stricter linting.

### 4.6 Inspect what was sent to the builder

If `COPY` cannot find a file, list exactly what the context contains:

```dockerfile
FROM busybox
COPY . /ctx
RUN find /ctx -maxdepth 2 | sort
```
```bash
docker build --progress=plain --no-cache -f Dockerfile.ctx .
```
- This throwaway Dockerfile copies the whole context and lists it, so you see precisely which files survived `.dockerignore`. Check `.dockerignore` rules (a `*` ignore with `!` exceptions is easy to get wrong).

### 4.7 Look at layers and sizes

```bash
docker history myapp:1.0
docker history --no-trunc --format "{{.Size}}\t{{.CreatedBy}}" myapp:1.0
```
- Lists each layer, its size and the instruction that made it, so you can spot the giant layer (a forgotten `apt` cache, a copied dataset). Third-party **dive** (`wagoodman/dive`) shows each layer's file changes interactively.

### 4.8 Build secrets and private dependencies

If a build needs a credential (private package index, SSH to clone a private repo), never put it in `ARG`/`ENV` (stays in the image history). Use BuildKit mounts:

```dockerfile
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install -r requirements.txt
```
```bash
docker build --secret id=pip_token,src=./pip_token.txt .
```
- The secret exists only during that `RUN` step at `/run/secrets/pip_token` and never lands in a layer. For SSH use `RUN --mount=type=ssh` with `docker build --ssh default .`. ([Chapter 10](10-security.md).)

### 4.9 Multi-platform build problems

- `exec format error` during a cross-platform build means QEMU emulation is not set up: `docker run --privileged --rm tonistiigi/binfmt --install all` registers the emulators (or use `docker/setup-qemu-action` in CI).
- Python packages without arm64 wheels compile from source and take forever or fail: add build tools (`gcc`, `python3-dev`) or pin versions that have wheels.
- Inspect builders: `docker buildx ls`; create one: `docker buildx create --name mybuilder --use`.

### 4.10 Builder cache problems

- Build cache grew huge: `docker buildx du` shows usage, `docker builder prune --filter "until=72h"` trims.
- Strange cache behaviour in CI: your CI cache backend (`type=gha`/`type=registry`) may be stale or evicted; run once with `--no-cache` to confirm.

---

## 5. Performance tuning tips

### 5.1 Faster builds

1. **Order layers** by change frequency (dependencies before source) so the cache hits.
2. **`.dockerignore`**: exclude `.git`, `node_modules`, `__pycache__`, `*.log`, `.venv`, datasets, model weights, `.env`. A huge context is uploaded to the builder every build; excluding it can turn a 30-second "sending build context" into a blink.
3. **BuildKit cache mounts** for package managers: `RUN --mount=type=cache,target=/root/.cache/pip pip install ...`, `--mount=type=cache,target=/var/cache/apt` (with `sharing=locked`) for apt, `/root/.npm` for npm, `/go/pkg/mod` for Go.
4. **Multi-stage builds** keep compilers out of the final image and let independent stages build in parallel (BuildKit runs independent stages concurrently).
5. **Pin and reuse base images**; use a **registry or CI cache backend** (`--cache-from`/`--cache-to`) so ephemeral CI runners do not start cold.
6. **Install only what you need**: `--no-install-recommends` for apt, `--no-cache-dir` for pip (in the final image), `npm ci --omit=dev`.
7. For ML images: pin heavy wheels (`torch`) in an early, rarely changing layer; consider a prebuilt base image from your registry containing CUDA + PyTorch, and keep **model weights out of the image** (mount a volume or download at startup; see [Chapter 8](08-docker-for-ai-and-ml.md)).

### 5.2 Smaller, faster-starting images

- Slim or distroless bases (`python:3.12-slim`, `gcr.io/distroless/...`, `alpine` when your dependencies support musl libc: some Python wheels do not, which can make Alpine *slower* to build and bigger in practice).
- Combine cleanup with install in the same layer; remove package caches.
- Check with `docker images` and `docker history`; use `dive` to find waste.
- Smaller images pull faster, which directly improves autoscaling and cold-start time.

### 5.3 Faster runtime

- **Avoid bind mounts for hot paths on Windows/macOS**; use volumes or the WSL 2 filesystem.
- Give the Docker VM **enough** CPU and memory (and not more than you can spare); undersized VMs cause swapping and OOM kills.
- Use the **native architecture**; emulation is 5-20x slower.
- Set correct limits so one container cannot starve others; use `docker stats` to find the noisy one.
- Use `init: true`/`--init` so zombie processes are reaped and signals work.
- Keep logs under control (rotation; avoid logging at debug in hot paths); heavy stdout logging can slow an app when the log driver blocks.
- Tune the app for containers: worker counts that match CPU limits (Gunicorn `workers = 2 * cores + 1` is a starting point *for cores you actually have*, not the host's core count), database connection pools sized per replica.
- Network performance: use user-defined networks (not `--link`), avoid userland proxy overhead for high-throughput ports on Linux if needed (`"userland-proxy": false` in daemon.json uses iptables only, verify your setup after the change), consider `network_mode: host` only when you understand the isolation trade-off (Linux only).
- Storage: use `overlay2` (the default driver on modern Linux) on ext4/xfs; put heavy-write data in volumes, not the container's writable layer (copy-on-write is slow for large files and databases).
- Startup time: avoid heavy work at import time; lazy-load models; use readiness probes so traffic waits.

### 5.4 Quick performance triage

```bash
docker stats --no-stream           # who uses CPU/memory?
docker top NAME                    # what processes?
docker system df                   # disk pressure?
docker info | grep -E "Storage Driver|Cgroup|Total Memory|CPUs"
```
- The `info` line items confirm the storage driver, cgroup version and the CPUs/memory the daemon (or Desktop VM) actually has. On Docker Desktop, "Total Memory" is the VM's allotment, which is often the real bottleneck.

---

## 6. The one-page cheat sheet

Copy this section into your own notes. `NAME` = container name or ID, `IMG` = image, `VOL` = volume, `NET` = network.

### Images

| Command | What it does |
|---|---|
| `docker pull IMG:TAG` | Download an image from a registry |
| `docker images` / `docker image ls` | List local images |
| `docker build -t NAME:TAG .` | Build an image from the Dockerfile in the current folder (`.` = build context) |
| `docker build -f path/Dockerfile -t NAME:TAG ctx` | Use a specific Dockerfile and context folder |
| `docker build --no-cache --pull .` | Rebuild ignoring cache; refresh base image |
| `docker build --target STAGE .` | Build only up to a named multi-stage stage |
| `docker build --progress=plain .` | Show full build output |
| `docker tag SRC:TAG REG/NAME:TAG` | Add another name/tag to an image |
| `docker push REG/NAME:TAG` | Upload an image to a registry |
| `docker history IMG` | Show an image's layers and sizes |
| `docker image inspect IMG` | Full JSON details (platform, env, cmd, layers) |
| `docker rmi IMG` / `docker image rm IMG` | Delete a local image |
| `docker image prune` | Delete dangling (untagged) images |
| `docker image prune -a` | Delete all images not used by a container |
| `docker save -o f.tar IMG` / `docker load -i f.tar` | Export / import images as a tar file (offline transfer) |
| `docker scout quickview IMG` | Summary of vulnerabilities (Docker Scout; plan limits apply, check Docker's pricing) |

### Containers

| Command | What it does |
|---|---|
| `docker run IMG` | Create and start a container (foreground) |
| `docker run -d --name NAME IMG` | Run in the background (`-d` detached) with a name |
| `docker run -it --rm IMG sh` | Interactive shell, deleted on exit |
| `docker run -p HOST:CONTAINER IMG` | Publish a port (`-p 8080:80`); `-p 127.0.0.1:8080:80` for local-only |
| `docker run -e KEY=value IMG` / `--env-file f` | Set environment variables |
| `docker run -v VOL:/path IMG` | Mount a named volume (persistent data) |
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
| `docker exec -it NAME sh` | Shell inside a running container |
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
| `docker volume rm VOL` | Delete a volume (**data is lost**) |
| `docker volume prune` | Delete volumes no container uses (**dangerous on servers**) |
| `docker run --rm -v VOL:/d alpine tar czf - -C /d . > backup.tgz` | Back up a volume to a tarball on the host |

### Networks

| Command | What it does |
|---|---|
| `docker network ls` | List networks |
| `docker network create NET` | Create a user-defined bridge network (gives DNS by container name) |
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
| `docker compose up -d --remove-orphans` | Also remove containers of services no longer defined |
| `docker compose down` | Stop and remove containers and networks (keeps named volumes) |
| `docker compose down -v` | ...and **delete named volumes** (data loss) |
| `docker compose ps` | Service status |
| `docker compose logs -f SERVICE` | Follow logs |
| `docker compose exec SERVICE sh` | Shell in a running service |
| `docker compose run --rm SERVICE CMD` | One-off container for a service |
| `docker compose build [--no-cache] SERVICE` | Build images |
| `docker compose pull` | Pull newer images |
| `docker compose restart SERVICE` | Restart service(s) |
| `docker compose stop` / `start` | Stop / start without removing |
| `docker compose config` | Show the final merged config (best for debugging) |
| `docker compose --profile NAME up -d` | Enable optional profile services |
| `docker compose -f a.yaml -f b.yaml up` | Merge several Compose files |
| `docker compose watch` | Sync/rebuild on file changes (development) |
| `docker compose cp`, `top`, `events`, `images` | Copy files, list processes, stream events, list images |

### Buildx (modern builder)

| Command | What it does |
|---|---|
| `docker buildx ls` | List builders |
| `docker buildx create --name b --use` | Create and select a builder |
| `docker buildx build --platform linux/amd64,linux/arm64 -t REG/IMG:TAG --push .` | Multi-arch build and push |
| `docker buildx build --load -t IMG .` | Build and load a single-platform image into local Docker |
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
| `docker system prune -a` | ...plus all unused images (**never add `--volumes` blindly**) |
| `docker builder prune` | Remove build cache |
| `docker context ls` / `docker context use NAME` | List / switch which Docker engine the CLI talks to |
| `docker login [REG]` / `docker logout` | Authenticate to a registry |
| `docker debug NAME` | Enhanced debug shell (Docker Desktop feature) |

### Useful formatting and one-liners

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"     # tidy container table
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' NAME   # container IP
docker rm -f $(docker ps -aq)                    # DANGER: removes ALL containers (do not run on servers)
docker stop $(docker ps -q)                      # stop all running containers
docker images --filter dangling=true             # list dangling images
docker ps -a --filter status=exited              # exited containers
docker run --rm -v "$PWD":/work -w /work IMG CMD # run a tool against the current folder
```

---

## 7. Common mistakes about troubleshooting itself

1. **Changing many things at once.** You will never learn which one fixed it.
2. **Not reading the whole error.** The first line is often generic; the cause is on the last lines or just above `ERROR: failed to solve`.
3. **Deleting everything ("nuke it from orbit") as the first step.** `docker system prune -a --volumes` fixes some problems and destroys your data; diagnose first.
4. **Debugging inside a container and forgetting the changes vanish** when it is recreated. Fix the Dockerfile or Compose file, not the running container.
5. **Using `latest` and not knowing what changed.** Pinned versions turn "it broke" into "we upgraded X."
6. **Blaming Docker for app bugs** (or vice versa). Run the app outside Docker once to see whether the problem is in the code.
7. **Disabling TLS verification permanently** to get past a corporate-proxy error.
8. **Skipping `docker compose config`.** It would have shown the blank variable or the indentation error in one second.
9. **Not checking the host**: full disk, exhausted memory, wrong time, firewall changes.
10. **Not writing the fix down.** Add each solved problem to a personal runbook; it becomes your most valuable asset (and, in an interview, a source of real stories, see [Chapter 13](13-interview-questions-roadmap-glossary.md)).

---

Previous: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md) | Next: [Chapter 13: Interview Questions, Roadmap and Glossary](13-interview-questions-roadmap-glossary.md)
