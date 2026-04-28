# cplanes — homelab platform, organized in planes

Single repo holding all GitOps-managed content for the `engatwork.com` homelab estate. Reconciled by ArgoCD instances on OKD.

GitHub: [`GaddipatiSriram/cplanes`](https://github.com/GaddipatiSriram/cplanes) (Public)

> Folder layout follows the **"Think in Planes"** model — see [Building a Kubernetes Platform: Think Big, Think in Planes](https://itnext.io/building-a-kubernetes-platform-think-big-think-in-planes-ede8bcba295f). Each plane is a coherent capability (control, security, observability, identity, service, finops, environment) with its own scope, lifecycle, and operational concerns.

## Boundary

This repo holds GitOps content (Helm charts + ArgoCD manifests). The **bootstrappers** that create the clusters live in three separate repos:

| Repo | What |
|---|---|
| [`cluster`](https://github.com/GaddipatiSriram/cluster) | kubeadm bootstrap + authoritative CoreDNS |
| [`okd`](https://github.com/GaddipatiSriram/okd) | OKD agent-based installer |
| [`ovirt-setup`](https://github.com/GaddipatiSriram/ovirt-setup) | oVirt Engine + pfSense + VM templates |

## Planes

```
cplanes/
├── argo-applications/       Argo manifests (the GitOps source of truth — reconciles everything below)
│
├── control/                 Control Plane
│   ├── argo-workflows/      DAG-based platform jobs
│   ├── backstage/           Developer portal
│   └── crossplane/          Declarative IaC for cross-cluster resources
│
├── security/                Security Plane (with sub-planes)
│   ├── secrets/             ↳ Vault, External Secrets Operator
│   ├── compliance/          ↳ OPA-Gatekeeper, Kyverno, Trivy, Falco, Wazuh, DefectDojo
│   └── ca-pki/              ↳ cert-manager
│
├── observability/           Observability Plane — Prometheus, Grafana, Alertmanager, Loki, Tempo, OTel + per-cluster agents
│
├── identity/                Identity Plane — Keycloak, Dex
│
├── service/                 Service Plane — data services exposed to apps
│   ├── cloudnative-pg/      Postgres operator
│   ├── strimzi/             Kafka KRaft operator
│   ├── apicurio/            Schema registry
│   ├── redis-operator/
│   ├── rabbitmq-operator/
│   ├── nats/
│   ├── debezium/            CDC connectors
│   └── opensearch/          Search engine
│
├── finops/                  FinOps Plane — Kubecost (cost allocation per namespace)
│
├── environment/             Environment Plane — Kube-Green (idle sleep), Kepler (per-pod energy)
│
├── storage/                 Storage Plane — Rook-Ceph (server + external CSI), Longhorn, MinIO, Velero
│
├── networking/              Networking Plane — ingress-nginx, MetalLB, Kong, K8GB, mesh/{istio,linkerd,cilium}
│
├── ci-cd/                   CI/CD Plane — Harbor, SonarQube, GitLab, Jenkins, Tekton, Argo Rollouts, ARC
│
└── docs/                    Cross-cutting reference — AWS comparison, etc.
```

**11 planes** + argo-applications + docs. Each plane carries the same four documents:

| File | Purpose |
|---|---|
| `README.md` | Current setup — what's running, why, cluster targets, dependencies |
| `todo.md` | Backlog — pending work, known bugs, migration paths |
| `QA.md` | Conceptual learning Q&A — "Why X?", "How does Y work?" |
| `interview-scenarios.md` | Architecture / migration / failure scenarios for practice |

Sub-planes inside `security/` (per the article's diagram) repeat the pattern at one level deeper.

## Cluster targets

ArgoCD targets clusters via labels on the cluster Secret. Selector patterns:

| Pattern | Selector | Use for |
|---|---|---|
| **A** | `tier In [mgmt,dev], role NotIn [storage]` | Cluster-local infra (cert-manager, ESO, ingress, MetalLB, trivy, mesh sidecar) |
| **B** | `role: forge` | Forge-cluster-only services (Vault, Keycloak — until OKD migration) |
| **C** | `role: observ` | Observability backends (Prom, Graf, Loki, Tempo, Kubecost) |
| **D** | `role: core` | Core foundational services (CoreDNS — currently in the `cluster/` bootstrap repo) |
| one-off | explicit `name:` match | Single-target Apps (Rook-Ceph server on mgmt-storage; ARC on mgmt-workload) |

| Cluster | VLAN | Role labels | Hosts |
|---|---|---|---|
| OKD (`mgmt-devops-okd`) | 11 | (operator-managed) | argocd-sre + argocd-devops + control-plane services |
| mgmt-core | 10 | `tier=mgmt role=core` | Authoritative CoreDNS |
| mgmt-storage | 13 | `tier=mgmt role=storage` | Rook-Ceph server |
| mgmt-forge | 14 | `tier=mgmt role=forge` | GitLab/Harbor/SonarQube + service-plane data infra |
| mgmt-observability | 12 | `tier=mgmt role=observ storage=ceph-rbd` | Prom/Graf/Alertmgr + (eventually) Kubecost |
| mgmt-workload | 11 (.120) | `tier=mgmt role=workload storage=ceph-rbd` | ARC controller + runners, ephemeral workloads |
| dev-{web,apps,data} (planned) | 20-22 | `tier=dev role={web,apps,data}` | Application workloads |

## Conventions

- **No `kubectl apply` against managed clusters** — everything via ArgoCD once the cluster is registered.
- **Cluster labels are the source of truth** for what deploys where. Defined in `cluster/k8s-bstrp/group_vars/<cluster>.yml`, propagated by Phase 8 of `bootstrap-k8s.yml`.
- **Sync waves** (`argocd.argoproj.io/sync-wave`): wave 0 = CRDs/foundations, wave 1 = identity, wave 2 = platform apps, wave 3 = workloads.
- **Secrets in git: never.** Use ESO + Vault for everything app-facing. Repo creds Secrets (`layers-secret-*.yaml`) are gitignored.
- **Homelab passwords** (`unix` / `pfsense`) are demo-only.

## Quick links

- AppSet patterns + service catalog: [`argo-applications/README.md`](./argo-applications/README.md)
- AWS-recommended EKS patterns (contrast doc): [`docs/aws/eks-bootstrap-argo.md`](./docs/aws/eks-bootstrap-argo.md)
- Original "Think in Planes" article: <https://itnext.io/building-a-kubernetes-platform-think-big-think-in-planes-ede8bcba295f>
