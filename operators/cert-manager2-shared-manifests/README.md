# cert-manager (Shared Manifests with Per-Environment PolicyGenerator)

Installs the OpenShift cert-manager operator and deploys an environment-specific
CA-based `ClusterIssuer`. Shared Kubernetes manifests live in a single
`manifests/` directory; each environment has its own PolicyGenerator that
references them via relative paths plus a local environment-specific
`ClusterIssuer`.

## Design

```
cert-manager-shared-manifests/
├── manifests/                          << shared K8s manifests (not deployed directly)
│   ├── base/
│   │   ├── namespace.yml
│   │   ├── cert-manager-namespace.yml
│   │   ├── operatorpolicy.yml
│   │   ├── ca-clusterissuer-secret.yml
│   │   └── trusted-ca-configmap.yml
│   └── health/
│       └── cert-manager-status.yml
├── policies/                           << deployed by ArgoCD (one app per env)
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── policy-generator-config.yaml
│   │   ├── placement.yml
│   │   └── ca-clusterissuer.yml        << env-specific
│   └── prod/
│       ├── kustomization.yaml
│       ├── policy-generator-config.yaml
│       ├── placement.yml
│       └── ca-clusterissuer.yml        << env-specific
└── README.md
```

### How it works

- Each `policies/{env}/` is a self-contained kustomize root with a
  PolicyGenerator. The PolicyGenerator references the shared manifests via
  relative paths (e.g. `../../manifests/base/namespace.yml`) and its own local
  `ca-clusterissuer.yml` for the environment-specific ClusterIssuer.
- All policy dependencies are **internal** to each environment's PolicyGenerator,
  no cross-ArgoCD-app string coupling.

## Policies (per environment)

| Policy | Remediation | What it does |
| --- | --- | --- |
| `{env}-cert-manager-operator` | enforce | Installs cert-manager via OLM, creates monitoring namespace, trusted CA ConfigMap, and CA TLS secret |
| `{env}-cert-manager-clusterissuer` | enforce | Creates a `ClusterIssuer` named `ca-clusterissuer-{env}`. Depends on the operator policy being Compliant |
