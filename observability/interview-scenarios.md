# observability — interview scenarios

---

## Scenario 1: Grafana datasource for Prometheus is RED ("not found")

**Context.** Fresh deploy of `prometheus-mgmt-observability` and `grafana-mgmt-observability`. Both Synced+Healthy. Grafana home shows Prometheus datasource as RED.

**The ask.** Diagnose.

**Walk-through.**
- Open Grafana → Datasources → Prometheus → Test. Errors usually show: connection refused, name resolution, 404 on `/api/v1/query`.
- If "name resolution failed": the datasource URL probably has `prometheus-server.observability.svc.cluster.local:9090`, but the Service might be named `prometheus-kube-prometheus-prometheus` (kube-prometheus-stack convention). Fix the URL.
- If "connection refused": the Prometheus pod might be CrashLoopBackOff. `kubectl logs` to see why — common causes: PVC didn't bind (Ceph CSI not installed yet on this cluster), or RBAC for Prometheus operator's CRDs failed.
- If "404 on /api/v1/query": pointing at the operator's API instead of the Prometheus instance Service. Wrong port or path.

**Tradeoffs.**
- Hard-coding the Service name in Grafana's datasource provisioning is brittle when the chart changes naming conventions. Better: expose the URL as a chart value.
- Could use Service name from a templated ConfigMap, but that adds reconciliation surface.

**Follow-up questions.**
- "Why is the chart called `kube-prometheus-stack` and not just `prometheus`?" → It's an umbrella that bundles Prometheus, Alertmanager, the operator, and its CRDs. The standalone `prometheus` chart doesn't ship the operator.
- "How would you scrape from workload clusters into this central Prometheus?" → Remote-write push from per-cluster Prometheus agents, OR pull (federate) — push is preferred at fleet scale.

---

## Scenario 2: Alertmanager fires duplicate alerts after a network blip

**Context.** A 30-second network partition between Alertmanager pods. After recovery, all 3 replicas dispatch the same alert independently → 3x notifications to Slack/email.

**The ask.** Why and how to prevent.

**Walk-through.**
- Alertmanager uses gossip-based clustering. During a partition, each pod thinks it's the only one and dispatches independently. After healing, the gossip protocol reconciles, but the alerts have already been sent.
- Mitigations:
  - Run a single replica (acceptable on homelab, loses HA).
  - Use silence stickiness: receivers (PagerDuty, etc.) deduplicate based on alert fingerprint. Slack doesn't natively.
  - Set `--cluster.peer-timeout` longer to absorb short partitions before any pod decides "I'm alone."
- The fundamental issue: Alertmanager's HA is best-effort, not strongly consistent.

**Tradeoffs.**
- Trade HA (replicas) for dedupe simplicity (single replica). Or: accept rare duplicates as the cost of HA.
- Could put Alertmanager behind a single ingress and route all dispatches through one path — adds a SPOF.

**Follow-up questions.**
- "What happens if Alertmanager itself is down when an alert should fire?" → Prometheus retains the alert in the AlertingRule's state; it dispatches when AM comes back, but `for:` clauses may have reset, leading to delayed notifications.
- "How would you test the gossip behavior?" → Inject a NetworkPolicy blocking `alertmanager-mesh` Service for 30s, watch logs.
