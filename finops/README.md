# finops — current setup

## What lives here

The **FinOps plane** measures cluster cost and allocates it back to teams/namespaces. Imaginary dollar amounts on a homelab; same allocation logic as production.

AppSets that drive deployment: `../argo-applications/sre/finops/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **kubecost** | 📋 placeholder | mgmt-observability | C-shaped (single instance, federated reads) | Per-namespace cost allocation, savings recommendations |

## Dependencies

- **Upstream**: observability plane (Prometheus federation reads) for the underlying metric stream; identity plane (OIDC) for UI auth.
- **Downstream**: stakeholder dashboards in Grafana sourced from Kubecost's API.

## Architectural decisions

- **Single Kubecost instance, federated reads** instead of per-cluster Kubecost — avoids running it N times. Trade: a kubecost-agent DaemonSet still runs per-cluster to ship per-pod metrics back.

## Known issues

- See `todo.md` once deployed.
