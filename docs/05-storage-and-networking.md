# Chapter 5: Storage and Networking

> **What you will learn in this chapter**
>
> - Why data disappears when a container is removed, and what the container's writable layer really is
> - The three ways to store data: volumes, bind mounts, and tmpfs (and when to use each)
> - `-v` vs `--mount`, named vs anonymous volumes, volume drivers
> - How to back up and restore a volume, and how to fix permission/UID problems
> - Windows and WSL path pitfalls
> - How Docker networking works: default bridge vs user-defined bridge, DNS by container name
> - Port publishing, `host`, `none`, `overlay`, `macvlan`, `ipvlan`
> - The localhost confusion (the #1 beginner bug), `host.docker.internal`
> - Why Docker bypasses `ufw`, and what to do about it
> - Debugging connectivity with `nicolaka/netshoot`, and putting a reverse proxy in front

**Prerequisites:** [Chapter 3](03-core-cli-commands.md) (basic CLI) and [Chapter 4](04-dockerfile-deep-dive.md) (images and layers).

**Where this fits:** Containers are *disposable* by design. That is a feature, but it raises two questions every real application must answer: "where does my data live?" and "how do my services find each other?" This chapter answers both. Next up: [Chapter 6: Docker Compose](06-docker-compose.md) wires storage and networking together declaratively.

---

# Part A: Storage

## A1. Why data disappears: the container's writable layer

Recall from [Chapter 4](04-dockerfile-deep-dive.md#3-layers-and-the-build-cache) that an image is a stack of **read-only** layers. When you start a container, Docker adds one thin **writable layer** on top. Everything the running program creates or modifies lands there.

```
   Container "web-1"
 +-----------------------------------+
 | WRITABLE container layer          |  <- new files, edits, logs go here
 +-----------------------------------+      (deleted when the container is removed)
 | image layer 3 (read-only)         |
 | image layer 2 (read-only)         |   <- shared by every container
 | image layer 1 (read-only)         |      started from this image
 +-----------------------------------+
```

This uses a technique called **copy-on-write** (CoW): if a container modifies a file that lives in a read-only image layer, Docker first copies that file up into the writable layer and modifies the copy. The original image layers are never touched, which is why a hundred containers can share one image.

### The consequences

1. **Remove the container, lose the data.** The writable layer's lifetime is exactly the container's lifetime. `docker rm` deletes it.
2. **Data is hard to share.** Another container cannot easily read this container's writable layer.
3. **It's slower.** Writes go through the union filesystem driver, with copy-on-write overhead. Write-heavy workloads (databases) suffer.
4. **It's tied to the host.** The data lives in a Docker-internal directory.

### See it happen

```bash
docker run -it --name scratchpad alpine sh
```

- `-it`: `-i` keep input open + `-t` allocate a terminal (an interactive shell).
- `--name scratchpad`: give the container a memorable name.
- `alpine sh`: image `alpine`, command `sh`.

Inside the container:

```sh
echo "important data" > /notes.txt
cat /notes.txt
exit
```

Back on the host:

```bash
docker start scratchpad                # stopped containers keep their writable layer
docker exec scratchpad cat /notes.txt  # prints: important data

docker rm scratchpad                   # now the container (and layer) is gone
docker run --rm alpine cat /notes.txt  # cat: can't open '/notes.txt': No such file or directory
```

The file survived a *stop*/*start* but **not** a `docker rm`. And since you will constantly recreate containers (deploying a new image version = removing the old container and creating a new one), anything worth keeping must live **outside** the writable layer.

You can see what a container changed with:

```bash
docker diff scratchpad
```

Output uses `A` (added), `C` (changed), `D` (deleted):

```
A /notes.txt
C /root
A /root/.ash_history
```

### The three solutions

Docker provides three ways to mount storage into a container that lives outside the writable layer:

```
 +--------------------------- HOST MACHINE ----------------------------+
 |                                                                     |
 |   Docker-managed area         Any folder you choose     RAM         |
 |   (/var/lib/docker/volumes)   (e.g. ~/project/src)                  |
 |          |                            |                    |        |
 |       VOLUME                      BIND MOUNT             TMPFS      |
 |          |                            |                    |        |
 |    +-----+----------------------------+--------------------+----+   |
 |    |                       CONTAINER                            |   |
 |    |    /var/lib/postgresql   /app/src            /tmp          |   |
 |    +------------------------------------------------------------+   |
 +---------------------------------------------------------------------+
```

| | Volume | Bind mount | tmpfs |
|---|---|---|---|
| Where data lives | Docker-managed location on the host (inside the VM on Docker Desktop) | Any path you specify on the host | Host RAM only |
| Survives container removal | Yes | Yes (it's your folder) | **No** (gone at stop) |
| Created/managed by | Docker (`docker volume ...`) | You | Docker |
| Best for | Databases, uploads, anything the app owns | Source code in development, config files, sharing host files | Secrets/scratch data you don't want on disk |
| Portable across machines | Yes (via backup, drivers) | No (depends on host path) | n/a |
| Performance | Good (native on Linux; best on Docker Desktop) | Native on Linux; **can be slow** on Docker Desktop/WSL when crossing the OS boundary | Fastest |
| Pre-populated from the image | Yes, if empty on first mount | No (hides the image's content at that path) | No |

---

## A2. Volumes

A **volume** is a directory managed by Docker, stored in a Docker-controlled part of the filesystem. On a Linux host it lives under `/var/lib/docker/volumes/<name>/_data`. On Docker Desktop (Windows/Mac) it lives inside Docker's Linux virtual machine, and you should **not** poke at it directly; use a helper container to look inside (shown in A9).

Think of a volume as an **external hard drive** that Docker manages: you can plug it into one container, unplug it (remove the container), and plug it into another, and the data stays.

### Basic volume commands

```bash
docker volume create pgdata
```

Creates a named volume called `pgdata`. Output: `pgdata`.

```bash
docker volume ls
```

Lists volumes. Example:

```
DRIVER    VOLUME NAME
local     pgdata
```

`local` is the default **volume driver** (stores on the local disk).

```bash
docker volume inspect pgdata
```

Shows details (JSON):

```json
[
    {
        "CreatedAt": "2026-10-23T09:41:12Z",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/pgdata/_data",
        "Name": "pgdata",
        "Options": null,
        "Scope": "local"
    }
]
```

- `Mountpoint`: where the data lives *inside the Docker host* (on Docker Desktop that's inside its VM).
- `Scope: local`: the volume exists on this one host.

```bash
docker volume rm pgdata        # delete a volume (fails if a container still uses it)
docker volume prune            # delete all UNUSED anonymous volumes (confirmation prompt)
docker volume prune -a         # delete ALL unused volumes, including named ones (dangerous!)
```

> **Warning:** Since Docker Engine 23.0, `docker volume prune` removes only unused **anonymous** volumes by default; `-a`/`--all` also removes unused **named** volumes. Both are **irreversible**. "Unused" means "no container (even a stopped one) references it", so a database volume whose container you removed looks "unused" and `prune -a` will delete your data. Be deliberate. Also, `docker system prune --volumes` includes volumes.

### Using a volume

```bash
docker run -d --name db \
  -e POSTGRES_PASSWORD=devonly \
  -v pgdata:/var/lib/postgresql \
  postgres:18
```

- `-d`: detached (run in the background).
- `-e POSTGRES_PASSWORD=devonly`: set an environment variable; the Postgres image requires a superuser password. (A throwaway dev value only; never reuse real passwords.)
- `-v pgdata:/var/lib/postgresql`: mount the named volume `pgdata` at that path in the container. Format is `NAME:CONTAINER_PATH`.
- `postgres:18`: the image.

> **Version note (verified from the official image documentation):** For PostgreSQL **17 and earlier**, mount the volume at `/var/lib/postgresql/data`. Starting with **18**, the image's data directory became version-specific (`/var/lib/postgresql/18/docker`) and the image's declared volume moved to `/var/lib/postgresql`, so you mount at `/var/lib/postgresql`. Mounting the old path on 18 leaves your named volume empty while the database writes to a hidden anonymous volume, a classic way to "lose" data. Always read the image's page on Docker Hub for the version you use.

Prove persistence:

```bash
docker exec -it db psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (42);"
docker rm -f db                                  # destroy the container...
docker run -d --name db2 -e POSTGRES_PASSWORD=devonly -v pgdata:/var/lib/postgresql postgres:18
docker exec -it db2 psql -U postgres -c "SELECT * FROM t;"   # ...data is still there: 42
```

(Wait a few seconds after starting for Postgres to be ready.) Clean up your own test objects: `docker rm -f db2` and `docker volume rm pgdata`.

### Volumes are populated from the image on first use

If you mount an **empty** named volume at a path where the image already has files, Docker **copies the image's files into the volume** the first time. That's how the Postgres image's initial directory structure ends up in your volume. This does *not* happen for bind mounts (a bind mount hides whatever the image had there).

### Named vs anonymous volumes

| | Named volume | Anonymous volume |
|---|---|---|
| Created by | `-v pgdata:/path` or `docker volume create` | `-v /path` (no name) or a Dockerfile `VOLUME` |
| Name | You choose (`pgdata`) | Random 64-char hex ID |
| Easy to find/reuse | Yes | Hard (looks like `4a80acc4d43a...`) |
| Cleaned up when? | Never automatically | Only when you use `docker rm -v` / `--rm` on the container, or prune |
| Use it? | **Yes, always, for data you care about** | Avoid; they pile up |

When you run `docker volume ls` and see dozens of random hex names, those are leftover anonymous volumes, mostly created by images with a `VOLUME` instruction when someone ran them without `-v`. The `--rm` flag on `docker run` also removes the container's *anonymous* volumes when it exits (not named ones).

---

## A3. Bind mounts

A **bind mount** maps a specific file or directory from your host into the container. Both sides see the *same* files in real time: edit on the host, the container sees it instantly (and vice versa).

Analogy: a **shortcut / shared folder** between your computer and the container.

```bash
# macOS / Linux / Git Bash / WSL
docker run --rm -it -v "$(pwd)":/app -w /app python:3.13-slim python -c "print(open('hello.txt').read())"
```

- `-v "$(pwd)":/app`: mount the current directory (`$(pwd)` prints its absolute path) at `/app` in the container. **Use an absolute path.** A bare name like `data` (no slash) would be interpreted as a *named volume*, not a folder! `./data` (with `./`) or an absolute path makes it a bind mount.
- `-w /app`: **working directory** for the command.
- The quotes protect paths containing spaces.

PowerShell:

```powershell
docker run --rm -it -v "${PWD}:/app" -w /app python:3.13-slim python -c "print(open('hello.txt').read())"
```

Typical uses:

- **Development:** mount your source folder so code changes show up without rebuilding the image ([Chapter 6](06-docker-compose.md#docker-compose-watch-hot-reload-for-development) shows Compose's `watch` as an alternative).
- **Config files:** `-v ./nginx.conf:/etc/nginx/nginx.conf:ro` (`:ro` = read-only).
- **Getting results out:** write output files to a mounted host folder.

Downsides:

- Depends on the host's directory layout, so it's not portable.
- The container can modify (and delete!) your host files unless you add `:ro`. A script with a bug can run `rm -rf /app/*` and delete your real project.
- **Hides** whatever the image had at that path. If your image has `/app/node_modules` and you bind-mount your source over `/app`, the container now sees your host's folder (with no `node_modules`, or the wrong platform's!). A common fix is an extra anonymous/named volume layered over just that subpath.
- Permission/UID mismatches (section A7).
- Slow on Docker Desktop for large trees (section A8).

---

## A4. tmpfs mounts

A **tmpfs** mount stores data in the host's **RAM** (memory), never on disk. It disappears when the container stops. (On Linux only.)

```bash
docker run --rm -it --tmpfs /scratch:rw,size=64m,mode=1777 alpine sh
```

- `--tmpfs /scratch`: create a RAM-backed filesystem at `/scratch`.
- `rw,size=64m,mode=1777`: options. `size=64m` caps it at 64 MB (a full tmpfs fills RAM, so always set a limit). `mode=1777` sets permissions (world-writable with sticky bit, like `/tmp`).

Use it for:

- Sensitive temporary data you don't want persisted on disk (decrypted tokens, session scratch)
- High-speed temp files (caches, sockets)
- Making a container's root filesystem read-only (`--read-only`) while still giving the app a writable `/tmp`:

```bash
docker run --rm --read-only --tmpfs /tmp python:3.13-slim python -c "open('/tmp/x','w').write('ok'); print('wrote /tmp/x')"
```

`--read-only` makes the whole container filesystem read-only (a great security hardening measure; see [Chapter 10](10-security.md)); the tmpfs gives the one exception.

---

## A5. `-v` vs `--mount`

Docker has two syntaxes for the same thing.

### `-v` / `--volume` (short, positional)

```
-v SOURCE:DESTINATION[:OPTIONS]
```

```bash
-v pgdata:/var/lib/postgresql                 # named volume
-v /var/lib/postgresql                        # anonymous volume
-v /home/me/app:/app                          # bind mount (absolute path)
-v ./config.yml:/etc/app/config.yml:ro        # bind mount, read-only
```

### `--mount` (long, explicit, key=value)

```bash
--mount type=volume,source=pgdata,target=/var/lib/postgresql
--mount type=bind,source="$(pwd)",target=/app,readonly
--mount type=tmpfs,target=/scratch,tmpfs-size=67108864
```

| Key | Meaning |
|---|---|
| `type` | `volume`, `bind`, or `tmpfs` |
| `source` (or `src`) | Volume name or host path (omit for tmpfs and anonymous volumes) |
| `target` (or `dst`, `destination`) | Path inside the container |
| `readonly` (or `ro`) | Mount read-only |
| `volume-subpath` | Mount only a subdirectory of a volume (newer feature) |
| `bind-propagation`, `volume-nocopy`, `tmpfs-size`, `tmpfs-mode` | Advanced options |

### Differences that matter

| | `-v` | `--mount` |
|---|---|---|
| Host path doesn't exist (bind) | **Silently creates it** (as a root-owned directory, a common source of confusing permission errors) | **Errors out** (safer; you notice typos) |
| Readability | Terse, easy to mistype | Self-documenting |
| Options | Limited set | Everything |
| Ambiguity | Name vs path decided by whether it looks like a path | Explicit `type=` |

`--mount` is the clearer, safer choice (Docker's documentation points new users to it), but `-v` is everywhere in tutorials, so you must read both. In Compose files, you'll see both a short syntax (`pgdata:/var/lib/postgresql`) and a long syntax (`type: volume`, `source:`, `target:`), see [Chapter 6](06-docker-compose.md#volumes).

---

## A6. Volume drivers

A **volume driver** (plugin) decides *where and how* a volume's data is stored. The default `local` driver stores on the host's disk. Other drivers store data on network storage or cloud block devices, so a volume can follow a container to a different host.

The built-in `local` driver is more flexible than people realize: it can mount a **network filesystem**:

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.50,rw,nfsvers=4 \
  --opt device=:/exports/shared \
  nfs-data
```

- `--driver local`: the default driver (stated for clarity).
- `--opt type=nfs`: the filesystem type to mount (the same as the Linux `mount -t nfs`).
- `--opt o=...`: mount options (NFS server address, read-write, NFS version).
- `--opt device=:/exports/shared`: the exported path on the NFS server.

Third-party volume plugins (distributed in the Docker plugin ecosystem, usually installed with `docker plugin install`) integrate cloud disks, distributed filesystems, and storage arrays. Check each plugin's maintenance status before depending on it: several older plugins are abandoned, and the ecosystem is much smaller today than in the Swarm era. For Kubernetes-style environments, storage is instead handled by CSI drivers ([Chapter 11](11-production-and-enterprise.md)).

Key point for beginners: **you will rarely need anything but the default driver on a single machine.** Know the concept so you're not surprised when you read "storage driver" (a different concept: how the Docker daemon stores *image layers*, e.g. `overlayfs`/`overlay2`) vs "volume driver" (where *volume data* lives).

---

## A7. Permissions and UID problems

### Why they happen

Linux file ownership is stored as **numeric** IDs: a **UID** (user ID) and **GID** (group ID). The *names* (`alice`, `postgres`) are just labels looked up in `/etc/passwd`. A container has its own `/etc/passwd`, but it shares the host's kernel, so the kernel only ever compares numbers.

Suppose on your Linux host you are UID 1000. A bind-mounted folder is owned by 1000. Inside a container, the app runs as `app` with UID 10001. The kernel compares 10001 with the folder's owner 1000 and says "permission denied". Or the reverse: the container (running as root, UID 0) creates files in your bind mount, and they appear on the host owned by **root**, which you can't edit without `sudo`.

### Common symptoms

| Symptom | Likely cause |
|---|---|
| `Permission denied` when the app writes into a mounted folder | UID mismatch |
| Files created by the container are owned by `root` on the host | Container runs as root |
| `chown: changing ownership ... Operation not permitted` | The filesystem (NFS, Windows share) doesn't support chown |
| Postgres/MySQL won't start: "data directory has wrong ownership/permissions" | The volume or mount isn't owned by the DB user |

### Fixes

1. **Run the container as your host user** (great for dev bind mounts):

   ```bash
   docker run --rm --user "$(id -u):$(id -g)" -v "$(pwd)":/app -w /app python:3.13-slim python gen.py
   ```
   - `--user UID:GID`: run the process as that numeric user/group. `$(id -u)` and `$(id -g)` print your UID and GID. Files created are then owned by you.
   - In Compose: `user: "${UID:-1000}:${GID:-1000}"`.

2. **Build the right ownership into the image** for named volumes. A named volume that is empty on first mount inherits the ownership/permissions of the image's directory at that path:

   ```dockerfile
   RUN mkdir -p /data && chown 10001:10001 /data
   USER 10001:10001
   ```
   Then mount a named volume at `/data` at run time (`-v mydata:/data`). Because the volume is empty on first use, Docker copies `/data`'s ownership from the image into it. Do the `chown` **before** the volume is first populated; an already-populated volume keeps whatever ownership it has.

3. **One-off fix with a helper container** for an existing volume:

   ```bash
   docker run --rm -v myvol:/data alpine chown -R 10001:10001 /data
   ```
   - `alpine`: a tiny image just to run `chown` as root against the volume mounted at `/data`.

4. **In Compose**, an "init" service that chowns a volume and exits, with the app depending on it via `condition: service_completed_successfully` ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)).

5. **SELinux hosts (Fedora, RHEL, CentOS):** bind mounts can fail with permission denied even when UIDs match, because SELinux labels block access. Add `:z` (label shared between containers) or `:Z` (private to this container) to the mount: `-v ./data:/data:Z`. Never use `:Z` on system directories like `/home` or `/etc`: it relabels them and can break your host.

6. **User namespace remapping / rootless Docker:** advanced setups where container "root" is mapped to an unprivileged host UID. File ownership on bind mounts then looks strange (shifted UIDs). Know that this exists if you see ownership like `100000`.

On Docker Desktop (Windows and macOS), the file sharing layer translates ownership for bind mounts, so you typically see fewer UID issues there than on Linux, which is why "it worked on my Mac but not on the Linux server" is a classic.

---

## A8. Windows and WSL path pitfalls

Windows adds its own complications. If you develop on Windows (like many readers), read this section carefully.

### Background: where does Docker actually run on Windows?

Docker Desktop on Windows runs the Docker engine inside a lightweight Linux VM. With the default **WSL 2** backend ("Windows Subsystem for Linux 2": a real Linux kernel running alongside Windows), you have two filesystems:

- The **Windows filesystem** (`C:\Users\you\...`), visible inside WSL as `/mnt/c/Users/you/...`
- The **WSL Linux filesystem** (ext4, e.g. `/home/you/...`), visible from Windows as `\\wsl$\Ubuntu\home\you\...` (or `\\wsl.localhost\...`)

Crossing between them goes through a translation layer that is **significantly slower** for many small files and doesn't carry Linux semantics perfectly.

### Pitfall 1: Path syntax differs per shell

| Shell | Mount the current folder |
|---|---|
| PowerShell | `-v "${PWD}:/app"` (or `-v "${PWD}\data:/data"`) |
| Command Prompt (cmd.exe) | `-v "%cd%:/app"` |
| Git Bash | `-v "$(pwd -W):/app"` or `-v "$(pwd):/app"` (see pitfall 2) |
| WSL (Ubuntu etc.) | `-v "$(pwd):/app"` |

Windows drive paths contain a colon (`C:\...`), which clashes with `-v`'s colon separator. Docker handles `C:\path:/container` specially, but it's a reason to prefer **`--mount`** on Windows: `--mount type=bind,source="${PWD}",target=/app`.

### Pitfall 2: Git Bash rewrites your paths

Git Bash (MSYS) automatically "converts" anything that looks like a Unix path into a Windows path before passing it to programs, so `docker run -v /app:/app ...` or `docker exec c ls /etc` may get mangled into `C:/Program Files/Git/app` or similar. Workarounds:

```bash
MSYS_NO_PATHCONV=1 docker exec mycontainer ls /etc      # disable conversion for one command
docker exec mycontainer ls //etc                         # a double leading slash also works
```

(Or use PowerShell/WSL, which don't do this.)

### Pitfall 3: Performance across the boundary

If your project lives under `C:\...` and you run Docker from WSL (or Docker Desktop mounts it), file operations like `npm install`, `git status`, or a hot-reload watcher are far slower than on the Linux filesystem, sometimes by 10x or more. **Best practice: keep your project inside the WSL filesystem** (`~/projects/my-app` in your WSL distro) and run Docker commands from there. Open it in VS Code with the WSL extension. Bind mounts then stay inside Linux and are fast.

### Pitfall 4: File-change events (hot reload) don't always propagate

Tools that rely on `inotify` (Linux file-change notifications) to restart the server may not see changes made from Windows on a `/mnt/c` bind mount. Symptoms: you save a file and nothing reloads. Fixes: keep the code in the WSL filesystem, enable polling in the tool (`WATCHPACK_POLLING=true` for webpack/Next.js, `CHOKIDAR_USEPOLLING=1`, `uvicorn --reload` polling via `watchfiles` config), or use Compose `watch` ([Chapter 6](06-docker-compose.md#docker-compose-watch-hot-reload-for-development)), which syncs files rather than relying on a bind mount.

### Pitfall 5: Line endings

Git on Windows may convert text files to CRLF (`\r\n`). A shell script copied into a Linux container with CRLF breaks with `exec ./entrypoint.sh: no such file or directory` (the "interpreter" is `/bin/sh\r`). Fix: add a `.gitattributes` with `*.sh text eol=lf` (and re-checkout), or set `git config core.autocrlf input`.

### Pitfall 6: Case sensitivity and permissions

Linux paths are case-sensitive (`App.py` != `app.py`); Windows is not (by default). Code that works on Windows can fail in the container with "file not found". Also, files on a Windows mount typically all appear with permissive modes (`rwxrwxrwx`), so a bind-mounted `chmod`-dependent program (like an SSH key requiring `0600`) can fail: copy such files into the image or a volume instead.

### Pitfall 7: Where are my volumes?

Named volumes are inside the Docker Desktop VM's disk image, not under `C:\`. Don't hunt for them in Explorer. To look inside, run a throwaway container with the volume mounted (shown next), or use Docker Desktop's Volumes tab, which has a file browser.

---

## A9. Backup and restore of a volume (tar method)

Because a volume is just a directory, the universal, reliable way to back it up is: run a **temporary helper container** that mounts both the volume and a host folder, and uses `tar` to archive one into the other.

### Look inside a volume

```bash
docker run --rm -v pgdata:/data alpine ls -la /data
```

- `-v pgdata:/data`: mount the volume at `/data`.
- `alpine ls -la /data`: list its contents.

### Backup

```bash
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /data .
```

- `-v pgdata:/data:ro`: the volume to back up, mounted **read-only** so the backup can't change it.
- `-v "$(pwd)":/backup`: bind-mount the current host folder as the destination.
- `tar czf /backup/pgdata-backup.tar.gz -C /data .`: `c` create an archive, `z` gzip-compress it, `f` write to this file; `-C /data` change into `/data` first; `.` archive everything there. Using `-C /data .` keeps paths in the archive relative (so restoring to any location works).
- The archive appears in your current host directory as `pgdata-backup.tar.gz`.

PowerShell version: replace `"$(pwd)"` with `"${PWD}"`.

### Restore

```bash
docker volume create pgdata-restored
docker run --rm \
  -v pgdata-restored:/data \
  -v "$(pwd)":/backup:ro \
  alpine sh -c "cd /data && tar xzf /backup/pgdata-backup.tar.gz"
```

- `x` extract, `z` gunzip, `f` read from this file.
- The `sh -c "cd /data && tar ..."` form runs two steps in one container.

Restore into a fresh volume and point a test container at it before touching production.

### Copying data between volumes

```bash
docker run --rm -v old_vol:/from:ro -v new_vol:/to alpine sh -c "cp -a /from/. /to/"
```

`cp -a` = archive mode (preserves ownership, permissions, timestamps); `/from/.` copies the directory *contents* including hidden files.

### Big caveat: databases and consistency

Copying a database's files while it is **running** can produce a corrupt, inconsistent backup (files are mid-write). For databases use the database's own tooling:

```bash
docker exec db pg_dump -U postgres mydb > mydb.sql          # logical backup (PostgreSQL)
docker exec -i db psql -U postgres mydb < mydb.sql          # restore
```

Or stop the container briefly before doing the tar backup. Back up *regularly*, store backups off the machine, and **test restores**: an untested backup is a hope, not a backup. ([Chapter 11](11-production-and-enterprise.md) covers backup strategy.)

---

## A10. Storage quick decision guide

```
 Is it data the app must KEEP (database, uploads, model cache)?
        -> Named volume.
 Do you want to edit files on your host and see them live in the container (dev)?
        -> Bind mount (or Compose watch).
 Is it a config file the container should read but never change?
        -> Bind mount with :ro, or Compose configs/secrets.
 Is it temporary, sensitive, or performance-critical scratch data?
        -> tmpfs.
 Is it throwaway output of a one-off job?
        -> Writable layer is fine (or bind mount the output folder).
```

### Common mistakes (storage)

1. Storing database data in the writable layer (no volume), then losing it on `docker rm` or a redeploy.
2. Using an anonymous volume by accident (`-v /var/lib/mysql`) and not knowing which random-named volume has the data.
3. Mounting the wrong path (see the Postgres 18 note) so the data lives in a different, anonymous volume.
4. Running `docker volume prune -a` or `docker system prune --volumes` on a machine whose databases are currently stopped.
5. Bind-mounting over `node_modules` / a virtualenv and shadowing the dependencies installed in the image.
6. Letting `-v ./dir:/path` auto-create a missing host directory as root.
7. Mixing up `-v data:/x` (named volume called `data`) with `-v ./data:/x` (folder).
8. Backing up a live database by copying files.
9. Keeping a project in `C:\` and bind-mounting into WSL containers, then complaining about slowness.
10. Treating `docker compose down -v` as harmless (it deletes the named volumes declared in the file; see [Chapter 6](06-docker-compose.md)).

### Try it yourself (storage)

1. Run an `alpine` container, write a file, `docker rm` it, and show the file is gone.
2. Repeat with a named volume and show the file survives.
3. Bind-mount a folder with `:ro` and try to write; read the error.
4. Back up your volume to a `.tar.gz`, delete the volume, create a new one, restore, and verify the file is back.
5. Run a container with `--read-only --tmpfs /tmp` and observe what works and what fails.

---

# Part B: Networking

## B1. The big picture

Docker networking answers: *How do containers talk to each other, to the host, and to the outside world, and how does the outside world talk to them?*

Each container gets its own **network namespace**: a private copy of the network stack (its own network interfaces, IP address, routing table, and firewall rules), isolated from the host and from other containers. It's like every container is a separate computer with its own network card. Docker then connects those virtual computers using **virtual networks**.

Terms you'll need (each explained once, here):

- **IP address**: a numeric address for a machine on a network (like `172.18.0.3`).
- **Port**: a numbered "door" (0-65535) on a machine; a program listens on a port (web servers typically 80/443).
- **Subnet**: a range of IP addresses that belong to one network (like `172.18.0.0/16`: the `/16` says the first 16 bits are the network part, so ~65,000 addresses).
- **Gateway**: the address that traffic goes through to leave a network.
- **NAT** (Network Address Translation): a router trick that rewrites addresses in packets so private addresses can talk to the outside (and so traffic arriving at one address gets forwarded to a private one). Docker uses NAT for published ports and for outbound traffic.
- **DNS** (Domain Name System): the phone book that translates names (`google.com`, or `db`) to IP addresses.
- **Bridge**: a virtual network switch that connects virtual network cables.
- **veth pair**: a virtual ethernet cable with two ends; one end sits in the container, the other plugs into the bridge.

### Network drivers at a glance

| Driver | What it does | Typical use |
|---|---|---|
| `bridge` | Private virtual network on one host; containers on it can talk to each other | **The default; 95% of use cases** |
| `host` | Container shares the host's network stack directly | Max performance, or many ports; Linux-focused |
| `none` | No network at all (only loopback) | Isolated batch jobs, maximum lockdown |
| `overlay` | Virtual network spanning **multiple Docker hosts** (VXLAN tunnels) | Docker Swarm multi-host |
| `macvlan` | Gives each container its own MAC and IP on your physical LAN | Legacy apps that must look like physical machines |
| `ipvlan` | Like macvlan but containers share the host's MAC address | Environments that limit MACs per port |

List the networks on your machine:

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
d3841588ee9f   bridge    bridge    local
8218b77415d0   host      host      local
436e8a16a4ef   none      null      local
```

Those three exist on every Docker installation. (`null` is the driver behind `none`.)

---

## B2. Default bridge vs user-defined bridge

### The default bridge (`bridge`, with a `docker0` interface on Linux)

If you start a container without specifying a network, it joins the default network named `bridge`. Containers there get IPs from a private range (commonly `172.17.0.0/16`) and can reach each other **by IP address only**; there is no automatic name resolution. Other drawbacks: all containers share the one network (no isolation between apps), and you must publish ports to reach them from outside.

```bash
docker run -d --name a1 alpine sleep 3600
docker run -d --name a2 alpine sleep 3600
docker exec a1 ping -c 1 a2
```

Result: `ping: bad address 'a2'`, because the default bridge has no DNS for container names. You'd need `docker inspect a2` to find its IP. (There is a legacy `--link` flag; it is deprecated: don't use it.)

### The user-defined bridge: what you should always use

```bash
docker network create appnet
docker run -d --name b1 --network appnet alpine sleep 3600
docker run -d --name b2 --network appnet alpine sleep 3600
docker exec b1 ping -c 2 b2
```

- `docker network create appnet`: creates a new bridge network named `appnet` (bridge is the default driver; the long form is `--driver bridge`).
- `--network appnet`: attach the container to it.

Output:

```
PING b2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.095 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.106 ms
```

It works by **name**. Docker runs an **embedded DNS server** inside each user-defined network (reachable at `127.0.0.11` from inside containers). When `b1` asks "who is `b2`?", Docker's DNS answers with b2's current IP. This is **service discovery** (finding other services by name rather than hard-coding IP addresses, which change every time a container is recreated).

| Feature | Default `bridge` | User-defined bridge |
|---|---|---|
| Name-based DNS between containers | No | **Yes** |
| Isolation between apps | All share one network | Only members of the same network can talk |
| Attach/detach running containers | Must stop container | Hot (`docker network connect`/`disconnect`) |
| Custom subnet/gateway/options | Limited (daemon-wide) | Per-network |
| Network aliases | No | Yes (`--network-alias`) |
| Recommended | Rarely | **Always** |

Compose ([Chapter 6](06-docker-compose.md)) creates a user-defined network for each project automatically, which is why services can reach each other by service name (`db`, `redis`) with zero configuration.

Clean up the experiment (only objects you created): `docker rm -f a1 a2 b1 b2 && docker network rm appnet`.

### Commands for managing networks

```bash
docker network create --driver bridge --subnet 10.50.0.0/24 --gateway 10.50.0.1 mynet
docker network inspect mynet
docker network connect mynet some-container            # attach a running container
docker network connect --alias api mynet some-container   # also give it a DNS alias "api"
docker network disconnect mynet some-container
docker network rm mynet                                # only if no containers are attached
docker network prune                                   # remove ALL unused networks
```

- `--subnet`: choose the IP range (avoid clashes with your corporate VPN, see B10).
- `--gateway`: the network's gateway address.
- A container can be on **several networks at once**. That's how you build tiers: a `frontend` network (proxy + app) and a `backend` network (app + database), with the proxy unable to reach the database.

### Network aliases

A container can have multiple DNS names on a network:

```bash
docker run -d --name pg --network appnet --network-alias database --network-alias db postgres:18
```

Other containers on `appnet` can reach it as `pg`, `database` or `db`. If several containers share an alias, Docker's DNS answers with **all** their IPs in rotating order (simple **round-robin** load distribution, but with no health checking).

### Internal networks

```bash
docker network create --internal backend-only
```

`--internal` makes a network with **no outbound access** to the outside world (and no published ports), so containers on it can only talk to each other. Great for databases that should never reach the internet.

---

## B3. Port publishing

A container on a bridge network has a private IP that isn't reachable from outside the host. To let your browser (or other machines) reach it, you **publish** a port: Docker forwards traffic from a host port to a container port.

```bash
docker run -d --name web -p 8080:80 nginx
```

`-p HOST_PORT:CONTAINER_PORT`: requests to the host on port 8080 are forwarded to port 80 inside the container. Open `http://localhost:8080`.

```
   Browser --> host:8080 ==[Docker NAT/proxy]==> container:80 (nginx)
```

### All the forms of `-p`

| Syntax | Meaning |
|---|---|
| `-p 8080:80` | Host port 8080 -> container 80, on **all** host interfaces (`0.0.0.0` and `::`) |
| `-p 127.0.0.1:8080:80` | Only reachable **from the host itself**. Recommended for databases and dev tools |
| `-p 8080:80/udp` | UDP instead of TCP (default is TCP) |
| `-p 8080:80/tcp -p 8080:80/udp` | Both |
| `-p 8000-8010:8000-8010` | A range |
| `-p 80` | Host port chosen randomly (see with `docker port`) |
| `-P` (capital) | Publish **all** ports declared with `EXPOSE` on random host ports |
| `-p [::1]:8080:80` | IPv6 loopback |

```bash
docker port web
```

Shows the mappings, e.g. `80/tcp -> 0.0.0.0:8080`.

### Exposing vs publishing

These two words are confused constantly.

| | **EXPOSE** (`EXPOSE 80` in Dockerfile, `expose:` in Compose) | **Publish** (`-p`, `ports:`) |
|---|---|---|
| What it does | Documents that the container *listens* on that port | Actually opens a path from host to container |
| Reachable from other containers on the same network? | They can reach **any** listening port anyway, regardless of EXPOSE | n/a |
| Reachable from the host/outside? | **No** | **Yes** |
| Needed for container-to-container? | **No** | **No**: containers on the same user-defined network connect directly to the container port |

Practical rule: **Do not publish ports you don't need on the host.** Your app's database should not have `-p 5432:5432` in production; only the app container (on the same network) needs to reach it. Publishing a database port to `0.0.0.0` exposes it to the whole network (and, with a cloud public IP, the internet).

> Note also: when two containers talk over a Docker network, they use the **container** port (80), never the host port (8080). The mapping `8080:80` exists only at the host boundary.

### How publishing works under the hood

On Linux, Docker adds firewall rules (using `iptables` or `nftables`, see B9) that DNAT (rewrite the destination of) packets arriving at the host port to the container's IP and port, and for some cases (traffic from the host itself, or IPv6->IPv4) uses a small userland helper process called `docker-proxy`. This is why you can find `docker-proxy` processes in `ps` output on a Docker host.

---

## B4. The #1 beginner bug: `localhost` confusion

**Inside a container, `localhost` (127.0.0.1) means *that container itself*.** Not your laptop. Not the other container. Each container has its own loopback interface.

```
 +------------- HOST ------------------------------------------+
 |  localhost:5432  <-- a Postgres you installed on your laptop |
 |                                                              |
 |   +-- container "api" ----+     +-- container "db" ------+   |
 |   | localhost = ITSELF    |     | localhost = ITSELF     |   |
 |   | (nothing on 5432 here)|     | Postgres on 5432       |   |
 |   +-----------------------+     +------------------------+   |
 +--------------------------------------------------------------+
```

### Symptom

Your API container has `DATABASE_URL=postgresql://user:pw@localhost:5432/app` and crashes with `connection refused`. It worked on your laptop because there the database really was on localhost.

### The fix depends on where the target lives

| From | To | Use | Notes |
|---|---|---|---|
| Container A | Container B (same user-defined network) | **`http://B-name:PORT`** (service name) | e.g. `postgresql://db:5432/app`. Use the container port, not a published host port |
| Container | Service running **on the host** (not in Docker) | `host.docker.internal` | See B5 |
| Host (your browser/curl) | Container | `localhost:PUBLISHED_PORT` | Requires `-p` |
| Container A | Container B on different networks | They can't, until you attach one to both networks | By design |
| Another machine | Container | `host-ip:PUBLISHED_PORT` | Requires `-p` bound to a non-loopback address |

### The mirror-image bug: the server binds only to `127.0.0.1` inside the container

If your server inside the container listens on `127.0.0.1:8000` (many dev servers default to this), then `-p 8000:8000` publishes the port but the connection arrives on the container's *network interface* (e.g. `172.17.0.2`), not its loopback, and the app doesn't answer. You'll see an empty reply or connection reset from the host.

Fix: make the app listen on **`0.0.0.0`** (all interfaces) inside the container:

| Framework | How |
|---|---|
| Uvicorn/FastAPI | `uvicorn app.main:app --host 0.0.0.0` |
| Flask | `flask run --host=0.0.0.0` |
| Django | `python manage.py runserver 0.0.0.0:8000` |
| Node/Express | `app.listen(3000, '0.0.0.0')` |
| Vite dev server | `vite --host` |
| Spring Boot | listens on all interfaces by default |

(The outer publish `-p 127.0.0.1:8000:8000` is the *host-side* restriction; it's unrelated and fine.)

---

## B5. Container to host: `host.docker.internal`

Sometimes a container needs to reach something running on your host machine (a database you started natively, a dev API, Ollama or another local service).

- **Docker Desktop (Windows/macOS):** the special DNS name **`host.docker.internal`** resolves to the host automatically.

  ```bash
  docker run --rm curlimages/curl -s http://host.docker.internal:11434/
  ```

- **Linux (Docker Engine):** the name isn't defined by default. Add it using the special value `host-gateway` (supported since Docker Engine 20.10):

  ```bash
  docker run --rm --add-host=host.docker.internal:host-gateway curlimages/curl -s http://host.docker.internal:11434/
  ```

  - `--add-host name:ip`: adds an entry to the container's `/etc/hosts`.
  - `host-gateway`: a keyword Docker replaces with the host's gateway IP address on the bridge.

  In Compose:

  ```yaml
  services:
    api:
      extra_hosts:
        - "host.docker.internal:host-gateway"
  ```

Common gotcha on Linux: the host service must listen on an address reachable from the bridge (e.g. `0.0.0.0` or the bridge gateway IP like `172.17.0.1`); a service bound only to `127.0.0.1` on the host isn't reachable from containers (loopback isn't forwarded). On Docker Desktop, connections via `host.docker.internal` are proxied to the host, so a service bound to host loopback generally still works, but check if your firewall blocks it. Also note host firewalls (including Windows Defender Firewall) can block the connection.

### Host networking as the alternative

`--network host` lets the container use the host's network directly (below).

---

## B6. The other drivers

### B6.1 `host`: share the host's network stack

```bash
docker run --rm --network host nginx
```

The container has **no network isolation**: it uses the host's IP and ports directly. `localhost` inside it **is** the host's localhost. Consequences:

- **No port mapping:** `-p`/`-P` are ignored (Docker prints a warning). If nginx listens on 80, it's host port 80.
- Port conflicts: two containers can't both bind port 80.
- Slightly faster (no NAT/proxy), and handy for apps that need thousands of ports or must see host interfaces (monitoring agents, network tools).
- **Less secure** (no network isolation).
- **Platform caveat (from Docker's docs):** host networking is a Linux-host feature. On Docker Desktop it is an opt-in setting available in Docker Desktop 4.34 and later (Settings -> Resources -> Network -> "Enable host networking", requires being signed in), with limitations: only TCP/UDP, can't bind to the host's own interfaces from the VM perspective, and not compatible with Enhanced Container Isolation. It does not apply to Windows containers. On Docker Desktop without that setting, `--network host` attaches to the **VM's** network, **not** your laptop's, which surprises many people.

### B6.2 `none`: no networking

```bash
docker run --rm --network none alpine ip addr
```

Only a loopback interface (`lo`) exists. Use for jobs that process local data and must never touch the network, or as a security measure (e.g., running untrusted code).

### B6.3 `overlay`: multi-host networks

An **overlay network** connects containers running on **different Docker hosts** as if they were on one network, by wrapping their traffic in tunnels (VXLAN: virtual extensible LAN). It's used with **Docker Swarm** (Docker's built-in clustering mode, `docker swarm init`) and requires a swarm. Typical use is Swarm services; if you're on Kubernetes, the cluster's **CNI** plugin does this job instead. Encryption of overlay traffic is optional (`--opt encrypted`).

```bash
docker swarm init
docker network create --driver overlay --attachable my-overlay
```

(`--attachable` lets standalone containers join, not just Swarm services.) Swarm is covered briefly in [Chapter 6](06-docker-compose.md#compose-vs-swarm-vs-kubernetes) and [Chapter 11](11-production-and-enterprise.md).

### B6.4 `macvlan`: containers as first-class LAN citizens

With **macvlan**, each container gets its own **MAC address** (a network card's hardware address) and an IP on your physical network, so other machines on the LAN see it as a separate physical device. No NAT, no port publishing.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 lan
docker run -d --network lan --ip 192.168.1.50 nginx
```

- `-d macvlan`: driver.
- `--subnet`/`--gateway`: must match your real LAN.
- `-o parent=eth0`: the host's physical network interface to attach to.
- `--ip`: a fixed address (pick one outside your DHCP range!).

Caveats: the **host cannot talk to its own macvlan containers** by default (a Linux kernel rule; workaround: create a macvlan sub-interface on the host); your network switch/Wi-Fi must allow multiple MAC addresses per port (most Wi-Fi adapters do not, so macvlan generally fails on Wi-Fi); cloud providers usually block it; and on Docker Desktop containers live inside a VM, so macvlan generally doesn't give you what you want there (I'm describing the architecture; check Docker's current docs if you need it). Use it for legacy software that needs a real LAN address, network appliances, or IoT bridging.

### B6.5 `ipvlan`

Like macvlan, but all containers **share the host's MAC address** while having distinct IPs. Useful where switches limit MAC addresses per port (and often works on Wi-Fi). Two modes: **L2** (containers appear on the same Layer-2 LAN) and **L3** (the host acts as a router between subnets: no broadcast, great for routed/BGP environments).

```bash
docker network create -d ipvlan --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 -o ipvlan_mode=l2 ipv
```

### Which driver?

```
 Single host, normal app?                       -> user-defined bridge (default driver)
 Need host's network performance / many ports?  -> host (Linux)
 Total isolation?                               -> none
 Multi-host with Swarm?                         -> overlay
 Container needs its own LAN IP?                -> macvlan / ipvlan
```

---

## B7. Inspecting networks

### Networks and who is on them

```bash
docker network inspect appnet
```

Abbreviated output:

```json
[
  {
    "Name": "appnet",
    "Driver": "bridge",
    "IPAM": {
      "Config": [ { "Subnet": "172.18.0.0/16", "Gateway": "172.18.0.1" } ]
    },
    "Containers": {
      "3f1a...": { "Name": "b1", "IPv4Address": "172.18.0.2/16" },
      "9c7d...": { "Name": "b2", "IPv4Address": "172.18.0.3/16" }
    }
  }
]
```

- `IPAM` ("IP Address Management"): the subnet and gateway.
- `Containers`: who is attached, with their IPs.

### A container's IP

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' b1
```

- `-f` (`--format`): a Go template to pull one field out of the JSON.
- This prints every IP the container has (one per attached network).

### From inside the container

```bash
docker exec b1 cat /etc/resolv.conf     # shows "nameserver 127.0.0.11" on user-defined networks
docker exec b1 cat /etc/hosts           # the container's own hosts file
docker exec b1 nslookup b2              # DNS lookup of another container
docker exec b1 ip addr                  # interfaces and IPs (if the 'ip' tool exists)
docker exec b1 wget -qO- http://b2:80   # test an HTTP connection (alpine has wget)
```

Many slim images lack these tools. That's what `netshoot` (B11) is for.

---

## B8. IPv6 note

By default, Docker networks are **IPv4 only**. IPv6 support has improved considerably in recent Engine releases, and the setup details have changed between versions, so treat the following as the general shape and check the documentation for your version:

```bash
docker network create --ipv6 --subnet 2001:db8:1::/64 v6net
```

- `--ipv6`: enable IPv6 on this user-defined network (you can supply an IPv6 subnet; the documentation example range `2001:db8::/32` is reserved for docs, so use your real prefix or a ULA `fd00::/8` range).
- For the **default** bridge and daemon-wide behaviour you edit `/etc/docker/daemon.json` (`"ipv6": true` and related keys such as `fixed-cidr-v6`) and restart the daemon.
- Published ports (`-p`) bind both IPv4 and IPv6 host addresses by default (`0.0.0.0` and `::`); use `127.0.0.1:` or `[::1]:` prefixes to restrict.
- Common surprise: an app listening only on IPv4 `0.0.0.0` won't answer IPv6 connections; and `localhost` may resolve to `::1` first. If "localhost works but 127.0.0.1 doesn't" (or vice versa) suspect IPv4/IPv6 mismatch.
- I have not verified the specifics for every Engine version (for example, which defaults changed in the latest releases), so consult `docs.docker.com/engine/daemon/ipv6/` for your installed version.

---

## B9. Firewalls and why Docker "bypasses ufw"

### The surprise

On a Linux server you enable the **ufw** ("Uncomplicated Firewall", a friendly front-end for the Linux firewall) and block port 8080:

```bash
sudo ufw deny 8080
docker run -d -p 8080:80 nginx
```

...and port 8080 is **still reachable from the internet**. Your firewall rule is effectively ignored.

### Why

Linux packet filtering is done by **netfilter**, configured with `iptables` or `nftables`. Packets pass through *chains* of rules. `ufw` adds its rules to the `INPUT`/`OUTPUT` chains, which handle traffic addressed *to or from the host itself*. But published container ports aren't addressed to the host process; Docker uses the `nat` table to rewrite the destination to the container's IP, and the packet then goes through the **FORWARD** chain, a path ufw's rules do not cover. Docker's documentation states plainly that Docker and ufw are incompatible in this respect: Docker diverts published-port traffic before it reaches the chains ufw uses, so ufw rules don't apply to it.

```
 Internet --> [nat PREROUTING: Docker DNAT to container] --> [FORWARD chain] --> container
                                                   (ufw's INPUT rules never see this)
```

(Docker also enables IP forwarding and sets the default forwarding policy to drop so the host doesn't become an open router; it adds its own allow rules for published ports. Docker Engine can also use an `nftables` firewall backend, which Docker documents as selectable via the `firewall-backend` daemon option; details and maturity vary by version, so verify before relying on it. On `firewalld` systems Docker creates a `docker` zone.)

### What to do about it

1. **Don't publish ports you don't need.** If only your app container needs the database, don't publish it at all.
2. **Bind to loopback** for things only the host should reach: `-p 127.0.0.1:5432:5432`. Then only local processes (or an SSH tunnel) can connect. This is the simplest and most reliable fix.
3. **Put one reverse proxy in front** and publish only 80/443 (B12).
4. **Use the `DOCKER-USER` chain**: Docker evaluates rules in this chain first and never overwrites it, so it's the supported place for your own filtering of published-container traffic. Example (allow only one subnet to reach published ports on interface `eth0`):

   ```bash
   sudo iptables -I DOCKER-USER -i eth0 ! -s 203.0.113.0/24 -j DROP
   ```
   Explanation: `-I DOCKER-USER` inserts the rule at the top of the chain; `-i eth0` matches traffic arriving on the external interface; `! -s 203.0.113.0/24` matches sources **outside** that subnet; `-j DROP` silently discards. (Test carefully so you don't lock yourself out; note `DOCKER-USER` sees traffic *after* DNAT, so match on container ports/IPs with `-m conntrack --ctorigdstport` if you need the original published port.)
5. **Use the cloud firewall** (AWS Security Groups, Azure NSG, GCP firewall rules) in front of the machine. It filters before packets reach the host, so Docker can't bypass it.
6. **Don't set `"iptables": false`** in `daemon.json` as a shortcut. Docker's docs warn it likely breaks container networking (no outbound masquerade, and port publishing stops working as expected), and replicating Docker's rules by hand is complex.
7. **Never expose the Docker daemon's TCP socket** (port 2375) without TLS. It's remote root on the host ([Chapter 10](10-security.md)).

Also remember the earlier lesson: `-p 5432:5432` binds `0.0.0.0` by default. Many real-world breaches are exactly "database published to the internet".

---

## B10. Other networking gotchas

### DNS inside containers

- User-defined networks: containers use Docker's embedded DNS (`127.0.0.11`), which forwards external lookups to the host's DNS servers.
- Corporate networks and VPNs often need specific resolvers. Set them per container (`--dns 10.0.0.2`, `--dns-search corp.example`) or daemon-wide in `daemon.json` (`"dns": ["10.0.0.2"]`).
- Symptom: `curl: (6) Could not resolve host` inside a container while it works on the host. Check `docker run --rm alpine nslookup example.com`.

### Subnet conflicts with VPNs

Docker's default networks use private ranges (`172.17.0.0/16` and onward through `172.x`, also `192.168.x`). If your company's VPN uses an overlapping range, traffic to those corporate addresses goes to a container network instead, and things mysteriously time out. Fix: pick your own ranges, either per network (`--subnet`) or daemon-wide with `default-address-pools` in `daemon.json`:

```json
{
  "default-address-pools": [
    { "base": "10.200.0.0/16", "size": 24 }
  ]
}
```

(`base` is the pool; `size: 24` means each new network gets a `/24` slice.) Restart the daemon after editing. On Docker Desktop, the equivalent is in Settings -> Resources -> Network.

### MTU problems

Over some VPNs or cloud overlays, the maximum packet size (MTU) is smaller than Docker assumes, causing large downloads to hang while small requests work. Fix by setting `"mtu": 1400` (or the correct value) in `daemon.json`, or `-o com.docker.network.driver.mtu=1400` on a network.

### Container ports vs host ports in URLs

An app in container A calling container B must use B's **internal** port. If you published B as `-p 9000:8080`, A calls `http://b:8080`, not 9000.

### Ephemeral IPs

Never hard-code a container's IP. It changes when the container is recreated. Use names.

### Container-to-container across Compose projects

Each Compose project has its own network by default. To connect two projects, put the services on a shared **external** network ([Chapter 6](06-docker-compose.md#networks)).

---

## B11. Debugging connectivity with `nicolaka/netshoot`

Application images are slim: no `ping`, `dig`, `curl`, `tcpdump`. Rather than installing tools into your app image, use **netshoot**, a community-maintained "network troubleshooting Swiss army knife" image packed with `curl`, `dig`, `nslookup`, `ping`, `traceroute`, `tcpdump`, `ss`, `ip`, `nmap`, `iperf3`, `netcat` and more.

### Method 1: join a Docker network

```bash
docker run -it --rm --network appnet nicolaka/netshoot
```

You get a shell *on that network* as if you were another container, so you can test from the outside-in:

```bash
# inside netshoot
nslookup db                       # does the name resolve? shows the container IP
ping -c 2 db                      # basic reachability
nc -zv db 5432                    # is the port open? ("succeeded!" or "Connection refused")
curl -v http://api:8000/health    # test the HTTP service
dig +short api                    # DNS answer only
```

- `nc -zv host port`: `nc` (netcat); `-z` scan without sending data; `-v` verbose.

### Method 2: share another container's network namespace

```bash
docker run -it --rm --network container:my-api nicolaka/netshoot
```

`--network container:my-api` makes the debug container use **exactly the same network stack as `my-api`**: same IP, same `localhost`, same open ports. Perfect for questions like "what is my-api actually listening on?" and for packet capture:

```bash
ss -tlnp                          # listening TCP ports in that namespace
tcpdump -i eth0 -n port 8000      # watch traffic hitting the app
```

This works even when the target is a distroless/scratch container with no tools at all.

### Method 3: host network

```bash
docker run -it --rm --network host nicolaka/netshoot
```

Only meaningful with true host networking (Linux). (Reminder: on Docker Desktop this joins the VM's network.)

### A systematic debugging checklist

1. **Is the container running?** `docker ps`; if not, `docker logs <name>`.
2. **Is the app listening on `0.0.0.0` on the right port?** `docker exec <c> ss -tlnp` (or netshoot Method 2).
3. **Are both containers on the same user-defined network?** `docker network inspect`.
4. **Does DNS resolve the name?** `nslookup <service>` from netshoot.
5. **Is the port reachable?** `nc -zv <service> <container-port>`.
6. **Are you using the container port (not the host-mapped one)?**
7. **Is the host port published and not already in use?** `docker port <c>`; on the host `ss -tlnp | grep 8080` (Linux) or `netstat -ano | findstr :8080` (Windows).
8. **Firewall?** Host firewall, cloud security group, corporate proxy.
9. **Is the app up *yet*?** Services might not be ready when the dependent starts ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)).

---

## B12. Reverse proxies in front

A **reverse proxy** is a server that sits in front of your apps, receives all incoming requests, and forwards each to the right backend. It's the standard way to expose several containerized services through ports 80/443. It also:

- terminates **TLS/HTTPS** (handles certificates in one place)
- routes by hostname or path (`api.example.com` -> api container; `/` -> frontend)
- load-balances across replicas
- adds caching, rate limiting, compression, security headers
- means backend containers need **no published ports**, shrinking the attack surface (and sidestepping the ufw problem)

```
                      published: 80, 443 only
 Internet --> [ reverse proxy container ] --+--> api   (no published port)
                                             +--> web   (no published port)
                                             +--> admin (no published port)
              all on a private user-defined network; DB on a second, internal network
```

Popular choices:

| Proxy | Style | Why |
|---|---|---|
| **nginx** | Config files | Ubiquitous, fast, flexible |
| **Caddy** | Simple config; **automatic HTTPS** via Let's Encrypt | Easiest way to get TLS |
| **Traefik** | Auto-discovers containers via Docker **labels** | Dynamic, great for Compose/Swarm |
| **HAProxy**, **Envoy** | High-performance / service-mesh style | Advanced load balancing |

### Example: nginx in front of a FastAPI app

`compose.yaml` (see [Chapter 6](06-docker-compose.md) for the syntax):

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "80:80"               # the ONLY published port
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api

  api:
    build: .
    expose:
      - "8000"                # documentation only; no host publishing
```

`nginx.conf`:

```nginx
server {
    listen 80;

    location / {
        proxy_pass http://api:8000;                 # "api" resolves via Docker DNS
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Line notes:

- `proxy_pass http://api:8000;`: forward requests to the service named `api`, container port 8000.
- `proxy_set_header ...`: pass the original host, client IP and scheme to the backend; otherwise the app only sees the proxy's IP. The app must be configured to trust these headers (for uvicorn: `--proxy-headers --forwarded-allow-ips="*"` only when the proxy is the sole way in).
- **nginx resolves `api` once at startup.** If the `api` container isn't resolvable yet, nginx fails with `host not found in upstream`. Solutions: start order with healthchecks ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)), or use a variable with Docker's resolver so it re-resolves: `resolver 127.0.0.11 valid=10s; set $upstream http://api:8000; proxy_pass $upstream;`.
- If the backend container is **recreated** and gets a new IP, nginx with a static `proxy_pass` hostname may keep the stale IP until reloaded (`docker compose exec proxy nginx -s reload`). The variable-plus-resolver form avoids that too.

### Example: Caddy with automatic HTTPS (production-style)

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data         # stores certificates; persist it!
  api:
    build: .
volumes:
  caddy_data:
```

`Caddyfile`:

```
api.example.com {
    reverse_proxy api:8000
}
```

Caddy obtains and renews a Let's Encrypt certificate for `api.example.com` automatically (given public DNS pointing at your host and open ports 80/443). That's the whole TLS setup.

---

## B13. Common mistakes (networking)

1. **Using `localhost` to reach another container.** Use the service name. (B4)
2. **App listening on `127.0.0.1` inside the container.** Use `0.0.0.0`. (B4)
3. **Using the host-mapped port between containers.** Use the container port.
4. **Staying on the default bridge** and wondering why names don't resolve. Create a user-defined network (Compose does this for you).
5. **Publishing databases to `0.0.0.0`.** Use `127.0.0.1:` or don't publish. (B3, B9)
6. **Trusting ufw** to protect published ports. (B9)
7. **Hard-coding container IPs.**
8. **Port already in use** (`bind: address already in use`): another process or container holds the host port. Pick another host port.
9. **Assuming `EXPOSE` publishes.** It doesn't. (B3)
10. **Using `--network host` on Docker Desktop** expecting laptop networking.
11. **Subnet clashes with VPN**; random timeouts to corporate addresses. (B10)
12. **Forgetting that `docker network prune` / `docker system prune`** remove unused networks, breaking assumptions of external-network setups.
13. **nginx stale upstream IP** after the backend container is recreated. (B12)

## B14. Try it yourself (networking)

1. Create a user-defined bridge network. Start two `alpine` containers on it and ping each by name. Repeat on the default bridge and observe the failure.
2. Run `nginx` with `-p 8080:80`, then `-p 127.0.0.1:8081:80`; from another machine (or via your LAN IP) compare reachability.
3. Run a Python HTTP server bound to `127.0.0.1` inside a container, publish it, and watch it fail; then rebind to `0.0.0.0` and watch it work.
4. Run `nicolaka/netshoot --network container:<name>` against a running container and use `ss -tlnp` to list its listening ports.
5. Start a service on your host (for example `python -m http.server 9000`) and reach it from a container using `host.docker.internal`.
6. Build a two-network layout (`frontend`, `backend --internal`) with a proxy, an app and a database, and prove the proxy cannot reach the database but the app can.

---

## Summary

**Storage**

- The container's writable layer vanishes with the container. Keep data in **named volumes** (databases, uploads), **bind mounts** (dev source code, read-only config), or **tmpfs** (RAM-only scratch).
- Prefer `--mount` for clarity; `-v` silently creates missing host directories.
- Back up volumes with a helper container and `tar`; use native tools (`pg_dump`) for databases.
- UID/GID mismatches are the main cause of permission errors; on Windows, keep code in the WSL filesystem and mind line endings and Git Bash path rewriting.

**Networking**

- Always use **user-defined bridge networks**: they give you DNS by container name. Compose does this automatically.
- `localhost` inside a container is the container itself. Use service names between containers, `host.docker.internal` for the host, and bind servers to `0.0.0.0`.
- Publishing (`-p`) is separate from exposing. Bind sensitive ports to `127.0.0.1`. Docker's published ports bypass ufw; protect with loopback binding, a reverse proxy, `DOCKER-USER` rules or an upstream cloud firewall.
- Debug with `nicolaka/netshoot`, preferably with `--network container:<name>`.

**Next:** [Chapter 6: Docker Compose](06-docker-compose.md), where you'll describe a whole multi-container application (services, volumes, networks, health checks) in one YAML file and start it with one command.
