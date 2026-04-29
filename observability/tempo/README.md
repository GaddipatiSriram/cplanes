# tempo

Distributed tracing backend on `mgmt-observability`. Single-binary mode,
filesystem-backed on `ceph-rbd`. Same architecture choices as the Loki
single-binary — trades scale for ops simplicity.

For domain context see [`../README.md`](../README.md). For per-receiver
config and migration path to distributed mode see the chart header in
[`Chart.yaml`](./Chart.yaml).

---

## What this chart deploys

- `Tempo` v2.9.0 single-binary StatefulSet, 100 GiB PVC.
- `Service/tempo` (ClusterIP) exposing OTLP gRPC :4317 and HTTP :4318.
- `Ingress/tempo` at `https://tempo.apps.mgmt-observability.engatwork.com`
  for cross-cluster OTLP HTTP push (vault-pki cert).
- `ServiceMonitor` so the central Prometheus scrapes Tempo's own metrics.

## Receivers

| Protocol | Port | Use |
|---|---|---|
| OTLP gRPC | 4317 | Primary — OTel gateway pushes here |
| OTLP HTTP | 4318 | Browsers, curl, low-power clients |

## Datasource wiring (Grafana)

Defined in `observability/grafana/values.yaml`. **Trace-to-Logs** is
configured via `tracesToLogsV2` so clicking a span in Tempo jumps to
the corresponding Loki logs filtered by namespace+pod. Trace-to-Metrics
similarly bridges to Prometheus.

This is the killer feature of having Loki + Tempo on the same Grafana —
debug a slow request by clicking from trace → relevant log lines without
copy-pasting timestamps.

## Migration path: filesystem → S3

When Ceph RGW is stood up, swap:

```yaml
tempo:
  tempo:
    storage:
      trace:
        backend: s3
        s3:
          endpoint: rgw.rook-ceph-external.svc:80
          bucket: tempo-traces
          access_key: <from ESO>
          secret_key: <from ESO>
          insecure: true
```

…and consider switching the chart from `grafana/tempo` (single-binary) to
`grafana/tempo-distributed` for HA.

## Common operations

**Verify ingest** — push a fake span via curl:
```bash
curl -X POST http://tempo.observability.svc:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d @sample-span.json
```

**Tail logs** — `kubectl logs -n observability sts/tempo -f`. Watch for
"received traces" lines.

**Query directly** — Tempo HTTP API at port 3100:
```bash
curl 'http://tempo.observability.svc:3100/api/search' \
  --data-urlencode 'tags=service.name=my-app'
```
