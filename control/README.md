# control — current setup

## What lives here

The **control plane** holds the platform's control tooling — the things that orchestrate other things. Backstage (developer portal), Argo Workflows (DAG-based platform jobs), Crossplane (declarative infrastructure provisioning). Per the article's planes diagram, this plane is the operator's window into the platform.

AppSets that drive deployment: `../argo-applications/sre/control/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **backstage** | 📋 placeholder | OKD | one-off | Developer portal; reads k8s + GitHub APIs cross-cluster |
| **argo-workflows** | ✅ live | OKD | one-off | DAG-based workflow engine |
| **crossplane** | 📋 placeholder | OKD | one-off | Manages remote-cluster resources via provider-kubernetes ProviderConfig kubeconfigs |

## Dependencies

- **Upstream**: identity plane (Keycloak OIDC), security plane (Vault for secrets, cert-manager for TLS), observability (Backstage's k8s plugin reads metrics).
- **Downstream**: every other plane is *managed* through control-plane tooling — Crossplane provisions ProviderConfigs targeting workload clusters, Argo Workflows runs jobs against any cluster, Backstage surfaces all of them.

## Architectural decisions

- **Control-plane services run on OKD** (the durable cluster). State must always be available. They dispatch *work* to other clusters via:
  - ArgoCD: `destination.server` per Application
  - Crossplane: `ProviderConfig` referencing remote-cluster kubeconfigs
  - Backstage: per-cluster kubeconfigs in `app-config.yaml`'s `kubernetes` section
- **No ARC controller here** — ARC's listener can only create runners in-cluster. ARC lives in the workload plane (`mgmt-workload` cluster), with its chart in `ci-cd/` for organization.

## Known issues

- See `todo.md` for tracked issues.
