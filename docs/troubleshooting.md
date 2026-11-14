# Troubleshooting reference

Use the current lab's expected output before changing configuration. Commands
below inspect state unless explicitly marked as a fix.

| Symptom | Inspect | Likely layer / next action |
|---|---|---|
| Cannot reach cluster | `kubectl config current-context`, `kubectl get nodes` | Wrong context or stopped local nodes; do not recreate the cluster |
| Vault Pending | `kubectl -n vault describe pod vault-0`, `kubectl -n vault get pvc` | Missing default StorageClass, pending PVC, or resource pressure |
| Vault Running but unready | `vault status`, `kubectl -n vault logs vault-0 --tail=80` | Before initialization/unseal this is expected |
| Connection refused locally | Check your port-forward terminal and `VAULT_ADDR` | Restart the forward after Pod replacement; check the chosen local port |
| HTTPS/HTTP mismatch | Inspect `VAULT_ADDR` and Helm values | This lab uses `http://127.0.0.1:8200`, not Vault CLI's default HTTPS |
| KV read 403 | `vault token capabilities kv/data/webapp/config` | Token/policy mismatch, expired/revoked token, or wrong KV v2 path |
| KV read 404 | `vault kv metadata get -mount=kv webapp/config` as admin | Wrong mount/path, missing data, or soft-deleted version; not a successful auth test |
| Kubernetes login rejected | Inspect role, audience, namespace, SA name | All must match; inspect TokenReview RBAC and Vault server logs |
| TokenReview 403 | `kubectl auth can-i create tokenreviews.authentication.k8s.io --as=system:serviceaccount:vault:vault` | Server needs chart's auth-delegator binding; do not grant it to the app |
| Login works, secret read fails | Inspect `token_policies` on the role and policy contents | Authentication succeeded; authorization did not |
| No Agent containers | Inspect Pod annotations, webhook, and injector logs | Annotations belong on Pod template; only newly-created Pods are mutated |
| Pod stuck in init | `kubectl -n vault-demo logs deployment/webapp -c vault-agent-init --tail=80` | Sealed/unreachable Vault, rejected role/audience, bad template or secret path |
| Rendered file missing | Inspect init/Agent logs and file annotation | `kv/data/...` API path and `.Data.data` for KV v2 are required |
| File permission denied | Inspect app UID/GID and volume permissions | This demo uses `0444`; do not solve by running app privileged |
| Updated secret not rendered | Inspect `vault-agent` logs and interval annotation | Allow polling time; compare static KV refresh with dynamic lease renewal |
| File updated, app unchanged | Inspect application reload behavior | Agent cannot force a process to reread files or update its environment |
| Requests fail after audit enablement | Server logs, audit path permissions and disk capacity | Audit failure can block requests; do not disable auditing as a routine workaround |
| OOMKilled / eviction | `kubectl -n vault describe pod vault-0`, Pod events | Review local capacity and resource limits before changing them |

Useful focused inspection:

```bash
kubectl -n vault get events --sort-by=.metadata.creationTimestamp
kubectl -n vault get deployments
kubectl -n vault logs deployment/vault-agent-injector --tail=80
kubectl get mutatingwebhookconfigurations
kubectl -n vault-demo describe pods -l app=webapp
kubectl -n vault-demo logs deployment/webapp -c vault-agent --tail=80
```

Release-derived names can differ across chart versions; discover actual names
before copying a log command. If the webhook's failure policy allows admission
to proceed during an injector outage, a Pod might be admitted **without**
injection. Check actual containers and the file; a Ready app alone is not proof
that a secret was delivered. Production should deliberately choose and test
admission failure behavior.

Never post raw `init.json`, token lookup output, JWTs, rendered secret files, or
unreviewed logs when asking for help. Share the sanitized error, relevant
resource names, versions, and the layer at which the failure occurs.
