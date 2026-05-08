# KV v2 Versioning, Soft-Delete, and Metadata

KV v2's versioning system lets you recover from accidental overwrites, track the history of a secret, and control how many versions are retained. This lesson covers reading specific versions, soft-deleting and restoring versions, permanent destruction, metadata management, check-and-set for safe concurrent writes, and cleaning up entire keys.

## How Versioning Works

Every `vault kv put` or `vault kv patch` to a KV v2 path creates a new **version**. Old versions are retained in storage and remain readable until they are explicitly deleted or destroyed.

```
vault kv put secret/myapp/config db_password=v1  --> version 1
vault kv put secret/myapp/config db_password=v2  --> version 2
vault kv put secret/myapp/config db_password=v3  --> version 3

  Version 1  Version 2  Version 3 (current)
  ---------  ---------  -------------------------
  db_password=v1  db_password=v2  db_password=v3
```

`vault kv get` without flags always returns the **current** (latest) version. You can read any previous version explicitly.

## Reading a Specific Version

```bash
# Read version 1
vault kv get -version=1 secret/myapp/config

# Read version 2
vault kv get -version=2 secret/myapp/config
```

Output:

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
db_password    v1
```

Via the HTTP API, append `?version=<n>` to the request:

```bash
curl \
  -H "X-Vault-Token: $VAULT_TOKEN" \
  "$VAULT_ADDR/v1/secret/data/myapp/config?version=1" | jq .data.data
```

## Listing Versions: `vault kv metadata get`

The metadata endpoint shows all versions for a key — their creation time, deletion time, and whether they have been destroyed.

```bash
vault kv metadata get secret/myapp/config
```

```
====== Metadata Path ======
secret/metadata/myapp/config

========== Metadata ==========
Key                     Value
---                     -----
cas_required            false
created_time            2024-03-10T12:00:00.000000000Z
current_version         3
custom_metadata         <nil>
delete_version_after    0s
max_versions            0
oldest_version          1
updated_time            2024-03-10T12:02:00.000000000Z

====== Version 1 ======
Key              Value
---              -----
created_time     2024-03-10T12:00:00.000000000Z
deletion_time    n/a
destroyed        false

====== Version 2 ======
Key              Value
---              -----
created_time     2024-03-10T12:01:00.000000000Z
deletion_time    n/a
destroyed        false

====== Version 3 ======
Key              Value
---              -----
created_time     2024-03-10T12:02:00.000000000Z
deletion_time    n/a
destroyed        false
```

## Soft-Delete: Recoverable Deletion

`vault kv delete` marks one or more versions as **deleted** — the data remains in storage and can be recovered. The current version is deleted by default.

```bash
# Soft-delete the current (latest) version
vault kv delete secret/myapp/config

# Soft-delete specific versions
vault kv delete -versions=1,2 secret/myapp/config
```

After deletion, `vault kv get secret/myapp/config` returns an empty response with deletion metadata:

```
====== Metadata =======
Key                Value
---                -----
deletion_time      2024-03-10T13:00:00.000000000Z
destroyed          false
version            3
```

`destroyed: false` means the data is still there — it is just flagged as deleted.

### Restoring a Soft-Deleted Version

```bash
# Restore version 3
vault kv undelete -versions=3 secret/myapp/config

# Restore multiple versions
vault kv undelete -versions=1,2,3 secret/myapp/config
```

After `undelete`, `vault kv get` returns the data normally and `deletion_time` is cleared.

## Permanent Destroy

`vault kv destroy` permanently removes the data for specific versions. The version entry remains in the metadata (so you can see it existed) but `destroyed: true` is set and the data is gone forever.

```bash
# Permanently destroy version 1
vault kv destroy -versions=1 secret/myapp/config

# Destroy multiple versions
vault kv destroy -versions=1,2 secret/myapp/config
```

After destroy, the metadata shows:

```
====== Version 1 ======
Key              Value
---              -----
created_time     2024-03-10T12:00:00.000000000Z
deletion_time    2024-03-10T13:05:00.000000000Z
destroyed        true
```

`vault kv get -version=1` returns an error — the data cannot be recovered.

### Delete vs Destroy

```
vault kv delete   --> soft-delete: data retained, recoverable with "undelete"
vault kv destroy  --> permanent:   data removed, NOT recoverable
```

Use `delete` for routine secret rotation (keep old versions available for rollback). Use `destroy` for compliance requirements (e.g., "this credential must be gone from all storage").

## Managing Metadata: `vault kv metadata put`

You can configure version retention policies and attach custom metadata to a key using `vault kv metadata put`.

### Capping the Number of Versions (`max_versions`)

By default, Vault retains all versions. Set `max_versions` to cap the history:

```bash
vault kv metadata put -max-versions=5 secret/myapp/config
```

Once the 6th version is written, version 1 is automatically soft-deleted. This prevents unbounded storage growth in active secrets.

### Auto-Expiring Versions (`delete_version_after`)

Automatically mark versions as soft-deleted after a duration:

```bash
# Versions become soft-deleted 30 days after creation
vault kv metadata put -delete-version-after=720h secret/myapp/config
```

This is useful for short-lived credentials that you write to KV but want Vault to clean up automatically.

### Custom Metadata

Attach arbitrary key-value pairs to the key (not to a specific version) for tracking, tagging, or tooling integration:

```bash
vault kv metadata put \
  -custom-metadata=owner=teamA \
  -custom-metadata=environment=production \
  -custom-metadata=rotation-schedule=weekly \
  secret/myapp/config
```

Custom metadata appears in `vault kv metadata get` and in the API response:

```
========== Metadata ==========
custom_metadata    map[environment:production owner:teamA rotation-schedule:weekly]
```

### Setting Multiple Metadata Fields at Once

```bash
vault kv metadata put \
  -max-versions=10 \
  -delete-version-after=168h \
  -custom-metadata=owner=teamA \
  -custom-metadata=environment=production \
  secret/myapp/config
```

## Check-and-Set (CAS): Preventing Race Conditions

In environments where multiple processes may write the same secret simultaneously, CAS ensures a write only succeeds if the secret is at the expected version. This prevents one writer from silently overwriting another writer's changes.

```bash
# Only write if the current version is 3
vault kv put -cas=3 secret/myapp/config db_password=new-value
```

If the current version is not 3 (e.g., another process already wrote version 4), the command fails:

```
Error writing data to secret/data/myapp/config: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/secret/data/myapp/config
Code: 400. Errors:

* check-and-set parameter did not match the current version
```

### Enabling CAS Requirement

You can require CAS for all writes to a key:

```bash
vault kv metadata put -cas-required=true secret/myapp/config
```

After this, every `vault kv put` and `vault kv patch` to this key must include `-cas=<current-version>` or it will be rejected:

```bash
# This will now fail without -cas
vault kv put secret/myapp/config db_password=value
# Error: check-and-set parameter required for this call

# This will succeed (assuming current version is 4)
vault kv put -cas=4 secret/myapp/config db_password=value
```

### CAS for the Initial Write

When a key does not yet exist, use `-cas=0`:

```bash
# Only succeed if the key does not exist (version 0 means "key doesn't exist")
vault kv put -cas=0 secret/newapp/config api_key=initialvalue
```

If the key already exists, the write fails, preventing accidental overwrites of pre-existing secrets.

## Deleting All Versions: `vault kv metadata delete`

To remove a key entirely — all versions and all metadata — use the metadata delete command:

```bash
vault kv metadata delete secret/myapp/config
```

This is a **permanent, non-recoverable operation**. All versions (including any that were merely soft-deleted) are destroyed, and the key is removed from the list. Use this when decommissioning an application or cleaning up test data.

After this command, `vault kv get secret/myapp/config` and `vault kv metadata get secret/myapp/config` both return errors as if the key never existed.

### Summary of Deletion Commands

```
vault kv delete secret/myapp/config              --> soft-delete current version
vault kv delete -versions=1,2 secret/myapp/config --> soft-delete specific versions
vault kv undelete -versions=3 secret/myapp/config --> restore soft-deleted version
vault kv destroy -versions=1 secret/myapp/config  --> permanently destroy version data
vault kv metadata delete secret/myapp/config       --> delete key + ALL versions permanently
```

## Common Pitfalls

- **Confusing `vault kv delete` and `vault kv destroy`** — `delete` is reversible; `destroy` is not. Always use `delete` unless you specifically need to permanently erase data. Audit which command is used in automation scripts before running them against production.
- **Setting `max_versions` too low on active secrets** — if you set `max_versions=1`, there is never a previous version to roll back to after an accidental overwrite. A minimum of 5–10 is reasonable for most secrets.
- **Using CAS without reading the current version first** — `vault kv put -cas=3 ...` will fail if the actual version is 4. Your workflow must read the metadata first (`vault kv metadata get`) to discover the current version, then write with `-cas=<current-version>`.
- **Running `vault kv metadata delete` thinking it is a soft-delete** — the name suggests metadata, but this command deletes everything: all version data and all metadata. There is no recovery. If you want to soft-delete the current version, use `vault kv delete` without the `metadata` subcommand.
- **Neglecting to set `max_versions` on high-churn secrets** — if a secret is rotated daily, Vault will retain every version indefinitely, consuming storage. Always configure `max_versions` for secrets that change frequently.

## Summary

- Every `vault kv put` or `vault kv patch` in KV v2 creates a new immutable version; old versions are retained until deleted or destroyed.
- Read a specific version with `vault kv get -version=<n>`; use `?version=<n>` in the HTTP API.
- `vault kv metadata get` shows all versions and their status (deletion time, destroyed flag).
- `vault kv delete` soft-deletes a version — data is retained and restorable with `vault kv undelete`.
- `vault kv destroy` permanently erases version data — use for compliance requirements.
- `vault kv metadata put` controls `max_versions` (cap version history), `delete_version_after` (auto-expire), and `custom_metadata` (arbitrary labels).
- Check-and-set (`-cas=<version>`) makes a write conditional on the current version number, preventing race conditions; require CAS for a key with `vault kv metadata put -cas-required=true`.
- `vault kv metadata delete` removes the key and ALL versions permanently and non-recoverably.
