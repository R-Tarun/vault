# 00 - Prepare without recreating the cluster

## Tools

For lab 01, use Bash, `kubectl`, `helm` (v3 or a compatible later release), and
`git`. Vault's CLI is already inside the server Pod, so installing Vault in WSL
is **not required** for deployment, initialization, or unsealing.

Labs 02 onward use a local `vault` CLI and `jq` for convenience. Install those
before continuing to lab 02, using their official instructions. Prefer a local
CLI matching the selected server version. Installing the client does not mean
you need to run a Vault server in WSL. `kind` and Docker maintain your existing
cluster; do not recreate it for this course.

Commands below only inspect local tools and your current cluster:

```bash
kubectl version --client
helm version
git --version
kubectl config get-contexts
kubectl --context kind-platform-lab get nodes
kubectl --context kind-platform-lab get storageclass
```

Expected: three Ready nodes and a default StorageClass (commonly `standard`
with kind's local-path provisioner). A PVC cannot bind without a provisioner.
If the context is named differently, inspect it and deliberately select that
context; do not create a replacement cluster.

Switch **only after checking the target**. Every cluster command in later labs
uses your current context:

```bash
kubectl config use-context kind-platform-lab
kubectl config current-context
kubectl get namespace vault vault-demo --ignore-not-found
helm list --all-namespaces
```

Stop if either namespace or a `vault` release belongs to other work. These labs
assume those names are unused and reserve them for this course. Do not adopt or
overwrite an existing installation. On a shared machine, also check that the
cluster belongs to you.

## Select and record a chart version

```bash
umask 077
mkdir -p .local
chmod 700 .local
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update hashicorp
helm search repo hashicorp/vault --versions
```

The first walkthrough used **chart `0.34.1` / Vault `2.0.4`**, Helm `3.19.2`,
and the user's existing Kubernetes `1.37.0` kind cluster. These are recorded
observations, not a general compatibility or production-support guarantee.
Chart version and Vault application version are different.

Use the same chart to reproduce lab 01. If intentionally selecting a different
version, review its compatibility, release notes, and values first:

```bash
VAULT_CHART_VERSION=0.34.1
helm show chart hashicorp/vault --version "$VAULT_CHART_VERSION"
helm show values hashicorp/vault --version "$VAULT_CHART_VERSION" > .local/chart-defaults.yaml
printf '%s\n' "$VAULT_CHART_VERSION" > .local/chart-version
```

Inspect the upstream defaults against [our overrides](../helm/vault-values.yaml).
Specifically check `standalone`, `dataStorage`, `auditStorage`, `authDelegator`,
and injector support. Stop on errors or incompatible values. These are manual
commands, not a fail-fast script; do not continue past a failed step.
Record the chart's `appVersion` in your own learning notes.

The chart normally grants the Vault ServiceAccount the TokenReview permission
through `system:auth-delegator`. Lab 03 verifies it instead of creating a second
ClusterRoleBinding.

## Checkpoint

- Explain why you must not use `kind create cluster` again.
- Explain the difference between Helm chart version and Vault server version.
- Explain why kind PVC persistence survives Pod replacement but not cluster
  deletion (or deletion of its underlying Docker nodes).

Next: [Deploy and unlock](01-deploy-and-unlock.md).
