# Chapter 10: Docker Security

> **What you will learn in this chapter**
> - The container threat model: why a container is *not* a tiny virtual machine, and what that means for you.
> - The single most dangerous file in Docker: the daemon socket.
> - Runtime hardening, one flag at a time: non-root, rootless mode, user namespaces, capabilities, `no-new-privileges`, seccomp/AppArmor, read-only filesystems, resource limits, and why `--privileged` is almost never the answer.
> - Network exposure, including the notorious "Docker bypasses ufw" problem.
> - Supply-chain security: trusted base images, digests, Docker Hardened Images, minimal images, scanning (Trivy, Docker Scout, Grype), SBOMs, provenance, signing (Cosign).
> - Secrets management done right, and what to do when a secret leaks (spoiler: rotate it).
> - Registry security, the CIS Docker Benchmark, docker-bench-security, runtime detection (Falco), logging and auditing.
> - A fully-explained hardened `docker run` and Compose file, a security checklist, and the classic anti-patterns.

Previous: [Chapter 8: Docker for AI and ML](08-docker-for-ai-and-ml.md) | Next: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md)

> **A note on how this chapter was checked.** Security facts go stale quickly. I verified key claims against official documentation in October 2026 (Docker's rootless, userns-remap, firewall, Scout, Hardened Images and MCP docs; NVIDIA, Sigstore and Aqua Security advisories). Where I could only find third-party sources, or could not verify a detail, I say so inline. Always run `--help` for your installed tool versions.

---

## 10.1 The threat model: what are we defending against?

**Threat model** is a fancy phrase for "who might attack what, how, and what do they gain?" Before locking doors, know what the house is made of.

### 10.1.1 Containers share the host's kernel

A **virtual machine (VM)** includes its own operating-system kernel running on virtualized hardware. A **container** is just a normal Linux process (or group of processes) that the *host's* kernel isolates using features:

* **Namespaces** give each container its own *view* of the system: its own process IDs, network stack, mount points, hostname, users.
* **cgroups** (control groups) limit how much CPU, memory and I/O it can use.
* **Capabilities, seccomp and LSMs (AppArmor/SELinux)** restrict what the process may *do*.

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

**Consequence:** if an attacker inside a container finds a **kernel vulnerability** (or a flaw in the container runtime, such as `runc`), they can potentially **escape** to the host and then attack every other container. Real examples have happened: CVE-2019-5736 (a `runc` flaw letting a malicious container overwrite the host's runc binary) and CVE-2024-21626 (another `runc` flaw, widely known as part of "Leaky Vessels"). Both were fixed by updating runc/Docker, which is the first lesson: **keep the host, the kernel and Docker patched.**

Analogy: VMs are separate apartments with their own walls and plumbing. Containers are separate rooms in one apartment, with locks on the doors, but they share the building's plumbing. A locked door is valuable, and the shared plumbing is the weak point.

### 10.1.2 Who are the attackers?

| Attacker | Scenario | Defence themes |
|---|---|---|
| **Remote attacker exploiting your app** (SQL injection, RCE in a library) | Gains code execution *inside* your container | Non-root, read-only FS, dropped capabilities, no secrets in the image, network egress limits: **limit the blast radius** |
| **Malicious or compromised image** | You pull an image with a backdoor or crypto-miner | Trusted sources, pinning by digest, scanning, signing, minimal images |
| **Compromised dependency** (supply chain) | A poisoned PyPI/npm package or a hijacked CI action | Lock files with hashes, SBOMs, provenance, pinned CI actions |
| **Insider / careless developer** | Commits a secret, mounts `/var/run/docker.sock`, runs `--privileged` | Policy, reviews, linting, least privilege |
| **Noisy neighbour / DoS** | One container eats all RAM/CPU/PIDs | Resource limits |
| **Network attacker** | Reaches an accidentally exposed database port | Bind to loopback, firewall, private networks |
| **Compromised CI/registry credentials** | Attacker pushes a malicious `latest` | Scoped tokens, MFA, signing + verification at deploy |

### 10.1.3 Defence in depth

No single flag makes a container "secure". You stack layers so that when one fails, the next holds:

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

The Docker CLI (`docker ...`) is only a client. It talks to the **Docker daemon** (`dockerd`), a long-running background service, through a **Unix socket** (a special file used for local inter-process communication) at `/var/run/docker.sock` on Linux. (On Docker Desktop for Windows the equivalent is a named pipe, `//./pipe/docker_engine`; the principle is identical.)

Whoever can send commands to that socket can ask the daemon to **start any container with any options**, and the daemon, in default installs, runs as **root**. So:

> **Access to the Docker socket is root access to the host.** There is no safe way to give "limited Docker access" through the standard socket.

### 10.2.2 Why it is so dangerous: a demonstration (read, do not run on a shared host)

If a container has the socket mounted:

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock some-image
```

then anything running inside that container (including an attacker who found a bug in your app) can do:

```bash
# inside the compromised container, with the docker client available
docker run --rm -it -v /:/host alpine chroot /host sh
```

* `-v /:/host` mounts the **entire host filesystem** into a new container.
* `chroot /host sh` makes `/host` the root and opens a shell, so the attacker is now effectively **root on the host**, able to read `/etc/shadow`, SSH keys, other containers' secrets, and install persistence.

Mounting the socket "just so my CI/dashboard/auto-update tool can see containers" is therefore the equivalent of handing over the host's root password. Tools that legitimately need it (Portainer, Traefik's Docker provider, Watchtower, Jenkins agents building images) are accepting this risk, so use the mitigations below.

### 10.2.3 Mitigations

| Mitigation | How |
|---|---|
| **Do not mount the socket** | Default answer. Use Compose, CI built-ins, or build tools that do not need it |
| **Use a socket proxy** | A tiny container (for example Tecnativa's `docker-socket-proxy`) that exposes only selected read-only API endpoints (like `GET /containers`) to the dependent app. The app never touches the real socket. Verify the proxy's current docs for the available permission toggles |
| **Mount read-only?** | `:ro` on the socket mount **does not help**: it only prevents replacing the socket file, not sending commands through it |
| **Rootless Docker** (section 10.4) | The daemon is not root, so a socket compromise is "one unprivileged user" rather than "host root" |
| **Build without the daemon** | Use BuildKit in rootless mode, Kaniko, Buildah, or `docker buildx` with a remote builder in CI instead of Docker-in-Docker over the host socket |
| **Docker-in-Docker (DinD) with care** | A `docker:dind` container needs `--privileged` and is itself risky; isolate it on disposable CI runners |
| **Never expose the daemon on TCP without TLS** | `dockerd -H tcp://0.0.0.0:2375` (unauthenticated) lets anyone on the network run containers as root. This is a classic way servers get hijacked for crypto mining |
| **Remote administration** | Use an SSH-based context: `docker context create prod --docker "host=ssh://deploy@server"`; then `docker --context prod ps`. SSH provides authentication and encryption |
| **Restrict the `docker` group** | Membership in the Linux group `docker` = root-equivalent. Treat it like `sudo`; do not add every user |
| **Docker Desktop Enhanced Container Isolation (ECI)** | A paid-tier Docker Desktop feature that runs containers in user-namespace-isolated, hardened sandboxes. Mentioned for completeness; check Docker's docs for availability and details |

---

## 10.3 Run as non-root

By default, the process inside a container runs as **root (UID 0)**, unless the image says otherwise. Container-root is *not* host-root (thanks to namespaces and dropped capabilities), but it is still far more powerful than necessary. If an attacker escapes through a kernel bug, they arrive on the host as real root. With a non-root user, they arrive as an unprivileged user and need another bug.

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

| Line | Explanation |
|---|---|
| `groupadd --system --gid 10001 app` | Create a *system* group (no login semantics) with a fixed numeric ID. A high number avoids clashes with real host users |
| `useradd --system --uid 10001 ... --shell /usr/sbin/nologin app` | Create the unprivileged user with no home directory and no interactive shell |
| `COPY --chown=app:app` | Files are owned by `app`, so the app can read them. (Do not make code *writable* by the app unless it must write there) |
| `USER 10001:10001` | Switch to the user **by number**. Numeric IDs let orchestrators (Kubernetes `runAsNonRoot: true`) verify the image is non-root; with a name, they cannot be sure |

### 10.3.2 At run time

```bash
docker run --user 10001:10001 myimage
```

* `--user uid:gid` overrides the user, useful for third-party images that run as root by default (if the app still works as non-root).

Check:

```bash
docker run --rm myimage id
# uid=10001(app) gid=10001(app) groups=10001(app)
```

### 10.3.3 Common non-root problems

| Symptom | Cause | Fix |
|---|---|---|
| `Permission denied` writing to a volume | Volume owned by root; app is UID 10001 | `chown` in the image *before* `VOLUME`, or `docker run --user` matching the volume, or an init step; in Kubernetes use `fsGroup` |
| `bind: permission denied` on port 80 | Ports below 1024 are privileged | Listen on 8080 and map `-p 80:8080`, or add the single capability `NET_BIND_SERVICE` |
| App wants to write in its own directory | Code directory is read-only/not owned | Write only to dedicated writable paths (`/tmp`, a volume) |

---

## 10.4 Rootless Docker and user namespaces

Two different features, often confused.

### 10.4.1 Rootless mode: the daemon itself is not root

From Docker's docs: *rootless mode runs the Docker daemon and containers as a non-root user inside a user namespace, to reduce the risk of daemon or runtime vulnerabilities.* Unlike `userns-remap`, **the daemon itself runs without root**.

Prerequisites (from Docker's rootless documentation):

* `newuidmap` and `newgidmap` (usually from the `uidmap` package).
* At least **65,536 subordinate UIDs and GIDs** for your user in `/etc/subuid` and `/etc/subgid`. (A **subordinate ID range** is a block of extra UIDs your unprivileged user is allowed to map into a user namespace, so "root in the container" can be "UID 231072 on the host".)

Install (as a normal user, after installing Docker's packages that include `docker-ce-rootless-extras`):

```bash
dockerd-rootless-setuptool.sh install
```

* Sets up a per-user systemd service for the daemon and creates a Docker CLI **context** named `rootless`.
* If a system-wide root daemon is already running, disable it first, or the script needs `--force`.

Verify:

```bash
docker info
```

You should see the `rootless` security option and context. Some tools need `DOCKER_HOST` pointing at the per-user socket, for example `export DOCKER_HOST=unix:///run/user/1000/docker.sock` (the number is your UID).

Trade-offs I know from general Docker documentation (the rootless page I fetched did not list them, so check Docker's rootless "known limitations" section for the current list): binding to ports below 1024 needs extra sysctl configuration, some networking features and storage drivers behave differently, performance of user-space networking can be lower, and cgroup resource limits need cgroup v2 with delegation. Rootless is excellent for developer machines, CI runners and multi-tenant hosts where a daemon compromise must not mean host root.

### 10.4.2 User namespace remapping (`userns-remap`): the daemon is root, container-root is not

This keeps a root daemon but **maps root inside every container to a high, unprivileged UID on the host.** From Docker's docs: a process that escapes the container has no real privileges on the host. The mapping lives in `/etc/subuid` and `/etc/subgid` with entries like `testuser:231072:65536` (user, starting host UID, count).

Enable it in `/etc/docker/daemon.json`:

```json
{
  "userns-remap": "default"
}
```

* `"default"` makes Docker create a user and group called `dockremap` and set up their subordinate ranges. You can also name your own user.
* Restart Docker afterwards: `sudo systemctl restart docker`.

Caveats (documented): enabling it effectively **hides existing images and containers** under a different storage location, so apply it to fresh hosts; volumes need ownership arranged for the remapped IDs; it cannot be combined with host PID/network namespace sharing; `--privileged` needs `--userns=host`; root in the container cannot `mknod`. You can opt a container out with `--userns=host`.

| Feature | Daemon runs as | Container root maps to | Best for |
|---|---|---|---|
| Default | root | **host root (UID 0)** (with restrictions) | Convenience |
| `userns-remap` | root | unprivileged high UID | Servers where you control the daemon and want escape mitigation |
| Rootless mode | **unprivileged user** | unprivileged UID in a user namespace | Dev machines, CI, shared hosts |

---

## 10.5 Capabilities: slicing up "root"

Traditionally "root" meant all-powerful. Linux splits root's powers into ~40 separate **capabilities**: for example `CAP_NET_BIND_SERVICE` (bind to ports under 1024), `CAP_NET_RAW` (craft raw packets, e.g. `ping`), `CAP_SYS_ADMIN` (a huge grab-bag, often called "the new root"), `CAP_SYS_PTRACE` (inspect other processes), `CAP_CHOWN`, `CAP_SETUID`...

Docker starts containers with a **reduced default set** (around a dozen-plus capabilities, enough for ordinary programs but excluding the dangerous ones like `SYS_ADMIN`). You can tighten further. The best-practice pattern is **drop everything, add back only what is proven necessary**:

```bash
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myimage
```

* `--cap-drop ALL` removes every capability.
* `--cap-add NET_BIND_SERVICE` adds back only the right to bind low ports.

Most web apps, APIs and workers need **no capabilities at all** once they listen on a high port as non-root. See what a running container has:

```bash
docker run --rm --cap-drop ALL alpine sh -c 'grep Cap /proc/self/status'
# CapInh: 0000000000000000
# CapPrm: 0000000000000000
# CapEff: 0000000000000000
# CapBnd: 0000000000000000
# CapAmb: 0000000000000000
```

* `/proc/self/status` shows the current process's capability bitmasks (`CapEff` = effective right now). All zeros means none.

Capabilities to be particularly suspicious of: `SYS_ADMIN`, `SYS_PTRACE`, `SYS_MODULE` (load kernel modules), `NET_ADMIN` (reconfigure networking; legitimately needed by VPN/firewall containers), `DAC_READ_SEARCH`, `DAC_OVERRIDE`.

---

## 10.6 `no-new-privileges`

```bash
docker run --security-opt no-new-privileges:true myimage
```

* Sets the Linux kernel's `no_new_privs` flag on the container's processes: **no process can gain more privileges than its parent** through setuid/setgid binaries or file capabilities. Without it, a compromised non-root process could run a setuid-root binary found in the image (like `su` or an old `ping`) to become root inside the container.
* Cost: almost none. Occasionally breaks images that deliberately rely on setuid helpers (e.g., some use `sudo` in entrypoints). Fix the image rather than weakening the flag.

---

## 10.7 seccomp and AppArmor/SELinux

### 10.7.1 seccomp: filter system calls

A **system call (syscall)** is a request from a program to the kernel (`open`, `read`, `mount`, `reboot`, `keyctl`...). Linux has well over 300. **seccomp** ("secure computing mode") lets you give a process a *filter*: an allow-list or deny-list of syscalls. Calls outside the list fail (or kill the process).

Docker applies a **default seccomp profile** automatically (you saw `name=seccomp,profile=builtin` in `docker info`'s security options). It blocks a few dozen dangerous or obsolete syscalls (such as `kexec_load`, `reboot`, `swapon`, and namespace-escape helpers) while allowing normal software to work. This default is a major reason ordinary containers are reasonably safe, and **you should never disable it**:

```bash
docker run --security-opt seccomp=unconfined myimage     # DON'T (shown only so you recognize it)
```

* `seccomp=unconfined` turns the filter off. It is sometimes suggested in forum posts to "fix" a mysterious permission error (for example a profiler or a game server). Find out *which* syscall is blocked and allow just that.

A stricter, custom profile:

```bash
docker run --security-opt seccomp=/path/to/profile.json myimage
```

* Supply a JSON profile that lists allowed syscalls. You can start from Docker's default profile (published in the Moby repository), remove what you do not need, and test thoroughly. Tools such as `strace` (run the app, record which syscalls it uses) and OCI-hooks/profile generators help. Profile maintenance is real work, so most teams use the default plus other layers.

### 10.7.2 AppArmor and SELinux: mandatory access control

**AppArmor** (Ubuntu/Debian/SUSE) and **SELinux** (Red Hat/Fedora family) are **Linux Security Modules (LSMs)**: kernel-level policies that restrict which files, capabilities and network actions a program may use *even as root*. Docker automatically applies an AppArmor profile named `docker-default` on AppArmor hosts, and SELinux labels on SELinux hosts (if enabled in the daemon).

* Do not disable them (`--security-opt apparmor=unconfined`, `--security-opt label=disable`) as a casual fix.
* SELinux bind-mount tip: on SELinux hosts, a bind-mounted directory may be unreadable until you add `:z` (shared label) or `:Z` (private label) to the volume option, e.g. `-v ./data:/data:Z`. Use `:Z` only on dedicated directories, because it relabels files on the host.

---

## 10.8 Read-only root filesystem and tmpfs

Most applications should not write to their own filesystem after start. Make it impossible:

```bash
docker run --read-only --tmpfs /tmp:rw,noexec,nosuid,size=64m myimage
```

* `--read-only` mounts the container's root filesystem read-only. Malware cannot drop tools, modify the app, or persist changes (they vanish anyway on restart, but this stops them *during* the attack).
* `--tmpfs /tmp:...` mounts a **tmpfs** (an in-memory filesystem) at `/tmp` so programs that need scratch space still work. Options: `rw` writable, `noexec` no running binaries from it, `nosuid` ignore setuid bits, `size=64m` cap its memory use.
* If the app fails with "Read-only file system", find which path it writes to and give *that path* a tmpfs or a volume (for instance `/var/run`, `/var/cache/nginx`, `/app/.cache`).

Expected error when something tries to write:

```
touch: /app/test.txt: Read-only file system
```

---

## 10.9 Resource limits: stop denial of service

Without limits, one runaway or malicious container can starve the host (a **denial of service**, DoS). Set limits for every service.

```bash
docker run --memory 512m --memory-swap 512m --cpus 1.0 --pids-limit 200 --ulimit nofile=4096:8192 myimage
```

| Flag | What it limits | Notes |
|---|---|---|
| `--memory 512m` (`-m`) | Hard RAM cap. Exceeding it triggers the **OOM killer** (out-of-memory), which kills the container's process (exit code 137) | Size it from real measurements (`docker stats`) |
| `--memory-swap 512m` | Total memory+swap. Setting it equal to `--memory` disables swap for the container | Avoids swap thrashing hurting the host |
| `--cpus 1.0` | Max CPU time: 1.0 = one core's worth | Fractional values allowed (`0.5`) |
| `--cpu-shares` | Relative weight under contention (soft) | Not a cap |
| `--pids-limit 200` | Max processes/threads | Stops **fork bombs** (`:(){ :|:& };:`) |
| `--ulimit nofile=4096:8192` | Open file descriptors (soft:hard) | Prevents descriptor exhaustion |
| `--storage-opt size=...` | Writable layer size | Only some storage drivers/filesystems support it |
| Log rotation (`--log-opt max-size=10m --log-opt max-file=3`) | Disk used by logs | The default `json-file` driver does **not** rotate by default and can fill the disk |

Watch usage live: `docker stats`.

```
CONTAINER ID   NAME      CPU %     MEM USAGE / LIMIT   MEM %     NET I/O         PIDS
a1b2c3d4e5f6   api       0.35%     120MiB / 512MiB     23.4%     1.2kB / 648B    14
```

(For Compose use `deploy.resources.limits` and `pids_limit`, or the top-level service keys `mem_limit`, `cpus`, `pids_limit`; see the Compose example in section 10.20.)

---

## 10.10 `--privileged`: don't

`docker run --privileged` is a master switch. It:

* grants **all capabilities**,
* disables seccomp and AppArmor/SELinux confinement (effectively),
* gives access to **all host devices** (`/dev/*`),
* lets the container mount filesystems and manipulate the kernel.

A privileged container is, in practice, **root on the host with extra steps.** A well-known one-line escape from a privileged container is to mount the host's disk device and read/write everything on it (for example by mounting `/dev/sda1`).

| You are tempted to use `--privileged` for... | Do this instead |
|---|---|
| Access one device (USB serial, GPU, `/dev/kvm`, `/dev/fuse`) | `--device /dev/ttyUSB0`, `--gpus all`, `--device /dev/kvm`, with `--group-add` for permissions |
| Run Docker inside Docker | Rootless BuildKit, Kaniko, Buildah, Sysbox runtime, or a dedicated disposable CI VM |
| Change one kernel setting | `--sysctl net.ipv4.ip_forward=1` (only namespaced sysctls are allowed) |
| Use network tools (`ping`, raw sockets, tcpdump) | `--cap-add NET_RAW` / `NET_ADMIN` as the single capability needed |
| "It fixes the permission error" | Find the actual denied operation (`dmesg`, `strace`, AppArmor logs) and grant the narrowest permission |
| Run `systemd` inside a container | Use a purpose-built approach (specialized runtimes) instead of broad privileges |

If you truly must use it (one-off helper like QEMU binfmt registration for cross-builds, as vLLM's docs describe), do it in a throw-away container on a trusted machine, never as a long-running service.

Other **dangerous options** to treat the same way:

| Option | Why dangerous |
|---|---|
| `--pid=host` | See and signal all host processes |
| `--network=host` | Container shares the host network stack, bypasses isolation, binds to any host port |
| `--ipc=host` | Shares host shared memory (sometimes needed for ML; accept knowingly, see [Chapter 8](08-docker-for-ai-and-ml.md)) |
| `--userns=host` | Disables user-namespace remapping for the container |
| `-v /:/host`, `-v /etc:/etc`, `-v /var/run/docker.sock:...` | Host filesystem or daemon control |
| `--cap-add SYS_ADMIN` | Near-root |
| `--security-opt seccomp=unconfined`, `apparmor=unconfined` | Removes kernel guard rails |

---

## 10.11 Network exposure

### 10.11.1 Published ports are public unless you say otherwise

`-p 8080:8080` means "publish container port 8080 on **all host interfaces** (0.0.0.0)". If your server has a public IP, the whole internet can reach it. **Bind to loopback** when the service is only for local use or sits behind a reverse proxy on the same host:

```bash
docker run -p 127.0.0.1:8080:8080 myimage
```

* `127.0.0.1:8080:8080` = `hostIP:hostPort:containerPort`. Only processes on the host itself can connect.

In Compose:

```yaml
ports:
  - "127.0.0.1:5432:5432"
```

If a container only talks to other containers, **do not publish at all**. Containers on the same user-defined network reach each other by service name without any published ports, so databases and caches should usually have no `ports:` entry.

### 10.11.2 "Docker bypasses ufw" (a classic trap)

You enable **ufw** (Uncomplicated Firewall) on Ubuntu, deny all incoming traffic, run `docker run -p 5432:5432 postgres`, and **the database is reachable from the internet anyway.** Why? Docker's official documentation states that Docker and ufw are incompatible in this way: Docker routes container traffic in the `nat` table, so traffic to published ports is diverted before it reaches the `INPUT`/`OUTPUT` chains where ufw's rules live, and ufw rules are effectively bypassed for published ports. (Also from the docs: Docker's firewall rules are created for bridge networks only, and Docker sets the IP-forwarding policy to drop so the host does not become an accidental router.)

What to do:

| Fix | Notes |
|---|---|
| **Bind published ports to `127.0.0.1`** (or a specific private IP) | Simplest and most reliable |
| **Use a reverse proxy** (Caddy, nginx, Traefik) as the only publicly published container, and keep everything else unpublished | Standard architecture |
| **Cloud firewall / security groups** in front of the host | Independent of host iptables; always use these |
| **Add rules in the `DOCKER-USER` iptables chain** | Docker evaluates this chain first for forwarded traffic and never overwrites it. Gotcha reported by several guides: the match happens *after* NAT translation, so rules must match the **container** port, not the published host port. Test with a scan from another machine |
| **Don't set `"iptables": false`** | Docker's docs warn this is likely to break container networking (no internet via masquerade, ports reachable from LAN) |
| **nftables backend** | Docker supports an nftables firewall backend via the `firewall-backend` daemon option (marked experimental in the docs I read; third-party sources say there is no `DOCKER-USER` chain there). Verify your Docker version's release notes before relying on it |
| Docs-adjacent claim | One third-party guide says Engine 28 adds a rule that drops packets to loopback-published ports arriving on other interfaces. I could not confirm this in Docker's docs, so still **test from another host** |

Test exposure from a different machine: `nmap -p 1-65535 <server-ip>` or at least `nc -vz <server-ip> 5432`.

### 10.11.3 Other network practices

* **Use user-defined bridge networks** (Compose does by default) to isolate stacks from each other; the default `bridge` network lets all containers talk.
* **Internal networks** for backends: `networks: backend: {internal: true}` gives containers no route to the outside world at all (great for databases and sandboxes).
* **Egress control:** if an app is compromised, outbound connections are how data leaves. Restrict where you can (proxy allow-lists, firewall rules, network policies in Kubernetes, `--network none` for batch jobs that need no network).
* **TLS:** terminate TLS at a reverse proxy; do not send credentials in clear text, even inside a data center, when networks are shared.
* **Never `--network host`** unless you fully understand the exposure.
* Services that default to *no authentication* (Ollama's API, Docker Model Runner's API (documented as unauthenticated), Redis in older setups, Jupyter without token, Elasticsearch in old versions, MongoDB in old versions) must never be on a public interface.

---

## 10.12 The image supply chain

Every image you run is code you did not write. Treat images like dependencies: choose, pin, verify, minimize and update.

### 10.12.1 Choose trusted sources

Docker Hub badges help you judge an image:

| Badge | Meaning |
|---|---|
| **Docker Official Image** | A curated set of images (ubuntu, python, postgres, nginx...) maintained with Docker, reviewed, with documented build recipes and regular updates |
| **Verified Publisher** | A commercial publisher vetted by Docker (for example NVIDIA, Grafana, HashiCorp) |
| **Docker-Sponsored Open Source** | Open-source projects sponsored by Docker |
| (no badge) | Anyone can publish anything. Treat with suspicion; read the Dockerfile/source, check download counts and maintenance, and prefer building yourself |

Typo-squatting (`pyhton`, `nginx-latest`) and "free software" images laced with miners are real threats.

### 10.12.2 Pin versions, then pin digests

A **tag** (like `python:3.12-slim`) is a *movable label*: the same tag points to different content over time (security patches, but also surprises). A **digest** is a cryptographic hash of the exact image content: immutable.

```dockerfile
FROM python:3.12-slim@sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

* `@sha256:...` pins exactly one image. (The digest above is a made-up placeholder; get a real one with the commands below.)

Get a digest:

```bash
docker buildx imagetools inspect python:3.12-slim
# Name:      docker.io/library/python:3.12-slim
# MediaType: application/vnd.oci.image.index.v1+json
# Digest:    sha256:...
```

* Prints the **index** digest (covers all architectures; recommended for portable pins).

Pin digests **and** keep them updated automatically with **Dependabot** or **Renovate** (bots that open pull requests when a new digest appears), so you get reproducibility *and* security patches through reviewed changes. Pinning without updating freezes vulnerabilities in place.

The same principle applies to everything in CI: the 2026 Trivy incident (section 10.14.4) showed that *mutable tags on GitHub Actions* can be hijacked. Pin actions to full commit SHAs.

### 10.12.3 Minimal images shrink the attack surface

Every package in an image is potential vulnerabilities and tools for an attacker (shells, `curl`, package managers).

| Option | What it is | Pros | Cons |
|---|---|---|---|
| `*-slim` (Debian slim) | Debian with fewer packages | Easy, glibc-compatible | Still has a shell and apt |
| **Alpine** | Tiny distribution using musl libc and BusyBox | Small | musl incompatibilities (notably some Python wheels, DNS edge cases); fine for Go/static apps |
| **Distroless** (Google) | Images with only your runtime and its libraries: no shell, no package manager | Few CVEs, hard for attackers to "live off the land" | Harder to debug (use ephemeral debug containers: `docker debug` / `kubectl debug`) |
| **`scratch`** | Empty image: only your static binary | The ultimate minimum | Go/Rust static binaries only; copy CA certificates and timezone data if needed |
| **Chainguard images** | Minimal, frequently rebuilt images with SBOMs and signatures (commercial/free tiers vary; check current terms) | Low CVE counts | Vendor-specific |
| **Docker Hardened Images (DHI)** | See next section | Free Community tier | Registry login required |

Multi-stage builds ([Chapter 4](04-dockerfile-deep-dive.md)) let you compile in a fat image and ship only the result in a minimal one:

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

* `CGO_ENABLED=0` builds a fully static Go binary (no C library needed).
* `distroless/static-debian12:nonroot` contains almost nothing but CA certificates and a non-root user; **the tag is an example; check the distroless project for current tags.**
* `COPY --from=build` copies only the binary out of the builder stage.

### 10.12.4 Docker Hardened Images (DHI)

From Docker's documentation (<https://docs.docker.com/dhi/>) and third-party news coverage in December 2025:

* **What:** minimal, production-oriented images (plus Helm charts and system packages) maintained by Docker, aiming at **near-zero known CVEs**, continuously scanned and patched.
* **License/cost:** Docker states the core features and the full catalog of open-source images are free under the **Apache 2.0** license in all tiers. Third-party coverage dates the free release to **December 2025**, with images pulled from `dhi.io` after logging in with a Docker account. (The official overview page I fetched did not show the registry name or pull commands. Check Docker's DHI "get started" page for the exact commands before use.)
* **Security features (all tiers):** distroless variants (Docker claims up to a 95% attack-surface reduction), **non-root by default**, packages built from source and signed, **SLSA Build Level 3 provenance, signed SBOMs, VEX statements and cryptographic signatures** on every image, based on Alpine and Debian with glibc and musl variants.
* **Tiers:** **Community** (free: core hardened images, no FIPS/STIG variants, no SLA on critical CVE fixes, no customization); **Select** (adds FIPS/STIG-ready variants, a **7-day SLA for critical CVE fixes**, up to 5 customizations); **Enterprise** (unlimited customizations, hardened system packages repo, optional extended lifecycle support).
* The docs say dev and runtime variants exist (typically: a dev variant with shell and package manager for building, a runtime variant without them for shipping); I could not verify the exact naming from the page I fetched.

Typical use shape (illustrative; confirm names in the DHI catalog):

```dockerfile
FROM dhi.io/python:3.12-dev AS build
# install dependencies...
FROM dhi.io/python:3.12 AS runtime
COPY --from=build /app /app
CMD ["python", "/app/main.py"]
```

* Build in a `-dev` variant (has tools), ship in the runtime variant (no shell, no package manager, non-root).
* You must `docker login dhi.io` first. **The image paths and tags here are placeholders for the pattern, not verified names.**

**Choosing:** use Official Images for convenience and learning; use hardened/minimal images for production services where CVE counts and audit requirements matter.

### 10.12.5 Safer Dockerfile habits

| Habit | Reason |
|---|---|
| Never `curl https://example.com/install.sh \| bash` in a `RUN` | You execute whatever the server serves at build time, unverified. Download a **pinned version**, verify its **checksum/signature**, then install |
| Verify downloads | `echo "<sha256>  file.tgz" \| sha256sum -c -` before extracting |
| Prefer `COPY` over `ADD` | `ADD` can fetch URLs and auto-extract archives (surprises) |
| `apt-get install --no-install-recommends` and clean lists in the same layer | Fewer packages, smaller image |
| Do not install `sudo`, `ssh`, compilers, `curl`/`wget` in runtime images unless required | Fewer attacker tools |
| Do not run `chmod 777` | Over-broad permissions |
| Add a `HEALTHCHECK` | Detect hung or crashed apps |
| Lint with **hadolint** | Catches many anti-patterns automatically |
| Keep `.dockerignore` strict | Keeps `.env`, keys, `.git` out of the build context and therefore out of layers |

Checksum-verified download example:

```dockerfile
ARG TOOL_VERSION=1.2.3
ARG TOOL_SHA256=<64-hex-digits-from-the-vendor-release-page>
RUN curl -fsSL -o /tmp/tool.tgz "https://example.com/tool-${TOOL_VERSION}-linux-amd64.tgz" \
 && echo "${TOOL_SHA256}  /tmp/tool.tgz" | sha256sum -c - \
 && tar -xzf /tmp/tool.tgz -C /usr/local/bin tool \
 && rm /tmp/tool.tgz
```

* `ARG`s hold the version and expected hash (non-secret values, so ARG is fine here).
* `sha256sum -c -` reads "hash  filename" from stdin and verifies the file; a mismatch returns non-zero and **fails the build**.
* The `&&` chain stops at the first failure.

---

## 10.13 Vulnerability scanning

A **vulnerability scanner** inventories the software packages inside an image (OS packages like `openssl`, language libraries like `requests`) and compares versions against databases of known flaws, identified by **CVE** IDs (Common Vulnerabilities and Exposures, e.g. `CVE-2024-12345`) with severities: LOW, MEDIUM, HIGH, CRITICAL (usually based on **CVSS**, the Common Vulnerability Scoring System, a 0-10 score).

Scanning finds **known** problems in what is *installed*; it does not find unknown bugs in your own code, misconfigurations, or secrets (though some scanners also do those).

### 10.13.1 Trivy

**Trivy** (by Aqua Security) is a widely used open-source scanner. Typical commands:

```bash
trivy image python:3.12-slim
trivy image --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 myorg/api:1.4.2
trivy image --format json --output report.json myorg/api:1.4.2
trivy fs .
trivy config .
```

| Command / flag | Meaning |
|---|---|
| `trivy image <ref>` | Scan an image (pulls it if needed) |
| `--severity HIGH,CRITICAL` | Report only these severities |
| `--ignore-unfixed` | Hide vulnerabilities with no available fix yet (you cannot act on them) |
| `--exit-code 1` | Exit with code 1 if any finding remains, making CI **fail**. (By default Trivy exits 0 even with findings) |
| `--format json --output report.json` | Machine-readable report |
| `trivy fs .` | Scan project files (lock files, secrets) |
| `trivy config .` | Scan Dockerfiles/Kubernetes/Terraform for misconfigurations |
| `--vex <file>` | Apply VEX statements (a source reported this flag; I did not verify details, so see Trivy's docs) |

Run Trivy without installing it (container):

```bash
docker run --rm -v trivy-cache:/root/.cache/ \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  aquasec/trivy:<pinned-version> image myorg/api:1.4.2
```

* Note this mounts the Docker socket (read-only does not protect it, see 10.2). A safer alternative is scanning a saved tarball: `docker save myorg/api:1.4.2 -o api.tar` then `trivy image --input api.tar`.
* **`<pinned-version>`**: pin a specific tag, not `latest` (see the next subsection for why).

Illustrative output (shape only; numbers are invented):

```
myorg/api:1.4.2 (debian 12.7)
=============================
Total: 3 (HIGH: 2, CRITICAL: 1)

┌───────────────┬────────────────┬──────────┬──────────┬───────────────────┬───────────────┬────────────────────────────────┐
│    Library    │ Vulnerability  │ Severity │  Status  │ Installed Version │ Fixed Version │             Title              │
├───────────────┼────────────────┼──────────┼──────────┼───────────────────┼───────────────┼────────────────────────────────┤
│ libssl3       │ CVE-2024-XXXXX │ CRITICAL │ fixed    │ 3.0.11-1~deb12u1  │ 3.0.14-1      │ openssl: example description   │
│ zlib1g        │ CVE-2023-XXXXX │ HIGH     │ fixed    │ 1:1.2.13.dfsg-1   │ 1:1.2.13-2    │ zlib: example description      │
└───────────────┴────────────────┴──────────┴──────────┴───────────────────┴───────────────┴────────────────────────────────┘
```

### 10.13.2 Docker Scout

**Docker Scout** is Docker's own analysis service: it builds an inventory of an image's components (an SBOM), compares it against a continuously updated vulnerability database, and shows results in the CLI, Docker Hub and the Docker Scout dashboard (per Docker's docs). It is integrated into the `docker` CLI on your machine (I confirmed `docker scout version` runs locally).

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
| `cves` | Detailed list of CVEs (also accepts directories, `docker save` tarballs and OCI layouts via prefixes such as `archive://`, `oci-dir://`, `fs://`, `sbom://`) |
| `--only-severity critical,high` | Filter by severity |
| `--only-fixed` / `--only-unfixed` | Show only fixable (or only unfixable) issues |
| `--ignore-base` | Exclude CVEs that come from the base image (focus on your own layers) |
| `--format sarif` | Output SARIF (a standard JSON report format that GitHub code scanning understands). Other formats include spdx, gitlab and markdown |
| `--exit-code` (`-e`) | Exit with code 2 when vulnerabilities are found, to fail CI |
| `recommendations` | Suggests better base-image tags (for example "update to this patch release to remove 12 CVEs") |
| `compare` | Differences between two images (new/fixed vulnerabilities and packages) |

(`quickview`, `recommendations` and `compare` come from my knowledge of the Scout CLI; the reference page I fetched covered `cves` in detail only, so run `docker scout --help`.) Scout's free versus paid feature boundaries (number of repositories enrolled, policy evaluation, etc.) change; check Docker's pricing page.

### 10.13.3 Grype (and Syft)

**Grype** (by Anchore) is another open-source scanner, and **Syft** is Anchore's SBOM generator. A typical flow: `syft myorg/api:1.4.2 -o spdx-json > sbom.json` then `grype sbom:./sbom.json`, or simply `grype myorg/api:1.4.2`. (From my general knowledge of these tools; I could not retrieve current Grype docs during research, so check `grype --help`.)

Using **two scanners** is sensible because their databases and matching differ; disagreements are informative.

### 10.13.4 A reminder that scanners are software too: the 2026 Trivy incident

In **March 2026**, Aqua Security's Trivy ecosystem suffered a multi-stage **supply-chain compromise**. According to Aqua's advisory (GHSA-69fq-xp46-6x23) and coverage by several security vendors:

* Attackers (credited by some vendors to a group calling itself "TeamPCP") gained access via credentials that were not fully rotated after an earlier incident.
* They rewrote **version tags** of the `aquasecurity/trivy-action` GitHub Action (sources say 75 or 76 of 77 tags) and `setup-trivy` to point to malicious commits that **harvested CI/CD secrets** (cloud keys, SSH keys, tokens) from runner memory and exfiltrated them.
* A malicious Trivy binary release (v0.69.4) and malicious Docker Hub images (v0.69.5 and v0.69.6) were also published through compromised credentials. Reported exposure windows were measured in hours for the Action and the binary and until about March 23 for the Docker Hub images.
* Only one `trivy-action` version (v0.35.0, protected by **immutable releases**) was reported unaffected. Aqua's remediation advice was to use pinned known-good versions, **pin actions to full commit SHAs**, audit workflow logs from the exposure window, and **rotate any secrets** that ran in affected pipelines.

(Sources differ on exact counts and times; consult the primary advisory for facts. By the time you read this, newer safe versions will exist.)

**Lessons that apply to every tool you use:**

1. **Pin by immutable identifier** (commit SHA for Actions; digest for images). Tags can be moved.
2. **Give CI jobs least privilege**: short-lived credentials (OIDC federation) instead of long-lived secrets stored in the CI.
3. **Scanning tools run with your secrets in scope**, so they deserve the same scrutiny as any dependency.
4. **Have a rotation runbook** before you need one (section 10.15.6).

### 10.13.5 How to triage scan results (without panicking)

A scanner on a typical image can produce dozens or hundreds of findings. Follow this order:

```
1. Is it fixable?            No fix -> record it, apply mitigations, re-check later (--ignore-unfixed for gating).
2. Where does it come from?  Base image OS package -> update/rebuild the base (often one change fixes many).
                             Your app dependency  -> upgrade the dependency/lock file.
3. How severe, really?       CRITICAL/HIGH first. CVSS is generic; context matters.
4. Is it reachable?          Is the vulnerable package/function actually used? Is the service exposed?
                             (A bug in a build tool left in the runtime image is less risky than one in your web framework.)
5. Is there a VEX?           Vendor VEX statements say "not affected" for specific products (see below).
6. Decide and record:        Fix now / fix soon / accept with expiry date / false positive.
```

* **Rebuild regularly.** The biggest cheap win: rebuild images on a schedule (weekly) so you pick up patched base images, even if your code did not change.
* **Switch to a smaller base** to eliminate whole classes of findings (shells, curl, Perl in Debian-based images).
* **VEX (Vulnerability Exploitability eXchange):** a machine-readable statement from a supplier, such as "CVE-X is *not affected* in this product because the vulnerable code path is not present", used to suppress false positives. Docker Hardened Images ship VEX statements (per Docker's docs), and Trivy/Scout can consume them.
* **Ignore files:** Trivy's `.trivyignore` lists accepted CVEs. Always add a comment with the reason and an **expiry date**, otherwise ignores become permanent blind spots.
* **Gate in CI sensibly:** fail builds on **fixable HIGH/CRITICAL**; report the rest. Gating on unfixable issues teaches developers to bypass the gate.
* **Scan continuously**, not only at build: new CVEs are published for images you shipped months ago. Registries (ECR, Harbor, Docker Hub with Scout) can re-scan stored images.

---

## 10.14 SBOMs and provenance attestations

### 10.14.1 SBOM

An **SBOM (Software Bill of Materials)** is a machine-readable ingredient list for an image: every package and version inside. Common formats: **SPDX** and **CycloneDX**. Why it matters: when the next big vulnerability (think "Log4Shell") appears, you can *query* your SBOMs ("which of our images contain this library?") in minutes instead of days, and regulators and customers increasingly ask for them.

### 10.14.2 Provenance

**Provenance** is a signed-or-signable statement of *how* an image was built: which source repo and commit, which builder, which build arguments, when. The common standard is **SLSA** (Supply-chain Levels for Software Artifacts, pronounced "salsa"). It lets you verify "this image really came from our CI at commit abc123, not from someone's laptop."

### 10.14.3 Creating them with BuildKit

From Docker's build attestation docs (<https://docs.docker.com/build/metadata/attestations/>):

```bash
docker buildx build --sbom=true --provenance=mode=max -t myorg/api:1.4.2 --push .
```

| Flag | Meaning |
|---|---|
| `--sbom=true` | Generate an SBOM attestation (default generator is Docker's BuildKit Syft-based scanner) and attach it to the image in the registry |
| `--provenance=mode=max` | Generate **maximum-detail** SLSA provenance (min is on by default in current BuildKit; max adds the build definition, the base64-encoded Dockerfile and source maps) |
| `--push` | Push to a registry (attestations are stored as extra manifests alongside the image) |

**Warnings from the docs:**

* `mode=max` **exposes the values of build arguments** in the provenance. So never pass secrets via `--build-arg`; use secret mounts, which "don't leak outside of the build and are never included in provenance attestations".
* The classic (non-containerd) Docker image store does not support attestations; you need the containerd image store or a `docker-container` builder. (On this guide's reference machine, `docker info` reports the `io.containerd.snapshotter.v1` driver type, so attestations are supported; check yours with `docker info`.) If you cannot switch, opt out with `--provenance=false --sbom=false`.
* Multi-platform and attested images are "image indexes" with extra attestation manifests (which is why registries show "unknown/unknown" platform entries; they are attestations, not broken images).

Inspect them:

```bash
docker buildx imagetools inspect myorg/api:1.4.2 --format '{{ json .SBOM }}'
docker buildx imagetools inspect myorg/api:1.4.2 --format '{{ json .Provenance }}'
docker scout sbom myorg/api:1.4.2
```

* `imagetools inspect --format` prints the attestation payloads as JSON.
* `docker scout sbom` prints an SBOM for an image (from attestation, or generated on the fly).

In GitHub Actions, `docker/build-push-action` accepts `sbom: true` and `provenance: mode=max` inputs (see the CI sketch in [Chapter 8](08-docker-for-ai-and-ml.md)).

Verifying provenance and SBOM *attestations* cryptographically is done with Cosign (`cosign verify-attestation`, next section).

---

## 10.15 Image signing: Cosign (and others)

Scanning tells you what is *in* an image; **signing** tells you *who* published it and that it was *not modified* since. Without verification at deploy time, anyone who can push to your registry (or hijack a tag) can run code in production.

### 10.15.1 Cosign and Sigstore

**Sigstore** is an open-source project and public infrastructure for signing software; **Cosign** is its command-line tool for containers. Its big idea is **keyless signing**: instead of managing a long-lived private key, you authenticate with an identity provider (OIDC: GitHub, Google, Microsoft), Sigstore's certificate authority (**Fulcio**) issues a **short-lived certificate** (minutes) tying the signature to that identity, and the signature is recorded in a public **transparency log** (**Rekor**) so it cannot be quietly forged or backdated.

Sign (from the Sigstore docs; keyless means omit `--key`):

```bash
cosign sign myorg/api@sha256:<digest>
```

* Prompts you to authenticate through an OIDC provider in a browser (in CI, it uses the workflow's own identity token).
* **Sign by digest**, not by tag: tags move, so signing `:latest` signs "whatever it pointed to when you signed", and the docs' examples sign `...@sha256:...` references.

In GitHub Actions (sketch):

```yaml
permissions:
  contents: read
  packages: write
  id-token: write        # lets the job request an OIDC identity token for keyless signing

steps:
  - uses: sigstore/cosign-installer@<pin-to-commit-sha>
  - run: cosign sign --yes ghcr.io/myorg/api@${{ steps.build.outputs.digest }}
```

* `id-token: write` is the permission allowing keyless signing; `--yes` skips the interactive confirmation.
* `steps.build.outputs.digest` is the digest output from the earlier build-push step (name depends on your step id).
* Pin the installer action to a commit SHA (lesson from 10.13.4).

Verify (shape from the Sigstore docs; the verify page documents `--certificate-identity` and `--certificate-oidc-issuer`):

```bash
cosign verify \
  --certificate-identity "https://github.com/myorg/api/.github/workflows/release.yml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  ghcr.io/myorg/api@sha256:<digest>
```

| Flag | Meaning |
|---|---|
| `--certificate-identity` | *Who* must have signed: here, a specific GitHub workflow file on the `main` branch. Use the **exact** identity in production rather than broad regular expressions (`--certificate-identity-regexp`), since loose patterns weaken the policy |
| `--certificate-oidc-issuer` | *Which identity provider* vouches for that identity (GitHub Actions' token issuer in this example) |

Verify attestations (SBOM/provenance) with `cosign verify-attestation` and a `--policy`. Check `cosign verify --help` for flags in your version.

**Security note on Cosign versions:** GitHub advisory GHSA-fx35-mq7g-6g98 reported that with a **legacy JSON bundle**, Cosign could silently skip the `--certificate-identity` / `--certificate-oidc-issuer` checks, so verification would succeed for any validly signed artifact. The advisory says the newer **protobuf bundle format** path is not affected, and recommends upgrading (to v3.1.3 or v2.6.5, as reported in the search snippets I saw) and migrating legacy bundles. Cosign v3's default is the standardized bundle format. Moral: keep Cosign up to date and test your verification with a **negative case** (a signature from the wrong identity should fail).

### 10.15.2 Other signing systems

| System | Notes |
|---|---|
| **Notation (Notary Project, "Notary v2")** | CNCF-adjacent signing standard using X.509 certificates, supported by several cloud registries |
| **Docker Content Trust (Notary v1)** | Older Docker-specific signing (`DOCKER_CONTENT_TRUST=1`). Considered legacy; I did not verify its current status in Docker's roadmap, so check before building new processes on it |
| **Registry-native signing/attestation** (ECR, ACR, GAR, GHCR attestations) | Cloud-specific; check provider docs |

### 10.15.3 Enforce verification at deploy time

Signing is useless if nobody checks. Enforce with an **admission controller** in Kubernetes (Sigstore **policy-controller**, **Kyverno**, **Connaisseur**, or cloud-native policies) so that unsigned or wrongly-signed images are rejected, and in non-Kubernetes environments with a deploy script that runs `cosign verify` and aborts on failure.

---

## 10.16 Secrets management

A **secret** is anything that grants access: passwords, API keys, tokens, private keys, certificates, signing keystores, database URLs with embedded passwords.

### 10.16.1 Where secrets leak (and why)

| Leak path | Why it is dangerous |
|---|---|
| `ENV SECRET=...` in a Dockerfile | Stored in image metadata; visible with `docker inspect` and `docker history`; copied into every container started from it |
| `ARG SECRET=...` used in a `RUN` | Build args are recorded in image history/metadata (and in max-mode provenance) |
| `COPY .env .` or any `COPY` of a credentials file | The file lives in a layer **forever**, even if a later `RUN rm` deletes it, because each layer is an immutable snapshot you can extract |
| Hard-coded in source, committed to git | Git history is permanent; forks and clones keep it |
| Environment variables at runtime | Visible via `docker inspect` to anyone with Docker access, and in `/proc/<pid>/environ`, crash dumps, debug pages (`phpinfo()`), and logs that print the environment; also inherited by every child process |
| Logs and error messages | Exceptions that print configuration or URLs with passwords |
| CI logs | `echo $SECRET` or verbose tooling output |

Demonstrate the first leak safely with a harmless value:

```bash
docker run --rm -e DEMO_TOKEN=not-a-real-secret alpine sleep 60 &
docker inspect --format '{{.Config.Env}}' $(docker ps -q --filter ancestor=alpine | head -1)
# [DEMO_TOKEN=not-a-real-secret PATH=/usr/local/sbin:...]
```

* Anyone who can run `docker inspect` (that is, anyone with socket access) can read runtime env vars.
* For Dockerfile-level leaks, `docker history --no-trunc <image>` shows every `RUN`/`ENV`/`ARG` instruction text.

### 10.16.2 Build-time secrets: BuildKit secret mounts

When a *build* needs a credential (private package registry token, a gated model download), use a **secret mount**. It is available during that one `RUN` step and **never stored in any layer**:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN --mount=type=secret,id=pip_token \
    PIP_INDEX_URL="https://__token__:$(cat /run/secrets/pip_token)@pypi.example.com/simple" \
    pip install --no-cache-dir -r requirements.txt
```

* `--mount=type=secret,id=pip_token` exposes the secret as the file `/run/secrets/pip_token` for this instruction only.
* The `$(cat ...)` reads it into a variable scoped to this single command.
* **pip prints index URLs in some error messages**; for stricter hygiene use a netrc/credential-helper file mounted as a secret instead. (The index URL is a placeholder.)

Build with:

```bash
docker build --secret id=pip_token,src=./pip_token.txt -t myimage .
# or from an environment variable:
docker build --secret id=pip_token,env=PIP_TOKEN -t myimage .
```

* `src=` reads from a file; `env=` reads from an environment variable of the shell running the build.

For **SSH keys** (cloning a private git repo during build): `RUN --mount=type=ssh git clone git@github.com:org/private.git` with `docker build --ssh default .`, which forwards your SSH agent without copying the key.

**Multi-stage builds do not "protect" secrets** that were put in an *earlier stage* of an image you also push (the builder-stage layers are discarded from the final image, but if you ever push or cache the builder stage, they are exposed). Secret mounts are the right tool; they avoid the question entirely.

### 10.16.3 Runtime secrets

| Method | How | Pros | Cons |
|---|---|---|---|
| **Environment variable** | `-e DB_PASSWORD` / Compose `environment` | Simple; every language reads it | Visible in `docker inspect`, `/proc`, logs; inherited by children. Acceptable for low-value dev secrets |
| **File mounted into the container** (secret file) | `-v ./secrets/db_password:/run/secrets/db_password:ro` or Compose/Swarm `secrets:` | Not in `docker inspect` env; can set file permissions; works with `*_FILE` convention | Slightly more code |
| **Compose `secrets:`** | Top-level `secrets:` mapped to a `file:` (or `environment:`); service lists it; mounted at `/run/secrets/<name>` | Standard, simple | In plain Compose it is a file bind mount, not encrypted storage |
| **Docker Swarm secrets** | `docker secret create`; encrypted at rest and in transit in the Swarm Raft store; mounted in-memory (tmpfs) into containers | Real secret store for Swarm | Swarm-only |
| **Kubernetes Secrets** | `kind: Secret` mounted as files or env | Native | By default **base64-encoded, not encrypted** in etcd: enable encryption at rest and tight RBAC; consider External Secrets Operator, Sealed Secrets, SOPS, or the Secrets Store CSI driver |
| **Secret managers** | HashiCorp Vault, AWS Secrets Manager / SSM Parameter Store, GCP Secret Manager, Azure Key Vault; app or sidecar fetches at start using the workload's cloud identity (IAM roles, workload identity) | Central audit, rotation, short-lived dynamic credentials | More infrastructure |

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

* The official Postgres image supports the `*_FILE` convention: `POSTGRES_PASSWORD_FILE` makes it read the password from that file instead of an env var.
* `./secrets/db_password.txt` must be **git-ignored** and have tight permissions (`chmod 600`).

**Best practice order:** short-lived credentials from an identity-based system (cloud IAM, Vault dynamic secrets, OIDC) > secret manager > mounted files > env vars > anything baked into images (never).

### 10.16.4 `.env` files and Compose

Compose reads `.env` for **variable substitution** inside the Compose file (e.g. `${POSTGRES_PASSWORD}`) and `env_file:` for passing variables to containers. Rules: add `.env` to `.gitignore` **and** `.dockerignore`; commit a `.env.example` with fake values; run `git ls-files | grep -i env` to verify you did not commit one; use pre-commit secret scanners (**gitleaks**, **trufflehog**, GitHub **push protection**).

### 10.16.5 What to do if a secret leaked: the golden rule

> **If a secret has been exposed anywhere beyond its intended holders, assume it is compromised and ROTATE IT. Deleting or hiding it does not un-leak it.**

Why rewriting history is not enough: the secret may already be cloned, forked, cached by CI, scraped by bots (public repos are scanned within minutes), present in a published image layer that someone pulled, or in logs.

Incident runbook:

1. **Revoke / rotate first.** Generate a new credential, deploy it, then invalidate the old one at the *provider* (cloud console, GitHub token settings, database `ALTER USER`...). Speed matters more than anything else. For **signing keystores** and anything that cannot easily be re-issued, consult the provider's key-compromise procedures: a leaked Android upload key can be reset through the Play Console, while an app-signing key has more serious implications.
2. **Find out what it could access** and review the access logs for the exposure window (cloud audit logs, database logs, provider dashboards).
3. **Remove the source of the leak:**
   * *In git:* remove the file in a new commit **and** purge history (`git filter-repo` or BFG Repo-Cleaner), force-push, and ask collaborators to re-clone. Also contact the host (e.g. GitHub support) to purge cached views and PR refs if it is public. Treat this as cleanup, not as a substitute for rotation.
   * *In an image/registry:* delete the affected tags and digests from the registry, rebuild **without** the secret, redeploy, and remember that anyone who pulled the old image still has it.
   * *In CI logs:* delete the logs/artifacts; rotate anyway.
4. **Prevent recurrence:** secret scanning in pre-commit and CI, `.dockerignore`, BuildKit secret mounts, short-lived credentials, least-privilege scopes.
5. **Write a short post-incident note** (what, when, how found, what changed).

This is exactly the lesson of the Trivy incident: the attackers' foothold persisted because an earlier credential rotation was **incomplete**. Rotate *everything* that could have been exposed.

---

## 10.17 Registry security

A **registry** stores and serves images (Docker Hub, GitHub Container Registry (GHCR), Amazon ECR, Google Artifact Registry, Azure Container Registry, Harbor, a self-hosted `registry:2`).

| Practice | Details |
|---|---|
| **Private repositories** for non-public images | Public by default on some services; check visibility settings |
| **Scoped access tokens**, not passwords | Docker Hub personal/organization access tokens with read-only or read-write scope; GHCR fine-grained tokens; cloud IAM roles. Use **separate tokens per CI pipeline** so you can revoke one |
| **Enable MFA/SSO** on registry accounts | A stolen account password should not let attackers push `latest` |
| **Least privilege** | Build jobs push; deploy hosts only pull |
| **Short-lived credentials in CI** | OIDC federation to cloud registries instead of stored keys |
| **Immutable tags** | Cloud registries (ECR, others) can forbid overwriting tags, so a published `1.4.2` cannot change under you |
| **Rate limits** | Docker Hub throttles pulls (historically about 100 pulls per 6 hours for anonymous users per IP and 200 for authenticated free accounts; Docker has revised its limits and plans over time, so check current numbers). CI fleets behind one NAT IP hit this fast. **Authenticate your pulls**, use a **pull-through cache/mirror** (a registry that proxies and caches Docker Hub), or mirror critical base images into your own registry. Mirroring also protects you if an upstream image or tag disappears |
| **Scanning on push** | ECR/ACR/GAR/Harbor/Docker Hub (with Scout) can scan and re-scan stored images |
| **Self-hosted registry** | Always TLS and authentication (`registry:2` has neither by default beyond what you configure); consider **Harbor**, which adds RBAC, scanning, replication, retention and signature policies |
| **Retention/cleanup** | Delete stale tags to reduce clutter and cost, but keep deployed digests |
| **Audit logs** | Know who pushed and pulled what |
| **Never put credentials in image names or in `~/.docker/config.json` on shared hosts** | `docker login` stores credentials in `~/.docker/config.json`; by default this can be base64 (not encrypted), so use a **credential helper** (`osxkeychain`, `wincred`, `pass`, `secretservice`) |

---

## 10.18 CIS Docker Benchmark and docker-bench-security

The **CIS Docker Benchmark** (from the Center for Internet Security) is a consensus checklist of hardening recommendations covering the host, the daemon configuration, daemon files, container images and build files, container runtime, security operations and Swarm. **docker-bench-security** is Docker's open-source script that **automatically checks** a host against it.

Run it (read-only checks; you run it on a Linux Docker host):

```bash
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sudo sh docker-bench-security.sh
```

* Clones the project and runs the audit script as root (needed to read daemon configs and audit rules). **Read the script before running anything as root.**

Output looks like:

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
| `[PASS]` | The check passed |
| `[WARN]` | Failed; investigate and fix, or document the accepted risk |
| `[INFO]` | Informational; manual review needed |
| `[NOTE]` | Advisory remark |

Version note: the docker-bench-security README says its tests are based on **CIS Docker Benchmark v1.6.0** (project release v1.6.1 was the latest I found), while CIS's own site lists a newer benchmark version (1.9.0 in the snippet I saw), so the script can lag the published benchmark. Use it as a **practical baseline scan**, and consult the current CIS PDF for the latest recommendations. Aqua's separate Go-based `docker-bench` also exists and targets CIS 1.6.0. Not every WARN matters equally: fix privileged containers, missing limits, exposed sockets and daemon TLS first; treat partition and auditing recommendations as production-hardening backlog.

---

## 10.19 Runtime security, logging and auditing

Prevention fails eventually. **Detection** tells you when.

### 10.19.1 Falco

**Falco** (a CNCF project) watches kernel events (system calls, using an eBPF probe or kernel module) and raises alerts when container behaviour matches **rules**. Typical rules, with plain-English meaning:

* a **shell was spawned inside a container** (normal web apps do not run `/bin/sh`),
* a container wrote to `/etc` or `/bin`,
* a process read **sensitive files** (`/etc/shadow`, cloud credentials),
* an **unexpected outbound connection** (crypto-miner pool),
* a container **mounted the Docker socket** or started privileged,
* `kubectl exec` / `docker exec` into production containers.

A rule has the shape (illustrative, adapted from the style of Falco's default rules; check Falco's docs for current syntax and macros):

```yaml
- rule: Shell in Container
  desc: A shell was spawned in a container
  condition: evt.type = execve and container.id != host and proc.name in (bash, sh, zsh)
  output: "Shell spawned in container (user=%user.name container=%container.name cmd=%proc.cmdline)"
  priority: WARNING
```

* `condition` is a filter expression over events; `output` is the alert text with fields filled in; `priority` is the severity.
* Falco's own installation docs describe how to run it (it needs elevated access to observe the kernel, usually as a privileged or specially-capable container, a host service, or a Kubernetes DaemonSet). Alerts typically go to a SIEM or chat via **Falcosidekick**.

Alternatives and companions: **Tetragon** and **Tracee** (eBPF-based), cloud-provider runtime-threat-detection features, and commercial CNAPP platforms.

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
| `docker events` | Live stream of daemon events (container start/stop/exec/die, image pulls, `oom`); pipe to logs/alerts |
| `docker diff <container>` | Lists files **Added (A), Changed (C), Deleted (D)** in the container's writable layer vs the image, a very quick way to see what an attacker modified |
| `docker logs` | Application stdout/stderr |
| `docker top` | Processes running inside the container as the host sees them |
| `docker inspect` | Full config: mounts, env, caps, security options, network |
| `docker exec` | A shell for debugging **but also what attackers do**; in production, restrict and audit it |

### 10.19.3 Logging and audit practices

* **Ship logs off the host** (Loki, ELK/OpenSearch, CloudWatch, Datadog...) using a logging driver (`--log-driver`) or an agent; local logs vanish with the host or container.
* **Rotate local logs** (`max-size`, `max-file`) so they do not fill the disk.
* **Use structured (JSON) logs** with request IDs; never log secrets or personal data.
* **Audit the daemon:** on Linux, `auditd` rules for `/usr/bin/dockerd`, `/etc/docker`, `/var/lib/docker`, `docker.sock` and `containerd` (the CIS benchmark lists the exact rules) record who touched Docker.
* **Alert on:** privileged container starts, new images from unexpected registries, containers running as root, restarts/OOM kills, `docker exec` into production.
* **Time sync (NTP)** and centralized timestamps so you can reconstruct incidents.
* **Retention** matching your compliance needs.

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

Every flag explained:

| Flag | Purpose |
|---|---|
| `-d` | Run in the background |
| `--name api` | Stable name for management and logs |
| `--user 10001:10001` | Non-root user and group (10.3) |
| `--read-only` | Immutable root filesystem (10.8) |
| `--tmpfs /tmp:rw,noexec,nosuid,size=64m` | Small writable in-memory scratch space that cannot execute binaries |
| `--cap-drop ALL` | Remove all Linux capabilities (10.5) |
| `--security-opt no-new-privileges:true` | Block privilege escalation (10.6) |
| `--memory 512m --memory-swap 512m` | Hard RAM cap with swap disabled (10.9) |
| `--cpus 1.0` | CPU cap |
| `--pids-limit 200` | Fork-bomb protection |
| `--ulimit nofile=4096:8192` | File-descriptor limit |
| `--network app-net` | Attach to a user-defined network (create with `docker network create app-net`); isolates from the default bridge |
| `-p 127.0.0.1:8000:8000` | Publish on loopback only (10.11) |
| `--restart unless-stopped` | Auto-restart after crashes/reboots |
| `--health-cmd ...`, `--health-interval`, `--health-retries` | Health check so Docker can mark the container `unhealthy` (orchestrators can act on it) |
| `--log-opt max-size/max-file` | Cap log disk usage |
| `-v ...:ro` + `-e API_KEY_FILE=...` | Secret delivered as a **read-only file** (the host path is an example; keep it root-owned and `chmod 600`/appropriate group) rather than an env var |
| `myorg/api@sha256:<digest>` | Image referenced by immutable digest (10.12.2) |

Notice what is **absent**: `--privileged`, `--network host`, the Docker socket, `seccomp=unconfined`, host mounts of sensitive paths. (The default seccomp and AppArmor/SELinux profiles remain active.)

Quick check that hardening took effect:

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

Explanation of the interesting parts:

| Setting | Why |
|---|---|
| Only `proxy` has `ports:` | The reverse proxy is the **only** publicly reachable container. `api` and `db` have no published ports at all |
| `cap_add: [NET_BIND_SERVICE]` for the proxy | Allows binding ports 80/443 as non-root; everything else dropped. (Some Caddy images set this up already; confirm for your tag) |
| `networks` with `internal: true` | `backend` and `data` networks have **no route to the internet**; a compromised API container cannot call out to an attacker's server directly (note: if your API must call external services such as a hosted LLM, it needs an additional non-internal network or an egress proxy) |
| `proxy` on `edge`+`backend`; `api` on `backend`+`data`; `db` only on `data` | Containers can talk only if they share a network. The proxy can reach the API but **cannot reach the database**; the API bridges both; the database is reachable only from the API. This is network segmentation |
| `read_only`, `tmpfs`, `user`, `cap_drop`, `security_opt` | Same as the `docker run` version |
| `mem_limit`, `memswap_limit`, `cpus`, `pids_limit` | Resource caps in Compose syntax (these top-level service keys work with plain `docker compose up`; the `deploy.resources.limits` form is also supported by current Compose) |
| `db` with `cap_drop: ALL` plus a few `cap_add` | The official Postgres image starts as root to fix data-directory ownership and then drops to the `postgres` user, so it needs a handful of capabilities. **If you add `user:` or the image/version changes, re-test**; start with none and add only what error messages demand |
| `secrets` with `*_FILE` variables | No password in `environment` or in `docker inspect` |
| `image: ...@sha256:<digest>` | Immutable pin (placeholder digest). `postgres:17` and `caddy:2` are tags for readability; pin digests in production |

Test with `docker compose up -d` and `docker compose ps`. (You will need a `Caddyfile` and a real API image; the digest and file contents here are placeholders.)

Validate the file and see the resolved configuration:

```bash
docker compose config
```

* Prints the fully merged and variable-substituted configuration, or errors if the YAML is invalid. Handy for reviewing exactly what will run.

---

## 10.21 Security checklist

Copy this into your repo's `SECURITY.md` or PR template.

**Host and daemon**
- [ ] Host OS, kernel and Docker Engine are patched; automatic security updates enabled.
- [ ] Docker socket is not mounted into application containers (or a socket proxy is used).
- [ ] Daemon is not exposed on unauthenticated TCP (2375); remote access uses SSH or mutual TLS.
- [ ] `docker` group membership is minimal; treated as root.
- [ ] Rootless mode or `userns-remap` considered/enabled where practical.
- [ ] `docker-bench-security` run; WARNs triaged.
- [ ] Hostile or multi-tenant code runs in VMs/microVMs/gVisor, not plain containers.

**Images and supply chain**
- [ ] Base images are Official/Verified/Hardened or built in-house; minimal (slim/distroless/DHI) where possible.
- [ ] Tags pinned; production references **digests**; a bot (Dependabot/Renovate) proposes updates.
- [ ] Multi-stage builds; no compilers, shells or package managers in final images where avoidable.
- [ ] No `curl | bash`; downloads verified by checksum/signature.
- [ ] `.dockerignore` excludes `.git`, `.env`, keys.
- [ ] Images scanned in CI (and continuously in the registry); fixable HIGH/CRITICAL block merges; ignores have reasons and expiry.
- [ ] Images rebuilt on a schedule for patched bases.
- [ ] SBOM and provenance attestations generated; images signed (Cosign); deploy verifies signatures and identity.
- [ ] CI actions and tools pinned to commit SHAs/digests; CI uses short-lived OIDC credentials.

**Runtime configuration**
- [ ] Containers run as a **non-root numeric UID**.
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
- [ ] TLS everywhere users or credentials are involved; unauthenticated services (Redis, Ollama, Jupyter, Model Runner) never public.

**Secrets**
- [ ] No secrets in Dockerfiles, `ENV`, `ARG`, images, or git.
- [ ] Build-time secrets via BuildKit `--mount=type=secret` / `--ssh`.
- [ ] Runtime secrets from files or a secret manager; short-lived where possible.
- [ ] Secret scanning (pre-commit + CI + push protection) enabled.
- [ ] Rotation runbook exists and has been rehearsed.

**Registry and operations**
- [ ] Private registries, scoped tokens, MFA/SSO; push and pull permissions separated.
- [ ] Pull-through cache or mirror of critical base images; pulls authenticated.
- [ ] Logs shipped centrally; audit rules for the daemon; alerts on privileged starts and unexpected `exec`.
- [ ] Runtime detection (Falco/Tetragon or cloud equivalent) for production.
- [ ] Incident response plan covers "compromised image", "leaked secret", "compromised host".

---

## 10.22 Common anti-patterns

| Anti-pattern | Why it is bad | Do this instead |
|---|---|---|
| **Secrets in `ENV`/`ARG`/layers** (`ENV API_KEY=...`, `COPY .env`) | Readable by anyone with the image or `docker inspect`/`history` | Secret mounts at build time; files/secret managers at runtime; rotate anything already leaked |
| **`curl ... \| bash` installs in Dockerfiles** | Runs unverified, changing code at build time | Pinned versions + checksums/signatures, or distro packages |
| **`latest` tags everywhere** (`FROM node:latest`, `image: foo:latest`) | Non-reproducible; silent upgrades and supply-chain swaps | Version tags, then digests, with automated update PRs |
| **Running as root** (no `USER`) | An escape or app bug has more power | `USER <numeric uid>` |
| **Mounting `docker.sock`** | Root on the host for anything inside | Socket proxy, rootless, or redesign |
| **`--privileged`** to "make it work" | Removes nearly all isolation | Specific `--device`/`--cap-add`/`--sysctl` |
| **Publishing every port to `0.0.0.0`** (`-p 5432:5432` on a public host) | Databases and admin UIs on the internet; ufw does not protect you | Loopback binding, internal networks, reverse proxy |
| **No resource limits** | One container can take down the host | `--memory`, `--cpus`, `--pids-limit` |
| **Fat runtime images** (compilers, `curl`, shells, `sudo`, SSH daemons) | More attack surface and CVEs | Multi-stage builds; minimal/hardened bases |
| **SSH servers inside containers** | Another daemon, keys and patching burden | `docker exec`/`kubectl exec` with RBAC and audit |
| **Treating image deletion/rewriting history as secret remediation** | Copies already exist | Rotate the credential |
| **Pulling random Hub images** | Typo-squats, miners, backdoors | Official/Verified/Hardened images; read the source; scan |
| **Disabling seccomp/AppArmor** to silence an error | Removes a key safety net | Diagnose the specific denial |
| **Using scanners once and forgetting** | New CVEs appear daily | Continuous scanning + scheduled rebuilds |
| **Unpinned CI actions/tools** | The 2026 Trivy incident: moved tags stole CI secrets | Pin to commit SHAs; least-privilege short-lived creds |
| **Long-lived static cloud keys in CI** | Large blast radius if stolen | OIDC federation and short-lived tokens |
| **Data and secrets in the container's writable layer** | Lost on removal; unaudited | Volumes and secret stores |
| **Assuming "it's in a container so it's safe"** | The kernel is shared | Defence in depth (section 10.1.3) |

---

## 10.23 Try it yourself

All commands below are safe on a development machine and touch only throw-away containers.

1. **See who you are.** `docker run --rm alpine id` shows `uid=0(root)`. Then `docker run --rm --user 10001:10001 alpine id`.
2. **Capability diff.** Compare `docker run --rm alpine sh -c 'grep CapEff /proc/self/status'` with the same command plus `--cap-drop ALL`, and with `--privileged` (look at how much larger the bitmask is, then forget you ever used `--privileged`).
3. **Read-only filesystem.** `docker run --rm --read-only alpine sh -c 'touch /x'` fails; add `--tmpfs /tmp` and `touch /tmp/x` works.
4. **Fork-bomb protection (safely).** `docker run --rm --pids-limit 20 alpine sh -c 'for i in $(seq 1 100); do sleep 30 & done; echo done'` and watch the errors once the limit is hit. (The container is disposable and the limit protects the host.)
5. **Memory limit.** `docker run --rm --memory 64m --memory-swap 64m python:3.12-slim python -c "x=bytearray(200*1024*1024)"` is killed (exit 137 in `docker run`'s status; check with `echo $?`).
6. **Port exposure.** Run `docker run -d --rm --name pubtest -p 8081:80 nginx:1.27-alpine` and note `docker ps` shows `0.0.0.0:8081`. Remove it (`docker rm -f pubtest`, your own test container) and rerun with `-p 127.0.0.1:8081:80`.
7. **Inspect a leak.** Build a throw-away image with `ENV FAKE_SECRET=hunter2`, then find it with `docker history --no-trunc` and `docker inspect`. Rebuild properly with a BuildKit secret mount and confirm it is gone.
8. **Scan.** `docker scout quickview <one of your images>` and `docker scout cves --only-severity critical,high --only-fixed <image>`. Pick one finding and walk the triage steps in 10.13.5.
9. **SBOM.** Build a tiny image with `docker buildx build --sbom=true --provenance=mode=max -t demo:sbom --load .` (if `--load` is not supported with attestations on your setup, use `--output type=docker` or push to a local registry; see the docs) and inspect with `docker buildx imagetools inspect`. This depends on your image store; check `docker info`.
10. **Pin a digest.** Use `docker buildx imagetools inspect python:3.12-slim` to get its digest and write a `FROM ...@sha256:...` line.
11. **Audit a Compose file.** Take any Compose file you use and go through the checklist in 10.21, noting every unchecked item as a ticket.
12. **(Linux host) Run docker-bench-security** and fix the first three WARNs that apply to your containers.

---

## 10.24 Chapter summary

* Containers share the host kernel; they are strong process isolation, **not** VM-grade isolation. Keep everything patched and stack defences.
* The Docker socket is root. Do not mount it; do not expose TCP without TLS; treat the `docker` group like `sudo`.
* Run non-root, drop all capabilities, set `no-new-privileges`, keep seccomp/AppArmor on, use read-only root filesystems, limit resources, avoid `--privileged`.
* Bind ports to `127.0.0.1` or publish only through a reverse proxy; remember Docker bypasses ufw.
* Choose trusted, minimal base images (including Docker Hardened Images, free under Apache 2.0 for the Community tier), pin digests, scan continuously (Trivy, Docker Scout, Grype), and triage by fixability, origin, severity and reachability.
* Generate SBOMs and provenance with BuildKit; sign with Cosign and **verify** at deploy; pin your CI tooling too (the 2026 Trivy compromise is the cautionary tale).
* Never put secrets in images, ENV/ARG, or git; use build secret mounts and runtime files or secret managers; if a secret leaks, **rotate it**.
* Secure registries with scoped tokens, MFA, mirrors and immutable tags; benchmark with CIS/docker-bench-security; detect with Falco; keep logs and audit trails.

Next: [Chapter 11: Production and Enterprise](11-production-and-enterprise.md), which scales these practices to orchestrators, policy, and organizations.
