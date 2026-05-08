# KV Secrets Engine: Storing and Retrieving Static Secrets

The KV (Key-Value) secrets engine is the most commonly used engine in Vault — it stores arbitrary key-value pairs encrypted at rest and makes them available to authorised clients. This lesson covers the difference between KV v1 and v2, how to enable the engine, and how to perform create, read, update, delete, and list operations from the CLI, the HTTP API, and the UI.

## KV v1 vs KV v2

Vault ships with two versions of the KV engine. Choose v2 for all new deployments — the only reason to use v1 is compatibility with very old Vault clusters.

| Feature | KV v1 | KV v2 |
|---------|-------|-------|
| Versioning | No — writes overwrite previous data | Yes — each write creates a new version |
| Soft-delete | No | Yes — deleted data can be recovered |
| Metadata | No | Yes — creation time, deletion time, custom labels |
| Check-and-set (CAS) | No | Yes — conditional writes prevent race conditions |
| API path prefix | `/v1/<mount>/<path>` | `/v1/<mount>/data/<path>` |
| `vault kv` sub-commands | `get`, `put`, `list`, `delete` | All v1 commands plus `patch`, `undelete`, `destroy`, `metadata` |

## Enabling the KV Engine

KV v2 is enabled by default at the `secret/` mount when you run `vault server -dev`. For a production server you enable it explicitly:

```bash
# Enable KV v2 at the default "secret/" path
vault secrets enable -path=secret kv-v2

# Enable KV v1 at a custom path (legacy use case)
vault secrets enable -version=1 -path=legacy kv

# Verify both are listed
vault secrets list
```

Expected output from `vault secrets list`:

```
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_xxxxxxxx    per-token private secret storage
identity/     identity     identity_xxxxxxxx     identity store
legacy/       kv           kv_xxxxxxxx           n/a
secret/       kv           kv_xxxxxxxx           n/a
sys/          system       system_xxxxxxxx       system endpoints used for control...
```

## KV v1 Operations

KV v1 is straightforward — every write replaces the previous value completely.

```bash
# Write a secret
vault kv put secret/myapp/config db_password=s3cret db_host=db.internal

# Read a secret
vault kv get secret/myapp/config

# List all keys under a prefix
vault kv list secret/myapp/

# Delete a secret (permanent in v1)
vault kv delete secret/myapp/config
```

Sample output of `vault kv get secret/myapp/config` in v1:

```
====== Data ======
Key            Value
---            -----
db_host        db.internal
db_password    s3cret
```

No metadata, no version information — just the raw key-value pairs.

## KV v2 CRUD Operations

KV v2 uses the same CLI sub-commands, but the response includes versioning metadata.

### Writing a Secret

```bash
vault kv put secret/myapp/config db_password=s3cret db_host=db.internal
```

```
====== Secret Path ======
secret/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2024-03-10T12:00:00.000000000Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

Notice `version: 1` — this is the first version of this secret.

### Reading a Secret

```bash
vault kv get secret/myapp/config
```

```
====== Secret Path ======
secret/data/myapp/config

======= Metadata =======
Key                Value
---                -----
created_time       2024-03-10T12:00:00.000000000Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1

====== Data ======
Key            Value
---            -----
db_host        db.internal
db_password    s3cret
```

### Listing Keys

```bash
vault kv list secret/myapp/
```

```
Keys
----
config
credentials
```

`list` shows only key names, not values. Use it to discover what secrets exist under a prefix.

### Deleting a Secret

```bash
# Soft-delete the current version (recoverable in v2 — see the next lesson)
vault kv delete secret/myapp/config
```

## Patching: Updating Specific Fields

`vault kv put` replaces **all** fields in a secret. If you have a secret with ten fields and only want to change one, `vault kv patch` updates only the specified fields without touching the rest.

```bash
# Current state: {db_password: s3cret, db_host: db.internal}
vault kv patch secret/myapp/config db_password=n3wsecret
# Result: {db_password: n3wsecret, db_host: db.internal} -- db_host unchanged
```

`patch` creates a new version (like any write in v2). Under the hood it reads the current secret, merges the new fields, and writes the result with a CAS check.

```bash
# Verify the patch
vault kv get secret/myapp/config
```

```
======= Metadata =======
version  2          <-- new version created

====== Data ======
db_host        db.internal
db_password    n3wsecret
```

## Reading from the HTTP API

All CLI commands are wrappers around the Vault HTTP API. In KV v2 the data lives under a `/data/` path segment inserted between the mount and the secret path.

```bash
# CLI path:  secret/myapp/config
# API path:  /v1/secret/data/myapp/config

curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  $VAULT_ADDR/v1/secret/data/myapp/config
```

Response:

```json
{
  "request_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "data": {
    "data": {
      "db_host": "db.internal",
      "db_password": "n3wsecret"
    },
    "metadata": {
      "created_time": "2024-03-10T12:00:00.000000000Z",
      "custom_metadata": null,
      "deletion_time": "",
      "destroyed": false,
      "version": 2
    }
  }
}
```

The secret data is nested at `.data.data` — the outer `data` is the API response envelope, the inner `data` is the KV v2 data object.

### Writing via the API

```bash
curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST \
  --data '{"data": {"db_password": "api-written"}}' \
  $VAULT_ADDR/v1/secret/data/myapp/config
```

Note the required `{"data": {...}}` wrapper — KV v2 expects the secret values inside a `data` key.

### Listing via the API

```bash
curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  -X LIST \
  $VAULT_ADDR/v1/secret/metadata/myapp/
```

## Getting JSON Output from the CLI

The `-format=json` flag makes `vault kv get` output raw JSON, which you can pipe to `jq` for scripting:

```bash
# Extract only the secret data fields
vault kv get -format=json secret/myapp/config | jq .data.data
```

```json
{
  "db_host": "db.internal",
  "db_password": "n3wsecret"
}
```

Extract a single value:

```bash
vault kv get -format=json secret/myapp/config | jq -r .data.data.db_password
# n3wsecret
```

This pattern is widely used in shell scripts that need to inject secrets into environment variables or config files.

## Reading from the UI

1. Log in to the Vault UI at `http://127.0.0.1:8200/ui`.
2. Click **Secrets** in the top navigation.
3. Click the **secret/** engine.
4. Navigate into **myapp/** then click **config**.
5. The UI displays the current version's key-value pairs. Click **Copy** next to any field to copy its value.
6. Click **Version history** to see all versions (v2 only).

## Common Pitfalls

- **Using `vault kv put` when you only want to update one field** — `put` replaces the entire secret. If you have `{a: 1, b: 2}` and run `vault kv put secret/myapp/config a=99`, the result is `{a: 99}` — `b` is gone. Use `vault kv patch` to update specific fields.
- **Forgetting the `/data/` path segment in the HTTP API** — the KV v2 API path for reading is `/v1/<mount>/data/<path>`, not `/v1/<mount>/<path>`. Using the wrong path returns a 404 or an incorrect response from a different KV operation (e.g., hitting the metadata endpoint).
- **Expecting KV v2 data at `.data` in JSON output** — the secret values are at `.data.data`, not `.data`. The extra nesting is intentional: the outer `.data` is Vault's API response envelope; the inner `.data` is the KV v2 data object that also contains `.metadata`.
- **Enabling KV without specifying the version** — `vault secrets enable kv` defaults to KV v1 on most Vault versions. To get v2, explicitly use `vault secrets enable kv-v2` or `vault secrets enable -version=2 kv`.
- **Listing without a trailing slash** — `vault kv list secret/myapp` (no slash) may return an error or unexpected results depending on the Vault version. Always append `/` when listing: `vault kv list secret/myapp/`.

## Summary

- KV v1 is a simple key-value store with no versioning; KV v2 adds versioning, soft-delete, metadata, and check-and-set.
- Enable KV v2 with `vault secrets enable -path=secret kv-v2`; it is enabled by default in dev mode.
- Core CRUD commands: `vault kv put` (write/overwrite), `vault kv get` (read), `vault kv list` (list keys), `vault kv delete` (soft-delete in v2).
- Use `vault kv patch` to update specific fields in an existing secret without overwriting others.
- The HTTP API path for KV v2 data is `/v1/<mount>/data/<path>` — note the inserted `/data/` segment.
- Pass the token in the `X-Vault-Token` header for authenticated API calls.
- Use `-format=json` with `vault kv get` and pipe to `jq` to extract values in scripts; secret data is at `.data.data` in the JSON output.
