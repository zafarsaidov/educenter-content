# Running the Vault Dev Server

The dev server is the fastest way to get Vault running on your local machine — one command gives you a fully initialised, unsealed Vault instance with a root token ready to use. This lesson walks through starting the dev server, reading its output, setting up your shell environment, and understanding why this mode must never be used in production.

## Starting the Dev Server

Run the following command in a terminal:

```bash
vault server -dev
```

Vault prints a block of information to stdout and stays running in the foreground. Keep this terminal open — closing it shuts down the server and erases all data.

### Reading the Startup Output

```
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Go Version: go1.21.5
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mlock: supported: false, enabled: false
           Recovery Mode: false
                 Storage: inmem
                 Version: Vault v1.16.0

==> Vault server started! Log data will stream in below:

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is:

Root Token: hvs.AAAAAAAAAAAAAAAAAAAAAAAAA

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: kPYyHHBvRFMHVsNKaHTCNvRK3E/wXy9tFD0lFjzCsIQ=
Root Token: hvs.AAAAAAAAAAAAAAAAAAAAAAAAA

Development mode should NOT be used in production installations!
```

Key pieces of information:

| Field | Value | What it means |
|-------|-------|---------------|
| `Api Address` | `http://127.0.0.1:8200` | Where clients send requests |
| `Cluster Address` | `https://127.0.0.1:8201` | Used for server-to-server HA communication |
| `Storage` | `inmem` | All data lives in memory only |
| `tls` | `disabled` | No TLS in dev mode |
| `Unseal Key` | base64 string | The single unseal key (dev mode only has one) |
| `Root Token` | `hvs.AAAA...` | Full superuser token — copy this |

## Setting Up Your Shell Environment

Open a **second terminal** (leave the first running the server) and export two environment variables that the `vault` CLI reads automatically:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.AAAAAAAAAAAAAAAAAAAAAAAAA'
```

Replace the token value with the actual `Root Token` printed by your server. You can add these lines to `~/.bashrc` or `~/.zshrc` during development so they persist across terminal sessions.

## Verifying the Connection

```bash
vault status
```

Expected output:

```
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.16.0
Build Date      2024-02-01T09:00:00Z
Storage Type    inmem
Cluster Name    vault-cluster-XXXXXXXX
Cluster ID      xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
HA Enabled      false
```

`Sealed: false` confirms the server is ready to accept requests.

## Exploring the Vault UI

The dev server also runs a web UI. Open a browser and navigate to:

```
http://127.0.0.1:8200/ui
```

On the login screen:

1. Select **Token** as the method.
2. Paste your root token into the **Token** field.
3. Click **Sign In**.

The UI gives you a point-and-click view of secrets engines, auth methods, policies, and leases — useful for learning and debugging, though most production workflows use the CLI or API.

```
+--------------------------------------+
|        Vault Web UI (port 8200)      |
|                                      |
|  Method: Token                       |
|  Token:  hvs.AAAA...                 |
|                                      |
|  [Sign In]                           |
+--------------------------------------+
         |
         v
+--------------------------------------+
|  Secrets  |  Access  |  Policies     |
|  -------- |  ------  |  --------     |
|  secret/  | token/   | default       |
|  cubby... | approle/ | root          |
+--------------------------------------+
```

## Why Dev Mode Is NOT for Production

Dev mode makes several trade-offs that are convenient for learning but catastrophic in production:

- **No persistence** — all secrets are stored in memory. Restarting the server erases everything.
- **Fixed unseal key** — the same unseal key is used every run, offering no real security.
- **No TLS** — all traffic between clients and the server is unencrypted.
- **Root token in plaintext** — the token is printed to stdout and readable by anyone with terminal access.
- **Single unseal key, threshold 1** — real deployments use key splitting (e.g., 5 shares, threshold 3) so no single person can unseal alone.

## Useful Dev Mode Flags

You can customise the dev server with flags for scripting and testing:

```bash
# Fix the root token to a known value (useful in CI or demo scripts)
vault server -dev -dev-root-token-id=mytoken

# Listen on all interfaces (needed if another container or VM needs to reach it)
vault server -dev -dev-listen-address=0.0.0.0:8200

# Combine both
vault server -dev -dev-root-token-id=mytoken -dev-listen-address=0.0.0.0:8200
```

When you use `-dev-root-token-id`, set `VAULT_TOKEN` to the same value:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='mytoken'
vault status
```

## Common Pitfalls

- **Forgetting to export `VAULT_ADDR`** — without it, the CLI defaults to `https://127.0.0.1:8200` (note HTTPS). The dev server runs on plain HTTP, so every command will fail with a TLS handshake error. Always export `VAULT_ADDR='http://127.0.0.1:8200'` before using the CLI against a dev server.
- **Running commands in the same terminal as the server** — the server occupies the foreground of its terminal. Open a second terminal and set your env vars there.
- **Restarting the dev server and using the old token** — each restart generates a new root token (unless you use `-dev-root-token-id`). If you see `permission denied` after a restart, re-export the new token.
- **Assuming dev mode behaviour in production** — the auto-unseal and auto-init in dev mode do not exist in a real deployment. You must run `vault operator init` and `vault operator unseal` explicitly.

## Summary

- `vault server -dev` starts an in-memory, pre-initialised, pre-unsealed Vault instance — perfect for learning and development.
- The startup output contains the `Unseal Key`, `Root Token`, and `Api Address` — copy the root token before running any other commands.
- Set `VAULT_ADDR` and `VAULT_TOKEN` in a second terminal so the CLI knows where to connect and how to authenticate.
- `vault status` confirms the server is running and unsealed.
- The web UI is available at `http://127.0.0.1:8200/ui` — log in with the root token.
- Dev mode is not for production: no persistence, no TLS, no key splitting, root token in plaintext.
- Use `-dev-root-token-id` to pin the token for scripted workflows, and `-dev-listen-address` to accept connections from other hosts.
