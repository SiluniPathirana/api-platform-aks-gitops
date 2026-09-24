# Identity Server on AKS — GitOps via ArgoCD

WSO2 Identity Server 7.3.0-1, on the real Azure AKS cluster
`wk-onboarding-aks`, managed by ArgoCD. Registered here after the fact --
the initial bring-up was done manually (`helm install`/`helm upgrade` by
hand) while chasing several real bugs (chart image-pull-secret bug, image
architecture mismatch, missing Postgres JDBC driver, an Azure Load
Balancer health-probe gotcha) one at a time. This directory is that final
working configuration, moved into Git.

## Prerequisites (once per cluster) -- already done, documented for the record

- ACR (`wkonboardingacr`) with the custom image pushed: `wso2is/is:7.3.0-pg`
  (public `wso2/wso2is:7.3.0`, `--platform linux/amd64`, layered with
  `postgresql-42.7.13.jar` in `repository/components/lib/` -- the official
  image ships no Postgres driver, only H2's).
- `wko-acr-pull` -- a `kubernetes.io/dockerconfigjson` Secret in `default`,
  built from the ACR admin account (interim workaround; `az aks update
  --attach-acr` needs Owner/User Access Administrator on the subscription,
  not available yet -- same gap noted in `gitops/gateway-operator-aks`).
- `keystores` (generic Secret: `primary.p12`, `tls.p12`, `internal.p12`,
  `client-truststore.p12`) and `is-tls` (TLS Secret, from `tls.p12`'s
  exported cert/key) -- both in `default`, created out-of-band via
  `generate-is-keystores.sh` (self-signed mode). Not managed by this
  Application.
- 4 Postgres databases on `wk-onboarding-pg.postgres.database.azure.com`:
  `is_identity`, `is_shared` (also used for the `user` datasource -- the
  chart's own H2 defaults point both at the same physical DB), `is_consent`,
  `is_agent_identity`. Dedicated least-privilege role `wso2is`, same
  Postgres-15-permissions pattern as the gateway's migration
  (`GRANT USAGE, CREATE ON SCHEMA public` + `REASSIGN OWNED BY`).
- NGINX Ingress Controller (community `ingress-nginx` chart, not AKS's
  managed Application Routing addon -- the chart's default Ingress
  annotations are NGINX-specific session-affinity settings, not guaranteed
  to be honored by a different ingress product).

## Known chart bug: image pull secret

`templates/deployment.yaml:249` reads `.Values.wso2.deployment.image.
imagePullSecret` instead of the `.Values.deployment.image.imagePullSecret`
it checks two lines above -- confirmed by pulling the chart source
directly and grepping it, not a config mistake on our side. Worked around
by setting the same value at both paths (see `00-application.yaml`'s
top-level `wso2:` block). If a future chart version fixes this upstream,
the workaround block is harmless to leave in place (an unused key), but
worth removing for cleanliness.

## Known chart bug: unfilled YAML values are silently dangerous

Not a chart bug, but worth recording: an unfilled `<<PLACEHOLDER>>` left
in a keystore password field doesn't fail the Helm install -- it renders
straight into the generated `carbon.xml`, whose literal `<<` breaks XML
parsing at server boot with a fairly opaque
`WstxUnexpectedCharException`. There's no schema validation catching this
earlier in the pipeline. Same risk applies to the `__SET_VIA_KEYVAULT__`
placeholders in this file right now -- see **Secrets** below.

## Secrets — current state and TODO

Unlike `api-portal-aks` / `gateway-operator-aks`, which both reference an
**existing Kubernetes Secret** for their credentials
(`secrets.existingSecret`, `passwordSecretRef`), this chart has no such
mechanism for most of what it needs. `deploymentToml.*` passwords are
plain Helm values baked directly into a ConfigMap. The one exception is
`deployment.secretStore.azure` -- a real Key Vault CSI Driver integration
the chart ships with, but it only covers the **internal keystore
password**, not the other 3 keystore passwords, the DB role password, or
the admin password.

`00-application.yaml` has all 6 of those as `__SET_VIA_KEYVAULT__`
placeholders. **This Application will not deploy successfully as
committed** -- that's deliberate, not an oversight, per the note above
about placeholders silently corrupting `carbon.xml`.

Real values currently exist only in a local, un-committed file on the
operator's machine (never pushed to Git), matching the exact values the
live release was actually installed with.

### To finish this properly

1. Store the 6 real secrets in `wk-onboarding-kv` (already provisioned):
   ```bash
   az keyvault secret set --vault-name wk-onboarding-kv --name is-primary-keystore-password --value "<...>"
   az keyvault secret set --vault-name wk-onboarding-kv --name is-tls-keystore-password --value "<...>"
   az keyvault secret set --vault-name wk-onboarding-kv --name is-internal-keystore-password --value "<...>"
   az keyvault secret set --vault-name wk-onboarding-kv --name is-truststore-password --value "<...>"
   az keyvault secret set --vault-name wk-onboarding-kv --name is-db-role-password --value "<...>"
   az keyvault secret set --vault-name wk-onboarding-kv --name is-admin-password --value "<...>"
   ```
2. Create a Service Principal (or Workload Identity) with `get` access to
   those secrets, and the `nodePublishSecretRef` Secret `deployment.
   secretStore.azure` needs.
3. Set `deployment.secretStore.enabled: true` with the Key Vault name,
   subscription/resource group, tenant ID, and SP App ID -- this covers
   the internal keystore password only.
4. For the other 5 (no chart-native Key Vault path exists for them):
   either extend `deploymentToml.extraConfigs` with WSO2's own cipher-tool
   `$secret{}` references (skipped during initial bring-up, see the
   original engagement notes for why), or accept them as Helm-value
   plaintext but restrict Git repo access accordingly -- a real tradeoff,
   not a clear-cut fix, worth a deliberate decision rather than defaulting
   silently.
5. Replace the `__SET_VIA_KEYVAULT__` placeholders here with the real
   mechanism chosen in steps 3-4, confirm the pod reaches `1/1 Running`
   (`kubectl exec` into it, hit `/api/health-check/v1.0/health` directly --
   don't just trust `STATUS: deployed` from Helm, that only means the
   manifest applied, not that the app booted), **then** flip `syncPolicy`
   to `automated: {prune: true, selfHeal: true}` to match the other two
   Applications in this repo.

## Known issue: Azure Load Balancer health probe (fixed, not yet in Git)

The NGINX Ingress Controller's Service was completely unreachable from
outside the cluster -- not a Kubernetes problem, an Azure Standard Load
Balancer one. Its health probe hit NGINX's `/` path expecting a literal
`200 OK`; NGINX returns `404` for any request without a matching `Host`
header (exactly what a generic probe looks like), so the LB marked the
backend permanently unhealthy and silently dropped all inbound traffic
(no RST, no error -- a pure connection timeout, from both the public
internet and from Azure's own network via Cloud Shell, which is what
made this genuinely hard to isolate from a plain NSG/UDR misconfiguration).

Confirmed via the Load Balancer's own Insights workbook in the Azure
Portal (`Health Probe Status by Frontend Port`: `8080`/`8443` at 99.98%,
`443`/`80` at a flat 0% since the LB's `Degraded` status began) -- the
kind of signal that isn't visible from `az network nsg/lb` CLI commands
alone.

Fix: pointed Azure's health probe at NGINX's own internal `/healthz`
endpoint (always returns `200`, regardless of `Host`) instead of the
real traffic port:

```bash
kubectl annotate svc -n ingress-nginx ingress-nginx-controller \
  service.beta.kubernetes.io/port_80_health-probe_protocol="http" \
  service.beta.kubernetes.io/port_80_health-probe_port="10254" \
  service.beta.kubernetes.io/port_80_health-probe_request-path="/healthz" \
  service.beta.kubernetes.io/port_443_health-probe_protocol="http" \
  service.beta.kubernetes.io/port_443_health-probe_port="10254" \
  service.beta.kubernetes.io/port_443_health-probe_request-path="/healthz" \
  --overwrite
```

This alone wasn't sufficient -- `/healthz` lives inside the ingress-nginx
pod's own network namespace (it isn't `hostNetwork`), and port `10254`
was never declared as an actual Service port, so nothing was reachable
at the node level for Azure's probe to hit. Had to also expose it as a
real Service port so kube-proxy creates the NodePort mapping:

```bash
kubectl patch svc -n ingress-nginx ingress-nginx-controller --type=json -p '[
  {"op": "add", "path": "/spec/ports/-", "value": {"name": "healthz", "port": 10254, "targetPort": 10254, "protocol": "TCP"}}
]'
```

Both changes were applied directly against the live `ingress-nginx-controller`
Service, **not through Git** -- that Service isn't managed by any
ArgoCD Application in this repo (ingress-nginx was installed as a
standalone Helm release, a cluster-wide prerequisite shared by every
component here, not specific to Identity Server). Worth a follow-up: this
repo doesn't currently track the ingress-nginx install at all, so this
fix -- and the whole ingress-nginx configuration -- would need to be
redone by hand if that release were ever deleted and reinstalled. Same
class of "known but not yet GitOps'd" gap as the gateway's manual
`fsGroup` patch.

## What's in this folder

| File | Purpose |
|---|---|
| `00-application.yaml` | The ArgoCD Application itself (chart source, values, sync policy) |

Unlike `gateway-operator-aks` (which manages several separate CRs), this
Application is a single Helm release -- no extra manifests needed.

## Applying this Application

Already applied directly (see repo-level notes); to reapply or on a
fresh cluster:

```bash
kubectl config use-context wk-onboarding-aks
kubectl apply -n argocd -f gitops/identity-server-aks/00-application.yaml
```

Since `syncPolicy` is empty (manual), ArgoCD will show this Application
as registered but `OutOfSync` / likely failing to render (the
`__SET_VIA_KEYVAULT__` placeholders) until the Secrets section above is
resolved. That's expected -- do not manually force-sync until then.
