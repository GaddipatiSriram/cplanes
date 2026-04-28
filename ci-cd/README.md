# ci-cd — current setup

## What lives here

The **CI/CD plane** holds pipeline tooling — source control, container registry, code quality, test/build runners, progressive delivery. These are *build-time* concerns. The control plane's tooling (ArgoCD, Argo Workflows, Crossplane) handles *deploy-time*.

AppSets that drive deployment: `../argo-applications/sre/ci-cd/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **harbor** | 📋 planned | mgmt-forge | B | Registry + Trivy + Notary; OIDC via Keycloak |
| **sonarqube** | 📋 planned | mgmt-forge | B | SAST gate; OIDC via Keycloak |
| **gitlab** | 📋 placeholder | mgmt-forge | B | Optional self-hosted source — currently using GitHub |
| **jenkins** | ✅ live (legacy, retiring) | OKD | one-off | Demo only; planned removal |
| **tekton** | 📋 placeholder | OKD | OperatorHub | OpenShift Pipelines ships with OCP |
| **argo-rollouts** | 📋 planned | every workload cluster | A | Progressive delivery (canary, blue/green) |
| **actions-runner-controller** | 📋 planned | mgmt-workload | role=workload selector | Chart lives here, but deploys to workload plane |

## Dependencies

- **Upstream**: identity plane (OIDC), secrets plane (DB creds, repo creds via ESO), data plane (PG via CNPG for Harbor + SonarQube + GitLab).
- **Downstream**: app-deployment workflows in `control/argo-workflows`, every Application that consumes images from Harbor.

## Architectural decisions

- **ARC's chart code in this folder, but deployed to mgmt-workload**: ARC's listener can only create runners in-cluster, so the controller co-locates with the runners on the workload plane. Putting the chart here matches the human concept ("CI/CD tooling"); the deployment target is determined by the AppSet selector, not the folder.
- **Pick ONE CI engine long-term** — Argo Workflows (control plane), Tekton (ci-cd plane), Jenkins (legacy). Probably Tekton for OpenShift-native parity, or Argo Workflows for cross-cluster orchestration.

## Known issues

- ARC's GitHub App credentials must be in Vault under `kv/forge/github-app/` before the chart can deploy. See `security/secrets/vault/todo.md`.
