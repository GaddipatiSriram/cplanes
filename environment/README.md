# environment — current setup

## What lives here

The **environment plane** holds sustainability and energy-aware lifecycle tooling. Per-pod power attribution (Kepler), idle-workload sleep (kube-green) — measuring and reducing the platform's environmental footprint.

AppSets that drive deployment: `../argo-applications/sre/environment/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **kube-green** | 📋 placeholder | every workload cluster (especially dev-*) | A | Sleeps idle deployments at night/weekends |
| **kepler** | 📋 placeholder | every workload cluster | A (DaemonSet) | eBPF-based per-pod energy attribution; exports Prometheus metrics |

## Dependencies

- **Upstream**: observability plane (Prometheus scrape targets for Kepler metrics); cluster control plane (kube-green needs RBAC to scale Deployments).
- **Downstream**: Grafana dashboards for energy metrics; possibly automated rightsizing controllers in the future.

## Architectural decisions

- **Idle reclamation in dev-* clusters first** — kube-green is most useful where idle workloads are common. Production should not sleep without explicit per-namespace policy.
- **eBPF for energy attribution** — Kepler uses CPU performance counters; lower overhead than userspace tap probing.

## Known issues

- See `todo.md` once deployed.
