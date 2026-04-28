# forge/metallb

MetalLB on every workload-capable cluster, deployed via ArgoCD
`argocd-sre`. Umbrella Helm chart wrapping the upstream
`metallb/metallb` chart plus a per-cluster `IPAddressPool` +
`L2Advertisement` pair.

L2-only mode (ARP). No BGP, no FRR.

## Why this exists

Workload clusters need a `LoadBalancer` Service implementation to give
each cluster's `ingress-nginx-controller` a stable VIP. Before this
chart, MetalLB was installed manually-from-raw-manifests on
`mgmt-forge`, **completely missing** on `mgmt-observability` (the
ingress-nginx LoadBalancer sat at `EXTERNAL-IP=<pending>` indefinitely
after the cluster rebuild), and would be re-broken on every future
cluster.

This chart closes the gap with one Pattern A `ApplicationSet` — every
new workload cluster picks up MetalLB the moment it's labeled
`tier=mgmt|dev role!=storage` and registered with `argocd-sre`.

## Files

- `Chart.yaml` — pins `metallb/metallb` `0.14.8` (matches the version
  already running on `mgmt-forge` so adoption-via-`ServerSideApply` is
  a no-version-diff).
- `values.yaml` — subchart resources right-sized for homelab + a
  `pool.*` block driven by the AppSet's per-cluster Helm parameters.
- `templates/ipaddresspool-l2adv.yaml` — renders `IPAddressPool` +
  `L2Advertisement` from `.Values.pool.{name,range,advertisementName}`.
  Sync wave 1 (subchart's CRDs install at wave 0).

## Pattern A wiring

ApplicationSet lives in [`argo-applications/sre/forge/metallb-appset.yaml`](https://github.com/GaddipatiSriram/argo/blob/main/sre/forge/metallb-appset.yaml).
Selector:

```yaml
matchExpressions:
  - { key: tier, operator: In,    values: [mgmt, dev] }
  - { key: role, operator: NotIn, values: [storage] }
```

| Cluster | Targeted? | Why / why not |
|---|---|---|
| `mgmt-forge` | ✅ | workload, hosts ingresses for Vault/Keycloak/ArgoCD/Harbor |
| `mgmt-observability` | ✅ | workload, hosts Grafana/Prometheus/Alertmanager UIs |
| `dev-{web,apps,data}` | ✅ (when registered) | workload |
| `mgmt-storage` | ❌ | excluded by `role NotIn [storage]` — Ceph server, no `LoadBalancer` Services |
| `mgmt-core` | ❌ | not in `argocd-sre` fleet (`argocd_register: false`) — has its own manual MetalLB serving the CoreDNS VIP `10.10.1.200` |
| OKD | ❌ | excluded automatically (no `tier` label) — OpenShift router + keepalived covers L4 |

## Per-cluster pool ranges

Each cluster gets `.200-.220` of its own `/24`, with `.201`
conventionally used for the cluster-local `ingress-nginx-controller`
VIP. Encoded as a goTemplate `if/else` ladder in the AppSet:

| Cluster | Pool range | Ingress VIP (`.201`) |
|---|---|---|
| `mgmt-forge` | `10.10.5.200-10.10.5.220` | `10.10.5.201` |
| `mgmt-observability` | `10.10.3.200-10.10.3.220` | `10.10.3.201` |
| `dev-web` | `10.20.1.200-10.20.1.220` | `10.20.1.201` |
| `dev-apps` | `10.20.2.200-10.20.2.220` | `10.20.2.201` |
| `dev-data` | `10.20.3.200-10.20.3.220` | `10.20.3.201` |

Mirrors the `metallb_ip_pool` entries already declared in
`cluster/k8s-bstrp/group_vars/<cluster>.yml` — extend both files
together when a new cluster joins.

## Gotchas

### 1. Subchart `image.repository` is the **full path**, not split

The upstream chart has no separate `registry` field — `image.repository`
is the full registry+repo string:

```yaml
controller:
  image:
    repository: quay.io/metallb/controller   # ← full path
    tag: v0.14.8
```

Setting `image.registry: quay.io / image.repository: metallb/controller`
silently defaults `repository` to `docker.io/metallb/controller`, which
fails to pull (`ErrImagePull` with `pull access denied, repository does
not exist`). Caused both initial pod crashes during this rollout.

Fixed in [`forge@20cf255`](https://github.com/GaddipatiSriram/forge/commit/20cf255).

### 2. `bgppeers.metallb.io` CRD shows OutOfSync forever

The chart ships the CRD with an empty `caBundle` on its conversion
webhook. The MetalLB controller injects its real caBundle at runtime.
Argo sees the diff every reconcile cycle and flags the Application
`OutOfSync` (despite all components `Healthy`).

Fixed in [`argo@8a7f7ff`](https://github.com/GaddipatiSriram/argo/commit/8a7f7ff)
by adding the field to the AppSet's `ignoreDifferences`:

```yaml
ignoreDifferences:
  - group: apiextensions.k8s.io
    kind: CustomResourceDefinition
    name: bgppeers.metallb.io
    jsonPointers:
      - /spec/conversion/webhook/clientConfig/caBundle
```

### 3. Argo's repo-server caches Helm renders aggressively

Even after pushing a chart fix and refreshing the Application, the
rendered manifest can be stale. If a value override doesn't take
effect on a sync, restart the repo-server before assuming the chart
is wrong:

```bash
kubectl --context admin -n argocd-sre rollout restart deploy argocd-sre-repo-server
```

This bit during the per-cluster `loadBalancerIPs` annotation rollout
on `ingress-nginx-mgmt-observability` — the chart was right, the cache
wasn't.

## Adoption note — `mgmt-forge` (DONE 2026-04-28)

`mgmt-forge` ran MetalLB from raw `kubectl apply` of upstream manifests
since pre-GitOps days. Adoption-by-coexistence wasn't possible because
both DaemonSets would race for the same hostPorts (`7472`/`7473`/`7946`)
and ARP-respond for the same VIPs (MAC-flap territory). The cutover
required dropping the manual install in one shot and letting the chart
take over.

| Resource | Manual name | Chart-rendered name | Conflict? |
|---|---|---|---|
| Deployment | `controller` | `metallb-controller` | port-clash on `7472` (metrics) |
| DaemonSet | `speaker` | `metallb-speaker` | port-clash on `7472`+`7946` (memberlist) |
| IPAddressPool | `mgmt-forge-pool` | `mgmt-forge-pool` | same name → Argo adopted via SSA |
| L2Advertisement | `mgmt-forge-l2` | `mgmt-forge-l2` | same name → Argo adopted via SSA |

### Cutover procedure (executed)

```bash
# 1. Bounce the argocd repo-server first — it caches Helm renders, and
#    we'd just deployed a chart fix (docker.io→quay.io image registry).
#    Without this bounce, the chart Deployment spec still pointed at
#    docker.io/metallb/controller and pods would have hit ImagePullBackOff
#    immediately after the manual install was deleted.
kubectl --context admin -n argocd-sre rollout restart \
  deploy argocd-sre-repo-server
kubectl --context admin -n argocd-sre rollout status \
  deploy argocd-sre-repo-server --timeout=60s

# 2. Hard-refresh the Application so it pulls the new render.
kubectl --context admin -n argocd-sre patch application metallb-mgmt-forge \
  --type=json -p='[{"op":"replace","path":"/operation","value":null}]'
kubectl --context admin -n argocd-sre annotate --overwrite \
  application metallb-mgmt-forge argocd.argoproj.io/refresh=hard

# 3. Verify the chart Deployment now has the right image.
kubectl --context mgmt-forge -n metallb-system get deploy metallb-controller \
  -o=jsonpath='{.spec.template.spec.containers[0].image}'
# → quay.io/metallb/controller:v0.14.8

# 4. Drop the manual install. (Chart speakers were Pending due to
#    hostPort collision; this delete frees the ports so they can schedule.)
kubectl --context mgmt-forge -n metallb-system delete \
  deployment controller
kubectl --context mgmt-forge -n metallb-system delete \
  daemonset speaker

# 5. Watch chart pods come up (~60 s — speakers init containers pull
#    quay.io/frrouting/frr:9.1.0 and quay.io/metallb/speaker:v0.14.8).
kubectl --context mgmt-forge -n metallb-system get pods --watch

# 6. Verify VIP still resolves end-to-end.
curl -k --resolve vault.apps.mgmt-forge.engatwork.com:443:10.10.5.201 \
     https://vault.apps.mgmt-forge.engatwork.com/v1/sys/health
# → HTTP 200 from 10.10.5.201
```

`mgmt-forge-pool` and `mgmt-forge-l2` kept their names through the
cutover — existing VIPs (`10.10.5.201` for ingress-nginx) stayed
allocated, Service annotations on Vault/Keycloak/ArgoCD ingress were
untouched, no kubeconfig edits needed.

### What the actual disruption looked like

Step 4 deleted both manual resources; chart speakers — which had been
Pending on hostPort collision — scheduled within ~3 s and finished
their init containers (FRR copy, metrics setup) ~60 s later. ARP
announcement gap was effectively the speaker init-container window,
because the L2Advertisement object kept pointing at the same pool the
whole time and the chart speakers became authoritative the moment they
hit Running. Post-cutover Vault probe came back HTTP 200 — no client
saw a connection failure during the brief gap because TCP retries
absorbed it.

## Verification

```bash
# AppSet generated one Application per matching cluster.
kubectl --context admin -n argocd-sre get applications \
  -l app.kubernetes.io/name=metallb

# IPAddressPool live with the right range.
kubectl --context mgmt-observability get ipaddresspools -A

# ingress-nginx Service got its VIP.
kubectl --context mgmt-observability get svc -n ingress-nginx \
  ingress-nginx-controller

# Confirm chain end-to-end with a TLS handshake to the VIP.
echo | openssl s_client -connect 10.10.3.201:443 \
  -servername some-future-host.apps.mgmt-observability.engatwork.com \
  2>/dev/null | openssl x509 -noout -subject -issuer
```

## Deployment history

| When | What | Commit |
|---|---|---|
| 2026-04-27 | Chart authored (Chart, values, IPAddressPool/L2Adv template) | [`forge@0050f3e`](https://github.com/GaddipatiSriram/forge/commit/0050f3e) |
| 2026-04-27 | AppSet authored — Pattern A, goTemplate ladder for pool ranges | [`argo@1ccae73`](https://github.com/GaddipatiSriram/argo/commit/1ccae73) |
| 2026-04-27 | Image registry fix (`docker.io` → `quay.io`) | [`forge@20cf255`](https://github.com/GaddipatiSriram/forge/commit/20cf255) |
| 2026-04-27 | `ingress-nginx` AppSet: per-cluster `loadBalancerIPs` via `valuesObject` | [`argo@557d860`](https://github.com/GaddipatiSriram/argo/commit/557d860) |
| 2026-04-27 | bgppeers CRD caBundle `ignoreDifferences` | [`argo@8a7f7ff`](https://github.com/GaddipatiSriram/argo/commit/8a7f7ff) |
| 2026-04-28 | `mgmt-forge` cutover from manual install → chart-managed (Synced+Healthy across the fleet) | (in-cluster, see procedure above) |

## References

- [MetalLB docs](https://metallb.universe.tf/)
- [`argo-applications/README.md`](https://github.com/GaddipatiSriram/argo/blob/main/README.md) — Pattern A/B/C/D selector reference
- [`cluster/k8s-bstrp/group_vars/<cluster>.yml`](https://github.com/GaddipatiSriram/cluster/tree/main/k8s-bstrp/group_vars) — source of per-cluster `metallb_ip_pool`
- [`forge/todo.md`](../todo.md) — outstanding follow-ups
