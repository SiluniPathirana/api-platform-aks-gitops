# Gateway Operator on AKS — GitOps via ArgoCD

Kubernetes manifests for running the WSO2 API Platform Gateway (Operator/CRD
mode) on the real Azure AKS cluster `wk-onboarding-aks`, managed by ArgoCD.
Same pattern as `gitops/gateway-operator/` (local Rancher Desktop
validation), now on actual cloud infrastructure -- with a newer chart
version and one extra bug found along the way.

## Prerequisites (once per cluster)

```bash
# Always target the AKS context explicitly -- az aks get-credentials
# silently switches current-context, and this kubeconfig now also has
# rancher-desktop and k3d-openchoreo contexts in it.
kubectl config use-context wk-onboarding-aks

# cert-manager -- the operator's admission webhook needs it
helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 --namespace cert-manager --create-namespace \
  --set crds.enabled=true --wait --timeout 5m

# Gateway Operator itself (newer than the local 0.6.0 validation --
# following the docs page for 1.2.0: image.tag=0.8.1)
helm install my-gateway-operator oci://ghcr.io/wso2/api-platform/helm-charts/gateway-operator \
  --version 0.8.0 --set image.tag=0.8.1

# The AES-256 encryption key Secret (gateway-encryption-keys) --
# NOT managed by ArgoCD/Git, created out-of-band the same way as the
# local setup's 02-aesgcm-key-secret.yaml.example.
```

Confirm the CRDs actually installed, and which API version they serve
(chart `0.8.0` on this cluster **also** only serves `v1alpha1`, not the
`v1` shown in WSO2's docs samples -- the same version-drift issue hit
locally on chart `0.6.0`, now confirmed on two different chart versions):

```bash
kubectl get crd apigateways.gateway.api-platform.wso2.com \
  -o jsonpath='{range .spec.versions[*]}{.name}{" served="}{.served}{"\n"}{end}'
```

## What's in this folder

| File | Purpose |
|---|---|
| `00-apigateway.yaml` | Bootstraps the whole gateway (controller + router + policy engine) |
| `01-gateway-custom-config.yaml` | Helm values override -- see comments for a real Operator merge bug hit here |
| `02-restapi-aks-test-api.yaml` | A minimal API (`GET /aks-test/ping` -> `https://httpbin.org/anything`) to prove the pipeline end to end |

## Known issue: `fsGroup` / PVC ownership (manual patch, not in Git)

Storage is SQLite for now, backed by an Azure Disk PVC. A fresh
`disk.csi.azure.com` volume mounts as `root:root`, but the gateway
controller process runs as `wso2` (UID/GID 10001), so it can't write to
`/app/data` without help.

The natural fix -- setting `podSecurityContext.fsGroup: 10001` via
`01-gateway-custom-config.yaml`'s `configRef` -- triggers a real Operator
bug: numeric leaf keys that aren't already present in the chart's default
`values.yaml` get stringified during the Operator's ConfigMap merge,
which Kubernetes then rejects:

```
json: cannot unmarshal string into Go struct field
PodSecurityContext.spec.template.spec.securityContext.fsGroup of type int64
```

Confirmed via direct chart inspection that this is an Operator bug, not a
chart-template bug (the template is a plain `toYaml` passthrough with no
special handling of `fsGroup`). Also confirmed there's no `initContainer`
/`extraInit` escape hatch in the chart, and no `chartRef`-style field on
`APIGateway` to swap in a patched chart (`kubectl explain
apigateway.spec --recursive` shows no such field -- the Operator hardcodes
its chart).

**Current workaround** -- applied directly against the live Deployment,
intentionally NOT tracked in any YAML here (since it would immediately
hit the same merge bug if it were):

```bash
kubectl --context wk-onboarding-aks patch deployment cluster-gw-gateway-controller \
  --type merge \
  -p '{"spec":{"template":{"spec":{"securityContext":{"fsGroup":10001}}}}}'
```

This must be **reapplied** any time the `APIGateway` is deleted/recreated
(e.g. to force the Operator to retry after `Max retries (10) exceeded`,
which it does not do on its own) or otherwise gets a fresh reconcile that
rebuilds the Deployment from scratch.

**Long-term fix in progress:** migrate storage to Postgres.
`spec.storage` (`type: postgres` + `connectionSecretRef`) is a real,
properly-typed `APIGateway` field -- confirmed via `kubectl explain
apigateway.spec --recursive` -- so it bypasses the ConfigMap-merge bug
entirely. All replicas of this one gateway cluster's controller will
share a single Postgres database (that's the HA coordination mechanism);
a separate `APIGateway` instance would get its own database. Blocked as
of now on an Azure Policy region restriction on `eastus` for
`az postgres flexible-server create` (AKS itself is unaffected by this
policy) -- permission request sent, pending a reply. Once granted:

1. Create the `gateway_controller` DB + `gateway` user.
2. Apply the schema script (check `gateway-1.1.0` chart's `files/`
   directory first; fall back to the local distribution's
   `resources/gateway-controller/db-scripts/gateway-controller-db.postgres.sql`).
3. Create a `gateway-db-connection` Secret holding the DSN
   (`postgres://user:pass@host:5432/db?sslmode=require`).
4. Add `spec.storage` to `00-apigateway.yaml` pointing at that Secret.
5. Verify, then drop the manual `fsGroup` patch above -- Postgres removes
   the need for local SQLite file storage entirely, so the PVC/ownership
   problem goes away rather than needing a fix.

Two bugs found while working through this (CRD version drift, and this
`fsGroup` merge bug) have draft GitHub issues written up, not yet posted.

## ArgoCD Application

```bash
kubectl apply --context wk-onboarding-aks -n argocd -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gateway-operator-aks
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/SiluniPathirana/api-platform-aks-gitops.git
    targetRevision: main
    path: gitops/gateway-operator-aks
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

From here: edit `02-restapi-aks-test-api.yaml` (add an operation, change
the upstream, whatever), `git push`, and watch ArgoCD pick up the diff and
re-apply it -- no manual `kubectl apply` needed. `selfHeal: true` also
means any manual `kubectl edit`/`kubectl patch` against these specific
resources gets reverted back to what's in Git automatically -- **this
does not apply to the `fsGroup` patch above**, since that patch targets
the Operator-managed `cluster-gw-gateway-controller` Deployment directly,
which is not one of the resources this Application tracks.
