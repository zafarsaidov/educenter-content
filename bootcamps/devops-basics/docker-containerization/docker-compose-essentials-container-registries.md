# Lesson 33: Docker Compose (Essentials) & Container Registries

**Module:** Docker – Containerization
**Duration:** 120-150 min
**Prerequisites:** Lesson 30 (Container Fundamentals & Docker Essentials), Lesson 31 (Dockerfile Best Practices & Multi-stage Builds), Lesson 32 (Docker Volumes, Networks & Security)

## Learning Objectives

By end of lesson student can:
- Explain why running multi-container apps with many `docker run` commands does not scale, and what Docker Compose replaces
- Read and write a `docker-compose.yml` with `services`, `image`, `build`, `ports`, `volumes`, `environment` and `depends_on`
- Define top-level named `volumes` and custom `networks` in a Compose file and attach services to them
- Operate a Compose project with `up -d`, `down`, `build`, `logs`, `ps`, `exec` and `restart`
- Explain how Compose names projects, containers, networks and volumes, and how service-name DNS works
- Pass configuration through `environment`, `env_file` and a `.env` file with `${VAR}` interpolation
- Tag an image with the `registry/user/image:tag` convention and apply semantic versioning to tags
- Log in to Docker Hub, push an image and pull it on another machine
- Run a private self-hosted registry (`registry:2`), push to it, and explain what Harbor adds on top
- Authenticate safely with `docker login` (token, `--password-stdin`, credentials store)
- Build and run a three-tier app (app + database + nginx reverse proxy) with one Compose file

## Topics

- Docker Compose: `docker-compose.yml` (services, image/build, ports, volumes, environment, depends_on)
- Compose commands: `up -d`, `down`, `build`, `logs`, `ps`, `exec`, `restart`; compose networks and volumes
- Docker Hub: login, push, pull; image tagging convention (user/image:tag, semantic versioning)
- Private registries: self-hosted registry, Harbor concept; `docker login` with credentials
- Practice: multi-container app with Compose (app + db + reverse proxy)

## Concepts

### The problem Compose solves

A real application is rarely one container. A typical web app is at least three: the application, a database, and a reverse proxy in front. Using only what you learned in Lessons 30-32, starting that stack looks like this:

```bash
docker network create appnet
docker volume create pgdata
docker run -d --name db --network appnet -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret -e POSTGRES_DB=app postgres:16-alpine
docker run -d --name app --network appnet -e DB_HOST=db -e DB_PASSWORD=secret myapp:1.0
docker run -d --name proxy --network appnet -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/conf.d/default.conf:ro nginx:1.27-alpine
```

Problems with this approach:
- Long commands are easy to mistype, hard to review, and impossible to diff in Git
- Startup order, restart, and cleanup are manual and error-prone (stop 3 containers, remove 3 containers, remove network, remove volume)
- Nobody else on the team can reproduce your exact setup without a README of shell commands

**Docker Compose** is a tool that lets you describe the whole stack declaratively in one YAML file and manage it as a single unit. You write *what* you want (services, networks, volumes), Compose creates and connects it. The file is committed to Git next to the code, so "how do I run this project locally?" becomes `docker compose up -d`.

### Compose v2: `docker compose` vs `docker-compose`

| | Compose v1 (legacy) | Compose v2 (current) |
|---|---|---|
| Command | `docker-compose` (hyphen, separate Python binary) | `docker compose` (space, CLI plugin of Docker) |
| Status | End of life, no updates | Maintained, ships with Docker Engine / Docker Desktop |
| File | `docker-compose.yml` | `compose.yaml` (preferred) or `docker-compose.yml` (still read) |

- This course uses `docker compose` (v2) and the file name `docker-compose.yml` because the plan and most tutorials use it.
- Compose v2 looks for these file names in the current directory, in order: `compose.yaml`, `compose.yml`, `docker-compose.yaml`, `docker-compose.yml`.
- The top-level `version:` key (`version: "3.8"`) is **obsolete**. Compose v2 ignores it and prints a warning. Do not write it.
- Check what you have: `docker compose version`. If missing on Linux: `sudo apt install docker-compose-plugin` (with Docker's official apt repo).

### Anatomy of a Compose file

A Compose file has top-level sections. Services is the only mandatory one:

```yaml
services:        # the containers (required)
  web: ...
  db: ...
volumes:         # named volumes shared/persisted across services
  pgdata:
networks:        # custom networks
  backend:
```

Each entry under `services:` becomes one container (or more, if scaled). The key (`web`, `db`) is the **service name**. It matters twice: it is used in commands (`docker compose logs web`) and it is the **DNS hostname** other containers use to reach that service.

### Service keys you must know

| Key | Meaning | Equivalent `docker run` flag |
|---|---|---|
| `image` | Image to run (`name:tag`); pulled if absent locally | positional image argument |
| `build` | Build the image from a Dockerfile instead of pulling | `docker build` |
| `container_name` | Fixed container name (optional, usually skip) | `--name` |
| `ports` | Publish ports `"HOST:CONTAINER"` | `-p` |
| `volumes` | Mounts: named volume, bind mount | `-v` |
| `environment` | Environment variables | `-e` |
| `env_file` | Load variables from file(s) | `--env-file` |
| `depends_on` | Start order between services | (none) |
| `networks` | Which networks the service joins | `--network` |
| `restart` | Restart policy (`no`, `always`, `on-failure`, `unless-stopped`) | `--restart` |
| `command` | Override image CMD | trailing command |
| `working_dir`, `user` | Working dir / user inside container | `-w`, `-u` |

#### `image` vs `build`

```yaml
services:
  db:
    image: postgres:16-alpine        # pull from registry
  app:
    build: ./app                     # build from ./app/Dockerfile
  app2:
    build:                           # long form
      context: ./app                 # directory sent to builder
      dockerfile: Dockerfile.prod    # Dockerfile name relative to context
      args:                          # --build-arg
        APP_VERSION: "1.2.0"
      target: runtime                # multi-stage --target
    image: myuser/app2:1.2.0         # name given to the built image
```

- If both `build` and `image` are set, Compose builds and tags the result with the `image` name. This is how you build locally and later push to a registry.
- `context` is the build context (same as the last argument of `docker build`). `.dockerignore` from Lesson 31 applies.
- `docker compose up` builds only if the image does not exist. After you change code run `docker compose up -d --build` or `docker compose build`.

#### `ports`

```yaml
ports:
  - "8080:80"            # host 8080 -> container 80, all interfaces
  - "127.0.0.1:5432:5432" # only reachable from the host itself
  - "9000"               # container 9000 on a random host port
```

- **Always quote** port mappings. YAML can parse `22:22` as a base-60 number in some parsers, producing nonsense.
- Publish only what must be reachable from outside. A database used only by the app **needs no `ports`** — containers on the same Compose network reach each other directly by service name and container port.
- `ports` vs `expose`: `expose` only documents a port between containers, it does not publish to the host. Rarely needed because containers on the same network can reach every port anyway.

#### `volumes`

```yaml
services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data     # named volume (declared below)
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro   # bind mount, read-only
  proxy:
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
volumes:
  pgdata:                                    # declaration required for named volumes
```

- A source starting with `./` or `/` is a **bind mount** (host path). A bare name (`pgdata`) is a **named volume** and *must* be declared in the top-level `volumes:` section.
- Relative paths are relative to the Compose file's directory, not your current shell directory.
- Append `:ro` for read-only mounts (good for config files).
- Named volumes survive `docker compose down`. They are deleted only by `docker compose down -v` or `docker volume rm`.

#### `environment` and `env_file`

```yaml
services:
  db:
    environment:
      POSTGRES_USER: appuser            # map form
      POSTGRES_PASSWORD: ${DB_PASSWORD} # taken from shell or .env file
      POSTGRES_DB: appdb
  app:
    environment:
      - DB_HOST=db                      # list form (KEY=value)
      - DB_PORT=5432
    env_file:
      - ./app.env                       # KEY=value lines
```

- Map form (`KEY: value`) and list form (`- KEY=value`) are equivalent. Do not mix them inside one service.
- If the same variable is in both `environment` and `env_file`, **`environment` wins**.
- Quote values that YAML could misread: `"yes"`, `"no"`, `"1.10"`, `"true"`, passwords containing `:` or `#`.

#### `depends_on`

```yaml
services:
  app:
    depends_on:
      - db
  proxy:
    depends_on:
      - app
```

- `depends_on` controls **start order** only: `db` is started before `app`; on `down`, stopped in reverse order.
- It does **not** wait for the database to be *ready* to accept connections. Postgres takes a few seconds to initialise; `app` can start, fail to connect, and crash.
- Real solutions: the application retries its connection (best), or a `healthcheck` on the database combined with a readiness condition (a brief mention only — covered with healthchecks in later orchestration lessons), or `restart: on-failure` as a crude fallback.

#### `restart`

| Value | Behaviour |
|---|---|
| `no` | Never restart (default) |
| `always` | Always restart, also after daemon/host reboot |
| `on-failure` | Restart only on non-zero exit code |
| `unless-stopped` | Like `always`, but not if you stopped it manually. Best default for servers |

### Compose project: names, networks and volumes

When you run `docker compose up`, everything created is grouped into a **project**.

- **Project name** defaults to the name of the directory containing the Compose file (lowercased). Override with `-p myproj` or `name:` at the top of the file.
- **Container names** follow `<project>-<service>-<index>`, e.g. `myapp-web-1`.
- **Default network**: Compose creates one network called `<project>_default` and attaches every service to it. No `networks:` section needed for simple stacks.
- **Named volumes** are prefixed with the project name: `pgdata` becomes `myapp_pgdata`. Check with `docker volume ls`.
- **Service-name DNS**: on a user-defined network Docker's embedded DNS resolves the service name `db` to the container's IP. So the app connects to host `db`, port `5432` — never `localhost` (inside the app container, `localhost` is the app container itself).

Run two copies of the same Compose file side by side by using different project names (`-p dev`, `-p test`), as long as published host ports do not collide.

### Compose networks (custom)

```yaml
services:
  proxy:
    networks: [frontend]
  app:
    networks: [frontend, backend]
  db:
    networks: [backend]
networks:
  frontend:
  backend:
```

- `proxy` can reach `app`, `app` can reach `db`, but `proxy` **cannot** reach `db` — they share no network. This is network segmentation: the database is invisible to the edge component.
- Networks are created with the project prefix (`myapp_frontend`).
- Reusing a network that already exists outside Compose: `networks: { shared: { external: true } }`.
- You learned `bridge`, `host`, and custom networks in Lesson 32. Compose just declares custom bridge networks for you.

### Variables: `${VAR}` interpolation and the `.env` file

Compose substitutes `${VAR}` in the Compose file from (in priority order): your shell environment, then the file named `.env` in the project directory.

```
# .env  (same directory as docker-compose.yml)
DB_PASSWORD=change-me
APP_TAG=1.0.0
HTTP_PORT=8080
```

```yaml
services:
  app:
    image: myuser/app:${APP_TAG}
  proxy:
    ports:
      - "${HTTP_PORT:-80}:80"        # default 80 if unset
  db:
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD must be set}   # fail fast if missing
```

| Syntax | Meaning |
|---|---|
| `${VAR}` | value of VAR, empty string (with a warning) if unset |
| `${VAR:-default}` | `default` if VAR is unset or empty |
| `${VAR:?message}` | abort with `message` if VAR is unset or empty |
| `$$` | literal `$` |

- `.env` is for **Compose-file substitution**. It is *not* automatically injected into containers — that is what `environment`/`env_file` do. (Classic confusion.)
- **Never commit `.env` with real secrets.** Add `.env` to `.gitignore`, commit a `.env.example` with fake values.
- Debug the final result with `docker compose config` — it prints the file with all variables substituted and validates syntax.

### Compose command lifecycle

All commands run in the directory with the Compose file (or use `-f path/to/file.yml`).

```
docker compose up -d ──► creates network/volumes, builds/pulls, creates+starts containers
docker compose ps    ──► status of the project's containers
docker compose logs  ──► output of containers
docker compose down  ──► stops + removes containers and networks (volumes kept)
```

`up` is idempotent: run it again after editing the file and Compose recreates only containers whose configuration changed.

### Container registries — why they exist

An image built on your laptop exists only on your laptop. A **registry** is a server that stores and distributes images. The Docker client pushes images to it and pulls images from it. It is the delivery path between "built" and "running on a server".

Vocabulary:
- **Registry**: the server (Docker Hub, GitHub Container Registry, Harbor, your own `registry:2`).
- **Repository**: a named collection of versions of one image inside a registry (`library/nginx`).
- **Tag**: a human-readable label on one version (`1.27`, `latest`).
- **Digest**: immutable content hash of an image, `sha256:ab12...`. A tag can be moved to point to different content; a digest cannot.

### Image naming convention

Full image reference:

```
[REGISTRY_HOST[:PORT]/][NAMESPACE/]REPOSITORY[:TAG|@DIGEST]
```

| Reference | Registry | Namespace | Repository | Tag |
|---|---|---|---|---|
| `nginx` | Docker Hub (default) | `library` (default) | `nginx` | `latest` (default) |
| `zafar/blog:1.2.0` | Docker Hub | `zafar` | `blog` | `1.2.0` |
| `ghcr.io/acme/api:2.0.1` | `ghcr.io` | `acme` | `api` | `2.0.1` |
| `localhost:5000/team/web:dev` | `localhost:5000` | `team` | `web` | `dev` |
| `registry.example.com/acme/api@sha256:ab12...` | custom | `acme` | `api` | pinned by digest |

- If no registry host is written, Docker assumes Docker Hub (`docker.io`).
- If no tag is written, Docker assumes `:latest`.
- The registry host is detected by the first path part containing a `.` or `:` or being `localhost`. This is why `localhost:5000/web` goes to a local registry but `zafar/web` goes to Docker Hub.
- For Docker Hub, the namespace **must be your Docker Hub username** (or organisation) to push.
- Allowed characters: repository names are lowercase letters, digits, `.`, `_`, `-`, `/`. Tags allow upper case but max 128 chars, no spaces.

### Tagging strategy and semantic versioning

**Semantic versioning (SemVer)**: `MAJOR.MINOR.PATCH`, e.g. `2.4.1`.
- MAJOR: breaking change
- MINOR: new backward-compatible feature
- PATCH: backward-compatible bug fix

Common practice is to push several tags that point to the **same image**:

```
zafar/blog:2.4.1   <- exact, immutable by convention
zafar/blog:2.4     <- latest patch of 2.4
zafar/blog:2       <- latest minor of 2
zafar/blog:latest  <- most recent release
```

Other tag styles seen in CI: git commit SHA (`blog:a1b2c3d`), branch (`blog:main`), build number, date (`blog:2026-10-17`). Use exact version or commit SHA for deployments.

**Why not `latest` in production:**
- `latest` is just a tag named "latest", not a magic "newest". It points to whatever the last push without a tag was.
- It is mutable: two servers pulling `latest` at different times can run different code.
- Rollback is impossible if you do not know the previous version. Always deploy an explicit tag.

### Docker Hub

Docker Hub (`hub.docker.com`) is the default public registry. Free tier: unlimited public repositories, 1 private repository, and pull-rate limits for anonymous and free users (limits change, check the current numbers in the docs).

- **Official images** (`nginx`, `postgres`, `python`) live under the implicit `library/` namespace.
- **Access tokens**: instead of your account password, create a Personal Access Token in Account Settings → Security. Tokens can be scoped (read-only, read/write) and revoked individually. Use a token for `docker login` on servers and in automation.
- Public repository: anyone can pull. Private repository: pull needs `docker login`.

### Self-hosted registry: `registry:2`

Docker publishes the **distribution registry** as an image named `registry` (tag `2`). It is a minimal, API-only registry (no web UI, no user management by default) that you can run anywhere in one command:

```bash
docker run -d --name registry -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  --restart unless-stopped registry:2
```

- Images are stored under `/var/lib/registry` — **mount a volume** or you lose all images when the container is removed.
- It speaks the Docker Registry HTTP API v2: `GET /v2/_catalog` lists repositories, `GET /v2/<repo>/tags/list` lists tags.
- **Docker requires TLS** for registries other than `localhost`. For `localhost:5000` plain HTTP is allowed; for `registry.internal:5000` you need a certificate (or list it under `insecure-registries` in `/etc/docker/daemon.json` — testing only).
- **Authentication** is optional and disabled by default: anyone who can reach the port can push and pull. Put basic auth (htpasswd) and TLS in front, or place nginx (Lesson 13) as TLS-terminating reverse proxy.

Basic auth with htpasswd:

```bash
mkdir -p auth
docker run --rm --entrypoint htpasswd httpd:2 -Bbn alice 'S3cretPass' > auth/htpasswd

docker run -d --name registry -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  -v $(pwd)/auth:/auth:ro \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

### Harbor (concept)

**Harbor** is an open-source, enterprise-grade registry (CNCF graduated project). Where `registry:2` is just storage plus an API, Harbor adds the features a company needs:

| Feature | `registry:2` | Harbor |
|---|---|---|
| Web UI | No | Yes |
| Users, groups, LDAP/OIDC login | No (htpasswd only) | Yes |
| Projects with role-based access | No | Yes (project admin, developer, guest) |
| Vulnerability scanning (Trivy built in) | No | Yes, scan on push |
| Image signing / policy ("block pull if critical CVE") | No | Yes |
| Replication between registries | No | Yes |
| Retention / garbage-collection policies | Manual | Scheduled |
| Robot accounts for CI | No | Yes |
| Proxy cache for Docker Hub (avoid rate limits) | Limited | Yes |

- Harbor itself runs as a multi-container Compose/Helm deployment (database, Redis, core, registry, portal).
- Other registries you will meet: GitHub Container Registry (`ghcr.io`), GitLab Container Registry (used in the CI/CD module), AWS ECR, Google Artifact Registry, Azure ACR. They all speak the same protocol, so `docker login/tag/push/pull` works identically.

### `docker login`: credentials and where they are stored

```bash
docker login                              # Docker Hub, interactive
docker login registry.example.com         # other registry
echo "$TOKEN" | docker login -u zafar --password-stdin    # non-interactive, safe
docker logout
```

- **`--password-stdin`** reads the secret from standard input so it does not appear in shell history or `ps` output. Never use `-p password` on the command line.
- Docker stores the result in `~/.docker/config.json`. By default (no credentials helper) the value in `auths` is only **base64 encoded** — trivially reversible, not encrypted:

```json
{ "auths": { "https://index.docker.io/v1/": { "auth": "emFmYXI6dG9rZW4=" } } }
```

- Safer: a **credentials store** (helper) that keeps secrets in the OS keychain. Configure `"credsStore"` in `config.json`: `osxkeychain` (macOS), `wincred` (Windows), `secretservice` or `pass` (Linux, via `docker-credential-helpers` package). Then `config.json` holds no secret.
- Log in as the *same user that runs docker*. `sudo docker login` writes `/root/.docker/config.json`, not yours.
- On servers use scoped tokens or robot accounts, not personal passwords.

## Commands / Syntax Reference

### Compose commands

| Command | Purpose | Example |
|---|---|---|
| `docker compose up` | Create and start everything (foreground) | `docker compose up` |
| `docker compose up -d` | Same, detached (background) | `docker compose up -d` |
| `docker compose up -d --build` | Rebuild images before starting | `docker compose up -d --build` |
| `docker compose up -d web` | Start one service (and its `depends_on`) | `docker compose up -d web` |
| `docker compose down` | Stop and remove containers + networks | `docker compose down` |
| `docker compose down -v` | Also remove named volumes (**deletes data**) | `docker compose down -v` |
| `docker compose build` | Build images for services with `build:` | `docker compose build app` |
| `docker compose build --no-cache` | Build ignoring cache | `docker compose build --no-cache app` |
| `docker compose pull` | Pull newest images for `image:` services | `docker compose pull` |
| `docker compose ps` | List project containers and status | `docker compose ps` |
| `docker compose ps -a` | Include stopped containers | `docker compose ps -a` |
| `docker compose logs` | Show logs of all services | `docker compose logs` |
| `docker compose logs -f --tail 50 app` | Follow last 50 lines of one service | `docker compose logs -f --tail 50 app` |
| `docker compose exec` | Run command in a running service container | `docker compose exec db psql -U appuser appdb` |
| `docker compose run --rm` | Start a new one-off container for a service | `docker compose run --rm app sh` |
| `docker compose restart` | Restart service(s) (no config reload) | `docker compose restart app` |
| `docker compose stop` / `start` | Stop / start without removing | `docker compose stop` |
| `docker compose config` | Validate and print the resolved file | `docker compose config` |
| `docker compose images` | Images used by the project | `docker compose images` |
| `docker compose top` | Processes inside each container | `docker compose top` |
| `docker compose -f other.yml up -d` | Use a specific Compose file | `docker compose -f prod.yml up -d` |
| `docker compose -p name up -d` | Set project name | `docker compose -p dev up -d` |
| `docker compose up -d --scale web=3` | Run 3 containers of a service | `docker compose up -d --scale web=3` |

Key differences:
- `down` removes containers; `stop` only stops them (they remain, `start` resumes).
- `restart` does **not** re-read the Compose file. After editing the file use `up -d` (it recreates what changed).
- `exec` needs a *running* container; `run` creates a fresh one (ignores published ports by default).
- `--scale` works only on services without a fixed `container_name` and without a fixed host port (several containers cannot bind the same host port).

### Registry commands

| Command | Purpose | Example |
|---|---|---|
| `docker login` | Authenticate to a registry | `docker login` |
| `docker login <host>` | Authenticate to another registry | `docker login localhost:5000` |
| `docker logout` | Remove stored credentials | `docker logout` |
| `docker tag SRC DST` | Add another name/tag to an existing image | `docker tag blog:dev zafar/blog:1.0.0` |
| `docker push` | Upload image to registry | `docker push zafar/blog:1.0.0` |
| `docker push --all-tags` | Push every local tag of the repository | `docker push --all-tags zafar/blog` |
| `docker pull` | Download image | `docker pull zafar/blog:1.0.0` |
| `docker search` | Search Docker Hub | `docker search --limit 5 nginx` |
| `docker image inspect --format` | Read digest/tags | `docker image inspect zafar/blog:1.0.0 --format '{{.RepoDigests}}'` |
| `docker images --digests` | List images with digests | `docker images --digests` |
| `docker build -t` | Build and tag | `docker build -t zafar/blog:1.0.0 .` |
| `docker rmi` | Remove local tag/image | `docker rmi zafar/blog:1.0.0` |

### Registry HTTP API (curl)

| Request | Purpose |
|---|---|
| `curl http://localhost:5000/v2/` | Health/version check (returns `{}`) |
| `curl http://localhost:5000/v2/_catalog` | List repositories |
| `curl http://localhost:5000/v2/team/web/tags/list` | List tags of a repository |
| `curl -u alice:pass http://localhost:5000/v2/_catalog` | Same with basic auth |

## Examples / Walkthrough

### 1. Minimal Compose file: nginx + one page

```bash
mkdir -p ~/compose-demo/html && cd ~/compose-demo
echo '<h1>Hello from Compose</h1>' > html/index.html
cat > docker-compose.yml <<'EOF'
services:
  web:
    image: nginx:1.27-alpine
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    restart: unless-stopped
EOF

docker compose config          # validate
docker compose up -d           # start
docker compose ps              # NAME  IMAGE  STATUS  PORTS
curl http://localhost:8080     # <h1>Hello from Compose</h1>
echo '<h1>Edited live</h1>' > html/index.html
curl http://localhost:8080     # bind mount -> change visible immediately
docker compose logs --tail 5 web
docker compose down            # removes container + default network
```

Show: `docker network ls` while it runs (`compose-demo_default`) and after `down` (gone).

### 2. Two services with DNS: wordpress-style app + db

```yaml
# docker-compose.yml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

  adminer:
    image: adminer:4
    ports:
      - "8081:8080"
    depends_on:
      - db

volumes:
  pgdata:
```

```bash
echo 'DB_PASSWORD=s3cret' > .env
echo '.env' >> .gitignore
docker compose up -d
docker compose ps
# browser: http://localhost:8081  -> System: PostgreSQL, Server: db (!), User: appuser
docker compose exec db psql -U appuser appdb -c '\l'
docker network ls | grep compose
docker volume ls | grep pgdata        # <project>_pgdata
docker compose down                    # volume stays
docker compose up -d                   # data still there
docker compose down -v                 # now data gone
```

Teaching points: the server name in Adminer is `db`, the service name. Data survives `down` but not `down -v`.

### 3. Build from a Dockerfile with Compose

```bash
mkdir -p ~/build-demo/app && cd ~/build-demo
cat > app/Dockerfile <<'EOF'
FROM python:3.12-alpine
WORKDIR /app
COPY server.py .
USER nobody
EXPOSE 8000
CMD ["python", "server.py"]
EOF
cat > app/server.py <<'EOF'
from http.server import BaseHTTPRequestHandler, HTTPServer
import os
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.end_headers()
        self.wfile.write(f"hello from {os.environ.get('APP_NAME','app')}\n".encode())
HTTPServer(("0.0.0.0", 8000), H).serve_forever()
EOF
cat > docker-compose.yml <<'EOF'
services:
  app:
    build: ./app
    image: localhost:5000/demo/app:0.1.0
    environment:
      APP_NAME: build-demo
    ports:
      - "8000:8000"
EOF
docker compose up -d --build
curl localhost:8000
docker images | grep demo/app          # tag came from image:
```

Change `server.py`, run `docker compose up -d --build` again — only the app container is recreated.

### 4. Practice app: app + db + nginx reverse proxy

Directory layout:

```
shop/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── app/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app.py
└── nginx/
    └── default.conf
```

`app/app.py` — counts visits in Postgres:

```python
import os, time
import psycopg2
from flask import Flask

app = Flask(__name__)

def connect():
    # retry: depends_on does not wait for Postgres to be ready
    for _ in range(30):
        try:
            return psycopg2.connect(
                host=os.environ["DB_HOST"], dbname=os.environ["DB_NAME"],
                user=os.environ["DB_USER"], password=os.environ["DB_PASSWORD"])
        except psycopg2.OperationalError:
            time.sleep(1)
    raise SystemExit("database not reachable")

conn = connect()
conn.autocommit = True
with conn.cursor() as cur:
    cur.execute("CREATE TABLE IF NOT EXISTS visits (id SERIAL PRIMARY KEY, at TIMESTAMP DEFAULT now())")

@app.route("/")
def index():
    with conn.cursor() as cur:
        cur.execute("INSERT INTO visits DEFAULT VALUES")
        cur.execute("SELECT count(*) FROM visits")
        return f"visits: {cur.fetchone()[0]}\n"
```

`app/requirements.txt`:

```
flask==3.0.3
psycopg2-binary==2.9.9
gunicorn==22.0.0
```

`app/Dockerfile`:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
USER nobody
EXPOSE 8000
CMD ["gunicorn", "-b", "0.0.0.0:8000", "app:app"]
```

`nginx/default.conf`:

```nginx
server {
    listen 80;
    location / {
        proxy_pass http://app:8000;     # "app" = service name (DNS)
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

`.env`:

```
DB_PASSWORD=change-me
HTTP_PORT=8080
```

`docker-compose.yml`:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "${HTTP_PORT:-80}:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend
    depends_on:
      - app
    restart: unless-stopped

  app:
    build: ./app
    image: shop/app:1.0.0
    environment:
      DB_HOST: db
      DB_NAME: shopdb
      DB_USER: shop
      DB_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD in .env}
    networks:
      - frontend
      - backend
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD in .env}
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks:
      - backend
    restart: unless-stopped

volumes:
  dbdata:

networks:
  frontend:
  backend:
```

Run and verify:

```bash
docker compose config                      # check substitution
docker compose up -d --build
docker compose ps
curl localhost:8080                        # visits: 1
curl localhost:8080                        # visits: 2
docker compose logs -f app                 # Ctrl+C to leave
docker compose exec db psql -U shop shopdb -c 'select count(*) from visits;'
docker compose exec proxy wget -qO- http://app:8000     # proxy -> app works
docker compose exec proxy sh -c 'nc -zv db 5432'        # fails/hangs: proxy not on backend
docker compose restart app
docker compose down                        # data stays in dbdata
docker compose up -d && curl localhost:8080             # counter continues
```

Segmentation proof: the proxy cannot reach `db` because they share no network.

### 5. Push to Docker Hub

```bash
docker login -u zafar                      # paste access token as password
docker build -t shop/app:1.0.0 ./app       # local name
docker tag shop/app:1.0.0 zafar/shop-app:1.0.0
docker tag shop/app:1.0.0 zafar/shop-app:1.0
docker tag shop/app:1.0.0 zafar/shop-app:1
docker tag shop/app:1.0.0 zafar/shop-app:latest
docker push --all-tags zafar/shop-app
docker images zafar/shop-app               # same IMAGE ID, four tags

# on another machine (or after docker rmi):
docker rmi zafar/shop-app:1.0.0
docker pull zafar/shop-app:1.0.0
docker run -d -p 8000:8000 zafar/shop-app:1.0.0
```

Observe: second push of identical layers prints `Layer already exists`. Only changed layers upload.

### 6. Private registry on localhost

```bash
docker run -d --name registry -p 5000:5000 -v registry-data:/var/lib/registry registry:2

docker pull alpine:3.20
docker tag alpine:3.20 localhost:5000/lab/alpine:3.20
docker push localhost:5000/lab/alpine:3.20

curl http://localhost:5000/v2/_catalog                  # {"repositories":["lab/alpine"]}
curl http://localhost:5000/v2/lab/alpine/tags/list      # {"name":"lab/alpine","tags":["3.20"]}

docker rmi localhost:5000/lab/alpine:3.20
docker pull localhost:5000/lab/alpine:3.20              # comes back from your registry
```

For a remote registry host (e.g. `registry.lab.local:5000`) without TLS (testing only):

```json
// /etc/docker/daemon.json
{ "insecure-registries": ["registry.lab.local:5000"] }
```

```bash
sudo systemctl restart docker
```

Push with authentication (after starting the htpasswd variant shown in Concepts):

```bash
docker login localhost:5000 -u alice
docker push localhost:5000/lab/alpine:3.20
curl -u alice:S3cretPass http://localhost:5000/v2/_catalog
```

### 7. Compose for the registry itself

```yaml
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry
    restart: unless-stopped
volumes:
  registry-data:
```

### 8. Pin by digest

```bash
docker pull nginx:1.27-alpine
docker image inspect nginx:1.27-alpine --format '{{index .RepoDigests 0}}'
# nginx@sha256:....
# use in Compose:  image: nginx@sha256:....   -> always the same bytes
```

## Common Pitfalls

- **`version: "3.8"` at the top of the file** — obsolete in Compose v2, prints a warning. Delete the line.
- **Using `localhost` as the DB host inside a container** — `localhost` is the container itself. Use the service name (`db`).
- **Assuming `depends_on` waits for readiness** — it only orders startup. The app crashes if the DB is still initialising. Add connection retry in the app or `restart: on-failure`.
- **Forgetting to declare a named volume** — `pgdata:/var/lib/...` without a top-level `volumes: pgdata:` gives `service refers to undefined volume`.
- **Using `docker compose down -v` casually** — `-v` deletes named volumes, i.e. your database. Plain `down` keeps data.
- **Editing the file and only running `restart`** — restart does not apply Compose changes. Run `docker compose up -d`.
- **Editing code but the container still runs old image** — `up -d` does not rebuild an existing image. Use `up -d --build` or `docker compose build`.
- **Unquoted ports (`- 80:80`) or special values (`yes`, `1.10`)** — YAML converts them to other types. Quote them.
- **Committing `.env` with passwords** — put it in `.gitignore`, publish `.env.example`.
- **Thinking `.env` populates container variables** — it only feeds `${VAR}` substitution in the Compose file. Use `environment` or `env_file` to pass into the container.
- **Publishing the database port (`ports: "5432:5432"`) to the whole world** — most stacks do not need it. If needed, bind to `127.0.0.1`.
- **Tabs in YAML** — YAML allows spaces only. Use 2-space indentation.
- **Pushing without retagging (`docker push blog:1.0` → `denied: requested access to the resource is denied`)** — the image name must include your Docker Hub namespace: `zafar/blog:1.0`.
- **Relying on `latest`** — mutable, hides which version runs. Deploy explicit SemVer or digest.
- **Not logged in / expired token** — `pull access denied` or `unauthorized: incorrect username or password`. Run `docker login` again.
- **Private registry without TLS** — `http: server gave HTTP response to HTTPS client`. Add TLS (right way) or `insecure-registries` (testing only), then restart Docker.
- **Registry container without a volume** — remove the container and every pushed image is gone.
- **`sudo docker login` then running docker as normal user** — credentials saved for root only.
- **Credentials in `config.json` believed to be encrypted** — they are only base64. Use a credentials store.
- **Name collisions between projects** — two directories both called `app` share the project name `app`, and `up` in one may recreate the other's containers. Use `-p` or `name:`.
- **Hard-coded `container_name`** — prevents `--scale` and collides across projects. Let Compose name containers.

## FAQ

**Q: What is the difference between `docker compose` and `docker-compose`?**
A: `docker-compose` (hyphen) is the old standalone v1 Python tool, end of life. `docker compose` (space) is v2, a plugin built into Docker. Same file format, v2 is the one to use.

**Q: Do I still need the `version:` line?**
A: No. It is obsolete and ignored by Compose v2. Remove it.

**Q: What is the difference between `docker-compose.yml` and `compose.yaml`?**
A: Both are read by `docker compose`. `compose.yaml` is the preferred modern name. If several exist, `compose.yaml` is picked first.

**Q: How does one container reach another in Compose?**
A: By service name. Compose puts all services on a shared network with built-in DNS, so `db` resolves to the database container. Use the container's internal port (`5432`), not the published host port.

**Q: Why can't my app connect to `localhost:5432` for the database?**
A: Inside the app container, `localhost` means the app container. The database is another container; use the hostname `db` (its service name).

**Q: Does `depends_on` make the app wait until the database is ready?**
A: No. Only start order. Make the app retry connecting, or use `restart: on-failure`. Healthcheck-based waiting exists and is mentioned later in orchestration lessons.

**Q: What is the difference between `up`, `start` and `restart`?**
A: `up` creates (if needed) and starts containers, applying config changes. `start` starts existing stopped containers. `restart` stops and starts running containers without re-reading the Compose file.

**Q: What is the difference between `stop` and `down`?**
A: `stop` halts containers but keeps them (and the network). `down` removes containers and networks. Volumes survive both unless you add `-v`.

**Q: Will `docker compose down` delete my database?**
A: Not if the data is on a named volume (or bind mount). Only `down -v` (or `docker volume rm`) deletes named volumes. Data written inside the container without a volume IS deleted with the container.

**Q: How do I see the logs of one service?**
A: `docker compose logs -f --tail 100 <service>`. Without a service name you get all, interleaved with colored prefixes.

**Q: How do I open a shell in a service?**
A: `docker compose exec <service> sh` (or `bash` if the image has it). For a service that is not running or crashes at start, `docker compose run --rm <service> sh`.

**Q: I changed the Compose file, how do I apply it?**
A: `docker compose up -d`. Compose compares the desired and current state and recreates only changed services.

**Q: I changed my app code, but nothing changed.**
A: Image was not rebuilt. Run `docker compose up -d --build`. If code is bind-mounted, no rebuild is needed.

**Q: Where do I put passwords?**
A: In a `.env` file (git-ignored) referenced as `${DB_PASSWORD}`, or an `env_file`. Never in committed YAML. Real secret stores come in later lessons.

**Q: What is the difference between `environment` and `env_file`?**
A: Same result, different source: `environment` is written inline in the Compose file, `env_file` loads `KEY=value` lines from a file. If both define the same key, `environment` wins.

**Q: What is the difference between `.env` and `env_file`?**
A: `.env` configures substitution of `${VAR}` inside the Compose file itself. `env_file` injects variables into the container.

**Q: Can I run multiple copies of the same stack?**
A: Yes, with different project names: `docker compose -p dev up -d` and `-p test up -d`. Avoid identical host ports.

**Q: How do I scale a service?**
A: `docker compose up -d --scale app=3`. The service must not publish a fixed host port or set `container_name`. Put a reverse proxy in front for load balancing.

**Q: What is a registry vs a repository?**
A: A registry is the server (Docker Hub). A repository is one image name with all its tags inside the registry (`zafar/blog`).

**Q: What does `docker push myimage` do wrong?**
A: Without a namespace the name points to Docker Hub's `library/`, which you cannot write. Retag as `youruser/myimage:tag`.

**Q: Does `docker tag` copy the image?**
A: No. It adds another name pointing to the same image ID. No disk space is used.

**Q: Is `latest` the newest version?**
A: No. It is only the default tag when none is given, and it moves only when somebody pushes it. Never trust it in production.

**Q: Should I use SemVer for every image?**
A: For releases, yes: `MAJOR.MINOR.PATCH`. For CI builds many teams use the Git commit SHA. For deployments use an exact, immutable reference.

**Q: What is a digest and when do I use it?**
A: A `sha256:` hash of the exact image content. Use `image@sha256:...` when you need a guarantee that the bytes never change (compliance, reproducibility).

**Q: Is it safe to put my Docker Hub password in `docker login -p`?**
A: No. It lands in shell history and process lists. Use a token with `--password-stdin`.

**Q: Where does Docker keep my login?**
A: `~/.docker/config.json`. Without a credentials helper it holds a base64 string (not encrypted). Use `credsStore` (`osxkeychain`, `wincred`, `secretservice`, `pass`).

**Q: Why am I hitting Docker Hub rate limits?**
A: Anonymous and free accounts have pull limits per time window. Log in, use a paid plan, or mirror images in your own registry or Harbor proxy cache.

**Q: Why does pushing to my own registry say "server gave HTTP response to HTTPS client"?**
A: Docker requires TLS except for `localhost`. Put TLS on the registry (or nginx in front), or for lab use add it to `insecure-registries` in `/etc/docker/daemon.json` and restart Docker.

**Q: How do I list what is in my self-hosted registry?**
A: `curl http://host:5000/v2/_catalog` and `curl http://host:5000/v2/<repo>/tags/list` (add `-u user:pass` with auth). `registry:2` has no UI; Harbor does.

**Q: Can I delete an image from `registry:2`?**
A: Deleting needs `REGISTRY_STORAGE_DELETE_ENABLED=true`, an API delete by digest, then `registry garbage-collect` to free disk. Awkward; one reason teams move to Harbor with retention policies.

**Q: Registry:2 or Harbor?**
A: `registry:2` for labs and tiny teams (simple, one container). Harbor when you need UI, users/roles, scanning, replication, and retention.

**Q: Do GitHub/GitLab registries work with the same commands?**
A: Yes. They implement the same API: `docker login ghcr.io`, tag with the host prefix, push, pull.

**Q: Compose file vs Kubernetes manifests?**
A: Compose targets a single host, great for local development and small deployments. Kubernetes (later module) orchestrates containers across many nodes. The concepts (services, volumes, networks, env) carry over.

## Practice / Exercise

### Core

1. **First Compose file.** Create a project with an nginx service on host port 8080 serving a bind-mounted `index.html` (read-only). Run `docker compose config`, `up -d`, `ps`, `curl`, change the HTML, `curl` again, then `down`. Check `docker network ls` before and after.
2. **Database with persistence.** Add Postgres with a named volume and credentials from `.env`. Create a table with `docker compose exec db psql`. Run `down`, `up -d`, confirm the table remains. Run `down -v`, `up -d`, confirm it is gone.
3. **Full stack.** Build the `shop` project from the walkthrough (proxy + app + db). Required: no `ports` on `app` or `db`; only the proxy publishes `${HTTP_PORT:-80}`; passwords only in `.env`; `.env` in `.gitignore`; `.env.example` committed.
4. **Prove DNS and segmentation.** From inside the proxy container reach `app` (works) and `db` (must fail). Explain why using the two custom networks.
5. **Break and fix `depends_on`.** Remove the retry loop from `app.py`, then `docker compose down && docker compose up -d`. Observe `docker compose logs app`. Restore the retry or add `restart: on-failure` and explain the difference between start order and readiness.
6. **Operate it.** Using only Compose commands: follow logs of `app`, restart `app`, open a shell in `proxy`, list images, scale nothing but explain why `app` could not be scaled with a fixed `container_name`.
7. **Tag and push.** Create a Docker Hub account and access token. Log in with `--password-stdin`. Tag the app as `1.0.0`, `1.0`, `1` and `latest`; push; verify four tags on the web UI; confirm all share one image ID locally.
8. **Pull elsewhere.** Remove the local image (`docker rmi`) and run the app from the pushed tag on another machine/VM, or at least after removal on the same machine.
9. **Private registry.** Run `registry:2` with a volume, push an image tagged `localhost:5000/lab/app:1.0.0`, list it with the catalog and tags API, remove the local copy and pull it back.

### Stretch

1. **Authenticated registry.** Add htpasswd basic auth to your registry. Show that `docker push` fails before `docker login localhost:5000` and works after. Look at `~/.docker/config.json` and decode the `auth` value with `base64 -d`. Configure a credentials store and compare.
2. **Registry via Compose.** Write one Compose file that runs `registry:2` with auth and a named volume. Put the `shop` app image `build:` + `image: localhost:5000/shop/app:1.0.0` in the other file and push it using only Compose (`docker compose build`, then `docker compose push`).
3. **Digest pinning.** Find the digest of `postgres:16-alpine` and change the Compose file to pin it by `@sha256:...`. Explain what you gain and what upgrade work you take on.
4. **Variable power.** Make the image tag come from `${APP_TAG}` with `:?` validation; run `APP_TAG=1.0.1 docker compose config` and compare to the `.env` value. Document which one wins.
5. **Two environments.** Run the same project twice with `-p dev` and `-p test` on different `HTTP_PORT` values. List containers, networks and volumes and explain the naming.
6. **Compose vs `docker run`.** Rewrite the `shop` stack as plain `docker network create` + `docker run` commands. Compare line count, ease of cleanup, and reviewability.
7. **Research.** Read the Harbor installation guide, list the containers it needs, and write five sentences on when you would pick Harbor over `registry:2` for a company.

## Further Reading

- Compose file reference: https://docs.docker.com/reference/compose-file/
- Compose CLI reference: https://docs.docker.com/reference/cli/docker/compose/
- Compose environment variables and interpolation: https://docs.docker.com/compose/how-tos/environment-variables/
- Networking in Compose: https://docs.docker.com/compose/how-tos/networking/
- Docker Hub quickstart and access tokens: https://docs.docker.com/docker-hub/
- `docker login` and credential stores: https://docs.docker.com/reference/cli/docker/login/
- Deploy a registry server: https://distribution.github.io/distribution/about/deploy/
- Harbor documentation: https://goharbor.io/docs/
- Semantic Versioning: https://semver.org/
