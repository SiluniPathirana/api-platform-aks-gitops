# Publishing an existing gateway API into the Portal catalog

The Gateway (`RestApi`, `apiVersion: gateway.api-platform.wso2.com/v1alpha1`)
and the Portal catalog (`RestApi`, `apiVersion:
api-portal.api-platform.wso2.com/v1`) are two **completely separate**
systems with two different databases (`gateway_controller` vs
`platform_api`) and two different YAML schemas that happen to share a
`kind` name. Deploying an API to the gateway does **not** make it appear
in the Portal -- they need to be published to independently. This
directory does that for `aks-test-api`
(`../../gateway-operator-aks/02-restapi-aks-test-api.yaml`).

Confirmed by reading the actual OpenAPI specs bundled in each component's
container image -- not guessed:
```bash
kubectl exec -n default <platform-api-pod> -- cat ./resources/openapi.yaml
kubectl exec -n default <api-portal-ui-pod> -- cat /app/docs/api-portal-openapi-spec-v0.9.yaml
```

## What's in this folder

| File | Purpose |
|---|---|
| `aks-test-api.manifest.yaml` | The Portal-side API manifest (`api.yaml`) |
| `aks-test-api.definition.yaml` | Its OpenAPI definition (GET /ping, GET /echo -- matching the gateway-side operations exactly) |

Neither file is a Kubernetes resource and neither is applied with
`kubectl` -- both are uploaded as plain files via `curl` multipart form
data to the Portal's own management API (`/api-portal/api/v0.9/apis`,
served by the `api-portal-ui` component itself -- confirmed via its
bundled OpenAPI spec, not platform-api's `/rest-apis`, which is a related
but different endpoint used by a different flow).

`endpoints.productionUrl`/`sandboxUrl` in the manifest point at the real,
already-deployed gateway endpoint -- confirmed live before publishing:
```bash
curl -sk https://wk-onboarding-gw.eastus.cloudapp.azure.com:8443/aks-test/ping
# -> 200
```
This is what makes a Portal-issued subscription actually reach the real
gateway deployment, rather than publishing a second, disconnected copy.

## Prerequisite: Platform API admin credentials

Only a bcrypt hash of the admin password exists in `api-portal-platform-api-secrets`
(`APIP_CP_ADMIN_PASSWORD_HASH`) -- the plaintext is never stored, by
design (same as every other credential in this engagement: printed once,
never recoverable). Reset it the same way as any other "lost" credential
here -- generate a new password + bcrypt hash, patch the Secret, restart:

```bash
NEW_PW=$(openssl rand -base64 32 | tr -dc 'A-Za-z0-9' | head -c 24)
NEW_HASH=$(python3 -c "import bcrypt,sys; print(bcrypt.hashpw(sys.argv[1].encode(), bcrypt.gensalt()).decode())" "$NEW_PW")
echo "SAVE THIS SECURELY, do not paste back into chat: $NEW_PW"

NEW_HASH_B64=$(printf '%s' "$NEW_HASH" | base64 -w0)   # -w0, not -i (macOS's -i flag doesn't exist on Linux base64 --
                                                         # using it silently produces an EMPTY value here, which then
                                                         # gets patched in as the new hash and breaks auth entirely,
                                                         # confirmed the hard way)
kubectl patch secret api-portal-platform-api-secrets -n default --type=json \
  -p "[{\"op\":\"replace\",\"path\":\"/data/APIP_CP_ADMIN_PASSWORD_HASH\",\"value\":\"$NEW_HASH_B64\"}]"
kubectl rollout restart deployment api-portal-aks-platform-api -n default
kubectl rollout status deployment api-portal-aks-platform-api -n default
```

## Prerequisite: subscription plans must already exist

`spec.subscriptionPlans` in the manifest **links** to existing org-level
plans -- it does not create them. `api-portal-ui`'s own
`config.organization.autoCreateSubscriptionPlans: true` did **not**
actually seed anything for this org (confirmed: `GET
/subscription-plans` returned `count: 0` before these were created
explicitly) -- that flag evidently does something else, or triggers
under different conditions than assumed. Create the plans first:

```bash
kubectl port-forward -n default svc/api-portal-aks-platform-api 19243:9243 &

TOKEN=$(curl -sk -X POST "https://localhost:19243/api/portal/v0.9/auth/login" \
  -d "username=admin&password=$NEW_PW" | python3 -c "import json,sys; print(json.load(sys.stdin)['token'])")

curl -sk -X POST "https://localhost:19243/api/v0.9/subscription-plans" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"bronze","displayName":"Bronze"}'

curl -sk -X POST "https://localhost:19243/api/v0.9/subscription-plans" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"id":"gold","displayName":"Gold"}'
```

**Note the ID vs name distinction** -- easy to get backwards (did, on the
first publish attempt, and got a generic `RESOURCE_NOT_FOUND` rather
than an obviously-scoped error): `POST /subscription-plans` takes a
lowercase `id` slug, but the manifest's `spec.subscriptionPlans` links by
**displayName** (`"Bronze"`, `"Gold"` -- capitalized, matching what was
passed to `displayName` above), not by `id`. Confirmed by reading
`api-portal-ui`'s own OpenAPI spec description for `POST /apis` directly.

## Publishing

```bash
TOKEN=$(curl -sk -X POST "https://localhost:19243/api/portal/v0.9/auth/login" \
  -d "username=admin&password=$NEW_PW" | python3 -c "import json,sys; print(json.load(sys.stdin)['token'])")

curl -sk -X POST "https://wk-onboarding-portal.eastus.cloudapp.azure.com:9543/api-portal/api/v0.9/apis" \
  -H "Authorization: Bearer $TOKEN" \
  -F "metadata=@aks-test-api.manifest.yaml;filename=api.yaml;type=application/yaml" \
  -F "definition=@aks-test-api.definition.yaml;filename=definition.yaml;type=application/octet-stream"
```

The uploaded filenames matter, not just the form field names -- the
server validates the multipart part's filename against an allowlist
(`api.yaml`, `metadata.yaml`, `metadata.yml`, `metadata.json`,
`mcp.yaml`) and rejects anything else with `COMMON_VALIDATION_ERROR`,
regardless of what the actual file on disk is called. Hence
`filename=api.yaml` overriding this directory's more descriptive
`aks-test-api.manifest.yaml` name in the upload itself.

A successful publish returns the created API's metadata with
`"status":"PUBLISHED"`. Verify:
```bash
curl -sk "https://wk-onboarding-portal.eastus.cloudapp.azure.com:9543/api-portal/api/v0.9/apis?name=AKS%20Test%20API" \
  -H "Authorization: Bearer $TOKEN"
```
Or check the catalog page directly:
`https://wk-onboarding-portal.eastus.cloudapp.azure.com:9543/api-portal/default/views/default`

## What this does NOT do yet

Publishing the API makes it **discoverable and subscribable** in the
Portal. It does not itself create a subscription -- that's a consumer
action (create an Application, subscribe to this API + a plan, get an
API key) normally done self-service through the Portal UI, not something
that belongs in this same publish step. See "Manage Subscriptions" in
the official docs for that flow if scripting it is needed later.
