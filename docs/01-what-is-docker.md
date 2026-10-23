# Chapter 1: What Is Docker?

Docker packages an application together with everything it needs to run (except the operating system kernel) into one portable unit. This chapter explains the problem Docker solves, how containers work inside the Linux kernel, the core terms (image, container, registry, Dockerfile), Docker's architecture, and where Docker ends and other tools begin.

**Prerequisites:** none. Docker does not need to be installed to read this chapter. [Chapter 2](02-installation-and-operating-systems.md) covers installation. Section 15 contains optional exercises that need Docker.

---

## Table of contents

1. [The problem: "it works on my machine"](#1-the-problem-it-works-on-my-machine)
2. [The shipping container analogy](#2-the-shipping-container-analogy)
3. [A short history of containers](#3-a-short-history-of-containers)
4. [Containers vs virtual machines](#4-containers-vs-virtual-machines)
5. [How containers actually work](#5-how-containers-actually-work)
6. [The four core nouns: image, container, registry, Dockerfile](#6-the-four-core-nouns-image-container-registry-dockerfile)
7. [Image layers and caching](#7-image-layers-and-caching)
8. [Docker architecture: what runs when you type `docker run`](#8-docker-architecture-what-runs-when-you-type-docker-run)
9. [Registries: Docker Hub and others](#9-registries-docker-hub-and-others)
10. [Docker Engine vs Docker Desktop vs the alternatives](#10-docker-engine-vs-docker-desktop-vs-the-alternatives)
11. [What is Docker Compose? (a first look)](#11-what-is-docker-compose-a-first-look)
12. [What Docker is NOT](#12-what-docker-is-not)
13. [Why this matters for AI developers](#13-why-this-matters-for-ai-developers)
14. [Common mistakes and misconceptions](#14-common-mistakes-and-misconceptions)
15. [Try it yourself](#15-try-it-yourself)
16. [Summary](#16-summary)

---

## 1. The problem: "it works on my machine"

An application is not only its code. It also depends on the environment it runs in. When that environment differs between machines, the same code behaves differently or fails.

| Hidden dependency | Example |
|---|---|
| Language runtime | Python 3.12 vs 3.9, Node 20 vs 22, Java 17 vs 21 |
| Libraries / packages | `numpy==1.26` vs `numpy==2.1` |
| System libraries | `libssl`, `libgomp`, `libcuda`, `glibc` vs `musl` |
| Operating system differences | Ubuntu vs Alpine vs Windows path separators |
| Configuration | Environment variables, config files |
| File layout | Folders that must exist, file permissions |
| Other services | A database, a cache, a message queue |

Before containers, teams used long README files, configuration management tools, or full virtual machines. These work but are slow, fragile, or heavy.

**Docker's approach:** package the application with everything it needs to run (except the OS kernel) into a single portable unit. The package is a **container image**. A running instance of it is a **container**.

Docker provides three properties:

1. **Reproducible:** the same image behaves the same on a laptop, a CI server, and production.
2. **Isolated:** two apps on one machine do not share dependencies. App A can use Python 3.8 while App B uses Python 3.13.
3. **Portable and fast:** an image is one artifact that runs on any host with a container runtime. A container starts in about a second or less.

---

## 2. The shipping container analogy

The standard intermodal shipping container is a steel box with fixed dimensions. Cranes, ships, trains, and trucks handle the box without knowing what is inside. Docker applies the same idea to software.

```
 BEFORE standard containers            AFTER standard containers
 ---------------------------           -------------------------------
 Every cargo type is special:          Every cargo is "a box":

  [sacks] [barrels] [crates]            +--------+ +--------+ +--------+
     |        |         |               | coffee | | cars   | | laptops|
  different  different different        +--------+ +--------+ +--------+
  cranes     carts     nets                  \         |         /
                                              \        |        /
                                          one crane, one ship, one truck
```

| Shipping | Docker |
|---|---|
| Standard steel box | Container image in a standard format (OCI) |
| Contents | App, language runtime, libraries |
| Cranes, ships, trucks | Docker, Kubernetes, cloud services: they handle any container the same way |
| Port | Registry (a place where images are stored and shared) |
| Shipping manifest | Dockerfile (a recipe describing what goes into the image) |

Limit of the analogy: a software container is not a box. It is a normal process on the host with isolation applied around it ([Section 5](#5-how-containers-actually-work)).

---

## 3. A short history of containers

Several names (LXC, containerd, runc, OCI, Moby, Podman) appear to "do containers". The history explains how they relate.

| Year | Event | Relevance |
|---|---|---|
| 1979 | Unix `chroot` | First way to give a process a different root folder. |
| 2000 | FreeBSD Jails | Stronger isolation on FreeBSD. |
| 2002 | Linux **namespaces** begin (mount namespace first) | Lets a process see a private view of the system. |
| 2004 | Solaris Zones | OS-level virtualization on Solaris. |
| 2006 to 2008 | Linux **cgroups** (control groups) merged into the kernel | Limits CPU, memory, and I/O for a group of processes. |
| 2008 | **LXC** (Linux Containers) | First complete container manager built from namespaces plus cgroups. |
| 2013 | **Docker** released (announced by Solomon Hykes of dotCloud at PyCon) | Added a friendly CLI, the Dockerfile format, layered images, and Docker Hub. The kernel features already existed. |
| 2014 | Docker 1.0; Google open-sources **Kubernetes** | Containers become mainstream; orchestration (running many containers across many machines) begins. |
| 2015 | **OCI** (Open Container Initiative) formed | Open standards for image format and runtime. Docker donated its image format and the `runc` code. |
| 2017 | Docker donates **containerd** to the CNCF (Cloud Native Computing Foundation); the open-source project is renamed **Moby** | Docker's engine is split into replaceable parts. |
| 2020 to 2021 | Docker Hub introduces pull rate limits; Docker Desktop gets paid terms for larger companies | Reason people look at alternatives (Podman, Rancher Desktop, Colima). |
| 2022 | Kubernetes 1.24 removes "dockershim" | Kubernetes talks to containerd directly. Docker-built images still run. Only Docker as a Kubernetes runtime was removed. |
| 2023 | Docker Engine 23.0 | BuildKit became the default builder. |
| 2025 | Docker Engine 29 | The containerd image store became the default for fresh installs. Docker added AI features such as Docker Model Runner. |
| 2026 | Docker Engine 29.9.x, Compose 5.x, Docker Desktop 4.96 on the test machine | Versions used in this guide. |

Verified vs. remembered: the Engine 29 containerd-image-store default and the current version numbers were checked against Docker's release notes and a live `docker version`. Older dates (1979, 2008, 2013, 2015, 2017) are widely documented and were not re-checked.

Docker combines older Linux kernel features, a developer-friendly CLI, a standard image format, and a public registry.

---

## 4. Containers vs virtual machines

Containers and virtual machines (VMs) both run multiple isolated environments on one machine, at different levels.

- **Virtual machine (VM):** a simulated computer, run by a **hypervisor** (VMware ESXi, Hyper-V, KVM, VirtualBox). Each VM has its own full operating system and its own kernel.
- **Kernel:** the core of an operating system. It talks to hardware (CPU, memory, disk, network) and decides which program does what. Linux is a kernel; Ubuntu, Debian, Fedora, and Alpine are Linux kernel plus different tools and libraries. Windows and macOS have their own kernels.
- **Container:** a process (or group of processes) running on the host's kernel with isolation applied. There is no second OS or kernel. All containers on a host share the host kernel.

```
        VIRTUAL MACHINES                              CONTAINERS
 +-----------+-----------+-----------+      +-----------+-----------+-----------+
 |   App A   |   App B   |   App C   |      |   App A   |   App B   |   App C   |
 +-----------+-----------+-----------+      +-----------+-----------+-----------+
 | Libraries | Libraries | Libraries |      | Libraries | Libraries | Libraries |
 +-----------+-----------+-----------+      +-----------+-----------+-----------+
 | Guest OS  | Guest OS  | Guest OS  |      |      Container runtime (Docker)   |
 | (kernel!) | (kernel!) | (kernel!) |      +-----------------------------------+
 +-----------+-----------+-----------+      |      HOST OS  (one shared kernel) |
 |         HYPERVISOR                |      +-----------------------------------+
 +-----------------------------------+      |            HARDWARE               |
 |      HOST OS (or bare metal)      |      +-----------------------------------+
 +-----------------------------------+
 |            HARDWARE               |
 +-----------------------------------+
```

| Property | Virtual machine | Container |
|---|---|---|
| What it virtualizes | Hardware (a whole simulated computer) | The OS's view (a private system view) |
| Kernel | Each VM has its own | Shared with the host |
| Typical size | Gigabytes | Megabytes to a few hundred MB |
| Startup time | Tens of seconds to minutes | Under a second to a few seconds |
| Memory overhead | Large (a whole OS in RAM) | Small (only the app's processes) |
| Density per host | Tens | Hundreds to thousands |
| Isolation strength | Strong (hardware-level boundary) | Weaker (kernel-level boundary; a kernel bug can affect all containers) |
| Can run a different OS than the host | Yes | No. A Linux container needs a Linux kernel. |
| Best for | Strong isolation, other operating systems, legacy workloads | Packaging apps, microservices, CI/CD, development environments |
| Portability unit | VM disk image (large) | Container image (small, layered) |

### Linux containers on Windows and macOS

On Windows and macOS, Docker Desktop starts a small hidden Linux VM. Containers run inside that VM and share its Linux kernel.

```
 Windows 11 / macOS laptop
 +----------------------------------------------------------+
 |  Docker Desktop                                          |
 |  +----------------------------------------------------+  |
 |  |  Hidden lightweight Linux VM                       |  |
 |  |  (Windows: WSL 2. Mac: Apple Virtualization        |  |
 |  |   Framework or Docker's own VMM)                   |  |
 |  |   +----------+ +----------+ +----------+           |  |
 |  |   |container | |container | |container |           |  |
 |  |   +----------+ +----------+ +----------+           |  |
 |  |         Linux kernel (in the VM)                   |  |
 |  +----------------------------------------------------+  |
 +----------------------------------------------------------+
```

On Windows with Docker Desktop, `docker run --rm alpine uname -r` prints a kernel version ending in `microsoft-standard-WSL2`, which is the WSL 2 kernel ([Section 15](#15-try-it-yourself)).

On a Linux server there is no VM in between. This is why Linux is the production OS for containers ([Chapter 2](02-installation-and-operating-systems.md)).

VMs and containers are layers, not rivals. In the cloud, containers commonly run inside VMs: VMs isolate customers from each other, and containers package and pack workloads within a customer.

---

## 5. How containers actually work

A container is not an object in the Linux kernel. It is a normal Linux process that is:

- placed in **namespaces** (control what it can see),
- restricted by **cgroups** (control what it can use),
- given a layered **filesystem** (controls which files it has).

| Feature | Meaning | Analogy |
|---|---|---|
| **Namespaces** | Control what a process can see | Blinkers on a horse: it sees only its own lane |
| **cgroups** | Control how much a process can use | A utility meter and circuit breaker for an apartment |
| **Union / overlay filesystem** | Gives the process its own files, built from stacked read-only layers | Transparent sheets stacked on a projector |

### 5.1 Namespaces: "what can this process see?"

A **namespace** wraps a global system resource so that processes inside it see their own private copy.

| Namespace | Isolates | Effect in a container |
|---|---|---|
| **PID** (process ID) | The list of running processes | The main process is PID 1. Host processes are invisible. |
| **NET** (network) | Interfaces, IP addresses, routing tables, ports | The container has its own `eth0`, IP, and port numbers. Two containers can both listen on port 80. |
| **MNT** (mount) | Filesystem mount points | The container sees its own root filesystem `/`. |
| **UTS** (Unix Time-sharing System) | Hostname and domain name | The container can have its own hostname. |
| **IPC** (inter-process communication) | Shared memory and message queues | Containers cannot read each other's shared memory. |
| **USER** | User and group ID mappings | `root` (UID 0) inside can map to an unprivileged user outside. Used by rootless mode and user-namespace remapping. |
| **CGROUP** | View of the cgroup hierarchy | The container sees only its own resource-accounting slice. |
| **TIME** | System clock offsets | Newer and rarely needed. |

Example (Linux, or inside WSL):

```bash
docker run --rm alpine ps
```

- `docker run`: create and start a container.
- `--rm`: delete the container when it exits.
- `alpine`: the image (a very small Linux distribution).
- `ps`: the command to run inside the container.

Output:

```
PID   USER     TIME  COMMAND
    1 root      0:00 ps
```

Only one process is visible, and it is `ps` itself, as PID 1. On the host the same process has a large PID such as 48213. The PID namespace provides the private numbering.

```bash
docker run --rm alpine hostname
```

Output (random ID; yours differs):

```
9f2c1a7d3b4e
```

By default the hostname is the first 12 characters of the container ID (UTS namespace).

**Creating a namespace by hand (Linux or WSL only, requires `sudo`):**

```bash
sudo unshare --pid --fork --mount-proc --uts bash
```

| Part | Meaning |
|---|---|
| `unshare` | Linux tool that runs a program in new namespaces |
| `--pid` | New PID namespace |
| `--fork` | Required so the new `bash` can become PID 1 in the namespace |
| `--mount-proc` | Remounts `/proc` so `ps` shows the new namespace's processes |
| `--uts` | Private hostname |

Inside that shell:

```bash
ps aux          # shows only 2 or 3 processes
hostname demo   # changes the hostname only inside this namespace
exit            # leave; the host hostname is unchanged
```

Docker combines all of these namespaces with cgroups and filesystems automatically.

### 5.2 cgroups: "how much can this process use?"

A **control group (cgroup)** is a kernel feature that groups processes and enforces limits and accounting for CPU time, memory, disk I/O, process count, and more. Without cgroups, one container could use all host RAM.

```bash
docker run --memory=512m --cpus=1.5 myapp
```

- `--memory=512m`: limit the container to 512 MiB of RAM.
- `--cpus=1.5`: limit it to 1.5 CPU cores' worth of time.

If the container exceeds its memory limit, the kernel's **OOM killer** (Out-Of-Memory killer) terminates it. This appears as **exit code 137** ([Chapter 3](03-core-cli-commands.md)).

cgroup v1 vs v2: modern distributions use cgroup v2 (a single hierarchy). Docker Engine 29's release notes say cgroup v1 is deprecated, with support continuing until at least May 2029. `docker info` on the test machine shows `Cgroup Version: 2`. Old tutorials that reference `/sys/fs/cgroup/memory/...` describe v1 paths.

### 5.3 Union / overlay filesystems: "what files does this process have?"

A container needs its own `/bin`, `/etc`, `/usr`, and app files. Copying a full Linux filesystem per container would be slow and wasteful. A **union filesystem** (Docker uses **overlayfs** on Linux) stacks several directories ("layers") and presents them as one merged directory.

```
              What the container sees at "/"
                  +---------------------+
                  |   MERGED VIEW       |
                  +----------^----------+
                             |  overlayfs merges these:
     +-----------------------+-----------------------+
     |  Container layer  (READ-WRITE, thin, per-container) |  <- upperdir: changes
     +-----------------------------------------------------+
     |  Image layer 3: COPY app/ /app          (read-only) |  \
     +-----------------------------------------------------+   \  lowerdir:
     |  Image layer 2: RUN pip install ...     (read-only) |   /  shared by all
     +-----------------------------------------------------+  /   containers from
     |  Image layer 1: base OS (python:3.13-slim) (read-only)|/    this image
     +-----------------------------------------------------+
```

Rules:

- **Image layers are read-only and shared.** Ten containers from one image share one copy of the layers on disk.
- Each container gets one thin **read-write layer** on top (the "container layer" or *upperdir*).
- **Copy-on-write (CoW):** when a container modifies a file in a lower layer, the file is first copied up into the writable layer and the copy is changed. The original is untouched. Deleting a file adds a "whiteout" marker that hides it in the merged view.
- **Deleting the container deletes the writable layer.** Containers are **ephemeral** (temporary). Persistent data belongs in **volumes**, which are storage outside the container layer ([Chapter 5](05-storage-and-networking.md)).

Storage driver vs image store: the classic Linux storage driver is `overlay2`. With the containerd image store (the default on fresh Engine 29 installs) the equivalent is the `overlayfs` snapshotter. `docker info` on the test machine reports `Storage Driver: overlayfs`. Both are overlay-based and need no configuration.

### 5.4 Other walls: capabilities, seccomp, AppArmor/SELinux

- **Linux capabilities:** the `root` privilege is split into about 40 smaller privileges (for example "bind to low ports", "change file ownership"). Docker drops most by default, so root inside a container is weaker than host root.
- **seccomp:** a filter that blocks dangerous **system calls** (requests a process makes to the kernel). Docker's default profile blocks roughly 40 to 50 of the 300+ syscalls.
- **AppArmor / SELinux:** mandatory access control systems that further restrict a container (AppArmor on Ubuntu/Debian, SELinux on Fedora/RHEL).

Details are in [Chapter 10 (Security)](10-security.md).

### 5.5 Putting it together

`docker run -d --memory=512m nginx` makes Docker and its helpers instruct the kernel to:

1. Create new PID, NET, MNT, UTS, IPC (and cgroup) namespaces.
2. Create a cgroup with a 512 MiB memory cap.
3. Assemble an overlay filesystem from the nginx image layers plus a new writable layer, and make it the root `/`.
4. Apply a capability set, a seccomp filter, and an AppArmor/SELinux profile.
5. Start the `nginx` process inside all of that.

```
  Host kernel (Linux)
  +---------------------------------------------------------------+
  |  Process: nginx (host PID 48213, container PID 1)             |
  |    in namespaces: pid, net, mnt, uts, ipc, cgroup             |
  |    in cgroup:     memory.max = 512 MiB                        |
  |    root fs:       overlay(lower = nginx layers, upper = RW)   |
  |    restricted by: capabilities, seccomp, AppArmor             |
  +---------------------------------------------------------------+
```

---

## 6. The four core nouns: image, container, registry, Dockerfile

### 6.1 Dockerfile: the recipe

A **Dockerfile** is a text file of instructions that builds an image.

```dockerfile
# Start from an existing image that has Python installed
FROM python:3.13-slim

# Set the working directory inside the image
WORKDIR /app

# Copy the dependency list and install it
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application code
COPY . .

# Command to run when a container starts
CMD ["python", "app.py"]
```

Every instruction is covered in [Chapter 4](04-dockerfile-deep-dive.md).

### 6.2 Image: the read-only package

An **image** is the result of building a Dockerfile. It is a read-only template containing a filesystem (app, libraries, a minimal OS userland) and metadata (default command, environment variables, exposed ports). An image does nothing until a container is started from it.

### 6.3 Container: a running instance of an image

A **container** is an instance created from an image: the image plus a writable layer plus a process. Many containers can be created from one image.

### 6.4 Registry: image storage

A **registry** is a server that stores and distributes images. `docker push` uploads an image to a registry; `docker pull` downloads it. See [Section 9](#9-registries-docker-hub-and-others).

### 6.5 How they relate

```
   Dockerfile ---- docker build ----> Image ---- docker push ----> Registry
   (recipe)                         (package)                     (warehouse)
                                       |  ^                          |
                              docker run |  | docker pull            |
                                       v  +---------------------------+
                                   Container
                                (running instance)
```

| Analogy | Dockerfile | Image | Container |
|---|---|---|---|
| Programming | Source code | Compiled program file | A running process |
| Object-oriented | Class definition | The class | An instance of the class |

### 6.6 The anatomy of an image name

```
   docker.io / library / nginx : 1.27 @ sha256:3f1c...
   \_______/   \______/  \___/   \___/   \_____________/
   registry   namespace  repo   tag      digest
```

| Part | Meaning | Default if omitted |
|---|---|---|
| **Registry** | Where the image lives | `docker.io` (Docker Hub) |
| **Namespace** | The user or organization that owns it | `library` (namespace for Docker Official Images) |
| **Repository** | The image's name | (required) |
| **Tag** | A label, usually a version | `latest` |
| **Digest** | A cryptographic hash (`sha256:...`) of the exact image contents | none |

- `nginx` means `docker.io/library/nginx:latest`.
- `ghcr.io/acme/chatbot:2.1` means registry `ghcr.io` (GitHub), owner `acme`, repository `chatbot`, tag `2.1`.

**Tags are mutable; digests are not.** A tag points at an image, and the publisher can move it. `nginx:latest` can be a different image next month. A digest identifies exact bytes and cannot change. For reproducible production deployments, pin by digest or by a specific version tag. `latest` does not mean "newest". It is only the tag used when none is given.

---

## 7. Image layers and caching

### 7.1 Every instruction makes a layer

Each filesystem-changing Dockerfile instruction produces a **layer**: a record of files added, changed, or deleted.

```
Dockerfile                                  Resulting image layers (bottom to top)
------------------------------------        ----------------------------------------
FROM python:3.13-slim                  -->  [ base layers from python:3.13-slim ]  (several)
WORKDIR /app                           -->  [ layer: creates /app ]       (tiny)
COPY requirements.txt .                -->  [ layer: requirements.txt ]   (tiny)
RUN pip install -r requirements.txt    -->  [ layer: installed packages ] (large)
COPY . .                               -->  [ layer: source code ]        (small)
CMD ["python","app.py"]                -->  (metadata only, no files)
```

Layers are **content-addressable**: each is identified by a hash of its contents. Identical layers are stored and downloaded once, even when many images use them. A second image based on `python:3.13-slim` pulls faster because the base layers already exist locally.

```bash
docker history nginx
```

`docker history <image>` lists the layers of an image and the instruction that created each ([Chapter 3](03-core-cli-commands.md)).

### 7.2 The build cache

On rebuild, Docker checks each step. If the instruction and its inputs are unchanged, it reuses the cached layer. If a step changed, that step and every later step are rebuilt.

```
 Rebuild after changing only app.py:

 FROM python:3.13-slim               CACHED
 WORKDIR /app                        CACHED
 COPY requirements.txt .             CACHED   (file unchanged)
 RUN pip install -r requirements.txt CACHED   (slow step skipped)
 COPY . .                            REBUILT  (app.py changed)
 CMD ...                             REBUILT
```

Instruction order therefore matters: put rarely changing steps (installing dependencies) before frequently changing steps (copying source). See [Chapter 4](04-dockerfile-deep-dive.md).

### 7.3 Benefits of layers

| Benefit | Explanation |
|---|---|
| Fast pulls/pushes | Only layers not already present are transferred. |
| Disk savings | Shared layers are stored once. |
| Fast builds | Unchanged steps come from cache. |
| Fast container start | Starting a container adds a thin writable layer instead of copying the image. |

Layers are additive. If one step copies a 1 GB file and a later step runs `rm` on it, the image stays 1 GB larger because the first layer still contains the file. The same applies to secrets: a password copied in one layer and deleted in the next can still be recovered from the image. Multi-stage builds and build secrets avoid this ([Chapter 4](04-dockerfile-deep-dive.md), [Chapter 10](10-security.md)).

---

## 8. Docker architecture: what runs when you type `docker run`

`docker run nginx` does not create the container directly. It sends a request through a chain of programs, each with one job.

### 8.1 The big picture

```
 +---------------+        REST API        +--------------------------------------------+
 | docker  (CLI) | ---------------------> |  dockerd  (Docker daemon / "Docker Engine")|
 | the client    |  over a Unix socket    |   - API server                             |
 |               |  /var/run/docker.sock  |   - images, networks, volumes, builds      |
 +---------------+  (Windows: named pipe) +----------------------+---------------------+
                                                                  | gRPC
                                                                  v
                                           +--------------------------------------------+
                                           |  containerd                                |
                                           |   - pulls/stores images, manages snapshots |
                                           |   - manages container lifecycle            |
                                           +----------------------+---------------------+
                                                                  | starts one per container
                                                                  v
                                           +--------------------------------------------+
                                           |  containerd-shim                           |
                                           |   - stays alive as the container's parent  |
                                           |   - lets dockerd/containerd restart safely |
                                           +----------------------+---------------------+
                                                                  | calls
                                                                  v
                                           +--------------------------------------------+
                                           |  runc  (OCI runtime)                       |
                                           |   - creates namespaces/cgroups/rootfs      |
                                           |   - execs the process, then EXITS          |
                                           +----------------------+---------------------+
                                                                  v
                                                        [ container process ]
```

### 8.2 Components

| Component | What it is | Job |
|---|---|---|
| **Docker client (`docker`)** | The command-line tool | Turns a command into an API request, sends it, prints the result. |
| **Docker daemon (`dockerd`)** | A background service (a **daemon** is a program that runs continuously in the background). Called "Docker Engine" when bundled with the API. | Exposes the REST API; manages images, networks, volumes, and builds; delegates running containers to containerd. |
| **containerd** | A high-level container runtime; graduated CNCF project, also used directly by Kubernetes | Pulls and stores images, manages snapshots and container lifecycle. |
| **containerd-shim** | A small helper process, one per container | Stays as the container's parent so containers survive restarts of `dockerd` or `containerd`; holds stdio open. |
| **runc** | A low-level OCI runtime (a small CLI tool written in Go) | Asks the kernel for namespaces, cgroups, and the root filesystem, starts the process, and exits. |
| **docker-init** | A small init program (`tini`) | Optional PID 1 inside the container (`--init`); forwards signals and reaps zombie processes. |

Real output from the test machine (`docker version`, trimmed):

```
Client:
 Version:           29.9.2
 API version:       1.56
 Go version:        go1.26.10
 OS/Arch:           windows/amd64
 Context:           desktop-linux

Server: Docker Desktop 4.96.3 (244472)
 Engine:
  Version:          29.9.2
  API version:      1.56 (minimum version 1.40)
  OS/Arch:          linux/amd64
 containerd:
  Version:          v2.4.2
 runc:
  Version:          1.5.2
 docker-init:
  Version:          0.19.0
```

The client is `windows/amd64` and the server is `linux/amd64`: the client runs on Windows and the engine runs in the hidden Linux VM. Client and daemon are separate programs and can run on different machines (see `docker context`, Chapters 3 and 11).

### 8.3 Life of `docker run hello-world`

```
 1. CLI:        docker run hello-world
 2. CLI  -> POST /containers/create to dockerd (API call)
 3. dockerd:    image available locally? No ->
 4.    dockerd (via containerd) pulls the image from the registry (Docker Hub)
 5.    image layers are unpacked into containerd's snapshot store
 6. dockerd -> containerd: create and start a container from this image
 7. containerd launches a containerd-shim
 8. shim runs runc with an OCI "bundle" (rootfs + config.json)
 9. runc sets up namespaces, cgroups, filesystem, security filters
10. runc starts the image's program and exits
11. shim watches the process and streams its stdout back
12. CLI prints the output; the process ends; the container is "Exited"
```

### 8.4 The API and the socket (security-relevant)

The CLI and daemon communicate through a REST API over a local **socket**. On Linux this is the Unix socket `/var/run/docker.sock`. On Windows Docker Desktop it is a named pipe (for example `npipe:////./pipe/dockerDesktopLinuxEngine`). Anything that can use this socket can ask the daemon to start containers, including privileged ones that mount the host disk. **Access to the Docker socket is effectively root access to the machine.** Do not expose it over the network without TLS, and avoid mounting `/var/run/docker.sock` into containers ([Chapter 10](10-security.md)).

### 8.5 OCI: the open standards

The **Open Container Initiative (OCI)**, a Linux Foundation project, defines three specifications so tools from different vendors interoperate.

| OCI spec | Defines | Effect |
|---|---|---|
| **Image spec** | Image format (layers, manifest, config) | An image built with Docker runs under Podman, containerd, Kubernetes, and vice versa. |
| **Runtime spec** | How to run a container from an unpacked bundle (`config.json`) | `runc` implements it; alternatives (`crun`, `youki`, gVisor's `runsc`, Kata) implement the same spec. |
| **Distribution spec** | The HTTP API registries use to push and pull images | Any compliant registry (Docker Hub, GHCR, ECR, Harbor) works with any compliant client. |

In practice, "Docker image" and "OCI image" are the same thing.

### 8.6 Where Kubernetes fits (preview)

**Kubernetes** (K8s) is an **orchestrator**: it decides which machines run which containers, restarts failed containers, scales them, and configures networking. It uses containerd (or CRI-O) directly and does not need `dockerd`. Docker builds images; Kubernetes runs them at scale ([Chapter 11](11-production-and-enterprise.md)).

---

## 9. Registries: Docker Hub and others

A registry stores images in **repositories**. A repository is a collection of tagged versions of one image (for example `nginx` with tags `1.27`, `latest`, `alpine`).

### 9.1 Docker Hub

**Docker Hub** (`hub.docker.com`) is the default public registry. `docker pull nginx` pulls from it.

| Kind | What it is | Trust level |
|---|---|---|
| **Docker Official Images** | Curated set (`nginx`, `postgres`, `python`, `redis`, `ubuntu`) maintained with Docker's review. They use the `library/` namespace, so no username is written. | High. Still scan them. |
| **Verified Publisher** | Images from commercial vendors verified by Docker (badge on Hub). | Good |
| **Docker-Sponsored Open Source** | Open-source projects whose hosting Docker sponsors. | Good |
| **Community images** | `someuser/something`. Anyone can publish. | Unknown. Inspect before use; do not run unknown images with privileges or secrets. |

**Pull rate limits.** Per Docker's documentation (verified while writing): unauthenticated users get 100 pulls per 6 hours, counted per IPv4 address (or IPv6 /64 subnet). A free authenticated Personal account gets 200 pulls per 6 hours. Pro, Team, and Business accounts get unlimited pulls. Shared corporate networks and CI hit the limit because many machines share one IP. Fixes: `docker login`, a registry mirror or pull-through cache, or hosting base images internally. The error looks like `toomanyrequests: You have reached your pull rate limit`.

### 9.2 Other registries

| Registry | Run by | Typical use |
|---|---|---|
| **GitHub Container Registry (GHCR)**, `ghcr.io` | GitHub | Images next to code, integrated with GitHub Actions |
| **Amazon ECR** | AWS | Images for ECS/EKS/Lambda |
| **Google Artifact Registry** | Google Cloud | Images for GKE/Cloud Run (replaced Container Registry) |
| **Azure Container Registry (ACR)** | Microsoft | Images for AKS and Azure services |
| **Quay.io** | Red Hat | Public and private repos, built-in scanning |
| **GitLab Container Registry** | GitLab | Built into GitLab projects |
| **Harbor** | CNCF open source | Self-hosted registry with scanning, signing, replication |
| **`registry:2` (CNCF Distribution)** | Open source | Reference registry runnable in one command |

All speak the OCI distribution protocol, so commands are identical. Only the host part of the image name changes: `docker pull ghcr.io/owner/image:tag`.

### 9.3 Public vs private

- **Public repositories:** anyone can pull; no login needed.
- **Private repositories:** require `docker login` first.
- **Air-gapped / offline:** export images with `docker save` to a tar file, move it, and import it with `docker load` ([Chapter 3](03-core-cli-commands.md)).

---

## 10. Docker Engine vs Docker Desktop vs the alternatives

"Docker" refers to several different things.

| Name | What it is |
|---|---|
| **Docker (the company)** | Docker, Inc., which sponsors much of the ecosystem and sells subscriptions and tools. |
| **Moby** | The open-source upstream project from which Docker Engine is assembled. |
| **Docker Engine** | The open-source runtime stack: `dockerd` + REST API + containerd + runc (and the `docker` CLI). Runs natively on Linux. Free. Used on production servers. |
| **Docker CLI** | The `docker` command, plus plugins such as `docker compose` and `docker buildx`. |
| **Docker Desktop** | A desktop application for Windows, macOS, and Linux that bundles Docker Engine (in a lightweight VM), the CLI, Compose, Buildx, a GUI dashboard, optional single-node Kubernetes, Docker Scout (vulnerability analysis), extensions, and Docker Model Runner (local AI models). It has its own license terms ([Chapter 2](02-installation-and-operating-systems.md)). |
| **Docker Hub** | The public registry service. |

```
   +---------------------- Docker Desktop (Windows / macOS / Linux) ---------------------+
   |  GUI dashboard | Compose | Buildx | Scout | Kubernetes (optional) | Model Runner     |
   |  +------------------- hidden Linux VM -------------------------------------------+  |
   |  |        +------------- Docker Engine (dockerd + containerd + runc) -----+      |  |
   |  |        |        containers run here                                    |      |  |
   |  +--------+-------------------------------------------------------------+------+  |
   +--------------------------------------------------------------------------------------+

   On a Linux server only the middle box is installed: Docker Engine (+ CLI + plugins).
```

### 10.1 The alternatives

Because the standards are open (OCI), other tools can build and run the same images.

| Tool | What it is | Difference from Docker | Typical use |
|---|---|---|---|
| **Podman** | Daemonless container engine from Red Hat; CLI nearly drop-in compatible (`alias docker=podman`) | No central root daemon; each container is a child of the user's process; rootless by default; has Kubernetes-style "pods" | Security-conscious Linux/RHEL environments; avoiding Docker Desktop licensing |
| **Podman Desktop** | Open-source GUI for Podman | GUI | Windows/macOS users who prefer Podman |
| **containerd + nerdctl** | containerd with a Docker-compatible CLI | Minimal, no dockerd | Kubernetes nodes, advanced users |
| **CRI-O** | Runtime built only for Kubernetes | Not for general developer use | OpenShift/Kubernetes nodes |
| **Rancher Desktop** | Free desktop app with containerd or dockerd | Open source, includes K8s | Free alternative to Docker Desktop |
| **Colima / Lima** | Command-line Linux VMs on macOS (and Linux) running Docker or containerd | Lightweight, CLI-only | Mac users avoiding Docker Desktop |
| **OrbStack** | Commercial, fast macOS app | Fast file sharing on Mac | Mac power users |
| **Buildah** | Builds OCI images without a daemon | Build only | CI pipelines |
| **Kaniko / BuildKit** | Build images where Docker cannot run (for example inside Kubernetes) | Build only | CI/CD |
| **LXC / LXD (Incus)** | "System containers" that behave like lightweight VMs | Full-OS containers, not app containers | Hosting many small "servers" |

This guide uses Docker. It is the most documented option and other tools copy its commands, so most of the content transfers.

---

## 11. What is Docker Compose? (a first look)

Real applications have several parts. A typical AI app might need a web/API container (FastAPI), a database (Postgres), a vector database (Qdrant or pgvector), a cache (Redis), and possibly a worker and an LLM server. Starting each with a separate `docker run` requires long commands with ports, volumes, networks, environment variables, and a correct startup order.

**Docker Compose** describes the whole multi-container application in one YAML file (`compose.yaml`) and starts, stops, and connects everything with one command.

```yaml
# compose.yaml
services:
  api:
    build: .                      # build the image from the Dockerfile in this folder
    ports:
      - "8000:8000"               # host port 8000 -> container port 8000
    environment:
      DATABASE_URL: postgresql://app:example@db:5432/appdb
    depends_on:
      - db

  db:
    image: postgres:17            # use an existing image
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: example  # demo value only; real secrets never go in the file
      POSTGRES_DB: appdb
    volumes:
      - dbdata:/var/lib/postgresql/data   # persistent storage

volumes:
  dbdata:
```

```bash
docker compose up -d     # create networks/volumes, build, start everything in the background
docker compose ps        # show what is running
docker compose down      # stop and remove it all
```

- `docker compose up`: read `compose.yaml`, create a private network (services reach each other by name, so `api` connects to `db` at hostname `db`), create the volume, build or pull images, and start the containers.
- `-d` (detached): run in the background.
- `docker compose ps`: list the project's containers.
- `docker compose down`: stop and remove the containers and network (named volumes are kept unless `-v` is added).

### 11.1 `docker compose` vs `docker-compose`

| | Compose v1 (legacy) | Compose v2 and later (current) |
|---|---|---|
| Command | `docker-compose` (hyphen) | `docker compose` (space) |
| Implementation | Separate Python program | Plugin built into the Docker CLI, written in Go |
| Status | End of life. Docker stopped updating it in 2023; it is no longer shipped with Docker Desktop. | Actively developed |
| Install | Separate `pip`/binary install | Included with Docker Desktop, or the `docker-compose-plugin` package on Linux |
| Version on the test machine | n/a | `Docker Compose version v5.6.1` |

Version note: Compose version numbers jumped from the 2.x line to 5.x (the test machine has v5.6.1, and Docker's Compose release page lists v5.6.1, released October 2026). Unverified: Docker's pages do not state why the numbering jumped. `docker compose` (space) is the current command and is the same tool as "Compose v2" in older tutorials. For tutorials using `docker-compose`, replace the hyphen with a space.

The `version:` field: older tutorials start YAML with `version: "3.8"`. Modern Compose follows the Compose Specification and ignores this field (it may warn). Omit it.

Full coverage (networks, volumes, health checks, profiles, `depends_on` conditions, watch mode, GPUs) is in [Chapter 6](06-docker-compose.md).

---

## 12. What Docker is NOT

### 12.1 Docker is not a virtual machine

There is no guest OS and no separate kernel.

- A container can only run software built for the host kernel's type (Linux containers need a Linux kernel).
- A kernel exploit inside a container can affect the host and other containers.
- Containers are meant to be ephemeral: stateless and disposable, not servers to SSH into and configure by hand.
- Typically there is one main process per container, not a whole "mini server" with init, SSH, cron, and a database.

### 12.2 Docker is not a security boundary by default

Isolation is not the same as a hardened security boundary. By default:

- Containers share the host kernel.
- The default user inside a container is often root (UID 0) unless the image sets another.
- Anyone who can run `docker` commands (on Linux, anyone in the `docker` group) can become root on the host, for example with `docker run -v /:/host --privileged ...`.
- Published ports can bypass host firewalls such as `ufw`, because Docker edits `iptables` rules directly (Docker's docs describe this).
- An image can contain malware or vulnerable libraries.

Hardening options: non-root users, dropped capabilities, read-only filesystems, rootless mode, user namespaces, seccomp/AppArmor profiles, image scanning, signed images ([Chapter 10](10-security.md)). For untrusted multi-tenant code, use stronger isolation such as gVisor, Kata Containers, or Firecracker microVMs.

### 12.3 Docker is not an orchestrator

Plain Docker runs containers on one machine. It does not by itself:

- spread containers across servers,
- move workloads from a failed server to another,
- do zero-downtime rolling updates across a cluster,
- auto-scale on load.

Orchestrators do this: **Kubernetes** (industry standard), Docker Swarm (Docker's built-in clustering mode, still present but less used), HashiCorp Nomad, or managed services (AWS ECS, Google Cloud Run, Azure Container Apps). Docker Compose is not an orchestrator either: it manages a multi-container app on a single host. See [Chapter 11](11-production-and-enterprise.md) for when to move beyond it.

### 12.4 Docker is not a backup or data store

Data in a container's writable layer is lost when the container is removed. Persistent data belongs in volumes or bind mounts, which need their own backups ([Chapter 5](05-storage-and-networking.md)).

### 12.5 Docker does not replace dependency management

Docker freezes a working environment. It does not fix an unpinned `requirements.txt`: `pip install numpy` with no version can produce different images over time. Pin dependencies and base images.

### 12.6 Docker is not always faster

On Linux a container has near-native CPU speed. On Windows and macOS there is a VM layer, and file access across the host/VM boundary (bind mounts of project folders) can be much slower ([Chapter 2](02-installation-and-operating-systems.md)). GPU workloads need extra setup ([Chapter 8](08-docker-for-ai-and-ml.md)).

---

## 13. Why this matters for AI developers

| AI/ML problem | How Docker helps |
|---|---|
| PyTorch needs a specific CUDA and cuDNN version | Use an NVIDIA CUDA base image. CUDA user-space libraries come in the image; only the GPU driver lives on the host. |
| Library version conflicts between projects | One image per project with pinned versions. |
| Shipping a model API | Package FastAPI plus model weights (or a downloader) into an image and deploy it to any cloud. |
| Local stack of Postgres + pgvector + Qdrant + Redis + API | Docker Compose: one command. |
| Running an open LLM locally | **Docker Model Runner** (`docker model run ...`) in Docker Desktop, or containers such as Ollama or vLLM. |
| Reproducible training runs | A frozen image plus pinned data versions. |
| Deploying on Kubernetes | Every Kubernetes workload starts as a container image. |

**Docker Model Runner (DMR)** pulls and runs AI models (such as `ai/smollm2`) with `docker model run`. Per Docker's documentation (verified while writing): enable it in Docker Desktop under **Settings > AI**, or on Linux Docker Engine by installing the `docker-model-plugin` package. Models are pulled from Docker Hub (or Hugging Face) as OCI artifacts. Engine-side TCP access is enabled by default on port 12434; Docker Desktop lets you enable host-side TCP and choose the port. The docs name llama.cpp and vLLM as inference backends. Unverified: every sub-command and GPU detail. [Chapter 8](08-docker-for-ai-and-ml.md) covers this and flags what could not be verified.

---

## 14. Common mistakes and misconceptions

| Misconception | Reality |
|---|---|
| A container is a small VM. | It is a process with namespace/cgroup/overlay isolation, sharing the host kernel. |
| Deleting the container keeps the data safe. | Deleting a container deletes its writable layer. Use volumes. |
| `latest` means the newest version. | It is a default tag name and can be old or changed at any time. Pin versions. |
| Docker is secure because it is isolated. | Isolation is not hardening. Default root user, shared kernel, and `docker` group membership (root-equivalent) remain. |
| Put app, DB, and cache in one container. | One main process per container; combine several with Compose. |
| `docker compose` and `docker-compose` are different tools. | Same product family. The hyphenated one is the retired v1. |
| Windows and Linux containers are interchangeable. | A Linux image cannot run on the Windows container engine, and vice versa. |
| Deleting a file in a later Dockerfile step shrinks the image or removes a secret. | Earlier layers still contain it. Use multi-stage builds. |
| Kubernetes replaced Docker. | Kubernetes stopped using `dockershim` (the Docker adapter). Docker-built OCI images still run on Kubernetes. |

---

## 15. Try it yourself

These steps need Docker installed ([Chapter 2](02-installation-and-operating-systems.md)). On Windows and macOS, Docker Desktop must be running.

### Exercise 1: hello-world and the PID namespace

```bash
docker run hello-world
docker run --rm alpine ps
```

Expected: a "Hello from Docker!" message (annotated in [Chapter 2](02-installation-and-operating-systems.md#7-verifying-your-installation-with-hello-world)), then one or two lines of `ps` output with `ps` as PID 1.

### Exercise 2: containers share the host kernel

```bash
docker run --rm alpine uname -r
docker run --rm ubuntu uname -r
docker run --rm debian uname -r
```

`uname -r` prints the kernel version. Expected: all three print the same version, because different distributions (userland) share one kernel. On Windows with Docker Desktop the value ends in `microsoft-standard-WSL2` (`6.18.54.1-microsoft-standard-WSL2` on the test machine).

### Exercise 3: layers are shared

```bash
docker pull python:3.13-slim
docker pull python:3.13-slim   # second pull transfers nothing
docker image ls                # list local images
```

Optional: `docker history python:3.13-slim` shows the layers.

### Exercise 4: containers are ephemeral

```bash
docker run --name scratch alpine sh -c "echo hello > /note.txt && cat /note.txt"
docker rm scratch                           # delete the container
docker run --rm alpine cat /note.txt        # a new container from the same image
```

- `--name scratch`: give the container a name.
- `sh -c "..."`: run the quoted string in a shell.

Expected: the first command prints `hello`; the last fails with `cat: can't open '/note.txt': No such file or directory`. The file existed only in the first container's writable layer.

### Exercise 5 (Linux/WSL only): `unshare`

Run the `unshare` command from [Section 5.1](#51-namespaces-what-can-this-process-see) and confirm `ps aux` shows only a few processes.

### Clean up

```bash
docker image rm alpine ubuntu debian hello-world python:3.13-slim
```

`docker image rm` removes images. `docker rm` removes containers ([Chapter 3](03-core-cli-commands.md)).

---

## 16. Summary

- A container is a process, not a machine: namespaces limit what it sees, cgroups limit what it uses, overlayfs provides its files.
- Images are layered, read-only, shared, and cached. A container is an image plus a thin writable layer and is disposable.
- The `docker` command is a client. `dockerd` manages, `containerd` runs the lifecycle, `runc` creates the container. All follow OCI standards.
- Docker Engine is the free Linux runtime. Docker Desktop packages Engine in a hidden Linux VM for Windows and macOS.
- Compose describes multi-container apps in one file. Orchestrators such as Kubernetes run containers across machines.
- Docker provides isolation, not hardened security. Persistent data belongs in volumes.

---

**Next:** [Chapter 2: Installation and Operating Systems](02-installation-and-operating-systems.md)
