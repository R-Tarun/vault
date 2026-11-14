# 01 - Deploy, initialize, and unseal

## 1. Review and install

Run from the repository root, after lab 00. This is a new installation, not an
upgrade procedure. **If your lab is already initialized and Ready, do not
repeat installation or initialization.** Go to the completion checklist.

The commands here use the Vault CLI **inside the server Pod**. No local Vault
CLI or WSL Vault server is required for this lab.

```bash
kubectl config use-context kind-platform-lab
VAULT_CHART_VERSION="$(cat .local/chart-version)"
helm show chart hashicorp/vault --version "$VAULT_CHART_VERSION"
helm template vault hashicorp/vault \
  --version "$VAULT_CHART_VERSION" \
  --namespace vault \
  -f helm/vault-values.yaml > .local/rendered-vault.yaml
```

The recorded walkthrough used chart `0.34.1`, which declares Vault `2.0.4`.
If you skipped saving the version during the interactive walkthrough, complete
lab 00's version-recording step first. Do not run Helm install again just to
record a version.

Inspect the rendered resources before installing. Confirm the server is **not**
in dev mode, the server has PVC-backed data and audit volumes, and there is no
public LoadBalancer/Ingress. Rendered manifests can contain webhook TLS material;
keep this output local even though there are no Vault secrets in values.

```bash
kubectl create namespace vault
kubectl label namespace vault istio-injection=disabled
helm install vault hashicorp/vault \
  --version "$VAULT_CHART_VERSION" \
  --namespace vault \
  --kube-context kind-platform-lab \
  -f helm/vault-values.yaml
kubectl -n vault get pods,pvc,svc
```

Do **not** add `--wait` yet: a newly installed Vault is uninitialized/sealed and
its readiness probe should fail. Wait for `vault-0` to be Running, even though
it is not Ready. Both the data and audit PVCs should be Bound.

```bash
kubectl --context kind-platform-lab -n vault exec vault-0 -- vault status
```

Expected: `Initialized: false`, `Sealed: true`. `vault status` returns exit code
2 for a sealed server; `kubectl exec` reports this as `command terminated with
exit code 2`. It does not mean installation failed.

## 2. Initialize exactly once

Disable shell tracing first. The JSON file below contains **both the root token
and your only unseal key**. It is intentionally a local learning shortcut, not a
production storage pattern.

```bash
set +x
umask 077
mkdir -p .local
chmod 700 .local

(
  set -o noclobber
  kubectl --context kind-platform-lab -n vault exec vault-0 -- \
    vault operator init \
      -key-shares=1 \
      -key-threshold=1 \
      -format=json > .local/init.json
)
```

The redirection is performed by your **local shell**, so the file is in your
repository's ignored `.local/` directory, not inside the Pod. `umask 077`
creates the file readable/writable only by its owner on filesystems honoring
Unix permissions. On WSL-mounted Windows drives, also restrict access using
Windows permissions; `chmod` alone may not provide the intended isolation.

`noclobber` refuses to overwrite an existing file. **If any initialization
command fails, stop and diagnose.** A failed request can leave an empty or
incomplete file, and a lost response does not prove the server stayed
uninitialized. Check status and preserve any existing material rather than
retrying blindly. Vault refuses to initialize an already-initialized backend.

Open the local JSON file privately and copy `unseal_keys_b64[0]`. Run:

```bash
kubectl --context kind-platform-lab -n vault exec -it vault-0 -- \
  vault operator unseal
```

Paste the key at the hidden prompt, not as a CLI argument or in chat. This
single-share lab needs only one submission. Then verify:

```bash
kubectl --context kind-platform-lab -n vault exec vault-0 -- vault status
kubectl --context kind-platform-lab -n vault wait \
  --for=condition=Ready pod/vault-0 --timeout=120s
kubectl --context kind-platform-lab -n vault get pods,pvc
```

Expected: initialized `true`, sealed `false`, storage type `file`, one Ready
server and one Ready injector, with two Bound 1Gi PVCs. Initialization and
unsealing do **not** require a root-token login.

## 3. Optional UI access

In a dedicated terminal, keep this running:

```bash
kubectl --context kind-platform-lab -n vault port-forward \
  --address 127.0.0.1 service/vault 8200:8200
```

Open `http://127.0.0.1:8200/ui/`, choose **Token** authentication, and use
`root_token` from the private JSON file. Do not save it in the browser,
screenshots, shell history, or chat. Root access is for this lab's bootstrap
only, not for applications.

If port 8200 is already in use, choose a free local port and adjust the URL.
Never bind the forward to `0.0.0.0` for this plaintext lab. Stop it with Ctrl+C
when finished. Stopping a forward does not stop or seal Vault.

## 4. Understand the lifecycle

- **Initialize** creates the encryption/root key hierarchy for this storage.
- **Unseal** reconstructs the unseal key so Vault can access encrypted storage.
- **Authenticate** establishes an identity and issues a token with policies.
- **Authorize** evaluates those policies on each request.
- Restarting this manually-unsealed server means it must be unsealed again.
  A successful Kubernetes scheduling event is not an unseal operation.

## 5. Completion checklist and recorded result

- [ ] `vault-0` and the injector are `1/1 Running`.
- [ ] Both `data-vault-0` and `audit-vault-0` are Bound, 1Gi each.
- [ ] Vault status reports initialized `true` and sealed `false`.
- [ ] The unseal key and root token are saved privately, outside Git tracking.
- [ ] You can explain initialize vs unseal vs authenticate.

During the walkthrough, the user shared Vault `2.0.4` status showing file
storage and HA disabled before initialization, then both Pods `1/1 Running`
after initialization/unseal. Both PVCs were reported Bound at 1Gi. UI login,
restart recovery, secret operations, and later labs have not yet been verified
in the shared output. No cluster commands were executed by the assistant.

## 6. Before lab 02: prepare the local client

The remaining guides use a local CLI through the loopback port-forward. Once
you install a compatible local Vault CLI and `jq`, verify:

```bash
vault version
jq --version
```

Keep the port-forward from section 3 running. In your working terminal, load
the token without printing it; stop if extraction fails:

```bash
set +x
export VAULT_ADDR=http://127.0.0.1:8200
if VAULT_TOKEN="$(jq -er '.root_token | select(type == "string" and length > 0)' .local/init.json)"; then
  export VAULT_TOKEN
else
  unset VAULT_TOKEN
  printf '%s\n' 'STOP: could not read the root token; do not continue.'
fi
```

Continue only if token extraction succeeded. The token is in your shell
environment; use a trusted local machine. In a new shell, repeat this setup
only when administrative access is needed. Unsetting an environment variable
or closing a shell is **not** token revocation.

Next: [KV, policies, and tokens](02-kv-policies-tokens.md).
