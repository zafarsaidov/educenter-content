# JWT/OIDC and Userpass Authentication

JWT auth lets any system that issues standard JSON Web Tokens authenticate with Vault without pre-sharing a password — Vault simply verifies the JWT signature using the issuer's public key. This makes it the natural choice for CI/CD pipelines (GitLab, GitHub Actions) and identity providers that speak OIDC. Userpass provides a simpler fallback for small teams where a full LDAP or OIDC setup is not practical.

## JWT Auth

The JWT auth method accepts a signed JWT and verifies it against a configured JWKS endpoint or a static public key. Vault then maps claims inside the token (project ID, branch, namespace, etc.) to Vault policies.

```
+------------------+     +------------------+     +------------------+
|  CI Job / System |     |  Vault Server    |     |  JWKS Endpoint   |
|                  |     |                  |     |  (GitLab / IdP)  |
+--------+---------+     +--------+---------+     +--------+---------+
         |                        |                        |
         | 1. POST auth/jwt/login |                        |
         |   (role + jwt)         |                        |
         |----------------------->|                        |
         |                        | 2. Fetch public key    |
         |                        | from JWKS endpoint     |
         |                        |----------------------->|
         |                        |<-----------------------|
         |                        |                        |
         |                        | 3. Verify signature    |
         |                        | 4. Check bound_claims  |
         |<-----------------------|                        |
         |   Vault token          |                        |
```

### Enabling JWT Auth

```bash
vault auth enable jwt
```

### Integrating with GitLab CI

GitLab issues a JWT to every CI job via the `CI_JOB_JWT_V2` variable. The JWT contains claims that identify the project, branch, pipeline, and environment. Vault can verify this JWT against GitLab's public JWKS endpoint without any pre-shared secrets.

**Step 1: Configure the JWT auth method with GitLab's JWKS endpoint.**

```bash
vault write auth/jwt/config \
  jwks_url="https://gitlab.com/-/jwks" \
  bound_issuer="https://gitlab.com"
```

**Step 2: Create a role that restricts login to a specific project and branch.**

```bash
vault write auth/jwt/role/gitlab-ci \
  role_type="jwt" \
  bound_audiences="vault" \
  bound_claims_type="glob" \
  bound_claims='{"project_id":"123","ref":"main","ref_type":"branch"}' \
  user_claim="sub" \
  policies="ci-policy" \
  ttl=15m
```

| Parameter | Description |
|-----------|-------------|
| `bound_audiences` | The `aud` claim value in the JWT must match this string |
| `bound_claims` | A map of claim names to required values; all claims must match |
| `user_claim` | Which JWT claim to use as the "username" for audit logs |
| `ttl` | How long the resulting Vault token is valid; keep it short for CI jobs |

The `bound_claims` filter is critical for security. Without it, any GitLab job — from any project on gitlab.com — could log in with this role.

**Step 3: Log in from the CI job.**

```yaml
# .gitlab-ci.yml
get-secret:
  script:
    - >
      VAULT_TOKEN=$(vault write -field=token auth/jwt/login
      role=gitlab-ci
      jwt=$CI_JOB_JWT_V2)
    - export VAULT_TOKEN
    - vault kv get -field=password secret/ci/myapp
```

### Integrating with GitHub Actions

GitHub Actions provides an OIDC token for each workflow run via the `actions/github-oidc-token` mechanism. The pattern is identical to GitLab:

```bash
# Configure with GitHub's JWKS endpoint
vault write auth/jwt/config \
  jwks_url="https://token.actions.githubusercontent.com/.well-known/jwks" \
  bound_issuer="https://token.actions.githubusercontent.com"

# Create a role bound to a specific repo and branch
vault write auth/jwt/role/github-actions \
  role_type="jwt" \
  bound_audiences="vault" \
  bound_claims='{"repository":"myorg/myrepo","ref":"refs/heads/main"}' \
  user_claim="sub" \
  policies="ci-policy" \
  ttl=15m
```

In the GitHub Actions workflow, exchange the OIDC token:

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    permissions:
      id-token: write   # required to request the OIDC token
    steps:
      - name: Authenticate to Vault
        run: |
          OIDC_TOKEN=$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
            "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=vault" | jq -r '.value')
          VAULT_TOKEN=$(vault write -field=token auth/jwt/login \
            role=github-actions \
            jwt=$OIDC_TOKEN)
          echo "VAULT_TOKEN=$VAULT_TOKEN" >> $GITHUB_ENV
```

## Userpass Auth

Userpass is the simplest auth method — a plain username and password stored inside Vault itself. It is suitable for small teams or development environments where setting up LDAP or an OIDC provider is impractical. For larger teams or production environments, prefer LDAP (which uses existing credentials) or OIDC (which uses an identity provider with proper MFA).

### Enabling Userpass Auth

```bash
vault auth enable userpass
```

### Creating Users

```bash
# Create a user with a policy
vault write auth/userpass/users/alice \
  password=s3cret \
  policies=admin-policy

# Create a user with multiple policies
vault write auth/userpass/users/bob \
  password=another-secret \
  policies=read-secrets,list-secrets
```

Passwords are stored as bcrypt hashes inside Vault's storage backend.

### Updating a Password

```bash
vault write auth/userpass/users/alice password=new-password
```

### Logging In

```bash
vault login -method=userpass username=alice
# Vault prompts for the password
```

Non-interactive login (for scripts — avoid in production):

```bash
vault login -method=userpass username=alice password=s3cret
```

## Choosing the Right Auth Method

Use this table as a starting point when designing your Vault integration:

| Who is authenticating? | Environment | Recommended method |
|------------------------|-------------|-------------------|
| Human operator | Has corporate LDAP/AD | LDAP auth |
| Human operator | Has OIDC provider (Okta, Google, etc.) | OIDC auth |
| Human operator | Small team, no LDAP/OIDC | Userpass |
| Automated workload | Running in Kubernetes | Kubernetes auth |
| Automated workload | Not in Kubernetes | AppRole |
| CI/CD pipeline | GitLab CI | JWT auth (CI_JOB_JWT_V2) |
| CI/CD pipeline | GitHub Actions | JWT auth (OIDC token) |
| CI/CD pipeline | Other (Jenkins, etc.) | AppRole |

A Vault deployment typically enables several auth methods simultaneously — operators log in via LDAP while applications use Kubernetes or AppRole auth.

## Common Pitfalls

- **No `bound_claims` on a public OIDC provider** — configuring JWT auth with `jwks_url=https://gitlab.com/-/jwks` and no `bound_claims` means any GitLab.com user or project can log in with that role. Always scope `bound_claims` to at least the project ID.
- **Long TTLs on CI tokens** — CI jobs are short-lived. Issue tokens with TTLs of 15–30 minutes. A token that outlives the job is an unnecessary attack surface.
- **Using Userpass in production when LDAP is available** — Userpass credentials are separate from corporate identity, which means separate onboarding, offboarding, and password rotation processes. If you already manage users in LDAP, use LDAP auth so leavers' access is revoked automatically.
- **`bound_audiences` mismatch** — the `aud` claim in the JWT must exactly match the `bound_audiences` value in the Vault role. In GitLab, you control the audience via the `id_tokens` block in `.gitlab-ci.yml`; if it is missing, the default audience may not match `vault`.
- **Forgetting `id-token: write` permission in GitHub Actions** — without this workflow permission, the `ACTIONS_ID_TOKEN_REQUEST_TOKEN` environment variable is not injected and the OIDC token cannot be requested.

## Summary

- JWT auth verifies a JSON Web Token signature using a JWKS endpoint or static public key — no pre-shared secret between Vault and the issuer.
- Enable with `vault auth enable jwt`; configure with `jwks_url` and `bound_issuer`; create roles with `bound_claims` to scope access to specific projects, branches, or repositories.
- GitLab CI provides `CI_JOB_JWT_V2`; GitHub Actions provides an OIDC token — both can authenticate directly to Vault without storing a Vault token in CI variables.
- Userpass stores credentials natively in Vault; suitable for small teams or development but should be replaced with LDAP or OIDC in larger or production environments.
- Use the decision table to pick the right auth method: LDAP for humans with a directory, Kubernetes auth for pods, AppRole for other machines, JWT/OIDC for CI/CD pipelines.
- Never configure JWT roles on public OIDC providers without `bound_claims` — this would allow any identity on that provider to authenticate.
