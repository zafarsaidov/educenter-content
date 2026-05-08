# External Secrets Operator

The External Secrets Operator (ESO) is a Kubernetes operator that reads secrets from external stores — Vault, AWS Secrets Manager, GCP Secret Manager, and others — and creates native Kubernetes `Secret` objects. Applications read a standard Kubernetes `Secret` as they always have, with no Vault SDK, no sidecar, and no changes to application code.

## How ESO Works

ESO introduces two custom resources:

| Resource              | Purpose                                                      |
|-----------------------|--------------------------------------------------------------|
| `SecretStore`         | Defines how to connect to the external secret store (namespaced) |
| `ClusterSecretStore`  | Same as `SecretStore` but available cluster-wide             |
| `ExternalSecret`      | Declares which secrets to fetch and which Kubernetes `Secret` to create or update |

The flow is:

```
+---------+     reads ClusterSecretStore     +-----+     fetches     +---------+
|   ESO   | --------------------------------> | ESO | -------------> |  Vault  |
| operator|                                  |     |                 +---------+
+---------+                                  +-----+
                                                |
                                     creates/updates
                                                |
                                                v
                                    Kubernetes Secret "myapp-secret"
                                                ^
                                                |
                                             reads
                                                |
                                         +-----------+
                                         | Application|
                                         | Pod        |
                                         +-----------+
```

ESO polls Vault at the interval defined by `refreshInterval` and updates the Kubernetes `Secret` when values change.

## Installing ESO

Add the Helm repository and install the operator into its own namespace:

```bash
helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace
```

Verify the operator is running:

```bash
kubectl get pods -n external-secrets
```

Expected output:

```
NAME                                               READY   STATUS    RESTARTS   AGE
external-secrets-xxxxxxxxxx-xxxxx                  1/1     Running   0          2m
external-secrets-cert-controller-xxxxxxxxxx-xxxxx  1/1     Running   0          2m
external-secrets-webhook-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
```

## Creating a `ClusterSecretStore` for Vault

A `ClusterSecretStore` defines the connection to Vault and the credentials ESO uses to authenticate. Using AppRole auth:

```yaml
# cluster-secret-store.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"          # KV mount path
      version: "v2"           # KV version
      auth:
        appRole:
          path: "approle"     # AppRole mount path in Vault
          roleId: "db02de05-fa39-4855-059b-67221c5c2f63"
          secretRef:
            name: vault-approle-secret   # k8s Secret containing the secret_id
            key: secretId
            namespace: external-secrets
```

The `secretRef` points to a Kubernetes `Secret` that holds the AppRole `secret_id`. Create it once:

```bash
kubectl create secret generic vault-approle-secret \
  --namespace external-secrets \
  --from-literal=secretId="<your-secret-id>"
```

Apply the store:

```bash
kubectl apply -f cluster-secret-store.yaml
```

Check its status:

```bash
kubectl get clustersecretstore vault-backend
```

A healthy store shows `READY = True`.

## Creating an `ExternalSecret`

An `ExternalSecret` tells ESO which Vault path to read and which Kubernetes `Secret` to create:

```yaml
# external-secret-myapp.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-external-secret
  namespace: myapp
spec:
  refreshInterval: "1h"

  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore

  target:
    name: myapp-secret          # Name of the Kubernetes Secret to create
    creationPolicy: Owner       # ESO owns and manages this Secret

  data:
    - secretKey: db_password    # Key in the Kubernetes Secret
      remoteRef:
        key: secret/data/myapp/config   # Vault path
        property: db_password           # Field within the Vault secret

    - secretKey: api_key
      remoteRef:
        key: secret/data/myapp/config
        property: api_key
```

Apply it:

```bash
kubectl apply -f external-secret-myapp.yaml
```

ESO immediately fetches the secrets from Vault and creates the Kubernetes `Secret` named `myapp-secret` in the `myapp` namespace.

## `refreshInterval`

`refreshInterval` controls how often ESO polls Vault and updates the Kubernetes `Secret`. The value uses Go duration syntax:

```yaml
refreshInterval: "1h"    # re-sync every hour
refreshInterval: "15m"   # re-sync every 15 minutes
refreshInterval: "0"     # never re-sync after initial creation
```

Choose an interval that balances freshness against Vault API load. For database credentials that rotate daily, `1h` is a reasonable choice. For TLS certificates that rotate hourly, use `5m` or less.

## Verifying the Sync

Check the `ExternalSecret` status:

```bash
kubectl get externalsecret -n myapp
```

```
NAME                     STORE          REFRESH INTERVAL   STATUS   READY
myapp-external-secret    vault-backend  1h                 SecretSynced   True
```

Read the resulting Kubernetes `Secret`:

```bash
kubectl get secret myapp-secret -n myapp
```

Decode a specific value:

```bash
kubectl get secret myapp-secret -n myapp \
  -o jsonpath='{.data.db_password}' | base64 -d
```

The decoded value should match what is stored at `secret/data/myapp/config` in Vault.

## Using the Secret in a Pod

Once the Kubernetes `Secret` exists, reference it in a `Deployment` exactly as you would any other secret:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: db_password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: myapp-secret
                  key: api_key
```

The application reads `DB_PASSWORD` and `API_KEY` as ordinary environment variables. It has no knowledge of Vault.

## ESO vs Vault Agent Injector

Vault also ships its own Kubernetes integration called the Vault Agent Injector. The two tools solve the same problem differently:

| Feature                        | ESO                                              | Vault Agent Injector                          |
|-------------------------------|--------------------------------------------------|-----------------------------------------------|
| Creates a Kubernetes `Secret` | Yes — standard k8s `Secret` object              | No — secrets are written to files in the pod  |
| Requires a sidecar container  | No                                               | Yes — `vault-agent` runs alongside the app    |
| App changes required          | None — reads from env or volume as normal        | None — reads files from a shared volume       |
| Supports dynamic leased secrets (database, PKI) | Limited — leases are not renewed | Yes — Agent renews leases while pod is alive |
| Secret visible in etcd        | Yes (encrypted at rest if configured)            | No                                            |
| Works without Kubernetes      | No                                               | Yes — Agent runs anywhere                     |
| Best for                      | Static or infrequently rotated secrets           | Dynamic leased secrets, PKI certificates      |

Choose ESO when your secrets are KV-style values that are replaced wholesale on rotation (API keys, database passwords changed via a rotation script). Choose the Vault Agent Injector when your secrets are dynamically generated with a TTL and must be renewed during the lifetime of the pod (database dynamic credentials, short-lived TLS certificates from Vault PKI).

## Common Pitfalls

- **Using `secretStoreRef.kind: SecretStore` when the store is a `ClusterSecretStore`** — the `kind` field in `secretStoreRef` must match exactly. A mismatch causes ESO to report `StoreNotFound` and no secret is created.
- **Wrong Vault path format** — for KV v2, the API path is `secret/data/<path>`, not `secret/<path>`. Using the wrong path returns a 404 from Vault and the `ExternalSecret` stays in an error state.
- **The AppRole `secret_id` Kubernetes Secret is in the wrong namespace** — `ClusterSecretStore` fetches the `secretRef` from the namespace specified in the `secretRef.namespace` field. If that namespace or secret does not exist, authentication fails.
- **`refreshInterval: "0"` in production** — setting the interval to `0` means ESO syncs once at creation and never again. If the secret is rotated in Vault, the Kubernetes `Secret` becomes stale permanently. Only use `0` for truly static secrets that never change.
- **Pods not restarting after secret rotation** — ESO updates the Kubernetes `Secret`, but running pods that loaded environment variables at startup still hold the old values. Use [Reloader](https://github.com/stakater/Reloader) or a deployment annotation to trigger a rolling restart when the secret changes.
- **`creationPolicy: Orphan` in production** — with `Orphan` policy, if the `ExternalSecret` is deleted, the Kubernetes `Secret` is not deleted. This can leave stale secrets with old credentials in the cluster. Use `Owner` policy so ESO garbage-collects the secret when the `ExternalSecret` is removed.

## Summary

- ESO syncs secrets from Vault (and other external stores) into native Kubernetes `Secret` objects — applications need no changes
- A `ClusterSecretStore` defines the Vault connection and AppRole credentials; it is cluster-scoped and reusable across namespaces
- An `ExternalSecret` declares which Vault paths to read, which keys to map, and which Kubernetes `Secret` to create
- `refreshInterval` controls how often ESO re-syncs; set it based on how frequently secrets rotate
- Verify sync status with `kubectl get externalsecret` and decode values with `kubectl get secret ... | base64 -d`
- ESO is best for KV-style secrets; use the Vault Agent Injector for dynamic leased secrets that require active lease renewal
- Running pods do not automatically pick up rotated secrets from environment variables — combine ESO with a rolling restart mechanism for zero-downtime rotation
