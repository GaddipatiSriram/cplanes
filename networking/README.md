# networking — current setup

## What lives here

The **networking plane** holds north-south traffic infrastructure (ingress, gateway, GLB) plus east-west (service mesh) plus the LB layer (MetalLB) that backs LoadBalancer Services on bare metal.

AppSets that drive deployment: `../argo-applications/sre/networking/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **ingress-nginx** | ✅ live (mgmt-forge) | every workload cluster | A | L7 ingress controller |
| **metallb** | ✅ live (mgmt-forge) | every workload cluster | A | L2 LoadBalancer for bare metal |
| **kong** | 📋 placeholder | OKD (CP) + every workload cluster (DP) | B + A | API gateway; auth + rate-limit + transforms |
| **k8gb** | 📋 placeholder | every cluster | A | Multi-cluster GLB; cross-cluster traffic routing via DNS |
| **mesh/istio** | 📋 placeholder | every workload cluster | A | Feature-rich service mesh; pick ONE |
| **mesh/linkerd** | 📋 placeholder | every workload cluster | A | Lightweight service mesh; pick ONE |
| **mesh/cilium** | 📋 placeholder | every workload cluster | A (replaces CNI) | eBPF-based; combines CNI + mesh |

## Dependencies

- **Upstream**: cert-manager (`security/ca-pki/`) for ingress TLS; ESO (`security/secrets/`) for ingress' OAuth credentials.
- **Downstream**: every service that exposes HTTP/HTTPS traffic; service mesh provides east-west mTLS and observability for the data plane.

## Architectural decisions

- **Pick ONE service mesh long-term** — three placeholders exist for evaluation; only one will be deployed.
- **MetalLB L2 mode** (not BGP) for homelab simplicity; BGP would require pfSense FRR config.
- **Kong's CP/DP split**: control plane on OKD (durable, low-traffic config store + xDS); data plane wherever traffic needs proxying. Single all-in-one Kong on OKD is simpler for homelab.

## Known issues

- MetalLB's `bgppeers.metallb.io` CRD self-mutates `caBundle` — `ignoreDifferences` rule in the AppSet.
