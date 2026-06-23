# cert-manager (Option D — Shared Manifests with Per-Environment PolicyGenerator)

Installs the OpenShift cert-manager operator and deploys an environment-specific
CA-based `ClusterIssuer`. Shared Kubernetes manifests live in a single
`manifests/` directory; each environment has its own PolicyGenerator that
references them via relative paths plus a local environment-specific
`ClusterIssuer`.

Inspired by the separation of concerns described in
[A Guide to multi-cluster GitOps with Policy Generator and Kustomize](https://www.redhat.com/en/blog/a-guide-to-multi-cluster-gitops-with-policy-generator-and-kustomize).

## Design

```
cert-manager-shared-manifests/
├── manifests/                          ← shared K8s manifests (not deployed directly)
│   ├── base/
│   │   ├── namespace.yml
│   │   ├── cert-manager-namespace.yml
│   │   ├── operatorpolicy.yml
│   │   ├── ca-clusterissuer-secret.yml
│   │   └── trusted-ca-configmap.yml
│   └── health/
│       └── cert-manager-status.yml
├── policies/                           ← deployed by ArgoCD (one app per env)
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── policy-generator-config.yaml
│   │   ├── placement.yml
│   │   └── ca-clusterissuer.yml        ← env-specific
│   └── prod/
│       ├── kustomization.yaml
│       ├── policy-generator-config.yaml
│       ├── placement.yml
│       └── ca-clusterissuer.yml        ← env-specific
└── README.md
```

### How it works

- `manifests/` is a shared library of Kubernetes manifests (operator install,
  namespaces, secrets, health checks). It is **not** deployed by ArgoCD directly.
- Each `policies/{env}/` is a self-contained kustomize root with a
  PolicyGenerator. The PolicyGenerator references the shared manifests via
  relative paths (e.g. `../../manifests/base/namespace.yml`) and its own local
  `ca-clusterissuer.yml` for the environment-specific ClusterIssuer.
- All policy dependencies are **internal** to each environment's PolicyGenerator —
  no cross-ArgoCD-app string coupling.

### Why relative paths work with ArgoCD

ArgoCD clones the full repository before running `kustomize build` on the
application's `path`. The PolicyGenerator plugin resolves manifest `path` fields
using the filesystem relative to its own config file. Since the full repo is
available in the clone, `../../manifests/base/namespace.yml` resolves correctly.

## Dependencies

- None (each environment is self-contained).

## Details

ACM Minimal Version: 2.12

## Policies (per environment)

| Policy | Remediation | What it does |
| --- | --- | --- |
| `{env}-cert-manager-operator` | enforce | Installs cert-manager via OLM, creates monitoring namespace, trusted CA ConfigMap, and CA TLS secret |
| `{env}-cert-manager-clusterissuer` | enforce | Creates a `ClusterIssuer` named `ca-clusterissuer-{env}`. Depends on the operator policy being Compliant |

## ArgoCD integration

The existing `operators-appset.yaml` discovers paths matching
`operators/*/base` and `operators/*/overlays/*`. For Option D, add a third
pattern so the ApplicationSet discovers `policies/dev` and `policies/prod`:

```yaml
directories:
  - path: operators/*/base
  - path: operators/*/overlays/*
  - path: operators/*/policies/*       # ← add this for Option D
```

This creates two ArgoCD Applications:

| ArgoCD App Name | Source Path |
| --- | --- |
| `acm-operators-cert-manager-shared-manifests-dev` | `operators/cert-manager-shared-manifests/policies/dev` |
| `acm-operators-cert-manager-shared-manifests-prod` | `operators/cert-manager-shared-manifests/policies/prod` |

## Comparison with the original `cert-manager`

| Aspect | Original | Option D |
| --- | --- | --- |
| ArgoCD apps | 3 (base + 2 overlays) | 2 (dev + prod, no shared base app) |
| Cross-app dependencies | Yes (overlay → base by policy name) | No (self-contained per env) |
| Shared manifests | Duplicated or coupled by name | Single source in `manifests/`, referenced by path |
| Operator duplication | Single install in base | Policy duplicated per env (targets different clusters) |
| Build step required | No | No |
| Env-specific files | 4 files per overlay | 1 file per env (`ca-clusterissuer.yml`) |
