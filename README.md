# layers — homelab GitOps content

Single repo holding all GitOps-managed content for the `engatwork.com` homelab estate. Reconciled by ArgoCD instances running on OKD.

GitHub: [`GaddipatiSriram/layers`](https://github.com/GaddipatiSriram/layers)

## Boundary

This repo holds only the **GitOps content** — Helm charts and ArgoCD `Application`/`ApplicationSet` manifests. The **bootstrappers** that create the clusters in the first place (oVirt + pfSense + kubeadm + OKD installer) live in three separate repos at `/root/learning/`:

| Repo | What | Why separate |
|---|---|---|
| [`cluster`](https://github.com/GaddipatiSriram/cluster) | kubeadm bootstrap + authoritative CoreDNS | Imperative Ansible; runs once per cluster. Lifecycle decoupled from continuous reconciliation. |
| [`okd`](https://github.com/GaddipatiSriram/okd) | OKD agent-based installer | Imperative Ansible; runs once per OKD instance. |
| [`ovirt-setup`](https://github.com/GaddipatiSriram/ovirt-setup) | oVirt Engine + pfSense + VM templates | Imperative Ansible; foundational hypervisor. |

## Layout — services grouped by domain

```
layers/
├── argo-applications/   ArgoCD Applications + ApplicationSets (the GitOps source of truth)
├── platform/            cluster-local Pattern A baseline (cert-manager, ingress-nginx, ESO, MetalLB)
├── identity/            IdP — Keycloak
├── secrets/             secret stores — Vault
├── forge/               software forge tools — Harbor, SonarQube, GitLab (planned)
├── data/                relational + streaming + cache — CNPG, Strimzi, Apicurio, Redis (planned)
├── search/              search + SIEM — OpenSearch, Wazuh, DefectDojo (planned)
├── observability/       metrics + logs + traces backends — Prometheus, Grafana, Alertmanager
├── security/            security tooling — Trivy operator (+ OPA, Falco planned)
├── devops/              CI/CD — Argo Workflows, Jenkins, Actions Runner Controller
├── storage/             storage tier — Rook-Ceph (server + external), Longhorn, MinIO
└── docs/                cross-cutting reference docs
```

Each domain folder carries the same four documents:

| File | Purpose |
|---|---|
| `README.md` | Current setup — what's running, why, cluster targets, dependencies. |
| `todo.md` | Backlog — pending work, known bugs, migration paths. |
| `QA.md` | Conceptual learning Q&A — "Why X?", "How does Y work?". |
| `interview-scenarios.md` | Architecture / migration / failure scenarios for practice. |

## Cluster targets

ArgoCD targets clusters via labels on the cluster Secret (Pattern A/B/C/D selectors — see `argo-applications/README.md`).

| Cluster | VLAN | Subnet | Role labels | Hosts |
|---|---|---|---|---|
| **OKD** (`mgmt-devops-okd`) | 11 | 10.10.2.0/24 (.110-112) | (no `tier` — operator-managed) | ArgoCD SRE + DevOps |
| **mgmt-core** | 10 | 10.10.1.0/24 | `tier=mgmt role=core` | Authoritative CoreDNS |
| **mgmt-storage** | 13 | 10.10.4.0/24 | `tier=mgmt role=storage` | Rook-Ceph server |
| **mgmt-forge** | 14 | 10.10.5.0/24 | `tier=mgmt role=forge` | GitLab/Harbor/SonarQube + integration tier |
| **mgmt-observability** | 12 | 10.10.3.0/24 | `tier=mgmt role=observ storage=ceph-rbd` | Prom/Graf/Alertmgr/Loki/Tempo |
| **mgmt-workload** | 11 (.120) | shared with OKD | `tier=mgmt role=workload storage=ceph-rbd` | ARC controller + runners, ephemeral workloads |
| **dev-{web,apps,data}** (planned) | 20-22 | 10.20.{1,2,3}.0/24 | `tier=dev role={web,apps,data}` | Application workloads |

## Quick links

- **AppSet patterns + service catalog**: [`argo-applications/README.md`](./argo-applications/README.md)
- **AWS-recommended EKS patterns** (contrast doc): [`docs/aws/eks-bootstrap-argo.md`](./docs/aws/eks-bootstrap-argo.md)

## Conventions

- **No `kubectl apply` against managed clusters** — everything via ArgoCD once a cluster is registered.
- **Cluster labels are the source of truth** for what deploys where. Defined in `cluster/k8s-bstrp/group_vars/<cluster>.yml`, propagated by Phase 8 of `bootstrap-k8s.yml`.
- **Sync waves** (`argocd.argoproj.io/sync-wave`): wave 0 = CRDs/foundations, wave 1 = identity, wave 2 = platform apps, wave 3 = workloads.
- **Secrets in git: never.** Use ESO + Vault for everything app-facing. Repo creds Secrets (`layers-secret-*.yaml`) are gitignored — apply manually on OKD; migration to Sealed Secrets / SOPS tracked in domain todo.md files.
- **Homelab passwords** (oVirt admin, SSH root) are `unix` / `pfsense` — do not replicate in real use.
# cplanes
