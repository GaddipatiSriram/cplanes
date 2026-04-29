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

| Component | Capability | Dynatrace equivalent | Status |
|---|---|---|---|
| `prometheus/` | metrics storage + scraping + operator + CRDs | Infra/APM metrics | ✅ done (Session 1) |
| `alertmanager/` | alert routing (Slack/email/PagerDuty/OnCall) | Davis alerting | ✅ done (Session 1) — routes are stub `null` receiver, real ones land with credentials |
| `grafana/` | unified UI for everything | Dynatrace UI | ✅ done (Session 1) — datasources Loki+Tempo+Prometheus+Alertmanager wired; OIDC pending |
| `loki/` | log aggregation, full-text search | Log management | ✅ done (Session 2) — SingleBinary, ceph-rbd 100Gi |
| `tempo/` | distributed traces backend | PurePath / tracing | ✅ done (Session 2) — single-binary, ceph-rbd 100Gi |
| `otel-gateway/` | OTLP ingress (traces + metrics + logs) | OneAgent ingest | ✅ done (Session 2) — gateway only; agents added when needed |
| `pyroscope/` | continuous profiling | Code-level profiling | not started (Session 3) |
| `oncall/` | incident management, on-call rotations | Davis incident routing | not started (Session 3) |
| `blackbox-exporter/` | synthetic HTTP/TCP/ICMP probes | Synthetic checks | not started (Session 2) |
| `k6-operator/` | scripted load + synthetic monitoring | Synthetic scripted | not started (Session 3) |
| `mimir/` | HA-scalable Prometheus replacement | (when needed) | future |

### Agents (every workload cluster, Pattern A — currently in `observability/agents/`)

| Component | Capability | Dynatrace equivalent | Status |
|---|---|---|---|
| `node-exporter/` | host metrics (CPU/RAM/disk/net) | OneAgent infra | ✅ done (Session 2) |
| `kube-state-metrics/` | k8s object state metrics | OneAgent k8s | ✅ done (Session 2) |
| `prometheus-agent/` | local scraper → central remote-write | OneAgent metrics | ✅ done (Session 2) — also ships prometheus-operator CRDs cluster-wide |
| `promtail/` | local log shipper → Loki | OneAgent logs | ✅ done (Session 2) |
| `otel-collector/` (DaemonSet) | local trace/metric/log collection → OTel gateway | OneAgent traces | not started — deferred until a consumer app needs auto-instr |
| `pyroscope-agent/` | per-pod profiling via eBPF | Code profiling | not started (Session 3) |
| `beyla/` | eBPF auto-instrumentation (HTTP/gRPC/SQL spans without code changes) | OneAgent auto-instr | not started (Session 3) |

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

## Deployment order — Sessions 1 + 2 done; Session 3 next

### Session 1 — backends ✅ done
1. ✅ Register mgmt-observability in argocd-sre (cluster Secret with `tier=mgmt role=observ env=nonprod`).
2. ✅ Cephx bootstrap from `storage/rook-ceph-external/generated/mgmt-observ-bundle.yaml`.
3. ✅ rook-ceph-external CSI + StorageClasses on mgmt-observability.
4. ✅ cert-manager / ingress-nginx / ESO selectors broadened.
5. ✅ Prometheus (Pattern C, ceph-rbd, remote-write receiver on).
6. ✅ Alertmanager (3 replicas, ceph-rbd, stub `null` receiver).
7. ✅ Grafana (ceph-rbd PVC, datasources for Prometheus + Alertmanager).

### Session 2 — logs + metrics + tracing fleet ✅ done
8. ✅ Loki SingleBinary on observability + Promtail Pattern A on every mgmt cluster.
9. ✅ Tempo single-binary on observability + OTel gateway (Pattern C).
10. ✅ Metrics fleet Pattern A: node-exporter + kube-state-metrics on every mgmt cluster + prometheus-agent on every cluster except `role=observ` (which already has the central Prometheus).
11. ✅ Grafana datasources extended: Loki + Tempo (with trace-to-logs and trace-to-metrics click-through).
12. ✅ Cert-manager Phase 1: vault-pki ClusterIssuer on every cert-manager cluster, 12 ingress refs migrated from `letsencrypt-prod` → `vault-pki`. All ingresses now serve engatwork.com Root CA-signed certs.

### Session 3 — profiling, synthetic, alerting routes (not started)
13. Pyroscope (continuous profiling backend) + pyroscope-agent / Beyla (per-pod profiling).
14. blackbox-exporter (synthetic HTTP/TCP/ICMP probes).
15. Grafana OnCall (incident management, on-call rotations) — alternative to PagerDuty.
16. Wire real Alertmanager receivers (Slack, email, PagerDuty webhooks) — needs creds via ESO.
17. Wire Grafana OIDC to Keycloak `engatwork` realm (now unblocked since Phase 1 cert work — Keycloak ingress has a real cert).
18. Author per-component dashboards (or import community dashboards as a stopgap).

### Session 4 — security observability + cross-cutting
19. Falco (runtime security threats, eBPF, Pattern A on every cluster).
20. Sloth (SLO YAML → PrometheusRules + recording rules).
21. OpenCost (per-pod cost attribution).
22. Robusta (alert enrichment + auto-remediation).

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
