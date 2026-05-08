# Initialising, Unsealing, and the Essential Vault CLI

A freshly installed Vault server is uninitialised and sealed — it holds no data and will not serve any requests until you explicitly initialise and unseal it. This lesson walks through `vault operator init`, the unseal process, token-based authentication, and the core CLI commands you will use in every Vault workflow.

## `vault operator init`: Initialising a New Vault

Initialisation generates the encryption keys that protect Vault's data and produces the initial root token. You run this **once** per cluster — never again unless you are re-keying.

```bash
vault operator init
```

Default output (5 key shares, threshold of 3):

```
Unseal Key 1: 4Zu5K8CkXJX7yFLkHuKhR2j3VqPpn1wy3dkF2aJfXtA=
Unseal Key 2: 7BmDpLcNqW9RvTsK5hJzX1oY6MeQgPm0fBnE3CiLuZs=
Unseal Key 3: 9DpHqRvYmX2KjB4wF6cNL8sT0eZaWgQ1nIoU5lAyCtM=
Unseal Key 4: 2FrZeKnTmB6VoJ9qH3dNS1uX7yPwGaQ5lCsR4iADXfE=
Unseal Key 5: 1CtYxJgHvN8WmP3kL6qFR0bT9AoZeUiS4dQy7nBsXlK=

Initial Root Token: hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

Vault initialized with 5 key shares and a key threshold of 3.
Please securely distribute the key shares listed above.
When Vault is re-sealed, restarted, or stopped, you must supply at
least 3 of these keys to unseal it again.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!
```

### Key Shares and Threshold

Vault uses **Shamir's Secret Sharing** to split the root encryption key into `N` shares where any `K` of them are sufficient to reconstruct the key.

```
Default: N=5 shares, K=3 threshold

  Share 1  Share 2  Share 3  Share 4  Share 5
    |         |        |        |        |
    +----+----+--------+        |        |
         |                              |
    Any 3 shares --> reconstruct root key --> unseal Vault
```

You can override the defaults:

```bash
vault operator init -key-shares=3 -key-threshold=2
```

### Saving Unseal Keys Safely

- **Distribute** each share to a different person (keyholder).
- **Never store all shares in one place** — doing so defeats the purpose of splitting.
- For production, use **PGP encryption**: provide one PGP public key per share so each keyholder receives their share encrypted to their key.

```bash
vault operator init \
  -key-shares=5 \
  -key-threshold=3 \
  -pgp-keys="alice.asc,bob.asc,carol.asc,dave.asc,eve.asc"
```

Each keyholder receives a base64-encoded, PGP-encrypted blob. They decrypt it with their private key to obtain their share when needed.

## `vault operator unseal`: Providing Key Shares

After every restart (or deliberate `vault operator seal`), Vault is sealed and must be unsealed before it can serve requests. Run `vault operator unseal` once per share until the threshold is met.

```bash
vault operator unseal
```

Vault prompts for a single unseal key:

```
Unseal Key (will be hidden):
```

Paste one share and press Enter. Repeat from a different terminal (or a different keyholder's machine) until the threshold is reached:

```bash
# First key share
vault operator unseal
# Key 1/3:  Unseal Progress 1/3

# Second key share
vault operator unseal
# Key 2/3:  Unseal Progress 2/3

# Third key share — threshold met, Vault unseals
vault operator unseal
# Sealed          false
# Unseal Progress 0/3
```

You can also pass the key directly (useful in automation, but be careful with shell history):

```bash
vault operator unseal "4Zu5K8CkXJX7yFLkHuKhR2j3VqPpn1wy3dkF2aJfXtA="
```

## `vault login`: Authenticating

After unsealing, authenticate with a token or an auth method.

### Token Authentication

```bash
vault login
```

Vault prompts for a token. Paste the root token (or any valid token):

```
Token (will be hidden):
```

Alternatively, pass it directly:

```bash
vault login hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Username/Password Authentication

If the `userpass` auth method is enabled:

```bash
vault login -method=userpass username=alice
```

Vault prompts for the password and, on success, prints a token that is cached at `~/.vault-token`.

## Environment Variables

The `vault` CLI reads three key environment variables:

| Variable | Purpose | Example |
|----------|---------|---------|
| `VAULT_ADDR` | URL of the Vault API endpoint | `http://127.0.0.1:8200` |
| `VAULT_TOKEN` | Token used to authenticate requests | `hvs.CAESI...` |
| `VAULT_NAMESPACE` | Namespace for Vault Enterprise deployments | `admin/teamA` |

Set them in your shell:

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
```

`VAULT_TOKEN` takes precedence over the token cached in `~/.vault-token`.

## Essential CLI Commands Reference

### System Status and Information

```bash
# Check seal status, HA state, version
vault status

# List all enabled secrets engines and their mount paths
vault secrets list

# List all enabled auth methods and their mount paths
vault auth list

# List all policies
vault policy list
```

### Token Management

```bash
# Inspect the currently authenticated token
vault token lookup

# Inspect a specific token
vault token lookup hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

# Renew the current token's TTL
vault token renew

# Renew a specific token
vault token renew hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

### Lease Management

Dynamic secrets (covered in later lessons) are tied to leases. Manage leases with:

```bash
# Renew a lease before it expires
vault lease renew database/creds/my-role/AbCdEfGhIjKlMnOp

# Revoke a lease immediately (credential is invalidated right away)
vault lease revoke database/creds/my-role/AbCdEfGhIjKlMnOp
```

### Quick Reference Table

| Command | What it does |
|---------|-------------|
| `vault status` | Seal/unseal state, HA info, version |
| `vault secrets list` | List enabled secrets engines |
| `vault auth list` | List enabled auth methods |
| `vault policy list` | List all policy names |
| `vault token lookup` | Inspect current token (TTL, policies, metadata) |
| `vault token renew` | Extend current token's TTL |
| `vault lease renew <lease-id>` | Extend a dynamic secret lease |
| `vault lease revoke <lease-id>` | Immediately revoke a lease |

## Using the HTTP API Directly with `curl`

Every CLI command maps to an HTTP API call. The API is useful for automation, debugging, or languages without an official Vault SDK.

```bash
# Health check — no token required
curl $VAULT_ADDR/v1/sys/health | jq .
```

Example response:

```json
{
  "initialized": true,
  "sealed": false,
  "standby": false,
  "performance_standby": false,
  "replication_performance_mode": "disabled",
  "replication_dr_mode": "disabled",
  "server_time_utc": 1710000000,
  "version": "1.16.0",
  "cluster_name": "vault-cluster-abc123",
  "cluster_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

For authenticated endpoints, pass the token in the `X-Vault-Token` header:

```bash
# List enabled secrets engines
curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/sys/mounts | jq .

# Read a secret (KV v2)
curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/secret/data/myapp/config | jq .data.data
```

### Common API Patterns

| Operation | Method | Path |
|-----------|--------|------|
| Health check | `GET` | `/v1/sys/health` |
| Seal status | `GET` | `/v1/sys/seal-status` |
| List mounts | `GET` | `/v1/sys/mounts` |
| Read KV v2 secret | `GET` | `/v1/<mount>/data/<path>` |
| Write KV v2 secret | `POST` | `/v1/<mount>/data/<path>` |
| List tokens | `LIST` | `/v1/auth/token/accessors` |

## Common Pitfalls

- **Running `vault operator init` on an already-initialised cluster** — the command will return an error. Initialisation is a one-time operation. If you need to generate a new root token, use `vault operator generate-root`.
- **Losing unseal keys** — Vault does not store the root key. If you lose enough shares to fall below the threshold, Vault is permanently sealed. Store each share securely and separately (PGP-encrypted, in a secrets manager, or printed and stored in a safe).
- **Unsealing with fewer shares than the threshold** — Vault silently accepts each share and reports the progress counter, but it will not unseal until the threshold is reached. If you stop at 2/3, Vault remains sealed with no error.
- **Forgetting to set `VAULT_ADDR` before `vault login`** — the CLI will try `https://127.0.0.1:8200` by default. If TLS is not configured, the connection fails. Always set `VAULT_ADDR` before running any `vault` command against a server.
- **Using the root token for day-to-day operations** — the root token has unlimited permissions and no TTL by default. Create purpose-specific tokens with minimal policies and revoke the root token after initial setup. Generate a new root token when needed with `vault operator generate-root`.

## Summary

- `vault operator init` is a one-time command that generates unseal key shares and the initial root token; use `-key-shares` and `-key-threshold` to control the split.
- Vault uses Shamir's Secret Sharing so no single keyholder can unseal alone — distribute shares to different people, optionally PGP-encrypted.
- `vault operator unseal` accepts one share at a time; repeat until the threshold is met and `Sealed` becomes `false`.
- `vault login` authenticates with a token or an auth method (e.g., `-method=userpass`); the token is cached at `~/.vault-token`.
- Set `VAULT_ADDR`, `VAULT_TOKEN`, and optionally `VAULT_NAMESPACE` so the CLI knows where to connect and how to authenticate.
- Core day-to-day commands: `vault status`, `vault secrets list`, `vault auth list`, `vault policy list`, `vault token lookup`, `vault token renew`, `vault lease renew`, `vault lease revoke`.
- Every CLI command has an HTTP API equivalent; use `curl -H "X-Vault-Token: $VAULT_TOKEN"` to call the API directly.
- Do not use the root token for routine operations — create scoped tokens and revoke the root token after initial configuration.
