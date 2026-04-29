# loki

Central log aggregation backend on `mgmt-observability`. Single-tenant,
single-binary, filesystem-backed.

For domain context (why split, how it fits with OpenSearch later) see
[`../README.md`](../README.md). For the long-form chart rationale (mode
choice, gateway-off, retention math) see the header in
[`Chart.yaml`](./Chart.yaml).

---

## What this chart deploys

- `Loki` v3.6.7 as a **SingleBinary StatefulSet** (1 replica, 100 GiB
  PVC on `ceph-rbd`).
- `Service/loki` (ClusterIP, port 3100) — the API.
- `Ingress/loki` at `https://loki.apps.mgmt-observability.engatwork.com`
  for cross-cluster Promtail push + ad-hoc curl. Cert from Let's
  Encrypt prod via cert-manager.
- `ServiceMonitor` (provided by the upstream chart) so the
  prometheus-operator scrapes Loki's own metrics.

## What's deliberately OFF

| Component | Why |
|---|---|
| Gateway (nginx sidecar) | Single-tenant, no header rewriting needed — direct ingress |
| MinIO | Filesystem store today; will move to Ceph RGW |
| chunksCache / resultsCache (memcached) | Caching is a scale concern, not a homelab one |
| lokiCanary | Synthetic prober — overkill at this size |
| selfMonitoring | We already run prometheus-operator |

## Datasource wiring (Grafana)

Defined in `observability/grafana/values.yaml`:
```yaml
- name: Loki
  uid: loki
  type: loki
  url: http://loki.observability.svc:3100
```

In-cluster Service URL — Grafana lives in the same cluster as Loki, no
ingress hop needed.

## Promtail wiring (cross-cluster)

Promtail (Pattern A DaemonSet on every workload cluster) pushes to
`https://loki.apps.mgmt-observability.engatwork.com/loki/api/v1/push`.
That hostname resolves to ingress-nginx on mgmt-observability; nginx
proxies to `Service/loki:3100` in-cluster. See
[`../agents/promtail/README.md`](../agents/promtail/README.md).

## Migration path: filesystem → S3

When Ceph RGW is stood up under `storage/`, this chart can move to
distributed-friendly object storage with two values changes:

```yaml
loki:
  loki:
    storage:
      type: s3
      s3:
        endpoint: rgw.rook-ceph-external.svc:80
        access_key_id: <from ESO>
        secret_access_key: <from ESO>
        bucketnames: loki-chunks
        s3forcepathstyle: true
        insecure: true
  deploymentMode: SimpleScalable   # optional; allows replicas
```

Promtail clients, Grafana datasource, ServiceMonitor — all unchanged.

## Common operations

**Tail logs** — `kubectl logs -n observability sts/loki -f`. Same pod
holds ingester + querier + compactor.

**Query directly** — `curl 'http://loki.observability.svc:3100/loki/api/v1/query_range' --data-urlencode 'query={namespace="observability"}'`.

**Inspect retention** — Grafana → Explore → Loki → set time range >30d
and query something old; should return empty (compactor purged it).

**Bump retention** — change `loki.loki.limits_config.retention_period`,
PR. Note that increasing retention without increasing PVC size will
crash Loki when disk fills.
