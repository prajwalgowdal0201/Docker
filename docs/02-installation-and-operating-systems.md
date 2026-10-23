# Chapter 2: Installation and Operating Systems

> **What you will learn in this chapter**
> - Which operating system to use for learning vs production (a clear answer)
> - Why containers are "Linux-based" and what that means for Windows and Mac users
> - A step-by-step **Windows 11** install: WSL 2, Docker Desktop, resource limits, and the single biggest performance tip
> - A **macOS** install, Apple Silicon vs Intel, and the `--platform` / multi-architecture issues that bite Mac users
> - A step-by-step **Ubuntu Linux** install of Docker Engine from the official repository, post-install steps, systemd, and rootless mode
> - Docker Engine on servers
> - A plain-English summary of **Docker Desktop licensing**
> - How to verify your install with `hello-world` and understand **every line** of its output
> - Uninstall and cleanup for each platform
> - The most common install errors and exactly how to fix them
>
> **Time to read:** 60 to 75 minutes, plus install time. **Prerequisite:** [Chapter 1](01-what-is-docker.md) (or at least know what an image and a container are).
>
> **Facts in this chapter** (system requirements, licensing thresholds, package names, supported Ubuntu versions) were checked against Docker's and Microsoft's official documentation while writing, in October 2026. Software changes. If a step in this chapter fails, the official page linked in the text is the source of truth, and the troubleshooting table in [Section 9](#9-common-install-errors-and-fixes) will likely save you.

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

| Purpose | Recommended OS | Why |
|---|---|---|
| **Production** (servers, cloud, Kubernetes nodes) | **Linux.** Always. | Containers are a Linux technology. On Linux, containers run **directly on the host kernel**: fastest, simplest, most secure, best supported. |
| **Learning / local development** | **Whatever your computer already is**, with the right setup: Windows 11 (with WSL 2), macOS, or Linux | Docker Desktop on Windows and macOS gives you a real Linux engine inside a lightweight VM. Everything you learn transfers 1:1 to Linux servers. |
| **Best learning experience if you have a choice** | Linux (e.g., Ubuntu), or Windows with WSL 2 | You see exactly what production sees; no VM layer to explain. |

If you remember only one sentence: **develop on anything, deploy on Linux.**

### 1.2 Why are containers "Linux-based"?

Recall from [Chapter 1](01-what-is-docker.md#5-how-containers-actually-work) that containers are made from Linux kernel features: **namespaces** (what a process sees), **cgroups** (what it can use), and **overlay filesystems**. These features exist in the **Linux kernel**. Windows and macOS have different kernels without those exact features.

Consequences:

- A "Linux container" needs a **Linux kernel** underneath it. On Windows and macOS there is none, so Docker Desktop starts a hidden Linux virtual machine and runs the containers inside it.
- The overwhelming majority of container images (nginx, postgres, python, node, pytorch...) are Linux images.
- Cloud providers run containers on Linux servers. Kubernetes worker nodes are almost always Linux.

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

Microsoft also built **Windows containers**: containers whose contents are Windows programs, running on the Windows kernel. They are real, but a niche:

| | Linux containers | Windows containers |
|---|---|---|
| Image content | Linux binaries | Windows binaries (e.g., .NET Framework apps, IIS) |
| Kernel needed | Linux | Windows (matching version for process isolation) |
| Image size | Tens of MB to hundreds of MB | Often several GB (the Windows base images are large) |
| Typical use | Nearly everything: web, AI/ML, data, microservices | Legacy Windows Server apps being modernized (old .NET Framework, IIS, COM) |
| Ecosystem | Enormous | Small |
| Availability | Everywhere | Windows hosts only; Docker Desktop requires Windows 10/11 Pro or Enterprise for Windows containers (Home and Education run Linux containers only, per Docker's docs) |

Windows containers have two isolation modes: **process isolation** (shares the host Windows kernel, like Linux containers; host and container Windows versions must match closely) and **Hyper-V isolation** (each container gets a thin utility VM with its own kernel; more isolated, slower to start). Docker Desktop can switch between "Linux containers mode" and "Windows containers mode".

**For AI development you will almost never use Windows containers.** PyTorch, CUDA, vLLM, Ollama, Postgres, Redis, Qdrant: all are Linux images. Whenever this guide says "container", it means **Linux container** unless stated otherwise. If you ever see an error mentioning `no matching manifest for windows/amd64`, you have Docker Desktop switched to Windows containers mode by mistake (switch it back via the whale icon, "Switch to Linux containers...").

### 1.4 Option summary by platform

| Platform | How you get Docker | Where containers run | Notes |
|---|---|---|---|
| **Linux** | **Docker Engine** (free, native), optionally Docker Desktop for Linux | Directly on the host kernel | Production-equivalent. No VM. |
| **Windows 11** | **Docker Desktop** with the **WSL 2** backend (recommended), or Docker Engine installed inside a WSL 2 Linux distro | In WSL 2's lightweight Linux VM | Best choice for most Windows users. |
| **macOS** | **Docker Desktop** (or alternatives: Colima, OrbStack, Podman Desktop, Rancher Desktop) | In a lightweight Linux VM | Apple Silicon Macs are ARM (`arm64`), which matters for images ([Section 3](#3-macos-apple-silicon-and-intel)). |
| **Windows Server** | Docker Engine / Mirantis Container Runtime for *Windows containers* | Windows kernel | Docker Desktop does **not** support Windows Server editions, per its docs. Niche. |

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
 |  |   - Ubuntu           (YOUR distro, optional but recommended) |  |
 |  |   Docker Engine (dockerd, containerd, runc) lives here       |  |
 |  +--------------------------------------------------------------+  |
 +--------------------------------------------------------------------+
```

**What is WSL 2?** The **Windows Subsystem for Linux, version 2** is a Microsoft feature that runs a real Linux kernel (built and updated by Microsoft) in a very lightweight virtual machine, tightly integrated with Windows. You can open an Ubuntu terminal on Windows, with real Linux commands, `apt`, `bash`, etc. WSL 1 (the older version) *translated* Linux system calls and did not run a real kernel. WSL 2 does, which is why Docker needs WSL 2.

### 2.2 System requirements (verified against Docker's docs, October 2026)

For the **WSL 2 backend** on x86-64 machines:

| Requirement | Detail |
|---|---|
| **WSL version** | WSL **2.1.5 or later** (use 2.6 or later if you want Docker's Enhanced Container Isolation) |
| **Windows version** | Windows 11 64-bit **23H2 (build 22631) or higher**, or Windows 10 64-bit **22H2 (build 19045)** (Enterprise, Pro, or Education, as listed on Docker's page) |
| **Windows Server service** | The "Server" service (`LanmanServer`) enabled and set to Automatic (it is by default) |
| **Windows features** | WSL 2 turned on (the `wsl --install` command does this) |
| **CPU** | 64-bit processor with **SLAT** (Second Level Address Translation: a hardware feature virtually every CPU since ~2010 has) |
| **RAM** | **8 GB** is the documented minimum. More is better, especially for AI work (16 GB+ recommended). |
| **Virtualization in BIOS/UEFI** | **Must be enabled** (Intel VT-x or AMD-V / "SVM") |

> **A note about Windows Home.** Docker's requirements page lists "Enterprise, Pro, or Education" for the Windows version bullets, but another line on the same page says Home and Education editions run Linux containers only, which implies Home is usable for Linux containers. Historically, Docker Desktop with WSL 2 has worked on Windows Home. I could not fully reconcile these two statements on the current page, so if you are on Home and the installer refuses, check [Docker's Windows install page](https://docs.docker.com/desktop/setup/install/windows-install/) and consider the "Docker Engine directly inside WSL" route in [Section 2.9](#29-alternative-docker-engine-directly-inside-wsl-no-docker-desktop). Windows containers (not needed in this guide) require Pro or Enterprise.

### 2.3 Step 0: check that virtualization is enabled

Docker (through WSL 2) needs your CPU's virtualization extensions switched on.

1. Press `Ctrl + Shift + Esc` to open **Task Manager**.
2. Open the **Performance** tab and select **CPU**.
3. Look at the bottom right for **Virtualization:** it should say **Enabled**.

If it says **Disabled**, restart into your BIOS/UEFI setup (usually by pressing `F2`, `Del`, `F10`, or `Esc` at boot; it varies by manufacturer), find a setting named **Intel Virtualization Technology**, **VT-x**, **SVM Mode**, or **AMD-V**, enable it, save, and reboot.

You can also check from PowerShell:

```powershell
systeminfo | Select-String "Hyper-V Requirements" -Context 0,4
```

What this does: `systeminfo` prints machine details; `Select-String` filters the output for the "Hyper-V Requirements" block. If a hypervisor is already running (which is the case once WSL 2 is installed), it prints "A hypervisor has been detected," which also means virtualization is on.

### 2.4 Step 1: install WSL 2

Open **PowerShell as Administrator** (right-click the Start button, choose **Terminal (Admin)** or **Windows PowerShell (Admin)**).

```powershell
wsl --install
```

What it does: enables the required Windows features (Windows Subsystem for Linux and Virtual Machine Platform), downloads the latest WSL Linux kernel, and installs the **Ubuntu** distribution by default. Per Microsoft, this single command works on Windows 10 version 2004 (build 19041) and higher and on Windows 11.

**Restart your computer** when asked. After the reboot, an **Ubuntu** window opens automatically and finishes installing. It will ask you to create a **Linux username and password**. These are only for the Linux environment and do not have to match your Windows login. Remember the password: you will need it for `sudo`.

Note from Microsoft's docs: `wsl --install` only works if WSL is not already installed. If you see the WSL help text instead, run `wsl --list --online` to see available distros and then `wsl --install -d <DistroName>`. If it hangs at 0.0%, try `wsl --install --web-download -d <DistroName>`.

Now check everything:

```powershell
wsl --update
wsl --version
wsl --list --verbose
```

What each does:

| Command | Meaning |
|---|---|
| `wsl --update` | Updates the WSL package (including the Linux kernel) to the latest version. Docker needs 2.1.5 or newer. |
| `wsl --version` | Prints the WSL, kernel, WSLg, and Windows versions. |
| `wsl --list --verbose` (short: `wsl -l -v`) | Lists installed distros with their state and WSL version. |

Example output of `wsl -l -v`:

```
  NAME              STATE           VERSION
* Ubuntu            Running         2
```

The `*` marks the default distro. **VERSION must be 2.** If it says 1, convert it:

```powershell
wsl --set-version Ubuntu 2
wsl --set-default-version 2
```

(The first converts one distro; the second makes all future installs use WSL 2.)

### 2.5 Step 2: install Docker Desktop

**Option A: download the installer.** Go to Docker's website, download **Docker Desktop for Windows** (choose the AMD64 build for Intel/AMD PCs, or the ARM64 build for Snapdragon-based Copilot+ PCs), and run `Docker Desktop Installer.exe`.

**Option B: winget (Windows Package Manager).**

```powershell
winget install -e --id Docker.DockerDesktop
```

What it does: `winget` is Windows' built-in package manager; `-e` (`--exact`) requires the ID to match exactly; `--id Docker.DockerDesktop` names the package. Add `--silent` for no prompts.

**Installer choices:**

| Prompt | Recommendation |
|---|---|
| **Use WSL 2 instead of Hyper-V** (a checkbox on the Configuration page) | **Checked.** WSL 2 is the recommended backend. |
| Per-user vs all-users installation | Per Docker's docs, **per-user is the default and needs no administrator rights**. All-users needs admin and is required for the Hyper-V backend. |
| Add shortcut to desktop | Your choice. |

Click **OK** / **Install**, wait, then **Close**. Start **Docker Desktop** from the Start menu and **accept the subscription agreement**. You can **skip signing in** to a Docker account (signing in is optional for personal use, though signing in raises your Docker Hub pull limit; see [Chapter 1](01-what-is-docker.md#91-docker-hub)).

On first start Docker Desktop shows an onboarding tour (you can skip it) and then a window with **"Engine running"** in the bottom-left corner. Wait for that green indicator before using the CLI.

**Command-line install (useful for IT teams and scripts)**, straight from Docker's documentation:

```powershell
# Per-user install, no admin needed
Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList 'install', '--user'

# All users (needs admin)
Start-Process 'Docker Desktop Installer.exe' -Wait -ArgumentList 'install'
```

Useful installer flags (from Docker's documented list): `--quiet` (suppress output), `--accept-license` (accept the agreement during install), `--installation-dir=<path>`, `--backend=<wsl-2|hyper-v|windows|docker-vmm>` (default backend; `wsl-2` is the default), `--always-run-service` (start the Docker service at boot to avoid later admin prompts), and `--no-windows-containers`.

### 2.6 Step 3: connect Docker to your Ubuntu distro (WSL integration)

Docker Desktop runs its engine in its own WSL 2 distro. To use the `docker` command *from inside your Ubuntu terminal*, enable integration:

1. Open **Docker Desktop > Settings** (gear icon).
2. **General:** make sure **Use the WSL 2 based engine** is ticked (it is on by default if you chose WSL 2).
3. **Resources > WSL integration:** turn on **Enable integration with my default WSL distro** and the toggle next to **Ubuntu**.
4. Click **Apply & restart**.

Now in an Ubuntu terminal, `docker version` works. (If you see `The command 'docker' could not be found in this WSL 2 distro`, this step was skipped.)

### 2.7 Step 4: verify

Open **PowerShell** (or Windows Terminal) and run:

```powershell
docker version
docker run hello-world
```

You should see the "Hello from Docker!" message. [Section 7](#7-verifying-your-installation-with-hello-world) explains every line. Also try it from your Ubuntu terminal to confirm integration works.

**The `docker-users` group (only for some setups).** For an *all-users* install where the Windows administrator who installed Docker is a different account from the person using it, add the user to the local group `docker-users`:

```powershell
net localgroup docker-users <YourUserName> /add
```

(Run as Administrator, then sign out and in.) Per Docker's docs this is needed for Windows containers or Hyper-V VM management, **not** for Linux containers on the WSL 2 backend. And note the warning in the docs: membership gives access to the Docker daemon socket, which is equivalent to administrative access on the host, so add only people who need it.

### 2.8 Backends explained: WSL 2 vs Hyper-V vs Windows containers

| Backend | What it is | When to choose it |
|---|---|---|
| **WSL 2** (default, recommended) | Linux containers in the WSL 2 VM. Shares memory dynamically with Windows, starts fast, integrates with your Linux distro. | **Almost everyone.** |
| **Hyper-V** | Linux containers in a dedicated Hyper-V VM that Docker Desktop manages. Requires the Hyper-V and Containers Windows features and an all-users install. Pro/Enterprise/Education only. | Rare: if your IT policy forbids WSL. |
| **Windows containers mode** | Runs Windows-based containers on the Windows kernel. | Only if you must run Windows apps in containers. |
| **Docker VMM** | Listed by Docker's installer docs as a `--backend` value (`docker-vmm`). On the Mac side it is Docker's own virtual machine manager. | I did not verify its status on Windows; ignore it unless Docker's docs tell you otherwise. |

### 2.9 Alternative: Docker Engine directly inside WSL (no Docker Desktop)

If Docker Desktop's license does not fit your situation (see [Section 6](#6-docker-desktop-licensing-summary)), or you simply want the production-like setup, you can install **Docker Engine** (free) inside your WSL 2 Ubuntu exactly as on a Linux server. Only one extra step: enable **systemd** inside WSL so services like `dockerd` run.

In your Ubuntu terminal:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true
EOF
```

What this does: writes a small config file, `/etc/wsl.conf`, that tells WSL to start systemd (Linux's service manager) as the first process. Per Microsoft's docs, systemd requires WSL from the Microsoft Store/recent releases (`wsl --version` must work).

Then, in PowerShell:

```powershell
wsl --shutdown
```

(`wsl --shutdown` stops the WSL VM so the config is read at next start. Microsoft's docs call this the "8 second rule": the VM must fully stop before changes apply.) Reopen Ubuntu, confirm with `systemctl is-system-running`, and then follow [Section 4](#4-linux-ubuntu-installing-docker-engine) exactly as if this were a normal Ubuntu machine. This gives you a Linux-native Docker Engine with no Docker Desktop. You will not get the Docker Desktop GUI, Docker Scout integration, or Model Runner out of the box.

> **Do not run both** Docker Desktop's WSL integration and a separately installed Engine in the same distro without understanding which `docker` command you are talking to. Use `docker context ls` to see the active endpoint.

### 2.10 Limit WSL 2's resources with `.wslconfig`

By default, the WSL 2 VM can take up to **50% of your Windows RAM** and **all logical processors**, and it may hold on to memory after your containers stop. With the WSL 2 backend, Docker Desktop's resource sliders are mostly not used; the limits come from WSL itself, via a file named **`.wslconfig`** in your Windows user folder: `C:\Users\<YourName>\.wslconfig`.

The file does not exist by default. Create it (the notes below are verified against Microsoft's current WSL configuration page):

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
| `[wsl2]` | A section header (INI-file style). All keys under it affect the WSL 2 VM. |
| `memory` | Hard cap on RAM for the whole VM, shared by *all* containers. Sizes use units like `8GB` or `512MB`. |
| `processors` | How many logical CPU cores the VM may use. |
| `swap` | Disk-backed "overflow memory". `0` disables it. |
| `localhostForwarding` | Lets you browse `http://localhost:8080` on Windows to reach a container's published port. |
| `[experimental]` | Opt-in features. `autoMemoryReclaim` returns cached memory to Windows (Microsoft's doc lists `dropCache` as the default, `gradual` reclaims slowly). `sparseVhd=true` makes newly created virtual disks sparse (they can shrink). |

Optional networking key under `[wsl2]`: `networkingMode=mirrored` (Windows 11 22H2+) makes WSL share the Windows network interfaces, which often fixes VPN and "localhost" oddities. Default is `NAT`.

**Apply the changes:**

```powershell
wsl --shutdown
```

Then restart Docker Desktop. Check inside a container:

```powershell
docker info --format "CPUs={{.NCPU}} Mem={{.MemTotal}}"
```

What it does: `docker info` prints daemon information; `--format` takes a Go template to pick fields. `NCPU` is the number of CPUs the engine sees and `MemTotal` is memory in bytes. (On the machine used to write this guide, `docker info` reports 22 CPUs and 15.34 GiB of memory.)

**How much should you give it?** For AI development with local models, give WSL as much as you can spare while leaving Windows comfortable: for a 32 GB machine, `memory=20GB` to `24GB`. For a 16 GB machine, `memory=10GB`. Too little causes containers to be killed with exit code 137 (out of memory); too much starves Windows.

> **Tip from Microsoft's docs:** there is also a graphical **WSL Settings** app in the Start menu that edits these settings for you.

### 2.11 The #1 Windows performance tip: keep your project files inside WSL

This single tip prevents most "Docker is so slow on Windows" complaints. Docker's own WSL best-practices page says it directly.

WSL 2 has two filesystems:

| Location | Path as seen from WSL | Speed | File-change notifications to Linux containers |
|---|---|---|---|
| **Linux filesystem** (ext4 virtual disk) | `/home/you/projects/...` (from Windows: `\\wsl.localhost\Ubuntu\home\you\projects`) | **Fast** | Works |
| **Windows filesystem** (via a network-like bridge called DrvFs/9P) | `/mnt/c/Users/you/projects/...` | **Much slower** for many small files | Linux containers only receive file-change events for files in the Linux filesystem, per Docker's docs, so hot reload can break |

**The rule:** when you plan to bind-mount a project folder into a container (`-v ./myproject:/app`, or in Compose), keep that folder under `/home/you/...` in WSL, not `/mnt/c/...`.

```bash
# In your Ubuntu terminal
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/example/my-ai-app.git     # clone INSIDE WSL
cd my-ai-app
code .          # opens VS Code connected to WSL (needs the VS Code "WSL" extension)
docker compose up
```

Open the folder in Windows File Explorer with `explorer.exe .` (note the dot) from the WSL terminal.

**A quick way to feel the difference:**

```bash
# Linux filesystem
mkdir -p ~/speedtest && cd ~/speedtest
time (for i in $(seq 1 3000); do echo x > f$i; done)

# Windows filesystem
mkdir -p /mnt/c/Users/Public/speedtest && cd /mnt/c/Users/Public/speedtest
time (for i in $(seq 1 3000); do echo x > f$i; done)
```

What this does: `time ( ... )` measures how long it takes to create 3000 tiny files with a loop. Expect the `/mnt/c` run to be noticeably slower (often several times slower; exact numbers depend on your machine and antivirus, so I will not quote any). Clean up: `rm -rf ~/speedtest /mnt/c/Users/Public/speedtest`.

### 2.12 Windows gotchas

| Gotcha | Explanation and fix |
|---|---|
| **Line endings (CRLF vs LF)** | Windows ends lines with `\r\n`; Linux with `\n`. A shell script checked out with CRLF fails inside a container with errors like `exec ./entrypoint.sh: no such file or directory` or `/bin/sh^M: bad interpreter`. Fix: add a `.gitattributes` file with `* text=auto eol=lf` (or at least `*.sh text eol=lf`) and re-checkout; or set `git config --global core.autocrlf input`. |
| **Quoting differences** | PowerShell, Command Prompt, and Bash quote and continue lines differently. In Bash, `\` continues a line; in PowerShell it is the backtick `` ` ``; in cmd it is `^`. Commands in this guide are written for Bash. Chapter 3 shows PowerShell equivalents where it matters. |
| **Path syntax in `-v`** | In PowerShell: `-v ${PWD}:/app`. In cmd: `-v %cd%:/app`. In Bash/WSL: `-v "$(pwd)":/app`. |
| **Antivirus slowing bind mounts** | Exclude your WSL project folder from real-time scanning if your IT policy allows. |
| **VPN/corporate proxy breaks DNS or pulls** | See the error table in [Section 9](#9-common-install-errors-and-fixes). Try `networkingMode=mirrored`. |
| **The `vmmem` / `vmmemWSL` process using lots of RAM** | That is the WSL 2 VM. Cap it with `.wslconfig` ([2.10](#210-limit-wsl-2s-resources-with-wslconfig)) and run `wsl --shutdown` to release memory. |
| **Disk space** | WSL's virtual disk (`ext4.vhdx`) grows when you pull images and does not always shrink. `sparseVhd=true` helps; `docker system prune` (Chapter 3) frees space *inside*. |
| **GPU for AI** | With an NVIDIA GPU, install the normal **Windows** NVIDIA driver (not a Linux driver inside WSL); Docker Desktop with WSL 2 then supports `--gpus all`. Full setup in [Chapter 8](08-docker-for-ai-and-ml.md). |

---

## 3. macOS: Apple Silicon and Intel

### 3.1 Requirements (verified against Docker's docs, October 2026)

| Item | Detail |
|---|---|
| **macOS version** | Docker supports the **current and two previous major macOS releases** |
| **Chip** | Two separate downloads: **Apple silicon** (M1, M2, M3, M4...) and **Intel** |
| **RAM** | At least 4 GB (more for AI work) |
| **Rosetta 2** | Recommended on Apple silicon but, per Docker, no longer strictly required (some optional command-line tools that use Darwin/AMD64 still need it). Install with `softwareupdate --install-rosetta` |

### 3.2 Install

**Option A: the DMG.**

1. Download the correct **Docker Desktop** DMG for your chip. (Not sure which? Apple menu > **About This Mac**: "Chip: Apple M..." means Apple silicon; "Processor: Intel..." means Intel.)
2. Open `Docker.dmg` and drag **Docker** into **Applications**.
3. Launch Docker from Applications and **accept the subscription agreement**. Per Docker's docs, Docker Desktop starts with a default configuration that needs no password or additional setup.

**Option B: Homebrew.**

```bash
brew install --cask docker-desktop
```

What it does: `brew` is the macOS package manager; `--cask` means "install a macOS application" (not a command-line formula). Homebrew's listing says the cask token is `docker-desktop`, renamed from the former token `docker`. (So on an old Homebrew, run `brew update` first.) Note: `brew install docker` (no `--cask`) installs only the **Docker CLI**, not Docker Desktop, and you will have no engine.

Verify as in [Section 7](#7-verifying-your-installation-with-hello-world).

**Resources:** Docker Desktop > **Settings > Resources** has sliders for **CPUs**, **Memory**, **Swap**, and **Disk image size**. Give AI workloads more memory than the default.

### 3.3 Apple silicon vs Intel: the architecture issue

**CPU architecture** is the "language" a processor speaks. The two you will meet:

| Name(s) | Where you see it | Machines |
|---|---|---|
| **amd64** (also `x86_64`, "x64") | Most servers and cloud VMs, Intel/AMD PCs, Intel Macs | The default for most cloud servers |
| **arm64** (also `aarch64`, "ARM64") | Apple silicon Macs, AWS Graviton, Raspberry Pi 4/5, many phones | Increasingly common in cloud |

A compiled program (and therefore a container image's executables) is built for **one** architecture. An `amd64` binary cannot natively run on an `arm64` CPU, and vice versa.

Here is the Mac trap: **you build an image on your Apple Silicon Mac. It is `arm64`. You deploy it to a typical cloud server, which is `amd64`. It fails with `exec format error`.** Learn this now and save yourself a day.

You can see the architecture of your engine:

```bash
docker info --format '{{.Architecture}}'      # e.g. aarch64 on Apple silicon, x86_64 on Intel/AMD
docker run --rm alpine uname -m               # same, from inside a container
```

### 3.4 Multi-architecture images

Many popular images (nginx, python, postgres, ubuntu...) are **multi-architecture** ("multi-arch"): one image name like `python:3.13-slim` is actually a **manifest list**, a small index that says "for amd64 use this set of layers, for arm64 use that set." When you pull, Docker picks the one matching your engine's architecture automatically. This is why `docker pull python` just works on both an M-series Mac and an Intel server.

Inspect which platforms an image supports:

```bash
docker buildx imagetools inspect python:3.13-slim
# or the older style:
docker manifest inspect python:3.13-slim
```

What it does: asks the registry for the image's manifest list and prints the platform entries (e.g., `linux/amd64`, `linux/arm64/v8`, `linux/arm/v7`...). The exact list varies per image and over time, so I will not paste one.

If an image only exists for amd64 and you are on arm64, Docker emits a warning like:

```
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
```

(and still runs it under **emulation**, slowly, if emulation is available).

### 3.5 The `--platform` flag

`--platform` tells Docker which architecture to pull, build, or run.

```bash
docker run --rm --platform linux/amd64 alpine uname -m
```

Expected output on an Apple silicon Mac: `x86_64`: even though your Mac is ARM, the container ran an amd64 image under emulation. Without `--platform`, you would see `aarch64`.

| Platform string | Meaning |
|---|---|
| `linux/amd64` | 64-bit Intel/AMD Linux |
| `linux/arm64` (also `linux/arm64/v8`) | 64-bit ARM Linux (Apple silicon, Graviton) |
| `linux/arm/v7` | 32-bit ARM (older Raspberry Pi OS 32-bit) |

**Emulation.** Docker Desktop can run foreign-architecture containers using emulation (QEMU, and on Apple silicon Rosetta for amd64 where enabled). Emulation works but can be several times slower, and some software (certain JVM, database, or JIT-heavy programs) can crash under it. It is for convenience and testing, not production.

### 3.6 Building images that run on servers

If your servers are amd64 and your Mac is arm64, build for the target:

```bash
# Build for amd64 only (what the server needs)
docker buildx build --platform linux/amd64 -t myapp:1.0 .

# Build for BOTH architectures and push in one go (creates a multi-arch manifest list)
docker buildx build --platform linux/amd64,linux/arm64 -t ghcr.io/you/myapp:1.0 --push .
```

What it does: `docker buildx build` is the modern build command (BuildKit, Docker's current builder engine). `--platform` lists targets; `-t` names/tags the result; `--push` uploads straight to the registry (needed for multi-platform results, because a single local image store traditionally can hold only one platform; with the containerd image store you can sometimes load multi-platform images locally). The trailing `.` is the build context (current folder). Chapter 4 covers building in depth.

### 3.7 Mac-specific notes

| Topic | Note |
|---|---|
| **VM technology** | Docker Desktop on Mac runs its Linux VM using a virtualization layer chosen in Settings (Apple's virtualization framework, or Docker's own VMM). Docker's installer docs mention the Docker VMM engine as a Mac install option; I did not verify the current default or the performance differences, so leave the default unless Docker's docs or an issue you are hitting say otherwise. |
| **File sharing** | Bind mounts cross the Mac/VM boundary and are slower than container-internal storage. Prefer **named volumes** (Chapter 5) for heavy data like databases and `node_modules`. |
| **GPU / AI** | Linux containers on a Mac cannot use the Mac's GPU directly (no NVIDIA, no Metal passthrough into containers). For fast local LLM inference on a Mac, look at **Docker Model Runner**, which Docker designed to run inference using the host's resources, or run Ollama natively (outside Docker). See [Chapter 8](08-docker-for-ai-and-ml.md). I did not independently verify the GPU details of Model Runner on macOS. Check Docker's current docs. |
| **Alternatives** | **Colima** (`brew install colima docker`), **OrbStack**, **Rancher Desktop**, **Podman Desktop**: all can run containers on a Mac without Docker Desktop. |
| **Apple's own tooling** | Apple also introduced its own open-source container tooling for macOS, which is a separate, newer ecosystem. This guide does not use it; I did not verify its current maturity. |

---

## 4. Linux (Ubuntu): installing Docker Engine

This is **the production-grade installation**, and everything here is verified against Docker's official "Install Docker Engine on Ubuntu" page (October 2026).

### 4.1 Supported Ubuntu versions and architectures

Per Docker's page, Docker Engine is installable on the 64-bit versions of:

- Ubuntu **26.10** ("Stonking")
- Ubuntu **26.04 LTS** ("Resolute")
- Ubuntu **24.04 LTS** ("Noble")
- Ubuntu **22.04 LTS** ("Jammy")

Architectures: x86_64/amd64, armhf, arm64, s390x, ppc64le. **Derivatives such as Linux Mint are not officially supported** (the repository setup below uses the Ubuntu codename, which can fail on derivatives). For production, prefer an **LTS** (Long-Term Support) release: it receives security updates for years.

> **What is LTS?** Ubuntu publishes a new LTS release every two years (e.g., 22.04, 24.04, 26.04) with five years of free security updates. Non-LTS releases (like 26.10) get only nine months of support. Servers should run LTS.

Check your version:

```bash
lsb_release -a        # or: cat /etc/os-release
uname -m              # architecture: x86_64 or aarch64
```

### 4.2 Step 1: remove conflicting unofficial packages

Ubuntu's own repositories carry unofficial Docker packages (`docker.io`, etc.). They conflict with Docker's official ones. Per Docker's docs the conflicting packages are: `docker.io`, `docker-compose`, `docker-compose-v2`, `docker-doc`, `podman-docker`, plus any separately installed `containerd` or `runc`.

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

What this does: loops over each name and removes it if installed. `apt-get remove` uninstalls a package; `-y` answers "yes" automatically. If apt reports "Package 'X' is not installed," that is fine. **Your existing images, containers, and volumes in `/var/lib/docker/` are not deleted by this**; only the old packages are.

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

What each part means:

| Piece | Meaning |
|---|---|
| `sudo` | Run as administrator ("superuser do"). |
| `apt` | Ubuntu's package manager (it downloads and installs software). |
| `ca-certificates`, `curl` | Certificates for HTTPS, and `curl`, a tool for downloading files. |
| `install -m 0755 -d /etc/apt/keyrings` | Creates a directory (`-d`) with permissions `0755` (owner can write; everyone can read). It stores trusted signing keys. |
| `curl -fsSL URL -o FILE` | Downloads quietly (`-s`), fails on HTTP errors (`-f`), shows errors (`-S`), follows redirects (`-L`), saving (`-o`) to a file. |
| **GPG key** (`docker.asc`) | Docker's public signing key. `apt` uses it to verify that packages really come from Docker and were not tampered with. |
| `chmod a+r` | Make the key readable by all (**a**ll users: **+r**ead). |
| `tee /etc/apt/sources.list.d/docker.sources` | Writes the here-document to a file with admin rights (`sudo tee` is the standard trick, because `sudo cat > file` would not elevate the redirect). |
| `Types: deb` / `URIs` / `Suites` / `Components` / `Signed-By` | The **deb822** format for an apt repository: `Suites` is your Ubuntu codename (e.g., `noble`), taken from `/etc/os-release`; `Components: stable` selects the stable channel; `Signed-By` points to the key. |

Verify the file:

```bash
cat /etc/apt/sources.list.d/docker.sources
```

Expected (for Ubuntu 24.04):

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
| `docker-ce` | **Docker Engine, Community Edition**: the `dockerd` daemon. |
| `docker-ce-cli` | The `docker` command-line client. |
| `containerd.io` | The containerd runtime (with runc). |
| `docker-buildx-plugin` | `docker buildx`: the modern builder (BuildKit front end). |
| `docker-compose-plugin` | `docker compose` (Compose v2 and later, installed as a CLI plugin). |

Docker's docs also describe installing a specific version (`apt list --all-versions docker-ce`, then `sudo apt install docker-ce=<version> docker-ce-cli=<version> ...`). Use that on servers where you must control versions. The `docker-ce-rootless-extras` package (used for rootless mode, below) is a separate optional package.

The Docker service starts automatically on Ubuntu/Debian after installation. Check it:

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

`enabled` means it will start at boot; `active (running)` means it is up now. Press `q` to leave the pager. If it is not running, start it with `sudo systemctl start docker`.

Test it (with `sudo`, since your user is not in the `docker` group yet):

```bash
sudo docker run hello-world
```

### 4.5 Step 4 (post-install): run Docker without `sudo`

By default the Docker socket is owned by `root` and the group `docker`. To run `docker` without typing `sudo` every time, add yourself to the `docker` group:

```bash
sudo groupadd docker 2>/dev/null || true     # the group usually already exists
sudo usermod -aG docker $USER                # add yourself to it
newgrp docker                                # activate in the current shell (or log out and back in)
docker run hello-world                       # now works without sudo
```

| Piece | Meaning |
|---|---|
| `groupadd docker` | Creates the group (ignore the error if it exists). |
| `usermod -aG docker $USER` | Modifies (`usermod`) the user; `-a` **a**ppend, `-G docker` to the supplementary **G**roup `docker`. **Always use `-a`**, or you will remove yourself from your other groups. `$USER` expands to your username. |
| `newgrp docker` | Starts a shell with the new group applied immediately. Otherwise, log out and back in. |

> **SECURITY WARNING (read this).** Per Docker's post-install page, the `docker` group **grants root-level privileges** to its members. Anyone who can run `docker` can start a container that mounts the host's entire filesystem and read or write anything as root, e.g., `docker run -v /:/host --rm -it alpine chroot /host`. On your own laptop that is usually acceptable. On shared or production servers, add only trusted administrators, or use **rootless mode** (Section 4.8) or `sudo`.

### 4.6 Step 5: make sure Docker starts on boot (systemd)

**systemd** is the service manager on virtually all modern Linux distributions: it starts, stops, and supervises background services. Docker is a systemd service.

```bash
sudo systemctl enable docker.service containerd.service   # start at boot
sudo systemctl disable docker.service containerd.service  # (the opposite)
```

Debian and Ubuntu enable it by default; on Fedora/RHEL you must run `enable` yourself.

Everyday service commands:

| Command | What it does |
|---|---|
| `sudo systemctl start docker` | Start the daemon now |
| `sudo systemctl stop docker` | Stop it (also stop `docker.socket` if you want it to stay off, because socket activation restarts it on demand) |
| `sudo systemctl restart docker` | Restart (running containers stop unless `live-restore` is on, see below) |
| `sudo systemctl status docker` | Show current state and recent log lines |
| `journalctl -u docker.service -f` | Follow the daemon's log (`-u` unit, `-f` follow). **First place to look when Docker misbehaves.** |
| `sudo systemctl daemon-reload` | After editing unit files |

### 4.7 Step 6: configure the daemon (`/etc/docker/daemon.json`) and log rotation

The daemon reads an optional JSON file, `/etc/docker/daemon.json`. (If you already have one, edit it by hand instead of running the `tee` command below, which **overwrites** the file.) The most valuable first setting is **log rotation**. Docker's post-install page warns that the default `json-file` logging driver keeps container logs **without limit** and can fill your disk. Note: Docker Desktop's logging driver shows as `local` in the test machine's `docker info`, but on a plain Linux Engine the default is `json-file`.

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
| `log-driver: local` | Use the `local` logging driver, which stores logs in an efficient format and **rotates by default**. (Alternative: keep `json-file` and set `max-size`/`max-file`.) |
| `max-size: 10m` | Rotate when a log file hits 10 MB. |
| `max-file: 3` | Keep at most 3 rotated files per container (so about 30 MB per container). |
| `live-restore: true` | Keep containers **running** while the Docker daemon restarts or upgrades. (Not compatible with Swarm mode.) |

Existing containers keep their old logging settings; the new defaults apply to containers you create afterwards. A bad `daemon.json` (invalid JSON) prevents the daemon from starting. If that happens, read `journalctl -u docker.service`.

### 4.8 Rootless mode (introduction)

**Rootless mode** runs the Docker daemon *and* your containers as a **normal, unprivileged user**, with no root involvement. If a container escapes, the attacker gets only that user's powers, not root. It uses **user namespaces** (Chapter 1) to map "root inside" to your unprivileged UID outside.

Per Docker's rootless documentation:

**Prerequisites**

- `newuidmap` and `newgidmap` installed (the `uidmap` package: `sudo apt install -y uidmap`).
- Your user must have a range of at least **65,536** subordinate UIDs/GIDs in `/etc/subuid` and `/etc/subgid` (on Ubuntu, creating a normal user normally sets this up; check with `grep $USER /etc/subuid /etc/subgid`).
- The `docker-ce-rootless-extras` package (or Docker's rootless install script).

**Install**

```bash
sudo apt install -y uidmap docker-ce-rootless-extras
# If the system-wide daemon is running, Docker recommends disabling it first:
sudo systemctl disable --now docker.service docker.socket
# As your normal user (NOT root, no sudo):
dockerd-rootless-setuptool.sh install
```

The tool sets up a **systemd user service** at `~/.config/systemd/user/docker.service` and prints instructions. Follow them: add the printed `export PATH=...` line, and set `DOCKER_HOST` (for UID 1000 it is `unix:///run/user/1000/docker.sock`). Manage it with:

```bash
systemctl --user start docker.service        # start (note --user)
systemctl --user restart docker.service
systemctl --user status docker.service
sudo loginctl enable-linger $USER            # keep it running when you log out / start at boot
docker info | grep -i rootless               # should show "rootless" in security options and context
```

| Piece | Meaning |
|---|---|
| `dockerd-rootless-setuptool.sh install` | Docker's helper that configures the rootless daemon for your user. |
| `systemctl --user` | Controls *your* per-user systemd services instead of system-wide ones. |
| `loginctl enable-linger` | Lets your user services keep running without an active login session, and start at boot. |
| `DOCKER_HOST` | Environment variable telling the CLI which daemon socket to talk to. |

**Trade-offs.** Rootless mode has limitations (Docker's page lists a Troubleshooting section; I could not verify the complete limitation list from the page I fetched, so confirm in the docs). Commonly known ones: binding to ports below 1024 needs extra configuration, some networking features and storage drivers are limited, and performance of the network layer can be lower. It is excellent for dev machines and shared hosts. We cover rootless in security context in [Chapter 10](10-security.md).

### 4.9 Firewall caveat (important)

Per Docker's Ubuntu page: when you publish a container port (`-p 8080:80`), Docker adds its own `iptables` rules that **bypass** `ufw` and `firewalld` rules. So `sudo ufw deny 8080` does **not** stop external access to a published port. Docker only works with `iptables-nft` and `iptables-legacy`; rules created directly with `nft` are not supported on a system with Docker installed. If you must restrict published ports, add rules to the `DOCKER-USER` chain, or bind the published port to localhost only (`-p 127.0.0.1:8080:80`). We revisit this in Chapters 5 and 10.

### 4.10 Other Linux distributions

The pattern is the same: add Docker's repository, install `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`.

| Distribution | Docs page to follow | Notes |
|---|---|---|
| **Debian** | "Install Docker Engine on Debian" | Nearly identical to Ubuntu, with `download.docker.com/linux/debian`. |
| **Fedora** | "Install Docker Engine on Fedora" | Uses `dnf`; the service is **not** enabled automatically: `sudo systemctl enable --now docker`. |
| **RHEL / CentOS Stream / Rocky / Alma** | "Install Docker Engine on RHEL/CentOS" | Uses `dnf`; podman is the native tool on RHEL. |
| **Arch** | Package `docker` from the official repos | Community maintained. |
| **Raspberry Pi OS / ARM** | The Debian instructions | Note that Docker Engine 29 stopped providing 32-bit Raspbian packages and Debian armhf packages target ARMv7 (per the release notes). |

**The "convenience script"** (`curl -fsSL https://get.docker.com | sh`) installs Docker in one command. It is fine for throwaway dev VMs; Docker itself documents it as not intended for production. Prefer the repository method above for anything long-lived, because you control versions and updates.

### 4.11 Docker Desktop for Linux (optional)

You *can* install **Docker Desktop for Linux** (`.deb`/`.rpm`) to get the GUI, extensions, and Model Runner on a Linux workstation. Requirements per Docker: Ubuntu, Debian, RHEL, or Fedora (x86-64), KVM virtualization, QEMU 5.2+, systemd, 4 GB RAM, and a GNOME, KDE, or MATE desktop. It runs a VM, with **its own separate** images and containers (Engine's are not visible in Desktop), and creates a `desktop-linux` Docker context. The same license terms as other Desktop editions apply. On servers and most Linux workstations, plain Docker Engine is the better choice.

---

## 5. Docker Engine on servers

On a production host (a cloud VM like AWS EC2 / Azure VM / GCE, or bare metal), you install **Docker Engine** only: no Docker Desktop. A sensible checklist:

| Step | Why |
|---|---|
| Use a **Linux LTS** distribution (Ubuntu LTS, Debian stable, RHEL-family) | Long security support |
| Install from **Docker's official repo**; **pin versions** and update deliberately | Avoid surprise upgrades |
| Configure **log rotation** in `daemon.json` ([4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)) | Prevent full disks |
| Consider **`live-restore: true`** | Keep containers up through daemon restarts |
| **Do not add everyday users to the `docker` group**; use `sudo` or rootless | Group is root-equivalent |
| **Never expose the Docker API on TCP without TLS** (port 2375 unencrypted is a classic way to get hacked) | Anyone who reaches it owns the machine |
| Control access with a **firewall / cloud security group**; remember Docker bypasses `ufw` for published ports | See 4.9 |
| Enable automatic **security updates** for the OS | Kernel and library patches |
| Run containers as **non-root users**, set resource limits and restart policies | Chapter 3 and Chapter 10 |
| Monitor disk (`docker system df`) and prune safely | Images and build cache accumulate |
| Prefer pulling from a **private registry** that you control | Reliability, rate limits, security |

**Managing a remote server from your laptop** without logging in each time: use a Docker **context** over SSH. This keeps the daemon private and uses SSH for authentication (no open Docker port):

```bash
docker context create myserver --docker "host=ssh://deploy@203.0.113.10"
docker context use myserver
docker ps                    # now lists containers on the REMOTE server
docker context use default   # switch back to local
```

What it does: `docker context create NAME --docker "host=ssh://USER@HOST"` stores a named endpoint; the local `docker` CLI then tunnels API calls over SSH to the remote `dockerd`. (The IP address above is from a reserved documentation range; replace it with yours. The remote user must be able to run `docker`.) Contexts are covered fully in [Chapter 3](03-core-cli-commands.md) and production deployments in [Chapter 11](11-production-and-enterprise.md).

---

## 6. Docker Desktop licensing summary

> **Not legal advice.** The figures below come from Docker's pricing FAQ and Docker Desktop install pages fetched in October 2026. Prices and terms change. Your company's legal or procurement team has the final word. Read Docker's current Subscription Service Agreement before relying on this.

### 6.1 What is free?

**Docker Engine** (the open-source runtime that runs on Linux) is **free**, with no license restrictions on the software itself. Docker's FAQ says open-source software like Docker Engine is accessible to all and is supported by the Moby project maintainers and community.

**Docker Desktop** (the app for Windows/macOS/Linux) is free **only** for:

- **Personal use**,
- **Education**,
- **Non-commercial open-source projects**, and
- **Small businesses**: fewer than **250 employees** *and* less than **US $10 million** in annual revenue.

The quoted clause from Docker's terms: use of Docker Desktop without a paid subscription is restricted to "(i) your use for a non-commercial open source project and/or (ii) use in a commercial undertaking with fewer than 250 employees and less than US $10,000,000 (or equivalent local currency) in annual revenue. Government Entities shall not use Docker Desktop ... without purchasing a Subscription."

### 6.2 When do you need to pay?

If your organization has **250 or more employees OR $10 million or more in annual revenue** (it is an **OR**, so exceeding either threshold triggers it), then professional use of Docker Desktop requires a **paid subscription** (Pro, Team, or Business) for each user. Also:

- **Government entities** need a paid subscription regardless of size (per the quoted terms).
- Per secondary sources that quote Docker's FAQ (I did not verify this line on Docker's own page), a small company that is a **subsidiary** of a larger one can fall under the larger company's headcount. Ask your legal team.
- Licensing is **per user**, not per machine, according to third-party summaries; confirm with Docker.
- **Using only the command line does not exempt you.** The license attaches to Docker Desktop, the product, not the GUI.

### 6.3 Plans and prices (from Docker's FAQ, October 2026)

| Plan | Price per user per month (annual billing) | Month-to-month | Intended for |
|---|---|---|---|
| **Personal** | $0 | $0 | Individuals, education, open source, small companies |
| **Pro** | $9 | $11 | Individual professionals needing more features |
| **Team** | $15 | $16 | Small teams (minimum 1 user per the FAQ) |
| **Business** | $24 | $24 | Larger organizations (SSO, security and governance controls) |

Docker's FAQ said these prices took effect on December 10, 2024 for new customers and at renewal for existing customers. Check Docker's pricing page for what is current today.

### 6.4 If you cannot (or do not want to) pay

| Option | Notes |
|---|---|
| **Docker Engine on Linux** | Free for everyone. Use a Linux workstation, VM, or server. |
| **Docker Engine inside WSL 2** (Windows) | Free: [Section 2.9](#29-alternative-docker-engine-directly-inside-wsl-no-docker-desktop). |
| **Podman / Podman Desktop** | Free and open source; Docker-compatible CLI. |
| **Rancher Desktop** | Free, open source, runs containerd or dockerd. |
| **Colima** (macOS/Linux) | Free CLI tool. |
| **OrbStack** (macOS) | Has its own commercial/personal-use terms. Check them. |

Docker Hub (the registry) has its own, separate account tiers and pull limits ([Chapter 1](01-what-is-docker.md#91-docker-hub)). You can use Docker Hub's public images without Docker Desktop.

---

## 7. Verifying your installation with hello-world

### 7.1 Two quick checks first

```bash
docker --version
docker compose version
```

Example output (from the machine used to write this guide):

```
Docker version 29.9.2, build a82e528
Docker Compose version v5.6.1
```

- `docker --version` prints just the client version and build.
- `docker compose version` confirms the **Compose plugin** is installed. If you get `docker: 'compose' is not a docker command`, the plugin is missing (install `docker-compose-plugin` on Linux, or reinstall Docker Desktop).

Now the full client-and-server check:

```bash
docker version
```

If you see a `Client:` block **and** a `Server:` block, the CLI reached the daemon. If you see only `Client:` followed by an error such as `Cannot connect to the Docker daemon`, the engine is not running ([Section 9](#9-common-install-errors-and-fixes)).

```bash
docker info
```

`docker info` prints daemon details: number of containers and images, storage driver, logging driver, cgroup version, kernel version, OS, CPUs, memory, runtimes, and so on. On the machine used here, for example, a few relevant lines are:

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

(Reading it: `OSType: linux` and a `...microsoft-standard-WSL2` kernel confirm this is a Linux engine in a WSL 2 VM on Windows. `Runtimes` lists `runc` as the default OCI runtime and an `nvidia` runtime that the GPU toolkit added.)

### 7.2 `docker run hello-world`

```bash
docker run hello-world
```

**What the command means:** `docker run` creates and starts a new container; `hello-world` is the image name (a tiny official image whose only job is to print a message and exit; it contains just one small executable).

**Typical output on the first run** (digest and layer IDs will differ on your machine, and Docker's output format sometimes changes slightly between versions):

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

| Output line | What it means |
|---|---|
| `Unable to find image 'hello-world:latest' locally` | Docker looked in the local image store first and did not find an image named `hello-world` with the tag `latest` (the default tag, because you did not specify one). Not an error; it is the normal "I will download it" message. On the second run you will not see this line. |
| `latest: Pulling from library/hello-world` | The daemon is downloading from **Docker Hub**. `library/` is the namespace for Docker Official Images; `hello-world` is the repository; `latest` is the tag ([Chapter 1](01-what-is-docker.md#66-the-anatomy-of-an-image-name)). |
| `e6590344b1a5: Pull complete` | This is one **layer**, identified by a short hash prefix. `Pull complete` means that layer finished downloading and was unpacked. A bigger image shows many such lines, with progress bars while downloading. `hello-world` has one tiny layer. (The hash shown here is an example; yours may differ because the image is rebuilt over time.) |
| `Digest: sha256:...` | The immutable **content hash** of the exact image you got. You could pin it with `hello-world@sha256:...` to always get these exact bytes. |
| `Status: Downloaded newer image for hello-world:latest` | The pull succeeded and a new image is now stored locally. If you already had the newest, you would see `Status: Image is up to date for hello-world:latest`. |
| *(blank line)* | End of the pull messages; below this is the **output of the container itself**. |
| `Hello from Docker!` and the next line | Printed by the program *inside the container*. This message appearing proves the whole chain worked: CLI, daemon, registry download, container creation, program execution, and output streaming back. |
| `1. The Docker client contacted the Docker daemon.` | Your `docker` CLI sent a REST API request to `dockerd` over the socket/named pipe ([Chapter 1](01-what-is-docker.md#8-docker-architecture-what-runs-when-you-type-docker-run)). |
| `2. The Docker daemon pulled the "hello-world" image from the Docker Hub. (amd64)` | The daemon fetched the image from the registry. `(amd64)` is the **CPU architecture** it chose: on an Apple silicon Mac or ARM server this says `(arm64v8)` instead. It matches your engine automatically via the multi-arch manifest list. |
| `3. The Docker daemon created a new container from that image which runs the executable that produces the output you are currently reading.` | The daemon made a container from the image and started its single program, the "hello" executable. |
| `4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.` | The container's standard output (stdout) was forwarded back through the daemon to the CLI and onto your screen. |
| `To try something more ambitious, you can run an Ubuntu container with: $ docker run -it ubuntu bash` | A suggestion: `-i` keeps input open, `-t` allocates a terminal, giving you an interactive Ubuntu shell inside a container. We explain `-it` thoroughly in [Chapter 3](03-core-cli-commands.md). |
| `Share images, automate workflows, and more with a free Docker ID: https://hub.docker.com/` | Marketing: a Docker account lets you push images to Docker Hub. |
| `For more examples and ideas, visit: https://docs.docker.com/get-started/` | Link to the official getting-started guide. |

### 7.4 What happened afterwards?

The program printed its text and exited. The container is now in the **Exited** state (not running, but not deleted). Look:

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
| `CONTAINER ID` | A short form of the container's unique ID. |
| `IMAGE` | The image it was created from. |
| `COMMAND` | What ran inside: `/hello`, the one executable in the image. |
| `CREATED` | When it was created. |
| `STATUS` | `Exited (0)` means it finished with **exit code 0 = success**. A non-zero code means failure ([Chapter 3](03-core-cli-commands.md)). |
| `PORTS` | Published ports (none). |
| `NAMES` | A random human-friendly name Docker generated (adjective + scientist/hacker surname; yours will differ). You can choose your own with `--name`. |

`-a` means "all": without it, `docker ps` shows only running containers. Clean up with:

```bash
docker rm vibrant_turing        # use YOUR container's name or ID from the list
docker rmi hello-world          # optional: remove the image too (rmi = remove image)
```

### 7.5 A second, more meaningful check

```bash
docker run --rm -it alpine sh
```

Inside the shell try `cat /etc/os-release`, `uname -a`, `ls`, then `exit`. `--rm` automatically deletes the container on exit; `-it` gives you an interactive terminal. If this works, your Docker is fully functional, and you are ready for [Chapter 3](03-core-cli-commands.md).

---

## 8. Uninstall and cleanup

> **Warning:** The steps that remove `/var/lib/docker`, or "Clean/Purge data" in Docker Desktop, **permanently delete all your images, containers, volumes (including database data), and networks.** Only do this if you are sure.

### 8.1 First, optionally clean up from inside Docker

Preview what is taking space:

```bash
docker system df
```

Remove everything stopped/unused (carefully; see the warnings in [Chapter 3](03-core-cli-commands.md#13-disk-usage-and-cleanup-system-df-and-prune)):

```bash
docker system prune -a --volumes
```

### 8.2 Windows (Docker Desktop)

1. **Docker Desktop > Troubleshoot** (bug icon) > **Clean / Purge data** (deletes all Docker data), then
2. Quit Docker Desktop (tray icon > **Quit Docker Desktop**).
3. **Windows Settings > Apps > Installed apps > Docker Desktop > Uninstall.** (Or `winget uninstall -e --id Docker.DockerDesktop`.)
4. Optional leftovers: delete `%USERPROFILE%\.docker` (your CLI config, credentials, and contexts), `%APPDATA%\Docker`, `%APPDATA%\Docker Desktop`, and `%LOCALAPPDATA%\Docker`.
5. Docker's WSL data: the installer normally removes the `docker-desktop` WSL distros. Check with `wsl -l -v`; if `docker-desktop` remains, remove it with `wsl --unregister docker-desktop`. (**`wsl --unregister` deletes that distro's data permanently.** Never unregister your own Ubuntu unless you want to delete it and everything in it.)
6. Optional: remove `.wslconfig` if you created one purely for Docker.

### 8.3 macOS

1. Docker Desktop > **Troubleshoot > Clean / Purge data** (optional).
2. Quit Docker, then drag **Docker.app** from **Applications** to the Trash. (Or `brew uninstall --cask docker-desktop`.)
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

`apt purge` removes the packages **and** their system configuration files. Files you created by hand (like `/etc/docker/daemon.json`) must be removed manually: `sudo rm -rf /etc/docker`. Remove your user from the group if desired: `sudo gpasswd -d $USER docker`. If you used rootless mode: `dockerd-rootless-setuptool.sh uninstall` and `rootlesskit rm -rf ~/.local/share/docker`.

---

## 9. Common install errors and fixes

| Symptom / exact error text | Cause | Fix |
|---|---|---|
| `error during connect: ... open //./pipe/dockerDesktopLinuxEngine: The system cannot find the file specified.` (Windows) | The CLI is fine but the **engine is not running**. Docker Desktop is closed or still starting. | Start Docker Desktop and wait for "Engine running". Check `docker context ls`: should show `desktop-linux` active. |
| `Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?` (Linux) | `dockerd` is not running, or you are using the wrong socket/context | `sudo systemctl start docker`; check `sudo systemctl status docker` and `journalctl -u docker.service`. If you use rootless or Desktop, check `echo $DOCKER_HOST` and `docker context ls`. |
| `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` | Your user is not in the `docker` group | `sudo usermod -aG docker $USER`, then **log out and back in** (or `newgrp docker`). See [4.5](#45-step-4-post-install-run-docker-without-sudo) and its security warning. |
| `WSL 2 installation is incomplete` / "WSL needs updating" | Old or missing WSL kernel | PowerShell (admin): `wsl --update`, then `wsl --shutdown` and restart Docker Desktop. |
| `Docker Desktop requires the Server service to be enabled` | Windows "Server" (`LanmanServer`) service disabled | `services.msc` > **Server** > Startup type **Automatic** > Start. (Docker's requirements list this explicitly.) |
| `Hardware assisted virtualization and data execution protection must be enabled in the BIOS` / `Virtualization support not detected` | Virtualization is disabled in BIOS/UEFI | Enable Intel VT-x / AMD-V (SVM) in firmware; see [2.3](#23-step-0-check-that-virtualization-is-enabled). |
| `Docker Desktop is unable to start` / stuck at "Starting the Docker Engine..." | Corrupted state, WSL problem, or conflicting software | `wsl --shutdown`, restart Docker Desktop; update WSL; check Troubleshoot > **Reset to factory defaults** (**deletes your containers/images**), and Windows Event Viewer. |
| `The command 'docker' could not be found in this WSL 2 distro. We recommend to activate the WSL integration in Docker Desktop settings.` | WSL integration disabled | Docker Desktop > Settings > Resources > **WSL integration**: enable your distro, **Apply & restart** ([2.6](#26-step-3-connect-docker-to-your-ubuntu-distro-wsl-integration)). |
| `no matching manifest for linux/arm64/v8 in the manifest list entries` (or `for windows/amd64`) | The image does not publish your architecture/OS. On Windows `windows/amd64` = Docker is in **Windows containers mode**. | Add `--platform linux/amd64` (emulated), find an arm64-capable image, or switch Docker Desktop to Linux containers. |
| `exec format error` or `exec /usr/local/bin/entrypoint.sh: exec format error` | Architecture mismatch (arm64 image on amd64 or vice versa), or a script missing its `#!` line | Build with `--platform`; confirm the first line of the script is `#!/bin/sh` or `#!/usr/bin/env bash`. |
| `exec ./start.sh: no such file or directory` though the file exists | Windows CRLF line endings in a script | Convert to LF; use `.gitattributes` ([2.12](#212-windows-gotchas)). |
| `toomanyrequests: You have reached your pull rate limit` | Docker Hub unauthenticated/Personal limit ([Chapter 1](01-what-is-docker.md#91-docker-hub)) | `docker login`, wait up to six hours, or use a mirror/cache. |
| `x509: certificate signed by unknown authority` when pulling | A corporate TLS-inspecting proxy re-signs HTTPS traffic with your company's certificate, which Docker does not trust | Install your company root CA certificate in the OS/Docker trust store (Docker Desktop: add it to the OS keychain/certificate store; on Linux, copy it to `/etc/docker/certs.d/<registry>/ca.crt` or the system CA store and restart Docker). Ask IT for the certificate. Do **not** disable TLS verification. |
| Pulls hang or `Temporary failure in name resolution` / DNS failures behind a VPN | VPN changes DNS/routing | Try `networkingMode=mirrored` in `.wslconfig` (Windows 11), restart WSL, or set DNS servers in `daemon.json`: `"dns": ["1.1.1.1", "8.8.8.8"]`. |
| `Bind for 0.0.0.0:80 failed: port is already allocated` / `address already in use` | Another program or container is using that host port | Choose another host port (`-p 8080:80`), or find the user: Linux `sudo ss -ltnp \| grep :80`; Windows `netstat -ano \| findstr :80`. |
| `E: Unable to locate package docker-ce` (Ubuntu) | Repo not added correctly, `apt update` not run, or an unsupported distro derivative | Re-check `/etc/apt/sources.list.d/docker.sources`; run `sudo apt update`; on Linux Mint/other derivatives set `Suites:` to the underlying Ubuntu codename (unsupported by Docker). |
| `NO_PUBKEY` / GPG error / `The following signatures couldn't be verified` | Missing/unreadable key | Re-download the key to `/etc/apt/keyrings/docker.asc` and `sudo chmod a+r` it. |
| `E: Conflicting values set for option Signed-By` | Old `docker.list` and new `docker.sources` both exist | Remove the old file: `ls /etc/apt/sources.list.d/` and delete the stale Docker entry. |
| `unknown shorthand flag` / `docker: 'compose' is not a docker command` | Compose plugin not installed, or you are following an old tutorial | Install `docker-compose-plugin` (Linux). Use `docker compose` (space), not `docker-compose`. |
| `docker.exe: error during connect: ... This error may indicate that the docker daemon is not running.` | Same as the first row | Start Docker Desktop. |
| `Windows Home: Hyper-V not available` | Home lacks Hyper-V | Use the **WSL 2** backend. |
| Docker Desktop fails to install on Windows Server | Unsupported | Docker's docs state Windows Server editions are not supported by Docker Desktop. Use a Linux VM or Docker Engine. |
| `docker: Error response from daemon: could not select device driver "" with capabilities: [[gpu]]` | `--gpus` used but no GPU runtime/toolkit available | See [Chapter 8](08-docker-for-ai-and-ml.md) (NVIDIA Container Toolkit, drivers). |
| Disk full / `no space left on device` | Images, build cache, volumes, unrotated logs | `docker system df`, then selective prune ([Chapter 3](03-core-cli-commands.md)); set log rotation ([4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation)). On Windows, compact the WSL disk. |
| A weird new `docker images` table with `DISK USAGE`, `CONTENT SIZE`, `EXTRA` columns | Not an error: the **containerd image store**, the default for fresh installs in Engine 29, uses this new layout | Normal. See [Chapter 3](03-core-cli-commands.md#32-docker-images--docker-image-ls-list-local-images) for how to read it. |

**General debugging approach:**

1. `docker version` shows whether the daemon is reachable (a `Server:` block).
2. `docker context ls` shows which engine your CLI targets (an asterisk marks the active one).
3. Read the log: Linux `journalctl -u docker.service -n 100 --no-pager`; Docker Desktop **Troubleshoot** menu or its log folder.
4. Search the exact error text. Forums (Docker, Stack Overflow, GitHub issues) usually contain the answer.

---

## 10. Common mistakes

| Mistake | Why it hurts | Better |
|---|---|---|
| Installing `docker.io` or `docker-compose` from Ubuntu's default repos | Outdated or conflicting with the official packages | Use Docker's official repo ([4.3](#43-step-2-set-up-dockers-official-apt-repository)) |
| Skipping the `-a` in `usermod -aG` | You remove yourself from `sudo` and other groups | Always `-aG` |
| Forgetting to log out/in after adding yourself to the `docker` group | Group membership does not apply yet | `newgrp docker` or re-login |
| Putting project files in `C:\Users\...` (accessed as `/mnt/c`) on Windows | Slow bind mounts, broken hot reload | Keep them in `~/projects` inside WSL |
| Letting WSL use all the memory by default | Windows becomes sluggish | Set `memory=` in `.wslconfig` |
| Building images on an Apple Silicon Mac and deploying to amd64 servers without `--platform` | `exec format error` in production | `docker buildx build --platform linux/amd64 ...` |
| Mixing up `docker-compose` (retired) and `docker compose` | Command not found / old behavior | Use `docker compose` |
| Trusting that `ufw deny 8080` blocks a published container port | Docker bypasses it via iptables | Bind to `127.0.0.1` or use the `DOCKER-USER` chain |
| Exposing the daemon on TCP port 2375 | Remote root for anyone who finds it | SSH contexts or TLS-protected 2376 |
| Running Docker Desktop commercially at a large company without a subscription | Licence violation | Pay, or use Engine/Podman/Rancher |
| Leaving the default `json-file` logs unbounded | Disk fills up | Rotation in `daemon.json` |
| Assuming a work-around that disables TLS verification is OK | Defeats supply-chain security | Install the corporate CA instead |

---

## 11. Try it yourself

1. **Install** Docker for your OS following the section above (Windows: 2, macOS: 3, Linux: 4).
2. **Verify** with `docker version`, `docker info`, and `docker run hello-world`. Match each line of the output to the table in [Section 7.3](#73-every-line-explained).
3. **Read your engine's facts** and write them down:
   ```bash
   docker info --format 'OS={{.OperatingSystem}} Arch={{.Architecture}} CPUs={{.NCPU}} MemBytes={{.MemTotal}} Storage={{.Driver}} Cgroup={{.CgroupVersion}}'
   ```
   (`--format` prints just the fields named in the Go template.)
4. **Windows only:** create `%UserProfile%\.wslconfig` with a `memory=` limit, run `wsl --shutdown`, restart Docker Desktop, and confirm with `docker info --format "{{.MemTotal}}"` that the limit changed.
5. **Windows only:** clone a repo into `~/projects` inside WSL and into `/mnt/c/Users/Public/`, then run the file-creation speed test from [2.11](#211-the-1-windows-performance-tip-keep-your-project-files-inside-wsl) in both.
6. **Mac only:** run `docker run --rm alpine uname -m` and then `docker run --rm --platform linux/amd64 alpine uname -m`, and explain why the outputs differ.
7. **Linux only:** run `sudo systemctl status docker`, then `journalctl -u docker.service -n 20 --no-pager`. Add log rotation from [4.7](#47-step-6-configure-the-daemon-etcdockerdaemonjson-and-log-rotation).
8. **Everyone:** run `docker context ls` and identify which context is active and what endpoint it uses.
9. Run `docker run --rm -it alpine sh`, look around with `cat /etc/os-release`, `hostname`, `ps`, then `exit`. Then run `docker ps -a` and confirm no container was left behind (the `--rm` flag removed it).

---

## 12. Summary

| Topic | Key point |
|---|---|
| OS choice | **Linux is the production OS.** Windows and macOS are development machines that run Linux containers in a hidden Linux VM. Windows containers exist but are niche. |
| Windows 11 | `wsl --install`, then Docker Desktop with the **WSL 2 backend**; enable WSL integration; limit resources via `.wslconfig` (then `wsl --shutdown`); **keep project files inside the WSL filesystem**. |
| macOS | Docker Desktop (Apple silicon or Intel build). Remember `arm64` vs `amd64`; use `--platform` and `docker buildx build --platform` so images run on servers. |
| Linux | Install **Docker Engine** from Docker's official apt repo (`docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`); post-install: `docker` group (root-equivalent!), `systemctl enable`, log rotation; consider **rootless**. |
| Servers | Engine only, pinned versions, log rotation, no exposed daemon, SSH contexts for remote control, mind that Docker bypasses `ufw`. |
| Licensing | **Docker Engine is free.** Docker Desktop is free for personal/education/open-source use and for companies under **250 employees and $10M revenue**; otherwise a paid subscription is required (Pro $9, Team $15, Business $24 per user/month, annual, at time of writing). Alternatives exist. |
| Verification | `docker version` (Client **and** Server), `docker run hello-world`, understand each output line. |
| Troubleshooting | Daemon not running, `docker` group, WSL integration, virtualization, proxies/certificates, architecture mismatch. |

**Next:** [Chapter 3: Core CLI Commands](03-core-cli-commands.md): the commands you will type every day, explained flag by flag.
