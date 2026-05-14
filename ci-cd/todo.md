# ci-cd — TODO

Backlog ordered by likely sequence.

---

## harness-delegate: pipeline-as-code, not Harness-SaaS-only

**Status:** open (2026-05-14).

The `argocd_sync_and_wait` pipeline (and any future ones) currently live
in Harness SaaS, created via the Harness REST API but not source-controlled.
A wipe of the Harness account loses every pipeline. Options:

1. **Harness Terraform Provider** — declare pipelines in `cplanes/ci-cd/harness-delegate/terraform/*.tf`. Each pipeline becomes a `harness_platform_pipeline` resource. Provider auth via the PAT.
2. **Harness Git Experience** — Harness can sync pipelines from a git folder. Each pipeline becomes `cplanes/ci-cd/harness-delegate/pipelines/<name>.yaml` and Harness picks them up via webhook.
3. **Status quo + periodic export** — `curl ... GET /pipelines/{id}` dumps the YAML; commit the result. Cheap, manual.

Recommend option 1 for declarative discipline matching the rest of the
GitOps pattern; option 2 if Harness's git-sync ends up cleaner than the
TF provider in practice. Until this closes, the canonical pipeline YAML
is also captured below as a recovery artifact.

### Recovery artifact: `argocd_sync_and_wait` pipeline YAML (2026-05-14)

```yaml
pipeline:
  name: argocd-sync-and-wait
  identifier: argocd_sync_and_wait
  projectIdentifier: default_project
  orgIdentifier: default
  variables:
    - name: appName
      type: String
      required: true
      value: <+input>
    - name: timeoutSeconds
      type: String
      default: "300"
      value: <+input>
  stages:
    - stage:
        name: sync-and-wait
        identifier: sync_and_wait
        type: Custom
        spec:
          execution:
            steps:
              - step:
                  type: ShellScript
                  name: sync_and_poll
                  identifier: sync_and_poll
                  spec:
                    shell: Bash
                    onDelegate: true
                    source:
                      type: Inline
                      spec:
                        script: |-
                          set -euo pipefail
                          APP="<+pipeline.variables.appName>"
                          TIMEOUT="<+pipeline.variables.timeoutSeconds>"
                          kubectl -n argocd-sre patch application "${APP}" --type=merge \
                            -p '{"operation":{"sync":{"prune":true,"syncOptions":["CreateNamespace=true"]}}}' || true
                          kubectl -n argocd-sre wait --for=jsonpath='{.status.sync.status}'=Synced \
                            "application/${APP}" --timeout="${TIMEOUT}s" || echo "  (Synced wait timed out)"
                          kubectl -n argocd-sre wait --for=jsonpath='{.status.health.status}'=Healthy \
                            "application/${APP}" --timeout="${TIMEOUT}s"
                          kubectl -n argocd-sre get "application/${APP}" \
                            -o jsonpath='sync={.status.sync.status}  health={.status.health.status}  revision={.status.sync.revision}{"\n"}'
                    delegateSelectors:
                      - mgmt-control
                  timeout: 15m
```

---

## harness-delegate: replace admin-account token with dedicated `harness` ArgoCD account

**Status:** open. The `argocd_sync_and_wait` pipeline avoids the bearer
token by talking to argocd-sre over kubectl via the delegate's SA (which
the chart binds to cluster-admin), so this is no longer load-bearing.
But future pipelines that hit ArgoCD's HTTP API directly (e.g., to query
sync history from outside the cluster, or drive Applications on a
remote ArgoCD instance) still need a token. The current token is from
`admin` — too privileged.

Plan:

1. Add `accounts.harness: "apiKey"` to `cplanes/control/argo-cd/values-sre.yaml` (next to the existing `accounts.admin: "apiKey, login"`).
2. RBAC policy in same values: define a `role:sync-only` allowing only `applications, sync, */*` + `applications, get, */*`, then `g, harness, role:sync-only`.
3. Set password for the `harness` account (one-shot kubectl exec; can't be declarative because passwords are stored hashed in argocd-secret).
4. Mint a token for `harness`, write to Vault `kv/forge/harness/argocd-token`, mirror to a Harness Secret via the future ESO-bridge pattern (today: manual paste).
5. Revoke the admin-scope token.

---

## harness-delegate: drop the dead bearer-token Secret if unused

**Status:** open. Harness Secret `argocd_sre_token` (the admin JWT, account-scoped) was created during the integration setup but is **not used** by `argocd_sync_and_wait` (which uses kubectl + delegate SA instead). Either:

- Delete it now (`curl -X DELETE -H 'x-api-key: <pat>' .../secrets/argocd_sre_token?accountIdentifier=...`) — clean slate.
- Keep it for direct-API workflows that may emerge (Backstage plugin, multi-cluster ArgoCD aggregation, etc.) — fine, just document that it's not auto-rotated and remove after the `harness` account migration above.

Vault path `kv/forge/harness/delegate` field `DELEGATE_TOKEN` is *still*
required by the delegate itself — don't conflate the two when revoking.

---

## harness-delegate: re-enable upgrader once UPGRADER_TOKEN flow is wired

**Status:** open (preserved from chart README). The chart's
`upgrader.enabled: true` blocks because the upstream template reads
plaintext `.Values.delegateToken` to seed `UPGRADER_TOKEN`. We disabled
upgrader. To re-enable: second ExternalSecret writing `UPGRADER_TOKEN`
to a Secret, then `upgrader.existingUpgraderToken: <that secret>`. Same
Vault path as `DELEGATE_TOKEN` since Harness uses the same token value
for both.

Until then: bump `delegateDockerImage` tag in chart `values.yaml`
manually when Harness ships a new release (~monthly).

---

## ci-cd: pipelines for apps/learning-app once it exists

**Status:** blocked on Phase 1 of `/root/learning/ROADMAP.md`.

Two pipelines to author once the Go service lands:

1. **Build pipeline** — Harness CI: multi-stage Docker, Trivy scan, push to registry (GHCR until Harbor stands up). Image tag: git SHA.
2. **Deploy pipeline** — Shell Script on the delegate: git-clone cplanes, `yq` bump `apps/learning-app/values.yaml` image tag, commit + push. Then invoke `argocd_sync_and_wait` as a downstream stage with `appName=learning-app-mgmt-control`. Pipeline status reflects the GitOps deploy + ArgoCD health, not just the git push.

Prerequisites:

- Harness Git connector pointing at `github.com/GaddipatiSriram/cplanes` with write access (GitHub PAT or deploy key, stored as a Harness Secret).
- Image registry: GHCR works today (free for public repos); Harbor later when that chart lands.
- A `harness` namespace SA in cplanes-repo for the git-bump commit author (or just sign with the same Harness Personal Access Token).

---

## Wire the plane's AppSet pattern into argo-applications/sre/ci-cd/

**Status:** harness-delegate AppSet is live (`harness-delegate-mgmt-control` Synced/Healthy on argocd-sre). Pattern: `role=control` (single-cluster, like Jenkins). Future ci-cd services follow their own patterns:

| Service | Cluster | Pattern |
|---|---|---|
| harness-delegate | mgmt-control | role=control ✅ |
| jenkins | mgmt-control | role=control ✅ |
| argo-rollouts | mgmt-control | role=control ✅ |
| sonarqube | mgmt-forge | role=forge ✅ |
| harbor (planned) | mgmt-forge | role=forge |
| ARC controller (planned) | mgmt-workload | explicit selector |

---

## Document plane-specific operational runbooks

**Status:** harness-delegate has its README runbook + the chart bootstrap flow. Jenkins, SonarQube, Argo Rollouts inherit from their upstream Helm charts. Once Harbor / ARC come online, each gets its own runbook covering: bootstrap creds in Vault, cluster registration, recovery from cold start.
