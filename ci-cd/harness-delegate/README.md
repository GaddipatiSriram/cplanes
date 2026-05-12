# harness-delegate — current setup

Harness Delegate runs on **mgmt-control** as a single outbound-only Deployment. Pipelines are authored and executed in **Harness SaaS** (`app.harness.io`); the delegate is the in-cluster worker that Harness reaches via reverse-proxy to deploy into the homelab.

This is the **SaaS + Delegate** mode, not the self-hosted Harness OSS install. We get the same authoring UX with ~2 GiB RAM cost in-cluster vs. ~20 GiB for self-hosted.

## Topology

```
   Harness SaaS (app.harness.io/gratis)
                 ▲
                 │ outbound HTTPS (WSS for pipeline assignment)
                 │
   ┌─────────────┴─────────────────────────────────────────┐
   │  mgmt-control kubeadm cluster                          │
   │                                                        │
   │   Namespace: harness-delegate                          │
   │   ┌──────────────────────────────────────────────┐    │
   │   │  Deployment/harness-delegate (1 replica)     │    │
   │   │    env DELEGATE_TOKEN ← Secret/harness-delegate-token│
   │   │    env ACCOUNT_ID     ← Secret/harness-delegate-token│
   │   │                                              │    │
   │   │    Polls Harness SaaS for pipeline steps,    │    │
   │   │    executes them in-cluster (kubectl/helm/   │    │
   │   │    argocd CLI all available).                │    │
   │   └──────────────────────────────────────────────┘    │
   │                                                        │
   │   Namespace: external-secrets                          │
   │   ExternalSecret/harness-delegate-token                │
   │     ← ClusterSecretStore/vault-forge                   │
   │     ← Vault path: kv/forge/harness/delegate            │
   └────────────────────────────────────────────────────────┘
                                       │
                                       │ ArgoCD API
                                       ▼
   ┌────────────────────────────────────────────────────────┐
   │  argocd-sre.apps.mgmt-control.engatwork.com            │
   │  (Argo CD Connector in Harness; triggers Application  │
   │   sync as a pipeline step)                             │
   └────────────────────────────────────────────────────────┘
```

The delegate is **outbound-only**. No Ingress, no MetalLB IP, no cert needed. The TLS is all on the Harness SaaS side.

## Bootstrap (manual, one-time)

ArgoCD will reconcile this chart, but the delegate will CrashLoopBackOff until the Vault path exists. Sequence:

1. **Sign up at <https://app.harness.io>** (free tier). Note your **Account ID** (visible in the URL after `/account/` once you're in).

2. **Mint a Delegate Token**: Account Settings → Account Resources → **Delegate Tokens** → New Token → name it `mgmt-control` → copy the token value (only shown once).

3. **Write both into Vault** from any host with Vault access (`mgmt-forge-01` works):
   ```bash
   vault kv put kv/forge/harness/delegate \
     DELEGATE_TOKEN='<paste delegate token>' \
     ACCOUNT_ID='<paste account id>'
   ```

4. **Force an ESO refresh** so the Secret materializes without waiting an hour:
   ```bash
   kubectl --context mgmt-control -n harness-delegate \
     annotate externalsecret harness-delegate-token \
     force-sync=$(date +%s) --overwrite
   ```

5. **Watch the Deployment come up**:
   ```bash
   kubectl --context mgmt-control -n harness-delegate get pods -w
   ```

6. **In Harness UI**: Setup → Delegates. The `mgmt-control` delegate should show Connected within 60s.

## Integrating with ArgoCD

Harness drives Argo as a connector — pipelines don't ship manifests to clusters, they trigger reconciliation on existing ArgoCD Applications.

Add the connector in Harness:

| Field | Value |
|---|---|
| Connector Name | `argocd-sre-mgmt-control` |
| Type | Argo CD |
| URL | `https://argocd-sre.apps.mgmt-control.engatwork.com` |
| Authentication | Bearer Token |
| Token | output of `argocd account generate-token --account admin` (run from any kubectl context that can talk to argocd-server) |
| Delegate selector | `mgmt-control` |

Then in a pipeline step, **Approval / Custom → Argo CD → Sync Application** with the Application name (e.g. `homer-mgmt-control`).

## Why this and not self-hosted Harness OSS?

| | SaaS + Delegate (this) | Self-hosted OSS |
|---|---|---|
| In-cluster RAM | ~2 GiB | ~20 GiB (9 microservices + MongoDB + Redis + TimescaleDB) |
| Setup time | ~2 hours | ~1 day |
| Pipeline authoring UX | Identical | Identical |
| Data plane (where deploys execute) | Identical (delegate in your cluster) | Identical |
| Available in homelab budget today | ✅ | ❌ (mgmt-control sized for 8-12 GiB nodes) |

If/when mgmt-control grows or a new cluster lands, the self-hosted path stays open — switching is a chart swap, the pipeline definitions stay in Harness's Git source.

## TODOs

- **Re-enable upgrader** once we want unattended delegate image bumps. Currently disabled because the chart's `templates/upgrader/upgraderSecret.yaml` reads `.Values.delegateToken` (plaintext) to populate UPGRADER_TOKEN — incompatible with our Vault/ESO pattern where `delegateToken` stays empty. To re-enable: write an `UPGRADER_TOKEN` ExternalSecret (same Vault path as DELEGATE_TOKEN, since Harness uses the same token for both), set `upgrader.existingUpgraderToken` to that Secret name, flip `upgrader.enabled` to `true`. Until then, bump `delegateDockerImage` tag in `values.yaml` manually when Harness ships a new release (~monthly cadence).
- Once a pipeline runs end-to-end, capture an `interview-scenarios.md` entry showing the cross-platform-tool architecture decision: why Harness + ArgoCD instead of (a) Harness alone with Harness GitOps, or (b) Argo Workflows + Rollouts alone.
- Replace `argocd account generate-token` with an OIDC-backed SA token once Keycloak group→argocd RBAC is wired (currently `platform-sre`/`platform-users` only grant UI access; programmatic SA tokens are still local-admin).
