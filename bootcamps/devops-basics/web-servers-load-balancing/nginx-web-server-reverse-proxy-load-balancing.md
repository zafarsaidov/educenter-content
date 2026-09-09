# Lesson 13: nginx – Web Server, Reverse Proxy & Load Balancing

**Module:** Web Servers & Load Balancing
**Duration:** 120-150 min
**Prerequisites:** Linux Basics module (Lessons 1-12) — especially systemd, permissions, and networking/ports

## Learning Objectives

By end of lesson student can:
- Install nginx from the official package and from the official nginx repository, and explain why the latter is often preferred
- Explain nginx's event-driven architecture and where its config files live
- Write and validate an nginx server block serving static content, using the most common directives correctly
- Explain how nginx picks a `location` block when several could match the same URL
- Configure nginx as a reverse proxy in front of a backend application
- Configure nginx as a load balancer across multiple backends, using different balancing methods
- Install certbot, generate a Let's Encrypt certificate, and operate certbot's day-2 commands (renew, revoke, delete, certificates)
- Configure the `stream` module as a full Layer 4 (TCP/UDP) proxy and load balancer, independent of HTTP
- Explain how HAProxy differs from nginx as a load balancer, and configure a basic HAProxy frontend/backend with active health checks

## Topics

- Installing nginx: `apt` package vs official nginx repository, verifying the install, default paths
- nginx overview: event-driven architecture, config structure (`/etc/nginx/nginx.conf`, `sites-available/enabled`, `conf.d/`), `nginx -t`, `systemctl reload`
- Most used server-block directives: `listen`, `server_name`, `root`, `index`, `access_log`/`error_log`, `client_max_body_size`, `keepalive_timeout`, `gzip`
- `location` blocks in depth: prefix match, exact match (`=`), regex match (`~`, `~*`), matching priority order, `try_files`, custom error pages
- Reverse proxy: `proxy_pass`, `proxy_set_header` (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`), `proxy_read_timeout`, `proxy_connect_timeout`, `proxy_buffering`
- Load balancing: `upstream` block with multiple backends; methods: round-robin (default), `ip_hash`, `least_conn`; `weight` parameter; passive health checks (`max_fails`, `fail_timeout`)
- `stream` module as a full TCP/UDP-level proxy: `stream` context, `upstream`, `server`, `proxy_pass`, `listen ... udp`, use cases (MySQL proxy, generic TCP load balancer, non-HTTP protocols)
- certbot: installing certbot and its nginx plugin, generating a Let's Encrypt certificate (`certbot --nginx`, `certbot certonly`), HTTP→HTTPS redirect, certificate auto-renewal (systemd timer)
- certbot day-2 commands: `certificates`, `renew`, `renew --dry-run`, `revoke`, `delete`, `--expand`
- HAProxy: installing, config structure (`global`, `defaults`, `frontend`, `backend`), balancing algorithms, active health checks, comparison with nginx

## Concepts

### Installing nginx

Two common install paths on Ubuntu:

- **`apt install nginx`** — installs whatever version is in Ubuntu's default repositories. Simple, but that version can lag well behind upstream nginx releases, since Ubuntu prioritizes stability over freshness.
- **Official nginx repository** — adds nginx's own APT repository (with its own GPG signing key, same mechanism as the PPAs covered in Lesson 7) so `apt install nginx` pulls current upstream releases instead. Preferred when you need recent features, faster security patches, or a specific nginx version.

Either way, the package installs the same layout and registers nginx as a systemd service, so `systemctl start/enable/status nginx` (Lesson 5) works immediately after install. `nginx -v` prints the installed version; `nginx -V` (capital V) additionally prints the exact compile-time flags and modules built in — useful for confirming whether the `stream` module is available, since on some minimal builds it's compiled as a separate package.

### Why nginx, and its architecture

nginx is built around an **event-driven, asynchronous** architecture: a small number of worker processes each handle thousands of connections concurrently by reacting to I/O events, rather than spawning a new OS thread/process per connection (the older Apache "prefork" model). This makes nginx efficient under high concurrency with a small memory footprint, which is why it's a default choice for reverse proxying and load balancing, not just serving static files.

### Configuration structure

nginx's main configuration is `/etc/nginx/nginx.conf`, which sets global settings and `include`s per-site configuration. On Ubuntu, site configs follow a convention:

| Path | Purpose |
|---|---|
| `/etc/nginx/sites-available/` | Where you write each site's config file |
| `/etc/nginx/sites-enabled/` | Symlinks into `sites-available/`; only files linked here are actually active |

Enabling a site means creating a symlink: `ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/`. This split lets you keep a config written but disabled, without deleting it.

Two commands matter for every change:
- `nginx -t` — tests the configuration syntax without applying it; always run this before reloading, since a broken config would otherwise take nginx down.
- `systemctl reload nginx` — applies configuration changes gracefully (workers finish in-flight requests, new workers pick up the new config), unlike `restart`, which drops connections.

### Server blocks

A `server` block defines how nginx handles requests for one site/domain. Key directives:

| Directive | Purpose |
|---|---|
| `listen` | Which IP/port to listen on, e.g. `listen 80;` |
| `server_name` | Which `Host` header(s) this block responds to, e.g. `example.com` |
| `root` | Filesystem directory to serve static files from |
| `index` | Default file to serve for a directory request, e.g. `index.html` |
| `location` | A block matching a URL path pattern, with its own directives |
| `try_files` | Tries a list of paths in order, serving the first that exists — commonly used to fall back to `index.html` for single-page apps, or to a custom error page |

Multiple `server` blocks can share one nginx instance, each handling a different `server_name` (name-based virtual hosting) — nginx picks the matching block based on the request's `Host` header.

Beyond the basics above, a handful of directives show up in nearly every real config:

| Directive | Purpose |
|---|---|
| `access_log` | Path (or `off`) for the request log — default logs every request with timing/status |
| `error_log` | Path and minimum severity level for error logging |
| `client_max_body_size` | Largest request body nginx accepts (e.g. file uploads); default is only 1MB, a very common source of unexplained upload failures |
| `keepalive_timeout` | How long an idle client connection is kept open for reuse, reducing TCP handshake overhead on repeat requests |
| `gzip` / `gzip_types` | Enables response compression, and which MIME types to compress — cuts bandwidth for text-based responses (HTML/CSS/JS/JSON) |
| `return` | Short-circuit a response without proxying, e.g. `return 301 https://$host$request_uri;` for a redirect |

### `location` block matching in depth

A `server` block can contain many `location` blocks, each matching a different URL path pattern. When a request comes in, nginx doesn't just use the first match it finds — it follows a strict priority order:

1. **Exact match** — `location = /path` — matches only that literal path, nothing else. Checked first; if it matches, nginx stops immediately.
2. **Prefix match with `^~`** — `location ^~ /static/` — matches any path starting with `/static/`; if this wins, nginx skips regex checks entirely.
3. **Regex match** — `location ~ /pattern` (case-sensitive) or `location ~* /pattern` (case-insensitive) — checked in the order they appear in the config; the first matching regex wins.
4. **Plain prefix match** — `location /path` (no modifier) — the longest matching prefix wins if no regex or `^~` block matched.

```nginx
location = /healthz {          # 1. exact match, fastest, checked first
    return 200 "ok";
}

location ^~ /static/ {         # 2. prefix match, skips regex checks if matched
    root /var/www;
}

location ~* \.(jpg|png|gif)$ { # 3. case-insensitive regex, checked in file order
    expires 30d;
}

location / {                   # 4. plain prefix fallback, matches everything else
    proxy_pass http://app_backend;
}
```

Getting this order backwards (e.g. assuming top-to-bottom always wins) is one of the most common sources of "why is this location block being ignored" confusion.

### Reverse proxy

A **reverse proxy** sits in front of one or more backend application servers and forwards client requests to them, then returns the backend's response to the client — the client only ever talks to nginx, never directly to the backend. `proxy_pass` inside a `location` block is what forwards the request.

Because the backend now sees the connection as coming from nginx (not the real client), a few headers matter to preserve the original request's identity:

| Header | Purpose |
|---|---|
| `Host` | Preserves the original requested hostname |
| `X-Real-IP` | The real client's IP address |
| `X-Forwarded-For` | Chain of proxy IPs the request passed through (useful when there's more than one hop) |
| `X-Forwarded-Proto` | Whether the original client request was `http` or `https` — without it, a backend behind an HTTPS-terminating nginx would wrongly believe every request arrived as plain HTTP |

`proxy_connect_timeout` and `proxy_read_timeout` control how long nginx waits when establishing a connection to the backend, and while waiting for the backend to respond, respectively — tuning these avoids either failing too aggressively on a slow-but-working backend, or hanging too long on a genuinely dead one. `proxy_buffering` (on by default) controls whether nginx buffers the backend's response in memory/disk before sending it to the client — turning it `off` streams the response through immediately, which matters for things like long-lived streaming responses or Server-Sent Events, at the cost of holding the backend connection open longer.

### Load balancing and the `upstream` block

An `upstream` block names a pool of backend servers that requests can be distributed across; `proxy_pass` then references the upstream's name instead of a single backend address. It's a top-level block (a sibling of `server {}`, inside `http {}`), so one `upstream` can be referenced by multiple `server` blocks if needed.

Beyond a bare `server` line per backend, each `server` entry inside `upstream` accepts several useful parameters:

| Parameter | Effect |
|---|---|
| `weight=N` | Proportionally more requests to this backend (covered below) |
| `max_fails=N` | Consecutive failures before this backend is marked unavailable |
| `fail_timeout=T` | How long a failed backend stays marked unavailable, and the window failures are counted over |
| `backup` | Only receives traffic if all non-backup servers are unavailable — a standby |
| `down` | Marks a backend permanently out of rotation (e.g. during planned maintenance), without deleting the line |
| `max_conns=N` | Caps concurrent connections nginx will send to this one backend |

`upstream` also supports a `keepalive N;` directive, which keeps a pool of already-open connections to backends ready for reuse instead of opening a fresh TCP connection per request — meaningfully reduces latency and backend load under high traffic (requires `proxy_http_version 1.1;` and clearing the `Connection` header in the matching `location` block to take effect).

| Method | Behavior |
|---|---|
| Round-robin (default) | Requests distributed evenly, in order, across all backends |
| `weight` | Modifies round-robin so some backends get proportionally more requests (e.g. a more powerful server) |
| `ip_hash` | Same client IP is always routed to the same backend — useful when a backend keeps session state in memory (sticky sessions) |
| `least_conn` | New requests go to whichever backend currently has the fewest active connections — useful when requests take varying amounts of time to process |

nginx also does basic **passive health checking** on upstream backends out of the box: `max_fails` sets how many consecutive failed attempts mark a backend as unavailable, and `fail_timeout` sets both how long it stays marked unavailable and the window over which failures are counted. `server 10.0.0.11:3000 max_fails=3 fail_timeout=30s;` stops sending traffic to a backend after 3 failures within 30 seconds, then retries it after that window — no external health-check tooling required for this basic case.

### HTTPS with certbot: installing and generating certificates

`certbot` automates obtaining and installing free TLS certificates from Let's Encrypt, a certificate authority that issues domain-validated certificates at no cost. Installation on Ubuntu is via `apt` — either the `certbot` package plus the `python3-certbot-nginx` plugin (which lets certbot edit nginx config directly), from the standard repositories or, for a more current version, Certbot's own recommended `snap` install (Lesson 7's `snap` package manager).

Two main ways to generate a certificate:

- **`certbot --nginx -d example.com`** — the integrated flow: certbot reads your existing nginx server block for that domain, requests the certificate, then edits the nginx config itself to add `listen 443 ssl`, the certificate file paths, and (optionally, if you confirm) a redirect from HTTP to HTTPS. Best when nginx is already configured and running for that domain.
- **`certbot certonly --nginx -d example.com`** — obtains the certificate the same way, but does **not** edit the nginx config — it only places the certificate files on disk. Useful when you want to write the `listen 443 ssl` block yourself, or when a domain's config isn't ready for certbot to touch automatically.

Either way, Let's Encrypt validates domain ownership before issuing anything — it makes an HTTP request to the domain (or a DNS challenge, for advanced cases) to confirm you actually control it, which is why DNS must already point at the server before requesting a certificate. Certificates issued this way are short-lived (90 days), so certbot also installs a **systemd timer** (Lesson 5's territory) that runs the renewal check twice daily and renews any certificate nearing expiry — no manual renewal needed once set up.

### certbot day-2 commands

Beyond the initial issuance, certbot exposes commands for the certificate's whole lifecycle:

| Command | Purpose |
|---|---|
| `certbot certificates` | List every certificate certbot manages, with domains, paths, and expiry dates |
| `certbot renew` | Renew any certificate within 30 days of expiry (this is what the automated timer calls) |
| `certbot renew --dry-run` | Simulate the renewal process without actually requesting a new certificate — safe to run anytime, doesn't count against Let's Encrypt's rate limits |
| `certbot revoke --cert-path <path>` | Invalidate a certificate before its expiry (e.g. after a suspected key compromise) |
| `certbot delete --cert-name <domain>` | Remove a certificate and its renewal configuration entirely |
| `certbot --nginx -d example.com --expand` | Add additional domains/subdomains to an existing certificate |

### The `stream` module as a full Layer 4 proxy

Everything above (`server`, `location`, `proxy_pass` for HTTP) operates in nginx's `http` context — it parses HTTP requests, headers, and paths. The `stream` module is a completely separate top-level context that proxies raw **TCP and UDP** traffic at Layer 4, with zero awareness of what's inside the packets — nginx just forwards bytes between client and backend as fast as possible. This is what makes it a *general-purpose* TCP/UDP proxy and load balancer, not just an HTTP tool: it works for any protocol, not only ones nginx understands.

Common uses:

- Proxying/load-balancing a database (e.g. MySQL on port 3306, PostgreSQL on 5432) across replicas
- Generic TCP load balancing for a custom application protocol that isn't HTTP at all
- UDP proxying (`listen 3306 udp;`) for protocols like DNS or some game/streaming protocols

`stream` supports its own `upstream` blocks with the same load-balancing methods (round-robin, `least_conn`, and a stream-specific `hash` for consistent routing) — conceptually the same load-balancing model as `http`'s `upstream`, just operating below the HTTP layer.

### HAProxy: a dedicated load balancer

HAProxy (High Availability Proxy) is software built from the ground up specifically to do one job extremely well: load balancing and proxying, at both Layer 4 (TCP) and Layer 7 (HTTP). Where nginx is primarily a web server that also load-balances well, HAProxy is primarily a load balancer — it has no concept of `root`/`index`/serving static files at all. In practice, many production stacks use nginx to serve/proxy application traffic and HAProxy in front of it (or in front of database clusters) specifically for its more advanced traffic-distribution and health-checking features.

HAProxy's config file (`/etc/haproxy/haproxy.cfg`) is organized into distinct sections, each with a specific role:

| Section | Purpose |
|---|---|
| `global` | Process-wide settings: user/group to run as, max connections, logging target |
| `defaults` | Shared settings inherited by every `frontend`/`backend` below, unless overridden |
| `frontend` | Where HAProxy listens for incoming traffic — bind address/port, and rules for which `backend` to send traffic to |
| `backend` | A pool of servers HAProxy distributes traffic across, plus the balancing algorithm and health-check settings |
| `listen` | A shortcut combining `frontend` + `backend` into one section, for simple single-purpose proxies |

A `frontend` defines `mode http` (Layer 7 — HAProxy parses HTTP, can route on headers/paths, like nginx's `location`) or `mode tcp` (Layer 4 — raw byte forwarding, like nginx's `stream`). This single config format handling both modes, in one tool, is one of HAProxy's key differences from nginx, which splits this across the separate `http` and `stream` contexts.

Balancing algorithms are set with `balance` inside a `backend`:

| Algorithm | Behavior |
|---|---|
| `roundrobin` | Same idea as nginx's default — even rotation across servers |
| `leastconn` | Route to the server with fewest active connections (nginx's `least_conn`) |
| `source` | Same client IP always routed to the same server (nginx's `ip_hash` equivalent) |

Where HAProxy differs most noticeably from nginx's out-of-the-box behavior is **active health checking**: adding `check` to a `server` line tells HAProxy to proactively probe that backend on a regular interval (not just reactively count failures from real traffic, as nginx's `max_fails` does) — a backend can be detected and pulled out of rotation before it ever serves a single failed real request. HAProxy also ships a built-in **stats page** (enabled via a `listen stats` section) showing live per-backend health, connection counts, and traffic — useful for a quick visual view of load-balancer state without external tooling.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `nginx -v` / `-V` | Print version / version + compiled modules | `nginx -V` |
| `nginx -t` | Test config syntax | `sudo nginx -t` |
| `nginx -s reload` | Reload config (alt to systemctl) | `sudo nginx -s reload` |
| `systemctl reload nginx` | Graceful config reload | `sudo systemctl reload nginx` |
| `systemctl status nginx` | Check service status | `sudo systemctl status nginx` |
| `ln -s` | Enable a site (symlink) | `sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/` |
| `certbot --nginx` | Obtain + install + auto-edit nginx config | `sudo certbot --nginx -d example.com` |
| `certbot certonly --nginx` | Obtain certificate only, no config edits | `sudo certbot certonly --nginx -d example.com` |
| `certbot certificates` | List managed certificates + expiry | `sudo certbot certificates` |
| `certbot renew` | Renew certificates nearing expiry | `sudo certbot renew` |
| `certbot renew --dry-run` | Test renewal without actually renewing | `sudo certbot renew --dry-run` |
| `certbot revoke` | Invalidate a certificate | `sudo certbot revoke --cert-path /etc/letsencrypt/live/example.com/cert.pem` |
| `certbot delete` | Remove a managed certificate | `sudo certbot delete --cert-name example.com` |
| `haproxy -c -f` | Check HAProxy config syntax | `sudo haproxy -c -f /etc/haproxy/haproxy.cfg` |
| `systemctl reload haproxy` | Reload HAProxy config | `sudo systemctl reload haproxy` |
| `systemctl status haproxy` | Check HAProxy service status | `sudo systemctl status haproxy` |

## Examples / Walkthrough

```bash
# --- installing nginx ---

# option 1: default Ubuntu repository (simple, may lag behind upstream releases)
sudo apt update
sudo apt install nginx

# option 2: official nginx repository (current upstream releases)
sudo apt install curl gnupg2 ca-certificates lsb-release
curl -fsSL https://nginx.org/keys/nginx_signing.key | sudo gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list
sudo apt update
sudo apt install nginx

# verify install
nginx -v                                # version only
nginx -V                                # version + compile flags/modules (check for --with-stream)
systemctl status nginx                  # confirm the systemd service is running
sudo systemctl enable nginx             # start automatically on boot
```

```nginx
# --- location matching priority demo ---
server {
    listen 80;
    server_name demo.example.com;
    root /var/www/demo;

    location = /healthz {              # 1. exact match — checked first, fastest
        return 200 "ok\n";
        add_header Content-Type text/plain;
    }

    location ^~ /static/ {             # 2. prefix match with ^~, skips regex checks
        root /var/www;
        expires 7d;
    }

    location ~* \.(jpg|jpeg|png|gif|css|js)$ {   # 3. case-insensitive regex
        expires 30d;
        access_log off;                # don't bother logging static asset hits
    }

    location / {                       # 4. plain prefix fallback
        try_files $uri $uri/ =404;
    }
}
```

```nginx
# --- /etc/nginx/sites-available/static-site: serving static files ---
server {
    listen 80;
    server_name static.example.com;

    root /var/www/static-site;
    index index.html;

    location / {
        try_files $uri $uri/ =404;         # serve file, then directory, then 404
    }

    error_page 404 /404.html;
    location = /404.html {
        internal;
    }
}
```

```bash
# enable the site and validate before reloading
sudo ln -s /etc/nginx/sites-available/static-site /etc/nginx/sites-enabled/
sudo nginx -t                                  # ALWAYS test before reload
sudo systemctl reload nginx
```

```nginx
# --- /etc/nginx/sites-available/app-proxy: reverse proxy to a backend on port 3000 ---
server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

```nginx
# --- /etc/nginx/sites-available/app-lb: load balancing across 3 backends ---
upstream app_backend {
    # default: round-robin across all three
    server 10.0.0.11:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.12:3000 max_fails=3 fail_timeout=30s;
    server 10.0.0.13:3000 weight=2;        # gets ~2x the requests of the others
    server 10.0.0.14:3000 backup;          # only used if all above are unavailable

    # alternative balancing methods (pick one, comment out the others):
    # ip_hash;                             # sticky sessions by client IP
    # least_conn;                          # route to backend with fewest active connections

    keepalive 32;                          # keep up to 32 idle connections open per worker, for reuse
}

server {
    listen 80;
    server_name lb.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;            # required for keepalive to backends to work
        proxy_set_header Connection "";     # clear the default "close", allow connection reuse
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
# --- installing certbot ---
sudo apt update
sudo apt install certbot python3-certbot-nginx    # certbot + its nginx plugin

# confirm DNS already points at this server before requesting anything (Lesson 10's dig)
dig app.example.com A

# --- generating a Let's Encrypt certificate ---

# option 1: integrated flow — certbot edits the nginx config for you
sudo certbot --nginx -d app.example.com
# certbot: detects the matching server block, requests a certificate, edits the config to add
# "listen 443 ssl", certificate paths, and offers to add an HTTP -> HTTPS redirect (choose yes)

# option 2: certificate only, no config edits — write the ssl block yourself
sudo certbot certonly --nginx -d app.example.com

sudo nginx -t                                      # confirm certbot's edits are valid
sudo systemctl reload nginx

curl -I https://app.example.com                    # confirm HTTPS responds
curl -I http://app.example.com                      # confirm HTTP redirects (look for 301 Location: https://...)

# --- certbot day-2 commands ---
sudo certbot certificates                          # list every managed cert, domains, expiry dates

# renewal is automated via a systemd timer installed by certbot; confirm it exists:
systemctl list-timers | grep certbot

sudo certbot renew --dry-run                        # simulate renewal, safe, doesn't hit rate limits
sudo certbot renew                                   # actually renew anything within 30 days of expiry

# add another subdomain to an existing certificate
sudo certbot --nginx -d app.example.com -d www.app.example.com --expand

# revoke and remove a certificate no longer needed
sudo certbot revoke --cert-path /etc/letsencrypt/live/app.example.com/cert.pem
sudo certbot delete --cert-name app.example.com
```

```nginx
# --- stream module: full Layer 4 (TCP/UDP) proxy and load balancer ---
# this block goes directly in /etc/nginx/nginx.conf, at the top level (NOT inside http {})
stream {
    # TCP example: load-balance a MySQL replica pool
    upstream mysql_backend {
        least_conn;
        server 10.0.0.21:3306 max_fails=2 fail_timeout=15s;
        server 10.0.0.22:3306 max_fails=2 fail_timeout=15s;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
        proxy_connect_timeout 5s;
        proxy_timeout 300s;              # stream's equivalent of proxy_read_timeout
    }

    # UDP example: proxy a UDP-based service (e.g. a custom protocol on port 5000)
    upstream udp_backend {
        server 10.0.0.31:5000;
        server 10.0.0.32:5000;
    }

    server {
        listen 5000 udp;                 # "udp" keyword is what makes this Layer 4 UDP, not TCP
        proxy_pass udp_backend;
    }
}
```

```bash
# after adding/editing a stream block, always test before reloading — same discipline as http config
sudo nginx -t
sudo systemctl reload nginx
ss -tuln | grep -E ":3306|:5000"        # confirm nginx itself is now listening on both ports
```

```bash
# --- installing HAProxy ---
sudo apt update
sudo apt install haproxy
systemctl status haproxy
sudo systemctl enable haproxy
```

```
# --- /etc/haproxy/haproxy.cfg: HTTP load balancing with active health checks ---
global
    log /dev/log local0
    maxconn 2000
    user haproxy
    group haproxy

defaults
    mode http
    timeout connect 5s
    timeout client  30s
    timeout server  30s
    log global

frontend web_front
    bind *:80
    default_backend web_back

backend web_back
    balance roundrobin
    option httpchk GET /healthz          # active health check: HAProxy itself sends this request
    server app1 10.0.0.11:3000 check
    server app2 10.0.0.12:3000 check
    server app3 10.0.0.13:3000 check weight 2 backup

# built-in stats page — live view of backend health and traffic, protect with auth in production
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

```bash
# always check syntax before reloading — same discipline as nginx -t
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy

curl -I http://lb.example.com                # confirm traffic reaches a backend
# visit http://<server-ip>:8404/stats in a browser to see live backend status
```

```
# --- TCP mode: HAProxy in front of a database pool (Layer 4, like nginx's stream) ---
frontend mysql_front
    bind *:3306
    mode tcp
    default_backend mysql_back

backend mysql_back
    mode tcp
    balance leastconn
    option tcp-check                     # active TCP-level health check (just confirms the port accepts a connection)
    server db1 10.0.0.21:3306 check
    server db2 10.0.0.22:3306 check
```

## Common Pitfalls

- **Reloading without testing first** — `sudo systemctl reload nginx` (or `nginx -s reload`) with a syntax error in the config can take the whole server down. Always run `sudo nginx -t` first; it catches syntax errors before they reach production traffic.
- **Editing `sites-available` but forgetting to symlink into `sites-enabled`** — the config file exists and looks correct, but nginx never loads it since only `sites-enabled/` is actually read. Equally common: forgetting to remove the `sites-enabled` symlink when disabling a site, leaving stale config active.
- **Forgetting `proxy_set_header Host`** — without it, the backend sees the internal proxy address as the `Host` header instead of the real requested domain, which breaks any backend logic (routing, absolute URLs, cookies) that depends on knowing the actual hostname.
- **Confusing `restart` and `reload`** — `systemctl restart nginx` drops all active connections; `systemctl reload nginx` applies config changes gracefully without interrupting in-flight requests. Prefer `reload` for routine config changes.
- **Running `certbot --nginx` before the domain's DNS actually points at the server** — Let's Encrypt validates domain ownership by making an HTTP request to the domain; if DNS isn't yet pointing at this nginx instance, certificate issuance fails. Confirm `dig <domain> A` (Lesson 10) resolves to the right IP first.
- **Putting a `stream` block inside `http {}`** — `stream` is a separate top-level context alongside `http` in `nginx.conf`, not nested inside it; nginx will fail to start if it's misplaced.
- **Assuming `location` blocks match top-to-bottom** — nginx actually applies a fixed priority order (exact match, then `^~` prefix, then regex in file order, then longest plain prefix), not simple top-to-bottom. A `location /` at the top of the file does not "win" over a more specific block further down.
- **`client_max_body_size` defaulting to 1MB** — file uploads that "just fail" with no obvious backend error are very often nginx silently rejecting the request body before it even reaches the backend. Explicitly set `client_max_body_size` in any server block expecting uploads.
- **Forgetting `proxy_http_version 1.1` and clearing `Connection` when using `keepalive` in an `upstream`** — the `keepalive` directive alone does nothing without also setting `proxy_http_version 1.1;` and `proxy_set_header Connection "";` in the matching `location` block; without both, nginx still closes each backend connection after one request.
- **Requesting a certbot certificate before DNS propagates** — Let's Encrypt's HTTP validation makes a real request to the domain; if `dig <domain> A` doesn't yet return this server's IP, issuance fails, sometimes counting against Let's Encrypt's rate limits if retried too aggressively.
- **Forgetting `mode http`/`mode tcp` mismatches in HAProxy** — a `frontend` and its `default_backend` must agree on mode; mixing an `http` frontend with a `tcp` backend (or vice versa) fails config validation. Set `mode` explicitly in both, or rely on a shared `defaults mode` only when every section in the file genuinely uses the same mode.
- **Exposing the HAProxy stats page without authentication** — `listen stats` with no `stats auth` line is reachable by anyone who can reach that port, and reveals backend IPs, traffic patterns, and health status. Add `stats auth user:password` (or restrict access at the firewall level, Lesson 11) before running this in anything beyond a lab.
- **Assuming HAProxy's `check` happens automatically** — a `server` line without the `check` keyword is never actively health-checked at all; HAProxy just assumes it's always up. Forgetting `check` silently disables the exact feature (active health checking) that's the main reason to reach for HAProxy over nginx's passive `max_fails`.

## FAQ

**Q: What's the practical difference between a web server and a reverse proxy, if both use nginx?**
A: Same software, different role. As a plain web server, nginx serves files directly from `root`. As a reverse proxy, nginx forwards the request to a separate backend application (which could be written in any language/framework) and relays its response back — nginx itself isn't generating the content.

**Q: Why does `X-Forwarded-For` matter if there's only one proxy in front of the backend?**
A: With a single proxy hop it's mostly equivalent to `X-Real-IP`. It matters more when there are multiple proxies chained together (e.g. a CDN, then nginx, then the app) — `X-Forwarded-For` accumulates each hop's IP as a comma-separated list, preserving the full path, while `X-Real-IP` typically only shows the immediately preceding hop.

**Q: When should I use `ip_hash` instead of round-robin?**
A: When the backend application stores session state in its own memory rather than in a shared store — if a user's requests bounce between different backend instances mid-session, they'd appear logged out or lose in-progress state. `ip_hash` keeps a given client pinned to the same backend, avoiding that. The better long-term fix is often a shared session store, but `ip_hash` is a quick, config-only mitigation.

**Q: Does certbot automatically renew certificates forever, with no maintenance?**
A: Practically yes — certbot installs a systemd timer that checks twice daily and renews any certificate within 30 days of expiry. Worth periodically confirming the timer is still active (`systemctl list-timers`) and that DNS/nginx config haven't changed in a way that would break the HTTP validation certbot relies on.

**Q: Why would I need the `stream` module instead of just proxying HTTP normally?**
A: `proxy_pass` in the `http` context understands and manipulates HTTP requests — but many protocols nginx might need to load-balance (raw database connections, custom TCP protocols) aren't HTTP at all. `stream` operates purely at the TCP/UDP level, just forwarding bytes, which works for any protocol regardless of what's inside the packets.

**Q: Should I install nginx via `apt` or the official nginx repository?**
A: Plain `apt install nginx` is simplest and fine for learning/most use cases. Add the official nginx repository when you need a more current version than Ubuntu ships, faster access to security patches, or specific modules — the install/config workflow is identical either way, only the package source differs.

**Q: What's the actual difference between `certbot --nginx` and `certbot certonly --nginx`?**
A: Both request and obtain the certificate from Let's Encrypt the same way. `--nginx` (without `certonly`) additionally edits your nginx config automatically to serve HTTPS with it. `certonly` only places the certificate files on disk under `/etc/letsencrypt/live/` and leaves your nginx config untouched — use it when you'd rather write the `ssl` directives yourself.

**Q: Do I need `max_fails`/`fail_timeout` if my backends are already healthy?**
A: Not strictly — nginx will still load-balance fine without them. They matter once a backend actually fails: without them, nginx keeps sending some requests to a dead backend indefinitely, causing errors for whichever clients get routed there, instead of temporarily routing around it.

**Q: Why would I add HAProxy in front of nginx instead of just using nginx's own load balancing?**
A: nginx's passive health checking (`max_fails`) only reacts after real traffic already failed against a bad backend. HAProxy's active `check` proactively probes backends on its own schedule, so it can pull a failing backend out of rotation before a real user request ever hits it. HAProxy's config also unifies HTTP and TCP load balancing in one file/tool, and its stats page gives an at-a-glance operational view nginx doesn't provide out of the box. For many stacks, nginx alone is plenty; HAProxy earns its place when health-check precision or a dedicated stats view matters.

**Q: Is HAProxy a replacement for nginx?**
A: Not typically — they're often complementary rather than competing. nginx serves static files and reverse-proxies application traffic; HAProxy specializes in distributing that traffic across many backend instances with fine-grained health checking. A common pattern is HAProxy in front of a pool of nginx instances, or HAProxy in front of a database cluster where nginx's `stream` module would otherwise be used.

## Practice / Exercise

**Core:**
1. Install nginx on a lab VM (try both the `apt` package and, separately if time allows, the official nginx repository), confirm the version and compiled modules with `nginx -v`/`nginx -V`.
2. Create a static site under `/var/www/`, write a server block for it, enable it via the `sites-enabled` symlink, and verify with `nginx -t` before reloading.
3. Build the 4-block `location` priority demo (exact `/healthz`, `^~ /static/`, regex for image extensions, plain `/` fallback) and confirm each one matches the request you expect it to, in the right priority order.
4. Run a simple backend app (or a basic HTTP server, e.g. `python3 -m http.server 3000`) and write a reverse-proxy server block pointing at it, including all four proxy headers (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`).
5. Run 2-3 instances of a simple backend on different ports, set up an `upstream` block with default round-robin, and confirm requests distribute across them (e.g. by having each instance return its own port number in the response).
6. Change the `upstream` block to use `least_conn`, then `ip_hash`, and explain the difference you'd expect to observe in each mode.
7. If you have a real domain pointed at a lab server, install certbot, generate a certificate with `certbot --nginx`, and confirm HTTPS works and HTTP redirects to HTTPS. Then run `certbot certificates` and `certbot renew --dry-run`.

**Stretch:**
1. Add a `weight` to one backend in your load-balanced upstream and send enough requests to confirm it receives proportionally more traffic.
2. Add `max_fails`/`fail_timeout` to your upstream, stop one backend, and confirm nginx stops routing to it and later retries it.
3. Set up a `stream` block proxying a TCP service (any simple TCP listener works for practice), plus a second `stream` server block proxying a UDP service, and confirm connectivity through nginx with a basic client tool for each.
4. Enable `keepalive` on an `upstream` (with the required `proxy_http_version 1.1` and cleared `Connection` header) and explain, in your own words, what problem it solves.
5. Write a `location` block using `try_files` to fall back to a custom `404.html` page, and test it against both an existing and a non-existing file.
6. Install HAProxy on a lab VM, configure a `frontend`/`backend` pair load-balancing across your 2-3 practice backends from exercise 5 above, with `option httpchk` and `check` on each server. Validate with `haproxy -c -f` before reloading.
7. Enable the HAProxy `listen stats` section, protect it with `stats auth`, and view live backend health/traffic from the stats page in a browser.

**Stretch (continued):**
6. Stop one backend HAProxy is checking and observe, via the stats page, how quickly HAProxy detects the failure and stops routing to it — compare this against how nginx's `max_fails` would have behaved for the same failure.
7. Configure an HAProxy `backend` in `mode tcp` proxying a raw TCP service, and compare its config shape against the equivalent nginx `stream` block from exercise 3.

## Further Reading

- [nginx documentation](https://nginx.org/en/docs/)
- [nginx: Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Certbot documentation](https://certbot.eff.org/)
- [HAProxy documentation](https://www.haproxy.org/#docs)
