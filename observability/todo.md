# observability — TODO

Backlog ordered by likely sequence.

---

## Add Loki + Promtail Pattern A agent

**Status:** not started.

Loki on `mgmt-observability` (Pattern C) + Promtail DaemonSet on every workload cluster (Pattern A) shipping logs to it. Done = `loki -> grafana datasource` queries return logs from at least 2 source clusters.

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
