# security — current setup

## What lives here

Cluster-wide security tooling — runtime vulnerability scanning, policy enforcement, threat detection. Most are Pattern A (every workload cluster runs its own).

AppSets that drive deployment: [`../argo-applications/sre/security/`](../argo-applications/sre/security/).

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **trivy-operator** | ✅ live on mgmt-forge | every workload cluster | A | Generates `VulnerabilityReport` CRDs per running pod image |
| **OPA / Gatekeeper** | 📋 planned | every workload cluster | A | Admission policies (no privileged pods, image-from-Harbor-only, etc.) |
| **Falco** | 📋 planned | every workload cluster | A | eBPF-based runtime threat detection |

## Dependencies

- **Upstream**: cluster-local Pattern A baseline (`platform/`) for cert-manager + ingress-nginx — Trivy operator's UI uses both.
- **Downstream**: DefectDojo (`search/`) consumes Trivy's `VulnerabilityReport` CRDs as a data source. SIEM (Wazuh) ingests Falco events.

## Architectural decisions

- **Trivy at TWO points**: trivy-operator scans *running* pods cluster-side; Harbor's built-in Trivy scans *images at rest* in the registry. Different threat models — runtime drift vs. supply-chain. Both are useful.
- **OPA over Kyverno** (when deployed): both are valid. OPA is more general-purpose (you can write Rego for non-k8s things too). Kyverno has nicer YAML-native syntax. Choosing OPA aligns with "platform engineering" patterns where central policy bundles are served to many clusters.
- **Falco runtime detection over Wazuh agent for k8s threats**: Wazuh focuses on host-level audit; Falco focuses on syscall-level container-aware detection. Use both — they don't overlap.

## Known issues

- trivy-operator's `trivy-operator-trivy-config` ConfigMap rewrites itself (DB checksums, last-update timestamps). ArgoCD reports drift forever; mitigated via `ignoreDifferences` on `/metadata/annotations`.
- The Trivy DB download can hit GitHub rate limits during cluster bootstrap. Workaround: pre-warm a mirror in Harbor.
