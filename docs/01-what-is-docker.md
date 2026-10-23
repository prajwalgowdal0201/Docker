# Chapter 1: What Is Docker?

> **What you will learn in this chapter**
> - The real problem Docker solves ("it works on my machine") and why it matters
> - A short, honest history of containers, so the names you see online make sense
> - Containers vs virtual machines: what is actually different
> - How containers really work inside the Linux kernel (namespaces, cgroups, overlay filesystems), explained without jargon overload
> - The four core nouns: **image, container, registry, Dockerfile**
> - Image layers and the build cache
> - Docker's architecture: client, daemon, containerd, runc, and the OCI standards
> - Registries (Docker Hub and friends)
> - Docker Engine vs Docker Desktop vs Podman and other alternatives
> - A first look at Docker Compose
> - What Docker is **not**
>
> **Time to read:** about 45 to 60 minutes. **Prerequisites:** none. You do not need Docker installed yet. In [Chapter 2](02-installation-and-operating-systems.md) we install it; the "Try it yourself" box at the end of this chapter is optional until then.

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
16. [Glossary of terms introduced in this chapter](#16-glossary-of-terms-introduced-in-this-chapter)
17. [Mental model summary](#17-mental-model-summary)
18. [Check your understanding](#18-check-your-understanding)

---

## 1. The problem: "it works on my machine"

Every software engineer eventually says (or hears) this sentence:

> "But it works on my machine!"

Here is a typical story. You write a Python app on your laptop. It uses Python 3.12, a library called `numpy` at a particular version, a system package called `libgomp`, an environment variable called `API_KEY`, and it expects a folder `/data` to exist. It runs perfectly.

You hand it to a teammate. Their laptop has Python 3.9, a different `numpy`, no `libgomp`, no `API_KEY`, and no `/data` folder. It crashes. You spend two hours comparing machines. Then you deploy it to a server: that server runs a different Linux distribution with a different C library, and it crashes *differently*.

The root cause is simple: **your program is not just your code.** It is your code plus everything it silently depends on:

| Hidden dependency | Example |
|---|---|
| Language runtime | Python 3.12 vs 3.9, Node 20 vs 22, Java 17 vs 21 |
| Libraries / packages | `numpy==1.26` vs `numpy==2.1` |
| System libraries | `libssl`, `libgomp`, `libcuda`, `glibc` vs `musl` |
| Operating system quirks | Ubuntu vs Alpine vs Windows path separators |
| Configuration | Environment variables, config files |
| File layout | Folders that must exist, permissions |
| Other services | A database, a cache, a message queue |

Before containers, teams tried to solve this with long README files ("install these 14 things in this order"), configuration management tools, or full virtual machines. All of them worked, but all were slow, fragile, or heavy.

**Docker's core idea:** *package the application together with everything it needs to run (except the operating system kernel), into a single portable unit that runs the same way everywhere.*

That unit is called a **container image** (the package) and, when running, a **container**.

### The three promises of Docker

1. **Reproducible:** the same image behaves the same on your laptop, your teammate's laptop, the CI server, and production.
2. **Isolated:** two apps on the same machine cannot step on each other's dependencies. App A can use Python 3.8 while App B uses Python 3.13, with no conflict.
3. **Portable and fast:** an image is one artifact you can copy anywhere that has a container runtime, and a container starts in about a second (often less), not minutes.

---

## 2. The shipping container analogy

Docker's logo is a whale carrying shipping containers, and the name comes from the same idea.

Before 1956, cargo shipping was chaos. Sacks of coffee, barrels, crates of machine parts, and cars were all loaded by hand, each needing different handling. Then the standard **intermodal shipping container** appeared: a steel box with fixed dimensions. It did not matter what was inside. Cranes, ships, trains, and trucks only needed to understand the *box*, not the contents.

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

Software containers do the same for applications:

| Shipping | Docker |
|---|---|
| Steel box with standard size | Container image in a standard format (OCI) |
| Contents (coffee, cars, ...) | Your app, language runtime, libraries |
| Cranes, ships, trucks | Docker, Kubernetes, cloud services: they handle *any* container the same way |
| Port | Registry (a place where images are stored and shared) |
| Shipping manifest listing contents | Dockerfile (a recipe describing what goes into the image) |

The analogy has a limit, and it is an important one that we will return to: a shipping container is a *box*, but a software container is really *a normal process on the host with walls built around it*. We will make that precise in [Section 5](#5-how-containers-actually-work).

---

## 3. A short history of containers

You do not need to memorize dates, but knowing the story explains why there are so many names (LXC, containerd, runc, OCI, Moby, Podman...) that all appear to "do containers".

| Year | Event | Why it matters |
|---|---|---|
| 1979 | Unix `chroot` | First way to give a process a different "root" folder. The grandparent of container filesystems. |
| 2000 | FreeBSD Jails | A much stronger isolation concept on FreeBSD. |
| 2002 | Linux **namespaces** begin (mount namespace first) | The Linux feature that makes a process see its own private view of the system. |
| 2004 | Solaris Zones | Commercial-grade OS-level virtualization on Solaris. |
| 2006 to 2008 | Linux **cgroups** ("control groups") merged into the kernel | The Linux feature that limits how much CPU/memory/IO a group of processes can use. |
| 2008 | **LXC** (Linux Containers) | First complete container manager built from namespaces plus cgroups. Powerful but hard to use. |
| 2013 | **Docker** is released (announced by Solomon Hykes of dotCloud at PyCon) | Docker did not invent the underlying kernel features. It made them *easy*: a friendly CLI, a simple recipe format (Dockerfile), layered images, and a public sharing site (Docker Hub). |
| 2014 | Docker 1.0; Google open-sources **Kubernetes** | Containers go mainstream; orchestration (running many containers across many machines) is born. |
| 2015 | **OCI** (Open Container Initiative) formed | Industry agreed on open standards for the image format and runtime, so containers are not owned by one company. Docker donated its image format and the `runc` runtime code. |
| 2017 | Docker donates **containerd** to the CNCF (Cloud Native Computing Foundation); the open-source project is renamed **Moby** | Docker's engine is split into replaceable parts. |
| 2020 to 2021 | Docker Hub introduces pull rate limits; Docker Desktop gets paid subscription terms for larger companies | Explains why people look for alternatives (Podman, Rancher Desktop, Colima, etc.). |
| 2022 | Kubernetes 1.24 removes "dockershim" | Kubernetes talks to containerd directly instead of through Docker. Docker-built images still run fine everywhere. Only Docker as a *Kubernetes runtime* was removed. |
| 2023 | Docker Engine 23.0 | BuildKit became the default image builder. |
| 2025 | Docker Engine 29 | The containerd image store became the default for fresh installs. Docker also adds AI-oriented features such as Docker Model Runner. |
| 2026 (now) | Docker Engine 29.9.x, Compose 5.x, Docker Desktop 4.96 on the author's test machine | The versions used when writing this guide. |

> **Verified vs. remembered:** The Engine 29 containerd-image-store default and the current version numbers were verified against Docker's official release notes and a live `docker version` while writing this chapter. Older historical dates (1979, 2008, 2013, 2015, 2017) are well-known facts from memory and widely documented; the exact day is not important.

### The takeaway

Docker = **old Linux kernel features** + **a great developer experience** + **a standard image format** + **a public registry**. The kernel features existed for years. Docker's genius was packaging them so a beginner could run `docker run` and have something working in seconds.

---

## 4. Containers vs virtual machines

Both containers and virtual machines (VMs) let you run multiple isolated environments on one physical machine. They do it at different levels.

### What is a virtual machine?

A **virtual machine (VM)** is a complete *pretend computer*, simulated in software by a program called a **hypervisor** (examples: VMware ESXi, Hyper-V, KVM, VirtualBox). Each VM has its own full operating system, including its own **kernel**.

> **What is a kernel?** The kernel is the core of an operating system. It is the part that talks directly to the hardware (CPU, memory, disk, network) and decides which program gets to do what. Linux is a kernel (Ubuntu, Debian, Fedora, and Alpine are all Linux kernel plus different collections of tools and libraries on top). Windows has its own kernel; macOS has another.

### What is a container?

A **container** is an ordinary process (or group of processes) running on the host's kernel, but with strict walls around it so that it *believes* it is alone on the machine. There is **no second operating system and no second kernel**. All containers on a host share that host's kernel.

### The diagram

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
 |            HARDWARE               |       Each container = a process with
 +-----------------------------------+       walls, NOT a whole computer.
  Each VM boots a full OS: GBs, minutes.      Starts in ~1 second, MBs of overhead.
```

### Comparison table

| Property | Virtual machine | Container |
|---|---|---|
| What it virtualizes | Hardware (a whole pretend computer) | The operating system's view (a pretend private system) |
| Kernel | Each VM has its own | **Shared** with the host |
| Typical size | Gigabytes (full OS disk) | Megabytes to a few hundred MB (often tens of MB for slim images) |
| Startup time | Tens of seconds to minutes | Usually under a second to a few seconds |
| Memory overhead | Large (a whole OS in RAM) | Small (just your processes) |
| Density per host | Tens | Hundreds to thousands |
| Isolation strength | **Strong** (hardware-level boundary) | **Weaker** (kernel-level boundary; a kernel bug can affect all) |
| Can run a different OS than the host? | Yes (Windows VM on a Linux host) | **No.** A Linux container needs a Linux kernel. |
| Best for | Strong isolation, running different operating systems, legacy full-OS workloads | Packaging and deploying apps, microservices, CI/CD, development environments |
| Portability unit | VM disk image (large) | Container image (small, layered) |

### "Wait, I run Linux containers on Windows and macOS. How, if they share the host kernel?"

Excellent question, and it trips up many beginners. The honest answer: **on Windows and macOS, Docker quietly starts a small Linux virtual machine, and your containers run inside that VM, sharing *that VM's Linux kernel*.**

```
 Your Windows 11 / macOS laptop
 +----------------------------------------------------------+
 |  Docker Desktop (the app you click)                      |
 |  +----------------------------------------------------+  |
 |  |  Hidden lightweight Linux VM                       |  |
 |  |  (on Windows: WSL 2. On Mac: Apple Virtualization  |  |
 |  |   Framework or Docker's own VMM)                   |  |
 |  |   +----------+ +----------+ +----------+           |  |
 |  |   |container | |container | |container |  <- share |  |
 |  |   +----------+ +----------+ +----------+  the VM's |  |
 |  |         Linux kernel (in the VM)           kernel   |  |
 |  +----------------------------------------------------+  |
 +----------------------------------------------------------+
```

You can prove this to yourself later. On a Windows machine running Docker Desktop, `docker run --rm alpine uname -r` prints a kernel version ending in `microsoft-standard-WSL2`, because the "Linux" under your container is Microsoft's WSL 2 kernel, not anything Docker invented. (We will do exactly this in [Section 15](#15-try-it-yourself).)

On a real Linux server there is no VM in between: containers run directly on the host kernel. That is why **Linux is the production operating system for containers** ([Chapter 2](02-installation-and-operating-systems.md) explains this in detail).

### Which one should I use?

They are not rivals; they are layers. In the cloud, you often run **containers inside VMs** (a cloud server is a VM; Docker runs containers inside it). VMs give you hard isolation between customers; containers give you packaging and density within a customer's workloads.

---

## 5. How containers actually work

Here is the secret that demystifies everything:

> **A container is not a real "thing" in the Linux kernel. There is no `container` object. A container is a normal Linux process that has been put inside a set of *namespaces* (what it can see), restricted by *cgroups* (what it can use), and given a layered *filesystem* (what files it has).**

Three Linux features, combined:

| Feature | One-line meaning | Analogy |
|---|---|---|
| **Namespaces** | Control what a process can **see** | Blinkers on a horse: it only sees its own lane |
| **cgroups** | Control how much a process can **use** | A utility meter and a circuit breaker for an apartment |
| **Union / overlay filesystem** | Gives the process its **own files**, built from stacked read-only layers | Transparent sheets stacked on a projector: you see the combined picture |

### 5.1 Namespaces: "what can this process see?"

A **namespace** wraps a global system resource so that processes inside the namespace see their *own private copy*. The Linux kernel offers several types:

| Namespace | What it isolates | What it means for a container |
|---|---|---|
| **PID** (process ID) | The list of running processes | Inside the container, your main process is PID 1 and you cannot see the host's other processes. |
| **NET** (network) | Network interfaces, IP addresses, routing tables, ports | The container has its own `eth0`, its own IP, and its own port numbers. Two containers can both listen on port 80 without conflict. |
| **MNT** (mount) | The filesystem mount points | The container sees its own root filesystem (`/`), not the host's. |
| **UTS** (Unix Time-sharing System) | Hostname and domain name | The container can have its own hostname. |
| **IPC** (inter-process communication) | Shared memory and message queues | Containers cannot snoop on each other's shared memory. |
| **USER** | User and group ID mappings | `root` (UID 0) inside can be mapped to an unprivileged user outside. This is how "rootless" and user-namespace remapping work. |
| **CGROUP** | The view of the cgroup hierarchy | The container sees only its own resource-accounting slice. |
| **TIME** | System clock offsets | Newer and rarely needed; lets a container have its own clock offset. |

**See it for yourself (Linux, or inside WSL):**

```bash
# Start a throwaway Alpine Linux container and list its processes
docker run --rm alpine ps
```

Example output:

```
PID   USER     TIME  COMMAND
    1 root      0:00 ps
```

Explanation: the container sees exactly one process, and it is `ps` itself, running as **PID 1**. On the host, there are hundreds of processes, and in the host's view that same `ps` has some large PID like 48213. The PID namespace gives the process a private numbering.

Now compare hostnames:

```bash
docker run --rm alpine hostname
```

Output (the value is a random ID, yours will differ):

```
9f2c1a7d3b4e
```

By default the container's hostname is the first 12 characters of its container ID, thanks to the UTS namespace.

**Build your own "container" by hand (Linux or WSL only, requires `sudo`):**

```bash
sudo unshare --pid --fork --mount-proc --uts bash
```

What this does: `unshare` is a Linux tool that runs a program in *new* namespaces. `--pid` creates a new PID namespace, `--fork` is required so the new bash can actually become PID 1 in it, `--mount-proc` remounts `/proc` so `ps` shows the new namespace's processes, `--uts` gives a private hostname. Inside that bash:

```bash
ps aux          # you will see only 2 or 3 processes
hostname demo   # changes the hostname ONLY inside this namespace
exit            # leave; the host hostname is unchanged
```

Congratulations: you just made a tiny container with two kernel calls and no Docker. Docker automates and combines *all* of these namespaces plus cgroups plus filesystems for you.

### 5.2 cgroups: "how much can this process use?"

A **control group (cgroup)** is a kernel feature that groups processes and enforces **limits** and **accounting** on resources: CPU time, memory, disk I/O, number of processes, and more.

Without cgroups, one runaway container could eat all the RAM on the host and crash everything. With cgroups, you can say:

```bash
docker run --memory=512m --cpus=1.5 myapp
```

and the kernel itself enforces "this group of processes may use at most 512 MiB of RAM and 1.5 CPU cores' worth of time." If the container tries to use more memory than its limit, the kernel's **OOM killer** (Out-Of-Memory killer) terminates it. You will meet this as **exit code 137** in [Chapter 3](03-core-cli-commands.md).

> **cgroup v1 vs v2:** There are two generations of cgroups. Modern distributions use **cgroup v2** (a single, cleaner hierarchy). Docker Engine 29's release notes say cgroup v1 is deprecated, with support continuing until at least May 2029. A live `docker info` on the test machine for this guide shows `Cgroup Version: 2`. You rarely need to care, except that some very old tutorials mention paths like `/sys/fs/cgroup/memory/...` that only exist on v1.

### 5.3 Union / overlay filesystems: "what files does this process have?"

A container needs its own filesystem: its own `/bin`, `/etc`, `/usr`, your app's files. But copying a whole Linux filesystem (hundreds of MB) for every container would be slow and wasteful.

The solution is a **union filesystem** (the Linux implementation Docker uses is called **overlayfs**). It stacks several directories ("layers") and presents them as a single merged directory:

```
              What the container sees at "/"
                  +---------------------+
                  |   MERGED VIEW       |   <- one normal-looking filesystem
                  +----------^----------+
                             |  overlayfs merges these:
     +-----------------------+-----------------------+
     |  Container layer  (READ-WRITE, thin, per-container) |  <- upperdir: your changes
     +-----------------------------------------------------+
     |  Image layer 3: COPY app/ /app          (read-only) |  \
     +-----------------------------------------------------+   \  lowerdir:
     |  Image layer 2: RUN pip install ...     (read-only) |   /  shared by all
     +-----------------------------------------------------+  /   containers made
     |  Image layer 1: base OS (python:3.13-slim) (read-only)|/    from this image
     +-----------------------------------------------------+
```

Key rules:

- **Image layers are read-only** and **shared.** Ten containers from the same image share the same layers on disk. Only one copy exists.
- Each container gets one extra thin **read-write layer** on top (called the "container layer" or *upperdir*).
- **Copy-on-write (CoW):** if a container modifies a file that lives in a lower, read-only layer, the file is first *copied up* into the container's writable layer, and the change is made to the copy. The original is untouched. If the container deletes a file, a "whiteout" marker hides it in the merged view.
- **When the container is deleted, the writable layer is deleted too.** All changes vanish. This is why *containers are ephemeral* (temporary) and why databases need **volumes** (persistent storage that lives outside the container layer; covered in [Chapter 5](05-storage-and-networking.md)).

> **Storage driver vs image store.** On a Linux host, Docker's default storage driver is `overlay2` (classic) or, with the containerd image store that Engine 29 uses by default on fresh installs, the `overlayfs` snapshotter. A live `docker info` on the test machine reports `Storage Driver: overlayfs`. You do not need to configure this; just know both are overlay-based.

### 5.4 Other walls: capabilities, seccomp, AppArmor/SELinux

Namespaces, cgroups, and overlayfs are the headline trio, but Docker adds more safety belts:

- **Linux capabilities:** the all-powerful `root` privilege is split into ~40 small privileges (e.g., "bind to low ports", "change file ownership"). Docker drops most of them by default, so even `root` inside a container is much weaker than real root.
- **seccomp:** a filter that blocks dangerous *system calls* (requests a process makes to the kernel). Docker ships a default profile blocking around 40 to 50 of the ~300+ syscalls.
- **AppArmor / SELinux:** mandatory access control systems that further restrict what a container can touch (on Ubuntu/Debian it is AppArmor; on Fedora/RHEL it is SELinux).

We cover these properly in [Chapter 10 (Security)](10-security.md).

### 5.5 Putting it together

When you run `docker run -d --memory=512m nginx`, Docker (with its helpers) tells the kernel to:

1. Create new PID, NET, MNT, UTS, IPC (and cgroup) namespaces.
2. Create a cgroup with a 512 MiB memory cap.
3. Assemble an overlay filesystem from the nginx image's layers plus a new writable layer, and make it the new root `/`.
4. Apply a capability set, a seccomp filter, and an AppArmor/SELinux profile.
5. Start the `nginx` process inside all of that.

The result: a regular process, with walls.

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

Almost everything in Docker is about these four things. If you get these straight, the rest of the guide is easy.

### 6.1 Dockerfile: the recipe

A **Dockerfile** is a plain text file containing step-by-step instructions to *build* an image. Think of it as a recipe.

```dockerfile
# Start from an existing image that has Python installed
FROM python:3.13-slim

# Set the working directory inside the image
WORKDIR /app

# Copy our dependency list and install it
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy our application code
COPY . .

# Say what to run when a container starts
CMD ["python", "app.py"]
```

We dissect every instruction in [Chapter 4](04-dockerfile-deep-dive.md). For now, notice it reads like English instructions.

### 6.2 Image: the frozen, read-only package

An **image** is the result of building a Dockerfile. It is a **read-only template** containing a filesystem (your app, libraries, a minimal OS userland) plus some metadata (default command, environment variables, exposed ports). An image does nothing by itself; it is like a program file sitting on disk that is not running.

### 6.3 Container: a running instance of an image

A **container** is a live, running (or stopped) instance created from an image: the image plus a writable layer plus a running process. You can create *many* containers from *one* image.

### 6.4 Registry: the warehouse

A **registry** is a server that stores and distributes images. **Docker Hub** is the best-known public one. You `docker push` an image up to a registry and `docker pull` it down elsewhere. Details in [Section 9](#9-registries-docker-hub-and-others).

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

The classic analogies:

| Analogy | Dockerfile | Image | Container |
|---|---|---|---|
| Cooking | The recipe | The frozen ready-meal in the box | The meal on your plate, being eaten |
| Programming | Source code | Compiled program file (e.g., `app.exe`) | A running process |
| Object-oriented | The class definition | The class | An *instance* (object) of the class |
| Architecture | Blueprint | The finished house design as built | A family actually living in one house (you can build many identical houses) |

### 6.6 The anatomy of an image name

You will type image names constantly, so learn to read them:

```
   docker.io / library / nginx : 1.27 @ sha256:3f1c...
   \_______/   \______/  \___/  \___/   \_____________/
   registry   namespace  repo   tag      digest
   (host)     (owner)    (name) (label)  (exact content hash)
```

| Part | Meaning | Default if omitted |
|---|---|---|
| **Registry** | Where the image lives | `docker.io` (Docker Hub) |
| **Namespace** | The user/organization that owns it | `library` (the special namespace for Docker Official Images) |
| **Repository** | The image's name | (required) |
| **Tag** | A human-friendly label, usually a version | `latest` |
| **Digest** | A cryptographic hash (`sha256:...`) of the image's exact contents | none |

So `nginx` means `docker.io/library/nginx:latest`. And `ghcr.io/acme/chatbot:2.1` means registry `ghcr.io` (GitHub), owner `acme`, repo `chatbot`, tag `2.1`.

> **Tags are mutable; digests are not.** A **tag** is just a sticky note that points at an image. The publisher can move it. `nginx:latest` today and `nginx:latest` next month can be completely different images. A **digest** identifies exact bytes and can never change. For reproducible, secure production deployments, pin by digest or by a specific version tag. Also: `latest` is **not** magic. It does not mean "the newest"; it is simply the tag used when you do not specify one.

---

## 7. Image layers and caching

### 7.1 Every instruction makes a layer

When Docker builds an image, each filesystem-changing instruction in the Dockerfile produces a new **layer**: a record of the files added, changed, or deleted by that step.

```
Dockerfile                                  Resulting image layers (bottom to top)
------------------------------------        ----------------------------------------
FROM python:3.13-slim                  -->  [ base layers from python:3.13-slim ]  (several)
WORKDIR /app                           -->  [ layer: creates /app ]       (tiny)
COPY requirements.txt .                -->  [ layer: requirements.txt ]   (tiny)
RUN pip install -r requirements.txt    -->  [ layer: installed packages ] (big!)
COPY . .                               -->  [ layer: your source code ]   (small)
CMD ["python","app.py"]                -->  (metadata only, no files)
```

Layers are **content-addressable**: each one is identified by a hash of its contents. Identical layers are stored only once on your disk and downloaded only once, even if used by many images. This is why pulling a second image based on `python:3.13-slim` is much faster than the first: the base layers are already there.

You can list the layers of a real image (we cover `docker history` in [Chapter 3](03-core-cli-commands.md)):

```bash
docker history nginx
```

### 7.2 The build cache

When you rebuild, Docker checks each step: "Have I run *this exact instruction* on *this exact input* before?" If yes, it **reuses the cached layer** and skips the work. If the instruction or its inputs changed, that step *and every step after it* is rebuilt.

```
 Rebuild after changing only app.py:

 FROM python:3.13-slim              CACHED   (nothing changed)
 WORKDIR /app                       CACHED
 COPY requirements.txt .            CACHED   (file unchanged)
 RUN pip install -r requirements.txt CACHED  <-- the slow step is skipped!
 COPY . .                           REBUILT  (app.py changed)
 CMD ...                            REBUILT
```

This is why **instruction order matters enormously**: put the things that change rarely (installing dependencies) *before* the things that change often (your source code). Get this wrong and every one-line code change triggers a five-minute dependency reinstall. We master this in [Chapter 4](04-dockerfile-deep-dive.md).

### 7.3 Why layers are a big deal

| Benefit | Explanation |
|---|---|
| **Fast pulls/pushes** | Only layers you do not already have are transferred. |
| **Disk savings** | Shared layers are stored once. |
| **Fast builds** | Unchanged steps come from cache. |
| **Fast container start** | Starting a container does not copy the image; it just adds a thin writable layer on top. |

One caveat: layers are **additive**. If you `COPY` a 1 GB file in one step and `RUN rm` it in the next, the image is *still* 1 GB bigger, because the first layer still contains the file; the delete only hides it in the merged view. Secrets have the same problem: a password copied into one layer and deleted in the next is **still retrievable** from the image. Chapter 4 and [Chapter 10](10-security.md) show the right techniques (multi-stage builds, build secrets).

---

## 8. Docker architecture: what runs when you type `docker run`

When you type `docker run nginx` you are *not* directly creating the container. You are talking to a chain of programs, each with one job. Knowing the chain helps you debug and helps you understand terms like "containerd", "runc", and "OCI".

### 8.1 The big picture

```
 +---------------+        REST API        +-------------------------------------------+
 | docker  (CLI) | ---------------------> |  dockerd  (Docker daemon / "Docker Engine")|
 | the client    |  over a Unix socket    |   - API server                            |
 | you type here |  /var/run/docker.sock  |   - images, networks, volumes, builds     |
 +---------------+  (Windows: named pipe) +----------------------+--------------------+
                                                                   | gRPC
                                                                   v
                                           +-------------------------------------------+
                                           |  containerd                                |
                                           |   - pulls/stores images, manages snapshots |
                                           |   - manages container lifecycle            |
                                           +----------------------+--------------------+
                                                                   | starts one per container
                                                                   v
                                           +-------------------------------------------+
                                           |  containerd-shim                           |
                                           |   - stays alive as the container's parent  |
                                           |   - lets dockerd/containerd restart safely |
                                           +----------------------+--------------------+
                                                                   | calls
                                                                   v
                                           +-------------------------------------------+
                                           |  runc  (OCI runtime)                       |
                                           |   - makes the namespaces/cgroups/rootfs    |
                                           |   - execs your process, then EXITS         |
                                           +----------------------+--------------------+
                                                                   v
                                                         [ your container process ]
```

### 8.2 The components, one at a time

| Component | What it is | Job | Analogy (restaurant) |
|---|---|---|---|
| **Docker client (`docker`)** | The command-line tool you type into | Turns your command into an API request and sends it; prints the result. It does almost no work itself. | The customer giving an order |
| **Docker daemon (`dockerd`)** | A background service (a *daemon* is a program that runs continuously in the background). Also called "Docker Engine" when bundled with the API. | Exposes the REST API; manages images, networks, volumes, and builds; delegates container running to containerd. | The waiter and manager |
| **containerd** | A high-level container runtime, a graduated CNCF project (also used directly by Kubernetes) | Pulls images, stores them, manages snapshots and the container lifecycle. | The kitchen manager |
| **containerd-shim** | A tiny helper process, one per container | Stays as the container's parent so containers survive if `dockerd` or `containerd` restarts or upgrades; holds stdio open. | A dedicated station chef's assistant |
| **runc** | A low-level OCI runtime (a small CLI tool written in Go) | Actually asks the kernel for namespaces, cgroups, and the root filesystem, then starts your process and exits. | The chef who cooks one dish |
| **docker-init** | A tiny init program (it is `tini`) | Optional PID 1 inside the container (`--init`) that forwards signals and cleans up zombie processes. | A kitchen porter |

You can see these versions on a live system. This is real output from the machine this guide was written on (`docker version`, trimmed):

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

Notice something instructive: the **Client** is `windows/amd64` but the **Server** (daemon) is `linux/amd64`. The client runs on Windows; the engine runs in the hidden Linux VM we discussed. Client and daemon are separate programs and can even live on different machines (this is how `docker context` and remote Docker hosts work, covered in Chapters 3 and 11).

### 8.3 Life of `docker run hello-world`

```
 1. You type:   docker run hello-world
 2. CLI  -> POST /containers/create to dockerd (the API call)
 3. dockerd: "Do I have the image locally?" No ->
 4.    dockerd (via containerd) pulls the image from the registry (Docker Hub)
 5.    image layers are unpacked into containerd's snapshot store
 6. dockerd -> containerd: "create and start a container from this image"
 7. containerd launches a containerd-shim
 8. shim runs runc with an OCI "bundle" (rootfs + config.json)
 9. runc sets up namespaces, cgroups, filesystem, security filters
10. runc starts the program in the image (the hello executable) and exits
11. shim keeps watching the process, streams its stdout back
12. CLI prints the text to your terminal; process ends; container becomes "Exited"
```

### 8.4 The API and the socket (important for security)

The CLI and daemon speak a **REST API** over a local **socket**. On Linux this is a Unix domain socket at `/var/run/docker.sock`; on Windows Docker Desktop it is a named pipe (the earlier `docker context ls` output shows endpoints like `npipe:////./pipe/dockerDesktopLinuxEngine`). Because anything that can talk to this socket can ask the daemon to start containers (including privileged ones that mount the host's disk), **access to the Docker socket is effectively root access to the machine.** Never expose it over the network without TLS, and be wary of mounting `/var/run/docker.sock` into containers. [Chapter 10](10-security.md) covers this.

### 8.5 OCI: the open standards

The **Open Container Initiative (OCI)** is a Linux Foundation project that defines three open specifications so that tools from different vendors interoperate:

| OCI spec | Defines | Why you care |
|---|---|---|
| **Image spec** | The format of a container image (layers, manifest, config) | An image built with Docker runs under Podman, containerd, Kubernetes, etc. and vice versa. |
| **Runtime spec** | How to run a container from an unpacked filesystem bundle (`config.json`) | `runc` implements it; alternatives (`crun`, `youki`, `gVisor`'s `runsc`, Kata) implement the same spec. |
| **Distribution spec** | The HTTP API registries use to push and pull images | Any registry (Docker Hub, GHCR, ECR, Harbor) works with any compliant client. |

This is why "Docker image" and "OCI image" are, in practice, the same thing. You are not locked into Docker.

### 8.6 Where Kubernetes fits (preview)

**Kubernetes** (K8s) is an **orchestrator**: it decides *which machines* run *which containers*, restarts them when they fail, scales them, and wires up networking. Kubernetes uses containerd (or CRI-O) directly to run containers; it does not need `dockerd`. Docker builds the images; Kubernetes runs them at scale. See [Chapter 11](11-production-and-enterprise.md).

---

## 9. Registries: Docker Hub and others

A **registry** stores images in **repositories** (a repository is a collection of tagged versions of one image, like `nginx` with tags `1.27`, `latest`, `alpine`). Think of it as GitHub, but for container images instead of source code.

### 9.1 Docker Hub

**Docker Hub** (`hub.docker.com`) is the default public registry. When you type `docker pull nginx`, you are pulling from Docker Hub. It hosts several kinds of images, and learning to tell them apart is a key safety skill:

| Kind | What it is | Trust level |
|---|---|---|
| **Docker Official Images** | A curated set (e.g., `nginx`, `postgres`, `python`, `redis`, `ubuntu`) maintained with Docker's review. They live in the `library/` namespace, so you write them with no username. | High. Still scan them. |
| **Verified Publisher** | Images from commercial vendors that Docker has verified (a badge on Hub). | Good |
| **Docker-Sponsored Open Source** | Open-source projects whose hosting Docker sponsors. | Good |
| **Community images** | `someuser/something`. Anyone can publish these. | **Unknown.** Inspect before using; never run an unknown image with privileges or secrets. |

**Pull rate limits.** Docker Hub limits how many image pulls you can make per six hours. Per Docker's current documentation (verified while writing this chapter): unauthenticated users get **100 pulls per 6 hours**, counted per IPv4 address (or IPv6 /64 subnet); a free authenticated **Personal** account gets **200 pulls per 6 hours**; **Pro, Team, and Business** accounts get unlimited pulls. This matters on shared corporate networks and CI, where many machines share one IP address. The fix is `docker login`, a registry mirror or pull-through cache, or hosting your own base images. If you hit it you will see an error like `toomanyrequests: You have reached your pull rate limit`.

### 9.2 Other registries

| Registry | Run by | Typical use |
|---|---|---|
| **GitHub Container Registry (GHCR)**, `ghcr.io` | GitHub | Images next to your code, integrated with GitHub Actions |
| **Amazon ECR** | AWS | Images for ECS/EKS/Lambda on AWS |
| **Google Artifact Registry** | Google Cloud | Images for GKE/Cloud Run (it replaced the older Container Registry) |
| **Azure Container Registry (ACR)** | Microsoft | Images for AKS and Azure services |
| **Quay.io** | Red Hat | Public and private repos, built-in scanning |
| **GitLab Container Registry** | GitLab | Built into GitLab projects |
| **Harbor** | CNCF open source | Self-hosted enterprise registry with scanning, signing, replication |
| **`registry:2` (CNCF Distribution)** | Open source | The tiny reference registry you can run yourself in one command |

All of them speak the OCI distribution protocol, so the commands are identical. Only the image name's host part changes: `docker pull ghcr.io/owner/image:tag`.

### 9.3 Public vs private

- **Public repos:** anyone can pull; no login needed.
- **Private repos:** require `docker login` first. Companies keep proprietary application images private.
- **Air-gapped / offline:** you can export images to a tar file (`docker save`) and carry them across, then `docker load` them (Chapter 3).

---

## 10. Docker Engine vs Docker Desktop vs the alternatives

"Docker" is an overloaded word. People use it to mean at least five different things. Untangle them now and you will avoid hours of confusion.

| Name | What it actually is |
|---|---|
| **Docker (the company)** | Docker, Inc., which sponsors most of the ecosystem and sells subscriptions and tools. |
| **Moby** | The open-source upstream project from which Docker Engine is assembled. |
| **Docker Engine** | The open-source container runtime stack: `dockerd` + the REST API + containerd + runc (and the `docker` CLI). Runs **natively on Linux**. Free. This is what runs on production servers. |
| **Docker CLI** | The `docker` command. Plus plugins such as `docker compose` and `docker buildx`. |
| **Docker Desktop** | A **desktop application** for Windows, macOS, and Linux that bundles Docker Engine (inside a lightweight VM), the CLI, Compose, Buildx, a GUI dashboard, an optional single-node Kubernetes, Docker Scout (vulnerability analysis), extensions, Docker Model Runner (local AI models), and more. It is a *product* with its own license terms ([Chapter 2](02-installation-and-operating-systems.md) summarizes them). |
| **Docker Hub** | The public registry (a website and service). |

```
   +---------------------- Docker Desktop (Windows / macOS / Linux) ---------------------+
   |  GUI dashboard | Compose | Buildx | Scout | Kubernetes (optional) | Model Runner     |
   |  +------------------- hidden Linux VM -------------------------------------------+  |
   |  |        +------------- Docker Engine (dockerd + containerd + runc) -----+      |  |
   |  |        |        your containers run here                               |      |  |
   |  +--------+-------------------------------------------------------------+------+  |
   +--------------------------------------------------------------------------------------+

   On a Linux server you install only the middle box: Docker Engine (+ CLI + plugins).
```

### 10.1 The alternatives

Because the standards are open (OCI), several tools can build and run the same images:

| Tool | What it is | How it differs from Docker | Good for |
|---|---|---|---|
| **Podman** | A daemonless container engine from Red Hat (CLI is almost drop-in compatible: `alias docker=podman`) | No central root daemon; each container is a child of the user's process; rootless by default; has "pods" like Kubernetes | Security-conscious Linux/RHEL environments; people avoiding Docker Desktop licensing |
| **Podman Desktop** | GUI for Podman | Open source GUI | Windows/macOS users who prefer Podman |
| **containerd + nerdctl** | containerd with a Docker-compatible CLI | Minimal, no dockerd | Kubernetes nodes, advanced users |
| **CRI-O** | A container runtime built only for Kubernetes | Not for general developer use | OpenShift/Kubernetes nodes |
| **Rancher Desktop** | Free desktop app with containerd or dockerd | Open source, includes K8s | Free alternative to Docker Desktop |
| **Colima / Lima** | Command-line Linux VMs on macOS (and Linux) running Docker or containerd | Lightweight, CLI-only | Mac users avoiding Docker Desktop |
| **OrbStack** | Commercial, fast macOS app | Very fast file sharing on Mac | Mac power users |
| **Buildah** | Builds OCI images without a daemon | Build only | CI pipelines |
| **Kaniko / BuildKit** | Build images in places where you cannot run Docker (e.g., inside Kubernetes) | Build only | CI/CD |
| **LXC / LXD (Incus)** | "System containers" that feel like lightweight VMs | Full OS-like containers, not app containers | Hosting many small "servers" |

**Which should you learn?** Docker. It is the de facto standard, the most documented, and every other tool copies its commands. Once you know Docker, switching to Podman takes about an hour. This guide uses Docker throughout, and nearly everything transfers.

---

## 11. What is Docker Compose? (a first look)

Running one container is easy. Real applications have several parts. A typical AI app might need:

- a **web/API** container (FastAPI),
- a **database** container (Postgres),
- a **vector database** container (Qdrant or pgvector),
- a **cache** (Redis),
- maybe a **worker** container, and an **LLM server** container.

Starting each one by hand needs a long `docker run` command with ports, volumes, networks, environment variables, and a correct startup order. You would have to remember all of it and share it with teammates.

**Docker Compose** solves this. You describe *the whole multi-container application* in one YAML file (`compose.yaml`), and then a single command starts, stops, and wires up everything.

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

What this does: `docker compose up -d` reads `compose.yaml`, creates a private network so the services can find each other by name (`db` is reachable from `api` at the hostname `db`), creates the volume, builds/pulls images, and starts the containers. `-d` means "detached": run in the background.

### 11.1 `docker compose` vs `docker-compose` (a very common source of confusion)

| | **Compose v1 (legacy)** | **Compose v2 and later (current)** |
|---|---|---|
| Command | `docker-compose` (with a hyphen) | `docker compose` (with a space) |
| Implementation | A separate program written in Python | A **plugin** built into the Docker CLI, written in Go |
| Status | **End of life.** Docker stopped updating it in 2023 and it is no longer shipped with Docker Desktop | **Actively developed** |
| Install | Separate `pip`/binary install | Comes with Docker Desktop, or the `docker-compose-plugin` package on Linux |
| Version on the test machine for this guide | n/a | `Docker Compose version v5.6.1` |

> **Version note.** Compose's version numbers jumped from the 2.x line to **5.x** (the test machine has v5.6.1 and Docker's Compose release page lists v5.6.1, released October 2026). I could not verify from Docker's pages *why* the numbering jumped, so I will not guess. What matters: `docker compose` (space) is the current command, and it is the same tool as "Compose v2" in older tutorials. If you find a tutorial using `docker-compose` (hyphen), the commands translate directly by replacing the hyphen with a space.

> **Also: the `version:` field.** Old tutorials start their YAML with `version: "3.8"`. Modern Compose follows the "Compose Specification" and ignores or warns about that top-level `version` field. Omit it.

A complete deep dive (networks, volumes, health checks, profiles, `depends_on` conditions, watch mode, GPUs) is in [Chapter 6](06-docker-compose.md).

---

## 12. What Docker is NOT

Misunderstanding these causes real-world incidents. Memorize them.

### 12.1 Docker is not a virtual machine

There is no guest operating system and no separate kernel. Consequences:

- A container can only run software built for the **host kernel's type** (Linux containers need a Linux kernel).
- If something in a container exploits a **kernel bug**, it may affect the host and other containers.
- Containers are meant to be **ephemeral**: stateless and disposable, not pet servers you SSH into and hand-configure.
- Typically **one main process per container**. You do not run a whole "mini server" with init, SSH, cron, and a database all in one.

### 12.2 Docker is not a security boundary by default

Containers are *isolated*, but isolation is not the same as a hardened security boundary. By default:

- Containers share the host kernel.
- The default user inside a container is often **root** (UID 0) unless the image says otherwise.
- Anyone who can run `docker` commands (on Linux, anyone in the `docker` group) can trivially become root on the host, for example with `docker run -v /:/host --privileged ...`.
- Published ports can bypass host firewalls such as `ufw` (Docker edits `iptables` rules directly; Docker's own docs warn about this).
- A container image can contain malware or vulnerable libraries.

Docker can be made quite secure (non-root users, dropped capabilities, read-only filesystems, rootless mode, user namespaces, seccomp/AppArmor profiles, image scanning, signed images), but **you** must do that. Chapter 10 is a full chapter on it. For truly hostile multi-tenant code (running other people's untrusted code), use stronger isolation such as gVisor, Kata Containers, or Firecracker microVMs.

### 12.3 Docker is not an orchestrator

Plain Docker runs containers on **one machine**. It does not (by itself):

- Spread containers across many servers,
- Automatically replace a failed server's workloads on another server,
- Do rolling updates with zero downtime across a cluster,
- Auto-scale based on load.

That is the job of an **orchestrator**: **Kubernetes** (the industry standard), Docker Swarm (Docker's built-in clustering mode, still present but far less used), HashiCorp Nomad, or managed services (AWS ECS, Google Cloud Run, Azure Container Apps). Docker Compose is *not* an orchestrator either. It manages a multi-container app on a single host. Compose is excellent for development, testing, and small single-server deployments, and Chapter 11 discusses when to graduate.

### 12.4 Docker is not a backup or data store

Anything written to a container's writable layer disappears when the container is removed. Persistent data must live in **volumes** or bind mounts, which need their own backups ([Chapter 5](05-storage-and-networking.md)).

### 12.5 Docker is not a replacement for good dependency management

Docker freezes a *working* environment. It does not fix an unpinned `requirements.txt`. If you build with `pip install numpy` (no version) today and next month, you may get different images. Pin your dependencies and base images.

### 12.6 Docker is not always faster

A container has near-native CPU speed on Linux. But on Windows and macOS there is a VM layer, and **file access across the host/VM boundary** (bind mounts of your project folder) can be significantly slower. [Chapter 2](02-installation-and-operating-systems.md) shows how to avoid this on Windows. GPU workloads need extra setup ([Chapter 8](08-docker-for-ai-and-ml.md)).

---

## 13. Why this matters for AI developers

You are learning Docker to be an effective AI developer, so here is how it maps to your work (the rest of the guide goes deep):

| AI/ML problem | How Docker helps |
|---|---|
| "My PyTorch needs CUDA 12.x and a specific cuDNN" | Use an NVIDIA CUDA base image. The CUDA user-space libraries come inside the image; only the GPU **driver** lives on the host. |
| "Library version hell between projects" | One image per project, with pinned versions. |
| "How do I ship my model API?" | Package FastAPI plus model weights (or a downloader) into an image and deploy it to any cloud. |
| "How do I run Postgres + pgvector + Qdrant + Redis + my API for local development?" | Docker Compose: one command. |
| "I want to run an open LLM locally" | **Docker Model Runner** (`docker model run ...`) in Docker Desktop, or run containers like Ollama or vLLM. |
| "My training run must be reproducible" | A frozen image plus pinned data versions. |
| "Deploy on Kubernetes" | Every Kubernetes workload starts as a container image. |

> **About Docker Model Runner.** Docker Model Runner (DMR) lets you pull and run AI models (such as `ai/smollm2`) with commands like `docker model run`. Per Docker's current documentation (verified while writing): enable it in Docker Desktop under **Settings > AI**, or on Docker Engine on Linux by installing the `docker-model-plugin` package. Models are pulled from Docker Hub (or Hugging Face) as OCI artifacts. Engine-side TCP access is enabled by default on port **12434**, and Docker Desktop lets you enable host-side TCP and choose the port. The docs name llama.cpp and vLLM as inference backends. I did not verify every sub-command or GPU nuance; [Chapter 8](08-docker-for-ai-and-ml.md) covers it in depth and flags what could not be verified.

---

## 14. Common mistakes and misconceptions

| # | Mistake or misconception | Reality |
|---|---|---|
| 1 | "A container is a small VM." | It is a **process** with namespaces/cgroups/overlay walls, sharing the host kernel. |
| 2 | "Images and containers are the same thing." | **Image = template (read-only). Container = running instance (image + writable layer + process).** |
| 3 | "If I delete the container my data is safe." | The opposite: deleting a container deletes its writable layer. Use volumes for data. |
| 4 | "`latest` means the newest version." | It is just a default tag name. It can be old or changed anytime. Pin versions. |
| 5 | "Docker is secure because it is isolated." | Isolation is not hardening. Default root user, shared kernel, and `docker` group = root equivalent. |
| 6 | "I'll put everything (app, DB, cache) in one container." | One main process per container; use Compose to combine several. |
| 7 | "`docker compose` and `docker-compose` are two different tools." | Same product family. The hyphenated one is the old, retired v1. |
| 8 | "Docker Desktop and Docker Engine are the same." | Desktop is a product wrapping Engine in a VM (with license terms); Engine is the free, native-Linux runtime. |
| 9 | "Windows containers and Linux containers are interchangeable." | A Linux image cannot run on the Windows container engine (and vice versa). 99% of what you will do is Linux containers. |
| 10 | "Deleting a file in a later Dockerfile step shrinks the image / removes the secret." | Earlier layers still contain it. Use multi-stage builds. |
| 11 | "Docker made containers possible." | Linux kernel features did; Docker made them usable and standardized the workflow. |
| 12 | "Kubernetes replaced Docker." | Kubernetes stopped using `dockershim` (the Docker adapter). Docker-built OCI images still run on Kubernetes. Docker remains the dominant developer tool. |

---

## 15. Try it yourself

> These steps need Docker installed. If you have not yet installed it, skip to [Chapter 2](02-installation-and-operating-systems.md) and come back. On Windows and macOS make sure Docker Desktop is running (whale icon in the tray or menu bar shows "Engine running").

### Exercise 1: say hello

```bash
docker run hello-world
```

Expected: a "Hello from Docker!" message. (We annotate every line of this output in [Chapter 2](02-installation-and-operating-systems.md#7-verifying-your-installation-with-hello-world).)

### Exercise 2: prove the PID namespace

```bash
docker run --rm alpine ps
```

Expected: only one or two lines, with `ps` as PID 1.

### Exercise 3: prove containers share the host kernel

```bash
# What kernel is the HOST (or Docker's Linux VM) running?
docker run --rm alpine uname -r

# Now run a completely different Linux distribution:
docker run --rm ubuntu uname -r
docker run --rm debian uname -r
```

Expected: **all three print the same kernel version.** Different distributions (userland), one shared kernel. On Windows with Docker Desktop the value ends in `microsoft-standard-WSL2` (for example, `6.18.54.1-microsoft-standard-WSL2` on the test machine). That is Exercise 3's proof that containers are not VMs.

### Exercise 4: prove layers are shared

```bash
docker pull python:3.13-slim
docker pull python:3.13-slim   # second time: "Image is up to date"
docker image ls
```

Notice the second pull transfers nothing. Optional: look at the layers with `docker history python:3.13-slim`.

### Exercise 5: prove containers are ephemeral

```bash
docker run --name scratch alpine sh -c "echo hello > /note.txt && cat /note.txt"
docker rm scratch                           # delete the container
docker run --rm alpine cat /note.txt        # a NEW container from the same image
```

Expected: the first run prints `hello`; the last command fails with `cat: can't open '/note.txt': No such file or directory`. The file lived only in the first container's writable layer.

### Exercise 6 (Linux/WSL only): build a mini-container with `unshare`

Use the `unshare` command from [Section 5.1](#51-namespaces-what-can-this-process-see) and confirm `ps aux` shows only a couple of processes.

### Clean up

```bash
docker image rm alpine ubuntu debian hello-world python:3.13-slim   # optional: remove the images you pulled
```

(`docker rm` removes containers; `docker image rm` removes images. Chapter 3 covers all of it.)

---

## 16. Glossary of terms introduced in this chapter

| Term | Plain-English meaning |
|---|---|
| **Container** | A running (or stopped) instance of an image: a normal process isolated by namespaces and limited by cgroups. |
| **Image** | A read-only template (layered filesystem + metadata) from which containers are created. |
| **Dockerfile** | A text file of instructions that builds an image. |
| **Registry** | A server that stores and distributes images (Docker Hub, GHCR, ECR...). |
| **Repository** | A named collection of tagged versions of one image inside a registry. |
| **Tag** | A movable, human-friendly label on an image version (`1.27`, `latest`). |
| **Digest** | An immutable `sha256:` hash identifying exact image contents. |
| **Layer** | One read-only slice of an image created by one Dockerfile instruction; shared and cached. |
| **Build cache** | Saved layers Docker reuses when an instruction and its inputs have not changed. |
| **Kernel** | The core of an OS that talks to hardware and enforces who can do what. |
| **Hypervisor** | Software that creates and runs virtual machines. |
| **Virtual machine (VM)** | A simulated computer with its own full OS and kernel. |
| **Namespace** | A Linux kernel feature that gives a process its own private view of a resource (PIDs, network, mounts, hostname...). |
| **cgroup (control group)** | A Linux kernel feature that limits and measures a group of processes' CPU, memory, I/O. |
| **OOM killer** | The kernel mechanism that kills processes when memory runs out (exit code 137 in containers). |
| **overlayfs / union filesystem** | Stacks read-only layers plus a writable layer into one merged view. |
| **Copy-on-write (CoW)** | Copying a file to the writable layer only when it is first modified. |
| **Capability** | One small slice of the traditional all-powerful "root" privilege. |
| **seccomp** | A kernel filter that blocks dangerous system calls. |
| **Daemon** | A background program that runs continuously. |
| **Docker client (CLI)** | The `docker` command; sends API requests to the daemon. |
| **dockerd** | The Docker daemon (the "Docker Engine" server process). |
| **containerd** | A high-level container runtime managing images and container lifecycle; used by Docker and Kubernetes. |
| **containerd-shim** | A small per-container helper process that keeps containers alive across daemon restarts. |
| **runc** | The low-level OCI runtime that creates the actual container process. |
| **OCI** | Open Container Initiative: open standards for image format, runtime, and registry protocol. |
| **Docker Engine** | The open-source runtime stack (dockerd + containerd + runc + API + CLI), native on Linux. |
| **Docker Desktop** | A desktop application bundling Engine (in a VM), the GUI, Compose, Buildx, and more; has license terms. |
| **Moby** | The open-source upstream project Docker Engine is built from. |
| **Podman** | A daemonless, rootless-by-default, Docker-compatible container engine. |
| **Compose** | A tool to define and run multi-container apps from a `compose.yaml`. |
| **Orchestrator** | Software (Kubernetes etc.) that runs containers across many machines and keeps them healthy. |
| **Ephemeral** | Temporary; will disappear when the container is removed. |
| **Volume** | Docker-managed persistent storage that outlives containers (Chapter 5). |
| **Docker socket** | The local endpoint (`/var/run/docker.sock`, or a Windows named pipe) through which the CLI talks to dockerd; powerful and security-sensitive. |

---

## 17. Mental model summary

Keep this picture in your head for the rest of the guide:

```
          WRITE                     BUILD                    SHARE                    RUN
   +---------------+         +---------------+        +---------------+       +---------------+
   |  Dockerfile   |  build  |     Image     |  push  |   Registry    |  pull |   Container   |
   |  (recipe)     | ------> | (frozen,      | -----> | (warehouse)   | ----> | (running      |
   |               |         |  layered)     |        |               |  run  |  process with |
   +---------------+         +---------------+        +---------------+       |  walls)       |
                                                                              +---------------+
```

1. **A container is a process, not a machine.** Namespaces limit what it sees, cgroups limit what it uses, overlayfs gives it files.
2. **Images are layered, read-only, shared, and cached.** Containers are images plus a thin writable layer, and are disposable.
3. **The `docker` command is just a client.** `dockerd` does the managing, `containerd` runs the lifecycle, and `runc` makes the container. All follow open OCI standards.
4. **Docker Engine is the free Linux runtime; Docker Desktop is a packaged app that hides a Linux VM so Windows and macOS can run Linux containers.**
5. **Compose describes multi-container apps in one file;** orchestrators (Kubernetes) run containers across many machines.
6. **Docker gives isolation, not magic security,** and containers are for stateless, reproducible workloads: persistent data lives in volumes.

---

## 18. Check your understanding

Try answering these without scrolling up. Answers are at the bottom.

1. In one sentence, what is the difference between an image and a container?
2. Why can Docker start a container in a second but a VM takes a minute?
3. Name the three Linux kernel features at the heart of containers, and say which controls what a process *sees* versus what it *uses*.
4. You run `docker pull nginx`. Write the full, expanded image name Docker uses.
5. Why is `FROM python:latest` a bad idea for a production Dockerfile?
6. You are on a Windows laptop. Where does the Linux kernel your containers use come from?
7. What is the difference between `docker compose` and `docker-compose`?
8. A teammate says, "Docker is secure because everything is in a sandbox." What is wrong with that statement?
9. Which component actually asks the Linux kernel to create namespaces and cgroups: `docker` CLI, `dockerd`, `containerd`, or `runc`?
10. You `COPY` a 2 GB dataset into an image and then `RUN rm` it in the next step. Is the image 2 GB smaller than before the `rm`? Why or why not?

<details>
<summary>Answers</summary>

1. An image is a read-only template; a container is a running (or stopped) instance of that image with a writable layer and a process.
2. A container does not boot an operating system. It just starts a process on the already-running host kernel inside namespaces/cgroups.
3. Namespaces (what it sees), cgroups (what it uses), overlay/union filesystem (which files it has).
4. `docker.io/library/nginx:latest`.
5. `latest` is a moving tag. Your builds will silently change over time, and a major version bump can break your app. Pin an exact version (and ideally a digest).
6. From a hidden lightweight Linux VM (WSL 2 with Docker Desktop's WSL 2 backend), not from Windows itself.
7. `docker compose` (space) is the current plugin-based Compose (v2+); `docker-compose` (hyphen) is the retired Python-based v1.
8. Containers share the host kernel, often run as root by default, and anyone with Docker access is effectively root on the host. Isolation needs deliberate hardening.
9. `runc` (invoked by containerd's shim).
10. No. The first layer still contains the file; the `rm` only hides it in the merged view. The image remains about 2 GB larger. Use multi-stage builds or avoid copying it at all.

</details>

---

**Next:** [Chapter 2: Installation and Operating Systems](02-installation-and-operating-systems.md): get Docker running on Windows 11, macOS, or Linux, and verify it works.
