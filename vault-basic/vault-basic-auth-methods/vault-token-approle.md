# Token and AppRole Authentication

Every operation in Vault is authorised by a token — even when you use a different auth method, the final result is always a token. This lesson covers the token auth method directly (creation, inspection, renewal, and revocation) and then introduces AppRole, the standard pattern for giving automated workloads their own Vault identity without embedding long-lived credentials.

## Token Auth

Vault ships with token auth enabled by default and it cannot be disabled. Every API call carries a token in the `X-Vault-Token` header. When you run `vault operator init`, the very first token you receive is the **root token** — it has every capability on every path and bypasses all policy checks.

```
+-------------------+
|   Client (CLI)    |
|                   |
|  VAULT_TOKEN=xxx  |
+--------+----------+
         |
         | X-Vault-Token: xxx
         v
+--------+----------+
|   Vault Server    |
|                   |
|  Token lookup     |
|  Policy eval      |
|  Route to engine  |
+-------------------+
```

The root token is all-powerful and should only be used for the initial bootstrap. Create narrower tokens for day-to-day operations, then revoke the root token.

### Creating Tokens

```bash
# Create a token attached to a policy with a 1-hour TTL
vault token create -policy=my-policy -ttl=1h

# Create a token with multiple policies
vault token create -policy=read-secrets -policy=list-secrets -ttl=2h

# Create a token that can only be used 5 times (useful for one-shot operations)
vault token create -policy=my-policy -use-limit=5 -ttl=30m
```

The command returns a `token` value along with its `accessor`, `policies`, and `lease_duration`.

### Token Roles

Token roles are reusable templates stored at `auth/token/roles/<name>`. Instead of specifying all parameters on every `vault token create` call, you define them once in a role and reference it.

```bash
# Create a token role
vault write auth/token/roles/my-app-role \
  allowed_policies="my-app-policy" \
  period=1h \
  orphan=true

# Create a token using the role
vault token create -role=my-app-role
```

A token created with `orphan=true` has no parent. Normally, revoking a parent token cascades to all its children — orphan tokens survive that revocation.

## Token Lookup

`vault token lookup` inspects a token and shows its policies, TTL, accessor, and metadata.

```bash
# Look up your own (currently active) token
vault token lookup

# Look up a specific token by value
vault token lookup hvs.CAESIBb...

# Look up a token by its accessor (without knowing the token itself)
vault token lookup -accessor <accessor-value>
```

Example output:

```
Key                  Value
---                  -----
accessor             abc123def456
creation_time        1710000000
creation_ttl         1h
display_name         token
entity_id            n/a
expire_time          2024-03-10T15:00:00Z
explicit_max_ttl     0s
id                   hvs.CAESIBb...
issue_time           2024-03-10T14:00:00Z
meta                 <nil>
num_uses             0
orphan               false
path                 auth/token/create
policies             [default my-policy]
renewable            true
ttl                  59m42s
type                 service
```

The **accessor** is a reference to a token that allows administrative operations (lookup, revoke) without knowing the actual token value. Store accessors rather than tokens when you need to track issued tokens.

## Token Renewal

A token's TTL counts down from creation. Before it expires, you can renew it — subject to `max_ttl`. Once a token reaches its `max_ttl`, renewal is rejected regardless of how much TTL remains.

```bash
# Renew your own token (extends TTL by the original lease duration)
vault token renew

# Renew a specific token
vault token renew hvs.CAESIBb...

# Renew and request a specific new TTL (cannot exceed max_ttl)
vault token renew -increment=2h hvs.CAESIBb...
```

```
Timeline:
  t=0       token created (ttl=1h, max_ttl=4h)
  t=50min   vault token renew → ttl reset to 1h (total elapsed: 50min < 4h ✓)
  t=3h50min vault token renew → ttl reset to 1h would exceed max_ttl=4h
            → Vault grants only the remaining 10min
  t=4h      token expires; cannot be renewed
```

## Token Revocation

Revoking a token immediately invalidates it and cascades revocation to all child tokens it created (unless those children are orphans).

```bash
# Revoke a specific token
vault token revoke hvs.CAESIBb...

# Revoke a token by its accessor
vault token revoke -accessor <accessor-value>

# Revoke all tokens with a specific policy (self-service revocation)
vault token revoke -mode=path auth/token/create
```

Use accessor-based revocation in administrative scripts — you never need to handle the raw token value.

## AppRole Auth

AppRole is designed for **machines and applications**, not humans. It authenticates using two separate credentials:

- **`role_id`** — identifies the application role; not secret, can be baked into a container image or config file
- **`secret_id`** — proves the identity of the specific instance; must be treated as a secret and delivered securely at runtime

Both credentials are required to log in. The separation means that compromising one (e.g., finding a `role_id` in source code) is not enough to obtain a Vault token.

```
+------------------+    +------------------+    +------------------+
|  Orchestrator    |    |  App Instance    |    |  Vault Server    |
|  (CI, Terraform) |    |  (container,     |    |                  |
|                  |    |   daemon, etc.)  |    |                  |
+--------+---------+    +--------+---------+    +--------+---------+
         |                       |                       |
         | 1. fetch secret_id    |                       |
         |------------------------------------------------>
         |<-----------------------------------------------|
         |                       |                       |
         | 2. deliver secret_id  |                       |
         |---------------------->|                       |
         |                       | 3. login(role_id,    |
         |                       |    secret_id)         |
         |                       |----------------------->
         |                       |<-----------------------|
         |                       |   Vault token          |
```

### Enabling AppRole

```bash
vault auth enable approle
```

### Creating a Role

```bash
vault write auth/approle/role/my-app \
  token_policies="my-app-policy" \
  token_ttl=1h \
  token_max_ttl=4h \
  secret_id_ttl=24h \
  secret_id_num_uses=5
```

Key parameters:

| Parameter | Description |
|-----------|-------------|
| `token_policies` | Policies attached to every token this role issues |
| `token_ttl` | Default TTL for tokens issued by this role |
| `token_max_ttl` | Hard cap on TTL — tokens cannot be renewed beyond this |
| `secret_id_ttl` | How long a generated `secret_id` remains valid |
| `secret_id_num_uses` | How many times a `secret_id` can be used (0 = unlimited) |

### Fetching the role_id

```bash
vault read auth/approle/role/my-app/role-id
```

Output:

```
Key        Value
---        -----
role_id    db02de05-fa39-4855-059b-67221c5c2f63
```

The `role_id` is stable for the lifetime of the role and is not sensitive. It is safe to store in a config file or environment variable alongside non-secret configuration.

### Generating a secret_id

```bash
vault write -f auth/approle/role/my-app/secret-id
```

The `-f` flag sends a POST with an empty body. Output:

```
Key                   Value
---                   -----
secret_id             3d471e85-ec17-59bd-9f53-4a40b3f19640
secret_id_accessor    6c66abf8-bc79-4a99-97a7-47a62e67d4aa
secret_id_ttl         24h
secret_id_num_uses    5
```

### Logging In with AppRole

```bash
vault write auth/approle/login \
  role_id=db02de05-fa39-4855-059b-67221c5c2f63 \
  secret_id=3d471e85-ec17-59bd-9f53-4a40b3f19640
```

On success, Vault returns a token:

```
Key                     Value
---                     -----
token                   hvs.CAESIBb...
token_accessor          abc123
token_duration          1h
token_renewable         true
token_policies          ["default" "my-app-policy"]
```

The application stores this token in memory and uses it for subsequent Vault API calls. When the token nears expiry, the application logs in again using the same `role_id` and a freshly generated `secret_id`.

### Response Wrapping for Secure secret_id Delivery

The main challenge with AppRole is delivering the `secret_id` to the application without it being visible in logs, environment variables, or config files. **Response wrapping** solves this: instead of returning the `secret_id` directly, Vault wraps it in a single-use token valid for a short window.

```bash
# Request a wrapped secret_id (valid for 60 seconds, unwrappable only once)
vault write -wrap-ttl=60s -f auth/approle/role/my-app/secret-id
```

Output:

```
Key                              Value
---                              -----
wrapping_token:                  hvs.WRAP...
wrapping_accessor:               abc456
wrapping_token_ttl:              1m
wrapping_token_creation_time:    2024-03-10T14:00:00Z
wrapping_token_creation_path:    auth/approle/role/my-app/secret-id
```

The orchestrator delivers only the `wrapping_token` to the application. The application unwraps it to get the real `secret_id`:

```bash
vault unwrap hvs.WRAP...
```

```
Key               Value
---               -----
secret_id         3d471e85-ec17-59bd-9f53-4a40b3f19640
secret_id_ttl     24h
```

Because the wrapping token can be unwrapped **exactly once**, any interception attempt is detectable: if the application fails to unwrap (the token was already used), you know a third party consumed it.

## Common Pitfalls

- **Using the root token in applications** — the root token has unlimited power and no TTL. If it leaks, every secret in Vault is compromised. Always create purpose-limited tokens with short TTLs and meaningful policies.
- **Not setting `token_max_ttl`** — without a `max_ttl`, tokens can be renewed indefinitely. Set a hard cap (e.g., `token_max_ttl=24h`) so that a compromised token eventually expires even if the attacker keeps renewing it.
- **Baking the `secret_id` into a container image** — `role_id` is safe to bake in, `secret_id` is not. The `secret_id` must be injected at runtime. Use response wrapping or a secrets injection sidecar (e.g., Vault Agent).
- **Setting `secret_id_num_uses=0` in production** — unlimited use means a leaked `secret_id` can be used indefinitely. Set a low number (e.g., 5–10) and rotate regularly.
- **Forgetting that token revocation cascades** — revoking a parent token revokes all its children. If your application creates child tokens, plan your revocation strategy carefully or use orphan tokens where appropriate.

## Summary

- Every Vault operation uses a token; the root token is all-powerful and should be used only for bootstrap.
- `vault token create -policy=<name> -ttl=<duration>` creates a scoped token; token roles are reusable creation templates.
- `vault token lookup` inspects a token; use accessors to look up or revoke tokens without handling raw token values.
- `vault token renew` extends a token's TTL up to `max_ttl`; after `max_ttl` the token expires regardless.
- `vault token revoke` invalidates a token and all its children (unless they are orphans).
- AppRole authenticates machines using two credentials: a non-secret `role_id` (baked into config) and a secret `secret_id` (delivered at runtime).
- Enable with `vault auth enable approle`; create roles at `auth/approle/role/<name>`; log in at `auth/approle/login`.
- Response wrapping (`-wrap-ttl`) turns a `secret_id` into a single-use delivery token — the safest way to hand off a `secret_id` to an application.
