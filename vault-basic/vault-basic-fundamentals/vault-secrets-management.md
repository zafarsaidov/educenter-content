# Secrets Management and Why It Matters

Every application that talks to a database, calls an external API, serves HTTPS traffic, or accesses a cloud provider needs credentials. Secrets management is the discipline of storing, distributing, rotating, and auditing those credentials so they never end up somewhere they should not be. This lesson explains what secrets are, how they leak in practice, and why Vault's approach solves problems that simpler tools cannot.

## What Is a Secret

A secret is any piece of data that grants access to a system or resource. If it leaks, an attacker can impersonate your application.

Common secrets in a typical backend service:

| Secret type | Example |
|---|---|
| Database password | `postgres://app:s3cr3t@db.internal:5432/prod` |
| API key | `<stripe-secret-key>` |
| TLS private key | `-----BEGIN EC PRIVATE KEY-----...` |
| SSH private key | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| Cloud provider credentials | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` |
| Encryption key | `32-byte AES key used to encrypt user data` |
| OAuth client secret | `GITHUB_CLIENT_SECRET` used by your auth service |

Secrets are required for your code to function — they are not optional. The only question is how you handle them.

## The Problem with Hardcoded Credentials

The most naive approach is to write credentials directly into source code:

```python
# application.py  (DO NOT DO THIS)
DB_PASSWORD = "hunter2"
STRIPE_KEY   = "<your-stripe-key>"
```

This is catastrophically bad for several reasons:

**Committed to Git.** Once a secret is committed, it lives in the repository's history forever. `git log -p`, `git blame`, and every clone of the repo expose it. Public repositories have bots that scan for leaked credentials within seconds of a push.

**Shared across environments.** The same value ends up in development, staging, and production — so a compromise of any one environment instantly compromises all three.

**No audit trail.** When the secret is embedded in code, there is no record of which process used it, when, or from where. Forensic investigation after a breach is nearly impossible.

**Manual rotation is painful.** Changing a hardcoded secret requires a code change, a new build, and a redeployment. In practice this means credentials are never rotated, giving attackers unlimited time to exploit a leaked key.

**Blast radius on leak.** A single leaked credential that is shared everywhere requires rotating every service simultaneously — a high-pressure, error-prone incident response.

## The Problem with Environment Variables

Environment variables are a step up from hardcoded credentials, but they introduce a different set of problems:

```bash
# .env file on developer laptop
DB_PASSWORD=hunter2
STRIPE_KEY=<your-stripe-key>
```

```python
import os
db_password = os.environ["DB_PASSWORD"]
```

**Still plaintext, still passed around.** The `.env` file ends up in Slack messages, email threads, shared drives, and developer laptops. It is copied into CI/CD pipeline settings as plaintext.

**Visible in process listings.** On Linux, environment variables of a running process are readable by any user with access to `/proc/<pid>/environ`, or by tools like `ps -e --environment`.

```bash
# Any user on the same host can do this:
cat /proc/$(pgrep my-app)/environ | tr '\0' '\n' | grep PASSWORD
```

**Logged accidentally.** Frameworks, crash reporters, and debug handlers routinely dump environment variables in logs. A misconfigured logging level can expose all secrets in plain text to your log aggregator.

**No rotation, no expiry.** An env var does not expire. If it leaks, it is valid indefinitely until you manually rotate it and redeploy every service that uses it.

**No access control.** Anyone who can access the server, the CI/CD configuration, or the Kubernetes manifest can read the secret. There is no concept of a role that can read a secret but not export it.

## Secrets Sprawl

In a real organisation, the same credential quietly multiplies across systems:

```
Production DB password "hunter2" exists in:

  ├── .env file on 3 developer laptops
  ├── CI/CD pipeline variable in GitLab UI (plaintext export)
  ├── Kubernetes Secret (base64-encoded, not encrypted at rest by default)
  ├── Helm chart values.yaml committed to a private repo
  ├── Ansible vars/main.yml (possibly encrypted with ansible-vault, possibly not)
  ├── Staging environment (same password as production)
  ├── A Slack message from 6 months ago: "here's the DB pass for the new service"
  └── An ex-employee's laptop that was never wiped
```

This is secrets sprawl. It is not a hypothetical — it is the default outcome when there is no central secrets management system. When a secret leaks (and it will), you cannot revoke it in one place because it is not in one place.

## Vault's Core Value Proposition

Vault is a secrets management system designed to be the single source of truth for all credentials in your organisation. Its core properties:

**Centralised storage.** All secrets live in one place. Applications never store credentials locally.

**Every access is authenticated.** No request reaches Vault without proving identity first — via a token, an AppRole credential, a Kubernetes service account JWT, an OIDC token, or an LDAP password.

**Every access is authorised.** After authentication, Vault checks a policy: does this identity have permission to read this path? If not, it returns a 403. The policy is explicit and auditable.

**Every access is logged.** Vault's audit log records every request and response — who asked, what path, what time, what IP, and whether it was allowed or denied. This is non-negotiable for compliance and incident response.

**Dynamic secrets.** Instead of storing a long-lived password, Vault can generate a credential on demand with a short TTL. When the TTL expires, Vault revokes it automatically. The leaked credential is already dead.

```
Traditional approach:
  App reads DB_PASSWORD="hunter2" from env → password valid forever → never rotated

Vault dynamic secrets:
  App requests DB creds from Vault → Vault creates a new user in Postgres → returns
  { "username": "v-app-xK3p", "password": "A1b2C3d4", "lease_duration": "1h" }
  → after 1h Vault drops the user → leaked creds expire before they can be used
```

**Encryption as a service.** Vault's Transit secrets engine lets applications encrypt and decrypt data without ever managing encryption keys. The key never leaves Vault.

## Vault vs Alternatives

| Feature | HashiCorp Vault | AWS Secrets Manager | Azure Key Vault | Doppler |
|---|---|---|---|---|
| Self-hosted | Yes | No | No | No |
| Cloud-native | Optional | AWS only | Azure only | SaaS |
| Open source | Yes (BSL) | No | No | No |
| Dynamic secrets | Yes (DB, PKI, SSH, cloud) | Limited (RDS rotation) | No | No |
| Encryption as a service | Yes (Transit engine) | No | Yes (Key Vault keys) | No |
| Multi-cloud / multi-env | Yes | No | No | Yes |
| Fine-grained policies | HCL policies, namespaces | IAM policies | RBAC | Project-level |
| Audit log | Yes (file, syslog, socket) | CloudTrail | Activity log | Yes |
| Kubernetes integration | Agent Injector, ESO, VSO | External Secrets Operator | External Secrets Operator | External Secrets Operator |

AWS Secrets Manager and Azure Key Vault are excellent choices if you are entirely within one cloud provider and want zero operational overhead. Vault is the right choice when you need to work across multiple clouds, need dynamic secrets, need self-hosting for compliance reasons, or want a unified secrets layer for mixed environments (cloud, on-prem, containers, bare metal).

## Common Pitfalls

- **Treating base64 as encryption.** Kubernetes Secrets encode values with base64, which is not encryption. Anyone with `kubectl get secret` access can decode the value in one command (`echo "aHVudGVyMg==" | base64 -d`). Do not store sensitive credentials in Kubernetes Secrets without encrypting them at rest with a KMS provider or syncing them from Vault.

- **Rotating the secret but not the old one.** When you generate a new credential, the old one is still valid until you explicitly revoke it. Rotation without revocation only reduces the window of exposure — it does not eliminate it. Vault's lease system handles revocation automatically for dynamic secrets.

- **Assuming a private repo is safe.** Private repositories leak constantly — through collaborator access, deploy keys, export features, and data breaches at the hosting provider. Never commit secrets to any repository, public or private.

- **One Vault token shared across all services.** A single root token used by ten services means that if any one of them is compromised, the attacker has access to everything. Each service should authenticate with its own identity and receive a token scoped to only the paths it needs.

- **Not enabling audit logging on day one.** Audit logging is the first thing you should enable after initialising Vault, not an afterthought. Without it you cannot reconstruct what happened after a security incident.

## Summary

- A secret is any credential that grants access to a resource — DB passwords, API keys, TLS private keys, SSH keys, and cloud provider credentials.
- Hardcoded credentials end up in git history, are shared across environments, provide no audit trail, and are never rotated.
- Environment variables are still plaintext — they appear in process listings, get logged by accident, and have no expiry or access control.
- Secrets sprawl means the same credential silently copies itself to CI/CD variables, `.env` files, Kubernetes Secrets, chat messages, and developer laptops.
- Vault provides a single source of truth where every access is authenticated, authorised, and logged.
- Dynamic secrets are credentials generated on demand with a short TTL — when the TTL expires, Vault revokes them automatically, eliminating the blast radius of a leak.
- Vault is the right choice for multi-cloud, on-prem, or compliance-constrained environments; AWS Secrets Manager and Azure Key Vault are simpler when you are entirely within one cloud provider.
