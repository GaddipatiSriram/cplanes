# storage/rook-ceph-external

**Consumer side** of Rook-Ceph — the CSI-only install that every non-
mgmt-storage cluster runs so pods on that cluster can mount volumes
backed by the central Ceph on mgmt-storage.

Pair with `storage/rook-ceph/` (server side, the actual Ceph cluster).

## What this directory is

```
rook-ceph-external/
├── generate.yml              ← Step 4: run once after Ceph is healthy.
│                               Wraps Rook's upstream
│                               create-external-cluster-resources.py
│                               (pinned in scripts/) and emits a
│                               sourceable env file + per-cluster bundles.
├── apply.yml                 ← Step 5: loop over every consumer cluster
│                               and install the CSI client stack.
├── scripts/
│   ├── create-external-cluster-resources.py  ← pinned from Rook release-1.15
│   └── import-external-cluster.sh            ← pinned from Rook release-1.15
├── templates/
│   └── consumer-bundle.yaml.j2               ← per-cluster manifest template
└── generated/
    ├── external-cluster.env                   ← env emitted by generate.yml
    ├── okd-bundle.yaml                        ← rendered per-cluster bundles
    ├── mgmt-forge-bundle.yaml
    ├── mgmt-core-bundle.yaml
    ├── dev-{data,apps,web}-bundle.yaml
    └── mgmt-observ-bundle.yaml
```

## Server vs client — what runs where

```
mgmt-storage  (server: storage/rook-ceph/)     any consumer (this dir)
─────────────────────────────────────────      ─────────────────────────

Rook operator (full)                           Rook operator (external mode)
MONs × 3       authoritative                   (none)
MGRs × 2       dashboard, autoscaler           (none)
OSDs × 3       actual disks (sdb per node)     (none)
MDS            CephFS metadata daemons         (none)
RGW            S3 gateway                      (none)

CSI plugins    so server can use storage       CSI plugins    ← the whole point.
               locally too                                      kubelet uses these
                                                                to attach RBD /
                                                                mount CephFS in pods.

CephCluster CR (full spec)                     CephCluster CR with
  cephVersion, mon.count=3, mgr, OSDs, …         spec.external.enable: true
                                                 (no daemon spec — just a pointer
                                                  at the remote mons)

CephBlockPool / Fs / ObjectStore               (none — consumed, not owned)

StorageClasses ceph-rbd, cephfs, ceph-rgw      StorageClasses ceph-rbd, cephfs
                                                 (same names; parameters point at
                                                  the remote mons + cephx keys)

Disks: /dev/sdb × 3                            Disks: none
                                                 (all I/O goes over the network
                                                  from consumer → mgmt-storage OSDs)
```

## What "external" means to Rook

Rook has a specific deployment mode called **external**. The same operator
binary runs on the consumer, but the CephCluster CR tells it:
`spec.external.enable: true`. The operator then:

- Does **not** create any Ceph daemons (no MONs, OSDs, MGRs, MDS, RGW).
- **Does** reconcile the client-side config: cephx Secrets, the ConfigMap
  holding MON endpoints, StorageClass parameter templating.
- Keeps the local CSI plugins (`csi-rbdplugin-*`, `csi-cephfsplugin-*`)
  running so they can do the actual volume lifecycle work.

The CSI plugins (both provisioner Deployments and node-side DaemonSets) are
the real workhorses on the consumer — operator-external mostly just exists
to shape their config and own the CephCluster CR for observability.

## How a PVC on a consumer becomes a real disk

```
  1. Pod on consumer cluster requests PVC (storageClassName: ceph-rbd).
  2. CSI provisioner (rook-ceph-external ns, this cluster) receives the
     CreateVolume gRPC from k8s.
  3. CSI provisioner authenticates to MONs at 10.10.4.100-102:3300 using
     the cephx key `csi-rbd-provisioner` (imported from generate.yml).
  4. MONs return the osdmap + grant the request.
  5. CSI provisioner tells Ceph "create RBD image csi-vol-xxx in replicapool,
     20 GiB". The image is created on the mgmt-storage OSDs (sdb devices).
  6. PVC is bound; pod scheduling continues.
  7. kubelet on the consumer's node sends NodeStageVolume to the local
     CSI node plugin (DaemonSet in this ns).
  8. krbd kernel module (or rbd-nbd userspace) maps the RBD image — this is
     the long-lived TCP session from this node to the OSDs that host the PGs
     for this image.
  9. kubelet NodePublishVolume → bind-mount the resulting block device
     into the pod's mount namespace.
  10. Pod writes → block device → network → mgmt-storage OSDs.
```

Every write hops the network from consumer → mgmt-storage. Reads may be
served by any of the 3 OSDs holding the replica of the relevant PG (Ceph
picks closest by default; with multi-DC clusters you'd tune this).

## Why the install is split this way

- **One backend, many consumers.** You don't want OSDs on workload
  clusters — their disks are small, their node count shifts, workloads
  fight storage for IO.
- **Separation of concerns.** Storage team owns `storage/rook-ceph/`;
  platform team owns consumer configs. Independent upgrade cadence.
- **Rook's external mode is built for this topology.**
  `create-external-cluster-resources.py` mints cephx keys with minimal
  scopes per consumer — not cluster-admin Ceph credentials. Each consumer
  gets read/write access to the specific pools it needs.

## Where this fits in the build order

1. `storage/rook-ceph/` must be Synced+Healthy on mgmt-storage first.
2. `generate.yml` — once, after Ceph is up. Emits
   `generated/external-cluster.env` and per-cluster bundles.
3. `apply.yml` — loops through each consumer cluster in `targets:` and
   runs the import script + Helm install + smoke PVC.
4. For a new consumer cluster that comes online later: re-run `apply.yml`
   with only that cluster in `targets:`. The existing `external-cluster.env`
   is still valid — cephx keys and MON endpoints don't change unless
   mgmt-storage is rebuilt.

## Gotchas (see `storage/rook-ceph/README.md` for full issue log)

- **OSDs must advertise host IPs, not pod IPs.** Without that, client
  side gets the osdmap and hangs forever connecting to unrouteable pod
  CIDRs. Fixed on mgmt-storage by pinning
  `network.addressRanges.public = 10.10.4.0/24`.
- **MON endpoints are v2-only (port 3300).** The chart emits mon tuples
  with both v1 (6789) and v2 (3300). Some older clients mis-select v1
  and hang on connect. Rook's script does the right thing; hand-rolled
  templates often don't.
- **cephx key scopes.** Each consumer's keys are scoped to specific pools
  and subvolume groups. If you re-generate, the old keys are invalidated
  — import the new env on every consumer.
