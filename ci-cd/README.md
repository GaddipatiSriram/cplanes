# ci-cd — current setup

## What lives here

The **CI/CD plane** holds pipeline tooling — source control, container registry, code quality, test/build runners, progressive delivery. These are *build-time* concerns. The control plane's tooling (ArgoCD, Argo Workflows, Crossplane) handles *deploy-time*.

AppSets that drive deployment: `../argo-applications/sre/ci-cd/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **harbor** | 📋 planned | mgmt-forge | B | Registry + Trivy + Notary; OIDC via Keycloak |
| **sonarqube** | ✅ live | mgmt-forge | B | SAST gate; OIDC via Keycloak (pending) |
| **gitlab** | 📋 placeholder | mgmt-forge | B | Optional self-hosted source — currently using GitHub |
| **jenkins** | ✅ live | mgmt-control | role=control | Migrated off OKD 2026-05; serves at jenkins.apps.mgmt-control.engatwork.com |
| **harness-delegate** | ✅ live (delegate only) | mgmt-control | role=control | Outbound-only worker for Harness SaaS pipelines; SaaS handles authoring/orchestration |
| **tekton** | 📋 placeholder | mgmt-control | role=control | Container-native pipelines; deferred behind Harness adoption |
| **argo-rollouts** | ✅ live | mgmt-control | role=control | Progressive delivery dashboard at rollouts.apps.mgmt-control.engatwork.com |
| **actions-runner-controller** | 📋 planned | mgmt-workload | role=workload selector | Chart lives here, but deploys to workload plane |

## Dependencies

- **Upstream**: identity plane (OIDC), secrets plane (DB creds, repo creds via ESO), data plane (PG via CNPG for Harbor + SonarQube + GitLab).
- **Downstream**: app-deployment workflows in `control/argo-workflows`, every Application that consumes images from Harbor.

## Architectural decisions

- **ARC's chart code in this folder, but deployed to mgmt-workload**: ARC's listener can only create runners in-cluster, so the controller co-locates with the runners on the workload plane. Putting the chart here matches the human concept ("CI/CD tooling"); the deployment target is determined by the AppSet selector, not the folder.
- **Harness SaaS + Delegate over self-hosted Harness OSS** (2026-05): self-hosted is 9 microservices and ~20 GiB RAM; the delegate is ~2 GiB and gives the same authoring UX. Trade-off accepted: the *control plane* of Harness lives on app.harness.io rather than in-cluster — fine for a homelab, would re-evaluate for an air-gapped prod scenario.
- **Harness triggers ArgoCD rather than replacing it**: Harness has a GitOps mode that bundles its own Argo agents — chose not to use it because we already have 63 Synced apps under argocd-sre and ripping that out for the sake of a single-vendor stack would burn weeks. Connector pattern instead: Harness pipelines drive `Application` sync via the ArgoCD HTTP API.
- **Multi-CI coexistence is the steady state, not transition** — Argo Workflows handles cross-cluster platform jobs, Jenkins covers the demo/legacy path, Harness fronts the developer pipeline UX, Argo Rollouts handles canary mechanics. Picking one "winner" is a 2027 question; for now the planes overlap by design.

## Known issues

- ARC's GitHub App credentials must be in Vault under `kv/forge/github-app/` before the chart can deploy. See `security/secrets/vault/todo.md`.
