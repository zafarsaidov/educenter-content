# Lesson 13: nginx – Web Server, Reverse Proxy & Load Balancing

**Module:** Web Servers & Load Balancing
**Duration:** 120-150 min
**Prerequisites:** Linux Basics module (Lessons 1-12) — especially systemd, permissions, and networking/ports

## Learning Objectives

By end of lesson student can:
- Explain nginx's event-driven architecture and where its config files live
- Write and validate an nginx server block serving static content
- Configure nginx as a reverse proxy in front of a backend application
- Configure nginx as a load balancer across multiple backends, using different balancing methods
- Enable HTTPS with `certbot` and set up automatic certificate renewal
- Explain the `stream` module and when TCP/UDP proxying is needed instead of HTTP proxying

## Topics

- nginx overview: event-driven architecture, config structure (`/etc/nginx/nginx.conf`, `sites-available/enabled`), `nginx -t`, `systemctl reload`
- Server blocks: `listen`, `server_name`, `root`, `index`, `location` blocks, `try_files`, custom error pages
- Reverse proxy: `proxy_pass`, `proxy_set_header` (`Host`, `X-Real-IP`, `X-Forwarded-For`), `proxy_read_timeout`, `proxy_connect_timeout`
- Load balancing: `upstream` block with multiple backends; methods: round-robin (default), `ip_hash`, `least_conn`; `weight` parameter
- HTTPS with certbot: install certbot, `certbot --nginx`, certificate auto-renewal (systemd timer), HTTP→HTTPS redirect
- Stream module: TCP/UDP proxying (`stream` block, `upstream`, `server`); use cases: MySQL proxy, TCP load balancer

## Concepts

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

### Reverse proxy

A **reverse proxy** sits in front of one or more backend application servers and forwards client requests to them, then returns the backend's response to the client — the client only ever talks to nginx, never directly to the backend. `proxy_pass` inside a `location` block is what forwards the request.

Because the backend now sees the connection as coming from nginx (not the real client), a few headers matter to preserve the original request's identity:

| Header | Purpose |
|---|---|
| `Host` | Preserves the original requested hostname |
| `X-Real-IP` | The real client's IP address |
| `X-Forwarded-For` | Chain of proxy IPs the request passed through (useful when there's more than one hop) |

`proxy_connect_timeout` and `proxy_read_timeout` control how long nginx waits when establishing a connection to the backend, and while waiting for the backend to respond, respectively — tuning these avoids either failing too aggressively on a slow-but-working backend, or hanging too long on a genuinely dead one.

### Load balancing

An `upstream` block names a pool of backend servers that requests can be distributed across; `proxy_pass` then references the upstream's name instead of a single backend address.

| Method | Behavior |
|---|---|
| Round-robin (default) | Requests distributed evenly, in order, across all backends |
| `weight` | Modifies round-robin so some backends get proportionally more requests (e.g. a more powerful server) |
| `ip_hash` | Same client IP is always routed to the same backend — useful when a backend keeps session state in memory (sticky sessions) |
| `least_conn` | New requests go to whichever backend currently has the fewest active connections — useful when requests take varying amounts of time to process |

### HTTPS with certbot

`certbot` automates obtaining and installing free TLS certificates from Let's Encrypt. `certbot --nginx` specifically integrates with nginx: it detects your existing server blocks, requests a certificate for the matching domain, and edits the nginx config itself to add the `listen 443 ssl` directive, certificate paths, and (optionally) a redirect from HTTP to HTTPS.

Let's Encrypt certificates are short-lived (90 days), so certbot also installs a **systemd timer** (Lesson 5's territory) that periodically checks and renews certificates automatically before they expire — no manual renewal needed once set up.

### The `stream` module

Everything above operates in nginx's `http` context — it understands HTTP requests, headers, and paths. The `stream` module is a separate context that proxies raw **TCP/UDP** traffic without any HTTP-layer awareness — nginx just forwards bytes between client and backend. This is needed for protocols that aren't HTTP at all: proxying/load-balancing a MySQL database (port 3306), or generic TCP load balancing where you want nginx's connection-distribution logic but the traffic isn't a web request.

## Commands / Syntax Reference

| Command | Purpose | Example |
|---|---|---|
| `nginx -t` | Test config syntax | `sudo nginx -t` |
| `nginx -s reload` | Reload config (alt to systemctl) | `sudo nginx -s reload` |
| `systemctl reload nginx` | Graceful config reload | `sudo systemctl reload nginx` |
| `systemctl status nginx` | Check service status | `sudo systemctl status nginx` |
| `ln -s` | Enable a site (symlink) | `sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/` |
| `certbot --nginx` | Obtain + install a certificate | `sudo certbot --nginx -d example.com` |
| `certbot renew --dry-run` | Test renewal without actually renewing | `sudo certbot renew --dry-run` |

## Examples / Walkthrough

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
    server 10.0.0.11:3000;
    server 10.0.0.12:3000;
    server 10.0.0.13:3000 weight=2;        # gets ~2x the requests of the others

    # alternative balancing methods (pick one, comment out the others):
    # ip_hash;                             # sticky sessions by client IP
    # least_conn;                          # route to backend with fewest active connections
}

server {
    listen 80;
    server_name lb.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
# --- HTTPS with certbot ---
sudo apt update
sudo apt install certbot python3-certbot-nginx    # certbot + its nginx plugin

sudo certbot --nginx -d app.example.com
# certbot: detects the matching server block, requests a certificate, edits the config to add
# "listen 443 ssl", certificate paths, and offers to add an HTTP -> HTTPS redirect

sudo nginx -t                                      # confirm certbot's edits are valid
sudo systemctl reload nginx

# renewal is automated via a systemd timer installed by certbot; confirm it exists:
systemctl list-timers | grep certbot

# test the renewal process without actually renewing (safe, doesn't use up rate limits)
sudo certbot renew --dry-run
```

```nginx
# --- stream module: TCP load balancing, e.g. for MySQL (port 3306) ---
# this block goes in /etc/nginx/nginx.conf, at the top level (NOT inside http {})
stream {
    upstream mysql_backend {
        server 10.0.0.21:3306;
        server 10.0.0.22:3306;
    }

    server {
        listen 3306;
        proxy_pass mysql_backend;
    }
}
```

## Common Pitfalls

- **Reloading without testing first** — `sudo systemctl reload nginx` (or `nginx -s reload`) with a syntax error in the config can take the whole server down. Always run `sudo nginx -t` first; it catches syntax errors before they reach production traffic.
- **Editing `sites-available` but forgetting to symlink into `sites-enabled`** — the config file exists and looks correct, but nginx never loads it since only `sites-enabled/` is actually read. Equally common: forgetting to remove the `sites-enabled` symlink when disabling a site, leaving stale config active.
- **Forgetting `proxy_set_header Host`** — without it, the backend sees the internal proxy address as the `Host` header instead of the real requested domain, which breaks any backend logic (routing, absolute URLs, cookies) that depends on knowing the actual hostname.
- **Confusing `restart` and `reload`** — `systemctl restart nginx` drops all active connections; `systemctl reload nginx` applies config changes gracefully without interrupting in-flight requests. Prefer `reload` for routine config changes.
- **Running `certbot --nginx` before the domain's DNS actually points at the server** — Let's Encrypt validates domain ownership by making an HTTP request to the domain; if DNS isn't yet pointing at this nginx instance, certificate issuance fails. Confirm `dig <domain> A` (Lesson 10) resolves to the right IP first.
- **Putting a `stream` block inside `http {}`** — `stream` is a separate top-level context alongside `http` in `nginx.conf`, not nested inside it; nginx will fail to start if it's misplaced.

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

## Practice / Exercise

**Core:**
1. Install nginx on a lab VM, create a static site under `/var/www/`, write a server block for it, enable it via the `sites-enabled` symlink, and verify with `nginx -t` before reloading.
2. Run a simple backend app (or a basic HTTP server, e.g. `python3 -m http.server 3000`) and write a reverse-proxy server block pointing at it, including all three proxy headers.
3. Run 2-3 instances of a simple backend on different ports, set up an `upstream` block with default round-robin, and confirm requests distribute across them (e.g. by having each instance return its own port number in the response).
4. Change the `upstream` block to use `least_conn`, then `ip_hash`, and explain the difference you'd expect to observe in each mode.
5. If you have a real domain pointed at a lab server, run `certbot --nginx` and confirm HTTPS works and HTTP redirects to HTTPS.

**Stretch:**
1. Add a `weight` to one backend in your load-balanced upstream and send enough requests to confirm it receives proportionally more traffic.
2. Set up a `stream` block proxying a TCP service (any simple TCP listener works for practice) and confirm connectivity through nginx with a basic client tool.
3. Write a `location` block using `try_files` to fall back to a custom `404.html` page, and test it against both an existing and a non-existing file.

## Further Reading

- [nginx documentation](https://nginx.org/en/docs/)
- [nginx: Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Certbot documentation](https://certbot.eff.org/)
