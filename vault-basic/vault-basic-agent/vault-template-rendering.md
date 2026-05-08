# Template Rendering with Vault Agent

Vault Agent uses consul-template syntax to read secrets from Vault and render them into files on disk, keeping those files up to date whenever secrets change. This means your application config files always contain current credentials without the application ever calling Vault directly.

## How Template Blocks Work

Each `template` block in the Agent config describes one output file. Agent fetches the referenced secrets, substitutes them into the template, and writes the result to the `destination` path. When Vault rotates the secret, Agent detects the change and re-renders the file automatically.

A `template` block has three ways to provide the template content:

| Field      | Purpose                                             |
|------------|-----------------------------------------------------|
| `source`   | Path to an external `.ctmpl` file on disk           |
| `contents` | Inline template string directly in the config file  |
| `destination` | Path where the rendered output is written        |

You can use either `source` or `contents` — not both in the same block.

## Template Syntax Basics

Templates use the consul-template language, which is Go's `text/template` with Vault-specific functions added.

Fetch a single value from a KV v2 secret:

```
{{ with secret "secret/data/myapp/config" }}
db_password = {{ .Data.data.db_password }}
{{ end }}
```

The `with secret` block fetches the secret at the given path. `.Data.data` is required for KV v2 because Vault wraps the values in a `data` envelope. For KV v1 the path is `.Data`.

Access nested fields:

```
{{ with secret "secret/data/myapp/config" }}
{{ .Data.data.db_host }}:{{ .Data.data.db_port }}
{{ end }}
```

## Full Template Example — Database Config

This template renders a complete database connection config for an application:

```
{{- with secret "secret/data/myapp/database" -}}
[database]
host     = {{ .Data.data.host }}
port     = {{ .Data.data.port }}
name     = {{ .Data.data.db_name }}
username = {{ .Data.data.username }}
password = {{ .Data.data.password }}
{{- end }}
```

Save this as `/etc/vault/templates/db.ctmpl`. The `-` after `{{` and before `}}` trims surrounding whitespace, which keeps the output file clean.

The corresponding `template` block in `agent.hcl`:

```hcl
template {
  source               = "/etc/vault/templates/db.ctmpl"
  destination          = "/etc/myapp/database.conf"
  perms                = "0640"
  error_on_missing_key = true
}
```

## Using `contents` for Inline Templates

For short templates, define the content directly in the Agent config instead of a separate file:

```hcl
template {
  contents    = "{{ with secret \"secret/data/myapp/config\" }}{{ .Data.data.api_key }}{{ end }}"
  destination = "/etc/myapp/api-key"
  perms       = "0600"
}
```

Inline templates are useful for writing a single secret value to a file (such as an API key or a password) without creating a separate `.ctmpl` file.

## Permissions and Ownership

Always set explicit permissions on rendered files. The `perms` field accepts an octal string:

```hcl
template {
  source      = "/etc/vault/templates/tls.ctmpl"
  destination = "/etc/myapp/tls.key"
  perms       = "0600"  # owner read/write only — private key
}
```

Run Vault Agent as the same user that owns the application process, or use a group that the application user belongs to and set `perms = "0640"`. This ensures the app can read the file but other users cannot.

## The `exec` Block — Reloading the App After Rendering

When secrets rotate, Agent re-renders the template but the running application still holds the old values in memory. Use the `exec` block to run a command after the template is updated:

```hcl
template {
  source      = "/etc/vault/templates/db.ctmpl"
  destination = "/etc/myapp/database.conf"
  perms       = "0640"

  exec {
    command = ["systemctl", "reload", "myapp"]
    timeout = "30s"
  }
}
```

The command runs every time the destination file changes. Keep the command fast — a `systemctl reload` sends `SIGHUP` to the process, which is much cheaper than a full restart.

For applications that do not support graceful reload, use a restart instead:

```hcl
exec {
  command = ["systemctl", "restart", "myapp"]
  timeout = "60s"
}
```

## `error_on_missing_key`

By default, consul-template renders an empty string if a key does not exist in a secret. This can silently produce a broken config file. Set `error_on_missing_key = true` to make Agent fail with an error instead:

```hcl
template {
  source               = "/etc/vault/templates/db.ctmpl"
  destination          = "/etc/myapp/database.conf"
  error_on_missing_key = true
}
```

With this flag set, if `secret/data/myapp/database` exists but is missing the `password` key, Agent logs an error and does not write the destination file. The previous rendered version (if any) is preserved.

## Watching for Secret Changes

Agent continuously watches the Vault paths referenced in all templates. When a secret is updated at Vault — for example, after a database credential rotation — Agent:

1. Detects the change via Vault's blocking query API
2. Re-fetches the updated secret
3. Re-renders any templates that reference that secret
4. Runs the `exec` command if one is defined

No manual intervention is required. The application picks up the new credentials on its next reload.

## Multiple Templates in One Agent Config

A single Agent process can manage multiple output files. Define one `template` block per output file:

```hcl
# Write the database config
template {
  source               = "/etc/vault/templates/db.ctmpl"
  destination          = "/etc/myapp/database.conf"
  perms                = "0640"
  error_on_missing_key = true

  exec {
    command = ["systemctl", "reload", "myapp"]
  }
}

# Write the TLS private key
template {
  contents    = "{{ with secret \"pki/issue/myapp\" }}{{ .Data.private_key }}{{ end }}"
  destination = "/etc/myapp/tls.key"
  perms       = "0600"
}

# Write the TLS certificate
template {
  contents    = "{{ with secret \"pki/issue/myapp\" }}{{ .Data.certificate }}{{ end }}"
  destination = "/etc/myapp/tls.crt"
  perms       = "0644"
}
```

Each template is rendered independently. Agent batches secret fetches where possible to minimize Vault API calls.

### Example: Rendering Both a Config File and an API Key

```hcl
template {
  source      = "/etc/vault/templates/app-config.ctmpl"
  destination = "/etc/myapp/config.ini"
  perms       = "0640"
}

template {
  contents    = "{{ with secret \"secret/data/myapp/tokens\" }}{{ .Data.data.stripe_api_key }}{{ end }}"
  destination = "/run/secrets/stripe-api-key"
  perms       = "0600"
}
```

The `/run/secrets/` directory (a tmpfs mount on Linux) is a good destination for sensitive single-value files because its contents do not persist across reboots.

## Common Pitfalls

- **Using `.Data` instead of `.Data.data` for KV v2** — KV v2 paths return a nested `data` object. Using `.Data.db_password` will produce an empty string (or error if `error_on_missing_key` is set). Always use `.Data.data.<key>` for KV v2.
- **Not setting `error_on_missing_key = true`** — without this flag, a typo in a key name produces an empty string in the rendered file with no warning. Enable this flag in all production templates.
- **Broad `perms` on sensitive files** — rendered files containing passwords or private keys should be `0600` (owner only) or at most `0640` (owner + group). Never use `0644` for secrets.
- **`exec` commands that take too long** — if the reload command times out, Agent logs an error but the rendered file still contains the new secret. Set a realistic `timeout` and prefer graceful reload (`SIGHUP`) over full restarts.
- **Assuming templates render synchronously at startup** — Agent renders templates as soon as it successfully authenticates and fetches secrets. If your application starts before Agent finishes rendering, the destination file may not exist yet. Use a startup dependency (e.g. a systemd `After=vault-agent.service` with a readiness check) to prevent the application from starting before its config is ready.
- **Modifying destination files manually** — Agent will overwrite the destination file on the next render cycle. Do not edit rendered files by hand; edit the template instead.

## Summary

- Agent uses consul-template syntax to fetch secrets from Vault and write them into local files
- Use `{{ with secret "path" }}{{ .Data.data.key }}{{ end }}` for KV v2 secrets; use `.Data.<key>` for KV v1
- Each `template` block specifies a `source` (`.ctmpl` file) or `contents` (inline), a `destination`, and optional `perms`
- Set `error_on_missing_key = true` to fail loudly instead of silently writing empty values
- Use the `exec` block to reload or restart the application after a template is re-rendered
- Agent watches Vault for secret changes and re-renders templates automatically — no cron jobs or manual rotation needed
- A single Agent process can manage multiple output files, each with its own `template` block and `exec` command
- Run Agent as the same user as the application and set strict file permissions on all rendered secret files
