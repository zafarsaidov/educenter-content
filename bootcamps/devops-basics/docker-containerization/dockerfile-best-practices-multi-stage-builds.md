# Lesson 31: Dockerfile Best Practices & Multi-stage Builds

**Module:** Docker – Containerization
**Duration:** 120-150 min
**Prerequisites:** Lesson 30 (container fundamentals, `docker run`, images vs containers, `docker ps`, `docker logs`, port mapping)

## Learning Objectives

By end of lesson student can:
- Write a Dockerfile from scratch using FROM, RUN, COPY, ADD, WORKDIR, ENV, ARG, EXPOSE, USER, CMD, ENTRYPOINT
- Explain how image layers and the build cache work, and order instructions to maximize cache hits
- Use a `.dockerignore` file to shrink build context and avoid leaking secrets
- Choose a minimal base image (slim, alpine, distroless, scratch) and explain trade-offs
- Run containers as a non-root user and combine RUN layers correctly
- Write a multi-stage Dockerfile that separates build and runtime stages (`AS`, `COPY --from`)
- Use `docker build -t`, `-f`, `--build-arg`, `--target`, `--no-cache` and inspect result with `docker history` and `docker images`
- Distinguish CMD from ENTRYPOINT, shell form from exec form, ARG from ENV

## Topics

- Dockerfile instructions: FROM, RUN, COPY, ADD, WORKDIR, ENV, ARG, EXPOSE, USER, CMD, ENTRYPOINT
- Layer caching: order matters, cache invalidation, `.dockerignore` file
- Best practices: minimal base images (alpine/distroless), non-root USER, combine RUN layers
- Multi-stage builds: `AS` alias, `COPY --from`, separating build and runtime stages
- Build options: `docker build -t`, `-f`, `--build-arg`, `--target`, `--no-cache`

## Concepts

### What a Dockerfile is

A **Dockerfile** is a plain-text recipe (file literally named `Dockerfile`, no extension) that tells Docker how to build an **image**. Each instruction is executed top to bottom. Most instructions create a new **layer** — a read-only filesystem diff stacked on top of the previous one. The final stack of layers is the image. A container is that image plus one thin writable layer.

Build flow:

```
Dockerfile + build context  --docker build-->  image (stack of layers)  --docker run-->  container
```

The **build context** is the directory (usually `.`) sent to the Docker daemon at build time. `COPY` can only read files from inside the context. Everything in the context is uploaded first — a huge context means a slow build (fix: `.dockerignore`).

Build engine: modern Docker uses **BuildKit** by default (parallel stages, better cache, `--mount` features). Output looks like `[internal] load build definition`, `[1/5] FROM ...`. Legacy builder prints `Step 1/5 : FROM ...`.

### Dockerfile instructions

#### FROM

Sets the **base image** and starts a build stage. Must be first instruction (only `ARG` may come before it).

```dockerfile
FROM python:3.12-slim
FROM node:20-alpine AS build      # named stage (multi-stage)
FROM scratch                      # empty image, for static binaries
```

- Always pin a tag (`python:3.12-slim`, not `python` / `latest`). `latest` is just a tag, not "newest guaranteed"; it moves under you and breaks reproducibility.
- For strict reproducibility pin by digest: `FROM python:3.12-slim@sha256:<digest>`.

#### RUN

Executes a command at **build time** inside a new layer and commits the result. Used to install packages, compile, create users.

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

Two forms:
- **Shell form**: `RUN apt-get update` → runs as `/bin/sh -c "apt-get update"`.
- **Exec form**: `RUN ["apt-get", "update"]` → no shell, no variable expansion, no `&&`.

RUN happens at build time. It does NOT run when the container starts (that is CMD/ENTRYPOINT).

#### COPY

Copies files from build context into the image.

```dockerfile
COPY package.json package-lock.json ./
COPY . .
COPY --chown=app:app src/ /app/src/
```

Syntax: `COPY <src>... <dest>`. Relative `dest` is relative to current WORKDIR. Source paths cannot escape the context (`../x` fails).

#### ADD

Like COPY plus two extras: auto-extracts local tar archives (`.tar`, `.tar.gz`, `.tgz`) and can fetch URLs.

```dockerfile
ADD app.tar.gz /opt/app/          # extracted automatically
ADD https://example.com/f.txt /f  # downloaded, NOT cached well, NOT extracted
```

**Rule of thumb: use COPY. Use ADD only when you need local tar auto-extraction.** For remote files prefer `RUN curl -fsSL ... | tar -xz` (so download + extract + cleanup fit in one layer) because ADD from URL adds the file to its own layer and you cannot delete it later to save space.

#### WORKDIR

Sets working directory for following RUN, CMD, ENTRYPOINT, COPY, ADD. Creates directory if missing.

```dockerfile
WORKDIR /app
```

Prefer WORKDIR over `RUN cd /app && ...` (`cd` in one RUN does not persist to the next instruction). Can be used multiple times; relative paths chain.

#### ENV

Sets environment variable available at **build time AND run time** (persisted in image config, visible in `docker inspect`).

```dockerfile
ENV NODE_ENV=production \
    PORT=3000
```

Can be overridden at run: `docker run -e PORT=8080 ...`. Never put secrets in ENV — they stay in image metadata.

#### ARG

Defines a **build-time-only** variable, set with `--build-arg`.

```dockerfile
ARG APP_VERSION=1.0.0
ARG BASE=3.12
FROM python:${BASE}-slim          # ARG before FROM is usable only in FROM line
ARG APP_VERSION                   # re-declare after FROM to use in stage
RUN echo "building $APP_VERSION"
```

```bash
docker build --build-arg APP_VERSION=2.1.0 -t myapp .
```

| | ARG | ENV |
|---|---|---|
| Available during build | yes | yes |
| Available in running container | **no** | yes |
| Set from CLI | `--build-arg` at build | `-e` at run (override) |
| Stored in image | name/default visible in `docker history` | in image config |

ARG values are NOT safe for secrets either: they show in `docker history`. Use BuildKit secret mounts (`RUN --mount=type=secret`) for secrets needed during build.

Trick: promote ARG to ENV when you want the value at runtime: `ARG VERSION` then `ENV APP_VERSION=$VERSION`.

#### EXPOSE

Documents the port the app listens on. **Does not publish the port.** Purely metadata (and used by `docker run -P`). Publishing is done with `docker run -p` (lesson 30).

```dockerfile
EXPOSE 8080
EXPOSE 53/udp
```

#### USER

Sets the user (and optionally group) for following RUN/CMD/ENTRYPOINT and for the running container. Default is **root** — risky: a compromised app has root inside the container.

```dockerfile
RUN groupadd -r app && useradd -r -g app app          # Debian/Ubuntu
# alpine: RUN addgroup -S app && adduser -S -G app app
USER app
```

Can also use numeric ID: `USER 10001:10001` (works with orchestrators that verify non-root by UID, and with images that lack passwd entries such as distroless/scratch).

#### CMD

Default command executed when container starts. **Only the last CMD counts.** Easily overridden by arguments to `docker run`.

```dockerfile
CMD ["python", "app.py"]
```

```bash
docker run myapp                 # runs python app.py
docker run myapp python other.py # CMD replaced
```

#### ENTRYPOINT

Defines the executable that always runs; `docker run` arguments are **appended** to it (unless `--entrypoint` is used to replace it).

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]            # default arguments to ENTRYPOINT
```

```bash
docker run myapp                  # python app.py --port 8000
docker run myapp --port 9000      # python app.py --port 9000
docker run --entrypoint sh -it myapp   # replace entrypoint (debugging)
```

CMD vs ENTRYPOINT matrix:

| Dockerfile | `docker run img` | `docker run img foo` |
|---|---|---|
| `CMD ["a","b"]` | `a b` | `foo` |
| `ENTRYPOINT ["a","b"]` | `a b` | `a b foo` |
| `ENTRYPOINT ["a"]` + `CMD ["b"]` | `a b` | `a foo` |

Mental model: **ENTRYPOINT = the program, CMD = its default arguments / default command.**

#### Shell form vs exec form (important)

```dockerfile
CMD python app.py          # shell form  -> /bin/sh -c "python app.py"
CMD ["python", "app.py"]   # exec form   -> python is PID 1
```

- Shell form: `sh` is PID 1, your app is its child. `docker stop` sends SIGTERM to `sh`, which does **not** forward it → app is killed after 10 s timeout with SIGKILL (no graceful shutdown).
- Exec form: app is PID 1, receives SIGTERM directly → graceful shutdown. **Always use exec form (JSON array, double quotes) for CMD and ENTRYPOINT.**
- Exec form does not do shell things: no `$VAR` expansion, no `&&`, no pipes. If you need them: `CMD ["sh", "-c", "echo $HOME && exec python app.py"]` (note `exec` so app replaces the shell).

#### Other instructions (brief)

- `LABEL maintainer="team@example.com" version="1.0"` — image metadata (key=value).
- `HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1` — tells Docker how to test container health; shows in `docker ps` as `(healthy)`.
- `SHELL ["/bin/bash", "-c"]` — changes default shell for shell form.
- `STOPSIGNAL SIGINT` — signal sent by `docker stop`.
- `VOLUME` — declares a mount point; data persistence is covered in lesson 32, avoid until then.

### Layers and the build cache

Each RUN, COPY, ADD creates a layer. Docker caches every layer. On rebuild it walks instructions top to bottom: if an instruction (and its inputs) is unchanged and all previous layers were cache hits, it reuses the cached layer (`CACHED` in BuildKit output). **At the first changed instruction, the cache is invalidated for that instruction and every instruction after it.**

What invalidates cache:
- RUN: the command text changed (Docker does NOT detect that `apt-get update` would return newer packages).
- COPY/ADD: the **content checksum** of copied files changed (mtime ignored).
- Base image (FROM) changed — new digest after pull.
- ARG value changed (for instructions that use it).

Golden rule: **put things that change rarely at top, things that change often at bottom.**

Bad (any source edit reinstalls dependencies every build):

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm ci
CMD ["node", "server.js"]
```

Good (dependencies only reinstall when package files change):

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .
CMD ["node", "server.js"]
```

Same idea for Python (`requirements.txt` first), Go (`go.mod go.sum` + `go mod download` first), Java (`pom.xml` first).

Demo to show live: build twice, second build is all `CACHED`; touch `server.js`, rebuild — only the last COPY re-runs; edit `package.json` — `npm ci` re-runs.

Cache gotcha: `RUN apt-get update` on its own line is cached forever → later `apt-get install` fails with 404 on stale lists. Always chain `update && install` in the same RUN.

### The .dockerignore file

File named `.dockerignore` in the context root. Same idea as `.gitignore`: listed paths are **excluded from the build context** (not sent to daemon, cannot be COPY'd).

```
.git
node_modules
**/__pycache__
*.pyc
.env
.env.*
*.log
Dockerfile
.dockerignore
docker-compose*.yml
dist
coverage
```

Benefits:
1. Faster builds (smaller context upload — watch the line `transferring context: 1.2MB` in output).
2. Better cache (changing a log file or `.git` does not invalidate `COPY . .`).
3. Security (`.env`, SSH keys, cloud credentials never end up in image layers).
4. Smaller images and no host `node_modules` overwriting the container's freshly installed ones.

Patterns: `*` matches in one dir level, `**` any depth, `!keep.txt` re-includes (exception). Allow-list style: `*` then `!src` `!package*.json`.

### Image size and minimal base images

Smaller image = faster pull/deploy, smaller attack surface, fewer CVEs.

| Base | Approx size | Notes |
|---|---|---|
| `ubuntu:24.04` | ~78 MB | full-featured, familiar, apt |
| `debian:bookworm-slim` | ~75 MB | glibc, apt, good default |
| `python:3.12` (full) | ~1 GB | includes compilers, headers |
| `python:3.12-slim` | ~130 MB | recommended default for Python |
| `node:20` (full) | ~1.1 GB | |
| `node:20-alpine` | ~135 MB | musl libc |
| `alpine:3.20` | ~8 MB | musl libc, `apk`, busybox |
| `gcr.io/distroless/static` | ~2 MB | no shell, no package manager |
| `scratch` | 0 MB | empty; only for static binaries |

**alpine**: tiny, uses musl instead of glibc → some prebuilt binaries/wheels (Python manylinux wheels, some Node native modules) fail or compile slowly. If you hit weird DNS/locale/segfault issues, switch to `-slim` (Debian).

**distroless** (Google): contains only app runtime (e.g. `gcr.io/distroless/python3`, `.../nodejs20`, `.../static`, `.../base`). No shell, no apt, no curl → very small attack surface; cannot `docker exec -it ... sh` (use `:debug` tag, which includes busybox shell, for troubleshooting). Tags `:nonroot` run as UID 65532.

**scratch**: `FROM scratch` + a statically linked binary (Go with `CGO_ENABLED=0`). Need to copy CA certs yourself if the app makes HTTPS calls.

Choosing: start with `-slim`; go alpine/distroless when you understand the trade-offs; use multi-stage to get a small runtime image with a big build image.

### Best practices checklist

1. **Pin base image tags** (never rely on `latest`).
2. **Minimal base** (`-slim`, alpine, distroless).
3. **Order for cache** — dependency manifests first, source last.
4. **Combine related RUN** commands with `&&` and clean up in the **same** layer. Deleting files in a later layer does NOT shrink the image — lower layers still contain them.
5. **`--no-install-recommends`** (apt) / `--no-cache` (apk) / `--no-cache-dir` (pip) to skip caches.
6. **Run as non-root** with USER.
7. **One process per container**; app in foreground, logs to stdout/stderr (so `docker logs` works).
8. **Exec form** for CMD/ENTRYPOINT.
9. **Use `.dockerignore`.**
10. **Multi-stage builds** to keep compilers/dev tools out of final image.
11. **No secrets** in ENV, ARG, COPY'd files, or layers (layers are permanent and extractable).
12. **COPY not ADD**, specific paths not blanket `COPY . .` when possible.
13. **Explicit WORKDIR**, absolute paths.
14. Lint with `hadolint Dockerfile` (optional tool, catches many of the above).

#### Combining RUN layers

Bad: 3 layers, apt lists stay in layer 1/2:

```dockerfile
RUN apt-get update
RUN apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*
```

Good: 1 layer, cleaned:

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
```

Trade-off: too much combining hurts cache granularity and readability. Group by purpose (install system packages together; app deps in their own step).

#### Non-root user full example

```dockerfile
FROM python:3.12-slim
RUN useradd --create-home --uid 10001 app
WORKDIR /home/app
COPY --chown=app:app requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=app:app . .
USER app
CMD ["python", "app.py"]
```

Notes: install packages as root first, switch with USER near the end; `COPY --chown` gives the user ownership (plain COPY files are owned by root); ports < 1024 cannot be bound by non-root → use 8080 not 80.

### Multi-stage builds

Problem: to compile an app you need compilers, SDKs, dev dependencies (hundreds of MB to GB), but to run it you need only the artifact. Single-stage images ship all the build tooling → big and insecure.

**Multi-stage build** = several `FROM` lines in one Dockerfile. Each FROM starts a fresh stage with its own filesystem. You `COPY --from=<stage>` only the needed artifacts into the final stage. Everything else (earlier stages) is discarded and not part of the final image.

Syntax:
- `FROM golang:1.22 AS builder` — name a stage with `AS`.
- `COPY --from=builder /src/app /app` — copy from named stage (or by index `--from=0`).
- `COPY --from=nginx:alpine /etc/nginx/nginx.conf /tmp/` — can even copy from an external image.
- Last stage is the default build result; `--target` selects an earlier one.

#### Go example (dramatic size win)

```dockerfile
# ---- build stage ----
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /out/server ./cmd/server

# ---- runtime stage ----
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /out/server /server
EXPOSE 8080
USER nonroot
ENTRYPOINT ["/server"]
```

Result: ~15 MB vs ~600 MB single-stage with Go toolchain.

#### Node example (build front-end, serve static)

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build               # outputs /app/dist

FROM nginx:1.27-alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

Final image has no Node, no node_modules — just nginx and static files.

#### Python example (build wheels, slim runtime)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /build
RUN apt-get update && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

FROM python:3.12-slim
RUN useradd --create-home --uid 10001 app
WORKDIR /home/app
COPY --from=builder /wheels /wheels
COPY requirements.txt .
RUN pip install --no-cache-dir --no-index --find-links=/wheels -r requirements.txt \
    && rm -rf /wheels
COPY --chown=app:app . .
USER app
CMD ["python", "app.py"]
```

Compilers (`build-essential`) exist only in the builder stage.

#### Multiple targets in one Dockerfile

```dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

FROM base AS dev
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

FROM base AS test
RUN npm ci
COPY . .
RUN npm test

FROM base AS prod
RUN npm ci --omit=dev
COPY . .
USER node
CMD ["node", "server.js"]
```

```bash
docker build --target dev  -t myapp:dev  .
docker build --target test -t myapp:test .   # build fails if tests fail
docker build --target prod -t myapp:prod .   # (also default: last stage)
```

BuildKit builds only the stages the chosen target depends on (skips unused ones).

### Build options

`docker build [OPTIONS] PATH` — PATH is the context directory (`.`).

| Option | Meaning |
|---|---|
| `-t name:tag` | Name and tag the result; repeatable (`-t app:1.0 -t app:latest`) |
| `-f path` | Use a Dockerfile with another name/location (`-f docker/Dockerfile.prod`); context still PATH |
| `--build-arg K=V` | Set ARG value |
| `--target stage` | Stop at named stage |
| `--no-cache` | Ignore cache for the whole build |
| `--pull` | Always attempt to pull newer base image |
| `--progress=plain` | Full un-collapsed build output (debugging) |
| `-q` | Quiet, print only image ID |

Tag defaults to `latest` if omitted. Inspect results:

```bash
docker images                     # sizes
docker history myapp:1.0          # layers, sizes, instructions that created them
docker image inspect myapp:1.0    # config: user, cmd, entrypoint, env, layers
```

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `docker build -t` | Build image and name it | `docker build -t myapp:1.0 .` |
| `docker build -f` | Use non-default Dockerfile | `docker build -f Dockerfile.prod -t myapp:prod .` |
| `docker build --build-arg` | Pass ARG value | `docker build --build-arg APP_VERSION=2.0 -t myapp .` |
| `docker build --target` | Build up to a stage | `docker build --target builder -t myapp:build .` |
| `docker build --no-cache` | Rebuild without cache | `docker build --no-cache -t myapp .` |
| `docker build --pull` | Refresh base image | `docker build --pull -t myapp .` |
| `docker build --progress=plain` | Verbose output | `docker build --progress=plain -t myapp .` |
| `docker images` | List local images and sizes | `docker images myapp` |
| `docker history` | Show layers of an image | `docker history --no-trunc myapp:1.0` |
| `docker image inspect` | Image metadata JSON | `docker image inspect -f '{{.Config.User}}' myapp` |
| `docker run --entrypoint` | Override ENTRYPOINT | `docker run --rm -it --entrypoint sh myapp` |
| `docker builder prune` | Clear build cache | `docker builder prune -f` |
| `docker image prune` | Remove dangling images | `docker image prune -f` |
| `docker rmi` | Remove an image | `docker rmi myapp:1.0` |

Instruction cheat sheet:

| Instruction | When it runs | Key point |
|---|---|---|
| FROM | build | base image, starts stage |
| RUN | build | new layer, shell/exec form |
| COPY | build | context → image, prefer over ADD |
| ADD | build | COPY + tar extract + URL |
| WORKDIR | build+run | sets cwd, creates dir |
| ENV | build+run | persisted variable |
| ARG | build only | `--build-arg` |
| EXPOSE | metadata | documents port, doesn't publish |
| USER | build+run | drop root |
| CMD | run | default command/args, overridable |
| ENTRYPOINT | run | fixed executable |

## Examples / Walkthrough

### 1. First Dockerfile: Python Flask app

```bash
mkdir -p ~/lab31/flask-app && cd ~/lab31/flask-app

cat > app.py <<'EOF'
import os
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return f"Hello from version {os.environ.get('APP_VERSION', 'unknown')}\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080)
EOF

echo "flask==3.0.3" > requirements.txt

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
ARG APP_VERSION=1.0.0
ENV APP_VERSION=$APP_VERSION
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python", "app.py"]
EOF

docker build -t flask-demo:1.0 .
docker run -d --name flask1 -p 8080:8080 flask-demo:1.0
curl localhost:8080                    # Hello from version 1.0.0
docker rm -f flask1
```

Explain: why `COPY requirements.txt` before `COPY . .` (cache), why `host="0.0.0.0"` (listen on all interfaces, else port mapping cannot reach app).

### 2. Observe the cache

```bash
docker build -t flask-demo:1.0 .             # all layers CACHED
echo "# change" >> app.py
docker build -t flask-demo:1.0 .             # only COPY . . and below re-run; pip install CACHED
echo "requests==2.32.3" >> requirements.txt
docker build -t flask-demo:1.0 .             # pip install re-runs
docker build --no-cache -t flask-demo:1.0 .  # everything re-runs
```

### 3. Add .dockerignore and measure context

```bash
dd if=/dev/zero of=junk.bin bs=1M count=200        # 200 MB junk
docker build --progress=plain -t flask-demo:1.0 . 2>&1 | grep -i "transferring context"
printf 'junk.bin\n.git\n__pycache__\n.env\n' > .dockerignore
docker build --progress=plain -t flask-demo:1.0 . 2>&1 | grep -i "transferring context"
```

Context drops from ~200 MB to a few KB.

### 4. ARG vs ENV

```bash
docker build --build-arg APP_VERSION=2.5.0 -t flask-demo:2.5 .
docker run --rm -e APP_VERSION=override flask-demo:2.5 printenv APP_VERSION   # override wins: override
docker run --rm flask-demo:2.5 printenv APP_VERSION                           # 2.5.0
docker history flask-demo:2.5 | grep APP_VERSION                              # build-arg visible in history!
```

### 5. CMD vs ENTRYPOINT live

```bash
cat > Dockerfile.ep <<'EOF'
FROM alpine:3.20
ENTRYPOINT ["echo", "Hello"]
CMD ["world"]
EOF
docker build -f Dockerfile.ep -t ep-demo .
docker run --rm ep-demo                 # Hello world
docker run --rm ep-demo DevOps          # Hello DevOps
docker run --rm --entrypoint date ep-demo   # prints date, entrypoint replaced
```

### 6. Shell form vs exec form — signal handling

```bash
cat > Dockerfile.sig <<'EOF'
FROM alpine:3.20
CMD sleep 300
EOF
docker build -f Dockerfile.sig -t sig-shell .
docker run -d --name s1 sig-shell
time docker stop s1                     # ~10 s: SIGTERM ignored by sh, then SIGKILL

sed -i.bak 's/^CMD.*/CMD ["sleep", "300"]/' Dockerfile.sig
docker build -f Dockerfile.sig -t sig-exec .
docker run -d --name s2 sig-exec
time docker stop s2                     # sleep is PID 1 and handles SIGTERM default → fast
docker rm s1 s2
```

(If sleep as PID 1 ignores SIGTERM on your system because PID 1 has no default signal handlers in the kernel, demo with a real app such as the Flask one: `docker stop` returns in <1 s with exec form when app handles SIGTERM.)

### 7. Non-root

```bash
docker run --rm flask-demo:1.0 id           # uid=0(root)
```

Add to Dockerfile before CMD:

```dockerfile
RUN useradd --create-home --uid 10001 app
USER app
```

```bash
docker build -t flask-demo:nonroot .
docker run --rm flask-demo:nonroot id       # uid=10001(app)
docker run --rm flask-demo:nonroot touch /etc/x   # Permission denied
```

### 8. Multi-stage: Go app, compare sizes

```bash
mkdir -p ~/lab31/go-app && cd ~/lab31/go-app
cat > main.go <<'EOF'
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello from Go")
	})
	http.ListenAndServe(":8080", nil)
}
EOF
cat > go.mod <<'EOF'
module example.com/hello

go 1.22
EOF

# Single stage
cat > Dockerfile.single <<'EOF'
FROM golang:1.22-alpine
WORKDIR /src
COPY . .
RUN go build -o /server .
CMD ["/server"]
EOF

# Multi-stage
cat > Dockerfile <<'EOF'
FROM golang:1.22-alpine AS builder
WORKDIR /src
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /server .

FROM scratch
COPY --from=builder /server /server
EXPOSE 8080
USER 10001
ENTRYPOINT ["/server"]
EOF

docker build -f Dockerfile.single -t go-single .
docker build -t go-multi .
docker images | grep go-          # go-single ~250MB vs go-multi ~5MB
docker run -d --rm -p 8081:8080 --name go1 go-multi
curl localhost:8081
docker stop go1
```

### 9. --target and docker history

```bash
docker build --target builder -t go-builder .     # stops after builder stage
docker run --rm go-builder ls -la /server          # inspect build output
docker history go-multi                            # only few tiny layers
docker history go-single                           # big toolchain layers
```

### 10. Debugging a failing build

```bash
docker build --progress=plain --no-cache -t myapp . 2>&1 | tail -40
# find failing step, then run the previous layer interactively:
docker run --rm -it python:3.12-slim sh            # replay the RUN by hand
```

## Common Pitfalls

- **`COPY . .` before dependency install** — every code change busts the cache and reinstalls all dependencies. Copy manifest first, install, then copy source.
- **Separate `RUN apt-get update` line** — cached stale package lists; later install fails with 404. Chain `update && install` in one RUN.
- **Deleting files in a later RUN to shrink image** — lower layers keep them; image does not shrink. Clean up in the same RUN that created the files, or use multi-stage.
- **Using `latest` tag for FROM** — build today differs from build next month. Pin version.
- **Shell-form CMD** (`CMD python app.py`) — app is not PID 1, SIGTERM not forwarded, 10 s kill delay on `docker stop`. Use exec form.
- **Single quotes in exec form** (`CMD ['python','app.py']`) — not valid JSON; Docker falls back to shell form silently. Use double quotes.
- **Expecting `$VAR` to expand in exec form** — no shell involved. Use `["sh","-c","..."]` or read env inside the app.
- **Secrets in ENV/ARG/COPY** — recoverable via `docker history` / `docker image inspect` / layer extraction, even if deleted later. Use runtime env/secret mounts, never bake in.
- **No `.dockerignore`** — `.git`, `node_modules`, `.env` copied into context and image; slow builds, leaked credentials.
- **Running as root** — default. Add `USER`. Remember `COPY --chown` and ports ≥ 1024.
- **Thinking EXPOSE publishes the port** — it does not. Use `docker run -p`.
- **`cd` in RUN expecting persistence** — each RUN is a new shell. Use WORKDIR.
- **Alpine + glibc binaries** — "not found" error on binaries compiled for glibc (the missing loader is the "not found"). Use slim, or compile statically/inside alpine.
- **Static Go binary on scratch built with CGO enabled** — `exec format error` / `no such file`. Set `CGO_ENABLED=0`.
- **`--from=` typo or stage name mismatch** — `COPY --from=build` when stage is `AS builder` fails: "stage not found".
- **ARG not redeclared after FROM** — ARG declared before FROM is empty inside stage. Re-declare `ARG NAME` after FROM.
- **Forgetting `--no-cache` doesn't refresh base image** — `--no-cache` ignores layer cache, but uses local base image; add `--pull` to fetch newer base.
- **Binding port 80 as non-root** — "permission denied". Listen on 8080+.

## FAQ

**Q: What is the difference between CMD and ENTRYPOINT?**
A: ENTRYPOINT is the fixed executable; `docker run` args are appended to it. CMD is the default command or default args and is replaced entirely by `docker run` args. Combine them: ENTRYPOINT = program, CMD = default arguments.

**Q: What is the difference between ADD and COPY?**
A: COPY only copies from context. ADD also auto-extracts local tar archives and can download URLs. Use COPY by default; ADD only for tar extraction.

**Q: What is the difference between ARG and ENV?**
A: ARG exists only during `docker build` (set with `--build-arg`). ENV persists in the image and is visible inside the running container. Neither is safe for secrets.

**Q: Does EXPOSE open the port to the outside?**
A: No. It is documentation. Use `-p host:container` on `docker run` to publish.

**Q: Why is my image so big?**
A: Likely a heavy base (full `python`/`node`), build tools left in, package caches not cleaned, or `COPY . .` including `node_modules`/`.git`. Check `docker history`, switch to `-slim`/alpine, add `.dockerignore`, use multi-stage.

**Q: Why does `docker build` reinstall dependencies every time?**
A: An earlier layer changed (usually `COPY . .` placed before install). Copy only dependency manifest first, install, then copy source.

**Q: What does `CACHED` mean in build output?**
A: Docker reused an existing layer because instruction and inputs are unchanged.

**Q: How do I force a rebuild without cache?**
A: `docker build --no-cache`. Add `--pull` to also refresh the base image.

**Q: Alpine or slim — which should I use?**
A: Slim (Debian) is the safer default: glibc, widest compatibility. Alpine is smaller but uses musl and may break native dependencies. Distroless/scratch are best for compiled static binaries in production.

**Q: What is distroless?**
A: Google-maintained minimal images with only the language runtime and CA certs: no shell, no package manager. Smaller attack surface; harder to debug (use `:debug` tags).

**Q: Can I `docker exec` into a scratch/distroless container?**
A: Not with a shell — none exists. Use the `:debug` variant of distroless, or build a debug target in multi-stage.

**Q: How do multi-stage builds reduce size?**
A: Only the final stage becomes the image. Compilers and dev dependencies stay in earlier stages that are discarded. You copy over just the artifact with `COPY --from`.

**Q: Can I copy from an image that is not a stage?**
A: Yes: `COPY --from=nginx:alpine /etc/nginx/nginx.conf /tmp/` pulls from that image.

**Q: What does `--target` do?**
A: Stops the build at the named stage. Useful for dev/test/prod targets from one Dockerfile and for debugging a builder stage.

**Q: What if my Dockerfile has another name?**
A: `docker build -f path/to/Dockerfile.prod -t app:prod .` (the last arg is still the context).

**Q: How do I pass a secret during build?**
A: Not with ARG/ENV. Use BuildKit: `RUN --mount=type=secret,id=npmrc cat /run/secrets/npmrc ...` and `docker build --secret id=npmrc,src=$HOME/.npmrc .`. Secret is not stored in layers.

**Q: Why does `docker stop` take 10 seconds?**
A: App ignores SIGTERM or is not PID 1 (shell-form CMD). Docker waits 10 s then sends SIGKILL. Use exec form, handle SIGTERM in the app.

**Q: Is `:latest` the newest version?**
A: No, it is only a default tag name, set by whoever pushed it. Pin explicit versions.

**Q: How do I see which command created each layer?**
A: `docker history <image>` (add `--no-trunc` for full commands).

**Q: Do I need `rm -rf /var/lib/apt/lists/*`?**
A: Yes, in the same RUN as `apt-get install`, to avoid ~30 MB of index data in the layer. (Official Debian/Ubuntu images are configured to auto-clean apt cache, but lists are still worth removing.)

**Q: Where do I learn about volumes, networks, resource limits, scanning for vulnerabilities?**
A: Lesson 32. Compose and pushing images to registries: lesson 33.

## Practice / Exercise

### Core

1. **Containerize an app.** Pick Node (Express), Python (Flask) or Go hello-world. Write a Dockerfile using FROM, WORKDIR, COPY, RUN, EXPOSE, CMD (exec form). Build with `docker build -t hello:1.0 .`, run with `-p`, test with `curl`.
2. **Cache optimization.** Start from a naive Dockerfile (`COPY . .` then install). Measure rebuild time after editing one source line. Reorder so manifest copy + install come first; re-measure. Record both times.
3. **.dockerignore.** Add a 100 MB dummy file and a `.env` with fake secret. Show context size before and after `.dockerignore`. Verify `.env` is absent: `docker run --rm hello:1.0 ls -la`.
4. **Non-root.** Add a user and `USER`. Verify with `docker run --rm hello:1.0 id` and `docker image inspect -f '{{.Config.User}}' hello:1.0`.
5. **ARG & ENV.** Add `ARG APP_VERSION` promoted to ENV; build with two different `--build-arg` values; show runtime override with `-e`.
6. **CMD vs ENTRYPOINT.** Create an image with `ENTRYPOINT ["ping","-c","3"]` and `CMD ["localhost"]` (alpine). Run it with and without args.
7. **Multi-stage.** Convert the Go (or Node/Python) single-stage Dockerfile into multi-stage. Compare `docker images` sizes. Record before/after MB.
8. **--target.** Build only the builder stage with `--target builder`; run `ls` in it to see artifacts.

### Stretch

1. Rewrite the Go multi-stage image to use `gcr.io/distroless/static:nonroot` instead of `scratch`; compare size and try `docker exec -it ... sh` in each — explain the result.
2. Build one Dockerfile with `dev`, `test`, `prod` targets (Node). Make the test target fail on purpose and observe the build fail.
3. Use `docker history --no-trunc` to find which layer contributes most to size; shrink it and show improvement.
4. Demonstrate secret leakage: bake `ARG TOKEN=abc123`, find it with `docker history`. Then redo with a BuildKit secret mount and prove it is absent.
5. Run `hadolint Dockerfile` (via `docker run --rm -i hadolint/hadolint < Dockerfile`) on your Dockerfile and fix every warning.
6. Demonstrate the `docker stop` delay with shell-form vs exec-form on a Flask app that handles SIGTERM (`signal.signal`), measure both.

## Further Reading

- Dockerfile reference: https://docs.docker.com/reference/dockerfile/
- Best practices for writing Dockerfiles: https://docs.docker.com/build/building/best-practices/
- Multi-stage builds: https://docs.docker.com/build/building/multi-stage/
- Build cache: https://docs.docker.com/build/cache/
- `docker build` CLI reference: https://docs.docker.com/reference/cli/docker/buildx/build/
- BuildKit secrets: https://docs.docker.com/build/building/secrets/
- Distroless images: https://github.com/GoogleContainerTools/distroless
- Hadolint (Dockerfile linter): https://github.com/hadolint/hadolint
