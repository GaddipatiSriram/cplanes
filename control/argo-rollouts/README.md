# argo-rollouts

Progressive delivery controller — extends Kubernetes with a `Rollout` CR
that orchestrates **canary** and **blue-green** deployments with
analysis-driven promotion (auto-promote when metrics look good,
auto-rollback when they don't).

## What this chart deploys

- **argo-rollouts controller** — watches `Rollout` CRs cluster-wide
- **dashboard** at `https://rollouts.apps.mgmt-control.engatwork.com` — visual rollout state
- **CRDs**: `Rollout`, `AnalysisTemplate`, `AnalysisRun`, `Experiment`, `ClusterAnalysisTemplate`
- **ServiceMonitor** so Prometheus auto-scrapes controller metrics

## Why it's here (cert + role mapping)

| Cert | What it tests |
|---|---|
| **CAPA** | Rollouts is one of the four Argo Project pillars; ~20% of CAPA |
| **CKAD** | Application Deployment domain (15%) — covers progressive delivery patterns |
| **CKS** | Indirectly — canary is a defense-in-depth deploy strategy |
| **CNPE** | Hands-on capstone explicitly tests SLO-gated rollouts via Rollouts + AnalysisTemplate |

| Role | What you'll do here |
|---|---|
| **DevOps Engineer** | Author Rollout specs replacing Deployments |
| **SRE** | Wire AnalysisTemplate to Prometheus error-rate queries |
| **Edge / Gateway Engineer** (the JD) | Traffic-shift via Rollouts + Kong/Envoy as canary substrate |

## Two strategies, side-by-side

```yaml
# canary — gradual traffic shift with pauses + analysis
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: { duration: 30s }
      - analysis:
          templates: [{ templateName: success-rate }]
      - setWeight: 50
      - pause: { duration: 30s }
      - setWeight: 100
```

```yaml
# blue-green — old + new run together; instant cutover via Service selector swap
strategy:
  blueGreen:
    activeService: web-active
    previewService: web-preview
    autoPromotionEnabled: false   # require manual `kubectl argo rollouts promote`
```

## Cert prep — what to practice

1. **Convert one Deployment to a Rollout** in `apps/learning-app/` chart; verify dashboard shows the rollout.
2. **Author an AnalysisTemplate** that queries Prometheus:
   ```yaml
   metrics:
     - name: success-rate
       interval: 30s
       successCondition: result[0] >= 0.99
       provider:
         prometheus:
           address: http://prometheus-operated.observability:9090
           query: |
             sum(rate(http_requests_total{job="learning-app",status=~"2.."}[2m]))
             /
             sum(rate(http_requests_total{job="learning-app"}[2m]))
   ```
3. **Trigger a bad rollout** by pushing a broken image; verify auto-rollback when AnalysisTemplate fails.
4. **Manual promote** via `kubectl argo rollouts promote learning-app -n learning`.

## Common operations

```bash
# install Argo Rollouts kubectl plugin
brew install argoproj/tap/kubectl-argo-rollouts   # or download from releases

# watch a rollout live
kubectl argo rollouts get rollout learning-app -n learning --watch

# manually promote past a pause
kubectl argo rollouts promote learning-app -n learning

# manually abort + rollback
kubectl argo rollouts abort learning-app -n learning
kubectl argo rollouts undo learning-app -n learning
```

## Cross-references

- [`/root/learning/certifications/capa.md`](../../../certifications/capa.md) — CAPA prep
- [`/root/learning/certifications/cnpe.md`](../../../certifications/cnpe.md) — CNPE capstone
- [`apps/`](../../../apps/) — where Rollout CRs land per service
