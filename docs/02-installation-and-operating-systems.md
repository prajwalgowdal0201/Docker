# Chapter 2: Installation and Operating Systems

This chapter covers which operating system to use, how to install Docker on Windows 11, macOS, and Ubuntu, how to configure Docker Engine on servers, Docker Desktop licensing, how to verify an installation, how to uninstall, and how to fix common install errors.

**Prerequisite:** [Chapter 1](01-what-is-docker.md), or knowledge of what an image and a container are.

Facts in this chapter (system requirements, licensing thresholds, package names, supported Ubuntu versions) were checked against Docker's and Microsoft's official documentation in October 2026. Software changes. If a step fails, the official page is the source of truth. See also [Section 9](#9-common-install-errors-and-fixes).

---

## Table of contents

1. [Which operating system should you use?](#1-which-operating-system-should-you-use)
2. [Windows 11: step by step](#2-windows-11-step-by-step)
3. [macOS: Apple Silicon and Intel](#3-macos-apple-silicon-and-intel)
4. [Linux (Ubuntu): installing Docker Engine](#4-linux-ubuntu-installing-docker-engine)
5. [Docker Engine on servers](#5-docker-engine-on-servers)
6. [Docker Desktop licensing summary](#6-docker-desktop-licensing-summary)
7. [Verifying your installation with hello-world](#7-verifying-your-installation-with-hello-world)
8. [Uninstall and cleanup](#8-uninstall-and-cleanup)
9. [Common install errors and fixes](#9-common-install-errors-and-fixes)
10. [Common mistakes](#10-common-mistakes)
11. [Try it yourself](#11-try-it-yourself)
12. [Summary](#12-summary)

---

## 1. Which operating system should you use?

### 1.1 The short, clear answer

| Purpose | Recommended OS | Reason |
|---|---|---|
| Production (servers, cloud, Kubernetes nodes) | Linux | Containers are a Linux technology. On Linux they run directly on the host kernel: fastest, simplest, best supported. |
| Learning / local development | The OS already in use: Windows 11 (with WSL 2), macOS, or Linux | Docker Desktop on Windows and macOS provides a real Linux engine in a lightweight VM. Everything transfers to Linux servers. |
| Best learning setup if there is a choice | Linux (for example Ubuntu), or Windows with WSL 2 | Matches production; no VM layer. |

Rule: develop on any OS, deploy on Linux.

### 1.2 Why containers are "Linux-based"

Containers use Linux kernel features: namespaces, cgroups, and overlay filesystems ([Chapter 1](01-what-is-docker.md#5-how-containers-actually-work)). Windows and macOS kernels do not provide these. Consequences:

- A Linux container needs a Linux kernel. On Windows and macOS, Docker Desktop starts a hidden Linux VM and runs containers inside it.
- Nearly all container images (nginx, postgres, python, node, pytorch) are Linux images.
- Cloud providers run containers on Linux servers, and Kubernetes worker nodes are almost always Linux.

```
 Linux server:                  Windows / macOS laptop:
 +-----------------+            +--------------------------+
 | containers      |            | containers               |
 +-----------------+            +--------------------------+
 | Docker Engine   |            | Docker Engine            |
 +-----------------+            +--------------------------+
 | Linux kernel    |            | Linux kernel (in a VM)   |   <- extra layer
 +-----------------+            +--------------------------+
 | hardware        |            | Windows/macOS + hardware |
 +-----------------+            +--------------------------+
```

### 1.3 What about Windows containers?

**Windows containers** run Windows programs on the Windows kernel. They are real but a niche.

| | Linux containers | Windows containers |
|---|---|---|
| Image content | Linux binaries | Windows binaries (for example .NET Framework apps, IIS) |
| Kernel needed | Linux | Windows (matching version for process isolation) |
| Image size | Tens to hundreds of MB | Often several GB |
| Typical use | Web, AI/ML, data, microservices | Legacy Windows Server apps (old .NET Framework, IIS, COM) |
| Ecosystem | Large | Small |
| Availability | Everywhere | Windows hosts only. Per Docker's docs, Docker Desktop needs Windows 10/11 Pro or Enterprise for Windows containers; Home and Education run Linux containers only. |

Windows containers have two isolation modes:

- **Process isolation:** shares the host Windows kernel; host and container Windows versions must match closely.
- **Hyper-V isolation:** each container gets a thin utility VM with its own kernel; more isolated, slower to start.

Docker Desktop switches between "Linux containers mode" and "Windows containers mode".

This guide uses Linux containers unless stated otherwise. The error `no matching manifest for windows/amd64` means Docker Desktop is in Windows containers mode. Switch back with the whale icon > "Switch to Linux containers...".

### 1.4 Option summary by platform

| Platform | How to get Docker | Where containers run | Notes |
|---|---|---|---|
| Linux | **Docker Engine** (free, native); optionally Docker Desktop for Linux | Directly on the host kernel | Production-equivalent. No VM. |
| Windows 11 | **Docker Desktop** with the **WSL 2** backend (recommended), or Docker Engine inside a WSL 2 Linux distro | WSL 2's lightweight Linux VM | Best choice for most Windows users. |
| macOS | **Docker Desktop**, or Colima, OrbStack, Podman Desktop, Rancher Desktop | Lightweight Linux VM | Apple Silicon is ARM (`arm64`), which affects images ([Section 3](#3-macos-apple-silicon-and-intel)). |
| Windows Server | Docker Engine / Mirantis Container Runtime for Windows containers | Windows kernel | Docker Desktop does not support Windows Server editions, per its docs. |

---

## 2. Windows 11: step by step

### 2.1 The architecture on Windows

```
 Windows 11
 +--------------------------------------------------------------------+
 |  PowerShell / Windows Terminal / VS Code                           |
 |  docker.exe  (the CLI client)                                      |
 |        |  talks over a named pipe                                  |
 |        v                                                           |
 |  Docker Desktop (app, dashboard, settings)                         |
 |  +--------------------------------------------------------------+  |
 |  |  WSL 2  (a lightweight Linux VM managed by Windows)          |  |
 |  |   - docker-desktop   (Docker Desktop's own internal distro)  |  |
 |  |   - Ubuntu           (user's distro, optional, recommended)  |  |
 |  |   Docker Engine (dockerd, containerd, runc) runs here        |  |
 |  +--------------------------------------------------------------+  |
 +--------------------------------------------------------------------+
```

**WSL 2** (Windows Subsystem for Linux, version 2) is a Microsoft feature that runs a real Linux kernel, built and updated by Microsoft, in a lightweight VM integrated with Windows. It provides Linux terminals (`bash`, `apt`) on Windows. WSL 1 translated Linux system calls and had no real kernel, so Docker requires WSL 2.

### 2.2 System requirements

Verified against Docker's docs, October 2026, for the WSL 2 backend on x86-64:

| Requirement | Detail |
|---|---|
| WSL version | 2.1.5 or later (2.6 or later for Docker's Enhanced Container Isolation) |
| Windows version | Windows 11 64-bit 23H2 (build 22631) or higher, or Windows 10 64-bit 22H2 (build 19045) (Enterprise, Pro, or Education, as listed on Docker's page) |
| Windows Server service | `LanmanServer` enabled and set to Automatic (default) |
| Windows features | WSL 2 enabled (`wsl --install` does this) |
| CPU | 64-bit with **SLAT** (Second Level Address Translation; present on nearly all CPUs since about 2010) |
| RAM | 8 GB documented minimum; 16 GB or more recommended for AI work |
| BIOS/UEFI virtualization | Must be enabled (Intel VT-x or AMD-V / "SVM") |

Windows Home: Docker's requirements page lists "Enterprise, Pro, or Education" for Windows versions, but another line on the same page says Home and Education run Linux containers only, which implies Home works for Linux containers. Docker Desktop with WSL 2 has historically worked on Home. Unverified: the two statements could not be reconciled on the current page. If the installer refuses on Home, check [Docker's Windows install page](https://docs.docker.com/desktop/setup/install/windows-install/) or use [Section 2.9](#29-alternative-docker-engine-directly-inside-wsl-no-docker-desktop). Windows containers require Pro or Enterprise.

### 2.3 Step 0: check that virtualization is enabled

1. Press `Ctrl + Shift + Esc` to open Task Manager.
2. Open the Performance tab and select CPU.
3. Check **Virtualization:** at the bottom right. It must say **Enabled**.

If it says Disabled, restart into BIOS/UEFI setup (usually `F2`, `Del`, `F10`, or `Esc` at boot; varies by manufacturer). Enable **Intel Virtualization Technology**, **VT-x**, **SVM Mode**, or **AMD-V**, save, and reboot.

PowerShell check:

```powershell
systeminfo | Select-String "Hyper-V Requirements" -Context 0,4
```

- `systeminfo`: prints machine details.
- `Select-String ... -Context 0,4`: filters for the "Hyper-V Requirements" block and shows 4 lines after it.

If a hypervisor is already running (true once WSL 2 is installed), the output says "A hypervisor has been detected", which also means virtualization is on.

### 2.4 Step 1: install WSL 2

Open PowerShell as Administrator (right-click Start, choose **Terminal (Admin)**).

```powershell
wsl --install
```

This enables the required Windows features (Windows Subsystem for Linux and Virtual Machine Platform), downloads the latest WSL Linux kernel, and installs **Ubuntu** by default. Per Microsoft, it works on Windows 10 version 2004 (build 19041) and higher and on Windows 11.

Restart when asked. An Ubuntu window then opens and finishes installing, and asks for a Linux username and password. These apply only to the Linux environment and need not match the Windows login. The password is required for `sudo`.

Notes from Microsoft's docs:

- `wsl --install` only works if WSL is not already installed. If the WSL help text appears, run `wsl --list --online` to list distros, then `wsl --install -d <DistroName>`.
- If the install hangs at 0.0%, try `wsl --install --web-download -d <DistroName>`.

Verify:

```powershell
wsl --update
wsl --version
wsl --list --verbose
```

| Command | Meaning |
|---|---|
| `wsl --update` | Updates the WSL package, including the Linux kernel. Docker needs 2.1.5 or newer. |
| `wsl --version` | Prints WSL, kernel, WSLg, and Windows versions. |
| `wsl --list --verbose` (short: `wsl -l -v`) | Lists installed distros with state and WSL version. |

Example output of `wsl -l -v`:

```
  NAME              STATE           VERSION
* Ubuntu            Running         2
```

`*` marks the default distro. VERSION must be 2. If it is 1, convert it:

```powershell
wsl --set-version Ubuntu 2
wsl --set-default-version 2
```

The first command converts one distro. The second makes future installs use WSL 2.

### 2.5 Step 2: install Docker Desktop

**Option A: installer.** Download Docker Desktop for Windows from Docker's website (AMD64 build for Intel/AMD PCs, ARM64 build for Snapdragon-based PCs) and run `Docker Desktop Installer.exe`.

**Option B: winget.**

```powershell
winget install -e --id Docker.DockerDesktop
```

- `winget`: the Windows Package Manager.
- `-e` (`--exact`): the ID must match exactly.
- `--id Docker.DockerDesktop`: the package to install.
- `--silent` (optional): no prompts.

Installer choices:

| Prompt | Recommendation |
|---|---|
| Use WSL 2 instead of Hyper-V (Configuration page) | Checked. WSL 2 is the recommended backend. |
| Per-user vs all-users installation | Per Docker's docs, per-user is the default and needs no administrator rights. All-users needs admin and is required for the Hyper-V backend. |
| Add shortcut to desktop | Optional. |

After installing, start Docker Desktop and accept the subscription agreement. Signing in to a Docker account is optional for personal use; signing in raises the Docker Hub pull limit ([Chapter 1](01-what-is-docker.md#91-docker-hub)). Wait for **"Engine running"** in the bottom-left corner before using the CLI.

Command-line install (for IT teams and scripts), from Docker's documentation:

```powershell
# Per-user install, no admin needed
Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList 'install', '--user'

# All users (needs admin)
Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList 'install'
```

Installer flags (from Docker's documented list):

| Flag | Meaning |
|---|---|
| `--quiet` | Suppress output |
| `--accept-license` | Accept the agreement during install |
| `--installation-dir=<path>` | Install location |
| `--backend=<wsl-2\|hyper-v\|windows\|docker-vmm>` | Default backend; `wsl-2` is the default |
| `--always-run-service` | Start the Docker service at boot to avoid later admin prompts |
| `--no-windows-containers` | Skip Windows containers support |

### 2.6 Step 3: connect Docker to your Ubuntu distro (WSL integration)

Docker Desktop runs its engine in its own WSL 2 distro. To use `docker` from an Ubuntu terminal:

1. Open Docker Desktop > Settings.
2. General: confirm **Use the WSL 2 based engine** is ticked (default when WSL 2 was chosen).
3. Resources > WSL integration: enable **Enable integration with my default WSL distro** and the toggle next to **Ubuntu**.
4. Click **Apply & restart**.

`docker version` then works in the Ubuntu terminal. The error `The command 'docker' could not be found in this WSL 2 distro` means this step was skipped.

### 2.7 Step 4: verify

In PowerShell or Windows Terminal:

```powershell
docker version
docker run hello-world
```

The "Hello from Docker!" message confirms the install ([Section 7](#7-verifying-your-installation-with-hello-world)). Repeat in the Ubuntu terminal to confirm integration.

**The `docker-users` group.** For an all-users install where the administrator who installed Docker differs from the user, add the user to the local group `docker-users`:

```powershell
net localgroup docker-users <YourUserName> /add
```

Run as Administrator, then sign out and in. Per Docker's docs this is needed for Windows containers or Hyper-V VM management, not for Linux containers on the WSL 2 backend. Membership gives access to the Docker daemon socket, which is equivalent to administrative access on the host. Add only users who need it.

### 2.8 Backends explained: WSL 2 vs Hyper-V vs Windows containers

| Backend | What it is | When to choose it |
|---|---|---|
| **WSL 2** (default, recommended) | Linux containers in the WSL 2 VM. Shares memory dynamically with Windows, starts fast, integrates with the Linux distro. | Almost all users |
| **Hyper-V** | Linux containers in a dedicated Hyper-V VM managed by Docker Desktop. Needs the Hyper-V and Containers Windows features and an all-users install. Pro/Enterprise/Education only. | IT policy forbids WSL |
| **Windows containers mode** | Windows-based containers on the Windows kernel. | Only when Windows apps must run in containers |
| **Docker VMM** | Listed in Docker's installer docs as a `--backend` value (`docker-vmm`). On Mac it is Docker's own virtual machine manager. | Unverified on Windows. Ignore unless Docker's docs say otherwise. |

### 2.9 Alternative: Docker Engine directly inside WSL (no Docker Desktop)

Docker Engine (free) can be installed inside a WSL 2 Ubuntu exactly as on a Linux server. This avoids Docker Desktop licensing ([Section 6](#6-docker-desktop-licensing-summary)) and matches production. It requires enabling **systemd** inside WSL.

In the Ubuntu terminal:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true
EOF
```

This writes `/etc/wsl.conf`, which tells WSL to start systemd (Linux's service manager) first. Per Microsoft's docs, systemd requires a recent WSL release (`wsl --version` must work).

In PowerShell:

```powershell
wsl --shutdown
```

`wsl --shutdown` stops the WSL VM so the config is read at the next start. Microsoft's docs call this the "8 second rule": the VM must fully stop before changes apply. Reopen Ubuntu, confirm with `systemctl is-system-running`, then follow [Section 4](#4-linux-ubuntu-installing-docker-engine). This setup has no Docker Desktop GUI, Docker Scout integration, or Model Runner.

Do not run Docker Desktop's WSL integration and a separately installed Engine in the same distro without knowing which `docker` is active. `docker context ls` shows the active endpoint.

### 2.10 Limit WSL 2's resources with `.wslconfig`

By default the WSL 2 VM can use up to 50% of Windows RAM and all logical processors, and may keep memory after containers stop. With the WSL 2 backend, Docker Desktop's resource sliders are mostly unused. Limits come from **`.wslconfig`** in the Windows user folder: `C:\Users\<YourName>\.wslconfig`. The file does not exist by default. Create it (verified against Microsoft's WSL configuration page):

```ini
[wsl2]
# Maximum RAM the WSL 2 VM may use (default: 50% of Windows RAM)
memory=12GB

# Number of virtual CPU cores (default: all logical processors)
processors=6

# Swap file size (default: 25% of memory size, rounded up to the nearest GB). 0 disables swap.
swap=4GB

# Make localhost ports in WSL reachable from Windows (default true)
localhostForwarding=true

[experimental]
# Return unused memory to Windows. Values: disabled, gradual, dropCache
autoMemoryReclaim=gradual

# Let the virtual disk shrink as files are deleted (default false)
sparseVhd=true
```

| Key | Meaning |
|---|---|
| `[wsl2]` | INI section header. Keys under it affect the WSL 2 VM. |
| `memory` | Hard cap on RAM for the whole VM, shared by all containers. Units such as `8GB`, `512MB`. |
| `processors` | Logical CPU cores the VM may use. |
| `swap` | Disk-backed overflow memory. `0` disables it. |
| `localhostForwarding` | Makes `http://localhost:8080` on Windows reach a container's published port. |
| `[experimental]` | Opt-in features. `autoMemoryReclaim` returns cached memory to Windows (Microsoft's doc lists `dropCache` as the default; `gradual` reclaims slowly). `sparseVhd=true` makes new virtual disks sparse so they can shrink. |

Optional networking key under `[wsl2]`: `networkingMode=mirrored` (Windows 11 22H2+) makes WSL share the Windows network interfaces, which often fixes VPN and localhost problems. The default is `NAT`.

Apply the changes:

```powershell
wsl --shutdown
```

Then restart Docker Desktop and check:

```powershell
docker info --format "CPUs={{.NCPU}} Mem={{.MemTotal}}"
```

- `docker info`: prints daemon information.
- `--format`: takes a Go template to select fields.
- `NCPU`: CPUs the engine sees. `MemTotal`: memory in bytes.

On the test machine, `docker info` reports 22 CPUs and 15.34 GiB.

Sizing: for AI work with local models, leave Windows enough memory. For a 32 GB machine use `memory=20GB` to `24GB`. For a 16 GB machine use `memory=10GB`. Too little causes containers to be killed with exit code 137 (out of memory). Too much starves Windows.

The Start menu also has a graphical **WSL Settings** app that edits these settings.

### 2.11 The #1 Windows performance tip: keep your project files inside WSL

Docker's WSL best-practices page recommends this. WSL 2 has two filesystems:

| Location | Path in WSL | Speed | File-change notifications to Linux containers |
|---|---|---|---|
| **Linux filesystem** (ext4 virtual disk) | `/home/you/projects/...` (from Windows: `\\wsl.localhost\Ubuntu\home\you\projects`) | Fast | Work |
| **Windows filesystem** (through a network-like bridge, DrvFs/9P) | `/mnt/c/Users/you/projects/...` | Much slower for many small files | Per Docker's docs, Linux containers receive file-change events only for files in the Linux filesystem, so hot reload can break |

Rule: when bind-mounting a project folder into a container (`-v ./myproject:/app`, or in Compose), keep the folder under `/home/you/...` in WSL, not `/mnt/c/...`.

```bash
# In the Ubuntu terminal
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/example/my-ai-app.git     # clone inside WSL
cd my-ai-app
code .          # opens VS Code connected to WSL (needs the VS Code "WSL" extension)
docker compose up
```

`explorer.exe .` (note the dot) opens the current WSL folder in Windows File Explorer.

Speed comparison:

```bash
# Linux filesystem
mkdir -p ~/speedtest && cd ~/speedtest
time (for i in $(seq 1 3000); do echo x > f$i; done)

# Windows filesystem
mkdir -p /mnt/c/Users/Public/speedtest && cd /mnt/c/Users/Public/speedtest
time (for i in $(seq 1 3000); do echo x > f$i; done)
```

`time ( ... )` measures how long creating 3000 small files takes. The `/mnt/c` run is typically several times slower. Exact numbers depend on the machine and antivirus and are not quoted here. Clean up: `rm -rf ~/speedtest /mnt/c/Users/Public/speedtest`.

### 2.12 Windows gotchas

| Issue | Explanation and fix |
|---|---|
| Line endings (CRLF vs LF) | Windows uses `\r\n`, Linux `\n`. A script checked out with CRLF fails in a container with `exec ./entrypoint.sh: no such file or directory` or `/bin/sh^M: bad interpreter`. Fix: add a `.gitattributes` with `* text=auto eol=lf` (or at least `*.sh text eol=lf`) and re-checkout, or run `git config --global core.autocrlf input`. |
| Quoting differences | Line continuation is `\` in Bash, a backtick `` ` `` in PowerShell, and `^` in cmd. Commands in this guide are written for Bash. Chapter 3 shows PowerShell equivalents where needed. |
| Path syntax in `-v` | PowerShell: `-v ${PWD}:/app`. cmd: `-v %cd%:/app`. Bash/WSL: `-v "$(pwd)":/app`. |
| Antivirus slowing bind mounts | Exclude the WSL project folder from real-time scanning if IT policy allows. |
| VPN/corporate proxy breaks DNS or pulls | See [Section 9](#9-common-install-errors-and-fixes). Try `networkingMode=mirrored`. |
| `vmmem` / `vmmemWSL` using much RAM | This is the WSL 2 VM. Cap it with `.wslconfig` ([2.10](#210-limit-wsl-2s-resources-with-wslconfig)) and run `wsl --shutdown` to release memory. |
| Disk space | WSL's virtual disk (`ext4.vhdx`) grows with pulled images and does not always shrink. `sparseVhd=true` helps. `docker system prune` (Chapter 3) frees space inside the disk. |
| GPU for AI | With an NVIDIA GPU, install the normal Windows NVIDIA driver (not a Linux driver inside WSL). Docker Desktop with WSL 2 then supports `--gpus all`. See [Chapter 8](08-docker-for-ai-and-ml.md). |

---

## 3. macOS: Apple Silicon and Intel

### 3.1 Requirements

Verified against Docker's docs, October 2026:

| Item | Detail |
|---|---|
| macOS version | Current and two previous major macOS releases |
| Chip | Two downloads: Apple silicon (M1, M2, M3, M4...) and Intel |
| RAM | At least 4 GB (more for AI work) |
| Rosetta 2 | Recommended on Apple silicon; per Docker, no longer strictly required (some optional command-line tools using Darwin/AMD64 still need it). Install with `softwareupdate --install-rosetta` |

### 3.2 Install

**Option A: DMG.**

1. Download the Docker Desktop DMG for the chip. Apple menu > About This Mac shows "Chip: Apple M..." (Apple silicon) or "Processor: Intel..." (Intel).
2. Open `Docker.dmg` and drag Docker into Applications.
3. Launch Docker and accept the subscription agreement. Per Docker's docs, the default configuration needs no password or extra setup.

**Option B: Homebrew.**

```bash
brew install --cask docker-desktop
```

- `brew`: the macOS package manager.
- `--cask`: install a macOS application rather than a command-line formula.
- Homebrew lists the cask token as `docker-desktop`, renamed from `docker`. On an old Homebrew, run `brew update` first.
- `brew install docker` (no `--cask`) installs only the Docker CLI, with no engine.

Verify as in [Section 7](#7-verifying-your-installation-with-hello-world).

Resources: Docker Desktop > Settings > Resources has sliders for CPUs, Memory, Swap, and Disk image size. Raise memory for AI workloads.

### 3.3 Apple silicon vs Intel: the architecture issue

**CPU architecture** is the instruction set a processor runs.

| Name(s) | Where it appears | Machines |
|---|---|---|
| **amd64** (also `x86_64`, "x64") | Most servers and cloud VMs, Intel/AMD PCs, Intel Macs | Default for most cloud servers |
| **arm64** (also `aarch64`) | Apple silicon Macs, AWS Graviton, Raspberry Pi 4/5, many phones | Increasingly common in cloud |

A compiled program, and therefore a container image's executables, is built for one architecture. An `amd64` binary cannot run natively on an `arm64` CPU, and vice versa.

Common failure: an image built on an Apple silicon Mac is `arm64`. Deployed to a typical `amd64` cloud server, it fails with `exec format error`.

Check the engine's architecture:

```bash
docker info --format '{{.Architecture}}'      # aarch64 on Apple silicon, x86_64 on Intel/AMD
docker run --rm alpine uname -m               # same, from inside a container
```

### 3.4 Multi-architecture images

Many popular images (nginx, python, postgres, ubuntu) are **multi-architecture** ("multi-arch"). One image name such as `python:3.13-slim` is a **manifest list**: an index that maps each architecture (amd64, arm64) to its own set of layers. On pull, Docker selects the entry matching the engine.

Inspect supported platforms:

```bash
docker buildx imagetools inspect python:3.13-slim
# or the older style:
docker manifest inspect python:3.13-slim
```

This asks the registry for the manifest list and prints the platform entries (for example `linux/amd64`, `linux/arm64/v8`, `linux/arm/v7`). The list varies by image and over time.

If an image exists only for amd64 and the host is arm64, Docker warns:

```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```

It still runs the image under **emulation** (slowly) if emulation is available.

### 3.5 The `--platform` flag

`--platform` selects the architecture to pull, build, or run.

```bash
docker run --rm --platform linux/amd64 alpine uname -m
```

On an Apple silicon Mac this prints `x86_64`: the amd64 image ran under emulation. Without `--platform` it prints `aarch64`.

| Platform string | Meaning |
|---|---|
| `linux/amd64` | 64-bit Intel/AMD Linux |
| `linux/arm64` (also `linux/arm64/v8`) | 64-bit ARM Linux (Apple silicon, Graviton) |
| `linux/arm/v7` | 32-bit ARM (older 32-bit Raspberry Pi OS) |

**Emulation.** Docker Desktop can run foreign-architecture containers with emulation (QEMU, and on Apple silicon Rosetta for amd64 where enabled). It can be several times slower, and some software (certain JVM, database, or JIT-heavy programs) can crash under it. Use it for convenience and testing, not production.

### 3.6 Building images that run on servers

When servers are amd64 and the Mac is arm64, build for the target:

```bash
# Build for amd64 only
docker buildx build --platform linux/amd64 -t myapp:1.0 .

# Build for both architectures and push (creates a multi-arch manifest list)
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/you/myapp:1.0 --push .
```

| Part | Meaning |
|---|---|
| `docker buildx build` | Modern build command (BuildKit, Docker's current builder) |
| `--platform` | Comma-separated build targets |
| `-t` | Name and tag for the result |
| `--push` | Upload directly to the registry. Needed for multi-platform results because a single local image store traditionally holds one platform. With the containerd image store, multi-platform images can sometimes load locally. |
| `.` | Build context (current folder) |

Chapter 4 covers builds in depth.

### 3.7 Mac-specific notes

| Topic | Note |
|---|---|
| VM technology | Docker Desktop on Mac runs its Linux VM on a virtualization layer chosen in Settings (Apple's virtualization framework, or Docker's own VMM). Docker's installer docs mention Docker VMM as a Mac option. Unverified: the current default and performance differences. Keep the default unless Docker's docs say otherwise. |
| File sharing | Bind mounts cross the Mac/VM boundary and are slower than container-internal storage. Use **named volumes** (Chapter 5) for heavy data such as databases and `node_modules`. |
| GPU / AI | Linux containers on a Mac cannot use the Mac's GPU directly (no NVIDIA, no Metal passthrough). For local LLM inference on a Mac, use **Docker Model Runner**, which Docker designed to use host resources, or run Ollama natively outside Docker ([Chapter 8](08-docker-for-ai-and-ml.md)). Unverified: Model Runner's GPU details on macOS. Check Docker's current docs. |
| Alternatives | **Colima** (`brew install colima docker`), **OrbStack**, **Rancher Desktop**, **Podman Desktop** run containers on a Mac without Docker Desktop. |
| Apple's own tooling | Apple introduced separate open-source container tooling for macOS. This guide does not use it. Unverified: its current maturity. |

---

## 4. Linux (Ubuntu): installing Docker Engine

This is the production-grade installation. It follows Docker's official "Install Docker Engine on Ubuntu" page (October 2026).

### 4.1 Supported Ubuntu versions and architectures

Docker Engine installs on 64-bit versions of:

- Ubuntu 26.10 ("Stonking")
- Ubuntu 26.04 LTS ("Resolute")
- Ubuntu 24.04 LTS ("Noble")
- Ubuntu 22.04 LTS ("Jammy")

Architectures: x86_64/amd64, armhf, arm64, s390x, ppc64le. Derivatives such as Linux Mint are not officially supported (the repository setup uses the Ubuntu codename, which can fail on derivatives).

**LTS** (Long-Term Support): Ubuntu publishes an LTS release every two years (22.04, 24.04, 26.04) with five years of free security updates. Non-LTS releases such as 26.10 get nine months. Servers should run LTS.

Check the version:

```bash
lsb_release -a        # or: cat /etc/os-release
uname -m              # architecture: x86_64 or aarch64
```

### 4.2 Step 1: remove conflicting unofficial packages

Ubuntu's repositories carry unofficial Docker packages that conflict with Docker's. Per Docker's docs the conflicting packages are `docker.io`, `docker-compose`, `docker-compose-v2`, `docker-doc`, `podman-docker`, and any separately installed `containerd` or `runc`.

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

- `apt-get remove`: uninstalls a package.
- `-y`: answers "yes" automatically.
- "Package 'X' is not installed" messages are harmless.
- Existing images, containers, and volumes in `/var/lib/docker/` are not deleted. Only the packages are removed.

### 4.3 Step 2: set up Docker's official `apt` repository

```bash
# 1. Update the package index and install prerequisites
sudo apt update
sudo apt install -y ca-certificates curl

# 2. Create the keyrings directory and download Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. Add the repository to apt's sources
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

# 4. Refresh the package index so apt sees Docker's packages
sudo apt update
```

| Piece | Meaning |
|---|---|
| `sudo` | Run as administrator. |
| `apt` | Ubuntu's package manager. |
| `ca-certificates`, `curl` | Certificates for HTTPS, and a download tool. |
| `install -m 0755 -d /etc/apt/keyrings` | Creates a directory (`-d`) with mode `0755` (owner writes, everyone reads). Stores trusted signing keys. |
| `curl -fsSL URL -o FILE` | `-f` fail on HTTP errors, `-s` silent, `-S` show errors, `-L` follow redirects, `-o` save to a file. |
| GPG key (`docker.asc`) | Docker's public signing key. `apt` uses it to verify packages come from Docker unmodified. |
| `chmod a+r` | Make the key readable by all users. |
| `tee /etc/apt/sources.list.d/docker.sources` | Writes the here-document to a file with admin rights. `sudo tee` is used because `sudo cat > file` does not elevate the redirect. |
| `Types` / `URIs` / `Suites` / `Components` / `Signed-By` | **deb822** repository format. `Suites` is the Ubuntu codename (for example `noble`) read from `/etc/os-release`. `Components: stable` selects the stable channel. `Signed-By` points to the key. |

Verify:

```bash
cat /etc/apt/sources.list.d/docker.sources
```

Expected (Ubuntu 24.04):

```
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: noble
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
```

### 4.4 Step 3: install the packages

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Package | What it is |
|---|---|
| `docker-ce` | Docker Engine, Community Edition: the `dockerd` daemon. |
| `docker-ce-cli` | The `docker` command-line client. |
| `containerd.io` | The containerd runtime (with runc). |
| `docker-buildx-plugin` | `docker buildx`, the modern builder (BuildKit front end). |
| `docker-compose-plugin` | `docker compose` (Compose v2 and later, as a CLI plugin). |

To install a specific version: `apt list --all-versions docker-ce`, then `sudo apt install docker-ce=<version> docker-ce-cli=<version> ...`. Use this on servers that need version control. `docker-ce-rootless-extras` (for rootless mode, below) is a separate optional package.

The service starts automatically on Ubuntu/Debian after installation:

```bash
sudo systemctl status docker
```

Example output (shortened):

```
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Fri 2026-10-23 09:12:41 UTC; 1min 3s ago
TriggeringUnit: docker.socket
       Docs: https://docs.docker.com
   Main PID: 1234 (dockerd)
```

`enabled` means it starts at boot. `active (running)` means it is up now. Press `q` to leave the pager. If it is not running: `sudo systemctl start docker`.

Test with `sudo` (the user is not in the `docker` group yet):

```bash
sudo docker run hello-world
```

### 4.5 Step 4 (post-install): run Docker without `sudo`

The Docker socket is owned by `root` and the group `docker`. Add the user to the `docker` group to run `docker` without `sudo`:

```bash
sudo groupadd docker 2>/dev/null || true     # the group usually already exists
sudo usermod -aG docker $USER                # add the user to it
newgrp docker                                # activate in the current shell (or log out and back in)
docker run hello-world                       # now works without sudo
```

| Piece | Meaning |
|---|---|
| `groupadd docker` | Creates the group (the error is harmless if it exists). |
| `usermod -aG docker $USER` | `-a` append, `-G docker` to the supplementary group `docker`. Always use `-a`, or the user is removed from other groups. `$USER` expands to the username. |
| `newgrp docker` | Starts a shell with the new group applied. Otherwise log out and back in. |

Security warning: per Docker's post-install page, the `docker` group grants root-level privileges to its members. Anyone who can run `docker` can start a container that mounts the host filesystem and read or write anything as root, for example `docker run -v /:/host --rm -it alpine chroot /host`. This is usually acceptable on a personal laptop. On shared or production servers, add only trusted administrators, or use rootless mode ([4.8](#48-rootless-mode-introduction)) or `sudo`.

### 4.6 Step 5: make sure Docker starts on boot (systemd)

**systemd** is the service manager on nearly all modern Linux distributions. It starts, stops, and supervises background services. Docker is a systemd service.

```bash
sudo systemctl enable docker.service containerd.service   # start at boot
sudo systemctl disable docker.service containerd.service  # (the opposite)
```

Debian and Ubuntu enable it by default. Fedora/RHEL need `enable`.

| Command | Effect |
|---|---|
| `sudo systemctl start docker` | Start the daemon now |
| `sudo systemctl stop docker` | Stop it (also stop `docker.socket` to keep it off, because socket activation restarts it on demand) |
| `sudo systemctl restart docker` | Restart (running containers stop unless `live-restore` is on) |
| `sudo systemctl status docker` | Show state and recent log lines |
| `journalctl -u docker.service -f` | Follow the daemon log (`-u` unit, `-f` follow). First place to look when Docker misbehaves. |
| `sudo systemctl daemon-reload` | Run after editing unit files |

### 4.7 Step 6: configure the daemon (`/etc/docker/daemon.json`) and log rotation

The daemon reads an optional JSON file, `/etc/docker/daemon.json`. If one exists, edit it by hand instead of running the `tee` command below, which overwrites the file. The most useful first setting is log rotation. Docker's post-install page warns that the default `json-file` logging driver keeps container logs without limit and can fill the disk. Docker Desktop's logging driver shows as `local` in the test machine's `docker info`, but on plain Linux Engine the default is `json-file`.

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true
}
EOF
sudo systemctl restart docker
docker info --format '{{.LoggingDriver}}'
```

| Key | Meaning |
|---|---|
| `log-driver: local` | Use the `local` logging driver: efficient format, rotates by default. (Alternative: keep `json-file` and set `max-size`/`max-file`.) |
| `max-size: 10m` | Rotate when a log file reaches 10 MB. |
| `max-file: 3` | Keep at most 3 rotated files per container (about 30 MB). |
| `live-restore: true` | Keep containers running while the daemon restarts or upgrades. Not compatible with Swarm mode. |

Existing containers keep their old logging settings. New defaults apply to containers created afterwards. Invalid JSON prevents the daemon from starting. Check `journalctl -u docker.service`.

### 4.8 Rootless mode (introduction)

**Rootless mode** runs the Docker daemon and containers as a normal unprivileged user. If a container escapes, the attacker has only that user's privileges. It uses user namespaces (Chapter 1) to map "root inside" to the user's unprivileged UID outside.

Prerequisites (per Docker's rootless documentation):

- `newuidmap` and `newgidmap` installed (the `uidmap` package: `sudo apt install -y uidmap`).
- At least 65,536 subordinate UIDs/GIDs for the user in `/etc/subuid` and `/etc/subgid`. On Ubuntu, creating a normal user usually sets this up. Check with `grep $USER /etc/subuid /etc/subgid`.
- The `docker-ce-rootless-extras` package (or Docker's rootless install script).

Install:

```bash
sudo apt install -y uidmap docker-ce-rootless-extras
# If the system-wide daemon is running, Docker recommends disabling it first:
sudo systemctl disable --now docker.service docker.socket
# As the normal user (not root, no sudo):
dockerd-rootless-setuptool.sh install
```

The tool creates a systemd user service at `~/.config/systemd/user/docker.service` and prints instructions. Add the printed `export PATH=...` line and set `DOCKER_HOST` (for UID 1000: `unix:///run/user/1000/docker.sock`). Manage the service:

```bash
systemctl --user start docker.service        # start (note --user)
systemctl --user restart docker.service
systemctl --user status docker.service
sudo loginctl enable-linger $USER            # keep it running after logout / start at boot
docker info | grep -i rootless               # should show "rootless" in security options and context
```

| Piece | Meaning |
|---|---|
| `dockerd-rootless-setuptool.sh install` | Docker's helper that configures the rootless daemon for the user. |
| `systemctl --user` | Controls per-user systemd services instead of system-wide ones. |
| `loginctl enable-linger` | Lets user services run without an active login session and start at boot. |
| `DOCKER_HOST` | Environment variable telling the CLI which daemon socket to use. |

Trade-offs: Docker's page has a Troubleshooting section. Unverified: the complete limitation list could not be confirmed from the fetched page. Commonly known limits: binding ports below 1024 needs extra configuration, some networking features and storage drivers are limited, and network performance can be lower. Rootless mode suits dev machines and shared hosts. See also [Chapter 10](10-security.md).

### 4.9 Firewall caveat (important)

Per Docker's Ubuntu page, publishing a container port (`-p 8080:80`) adds `iptables` rules that bypass `ufw` and `firewalld` rules. `sudo ufw deny 8080` does not block external access to a published port. Docker works only with `iptables-nft` and `iptables-legacy`; rules created directly with `nft` are not supported on a system with Docker installed. To restrict published ports, add rules to the `DOCKER-USER` chain, or bind the port to localhost only (`-p 127.0.0.1:8080:80`). See also Chapters 5 and 10.

### 4.10 Other Linux distributions

The pattern is the same: add Docker's repository, then install `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.

| Distribution | Docs page | Notes |
|---|---|---|
| Debian | "Install Docker Engine on Debian" | Nearly identical to Ubuntu, using `download.docker.com/linux/debian`. |
| Fedora | "Install Docker Engine on Fedora" | Uses `dnf`. The service is not enabled automatically: `sudo systemctl enable --now docker`. |
| RHEL / CentOS Stream / Rocky / Alma | "Install Docker Engine on RHEL/CentOS" | Uses `dnf`. Podman is the native tool on RHEL. |
| Arch | Package `docker` from the official repos | Community maintained. |
| Raspberry Pi OS / ARM | The Debian instructions | Per the release notes, Docker Engine 29 stopped providing 32-bit Raspbian packages, and Debian armhf packages target ARMv7. |

**Convenience script:** `curl -fsSL https://get.docker.com | sh` installs Docker in one command. It suits throwaway dev VMs. Docker documents it as not intended for production. Use the repository method for long-lived systems to control versions and updates.

### 4.11 Docker Desktop for Linux (optional)

Docker Desktop for Linux (`.deb`/`.rpm`) adds the GUI, extensions, and Model Runner on a Linux workstation. Requirements per Docker: Ubuntu, Debian, RHEL, or Fedora (x86-64), KVM virtualization, QEMU 5.2+, systemd, 4 GB RAM, and a GNOME, KDE, or MATE desktop. It runs a VM with its own separate images and containers (Engine's are not visible in Desktop) and creates a `desktop-linux` Docker context. The same license terms apply. On servers and most Linux workstations, plain Docker Engine is the better choice.

---

## 5. Docker Engine on servers

On a production host (cloud VM such as AWS EC2, Azure VM, GCE, or bare metal), install Docker Engine only, without Docker Desktop.

| Step | Reason |
|---|---|
| Use a Linux LTS distribution (Ubuntu LTS, Debian stable, RHEL-family) | Long security support |
| Install from Docker's official repo; pin versions and update deliberately | Avoid surprise upgrades |
| Configure log rotation in `daemon.json` ([4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)) | Prevent full disks |
| Consider `live-restore: true` | Keep containers up through daemon restarts |
| Do not add everyday users to the `docker` group; use `sudo` or rootless | The group is root-equivalent |
| Never expose the Docker API on TCP without TLS (unencrypted port 2375 is a common attack path) | Anyone who reaches it controls the machine |
| Control access with a firewall or cloud security group; Docker bypasses `ufw` for published ports | See [4.9](#49-firewall-caveat-important) |
| Enable automatic OS security updates | Kernel and library patches |
| Run containers as non-root users; set resource limits and restart policies | Chapters 3 and 10 |
| Monitor disk (`docker system df`) and prune safely | Images and build cache accumulate |
| Pull from a private registry under your control | Reliability, rate limits, security |

**Managing a remote server from a laptop:** use a Docker **context** over SSH. The daemon stays private and SSH handles authentication (no open Docker port).

```bash
docker context create myserver --docker "host=ssh://deploy@203.0.113.10"
docker context use myserver
docker ps                    # now lists containers on the remote server
docker context use default   # switch back to local
```

`docker context create NAME --docker "host=ssh://USER@HOST"` stores a named endpoint. The local CLI then tunnels API calls over SSH to the remote `dockerd`. The IP address is from a reserved documentation range; replace it. The remote user must be able to run `docker`. See [Chapter 3](03-core-cli-commands.md) for contexts and [Chapter 11](11-production-and-enterprise.md) for production deployments.

---

## 6. Docker Desktop licensing summary

Not legal advice. The figures come from Docker's pricing FAQ and Docker Desktop install pages, October 2026. Prices and terms change. Legal or procurement teams decide. Check Docker's current Subscription Service Agreement.

### 6.1 What is free?

**Docker Engine** (the open-source Linux runtime) is free, with no license restrictions on the software. Docker's FAQ says open-source software such as Docker Engine is accessible to all and supported by the Moby project maintainers and community.

**Docker Desktop** is free only for:

- personal use,
- education,
- non-commercial open-source projects,
- small businesses: fewer than 250 employees and less than US $10 million in annual revenue.

Quoted from Docker's terms: use of Docker Desktop without a paid subscription is restricted to "(i) your use for a non-commercial open source project and/or (ii) use in a commercial undertaking with fewer than 250 employees and less than US $10,000,000 (or equivalent local currency) in annual revenue. Government Entities shall not use Docker Desktop ... without purchasing a Subscription."

### 6.2 When payment is required

If the organization has 250 or more employees or $10 million or more in annual revenue (either threshold triggers it), professional use of Docker Desktop requires a paid subscription (Pro, Team, or Business) per user. Also:

- Government entities need a paid subscription regardless of size (per the quoted terms).
- Unverified (from secondary sources quoting Docker's FAQ, not checked on Docker's page): a small company that is a subsidiary of a larger one can fall under the larger company's headcount.
- Per third-party summaries, licensing is per user, not per machine. Confirm with Docker.
- Using only the command line does not exempt a user. The license attaches to Docker Desktop, the product.

### 6.3 Plans and prices

From Docker's FAQ, October 2026:

| Plan | Per user per month (annual billing) | Month-to-month | Intended for |
|---|---|---|---|
| Personal | $0 | $0 | Individuals, education, open source, small companies |
| Pro | $9 | $11 | Individual professionals needing more features |
| Team | $15 | $16 | Small teams (minimum 1 user per the FAQ) |
| Business | $24 | $24 | Larger organizations (SSO, security and governance controls) |

Docker's FAQ stated these prices took effect on December 10, 2024 for new customers and at renewal for existing customers. Check Docker's pricing page for current prices.

### 6.4 Alternatives to paying

| Option | Notes |
|---|---|
| Docker Engine on Linux | Free for everyone. Use a Linux workstation, VM, or server. |
| Docker Engine inside WSL 2 (Windows) | Free: [Section 2.9](#29-alternative-docker-engine-directly-inside-wsl-no-docker-desktop). |
| Podman / Podman Desktop | Free, open source, Docker-compatible CLI. |
| Rancher Desktop | Free, open source, runs containerd or dockerd. |
| Colima (macOS/Linux) | Free CLI tool. |
| OrbStack (macOS) | Has its own commercial/personal-use terms. Check them. |

Docker Hub has separate account tiers and pull limits ([Chapter 1](01-what-is-docker.md#91-docker-hub)). Public images from Docker Hub can be used without Docker Desktop.

---

## 7. Verifying your installation with hello-world

### 7.1 Quick checks

```bash
docker --version
docker compose version
```

Example output (test machine):

```
Docker version 29.9.2, build a82e528
Docker Compose version v5.6.1
```

- `docker --version`: prints the client version and build.
- `docker compose version`: confirms the Compose plugin is installed. `docker: 'compose' is not a docker command` means the plugin is missing (install `docker-compose-plugin` on Linux, or reinstall Docker Desktop).

```bash
docker version
```

A `Client:` block and a `Server:` block mean the CLI reached the daemon. `Client:` followed by an error such as `Cannot connect to the Docker daemon` means the engine is not running ([Section 9](#9-common-install-errors-and-fixes)).

```bash
docker info
```

`docker info` prints daemon details: container and image counts, storage driver, logging driver, cgroup version, kernel version, OS, CPUs, memory, runtimes. Relevant lines from the test machine:

```
 Server Version: 29.9.2
 Storage Driver: overlayfs
 Logging Driver: local
 Cgroup Driver: cgroupfs
 Cgroup Version: 2
 Runtimes: io.containerd.runc.v2 nvidia runc
 Default Runtime: runc
 Kernel Version: 6.18.54.1-microsoft-standard-WSL2
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 22
 Total Memory: 15.34GiB
```

`OSType: linux` and the `...microsoft-standard-WSL2` kernel show a Linux engine in a WSL 2 VM on Windows. `Runtimes` lists `runc` as the default OCI runtime and an `nvidia` runtime added by the GPU toolkit.

### 7.2 `docker run hello-world`

```bash
docker run hello-world
```

`docker run` creates and starts a container. `hello-world` is the image: a tiny official image containing one small executable that prints a message and exits.

First-run output (digest and layer IDs differ per machine, and the format changes slightly between versions):

```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
e6590344b1a5: Pull complete
Digest: sha256:<a long hash that will differ on your machine>
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

### 7.3 Every line explained

| Output line | Meaning |
|---|---|
| `Unable to find image 'hello-world:latest' locally` | The local image store has no `hello-world` image with tag `latest` (the default tag). Not an error; Docker will download it. Absent on later runs. |
| `latest: Pulling from library/hello-world` | The daemon downloads from Docker Hub. `library/` is the namespace for Docker Official Images, `hello-world` the repository, `latest` the tag ([Chapter 1](01-what-is-docker.md#66-the-anatomy-of-an-image-name)). |
| `e6590344b1a5: Pull complete` | One layer, identified by a short hash prefix, finished downloading and unpacking. Larger images show many such lines with progress bars. The hash is an example and can differ because the image is rebuilt over time. |
| `Digest: sha256:...` | The immutable content hash of the pulled image. Pin it with `hello-world@sha256:...` to get the same bytes every time. |
| `Status: Downloaded newer image for hello-world:latest` | The pull succeeded and the image is stored locally. If already current: `Status: Image is up to date for hello-world:latest`. |
| (blank line) | End of pull messages. What follows is the container's own output. |
| `Hello from Docker!` and the next line | Printed by the program inside the container. It confirms the whole chain works: CLI, daemon, registry download, container creation, execution, output streaming. |
| `1. The Docker client contacted the Docker daemon.` | The CLI sent a REST API request to `dockerd` over the socket or named pipe ([Chapter 1](01-what-is-docker.md#8-docker-architecture-what-runs-when-you-type-docker-run)). |
| `2. The Docker daemon pulled the "hello-world" image from the Docker Hub. (amd64)` | The daemon fetched the image. `(amd64)` is the selected CPU architecture. On Apple silicon or ARM servers it shows `(arm64v8)`. The match is automatic through the multi-arch manifest list. |
| `3. The Docker daemon created a new container ...` | The daemon created a container from the image and started its single program. |
| `4. The Docker daemon streamed that output ...` | The container's standard output (stdout) was forwarded through the daemon to the CLI and the terminal. |
| `To try something more ambitious ... $ docker run -it ubuntu bash` | Suggestion (`-i` keeps input open, `-t` allocates a terminal; see [Chapter 3](03-core-cli-commands.md)). The remaining lines are links to Docker Hub and the getting-started guide. |

### 7.4 State after the run

The program printed its text and exited. The container is in the **Exited** state (stopped, not deleted).

```bash
docker ps -a
```

Example output:

```
CONTAINER ID   IMAGE         COMMAND    CREATED          STATUS                      PORTS     NAMES
3f4a1b9c2d7e   hello-world   "/hello"   10 seconds ago   Exited (0) 9 seconds ago              vibrant_turing
```

| Column | Meaning |
|---|---|
| `CONTAINER ID` | Short form of the container's unique ID. |
| `IMAGE` | The image it was created from. |
| `COMMAND` | What ran inside: `/hello`, the one executable in the image. |
| `CREATED` | Creation time. |
| `STATUS` | `Exited (0)`: finished with exit code 0 (success). A non-zero code means failure ([Chapter 3](03-core-cli-commands.md)). |
| `PORTS` | Published ports (none). |
| `NAMES` | Random generated name (adjective + surname). Set one with `--name`. |

`-a` shows all containers. Without it, `docker ps` shows only running ones. Clean up:

```bash
docker rm vibrant_turing        # use the container name or ID from the list
docker rmi hello-world          # optional: remove the image (rmi = remove image)
```

### 7.5 A second check

```bash
docker run --rm -it alpine sh
```

In the shell, run `cat /etc/os-release`, `uname -a`, `ls`, then `exit`. `--rm` deletes the container on exit and `-it` provides an interactive terminal. If this works, Docker is functional. Continue with [Chapter 3](03-core-cli-commands.md).

---

## 8. Uninstall and cleanup

Warning: removing `/var/lib/docker`, or using "Clean / Purge data" in Docker Desktop, permanently deletes all images, containers, volumes (including database data), and networks.

### 8.1 Optional cleanup from inside Docker

Preview disk usage:

```bash
docker system df
```

Remove everything stopped or unused (see the warnings in [Chapter 3](03-core-cli-commands.md#13-disk-usage-and-cleanup-system-df-and-prune)):

```bash
docker system prune -a --volumes
```

### 8.2 Windows (Docker Desktop)

1. Docker Desktop > Troubleshoot (bug icon) > **Clean / Purge data** (deletes all Docker data).
2. Quit Docker Desktop (tray icon > Quit Docker Desktop).
3. Windows Settings > Apps > Installed apps > Docker Desktop > Uninstall, or `winget uninstall -e --id Docker.DockerDesktop`.
4. Optional leftovers: delete `%USERPROFILE%\.docker` (CLI config, credentials, contexts), `%APPDATA%\Docker`, `%APPDATA%\Docker Desktop`, `%LOCALAPPDATA%\Docker`.
5. WSL data: the installer normally removes the `docker-desktop` WSL distros. Check with `wsl -l -v`. If `docker-desktop` remains, run `wsl --unregister docker-desktop`. `wsl --unregister` permanently deletes that distro's data. Do not unregister your own Ubuntu unless it and its contents should be deleted.
6. Optional: remove `.wslconfig` if it was created only for Docker.

### 8.3 macOS

1. Docker Desktop > Troubleshoot > Clean / Purge data (optional).
2. Quit Docker, then drag Docker.app from Applications to the Trash, or run `brew uninstall --cask docker-desktop`.
3. Optional leftovers: `~/.docker`, `~/Library/Containers/com.docker.docker`, `~/Library/Group Containers/group.com.docker`, `~/Library/Application Support/Docker Desktop`.

### 8.4 Ubuntu / Debian (Docker Engine)

From Docker's official uninstall steps:

```bash
# 1. Remove the packages
sudo apt purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

# 2. (DESTRUCTIVE, optional) delete all images, containers, volumes, and networks
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd

# 3. Remove the repository definition and key
sudo rm /etc/apt/sources.list.d/docker.sources
sudo rm /etc/apt/keyrings/docker.asc
```

`apt purge` removes the packages and their system configuration files. Hand-created files such as `/etc/docker/daemon.json` must be removed manually: `sudo rm -rf /etc/docker`. To remove the user from the group: `sudo gpasswd -d $USER docker`. For rootless mode: `dockerd-rootless-setuptool.sh uninstall` and `rootlesskit rm -rf ~/.local/share/docker`.

---

## 9. Common install errors and fixes

| Symptom / error text | Cause | Fix |
|---|---|---|
| `error during connect: ... open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.` (Windows) | The engine is not running. Docker Desktop is closed or still starting. | Start Docker Desktop and wait for "Engine running". `docker context ls` should show `desktop-linux` active. |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` (Linux) | `dockerd` is not running, or the wrong socket/context is in use | `sudo systemctl start docker`; check `sudo systemctl status docker` and `journalctl -u docker.service`. With rootless or Desktop, check `echo $DOCKER_HOST` and `docker context ls`. |
| `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` | User is not in the `docker` group | `sudo usermod -aG docker $USER`, then log out and back in (or `newgrp docker`). See [4.5](#45-step-4-post-install-run-docker-without-sudo) and its security warning. |
| `WSL 2 installation is incomplete` / "WSL needs updating" | Old or missing WSL kernel | PowerShell (admin): `wsl --update`, then `wsl --shutdown` and restart Docker Desktop. |
| `Docker Desktop requires the Server service to be enabled` | Windows `LanmanServer` service disabled | `services.msc` > Server > Startup type Automatic > Start. |
| `Hardware assisted virtualization and data execution protection must be enabled in the BIOS` / `Virtualization support not detected` | Virtualization disabled in BIOS/UEFI | Enable Intel VT-x / AMD-V (SVM). See [2.3](#23-step-0-check-that-virtualization-is-enabled). |
| `Docker Desktop is unable to start` / stuck at "Starting the Docker Engine..." | Corrupted state, WSL problem, or conflicting software | `wsl --shutdown`, restart Docker Desktop, update WSL. Check Troubleshoot > Reset to factory defaults (deletes containers and images) and Windows Event Viewer. |
| `The command 'docker' could not be found in this WSL 2 distro. We recommend to activate the WSL integration in Docker Desktop settings.` | WSL integration disabled | Docker Desktop > Settings > Resources > WSL integration: enable the distro, Apply & restart ([2.6](#26-step-3-connect-docker-to-your-ubuntu-distro-wsl-integration)). |
| `no matching manifest for linux/arm64/v8 in the manifest list entries` (or `for windows/amd64`) | The image does not publish this architecture/OS. `windows/amd64` on Windows means Windows containers mode. | Add `--platform linux/amd64` (emulated), use an arm64-capable image, or switch Docker Desktop to Linux containers. |
| `exec format error` or `exec /usr/local/bin/entrypoint.sh: exec format error` | Architecture mismatch (arm64 image on amd64 or vice versa), or a script missing its `#!` line | Build with `--platform`. Confirm the script's first line is `#!/bin/sh` or `#!/usr/bin/env bash`. |
| `exec ./start.sh: no such file or directory` though the file exists | Windows CRLF line endings | Convert to LF; use `.gitattributes` ([2.12](#212-windows-gotchas)). |
| `toomanyrequests: You have reached your pull rate limit` | Docker Hub unauthenticated/Personal limit ([Chapter 1](01-what-is-docker.md#91-docker-hub)) | `docker login`, wait up to six hours, or use a mirror/cache. |
| `x509: certificate signed by unknown authority` when pulling | A corporate TLS-inspecting proxy re-signs HTTPS traffic with a company certificate that Docker does not trust | Install the company root CA in the OS/Docker trust store (Docker Desktop: add to the OS keychain/certificate store. Linux: copy it to `/etc/docker/certs.d/<registry>/ca.crt` or the system CA store, then restart Docker). Ask IT for the certificate. Do not disable TLS verification. |
| Pulls hang, or `Temporary failure in name resolution` / DNS failures behind a VPN | VPN changes DNS/routing | Try `networkingMode=mirrored` in `.wslconfig` (Windows 11), restart WSL, or set DNS in `daemon.json`: `"dns": ["1.1.1.1", "8.8.8.8"]`. |
| `Bind for 0.0.0.0:80 failed: port is already allocated` / `address already in use` | Another program or container uses the host port | Use another host port (`-p 8080:80`), or find the user: Linux `sudo ss -ltnp \| grep :80`; Windows `netstat -ano \| findstr :80`. |
| `E: Unable to locate package docker-ce` (Ubuntu) | Repo not added correctly, `apt update` not run, or an unsupported derivative | Re-check `/etc/apt/sources.list.d/docker.sources`; run `sudo apt update`. On Linux Mint and other derivatives set `Suites:` to the underlying Ubuntu codename (unsupported by Docker). |
| `NO_PUBKEY` / GPG error / `The following signatures couldn't be verified` | Missing or unreadable key | Re-download the key to `/etc/apt/keyrings/docker.asc` and run `sudo chmod a+r` on it. |
| `E: Conflicting values set for option Signed-By` | Old `docker.list` and new `docker.sources` both exist | Remove the stale Docker entry in `/etc/apt/sources.list.d/`. |
| Docker Desktop fails to install on Windows Server | Unsupported | Docker's docs state Windows Server editions are not supported by Docker Desktop. Use a Linux VM or Docker Engine. |
| `docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]` | `--gpus` used but no GPU runtime/toolkit | See [Chapter 8](08-docker-for-ai-and-ml.md) (NVIDIA Container Toolkit, drivers). |
| Disk full / `no space left on device` | Images, build cache, volumes, unrotated logs | `docker system df`, then selective prune ([Chapter 3](03-core-cli-commands.md)). Set log rotation ([4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)). On Windows, compact the WSL disk. |
| New `docker images` table with `DISK USAGE`, `CONTENT SIZE`, `EXTRA` columns | Not an error. The containerd image store, the default for fresh Engine 29 installs, uses this layout. | Normal. See [Chapter 3](03-core-cli-commands.md#32-docker-images--docker-image-ls-list-local-images). |

General debugging:

1. `docker version`: a `Server:` block means the daemon is reachable.
2. `docker context ls`: shows which engine the CLI targets (an asterisk marks the active one).
3. Read the log. Linux: `journalctl -u docker.service -n 100 --no-pager`. Docker Desktop: Troubleshoot menu or its log folder.
4. Search the exact error text in Docker forums, Stack Overflow, and GitHub issues.

---

## 10. Common mistakes

| Symptom | Cause | Fix |
|---|---|---|
| Missing `sudo` and other groups after setup | `usermod -G` used without `-a` | Always `usermod -aG docker $USER` |
| `docker` still needs `sudo` after `usermod` | Group membership not applied yet | `newgrp docker` or log out and in |
| Slow bind mounts, broken hot reload on Windows | Project files in `C:\Users\...` (`/mnt/c`) | Keep them in `~/projects` inside WSL |
| Windows becomes sluggish | WSL may use most RAM by default | Set `memory=` in `.wslconfig` |
| `exec format error` in production | Image built on Apple Silicon without `--platform` | `docker buildx build --platform linux/amd64 ...` |
| `ufw deny 8080` does not block a container port | Docker bypasses `ufw` through iptables | Bind to `127.0.0.1` or use the `DOCKER-USER` chain |
| Daemon reachable from the network | Daemon exposed on TCP 2375 | SSH contexts or TLS-protected port 2376 |
| Disk fills up | Default `json-file` logs are unbounded | Log rotation in `daemon.json` |
| Pulls fail behind a corporate proxy | TLS inspection with an untrusted CA | Install the corporate CA. Do not disable TLS verification. |

---

## 11. Try it yourself

1. Install Docker for the OS ([Windows](#2-windows-11-step-by-step), [macOS](#3-macos-apple-silicon-and-intel), [Linux](#4-linux-ubuntu-installing-docker-engine)).
2. Run `docker version`, `docker info`, and `docker run hello-world`. Match each output line to the table in [7.3](#73-every-line-explained).
3. Print the engine's facts:
   ```bash
   docker info --format 'OS={{.OperatingSystem}} Arch={{.Architecture}} CPUs={{.NCPU}} MemBytes={{.MemTotal}} Storage={{.Driver}} Cgroup={{.CgroupVersion}}'
   ```
4. Windows: create `%UserProfile%\.wslconfig` with a `memory=` limit, run `wsl --shutdown`, restart Docker Desktop, and confirm with `docker info --format "{{.MemTotal}}"`. Run the speed test from [2.11](#211-the-1-windows-performance-tip-keep-your-project-files-inside-wsl).
5. Mac: compare `docker run --rm alpine uname -m` with `docker run --rm --platform linux/amd64 alpine uname -m`.
6. Linux: check `sudo systemctl status docker` and `journalctl -u docker.service -n 20 --no-pager`. Add log rotation ([4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)).
7. Run `docker context ls` and identify the active context and its endpoint.
8. Run `docker run --rm -it alpine sh`, then `exit`, then `docker ps -a` to confirm `--rm` removed the container.

---

## 12. Summary

- Linux is the production OS. Windows and macOS run Linux containers in a hidden Linux VM.
- Windows: `wsl --install`, Docker Desktop with the WSL 2 backend, limits in `.wslconfig`, project files inside WSL.
- macOS: watch `arm64` vs `amd64`; use `--platform`.
- Linux: install Docker Engine from the official apt repo, configure log rotation, treat the `docker` group as root-equivalent.
- Docker Engine is free. Docker Desktop is free for personal, education, open-source use, and companies under 250 employees and $10M revenue.
- Verify with `docker version` (Client and Server) and `docker run hello-world`.

**Next:** [Chapter 3: Core CLI Commands](03-core-cli-commands.md)
