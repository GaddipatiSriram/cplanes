# docs — cross-cutting reference

This folder holds reference material that doesn't belong to a single domain — comparison docs, learning material spanning multiple domains, vendor-specific contrasts.

## Contents

| Path | Topic | Purpose |
|---|---|---|
| [`aws/eks-bootstrap-argo.md`](./aws/eks-bootstrap-argo.md) | AWS EKS lifecycle + ArgoCD registration patterns | Teaching doc — how the big-cloud version of the homelab patterns works, what real EKS shops do with eksctl / Karpenter / IRSA / Pod Identity / Cluster API. |

## When to add a doc here

Use `docs/` for content that:
- **Spans multiple domains** (e.g., a "secrets architecture across forge + platform + workload" piece)
- **Compares against another vendor or stack** (AWS, GCP, vanilla k8s, OpenShift)
- **Is purely educational** with no operational impact (study notes, reading lists, glossaries)

Use a domain folder's `README.md` / `QA.md` / `interview-scenarios.md` for content scoped to that domain.

## Subfolders

Group docs by topic when there are multiple. Today: only `aws/`. Future: `gcp/`, `azure/`, `vendor-comparisons/`, etc.
