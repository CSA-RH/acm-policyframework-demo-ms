# ACS (Red Hat Advanced Cluster Security) — ACM Policy Deployment

This directory deploys **RHACS** across an ACM-managed fleet using the
**PolicyGenerator** plugin.  Central runs on the hub; every managed cluster
gets the operator and a SecuredCluster instance that phones home to Central.

## Directory layout

```
acs/base/
├── kustomization.yaml              Kustomize entry-point
├── policy-generator-config.yaml    PolicyGenerator — defines all 4 policies
├── placement-hub.yml               Placement → local-cluster
├── placement-managed.yml           Placement → environment ∈ [dev, prod]
│
├── operator-namespace.yml          Namespace  rhacs-operator   (shared)
├── stackrox-namespace.yml          Namespace  stackrox         (shared)
├── operatorpolicy.yml              OperatorPolicy / Subscription (shared)
│
├── hub/
│   ├── central.yml                 Central CR (route-exposed, scanner on)
│   ├── init-bundle-rbac.yml        SA + Role + RoleBinding for the Job
│   └── init-bundle-job.yml         Job that calls the Central API to create
│                                   an init bundle and applies the secrets
└── managed/
    ├── securedcluster.yml          SecuredCluster CR (hub-templated)
    └── init-bundle-secrets.yml     3 TLS secrets copied from hub via
                                    {{hub … hub}} templates
```

## Policies and execution order

The PolicyGenerator produces four policies with `extraDependencies` that
enforce a safe ordering:

```
┌─────────────────────────┐
│  1. acs-hub-central     │  Hub — operator + Central
└────────────┬────────────┘
             │ depends (Compliant)
┌────────────▼────────────┐
│  2. acs-hub-init-bundle │  Hub — RBAC + Job → creates TLS secrets
└────────────┬────────────┘
             │ depends (Compliant)
┌────────────▼──────────────────┐     ┌──────────────────────────────────┐
│  4. acs-managed-init-bundle   │     │  3. acs-managed-securedcluster   │
│  Managed — copies TLS secrets │     │  Managed — operator + SC         │
│  from hub via hub templates   │     │  (no dependency — starts early)  │
└───────────────────────────────┘     └──────────────────────────────────┘
```

| # | Policy | Placement | Description |
|---|--------|-----------|-------------|
| 1 | `acs-hub-central` | `acs-hub-placement` → `local-cluster` | Installs the RHACS operator and deploys Central (route-exposed, Scanner enabled). |
| 2 | `acs-hub-init-bundle` | `acs-hub-placement` → `local-cluster` | Creates a ServiceAccount, Role, RoleBinding, and a **Job** that waits for Central, calls the `/v1/cluster-init/init-bundles` API, and applies the resulting TLS secrets (`collector-tls`, `sensor-tls`, `admission-control-tls`) in the `stackrox` namespace. |
| 3 | `acs-managed-securedcluster` | `acs-managed-placement` → `environment ∈ [dev, prod]` | Installs the RHACS operator and deploys a SecuredCluster CR. The `clusterName` and `centralEndpoint` are resolved at deploy-time via `{{hub .ManagedClusterName hub}}` and a `lookup` on the Central route. |
| 4 | `acs-managed-init-bundle` | `acs-managed-placement` → `environment ∈ [dev, prod]` | Copies the three init-bundle TLS secrets from the hub `stackrox` namespace to each managed cluster using `{{hub … hub}}` templates. Depends on policy 2 being Compliant (secrets must exist on the hub first). |

## Prerequisites

- **ACM 2.9+** with the PolicyGenerator plugin enabled in ArgoCD
  (the `argocd-policygenerator` plugin must be registered).
- The hub's global pull-secret must include credentials for
  `registry.redhat.io` (standard on OpenShift clusters).
- Managed clusters must have the label `environment: dev` or
  `environment: prod` to be targeted by the managed-cluster placement.

## How it works — init bundle flow

1. **Policy 1** creates the `rhacs-operator` namespace, installs the operator
   via OLM (`OperatorPolicy`), creates the `stackrox` namespace, and deploys
   the `Central` CR with an OpenShift route.

2. **Policy 2** waits for policy 1 to be Compliant (via `extraDependencies`),
   then creates a Kubernetes **Job** on the hub.  The Job:
   - Polls `https://central.stackrox.svc:443/v1/ping` until Central is ready
     (up to 30 × 20 s = 10 min).
   - Checks whether an init bundle named `acm-init-bundle` already exists.
   - If not, calls `POST /v1/cluster-init/init-bundles` with the admin
     password from the auto-generated `central-htpasswd` secret.
   - Decodes the `kubectlBundle` field (base64-encoded YAML) and applies
     the three resulting secrets to the `stackrox` namespace via `oc apply`.

3. **Policy 3** runs in parallel — it installs the operator and creates the
   `SecuredCluster` CR on every managed cluster.  The CR references the
   Central route via a hub template lookup, and the cluster name is set to
   `{{hub .ManagedClusterName hub}}`.

4. **Policy 4** waits for policy 2 to be Compliant, then uses hub templates
   to copy the three TLS secrets (`collector-tls`, `sensor-tls`,
   `admission-control-tls`) from the hub's `stackrox` namespace to each
   managed cluster.  Once these secrets land, the SecuredCluster pods
   authenticate to Central and the cluster appears in the ACS console.

## Resource configuration

Central and its database have explicit CPU/memory requests and limits
(sourced from the reference workshop setup):

| Component | Requests | Limits |
|-----------|----------|--------|
| `central` | 500 m CPU, 2 Gi RAM | 2 CPU, 8 Gi RAM |
| `central.db` | 500 m CPU, 2 Gi RAM | 4 CPU, 8 Gi RAM |

Scanner is set to auto-scale between 1 and 3 replicas.

The SecuredCluster CR enables **hard admission-control enforcement**
(`enforcement: Enabled`), scans unknown images before admitting them
(`contactImageScanners: ScanIfMissing`), tolerates taints on all nodes,
and enables the delegated scanner in `AutoSense` mode.  The deprecated
`listenOnCreates` / `listenOnUpdates` / `listenOnEvents` fields are not
used — RHACS 4.9+ manages listener behaviour through `enforcement`.

## Shared manifests

`operator-namespace.yml`, `stackrox-namespace.yml`, and `operatorpolicy.yml`
are referenced by both the hub and managed policies.  Each reference uses a
unique `name` (e.g. `acs-hub-operator-ns` vs `acs-managed-operator-ns`) so
the generated ConfigurationPolicies do not collide.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `acs-hub-central` stays NonCompliant | `oc get operatorpolicy -n acm-policies` — verify CatalogSource health and operator CSV status. |
| Init-bundle Job keeps failing | `oc logs job/create-cluster-init-bundle -n stackrox` — the Job retries up to 10 times (`backoffLimit: 10`). Common cause: Central not ready within the 10-min window. |
| `acs-managed-init-bundle` stays NonCompliant | The hub-template lookups return empty until the secrets exist on the hub. Verify the Job completed: `oc get secret collector-tls sensor-tls admission-control-tls -n stackrox` on the hub. |
| SecuredCluster pods in CrashLoop on managed cluster | Usually means the init-bundle secrets have not arrived yet. Check policy 4 compliance and verify the secrets exist in `stackrox` on the managed cluster. |
| Central route not resolving in hub template | `oc get route central -n stackrox` — the route must exist before managed-cluster policies can resolve `centralEndpoint`. |

## Re-creating the init bundle

If the init bundle needs to be rotated:

```bash
# 1. Revoke the old bundle in the ACS UI or via API
# 2. Delete the Job so the policy recreates it
oc delete job create-cluster-init-bundle -n stackrox

# 3. Delete the old secrets on the hub
oc delete secret collector-tls sensor-tls admission-control-tls -n stackrox

# The ACM policies will re-converge:
#   - Hub: Job runs again → new secrets
#   - Managed: hub-template policies push updated secrets
```
