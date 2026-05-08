# The Vault Policy Language

Policies are the authorisation layer in Vault — named HCL documents that state exactly which paths a token can access and with which operations. If no policy grants a capability for a given path, the request is denied. This lesson covers every part of the policy language: capabilities, path matching, the two built-in policies, and the full lifecycle of a policy from write to deletion.

## What Policies Are

When Vault receives an API request, it looks up the token, finds all policies attached to it, evaluates every policy rule whose path matches the requested path, and collects the union of capabilities. If the required capability (e.g., `read`) appears in that union, the request succeeds; otherwise it is denied.

```
Request: GET secret/data/myapp/db-password
Token policies: [default, my-app-policy]

Evaluate default:
  path "auth/token/lookup-self" { capabilities = ["read"] }   → no match
  path "secret/data/myapp/*"    { ... }                       → no match (not in default)

Evaluate my-app-policy:
  path "secret/data/myapp/*" { capabilities = ["read"] }      → MATCH ✓

Union of capabilities for this path: [read]
Required capability: read
Decision: ALLOW
```

Policies are attached to a token at creation time and do not change for the lifetime of that token. To change what a token can do, you revoke it and issue a new one.

## Policy Language Structure

A policy file contains one or more `path` blocks. Each block specifies a path (or glob pattern) and a set of capabilities.

```hcl
# Allow reading any secret under myapp
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

# Allow listing the keys at the myapp metadata path
path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}

# Deny everything under a specific sub-path, even if another rule would allow it
path "secret/data/myapp/internal/*" {
  capabilities = ["deny"]
}
```

Comments use `#`. Path strings are quoted. Capability values are a list of strings.

## Capabilities

| Capability | HTTP Method | What it means |
|------------|-------------|---------------|
| `create` | POST (new path) | Write a value at a path that does not yet exist |
| `read` | GET | Read the current value at a path |
| `update` | POST/PUT (existing path) | Overwrite an existing value at a path |
| `delete` | DELETE | Delete a secret or metadata |
| `list` | LIST | Enumerate the keys at a path — does **not** allow reading their values |
| `sudo` | any | Access root-protected paths (e.g., `sys/seal`, `sys/raw`); very rarely needed |
| `deny` | any | Explicitly deny access; overrides every other capability |

Important nuances:

- **`list` does not imply `read`** — a token that can list keys at `secret/metadata/myapp/` will see key names but cannot read the secret values at `secret/data/myapp/<key>` unless it also has `read` on that path.
- **`create` and `update` are distinct** — some secrets engines distinguish between creating a new secret (`create`) and modifying an existing one (`update`). When in doubt, grant both.
- **`deny` takes absolute precedence** — if any policy attached to a token has `deny` for a path, the request is denied even if every other policy grants all capabilities.
- **`sudo` is for system paths** — regular application policies never need `sudo`. It is required only for a small number of administrative operations like sealing Vault or accessing raw storage.

## Path Globbing

Vault supports three types of path matching:

### Exact Path

No wildcard — matches only the precise path specified.

```hcl
path "secret/data/myapp/config" {
  capabilities = ["read"]
}
```

This matches `secret/data/myapp/config` and nothing else. `secret/data/myapp/config/extra` and `secret/data/myapp/other` do not match.

### Trailing Glob (`*`)

`*` at the end of a path matches zero or more characters including path separators.

```hcl
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}
```

This matches:
- `secret/data/myapp/config`
- `secret/data/myapp/db/password`
- `secret/data/myapp/` (the path itself)

The `*` must be the final character in the path string.

### Single-Segment Glob (`+`)

`+` matches exactly one path segment (characters up to the next `/`). It can appear anywhere in the path and multiple times.

```hcl
# Match the "config" key in any app's secret path
path "secret/data/+/config" {
  capabilities = ["read"]
}
```

This matches:
- `secret/data/myapp/config`
- `secret/data/otherapp/config`

But not:
- `secret/data/myapp/settings/config` (two segments, not one)
- `secret/data/config` (no segment before `config`)

`+` is especially useful in policy templates where you want to scope access to one position in the path hierarchy without listing every possible value.

## Built-in Policies

Vault ships with two policies that are always present and cannot be deleted.

### `root`

Assigned only to the root token. Grants every capability on every path with no restrictions. Never assign this policy to application tokens.

### `default`

Automatically added to every token unless explicitly excluded with `vault token create -no-default-policy`. It grants a small set of safe self-service permissions:

```hcl
# Allows a token to look up its own properties
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

# Allows a token to renew itself
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Allows a token to revoke itself
path "auth/token/revoke-self" {
  capabilities = ["update"]
}

# Allows access to the token's own cubbyhole (private key-value storage)
path "cubbyhole/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Allows looking up the token's own capabilities at a given path
path "sys/capabilities-self" {
  capabilities = ["update"]
}
```

The `default` policy is safe to leave attached to all tokens. Only remove it if you need complete isolation (e.g., a token that must not be able to renew itself).

## Policy Lifecycle

### Writing a Policy

Create a file `my-policy.hcl` and write it to Vault:

```hcl
# my-policy.hcl
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

path "secret/metadata/myapp/*" {
  capabilities = ["list"]
}
```

```bash
vault policy write my-policy my-policy.hcl
```

Policy names are lowercase, alphanumeric, and may include hyphens and underscores. The name is how tokens reference the policy at creation time.

### Listing Policies

```bash
vault policy list
```

Output:

```
default
my-policy
root
```

### Reading a Policy

```bash
vault policy read my-policy
```

Prints the full HCL content of the policy.

### Deleting a Policy

```bash
vault policy delete my-policy
```

Deleting a policy does not revoke existing tokens that have it attached. Those tokens retain whatever cached permissions they had until they expire or are explicitly revoked. New tokens can no longer reference the deleted policy name.

## Common Pitfalls

- **Granting `secret/*` instead of `secret/data/*`** — in KV v2, the actual secret data lives at the `/data/` prefix (`secret/data/<key>`). Granting `secret/*` matches the mount path but not the data sub-paths, so reads will fail. Always include `data` in KV v2 paths.
- **Granting `read` without `list`** — an application that reads secrets by iterating over keys (rather than hardcoding paths) also needs `list` on the metadata path (`secret/metadata/myapp/`). Without `list`, the application cannot discover key names.
- **Using `deny` when you meant to omit a path** — omitting a path and setting `deny` both result in "access denied", but `deny` actively overrides any other policy. Use omission for paths you simply do not grant; reserve `deny` for explicit overrides of broader globs.
- **Forgetting that policy changes do not affect issued tokens** — updating a policy does not change the capabilities of tokens already issued with that policy. If you need to restrict an issued token immediately, revoke it.

## Summary

- A policy is a named HCL file containing `path` blocks that list the capabilities a token has at that path.
- Capabilities are `create`, `read`, `update`, `delete`, `list`, `sudo`, and `deny`; `deny` overrides everything else.
- `list` does not imply `read` — they are distinct capabilities that must be granted separately.
- Path matching uses exact paths, trailing `*` for prefix globbing, and `+` for single-segment wildcards.
- The `root` policy is all-powerful; the `default` policy provides safe self-service permissions and is automatically attached to every token.
- Manage policies with `vault policy write`, `vault policy list`, `vault policy read`, and `vault policy delete`.
- Deleting or modifying a policy does not affect tokens already in circulation — revoke them to enforce the change immediately.
