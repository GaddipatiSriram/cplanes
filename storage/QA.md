# Storage Design Q&A

Record of the design discussion that shaped the storage tier. Kept so the
reasoning behind the chosen architecture is searchable and auditable.

---

## Q1. Can we use the `mgmt-storage` cluster for storage needs?

**A.** Yes — it's the intended target. `mgmt-storage-0{1,2,3}` are
provisioned in oVirt on VLAN 104 (`10.10.4.100-102`). They are Rocky Linux 9
VMs with no Kubernetes bootstrap yet, so we have a clean slate.

---

## Q2. Should we use Longhorn on `mgmt-storage`?

**A. No — dropped.**

Longhorn is a **single-cluster CSI**. Volumes Longhorn creates can only be
mounted by pods running in the same Kubernetes cluster where Longhorn is
installed. Installing Longhorn on `mgmt-storage` would only benefit
workloads on `mgmt-storage` itself — not OKD, not mgmt-core, not dev-data.

Longhorn *can* front RWX volumes with a NFS gateway (ganesha), but at that
point consumers are talking NFS, not Longhorn — Longhorn is only an
implementation detail behind the NFS. Ceph does that job with more features
(snapshots, replication, object + file + block in one backend).

Conclusion: Longhorn is for the *opposite* topology (workloads local to the
storage cluster). Not a fit here.

---

## Q3. Can we use the oVirt data LUN at the host level as common storage?

**A. No — architecturally fragile.**

"Host-level" means exporting from the Rocky 9 hypervisor itself (e.g., NFS
export of a partition on the LUN). Problems:

- The data LUN is already owned by oVirt/VDSM as the **storage domain**.
  It cannot be reformatted for non-oVirt use without destroying VM disks.
- The hypervisor would gain a new responsibility (NFS server). Kernel
  updates, reboots, hardware faults all become storage outages.
- No HA — one host is a single point of failure for every cluster's PVCs.
- Mixes two concerns (virtualization + storage) on one box.

**Better:** create a large oVirt disk on the data LUN, attach it to a VM on
the `mgmt-storage` cluster, and serve from the VM. Same physical LUN, clean
boundary. This is what the plan does (three 500 GB disks, one per
mgmt-storage node, consumed by Ceph OSDs).

---

## Q4. What is the equivalent of "shared EBS" across k8s clusters?

**A.** "Shared EBS" maps to two AWS services: EFS (shared file) and S3
(shared object). True block-level shared attach is rare. On-prem equivalents:

| Need | AWS | On-prem |
|---|---|---|
| Single-attach block | EBS | Ceph RBD, iSCSI |
| Shared file (RWX) | EFS | CephFS, NFS |
| Object / S3 | S3 | Ceph RGW, MinIO |
| HPC parallel FS | FSx Lustre | Lustre, BeeGFS (not needed here) |

For the homelab, **Ceph in external-cluster mode** gives all three from one
backend. NFS + MinIO as an alternative covers ~80% of needs with less
operational surface but fewer features.

---

## Q5. Will Ceph actually cover all our storage needs?

**A. ~95% yes, with specific gaps documented.**

Covered:
- Per-pod block (databases, queues, TSDBs) via RBD
- Shared file (CI workspaces, uploads) via CephFS
- S3 (backups, registry blobs, logs, artifacts) via RGW
- Snapshots, clones, online resize, replication (3×)
- Velero backup targets via RGW

Not covered by this plan:
- Ultra-low-latency block for high-QPS production DBs → add local-NVMe CSI
  on the specific consumer cluster if measured latency matters.
- SMB / Windows file shares → Samba gateway can be added later on Ceph.
- Parallel HPC filesystems (Lustre/BeeGFS) → specialized, not a homelab need.
- Cross-site DR → single-site Ceph. Velero → S3 handles off-cluster backup.
- Non-storage needs that look like storage: Secrets/ConfigMaps (use k8s
  primitives), ephemeral scratch (`emptyDir`), host-local paths for
  DaemonSets (`hostPath`).

---

## Q6. Where does `dev-data` fit — we need storage for app databases?

**A.** `dev-data` is a consumer like any other cluster. App databases want
**block storage (RWO)**, which is `ceph-rbd`. Each DB gets a private,
replicated, snapshottable RBD volume.

```yaml
volumeClaimTemplates:
- metadata: { name: data }
  spec:
    accessModes: [ReadWriteOnce]
    storageClassName: ceph-rbd
    resources: { requests: { storage: 50Gi } }
```

Because `dev-data` is database-centric, its **default StorageClass** is set
to `ceph-rbd` (other consumer clusters default to `cephfs`).

### Latency note for DBs

Ceph RBD over network adds ~1–3 ms per op vs ~50 µs for local SSD. For:
- Dev/staging/CI DBs → invisible, use Ceph.
- Production high-QPS OLTP or real-time analytics → add a `local-nvme`
  StorageClass on `dev-data` for just that workload. Same cluster, second
  tier, no rearchitecture.

For the current scope (homelab dev), Ceph RBD is the right call.

---

## Q7. Explain the whole picture more clearly.

**A.** Summary in plain terms:

1. **Three shapes of storage exist**: block (one-owner disk, for DBs),
   file (shared folder, for CI workspaces), object (S3 API, for backups
   and artifacts).
2. **Ceph provides all three from one backend.** Running Ceph once on
   `mgmt-storage` means every cluster gets every shape.
3. **"External mode"** means Ceph daemons run only on `mgmt-storage`;
   consumer clusters install a tiny CSI driver that talks to Ceph over the
   network. Like installing a database client vs running another database.
4. **End-to-end example for a Postgres on dev-data:**
   a. StatefulSet asks for a `ceph-rbd` PVC.
   b. The external CSI on dev-data calls the Ceph mons on mgmt-storage.
   c. Ceph carves a new RBD image, replicates 3×, returns it.
   d. CSI maps the image to the dev-data node; kubelet mounts it into
      the Postgres pod.
   e. Postgres sees a normal disk. It doesn't know the bytes are
      replicated on another cluster.
5. **Which StorageClass for what workload** — see `README.md` §"Storage
   choices per consumer cluster".

---

## Q8. Should we also run MinIO?

**A.** Not as primary. Ceph RGW is already S3-compatible. Adding MinIO
alongside creates two S3 endpoints to operate for the same purpose.

MinIO may make sense later as a *second* S3 endpoint in a different
cluster for the specific purpose of **backing up Ceph itself** (you don't
want Ceph's backup target to live inside Ceph). That's a future-you
problem — keep the `minio/` folder as a placeholder.

---

## Q. Rook-Ceph is installed on mgmt-storage. What does `rook-ceph-external` do on every other cluster?

**A.** The two halves of a single Ceph deployment:

- **`storage/rook-ceph/`** (mgmt-storage only) runs the actual Ceph:
  operator, MONs, MGRs, OSDs, MDS, RGW. Owns the disks. Serves data.
- **`storage/rook-ceph-external/`** (every consumer cluster) installs the
  **same Rook operator in "external mode"** plus the **CSI plugins**. No
  Ceph daemons. The CephCluster CR on a consumer has
  `spec.external.enable: true`, which tells the operator "don't create
  anything; just reconcile client config."

When a pod on a consumer (e.g. OKD) creates a PVC against `ceph-rbd`, the
local CSI plugin talks to the MONs on mgmt-storage (10.10.4.100-102:3300),
gets the osdmap, creates an RBD image on the remote OSDs, and the kernel
maps it over the network. All I/O flows consumer → mgmt-storage for the
lifetime of the pod.

Rationale for the split: one storage backend, many consumers. You don't
want OSDs on workload clusters (their disks are small, their node count
shifts, workloads fight storage for IO). Rook's external-cluster mode is
designed exactly for this pattern.

Full operational docs + per-PVC flow diagram in
`storage/rook-ceph-external/README.md`.

---

## Open decisions at time of writing

- [ ] Disk size per mgmt-storage node — plan currently says 500 GB × 3. Confirm.
- [ ] Whether to allocate a second data LUN in oVirt specifically for
      Ceph (isolates I/O from VM root disks) or reuse the existing LUN.
- [ ] Default StorageClass per consumer cluster — defaults in the table
      above are proposals; revisit after observing real usage.
- [ ] S3 DNS name — `s3.mgmt-storage.engatwork.com` assumed; needs CoreDNS
      record + route through pfSense.
