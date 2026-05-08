# Vault Architecture

Vault is not just a key-value store with a password on it — it is a system designed so that even a full copy of its storage is useless to an attacker without the unseal keys. This lesson explains the components that make Vault work, why the seal/unseal model exists, how Shamir's Secret Sharing protects the master key, and what happens at every step when your application requests a secret.

## Vault Components

A running Vault deployment has four distinct parts:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Vault Server                             │
│                                                                 │
│  ┌─────────────┐    ┌──────────────────────────────────────┐    │
│  │  Listener   │    │           Vault Core                 │    │
│  │ (TCP/TLS)   │◄──►│  Auth │ Policy │ Secrets Engines     │    │
│  └─────────────┘    └───────────────────┬──────────────────┘    │
│                                         │                       │
│                              ┌──────────▼───────────┐           │
│                              │   Encryption Barrier │           │
│                              │    (AES-256-GCM)     │           │
│                              └──────────┬───────────┘           │
│                                         │                       │
│                              ┌──────────▼───────────┐           │
│                              │   Storage Backend    │           │
│                              │ (Raft, Consul, file) │           │
│                              └──────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │ HTTPS (CLI / SDK / direct API calls)
         │
┌────────┴────────┐
│  Vault Client   │
│ (CLI, SDK, API) │
└─────────────────┘
```

**Server** — the long-running daemon (`vault server`). It handles all client requests, runs the auth methods, enforces policies, and serves the secrets engines. You never interact with storage directly; all reads and writes go through the server.

**Client** — anything that talks to the server over HTTPS. This includes the `vault` CLI, official SDKs (Go, Python, Java, .NET, Ruby, Node.js), third-party tools like Terraform and Ansible, or a raw `curl` request to the REST API.

**Storage backend** — where the server persists data. Vault supports multiple backends: integrated Raft (the recommended default), HashiCorp Consul, Amazon S3, PostgreSQL, and others. Critically, **everything written to storage is already encrypted** by the time it hits the backend. The storage backend sees only ciphertext.

**Encryption barrier** — a logical layer between Vault Core and the storage backend. All data is encrypted with AES-256-GCM before being written and decrypted after being read. The keys used for this encryption never leave Vault's memory.

## The Seal/Unseal Model

Every time the Vault server process starts (or restarts), it begins in a **sealed** state.

```
Server restarts
      │
      ▼
┌─────────────┐
│   SEALED    │   Vault knows WHERE data is (storage backend is reachable)
│             │   but CANNOT READ it (encryption key is not loaded)
└─────────────┘
      │  operator provides unseal key shares
      ▼
┌─────────────┐
│  UNSEALED   │   Encryption key is reconstructed and held in memory
│             │   Vault can now encrypt/decrypt data — ready for requests
└─────────────┘
```

**Why does this matter?** If an attacker steals the storage backend — a full database dump, a disk snapshot, a cloud object storage bucket — they have only encrypted blobs. Without the unseal process, the data is unreadable. The storage backend and the unseal keys are designed to be stored and controlled separately.

In production, Vault is typically deployed behind a process manager that restarts it automatically (systemd, Kubernetes). But after every restart, it must be manually unsealed (or configured for auto-unseal with a cloud KMS). This is intentional: it forces a human or an automated process to make a conscious decision to bring Vault back online.

## Shamir's Secret Sharing

When you initialise Vault for the first time with `vault operator init`, Vault generates a **master key** and immediately splits it using Shamir's Secret Sharing algorithm.

The idea: the master key is divided into `N` key shares. Any `K` of those shares can mathematically reconstruct the original key. With fewer than `K` shares you get zero information about the master key — not even a hint.

```
vault operator init -key-shares=5 -key-threshold=3

Vault produces 5 key shares:
  Share 1: 6ecb1a...
  Share 2: 2f4d89...
  Share 3: a01c55...
  Share 4: 7b3e12...
  Share 5: 9d77f4...

Any 3 of these 5 shares can unseal Vault.
No single person holds the full master key.
```

**Typical distribution:** Each share is given to a different trusted team member (a "keyholder"). Unsealing requires `K` of them to cooperate, which means:

- No single person can unseal Vault alone.
- Even if one or two keyholders are compromised, Vault stays sealed.
- If keyholders leave the organisation, you can rekey Vault to generate new shares.

The default values in Vault are 5 shares and a threshold of 3. For a solo developer running a personal Vault instance, `1` share and `1` threshold is convenient (though it eliminates the key-splitting protection).

## The Encryption Barrier in Detail

After unseal, the encryption key resides **only in memory**. It is never written to disk or to the storage backend. This is why sealing (or rebooting) makes Vault unreadable again — the key is gone from RAM.

```
Unseal process:
  3 key shares provided
        │
        ▼
  Shamir reconstruction → master key (in memory only)
        │
        ▼
  master key decrypts the encryption key (stored in backend, encrypted)
        │
        ▼
  encryption key loaded into memory → Vault is unsealed

Write path:
  Vault Core produces plaintext data
        │
        ▼
  Encryption barrier encrypts with AES-256-GCM using the in-memory encryption key
        │
        ▼
  Ciphertext written to storage backend

Read path:
  Ciphertext read from storage backend
        │
        ▼
  Encryption barrier decrypts with in-memory encryption key
        │
        ▼
  Plaintext returned to Vault Core
```

The encryption key itself is stored in the backend — but it is encrypted with the master key. The master key is reconstructed from unseal shares and held only in memory. There is no unencrypted key on disk at any time.

## Vault Request Data Flow

Every client request travels through the same pipeline:

```
Client
  │
  │  HTTPS request
  │  e.g. GET /v1/secret/data/db/prod
  ▼
Listener (TLS termination)
  │
  │  Token extracted from X-Vault-Token header
  ▼
Auth Layer
  │  Is this token valid? Is it expired?
  │  Which policies are attached to it?
  ▼
Policy Engine
  │  Does any attached policy allow
  │  capability "read" on path "secret/data/db/prod"?
  │  If NO → 403 Forbidden
  ▼
Secrets Engine (KV v2, mounted at "secret/")
  │  Retrieve the secret at path "data/db/prod"
  │  (storage read → decrypt → return plaintext)
  ▼
Response
  │  JSON body with secret data + lease metadata
  ▼
Client
```

Every step is logged to the audit device if one is enabled. The policy check happens before the secrets engine is invoked — Vault never touches the data if the caller is not authorised.

## Dev Mode vs Production Mode

Vault ships with a built-in dev mode (`vault server -dev`) designed for local experimentation. Understanding the difference is critical before deploying anything real.

| Property | Dev mode | Production mode |
|---|---|---|
| Storage | In-memory (ephemeral) | Persistent (Raft, Consul, etc.) |
| TLS | Disabled | Required |
| Seal state on start | Auto-unsealed | Sealed — requires manual unseal |
| Root token | Fixed (`root` by default) | Generated once during `operator init` |
| Init/unseal steps | Skipped entirely | Required |
| Data persistence | Lost on restart | Survives restarts |
| Suitable for | Local development, demos, testing | Production, staging, any persistent use |
| KV engine | Mounted at `secret/` automatically | Must be enabled manually |

Dev mode prints the root token and unseal key to stdout on startup so you can connect immediately:

```
==> Vault server configuration:

             Api Address: http://127.0.0.1:8200
                     Cgo: disabled
         Cluster Address: https://127.0.0.1:8201
              Listener 1: tcp (addr: "127.0.0.1:8200", cluster address: "127.0.0.1:8201", max_request_duration: "1m30s", max_request_size: "33554432", tls: "disabled")
               Log Level: info
                   Mounts: cubbyhole/ (cubbyhole), identity/ (identity), secret/ (kv_v2), sys/ (system)
                 Storage: inmem
                 Version: Vault v1.17.0

WARNING! dev mode is enabled! In this mode, Vault runs entirely in-memory
and starts unsealed with a single unseal key. The root token is:

hvs.root

The unseal key and root token are displayed below in case you want to
seal/unseal the Vault or re-authenticate.

Unseal Key: SrIZz...
Root Token: hvs.root
```

Never use dev mode outside of a local laptop. The in-memory storage means every secret you write disappears when the process exits, and the fixed root token is a security liability.

## Common Pitfalls

- **Losing unseal key shares.** If you lose enough shares that you cannot meet the threshold, Vault is permanently sealed and all data is unrecoverable. Store key shares in separate secure locations — not all in the same password manager or the same team Slack channel.

- **Storing all key shares in one place.** The entire point of Shamir's Secret Sharing is that no single point of compromise unlocks Vault. If all five shares live in one LastPass vault or one S3 bucket, you have one point of failure.

- **Restarting Vault without a plan for unsealing.** In production, a server reboot (OS update, instance replacement, crash) leaves Vault sealed. Plan for this: either configure auto-unseal with a cloud KMS (AWS KMS, Azure Key Vault, GCP KMS) or have an on-call process for manual unsealing.

- **Using dev mode in staging or CI.** Dev mode's root token is `root` (or whatever you pass via `-dev-root-token-id`). If that token is committed to a config file or exposed in CI logs, any caller can access the Vault instance. Use dev mode only locally with no sensitive data.

- **Ignoring TLS in production.** Vault communicates secrets over HTTP in dev mode. In any other environment, TLS is mandatory. Without TLS, secrets are transmitted in plaintext and anyone on the network path can read them.

## Summary

- The Vault server handles all requests; the storage backend holds only encrypted ciphertext and never sees plaintext data.
- Vault starts sealed on every restart — it knows where data is stored but cannot decrypt it until unsealed.
- Shamir's Secret Sharing splits the master key into `N` shares; any `K` of them reconstruct it — no single keyholder can unseal Vault alone.
- The encryption barrier uses AES-256-GCM; the encryption key lives only in memory after unseal, never on disk.
- Every request flows through TLS termination → token validation → policy check → secrets engine → response; the policy check happens before any data is accessed.
- Dev mode is ephemeral, auto-unsealed, and uses a fixed root token — safe only for local development and demos, never for production.
