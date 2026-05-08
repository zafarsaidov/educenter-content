# HashiCorp Vault Advanced

<!-- TODO: add a 2–4 sentence course description here. -->

## Curriculum

<!-- 5 sections · 14 themes -->

### PKI Secrets Engine

Building a CA hierarchy with Vault PKI, issuing and revoking certificates, and automated certificate renewal.

- [PKI CA Hierarchy](./vault-adv-pki/vault-pki-ca-hierarchy.md)
- [Issuing and Revoking Certificates](./vault-adv-pki/vault-pki-issuing-revoking.md)
- [PKI Auto-Renewal](./vault-adv-pki/vault-pki-auto-renewal.md)

### Transit Secrets Engine

Encryption-as-a-service with the Transit engine: encrypting/decrypting data and key management.

- [Transit Encrypt and Decrypt](./vault-adv-transit/vault-transit-encrypt-decrypt.md)
- [Transit Key Management](./vault-adv-transit/vault-transit-key-management.md)

### Vault in Kubernetes

Deploying Vault with Helm, Kubernetes auth method and Vault Agent Injector, ESO vs VSO comparison.

- [Vault Helm Deployment](./vault-adv-kubernetes/vault-helm-deployment.md)
- [Kubernetes Auth and Injector](./vault-adv-kubernetes/vault-kubernetes-auth-injector.md)
- [ESO vs VSO](./vault-adv-kubernetes/vault-eso-vs-vso.md)

### High Availability

Vault HA architecture, setting up a Raft integrated storage cluster, and auto-unseal with cloud KMS.

- [HA Architecture](./vault-adv-ha/vault-ha-architecture.md)
- [Raft Cluster Setup](./vault-adv-ha/vault-raft-cluster-setup.md)
- [Auto-Unseal](./vault-adv-ha/vault-auto-unseal.md)

### Audit, Monitoring, and Operations

Audit devices for compliance, monitoring Vault with metrics and logs, and day-2 operations.

- [Audit Devices](./vault-adv-operations/vault-audit-devices.md)
- [Monitoring Vault](./vault-adv-operations/vault-monitoring.md)
- [Vault Operations](./vault-adv-operations/vault-operations.md)
