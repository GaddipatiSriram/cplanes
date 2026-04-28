# Storage Tier

Cross-cluster storage backend for the `engatwork` homelab.
Runs on the `mgmt-storage` Kubernetes cluster and serves block, file, and
object storage to every other cluster (OKD, mgmt-core, dev-*, mgmt-observ,
mgmt-forge) via an external CSI driver and an S3 endpoint.

> **Open backlog**: see [`todo.md`](./todo.md).

## Status

| Piece | State |
|---|---|
| **mgmt-storage cluster** (3-node k8s on 10.10.4.100-102) | ✅ up, HEALTH_OK |
| **Disks** (3 × 500 GB OSD disks, 3 × 50 GB root disks) | ✅ provisioned via `cluster/mgmt-storage-prep/` |
| **Rook-Ceph server** (operator + CephCluster + pools + FS + RGW) | ✅ running, Argo-managed (Stage 2 complete) |
| **Central cluster OSD host-IP fix** (addressRanges.public/cluster) | ✅ landed — see `rook-ceph/README.md` issue #1 |
| **Argo-managed server-side Helm charts** (`rook-ceph/{operator,cluster}/`) | ✅ Synced+Healthy on mgmt-storage |
| **Argo Applications in argocd-sre** (`rook-ceph-{operator,cluster}-mgmt-storage`) | ✅ both Synced+Healthy |
| **Ansible legacy install path** (`rook-ceph/legacy/`) | ✅ retired + deleted |
| **Consumer-side CSI install on OKD** (`rook-ceph-external/`) | ⚠️ installed via Ansible (pre-dates Stage 2); not yet Argo-managed. Jenkins PVC relies on it. |
| **Consumer-side CSI install on mgmt-forge** | ❌ not installed — blocks any mgmt-forge workload wanting Ceph storage |
| **Consumer-side CSI install on dev-* / mgmt-core / mgmt-observ** | ❌ not installed |
| **Stage 3 — Argo-ify consumer side** (`rook-ceph-external/` → Argo Applications per consumer) | 📋 planned, see `todo.md` |
| **Stage 4 — cephx key rotation via Vault + ESO** | 📋 planned, see `todo.md` |
| **Ceph backup strategy** | ❌ not started |
| **Ceph monitoring** (Prometheus exporters) | ❌ disabled; depends on mgmt-observ coming up |
| **S3 endpoint DNS + pfSense route** (`s3.mgmt-storage.engatwork.com`) | ❌ not configured; no consumers for object store yet |

## Goals

- One storage backend, all three shapes (block / file / object).
- Consumer clusters install only a lightweight client; no storage daemons
  run outside `mgmt-storage`.
- Reuse the existing oVirt data LUN — do not require new physical disks.
- No storage logic on the hypervisor host.
- Snapshots, resize, and replication out of the box.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  mgmt-storage cluster  (Rocky Linux 9 · k8s 1.31 · 10.10.4.100-102) │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  Rook-Ceph operator                                                 │
│    ├── CephCluster           (3 mons · 3 mgrs · 3+ OSDs)            │
│    ├── CephBlockPool         → StorageClass   `ceph-rbd`   (block) │
│    ├── CephFilesystem        → StorageClass   `cephfs`     (file)  │
│    └── CephObjectStore       → S3 endpoint    `ceph-rgw`           │
│                                                                     │
│  Ceph OSDs back onto oVirt disks carved from the data LUN:          │
│    mgmt-storage-01  ──  vdb (500 GB raw)                            │
│    mgmt-storage-02  ──  vdb (500 GB raw)                            │
│    mgmt-storage-03  ──  vdb (500 GB raw)                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
         │                        │                        │
         │ external-cluster       │ external-cluster       │ HTTPS S3
         │ Rook-Ceph CSI          │ Rook-Ceph CSI          │
         ▼                        ▼                        ▼
┌──────────────┐         ┌──────────────┐         ┌────────────────┐
│     OKD      │         │  mgmt-core   │         │   dev-data     │
│(mgmt-devops) │         │              │         │                │
│              │         │              │         │                │
│ ceph-rbd     │         │ ceph-rbd     │         │ ceph-rbd  ★    │
│ cephfs       │         │ cephfs       │         │ cephfs         │
│ ceph-rgw     │         │ ceph-rgw     │         │ ceph-rgw       │
└──────────────┘         └──────────────┘         └────────────────┘

  (dev-apps, dev-web, mgmt-observ, mgmt-forge connect the same way)

  ★ dev-data uses ceph-rbd as its default — app databases get
    private, replicated, snapshottable block volumes.
```

## Three shapes, one backend

| Shape | Ceph component | StorageClass / endpoint | Access | Use for |
|---|---|---|---|---|
| **Block** | RBD | `ceph-rbd` | RWO (one pod) | Databases, message queues, Jenkins master home, Prometheus TSDB |
| **File** | CephFS | `cephfs` | RWX (many pods) | CI shared workspaces, ArgoCD repo cache, shared uploads |
| **Object** | RGW | `https://s3.mgmt-storage.engatwork.com` (S3 API) | any HTTP client | Backups (Velero), registry blobs (Harbor/Quay), Loki chunks, Tekton Results, CI artifacts |

## Storage choices per consumer cluster

| Cluster | Default SC | Also uses | Notes |
|---|---|---|---|
| **OKD** (mgmt-devops) | `cephfs` | `ceph-rbd`, `ceph-rgw` | Tekton workspaces (RWX) + Jenkins master (RBD) |
| **mgmt-core** | `cephfs` | `ceph-rbd` | Internal services + CoreDNS |
| **dev-data** | **`ceph-rbd`** | `ceph-rgw` | App databases — block is the right fit |
| **dev-apps** | `cephfs` | `ceph-rbd` | Shared app assets/configs |
| **dev-web** | `cephfs` | `ceph-rbd` | Static uploads |
| **mgmt-observ** | `ceph-rbd` | `ceph-rgw` | Prometheus TSDB + Thanos/Loki to S3 |
| **mgmt-forge** | `cephfs` | `ceph-rgw` | Build caches |

## Capacity planning (initial)

| Item | Value | Rationale |
|---|---|---|
| Replicas (Ceph `size`) | 3 | Matches 3 mgmt-storage nodes |
| Raw capacity per node | 500 GB | Pulled from oVirt data LUN as a single disk |
| Total raw | 1.5 TB | |
| **Usable capacity** | **~450 GB** | `raw / replicas` minus Ceph overhead (~10%) |
| Reserve for growth | Leave 30% headroom in Ceph | Ceph `nearfull` alerts trigger at 85% |

Grow later by attaching additional oVirt disks (Ceph detects and onboards new OSDs automatically).

## What this plan does NOT include

Documented gaps so nobody is surprised:

- **Ultra-low-latency block** for high-QPS production DBs. Add a local-NVMe
  CSI (OpenEBS LocalPV) on the specific consumer cluster *only* if measured
  latency is a problem.
- **SMB / Windows file shares.** Not native. Samba gateway can be added on
  the Ceph cluster later.
- **Cross-site disaster recovery.** Single-site Ceph. Velero + S3 (RGW) is
  the path for backups off-cluster.
- **Longhorn, MinIO.** Evaluated, see `longhorn/README.md` and
  `minio/README.md` for why they're not the primary choice.

## Folder layout

```
storage/
├── README.md                   ← this file (plan + architecture)
├── QA.md                       ← Q&A log from design discussion
├── rook-ceph/                  ← Rook-Ceph install on mgmt-storage (server side)
│   ├── operator/               ← umbrella chart → rook-release/rook-ceph (operator+CRDs)
│   ├── cluster/                ← umbrella chart → rook-release/rook-ceph-cluster (CephCluster+pools+fs+RGW)
│   └── README.md               ← issues catalog + pattern docs
├── rook-ceph-external/         ← CSI-only install on each consumer cluster
├── minio/                      ← Optional secondary S3 (not primary)
└── longhorn/                   ← Evaluated, not chosen — see notes
```

Rook-Ceph uses the **two-chart GitOps pattern** recommended by Rook upstream:
operator/ (wave 0) installs the operator + CRDs; cluster/ (wave 1) installs
the CephCluster CR + pools + filesystem + object store as Helm values. Both
are managed by `argocd-sre` via Applications in
`argo-applications/sre/mgmt-storage/rook-ceph-{operator,cluster}-app.yaml`.

## Related layers (live in other repos)

- **mgmt-storage node disk prep** (VM-layer work, lives alongside
  `deploy_vms.yml`):
  `ovirt-setup/playbooks/compute/mgmt-storage-prep/` — resizes root
  disks via the oVirt API, creates the dedicated 500 GB Ceph OSD
  disks per VM, rescans inside each VM so Rook can pick them up.
- **k8s cluster bootstrap** on those VMs:
  `cluster/k8s-bstrp/` with inventory `inventory/mgmt-storage.ini` —
  produces the kubeconfig that everything in this repo consumes.

## Build order

1. `ovirt-setup/playbooks/compute/mgmt-storage-prep/01-prepare-disks.yml`
   — grow root disks and attach the dedicated Ceph OSD disks for the
   mgmt-storage VMs (Step 2 in the original design).
2. `cluster/k8s-bstrp/bootstrap-k8s.yml` with `-i inventory/mgmt-storage.ini`
   — install k8s on the three VMs, merge the kubeconfig, register the
   cluster with argocd-sre (handled by Phase 8 of the bootstrap).
3. Register mgmt-storage with argocd-sre and apply the two Rook
   Applications — see `argo-applications/sre/mgmt-storage/README.md` for the
   6-step runbook. Wave 0 = operator, wave 1 = CephCluster + pools.
   Wait for `HEALTH_OK`.
4. `storage/rook-ceph-external/generate.yml` — wraps Rook's
   `create-external-cluster-resources.py` to emit the env file and
   per-cluster bundles.
5. `storage/rook-ceph-external/apply.yml` — per consumer cluster,
   install Rook in external mode + CSI + StorageClasses.
6. Set the cluster-appropriate default StorageClass in each consumer.
7. Reinstall pending workloads that were blocked on storage (Jenkins,
   Tekton Results, ArgoCD repo server, etc.).

## References

- [Rook-Ceph external cluster docs](https://rook.io/docs/rook/v1.15/CRDs/Cluster/external-cluster/)
- [CephFS CSI](https://github.com/ceph/ceph-csi)
- Local context: `ovirt-setup/README.md`, `cluster/README.md`, `okd/README.md`
# storage
