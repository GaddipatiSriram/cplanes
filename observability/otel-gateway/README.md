# otel-gateway

OpenTelemetry Collector in **gateway** mode on `mgmt-observability` —
single endpoint apps push OTLP to, fans out to Tempo (traces),
Prometheus (metrics), and Loki (logs).

For domain context see [`../README.md`](../README.md). For agent-vs-gateway
trade-off discussion see the chart header in [`Chart.yaml`](./Chart.yaml).

---

## What this chart deploys

- `Deployment/otel-gateway` running `otel/opentelemetry-collector-contrib`
  v0.150.1 (the contrib image — has all receivers/processors/exporters).
- `Service/otel-gateway` (ClusterIP) exposing OTLP gRPC :4317, OTLP HTTP
  :4318, and self-metrics :8888.
- `ServiceMonitor` so the central Prometheus scrapes the gateway's own
  metrics (received batches, dropped spans, queue depth).

## Pipelines

```
                ┌─ otlp/tempo (gRPC) → tempo.observability.svc:4317
otlp:4317  ─────┤
otlp:4318  ─────┼─ prometheusremotewrite → prometheus-operated.observability.svc:9090
                │
                └─ loki → loki.observability.svc:3100
```

Three pipelines configured in `values.yaml`:
- `traces`: receivers=[otlp] → batch+memory_limiter → exporters=[otlp/tempo]
- `metrics`: receivers=[otlp, self-prometheus-scrape] → batch+memory_limiter → exporters=[prometheusremotewrite]
- `logs`: receivers=[otlp] → batch+memory_limiter → exporters=[loki]

## App-side push

From any pod inside `mgmt-observability`:
```yaml
env:
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-gateway.observability.svc:4317"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: grpc
```

From other clusters: cross-cluster OTLP push via Ingress isn't wired yet
(gRPC over Ingress is fiddly). Until then, add per-cluster OTel agent
DaemonSets when an app needs trace ingest from outside `mgmt-observability`.

## Why gateway and not agent?

Trade-offs:

| Topology | Pods | Pro | Con |
|---|---|---|---|
| **Gateway only** (today) | 1 on observability | Single endpoint, easy to apply tail-sampling/redaction centrally, no DaemonSet sprawl | One network hop per push; cross-cluster apps need Ingress |
| **Agent + Gateway** | DaemonSet on every cluster + 1 gateway | Apps push localhost:4317 (zero-network ingest), agents handle batching | 25+ collector pods across the homelab |
| **Agent only** | DaemonSet on every cluster | Distributed by default | No central place for routing/sampling, app-to-cluster coupling |

We start with gateway and add agents when an app needs them. The
processing config (sampling, attribute redaction) lives in this chart so
adding agents later is purely an ingest-path change.

## Migration to per-cluster agents (when needed)

1. Author `cplanes/observability/agents/otel-collector/` (Pattern A,
   DaemonSet mode).
2. Each agent's exporter points at `https://otel-gateway.apps.mgmt-observability...`
   (Ingress) or — once the Ingress with vault-pki TLS is in place —
   directly at the gateway's Service via cross-cluster routing.
3. Apps push `localhost:4317` instead of the gateway URL; agent forwards.

## Common operations

**Verify a span lands in Tempo end-to-end** —
```bash
# from any pod on observability cluster:
curl -X POST http://otel-gateway.observability.svc:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d @sample-span.json

# Then in Grafana → Explore → Tempo, search by service name.
```

**Inspect dropped batches** — Grafana → Explore → Prometheus:
```
otelcol_exporter_send_failed_spans
otelcol_processor_dropped_spans
```

**Tail logs** — `kubectl -n observability logs deploy/otel-gateway -f`.
