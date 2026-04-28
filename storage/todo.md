# Storage — TODO

Active work backlog for the storage tier. Items are roughly ordered by
when they'll actually be tackled.

---

## Stage 3 — Argo-ify rook-ceph-external (consumer side)

**Status: in progress — chart authored, ApplicationSet authored, rolling out to mgmt-forge as the test case.**

Adopted a single **ApplicationSet** (not per-cluster Applications) because
everything EXCEPT cluster destination is identical across consumers.
Cephx Secrets + mon-endpoints ConfigMap still live outside Git and are
bootstrapped manually per cluster from `generated/<cluster>-bundle.yaml`.

Selector:
```yaml
matchExpressions:
  - { key: tier, operator: In,    values: [mgmt, dev] }
  - { key: role, operator: NotIn, values: [storage] }
```
Excludes mgmt-storage (role=storage) and OKD (no tier label) automatically.
OKD stays on its Ansible-installed `rook-ceph-external` for now.

Pattern mirrors Stage 2 (which put the server-side `rook-ceph/` under
Argo). Consumer-side install on every non-mgmt-storage cluster gets the
same treatment, one Argo Application per consumer.

### Why it's not trivial

The existing `apply.yml` Ansible playbook deploys three classes of
objects per consumer:

1. **Rook operator (external mode)** + its RBAC/CRDs — from the
   `rook-release/rook-ceph` Helm chart with external-mode values.
2. **CephCluster CR** (`spec.external.enable: true`) + StorageClasses —
   declarative; fits Helm templates cleanly.
3. **cephx Secrets + `rook-ceph-mon-endpoints` ConfigMap** — **secrets
   cannot live in Git.** They're emitted by `generate.yml` (which wraps
   Rook's official `create-external-cluster-resources.py`) and must be
   applied to each consumer out-of-band.

Argo can own (1) and (2). (3) needs a one-time manual bootstrap per
consumer OR an ESO-backed pattern (Vault → external-secrets → k8s
Secret). The ESO path has chicken-and-egg issues (mgmt-forge needs ESO
first, which needs Vault, which wants Ceph storage), so the initial
approach is manual-bootstrap.

### Layout after Stage 3

```
storage/rook-ceph-external/
├── README.md                               # existing
├── chart/                                  # NEW — umbrella
│   ├── Chart.yaml                          # depends on rook-release/rook-ceph
│   ├── values.yaml                         # external-mode operator values
│   └── templates/
│       ├── cephcluster-external.yaml       # spec.external.enable: true
│       ├── storageclass-ceph-rbd.yaml
│       └── storageclass-cephfs.yaml
├── generate.yml                            # keep — cephx generation
├── scripts/, templates/, generated/        # keep
└── apply.yml                               # retire (move to legacy/ or delete)
```

Argo Applications, one per consumer:

```
argo-applications/sre/okd/rook-ceph-external-app.yaml         # adopt existing install
argo-applications/sre/mgmt-forge/rook-ceph-external-app.yaml  # fresh install
argo-applications/sre/dev-{apps,data,web}/...                 # future
argo-applications/sre/mgmt-{core,observ}/...                  # future
```

### Concrete tasks

- [x] **Author the umbrella chart** at `storage/rook-ceph-external/chart/`
  - [x] `Chart.yaml` depending on `rook-release/rook-ceph` v1.15.5
  - [x] `values.yaml` with external-mode operator toggles
  - [x] `templates/cephcluster-external.yaml` — CephCluster CR with
        `spec.external.enable: true`
  - [x] `templates/storageclass-ceph-rbd.yaml` — matches live OKD SC
        parameters (will adopt cleanly if we migrate OKD later)
  - [x] `templates/storageclass-cephfs.yaml` — same

- [x] **Author ApplicationSet** at `argo-applications/sre/rook-ceph-external-appset.yaml`
      with the tier/role selector above.

- [ ] **Commit + push** storage repo; commit learning repo Argo manifest.

- [ ] **mgmt-forge fresh install** (the test case)
  - [ ] Bootstrap cephx + mon-endpoints: `kubectl --context mgmt-forge apply
        -f storage/rook-ceph-external/generated/mgmt-forge-bundle.yaml`.
        Known gotcha: bundle has mon port 6789 (v1); may need to patch to
        3300 (v2) or confirm mgmt-storage still listens on v1.
  - [ ] Apply the ApplicationSet: `kubectl --context admin apply -f
        argo-applications/sre/rook-ceph-external-appset.yaml`.
  - [ ] Verify `rook-ceph-external-mgmt-forge` Application is generated
        and reaches Synced+Healthy.
  - [ ] Smoke: create a scratch PVC against `ceph-rbd` on mgmt-forge →
        Bound within 15s. Exec into a pod, write a file, verify.

- [ ] **OKD deferred** — keep existing Ansible install. Migrate later by
      adding `tier` label to the in-cluster Secret; ApplicationSet will
      pick it up and adopt via SSA. Stage-2-style SC immutability gotcha
      will apply; Jenkins PVC unaffected (PVCs don't re-read SC after
      binding).

- [ ] **Retire `apply.yml`**
  - [ ] Move to `storage/rook-ceph-external/legacy/` (same pattern as
        Stage 2's `storage/rook-ceph/legacy/`, which was deleted once
        Argo adoption was verified).
  - [ ] Update `storage/rook-ceph-external/README.md` to point at the
        Argo path as primary.

- [ ] **Extend to remaining consumers** (one at a time, after the first
      two are stable):
  - [ ] `mgmt-core`
  - [ ] `mgmt-observ`
  - [ ] `dev-apps`
  - [ ] `dev-data`
  - [ ] `dev-web`
  Each needs: regenerate bundle if mon state changed → manual Secret
  apply → Argo Application.

---

## Stage 4 — cephx rotation via ESO (future)

Replace the manual-bootstrap of cephx Secrets with a Vault → ESO flow.

- [ ] Store each consumer's cephx keys in Vault under
      `kv/storage/rook-ceph-external/<cluster>/…` during
      `generate.yml`.
- [ ] Each consumer's rook-ceph-external chart gains ExternalSecret
      CRDs that materialize the keys from Vault.
- [ ] Key rotation becomes: `generate.yml` writes new keys to Vault →
      ESO refreshes each consumer's Secrets within `refreshInterval`.
- [ ] **Prerequisite**: mgmt-forge must have Vault + ESO healthy
      (currently: Vault is running, ESO is running, but only on
      mgmt-forge — dev-* clusters need them too). Circular dependency
      to untangle.

---

## Smaller items

- [ ] **Delete legacy `storage/rook-ceph/legacy/`**
  Status: **DONE** (Stage 2 wrap-up, commit `e404350`).

- [ ] **Delete orphaned Helm release secret on mgmt-storage**
  Status: **DONE** (Stage 2 wrap-up).

- [ ] **Document the Stage-1 `addressRanges.public/cluster` fix in a
      runbook so it's discoverable by future operators**
  Status: **DONE** — `storage/rook-ceph/README.md` issue #1 has the
  full diagnosis + fix.

- [ ] **Rename `storage/external-consumers/` → `storage/rook-ceph-external/`
      so name mirrors `storage/rook-ceph/` (server)**
  Status: **DONE**.

- [ ] **Add CI/pre-commit hooks to lint Helm values against chart
      schema** — would catch things like the CephObjectStoreUser
      schema mismatch before Argo does.

- [ ] **Backup strategy for Ceph** (rbd-mirror, rgw bucket notifications
      to external S3, or CephFS snapshot schedule) — no backups today.

- [ ] **Monitoring** — `CephCluster.spec.monitoring.enabled: false`
      today. Flip on once Prometheus is up on mgmt-observ, wire the
      Ceph exporters.

- [ ] **S3 DNS + Route** — `s3.mgmt-storage.engatwork.com` needs a
      CoreDNS record + pfSense route to the ceph-rgw Service VIP.
      Nothing consumes the object store yet.

- [ ] **MinIO** — keep `storage/minio/` as placeholder; only if we
      decide we need a second S3 endpoint for Ceph's own backup target.
