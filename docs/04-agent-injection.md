# 04 - Deliver a secret with Vault Agent

## 1. Confirm prerequisites and deploy

Check that Vault is unsealed, the injector is Ready, and lab 03's login worked:

```bash
vault status
kubectl -n vault get pods
vault read auth/kubernetes/role/webapp
```

The manifest declares a projected token volume named `vault-auth` with
`audience: vault`, matching the role's `audience=vault`. The
`agent-service-account-token-volume-name` annotation tells the injector to mount
that volume into its Agent containers. Automatic ServiceAccount token mounting
is disabled, and the nginx container does not mount the projected JWT. The
kubelet rotates this short-lived token; you do not copy it into a Secret.

```bash
kubectl apply -f kubernetes/webapp.yaml
kubectl -n vault-demo rollout status deployment/webapp --timeout=180s
kubectl -n vault-demo get pods
```

Expected: the Pod has an Agent init container, a `webapp` container, and a
`vault-agent` sidecar. Injection occurs when the Pod is **created**, not by
changing containers in an already-existing Pod.

```bash
kubectl -n vault-demo get pods -l app=webapp \
  -o jsonpath='{range .items[*]}{.spec.initContainers[*].name}{"\n"}{.spec.containers[*].name}{"\n"}{end}'
kubectl -n vault-demo exec deployment/webapp -c webapp -- \
  cat /vault/secrets/config
kubectl -n vault-demo get secrets
```

Only fake values should appear: `username=demo-user`, `password=demo-only-v2`.
No Kubernetes Secret containing these values is created by this approach.
The workload sees a file, not the root token or your `.local/` files.

File permissions are `0444` **inside this demo Pod** to allow the non-root app
and Agent's different UIDs to share it. This is not a host-readable file; still,
any container mounting that volume can read it. Use a reviewed UID/GID and
volume-permission design for a real workload.

## 2. Update the static secret

From the administrative shell:

```bash
vault kv put -mount=kv webapp/config username=demo-user password=demo-only-v3
```

The annotation sets a **15-second static-secret rendering interval** for this
lab. Allow approximately 30 seconds, then inspect again:

```bash
kubectl -n vault-demo exec deployment/webapp -c webapp -- \
  cat /vault/secrets/config
```

Expected: `demo-only-v3`, without restarting the Pod. If unchanged, inspect
Agent logs; do not assume immediate push delivery. Static KV secrets do not
have renewable leases like dynamic database credentials.

A changed file does **not** prove an application reloaded its credentials.
This nginx demo only verifies delivery; a real service must watch/reload files
or have an intentional restart mechanism. Environment variables in an existing
process do not update because a file changed.

## 3. Self-check and optional failure exercise

```bash
kubectl -n vault-demo logs deployment/webapp -c vault-agent --tail=80
kubectl -n vault-demo describe deployment webapp
```

Logs can contain sensitive operational metadata; do not publish them unreviewed.

For an optional failure exercise, change the Deployment's role annotation to a
nonexistent role using `kubectl edit deployment webapp -n vault-demo`. A new
Pod should remain in init with Agent login errors. Restore role `webapp`, then
verify rollout completes. Keep `kubernetes/webapp.yaml` as the known-good state;
applying it again restores its declared annotations.

## Checkpoint

Explain why the application needs neither a root token nor a Kubernetes Secret,
why admission mutation differs from runtime secret renewal, and why application
reload behavior is separate from Agent templating.

Next: [Operations, recovery, and cleanup](05-operations-and-cleanup.md).
