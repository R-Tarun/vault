# 02 - KV v2, least privilege, and token lifecycle

Use your administrative shell from lab 01. Everything here is fake lab data.
Do not use a real database password or other real credential.

## 1. Enable the engine and create versions

```bash
vault secrets enable -path=kv kv-v2
vault kv put -mount=kv webapp/config username=demo-user password=demo-only-v1
vault kv put -mount=kv other/config value=not-for-webapp
vault kv get -mount=kv webapp/config
vault kv put -mount=kv webapp/config username=demo-user password=demo-only-v2
vault kv get -mount=kv -version=1 webapp/config
vault kv get -mount=kv webapp/config
```

Expected: two versions; version 1 has `demo-only-v1`, current has
`demo-only-v2`. `put` replaces the complete secret document; `patch` updates
selected fields when enabled/supported.

```bash
vault kv delete -mount=kv -versions=1 webapp/config
vault kv get -mount=kv -version=1 webapp/config
```

The read should fail because version 1 was soft-deleted. Restore it:

```bash
vault kv undelete -mount=kv -versions=1 webapp/config
vault kv get -mount=kv -version=1 webapp/config
```

Understand, but do not run yet: `kv destroy` permanently removes selected
versions; `kv metadata delete` removes all versions and metadata. Soft deletion
and permanent destruction are different operations.

## 2. Apply and test an exact-path policy

The CLI uses `webapp/config` under mount `kv`, but the KV v2 API/policy path is
`kv/data/webapp/config`. Metadata/listing lives under `kv/metadata/...` and is
not granted by this policy.

```bash
vault policy write webapp policies/webapp.hcl
vault policy read webapp
vault token create -policy=webapp -ttl=15m -format=json > .local/webapp-token.json
chmod 600 .local/webapp-token.json
```

In a **subshell**, test as that token; leaving the subshell preserves your
administrative shell's token:

```bash
(
  VAULT_TOKEN="$(jq -er '.auth.client_token' .local/webapp-token.json)" || exit 1
  export VAULT_TOKEN
  vault token capabilities kv/data/webapp/config
  vault kv get -mount=kv webapp/config
  vault kv put -mount=kv webapp/config username=unauthorized
  vault kv get -mount=kv other/config
  vault kv list -mount=kv webapp
)
```

Expected:

| Operation | Result |
|---|---|
| Capabilities on exact data path | `read` |
| Read app secret | Success |
| Write app secret | Permission denied / HTTP 403 |
| Read other secret | Permission denied / HTTP 403 |
| List metadata | Permission denied / HTTP 403 |

The negative commands deliberately return nonzero. Inspect **each** result, not
only the final shell exit status. A failed login, connection error, or 404 is not
proof that authorization works. Do not run these tests under `set -e`.

The default policy also grants token self-service, such as self-lookup and
self-renewal. It does not grant application secret access. The exact policy
deliberately does not give a nice browsing experience in the UI.

## 3. Revoke and prove revocation

Use the token's accessor, not its token value, in command-line arguments:

```bash
vault token revoke -accessor "$(jq -er '.auth.accessor' .local/webapp-token.json)"
(
  VAULT_TOKEN="$(jq -er '.auth.client_token' .local/webapp-token.json)" || exit 1
  export VAULT_TOKEN
  vault kv get -mount=kv webapp/config
)
```

Expected: permission denied / invalid token, even before its 15-minute TTL
would have expired. If it already expired, create a new test token and repeat
the revoke exercise. Review token TTL, max TTL, renewal, and revocation as
distinct concepts; renewal does not always extend a token indefinitely.

## Checkpoint

Explain why `kv/webapp/config` would be the wrong policy path, why a token with
read permission cannot list, and why deleting a secret version is not the same
as revoking a token.

Next: [Kubernetes authentication](03-kubernetes-auth.md).
