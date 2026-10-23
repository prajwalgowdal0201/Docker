# Chapter 5: Storage and Networking

This chapter covers where container data lives and how containers communicate. Part A covers storage: the writable layer, volumes, bind mounts, tmpfs, backups and permissions. Part B covers networking: bridge networks, DNS by container name, port publishing, the other network drivers, firewalls, debugging and reverse proxies.

**Prerequisites:** [Chapter 3](03-core-cli-commands.md) (basic CLI) and [Chapter 4](04-dockerfile-deep-dive.md) (images and layers).

---

# Part A: Storage

## A1. Why data disappears: the container's writable layer

An image is a stack of read-only layers ([Chapter 4](04-dockerfile-deep-dive.md#3-layers-and-the-build-cache)). When a container starts, Docker adds one thin **writable layer** on top. Everything the running program creates or modifies lands there.

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

**Copy-on-write** (CoW): when a container modifies a file in a read-only image layer, Docker copies the file into the writable layer and modifies the copy. Image layers are never changed, so many containers can share one image.

Consequences:

1. The writable layer lives exactly as long as the container. `docker rm` deletes it.
2. Other containers cannot easily read it.
3. Writes go through the union filesystem driver, so write-heavy workloads (databases) are slower.
4. The data sits in a Docker-internal directory on the host.

### Demonstration

```bash
docker run -it --name scratchpad alpine sh
```

| Part | Meaning |
|---|---|
| `-it` | `-i` keep input open, `-t` allocate a terminal. |
| `--name scratchpad` | Name the container. |
| `alpine sh` | Image `alpine`, command `sh`. |

Inside the container:

```sh
echo "important data" > /notes.txt
cat /notes.txt
exit
```

On the host:

```bash
docker start scratchpad                # stopped containers keep their writable layer
docker exec scratchpad cat /notes.txt  # prints: important data

docker rm scratchpad                   # container and layer are deleted
docker run --rm alpine cat /notes.txt  # cat: can't open '/notes.txt': No such file or directory
```

The file survives `stop`/`start` but not `docker rm`. Deploying a new image version removes the old container, so data worth keeping must live outside the writable layer.

`docker diff scratchpad` lists a container's filesystem changes (`A` added, `C` changed, `D` deleted).

### The three storage types

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
| Where data lives | Docker-managed location on the host (inside the VM on Docker Desktop) | Any host path you specify | Host RAM only |
| Survives container removal | Yes | Yes (it is your folder) | No (gone at stop) |
| Created/managed by | Docker (`docker volume ...`) | You | Docker |
| Best for | Databases, uploads, data the app owns | Source code in development, config files | Sensitive or scratch data that should not touch disk |
| Portable across machines | Yes (via backup, drivers) | No (depends on host path) | n/a |
| Performance | Good (native on Linux; best on Docker Desktop) | Native on Linux; can be slow on Docker Desktop/WSL across the OS boundary | Fastest |
| Pre-populated from the image | Yes, if empty on first mount | No (hides the image's content at that path) | No |

---

## A2. Volumes

A **volume** is a directory managed by Docker. On Linux it lives under `/var/lib/docker/volumes/<name>/_data`. On Docker Desktop (Windows/Mac) it lives inside Docker's Linux VM; do not access it directly, use a helper container (A9).

Analogy: a volume is an external drive that can be moved between containers while keeping its data.

### Basic volume commands

```bash
docker volume create pgdata      # create a named volume; prints: pgdata
docker volume ls                 # list volumes
docker volume inspect pgdata     # show details as JSON
```

`docker volume ls` output (`local` is the default **volume driver**, which stores on local disk):

```
DRIVER    VOLUME NAME
local     pgdata
```

`docker volume inspect pgdata` prints JSON including:

- `Mountpoint`: where the data lives inside the Docker host (inside the VM on Docker Desktop).
- `Scope: local`: the volume exists on this host only.

```bash
docker volume rm pgdata        # delete a volume (fails if a container still uses it)
docker volume prune            # delete unused anonymous volumes (confirmation prompt)
docker volume prune -a         # delete ALL unused volumes, including named ones
```

> **Warning:** Since Docker Engine 23.0, `docker volume prune` removes only unused anonymous volumes; `-a`/`--all` also removes unused named volumes. Both are irreversible. "Unused" means no container, even a stopped one, references the volume. A database volume whose container was removed counts as unused, and `prune -a` deletes it. `docker system prune --volumes` also removes volumes.

### Using a volume

```bash
docker run -d --name db \
  -e POSTGRES_PASSWORD=devonly \
  -v pgdata:/var/lib/postgresql \
  postgres:18
```

| Part | Meaning |
|---|---|
| `-d` | Run detached (in the background). |
| `-e POSTGRES_PASSWORD=devonly` | Set an environment variable. The Postgres image requires a superuser password. Use throwaway values only. |
| `-v pgdata:/var/lib/postgresql` | Mount named volume `pgdata` at that container path (`NAME:CONTAINER_PATH`). |
| `postgres:18` | The image. |

> **Version note (verified from the official image documentation):** For PostgreSQL 17 and earlier, mount the volume at `/var/lib/postgresql/data`. From 18, the data directory is version-specific (`/var/lib/postgresql/18/docker`) and the declared volume moved to `/var/lib/postgresql`, so mount there. Mounting the old path on 18 leaves the named volume empty while the database writes to a hidden anonymous volume. Check the image's Docker Hub page for the version in use.

Verify persistence:

```bash
docker exec -it db psql -U postgres -c "CREATE TABLE t(x int); INSERT INTO t VALUES (42);"
docker rm -f db                                  # destroy the container
docker run -d --name db2 -e POSTGRES_PASSWORD=devonly -v pgdata:/var/lib/postgresql postgres:18
docker exec -it db2 psql -U postgres -c "SELECT * FROM t;"   # data is still there: 42
```

Wait a few seconds after starting for Postgres to be ready. Clean up: `docker rm -f db2` and `docker volume rm pgdata`.

### Volumes are populated from the image on first use

When an empty named volume is mounted at a path where the image has files, Docker copies the image's files into the volume on first mount. This does not happen for bind mounts, which hide whatever the image had at that path.

### Named vs anonymous volumes

| | Named volume | Anonymous volume |
|---|---|---|
| Created by | `-v pgdata:/path` or `docker volume create` | `-v /path` (no name) or a Dockerfile `VOLUME` |
| Name | You choose | Random 64-character hex ID |
| Easy to find/reuse | Yes | No |
| Cleaned up | Never automatically | With `docker rm -v`, `--rm` on the container, or prune |
| Use for data you care about | Yes | No; they accumulate |

Many random hex names in `docker volume ls` are leftover anonymous volumes, usually from images with a `VOLUME` instruction run without `-v`. `--rm` on `docker run` removes the container's anonymous volumes on exit, not named ones.

---

## A3. Bind mounts

A **bind mount** maps a host file or directory into the container. Both sides see the same files in real time.

```bash
# macOS / Linux / Git Bash / WSL
docker run --rm -it -v "$(pwd)":/app -w /app python:3.13-slim python -c "print(open('hello.txt').read())"
```

| Part | Meaning |
|---|---|
| `-v "$(pwd)":/app` | Mount the current directory (`$(pwd)` prints its absolute path) at `/app`. Use an absolute path or `./`. A bare name such as `data` is treated as a named volume. |
| `-w /app` | Working directory for the command. |
| Quotes | Protect paths containing spaces. |

PowerShell:

```powershell
docker run --rm -it -v "${PWD}:/app" -w /app python:3.13-slim python -c "print(open('hello.txt').read())"
```

Typical uses:

- **Development:** mount source code so changes appear without rebuilding ([Chapter 6](06-docker-compose.md#docker-compose-watch-hot-reload-for-development) covers Compose `watch` as an alternative).
- **Config files:** `-v ./nginx.conf:/etc/nginx/nginx.conf:ro` (`:ro` is read-only).
- **Extracting results:** write output to a mounted host folder.

Limitations:

- Depends on host directory layout, so it is not portable.
- The container can modify or delete host files unless mounted `:ro`.
- Hides whatever the image had at that path. Bind-mounting source over `/app` hides the image's `/app/node_modules`. A common fix is an extra named or anonymous volume mounted over that subpath.
- Permission and UID mismatches (A7).
- Slow on Docker Desktop for large trees (A8).

---

## A4. tmpfs mounts

A **tmpfs** mount stores data in host RAM, never on disk. It disappears when the container stops. Linux only.

```bash
docker run --rm -it --tmpfs /scratch:rw,size=64m,mode=1777 alpine sh
```

| Part | Meaning |
|---|---|
| `--tmpfs /scratch` | Create a RAM-backed filesystem at `/scratch`. |
| `size=64m` | Cap at 64 MB. A full tmpfs consumes RAM, so always set a limit. |
| `mode=1777` | World-writable with sticky bit, like `/tmp`. |

Uses: sensitive temporary data, fast temp files, and a writable `/tmp` on a read-only container:

```bash
docker run --rm --read-only --tmpfs /tmp python:3.13-slim python -c "open('/tmp/x','w').write('ok'); print('wrote /tmp/x')"
```

`--read-only` makes the container filesystem read-only (a hardening measure; see [Chapter 10](10-security.md)). The tmpfs is the one writable exception.

---

## A5. `-v` vs `--mount`

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

### `--mount` (long, explicit key=value)

```bash
--mount type=volume,source=pgdata,target=/var/lib/postgresql
--mount type=bind,source="$(pwd)",target=/app,readonly
--mount type=tmpfs,target=/scratch,tmpfs-size=67108864
```

| Key | Meaning |
|---|---|
| `type` | `volume`, `bind` or `tmpfs` |
| `source` (or `src`) | Volume name or host path (omit for tmpfs and anonymous volumes) |
| `target` (or `dst`, `destination`) | Path inside the container |
| `readonly` (or `ro`) | Mount read-only |
| `volume-subpath` | Mount only a subdirectory of a volume (newer feature) |
| `bind-propagation`, `volume-nocopy`, `tmpfs-size`, `tmpfs-mode` | Advanced options |

### Differences

| | `-v` | `--mount` |
|---|---|---|
| Bind host path does not exist | Silently creates it as a root-owned directory (a source of permission errors) | Errors out |
| Readability | Terse | Self-documenting |
| Options | Limited set | All |
| Name vs path | Inferred from whether the source looks like a path | Explicit `type=` |

`--mount` is clearer and safer; Docker's documentation points new users to it. `-v` is common in tutorials. Compose accepts both a short syntax (`pgdata:/var/lib/postgresql`) and a long syntax (`type: volume`, `source:`, `target:`); see [Chapter 6](06-docker-compose.md#volumes).

---

## A6. Volume drivers

A **volume driver** (plugin) decides where and how volume data is stored. The default `local` driver uses the host's disk. Other drivers use network storage or cloud block devices.

The `local` driver can mount a network filesystem:

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.50,rw,nfsvers=4 \
  --opt device=:/exports/shared \
  nfs-data
```

| Option | Meaning |
|---|---|
| `--driver local` | The default driver. |
| `--opt type=nfs` | Filesystem type (same as `mount -t nfs`). |
| `--opt o=...` | Mount options: NFS server address, read-write, NFS version. |
| `--opt device=:/exports/shared` | Exported path on the NFS server. |

Third-party volume plugins (installed with `docker plugin install`) integrate cloud disks, distributed filesystems and storage arrays. Check each plugin's maintenance status first; several older plugins are abandoned. Kubernetes handles storage with CSI drivers ([Chapter 11](11-production-and-enterprise.md)).

On a single machine the default driver is usually enough. Do not confuse the two terms:

- **Storage driver**: how the Docker daemon stores image layers (for example `overlayfs`/`overlay2`).
- **Volume driver**: where volume data lives.

---

## A7. Permissions and UID problems

Linux stores file ownership as numeric IDs: a **UID** (user ID) and **GID** (group ID). Names such as `alice` are labels looked up in `/etc/passwd`. A container has its own `/etc/passwd` but shares the host kernel, which compares only numbers.

Example: on a Linux host you are UID 1000 and a bind-mounted folder is owned by 1000. The container app runs as UID 10001, so the kernel denies access. In the reverse case, a container running as root (UID 0) creates files in the bind mount that appear on the host owned by root.

| Symptom | Likely cause |
|---|---|
| `Permission denied` when the app writes into a mounted folder | UID mismatch |
| Files created by the container are owned by `root` on the host | Container runs as root |
| `chown: changing ownership ... Operation not permitted` | The filesystem (NFS, Windows share) does not support chown |
| Postgres/MySQL fails: "data directory has wrong ownership/permissions" | Volume or mount not owned by the DB user |

### Fixes

1. **Run the container as the host user** (development bind mounts):

   ```bash
   docker run --rm --user "$(id -u):$(id -g)" -v "$(pwd)":/app -w /app python:3.13-slim python gen.py
   ```
   - `--user UID:GID` runs the process as that numeric user and group. `$(id -u)` and `$(id -g)` print your IDs.
   - Compose: `user: "${UID:-1000}:${GID:-1000}"`.

2. **Set ownership in the image** for named volumes. An empty named volume inherits the ownership and permissions of the image's directory at that path on first mount:

   ```dockerfile
   RUN mkdir -p /data && chown 10001:10001 /data
   USER 10001:10001
   ```
   Mount a named volume at `/data` at run time (`-v mydata:/data`). Do the `chown` before the volume is first populated; a populated volume keeps its ownership.

3. **One-off fix with a helper container** for an existing volume:

   ```bash
   docker run --rm -v myvol:/data alpine chown -R 10001:10001 /data
   ```
   The `alpine` container runs `chown` as root against the volume mounted at `/data`.

4. **Compose init service**: a service that chowns a volume and exits, with the app depending on it via `condition: service_completed_successfully` ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)).

5. **SELinux hosts (Fedora, RHEL, CentOS):** bind mounts can fail even when UIDs match because SELinux labels block access. Add `:z` (shared between containers) or `:Z` (private to this container): `-v ./data:/data:Z`. Do not use `:Z` on system directories such as `/home` or `/etc`; it relabels them and can break the host.

6. **User namespace remapping / rootless Docker:** container root maps to an unprivileged host UID, so bind-mount ownership appears shifted (for example `100000`).

Docker Desktop (Windows/macOS) translates ownership for bind mounts, so UID issues are less common there than on Linux.

---

## A8. Windows and WSL path pitfalls

Docker Desktop on Windows runs the engine in a Linux VM. With the default **WSL 2** backend (Windows Subsystem for Linux 2, a real Linux kernel running alongside Windows) there are two filesystems:

- **Windows filesystem** (`C:\Users\you\...`), visible in WSL as `/mnt/c/Users/you/...`
- **WSL Linux filesystem** (ext4, for example `/home/you/...`), visible from Windows as `\\wsl$\Ubuntu\home\you\...` (or `\\wsl.localhost\...`)

Crossing between them goes through a translation layer that is much slower for many small files and does not carry Linux semantics exactly.

### Pitfall 1: path syntax differs per shell

| Shell | Mount the current folder |
|---|---|
| PowerShell | `-v "${PWD}:/app"` (or `-v "${PWD}\data:/data"`) |
| Command Prompt | `-v "%cd%:/app"` |
| Git Bash | `-v "$(pwd -W):/app"` or `-v "$(pwd):/app"` (see pitfall 2) |
| WSL | `-v "$(pwd):/app"` |

Windows drive paths contain a colon (`C:\...`), which conflicts with the `-v` separator. Docker handles `C:\path:/container` specially, but `--mount` avoids the issue: `--mount type=bind,source="${PWD}",target=/app`.

### Pitfall 2: Git Bash rewrites paths

Git Bash (MSYS) converts anything that looks like a Unix path into a Windows path before passing it to programs. `docker run -v /app:/app ...` or `docker exec c ls /etc` can become `C:/Program Files/Git/app`. Workarounds:

```bash
MSYS_NO_PATHCONV=1 docker exec mycontainer ls /etc      # disable conversion for one command
docker exec mycontainer ls //etc                         # a double leading slash also works
```

PowerShell and WSL do not do this.

### Pitfall 3: performance across the boundary

With a project under `C:\...`, operations such as `npm install`, `git status` and file watchers can be 10x or more slower than on the Linux filesystem. Keep the project inside the WSL filesystem (`~/projects/my-app`), run Docker commands from there, and open it in VS Code with the WSL extension.

### Pitfall 4: file-change events do not always propagate

Tools that use `inotify` (Linux file-change notifications) may not see changes made from Windows on a `/mnt/c` bind mount, so hot reload does not trigger. Fixes: keep code in the WSL filesystem; enable polling (`WATCHPACK_POLLING=true` for webpack/Next.js, `CHOKIDAR_USEPOLLING=1`, `watchfiles` configuration for `uvicorn --reload`); or use Compose `watch` ([Chapter 6](06-docker-compose.md#docker-compose-watch-hot-reload-for-development)), which syncs files instead of using a bind mount.

### Pitfall 5: line endings

Git on Windows may convert text files to CRLF. A shell script with CRLF fails in a Linux container with `exec ./entrypoint.sh: no such file or directory` (the interpreter becomes `/bin/sh\r`). Fix: add `*.sh text eol=lf` to `.gitattributes` and re-checkout, or run `git config core.autocrlf input`.

### Pitfall 6: case sensitivity and permissions

Linux paths are case-sensitive (`App.py` and `app.py` differ); Windows by default is not. Code that works on Windows can fail in the container with "file not found". Files on a Windows mount typically appear as `rwxrwxrwx`, so programs that need strict modes (an SSH key requiring `0600`) can fail. Copy such files into the image or a volume.

### Pitfall 7: where are my volumes?

Named volumes are inside the Docker Desktop VM's disk image, not under `C:\`. Inspect them with a throwaway container (A9) or the Volumes tab in Docker Desktop, which has a file browser.

---

## A9. Backup and restore of a volume (tar method)

A volume is a directory, so a backup is a temporary helper container that mounts both the volume and a host folder and archives one into the other with `tar`.

### Look inside a volume

```bash
docker run --rm -v pgdata:/data alpine ls -la /data
```

### Backup

```bash
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)":/backup \
  alpine tar czf /backup/pgdata-backup.tar.gz -C /data .
```

| Part | Meaning |
|---|---|
| `-v pgdata:/data:ro` | The volume to back up, mounted read-only. |
| `-v "$(pwd)":/backup` | Bind-mount the current host folder as the destination. |
| `tar czf /backup/pgdata-backup.tar.gz` | `c` create, `z` gzip, `f` write to this file. |
| `-C /data .` | Change into `/data` and archive everything. Paths in the archive stay relative. |

The archive appears in the current host directory. In PowerShell replace `"$(pwd)"` with `"${PWD}"`.

### Restore

```bash
docker volume create pgdata-restored
docker run --rm \
  -v pgdata-restored:/data \
  -v "$(pwd)":/backup:ro \
  alpine sh -c "cd /data && tar xzf /backup/pgdata-backup.tar.gz"
```

`x` extracts, `z` gunzips, `f` reads from this file. `sh -c "cd /data && tar ..."` runs both steps in one container. Restore into a fresh volume and test it before touching production.

### Copy data between volumes

```bash
docker run --rm -v old_vol:/from:ro -v new_vol:/to alpine sh -c "cp -a /from/. /to/"
```

`cp -a` preserves ownership, permissions and timestamps. `/from/.` copies the directory contents including hidden files.

### Databases and consistency

Copying a running database's files can produce an inconsistent backup. Use the database's own tooling:

```bash
docker exec db pg_dump -U postgres mydb > mydb.sql          # logical backup (PostgreSQL)
docker exec -i db psql -U postgres mydb < mydb.sql          # restore
```

Alternatively stop the container before the tar backup. Back up regularly, store backups off the machine, and test restores ([Chapter 11](11-production-and-enterprise.md) covers backup strategy).

---

## A10. Storage quick decision guide

```
 Data the app must keep (database, uploads, model cache)?
        -> Named volume.
 Edit host files and see them live in the container (dev)?
        -> Bind mount (or Compose watch).
 Config file the container reads but never changes?
        -> Bind mount with :ro, or Compose configs/secrets.
 Temporary, sensitive or performance-critical scratch data?
        -> tmpfs.
 Throwaway output of a one-off job?
        -> Writable layer (or bind mount the output folder).
```

### Common mistakes (storage)

| Symptom | Cause | Fix |
|---|---|---|
| Data lost after `docker rm` or redeploy | Data in the writable layer | Named volume |
| Unknown random-named volume holds the data | Anonymous volume (`-v /var/lib/mysql`) | Named volume |
| Data in an unexpected anonymous volume | Wrong mount path (Postgres 18 note) | Use the image's documented data path |
| Database volume deleted | `docker volume prune -a` or `docker system prune --volumes` while the database was stopped | Do not prune volumes on hosts with stopped database containers |
| Image dependencies missing in container | Bind mount over `node_modules` or a virtualenv | Add a volume over that subpath |
| Root-owned host directory appears | `-v ./dir:/path` auto-created a missing directory | Use `--mount`, or create the directory first |
| Data lands in the wrong place | `-v data:/x` (named volume) confused with `-v ./data:/x` (folder) | Use `./` or an absolute path for folders |
| Corrupt backup | Live database copied as files | `pg_dump`, or stop the container |
| Slow bind mounts | Project under `C:\` mounted into WSL containers | Keep the project in the WSL filesystem |
| Volumes deleted unexpectedly | `docker compose down -v` | Omit `-v` ([Chapter 6](06-docker-compose.md)) |

---

# Part B: Networking

## B1. The big picture

Each container gets its own **network namespace**: a private network stack (interfaces, IP address, routing table, firewall rules) isolated from the host and other containers. Docker connects containers with virtual networks.

Terms:

- **IP address**: numeric address of a machine on a network (for example `172.18.0.3`).
- **Port**: numbered endpoint (0-65535) on a machine; a program listens on a port (web servers typically 80/443).
- **Subnet**: a range of IP addresses in one network (`172.18.0.0/16`: the first 16 bits are the network part, about 65,000 addresses).
- **Gateway**: the address traffic passes through to leave a network.
- **NAT** (Network Address Translation): rewrites addresses in packets so private addresses can reach the outside and incoming traffic can be forwarded to a private address. Docker uses NAT for published ports and outbound traffic.
- **DNS** (Domain Name System): translates names (`google.com`, `db`) to IP addresses.
- **Bridge**: a virtual network switch.
- **veth pair**: a virtual ethernet cable with two ends; one end in the container, the other on the bridge.

### Network drivers

| Driver | What it does | Typical use |
|---|---|---|
| `bridge` | Private virtual network on one host; containers on it can talk to each other | Default; most use cases |
| `host` | Container shares the host's network stack | Maximum performance, many ports; Linux-focused |
| `none` | No network (loopback only) | Isolated batch jobs, lockdown |
| `overlay` | Virtual network across multiple Docker hosts (VXLAN tunnels) | Docker Swarm multi-host |
| `macvlan` | Each container gets its own MAC and IP on the physical LAN | Legacy apps that must appear as physical machines |
| `ipvlan` | Like macvlan, but containers share the host's MAC address | Environments limiting MACs per port |

```bash
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
d3841588ee9f   bridge    bridge    local
8218b77415d0   host      host      local
436e8a16a4ef   none      null      local
```

These three exist on every installation. `null` is the driver behind `none`.

---

## B2. Default bridge vs user-defined bridge

### Default bridge

A container started without a network joins the default network `bridge` (the `docker0` interface on Linux). Containers get IPs from a private range (commonly `172.17.0.0/16`) and can reach each other by IP only, with no name resolution. All containers share the one network, and ports must be published to reach them from outside.

```bash
docker run -d --name a1 alpine sleep 3600
docker run -d --name a2 alpine sleep 3600
docker exec a1 ping -c 1 a2
```

Result: `ping: bad address 'a2'`. The legacy `--link` flag is deprecated; do not use it.

### User-defined bridge (recommended)

```bash
docker network create appnet
docker run -d --name b1 --network appnet alpine sleep 3600
docker run -d --name b2 --network appnet alpine sleep 3600
docker exec b1 ping -c 2 b2
```

| Part | Meaning |
|---|---|
| `docker network create appnet` | Create a bridge network named `appnet` (`--driver bridge` is the default). |
| `--network appnet` | Attach the container to it. |

Output:

```
PING b2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.095 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.106 ms
```

Docker runs an **embedded DNS server** in each user-defined network (at `127.0.0.11` inside containers) that resolves container names to current IPs. This is **service discovery**: finding services by name instead of by IP, which changes whenever a container is recreated.

| Feature | Default `bridge` | User-defined bridge |
|---|---|---|
| Name-based DNS between containers | No | Yes |
| Isolation between apps | All share one network | Only members of the same network communicate |
| Attach/detach running containers | Must stop container | Live (`docker network connect`/`disconnect`) |
| Custom subnet/gateway/options | Limited (daemon-wide) | Per network |
| Network aliases | No | Yes (`--network-alias`) |
| Recommended | Rarely | Yes |

Compose ([Chapter 6](06-docker-compose.md)) creates a user-defined network per project, so services reach each other by service name (`db`, `redis`) with no configuration.

Clean up: `docker rm -f a1 a2 b1 b2 && docker network rm appnet`.

### Managing networks

```bash
docker network create --driver bridge --subnet 10.50.0.0/24 --gateway 10.50.0.1 mynet
docker network inspect mynet
docker network connect mynet some-container               # attach a running container
docker network connect --alias api mynet some-container   # also add DNS alias "api"
docker network disconnect mynet some-container
docker network rm mynet                                   # only if no containers are attached
docker network prune                                      # remove ALL unused networks
```

- `--subnet` sets the IP range (avoid clashes with a corporate VPN; see B10).
- `--gateway` sets the gateway address.
- A container can join several networks. This builds tiers: a `frontend` network (proxy + app) and a `backend` network (app + database), so the proxy cannot reach the database.

### Network aliases

```bash
docker run -d --name pg --network appnet --network-alias database --network-alias db postgres:18
```

Containers on `appnet` can reach this one as `pg`, `database` or `db`. If several containers share an alias, Docker DNS returns all their IPs in rotating order (round-robin, with no health checking).

### Internal networks

```bash
docker network create --internal backend-only
```

`--internal` creates a network with no outbound access and no published ports, so containers on it can only talk to each other. Use it for databases that must not reach the internet.

---

## B3. Port publishing

A container on a bridge network has a private IP that is not reachable from outside the host. **Publishing** a port makes Docker forward traffic from a host port to a container port.

```bash
docker run -d --name web -p 8080:80 nginx
```

`-p HOST_PORT:CONTAINER_PORT` forwards host port 8080 to container port 80. Open `http://localhost:8080`.

```
   Browser --> host:8080 ==[Docker NAT/proxy]==> container:80 (nginx)
```

| Syntax | Meaning |
|---|---|
| `-p 8080:80` | Host 8080 to container 80, on all host interfaces (`0.0.0.0` and `::`) |
| `-p 127.0.0.1:8080:80` | Reachable only from the host. Recommended for databases and dev tools |
| `-p 8080:80/udp` | UDP instead of TCP (default TCP) |
| `-p 8080:80/tcp -p 8080:80/udp` | Both |
| `-p 8000-8010:8000-8010` | A range |
| `-p 80` | Random host port (see with `docker port`) |
| `-P` (capital) | Publish all `EXPOSE`d ports on random host ports |
| `-p [::1]:8080:80` | IPv6 loopback |

`docker port web` shows mappings, for example `80/tcp -> 0.0.0.0:8080`.

### Exposing vs publishing

| | **EXPOSE** (`EXPOSE 80` in Dockerfile, `expose:` in Compose) | **Publish** (`-p`, `ports:`) |
|---|---|---|
| What it does | Documents that the container listens on that port | Opens a path from host to container |
| Reachable from other containers on the same network | They can reach any listening port regardless of EXPOSE | n/a |
| Reachable from host/outside | No | Yes |
| Needed for container-to-container | No | No: containers on the same user-defined network connect directly to the container port |

Do not publish ports that are not needed on the host. A production database should not have `-p 5432:5432`; only the app container on the same network needs it. Publishing to `0.0.0.0` exposes the port to the whole network and, with a public IP, the internet.

Containers talking over a Docker network use the container port (80), never the host port (8080). The mapping `8080:80` exists only at the host boundary.

### How publishing works

On Linux, Docker adds firewall rules (`iptables` or `nftables`, see B9) that DNAT (rewrite the destination of) packets arriving at the host port to the container's IP and port. In some cases (traffic from the host itself, IPv6 to IPv4) it uses a userland helper, `docker-proxy`, which is why `docker-proxy` processes appear in `ps` output.

---

## B4. `localhost` inside containers

Inside a container, `localhost` (127.0.0.1) refers to that container itself, not the host or another container. Each container has its own loopback interface.

```
 +------------- HOST ------------------------------------------+
 |  localhost:5432  <-- a Postgres installed on the host        |
 |                                                              |
 |   +-- container "api" ----+     +-- container "db" ------+   |
 |   | localhost = ITSELF    |     | localhost = ITSELF     |   |
 |   | (nothing on 5432 here)|     | Postgres on 5432       |   |
 |   +-----------------------+     +------------------------+   |
 +--------------------------------------------------------------+
```

Symptom: a container with `DATABASE_URL=postgresql://user:pw@localhost:5432/app` fails with `connection refused`, although it worked on the host where the database really was on localhost.

| From | To | Use | Notes |
|---|---|---|---|
| Container A | Container B (same user-defined network) | `B-name:PORT` | For example `postgresql://db:5432/app`. Use the container port, not a published host port |
| Container | Service on the host (not in Docker) | `host.docker.internal` | See B5 |
| Host (browser/curl) | Container | `localhost:PUBLISHED_PORT` | Requires `-p` |
| Container A | Container B on different networks | Not possible until one is attached to both networks | By design |
| Another machine | Container | `host-ip:PUBLISHED_PORT` | Requires `-p` bound to a non-loopback address |

### The reverse problem: server bound to `127.0.0.1` inside the container

If the server in the container listens on `127.0.0.1:8000` (the default for many dev servers), `-p 8000:8000` publishes the port but connections arrive on the container's network interface (for example `172.17.0.2`), not loopback, and the app does not answer. The host sees an empty reply or connection reset.

Fix: listen on `0.0.0.0` (all interfaces) inside the container.

| Framework | How |
|---|---|
| Uvicorn/FastAPI | `uvicorn app.main:app --host 0.0.0.0` |
| Flask | `flask run --host=0.0.0.0` |
| Django | `python manage.py runserver 0.0.0.0:8000` |
| Node/Express | `app.listen(3000, '0.0.0.0')` |
| Vite dev server | `vite --host` |
| Spring Boot | Listens on all interfaces by default |

The host-side restriction `-p 127.0.0.1:8000:8000` is unrelated and fine.

---

## B5. Container to host: `host.docker.internal`

Use this when a container needs a service running on the host (a native database, dev API, Ollama).

**Docker Desktop (Windows/macOS):** the DNS name `host.docker.internal` resolves to the host automatically.

```bash
docker run --rm curlimages/curl -s http://host.docker.internal:11434/
```

**Linux (Docker Engine):** the name is not defined by default. Add it with the special value `host-gateway` (Docker Engine 20.10+):

```bash
docker run --rm --add-host=host.docker.internal:host-gateway curlimages/curl -s http://host.docker.internal:11434/
```

- `--add-host name:ip` adds an entry to the container's `/etc/hosts`.
- `host-gateway` is replaced with the host's gateway IP on the bridge.

Compose:

```yaml
services:
  api:
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

Notes:

- On Linux, the host service must listen on an address reachable from the bridge (`0.0.0.0` or the bridge gateway IP such as `172.17.0.1`). A service bound only to host `127.0.0.1` is not reachable.
- On Docker Desktop, `host.docker.internal` connections are proxied to the host, so a service bound to host loopback generally still works.
- Host firewalls (including Windows Defender Firewall) can block the connection.
- Alternative: `--network host` (B6.1).

---

## B6. The other drivers

### B6.1 `host`: share the host's network stack

```bash
docker run --rm --network host nginx
```

The container has no network isolation: it uses the host's IP and ports, and `localhost` inside it is the host's localhost.

- `-p`/`-P` are ignored (Docker prints a warning). If nginx listens on 80, it uses host port 80.
- Two containers cannot bind the same port.
- Slightly faster (no NAT/proxy); suits apps needing thousands of ports or host interfaces (monitoring agents, network tools).
- Less secure (no isolation).
- **Platform (from Docker's docs):** host networking is a Linux-host feature. On Docker Desktop it is an opt-in setting in 4.34 and later (Settings -> Resources -> Network -> "Enable host networking", requires sign-in) with limitations: TCP/UDP only, cannot bind the host's own interfaces from the VM perspective, and not compatible with Enhanced Container Isolation. It does not apply to Windows containers. Without that setting, `--network host` attaches to the VM's network, not the laptop's.

### B6.2 `none`: no networking

```bash
docker run --rm --network none alpine ip addr
```

Only a loopback interface (`lo`) exists. Use for jobs that must never touch the network, or for running untrusted code.

### B6.3 `overlay`: multi-host networks

An **overlay network** connects containers on different Docker hosts as if on one network, by wrapping traffic in VXLAN tunnels. It requires **Docker Swarm** (Docker's built-in clustering mode). On Kubernetes, the cluster's **CNI** plugin does this job. Encryption is optional (`--opt encrypted`).

```bash
docker swarm init
docker network create --driver overlay --attachable my-overlay
```

`--attachable` lets standalone containers join, not only Swarm services. See [Chapter 6](06-docker-compose.md#compose-vs-swarm-vs-kubernetes) and [Chapter 11](11-production-and-enterprise.md).

### B6.4 `macvlan`: containers on the physical LAN

With **macvlan**, each container gets its own **MAC address** (hardware address of a network card) and an IP on the physical network. Other machines see a separate device. No NAT, no port publishing.

```bash
docker network create -d macvlan \
  --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 lan
docker run -d --network lan --ip 192.168.1.50 nginx
```

| Part | Meaning |
|---|---|
| `-d macvlan` | Driver. |
| `--subnet` / `--gateway` | Must match the real LAN. |
| `-o parent=eth0` | Host physical interface to attach to. |
| `--ip` | Fixed address; choose one outside the DHCP range. |

Caveats:

- The host cannot talk to its own macvlan containers by default (Linux kernel rule; workaround: create a macvlan sub-interface on the host).
- The switch or Wi-Fi must allow multiple MAC addresses per port. Most Wi-Fi adapters do not, so macvlan generally fails on Wi-Fi.
- Cloud providers usually block it.
- On Docker Desktop, containers run inside a VM, so macvlan generally does not give LAN-level access. Unverified: check Docker's current docs if needed.

Use it for legacy software that needs a real LAN address, network appliances, or IoT bridging.

### B6.5 `ipvlan`

Like macvlan, but containers share the host's MAC address and have distinct IPs. Useful where switches limit MAC addresses per port (and often works on Wi-Fi). Modes: **L2** (containers appear on the same Layer-2 LAN) and **L3** (the host routes between subnets, no broadcast; suits routed/BGP environments).

```bash
docker network create -d ipvlan --subnet=192.168.1.0/24 --gateway=192.168.1.1 \
  -o parent=eth0 -o ipvlan_mode=l2 ipv
```

### Which driver

```
 Single host, normal app?                       -> user-defined bridge (default driver)
 Host network performance / many ports?         -> host (Linux)
 Total isolation?                               -> none
 Multi-host with Swarm?                         -> overlay
 Container needs its own LAN IP?                -> macvlan / ipvlan
```

---

## B7. Inspecting networks

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

`IPAM` (IP Address Management) holds the subnet and gateway. `Containers` lists attached containers and their IPs.

A container's IP:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' b1
```

`-f` (`--format`) applies a Go template to extract one field. The command prints one IP per attached network.

From inside the container:

```bash
docker exec b1 cat /etc/resolv.conf     # "nameserver 127.0.0.11" on user-defined networks
docker exec b1 cat /etc/hosts           # the container's hosts file
docker exec b1 nslookup b2              # DNS lookup of another container
docker exec b1 ip addr                  # interfaces and IPs (if 'ip' exists)
docker exec b1 wget -qO- http://b2:80   # HTTP test (alpine has wget)
```

Slim images often lack these tools; use `netshoot` (B11).

---

## B8. IPv6 note

Docker networks are IPv4-only by default. IPv6 support has changed between Engine versions. The following is the general shape; check the documentation for your version.

```bash
docker network create --ipv6 --subnet 2001:db8:1::/64 v6net
```

- `--ipv6` enables IPv6 on this user-defined network. `2001:db8::/32` is reserved for documentation; use a real prefix or a ULA (`fd00::/8`) range.
- For the default bridge and daemon-wide behaviour, edit `/etc/docker/daemon.json` (`"ipv6": true` and related keys such as `fixed-cidr-v6`) and restart the daemon.
- Published ports bind both IPv4 and IPv6 host addresses by default (`0.0.0.0` and `::`). Use `127.0.0.1:` or `[::1]:` prefixes to restrict.
- An app listening only on IPv4 `0.0.0.0` does not answer IPv6 connections, and `localhost` may resolve to `::1` first. If `localhost` works but `127.0.0.1` does not (or the reverse), suspect an IPv4/IPv6 mismatch.
- Unverified: specifics for every Engine version, including which defaults changed in the latest releases. See `docs.docker.com/engine/daemon/ipv6/`.

---

## B9. Firewalls and why Docker "bypasses ufw"

**ufw** ("Uncomplicated Firewall") is a front-end for the Linux firewall. On a Linux server:

```bash
sudo ufw deny 8080
docker run -d -p 8080:80 nginx
```

Port 8080 is still reachable from the internet; the ufw rule is effectively ignored.

### Why

Linux packet filtering is done by **netfilter**, configured with `iptables` or `nftables`. Packets pass through chains of rules. `ufw` adds rules to the `INPUT`/`OUTPUT` chains, which handle traffic to or from the host itself. Published container ports are not addressed to a host process. Docker uses the `nat` table to rewrite the destination to the container's IP, and the packet then traverses the **FORWARD** chain, which ufw's rules do not cover. Docker's documentation states that Docker and ufw are incompatible in this respect.

```
 Internet --> [nat PREROUTING: Docker DNAT to container] --> [FORWARD chain] --> container
                                                   (ufw's INPUT rules never see this)
```

Docker also enables IP forwarding and sets the default forwarding policy to drop, adding its own allow rules for published ports. Docker Engine can use an `nftables` firewall backend, selectable with the `firewall-backend` daemon option. Unverified: details and maturity vary by version; verify before relying on it. On `firewalld` systems Docker creates a `docker` zone.

### What to do

1. **Do not publish ports that are not needed.** If only the app container needs the database, do not publish it.
2. **Bind to loopback** for host-only access: `-p 127.0.0.1:5432:5432`. Only local processes (or an SSH tunnel) can connect. This is the simplest reliable fix.
3. **Put one reverse proxy in front** and publish only 80/443 (B12).
4. **Use the `DOCKER-USER` chain.** Docker evaluates it first and never overwrites it, so it is the supported place for custom filtering of published-container traffic. Example (allow only one subnet to reach published ports on `eth0`):

   ```bash
   sudo iptables -I DOCKER-USER -i eth0 ! -s 203.0.113.0/24 -j DROP
   ```
   - `-I DOCKER-USER`: insert at the top of the chain.
   - `-i eth0`: match traffic arriving on the external interface.
   - `! -s 203.0.113.0/24`: match sources outside that subnet.
   - `-j DROP`: discard silently.
   - Test carefully to avoid locking yourself out. `DOCKER-USER` sees traffic after DNAT; to match the original published port use `-m conntrack --ctorigdstport`.
5. **Use the cloud firewall** (AWS Security Groups, Azure NSG, GCP firewall rules). It filters before packets reach the host, so Docker cannot bypass it.
6. **Do not set `"iptables": false`** in `daemon.json`. Docker's docs warn it likely breaks container networking (no outbound masquerade, port publishing stops working), and replicating Docker's rules by hand is complex.
7. **Never expose the Docker daemon's TCP socket** (port 2375) without TLS. It grants remote root on the host ([Chapter 10](10-security.md)).

`-p 5432:5432` binds `0.0.0.0` by default; a database published to the internet is a common real-world breach.

---

## B10. Other networking gotchas

### DNS inside containers

- User-defined networks use Docker's embedded DNS (`127.0.0.11`), which forwards external lookups to the host's DNS servers.
- Corporate networks and VPNs may need specific resolvers: per container (`--dns 10.0.0.2`, `--dns-search corp.example`) or daemon-wide in `daemon.json` (`"dns": ["10.0.0.2"]`).
- Symptom: `curl: (6) Could not resolve host` inside a container while it works on the host. Test with `docker run --rm alpine nslookup example.com`.

### Subnet conflicts with VPNs

Docker's default networks use private ranges (`172.17.0.0/16` onward through `172.x`, also `192.168.x`). If a corporate VPN uses an overlapping range, traffic to those addresses goes to a container network and times out. Set ranges per network (`--subnet`) or daemon-wide with `default-address-pools` in `daemon.json`:

```json
{
  "default-address-pools": [
    { "base": "10.200.0.0/16", "size": 24 }
  ]
}
```

`base` is the pool; `size: 24` gives each new network a `/24` slice. Restart the daemon after editing. On Docker Desktop, use Settings -> Resources -> Network.

### MTU problems

Over some VPNs or cloud overlays the maximum packet size (MTU) is smaller than Docker assumes, so large downloads hang while small requests work. Set `"mtu": 1400` (or the correct value) in `daemon.json`, or `-o com.docker.network.driver.mtu=1400` on a network.

### Container ports vs host ports in URLs

A container calling another container uses the internal port. If B is published as `-p 9000:8080`, A calls `http://b:8080`.

### Ephemeral IPs

Container IPs change when a container is recreated. Use names.

### Across Compose projects

Each Compose project has its own network by default. Connect two projects with a shared external network ([Chapter 6](06-docker-compose.md#networks)).

---

## B11. Debugging connectivity with `nicolaka/netshoot`

Application images are slim and lack `ping`, `dig`, `curl` and `tcpdump`. **netshoot** is a community-maintained troubleshooting image containing `curl`, `dig`, `nslookup`, `ping`, `traceroute`, `tcpdump`, `ss`, `ip`, `nmap`, `iperf3`, `netcat` and more.

### Method 1: join a Docker network

```bash
docker run -it --rm --network appnet nicolaka/netshoot
```

Inside the shell:

```bash
nslookup db                       # does the name resolve? shows the container IP
ping -c 2 db                      # basic reachability
nc -zv db 5432                    # is the port open? ("succeeded!" or "Connection refused")
curl -v http://api:8000/health    # test the HTTP service
dig +short api                    # DNS answer only
```

`nc -zv host port`: `nc` is netcat; `-z` scans without sending data; `-v` is verbose.

### Method 2: share another container's network namespace

```bash
docker run -it --rm --network container:my-api nicolaka/netshoot
```

`--network container:my-api` uses the same network stack as `my-api`: same IP, same `localhost`, same open ports. Use it to see what `my-api` listens on and to capture packets:

```bash
ss -tlnp                          # listening TCP ports in that namespace
tcpdump -i eth0 -n port 8000      # watch traffic hitting the app
```

This works even when the target is a distroless/scratch container with no tools.

### Method 3: host network

```bash
docker run -it --rm --network host nicolaka/netshoot
```

Meaningful only with true host networking (Linux). On Docker Desktop this joins the VM's network.

### Debugging checklist

1. Is the container running? `docker ps`; if not, `docker logs <name>`.
2. Is the app listening on `0.0.0.0` on the right port? `docker exec <c> ss -tlnp` (or netshoot Method 2).
3. Are both containers on the same user-defined network? `docker network inspect`.
4. Does DNS resolve the name? `nslookup <service>` from netshoot.
5. Is the port reachable? `nc -zv <service> <container-port>`.
6. Is the container port used, not the host-mapped one?
7. Is the host port published and free? `docker port <c>`; on the host `ss -tlnp | grep 8080` (Linux) or `netstat -ano | findstr :8080` (Windows).
8. Is a firewall in the way (host firewall, cloud security group, corporate proxy)?
9. Is the dependency up yet? ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)).

---

## B12. Reverse proxies in front

A **reverse proxy** sits in front of the apps, receives incoming requests and forwards each to the right backend. It is the standard way to expose several containerized services through ports 80/443. It also:

- terminates TLS/HTTPS (certificates in one place)
- routes by hostname or path (`api.example.com` to the api container; `/` to the frontend)
- load-balances across replicas
- adds caching, rate limiting, compression and security headers
- removes the need to publish backend ports, reducing attack surface (and avoiding the ufw problem)

```
                      published: 80, 443 only
 Internet --> [ reverse proxy container ] --+--> api   (no published port)
                                             +--> web   (no published port)
                                             +--> admin (no published port)
              all on a private user-defined network; DB on a second, internal network
```

| Proxy | Style | Notes |
|---|---|---|
| nginx | Config files | Ubiquitous, fast, flexible |
| Caddy | Simple config; automatic HTTPS via Let's Encrypt | Easiest TLS setup |
| Traefik | Discovers containers through Docker labels | Dynamic; suits Compose/Swarm |
| HAProxy, Envoy | High-performance / service-mesh style | Advanced load balancing |

### Example: nginx in front of a FastAPI app

`compose.yaml` (syntax in [Chapter 6](06-docker-compose.md)):

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "80:80"               # the only published port
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

- `proxy_pass http://api:8000;` forwards requests to service `api`, container port 8000.
- `proxy_set_header ...` passes the original host, client IP and scheme; otherwise the app sees only the proxy's IP. The app must trust these headers (uvicorn: `--proxy-headers --forwarded-allow-ips="*"`, only when the proxy is the sole way in).
- nginx resolves `api` once at startup. If `api` is not resolvable yet, nginx fails with `host not found in upstream`. Fix with start order and healthchecks ([Chapter 6](06-docker-compose.md#depends_on-and-startup-order)), or use a variable with Docker's resolver so it re-resolves: `resolver 127.0.0.11 valid=10s; set $upstream http://api:8000; proxy_pass $upstream;`.
- If the backend is recreated with a new IP, a static `proxy_pass` hostname may keep the stale IP until reload (`docker compose exec proxy nginx -s reload`). The variable-plus-resolver form avoids this.

### Example: Caddy with automatic HTTPS

```yaml
services:
  caddy:
    image: caddy:2
    ports: ["80:80", "443:443"]
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data         # stores certificates; persist it
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

Caddy obtains and renews a Let's Encrypt certificate for `api.example.com` automatically, provided public DNS points at the host and ports 80/443 are open.

---

## B13. Common mistakes (networking)

| Symptom | Cause | Fix |
|---|---|---|
| `connection refused` to another container | `localhost` used | Use the service name (B4) |
| Published port does not answer | App listens on `127.0.0.1` inside the container | Listen on `0.0.0.0` (B4) |
| Container-to-container call fails | Host-mapped port used | Use the container port |
| Names do not resolve | Default bridge in use | User-defined network (Compose does this) |
| Database reachable from the internet | Published to `0.0.0.0` | `127.0.0.1:` or do not publish (B3, B9) |
| Blocked port still reachable | ufw does not filter published ports | See B9 |
| `bind: address already in use` | Another process or container holds the host port | Use another host port |
| Port not reachable after `EXPOSE` | `EXPOSE` does not publish | `-p` / `ports:` (B3) |
| `--network host` does not give laptop networking | Docker Desktop attaches to the VM's network | See B6.1 |
| Random timeouts to corporate addresses | Subnet clash with VPN | Change address pools (B10) |
| External-network setup breaks | `docker network prune` / `docker system prune` removed unused networks | Keep a container attached or recreate the network |
| nginx errors after backend recreate | Stale upstream IP | Reload nginx or use the resolver variable form (B12) |

---

## Summary

- Use named volumes for data, bind mounts for development source and read-only config, tmpfs for RAM-only scratch. Prefer `--mount`.
- Back up volumes with a helper container and `tar`; use native tools (`pg_dump`) for databases.
- Use user-defined bridge networks for DNS by container name. `localhost` in a container is the container itself; bind servers to `0.0.0.0`.
- Publishing is separate from exposing. Bind sensitive ports to `127.0.0.1`. Published ports bypass ufw; use loopback binding, a reverse proxy, `DOCKER-USER` rules or a cloud firewall.
- Debug with `nicolaka/netshoot`, preferably with `--network container:<name>`.

**Next:** [Chapter 6: Docker Compose](06-docker-compose.md).
