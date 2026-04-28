# /root/learning/layers/observability

Top-level project for the **observability stack** — metrics, logs,
traces, and alerting — deployed to the `mgmt-observability` cluster
(`role=observ`, VLAN 12, `10.10.3.0/24`).

Peer project to `storage/` — both are cross-cluster concerns that
deserve their own roots rather than living inside `forge/`.

> `forge/` = platform services (identity + secrets + CI/CD etc.).
> `storage/` = block/file/object backends.
> `observability/` = what you see after everything else is running.

## Why split, and why on mgmt-observability

### Split (one chart per component) instead of kube-prometheus-stack

`kube-prometheus-stack` is the popular monolith chart — Prometheus +
Alertmanager + Grafana + node-exporter + kube-state-metrics + 30
dashboards + ServiceMonitors in one install. Fine for a single
cluster observing only itself. We chose NOT to use it because:

1. **Multi-cluster observability** requires the backend to accept
   remote-write from other clusters. The monolith assumes a
   single-cluster deployment.
2. **Swap-ability**: when we outgrow Prometheus single-instance,
   swap to Mimir without touching Grafana's chart. The monolith
   wires components together tightly.
3. **Different lifecycles**. Grafana UI updates shouldn't restart
   Prometheus.
4. **Sync-wave ordering**: Prometheus (the data source) must exist
   before Grafana queries it. Split charts + Argo waves make that
   explicit.
5. **Ownership at scale**: in real orgs, metrics infra and
   dashboards are different teams.

Trade-off: more charts to author. Acceptable — each is a thin
umbrella over the upstream, and they compose by convention
(Grafana datasources point at `<service>.observability.svc.cluster.local`).

### Why mgmt-observability hosts the backends

Observability is CENTRAL — one Grafana, one Prometheus store, one
Alertmanager. A dedicated cluster:

- Isolates failure domain: Vault thrashing on mgmt-forge doesn't
  disrupt Prometheus queries.
- Right-sizes independently: Prometheus needs big disks; Grafana
  needs RAM. Dedicated cluster = sensible defaults for both.
- Clear access model: observability team gets cluster-admin on
  mgmt-observability without touching platform secrets.
- Network posture: backends firewalled to receive only
  remote-write / push from workload clusters.

mgmt-observability labels: `tier=mgmt role=observ`. ApplicationSets
in `argo-applications/sre/observability/` use Pattern C selector (`role: observ`).

## Architecture

```
  workload clusters              mgmt-observability (role=observ)
  (mgmt-forge, OKD,              ─────────────────────────────────
   dev-*, mgmt-core)
  ─────────────────              ┌─────────────────────────────┐
                                 │                             │
  Prometheus agent  ──remote-write────→  Prometheus  (metrics) │
  (Pattern A, fut)                 │     ↑                     │
                                   │     │ datasource          │
  Promtail daemon  ──push────────────→   Loki        (logs)    │
  (Pattern A, fut)                 │     ↑                     │
                                   │     │ datasource          │
  OTel Collector   ──OTLP traces────→    Tempo       (traces)  │
  agent daemon                     │     ↑                     │
  (Pattern A, fut)                 │     │ datasource          │
                                   │     │                     │
                                   │   Grafana  (UI)           │
                                   │     │                     │
                                   │     ↓ rule eval           │
                                   │   Alertmanager ──→ Slack /│
                                   │                   PagerDuty│
                                   │                   email   │
                                   └─────────────────────────────┘
```

## Components — full plan

The original "split LGTM stack" intent has been expanded to a Dynatrace-
capability-equivalent **enterprise-grade OSS stack**. Same split-charts
philosophy (each component owns its own lifecycle), but the feature
surface now matches what an APM vendor would charge per-host for.

Implementation choice: depend on `prometheus-community/kube-prometheus-stack`
in `observability/prometheus/` BUT only enable the operator + a single
Prometheus instance there.  Every other component (Alertmanager,
Grafana, kube-state-metrics, node-exporter, …) is a sibling chart
emitting Operator CRDs (`Alertmanager`, `ServiceMonitor`,
`PrometheusRule`, …) that the shared operator reconciles. Net: one
operator, many independently-evolvable backends.

### Backends (mgmt-observability, Pattern C selector `role=observ`)

| Component | Capability | Dynatrace equivalent | Session |
|---|---|---|---|
| `prometheus/` | metrics storage + scraping + operator + CRDs | Infra/APM metrics | **1** |
| `alertmanager/` | alert routing (Slack/email/PagerDuty/OnCall) | Davis alerting | **1** |
| `grafana/` | unified UI for everything | Dynatrace UI | **1** |
| `loki/` | log aggregation, full-text search | Log management | 2 |
| `tempo/` | distributed traces backend | PurePath / tracing | 2 |
| `otel-gateway/` | OTLP ingress (traces + metrics + logs) | OneAgent ingest | 2 |
| `pyroscope/` | continuous profiling | Code-level profiling | 3 |
| `oncall/` | incident management, on-call rotations | Davis incident routing | 3 |
| `blackbox-exporter/` | synthetic HTTP/TCP/ICMP probes | Synthetic checks | 2 |
| `k6-operator/` | scripted load + synthetic monitoring | Synthetic scripted | 3 |
| `mimir/` | HA-scalable Prometheus replacement | (when needed) | future |

### Agents (every workload cluster, Pattern A — `observability-agents/` sibling repo)

| Component | Capability | Dynatrace equivalent | Session |
|---|---|---|---|
| `node-exporter/` | host metrics (CPU/RAM/disk/net) | OneAgent infra | 2 |
| `kube-state-metrics/` | k8s object state metrics | OneAgent k8s | 2 |
| `prometheus-agent/` | local scraper → central remote-write | OneAgent metrics | 2 |
| `promtail/` | local log shipper → Loki | OneAgent logs | 2 |
| `otel-collector/` (DaemonSet) | local trace/metric/log collection → OTel gateway | OneAgent traces | 2 |
| `pyroscope-agent/` | per-pod profiling via eBPF | Code profiling | 3 |
| `beyla/` | eBPF auto-instrumentation (HTTP/gRPC/SQL spans without code changes) | OneAgent auto-instr | 3 |

### Cross-cutting (forge/ + observability-agents/)

| Component | Capability | Where | Session |
|---|---|---|---|
| `forge/sloth/` | SLO YAML → PrometheusRules + recording rules | mgmt-forge | 3 |
| `forge/falco/` | runtime security threats (eBPF) | every cluster (Pattern A) | 4 |
| `forge/opencost/` | per-pod cost attribution | every cluster (Pattern A) | 4 |
| `observability/robusta/` | alert enrichment + auto-remediation | mgmt-observability | 4 |

Total: **17 services across 4 sessions**. Each chart is a thin umbrella
over an upstream community/Grafana-Labs Helm chart with our values.

## Why each piece is "enterprise-grade"

- **LGTM+P stack** (Loki / Grafana / Tempo / Mimir / Pyroscope) — what
  Grafana Labs runs at multi-petabyte scale for their cloud customers.
  Same OSS bits as their paid product; no feature gap at the
  collection layer.
- **OpenTelemetry** — CNCF graduated standard. Datadog, Dynatrace,
  Honeycomb, AWS all consume OTLP natively. Instrument once, switch
  backends without re-instrumenting.
- **Grafana Beyla** — open-source answer to Dynatrace OneAgent.
  eBPF auto-instrumentation, no code changes, gives HTTP/gRPC/SQL spans
  for any binary on the node.
- **Falco** — CNCF runtime-security project. Datadog acquired Sysdig
  (Falco's authors) and rebrands it as "Cloud Workload Security" —
  same code.
- **Tempo + Pyroscope + Grafana** — gives **Trace-to-Profile**: click a
  slow span in a trace, jump to the CPU profile of the goroutine. The
  killer Datadog/Dynatrace feature, fully reproduced.
- **Grafana OnCall** — open-source PagerDuty replacement.

## Resource budget — does it fit?

mgmt-observability cluster: 3 nodes × 16 GiB / 4 cores / 100 GiB = 48 GiB / 12 cores / 300 GiB.

Backend steady-state requests (all 11 enabled): **~7.7 GiB / 2 cores**. Limits 2× that. That's ~5–6× headroom.

Workload-cluster agent footprint (3-node cluster): **~3 GiB total** across all 7 agents. About 6 % of an `okd`-preset cluster.

Storage on ceph-rbd (mgmt-storage backs PVCs):
- Prometheus 50 Gi (15d retention)
- Loki 100 Gi (30d retention, chunks compress well)
- Tempo 100 Gi (or S3-compat object via Ceph RGW once that's stood up)
- Pyroscope 50 Gi
- ~300 Gi total against mgmt-storage's 1.5 TiB capacity. Headroom fine.

Real risks called out: Prometheus query bursts, Loki ingestion spikes,
Tempo trace fan-out, single-instance failure domains. Mitigations
deferred to Mimir / Loki distributed mode / Tempo distributors when
demand justifies.

## GitOps wiring

```
observability/<component>/          ← umbrella chart per component
  └── Chart.yaml + values.yaml + optional templates/

argo-applications/sre/observability/             ← Argo wiring
  ├── prometheus-appset.yaml        ← Pattern C, selector role=observ
  ├── grafana-appset.yaml
  ├── alertmanager-appset.yaml
  └── ...
```

Each ApplicationSet:
- `selector.matchLabels: {role: observ}` — generates one Application
  per cluster labeled role=observ (today: mgmt-observability only).
- `source.repoURL: https://github.com/GaddipatiSriram/observability.git`
  (once git-init'd and pushed).
- `source.path: <component>`.
- `destination.namespace: observability` — single namespace for all
  backends; Grafana reaches peers via
  `<service>.observability.svc.cluster.local`.
- Sync-waves: Prometheus=0, Alertmanager=0, Grafana=1 (depends on
  datasources), Loki=0, Tempo=0.

## Integrations with the rest of the platform

| External piece | How it plugs in |
|---|---|
| **cert-manager** (forge/) | Issues TLS certs for Grafana ingress, Prometheus remote-write endpoint. Today: ingress-nginx default cert. Target: Vault PKI Issuer. |
| **Keycloak** (forge/) | Grafana OIDC client in `forge` realm. Users log in via Keycloak; RBAC via group claim. |
| **Vault** (forge/) | Grafana admin password, SMTP creds, PagerDuty API key. ExternalSecret materializes in `observability` ns. |
| **Rook-Ceph-External** (storage/) | `ceph-rbd` StorageClass for Prometheus/Loki/Tempo PVCs. Must be on mgmt-observability before backends deploy. |
| **Ingress-nginx** (forge/) | Exposes Grafana UI + Alertmanager webhook receivers. Needs to be on mgmt-observability (Pattern A — broaden cert-manager/ingress-nginx/ESO selectors per `storage/todo.md`). |

## Deployment order (this session + next)

1. **Register mgmt-observability in argocd-sre** (one-time)
   — apply `bootstrap-sa.yaml` on mgmt-observability +
   `cluster-secret.yaml` on OKD argocd-sre ns with labels
   `tier=mgmt role=observ env=nonprod`.
2. **Cephx bootstrap** from
   `storage/rook-ceph-external/generated/mgmt-observ-bundle.yaml`.
3. **rook-ceph-external ApplicationSet auto-fires** (matches Pattern A
   selector). CSI + StorageClasses available on mgmt-observability.
4. **Broaden cert-manager / ingress-nginx / external-secrets
   selectors** (`storage/todo.md` task) — they auto-install on
   mgmt-observability too.
5. **Deploy Prometheus** — Pattern C ApplicationSet; PVC on
   ceph-rbd; remote-write receiver enabled.
6. **Deploy Alertmanager** — alongside Prometheus.
7. **Deploy Grafana** — depends on Prometheus + Alertmanager being
   Cluster IP services; datasources provisioned via values.yaml.
8. **Session 2**: Loki, Tempo, OTel gateway; then agents on workload
   clusters.

## Reference apps to study

- **Grafana LGTM stack** (Loki/Grafana/Tempo/Mimir) — canonical
  deployment pattern for split backends.
- **kube-prometheus-stack values.yaml** — even though we're not
  using the chart, it's the best single reference for
  prometheus-operator ServiceMonitor/PodMonitor/PrometheusRule
  configuration.
- **Grafana OnCall** — alert routing / incident management
  alternative to Alertmanager, built into the same ecosystem.
- **VictoriaMetrics** — Prometheus-compatible alternative; simpler
  ops for mid-scale.

## Cross-references

- [`QA.md`](./QA.md) — learning Q&A: three pillars, tool comparisons,
  cardinality, multi-cluster patterns, SLI/SLO, OTel, retention.
- `argo-applications/README.md` (workspace root) — selector patterns; this stack
  is Pattern C for backends, Pattern A for agents.
- `forge/README.md` — platform vision (identity+secrets+trust),
  where observability sits in the overall architecture.
- `storage/rook-ceph-external/README.md` — cephx bootstrap as a
  per-consumer prerequisite.
# observability
