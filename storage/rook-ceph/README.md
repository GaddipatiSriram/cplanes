# storage/rook-ceph

Rook-Ceph **central install** on the `mgmt-storage` cluster
(`10.10.4.100-102`) — the server side: operator, MONs, MGRs, OSDs, MDS, RGW.

Paired with **`storage/rook-ceph-external/`** — the client side, CSI-only
install that every other cluster runs so its pods can mount volumes
backed by this cluster. See that README for the per-PVC flow and the
server-vs-client breakdown.

## Files

- `install.yml` — Ansible playbook: helm install Rook operator + apply CephCluster/pools/fs/rgw.
- `values-operator.yaml` — values for `rook-release/rook-ceph` operator.
- `cephcluster.yaml` — `CephCluster` CR (3 mons, 3 mgrs, 3 OSDs on `/dev/vdb`).
- `pools.yaml` — `CephBlockPool replicapool`, `CephFilesystem myfs`, `CephObjectStore my-store`.

## Known issues (catalog)

Running log of issues we've hit with this install. Keep entries terse —
symptom, root cause, fix. Update when solved.

---

### #1 — External consumers hang on volume provisioning (deadline exceeded)

**Status.** **CLOSED** 2026-04-23 — fix committed in `cephcluster.yaml`
(`network.addressRanges.public/cluster = 10.10.4.0/24`) and verified
end-to-end from OKD.

---

**Symptom.** On any cluster outside `mgmt-storage` (OKD, mgmt-forge,
dev-*), provisioning a PVC against `ceph-rbd` or `cephfs` sits in
`Pending` forever. CSI provisioner logs:

```
context deadline exceeded
→ Aborted: operation with the given Volume ID ... already exists (x100s)
```

Deceptively:

- `ceph -s` from the consumer cluster returns `HEALTH_OK`.
- `ceph df`, `ceph osd tree`, `ceph pg stat` all work from consumers.
- Only **OSD-backed** ops (`rbd create`, subvolume create) hang.

**Root cause.** The CephCluster had `network.provider: host` (so OSD
pods ran with `hostNetwork: true` and had the `10.10.4.x` node IP on
`enp1s0`). That's usually enough. But the storage nodes also run
**Cilium**, which gives each pod a SECOND interface on the pod CIDR
(`cilium_host` at `10.245.x.y`) alongside the host address.

When each OSD daemon started, it had to pick which of its addresses to
register as its public address in the **osdmap** (the authoritative
map MONs hand out to clients). With no explicit `public_network`
pinned, Ceph's interface selection is implementation-defined — it
looks at the default route, the first-bound address, etc. Two of the
three OSDs happened to pick `10.10.4.x` (right). **`osd.1` happened
to pick `10.245.1.23`** — the Cilium mesh IP — and `osd.0`'s
cluster-network selection was also on the pod CIDR.

The osdmap is persistent in the mon store. Every client that asks for
the OSD map gets the registered addresses back. From inside the
storage cluster, `10.245.x.y` is routable (pod-to-pod). From anywhere
else it is not. So any client write that landed on a PG primaried by
`osd.1` (≈ 1/3 of all writes) connected to MON fine, received the
osdmap, then hung 10s+ on TCP to `10.245.1.23:6802` and gave up with
`DeadlineExceeded`.

Verified with:

```
rbd --id csi-rbd-provisioner --key <k> \
    --mon-host 10.10.4.100:3300,10.10.4.101:3300,10.10.4.102:3300 \
    -p replicapool --debug-ms=1 create debug-test --size 1G
# trace shows: >> [v2:10.245.1.23:6802/...] ...
# tick see no progress in more than 10000000 us during connecting ... fault
```

And `ceph osd dump` (run from inside the storage cluster) shows what
each OSD actually registered:

```
osd.0 [v2:10.10.4.100:6800/...] [v2:10.245.0.31:6802/...]  ← cluster addr wrong
osd.1 [v2:10.245.1.23:6802/...] [v2:10.245.1.23:6804/...]  ← BOTH wrong
osd.2 [v2:10.10.4.102:6802/...] [v2:10.10.4.102:6804/...]  ← correct
```

**Fix.** Pin both public and cluster networks explicitly in
`cephcluster.yaml`:

```yaml
spec:
  network:
    provider: host
    addressRanges:
      public:  ["10.10.4.0/24"]
      cluster: ["10.10.4.0/24"]
```

Then:

1. `kubectl apply -f cephcluster.yaml` — operator picks up the change.
2. Operator does **not** automatically roll OSD pods (the change
   touches ceph config, not pod templates). Bounce the misregistered
   OSDs one at a time to preserve `min_size: 2`:

   ```
   kubectl -n rook-ceph delete pod -l osd=0
   # wait for 2/2 Ready, verify osd.0 addr via `ceph osd dump`
   kubectl -n rook-ceph delete pod -l osd=1
   ```

   On restart, each OSD re-announces its addresses using only the
   `10.10.4.0/24` interface.
3. Verify from a consumer cluster:

   ```
   rbd --id csi-rbd-provisioner --key <k> --mon-host ... \
       -p replicapool create smoke --size 1G   # should exit 0 in seconds
   ```

**Why it was hard to spot.**

- `ceph -s` and friends *lie* in this failure mode. MON ↔ OSD traffic
  flows over the cluster network, which was self-consistently on
  `10.245.x.y` inside the storage cluster. Ceph itself thought it was
  healthy because internally it was. Only external clients suffered.
- `ceph osd dump` is the one artifact that exposes the advertised
  addresses. If you only run it from inside the storage cluster and
  don't cross-check against what an external client actually sees,
  the mismatch is invisible.
- The noisy symptom was the 800+ retry `Aborted: operation ... already
  exists` log spam (see Issue #2 below). That's a downstream artifact:
  the CSI provisioner retries a timed-out operation before the
  original is fully cleaned up. It obscured the real "DeadlineExceeded"
  failure underneath. The `--debug-ms=1` trace was the breakthrough —
  it's the only thing that shows "I'm trying to connect to X and
  getting nothing back" rather than "operation is stuck."

**Lesson.** For any Rook-Ceph cluster that will be consumed
externally, always pin `addressRanges.public` (and `.cluster`) to the
specific VLAN subnet, even with `provider: host`. Host-network alone
is not enough when the CNI adds extra interfaces to every pod. This
applies to Cilium, Calico with cluster mesh, Multus setups, any
overlay that multi-homes pods.

---

### #2 — Stale PVCs wedge the CSI provisioner workers (800+ retry storm)

**Symptom.** Every new PVC against `ceph-rbd` or `cephfs` fails with
`Aborted: operation with the given Volume ID ... already exists`.
Provisioner logs show a single claim with failure count in the
hundreds that never completes.

**Root cause.** A downstream failure (see #1) leaves an in-flight
CSI CreateVolume operation pending in the provisioner. The workqueue
keeps the stale claim hot and each new PVC retry re-collides with
the in-flight operation key.

**Fix.** Delete the stuck PVC and restart the provisioner pods so
in-memory work queues drain:

```
kubectl --context admin -n default delete pvc <stuck-pvc>
kubectl --context admin -n rook-ceph-external delete pod \
    -l app=csi-rbdplugin-provisioner
kubectl --context admin -n rook-ceph-external delete pod \
    -l app=csi-cephfsplugin-provisioner
```

New provisioner pods take ~30–60s to acquire leader leases before
accepting claims.

**Status.** Workaround only. Address by solving #1 so CSI stops
hanging and creating stale operations in the first place.

---

### #3 — local-path fallback fails on OKD (SCOS /opt is read-only)

**Symptom.** On OKD (SCOS/CoreOS nodes), PVCs against the
`local-path` StorageClass sit `Pending` with
`create process timeout after 120 seconds`. The local-path-provisioner
helper pod logs `mkdir: can't create directory
'/opt/local-path-provisioner/pvc-...': Permission denied`.

**Root cause.** SCOS (and RHCOS) treat `/opt` as part of the
immutable root filesystem. The stock `local-path-provisioner`
`ConfigMap` `local-path-config` defaults `paths` to
`["/opt/local-path-provisioner"]` — unwritable on these nodes.

**Fix (untested, planned).** Point the provisioner at a writable
path under `/var`, and grant the helper pod SCC that permits the
hostPath mount. Rough shape:

```yaml
# kubectl -n local-path-storage edit configmap local-path-config
{
  "nodePathMap": [
    { "node": "DEFAULT_PATH_FOR_NON_LISTED_NODES",
      "paths": ["/var/mnt/local-path-provisioner"] }
  ]
}
```
```
oc adm policy add-scc-to-user hostmount-anyuid \
   -z local-path-provisioner-service-account -n local-path-storage
```
Then bounce the provisioner Deployment. Verify a test PVC binds
before trusting it for real workloads.

**Interim workaround.** `emptyDir` for anything that can tolerate
loss on pod restart. No persistent option on OKD until #1 or #3
is fixed.

**Status.** Open. Both #1 and #3 block real stateful workloads
on OKD.
