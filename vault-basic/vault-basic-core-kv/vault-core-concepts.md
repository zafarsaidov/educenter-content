# Vault Core Concepts

Vault is built around a small set of primitives that compose together: paths, secrets engines, auth methods, policies, tokens, leases, and audit devices. Understanding how these pieces relate to each other gives you a reliable mental model for every Vault operation you will ever perform.

## Paths: Everything is a URL

Every operation in Vault targets a **path** — a slash-separated string that looks like a filesystem path or URL route. Reading a secret, listing auth methods, creating a policy — all of these are HTTP requests to a path.

```
vault kv get secret/myapp/db
         |      |     |
         |      |     +-- the path within the engine ("myapp/db")
         |      +-------- the mount point of the secrets engine ("secret/")
         +--------------- the CLI subcommand
```

The full path structure is:

```
<mount>/<path>
```

- `<mount>` — where the engine (or auth method, or system endpoint) is mounted.
- `<path>` — the route within that engine. Each engine defines what paths it understands.

### System Paths

Vault also exposes internal endpoints under the `sys/` prefix:

```
sys/mounts         -- manage secrets engine mounts
sys/auth           -- manage auth method mounts
sys/policies       -- manage policies
sys/health         -- cluster health check
sys/seal           -- seal the cluster
sys/unseal         -- unseal the cluster
```

## Secrets Engines

A **secrets engine** is a pluggable backend mounted at a path. When you send a request to a path, Vault routes it to the engine mounted there, which handles the request and returns data.

```
Client Request: GET /v1/secret/data/myapp/config
                         |
                         v
              +-------------------+
              |   Vault Router    |
              +-------------------+
                         |
              matches mount "secret/" --> KV v2 engine
                         |
              +-------------------+
              |    KV v2 Engine   |  --> reads from storage
              +-------------------+
```

### Default Mounts

| Mount | Engine | Purpose |
|-------|--------|---------|
| `secret/` | KV v2 | General-purpose static secrets |
| `sys/` | System | Vault management operations |
| `identity/` | Identity | Entities and groups |
| `cubbyhole/` | Cubbyhole | Per-token private storage |

### Enabling and Disabling Engines

```bash
# Enable KV v2 at the path "secret/"
vault secrets enable -path=secret kv-v2

# Enable the database secrets engine at "database/"
vault secrets enable -path=database database

# List all enabled engines
vault secrets list

# Disable an engine (WARNING: this deletes all data stored by that engine)
vault secrets disable secret/
```

## Auth Methods

An **auth method** is how a client proves its identity to Vault. Once authenticated, the auth method issues a **token** that the client uses for subsequent requests.

```
+----------+                  +------------------+
|  Client  |  credentials --> |   Auth Method    |
|          |                  |  (e.g. AppRole)  |
|          |  <-- token ----- |                  |
+----------+                  +------------------+
                                       |
                              checks configured policies
                              attaches them to token
```

Auth methods are mounted under the `auth/` prefix:

| Mount | Method | Used by |
|-------|--------|---------|
| `auth/token` | Token | Anything with a token |
| `auth/approle` | AppRole | Applications, CI/CD pipelines |
| `auth/kubernetes` | Kubernetes | Pods running in Kubernetes |
| `auth/userpass` | Username/Password | Human operators |
| `auth/github` | GitHub | Developers using GitHub org membership |
| `auth/aws` | AWS | EC2 instances, Lambda functions |

```bash
# Enable the userpass auth method
vault auth enable userpass

# List enabled auth methods
vault auth list

# Disable an auth method
vault auth disable userpass/
```

## Policies

A **policy** is a named HCL document that lists what paths a token can access and with what capabilities. Policies are **additive** — a token with multiple policies gets the union of all their rules.

```hcl
# Example policy: myapp-read-only.hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

path "secret/data/myapp/db" {
  capabilities = ["read"]
}
```

### Capabilities

| Capability | HTTP Method | Description |
|-----------|-------------|-------------|
| `create` | POST/PUT | Write a new value |
| `read` | GET | Read a value |
| `update` | POST/PUT | Overwrite an existing value |
| `delete` | DELETE | Delete a value |
| `list` | LIST | List keys |
| `sudo` | — | Required for certain privileged endpoints |
| `deny` | — | Explicitly deny access (overrides other policies) |

```bash
# Write a policy from a file
vault policy write myapp-read-only myapp-read-only.hcl

# Read a policy
vault policy read myapp-read-only

# List all policies
vault policy list

# Delete a policy
vault policy delete myapp-read-only
```

Two built-in policies exist and cannot be deleted:

- `root` — unlimited access; assigned only to root tokens.
- `default` — a minimal set of permissions (look up own token, renew own token, etc.) assigned to all tokens automatically.

## Tokens

Tokens are Vault's **central authentication primitive**. Every request to Vault must carry a valid token. Auth methods produce tokens; the `vault token create` command also creates them directly.

### Token Properties

```bash
vault token lookup
```

```
Key                  Value
---                  -----
accessor             AbCdEfGhIjKlMnOpQrSt
creation_time        1710000000
creation_ttl         768h
display_name         root
entity_id            n/a
expire_time          n/a
explicit_max_ttl     0s
id                   hvs.CAESIXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
meta                 <nil>
num_uses             0
orphan               true
path                 auth/token/root
policies             [root]
renewable            false
ttl                  0s
type                 service
```

| Field | Description |
|-------|-------------|
| `ttl` | Time remaining before the token expires. `0s` means no expiry (root token). |
| `creation_ttl` | TTL the token started with. |
| `renewable` | Whether `vault token renew` can extend the TTL. |
| `policies` | The list of policies attached to this token. |
| `num_uses` | Number of remaining uses. `0` means unlimited. |
| `orphan` | `true` if this token has no parent in the token tree. |
| `type` | `service` (persistent, renewable) or `batch` (lightweight, not renewable). |

### Service Tokens vs Batch Tokens

```
Service Token                 Batch Token
--------------                -----------
- Stored in Vault storage     - Encoded in the token value itself
- Renewable                   - Not renewable
- Creates a parent-child tree - No parent (effectively orphan)
- More overhead               - Lightweight, scales better
- Default type                - Use for high-volume auth (e.g., Kubernetes)
```

### Token Hierarchy (Parent-Child Tree)

When you create a token, it becomes a child of the token used to create it. Revoking a parent token **recursively revokes all descendants**.

```
root token
├── app-token (policies: myapp)
│   ├── worker-token-1
│   └── worker-token-2
└── ops-token (policies: ops)
    └── audit-token
```

Revoking `app-token` also revokes `worker-token-1` and `worker-token-2`.

An **orphan token** has no parent and is not revoked when other tokens are revoked. Create one with:

```bash
vault token create -orphan -policy=myapp
```

## Leases

**Dynamic secrets** (like database credentials generated on demand) are associated with a **lease** — a time period for which the credential is valid.

```
vault read database/creds/my-role

Key                Value
---                -----
lease_id           database/creds/my-role/AbCdEfGhIjKlMnOp
lease_duration     1h
lease_renewable    true
password           A1B2C3D4E5F6G7H8
username           v-token-my-role-AbCdEfGh
```

| Field | Description |
|-------|-------------|
| `lease_id` | Unique identifier for this lease — used to renew or revoke it. |
| `lease_duration` | How long this credential is valid. |
| `lease_renewable` | Whether the lease can be renewed before expiry. |

When the lease expires, Vault automatically revokes the credential at the source (e.g., drops the database user). You can also revoke early:

```bash
vault lease renew  database/creds/my-role/AbCdEfGhIjKlMnOp
vault lease revoke database/creds/my-role/AbCdEfGhIjKlMnOp
```

Static secrets stored in KV do not have leases.

## Audit Devices

Every request and response that passes through Vault is logged to **audit devices**. Audit logs are your tamper-evident record of all secret access.

```bash
# Enable a file audit device
vault audit enable file file_path=/var/log/vault/audit.log

# List enabled audit devices
vault audit list

# Disable an audit device
vault audit disable file/
```

### Supported Audit Device Types

| Type | Description |
|------|-------------|
| `file` | Appends JSON log entries to a file on disk. |
| `syslog` | Sends log entries to the local syslog daemon. |
| `socket` | Sends log entries over a TCP or UDP socket. |

### What Gets Logged

Each log entry contains the request method, path, token accessor, client IP, response status, and — critically — the **response data**. Sensitive values (passwords, tokens) are **HMAC-hashed** using the audit device's HMAC key, so they cannot be read from the log but can be verified.

```json
{
  "time": "2024-03-10T12:00:00.000Z",
  "type": "response",
  "auth": {
    "accessor": "AbCdEfGhIjKlMnOpQrSt",
    "policies": ["myapp"],
    "token_type": "service"
  },
  "request": {
    "id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "operation": "read",
    "path": "secret/data/myapp/config"
  },
  "response": {
    "data": {
      "data": {
        "db_password": "hmac-sha256:AAAAAAAAAAAAAAAAAAA="
      }
    }
  }
}
```

### The Safety Mechanism

If **all** enabled audit devices fail (disk full, socket unreachable), Vault **stops accepting requests**. This is intentional — Vault refuses to operate without an audit trail. The implication: if you enable an audit device, keep it healthy. Consider enabling two audit devices so that failure of one does not take down Vault.

## Common Pitfalls

- **Confusing the mount point with the path** — `vault kv get secret/myapp/db` addresses the engine mounted at `secret/` and the path `myapp/db` within it. If you mount KV at `apps/`, the command becomes `vault kv get apps/myapp/db`. Mixing these up produces "no handler for route" errors.
- **Assigning the root policy to application tokens** — root access is unlimited and permanent. Applications should receive the narrowest possible policy. Use `vault token create -policy=myapp` to create scoped tokens.
- **Forgetting that revoking a parent token revokes all children** — if an operator's token was used to create application tokens, revoking the operator's token will cascade and revoke the application tokens too. Use orphan tokens for long-lived application identities.
- **Disabling an audit device without a backup** — if you disable the only audit device, Vault can operate without any audit trail until you re-enable one. This is a compliance and security risk.
- **Enabling a single audit device that writes to a full disk** — Vault will stop serving requests when the audit device fails. Monitor the audit log disk and set up alerting before it fills.

## Summary

- Every Vault operation targets a **path** in the form `<mount>/<path>`; Vault routes the request to the engine mounted at `<mount>`.
- **Secrets engines** are pluggable backends (KV, Database, PKI, etc.) mounted at a path; enable with `vault secrets enable`, list with `vault secrets list`.
- **Auth methods** validate client identity and issue tokens; they are mounted under `auth/`; enable with `vault auth enable`.
- **Policies** are named HCL documents listing capabilities (read, write, delete, list, etc.) on specific path patterns; a token inherits the union of all its attached policies.
- **Tokens** are the universal authentication primitive; service tokens are persistent and renewable; batch tokens are lightweight but not renewable; revoking a parent token cascades to all children.
- **Leases** tie dynamic secrets to a validity window; when a lease expires Vault revokes the credential at the source; use `vault lease renew` and `vault lease revoke` to manage them.
- **Audit devices** log every request and response (with sensitive values HMAC-hashed); if all audit devices fail, Vault stops accepting requests to preserve the audit guarantee.
