# Vault Agent Basics

Vault Agent is a client-side daemon that runs alongside your application, handles authentication to Vault, renews tokens automatically, and renders secrets into local files — so the application itself never needs Vault credentials or the Vault SDK. This pattern decouples secret management from application code and works with any programming language.

## What Vault Agent Does

Without Vault Agent, every application must:

1. Store a Vault token or AppRole credentials somewhere accessible
2. Call the Vault API at startup to fetch secrets
3. Implement token renewal logic to avoid token expiry
4. Re-fetch secrets when they are rotated

Vault Agent handles all of this on behalf of the application. The app simply reads a local file that Agent keeps up to date.

```
+---------------+        auth + renew        +-------------+
|  Vault Server | <------------------------- | Vault Agent |
+---------------+                            +-------------+
                                                    |
                                            renders templates
                                                    |
                                                    v
                                       /etc/myapp/config.ini
                                                    ^
                                                    |
                                                 reads
                                                    |
                                             +-----------+
                                             | Application|
                                             +-----------+
```

The Vault server is only ever contacted by Vault Agent. The application is completely isolated from the Vault API.

## Agent Config File Structure

Vault Agent is configured with an HCL file. The file has four main blocks:

| Block       | Purpose                                                  |
|-------------|----------------------------------------------------------|
| `vault`     | Address of the Vault server                              |
| `auto_auth` | How Agent authenticates to Vault and where to store the token |
| `cache`     | Enable local caching of tokens and secrets               |
| `template`  | Which secrets to fetch and how to render them into files |

```hcl
# agent.hcl

vault {
  address = "https://vault.example.com:8200"
}

auto_auth {
  method {
    type = "approle"

    config = {
      role_id_file_path   = "/etc/vault/role-id"
      secret_id_file_path = "/etc/vault/secret-id"
    }
  }

  sink {
    type = "file"

    config = {
      path = "/tmp/vault-token"
    }
  }
}

cache {
  use_auto_auth_token = true
}

template {
  source      = "/etc/vault/templates/config.ctmpl"
  destination = "/etc/myapp/config.ini"
  perms       = "0640"
}
```

## The `auto_auth` Block

`auto_auth` is the core of Agent's value. It tells Agent which auth method to use and where to persist the resulting token.

### `method` Sub-block

The `method` sub-block defines how Agent authenticates. With AppRole:

```hcl
method {
  type = "approle"

  config = {
    role_id_file_path            = "/etc/vault/role-id"
    secret_id_file_path          = "/etc/vault/secret-id"
    remove_secret_id_file_after_reading = false
  }
}
```

- `role_id_file_path` — path to a file containing the AppRole `role_id`
- `secret_id_file_path` — path to a file containing the AppRole `secret_id`
- `remove_secret_id_file_after_reading` — set to `true` in production to delete the `secret_id` file after first use (response-wrapped secret IDs are single-use by design)

### `sink` Sub-block

The `sink` sub-block tells Agent where to write the token it receives after authenticating. This lets other tools or scripts on the same host use the token if needed.

```hcl
sink {
  type = "file"

  config = {
    path = "/tmp/vault-token"
    mode = 0640
  }
}
```

Agent automatically renews the token before it expires. If renewal fails (for example, the token's max TTL is reached), Agent re-authenticates and writes a fresh token to the sink path.

## The `cache` Block

The `cache` block enables Agent to cache tokens and responses locally. This reduces the number of round-trips to the Vault server, which is especially useful when many pods or processes share one Agent instance.

```hcl
cache {
  use_auto_auth_token = true
}
```

When `use_auto_auth_token = true`, Agent uses the token it acquired via `auto_auth` for all proxied requests. Applications can point their Vault client at `http://127.0.0.1:8200` (Agent's listener) and Agent will inject the token automatically.

To enable the listener for proxied requests:

```hcl
listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = true
}
```

## Full Working Config Example

This config uses AppRole auto-auth and renders a database config file:

```hcl
# /etc/vault/agent.hcl

vault {
  address = "https://vault.example.com:8200"
}

auto_auth {
  method {
    type = "approle"

    config = {
      role_id_file_path   = "/etc/vault/role-id"
      secret_id_file_path = "/etc/vault/secret-id"
    }
  }

  sink {
    type = "file"

    config = {
      path = "/tmp/vault-token"
    }
  }
}

cache {
  use_auto_auth_token = true
}

listener "tcp" {
  address     = "127.0.0.1:8200"
  tls_disable = true
}

template {
  source      = "/etc/vault/templates/db.ctmpl"
  destination = "/etc/myapp/database.conf"
  perms       = "0640"

  error_on_missing_key = true
}
```

## Running Vault Agent

Install the `vault` binary (Agent is built into the same binary as the server) and run:

```bash
vault agent -config=/etc/vault/agent.hcl
```

Run Agent as a systemd service so it starts on boot and restarts on failure:

```ini
# /etc/systemd/system/vault-agent.service
[Unit]
Description=Vault Agent
After=network.target

[Service]
User=vault-agent
ExecStart=/usr/local/bin/vault agent -config=/etc/vault/agent.hcl
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl enable vault-agent
systemctl start vault-agent
systemctl status vault-agent
```

Agent logs to stdout by default. Add `log_level = "info"` to the config file to control verbosity.

## Preparing AppRole Credentials

Before starting Agent, write the `role_id` and `secret_id` to the paths the config expects:

```bash
# Fetch the role_id from Vault (done once during provisioning)
vault read -field=role_id auth/approle/role/myapp/role-id \
  > /etc/vault/role-id

# Generate a secret_id
vault write -field=secret_id -f auth/approle/role/myapp/secret-id \
  > /etc/vault/secret-id

# Restrict permissions — only the vault-agent user should read these
chmod 640 /etc/vault/role-id /etc/vault/secret-id
chown vault-agent:vault-agent /etc/vault/role-id /etc/vault/secret-id
```

## Common Pitfalls

- **Leaving `secret_id` on disk in production** — use response-wrapped secret IDs and set `remove_secret_id_file_after_reading = true`. A `secret_id` sitting in a file is a credential that any process with read access can steal.
- **Running Agent as root** — create a dedicated low-privilege user (`vault-agent`) and run the service as that user. Agent only needs to read its config and credential files and write to the sink and destination paths.
- **Not setting `perms` on the sink or template destination** — the default permissions may be too broad. Always set `mode` on the sink and `perms` on template destinations explicitly.
- **Pointing the app at Vault directly instead of Agent** — if the app bypasses Agent and calls Vault directly with the token from the sink file, it will break when Agent re-authenticates and writes a new token. Use the Agent listener (`127.0.0.1:8200`) instead.
- **Forgetting that Agent needs network access to Vault at startup** — if Vault is unreachable when Agent starts, it will fail to authenticate and no templates will be rendered. The app may start with a stale or empty config file.

## Summary

- Vault Agent is a daemon that authenticates to Vault on behalf of your application and keeps tokens renewed automatically
- The application reads secrets from a local file written by Agent — no Vault SDK or API calls needed in application code
- Agent is configured with an HCL file containing `vault`, `auto_auth`, `cache`, and `template` blocks
- The `auto_auth` block specifies the auth method (e.g. AppRole) and a `sink` where the acquired token is written
- The `cache` block reduces load on the Vault server by caching tokens and lease responses locally
- The `listener` block lets applications proxy their Vault API calls through Agent, which injects the token automatically
- Run Agent as a dedicated low-privilege user with a systemd service for reliability
- Protect AppRole credentials with strict file permissions and consider single-use response-wrapped secret IDs
