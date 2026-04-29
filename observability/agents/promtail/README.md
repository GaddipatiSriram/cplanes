# promtail

Pattern A DaemonSet — runs on every workload cluster, ships container
stdout/stderr from `/var/log/pods/` to Loki on mgmt-observability.

For Loki itself see [`../../loki/`](../../loki/).
For the broader agent fleet plan see [`../README.md`](../README.md) (TBD)
and [`../../README.md`](../../README.md).

---

## Topology

```
        mgmt-observability
        ┌──────────────────────────┐
        │  Loki (SingleBinary)     │
        │   Service/loki:3100      │
        │   ↑                      │
        │ ingress-nginx            │
        │   ↑ HTTPS push           │
        └──┼───────────────────────┘
           │
   ┌───────┼───────────────────────────────────────────────┐
   │       │                                               │
   │  mgmt-control     mgmt-forge        mgmt-storage      │
   │  ┌──────────┐    ┌──────────┐      ┌──────────┐       │
   │  │promtail  │    │promtail  │      │promtail  │       │
   │  │ DS-pod   │    │ DS-pod   │      │ DS-pod   │       │
   │  │  per node│    │  per node│      │  per node│       │
   │  └──────────┘    └──────────┘      └──────────┘       │
   └───────────────────────────────────────────────────────┘
```

## Per-cluster identity (the `cluster` label)

Promtail tags every stream with `cluster=<name>`. Same Loki instance,
multiple sources — queries like `{cluster="mgmt-control",namespace="argocd-sre"}`
find the right pod logs.

The label value is injected by the AppSet via Helm parameter:

```yaml
helm:
  parameters:
    - name: promtail.config.clients[0].external_labels.cluster
      value: '{{name}}'
```

`{{name}}` is the cluster Secret's name (e.g. `mgmt-control`,
`mgmt-forge`).

## What gets shipped

| Source | Path | Labels |
|---|---|---|
| Pod logs (every container, every namespace) | `/var/log/pods/<ns>_<pod>_<uid>/<container>/*.log` | `namespace`, `pod`, `container`, `node_name`, `app` |

The chart's default `scrapeConfigs` already does kubernetes service
discovery + the standard relabeling (namespace, pod, container, node).
We don't override scrape configs — same shape on every cluster keeps
queries portable.

## What's NOT shipped (and why)

- **Host /var/log** — node-level systemd journal, kubelet logs.
  Useful but adds complexity (separate scrape config, journal access).
  Add later as `extraScrapeConfigs` if the need surfaces.
- **k8s audit log** — written to a file by kube-apiserver. Will be
  shipped to **OpenSearch** (security plane) under the routed-by-source
  decision. Not Promtail's scope.
- **Falco events** — runtime security, also OpenSearch-bound.

## Resource & scale notes

- ~50–100 MB memory per pod steady-state, depending on log volume.
- A 3-node cluster ≈ 3 pods × 256 MiB limit = 768 MiB ceiling per
  cluster, ~1.5 % of mgmt-observability's 48 GiB.
- `priorityClassName: system-node-critical` so Promtail isn't evicted
  before payload pods during pressure — losing logs during incidents is
  exactly when you need them.

## Common operations

**Verify it's pushing** — on Loki side:
```bash
kubectl --context mgmt-observability -n observability logs sts/loki | grep -i 'received'
# or via the API:
curl https://loki.apps.mgmt-observability.engatwork.com/loki/api/v1/labels | jq
# Expect: cluster, namespace, pod, container, node_name, app, ...
```

**Verify per-cluster labels** — Grafana → Explore → Loki:
```
{cluster="mgmt-control"} | json
```

**Restart all promtails on a cluster** —
```bash
kubectl --context <cluster> -n observability rollout restart ds/promtail
```

**Check positions file** — Promtail writes `/run/promtail/positions.yaml`
to track read offsets. If you lose it (pod recreated without persistence),
Promtail re-reads from log-tail (`__path__` start), not from beginning —
some duplication on restart, by design.
