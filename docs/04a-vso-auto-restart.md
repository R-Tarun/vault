# 04a - Automatically restart a workload with Vault Secrets Operator

This optional lab follows labs 03 and 04. It adds a **separate** Deployment to
compare Vault Secrets Operator (VSO) with Agent injection; it does not change
the Agent-based `webapp`.

```text
Vault KV v2 -> VSO polls -> Kubernetes Secret -> Deployment rollout -> new Pod
```

VSO requests a short-lived JWT for a dedicated ServiceAccount, exchanges it
for a Vault token with the existing read-only `webapp` policy, and syncs the
secret. When the source changes, `rolloutRestartTargets` causes VSO to patch
the Deployment's Pod template with a restart timestamp. Kubernetes performs a
rolling replacement; this is **not** a container restart by Vault Agent.

## Important tradeoffs

- VSO creates a **Kubernetes Secret** containing the fake data. Base64 is not
  encryption. Secret access is governed by Kubernetes RBAC; configure and
  verify appropriate etcd encryption at rest for real deployments. Pods able
  to consume the Secret can access it. This course does not establish those
  protections.
- The VSO chart creates cluster-scoped CRDs and RBAC and can manage resources
  across namespaces. Review the rendered resources and access before installing.
  The operator can obtain ServiceAccount tokens and read/manage Secrets.
- Vault traffic is plaintext HTTP in this isolated lab, including VSO's
  connection. Never use real credentials here.
- Polling is eventual, not instantaneous. `30s` is an educational interval;
  backoff, scheduling, and startup add delay.
- `hmacSecretData: true` enables change comparison; VSO ignores restart targets
  when it is disabled. The HMAC is used for comparison, not to encrypt the
  destination Kubernetes Secret.
- A rollout may interrupt traffic, especially with a single replica. A new
  Pod UID and updated environment show replacement and delivery, not real
  application readiness for rotated credentials or zero downtime.
- Environment variables are fixed at process startup. A new Pod must start
  to consume the updated environment. If the application supports reloading a
  mounted file, avoid unnecessary rollouts.
- Use GitOps to manage manifests, not actual secret values. A GitOps controller
  may reconcile Pod-template annotations; test coexistence and narrowly
  configure drift handling only if necessary. Do not ignore all differences.

## 1. Prerequisites and conflict checks

Finish labs 03 and 04. Vault must be unsealed; `webapp/config` should contain
the fake `demo-only-v3` value from lab 04, and the original Agent workload
should be healthy. Commands below use the current context; confirm it before
doing anything:

```bash
kubectl config current-context
vault status
vault read auth/kubernetes/role/webapp
kubectl -n vault-demo rollout status deployment/webapp
helm list --all-namespaces
kubectl get namespace vault-secrets-operator --ignore-not-found
kubectl get crd vaultconnections.secrets.hashicorp.com \
  vaultauths.secrets.hashicorp.com vaultstaticsecrets.secrets.hashicorp.com \
  --ignore-not-found
kubectl -n vault-demo get deployment,serviceaccount,secret \
  | grep -E 'vso-webapp|vso-webapp-config' || true
```

Stop if the context is not `kind-platform-lab`, or if the VSO namespace,
release, listed CRDs, or lab resource names are already used. Do not reuse,
overwrite, upgrade, or uninstall another team's operator. The chart may install
additional CRDs; inspect rendered output too. A missing grep match is expected.

## 2. Choose and install the operator

The Vault server Helm chart and VSO are **separate charts**. The user-reported
walkthrough used VSO chart **`1.6.2`** with Vault **`2.0.4`** on kind
Kubernetes **`1.37.0`**. That is one lab observation, not a general support
guarantee. Review the chart's release notes and compatibility with your
environment before selecting a version.

```bash
helm repo update hashicorp
helm search repo hashicorp/vault-secrets-operator --versions
read -r -p "Selected VSO chart version: " VSO_CHART_VERSION
if [ -z "$VSO_CHART_VERSION" ]; then
  printf '%s\n' 'STOP: a chart version is required.'
else
  helm show chart hashicorp/vault-secrets-operator --version "$VSO_CHART_VERSION"
fi
```

Stop if the variable is empty, the chart inspection fails, or the version is
unsuitable. Render CRDs and other resources to your ignored local directory:

```bash
umask 077
mkdir -p .local
chmod 700 .local
helm template vault-secrets-operator hashicorp/vault-secrets-operator \
  --version "$VSO_CHART_VERSION" \
  --namespace vault-secrets-operator \
  --include-crds > .local/rendered-vso.yaml
```

Inspect `.local/rendered-vso.yaml` privately, including CRDs, ClusterRoles,
ClusterRoleBindings, images, and install hooks. Check for conflicts with
existing cluster resources. If you did not reserve the namespace exclusively
for this course, stop.

```bash
kubectl create namespace vault-secrets-operator
kubectl label namespace vault-secrets-operator istio-injection=disabled
helm install vault-secrets-operator hashicorp/vault-secrets-operator \
  --version "$VSO_CHART_VERSION" \
  --namespace vault-secrets-operator \
  --kube-context kind-platform-lab
kubectl -n vault-secrets-operator get deployments,pods
kubectl get crd vaultconnections.secrets.hashicorp.com \
  vaultauths.secrets.hashicorp.com vaultstaticsecrets.secrets.hashicorp.com
```

Wait until the operator Deployment and Pod are Ready. If they are not, inspect
their names and events with `kubectl -n vault-secrets-operator describe pod
<pod-name>`; do not continue on a failed install or missing CRDs.

## 3. Bind a separate identity and synchronize

The manifest creates a dedicated `vso-webapp` ServiceAccount and Deployment.
The Deployment may initially be unready because its Secret does not exist until
the operator syncs it. The application Pod does **not** need the ServiceAccount
JWT mounted: VSO requests one through the Kubernetes TokenRequest API.

```bash
kubectl apply -f kubernetes/vso-webapp.yaml
vault write auth/kubernetes/role/vso-webapp \
  bound_service_account_names=vso-webapp \
  bound_service_account_namespaces=vault-demo \
  audience=vault \
  token_policies=webapp \
  token_ttl=15m \
  token_max_ttl=1h
vault read auth/kubernetes/role/vso-webapp
kubectl apply -f kubernetes/vso-resources.yaml
kubectl -n vault-demo rollout status deployment/vso-webapp --timeout=180s
kubectl -n vault-demo get vaultconnection,vaultauth,vaultstaticsecret
kubectl -n vault-demo get secret vso-webapp-config
```

Expected: Deployment Ready and destination Secret present. Do **not** use
`kubectl get secret -o yaml`, print environment variables, or share Secret
contents. The three VSO resources and their references are namespaced in
`vault-demo`. `VaultStaticSecret.spec.path` is relative to mount `kv` and is
`webapp/config`; the Vault **policy** still uses `kv/data/webapp/config`.

Check only whether the demo application process started with the expected fake
value; this command prints no secret:

```bash
kubectl -n vault-demo exec deployment/vso-webapp -c webapp -- \
  sh -c 'test "$DEMO_PASSWORD" = demo-only-v3 && echo "Initial value matches"'
```

Stop if this fails. Inspect events and VSO resources with `kubectl describe`;
logs and event details can contain sensitive metadata, so sanitize them.
Check the `vso-webapp` role, `vault` JWT audience, `webapp` policy, and the
Vault server's TokenReview permission before changing access.

## 4. Rotate and prove automatic replacement

Wait for the initial rollout to settle, then record the old Pod **UID**, not
just its container restart count:

```bash
kubectl -n vault-demo rollout status deployment/vso-webapp --timeout=180s
OLD_POD_UID="$(kubectl -n vault-demo get pods -l app=vso-webapp \
  -o jsonpath='{.items[0].metadata.uid}')"
printf 'Original Pod UID: %s\n' "$OLD_POD_UID"
```

If the UID is empty or more than one Pod exists, stop and wait for the rollout
to settle; do not treat an unrelated rollout as proof. Now update Vault:

```bash
vault kv put -mount=kv webapp/config username=demo-user password=demo-only-v4
```

Wait for polling, sync, and rollout. Inspect the VSO restart annotation and the
Pod list; a single immediate `rollout status` can return before VSO detects the
change, so it is not sufficient proof by itself:

```bash
kubectl -n vault-demo get deployment vso-webapp \
  -o jsonpath='{.spec.template.metadata.annotations.vso\.secrets\.hashicorp\.com/restartedAt}{"\n"}'
kubectl -n vault-demo get pods -l app=vso-webapp \
  -o custom-columns=NAME:.metadata.name,UID:.metadata.uid,PHASE:.status.phase
```

Allow at least two minutes for retries if needed. Once a **new UID** appears
and the old Pod is gone, run:

```bash
kubectl -n vault-demo rollout status deployment/vso-webapp --timeout=180s
NEW_POD_UID="$(kubectl -n vault-demo get pods -l app=vso-webapp \
  -o jsonpath='{.items[0].metadata.uid}')"
printf 'New Pod UID: %s\n' "$NEW_POD_UID"
test -n "$OLD_POD_UID" && test -n "$NEW_POD_UID" && \
  test "$OLD_POD_UID" != "$NEW_POD_UID" && echo 'Pod was replaced'
kubectl -n vault-demo exec deployment/vso-webapp -c webapp -- \
  sh -c 'test "$DEMO_PASSWORD" = demo-only-v4 && echo "Updated value matches"'
```

Pass only if the annotation is present, UID changed, rollout completed, and
the new Pod has the updated value. Confirm the original Agent-based `webapp`
still renders v4 without a VSO-targeted rollout. The two mechanisms are
independent; VSO is configured to restart only `vso-webapp`.

If it fails, check the `VaultStaticSecret` status and events, the operator
Deployment logs, and rollout events. Verify the destination Secret exists and
the operator has permission to patch the target Deployment. Do not grant the
application root Vault access or broad Kubernetes permissions as a workaround.

## 5. Optional lab cleanup

Only remove these resources if they were created for this lab. Do this before
lab 05's namespace/Vault teardown. Deleting the VSO resource or Helm release
can affect other consumers; stop if the operator is shared.

```bash
kubectl config current-context
kubectl -n vault-demo get vaultstaticsecret,vaultauth,vaultconnection
helm -n vault-secrets-operator list
```

After confirming the intended context and ownership:

```bash
kubectl delete -f kubernetes/vso-resources.yaml
kubectl delete -f kubernetes/vso-webapp.yaml
kubectl -n vault-demo get secret vso-webapp-config --ignore-not-found
```

If the Secret remains, and only after confirming it belongs to this lab:

```bash
kubectl -n vault-demo delete secret vso-webapp-config
```

Remove the dedicated role while Vault is available:

```bash
vault delete auth/kubernetes/role/vso-webapp
helm uninstall vault-secrets-operator -n vault-secrets-operator
kubectl delete namespace vault-secrets-operator
```

Inspect the chart's CRDs and remaining cluster-scoped resources. Helm CRD
retention varies by packaging and lifecycle; **do not delete CRDs blindly**:
deleting a CRD also deletes all instances cluster-wide. Delete them only after
checking they are exclusively this lab's and no other consumers remain. Remove
the named rendered artifact if it contains only this lab's data:

```bash
rm -f -- .local/rendered-vso.yaml
unset VSO_CHART_VERSION OLD_POD_UID NEW_POD_UID
```

The KV secret remains at `demo-only-v4`, so lab 05 should expect the latest
version/value rather than assume v3. The original Agent workload and Vault
installation remain available.

## Checkpoint

- [ ] Initial VSO sync produces the destination Secret and starts the Pod.
- [ ] A Vault secret update produces a new Pod UID and a successful rollout.
- [ ] The replacement Pod sees the updated fake value.
- [ ] You can explain the security tradeoff versus Agent's shared file.
- [ ] Optional resources are cleaned up without disrupting the Agent lab.

In the recorded walkthrough the user confirmed initial synchronization, the
VSO restart annotation, a changed Pod UID, a successful rollout, and delivery
of the updated fake value. The optional cleanup was **not** performed.

Reference: [VSO API](https://github.com/hashicorp/vault-secrets-operator/blob/main/docs/api/api-reference.md)
and [VSO documentation](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/vso/sources/vault).
Next: [Operations, recovery, and cleanup](05-operations-and-cleanup.md).
