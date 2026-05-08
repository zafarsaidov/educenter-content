# Writing and Testing Policies

Good policies follow the principle of least privilege: grant only the exact paths and capabilities an application needs, nothing more. This lesson works through two realistic policy examples, demonstrates policy templating with identity metadata, shows how to assign policies to tokens and auth method roles, and covers the `vault token capabilities` command for verifying effective permissions.

## Principle of Least Privilege

Every path you add to a policy is an attack surface. A leaked token can only reach paths the policy permits. Start with the narrowest possible policy — a single path with a single capability — and expand only when the application demonstrates a specific need.

Ask three questions for each path you consider adding:

1. Does the application need to access this path at runtime? (Not during setup, not occasionally — at runtime.)
2. Which specific capabilities does it need? (Not "read and write" unless both are genuinely required.)
3. Can you scope the path to a specific prefix rather than a wildcard?

## Worked Example: Application Policy

An application needs to read its own secrets from KV v2 at `secret/data/myapp/*` and list the available keys at `secret/metadata/myapp/` so it can iterate over them on startup.

```hcl
# myapp-policy.hcl

# Read any secret under myapp
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

# List available secret keys so the app can discover them
# Note: "list" on /metadata/ is separate from "read" on /data/
path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}
```

Write it to Vault:

```bash
vault policy write myapp-policy myapp-policy.hcl
```

What this policy does NOT grant:

- The application cannot create or update secrets (no `create` or `update`).
- It cannot read secrets from any other application's path.
- It cannot read KV v2 metadata details (version history, custom metadata) — only enumerate key names.

## Worked Example: CI Pipeline Policy

A CI pipeline needs to read deployment secrets from `secret/data/ci/*` (database URLs, API keys used during tests) and write build artifacts — checksums, build metadata — to `secret/data/ci/artifacts/*`.

```hcl
# ci-pipeline-policy.hcl

# Read CI secrets (credentials, config) — read-only
path "secret/data/ci/*" {
  capabilities = ["read"]
}

# Write build artifacts — create and update allowed, but not delete
path "secret/data/ci/artifacts/*" {
  capabilities = ["create", "update"]
}

# List artifact keys so the pipeline can check what exists
path "secret/metadata/ci/artifacts/*" {
  capabilities = ["list"]
}
```

Notice that the more permissive `create`/`update` capability is scoped to the narrower `ci/artifacts/*` sub-path. The broader `ci/*` path only allows reading.

```bash
vault policy write ci-pipeline-policy ci-pipeline-policy.hcl
```

## Policy Templating with Identity

Hard-coding a specific username or application name in a policy makes the policy non-reusable. Vault supports **identity templating** — at policy evaluation time, Vault replaces template expressions with values from the authenticated entity's identity.

### `{{identity.entity.name}}`

Inserts the name of the authenticated entity (usually derived from the username used to log in).

```hcl
# Give every user read/write access to their own personal secrets namespace
path "secret/data/users/{{identity.entity.name}}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/metadata/users/{{identity.entity.name}}/*" {
  capabilities = ["list", "delete"]
}
```

With this single policy, `alice` can access `secret/data/users/alice/*` and `bob` can access `secret/data/users/bob/*`. Neither can access the other's path.

### `{{identity.entity.aliases.<mount_accessor>.name}}`

Uses the alias name (the username as seen by a specific auth method) rather than the entity name. This is useful when the entity name differs from the login username.

To find the mount accessor for an auth method:

```bash
vault auth list -format=json | jq '."ldap/".accessor'
# or
vault auth list
# Look at the "Accessor" column for the ldap/ row
```

```hcl
# Use the LDAP username (alias) as the path segment
path "secret/data/users/{{identity.entity.aliases.auth_ldap_abc123.name}}/*" {
  capabilities = ["create", "read", "update", "delete"]
}
```

Replace `auth_ldap_abc123` with the actual accessor value for your LDAP auth mount.

### Full Example: Per-User Policy

```hcl
# user-personal-policy.hcl
# Assign this single policy to all human operators.
# Each user automatically gets access only to their own path.

path "secret/data/personal/{{identity.entity.name}}/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

path "secret/metadata/personal/{{identity.entity.name}}/*" {
  capabilities = ["list"]
}

# Everyone can read shared team secrets
path "secret/data/shared/*" {
  capabilities = ["read"]
}
```

## Assigning Policies to Tokens

### Direct Token Creation

```bash
# Create a token with one policy
vault token create -policy=myapp-policy

# Create a token with multiple policies
vault token create -policy=myapp-policy -policy=ci-pipeline-policy -ttl=1h
```

A token can carry multiple policies. The effective capabilities at any path are the union of all matching rules across all attached policies. Remember that `deny` overrides the union.

### AppRole Roles

```bash
vault write auth/approle/role/my-app \
  token_policies="myapp-policy,read-shared" \
  token_ttl=1h \
  token_max_ttl=4h
```

Every token issued by this AppRole role will carry both `myapp-policy` and `read-shared`.

### Kubernetes Auth Roles

```bash
vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=production \
  policies=myapp-policy \
  ttl=1h
```

### LDAP Group Mapping

```bash
# Map the "engineers" LDAP group to a policy
vault write auth/ldap/groups/engineers policies=myapp-policy,read-shared
```

## Testing Policies

### `vault token capabilities`

`vault token capabilities <token> <path>` returns the capabilities a specific token has at a specific path. Use it to verify a policy before deploying an application.

```bash
# Check capabilities of a token at a specific path
vault token capabilities hvs.CAESIBb... secret/data/myapp/config
```

Output if access is granted:

```
create, read, update
```

Output if access is denied:

```
deny
```

```bash
# Check capabilities of your current token (no token argument needed)
vault token capabilities secret/data/myapp/config
```

### Systematic Policy Verification

Before deploying a new application, verify each path the application will use:

```bash
# 1. Create a test token with only the application's policy
TEST_TOKEN=$(vault token create -policy=myapp-policy -ttl=15m -field=token)

# 2. Test each path the application needs
vault token capabilities $TEST_TOKEN secret/data/myapp/config
vault token capabilities $TEST_TOKEN secret/metadata/myapp/

# 3. Verify paths the application should NOT access are denied
vault token capabilities $TEST_TOKEN secret/data/otherapp/config
# Expected: deny

# 4. Revoke the test token
vault token revoke $TEST_TOKEN
```

## Common Policy Mistakes

### Granting `secret/*` Instead of `secret/data/*` in KV v2

KV v2 uses a two-level path hierarchy. The actual secret data lives at `secret/data/<key>`, not `secret/<key>`. A policy granting `secret/*` would match the mount itself but not the sub-paths where data is stored.

```hcl
# WRONG for KV v2 - does not reach actual secret data
path "secret/*" {
  capabilities = ["read"]
}

# CORRECT for KV v2 - matches the data sub-path
path "secret/data/myapp/*" {
  capabilities = ["read"]
}
```

### Forgetting `list` When the Application Iterates Over Keys

If your application calls `vault kv list` to discover available secrets, it needs `list` on the `secret/metadata/<prefix>/` path. The `read` capability on `secret/data/<prefix>/*` does not enable listing.

```hcl
# MISSING: application cannot list keys
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

# CORRECT: add the metadata list path
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}
```

### Forgetting `/metadata/` for KV v2 Metadata Operations

KV v2 metadata (custom metadata, max_versions, delete_version_after) lives at `secret/metadata/<key>`, not `secret/data/<key>`. If your application or operator needs to set custom metadata:

```hcl
# Allow reading and writing KV v2 metadata
path "secret/metadata/myapp/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

## Common Pitfalls

- **Granting `*` on the KV mount root** — `path "secret/*"` is very broad and, in KV v2, still does not map to actual secret data without the `/data/` sub-path. Scope policies to specific sub-paths under `secret/data/` and `secret/metadata/`.
- **Not testing `deny` paths explicitly** — verifying that an application can read its own secrets is only half the test. Also verify that it cannot read another application's secrets. Untested deny paths lead to privilege escalation.
- **Omitting `list` from CI pipelines that iterate keys** — pipelines often discover secrets by listing a prefix and then reading each one. Forgetting `list` produces a confusing `403` error at the list step, not the read step.
- **Hardcoding usernames in policies instead of using templates** — `path "secret/data/users/alice/*"` creates a separate policy per user. Use `{{identity.entity.name}}` to write one policy that serves all users.
- **Assigning too many policies to a single token** — having 20 policies on a token is hard to audit and may produce unexpected capability unions. Prefer fewer, well-scoped policies. If two applications need slightly different access, create two different roles with two different policies rather than sharing one broad policy.

## Summary

- Apply the principle of least privilege: grant only the paths and capabilities each application or user actually needs at runtime.
- For a read-only application: `secret/data/<app>/*` with `["read"]` plus `secret/metadata/<app>/*` with `["list"]`.
- For a CI pipeline that also writes: scope `create` and `update` to the specific sub-path where writes are needed.
- Identity templates (`{{identity.entity.name}}`) let a single policy serve all users by inserting the authenticated entity's name into the path at evaluation time.
- `vault token create -policy=<name>` and auth method role configurations (AppRole, Kubernetes, LDAP) are how policies are attached to tokens.
- `vault token capabilities <token> <path>` shows the effective capabilities at a specific path — use it to verify policies before deploying.
- Common mistakes: using `secret/*` instead of `secret/data/*` in KV v2, omitting `list` when iteration is needed, and forgetting the `/metadata/` path for metadata operations.
