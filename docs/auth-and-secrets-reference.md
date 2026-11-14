# Vault authentication methods and secrets engines

## The difference

| Component | Question | Example from this course |
|---|---|---|
| Auth method | Who is requesting access? | Kubernetes validates a ServiceAccount JWT and issues a Vault token |
| Policy | What can the token do? | `webapp` permits `read` on `kv/data/webapp/config` |
| Secrets engine | What secret or operation is being requested? | KV v2 returns the stored application configuration |

Typical flow:

```text
Workload or user -> auth method -> Vault token with policies
                                      |
                                      v
                         policy check on requested path
                                      |
                                      v
                            secrets engine responds
```

An auth **role** binds login constraints and token settings such as policies;
it is not itself a policy. A secrets-engine role has engine-specific meaning,
such as which database account to create. Both can be named `webapp` without
being the same object. Multiple auth methods and engines can coexist, each
mounted at its own path.

## Common authentication methods

| Method | Who or what authenticates | Typical use | Important consideration |
|---|---|---|---|
| **Kubernetes** | A Kubernetes ServiceAccount JWT | Pods in a Kubernetes cluster | Bind the expected ServiceAccount, namespace, and audience; Vault checks the JWT with Kubernetes TokenReview. |
| **JWT / OIDC** | A signed JWT, or an identity from an OIDC provider | CI/workload identity or browser-based human SSO | Verify issuer, audience, and relevant claims; JWT and interactive OIDC are related but use different login flows. |
| **AppRole** | Machine using a RoleID and normally a SecretID | Automation without a suitable platform identity | Deliver and protect the SecretID securely; use short lifetimes and appropriate usage limits. |
| **AWS** | AWS workload identity | EC2 or IAM-based workloads | Configure the appropriate IAM or EC2 validation and tightly bound role conditions. |
| **LDAP** | User credentials checked against an LDAP directory | Organizations using LDAP or Active Directory | Protect directory connections and map users/groups to appropriate policies. |
| **TLS certificate** | Client presenting a trusted certificate | Services with an existing client-certificate identity | Manage trusted CAs, certificate lifecycle, and role constraints. |
| **Userpass** | Vault-managed username and password | Local learning, small demos, or controlled break-glass scenarios | Local user management and password handling do not provide centralized SSO. |
| **Token** | A Vault token presented directly | Existing Vault clients and controlled bootstrap | A token is already a Vault credential; protect, scope, expire, and revoke it appropriately. Avoid root tokens for applications. |

In this course you initially use the root token for bootstrap, then learn
Kubernetes auth for workloads. Authenticated clients generally receive a Vault
token; the token auth method also lets clients present an existing token
directly. A Vault token is distinct from a Kubernetes JWT.

## Common secrets engines

| Engine | What it provides | Typical use | Important consideration |
|---|---|---|---|
| **KV v2** | Stored key/value data with versioning | Existing API keys, passwords, and configuration | Values are static: Vault does not automatically rotate the external system's password. Soft delete differs from permanent destruction. |
| **Database** | Dynamic database credentials, or configured static-role rotation | Time-limited database access | Requires database connectivity and a sufficiently privileged configured database identity; leases, renewal, and revocation depend on configuration. |
| **PKI** | X.509 certificate issuance and CA management | Short-lived service certificates | Plan CA protection, trust distribution, certificate renewal, and revocation. |
| **Transit** | Encryption, decryption, signing, and other cryptographic operations | Applications protecting their own data | Vault manages cryptographic keys and operations; Transit does not store the submitted application plaintext as a KV secret. |
| **AWS** | AWS credentials based on configured roles | Temporary AWS access for applications | Requires AWS integration and appropriate permissions; supported credential types and lifecycle depend on role configuration. |
| **SSH** | Signed SSH certificates or configured one-time passwords | Short-lived SSH access | Hosts must be configured to trust the signing CA or validate OTPs. |

**Static versus dynamic:** KV v2 stores a value you write. A dynamic database
role can create a credential on demand with a lease; that lease has a separate
lifecycle from the Vault token used to request it. A token's expiration does
not mean every already-delivered file or external copy is erased. For a real
integration, check that the external system actually honors revocation.

**AWS auth vs AWS secrets engine:** AWS auth uses AWS identity to log in to
Vault. The AWS secrets engine supplies AWS credentials to an authorized Vault
client. They solve different problems and may be used independently or together.

**Certificate auth vs PKI:** certificate auth verifies a client certificate to
log in. PKI issues and manages certificates. Issuing a certificate does not
automatically configure certificate-based Vault login.

## Choosing a starting point

- **Human administrator:** OIDC with an existing identity provider is a common
  starting point; design policies and group mappings for least privilege.
- **Kubernetes workload:** Kubernetes auth or an appropriate JWT identity;
  start with the dedicated ServiceAccount and policy used in lab 03.
- **CI outside Kubernetes:** use the CI platform's signed identity with JWT
  auth where available, or AppRole with a secure SecretID delivery process.
- **Database application:** consider dynamic database credentials when the
  application can handle renewal, expiry, and reconnection.
- **Existing static application password:** KV v2 is straightforward, but
  coordinate any rotation with the external system and application reload.

These are starting points, not universal recommendations. Validate
compatibility, editions, external dependencies, and operational requirements
against the documentation for your Vault version before enabling a feature.

## Official documentation

- [Authentication methods](https://developer.hashicorp.com/vault/docs/auth)
- [Secrets engines](https://developer.hashicorp.com/vault/docs/secrets)
- [Policies](https://developer.hashicorp.com/vault/docs/concepts/policies)
- [Leases](https://developer.hashicorp.com/vault/docs/concepts/lease)

This reference is conceptual; it does not enable any method or engine.
