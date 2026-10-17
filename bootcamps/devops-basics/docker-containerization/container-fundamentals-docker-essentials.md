# Lesson 30: Container Fundamentals & Docker Essentials

**Module:** Docker – Containerization
**Duration:** 120-150 min
**Prerequisites:** Lessons 1-29 — especially Linux processes and signals (Lesson 8), Linux networking and ports (Lesson 10), file systems (Lesson 12), the `nginx` web server (Lesson 13), and `apt` / systemd (Lessons 5, 7)

## Learning Objectives

By end of lesson student can:
- Explain what a container is and how it differs from a virtual machine
- Name the three Linux kernel features behind containers: namespaces, cgroups, union filesystem (overlay2)
- Describe Docker architecture: client (`docker` CLI), daemon (`dockerd`), `containerd`, `runc`, registry
- Install Docker Engine on Ubuntu and run `docker` without `sudo`
- Pull images and run containers with `-d`, `-it`, `--name`, `--rm`
- Publish ports with `-p`, mount host paths with `-v`, pass configuration with `-e` and `--env-file`
- Walk a container through its lifecycle: create, start, stop, rm; inspect it with `logs`, `inspect`, `stats`, `exec`
- Clean up stopped containers, unused images and disk space safely

## Topics

- Container vs VM: Linux namespaces, cgroups, union filesystem (overlay2)
- Docker architecture: daemon (`dockerd`), client (CLI), registry, `containerd`
- Images and containers: `pull`, `images`, `run` (`-d`, `-it`, `--name`, `--rm`), `ps -a`, `exec -it`
- Port mapping (`-p`), volume mounts (`-v`), environment variables (`-e`, `--env-file`)
- Container lifecycle: create → start → stop → rm; `logs`, `inspect`, `stats`

## Concepts

### The problem containers solve

"It works on my machine." An application needs a specific runtime (Python 3.12, Node 20), specific libraries, specific config files. Installing everything directly on a server (the way we installed nginx in Lesson 13) causes problems:

- **Dependency conflicts** — app A needs OpenSSL 1.1, app B needs OpenSSL 3. They cannot share one OS.
- **Environment drift** — dev laptop, staging and production slowly differ; bugs appear only in one.
- **Slow, manual setup** — a new server means re-running a long list of install steps (Ansible, Lessons 24-29, automates this but the install still happens on the real host).
- **Poor isolation** — one misbehaving app can eat all RAM/CPU or crash neighbours.

A **container** packages an application together with everything it needs (runtime, libraries, config defaults) into one portable unit that runs the same on any Linux host with a container engine. Build once, run anywhere.

### What a container actually is

A container is **not** a tiny virtual machine. It is an ordinary Linux **process** (or a group of processes) that the kernel isolates and limits. There is no second operating system and no hypervisor. The container shares the **host's kernel**.

Prove it on any Docker host: run `sleep 1000` in a container, then `ps aux | grep sleep` on the host — the process is visible in the host's process list, just with a different PID view inside the container.

### Container vs virtual machine

| | Virtual machine | Container |
|---|---|---|
| Isolation unit | Whole emulated computer | Isolated process(es) |
| Kernel | Own guest kernel per VM | Shares host kernel |
| Includes | Full OS (GBs) | App + libs only (MBs to hundreds of MB) |
| Startup | Tens of seconds to minutes (boot an OS) | Milliseconds to seconds (start a process) |
| Density | Handfuls per host | Dozens to hundreds per host |
| Isolation strength | Strong (hardware-level, separate kernels) | Weaker (shared kernel; kernel bug = risk to all) |
| OS variety | Any OS (Windows VM on Linux host) | Same kernel family only: Linux containers need a Linux kernel |
| Typical use | Different OSes, strong tenant isolation, long-lived servers | Microservices, CI jobs, reproducible deployments, fast scaling |

Important consequence: a container image built for Linux cannot run natively on a Windows or macOS kernel. Docker Desktop on Mac/Windows quietly runs a small Linux VM and puts your containers inside it.

Containers and VMs are complementary — most production container workloads run **inside** VMs (cloud instances).

### The three kernel building blocks

Docker did not invent containers; it made existing Linux features easy to use. Three features do the work.

#### 1. Namespaces — "what a process can see"

A **namespace** gives a process its own private view of a system resource. The kernel has several types:

| Namespace | Isolates | Effect inside the container |
|---|---|---|
| `pid` | Process IDs | Container's first process is PID 1; it cannot see host processes |
| `net` | Network stack | Own interfaces, IP address, routing table, port space (two containers can both listen on port 80) |
| `mnt` | Mount points | Own root filesystem `/`; host files invisible |
| `uts` | Hostname/domain | Container has its own hostname |
| `ipc` | Shared memory, message queues | Cannot talk to host processes via IPC |
| `user` | User/group IDs | Root (UID 0) in the container can map to an unprivileged UID on the host |

Try it without Docker (Lesson 8 knowledge helps):

```bash
sudo unshare --pid --fork --mount-proc bash   # new PID namespace
ps aux                                          # only bash and ps visible, bash is PID 1
exit
```

That is the essence of a container: a process started with new namespaces.

#### 2. cgroups (control groups) — "how much a process can use"

**cgroups** let the kernel limit, account for, and prioritize resources used by a group of processes: CPU time, memory, disk I/O, number of processes. Namespaces hide things; cgroups cap things. Without cgroups one container could consume all host RAM and trigger the kernel OOM killer on everything else. Docker exposes cgroups as run flags (`--memory`, `--cpus`, `--pids-limit` — covered in lesson 32). In this lesson `docker stats` simply *shows* the cgroup accounting.

View cgroup data on the host: `cat /sys/fs/cgroup/` (cgroup v2 on modern Ubuntu); each container has a directory under `system.slice/docker-<id>.scope`.

#### 3. Union filesystem (overlay2) — "how files are stored"

An image is made of read-only **layers** stacked on top of each other. A **union filesystem** merges those layers into one view. Docker's default storage driver is **overlay2**.

```
 ┌────────────────────────────┐
 │ Container writable layer   │  ← read-write, per container, deleted with container
 ├────────────────────────────┤
 │ Layer 3: app files         │  ← read-only, part of the image
 │ Layer 2: installed packages│  ← read-only
 │ Layer 1: base OS files     │  ← read-only
 └────────────────────────────┘
```

Key behaviors:
- **Sharing:** ten containers from the same image share the same read-only layers on disk; each adds only its own thin writable layer. This is why containers start fast and use little disk.
- **Copy-on-write:** when a container modifies a file that lives in a lower layer, the file is first copied up into the writable layer, then changed. The image layer is never altered.
- **Ephemeral writes:** when the container is removed, its writable layer — and everything written into it — is deleted. Data that must survive needs a mounted volume (introduced below with `-v`; managed volumes in lesson 32).

Layers live under `/var/lib/docker/overlay2/` on the host.

### Docker architecture

```
 ┌──────────────┐  REST API over   ┌───────────────────────────┐     ┌──────────────┐
 │ docker CLI   │ ───────────────► │ dockerd (Docker daemon)   │ ──► │ containerd   │
 │ (client)     │  /var/run/       │ images, networks, volumes │     │ lifecycle    │
 └──────────────┘  docker.sock     └───────────────────────────┘     └──────┬───────┘
                                              │ pull/push                    │
                                              ▼                              ▼
                                     ┌────────────────┐              ┌────────────┐
                                     │ Registry       │              │ runc       │
                                     │ (Docker Hub)   │              │ creates the│
                                     └────────────────┘              │ container  │
                                                                     └────────────┘
```

| Component | Role |
|---|---|
| **Docker client** (`docker`) | The CLI you type into. It does no real work; it sends API requests to the daemon |
| **Docker daemon** (`dockerd`) | Long-running background service (systemd unit `docker.service`). Manages images, containers, networks, volumes. Listens on the Unix socket `/var/run/docker.sock` |
| **containerd** | Lower-level container runtime daemon. Pulls image content, manages container lifecycle and snapshots. Also used directly by Kubernetes |
| **runc** | Tiny tool that talks to the kernel and actually creates the namespaces/cgroups and starts the process. Implements the OCI runtime spec |
| **Registry** | Server that stores and distributes images. **Docker Hub** (`hub.docker.com`) is the default public registry. Private registries covered in lesson 33 |
| **Image** | Read-only template: layers + metadata (default command, env, exposed ports) |
| **Container** | A running (or stopped) instance of an image |

Flow of `docker run nginx`: client → daemon → daemon checks for local image, pulls from the registry if missing → containerd → runc creates namespaces + cgroups + overlay mount → nginx process starts as PID 1 in the container.

Security note: access to `/var/run/docker.sock` is effectively **root on the host** (you can start a privileged container mounting `/`). Members of the `docker` group are therefore root-equivalent. Grant it carefully.

OCI (Open Container Initiative) standards define the image format and runtime spec, which is why Docker images also run on Podman, containerd, CRI-O, and Kubernetes.

### Images vs containers

Analogy: **image = class / recipe / ISO file**, **container = object / cooked meal / running instance**. One image → many containers.

**Image names** have the form: `[registry/][namespace/]repository[:tag]`

| Name | Meaning |
|---|---|
| `nginx` | Docker Hub official image `docker.io/library/nginx`, tag `latest` |
| `nginx:1.27` | Specific version tag |
| `nginx:1.27-alpine` | Same version, built on Alpine Linux (smaller) |
| `bitnami/redis:7.2` | Namespace `bitnami` (a publisher) on Docker Hub |
| `ghcr.io/owner/app:v1` | Image on another registry (GitHub Container Registry) |

A **tag** is a movable label, not a guarantee. `latest` is just a default tag name: it does **not** mean "newest", and it can change under you. In production pin an exact version (or a digest `nginx@sha256:...`). An image is identified uniquely by its **digest** (`sha256:...` of its content), and by a short **image ID**.

Variants you will see: `-alpine` (tiny, uses musl libc), `-slim` (trimmed Debian), `-bookworm`/`-jammy` (specific OS). Choosing a base is revisited in lesson 31.

### Installing Docker Engine on Ubuntu

Use Docker's official apt repository (the `docker.io` package in Ubuntu repos is older):

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" \
 | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Verify and enable:

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
docker --version
sudo docker run hello-world
```

**Run without sudo:** the socket belongs to group `docker`.

```bash
sudo usermod -aG docker $USER
newgrp docker            # or log out and back in
docker ps                # works without sudo
```

Quick alternative for labs only: `curl -fsSL https://get.docker.com | sh` (convenience script — fine for a throwaway VM, not for audited production hosts).

Useful info commands: `docker version` (client + server), `docker info` (storage driver `overlay2`, cgroup version, number of containers/images, root dir `/var/lib/docker`).

### Running containers: `docker run`

`docker run [OPTIONS] IMAGE [COMMAND] [ARG...]` = `docker create` + `docker start` (+ `docker pull` if the image is missing). Everything after the image name is the command executed *inside* the container, overriding the image's default command.

| Option | Meaning |
|---|---|
| `-d` / `--detach` | Run in background, print container ID. Used for servers |
| `-i` | Keep STDIN open (interactive) |
| `-t` | Allocate a pseudo-terminal (TTY) |
| `-it` | Both — gives an interactive shell. Use for `bash`, `sh`, `python` REPL |
| `--name web` | Human-readable name (must be unique); otherwise Docker invents one like `quirky_turing` |
| `--rm` | Delete the container automatically when it exits. Ideal for one-off commands |
| `-p HOST:CONTAINER` | Publish a port |
| `-v HOST:CONTAINER` | Mount a path |
| `-e KEY=VALUE` | Set an environment variable |
| `--env-file FILE` | Load environment variables from a file |
| `--restart` | Restart policy (`no`, `on-failure`, `always`, `unless-stopped`) |

**Foreground vs detached.** Without `-d` the container's output is attached to your terminal and Ctrl+C stops it. With `-d` it runs in the background; use `docker logs` to see its output.

**A container lives only as long as its main process (PID 1).** If PID 1 exits, the container stops. This is the most important mental model: a container running `bash` without `-it` exits immediately because bash gets no input and ends. A container is "a process," not "a server."

### Port mapping: `-p`

A container has its own network namespace and IP (typically `172.17.0.x` on the default `bridge` network), unreachable from outside the host. **Publishing** a port creates a rule (via iptables, Lesson 11) that forwards traffic arriving at a host port to the container port.

```
-p 8080:80                  host 8080 → container 80  (all host interfaces, 0.0.0.0)
-p 127.0.0.1:8080:80        only reachable from the host itself
-p 80                       random free host port → container 80 (see with docker port)
-p 8080:80/udp              UDP instead of TCP
-P                          publish every EXPOSEd port to random host ports
```

Format is always **host:container** — left side is outside, right side is inside. Two containers cannot publish the same host port, but both can listen on port 80 internally.

Docker publishes ports by editing iptables, and these rules bypass `ufw` rules (Lesson 11). A port published with `-p 8080:80` is reachable from the internet even if ufw denies 8080. Bind to `127.0.0.1` when you only want local access (e.g. behind an nginx reverse proxy).

### Volume mounts: `-v` (introduction)

Remember: the container's writable layer disappears with the container. `-v` connects storage that lives *outside* the container.

```
-v /host/path:/container/path          bind mount: a host directory/file appears inside
-v /host/path:/container/path:ro       same, read-only inside the container
-v /container/path                     anonymous volume: Docker creates storage, random name
```

- A **bind mount** maps an exact host path. Great for development (edit code on host, container sees it live) and for supplying config files. The host path must be **absolute** (use `$(pwd)` for current dir).
- An **anonymous volume** is created and managed by Docker under `/var/lib/docker/volumes/<random-id>/_data`. It outlives the container but has an unreadable name, so it is easy to lose track of.
- Mounting over a non-empty container directory **hides** the image's original content at that path.
- Permission trouble is common: files created in the container as root appear root-owned on the host.

Named volumes, the `docker volume` commands, tmpfs, and when to choose which are covered in lesson 32.

### Environment variables: `-e` and `--env-file`

Twelve-factor style: configuration comes from the environment, not from files baked into the image. The same image runs in dev and production with different variables.

```bash
docker run -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=shop postgres:16
docker run --env-file ./app.env myimage
```

`app.env` format: one `KEY=VALUE` per line, no quotes needed, `#` comments allowed, no `export`.

```
POSTGRES_USER=shop
POSTGRES_PASSWORD=s3cret
POSTGRES_DB=shopdb
```

`-e KEY` (without a value) forwards the variable from your current shell. Values given with `-e` override those from `--env-file` and from the image defaults. Environment variables are visible via `docker inspect` and to anyone with docker access — do not treat them as strongly secret (proper secret handling comes in Kubernetes lessons).

### Container lifecycle

```
            docker create          docker start
   image ───────────────► Created ─────────────► Running ◄──┐
                                                   │  │     │ docker restart / unpause
                                    docker pause   │  │ docker stop (SIGTERM, then SIGKILL after 10 s)
                                       ▼           │  ▼     │
                                    Paused         │ Exited (stopped)
                                                   │  │
                                     docker kill (SIGKILL)  docker rm
                                                      ▼
                                                   Removed
```

States visible in `docker ps -a`: `Created`, `Up`, `Up (Paused)`, `Exited (code)`, `Restarting`.

- `docker stop` sends **SIGTERM** (Lesson 8) to PID 1, waits 10 seconds (change with `-t`), then sends **SIGKILL**. An app that ignores SIGTERM always takes 10 s to stop.
- `docker kill` sends SIGKILL immediately (or `-s SIGNAL`).
- `docker rm` deletes a **stopped** container and its writable layer (`-f` also kills a running one). `docker rm` on a container does **not** remove the image.
- `docker start` on a stopped container re-runs it with the *same* settings and the *same* writable layer; data written earlier is still there. `docker run` always creates a **new** container.
- The **exit code** in `Exited (N)` tells why: `0` success, `1` app error, `137` killed by SIGKILL (128+9, often OOM or `docker kill`), `143` terminated by SIGTERM (128+15), `127` command not found, `126` not executable.

### Inspecting containers

- `docker logs` shows what PID 1 wrote to stdout/stderr. This is why containerized apps log to stdout rather than to files. nginx's official image symlinks its log files to `/dev/stdout` and `/dev/stderr` for exactly this reason.
- `docker inspect` dumps full JSON: IP address, mounts, env, ports, state, restart count, health. Use `--format` (Go template) to extract a field.
- `docker stats` shows live CPU %, memory usage/limit, network I/O, block I/O, and PID count per container (reads cgroup data).
- `docker top` lists processes in a container (as seen from the host).
- `docker exec` runs an additional command in a *running* container's namespaces (`-it` for a shell). It does not restart anything.
- `docker port`, `docker diff` (what changed in the writable layer), `docker cp` (copy files host ↔ container).

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `docker version` | Client and server versions | `docker version` |
| `docker info` | Daemon details (storage driver, cgroups, counts) | `docker info \| grep -i "storage driver"` |
| `docker pull IMAGE[:TAG]` | Download an image | `docker pull nginx:1.27` |
| `docker images` / `docker image ls` | List local images | `docker images` |
| `docker image inspect` | Image metadata JSON | `docker image inspect nginx --format '{{.Config.ExposedPorts}}'` |
| `docker history IMAGE` | Show layers of an image and their size | `docker history nginx:1.27` |
| `docker search TERM` | Search Docker Hub | `docker search --limit 5 redis` |
| `docker rmi IMAGE` / `docker image rm` | Remove local image | `docker rmi nginx:1.27` |
| `docker run IMAGE` | Create + start container | `docker run -d --name web -p 8080:80 nginx` |
| `docker run -it IMAGE sh` | Interactive shell | `docker run -it --rm alpine sh` |
| `docker run --rm IMAGE CMD` | One-off command, auto-cleanup | `docker run --rm alpine cat /etc/os-release` |
| `docker ps` | Running containers | `docker ps` |
| `docker ps -a` | All containers incl. stopped | `docker ps -a` |
| `docker ps -q` | Only IDs (for scripting) | `docker stop $(docker ps -q)` |
| `docker ps --filter` / `--format` | Filter / custom columns | `docker ps -a --filter status=exited --format '{{.Names}} {{.Status}}'` |
| `docker create IMAGE` | Create without starting | `docker create --name c1 nginx` |
| `docker start NAME` | Start stopped container | `docker start web` |
| `docker stop NAME` | SIGTERM, then SIGKILL after 10 s | `docker stop -t 30 web` |
| `docker restart NAME` | Stop + start | `docker restart web` |
| `docker pause` / `unpause` | Freeze / unfreeze processes (cgroup freezer) | `docker pause web` |
| `docker kill NAME` | SIGKILL immediately | `docker kill web` |
| `docker rm NAME` | Remove stopped container | `docker rm web` |
| `docker rm -f NAME` | Force-remove running container | `docker rm -f web` |
| `docker container prune` | Remove all stopped containers | `docker container prune -f` |
| `docker logs NAME` | Show output | `docker logs web` |
| `docker logs -f --tail 50 --since 10m NAME` | Follow, last lines, time window | `docker logs -f --tail 50 web` |
| `docker exec -it NAME CMD` | Command in running container | `docker exec -it web bash` |
| `docker inspect NAME` | Full JSON details | `docker inspect -f '{{.NetworkSettings.IPAddress}}' web` |
| `docker stats` | Live resource usage | `docker stats --no-stream` |
| `docker top NAME` | Processes inside | `docker top web` |
| `docker port NAME` | Show port mappings | `docker port web` |
| `docker diff NAME` | Files changed in writable layer | `docker diff web` |
| `docker cp SRC DEST` | Copy files host ↔ container | `docker cp web:/etc/nginx/nginx.conf .` |
| `docker rename OLD NEW` | Rename container | `docker rename web web1` |
| `docker system df` | Disk used by images/containers/volumes | `docker system df` |
| `docker system prune` | Remove stopped containers, dangling images, unused networks | `docker system prune` |
| `docker image prune -a` | Remove all unused images | `docker image prune -a` |

### `docker run` flag cheat list (this lesson)

| Flag | Example |
|---|---|
| `-d` | `docker run -d nginx` |
| `-it` | `docker run -it ubuntu:24.04 bash` |
| `--name` | `--name db` |
| `--rm` | `docker run --rm alpine echo hi` |
| `-p` | `-p 127.0.0.1:5432:5432` |
| `-v` | `-v $(pwd)/html:/usr/share/nginx/html:ro` |
| `-e` | `-e MYSQL_ROOT_PASSWORD=secret` |
| `--env-file` | `--env-file .env` |
| `--restart` | `--restart unless-stopped` |
| `--hostname` | `--hostname web01` |
| `-w` | `-w /app` (working directory) |
| `-u` | `-u 1000:1000` (run as UID:GID) |

## Examples / Walkthrough

### 1. Install check and hello-world

```bash
docker version
docker run hello-world          # pulls image, runs, prints message, exits
docker ps                       # empty: container already exited
docker ps -a                    # shows hello-world with Exited (0)
docker images                   # hello-world image is now local (~10 kB)
```

Narrate what happened: client → daemon → no local image → pull from Docker Hub → create → start → output streamed back → PID 1 exited → container stopped.

### 2. Pull and explore images

```bash
docker pull alpine:3.20
docker pull nginx:1.27
docker images
docker history nginx:1.27                      # one row per layer
docker image inspect alpine:3.20 --format '{{.Os}}/{{.Architecture}}  {{.Size}}'
docker search --limit 5 nginx
```

Compare sizes: `alpine` ≈ 8 MB, `nginx` ≈ 190 MB, `ubuntu` ≈ 80 MB. Containers are not automatically small — the image content decides.

### 3. One-off and interactive containers

```bash
docker run --rm alpine echo "hello from a container"
docker run --rm alpine cat /etc/os-release          # container thinks it's Alpine
cat /etc/os-release                                  # host is still Ubuntu: same kernel, different userland
docker run --rm alpine uname -r                      # same kernel version as host: shared kernel!

docker run -it --name sandbox ubuntu:24.04 bash
# inside the container:
hostname                    # container ID
ps aux                      # only bash — PID namespace
apt update && apt install -y curl
echo "created in container" > /tmp/note.txt
exit                        # bash (PID 1) exits → container stops

docker ps -a                # sandbox: Exited (0)
docker start -ai sandbox    # attach again
cat /tmp/note.txt           # file still there: writable layer survives stop
exit
docker rm sandbox           # now it is gone, along with note.txt
```

### 4. Detached web server with port mapping

```bash
docker run -d --name web -p 8080:80 nginx:1.27
docker ps
curl -I http://localhost:8080          # HTTP/1.1 200 OK, Server: nginx/1.27.x
docker port web                        # 80/tcp -> 0.0.0.0:8080
docker logs web
docker logs -f web                     # in second terminal run curl; see access log lines; Ctrl+C
docker top web                         # nginx master + workers, as host sees them
ps aux | grep "nginx: master"          # also visible on the host!
```

Show the second instance on another port: `docker run -d --name web2 -p 8081:80 nginx:1.27` — both listen on 80 internally, different host ports.

### 5. exec: debug a running container

```bash
docker exec web nginx -v                          # one command
docker exec -it web bash                          # shell
  ls /etc/nginx/conf.d/
  cat /etc/nginx/conf.d/default.conf
  apt update && apt install -y procps && ps aux
  exit
docker exec web cat /etc/hostname
docker inspect -f '{{.NetworkSettings.IPAddress}}' web      # e.g. 172.17.0.2
curl -s http://172.17.0.2                                   # container IP reachable from the host
```

### 6. Bind-mount a website

```bash
mkdir -p ~/site && echo "<h1>Served from the host</h1>" > ~/site/index.html
docker rm -f web
docker run -d --name web -p 8080:80 -v ~/site:/usr/share/nginx/html:ro nginx:1.27
curl http://localhost:8080                       # your page
echo "<h1>Edited live</h1>" > ~/site/index.html
curl http://localhost:8080                       # new content immediately, no restart
docker exec web sh -c 'echo hack > /usr/share/nginx/html/x'   # fails: read-only (:ro)
docker inspect -f '{{json .Mounts}}' web
```

Also mount one config file: `-v ~/my.conf:/etc/nginx/conf.d/default.conf:ro` and `docker exec web nginx -s reload` (reuses Lesson 13 knowledge).

### 7. Environment variables and a database

```bash
docker run -d --name db \
  -e POSTGRES_USER=shop \
  -e POSTGRES_PASSWORD=s3cret \
  -e POSTGRES_DB=shopdb \
  -p 127.0.0.1:5432:5432 \
  postgres:16

docker logs db | tail -5                         # "database system is ready to accept connections"
docker exec -it db psql -U shop -d shopdb -c '\l'
docker exec db env | grep POSTGRES
```

Same thing with an env file:

```bash
cat > db.env <<'EOF'
POSTGRES_USER=shop
POSTGRES_PASSWORD=s3cret
POSTGRES_DB=shopdb
EOF
docker rm -f db
docker run -d --name db --env-file db.env -p 127.0.0.1:5432:5432 postgres:16
```

Show the failure case: run `docker run -d --name bad postgres:16` with no password → `docker ps -a` shows `Exited (1)`; `docker logs bad` explains "Database is uninitialized and superuser password is not specified". Lesson: **always read `docker logs` first when a container exits.**

### 8. Data lives in the writable layer — and dies with the container

```bash
docker run -d --name db2 -e POSTGRES_PASSWORD=pw postgres:16
docker exec db2 psql -U postgres -c "CREATE TABLE t(id int);"
docker rm -f db2                                  # data gone with the container
docker run -d --name db2 -e POSTGRES_PASSWORD=pw postgres:16
docker exec db2 psql -U postgres -c '\dt'         # "Did not find any relations"
```

Why: the postgres image declares an anonymous volume at `/var/lib/postgresql/data`, but `docker rm` without `-v` leaves it orphaned and the *new* container gets a *new* anonymous volume. Fix with a bind mount `-v ~/pgdata:/var/lib/postgresql/data` (try it). Named volumes in lesson 32.

### 9. Lifecycle tour

```bash
docker create --name life nginx:1.27        # Created
docker ps -a --filter name=life
docker start life                           # Up
docker pause life                           # Up (Paused): curl hangs, process frozen
docker unpause life
docker stop life                            # SIGTERM → nginx exits gracefully fast
docker ps -a --format '{{.Names}}\t{{.Status}}'
docker start life
docker kill life                            # SIGKILL → Exited (137)
docker rm life
```

Compare `time docker stop` for nginx (instant, handles SIGTERM) versus `docker run -d --name slow alpine sleep 1000` (takes ~10 s because `sleep` as PID 1 ignores SIGTERM without a handler). Explain PID 1 signal behavior.

### 10. Restart policy

```bash
docker run -d --name keep --restart unless-stopped nginx:1.27
sudo systemctl restart docker
docker ps                                   # keep came back automatically
docker run -d --name crashy --restart on-failure:3 alpine sh -c 'exit 1'
sleep 5; docker ps -a; docker inspect -f '{{.RestartCount}}' crashy
```

### 11. Monitoring and cleanup

```bash
docker stats --no-stream
docker stats web                            # live; Ctrl+C
docker system df
docker ps -a --filter status=exited -q | xargs -r docker rm
docker container prune -f
docker rmi alpine:3.20                      # fails if a container uses the image
docker image prune                          # dangling only
docker system prune                         # asks confirmation
```

Generate load to see numbers move: `docker exec web sh -c 'while true; do :; done'` in one terminal (stats CPU rises), Ctrl+C to stop.

## Common Pitfalls

- **Container exits immediately** — PID 1 finished (e.g. `docker run ubuntu` runs `bash` with no TTY). Use `-it` for shells, `-d` plus a long-running command for services. Always check `docker logs NAME` and `docker ps -a`.
- **Using `docker run` when you meant `docker start`** — `run` always creates a new container; repeated runs pile up stopped ones and cause `Conflict. The container name "/web" is already in use`. Use `docker start web`, or `docker rm web` first, or add `--rm`.
- **Reversed port mapping** — `-p 80:8080` publishes host 80 → container 8080. Remember host:container, outside:inside.
- **Application listens on 127.0.0.1 inside the container** — published ports then do not work, because traffic arrives on the container's `eth0`. The app must bind `0.0.0.0`.
- **`-p` bypasses ufw** — a published port is open to the world even if ufw denies it. Bind to `127.0.0.1:` or firewall via the `DOCKER-USER` iptables chain.
- **Losing data on `docker rm`** — everything in the writable layer is deleted. Mount a host path or volume for databases and uploads.
- **Relying on `latest`** — the tag moves; two hosts pulling at different times run different versions. Pin versions.
- **Mounting a host path that does not exist** — with `-v /nope:/data` Docker creates `/nope` as a root-owned *directory* on the host. With `--mount` syntax it errors instead. Create paths first.
- **Relative paths in `-v`** — `-v ./site:/x` is interpreted as a *named volume* called `./site` in old Docker; use `$(pwd)/site` or an absolute path.
- **`-v` over a populated directory** hides the image's files there (e.g. mounting an empty dir over `/usr/share/nginx/html` gives 403/404).
- **Permission denied on bind mounts** — container user's UID differs from the host file owner. Check `ls -ln`, fix ownership, or run with `-u $(id -u):$(id -g)`.
- **`permission denied while trying to connect to the Docker daemon socket`** — your user is not in the `docker` group (re-login after `usermod -aG`), or prefix `sudo`.
- **Port already allocated** — `Bind for 0.0.0.0:8080 failed: port is already allocated`: another container or host process (nginx from Lesson 13!) uses the port. Check `docker ps`, `ss -tlnp`.
- **Editing files inside a container to "fix" it** — the change vanishes when the container is recreated. Fix the image or mount config instead.
- **Putting secrets in `-e` on the command line** — they land in shell history and `docker inspect`. Prefer `--env-file` with `chmod 600`, and never commit it to Git (Lesson 14).
- **Disk fills up** — stopped containers, old images and orphaned volumes accumulate in `/var/lib/docker`. Check `docker system df` and prune regularly. Container logs under the default `json-file` driver also grow without limit unless rotation is configured.
- **Treating a container like a VM** — SSH-ing in, installing packages by hand, running multiple services. Containers should be one process per container, disposable and recreated from the image.
- **Running as root inside** — default user in most images is root. It is still root-equivalent for many attacks; hardening is covered in lessons 31 and 32.

## FAQ

**Q: What is the difference between a container and a virtual machine?**
A: A VM emulates hardware and runs a complete guest OS with its own kernel. A container is just an isolated process on the host that shares the host kernel. Containers start in a second and are megabytes in size; VMs boot in minutes and are gigabytes. VMs isolate more strongly, containers are lighter.

**Q: Is Docker the same as a container?**
A: No. A container is a Linux kernel concept (namespaces + cgroups). Docker is a tool/platform that builds, distributes and runs containers. Alternatives: Podman, containerd, CRI-O, LXC.

**Q: Can I run Windows containers or macOS apps in Linux Docker containers?**
A: No. Containers share the host kernel; a Linux container needs a Linux kernel. On Windows/macOS, Docker Desktop runs a hidden Linux VM.

**Q: What is the difference between an image and a container?**
A: Image: read-only template of layers. Container: a running or stopped instance of an image with its own writable layer. You can start many containers from one image.

**Q: What does `docker run` do step by step?**
A: Looks for the image locally; if absent, pulls it from the registry; creates a container (writable layer, namespaces, network); starts the image's default command (or the one you gave) as PID 1.

**Q: What is the difference between `docker run`, `docker start` and `docker exec`?**
A: `run` creates a **new** container from an image and starts it. `start` restarts an **existing** stopped container. `exec` runs an extra command inside an **already running** container.

**Q: What is the difference between `-d` and `-it`?**
A: `-d` runs in the background (servers). `-it` attaches your terminal for interactive use (shells). They can be combined (`-dit`) to keep an idle shell container alive.

**Q: Why did my container stop right after starting?**
A: Its main process ended or crashed. Run `docker ps -a` for the exit code and `docker logs NAME` for the reason. A container with nothing long-running to do exits by design.

**Q: How do I keep a container running for debugging?**
A: Give it a long-running process: `docker run -d --name dbg alpine sleep infinity`, then `docker exec -it dbg sh`.

**Q: What is the difference between `docker stop` and `docker kill`?**
A: `stop` sends SIGTERM, waits (default 10 s) so the app can shut down cleanly, then SIGKILL. `kill` sends SIGKILL immediately with no cleanup.

**Q: What is `docker attach` vs `docker exec`?**
A: `attach` connects your terminal to PID 1's stdin/stdout — exiting or Ctrl+C can stop the container (detach with Ctrl+P, Ctrl+Q). `exec` starts a separate new process and is almost always what you want.

**Q: Where are images and containers stored?**
A: Under `/var/lib/docker` (by default): `overlay2/` for layers, `containers/` for per-container metadata and logs, `volumes/` for volumes. Check with `docker info | grep "Docker Root Dir"`.

**Q: What does `-p 8080:80` mean?**
A: Forward host port 8080 to container port 80. Browse `http://HOST:8080`. Left = host, right = container.

**Q: Do I need `EXPOSE` or `-p` to reach a container?**
A: `-p` (or `-P`) is what actually publishes a port. `EXPOSE` in an image is only documentation (covered in lesson 31). Containers on the same Docker network can still reach each other's ports without publishing (lesson 32).

**Q: How do two containers talk to each other?**
A: By container IP on the default bridge network, or by name on user-defined networks. The name-based approach is covered in lesson 32; for now, publish ports and use the host.

**Q: What is the difference between a bind mount and an anonymous volume?**
A: A bind mount uses an exact host path you choose. An anonymous volume is storage Docker creates for you with a random name. Named volumes (a better version of anonymous ones) are in lesson 32.

**Q: Why is my data gone after removing the container?**
A: It was in the container's writable layer. Persist with `-v` to a host path or volume.

**Q: How do I see the logs of a stopped container?**
A: `docker logs NAME` works on stopped containers as long as they are not removed.

**Q: How do I get a shell in a minimal image that has no bash?**
A: Try `sh` (`docker exec -it NAME sh`). Alpine has `sh` (busybox); distroless images have no shell at all.

**Q: What does `docker ps` show vs `docker ps -a`?**
A: `ps` = running only; `ps -a` = all containers including exited and created.

**Q: How do I remove all stopped containers / all containers?**
A: Stopped: `docker container prune`. Everything including running: `docker rm -f $(docker ps -aq)` — destructive, double-check the host first.

**Q: How do I free disk space?**
A: `docker system df` to see usage, then `docker container prune`, `docker image prune -a`, or `docker system prune`. Add `--volumes` to `system prune` only if you are sure — it deletes unused volumes' data.

**Q: What does exit code 137 mean?**
A: 128 + 9: the process received SIGKILL — `docker kill`, a `docker stop` timeout, or the kernel OOM killer.

**Q: Is it safe to give users access to the `docker` group?**
A: It is equivalent to giving them root on the host, because they can mount `/` into a container. Only add trusted admins.

**Q: What is `containerd` and do I need to care?**
A: The runtime daemon `dockerd` delegates to. You rarely touch it, but Kubernetes (lessons 34+) uses it directly, so the name will come back.

**Q: How do I run a container on a specific CPU architecture?**
A: Use `--platform linux/amd64` (or `linux/arm64`). Mismatched architecture needs emulation or gives `exec format error`.

**Q: Can I limit how much RAM or CPU a container uses?**
A: Yes, via cgroups with `--memory`, `--cpus` — covered in lesson 32. Today `docker stats` only monitors.

**Q: How do I change the environment variable of a running container?**
A: You cannot. Recreate the container (`docker rm -f`, then `docker run` with new `-e`).

## Practice / Exercise

### Core (all students)

1. Install Docker Engine from the official apt repo, add your user to the `docker` group, and verify with `docker run hello-world`. Record the output of `docker info | grep -E "Storage Driver|Cgroup|Docker Root Dir"`.
2. Pull `alpine:3.20`, `nginx:1.27`, `ubuntu:24.04`. List them with `docker images` and note sizes. Use `docker history nginx:1.27` and count the layers.
3. Prove the shared kernel: compare `uname -r` on the host with `docker run --rm alpine uname -r`. Show the container's `/etc/os-release` differs from the host's.
4. Start `nginx:1.27` detached as `web` mapped to host port `8080`. Confirm with `curl -I localhost:8080`, `docker ps`, `docker port web`. Read `docker logs web` after a few requests.
5. Find the nginx master process in the host's `ps aux` output and in `docker top web`. Explain why it appears in both.
6. Replace the default page by bind-mounting `~/site/index.html` read-only. Edit the file on the host and verify the change without restarting. Try writing to the mount from inside and observe the error.
7. Run `postgres:16` with `-e` variables, then repeat with `--env-file`. Connect with `docker exec -it db psql ...`. Then start one **without** a password, read `docker logs` and explain the exit code.
8. Walk a container through the full lifecycle: `create` → `start` → `pause` → `unpause` → `stop` → `start` → `kill` → `rm`. After each step run `docker ps -a` and note the status text and exit code.
9. Start an interactive Ubuntu container, create a file, exit, re-`start` it and show the file is still there; then `rm` it and show the data is gone.
10. Clean up: remove all stopped containers, run `docker system df` before and after, and prune dangling images.

### Stretch

1. Run two nginx containers on ports 8081 and 8082 with different bind-mounted pages; put the host nginx from Lesson 13 in front of both as a load balancing `upstream` with `proxy_pass http://127.0.0.1:8081` and `:8082`.
2. Use `unshare --pid --fork --mount-proc bash` (and `--net`, `--uts`) to build a "poor man's container" and show `ps aux` / `hostname` isolation. Find a running container's cgroup directory under `/sys/fs/cgroup/system.slice/` and read `memory.current`.
3. Use `docker inspect --format` to print for one container: its IP address, its mounts, its restart policy, and its start time. Write a shell one-liner (Lesson 16) listing every container name with its IP.
4. Demonstrate the ufw bypass: `sudo ufw enable`, `ufw deny 9090`, run a container with `-p 9090:80`, and reach it from another machine. Then rebind to `127.0.0.1:9090:80` and show the difference.
5. Find the overlay2 directory of a container with `docker inspect -f '{{.GraphDriver.Data.UpperDir}}' NAME`, create a file inside the container and locate it on the host filesystem (needs sudo). Relate it to `docker diff`.
6. Demonstrate `docker stop` timing: compare `time docker stop` for nginx versus `alpine sleep 1000`. Explain the PID 1 signal handling difference and test `docker run --init`.
7. Set `--restart unless-stopped`, reboot or restart the Docker service, and check which containers come back, then explain `always` vs `unless-stopped`.

## Further Reading

- Docker overview and architecture: https://docs.docker.com/get-started/docker-overview/
- Install Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- Post-install steps for Linux (non-root, startup): https://docs.docker.com/engine/install/linux-postinstall/
- `docker run` reference: https://docs.docker.com/reference/cli/docker/container/run/
- Docker storage drivers / overlay2: https://docs.docker.com/engine/storage/drivers/overlayfs-driver/
- Linux namespaces: `man 7 namespaces`; cgroups v2: `man 7 cgroups`
- Open Container Initiative specs: https://opencontainers.org/
- Docker and iptables (why `-p` bypasses ufw): https://docs.docker.com/engine/network/packet-filtering-firewalls/
