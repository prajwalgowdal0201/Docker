# Chapter 10: Docker Security

This chapter covers the container threat model, the Docker socket, runtime hardening flags, network exposure, image supply-chain security, secrets management, registry security, benchmarks, runtime detection, and a hardened `docker run` and Compose example. It ends with a checklist and a list of anti-patterns.

Previous: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md) | Next: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md)

> **Verification note.** Key claims were checked against official documentation in October 2026 (Docker rootless, userns-remap, firewall, Scout, Hardened Images and MCP docs; NVIDIA, Sigstore and Aqua Security advisories). Where only third-party sources were found, or a detail could not be verified, it is marked inline. Run `--help` for the tool versions you have installed.

---

## 10.1 The threat model: what are we defending against?

A **threat model** states who might attack what, how, and what they gain.

### 10.1.1 Containers share the host's kernel

A **virtual machine (VM)** runs its own operating-system kernel on virtualized hardware. A **container** is a normal Linux process that the host kernel isolates with these features:

- **Namespaces**: give each container its own view of process IDs, network stack, mount points, hostname and users.
- **cgroups** (control groups): limit CPU, memory and I/O.
- **Capabilities, seccomp and LSMs (AppArmor/SELinux)**: restrict what the process may do.

```
            VIRTUAL MACHINES                              CONTAINERS
+-----------+-----------+-----------+        +-----------+-----------+-----------+
|   App A   |   App B   |   App C   |        |   App A   |   App B   |   App C   |
+-----------+-----------+-----------+        +-----------+-----------+-----------+
| Guest OS  | Guest OS  | Guest OS  |        |        (just processes,            |
| (kernel)  | (kernel)  | (kernel)  |        |   isolated by namespaces/cgroups)  |
+-----------+-----------+-----------+        +------------------------------------+
|           Hypervisor              |        |         ONE SHARED HOST KERNEL     |
+-----------------------------------+        +------------------------------------+
|            Hardware               |        |            Hardware                |
+-----------------------------------+        +------------------------------------+
```

A kernel vulnerability, or a flaw in the container runtime (such as `runc`), can let an attacker **escape** the container to the host and reach every other container. Examples:

- CVE-2019-5736: a `runc` flaw that let a malicious container overwrite the host's runc binary.
- CVE-2024-21626: a `runc` flaw, part of "Leaky Vessels".

Both were fixed by updating runc/Docker. Keep the host, kernel and Docker patched.

Analogy: VMs are separate apartments. Containers are locked rooms in one apartment that share the building's plumbing.

### 10.1.2 Who are the attackers?

| Attacker | Scenario | Defence |
|---|---|---|
| Remote attacker exploiting the app | Code execution inside the container | Non-root, read-only filesystem, dropped capabilities, egress limits |
| Malicious or compromised image | Backdoor or crypto-miner in a pulled image | Trusted sources, digest pinning, scanning, signing |
| Compromised dependency | Poisoned PyPI/npm package or CI action | Hashed lock files, SBOMs, pinned CI actions |
| Careless insider | Commits a secret, mounts `docker.sock`, runs `--privileged` | Policy, review, linting, least privilege |
| Noisy neighbour | One container consumes all RAM/CPU/PIDs | Resource limits |
| Network attacker | Reaches an exposed database port | Loopback binding, firewall, private networks |
| Compromised CI/registry credentials | Malicious `latest` pushed | Scoped tokens, MFA, signature verification at deploy |

### 10.1.3 Defence in depth

No single flag makes a container secure. Layers are stacked so that when one fails, the next holds.

```
 Layer 7  Detection & response     Falco, logs, alerts
 Layer 6  Supply chain             trusted base, pinned digests, scan, SBOM, sign
 Layer 5  Secrets                  never in image/env/git; rotate
 Layer 4  Network                  loopback, private networks, egress control, firewall
 Layer 3  Container runtime flags  non-root, cap-drop, no-new-privs, read-only, limits, seccomp
 Layer 2  Daemon & runtime         rootless/userns, patched Docker, locked socket
 Layer 1  Host & kernel            patched, minimal OS, CIS hardened, VM isolation for hostile code
```

---

## 10.2 The Docker daemon socket: `/var/run/docker.sock` equals root

### 10.2.1 What it is

The `docker` CLI is a client of the **Docker daemon** (`dockerd`), a background service. They communicate over a **Unix socket** (a special file for local inter-process communication) at `/var/run/docker.sock` on Linux. On Docker Desktop for Windows the equivalent is the named pipe `//./pipe/docker_engine`.

Anything that can send commands to the socket can start any container with any options. In default installs the daemon runs as root.

> **Access to the Docker socket is root access to the host.** The standard socket has no limited-access mode.

### 10.2.2 Demonstration (read only; do not run on a shared host)

A container started with the socket mounted:

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock some-image
```

Code inside that container can then run:

```bash
# inside the compromised container, with the docker client available
docker run --rm -it -v /:/host alpine chroot /host sh
```

- `-v /:/host`: mounts the entire host filesystem into a new container.
- `chroot /host sh`: makes `/host` the root and opens a shell. The result is root on the host.

Tools that legitimately need the socket (Portainer, Traefik's Docker provider, Watchtower, Jenkins agents that build images) accept this risk. Apply the mitigations below.

### 10.2.3 Mitigations

| Mitigation | How |
|---|---|
| Do not mount the socket | Default answer. Use Compose, CI built-ins or build tools that do not need it |
| Use a socket proxy | A small container (for example Tecnativa's `docker-socket-proxy`) exposes only selected read-only API endpoints (such as `GET /containers`). The app never touches the real socket. Check the proxy's current docs for its permission toggles |
| Mount read-only | `:ro` on the socket mount does **not** help. It only prevents replacing the socket file, not sending commands through it |
| Rootless Docker (10.4) | The daemon is not root, so a socket compromise yields one unprivileged user instead of host root |
| Build without the host daemon | Rootless BuildKit, Kaniko, Buildah, or `docker buildx` with a remote builder |
| Docker-in-Docker (DinD) | `docker:dind` needs `--privileged` and is itself risky. Use it only on disposable CI runners |
| Never expose the daemon on TCP without TLS | `dockerd -H tcp://0.0.0.0:2375` is unauthenticated: anyone on the network can run containers as root. This is a common route to crypto-mining hijacks |
| Remote administration | Use an SSH context: `docker context create prod --docker "host=ssh://deploy@server"`, then `docker --context prod ps` |
| Restrict the `docker` group | Membership is root-equivalent. Treat it like `sudo` |
| Docker Desktop Enhanced Container Isolation (ECI) | Paid-tier Docker Desktop feature that runs containers in user-namespace-isolated sandboxes. Unverified details: check Docker's docs for availability |

---

## 10.3 Run as non-root

By default the process in a container runs as root (UID 0) unless the image sets a user. Container-root is not host-root (namespaces and dropped capabilities limit it), but it is more powerful than needed. After a kernel escape, container-root becomes host root; a non-root user needs a second bug.

### 10.3.1 In the Dockerfile

```dockerfile
FROM python:3.12-slim

RUN groupadd --system --gid 10001 app \
 && useradd  --system --uid 10001 --gid app --no-create-home --shell /usr/sbin/nologin app

WORKDIR /app
COPY --chown=app:app . /app

USER 10001:10001
CMD ["python", "app.py"]
```

| Line | Purpose |
|---|---|
| `groupadd --system --gid 10001 app` | Creates a system group with a fixed numeric ID. A high number avoids clashes with host users |
| `useradd --system --uid 10001 ... --shell /usr/sbin/nologin app` | Creates an unprivileged user with no home directory and no interactive shell |
| `COPY --chown=app:app` | Files are owned by `app` so it can read them. Do not make code writable unless the app must write there |
| `USER 10001:10001` | Switches user by number. Orchestrators (Kubernetes `runAsNonRoot: true`) can verify numeric IDs; they cannot verify names |

### 10.3.2 At run time

```bash
docker run --user 10001:10001 myimage
```

- `--user uid:gid`: overrides the image's user. Useful for third-party images that run as root, if the app still works as non-root.

Check:

```bash
docker run --rm myimage id
# uid=10001(app) gid=10001(app) groups=10001(app)
```

### 10.3.3 Common non-root problems

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` writing to a volume | Volume owned by root; app is UID 10001 | `chown` in the image before `VOLUME`, run with a matching `--user`, or use an init step. In Kubernetes use `fsGroup` |
| `bind: permission denied` on port 80 | Ports below 1024 are privileged | Listen on 8080 and map `-p 80:8080`, or add the capability `NET_BIND_SERVICE` |
| App cannot write in its own directory | Code directory is read-only or not owned by the user | Write only to dedicated writable paths (`/tmp`, a volume) |

---

## 10.4 Rootless Docker and user namespaces

These are two different features.

### 10.4.1 Rootless mode: the daemon itself is not root

Per Docker's docs, rootless mode runs the daemon and containers as a non-root user inside a user namespace to reduce the impact of daemon or runtime vulnerabilities. Unlike `userns-remap`, the daemon itself runs without root.

Prerequisites (from Docker's rootless documentation):

- `newuidmap` and `newgidmap` (usually from the `uidmap` package).
- At least 65,536 **subordinate UIDs and GIDs** for the user in `/etc/subuid` and `/etc/subgid`. A subordinate ID range is a block of extra UIDs an unprivileged user may map into a user namespace, so root in the container can be, for example, UID 231072 on the host.

Install as a normal user, after installing Docker's packages that include `docker-ce-rootless-extras`:

```bash
dockerd-rootless-setuptool.sh install
```

- Sets up a per-user systemd service for the daemon and creates a Docker CLI **context** named `rootless`.
- If a system-wide root daemon is running, disable it first, or the script needs `--force`.

Verify with `docker info`. It should show the `rootless` security option and context. Some tools need `DOCKER_HOST` set to the per-user socket, for example `export DOCKER_HOST=unix:///run/user/1000/docker.sock` (1000 is the user's UID).

Trade-offs: the rootless page fetched did not list them, so check Docker's "known limitations" section. From general Docker documentation: binding ports below 1024 needs extra sysctl configuration, some networking features and storage drivers behave differently, user-space networking can be slower, and cgroup resource limits need cgroup v2 with delegation. Rootless suits developer machines, CI runners and multi-tenant hosts.

### 10.4.2 User namespace remapping (`userns-remap`): the daemon is root, container-root is not

The daemon stays root, but root inside every container maps to a high, unprivileged UID on the host. A process that escapes has no real privileges on the host. The mapping lives in `/etc/subuid` and `/etc/subgid`, with entries like `testuser:231072:65536` (user, starting host UID, count).

Enable it in `/etc/docker/daemon.json`:

```json
{
  "userns-remap": "default"
}
```

- `"default"`: Docker creates a user and group `dockremap` and their subordinate ranges. A custom user name can be used instead.
- Restart Docker: `sudo systemctl restart docker`.

Documented caveats:

- Enabling it hides existing images and containers (different storage location). Apply it to fresh hosts.
- Volumes need ownership arranged for the remapped IDs.
- It cannot be combined with host PID or network namespace sharing.
- `--privileged` needs `--userns=host`.
- Root in the container cannot `mknod`.
- A container can opt out with `--userns=host`.

| Mode | Daemon runs as | Container root maps to | Best for |
|---|---|---|---|
| Default | root | host root (UID 0), with restrictions | Convenience |
| `userns-remap` | root | unprivileged high UID | Servers where the daemon is controlled and escape mitigation is wanted |
| Rootless | unprivileged user | unprivileged UID in a user namespace | Dev machines, CI, shared hosts |

---

## 10.5 Capabilities: slicing up "root"

Linux splits root's powers into about 40 **capabilities**. Examples:

- `CAP_NET_BIND_SERVICE`: bind ports below 1024.
- `CAP_NET_RAW`: craft raw packets (for example `ping`).
- `CAP_SYS_ADMIN`: a very broad set of powers, often called "the new root".
- `CAP_SYS_PTRACE`: inspect other processes.
- `CAP_CHOWN`, `CAP_SETUID`.

Docker starts containers with a reduced default set that excludes dangerous ones such as `SYS_ADMIN`. Best practice is to drop everything and add back only what is proven necessary:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

- `--cap-drop ALL`: removes every capability.
- `--cap-add NET_BIND_SERVICE`: adds back only the right to bind low ports.

Most web apps, APIs and workers need no capabilities once they listen on a high port as non-root. To see a container's capabilities:

```bash
docker run --rm --cap-drop ALL alpine sh -c 'grep Cap /proc/self/status'
# CapInh: 0000000000000000
# CapPrm: 0000000000000000
# CapEff: 0000000000000000
# CapBnd: 0000000000000000
# CapAmb: 0000000000000000
```

- `/proc/self/status` shows the process's capability bitmasks. `CapEff` is the set effective now. All zeros means none.

Treat these capabilities with suspicion: `SYS_ADMIN`, `SYS_PTRACE`, `SYS_MODULE` (load kernel modules), `NET_ADMIN` (reconfigure networking; legitimately needed by VPN/firewall containers), `DAC_READ_SEARCH`, `DAC_OVERRIDE`.

---

## 10.6 `no-new-privileges`

```bash
docker run --security-opt no-new-privileges:true myimage
```

- Sets the kernel `no_new_privs` flag: no process can gain more privileges than its parent through setuid/setgid binaries or file capabilities. Without it, a compromised non-root process could run a setuid-root binary in the image (such as `su`) to become root in the container.
- Cost is low. It can break images that rely on setuid helpers (for example `sudo` in entrypoints). Fix the image rather than weakening the flag.

---

## 10.7 seccomp and AppArmor/SELinux

### 10.7.1 seccomp: filter system calls

A **system call (syscall)** is a request from a program to the kernel (`open`, `read`, `mount`, `reboot`). Linux has over 300. **seccomp** (secure computing mode) applies an allow-list or deny-list filter to a process. Calls outside the list fail or kill the process.

Docker applies a **default seccomp profile** automatically (`name=seccomp,profile=builtin` in `docker info` security options). It blocks a few dozen dangerous or obsolete syscalls (such as `kexec_load`, `reboot`, `swapon`) and allows normal software. Do not disable it:

```bash
docker run --security-opt seccomp=unconfined myimage     # do not use
```

- `seccomp=unconfined`: turns the filter off. Forum posts sometimes suggest it to fix a permission error. Identify the blocked syscall and allow only that.

Custom profile:

```bash
docker run --security-opt seccomp=/path/to/profile.json myimage
```

- Supplies a JSON profile listing allowed syscalls. Start from Docker's default profile (published in the Moby repository), remove what is not needed, and test. `strace` records which syscalls an app uses. Maintaining profiles is real work, so most teams use the default plus other layers.

### 10.7.2 AppArmor and SELinux: mandatory access control

**AppArmor** (Ubuntu/Debian/SUSE) and **SELinux** (Red Hat/Fedora) are **Linux Security Modules (LSMs)**: kernel policies that restrict which files, capabilities and network actions a program may use, even as root. Docker applies an AppArmor profile named `docker-default` on AppArmor hosts, and SELinux labels on SELinux hosts when enabled in the daemon.

- Do not disable them (`--security-opt apparmor=unconfined`, `--security-opt label=disable`) as a casual fix.
- On SELinux hosts, a bind-mounted directory may be unreadable until `:z` (shared label) or `:Z` (private label) is added, for example `-v ./data:/data:Z`. Use `:Z` only on dedicated directories because it relabels host files.

---

## 10.8 Read-only root filesystem and tmpfs

```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m myimage
```

- `--read-only`: mounts the root filesystem read-only. Malware cannot drop tools or modify the app during the attack.
- `--tmpfs /tmp:...`: mounts a **tmpfs** (an in-memory filesystem) at `/tmp` for scratch space.
  - `rw`: writable.
  - `noexec`: no running binaries from it.
  - `nosuid`: ignore setuid bits.
  - `size=64m`: cap its memory use.
- If the app fails with "Read-only file system", find the path it writes to and give that path a tmpfs or volume (for example `/var/run`, `/var/cache/nginx`, `/app/.cache`).

Expected error on a write attempt:

```
touch: /app/test.txt: Read-only file system
```

---

## 10.9 Resource limits: stop denial of service

Without limits, one runaway or malicious container can starve the host (**denial of service**, DoS). Set limits for every service.

```bash
docker run --memory 512m --memory-swap 512m --cpus 1.0 --pids-limit 200 --ulimit nofile=4096:8192 myimage
```

| Flag | Limits | Notes |
|---|---|---|
| `--memory 512m` (`-m`) | Hard RAM cap. Exceeding it triggers the **OOM killer** (out-of-memory), which kills the process (exit code 137) | Size from measurements (`docker stats`) |
| `--memory-swap 512m` | Total memory plus swap. Equal to `--memory` disables swap | Avoids swap thrashing |
| `--cpus 1.0` | Maximum CPU time; 1.0 is one core | Fractions allowed (`0.5`) |
| `--cpu-shares` | Relative weight under contention | Soft; not a cap |
| `--pids-limit 200` | Maximum processes/threads | Stops **fork bombs** (`:(){ :\|:& };:`) |
| `--ulimit nofile=4096:8192` | Open file descriptors (soft:hard) | Prevents descriptor exhaustion |
| `--storage-opt size=...` | Writable layer size | Only some storage drivers support it |
| `--log-opt max-size=10m --log-opt max-file=3` | Disk used by logs | The default `json-file` driver does not rotate and can fill the disk |

Watch usage with `docker stats`:

```
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         PIDS
a1b2c3d4e5f6   api       0.35%     120MiB / 512MiB     23.4%     1.2kB / 648B    14
```

In Compose use `deploy.resources.limits` and `pids_limit`, or the service keys `mem_limit`, `cpus`, `pids_limit` (see 10.20).

---

## 10.10 `--privileged`: don't

`docker run --privileged`:

- grants all capabilities,
- effectively disables seccomp and AppArmor/SELinux confinement,
- gives access to all host devices (`/dev/*`),
- lets the container mount filesystems and manipulate the kernel.

A privileged container is effectively root on the host. A known escape is mounting the host's disk device (for example `/dev/sda1`) and reading or writing everything on it.

| Reason for using `--privileged` | Alternative |
|---|---|
| Access one device (USB serial, GPU, `/dev/kvm`, `/dev/fuse`) | `--device /dev/ttyUSB0`, `--gpus all`, `--device /dev/kvm`, with `--group-add` for permissions |
| Run Docker inside Docker | Rootless BuildKit, Kaniko, Buildah, Sysbox runtime, or a disposable CI VM |
| Change one kernel setting | `--sysctl net.ipv4.ip_forward=1` (only namespaced sysctls are allowed) |
| Network tools (`ping`, raw sockets, tcpdump) | `--cap-add NET_RAW` or `NET_ADMIN`, whichever is needed |
| Fix a permission error | Find the denied operation (`dmesg`, `strace`, AppArmor logs) and grant the narrowest permission |
| Run `systemd` in a container | Use a purpose-built approach (specialized runtimes) |

If it is unavoidable (a one-off helper such as QEMU binfmt registration for cross-builds, as vLLM's docs describe), run it in a throw-away container on a trusted machine, never as a long-running service.

Other dangerous options:

| Option | Risk |
|---|---|
| `--pid=host` | See and signal all host processes |
| `--network=host` | Shares the host network stack, bypasses isolation, binds to any host port |
| `--ipc=host` | Shares host shared memory (sometimes needed for ML; see [Chapter 8](08-docker-for-ai-and-ml.md)) |
| `--userns=host` | Disables user-namespace remapping for the container |
| `-v /:/host`, `-v /etc:/etc`, `-v /var/run/docker.sock:...` | Host filesystem or daemon control |
| `--cap-add SYS_ADMIN` | Near-root |
| `--security-opt seccomp=unconfined`, `apparmor=unconfined` | Removes kernel guard rails |

---

## 10.11 Network exposure

### 10.11.1 Published ports are public unless restricted

`-p 8080:8080` publishes container port 8080 on all host interfaces (0.0.0.0). On a host with a public IP, the internet can reach it. Bind to loopback when the service is local-only or sits behind a reverse proxy on the same host:

```bash
docker run -p 127.0.0.1:8080:8080 myimage
```

- Format is `hostIP:hostPort:containerPort`. Only processes on the host can connect.

In Compose:

```yaml
ports:
  - "127.0.0.1:5432:5432"
```

If a container only talks to other containers, do not publish it. Containers on the same user-defined network reach each other by service name, so databases and caches usually need no `ports:` entry.

### 10.11.2 Docker bypasses ufw

With **ufw** (Uncomplicated Firewall) on Ubuntu set to deny all incoming traffic, `docker run -p 5432:5432 postgres` can still be reachable from the internet. Per Docker's documentation, Docker routes container traffic in the `nat` table, so traffic to published ports is diverted before it reaches the `INPUT`/`OUTPUT` chains where ufw's rules apply. ufw rules are effectively bypassed for published ports. Also per the docs: Docker's firewall rules are created for bridge networks only, and Docker sets the IP-forwarding policy to drop so the host does not become a router.

| Fix | Notes |
|---|---|
| Bind published ports to `127.0.0.1` (or a specific private IP) | Simplest and most reliable |
| Use a reverse proxy (Caddy, nginx, Traefik) as the only published container | Standard architecture |
| Cloud firewall / security groups in front of the host | Independent of host iptables. Always use these |
| Add rules in the `DOCKER-USER` iptables chain | Docker evaluates this chain first for forwarded traffic and never overwrites it. Several guides report the match happens after NAT translation, so rules must match the container port, not the published host port. Test with a scan from another machine |
| Do not set `"iptables": false` | Docker's docs warn it is likely to break container networking |
| nftables backend | Docker supports an nftables backend via the `firewall-backend` daemon option (marked experimental in the docs read; third-party sources say there is no `DOCKER-USER` chain there). Verify against your Docker version's release notes |
| Loopback packet drop rule | Unverified: one third-party guide says Engine 28 adds a rule that drops packets to loopback-published ports arriving on other interfaces. Not confirmed in Docker's docs; test from another host |

Test exposure from another machine: `nmap -p 1-65535 <server-ip>` or `nc -vz <server-ip> 5432`.

### 10.11.3 Other network practices

- **User-defined bridge networks** (Compose default) isolate stacks. The default `bridge` network lets all containers talk.
- **Internal networks** for backends: `networks: backend: {internal: true}` gives containers no route to the outside.
- **Egress control**: outbound connections are how data leaves a compromised app. Use proxy allow-lists, firewall rules, Kubernetes network policies, or `--network none` for batch jobs that need no network.
- **TLS**: terminate TLS at a reverse proxy. Do not send credentials in clear text on shared networks.
- **Do not use `--network host`** unless the exposure is understood.
- Services that default to no authentication (Ollama's API, Docker Model Runner's API (documented as unauthenticated), older Redis, Jupyter without a token, old Elasticsearch and MongoDB) must never be on a public interface.

---

## 10.12 The image supply chain

Every image is code from someone else. Treat images like dependencies: choose, pin, verify, minimize and update.

### 10.12.1 Choose trusted sources

| Docker Hub badge | Meaning |
|---|---|
| **Docker Official Image** | Curated images (ubuntu, python, postgres, nginx) maintained with Docker, reviewed, with documented build recipes and regular updates |
| **Verified Publisher** | Commercial publisher vetted by Docker (for example NVIDIA, Grafana, HashiCorp) |
| **Docker-Sponsored Open Source** | Open-source projects sponsored by Docker |
| (no badge) | Anyone can publish anything. Read the Dockerfile/source, check download counts and maintenance, prefer building yourself |

Typo-squatting (`pyhton`, `nginx-latest`) and images laced with miners are real threats.

### 10.12.2 Pin versions, then pin digests

A **tag** (such as `python:3.12-slim`) is a movable label. A **digest** is a cryptographic hash of the exact image content and is immutable.

```dockerfile
FROM python:3.12-slim@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

- `@sha256:...`: pins exactly one image. The digest shown is a placeholder.

Get a digest:

```bash
docker buildx imagetools inspect python:3.12-slim
# Name:      docker.io/library/python:3.12-slim
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:...
```

- Prints the **index** digest, which covers all architectures and is recommended for portable pins.

Keep digests updated with **Dependabot** or **Renovate** (bots that open pull requests when a new digest appears). Pinning without updating freezes vulnerabilities in place.

The same applies to CI: the 2026 Trivy incident (10.13.4) showed that mutable tags on GitHub Actions can be hijacked. Pin actions to full commit SHAs.

### 10.12.3 Minimal images shrink the attack surface

Every package in an image is a possible vulnerability and a possible attacker tool (shells, `curl`, package managers).

| Option | Description | Pros | Cons |
|---|---|---|---|
| `*-slim` (Debian slim) | Debian with fewer packages | Easy, glibc-compatible | Still has a shell and apt |
| **Alpine** | Small distribution using musl libc and BusyBox | Small | musl incompatibilities (some Python wheels, DNS edge cases). Fine for Go/static apps |
| **Distroless** (Google) | Only the runtime and its libraries; no shell or package manager | Few CVEs | Harder to debug (use `docker debug` / `kubectl debug`) |
| **`scratch`** | Empty image holding one static binary | Minimum size | Static Go/Rust binaries only; copy CA certificates and timezone data if needed |
| **Chainguard images** | Minimal, frequently rebuilt, with SBOMs and signatures (commercial/free tiers vary; check current terms) | Low CVE counts | Vendor-specific |
| **Docker Hardened Images (DHI)** | See 10.12.4 | Free Community tier | Registry login required |

Multi-stage builds ([Chapter 4](04-dockerfile-deep-dive.md)) compile in a large image and ship the result in a minimal one:

```dockerfile
FROM golang:1.23 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./cmd/app

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=build /out/app /app
USER nonroot
ENTRYPOINT ["/app"]
```

- `CGO_ENABLED=0`: builds a static Go binary (no C library needed).
- `distroless/static-debian12:nonroot`: contains little more than CA certificates and a non-root user. The tag is an example; check the distroless project for current tags.
- `COPY --from=build`: copies only the binary from the builder stage.

### 10.12.4 Docker Hardened Images (DHI)

Source: Docker's documentation (<https://docs.docker.com/dhi/>) and third-party coverage from December 2025.

- **What**: minimal, production-oriented images (plus Helm charts and system packages) maintained by Docker, aiming at near-zero known CVEs.
- **License**: Docker states the core features and open-source catalog are free under **Apache 2.0** in all tiers. Third-party coverage dates the free release to December 2025, with images pulled from `dhi.io` after a Docker account login. Unverified: the overview page did not show the registry name or pull commands. Check Docker's DHI "get started" page.
- **Features (all tiers)**: distroless variants (Docker claims up to 95% attack-surface reduction), non-root by default, signed source-built packages, SLSA Build Level 3 provenance, signed SBOMs, VEX statements. Alpine and Debian bases, glibc and musl variants.

| Tier | Includes |
|---|---|
| Community | Free; core hardened images; no FIPS/STIG variants, no CVE-fix SLA, no customization |
| Select | Adds FIPS/STIG-ready variants, 7-day SLA for critical CVE fixes, up to 5 customizations |
| Enterprise | Unlimited customizations, hardened system packages repository, optional extended lifecycle support |

The docs say dev and runtime variants exist. Unverified: exact naming. Illustrative usage (confirm names in the DHI catalog; paths and tags are placeholders):

```dockerfile
FROM dhi.io/python:3.12-dev AS build
# install dependencies...
FROM dhi.io/python:3.12 AS runtime
COPY --from=build /app /app
CMD ["python", "/app/main.py"]
```

- Build in the `-dev` variant (has tools). Ship the runtime variant (no shell or package manager, non-root).
- Run `docker login dhi.io` first.

Use Official Images for convenience and learning. Use hardened or minimal images for production where CVE counts and audit requirements matter.

### 10.12.5 Safer Dockerfile habits

| Habit | Reason |
|---|---|
| Never `curl https://example.com/install.sh \| bash` in a `RUN` | Executes whatever the server returns, unverified. Download a pinned version, verify its checksum/signature, then install |
| Verify downloads | `echo "<sha256>  file.tgz" \| sha256sum -c -` before extracting |
| Prefer `COPY` over `ADD` | `ADD` can fetch URLs and auto-extract archives |
| `apt-get install --no-install-recommends`, clean lists in the same layer | Fewer packages, smaller image |
| Do not install `sudo`, `ssh`, compilers, `curl`/`wget` in runtime images unless required | Fewer attacker tools |
| Do not `chmod 777` | Over-broad permissions |
| Add a `HEALTHCHECK` | Detects hung or crashed apps |
| Lint with **hadolint** | Catches common anti-patterns |
| Keep `.dockerignore` strict | Keeps `.env`, keys and `.git` out of the build context and layers |

Checksum-verified download:

```dockerfile
ARG TOOL_VERSION=1.2.3
ARG TOOL_SHA256=<64-hex-digits-from-the-vendor-release-page>
RUN curl -fsSL -o /tmp/tool.tgz "https://example.com/tool-${TOOL_VERSION}-linux-amd64.tgz" \
 && echo "${TOOL_SHA256}  /tmp/tool.tgz" | sha256sum -c - \
 && tar -xzf /tmp/tool.tgz -C /usr/local/bin tool \
 && rm /tmp/tool.tgz
```

- `ARG`: holds the version and expected hash. These are not secrets, so `ARG` is acceptable.
- `sha256sum -c -`: reads "hash  filename" from stdin and verifies the file. A mismatch returns non-zero and fails the build.
- `&&` chain: stops at the first failure.

---

## 10.13 Vulnerability scanning

A **vulnerability scanner** inventories the software packages in an image (OS packages such as `openssl`, language libraries such as `requests`) and compares versions with databases of known flaws. Flaws are identified by **CVE** IDs (Common Vulnerabilities and Exposures, for example `CVE-2024-12345`) and rated LOW, MEDIUM, HIGH or CRITICAL, usually from **CVSS** (Common Vulnerability Scoring System, 0-10).

Scanning finds known problems in installed software. It does not find unknown bugs in your own code, misconfigurations or secrets (some scanners also check those).

### 10.13.1 Trivy

**Trivy** (Aqua Security) is a widely used open-source scanner.

```bash
trivy image python:3.12-slim
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 myorg/api:1.4.2
trivy image --format json --output report.json myorg/api:1.4.2
trivy fs .
trivy config .
```

| Command / flag | Meaning |
|---|---|
| `trivy image <ref>` | Scans an image (pulls it if needed) |
| `--severity HIGH,CRITICAL` | Reports only these severities |
| `--ignore-unfixed` | Hides vulnerabilities with no available fix |
| `--exit-code 1` | Exits 1 if any finding remains, failing CI. By default Trivy exits 0 even with findings |
| `--format json --output report.json` | Machine-readable report |
| `trivy fs .` | Scans project files (lock files, secrets) |
| `trivy config .` | Scans Dockerfiles, Kubernetes and Terraform for misconfigurations |
| `--vex <file>` | Applies VEX statements. Unverified: reported by one source; see Trivy's docs |

Run Trivy as a container:

```bash
docker run --rm -v trivy-cache:/root/.cache/ \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  aquasec/trivy:<pinned-version> image myorg/api:1.4.2
```

- This mounts the Docker socket; `:ro` does not protect it (see 10.2). A safer alternative is scanning a tarball: `docker save myorg/api:1.4.2 -o api.tar`, then `trivy image --input api.tar`.
- `<pinned-version>`: pin a specific tag, not `latest` (see 10.13.4).

Output is a table per target with columns Library, Vulnerability, Severity, Status, Installed Version, Fixed Version and Title.

### 10.13.2 Docker Scout

**Docker Scout** is Docker's analysis service. It builds an inventory of an image's components (an SBOM), compares it with a continuously updated vulnerability database, and shows results in the CLI, Docker Hub and the Scout dashboard (per Docker's docs). It is integrated into the `docker` CLI (`docker scout version` was confirmed to run locally).

```bash
docker scout quickview myorg/api:1.4.2
docker scout cves myorg/api:1.4.2
docker scout cves --only-severity critical,high --only-fixed --exit-code myorg/api:1.4.2
docker scout recommendations myorg/api:1.4.2
docker scout compare --to myorg/api:1.4.1 myorg/api:1.4.2
```

| Command / flag | Meaning |
|---|---|
| `quickview` | One-screen summary: vulnerability counts for the image and its base image, with suggestions |
| `cves` | Detailed CVE list. Also accepts directories, `docker save` tarballs and OCI layouts via prefixes such as `archive://`, `oci-dir://`, `fs://`, `sbom://` |
| `--only-severity critical,high` | Filters by severity |
| `--only-fixed` / `--only-unfixed` | Shows only fixable (or only unfixable) issues |
| `--ignore-base` | Excludes CVEs from the base image |
| `--format sarif` | Outputs SARIF, a JSON report format that GitHub code scanning reads. Other formats: spdx, gitlab, markdown |
| `--exit-code` (`-e`) | Exits with code 2 when vulnerabilities are found, to fail CI |
| `recommendations` | Suggests better base-image tags |
| `compare` | Shows differences between two images (new/fixed vulnerabilities and packages) |

Unverified: `quickview`, `recommendations` and `compare` come from general knowledge of the Scout CLI. The reference page fetched covered `cves` in detail only. Run `docker scout --help`. Scout's free versus paid boundaries change; check Docker's pricing page.

### 10.13.3 Grype (and Syft)

**Grype** (Anchore) is an open-source scanner. **Syft** is Anchore's SBOM generator. Typical flow:

```bash
syft myorg/api:1.4.2 -o spdx-json > sbom.json
grype sbom:./sbom.json
# or simply:
grype myorg/api:1.4.2
```

Unverified: this is from general knowledge; current Grype docs could not be retrieved. Check `grype --help`.

Two scanners are sensible because their databases and matching differ. Disagreements are informative.

### 10.13.4 Scanners are software too: the 2026 Trivy incident

In **March 2026**, Aqua Security's Trivy ecosystem suffered a **supply-chain compromise**. Per Aqua's advisory (GHSA-69fq-xp46-6x23) and several security vendors:

- Attackers (credited by some vendors to a group calling itself "TeamPCP") gained access through credentials not fully rotated after an earlier incident.
- They rewrote version tags of `aquasecurity/trivy-action` (75 or 76 of 77 tags, per sources) and `setup-trivy` to point to malicious commits that harvested CI/CD secrets from runner memory.
- A malicious Trivy binary (v0.69.4) and Docker Hub images (v0.69.5, v0.69.6) were also published. Exposure windows were hours for the Action and binary, and until about March 23 for the images.
- Only `trivy-action` v0.35.0 (protected by **immutable releases**) was reported unaffected.
- Aqua advised pinned known-good versions, pinning actions to full commit SHAs, auditing workflow logs for the exposure window, and rotating secrets that ran in affected pipelines.

Sources differ on exact counts and times; consult the primary advisory.

Lessons:

1. Pin by immutable identifier: commit SHA for Actions, digest for images.
2. Give CI jobs least privilege: short-lived credentials (OIDC federation), not stored long-lived secrets.
3. Scanners run with your secrets in scope and need the same scrutiny as any dependency.
4. Prepare a rotation runbook in advance (10.16.5).

### 10.13.5 How to triage scan results

A scan of a typical image can produce dozens or hundreds of findings. Triage in this order:

```
1. Is it fixable?            No fix -> record it, apply mitigations, re-check later (--ignore-unfixed for gating).
2. Where does it come from?  Base image OS package -> update/rebuild the base (often one change fixes many).
                             Your app dependency  -> upgrade the dependency/lock file.
3. How severe, really?       CRITICAL/HIGH first. CVSS is generic; context matters.
4. Is it reachable?          Is the vulnerable package/function actually used? Is the service exposed?
                             (A bug in a build tool left in the runtime image is less risky than one in your web framework.)
5. Is there a VEX?           Vendor VEX statements say "not affected" for specific products.
6. Decide and record:        Fix now / fix soon / accept with expiry date / false positive.
```

- **Rebuild regularly.** Rebuild images weekly to pick up patched base images, even if the code did not change.
- **Use a smaller base** to remove whole classes of findings (shells, curl, Perl).
- **VEX (Vulnerability Exploitability eXchange)**: a machine-readable supplier statement such as "CVE-X is not affected in this product because the vulnerable code path is not present". Used to suppress false positives. Docker Hardened Images ship VEX statements (per Docker's docs), and Trivy and Scout can consume them.
- **Ignore files**: Trivy's `.trivyignore` lists accepted CVEs. Add a comment with the reason and an expiry date.
- **Gate CI** on fixable HIGH/CRITICAL and report the rest. Gating on unfixable issues teaches developers to bypass the gate.
- **Scan continuously.** New CVEs are published for images shipped months ago. Registries (ECR, Harbor, Docker Hub with Scout) can re-scan stored images.

---

## 10.14 SBOMs and provenance attestations

### 10.14.1 SBOM

An **SBOM (Software Bill of Materials)** is a machine-readable list of every package and version in an image. Common formats: **SPDX** and **CycloneDX**. When a major vulnerability appears (for example Log4Shell), SBOMs can be queried ("which images contain this library?"). Regulators and customers increasingly require them.

### 10.14.2 Provenance

**Provenance** is a signed or signable statement of how an image was built: source repository and commit, builder, build arguments, time. The common standard is **SLSA** (Supply-chain Levels for Software Artifacts, pronounced "salsa"). It allows verifying that an image came from your CI at a given commit and not from a laptop.

### 10.14.3 Creating them with BuildKit

From Docker's build attestation docs (<https://docs.docker.com/build/metadata/attestations/>):

```bash
docker buildx build --sbom=true --provenance=mode=max -t myorg/api:1.4.2 --push .
```

| Flag | Meaning |
|---|---|
| `--sbom=true` | Generates an SBOM attestation (default generator is BuildKit's Syft-based scanner) and attaches it to the image in the registry |
| `--provenance=mode=max` | Generates maximum-detail SLSA provenance. `min` is on by default in current BuildKit; `max` adds the build definition, the base64-encoded Dockerfile and source maps |
| `--push` | Pushes to a registry. Attestations are stored as extra manifests alongside the image |

Warnings from the docs:

- `mode=max` exposes the values of build arguments in the provenance. Never pass secrets via `--build-arg`. Secret mounts "don't leak outside of the build and are never included in provenance attestations".
- The classic (non-containerd) Docker image store does not support attestations. Use the containerd image store or a `docker-container` builder. On the reference machine, `docker info` reports the `io.containerd.snapshotter.v1` driver type, so attestations are supported; check yours with `docker info`. To opt out, use `--provenance=false --sbom=false`.
- Multi-platform and attested images are image indexes with extra attestation manifests. Registries show these as "unknown/unknown" platform entries; they are attestations, not broken images.

Inspect them:

```bash
docker buildx imagetools inspect myorg/api:1.4.2 --format '{{ json .SBOM }}'
docker buildx imagetools inspect myorg/api:1.4.2 --format '{{ json .Provenance }}'
docker scout sbom myorg/api:1.4.2
```

- `imagetools inspect --format`: prints the attestation payloads as JSON.
- `docker scout sbom`: prints an SBOM for an image (from attestation, or generated on the fly).

In GitHub Actions, `docker/build-push-action` accepts `sbom: true` and `provenance: mode=max` (see the CI sketch in [Chapter 8](08-docker-for-ai-and-ml.md)). Attestations are verified cryptographically with `cosign verify-attestation` (10.15).

---

## 10.15 Image signing: Cosign (and others)

Scanning shows what is in an image. **Signing** shows who published it and that it has not been modified. Without verification at deploy time, anyone who can push to the registry or hijack a tag can run code in production.

### 10.15.1 Cosign and Sigstore

**Sigstore** is an open-source project and public infrastructure for signing software. **Cosign** is its command-line tool for containers. It supports **keyless signing**:

1. The signer authenticates with an identity provider (OIDC: GitHub, Google, Microsoft).
2. Sigstore's certificate authority (**Fulcio**) issues a short-lived certificate (minutes) tying the signature to that identity.
3. The signature is recorded in a public **transparency log** (**Rekor**), so it cannot be quietly forged or backdated.

Sign (from the Sigstore docs; keyless means omit `--key`):

```bash
cosign sign myorg/api@sha256:<digest>
```

- Prompts for authentication through an OIDC provider in a browser. In CI it uses the workflow's identity token.
- Sign by digest, not by tag. Signing `:latest` signs whatever it pointed to at that moment.

GitHub Actions sketch:

```yaml
permissions:
  contents: read
  packages: write
  id-token: write        # lets the job request an OIDC identity token for keyless signing

steps:
  - uses: sigstore/cosign-installer@<pin-to-commit-sha>
  - run: cosign sign --yes ghcr.io/myorg/api@${{ steps.build.outputs.digest }}
```

- `id-token: write`: the permission that allows keyless signing.
- `--yes`: skips the interactive confirmation.
- `steps.build.outputs.digest`: digest output from the earlier build-push step (the name depends on your step id).
- Pin the installer action to a commit SHA (lesson from 10.13.4).

Verify (shape from the Sigstore docs):

```bash
cosign verify \
  --certificate-identity "https://github.com/myorg/api/.github/workflows/release.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/api@sha256:<digest>
```

| Flag | Meaning |
|---|---|
| `--certificate-identity` | Who must have signed. Here, a specific workflow file on `main`. Use the exact identity in production; `--certificate-identity-regexp` with loose patterns weakens the policy |
| `--certificate-oidc-issuer` | Which identity provider vouches for that identity (GitHub Actions' token issuer here) |

Verify attestations (SBOM/provenance) with `cosign verify-attestation` and a `--policy`. Check `cosign verify --help` for your version.

**Cosign version advisory.** GHSA-fx35-mq7g-6g98 reported that with a legacy JSON bundle, Cosign could skip the `--certificate-identity` and `--certificate-oidc-issuer` checks, so verification succeeded for any validly signed artifact. The advisory says the protobuf bundle path is not affected and recommends upgrading (v3.1.3 or v2.6.5, as reported in search snippets; unverified) and migrating legacy bundles. Cosign v3 defaults to the standardized bundle format. Keep Cosign updated and test verification with a negative case: a signature from the wrong identity must fail.

### 10.15.2 Other signing systems

| System | Notes |
|---|---|
| **Notation (Notary Project, "Notary v2")** | CNCF-adjacent signing standard using X.509 certificates, supported by several cloud registries |
| **Docker Content Trust (Notary v1)** | Older Docker-specific signing (`DOCKER_CONTENT_TRUST=1`). Legacy. Unverified: current roadmap status not checked. Chapter 11 notes it was removed from the Docker CLI in Engine 29 per the release notes |
| **Registry-native signing/attestation** (ECR, ACR, GAR, GHCR) | Cloud-specific; check provider docs |

### 10.15.3 Enforce verification at deploy time

Signing has no effect unless verified. In Kubernetes, use an **admission controller** (Sigstore **policy-controller**, **Kyverno**, **Connaisseur**, or cloud-native policies) to reject unsigned or wrongly signed images. Elsewhere, have the deploy script run `cosign verify` and abort on failure.

---

## 10.16 Secrets management

A **secret** is anything that grants access: passwords, API keys, tokens, private keys, certificates, signing keystores, database URLs with embedded passwords.

### 10.16.1 Where secrets leak

| Leak path | Why it is dangerous |
|---|---|
| `ENV SECRET=...` in a Dockerfile | Stored in image metadata. Visible with `docker inspect` and `docker history`. Copied into every container |
| `ARG SECRET=...` used in a `RUN` | Build args are recorded in image history/metadata (and in max-mode provenance) |
| `COPY .env .` or any `COPY` of a credentials file | The file stays in a layer even if a later `RUN rm` deletes it, because each layer is an immutable snapshot that can be extracted |
| Hard-coded in source and committed | Git history is permanent. Forks and clones keep it |
| Runtime environment variables | Visible via `docker inspect` to anyone with Docker access, and in `/proc/<pid>/environ`, crash dumps, debug pages (`phpinfo()`) and logs that print the environment. Inherited by child processes |
| Logs and error messages | Exceptions that print configuration or URLs with passwords |
| CI logs | `echo $SECRET` or verbose tooling output |

Demonstration of the first leak with a harmless value:

```bash
docker run --rm -e DEMO_TOKEN=not-a-real-secret alpine sleep 60 &
docker inspect --format '{{.Config.Env}}' $(docker ps -q --filter ancestor=alpine | head -1)
# [DEMO_TOKEN=not-a-real-secret PATH=/usr/local/sbin:...]
```

- Anyone who can run `docker inspect` (anyone with socket access) can read runtime env vars.
- `docker history --no-trunc <image>` shows every `RUN`/`ENV`/`ARG` instruction text.

### 10.16.2 Build-time secrets: BuildKit secret mounts

When a build needs a credential (private package registry token, gated model download), use a **secret mount**. It is available during that single `RUN` step and is never stored in a layer.

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install --no-cache-dir -r requirements.txt
```

- `--mount=type=secret,id=pip_token`: exposes the secret as the file `/run/secrets/pip_token` for this instruction only.
- `$(cat ...)`: reads it into a variable scoped to this command.
- pip prints index URLs in some error messages. For stricter hygiene, mount a netrc or credential-helper file as the secret. The index URL is a placeholder.

Build:

```bash
docker build --secret id=pip_token,src=./pip_token.txt -t myimage .
# or from an environment variable:
docker build --secret id=pip_token,env=PIP_TOKEN -t myimage .
```

- `src=`: reads from a file.
- `env=`: reads from an environment variable of the shell running the build.

For **SSH keys** (cloning a private repo during a build): `RUN --mount=type=ssh git clone git@github.com:org/private.git` with `docker build --ssh default .`. This forwards the SSH agent without copying the key.

Multi-stage builds do not protect secrets placed in an earlier stage if that stage is ever pushed or cached. Use secret mounts instead.

### 10.16.3 Runtime secrets

| Method | How | Pros | Cons |
|---|---|---|---|
| Environment variable | `-e DB_PASSWORD` / Compose `environment` | Simple; every language reads it | Visible in `docker inspect`, `/proc`, logs; inherited by children. Acceptable for low-value dev secrets |
| File mounted into the container | `-v ./secrets/db_password:/run/secrets/db_password:ro`, or Compose/Swarm `secrets:` | Not in `docker inspect` env; file permissions settable; works with the `*_FILE` convention | Slightly more code |
| Compose `secrets:` | Top-level `secrets:` mapped to a `file:` (or `environment:`); service lists it; mounted at `/run/secrets/<name>` | Standard, simple | In plain Compose it is a file bind mount, not encrypted storage |
| Docker Swarm secrets | `docker secret create`; encrypted at rest and in transit in the Swarm Raft store; mounted in memory (tmpfs) | Real secret store | Swarm only |
| Kubernetes Secrets | `kind: Secret` mounted as files or env | Native | By default base64-encoded, not encrypted, in etcd. Enable encryption at rest and tight RBAC; consider External Secrets Operator, Sealed Secrets, SOPS or the Secrets Store CSI driver |
| Secret managers | HashiCorp Vault, AWS Secrets Manager / SSM Parameter Store, GCP Secret Manager, Azure Key Vault. The app or a sidecar fetches at start using the workload's cloud identity (IAM roles, workload identity) | Central audit, rotation, short-lived dynamic credentials | More infrastructure |

Compose secrets example:

```yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

- The official Postgres image supports the `*_FILE` convention: `POSTGRES_PASSWORD_FILE` makes it read the password from that file instead of an env var.
- `./secrets/db_password.txt` must be git-ignored and have tight permissions (`chmod 600`).

Preference order: short-lived credentials from an identity-based system (cloud IAM, Vault dynamic secrets, OIDC) > secret manager > mounted files > env vars > anything baked into images (never).

### 10.16.4 `.env` files and Compose

Compose reads `.env` for variable substitution in the Compose file (for example `${POSTGRES_PASSWORD}`). `env_file:` passes variables to containers.

- Add `.env` to `.gitignore` and `.dockerignore`.
- Commit a `.env.example` with fake values.
- Run `git ls-files | grep -i env` to check no `.env` was committed.
- Use pre-commit secret scanners (**gitleaks**, **trufflehog**) and GitHub **push protection**.

### 10.16.5 What to do if a secret leaked

> **If a secret has been exposed beyond its intended holders, treat it as compromised and rotate it. Deleting or hiding it does not undo the leak.**

Rewriting history is not enough because the secret may already be cloned, forked, cached by CI, scraped by bots (public repositories are scanned within minutes), present in a pulled image layer, or in logs.

Runbook:

1. **Revoke and rotate first.** Generate a new credential, deploy it, then invalidate the old one at the provider (cloud console, GitHub token settings, database `ALTER USER`). For signing keystores and other credentials that are hard to re-issue, follow the provider's key-compromise procedure. A leaked Android upload key can be reset through the Play Console; an app-signing key has more serious implications.
2. **Determine what it could access** and review access logs for the exposure window (cloud audit logs, database logs, provider dashboards).
3. **Remove the source of the leak.**
   - Git: remove the file in a new commit and purge history (`git filter-repo` or BFG Repo-Cleaner), force-push, and have collaborators re-clone. For public repositories, ask the host (for example GitHub support) to purge cached views and PR refs. This is cleanup, not a substitute for rotation.
   - Image/registry: delete the affected tags and digests, rebuild without the secret, redeploy. Anyone who pulled the old image still has it.
   - CI logs: delete logs and artifacts, and rotate anyway.
4. **Prevent recurrence**: secret scanning in pre-commit and CI, `.dockerignore`, BuildKit secret mounts, short-lived credentials, least-privilege scopes.
5. **Write a short post-incident note**: what, when, how found, what changed.

In the Trivy incident, the attackers' foothold persisted because an earlier credential rotation was incomplete. Rotate everything that could have been exposed.

---

## 10.17 Registry security

A **registry** stores and serves images (Docker Hub, GitHub Container Registry (GHCR), Amazon ECR, Google Artifact Registry, Azure Container Registry, Harbor, a self-hosted `registry:2`).

| Practice | Details |
|---|---|
| Private repositories for non-public images | Some services default to public. Check visibility settings |
| Scoped access tokens, not passwords | Docker Hub access tokens with read-only or read-write scope; GHCR fine-grained tokens; cloud IAM roles. Use separate tokens per CI pipeline so one can be revoked |
| MFA/SSO on registry accounts | A stolen password must not allow pushing `latest` |
| Least privilege | Build jobs push. Deploy hosts only pull |
| Short-lived credentials in CI | OIDC federation to cloud registries instead of stored keys |
| Immutable tags | Cloud registries (ECR and others) can forbid overwriting tags, so a published `1.4.2` cannot change |
| Rate limits | Docker Hub throttles pulls (historically about 100 per 6 hours for anonymous users per IP and 200 for authenticated free accounts; limits have been revised, so check current numbers). CI fleets behind one NAT IP hit this quickly. Authenticate pulls, use a **pull-through cache/mirror** (a registry that proxies and caches Docker Hub), or mirror critical base images into your own registry. Mirroring also protects against an upstream image or tag disappearing |
| Scanning on push | ECR/ACR/GAR/Harbor/Docker Hub (with Scout) can scan and re-scan stored images |
| Self-hosted registry | Always use TLS and authentication (`registry:2` has neither by default). Consider **Harbor** (RBAC, scanning, replication, retention, signature policies) |
| Retention and cleanup | Delete stale tags but keep deployed digests |
| Audit logs | Record who pushed and pulled what |
| Credentials in `~/.docker/config.json` | `docker login` stores credentials there, by default possibly as base64 (not encrypted). Use a **credential helper** (`osxkeychain`, `wincred`, `pass`, `secretservice`). Never put credentials in image names |

---

## 10.18 CIS Docker Benchmark and docker-bench-security

The **CIS Docker Benchmark** (Center for Internet Security) is a consensus checklist of hardening recommendations covering the host, daemon configuration, daemon files, images and build files, container runtime, security operations and Swarm. **docker-bench-security** is Docker's open-source script that checks a host against it automatically.

Run on a Linux Docker host:

```bash
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sudo sh docker-bench-security.sh
```

- Clones the project and runs the audit as root (needed to read daemon configs and audit rules). Read the script before running anything as root.

Output shape:

```
[INFO] 1 - Host Configuration
[WARN] 1.1.1 - Ensure a separate partition for containers has been created
[PASS] 1.1.3 - Ensure auditing is configured for the Docker daemon
...
[WARN] 5.4  - Ensure that privileged containers are not used
[WARN] 5.10 - Ensure that the memory usage for containers is limited
[PASS] 5.12 - Ensure that the container's root filesystem is mounted as read only
```

| Marker | Meaning |
|---|---|
| `[PASS]` | Check passed |
| `[WARN]` | Failed. Fix, or document the accepted risk |
| `[INFO]` | Informational; manual review needed |
| `[NOTE]` | Advisory remark |

Version note: the docker-bench-security README says its tests are based on **CIS Docker Benchmark v1.6.0** (project release v1.6.1 was the latest found). CIS's site lists a newer benchmark (1.9.0 in the snippet seen), so the script can lag. Use it as a baseline scan and consult the current CIS PDF. Aqua's separate Go-based `docker-bench` also exists and targets CIS 1.6.0. Fix privileged containers, missing limits, exposed sockets and daemon TLS first. Treat partition and auditing recommendations as production-hardening backlog.

---

## 10.19 Runtime security, logging and auditing

Prevention eventually fails. Detection shows when.

### 10.19.1 Falco

**Falco** (a CNCF project) watches kernel events (syscalls, through an eBPF probe or kernel module) and raises alerts when container behaviour matches **rules**. Typical rules: a shell spawned in a container, writes to `/etc` or `/bin`, reads of sensitive files, unexpected outbound connections, a container mounting the Docker socket or starting privileged, `exec` into production containers.

Rule shape (illustrative, adapted from the style of Falco's default rules; check Falco's docs for current syntax and macros):

```yaml
- rule: Shell in Container
  desc: A shell was spawned in a container
  condition: evt.type = execve and container.id != host and proc.name in (bash, sh, zsh)
  output: "Shell spawned in container (user=%user.name container=%container.name cmd=%proc.cmdline)"
  priority: WARNING
```

- `condition`: a filter expression over events.
- `output`: the alert text with fields filled in.
- `priority`: the severity.

Falco needs elevated access to observe the kernel (usually a privileged or specially-capable container, a host service, or a Kubernetes DaemonSet); see Falco's installation docs. Alerts typically go to a SIEM or chat through **Falcosidekick**.

Alternatives: **Tetragon** and **Tracee** (eBPF-based), cloud-provider runtime threat detection, and commercial CNAPP platforms.

### 10.19.2 Built-in tools for visibility and forensics

```bash
docker events --since 1h
docker diff <container>
docker logs --since 30m <container>
docker top <container>
docker stats --no-stream
docker inspect <container>
```

| Command | Use |
|---|---|
| `docker events` | Live stream of daemon events (container start/stop/exec/die, image pulls, `oom`). Pipe to logs or alerts |
| `docker diff <container>` | Lists files Added (A), Changed (C) or Deleted (D) in the container's writable layer versus the image. Shows what an attacker modified |
| `docker logs` | Application stdout/stderr |
| `docker top` | Processes in the container as the host sees them |
| `docker inspect` | Full config: mounts, env, capabilities, security options, network |
| `docker exec` | Debugging shell, and also what attackers use. Restrict and audit it in production |

### 10.19.3 Logging and audit practices

- Ship logs off the host (Loki, ELK/OpenSearch, CloudWatch, Datadog) with a logging driver (`--log-driver`) or an agent. Local logs vanish with the host or container.
- Rotate local logs (`max-size`, `max-file`).
- Use structured (JSON) logs with request IDs. Never log secrets or personal data.
- Audit the daemon: `auditd` rules for `/usr/bin/dockerd`, `/etc/docker`, `/var/lib/docker`, `docker.sock` and `containerd` (the CIS benchmark lists the exact rules).
- Alert on privileged container starts, images from unexpected registries, containers running as root, restarts/OOM kills, and `docker exec` into production.
- Keep time synchronized (NTP) and timestamps centralized.
- Set retention to match compliance needs.

---

## 10.20 Putting it together: a hardened `docker run` and Compose file

### 10.20.1 Hardened `docker run`

```bash
docker run -d \
  --name api \
  --user 10001:10001 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=64m \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --memory 512m --memory-swap 512m \
  --cpus 1.0 \
  --pids-limit 200 \
  --ulimit nofile=4096:8192 \
  --network app-net \
  -p 127.0.0.1:8000:8000 \
  --restart unless-stopped \
  --health-cmd "python -c \"import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/healthz')\"" \
  --health-interval 30s --health-retries 3 \
  --log-opt max-size=10m --log-opt max-file=3 \
  -v /run/secrets-host/api_key:/run/secrets/api_key:ro \
  -e API_KEY_FILE=/run/secrets/api_key \
  myorg/api@sha256:<digest>
```

| Flag | Purpose |
|---|---|
| `--user`, `--read-only`, `--tmpfs`, `--cap-drop`, `--security-opt`, `--memory`/`--memory-swap`, `--cpus`, `--pids-limit`, `--ulimit` | Described in 10.3 and 10.5 to 10.9 |
| `--network app-net` | Attaches to a user-defined network (create with `docker network create app-net`) |
| `-p 127.0.0.1:8000:8000` | Publishes on loopback only (10.11) |
| `--restart unless-stopped` | Restarts after crashes and reboots |
| `--health-cmd`, `--health-interval`, `--health-retries` | Health check so Docker can mark the container `unhealthy` |
| `--log-opt max-size/max-file` | Caps log disk usage |
| `-v ...:ro` + `-e API_KEY_FILE=...` | Delivers the secret as a read-only file instead of an env var. The host path is an example; keep it root-owned with `chmod 600` or an appropriate group |
| `myorg/api@sha256:<digest>` | Image pinned by digest (10.12.2) |

Absent on purpose: `--privileged`, `--network host`, the Docker socket, `seccomp=unconfined`, host mounts of sensitive paths. The default seccomp and AppArmor/SELinux profiles stay active.

Check the result:

```bash
docker inspect api --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}} CapDrop={{.HostConfig.CapDrop}} SecOpt={{.HostConfig.SecurityOpt}} PidsLimit={{.HostConfig.PidsLimit}}'
# User=10001:10001 ReadOnly=true CapDrop=[ALL] SecOpt=[no-new-privileges:true] PidsLimit=200
```

### 10.20.2 Hardened Compose file

```yaml
name: secure-app

services:
  proxy:
    image: caddy:2
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
    networks: [edge, backend]
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE]
    security_opt: ["no-new-privileges:true"]
    restart: unless-stopped

  api:
    image: myorg/api@sha256:<digest>
    user: "10001:10001"
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=64m
    cap_drop: [ALL]
    security_opt:
      - no-new-privileges:true
    mem_limit: 512m
    memswap_limit: 512m
    cpus: 1.0
    pids_limit: 200
    environment:
      DB_HOST: db
      DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    networks: [backend, data]
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/healthz')"]
      interval: 30s
      timeout: 3s
      retries: 3
    logging:
      driver: json-file
      options: {max-size: "10m", max-file: "3"}
    restart: unless-stopped

  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_DB: app
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [data]
    cap_drop: [ALL]
    cap_add: [CHOWN, SETUID, SETGID, FOWNER, DAC_OVERRIDE]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 1g
    pids_limit: 200
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 10s
      retries: 10
    restart: unless-stopped

networks:
  edge: {}
  backend:
    internal: true
  data:
    internal: true

volumes:
  caddy_data:
  pgdata:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

| Setting | Purpose |
|---|---|
| Only `proxy` has `ports:` | The reverse proxy is the only publicly reachable container. `api` and `db` publish nothing |
| `cap_add: [NET_BIND_SERVICE]` on the proxy | Allows binding ports 80/443 without root. Everything else is dropped. Some Caddy images set this up already; confirm for your tag |
| `internal: true` networks | `backend` and `data` have no route to the internet, so a compromised API cannot call out directly. If the API must call external services (such as a hosted LLM), add a non-internal network or an egress proxy |
| `proxy` on `edge`+`backend`, `api` on `backend`+`data`, `db` only on `data` | Containers communicate only if they share a network. The proxy reaches the API but not the database. The database is reachable only from the API (network segmentation) |
| `read_only`, `tmpfs`, `user`, `cap_drop`, `security_opt` | Same as the `docker run` version |
| `mem_limit`, `memswap_limit`, `cpus`, `pids_limit` | Resource caps as service keys (work with plain `docker compose up`; `deploy.resources.limits` is also supported by current Compose) |
| `db` with `cap_drop: ALL` plus a few `cap_add` | The official Postgres image starts as root to fix data-directory ownership, then drops to the `postgres` user, so it needs a few capabilities. Re-test if you add `user:` or the image changes. Start with none and add only what error messages require |
| `secrets` with `*_FILE` variables | No password in `environment` or `docker inspect` |
| `image: ...@sha256:<digest>` | Immutable pin (placeholder digest). `postgres:17` and `caddy:2` are tags for readability; pin digests in production |

Run with `docker compose up -d` and `docker compose ps`. A `Caddyfile` and a real API image are required; the digest and file contents here are placeholders.

Validate and view the resolved configuration:

```bash
docker compose config
```

- Prints the merged, variable-substituted configuration, or errors if the YAML is invalid.

---

## 10.21 Security checklist

**Host and daemon**
- [ ] Host OS, kernel and Docker Engine are patched; automatic security updates enabled.
- [ ] Docker socket is not mounted into application containers (or a socket proxy is used).
- [ ] Daemon is not exposed on unauthenticated TCP (2375); remote access uses SSH or mutual TLS.
- [ ] `docker` group membership is minimal and treated as root.
- [ ] Rootless mode or `userns-remap` considered or enabled where practical.
- [ ] `docker-bench-security` run; WARNs triaged.
- [ ] Hostile or multi-tenant code runs in VMs/microVMs/gVisor, not plain containers.

**Images and supply chain**
- [ ] Base images are Official/Verified/Hardened or built in-house; minimal (slim/distroless/DHI) where possible.
- [ ] Tags pinned; production references digests; a bot (Dependabot/Renovate) proposes updates.
- [ ] Multi-stage builds; no compilers, shells or package managers in final images where avoidable.
- [ ] No `curl | bash`; downloads verified by checksum or signature.
- [ ] `.dockerignore` excludes `.git`, `.env`, keys.
- [ ] Images scanned in CI and continuously in the registry; fixable HIGH/CRITICAL block merges; ignores have reasons and expiry.
- [ ] Images rebuilt on a schedule for patched bases.
- [ ] SBOM and provenance attestations generated; images signed (Cosign); deploy verifies signatures and identity.
- [ ] CI actions and tools pinned to commit SHAs/digests; CI uses short-lived OIDC credentials.

**Runtime configuration**
- [ ] Containers run as a non-root numeric UID.
- [ ] `--cap-drop ALL` plus minimal `--cap-add`.
- [ ] `no-new-privileges:true`.
- [ ] Default seccomp and AppArmor/SELinux profiles left enabled.
- [ ] `--read-only` root filesystem with tmpfs/volumes for writable paths.
- [ ] Memory, CPU, PIDs, ulimits and log-size limits set.
- [ ] No `--privileged`, `--network host`, `--pid host`, host root mounts.
- [ ] Health checks defined; restart policy set.

**Network**
- [ ] Only the reverse proxy publishes ports publicly; others have no `ports:` or bind `127.0.0.1`.
- [ ] Backends on `internal` networks; egress restricted where feasible.
- [ ] External exposure tested from another machine; cloud security groups/firewalls in place (do not rely on ufw alone).
- [ ] TLS wherever users or credentials are involved; unauthenticated services (Redis, Ollama, Jupyter, Model Runner) never public.

**Secrets**
- [ ] No secrets in Dockerfiles, `ENV`, `ARG`, images or git.
- [ ] Build-time secrets via BuildKit `--mount=type=secret` / `--ssh`.
- [ ] Runtime secrets from files or a secret manager; short-lived where possible.
- [ ] Secret scanning (pre-commit, CI, push protection) enabled.
- [ ] Rotation runbook exists and has been rehearsed.

**Registry and operations**
- [ ] Private registries, scoped tokens, MFA/SSO; push and pull permissions separated.
- [ ] Pull-through cache or mirror of critical base images; pulls authenticated.
- [ ] Logs shipped centrally; audit rules for the daemon; alerts on privileged starts and unexpected `exec`.
- [ ] Runtime detection (Falco/Tetragon or cloud equivalent) in production.
- [ ] Incident response plan covers "compromised image", "leaked secret", "compromised host".

---

## 10.22 Common anti-patterns

| Anti-pattern | Instead |
|---|---|
| Secrets in `ENV`/`ARG`/layers | Secret mounts at build time; files or secret managers at runtime; rotate anything leaked |
| `curl ... \| bash` in Dockerfiles | Pinned versions with checksums/signatures, or distro packages |
| `latest` tags | Version tags, then digests, with automated update PRs |
| Running as root | `USER <numeric uid>` |
| Mounting `docker.sock`, `--privileged`, host mounts | Socket proxy, rootless, specific `--device`/`--cap-add`/`--sysctl` |
| Publishing ports on `0.0.0.0` (ufw does not protect them) | Loopback binding, internal networks, reverse proxy |
| No resource limits | `--memory`, `--cpus`, `--pids-limit` |
| Large runtime images, SSH servers in containers | Multi-stage builds, minimal bases, `docker exec` with audit |
| Deleting images or rewriting history as secret remediation | Rotate the credential |
| Pulling random Hub images | Official/Verified/Hardened images; read the source; scan |
| Disabling seccomp/AppArmor to silence an error | Diagnose the specific denial |
| Scanning once | Continuous scanning and scheduled rebuilds |
| Unpinned CI actions; long-lived static cloud keys in CI | Commit-SHA pins; OIDC and short-lived tokens |
| Data in the writable layer | Volumes and secret stores |

---

## 10.23 Try it yourself

All commands are safe on a development machine and use throw-away containers.

| Task | Command | Expected result |
|---|---|---|
| Default user | `docker run --rm alpine id`, then add `--user 10001:10001` | `uid=0(root)`, then `uid=10001` |
| Capabilities | `docker run --rm alpine sh -c 'grep CapEff /proc/self/status'`, then add `--cap-drop ALL`, then `--privileged` | Default mask, all zeros, largest mask |
| Read-only filesystem | `docker run --rm --read-only alpine sh -c 'touch /x'`, then add `--tmpfs /tmp` and `touch /tmp/x` | Fails, then works |
| PIDs limit | `docker run --rm --pids-limit 20 alpine sh -c 'for i in $(seq 1 100); do sleep 30 & done; echo done'` | Fork errors once the limit is hit |
| Memory limit | `docker run --rm --memory 64m --memory-swap 64m python:3.12-slim python -c "x=bytearray(200*1024*1024)"`, then `echo $?` | Killed; exit code 137 |
| Port exposure | `docker run -d --rm --name pubtest -p 8081:80 nginx:1.27-alpine`, check `docker ps`, `docker rm -f pubtest`, rerun with `-p 127.0.0.1:8081:80` | `0.0.0.0:8081`, then `127.0.0.1:8081` |
| Secret leak | Build an image with `ENV FAKE_SECRET=hunter2`, find it with `docker history --no-trunc` and `docker inspect`, rebuild with a secret mount | Secret no longer visible |
| Scan | `docker scout quickview <image>`; `docker scout cves --only-severity critical,high --only-fixed <image>`; triage one finding (10.13.5) | List of fixable findings |
| SBOM | `docker buildx build --sbom=true --provenance=mode=max -t demo:sbom --load .`, then `docker buildx imagetools inspect`. If `--load` is unsupported with attestations, use `--output type=docker` or a local registry. Depends on the image store; check `docker info` | Attestation payloads |
| Digest pin | `docker buildx imagetools inspect python:3.12-slim`, write a `FROM ...@sha256:...` line | Digest-pinned `FROM` |
| Audit | Check one Compose file against 10.21; on a Linux host, run docker-bench-security and fix the first three applicable WARNs | List of gaps |

---

Next: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md)
