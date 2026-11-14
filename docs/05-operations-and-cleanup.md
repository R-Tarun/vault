# 05 - Audit, recover, troubleshoot, clean up

## 1. Enable an audit device

Use your administrative shell. The chart provides a separate persistent audit
volume at `/vault/audit`.

```bash
vault audit enable file file_path=/vault/audit/audit.log
vault audit list
vault kv get -mount=kv webapp/config
kubectl -n vault exec vault-0 -- tail -n 4 /vault/audit/audit.log
```

Expected: request and response entries, including operation and request path.
Sensitive string values are normally HMACed; audit output still contains
security-relevant metadata. Do not enable raw logging or commit/copy these logs
to a public place. Not all metadata is secret-redacted automatically.

If already enabled, inspect `vault audit list` instead of enabling a duplicate
device. Vault must successfully write to at least one enabled audit device to
serve audited requests. A full/broken audit volume can therefore block requests.
This lab does not configure rotation: keep it short-lived and inspect capacity.

## 2. Restart and recover without losing KV data

This deliberately interrupts **your lab's single server**. First check that
the local unseal key is available. Do not perform this against a shared or
production service.

```bash
kubectl -n vault get pvc
kubectl -n vault delete pod vault-0
kubectl -n vault get pods -w
```

Stop watching with Ctrl+C when the new `vault-0` is Running. It will be unready.
The old port-forward usually exits when its Pod is deleted. Stop any remaining
old forward, then start a new one in its dedicated terminal:

```bash
kubectl -n vault port-forward --address 127.0.0.1 service/vault 8200:8200
```

From the working terminal:

```bash
vault status
vault operator unseal
vault status
vault kv get -mount=kv webapp/config
vault audit list
kubectl -n vault wait --for=condition=Ready pod/vault-0 --timeout=120s
```

Paste the original unseal key at the prompt. Expected before unseal:
initialized `true`, sealed `true` (status exit 2). After unseal: the latest KV
secret still exists (`demo-only-v3` after lab 04, or `demo-only-v4` after
optional lab 04a), and the audit device remains enabled. Do **not** initialize
again.

The Agent may retry while Vault is unavailable. Already-rendered static files
can remain readable during an outage; Vault sealing is not remote erasure of
copies already delivered to applications. After recovery:

```bash
kubectl -n vault-demo logs deployment/webapp -c vault-agent --tail=80
kubectl -n vault-demo exec deployment/webapp -c webapp -- \
  cat /vault/secrets/config
```

Persistence is not backup. This file-storage lab does not support `vault
operator raft snapshot`; that command requires integrated Raft storage.
Backup/restore and HA are separate future labs.

## 3. Troubleshooting drill

Use the [symptom guide](troubleshooting.md). Work through: context -> Pod/PVC
state -> seal status -> auth -> policy path -> injection -> application file.

Do not "fix" a 403 by attaching `root` or wildcards. Do not solve an unready
server by turning on dev mode. Write down the failed layer and the evidence.

## 4. Pause without destroying data

Stop your port-forward using Ctrl+C. Clear the shell token:

```bash
unset VAULT_TOKEN
```

This does not revoke tokens or stop Vault. Leave the local key file protected if
you intend to resume. Later, restart the port-forward and set `VAULT_ADDR` and
`VAULT_TOKEN` as in lab 01; unseal only if needed.

## 5. Destructive cleanup: only when finished

These commands destroy **this course's data**. They assume the two namespaces
were created exclusively for this course. If they contain other resources,
stop and scope cleanup to your own resources instead.

If you installed VSO in optional lab 04a, first follow its
[scoped cleanup](04a-vso-auto-restart.md#5-optional-lab-cleanup) while Vault
and `vault-demo` still exist. Review any retained CRDs separately.

First inspect the context and inventory:

```bash
kubectl config current-context
helm -n vault list
kubectl -n vault get all,pvc
kubectl -n vault-demo get all,serviceaccounts
kubectl get pv
```

Confirm the context is your intended local cluster, then perform each step
manually:

```bash
kubectl delete -f kubernetes/webapp.yaml
helm uninstall vault -n vault
kubectl delete namespace vault
kubectl get pv
```

The app manifest includes `vault-demo`, so deleting it deletes that entire
namespace. Deleting `vault` deletes its PVCs, including retained StatefulSet
claims. PV cleanup depends on the StorageClass reclaim policy; verify it.
Do not delete unrelated PVs. Helm normally removes its injector webhook and
ClusterRoleBindings; if uninstall fails, inspect release resources rather than
deleting cluster-wide resources by broad label patterns.

Stop **your** port-forward terminal with Ctrl+C. If your local files contain
only this completed lab's material, remove those named files explicitly:

```bash
rm -f -- .local/init.json .local/webapp-token.json \
  .local/kubernetes-login.json .local/webapp.jwt \
  .local/wrong-serviceaccount.jwt .local/rendered-vault.yaml \
  .local/chart-defaults.yaml .local/chart-version
unset VAULT_TOKEN VAULT_ADDR VAULT_CHART_VERSION
```

Deleting a file is not guaranteed secure erasure on SSDs/WSL; encryption and
appropriate secret handling matter. Do not delete the existing kind cluster:
other learning workloads may still be using it.

## Final evidence checklist

- [ ] Vault is persistent, non-dev, initialized, and manually unsealed.
- [ ] KV version read/delete/undelete behavior is understood.
- [ ] Restricted token reads only its exact secret; writes and other reads fail.
- [ ] Token revocation is verified independently of TTL expiration.
- [ ] Correct Kubernetes ServiceAccount logs in; wrong one is rejected.
- [ ] Injector delivers a file and updates it without a Pod restart.
- [ ] Audit entries exist and survive the server restart.
- [ ] Restart requires unseal but preserves secrets and configuration.
- [ ] Cleanup removes only the lab; existing cluster workloads remain.

Record pass/fail and observations privately, **without** tokens, keys, JWTs, or
secret values. These are acceptance criteria for each learner to execute. The
user-reported walkthrough confirmed a nonempty audit log, unseal recovery,
retained KV version 4 and audit configuration, a nonempty audit log after restart,
and healthy workloads. It did not inspect raw audit entries in the shared
output or perform cleanup.
