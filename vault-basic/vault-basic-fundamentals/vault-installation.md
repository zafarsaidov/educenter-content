# Installing and Running Vault

Vault is distributed as a single statically-linked binary with no runtime dependencies — the same binary is the server and the client. This lesson covers every supported installation method, explains the `IPC_LOCK` capability that protects secrets in memory, and walks through what dev mode and production mode look like in practice.

## Installing on Ubuntu and Debian

HashiCorp maintains an official apt repository. This is the recommended installation method on Debian-based systems because it handles GPG key verification, provides clean upgrades via `apt upgrade`, and pins you to a specific major version if you choose.

```bash
# Install prerequisites for adding a signed apt repository
apt update
apt install -y gpg wget

# Add the HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# Verify the fingerprint (optional but recommended)
gpg --no-default-keyring \
    --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
    --fingerprint

# Add the HashiCorp repository for your Ubuntu/Debian release
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  tee /etc/apt/sources.list.d/hashicorp.list

# Install Vault
apt update && apt install -y vault

# Verify the installation
vault version
# Output: Vault v1.17.0 (...)
```

After installation the `vault` binary is at `/usr/bin/vault`. The package also installs a systemd unit file at `/usr/lib/systemd/system/vault.service` and a default configuration skeleton at `/etc/vault.d/vault.hcl`.

## Installing on RHEL and CentOS

HashiCorp provides an rpm repository for RHEL, Rocky Linux, AlmaLinux, CentOS, and Fedora.

```bash
# Install the HashiCorp repository configuration
dnf install -y dnf-plugins-core
dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Vault
dnf install -y vault

# Verify
vault version
```

On older systems that use `yum` instead of `dnf`:

```bash
yum install -y yum-utils
yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
yum install -y vault
```

## Installing the Binary Directly

The binary installation method works on any Linux distribution and on macOS. It does not require root access and does not interact with your package manager.

```bash
# Set the version you want (check https://releases.hashicorp.com/vault/ for latest)
VAULT_VERSION="1.17.0"
ARCH="amd64"   # use arm64 for Apple Silicon or 64-bit ARM servers

# Download the zip archive
wget "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_${ARCH}.zip"

# Download and verify the checksum file
wget "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_SHA256SUMS"
sha256sum --check --ignore-missing "vault_${VAULT_VERSION}_SHA256SUMS"
# Expected: vault_1.17.0_linux_amd64.zip: OK

# Unzip and install
unzip "vault_${VAULT_VERSION}_linux_${ARCH}.zip"
chmod +x vault
mv vault /usr/local/bin/

# Verify
vault version
```

Always verify the SHA-256 checksum. HashiCorp also provides a GPG-signed checksum file (`vault_${VAULT_VERSION}_SHA256SUMS.sig`) if you want to verify the signature against HashiCorp's public key.

## Running Vault with Docker

The official HashiCorp image is `hashicorp/vault` on Docker Hub.

```bash
# Run Vault in dev mode — useful for local experimentation
docker run \
  --rm \
  --name vault-dev \
  --cap-add=IPC_LOCK \
  -e VAULT_DEV_ROOT_TOKEN_ID=root \
  -p 8200:8200 \
  hashicorp/vault
```

For a production-style container with a config file mounted from the host:

```bash
docker run \
  --name vault \
  --cap-add=IPC_LOCK \
  -v /path/to/vault/config:/vault/config \
  -v /path/to/vault/data:/vault/data \
  -p 8200:8200 \
  hashicorp/vault server -config=/vault/config/vault.hcl
```

## What IPC_LOCK Is and Why It Matters

`IPC_LOCK` is a Linux capability that allows a process to lock memory pages, preventing the OS from swapping them to disk.

Vault stores the unseal key (the reconstructed master key) and decrypted secrets in memory after unseal. Without `IPC_LOCK`, the Linux kernel can write these memory pages to the swap partition when memory is under pressure. If an attacker gets access to the swap device — a disk image, a cloud snapshot, a memory dump — they could extract the plaintext secrets and unseal keys directly from swap.

With `IPC_LOCK` granted, Vault calls `mlock(2)` to pin its memory pages in RAM. The kernel will never write those pages to disk.

```bash
# Vault warns you if it cannot lock memory
vault server -dev
# WARNING! Cannot lock memory, unable to use mlock syscall.
# This may leave sensitive information in swap.
# Install libcap2-bin and use setcap to give vault CAP_IPC_LOCK without running as root.

# Grant the capability to the binary without running as root
setcap cap_ipc_lock=+ep /usr/bin/vault
```

In production, always ensure either:
- The Vault process runs with `CAP_IPC_LOCK` (via `setcap` or `--cap-add=IPC_LOCK` in Docker), or
- Swap is disabled entirely on the host (`swapoff -a`)

## Running Vault in Dev Mode

Dev mode is the fastest way to get a working Vault instance locally. One command, no configuration file needed.

```bash
vault server -dev
```

On startup, Vault prints everything you need to connect:

```
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201",
                          max_request_duration: "1m30s", tls: "disabled")
                 Storage: inmem
                 Version: Vault v1.17.0

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is:

hvs.root

Development Mode WARNING: Do NOT run dev mode in production.

Unseal Key: SrIZz8AJnGgxDLjT0Nv39mLqjYLWZBEEy5N3V7fP/8k=
Root Token: hvs.root
```

To connect to it from another terminal:

```bash
# Tell the CLI where Vault is listening
export VAULT_ADDR="http://127.0.0.1:8200"

# Authenticate with the root token that was printed on startup
export VAULT_TOKEN="hvs.root"

# Confirm connectivity
vault status
# Key             Value
# ---             -----
# Seal Type       shamir
# Initialized     true
# Sealed          false
# ...
```

You can customise the root token to make scripts more readable:

```bash
vault server -dev -dev-root-token-id="my-dev-token"
```

The KV v2 secrets engine is automatically enabled and mounted at `secret/` in dev mode. You can start writing secrets immediately:

```bash
vault kv put secret/myapp db_password="devpassword"
vault kv get secret/myapp
```

## Dev Mode vs Production Mode

| Property | Dev mode | Production mode |
|---|---|---|
| Command | `vault server -dev` | `vault server -config=/etc/vault.d/vault.hcl` |
| Storage | In-memory (all data lost on exit) | Persistent (Raft, Consul, S3, etc.) |
| TLS | Disabled (HTTP) | Required (HTTPS) |
| Seal state on start | Already unsealed | Sealed — must run `vault operator unseal` |
| Init process | Skipped | `vault operator init` required once |
| Root token | Printed to stdout, fixed at startup | Generated once during init, never shown again |
| KV engine | Auto-mounted at `secret/` | Must be enabled manually |
| Bind address | `127.0.0.1:8200` only | Configurable (typically `0.0.0.0:8200`) |
| Use case | Local development, CI tests, demos | Any real environment |

A minimal production configuration file (`vault.hcl`) looks like this:

```hcl
# /etc/vault.d/vault.hcl

storage "raft" {
  path    = "/opt/vault/data"
  node_id = "vault-node-1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_cert_file = "/opt/vault/tls/vault.crt"
  tls_key_file  = "/opt/vault/tls/vault.key"
}

api_addr     = "https://vault.example.com:8200"
cluster_addr = "https://vault.example.com:8201"
ui           = true
```

Starting a production server and running the bootstrap sequence:

```bash
# Start the server (daemonised via systemd in practice)
vault server -config=/etc/vault.d/vault.hcl

# In a second terminal — initialise (one-time, first-ever startup only)
export VAULT_ADDR="https://vault.example.com:8200"
vault operator init -key-shares=5 -key-threshold=3
# Prints 5 Unseal Keys and 1 Initial Root Token — SAVE THESE SECURELY

# Unseal by providing 3 of the 5 key shares (run 3 times with different keys)
vault operator unseal   # enter key share 1
vault operator unseal   # enter key share 2
vault operator unseal   # enter key share 3

# Confirm Vault is unsealed
vault status
# Sealed: false

# Log in with the initial root token
vault login
# Token (will be hidden):
```

After this bootstrap sequence, you never see the initial root token again. Store it in a secure offline location and use it only for initial setup tasks before creating more narrowly-scoped tokens.

## Common Pitfalls

- **Running dev mode in any shared environment.** Dev mode uses HTTP, has no real auth model, and its root token is visible in the process list (`ps aux | grep vault`). Anyone on the same host can connect and read all secrets.

- **Forgetting to save the initial root token and unseal keys.** `vault operator init` output is shown exactly once. If you lose the unseal key shares below the threshold, Vault is permanently sealed and unrecoverable. If you lose the root token before creating other admin tokens, you cannot manage Vault. Copy this output to at least two separate secure locations before proceeding.

- **Skipping TLS in production.** Vault transmits your secrets in the response body. Without TLS, a network-level observer (a rogue container on the same host, a misconfigured load balancer, a cloud VPC tap) can read every secret in plaintext.

- **Not granting `IPC_LOCK`.** Without memory locking, secrets can leak to swap. On Linux this is either `setcap cap_ipc_lock=+ep /usr/bin/vault`, `--cap-add=IPC_LOCK` in Docker/Kubernetes, or disabling swap entirely.

- **Using `/tmp` as the Raft storage path.** Some tutorials point the Raft storage path at `/tmp` for convenience. `/tmp` is cleared on reboot, making your "production" Vault ephemeral. Always use a dedicated persistent volume or directory (e.g., `/opt/vault/data`).

## Summary

- Install Vault via the official HashiCorp apt or dnf repository to get managed upgrades and a systemd unit file.
- The binary installation works on any platform: download the zip from `releases.hashicorp.com`, verify the SHA-256 checksum, and move the binary to `/usr/local/bin/`.
- When running in Docker, always pass `--cap-add=IPC_LOCK` to prevent secrets from being written to swap.
- `IPC_LOCK` lets Vault call `mlock(2)` to pin memory pages in RAM — without it, decrypted secrets and unseal keys can leak to the swap partition.
- Dev mode (`vault server -dev`) is ephemeral, auto-unsealed, HTTP-only, and prints the root token on startup — it is safe only for local development.
- Production mode requires a config file, TLS certificates, `vault operator init` to generate unseal keys and the root token, and `vault operator unseal` after every restart.
- The initial root token and unseal key shares are shown exactly once — store them securely before doing anything else.
