# storage — interview scenarios

---

## Scenario 1: Ceph cluster `HEALTH_WARN` with `PG_DEGRADED`

**Context.** `ceph -s` from the toolbox pod on `mgmt-storage` shows `HEALTH_WARN`, with `PG_DEGRADED: 24 pgs degraded`. The cluster has 3 OSDs.

**The ask.** Diagnose and recover.

**Walk-through.**
- **Identify which OSD is missing**: `ceph osd tree`. Look for any `down` OSDs.
- **If an OSD is `down`**: check its pod — `kubectl get pods -n rook-ceph -l app=rook-ceph-osd`. Likely `CrashLoopBackOff`. `kubectl logs` to see why. Common causes:
  - Disk failed (read errors in logs) → replace disk, re-create OSD.
  - Slow disk / IO timeout → check `ceph osd perf` for high latency.
  - OSD's pod evicted on node memory pressure → check node `kubectl top node`.
- **If all OSDs are `up`**: it's a backfill/recovery in progress. `ceph -s` will show "X% degraded objects, recovering at Y MB/s." Just wait.
- **Tune**: if recovery is slow, `ceph config set osd osd_max_backfills 4` (default 1) — more aggressive recovery, more client IO impact.

**Tradeoffs.**
- `osd_max_backfills` higher = faster recovery but client IO suffers (hourly throughput drops 30-50%). Choose based on whether you're in maintenance window.
- Could over-replicate (size 3 → 4) to absorb future single failures, but doubles rebuild time when an OSD dies.

**Follow-up questions.**
- "What if 2 of 3 OSDs are down simultaneously?" → With size=3 / min_size=2 pools, IO blocks. Recovery requires bringing at least one back. Practice: OSD logs for clues. Worst case: redeploy with `--no-data-loss` flag if metadata is intact.
- "How would you alert on this?" → Prometheus alert: `up{job="rook-ceph-mgr"} == 0` for the manager; Ceph-specific exporter for `ceph_health_status != 0`.

---

## Scenario 2: A CephFS PVC mount fails with `MountVolume.SetUp failed: rpc error: timed out`

**Context.** A new namespace's deploy creates a PVC against `cephfs` StorageClass. PVC `Bound`, but the consumer Pod hangs in `ContainerCreating` with the timeout error.

**The ask.** Diagnose.

**Walk-through.**
- This is a Ceph CSI issue, not a Ceph cluster issue. The PVC bound (provisioner ran), but mount is failing.
- Check the CSI driver pods on this consumer cluster: `kubectl get pods -n rook-ceph-external -l app=csi-cephfsplugin`. Each node has one. If any are CrashLoopBackOff, mount fails on that node.
- If pods are running but mount times out: check the cephx Secrets in the consumer cluster. `kubectl get secret -n rook-ceph-external | grep csi-cephfs`. Should see `rook-csi-cephfs-{node,provisioner}`. If missing, the bundle apply didn't complete.
- Check connectivity from the consumer to mgmt-storage: `kubectl exec -it -n rook-ceph-external <csi-pod> -- nc -zv 10.10.4.<mon-ip> 6789` (or 3300 for v2). pfSense routing rule between consumer's VLAN and VLAN 13 must allow mon ports.
- Check `dmesg` on the affected k8s node: kernel ceph driver errors show up there. Common: `libceph: bad option -o name=...` (cephx mismatch) or "no route to host" (firewall / pfSense).

**Tradeoffs.**
- Could fall back to RBD instead of CephFS for this workload (RBD usually has fewer mount issues; CephFS needs the kernel module).
- Could pre-provision a "warmed" PVC at namespace creation to surface mount issues before the app deploys.

**Follow-up questions.**
- "What's the difference between `ceph-rbd` and `cephfs` for this use case?" → RBD = block device (one-pod-at-a-time, fast). CephFS = shared filesystem (multi-pod ReadWriteMany, slower). PVC requirements determine which.
- "How would you migrate a PVC from CephFS to RBD?" → Backup-and-restore via Velero, since RWX-vs-RWO change isn't in-place.
