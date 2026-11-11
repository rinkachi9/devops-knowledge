# Secrets management

## Summary

Secrets management is the secure handling, storage, distribution, rotation and auditing of sensitive credentials: passwords, API keys, tokens, certificates, SSH keys and encryption keys. The goal is that a secret is available to exactly the systems and people that need it, only when they need it, and never sits in source code, images, logs or chat. A secrets manager (HashiCorp Vault, a cloud secret store, Kubernetes Secrets with encryption) centralizes storage, enforces access control, records every access and, at its best, issues short-lived *dynamic secrets* that expire on their own.

## Prerequisites

- [CI/CD overview](../ci-cd/ci-cd-overview.md) for where pipelines need credentials.
- Basic cryptography vocabulary: symmetric encryption, TLS ([SSL and TLS](../networking/network-security/ssl-and-tls.md)).

## Core concepts

### The problem secrets management solves

Credentials leak from many places: hardcoded in source code, committed to Git (and kept forever in history), baked into container images, printed in CI logs, shared in chat, copied into wikis. Static, long-lived credentials make each leak worse, because they work until someone notices and rotates them, which can be years. Without a central store nobody knows which credentials exist, who used them or where they are copied. Secrets management replaces scattered static secrets with one controlled, audited source and, where possible, credentials that live minutes instead of years.

*Analogy.* A hotel key card system instead of metal keys. Metal keys can be copied, never expire and you cannot tell who used one. Key cards are issued at reception to a verified guest, open only the right doors, expire at checkout and every use is logged. The analogy breaks at revocation: a lost metal key means changing the lock, while a key card is simply disabled; with secrets, revoking a static key still means updating every system that used it, which is why dynamic secrets matter.

### Why it matters

1. **Security and risk mitigation.** Protects API keys, database passwords, SSH keys and other secrets from unauthorized access, and prevents credential leaks in code repositories (leaked keys on public GitHub are found by attackers within minutes).
2. **Compliance and regulatory requirements.** Supports GDPR, HIPAA, PCI DSS and ISO/IEC 27001 controls through protected credentials, access control and audit trails of who accessed which secret.
3. **Automation and DevOps integration.** Integrates with CI/CD pipelines, Kubernetes and cloud platforms for secure secret distribution, and enables automatic rotation.

### Key features of a secrets manager

1. **Encryption at rest and in transit.** Secrets stay encrypted in storage and during transmission (TLS).
2. **Access control.** Role-based access control (RBAC) or identity policies enforce *least privilege*: each user or service can read only the secrets it needs.
3. **Secret rotation.** Automatic rotation limits the impact of a compromised credential.
4. **Auditing and logging.** Every access is logged to detect anomalies and support investigations.
5. **Integration.** Works with AWS, Azure, Google Cloud, Kubernetes, Terraform and CI/CD tools.
6. **Centralized storage.** One secured store replaces credentials scattered across files and systems.
7. **Dynamic secrets** (in some tools). Credentials generated on demand for one client with a short lease, then revoked automatically.

### The secret zero problem

Every client must authenticate to the secrets manager, and that authentication needs a credential: the *secret zero*. If secret zero is a static token stored in a file, the problem has only moved. Mature setups use an identity the platform already proves: a cloud instance or workload identity (AWS IAM role, Google Cloud service account, Azure managed identity), a Kubernetes service account token, or an OIDC token issued by the CI system. The secrets manager trusts the platform's signature instead of a stored password.

The sequence shows a pipeline job that receives a short-lived database credential without any stored secret.

```mermaid
sequenceDiagram
  participant CI as CI job
  participant IdP as CI OIDC issuer
  participant V as Secrets manager (Vault)
  participant DB as Database
  CI->>IdP: request identity token for this job
  IdP-->>CI: signed JWT (repo, branch, job)
  CI->>V: log in with JWT
  V->>V: verify signature and claims against role
  V->>DB: create user with 15 minute lease
  V-->>CI: username and password (lease 15 min)
  CI->>DB: run migration
  Note over V,DB: lease expires, Vault drops the user
```

### Secrets in CI/CD

1. **Integrating secrets into pipelines.** Store secrets in Vault, AWS Secrets Manager or Azure Key Vault and inject them at run time as environment variables or files; prefer OIDC federation so the pipeline holds no static cloud keys.
2. **Preventing hardcoded secrets.** Secret scanning tools (for example GitGuardian, TruffleHog, Gitleaks, GitHub secret scanning with push protection) detect secrets before or after commit. Replace hardcoded passwords with API-driven retrieval.
3. **Automating rotation.** Avoid long-lived credentials; for example rotate database credentials automatically every 30 days, or better, use dynamic credentials per run.
4. **Kubernetes.** Kubernetes Secrets are only base64 encoded by default, not encrypted. Enable encryption at rest in etcd (ideally with a KMS provider), restrict access with RBAC, and consider syncing from an external manager (External Secrets Operator, the Secrets Store CSI driver, Vault Agent injector or CyberArk Conjur).

### Best practices

1. **Store secrets securely.** Never store secrets in code repositories (GitHub, GitLab); use a dedicated solution such as HashiCorp Vault or AWS Secrets Manager.
2. **Apply least privilege (RBAC and IAM).** Restrict access with identity and access management policies; for example allow only one specific application to read the database credentials.
3. **Automate rotation.** For example rotate API keys every 7 days with AWS Secrets Manager; rotation must be tested, because a rotation that breaks the consumer is an outage.
4. **Monitor and audit usage.** Enable audit logging and forward it to a SIEM (for example Splunk or the Elastic Stack) to detect unauthorized access.
5. **Encrypt at rest and in transit.** Use strong encryption at rest (for example AES-256 in GCM mode, usually through envelope encryption with a KMS) and TLS for transmission.
6. **Use environment variables carefully.** Injecting secrets as environment variables is common, but variables appear in process listings to the same user, in crash dumps and in debug output; never print them in logs and prefer mounted files with tight permissions for highly sensitive values.
7. **Plan for leaks.** Have a procedure: revoke, rotate, check audit logs for use, and remove from Git history (rewriting history does not help once a secret was public, so rotation comes first).

### Kinds of secrets

| Kind | Examples | Typical lifetime today | Better option |
| --- | --- | --- | --- |
| Human credentials | Passwords, SSH private keys, personal access tokens | Months to years | SSO with MFA, short-lived SSH certificates, fine-grained tokens with expiry |
| Service credentials | Database passwords, API keys of SaaS providers | Months to years | Dynamic credentials, workload identity federation |
| Cloud credentials | AWS access keys, service account JSON keys | Often never rotated | Workload identity (IAM roles, OIDC federation); no keys at all |
| Machine identities | TLS private keys, mTLS certificates | Days to a year | Automated issuance with ACME or a service mesh CA ([ACME](../networking/network-security/acme.md)) |
| Encryption keys | Data keys, signing keys | Years | Keys inside a KMS or HSM, used through an API and never exported |

The general direction is clear: the best secret is one that does not exist (identity federation instead of keys), the second best is one that lives minutes (dynamic secrets), and long-lived static secrets are a last resort that must be stored, rotated and audited carefully.

### The secret lifecycle

```mermaid
stateDiagram-v2
  [*] --> Created: generated by the manager or the target system
  Created --> Stored: encrypted in the secrets manager
  Stored --> Distributed: delivered to an authenticated workload
  Distributed --> InUse
  InUse --> Rotated: schedule or suspected leak
  Rotated --> Stored: new version, old one kept briefly
  InUse --> Revoked: lease expiry or incident
  Revoked --> Destroyed
  Destroyed --> [*]
```

The state diagram shows the stages every secret passes through. Each transition needs an owner and, ideally, automation: who creates the secret, how it reaches the workload, what triggers rotation and how revocation is done in an incident.

### Exposure window

The damage a leaked credential can do is bounded by how long it stays valid after the leak. If a leak happens at a random time, the expected remaining validity is about half the rotation period:

| Credential policy | Rotation period | Expected validity after a leak |
| --- | --- | --- |
| Static, rotated yearly | 365 days | about 182 days |
| Static, rotated every 90 days | 90 days | about 45 days |
| Static, rotated every 30 days | 30 days | about 15 days |
| Dynamic, 1-hour lease | 1 hour | about 30 minutes (0.02 days) |

```mermaid
xychart-beta
  title "Expected validity of a leaked credential (days)"
  x-axis ["Yearly rotation", "90-day rotation", "30-day rotation", "1-hour lease"]
  y-axis "Days" 0 --> 190
  bar [182, 45, 15, 0.02]
```

The chart plots the table: shortening a credential's life reduces the exposure window by orders of magnitude, which is the main argument for dynamic secrets and federation. Attackers also do not wait: credentials pushed to public repositories are commonly found and tried by automated scanners within minutes.

### Delivery patterns

How a secret reaches the application decides how easily it leaks inside the host:

| Pattern | How it works | Risks | Use |
| --- | --- | --- | --- |
| Environment variable | CI system or orchestrator injects `DB_PASSWORD` | Visible to the same user in `/proc/<pid>/environ`, inherited by child processes, dumped in crash reports and debug pages | Simple cases, non-critical secrets |
| Mounted file | Secret written to a tmpfs file (Kubernetes secret volume, Vault Agent template) | Readable by anyone with file access in the container | Default for Kubernetes workloads |
| Sidecar or agent | Vault Agent or a CSI driver renews and rewrites secrets | More moving parts | Dynamic secrets, automatic renewal |
| Direct API or SDK | Application fetches secrets from the manager at start and on renewal | Application needs manager-specific code | Applications that must react to rotation |
| Identity federation | No secret: the workload presents a platform-signed token to the target service | Requires support in the target (cloud IAM, Vault, databases with IAM auth) | Best option wherever available |

### Envelope encryption

Secrets managers and cloud KMSs protect stored data with envelope encryption: each secret (or object) is encrypted with its own data encryption key (DEK), and the DEK is encrypted with a key encryption key (KEK) that never leaves the KMS or HSM.

```mermaid
flowchart LR
  secret["Secret value"] -->|"AES-256-GCM with DEK"| ct["Ciphertext"]
  dek["Data encryption key"] -->|"encrypted by KEK in KMS"| edek["Encrypted DEK"]
  ct --> store[("Storage")]
  edek --> store
  kek["Key encryption key<br/>inside KMS or HSM"] -.-> edek
```

The diagram shows why a stolen database or backup of the secret store is not enough to read secrets: the attacker also needs the KMS to decrypt the DEKs, and every KMS call is authenticated and logged. Rotating the KEK only requires re-encrypting the small DEKs, not all data. Kubernetes encryption at rest with a KMS provider (v2 API) uses the same scheme for etcd.

### Zero-downtime rotation

Rotating a static secret without an outage needs two valid versions for a while:

```mermaid
sequenceDiagram
  participant R as Rotation job
  participant T as Target system (database)
  participant M as Secrets manager
  participant A as Applications
  R->>T: create new credential (version 2), keep version 1 valid
  R->>M: store version 2 as current, version 1 as previous
  A->>M: refresh (on schedule or on error)
  M-->>A: version 2
  Note over A: all instances switch to version 2
  R->>T: verify no logins with version 1, then disable it
```

The sequence follows the pattern AWS Secrets Manager calls the alternating users strategy: the old credential is removed only after every consumer has switched. Rotation scripts must be tested like any other code, because a rotation that breaks consumers is an outage.

### Secret scanning

Scanning catches secrets that slipped into places they should not be. Place it at several points, because each catches different leaks:

| Where | Tool examples | Catches |
| --- | --- | --- |
| Developer machine, pre-commit hook | Gitleaks, TruffleHog, detect-secrets | Secrets before they enter history |
| Push protection on the Git server | GitHub secret scanning push protection, GitLab secret push protection | Secrets in any push, even without local hooks |
| CI pipeline | Gitleaks, TruffleHog | Secrets in merge requests, including history of the branch |
| Container images and artifacts | Trivy, TruffleHog | Secrets baked into image layers |
| Continuous monitoring of public sources | GitHub partner program, GitGuardian | Leaked tokens of your organization on public GitHub |

Scanners use provider-specific patterns (for example the `AKIA` prefix of AWS access key IDs), entropy checks and, in the best tools, live verification against the provider's API to cut false positives. Every finding must lead to rotation; removing the string from the repository is not enough.

### Tools

| Tool | Features | Best for |
| --- | --- | --- |
| HashiCorp Vault (and the OpenBao fork) | Central secret storage, dynamic secrets with leases, encryption as a service (transit engine), API access, many auth methods | Enterprises, multi-cloud, Kubernetes |
| AWS Secrets Manager | AWS integration, automatic rotation with Lambda, IAM-based access | AWS workloads |
| Azure Key Vault | Secrets, keys and certificates, Microsoft Entra ID (formerly Azure AD) authentication, rotation support | Azure workloads |
| Google Cloud Secret Manager | Central secrets for Google Cloud, IAM permissions, automatic versioning and access audit logs | Google Cloud workloads |
| Kubernetes Secrets | Application secrets inside the cluster; integrates with external managers | Kubernetes-native applications, with encryption at rest enabled |
| Doppler | Real-time secret sync across cloud providers and CI/CD | Multi-cloud, developer-friendly setups |
| CyberArk Conjur | Secrets for DevOps, integrates with Ansible, Kubernetes and CI/CD | Enterprises needing advanced access control |
| SOPS, Sealed Secrets | Encrypted secrets that can be stored in Git | GitOps workflows |

Vault changed its license to the Business Source License in 2023 together with Terraform; OpenBao is the open source fork under the Linux Foundation.

## Worked example

Dynamic PostgreSQL credentials with HashiCorp Vault. Untested; Vault 1.17 CLI syntax, checked 2026-11. Host names and passwords are placeholders.

```bash
# 1. Enable the database secrets engine and connect it to PostgreSQL
vault secrets enable database
vault write database/config/shop \
  plugin_name=postgresql-database-plugin \
  connection_url="postgresql://{{username}}:{{password}}@db.example.com:5432/shop" \
  allowed_roles="shop-readonly" \
  username="vault_admin" password="<VAULT_ADMIN_PASSWORD>"

# 2. Define a role: SQL to create a user, and the lease length
vault write database/roles/shop-readonly \
  db_name=shop \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" max_ttl="24h"

# 3. A client with the right policy asks for credentials
vault read database/creds/shop-readonly
```

Expected output (shape):

```text
Key                Value
---                -----
lease_id           database/creds/shop-readonly/AbCdEf123
lease_duration     1h
password           <generated>
username           v-token-shop-rea-x7k2p9
```

Each client gets its own user, valid for one hour. When the lease ends (or `vault lease revoke` is run during an incident), Vault drops the user from PostgreSQL. A leaked credential is useless after an hour, and the audit log shows which client received it.

## Trade offs and when to use it

- **Cloud secret stores** are the simplest choice inside one cloud: no servers to run, native identity. They tie you to that cloud.
- **Vault or OpenBao** fit multi-cloud and on-premises, and offer dynamic secrets and encryption as a service, but are critical infrastructure to operate (high availability, unsealing, backups). If Vault is down, deployments and possibly applications stop.
- **Secrets in Git** (SOPS, Sealed Secrets) fit GitOps but turn key management into the new problem: whoever has the decryption key has every secret.
- **Dynamic secrets** give the strongest security but need support from the target system (databases, cloud IAM) and clients that renew leases.

## Common mistakes

- **Committing a secret and deleting it in the next commit.** It stays in the history. Fix: rotate immediately, then clean history if needed; add pre-commit scanning.
- **Treating base64 as encryption** in Kubernetes Secrets. Fix: encryption at rest, RBAC, external secret managers.
- **One shared admin credential for all services.** A single leak compromises everything and audit logs cannot tell who did what. Fix: one identity per service, least privilege.
- **Printing environment variables in CI logs** (`env`, `set -x`). Fix: rely on the CI system's masking, avoid shell tracing in steps that use secrets.
- **Rotation without consumers in mind.** Applications cache the old password and fail. Fix: overlapping validity (two active versions) and reload logic.

## Practice

1. What is the secret zero problem, and how does workload identity solve it?
2. Why are Kubernetes Secrets not secure by default?
3. What is the advantage of a dynamic database credential over a rotated static one?
4. A developer pushed an AWS access key to a public repository ten minutes ago and deleted it. List the response steps in order.
5. Why is the expected validity of a leaked credential about half its rotation period?
6. What is envelope encryption, and why does it make KEK rotation cheap?
7. Why does zero-downtime rotation need two valid versions of a credential?

Answers:

1. Clients need a credential to fetch secrets, so the first credential must be stored somewhere. Workload identity lets the platform prove the client's identity with a signed token, so no stored credential is needed.
2. Their values are only base64 encoded, and anyone who can read etcd backups or has `get secrets` permission can read them; encryption at rest and RBAC are needed.
3. Each client gets a unique credential with a short lease that is revoked automatically, so leaks expire quickly and audit logs identify the client.
4. Deactivate and rotate the key immediately; check CloudTrail for any use of the key; remove it from Git history if the repository stays public; add secret scanning with push protection to prevent recurrence.
5. A leak happens at a random moment between rotations, so on average half of the period remains until the next rotation.
6. Each secret is encrypted with its own data key, and only the data keys are encrypted with the key encryption key in the KMS; rotating the KEK re-encrypts the small data keys, not all secrets.
7. Consumers switch at different times; if the old version is disabled before every consumer has the new one, those consumers fail.

## Further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) (OWASP): a vendor neutral checklist for storage, rotation and CI/CD.
- [Vault documentation](https://developer.hashicorp.com/vault/docs) (HashiCorp): secrets engines, auth methods, leases and policies.
- [Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) (Kubernetes): encryption at rest, RBAC and external stores.
- [AWS Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) (Amazon Web Services): rotation and IAM access.
- [OpenBao documentation](https://openbao.org/docs/) (OpenBao, Linux Foundation): the open source fork of Vault.
- [AWS Secrets Manager rotation strategies](https://docs.aws.amazon.com/secretsmanager/latest/userguide/rotation-strategy.html) (Amazon Web Services): single user and alternating users rotation.
- [Kubernetes KMS provider for data encryption](https://kubernetes.io/docs/tasks/administer-cluster/kms-provider/) (Kubernetes): envelope encryption of Secrets in etcd.
- [DevSecOps overview](devsecops-overview.md): where secrets management fits among the other DevSecOps controls.
