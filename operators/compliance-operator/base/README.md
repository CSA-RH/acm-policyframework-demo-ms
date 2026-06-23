# Compliance Operator

Installs the OpenShift Compliance Operator and configures a CIS benchmark scan with ACM integration.

## Dependencies
  - None

## Details
ACM Minimal Version: 2.12

Documentation: [latest](https://docs.redhat.com/en/documentation/openshift_container_platform/latest/html-single/security_and_compliance/compliance-operator#compliance-operator-installation)

## Policies

| Policy | Remediation | What it does |
| --- | --- | --- |
| `compliance-operator` | enforce | Installs the Compliance Operator via OLM (`openshift-compliance` namespace, privileged pod-security label, OperatorGroup, Subscription) |
| `compliance-cis-scan` | inform | Deploys the CIS ScanSetting + ScanSettingBinding, checks ComplianceSuite completion, and reports failed ComplianceCheckResults to ACM |

## CIS Scan Profiles

The CIS scan runs against three profiles:

- **`ocp4-cis`** — OpenShift CIS benchmark (platform-level controls)
- **`ocp4-cis-node`** — OpenShift CIS benchmark (node-level controls)
- **`rhcos4-moderate`** — RHCOS moderate security profile

Scans run hourly (`0 */1 * * *`) on both `master` and `worker` roles.

## ACM Integration

The `compliance-cis-scan` policy integrates scan results into the ACM Governance dashboard through two ConfigurationPolicies:

1. **`cis-suite-status`** — checks that the `cis-compliance` ComplianceSuite has reached `phase: DONE`. Non-compliant while the scan is still running or if the suite does not exist yet.
2. **`cis-results`** — uses `mustnothave` to flag any `ComplianceCheckResult` with label `check-status: FAIL` for the `cis-compliance` suite. Each failing CIS control appears as a policy violation in ACM.

---
**Notes:**
  - Requires the namespace to have the `privileged` pod-security label (included in `manifests/namespace.yml`).
  - The `OperatorPolicy` uses a static `nodeSelector` targeting `node-role.kubernetes.io/worker`. If your clusters have dedicated infra nodes, change the nodeSelector in `manifests/operatorpolicy.yml`.
  - The `complianceConfig` in the `OperatorPolicy` must include all controller-defaulted fields (including `deprecationsPresent: Compliant`), otherwise the governance agent enters an infinite reconciliation loop.
  - The ScanSetting does not specify a `storageClassName` — the cluster default is used. Uncomment and set `storageClassName` in `manifests/scan/scansetting-cis.yml` if your cluster has no default StorageClass.
  - NERC CIP scan manifests are included under `manifests/scan/` but commented out in `policy-generator.yml`.
  - Based on [bry-tam/acm-policy-samples](https://github.com/bry-tam/acm-policy-samples/tree/main/policies/operators/compliance-operator) and the [ACM policy-collection CIS scan pattern](https://github.com/open-cluster-management-io/policy-collection/blob/main/stable/CM-Configuration-Management/policy-compliance-operator-cis-scan.yaml).
