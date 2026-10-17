# Lesson 32: Docker Volumes, Networks & Security

**Module:** Docker – Containerization
**Duration:** 120-150 min
**Prerequisites:** Lesson 30 (Container Fundamentals & Docker Essentials), Lesson 31 (Dockerfile Best Practices & Multi-stage Builds). Student can run containers, map ports, write a basic Dockerfile.

## Learning Objectives

By end of lesson student can:
- Explain why container data is ephemeral and pick the right storage type (bind mount, named volume, tmpfs) for a given workload
- Create, inspect, back up, restore, share and prune Docker volumes
- Explain the bridge, host, none and overlay network drivers and when each applies
- Build a user-defined bridge network and use container DNS to connect containers by name
- Publish ports, inspect container IPs, and debug container connectivity
- Harden a container: non-root user, read-only root filesystem, dropped capabilities, no-new-privileges
- Limit CPU, memory and PIDs for a container and observe the effect with `docker stats`
- Scan an image for vulnerabilities with Trivy, filter by severity and use exit codes to fail a pipeline

## Topics

- Volume types: bind mounts (`-v /host:/container`), named volumes, tmpfs; when to use each
- Volume commands: `docker volume create/ls/inspect/rm/prune`; sharing volumes between containers
- Docker networking deep dive: bridge, host, none, overlay; custom networks; container DNS
- Docker security: non-root `USER`, read-only filesystem (`--read-only`), capabilities (`--cap-drop/--cap-add`)
- Resource limits: `--memory`, `--cpus`, `--pids-limit`; `docker stats`; Trivy image scan intro

## Concepts

### Why containers need external storage

A container is a thin writable layer on top of read-only image layers (union filesystem, overlay2 — lesson 30). Everything the process writes goes to that writable layer. When the container is removed (`docker rm`), the writable layer is deleted with it. Data written inside the container is therefore **ephemeral**.

```
Container writable layer   <- lost on docker rm
-----------------------------
Image layer 3 (read-only)
Image layer 2 (read-only)
Image layer 1 (read-only)
```

Problems with writing into the container layer:
- Data lost when container is deleted/recreated (every image upgrade = new container).
- Writable layer is slower than a native filesystem (copy-on-write through the storage driver).
- Data is hard to share with other containers or to back up from the host.

Solution: mount storage **outside** the container layer into a path inside the container. Docker offers three mount types.

```
              +------------------- Docker host -------------------+
              |                                                   |
 Container -- | bind mount   -> any host path (/srv/site)         |
 Container -- | named volume -> /var/lib/docker/volumes/<name>    |
 Container -- | tmpfs        -> host RAM (never touches disk)     |
              +---------------------------------------------------+
```

### Bind mounts

A bind mount maps an **existing host path** (file or directory) into the container. Host and container see the same files in real time.

- Syntax: `-v /host/path:/container/path[:ro]` or the explicit `--mount type=bind,source=/host/path,target=/container/path[,readonly]`.
- Host path must exist with `-v`? Docker **creates it as a root-owned directory** if missing with `-v` (surprise!). With `--mount` the command **fails** if the path does not exist — safer.
- Good for: development (live code reload), injecting config files (`nginx.conf`), reading host data (logs, `/var/run/docker.sock` — dangerous, see pitfalls).
- Bad for: production data (depends on host layout, permissions, SELinux labels), portability.
- Permissions: files keep host UID/GID. If container runs as UID 1000 and host dir is owned by root, writes fail with `Permission denied`.
- SELinux hosts (RHEL, Fedora, Rocky): add `:z` (shared) or `:Z` (private) to relabel: `-v /srv/site:/usr/share/nginx/html:ro,Z`.
- Mount over a non-empty image directory: the host dir **hides** the image contents at that path (it does not merge).

### Named volumes

A named volume is storage **managed by Docker**. Data lives under `/var/lib/docker/volumes/<name>/_data` on Linux. You refer to it by name, not path.

- Syntax: `-v mydata:/var/lib/mysql` or `--mount type=volume,source=mydata,target=/var/lib/mysql`.
- If the volume does not exist, Docker creates it automatically on first use.
- **Initial population:** an empty new volume mounted over a path that has content in the image gets **copied** that content on first mount (a named volume only; bind mounts never do this).
- Survives `docker rm` of the container. Only removed by `docker volume rm` / `prune` / `docker rm -v` (for anonymous volumes).
- Good for: databases, uploads, any persistent app state; portable across hosts (no host path assumption); drivers can place volumes on NFS/cloud storage.
- **Anonymous volume:** `-v /var/lib/mysql` (no name) or a `VOLUME` instruction in a Dockerfile. Docker generates a random 64-char hex name. Hard to manage, accumulates as junk. Always prefer named.

### tmpfs mounts

A tmpfs mount stores data in the **host RAM** (and swap if memory is tight). Data disappears when the container stops. Linux only.

- Syntax: `--tmpfs /tmp` or `--tmpfs /tmp:rw,noexec,nosuid,size=64m` or `--mount type=tmpfs,target=/tmp,tmpfs-size=64m`.
- Good for: secrets that must not touch disk, scratch/cache space, writable `/tmp` when root FS is read-only.
- Cannot be shared between containers. Counts against container memory when a memory limit is set.

### Choosing the right mount

| Need | Use |
|---|---|
| Database / persistent app data | Named volume |
| Live-edit source code during development | Bind mount |
| Inject a single config file from host | Bind mount (`:ro`) |
| Share data between containers | Named volume (same volume in both) |
| Back up easily with Docker tooling | Named volume |
| Temporary, sensitive, fast scratch data | tmpfs |
| Host-specific path required (e.g., `/var/log`) | Bind mount |

### `-v` vs `--mount`

| | `-v` / `--volume` | `--mount` |
|---|---|---|
| Syntax | `src:dst:opts` (colon-separated) | `key=value,key=value` |
| Missing host path (bind) | Silently creates root-owned dir | Error |
| Readability | Terse | Explicit, self-documenting |
| tmpfs | `--tmpfs` flag | `type=tmpfs` |

Both work; `--mount` is recommended for scripts and documentation. `-v` is common in tutorials and quick commands.

### Sharing volumes between containers

Two ways:
1. Mount the **same named volume** in several containers: `-v shared:/data` in each.
2. `--volumes-from <container>`: mount all volumes of another container. Useful for backup helper containers.

Warning: Docker does no locking. Two containers writing the same files (e.g., two databases on one data dir) corrupt data. Share read-only where possible (`:ro`).

### Backup and restore of a volume

Use a throwaway container that mounts the volume plus a host directory:

```
volume (data) --ro--> [alpine tar] --> /backup/data.tar.gz on host
```

### Docker networking overview

Docker networking is built on Linux network namespaces, virtual ethernet pairs (veth) and a software bridge. Each container gets its own network namespace (own interfaces, routing table, iptables view). The Docker daemon manages **network drivers**:

| Driver | Description | Typical use |
|---|---|---|
| `bridge` | Private internal network on the host; containers get private IPs; NAT to outside | Default; single-host apps |
| `host` | Container shares the host network namespace; no isolation, no NAT, no port mapping | Max network performance, tools needing host network |
| `none` | Only loopback interface; no network | Isolated batch jobs, security sandbox |
| `overlay` | Multi-host network spanning several Docker daemons (VXLAN) | Docker Swarm clusters |
| `macvlan` | Container gets its own MAC/IP on the physical LAN | Legacy apps expecting LAN presence |
| `ipvlan` | Like macvlan but shares MAC | Environments limiting MACs per port |

Show built-in networks: `docker network ls` lists `bridge`, `host`, `none` out of the box.

### The default bridge network (`docker0`)

When Docker starts it creates a Linux bridge `docker0` (typically `172.17.0.0/16`, gateway `172.17.0.1`). Containers started with no `--network` attach here.

```
 Internet <--> eth0 (host) <--NAT/iptables MASQUERADE--> docker0 (172.17.0.1)
                                                         |        |
                                                      veth     veth
                                                       |        |
                                                  c1 172.17.0.2  c2 172.17.0.3
```

Limitations of the default bridge:
- **No automatic DNS** between containers — only by IP (IPs change on restart). Legacy `--link` exists; deprecated, do not teach/use.
- All containers on it can talk to each other (no segmentation).
- Settings changes need daemon restart.

Outbound: container -> internet works through masquerade NAT. Inbound: only through published ports (`-p`), implemented as DNAT iptables rules (lesson 11 iptables knowledge applies — `iptables -t nat -L -n` shows Docker's rules).

### User-defined bridge networks

Created with `docker network create <name>`. **Always prefer these.** Differences vs default bridge:

- **Automatic DNS resolution by container name** (and network alias) via Docker's embedded DNS server at `127.0.0.11` inside the container.
- Better isolation: only containers on the same network can talk; others cannot reach them.
- Containers can be attached/detached live: `docker network connect/disconnect`.
- Configurable subnet, gateway, IP range.

```
 network "backend"  : [api] [db]          api -> db  : ping db  (DNS works)
 network "frontend" : [web] [api]         web -> api : curl http://api:8080
 web cannot reach db (different network)  — segmentation
```

A container can be attached to **several** networks (an `api` container bridging frontend and backend).

### Container DNS in detail

- Embedded DNS server listens on `127.0.0.11:53` inside containers on user-defined networks; `/etc/resolv.conf` points to it.
- Resolves: container name, `--network-alias`, and (for services) service names.
- Unknown names are forwarded to the host's DNS servers (or `--dns` values).
- Container `--hostname` sets only the container's own hostname, not network DNS name.
- Custom DNS: `--dns 1.1.1.1`, `--dns-search example.com`, `--add-host myhost:10.0.0.5` (appends to `/etc/hosts`).

### Publishing ports (recap and depth)

Containers on a bridge network are reachable from the host via their IP, but **not** from outside unless ports are published.

| Form | Meaning |
|---|---|
| `-p 8080:80` | Host 8080 -> container 80, on all host IPv4 (0.0.0.0) |
| `-p 127.0.0.1:8080:80` | Bind to host loopback only (not reachable from the network) |
| `-p 8080:80/udp` | UDP |
| `-p 80` | Random high host port -> container 80 |
| `-P` | Publish all `EXPOSE`d ports to random host ports |
| `-p 8000-8010:8000-8010` | Port range |

Important security note: Docker inserts its own iptables rules (in `DOCKER` chain) **before** UFW/firewalld rules. A `-p 8080:80` is exposed to the internet even if `ufw deny 8080` is set. Bind to `127.0.0.1` for services that should be private, or put a reverse proxy in front (nginx, lesson 13).

### Host network mode

`--network host`: no network namespace isolation. Container process binds directly on host interfaces. `-p` is ignored (warning printed). Pros: no NAT overhead, can see all host interfaces. Cons: port conflicts with host services, no isolation, Linux only (on Docker Desktop for Mac/Windows it does not map to the real host).

### None network mode

`--network none`: container has only `lo`. Use for processing untrusted data offline, or when you attach custom networking by hand.

### Overlay network (concept only)

Overlay connects containers across **multiple Docker hosts** by wrapping traffic in VXLAN tunnels (UDP 4789). It requires Docker Swarm mode (`docker swarm init`) or an external key-value store. Single-host setups do not need it. In Kubernetes (lessons 34+), pod networking is done by CNI plugins instead, so overlay is rarely used directly today. We only mention it for vocabulary.

### Docker security model

Containers share the host kernel. Isolation relies on namespaces, cgroups, capabilities, seccomp and LSM profiles (AppArmor/SELinux). A kernel exploit or a misconfigured container can escape. Defense in depth — apply **least privilege**.

Key hardening layers covered today:
1. Do not run as root inside the container (`USER`, `--user`).
2. Read-only root filesystem (`--read-only`) with tmpfs for needed writable paths.
3. Drop Linux capabilities (`--cap-drop ALL`, add back only what is needed).
4. `--security-opt no-new-privileges`.
5. Resource limits (prevent DoS: memory hog, fork bomb).
6. Scan images for known CVEs (Trivy).
7. Minimal base images, pinned versions (lesson 31).

### Running as non-root

By default the container process runs as `root` (UID 0) **inside** the container. Without user namespaces remapping, UID 0 in the container is UID 0 on the host as far as the kernel is concerned (though capabilities are reduced). If an attacker escapes, they are root on the host.

Ways to run as non-root:
- In Dockerfile: `RUN addgroup -S app && adduser -S -G app app` then `USER app` (Alpine syntax), or `useradd -r -u 10001 app` (Debian).
- At runtime: `docker run --user 1000:1000 ...` (overrides image `USER`).
- Use numeric UID in `USER 10001` so Kubernetes `runAsNonRoot` can verify it later.

Gotchas: non-root cannot bind ports < 1024 by default (use 8080, or `--sysctl net.ipv4.ip_unprivileged_port_start=0`), and needs write permission on mounted volumes (`chown` or `--user` matching).

### Read-only root filesystem

`docker run --read-only` mounts the container root FS read-only. Malware cannot drop binaries or modify app files. Apps still need writable scratch spaces — supply them as tmpfs or volumes:

```
docker run --read-only --tmpfs /tmp --tmpfs /run -v appdata:/data myapp
```

### Linux capabilities

Traditional Unix: root = all powers. Linux splits root's powers into ~40 **capabilities** (e.g., `CHOWN`, `NET_BIND_SERVICE`, `SYS_ADMIN`, `NET_RAW`). Docker starts containers with a default reduced set of 14 capabilities (CHOWN, DAC_OVERRIDE, FSETID, FOWNER, MKNOD, NET_RAW, SETGID, SETUID, SETFCAP, SETPCAP, NET_BIND_SERVICE, SYS_CHROOT, KILL, AUDIT_WRITE).

Best practice: **drop all, add only what is needed**.

- `--cap-drop ALL --cap-add NET_BIND_SERVICE` — e.g., nginx binding port 80 as non-root-capable process.
- `--cap-add SYS_ADMIN` and `--privileged` are near-equivalent to host root. Avoid.
- `--privileged`: gives all capabilities, access to all host devices, disables seccomp/AppArmor confinement. Use only for docker-in-docker CI or low-level system tools, never for apps.

### no-new-privileges and seccomp

- `--security-opt no-new-privileges:true` blocks setuid binaries from gaining more privileges.
- Docker applies a default **seccomp** profile blocking ~44 dangerous syscalls. Do not disable (`--security-opt seccomp=unconfined`) without strong reason.

### Resource limits (cgroups)

Without limits a container may use all host CPU/RAM. Limits use cgroups (lesson 30).

| Flag | Effect |
|---|---|
| `--memory 256m` (`-m`) | Hard memory limit. Exceeded -> kernel OOM-kills the process (container exits with code 137) |
| `--memory-swap 512m` | Memory+swap total. Equal to `--memory` disables swap |
| `--memory-reservation 128m` | Soft limit, applied under host memory pressure |
| `--cpus 1.5` | Max 1.5 CPU cores of time |
| `--cpu-shares 512` | Relative weight (default 1024) when CPUs are contended; no limit if idle |
| `--cpuset-cpus 0,1` | Pin to specific cores |
| `--pids-limit 100` | Max processes/threads; stops fork bombs |
| `--ulimit nofile=1024:2048` | Per-process limits (open files) |
| `--oom-kill-disable` | Dangerous; avoid |

`docker stats` shows live CPU %, memory usage / limit, net I/O, block I/O, PIDs. `docker stats --no-stream` prints once.

### Image vulnerability scanning with Trivy (intro)

Images contain OS packages and app libraries. Each version may carry known vulnerabilities (**CVEs** — Common Vulnerabilities and Exposures, each rated by **CVSS** into severities `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`). **Trivy** (open source, Aqua Security) is a single-binary scanner that detects:
- OS package vulnerabilities (Alpine, Debian, Ubuntu, RHEL...)
- Language dependencies (npm, pip, Maven, Go modules, ...)
- Misconfigurations (Dockerfile, Kubernetes, Terraform) — `trivy config`
- Secrets accidentally baked in the image
- Licenses and SBOM generation

Workflow: build image -> `trivy image <name>` -> fix (upgrade base image, update packages, switch to alpine/distroless) -> rescan. In CI pipelines (lessons 59+) Trivy fails the build on HIGH/CRITICAL findings using `--exit-code 1`.

Trivy downloads a vulnerability DB (first run takes a minute), caches it in `~/.cache/trivy`. `--skip-db-update` for offline reuse. Not every CVE is fixable: `--ignore-unfixed` hides findings that have no patched version yet.

## Commands / Syntax Reference

### Volumes

| Command | Purpose | Example |
|---|---|---|
| `docker volume create` | Create named volume | `docker volume create pgdata` |
| `docker volume ls` | List volumes | `docker volume ls -f dangling=true` |
| `docker volume inspect` | Show mountpoint, driver, labels | `docker volume inspect pgdata` |
| `docker volume rm` | Remove volume (must be unused) | `docker volume rm pgdata` |
| `docker volume prune` | Remove all unused local volumes | `docker volume prune -f` |
| `docker run -v name:/path` | Mount named volume | `docker run -d -v pgdata:/var/lib/postgresql/data postgres:16` |
| `docker run -v /host:/ctr:ro` | Read-only bind mount | `docker run -v $PWD/conf:/etc/nginx/conf.d:ro nginx` |
| `docker run --mount` | Explicit mount | `docker run --mount type=volume,src=pgdata,dst=/data alpine` |
| `docker run --tmpfs` | RAM-backed mount | `docker run --tmpfs /tmp:size=64m alpine` |
| `docker run --volumes-from` | Reuse another container's mounts | `docker run --volumes-from db alpine ls /var/lib/postgresql/data` |
| `docker rm -v` | Remove container and its anonymous volumes | `docker rm -v web` |
| `docker system df -v` | Disk used by images/containers/volumes | `docker system df -v` |

### Networks

| Command | Purpose | Example |
|---|---|---|
| `docker network ls` | List networks | `docker network ls` |
| `docker network create` | Create user-defined bridge | `docker network create --subnet 10.10.0.0/24 appnet` |
| `docker network inspect` | Subnet, gateway, attached containers | `docker network inspect appnet` |
| `docker network connect` | Attach running container | `docker network connect appnet web` |
| `docker network disconnect` | Detach container | `docker network disconnect appnet web` |
| `docker network rm` | Remove network | `docker network rm appnet` |
| `docker network prune` | Remove unused networks | `docker network prune` |
| `docker run --network` | Start on a network | `docker run -d --network appnet --name api myapi` |
| `docker run --network-alias` | Extra DNS name | `docker run --network appnet --network-alias db-primary postgres:16` |
| `docker run --network host` | Use host namespace | `docker run --network host nginx` |
| `docker run --network none` | No networking | `docker run --network none alpine ip a` |
| `docker run -p` | Publish port | `docker run -p 127.0.0.1:8080:80 nginx` |
| `docker port` | Show port mappings | `docker port web` |
| `docker run --dns/--add-host` | Custom DNS / hosts entry | `docker run --add-host db:10.0.0.5 alpine cat /etc/hosts` |
| `docker inspect -f` | Extract container IP | `docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web` |

### Security and limits

| Command / flag | Purpose | Example |
|---|---|---|
| `--user` | Run as UID:GID | `docker run --user 1000:1000 alpine id` |
| `--read-only` | Read-only root FS | `docker run --read-only --tmpfs /tmp alpine touch /x` |
| `--cap-drop` / `--cap-add` | Adjust capabilities | `docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx` |
| `--security-opt no-new-privileges` | Block privilege escalation | `docker run --security-opt no-new-privileges alpine id` |
| `--memory` / `-m` | Memory limit | `docker run -m 128m nginx` |
| `--cpus` | CPU limit | `docker run --cpus 0.5 nginx` |
| `--pids-limit` | Process limit | `docker run --pids-limit 50 alpine sh` |
| `docker stats` | Live resource usage | `docker stats --no-stream` |
| `docker update` | Change limits on running container | `docker update --memory 256m --memory-swap 256m web` |
| `trivy image` | Scan image | `trivy image nginx:1.27-alpine` |
| `trivy image --severity` | Filter severities | `trivy image --severity HIGH,CRITICAL myapp:1.0` |
| `trivy image --exit-code` | Non-zero exit on findings | `trivy image --exit-code 1 --severity CRITICAL myapp:1.0` |
| `trivy image --ignore-unfixed` | Only show fixable CVEs | `trivy image --ignore-unfixed myapp:1.0` |
| `trivy image -f json -o` | Machine-readable report | `trivy image -f json -o report.json myapp:1.0` |
| `trivy config` | Scan Dockerfile/IaC misconfig | `trivy config .` |

## Examples / Walkthrough

### 1. Prove container data is ephemeral

```bash
docker run -d --name tmp1 alpine sh -c 'echo hello > /data.txt; sleep 3600'
docker exec tmp1 cat /data.txt        # hello
docker rm -f tmp1                      # container + writable layer deleted
docker run --rm alpine cat /data.txt   # No such file: new container, clean image
```

### 2. Named volume for a database

```bash
docker volume create pgdata
docker volume inspect pgdata           # Mountpoint: /var/lib/docker/volumes/pgdata/_data

docker run -d --name pg \
  -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16-alpine

docker exec -it pg psql -U postgres -c "CREATE TABLE t(id int); INSERT INTO t VALUES (1),(2);"

# destroy container, keep volume
docker rm -f pg

# new container, same volume -> data is still there
docker run -d --name pg2 -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data postgres:16-alpine
sleep 5
docker exec pg2 psql -U postgres -c "SELECT * FROM t;"    # 2 rows
```

Teaching point: upgrading `postgres:16-alpine` to a newer minor version = new container, same volume, data preserved.

### 3. Peek at the volume on the host (Linux)

```bash
sudo ls /var/lib/docker/volumes/pgdata/_data
# On Docker Desktop (Mac/Windows) the Docker VM holds this path; use a helper container instead:
docker run --rm -v pgdata:/data alpine ls /data
```

### 4. Bind mount for live development

```bash
mkdir -p ~/site && echo '<h1>v1</h1>' > ~/site/index.html
docker run -d --name web -p 8080:80 -v ~/site:/usr/share/nginx/html:ro nginx:alpine
curl localhost:8080                    # <h1>v1</h1>
echo '<h1>v2</h1>' > ~/site/index.html
curl localhost:8080                    # <h1>v2</h1> — no restart needed
docker exec web sh -c 'echo x > /usr/share/nginx/html/y'   # fails: Read-only file system
docker rm -f web
```

Compare `-v` with a missing path:

```bash
docker run --rm -v /tmp/does-not-exist:/x alpine ls /x        # works, Docker created root-owned dir
docker run --rm --mount type=bind,src=/tmp/nope2,dst=/x alpine ls /x
# docker: Error response from daemon: invalid mount config for type "bind": bind source path does not exist
```

### 5. tmpfs

```bash
docker run --rm --tmpfs /scratch:size=16m alpine sh -c \
  'df -h /scratch; dd if=/dev/zero of=/scratch/big bs=1M count=32'
# dd: write error: No space left on device — size limit enforced
```

### 6. Share a volume between containers

```bash
docker volume create shared
docker run -d --name writer -v shared:/data alpine \
  sh -c 'while true; do date >> /data/log.txt; sleep 2; done'
docker run --rm -v shared:/data:ro alpine tail -3 /data/log.txt     # reader, read-only
docker run --rm --volumes-from writer alpine ls /data               # same files
docker rm -f writer
```

### 7. Backup and restore a volume

```bash
mkdir -p ~/backup
# backup: tar the volume into a host dir
docker run --rm -v pgdata:/source:ro -v ~/backup:/backup alpine \
  tar czf /backup/pgdata-$(date +%F).tar.gz -C /source .

# simulate loss
docker volume rm pgdata     # fails if a container still uses it: remove container first

# restore into a fresh volume
docker volume create pgdata-restored
docker run --rm -v pgdata-restored:/target -v ~/backup:/backup alpine \
  tar xzf /backup/pgdata-$(date +%F).tar.gz -C /target
```

For databases prefer logical dumps (`pg_dump`, `mysqldump`) — a file-level copy of a running DB can be inconsistent.

### 8. Clean up volumes

```bash
docker volume ls
docker volume ls -f dangling=true      # not used by any container
docker volume prune                    # asks confirmation; deletes ALL unused volumes — data loss!
docker volume prune -a                 # newer Docker: also named unused volumes
docker system df -v                    # see space per volume
```

### 9. Explore default networks

```bash
docker network ls
docker network inspect bridge | head -40
ip addr show docker0                   # Linux host: 172.17.0.1/16
docker run -d --name c1 alpine sleep 3600
docker run -d --name c2 alpine sleep 3600
docker inspect -f '{{.NetworkSettings.IPAddress}}' c2     # e.g. 172.17.0.3
docker exec c1 ping -c1 172.17.0.3      # works by IP
docker exec c1 ping -c1 c2              # bad address 'c2' — no DNS on default bridge
docker rm -f c1 c2
```

### 10. User-defined network with DNS

```bash
docker network create appnet
docker run -d --name db --network appnet -e POSTGRES_PASSWORD=secret postgres:16-alpine
docker run -d --name web --network appnet nginx:alpine
docker exec web ping -c2 db            # resolves: embedded DNS
docker exec web cat /etc/resolv.conf   # nameserver 127.0.0.11
docker exec web nslookup db 127.0.0.11
docker network inspect appnet          # both containers listed with IPs
```

### 11. Segmentation with two networks

```bash
docker network create frontend
docker network create backend
docker run -d --name database --network backend -e POSTGRES_PASSWORD=x postgres:16-alpine
docker run -d --name api --network backend alpine sleep 3600
docker run -d --name proxy --network frontend -p 8081:80 nginx:alpine

docker network connect frontend api        # api now on both networks
docker exec api ping -c1 database          # ok (backend)
docker exec api ping -c1 proxy             # ok (frontend)
docker exec proxy ping -c1 database        # fails: bad address — isolated
```

### 12. Custom subnet and fixed IP

```bash
docker network create --subnet 10.50.0.0/24 --gateway 10.50.0.1 fixednet
docker run -d --name fixed --network fixednet --ip 10.50.0.10 alpine sleep 3600
docker inspect -f '{{.NetworkSettings.Networks.fixednet.IPAddress}}' fixed   # 10.50.0.10
```

(`--ip` works only on user-defined networks with an explicit subnet.)

### 13. Host and none networks

```bash
docker run --rm --network host alpine ip addr           # sees host interfaces
docker run -d --network host --name hostweb nginx:alpine
curl localhost:80                                       # nginx on host port 80, no -p needed
docker rm -f hostweb
docker run --rm --network none alpine ip addr           # only lo
docker run --rm --network none alpine ping -c1 8.8.8.8 # Network is unreachable
```

### 14. Port publishing variants

```bash
docker run -d --name p1 -p 127.0.0.1:9001:80 nginx:alpine
curl localhost:9001                                    # ok
curl http://<host-lan-ip>:9001                         # connection refused: loopback only
docker run -d --name p2 -P nginx:alpine                # random port
docker port p2                                         # 80/tcp -> 0.0.0.0:32768
sudo iptables -t nat -L DOCKER -n | head               # DNAT rules by Docker
docker rm -f p1 p2
```

### 15. Hardened container

```bash
docker run --rm -it \
  --user 10001:10001 \
  --read-only --tmpfs /tmp:rw,noexec,nosuid,size=16m \
  --cap-drop ALL \
  --security-opt no-new-privileges:true \
  --pids-limit 64 --memory 128m --cpus 0.5 \
  alpine sh

# inside:
id                       # uid=10001
touch /etc/x             # Read-only file system
touch /tmp/ok            # works (tmpfs)
ping -c1 8.8.8.8         # operation not permitted: NET_RAW dropped
```

Compare with root default:

```bash
docker run --rm alpine id                          # uid=0(root)
docker run --rm alpine sh -c 'apk add curl' 2>&1 | tail -1   # works as root
docker run --rm --cap-drop ALL alpine chown nobody /tmp      # Operation not permitted
```

Nginx bound to port 80 with minimum capabilities:

```bash
docker run -d --name hardnginx -p 8082:80 \
  --read-only --tmpfs /var/cache/nginx --tmpfs /var/run --tmpfs /tmp \
  --cap-drop ALL --cap-add CHOWN --cap-add SETUID --cap-add SETGID --cap-add NET_BIND_SERVICE \
  nginx:alpine
docker logs hardnginx
```

### 16. Non-root user in a Dockerfile

```dockerfile
FROM alpine:3.20
RUN addgroup -S app -g 10001 && adduser -S app -u 10001 -G app
WORKDIR /app
COPY --chown=app:app hello.sh .
USER 10001
CMD ["sh", "hello.sh"]
```

```bash
echo 'echo "I am $(id -u)"' > hello.sh
docker build -t nonroot-demo:1.0 .
docker run --rm nonroot-demo:1.0                 # I am 10001
docker run --rm --user root nonroot-demo:1.0     # override at runtime (anyone with docker access can do this)
```

### 17. Memory limit and OOM kill

```bash
docker run -d --name hog -m 64m --memory-swap 64m alpine \
  sh -c 'x=""; while true; do x="$x$(head -c 1000000 /dev/zero | tr "\0" a)"; done'
sleep 5
docker ps -a --filter name=hog                  # Exited (137)
docker inspect -f '{{.State.OOMKilled}} {{.State.ExitCode}}' hog    # true 137
docker rm hog
```

### 18. CPU limit and docker stats

```bash
docker run -d --name burn1 --cpus 0.5 alpine sh -c 'while :; do :; done'
docker run -d --name burn2 alpine sh -c 'while :; do :; done'
docker stats --no-stream burn1 burn2
# burn1 ~50% CPU, burn2 ~100% (one full core)
docker update --cpus 0.2 burn1                  # change live
docker stats                                    # live view, Ctrl+C to quit
docker rm -f burn1 burn2
```

### 19. PIDs limit stops a fork bomb

```bash
docker run --rm --pids-limit 20 alpine sh -c \
  'for i in $(seq 1 100); do sleep 30 & done; echo spawned'
# sh: can't fork: Resource temporarily unavailable — limit enforced
```

### 20. Trivy: install

```bash
# macOS
brew install trivy
# Debian/Ubuntu (official apt repo)
sudo apt-get install -y wget gnupg
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install -y trivy
# Or run without installing:
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image alpine:3.20
trivy --version
```

### 21. Trivy: scan an image

```bash
trivy image python:3.9                # big Debian-based image: many findings
trivy image python:3.12-alpine        # far fewer
```

Output sections: target (OS / language packages), a table with `Library | Vulnerability (CVE id) | Severity | Status | Installed Version | Fixed Version | Title`. `Status: fixed` means a patched version exists -> upgrade; `affected`/`will_not_fix` means wait or switch base.

Compare base images:

```bash
trivy image --severity HIGH,CRITICAL --quiet python:3.9 | tail -5
trivy image --severity HIGH,CRITICAL --quiet python:3.12-alpine | tail -5
```

### 22. Trivy: severity, unfixed, exit code, formats

```bash
trivy image --severity CRITICAL nginx:1.21                       # only critical
trivy image --severity HIGH,CRITICAL --ignore-unfixed nginx:1.21
trivy image --exit-code 1 --severity CRITICAL nginx:1.21; echo "exit=$?"   # exit=1 when found
trivy image --exit-code 0 nginx:alpine; echo "exit=$?"                     # always 0
trivy image -f json -o report.json nginx:1.21                    # JSON for tooling
trivy image -f table --scanners vuln nginx:1.21                  # vulnerabilities only
trivy image --scanners secret myapp:1.0                          # look for baked-in secrets
```

### 23. Fix loop with your own image

```bash
cat > Dockerfile <<'EOF'
FROM node:16
WORKDIR /app
COPY . .
CMD ["node","-e","setInterval(()=>console.log('up'),1000)"]
EOF
docker build -t scan-demo:old .
trivy image --severity HIGH,CRITICAL --quiet scan-demo:old | grep Total

sed -i.bak 's/node:16/node:22-alpine/' Dockerfile
docker build -t scan-demo:new .
trivy image --severity HIGH,CRITICAL --quiet scan-demo:new | grep Total
```

Result: switching base image removes most findings — the cheapest security win.

### 24. Trivy: scan Dockerfile misconfigurations

```bash
trivy config .
# e.g. AVD-DS-0002 (HIGH): Specify at least 1 USER command in Dockerfile with non-root user
# AVD-DS-0026: Add HEALTHCHECK instruction
```

### 25. Cleanup

```bash
docker rm -f $(docker ps -aq) 2>/dev/null
docker network prune -f
docker volume prune -f
docker rmi scan-demo:old scan-demo:new nonroot-demo:1.0
```

## Common Pitfalls

- **Data vanishes after `docker rm`** — wrote into writable layer. Mount a named volume at the app's data directory (check image docs, e.g. `/var/lib/mysql`, `/var/lib/postgresql/data`).
- **`docker volume prune` deletes valuable data** — prune removes every volume not attached to a *container* (stopped containers still count as attached). Removed containers first + prune = data gone. Check `docker volume ls` first, and back up.
- **Anonymous volumes pile up** — `VOLUME` in image or `-v /path` creates random-named volumes. Use named volumes; `docker rm -v` or periodic `docker system df -v` review.
- **`-v ./relative:/path` fails or creates a named volume** — `-v` needs an absolute path (use `$PWD/dir`); a bare name without `/` is treated as a volume name. `--mount` with `src=$(pwd)/dir` is explicit.
- **Bind mount `Permission denied`** — UID mismatch between host dir owner and container user. Fix with `chown` on host, `--user $(id -u):$(id -g)`, or use a named volume.
- **Bind mount hides image content** — mounting an empty host dir on `/app` or `/usr/share/nginx/html` makes the container see empty dir. Named volumes copy image content on first use; bind mounts do not.
- **Mounting `/var/run/docker.sock`** — gives the container full control of the Docker daemon = root on host. Avoid; if unavoidable, read-only is not enough (the API is still writable).
- **Database corrupted by two containers on one volume** — never point two DB instances at the same data directory.
- **Pinging container by name on the default bridge** — no DNS there. Create a user-defined network and attach containers to it.
- **`localhost` inside a container is the container itself** — to reach a sibling use its name on a shared user-defined network; to reach the host use `host.docker.internal` (Docker Desktop; on Linux add `--add-host host.docker.internal:host-gateway`).
- **Published port bypasses UFW** — Docker writes iptables rules ahead of UFW. Use `-p 127.0.0.1:port:port` for private services, or manage the `DOCKER-USER` chain.
- **Service listens on `127.0.0.1` inside container** — published port gets connection reset/empty reply. App must bind `0.0.0.0` inside the container.
- **`--network host` on macOS/Windows does nothing useful** — Docker runs in a VM there.
- **Port conflict** — `bind: address already in use` when host port is taken (`ss -tlnp | grep 8080`). Change host side of `-p`.
- **Container IPs hard-coded in configs** — IPs change on recreate. Use names/aliases.
- **Running as root and assuming "it's a container, so it's safe"** — containers share kernel. Add `USER`, drop caps.
- **`--read-only` app crashes** — app writes to `/tmp`, `/var/run`, cache or logs. Read errors (`Read-only file system`), then add targeted `--tmpfs`/volume mounts.
- **Non-root cannot bind port 80** — use port 8080+ inside the container, or add `NET_BIND_SERVICE`.
- **`--privileged` used to "make it work"** — find the specific capability or device (`--cap-add`, `--device`) instead.
- **Memory limit set but app killed with exit code 137** — OOM kill. Raise `--memory` or fix leak; JVM/Node need heap settings aware of the cgroup limit.
- **`--cpus` confused with `--cpu-shares`** — `--cpus` is a hard cap; shares only matter under contention.
- **Trivy shows hundreds of CVEs and students panic** — filter `--severity HIGH,CRITICAL --ignore-unfixed`, then prioritise base image upgrade. Many findings are in unused packages.
- **Scanning once and forgetting** — new CVEs appear daily; rescan regularly and in CI.
- **Trivy slow / rate-limited on first run** — DB download from a registry (GHCR/mirror.gcr.io); retry, or use `--db-repository` mirror, then cache.
- **`latest` tag scans differ each day** — pin versions (lesson 31) for reproducible scan results.

## FAQ

**Q: What is the difference between a volume and a bind mount?**
A: A named volume is created and managed by Docker (stored in `/var/lib/docker/volumes`), referenced by name, portable, pre-populated from the image. A bind mount maps any existing host path directly; it depends on host layout and never copies image content. Volumes for persistent app data, bind mounts for dev code and host config files.

**Q: Where does Docker store volume data on the host?**
A: `/var/lib/docker/volumes/<volume-name>/_data` on Linux (find via `docker volume inspect <name>` -> `Mountpoint`). On Docker Desktop it is inside the Docker VM, not directly on the Mac/Windows filesystem.

**Q: Does data in a volume survive if I delete the container?**
A: Yes. `docker rm` leaves named volumes. Only `docker rm -v` (anonymous volumes), `docker volume rm` or `docker volume prune` delete them.

**Q: How do I back up a volume?**
A: Run a temporary container mounting the volume (read-only) and a host directory, then `tar` the data — see walkthrough 7. For databases prefer `pg_dump` / `mysqldump` run via `docker exec`.

**Q: How do I move data from a bind mount into a named volume?**
A: `docker run --rm -v /host/dir:/from:ro -v myvol:/to alpine cp -a /from/. /to/`.

**Q: What does `:ro` do?**
A: Mounts the path read-only inside the container; writes fail with "Read-only file system". Good for config files and shared data readers.

**Q: What's the difference between `-v` and `--mount`?**
A: Same capability, different syntax. `--mount` uses `key=value` pairs and errors on a missing bind source; `-v` is short and silently creates missing host directories.

**Q: What is tmpfs good for?**
A: Fast, memory-only, never written to disk, wiped at stop. Use for temporary or sensitive data, and for writable `/tmp` when the root FS is read-only.

**Q: Why can't my containers ping each other by name?**
A: They are probably on the default `bridge` network, which has no DNS. Create a user-defined network (`docker network create mynet`) and run both containers with `--network mynet`.

**Q: How does container DNS work?**
A: On user-defined networks Docker runs an embedded DNS server (`127.0.0.11`) that resolves container names and network aliases; other names are forwarded to the host's resolvers.

**Q: How can a container reach a service running on my host?**
A: Use `host.docker.internal` (Docker Desktop, or add `--add-host host.docker.internal:host-gateway` on Linux), or the bridge gateway IP (`172.17.0.1`), and make sure the host service listens on that interface, not only `127.0.0.1`.

**Q: What is the difference between `EXPOSE` and `-p`?**
A: `EXPOSE` is documentation (metadata) of the port the app uses; it does not open anything. `-p host:container` actually publishes the port. `-P` publishes all `EXPOSE`d ports on random host ports.

**Q: When should I use `--network host`?**
A: When you need maximum network performance or must see host interfaces (monitoring agents, network tools). You lose port isolation. Avoid for normal apps.

**Q: What is an overlay network and do I need it?**
A: A multi-host virtual network (VXLAN) connecting containers on different Docker hosts; needs Swarm mode. Single-host and Kubernetes users do not create overlay networks by hand.

**Q: Why is my published port reachable even though UFW blocks it?**
A: Docker adds its own iptables rules that run before UFW's. Bind to `127.0.0.1`, use a reverse proxy, or put restrictions in the `DOCKER-USER` iptables chain.

**Q: Can two containers use the same host port?**
A: No — one host port maps to one container. Run different host ports (`-p 8081:80`, `-p 8082:80`) or put a reverse proxy in front.

**Q: Is running as root in a container really dangerous?**
A: Container root is namespaced but shares the host kernel; with a bug, misconfiguration (mounted socket, `--privileged`, extra capabilities) it can become host root. Running as non-root removes a large class of attacks, so always set `USER`.

**Q: What are Linux capabilities?**
A: Fine-grained pieces of root's power (bind low ports, change file owner, raw sockets, mount...). Docker gives a reduced default set; best practice `--cap-drop ALL` then `--cap-add` only what the app needs.

**Q: What does `--privileged` do?**
A: Grants all capabilities, access to host devices and disables most confinement. Treat it as root on the host; avoid.

**Q: Does `--read-only` stop volumes from working?**
A: No. It only makes the image's root filesystem read-only. Mounted volumes and tmpfs remain writable (unless `:ro`).

**Q: What happens when a container exceeds `--memory`?**
A: The kernel OOM-kills its main process; container exits with code 137 (`OOMKilled: true` in `docker inspect`). With a restart policy it restarts.

**Q: What does `--cpus 1.5` mean?**
A: The container may use at most 1.5 CPU cores' worth of time total (e.g., 150% in `docker stats`).

**Q: How do I see how much resource a container uses?**
A: `docker stats` (live) or `docker stats --no-stream`; for history use monitoring tools (lessons 67+).

**Q: Can I change limits without recreating the container?**
A: Yes for most: `docker update --memory 512m --cpus 1 <container>`.

**Q: What does Trivy scan?**
A: Images (OS + language packages), filesystems, Git repos, and config files (Dockerfile, Kubernetes, Terraform), for CVEs, misconfigurations, secrets and licenses.

**Q: How do I make Trivy fail a CI job?**
A: `trivy image --exit-code 1 --severity HIGH,CRITICAL <image>`: exit code 1 if any matching vulnerability is found, 0 otherwise.

**Q: Which severity levels exist?**
A: `UNKNOWN`, `LOW`, `MEDIUM`, `HIGH`, `CRITICAL`. Filter with a comma list.

**Q: Why does Trivy report vulnerabilities with no fix?**
A: The upstream distro has not released a patch yet. Use `--ignore-unfixed` to focus on actionable items, and prefer smaller base images to reduce exposure.

**Q: What is the quickest way to cut vulnerabilities?**
A: Update to a newer base image version, switch to `-alpine`/`-slim`/distroless, remove unneeded packages, and use multi-stage builds so build tools never reach the final image (lesson 31).

**Q: How do I clean up everything unused?**
A: `docker system prune` (stopped containers, unused networks, dangling images, build cache). Add `--volumes` to also delete unused volumes — careful.

**Q: How are registries and Compose related to these topics?**
A: Compose (declaring volumes/networks for multi-container apps in one file) and registries (sharing images) are covered in lesson 33.

## Practice / Exercise

### Core (all students, ~60 min)

1. **Ephemeral vs persistent.** Run an `alpine` container that writes a file, delete it, show the file is gone. Repeat with a named volume `notes` and show the file survives container deletion.
2. **Postgres on a volume.** Start `postgres:16-alpine` with a named volume, create a table, remove the container, start a new one on the same volume and prove the table exists. Show the volume's `Mountpoint` with `docker volume inspect`.
3. **Bind mount site.** Serve a local folder with `nginx:alpine` on port 8080 read-only; edit `index.html` on the host and refresh with `curl`. Prove the container cannot write to it.
4. **Backup/restore.** Back up your Postgres volume to a `.tar.gz` using an alpine helper container; restore it into a new volume and start a database on it.
5. **DNS.** Show that two containers on the default bridge cannot ping by name. Create network `labnet`, run two containers on it and ping by name; `docker exec <c> cat /etc/resolv.conf`.
6. **Segmentation.** Build `frontend` and `backend` networks with `proxy`, `api`, `database`; `api` joins both. Prove `proxy` cannot reach `database`.
7. **Port binding.** Publish one container on `127.0.0.1:9001` and another on all interfaces; test from the host and from a neighbour's machine/another interface. Use `docker port`.
8. **Hardening.** Run an alpine shell with `--user 10001`, `--read-only`, `--tmpfs /tmp`, `--cap-drop ALL`, `--security-opt no-new-privileges`. Record which commands fail and why.
9. **Limits.** Start a container with `--memory 64m` and cause an OOM kill; read `OOMKilled` and exit code via `docker inspect`. Start a `--cpus 0.5` busy loop and show ~50% in `docker stats`.
10. **Trivy basics.** Scan `nginx:1.21` and `nginx:alpine`. Compare totals for `HIGH,CRITICAL`; run with `--exit-code 1` and print `$?`.

### Stretch

1. **Non-root image.** Write a Dockerfile (from lesson 31 knowledge) for a small app with numeric non-root `USER`, a writable data dir backed by a named volume with correct ownership, and run it with `--read-only`. Scan it with Trivy and aim for zero HIGH/CRITICAL findings that have fixes (`--ignore-unfixed`).
2. **Minimal-capability nginx.** Find the smallest set of `--cap-add` values with `--cap-drop ALL` that lets `nginx:alpine` start with `--read-only` (hint: read logs, add tmpfs for `/var/cache/nginx`, `/var/run`).
3. **Fixed IP network.** Create a network `10.77.0.0/24`, attach containers with fixed IPs, inspect the Linux bridge with `ip addr` / `bridge link` and the NAT rules with `iptables -t nat -S`.
4. **Fork bomb safety.** Run `:(){ :|:& };:` in an alpine container with `--pids-limit 30` and `--memory 64m`; show the host stays responsive (do it only in a lab VM).
5. **Report pipeline.** Script: for every local image, run `trivy image -f json`, parse with `jq` the count of CRITICAL vulnerabilities, and print a sorted table; exit non-zero if any image exceeds a threshold.
6. **Volume migration.** Move data from a bind-mounted directory to a named volume while a DB is stopped, verifying checksums (`sha256sum`) before and after.

## Further Reading

- Docker storage overview (volumes, bind mounts, tmpfs): https://docs.docker.com/engine/storage/
- Docker networking overview and drivers: https://docs.docker.com/engine/network/
- Bridge network driver and user-defined bridges: https://docs.docker.com/engine/network/drivers/bridge/
- Publishing ports and firewall behaviour: https://docs.docker.com/engine/network/packet-filtering-firewalls/
- Docker Engine security: https://docs.docker.com/engine/security/
- Linux capabilities man page: https://man7.org/linux/man-pages/man7/capabilities.7.html
- Resource constraints for containers: https://docs.docker.com/engine/containers/resource_constraints/
- Docker Bench for Security (CIS benchmark script): https://github.com/docker/docker-bench-security
- OWASP Docker Security Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html
- Trivy documentation: https://trivy.dev/latest/docs/
