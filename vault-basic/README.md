# HashiCorp Vault Basic

<!-- TODO: add a 2–4 sentence course description here. -->

## Curriculum

<!-- 6 sections · 17 themes -->

### Fundamentals and Architecture

Secrets management concepts, Vault architecture (storage backend, barrier, auth methods, secrets engines), and installation.

- [Secrets Management Fundamentals](./vault-basic-fundamentals/vault-secrets-management.md)
- [Vault Architecture](./vault-basic-fundamentals/vault-architecture.md)
- [Vault Installation](./vault-basic-fundamentals/vault-installation.md)

### Installation and Configuration

Running Vault in dev mode, configuring production server, initialization, unsealing, and CLI basics.

- [Dev Server](./vault-basic-installation/vault-dev-server.md)
- [Production Configuration](./vault-basic-installation/vault-production-config.md)
- [Init, Unseal, and CLI](./vault-basic-installation/vault-init-unseal-cli.md)

### Core Concepts and KV Secrets Engine

Paths, mounts, leases, tokens, and the KV v1/v2 secrets engine including versioning.

- [Core Concepts](./vault-basic-core-kv/vault-core-concepts.md)
- [KV Secrets Engine](./vault-basic-core-kv/vault-kv-secrets-engine.md)
- [KV v2 Versioning](./vault-basic-core-kv/vault-kv-v2-versioning.md)

### Authentication Methods

Token and AppRole auth, Kubernetes and LDAP auth, JWT/OIDC and Userpass authentication.

- [Token and AppRole Authentication](./vault-basic-auth-methods/vault-token-approle.md)
- [Kubernetes and LDAP Authentication](./vault-basic-auth-methods/vault-kubernetes-ldap-auth.md)
- [JWT, OIDC, and Userpass](./vault-basic-auth-methods/vault-jwt-oidc-userpass.md)

### Policies

HCL policy language, writing capability rules, and testing policies with auth methods.

- [Policy Language](./vault-basic-policies/vault-policy-language.md)
- [Writing and Testing Policies](./vault-basic-policies/vault-writing-testing-policies.md)

### Vault Agent

Vault Agent for automatic authentication, template rendering for config file injection, and External Secrets Operator.

- [Vault Agent Basics](./vault-basic-agent/vault-agent-basics.md)
- [Template Rendering](./vault-basic-agent/vault-template-rendering.md)
- [External Secrets Operator](./vault-basic-agent/vault-external-secrets-operator.md)
