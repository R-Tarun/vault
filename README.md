# Vault on kind: learn by doing

A manual, incremental HashiCorp Vault course for the existing `platform-lab`
kind cluster. No automation runs the labs for you. This repository does not
recreate your cluster or depend on the Argo CD, Grafana, or Istio exercises.

## Scope and learning outcomes

| Lab | Learn | Evidence you collect |
|---|---|---|
| [00: Prerequisites](docs/00-prerequisites.md) | Context safety, architecture, tool and chart selection | Correct cluster, default StorageClass, selected chart version |
| [01: Deploy and unlock](docs/01-deploy-and-unlock.md) | Storage, initialization, seal/unseal, root token | Initialized, unsealed Vault with a bound PVC |
| [02: KV, policies, tokens](docs/02-kv-policies-tokens.md) | KV v2 versions, least privilege, TTLs, revocation | Allowed read and denied write/cross-app read |
| [03: Kubernetes authentication](docs/03-kubernetes-auth.md) | ServiceAccounts, TokenReview, roles, identity | Workload JWT exchanges for a restricted Vault token |
| [04: Agent injection](docs/04-agent-injection.md) | Admission mutation, templates, sidecar lifecycle | File delivered without a Kubernetes Secret; update rendered |
| [04a: VSO auto-restart](docs/04a-vso-auto-restart.md) (optional) | Kubernetes Secret sync and automatic rolling replacement | A changed Vault secret produces a new Pod UID with updated data |
| [05: Operations and cleanup](docs/05-operations-and-cleanup.md) | Audit, restart/unseal recovery, debugging, scoped teardown | Read access audited; data survives a restart |

Use the labs in order. All commands are for **you** to run from the repository
root in a Bash-compatible terminal (for example WSL). Only local static checks
may be run while authoring this repository; no Vault, Helm, or cluster operation
is required to create it.

## Walkthrough status

The user-reported walkthrough deployed chart **`0.34.1` / Vault `2.0.4`** on
the existing kind cluster. After initialization/unseal, the Vault server and
injector were both `1/1 Running`; data and audit PVCs were Bound at 1Gi each.
Initialization and unsealing used `kubectl exec`, not a WSL-installed Vault CLI.
See [lab 01](docs/01-deploy-and-unlock.md) for the exact sequence.

The user subsequently reported these results on the same local cluster:

- KV v2 versioning and soft-delete/restore worked; a restricted token could
  read its own secret but received 403 for another path. Token revocation was
  performed.
- Kubernetes auth issued a token with the `webapp` policy; the authorized
  ServiceAccount could read its secret, and the wrong ServiceAccount was
  reported rejected.
- Vault Agent rendered the updated secret while the Pod UID stayed unchanged
  and its containers had zero restarts.
- VSO chart **`1.6.2`** synced a Kubernetes Secret and triggered a successful
  rollout: the Pod UID changed, the restart annotation appeared, and the new
  Pod received the updated value.
- A file audit device was enabled and its log was nonempty. Vault restarted
  sealed, was manually unsealed, and retained KV version 4, its audit
  configuration, and a nonempty audit file. Both demo workloads were healthy
  afterward, and the Agent file was current.

These are user-reported observations from this walkthrough, not a production
compatibility guarantee. Raw audit entries, UI login, backup/restore, and
cleanup were not verified in the shared results. **Vault, VSO, the workloads,
and their storage were left running**; cleanup remains optional and untested.
The assistant did not run Vault, Helm, or Kubernetes commands.

## Architecture

```text
Your terminal -- localhost:8200 port-forward --> vault-0
                                                |
                                                +-- PVC: file storage
                                                +-- Kubernetes TokenReview API

Pod creation --> Vault Agent Injector webhook --> mutated webapp Pod
                                                   |
                               ServiceAccount JWT --> Kubernetes auth in Vault
                                                   |
                                      webapp policy --> KV v2 read
                                                   |
                                      Agent template --> /vault/secrets/config
                                                   |
                                                  nginx
```

The injector adds an init container and a running Agent sidecar. The Agent
authenticates, obtains short-lived Vault tokens, and renders a file on a shared
memory-backed volume. The application does not receive the administrator token.
For clarity, nginx only demonstrates file delivery; it does not use a database
or serve the secret file over HTTP.

## Lab-only choices

- One Vault server, persistent **file** storage, and manual Shamir unseal.
- HTTP with TLS disabled, including inside the cluster. Loopback-only
  port-forwarding limits host exposure but does **not** make cluster traffic
  encrypted. Use only disposable fake data on an isolated local cluster.
- One unseal share with threshold one, to make the lifecycle understandable.
  Real deployments require separation of duties or an appropriate auto-unseal
  design; never keep root tokens and unseal material together in production.
- The initial root token bootstraps the lab. Workloads use a restricted policy.
  Human non-root administration and root-token revocation are later exercises.
- No Istio injection in the `vault` or `vault-demo` namespaces. This course does
  not configure mesh mTLS or sidecar ordering.
- No production readiness, compliance, or HA claims. Kubernetes replicas alone
  do not turn file storage into HA.

## Repository map

- [Helm overrides](helm/vault-values.yaml): persistent standalone server and injector.
- [Workload](kubernetes/webapp.yaml): namespace, ServiceAccount, and Deployment.
- [VSO workload](kubernetes/vso-webapp.yaml) and [VSO resources](kubernetes/vso-resources.yaml):
  optional Secret sync and rolling-restart example.
- [Read-only policy](policies/webapp.hcl): one KV v2 secret path.
- [Auth methods and secrets engines](docs/auth-and-secrets-reference.md):
  common options, use cases, and how they differ.
- [Troubleshooting reference](docs/troubleshooting.md): symptoms and diagnostic order.
- [Existing kind configuration](kind-platform.yaml): reference only; do not recreate
  an already-running cluster.

Never commit `.local/`, credentials, rendered files, unseal material, snapshots,
or audit output. Git ignore rules are a guardrail, not encryption or a secret
scanner. Inspect staged changes before any commit.

## What comes next (not implemented here)

After these essentials: three-node **integrated Raft** HA with separate PVCs,
TLS, quorum and join/unseal behavior; snapshots and tested restore; human OIDC
or another non-root admin auth method; AppRole and response wrapping; dynamic
database credentials and leases; PKI; transit encryption; and Kubernetes CSI.
The optional VSO lab is a starting point, not a production deployment guide.

GitOps should store policies, roles, and manifests, never plaintext secret values
or bootstrap tokens. AWS auto-unseal and multi-region disaster recovery need
separate designs. Multiple clusters/regions do not automatically provide Vault
replication, traffic failover, or data replication; some replication features
also require Vault Enterprise.

## Official references

- [Vault concepts](https://developer.hashicorp.com/vault/docs/concepts)
- [Helm configuration](https://developer.hashicorp.com/vault/docs/platform/k8s/helm/configuration)
- [Kubernetes auth](https://developer.hashicorp.com/vault/docs/auth/kubernetes)
- [Injector annotations](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations)
- [KV v2](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)
- [Audit devices](https://developer.hashicorp.com/vault/docs/audit)

Documentation links are provided for reference. The prerequisite guide records
the walkthrough's chart version for repeatability; it does not assert the
latest available release. Review compatibility, values, and release notes
before intentionally changing versions.
