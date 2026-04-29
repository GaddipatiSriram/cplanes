# observability — TODO

Backlog ordered by likely sequence.

---

## ✅ Loki + Promtail (Pattern A logs fleet)

**Status:** done (2026-04-29). Loki SingleBinary on mgmt-observability, Promtail DaemonSet on all 4 mgmt clusters. Verified end-to-end: `cluster` label in Loki carries values `[mgmt-control, mgmt-forge, mgmt-observability, mgmt-storage]`, namespace label exposes the full platform.

Decision (settled this session): Loki and OpenSearch will **both** exist, routed by source — operational/app logs to Loki, security/audit/compliance logs to OpenSearch (the latter lives in `cplanes/security/`, not here). No data overlap.

Outstanding follow-ups (Phase 2 TLS work — SESSION-NOTES):
- Drop `tls_config.insecure_skip_verify: true` from the Promtail Loki client once engatwork CA is mounted into the Promtail DaemonSet.

---

## ✅ Metrics fleet (node-exporter + kube-state-metrics + prometheus-agent)

**Status:** done (2026-04-29). Pattern A on all `tier=mgmt` clusters; prometheus-agent excludes `role=observ` since the central Prometheus on mgmt-observability is the receiver.

Verified end-to-end via central Prometheus:
- 12 nodes reporting `node_load1` across all 4 clusters
- 230+ pods reporting `kube_pod_info`
- 11 distinct scrape jobs flowing
- `cluster` external label tagging samples per source cluster

Side benefit: prometheus-operator CRDs are now present on every workload cluster. Promtail's ServiceMonitor (originally disabled because workload clusters had no CRD) is re-enabled in the same change.

Outstanding follow-up:
- Drop `tlsConfig.insecureSkipVerify: true` on the prometheus-agent remote-write client once engatwork CA is mounted into agent pods (Phase 2 TLS).

---

## ✅ Tempo + OTel gateway (tracing backend + ingest)

**Status:** done (2026-04-29). Tempo single-binary on mgmt-observability (filesystem chunk store, 100Gi PVC, OTLP gRPC + HTTP receivers). OTel Collector contrib in gateway mode pipelines traces→Tempo, metrics→Prometheus remote-write, logs→Loki OTLP.

Grafana datasources: Tempo added with `tracesToLogsV2` wired to Loki and `tracesToMetrics` wired to Prometheus — click-through trace→log and trace→metric available in Explore.

Choice on topology: gateway only today (single endpoint at `otel-gateway.observability.svc:4317`). Per-cluster OTel agents deferred until an app needs auto-instrumentation pushing to localhost.

Outstanding follow-up:
- Add per-cluster OTel agent DaemonSet when a real consumer service (or Beyla auto-instr) needs it. Cross-cluster OTLP push via Ingress can also be wired then.
- Loki standalone exporter was removed from otelcol-contrib v0.130+; we use `otlphttp` to Loki's native OTLP endpoint (Loki v3.x feature).

---

## Wire Grafana OIDC to Keycloak `engatwork` realm

**Status:** unblocked (Keycloak ingress now has a vault-pki cert from Phase 1 cert work). Grafana still uses the bootstrap admin password from a Secret; OIDC config can land any session.

Once Grafana's OAuth points at Keycloak's discovery URL, the user logs in via SSO and is mapped to admin role via the `groups` claim (`platform-sre`/`platform-users`). Done = login at `grafana.apps.mgmt-observability.engatwork.com` redirects to Keycloak, lands back signed in.

The `apps-launcher`-style flow already works on mgmt-control's argocd; this is the same pattern with Grafana as the consumer. Add a Crossplane Client CR `grafana-oidc` in `identity/keycloak-config/`, mirror client_secret to Vault, ESO sync into observability ns, point grafana values' `auth.generic_oauth` block at it.

---

## Author Grafana dashboards

**Status:** not started. Backends are running but Grafana shows zero dashboards on home page.

Two paths:
- Quick: import community dashboards via the chart's `dashboards:` block — Node Exporter Full (1860), kube-state-metrics overview (13332), Loki/Promtail (17320), Tempo Operational (14642).
- Long: each chart in the platform ships its own ConfigMap labeled `grafana_dashboard: "1"` (sidecar auto-discovery is enabled). Pattern matches "every chart owns its own alerts via PrometheusRule, every chart owns its dashboards via ConfigMap."

Pick one to start; both can coexist.

---

## Decide Loki vs OpenSearch for log storage

**Status:** **decided** — both, routed by source. Loki = operational/app logs, OpenSearch = security/audit/compliance. OpenSearch will live in `cplanes/security/` not `cplanes/observability/`. No data overlap.
