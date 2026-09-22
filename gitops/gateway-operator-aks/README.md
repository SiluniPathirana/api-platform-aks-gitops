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
# following the docs page for 1.2.0: image.tag=0.8.1). gateway.helm.chartVersion
# is the Operator's OWN configurable value naming which internal "gateway"
# chart version it deploys for every APIGateway it manages -- defaults to
# 1.1.0, which has NO real Postgres support (see "Postgres migration"
# below for why that matters). Pinned to 1.2.3 (latest available as of
# this setup) here so a fresh install doesn't need a follow-up upgrade.
helm install my-gateway-operator oci://ghcr.io/wso2/api-platform/helm-charts/gateway-operator \
  --version 0.8.0 --set image.tag=0.8.1 --set gateway.helm.chartVersion=1.2.3

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

**Long-term fix: done -- storage migrated to Postgres.** The path there
had more real product gaps than expected. In order:

**1. `spec.storage` on the CRD looks right but does nothing.**
`type: postgres` + `connectionSecretRef` is a real, properly-typed
`APIGateway` field -- confirmed via `kubectl explain
apigateway.spec.storage --recursive` (`type`, `connectionSecretRef.name`,
`connectionSecretRef.key`), and setting it genuinely bumps the CR's
`generation` and triggers a real Helm upgrade. But empirically confirmed
(via `helm get values`, on **both** chart `1.1.0` and `1.2.3`) that the
Operator's CR-to-values translation never actually populates the chart's
storage fields from it -- the release silently stays on `sqlite`
regardless. Distinct from the `fsGroup` bug above: there the field gets
corrupted; here it's just dropped. `spec.storage` is deliberately **left
unset** in `00-apigateway.yaml` -- Postgres is instead configured
directly via `01-gateway-custom-config.yaml`'s `configRef` ConfigMap
(`gateway.config.controller.storage.*` + `gateway.controller.postgres.
passwordSecretRef`), the same proven workaround pattern as `fsGroup` and
the `gatewayRuntime` scaling.

**2. Chart `1.1.0` has no real Postgres implementation at all.** Its
`values.yaml` marks Postgres `(future support)`, with only a `sqlite:`
sub-block actually defined -- so even bypassing the CRD bug, `1.1.0`
can't do it. Chart `1.2.3` (latest available) is the first version that
implements it for real
(`gateway.config.controller.storage.postgres.{host,port,database,user,
sslmode}` + a separate `gateway.controller.postgres.passwordSecretRef`
so the password stays out of the plaintext ConfigMap). The Operator
hardcodes which internal chart version it deploys, but that's just a
configurable value on the **already-installed** Operator release --
`gateway.helm.chartVersion` -- so no Operator version upgrade was
needed, just:
```bash
helm --kube-context wk-onboarding-aks -n default upgrade my-gateway-operator \
  oci://ghcr.io/wso2/api-platform/helm-charts/gateway-operator --version 0.8.0 \
  --reuse-values --set gateway.helm.chartVersion=1.2.3
```
followed by a `configRevision` bump on `cluster-gw` to force a reconcile
onto it.

**3. Postgres 15+ permission gotchas.** Creating a dedicated
least-privilege `gateway` role and granting `ALL PRIVILEGES` on its
tables (the standard first pass) was not enough:
- `permission denied for schema public` -- since Postgres 15, `CREATE`
  on the `public` schema is no longer granted to non-owner roles by
  default (the controller re-runs its own schema-init routine on every
  startup, not just once). Fix: `GRANT USAGE, CREATE ON SCHEMA public TO
  gateway;`
- `must be owner of table artifacts` -- `GRANT` covers DML, not `ALTER
  TABLE`; only an object's owner can alter it, and the tables were
  created by `gwadmin`. Fix: `REASSIGN OWNED BY gwadmin TO gateway;`
  (run once against `gateway_controller`, transfers ownership of
  everything `gwadmin` owns there to `gateway`).

**4. After the switch, existing API config doesn't carry over
automatically.** The controller's `artifacts` table starts genuinely
empty on the new Postgres database -- it isn't repopulated from the live
`RestApi`/`APIGateway` Kubernetes resources just because the storage
backend changed. `curl` against `aks-test-api` returned `404` until its
`RestApi` CR was deleted and recreated (ArgoCD's `selfHeal` reapplied it
from git), which forced the Operator to genuinely re-push it into the
controller's REST API -- confirmed via a real row appearing in
`artifacts` and the endpoint returning `200` again. A plain annotation
bump was not enough to trigger this; only a real delete+recreate did.

**Final working configuration:**
- Server: `wk-onboarding-pg.postgres.database.azure.com` (Azure Postgres
  Flexible Server, Burstable `Standard_B1ms`, `eastus2`,
  `wk-onboarding-rg`), `--public-access 0.0.0.0` (Azure-internal traffic
  only -- not open to the public internet; all `psql` work here was run
  from temporary pods inside the AKS cluster for this reason, never from
  a local machine).
- Database: `gateway_controller`. App-level role: `gateway`
  (least-privilege, distinct from the `gwadmin` server admin superuser;
  owns its own tables per point 3 above).
- Schema: applied from the local distribution's
  `resources/gateway-controller/db-scripts/gateway-controller-db.postgres.sql`
  (nothing bundled in any pulled `gateway` chart's `files/` directory).
- Connection: a `gateway-db-connection` Secret holds both a `dsn` key
  (full connection string, used for manual verification) and a
  `password` key (used by `gateway.controller.postgres.passwordSecretRef`)
  -- created out-of-band, never in git.
- A known lingering cosmetic issue: `cluster-gw-gateway-controller-data`
  (the old SQLite PVC) was not automatically pruned by the Operator's
  Helm upgrade even though the chart's PVC template is conditional on
  `storage.type == "sqlite"` -- Helm normally prunes resources dropped
  from a chart's render, so this suggests the Operator's Helm invocation
  doesn't do full three-way-merge pruning. Harmless (unused, unmounted)
  but safe to delete manually: `kubectl delete pvc
  cluster-gw-gateway-controller-data`.

The manual `fsGroup` patch above is now obsolete **for the controller**
specifically -- Postgres removes its need for local SQLite file storage,
so the PVC/ownership problem doesn't apply to it anymore. (It was never
relevant to the `gatewayRuntime` router pods, which have no persistent
storage of their own.)

Three bugs found while working through this (CRD version drift, the
`fsGroup` merge bug, and `spec.storage` silently not translating to Helm
values) have draft GitHub issues written up, not yet posted.

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
