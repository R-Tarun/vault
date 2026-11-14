# 03 - Authenticate a Kubernetes workload

For an overview of other choices and how auth differs from a secrets engine,
see the [auth methods and secrets engines reference](auth-and-secrets-reference.md).

## 1. Enable and configure Kubernetes auth

Use your administrative shell. This assumes Vault runs **inside the same
cluster** as the workload. It deliberately omits an external reviewer JWT and
CA: Vault reads its mounted ServiceAccount token and CA locally and can reread
the rotated token. Do not paste a long-lived ServiceAccount JWT into Git.

```bash
kubectl auth can-i create tokenreviews.authentication.k8s.io \
  --as=system:serviceaccount:vault:vault
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host=https://kubernetes.default.svc:443
```

Expected: `yes` for TokenReview access. If impersonation itself is forbidden,
use a cluster administrator to inspect this check; do not grant broad rights to
the workload to make the error disappear.

The chart's `authDelegator` binding is for the **Vault server**, not the app.
The app needs no RBAC permission to read Kubernetes Secrets or call TokenReview.

## 2. Create the identity before the Deployment

Create just the namespace and ServiceAccount now, not the annotated app:

```bash
kubectl create namespace vault-demo
kubectl label namespace vault-demo istio-injection=disabled
kubectl -n vault-demo create serviceaccount webapp
```

The full manifest in lab 04 declares the same objects and can apply over them.

Bind a Vault role to the **exact ServiceAccount and namespace**. `webapp` is
both the chosen role name and policy name, but roles and policies are separate
objects. The audience below matches the explicit test JWT and the injector
projected token volume in lab 04:

```bash
vault write auth/kubernetes/role/webapp \
  bound_service_account_names=webapp \
  bound_service_account_namespaces=vault-demo \
  audience=vault \
  token_policies=webapp \
  token_ttl=15m \
  token_max_ttl=1h
vault read auth/kubernetes/role/webapp
```

## 3. Prove JWT-to-token authentication independently of the injector

Create a short-lived JWT with an explicit audience. Keep both JWTs and returned
Vault tokens in private local files:

```bash
umask 077
kubectl -n vault-demo create token webapp \
  --audience=vault --duration=10m > .local/webapp.jwt
vault write -format=json auth/kubernetes/login \
  role=webapp jwt=@.local/webapp.jwt > .local/kubernetes-login.json
jq '.auth | {policies, lease_duration, renewable}' .local/kubernetes-login.json
```

Expected: `webapp` (and normally `default`) among the policies; a nonzero token
TTL. Avoid displaying `.auth.client_token`.

```bash
(
  VAULT_TOKEN="$(jq -er '.auth.client_token' .local/kubernetes-login.json)" || exit 1
  export VAULT_TOKEN
  vault kv get -mount=kv webapp/config
  vault kv get -mount=kv other/config
)
```

Expected: first read succeeds, second is denied with 403.

Now prove identity binding, not merely policy binding:

```bash
kubectl -n vault-demo create token default \
  --audience=vault --duration=10m > .local/wrong-serviceaccount.jwt
vault write auth/kubernetes/login role=webapp jwt=@.local/wrong-serviceaccount.jwt
```

Expected: login is rejected because `default` is not bound to this role. A
successful login here is a failed lab checkpoint: inspect role bindings before
proceeding.

## 4. Revoke the manual test token

From your administrative shell:

```bash
vault token revoke -accessor "$(jq -er '.auth.accessor' .local/kubernetes-login.json)"
```

Deleting the JWT file does not invalidate the JWT. Its expiration limits its
lifetime. Deleting a ServiceAccount or a bound Pod may affect Kubernetes token
validation; do not confuse that with revoking an already-issued Vault token.

## Checkpoint

Explain the difference between a Kubernetes JWT and a Vault token, why
`audience` matters, and how the server's TokenReview identity differs from the
workload's login identity.

Next: [Agent injection](04-agent-injection.md).
