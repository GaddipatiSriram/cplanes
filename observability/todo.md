# observability — TODO

Backlog ordered by likely sequence.

---

## Add Loki + Promtail Pattern A agent

**Status:** done (2026-04-29). Loki SingleBinary on mgmt-observability, Promtail DaemonSet on all 4 mgmt clusters. Verified end-to-end: `cluster` label in Loki carries values `[mgmt-control, mgmt-forge, mgmt-observability, mgmt-storage]`, namespace label exposes the full platform.

Decision (settled this session): Loki and OpenSearch will **both** exist, routed by source — operational/app logs to Loki, security/audit/compliance logs to OpenSearch (the latter lives in `cplanes/security/`, not here). No data overlap.

Outstanding follow-ups under the broader TLS workaround push (SESSION-NOTES):
- Drop `tls_config.insecure_skip_verify: true` from the Promtail Loki client once cert-manager produces a real cert for `loki.apps.mgmt-observability.engatwork.com`.
- Re-enable `serviceMonitor.enabled: true` in promtail values once `agents/prometheus-agent/` ships and brings the prometheus-operator CRDs to every cluster.

---

## Add Tempo + OTel collector

**Status:** not started.

Distributed tracing backend on `mgmt-observability` + OTel collector agent (Pattern A) on every workload cluster forwarding traces. Decide: OTel collector "agent" mode per-cluster + central "gateway" on observ, or single gateway only. Done = sample app's trace appears in Grafana's Tempo datasource.

---

## Wire Grafana OIDC to Keycloak `forge` realm

**Status:** blocked on Keycloak ingress cert migration to Vault PKI (`identity/keycloak/QA.md` Q11).

Once Keycloak's discovery URL is trusted (signed by Vault PKI's CA, which ESO injects as caBundle into all consumers), Grafana's OAuth config picks up the realm. Done = login at `grafana.apps.mgmt-observability.engatwork.com` redirects to Keycloak, MFA prompt, redirect back, admin role assigned via group claim.

---

## Decide Loki vs OpenSearch for log storage

**Status:** open question; will surface when search/ tier deploys.

OpenSearch (going to live on `mgmt-observability` per the search co-location plan) is also a log search engine. Running both is wasteful. Either: (a) Loki for app logs + OpenSearch for SIEM/security logs only; (b) OpenSearch for everything, drop Loki. Decide before either is wired up.
