# Vault Production Configuration

A production Vault deployment starts with a configuration file written in HCL (HashiCorp Configuration Language) that tells Vault where to store data, how to accept connections, and what address to advertise to clients. This lesson covers the key configuration blocks, shows complete examples for both single-node and HA setups, and explains how to run Vault as a managed systemd service.

## HCL Format Basics

HCL uses a straightforward `key = value` syntax with block structures for grouping related settings. Strings are double-quoted; booleans and numbers are unquoted.

```hcl
# Comment
ui = true
log_level = "info"

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1
}
```

Blocks have a type (`listener`), an optional label (`"tcp"`), and a body enclosed in `{}`. Most Vault config files are composed of a handful of top-level keys and two to four blocks.

## The `listener` Block

The `listener` block configures how Vault accepts client connections. The only supported type is `tcp`.

```hcl
listener "tcp" {
  address       = "0.0.0.0:8200"
  tls_cert_file = "/etc/vault.d/tls/vault.crt"
  tls_key_file  = "/etc/vault.d/tls/vault.key"
}
```

| Parameter | Description |
|-----------|-------------|
| `address` | IP and port Vault listens on. Use `0.0.0.0:8200` to accept connections on all interfaces. |
| `tls_cert_file` | Absolute path to the PEM-encoded TLS certificate. |
| `tls_key_file` | Absolute path to the PEM-encoded private key. |
| `tls_disable` | Set to `1` to disable TLS entirely. Use only in lab environments — never in production. |

For lab and learning environments where you do not yet have a certificate:

```hcl
listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}
```

## The `storage` Block

The `storage` block tells Vault where to persist data. The choice of backend determines durability and high-availability characteristics.

### File Backend (Single Node)

The simplest option — Vault writes encrypted data to a local directory. No external dependencies, but no HA.

```hcl
storage "file" {
  path = "/opt/vault/data"
}
```

The directory must exist and be owned by the user running Vault:

```bash
sudo mkdir -p /opt/vault/data
sudo chown vault:vault /opt/vault/data
```

### Integrated Raft Backend (Built-in HA)

Raft is the recommended backend for new deployments. It is built into Vault — no external database required — and supports multi-node HA clusters.

```hcl
storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}
```

| Parameter | Description |
|-----------|-------------|
| `path` | Directory where Raft stores its WAL and snapshots. |
| `node_id` | A unique identifier for this node within the cluster. |

For a three-node Raft cluster each node also specifies `retry_join` blocks pointing to the other peers, but that is covered in the advanced module.

## `api_addr` and `cluster_addr`

These two top-level keys are critical in any deployment where Vault may redirect clients or communicate with peer nodes.

```hcl
api_addr     = "https://vault.example.com:8200"
cluster_addr = "https://vault-node-1.internal:8201"
```

| Key | Purpose |
|-----|---------|
| `api_addr` | The address Vault advertises to clients. Used in HTTP redirects (e.g., when a standby node redirects a write request to the active node). Should be a stable DNS name or load-balancer address. |
| `cluster_addr` | The address other Vault nodes use for server-to-server communication (Raft replication, forwarding). Must be reachable by all cluster peers. Port `8201` is the convention. |

## UI and Log Level

```hcl
ui        = true
log_level = "info"
```

`ui = true` enables the web interface at `/ui`. Valid `log_level` values in increasing verbosity are: `error`, `warn`, `info`, `debug`, `trace`. Use `info` for production; `debug` or `trace` only when diagnosing an issue.

## Full Minimal Lab Config

A complete, working configuration for a single-node lab that uses the file backend and disables TLS:

```hcl
# /etc/vault.d/vault.hcl

ui        = true
log_level = "info"

api_addr = "http://127.0.0.1:8200"

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = 1
}

storage "file" {
  path = "/opt/vault/data"
}
```

## Starting Vault with a Config File

```bash
vault server -config=/etc/vault.d/vault.hcl
```

Vault reads the file and starts in the foreground, streaming logs to stdout. In production you run it as a background service (see next section).

You can also point `-config` at a directory; Vault will merge all `.hcl` files in that directory. This is useful for splitting large configs into logical files:

```
/etc/vault.d/
├── vault.hcl          # main config: storage, api_addr, cluster_addr
├── listener.hcl       # listener block
└── telemetry.hcl      # telemetry block (optional)
```

## Running Vault as a systemd Service

Create the unit file at `/etc/systemd/system/vault.service`:

```ini
[Unit]
Description=HashiCorp Vault - A tool for managing secrets
Documentation=https://developer.hashicorp.com/vault/docs
Requires=network-online.target
After=network-online.target
ConditionFileNotEmpty=/etc/vault.d/vault.hcl

[Service]
Type=notify
User=vault
Group=vault
ProtectSystem=full
ProtectHome=read-only
PrivateTmp=yes
PrivateDevices=yes
SecureBits=keep-caps
AmbientCapabilities=CAP_IPC_LOCK
CapabilityBoundingSet=CAP_SYSLOG CAP_IPC_LOCK
NoNewPrivileges=yes
ExecStart=/usr/bin/vault server -config=/etc/vault.d/vault.hcl
ExecReload=/bin/kill --signal HUP $MAINPID
KillMode=process
KillSignal=SIGINT
Restart=on-failure
RestartSec=5
TimeoutStopSec=30
LimitNOFILE=65536
LimitMEMLOCK=infinity

[Install]
WantedBy=multi-user.target
```

Key points in the unit file:

- `User=vault` / `Group=vault` — Vault runs as a dedicated non-root user. Create it with `sudo useradd --system -d /etc/vault.d -s /bin/false vault`.
- `AmbientCapabilities=CAP_IPC_LOCK` — allows Vault to lock memory (mlock) so secrets are never swapped to disk.
- `LimitMEMLOCK=infinity` — removes the memory-lock limit for the service.
- `Restart=on-failure` — systemd automatically restarts Vault if it crashes.

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable vault
sudo systemctl start vault
sudo systemctl status vault
```

Check live logs:

```bash
sudo journalctl -u vault -f
```

## Common Pitfalls

- **`tls_disable = 1` in production** — disabling TLS exposes tokens and secrets in plaintext over the network. It is acceptable only in an isolated lab. Always use a signed certificate in any environment accessible beyond localhost.
- **Wrong ownership on the data directory** — if the `vault` user cannot write to the `storage.path` directory, Vault will fail to start. Run `chown -R vault:vault /opt/vault/data` after creating the directory.
- **Missing `api_addr`** — without `api_addr`, Vault cannot generate correct redirect responses. Clients get redirected to a wrong or unreachable address. Always set this to the stable address clients use to reach Vault.
- **Config file not found** — if the path passed to `-config` does not exist or is not readable by the `vault` user, the process exits immediately. Verify with `sudo -u vault cat /etc/vault.d/vault.hcl`.
- **Forgetting `daemon-reload` after editing the unit file** — systemd caches unit files. Run `sudo systemctl daemon-reload` any time you edit `/etc/systemd/system/vault.service`, or your changes will be silently ignored.

## Summary

- Vault configuration files use HCL: `key = value` pairs and named blocks like `listener` and `storage`.
- The `listener "tcp"` block sets the bind address and TLS parameters; use `tls_disable = 1` only in lab environments.
- Use the `file` backend for a simple single-node deployment; use `raft` for built-in HA without external dependencies.
- `api_addr` is the address clients are redirected to; `cluster_addr` is used for server-to-server (Raft/forwarding) communication.
- `ui = true` enables the web interface; `log_level` controls verbosity.
- Start Vault with `vault server -config=/etc/vault.d/vault.hcl` or run it as a systemd service with `systemctl enable vault && systemctl start vault`.
- The `vault` user needs ownership of the data directory and `CAP_IPC_LOCK` capability to lock memory.
