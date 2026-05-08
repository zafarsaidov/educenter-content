# Kubernetes and LDAP Authentication

Kubernetes auth lets every pod authenticate to Vault using the service account JWT that Kubernetes already injects into the pod automatically — no passwords, no manually distributed secrets. LDAP auth connects Vault to an existing corporate directory so human operators log in with the same credentials they use everywhere else.

## Kubernetes Auth

Every Kubernetes pod receives a service account JWT at `/var/run/secrets/kubernetes.io/serviceaccount/token`. This token is issued and signed by the Kubernetes API server. Vault can verify that signature by calling the Kubernetes API, which means **the pod already has a credential it can use with Vault without any additional setup**.

```
+-------------------+      +-------------------+      +-------------------+
|   Pod             |      |   Vault Server    |      |   Kubernetes API  |
|                   |      |                   |      |                   |
|  SA JWT at        |      |                   |      |                   |
|  /var/run/...     |      |                   |      |                   |
+--------+----------+      +--------+----------+      +--------+----------+
         |                          |                           |
         | 1. POST auth/kubernetes/ |                           |
         |    login                 |                           |
         |   (role + jwt)           |                           |
         |------------------------->|                           |
         |                          | 2. TokenReview API call   |
         |                          |--------------------------->|
         |                          |<---------------------------|
         |                          |   valid / service account  |
         |                          |   name / namespace         |
         |                          |                           |
         |                          | 3. Check role binding     |
         |                          |   (SA name + namespace    |
         |                          |    match role?)           |
         |<-------------------------|                           |
         |   Vault token            |                           |
```

### Enabling Kubernetes Auth

```bash
vault auth enable kubernetes
```

### Configuring Kubernetes Auth

Vault needs to know the address of the Kubernetes API and how to verify JWT signatures. Run these commands from a pod inside the cluster, or supply the CA certificate manually:

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

| Parameter | Description |
|-----------|-------------|
| `kubernetes_host` | URL of the Kubernetes API server |
| `kubernetes_ca_cert` | CA certificate used to verify the API server's TLS cert |
| `token_reviewer_jwt` | A service account JWT with `tokenreviews` RBAC permission; Vault uses this to call the TokenReview API |

When running Vault inside the same cluster as the workloads, the in-cluster service account discovery often eliminates the need to set `token_reviewer_jwt` explicitly.

### Creating a Role

A Kubernetes auth role binds a Vault policy to a Kubernetes service account and namespace. A pod can only log in if its service account and namespace match the role.

```bash
vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=production \
  policies=my-app-policy \
  ttl=1h
```

To allow a service account from multiple namespaces, pass a comma-separated list:

```bash
vault write auth/kubernetes/role/my-app \
  bound_service_account_names=my-app-sa \
  bound_service_account_namespaces=production,staging \
  policies=my-app-policy \
  ttl=1h
```

Use `*` to allow any service account name or any namespace — but restrict this to development environments only.

### Logging In from a Pod

In a running pod, authenticate using the injected service account token:

```bash
vault write auth/kubernetes/login \
  role=my-app \
  jwt=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
```

On success:

```
Key                                 Value
---                                 -----
token                               hvs.CAESIBb...
token_accessor                      abc123
token_duration                      1h
token_renewable                     true
token_policies                      ["default" "my-app-policy"]
auth.metadata.role                  my-app
auth.metadata.service_account_name  my-app-sa
auth.metadata.service_account_namespace  production
```

In practice, you rarely call this endpoint manually. Vault Agent (covered in Section 6) handles authentication automatically and writes the resulting token to a file that your application reads.

### Required RBAC for Vault's Token Reviewer

Vault needs a service account with permission to call the `tokenreviews` API. Create a `ClusterRole` and bind it:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-token-reviewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault
    namespace: vault
```

The `system:auth-delegator` built-in role grants exactly the permissions Vault needs.

## LDAP Auth

LDAP auth lets users authenticate to Vault using their existing corporate directory credentials (Active Directory, OpenLDAP, FreeIPA, etc.). This is ideal for human operators — they use the same username and password they use for email and VPN, and Vault maps their directory group membership to Vault policies automatically.

```
+-------------------+      +-------------------+      +-------------------+
|   Operator        |      |   Vault Server    |      |   LDAP Directory  |
|   (vault login)   |      |                   |      |   (AD/OpenLDAP)   |
+--------+----------+      +--------+----------+      +--------+----------+
         |                          |                           |
         | vault login -method=ldap |                           |
         | username=john            |                           |
         | (prompts for password)   |                           |
         |------------------------->|                           |
         |                          | 1. Bind with binddn       |
         |                          |   credentials             |
         |                          |-------------------------->|
         |                          | 2. Search for user DN     |
         |                          |-------------------------->|
         |                          | 3. Bind as user to        |
         |                          |   verify password         |
         |                          |-------------------------->|
         |                          | 4. Fetch group membership |
         |                          |-------------------------->|
         |                          |<--------------------------|
         |                          |   groups: [engineers,     |
         |                          |            devops]        |
         |<-------------------------|                           |
         |   Vault token with       |                           |
         |   mapped policies        |                           |
```

### Enabling LDAP Auth

```bash
vault auth enable ldap
```

### Configuring LDAP Auth

```bash
vault write auth/ldap/config \
  url="ldap://ldap.example.com" \
  userdn="ou=Users,dc=example,dc=com" \
  groupdn="ou=Groups,dc=example,dc=com" \
  groupfilter="(&(objectClass=groupOfNames)(member={{.UserDN}}))" \
  groupattr="cn" \
  binddn="cn=vault-service,ou=ServiceAccounts,dc=example,dc=com" \
  bindpass="vault-service-password" \
  userattr="uid" \
  insecure_tls=false \
  starttls=true
```

Key parameters:

| Parameter | Description |
|-----------|-------------|
| `url` | LDAP server URL; use `ldaps://` for TLS, `ldap://` with `starttls=true` for STARTTLS |
| `userdn` | Base DN for user searches |
| `groupdn` | Base DN for group searches |
| `binddn` | DN of the service account Vault uses to search the directory |
| `bindpass` | Password for the bind service account |
| `userattr` | Attribute that stores the username (commonly `uid` or `sAMAccountName` for AD) |
| `groupattr` | Attribute on a group entry that holds the group's name |

For Active Directory, replace `userattr="uid"` with `userattr="sAMAccountName"` and adjust `userdn`/`groupdn` to match your AD structure.

### Mapping LDAP Groups to Vault Policies

After Vault fetches a user's group membership, it looks up each group name in `auth/ldap/groups/` to find which Vault policies to assign.

```bash
# Map the "engineers" LDAP group to the dev-policy Vault policy
vault write auth/ldap/groups/engineers policies=dev-policy

# Map the "devops" LDAP group to both dev-policy and ops-policy
vault write auth/ldap/groups/devops policies=dev-policy,ops-policy

# Map a specific user (overrides group mapping for that user)
vault write auth/ldap/users/alice policies=admin-policy
```

Group-to-policy mappings are evaluated at login time. Changing a mapping takes effect immediately on the next login — you do not need to revoke existing tokens.

### Logging In with LDAP

```bash
vault login -method=ldap username=john
# Vault prompts for the password interactively
```

To supply the password non-interactively (e.g., in a script):

```bash
vault login -method=ldap username=john password=mypassword
```

Interactive mode is strongly preferred because it avoids the password appearing in shell history or process listings.

## Common Pitfalls

- **Kubernetes: wrong `kubernetes_host`** — if Vault is running outside the cluster, `https://kubernetes.default.svc` will not resolve. Use the external API server endpoint (e.g., `https://10.0.0.1:6443`) and provide the appropriate CA certificate.
- **Kubernetes: pod service account has no matching role** — if the service account name or namespace does not match any `bound_service_account_names` / `bound_service_account_namespaces` in a role, the login will fail with `permission denied`. Double-check the actual service account name with `kubectl get pods <pod-name> -o jsonpath='{.spec.serviceAccountName}'`.
- **Kubernetes: expired projected service account tokens** — Kubernetes rotates projected SA tokens automatically. Do not cache the token indefinitely; re-read it from the filesystem before each Vault login.
- **LDAP: using `insecure_tls=true` in production** — this skips certificate verification and exposes credentials to interception. Always configure a valid CA certificate (`certificate` parameter) or use `ldaps://` with a properly signed cert.
- **LDAP: `bindpass` stored in plain text** — the bind password is stored in Vault's storage backend. Rotate it regularly and use a dedicated, low-privilege service account for the bind DN.
- **LDAP: group filter does not match AD structure** — OpenLDAP and Active Directory use different group object classes and member attributes. Test your `groupfilter` with an `ldapsearch` command before configuring Vault.

## Summary

- Kubernetes auth validates a pod's service account JWT by calling the Kubernetes TokenReview API — pods authenticate without any injected passwords.
- Enable with `vault auth enable kubernetes`; configure with the API server URL and CA cert; create roles that bind policies to service account + namespace pairs.
- Pods log in at `auth/kubernetes/login` with their role name and the contents of `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- LDAP auth proxies corporate directory credentials to Vault; users log in with their existing username and password.
- Enable with `vault auth enable ldap`; configure the server URL, bind credentials, and user/group base DNs; map LDAP group names to Vault policies at `auth/ldap/groups/<name>`.
- `vault login -method=ldap username=<user>` prompts for a password and returns a token with the policies mapped from the user's directory groups.
