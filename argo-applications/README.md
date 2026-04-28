# argo-applications/

ArgoCD-managed configuration for the `engatwork` homelab estate. Two
ArgoCD instances run on OKD (in `argocd-sre` and `argocd-devops`
namespaces). This directory holds the `Application` / `ApplicationSet`
manifests each instance reconciles.

Chart code now lives in **the same repo** under `/root/learning/layers/<domain>/` — see [`../README.md`](../README.md) for the domain layout. AppSets reference charts via `repoURL: https://github.com/GaddipatiSriram/layers.git` and `path: <domain>/<chart>` (e.g. `path: platform/cert-manager`).

> **Note**: this README still has sections written for the pre-consolidation
> multi-repo layout. The folder layout below is current. The "service
> catalog" and "ApplicationSet summary" sections further down may
> reference paths under the old `forge/` umbrella; treat
> `forge/<chart>` mentions as `layers/<new-domain>/<chart>` per the
> reorg map in [`/root/learning/layers/forge/README.md`](../forge/README.md).

## Folder layout (current)

```
argo-applications/
├── README.md                         ← this file
├── todo.md
├── QA.md
├── interview-scenarios.md
└── sre/                              ← argocd-sre instance (platform ops)
    ├── bootstrap-sa.yaml             ← SA+CRB+token bootstrap (one-time)
    ├── platform/                     ← Pattern A AppSets (every workload cluster)
    │   ├── cert-manager-appset.yaml
    │   ├── ingress-nginx-appset.yaml
    │   ├── external-secrets-appset.yaml
    │   └── metallb-appset.yaml
    ├── identity/                     ← Pattern B (role=forge today; OKD eventually)
    │   └── keycloak-appset.yaml
    ├── secrets/                      ← Pattern B
    │   └── vault-appset.yaml
    ├── forge/                        ← role=forge AppSets + repo creds
    │   ├── values.yaml
    │   ├── layers-secret-sre.yaml    ← git repo creds (gitignored)
    │   └── layers-secret-devops.yaml ← git repo creds (gitignored)
    ├── observability/                ← Pattern C (role=observ)
    │   ├── prometheus-appset.yaml
    │   ├── grafana-appset.yaml
    │   └── alertmanager-appset.yaml
    ├── security/                     ← Pattern A
    │   └── trivy-operator-appset.yaml
    ├── devops/                       ← OKD-targeted CI/CD Applications
    │   ├── argo-workflows-app.yaml
    │   └── jenkins-app.yaml
    ├── storage/                      ← Pattern A consumer install
    │   └── rook-ceph-external-appset.yaml
    ├── data/                         ← (placeholder — future CNPG, Strimzi AppSets)
    ├── search/                       ← (placeholder — future OpenSearch, Wazuh AppSets)
    └── mgmt-storage/                 ← bootstrap + server-side Rook-Ceph
        ├── bootstrap-sa.yaml         ← SA+CRB+token on mgmt-storage (one-time)
        ├── cluster-secret.yaml       ← registers cluster in argocd-sre
│   ├── mgmt-storage/                 ← bootstrap + server-side Rook-Ceph
│   │   ├── bootstrap-sa.yaml         ← SA+CRB+token on mgmt-storage (one-time)
│   │   ├── cluster-secret.yaml       ← registers cluster in argocd-sre
│   │   ├── storage-repo-secret.yaml
│   │   ├── rook-ceph-operator-app.yaml
│   │   ├── rook-ceph-cluster-app.yaml
│   │   └── README.md                 ← cluster-onboarding runbook
│   └── storage/                      ← cross-cluster storage consumer wiring
│       └── rook-ceph-external-appset.yaml
└── devops/                           ← argocd-devops instance (developer tooling)
    └── (currently empty — reserved for CI/CD, apps)
```

## Two ArgoCD instances, why

Separation of concerns:

- **argocd-sre**: platform ops team owns. Rook-Ceph, Keycloak, Vault,
  cert-manager, ingress-nginx, external-secrets, rook-ceph-external,
  cluster registrations. Anything that affects the platform's
  availability.
- **argocd-devops**: product/developer team owns. App deployments,
  CI/CD pipelines, workload-specific config. Lower blast radius if
  misconfigured — can't take down the platform.

Both instances run on the OKD cluster but reconcile Applications to
whatever cluster destination the Application declares.

## Selector patterns

Four distinct patterns cover the platform:

```yaml
# Pattern A — cluster-local infra (every workload-capable cluster)
# USE FOR: cert-manager, ingress-nginx, external-secrets,
#          rook-ceph-external, trivy-operator, node-exporter,
#          Promtail, OTel agents, actions-runner-controller
selector:
  matchExpressions:
    - { key: tier, operator: In,    values: [mgmt, dev] }
    - { key: role, operator: NotIn, values: [storage] }

# Pattern B — central platform services (one instance on forge)
# USE FOR: Vault, Keycloak, Harbor, SonarQube
selector:
  matchLabels:
    role: forge

# Pattern C — central observability backends
# USE FOR: Grafana, Prometheus/Mimir, Loki, Tempo, Alertmanager
selector:
  matchLabels:
    role: observ

# Pattern D — shared infrastructure (mgmt-core specifics)
# USE FOR: CoreDNS external zone, NTP, bastion (future)
selector:
  matchLabels:
    role: core
```

## Lifecycle vocabulary

Not every install is Argo-managed. Vocabulary used below:

- **Bootstrap (one-time)** — applied once, not reconciled. Usually
  kubectl apply or Ansible. Examples: cephx Secrets (can't live in Git),
  cluster registration Secret, SA tokens on target clusters, kubeadm init.
- **Ansible-managed** — maintained via playbook runs. Legacy on some
  clusters; goal is to migrate all to Argo.
- **OLM** — installed via OKD's OperatorHub / Operator Lifecycle
  Manager. Different mechanism from Argo/Helm.
- **Argo Application** — single-cluster-target manifest. One-to-one.
- **Argo ApplicationSet** — generates N Applications from a selector.
  One-to-many. Scales without config edits.

---

## Service catalog — by layer

Cluster roles in use today: `core`, `storage`, `forge`, `observ` (planned),
`apps`/`data`/`web` (dev clusters, planned). Plus OKD itself (no tier/role).

### OKD — tier=(none), hosts ArgoCD + CI/CD tooling

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **ArgoCD SRE** | platform GitOps controller | yes: `argocd-sre-server-argocd-sre.apps.mgmt-devops-okd.engatwork.com` | OLM (ArgoCD Operator) | ✅ |
| **ArgoCD DevOps** | app-layer GitOps controller | yes: `argocd-devops-server-argocd-devops.apps.mgmt-devops-okd.engatwork.com` | OLM | ✅ |
| **Tekton / OpenShift Pipelines** | k8s-native CI pipelines | yes: OpenShift console plugin | OLM (ships with OCP) | ✅ |
| **Argo Workflows** | workflow engine (DAG pipelines) | yes: `workflows.apps.mgmt-devops-okd.engatwork.com` | Argo Application (`argo-applications/sre/okd/`) | ✅ |
| **Jenkins** | legacy CI (demo) | yes: `jenkins.apps.mgmt-devops-okd.engatwork.com` | Argo Application | ✅ (emptyDir→ceph-rbd) |
| **rook-ceph-external** | Ceph CSI client | no | Ansible-managed (pre-Stage-3) | ⚠️ not yet Argo |

### mgmt-core — role=core, shared infrastructure

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **CoreDNS (authoritative `*.engatwork.com`)** | DNS server behind MetalLB VIP `10.10.1.200` | no | Ansible (`cluster/coredns/deploy-coredns.yml`) — one-time | ✅ |
| **NTP server** | time source for VLANs | no | planned | 📋 |
| **Bastion / jump** | SSH entry | no | planned | 📋 |

### mgmt-storage — role=storage, Ceph server

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **Rook-Ceph operator** | Ceph daemon controller + CRDs | no (operator) | Argo Application (`argo-applications/sre/mgmt-storage/rook-ceph-operator-app.yaml`), wave 0 | ✅ |
| **CephCluster + pools + FS + RGW** | MONs, MGRs, OSDs, MDS, RGW daemons | Ceph dashboard: `kubectl port-forward` to mgr | Argo Application (`rook-ceph-cluster-app.yaml`), wave 1 | ✅ |
| **Ceph toolbox pod** | debugging (`ceph -s`, `rbd ls`, etc.) | no (CLI via exec) | part of cluster chart | ✅ |

### mgmt-forge — role=forge, central platform services

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **cert-manager** | TLS cert automation | no | Argo AppSet (Pattern B today; should broaden to Pattern A) | ✅ |
| **ingress-nginx** | L7 ingress, MetalLB VIP | no | Argo AppSet (Pattern B today; should broaden to Pattern A) | ✅ |
| **external-secrets** | Vault → k8s Secret bridge | no | Argo AppSet (Pattern B today; should broaden to Pattern A) | ✅ |
| **Vault (server)** | secret store, PKI | yes: `vault.apps.mgmt-forge.engatwork.com` | Argo AppSet (Pattern B — stays central) | ✅ |
| **Keycloak** | SSO / OIDC provider | yes: `keycloak.apps.mgmt-forge.engatwork.com` | Argo AppSet (Pattern B — stays central) | ✅ |
| **rook-ceph-external** | Ceph CSI client | no | Argo AppSet (Pattern A, `argo-applications/sre/storage/`) | ✅ (Stage 3) |
| **Harbor** | container registry + Trivy scanner | yes | Argo AppSet (Pattern B) | 📋 |
| **SonarQube** | code quality / SAST | yes | Argo AppSet (Pattern B) | 📋 |
| **trivy-operator** | runtime vuln scanner | no (CRDs: `VulnerabilityReport`) | Argo AppSet (Pattern A — every cluster) | 📋 |
| **actions-runner-controller** | GitHub self-hosted runners | no | Argo AppSet (Pattern A) | 📋 |

### mgmt-observability — role=observ, central observability backends (PLANNED)

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **Prometheus (or Mimir)** | metrics storage; receives remote-write from all clusters | yes (basic) | Argo AppSet (Pattern C) | 📋 |
| **Grafana** | dashboards, fronts all telemetry backends | yes | Argo AppSet (Pattern C) | 📋 |
| **Loki** | log aggregation, ingests from Promtail agents | no (queried via Grafana) | Argo AppSet (Pattern C) | 📋 |
| **Tempo** | trace aggregation, ingests from OTel gateway | no (queried via Grafana) | Argo AppSet (Pattern C) | 📋 |
| **Alertmanager** | alert routing, Slack/email dispatch | yes | Argo AppSet (Pattern C) | 📋 |
| **OTel gateway** | central OTel collector, receives from cluster-local agents | no | Argo AppSet (Pattern C) | 📋 |

### Cluster-local observability agents (Pattern A — every workload cluster, PLANNED)

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **node-exporter** (DaemonSet) | host metrics to local Prometheus/remote-write | no | Argo AppSet (Pattern A) | 📋 |
| **kube-state-metrics** | k8s object metrics | no | Argo AppSet (Pattern A) | 📋 |
| **Promtail** (DaemonSet) | log shipper → central Loki | no | Argo AppSet (Pattern A) | 📋 |
| **OpenTelemetry Collector** (agent mode) | trace/metric forwarder → central OTel | no | Argo AppSet (Pattern A) | 📋 |

---

## ApplicationSet summary

Every ApplicationSet in `argo-applications/sre/`, its selector pattern, what it
targets today, and what it'll target as more clusters come online.

| AppSet | File | Pattern | Selector | Targets today | Will target (future) | Source chart | Status |
|---|---|---|---|---|---|---|---|
| **rook-ceph-external** | `argo-applications/sre/storage/rook-ceph-external-appset.yaml` | A | `tier In [mgmt,dev]` AND `role NotIn [storage]` | mgmt-forge | mgmt-observability, mgmt-core, dev-* (once registered) | `storage/rook-ceph-external/chart/` | ✅ |
| **cert-manager** | `argo-applications/sre/forge/cert-manager-appset.yaml` | B (should be A) | `role: forge` | mgmt-forge | (every workload cluster, after Pattern-A refactor) | `forge/cert-manager/` | ✅ |
| **ingress-nginx** | `argo-applications/sre/forge/ingress-nginx-appset.yaml` | B (should be A) | `role: forge` | mgmt-forge | (every workload cluster, after Pattern-A refactor) | `forge/ingress-nginx/` | ✅ |
| **external-secrets** | `argo-applications/sre/forge/external-secrets-appset.yaml` | B (should be A) | `role: forge` | mgmt-forge | (every workload cluster, after Pattern-A refactor) | `forge/external-secrets/` | ✅ |
| **vault** | `argo-applications/sre/forge/vault-appset.yaml` | B | `role: forge` | mgmt-forge | mgmt-forge only (central service) | `forge/vault/` | ✅ |
| **keycloak** | `argo-applications/sre/forge/keycloak-appset.yaml` | B | `role: forge` | mgmt-forge | mgmt-forge only (central service) | `forge/keycloak/` | ✅ |
| **trivy-operator** | *not yet authored* | A | `tier In [mgmt,dev]` AND `role NotIn [storage]` | — | every workload cluster | `forge/trivy-operator/` | 📋 |
| **harbor** | *not yet authored* | B | `role: forge` | — | mgmt-forge | `forge/harbor/` | 📋 |
| **sonarqube** | *not yet authored* | B | `role: forge` | — | mgmt-forge | `forge/sonarqube/` | 📋 |
| **prometheus / mimir** | *not yet authored* | C | `role: observ` | — | mgmt-observability | TBD | 📋 |
| **grafana** | *not yet authored* | C | `role: observ` | — | mgmt-observability | TBD | 📋 |
| **loki / tempo / alertmanager** | *not yet authored* | C | `role: observ` | — | mgmt-observability | TBD | 📋 |
| **obs agents** (node-exporter, Promtail, OTel) | *not yet authored* | A | `tier In [mgmt,dev]` AND `role NotIn [storage]` | — | every workload cluster | TBD | 📋 |

### Single-cluster Argo Applications (not ApplicationSets)

Used for genuine one-offs where there's only ever one target cluster:

| Application | File | Target | Source | Status |
|---|---|---|---|---|
| `rook-ceph-operator-mgmt-storage` | `argo-applications/sre/mgmt-storage/rook-ceph-operator-app.yaml` | mgmt-storage | `storage/rook-ceph/operator/` | ✅ |
| `rook-ceph-cluster-mgmt-storage` | `argo-applications/sre/mgmt-storage/rook-ceph-cluster-app.yaml` | mgmt-storage | `storage/rook-ceph/cluster/` | ✅ |
| `argo-workflows-okd` | `argo-applications/sre/okd/argo-workflows-app.yaml` | OKD (in-cluster) | `forge/devops-ci/argo-workflows/` | ✅ |
| `jenkins-okd` | `argo-applications/sre/okd/jenkins-app.yaml` | OKD (in-cluster) | `forge/devops-ci/jenkins/` | ✅ |

### Everything else — not Argo-managed

| Service | Why not Argo | Where to look instead |
|---|---|---|
| **Shared CoreDNS on mgmt-core** | one-shot bootstrap; no reconciliation value | `cluster/coredns/deploy-coredns.yml` (Ansible) |
| **ArgoCD itself** (SRE + DevOps) | can't GitOps yourself into existence | OLM Subscription on OKD |
| **Tekton / OpenShift Pipelines** | ships with OCP via OLM | OLM, managed by Red Hat operator |
| **rook-ceph-external on OKD** | pre-Stage-3; still on Ansible | `cluster/rook-ceph-external/apply.yml`; migration queued in `storage/todo.md` |
| **Cephx Secrets + mon-endpoints ConfigMap** | secrets can't live in Git | one-shot `kubectl apply -f <cluster>-bundle.yaml` |
| **Vault first-time init + unseal** | cryptographic ceremony | manual; unseal keys stored outside Git |
| **Keycloak realm setup** | data, not infra | to be declaratively managed via keycloak-config-cli later |

### dev-apps / dev-data / dev-web — tier=dev, application workloads (PLANNED)

| Service | Nature | UI | Install | Status |
|---|---|---|---|---|
| **app-specific workloads** (databases, business apps) | varies | varies | Argo Applications managed by `argocd-devops` | 📋 |
| **cluster-local infra** (cert-manager, ingress, ESO, ceph CSI) | — | — | Argo AppSets (Pattern A, automatic once cluster registered) | 📋 |
| **observability agents** | — | — | Argo AppSets (Pattern A) | 📋 |

---

## Deployment order — green-field build sequence

Each phase establishes dependencies the next phase relies on. Most
ordering is strict (don't start phase N+1 before phase N is healthy).
Within a phase, items marked **[parallel]** can run concurrently.

### Phase 0 — Hypervisor layer (before anything k8s)
Ordering hard — nothing k8s starts without these.
1. **oVirt engine** up (`ovirt-setup/playbooks/ovirt/site.yml`).
2. **pfSense VM** created + base config (VLANs, NAT, firewall rules).
3. **Data LUN** attached to oVirt as a storage domain.
4. **pfSense DNS forwarder** configured (but target `10.10.1.200`
   doesn't exist yet — placeholder until Phase 1).

### Phase 1 — Core DNS (mgmt-core)
*Nothing resolves `*.engatwork.com` until this phase completes.*
1. **mgmt-core VMs** deployed in oVirt (`ovirt-setup/playbooks/compute/`
   with `vars/mgmt-core.yml`).
2. **k8s bootstrap** on mgmt-core (`cluster/k8s-bstrp/bootstrap-k8s.yml`
   with `-i inventory/mgmt-core.ini`). Brings up:
   - k8s control plane + workers
   - Flannel CNI
   - MetalLB (VIP pool 10.10.1.200-220)
3. **Shared CoreDNS** on mgmt-core (`cluster/coredns/deploy-coredns.yml`).
   Serves `*.engatwork.com` at MetalLB VIP **10.10.1.200**.
4. **Reconfigure pfSense**: conditional forwarder
   `engatwork.com → 10.10.1.200`. Now every VM on every VLAN can resolve
   internal names.

Gate: `nslookup anything.engatwork.com` from any VM returns the
MetalLB VIP → pass. Without this, OKD install can't start.

### Phase 2 — OKD cluster (mgmt-devops)
*Provides the ArgoCD home. Everything Argo-managed depends on this.*
1. **pfSense HAProxy VIPs** for OKD API (10.10.2.50) + Ingress
   (10.10.2.51). Route in pfSense front-ends.
2. **OKD install** via Agent-based Installer (`okd/site.yml`, 8 phases).
   Cluster ~30-45 min to converge.
3. **DNS records** for OKD: `api.mgmt-devops-okd.engatwork.com` →
   10.10.2.50, `*.apps.mgmt-devops-okd.engatwork.com` → 10.10.2.51
   (added to CoreDNS via `cluster/coredns/playbooks/update-okd-dns.yml`).

Gate: OKD console reachable at
`https://console-openshift-console.apps.mgmt-devops-okd.engatwork.com/`.

### Phase 3 — ArgoCD instances on OKD
*First thing that "becomes GitOps-capable."*
1. **ArgoCD Operator** via OperatorHub (OLM subscription).
2. **ArgoCD SRE** CR in `argocd-sre` namespace (platform GitOps).
3. **ArgoCD DevOps** CR in `argocd-devops` namespace (app GitOps).
4. **Route / Ingress** for both consoles.
5. **Git repo credentials** Secrets applied
   (`argo-applications/sre/forge/forge-secret-sre.yaml`,
   `argo-applications/sre/forge/forge-secret-devops.yaml`).

Gate: both ArgoCD consoles reachable, can fetch Helm charts from
`GaddipatiSriram/forge` + `storage` repos.

### Phase 4 — Storage server (mgmt-storage)
*Blocks every stateful workload in the estate.*
1. **mgmt-storage VMs** deployed (`ovirt-setup/playbooks/compute/` with
   `vars/mgmt-storage.yml`, currently missing — queued in `todo.md`).
2. **Extra-disk prep** (resize root + attach 500 GB OSD disks) via
   `ovirt-setup/playbooks/compute/mgmt-storage-prep/01-prepare-disks.yml`.
3. **k8s bootstrap** on mgmt-storage
   (`cluster/k8s-bstrp/bootstrap-k8s.yml -i inventory/mgmt-storage.ini`).
4. **Register with argocd-sre**: apply `argo-applications/sre/mgmt-storage/bootstrap-sa.yaml`
   on mgmt-storage + `cluster-secret.yaml` on OKD.
5. **Argo Application: rook-ceph-operator** (wave 0).
6. **Argo Application: rook-ceph-cluster** (wave 1, depends on wave 0
   CRDs).
7. **Verify HEALTH_OK** via `ceph -s` from the toolbox pod.

Gate: `ceph -s` returns HEALTH_OK, 3 OSDs up, pools created.

### Phase 5 — Central platform services (mgmt-forge)
*The forge cluster hosts everything app-facing: Vault, Keycloak, etc.*

Phase 5 has a **strict internal ordering** because stateful services
(Vault) depend on Ceph storage, which depends on the CSI client
landing first. Sequence:

1. **mgmt-forge VMs** + **k8s bootstrap** (register with argocd-sre via
   Phase 8 of `bootstrap-k8s.yml`; labels `tier=mgmt role=forge`).
2. **Cephx bootstrap (one-time, out-of-band)**:
   ```
   kubectl --context mgmt-forge apply -f \
     storage/rook-ceph-external/generated/mgmt-forge-bundle.yaml
   ```
   Creates the `rook-ceph-external` namespace + cephx Secrets
   (`rook-ceph-mon`, `rook-csi-rbd-{node,provisioner}`,
   `rook-csi-cephfs-{node,provisioner}`) + `rook-ceph-mon-endpoints`
   ConfigMap. None of this is in Git (cryptographic material).
3. **rook-ceph-external ApplicationSet auto-generates Application**
   (within 30s of the cluster Secret appearing, selector
   `tier In [mgmt,dev]` AND `role NotIn [storage]` matches). Installs:
   - Rook operator in external mode
   - CephCluster CR with `spec.external.enable: true`
   - CSI drivers (RBD + CephFS)
   - StorageClasses `ceph-rbd`, `cephfs`

   **Gate before continuing**: PVC against `ceph-rbd` binds in <15s.
   Nothing stateful beyond this point works without this step.

4. **Forge-stack ApplicationSets auto-fire** (matching `role: forge`
   selector, happen in parallel):
   - `cert-manager` → TLS issuers (Pattern B, should broaden to A)
   - `ingress-nginx` → L7 ingress + MetalLB VIP (Pattern B, should broaden to A)
   - `external-secrets` → Vault bridge (Pattern B, should broaden to A)
   - `vault` → Vault server **[needs ceph-rbd for file backend]**
   - `keycloak` → SSO server **[needs ceph-rbd for Postgres]**

5. **One-time interactive bootstraps** (can't be Git-managed):
   - **Vault `operator init`** → save the 5 unseal keys + 1 root
     token it emits.
   - **Vault unseal** each pod (×3 with 3 of the 5 keys).
   - **Vault post-install playbook** (KV v2 engine, k8s auth method,
     forge-reader role) — `forge/vault/playbooks/configure.yml`.
   - **Keycloak realm** setup — create `forge` realm + OIDC clients
     (Harbor, ArgoCD, Grafana will register here later).

Gate: Vault unsealed + reachable, Keycloak admin login works, ESO
`ClusterSecretStore` is `Valid`, test PVC on ceph-rbd binds.

### Phase 6 — Observability backends (mgmt-observability) [PLANNED]
*Optional for the estate to work, but everything after benefits from it.*

Same internal ordering rule as Phase 5: cephx + CSI before anything
stateful.

1. **mgmt-observability VMs** + **k8s bootstrap**
   (labels `tier=mgmt role=observ`).
2. **Cephx bootstrap** on mgmt-observability:
   `kubectl --context mgmt-observability apply -f \
     storage/rook-ceph-external/generated/mgmt-observ-bundle.yaml`
   (regenerate first if stale — `storage/rook-ceph-external/generate.yml`).
3. **rook-ceph-external ApplicationSet auto-fires** (Pattern A selector
   already matches `tier=mgmt role=observ` — no changes needed).
4. **Pattern-A ApplicationSets** (cert-manager, ingress-nginx, ESO) fire
   for this cluster **only after their selectors are broadened** from
   `role: forge` to Pattern A (see `storage/todo.md` refactor task).
5. **Author + apply Pattern C ApplicationSets** for observability
   backends:
   - Prometheus/Mimir — **[must be first for remote-write endpoint]**
   - Loki — log backend
   - Tempo — trace backend
   - Alertmanager — alert routing
   - OTel gateway — central collector
   - Grafana — **[last, fronts everything above]**
6. **Cluster-local agent ApplicationSets (Pattern A)** roll out to all
   workload clusters: node-exporter, kube-state-metrics, Promtail, OTel
   agent. These remote-write back to the backends in step 5.

Gate: Grafana UI up, default dashboards show data from every workload
cluster.

### Phase 7 — CI/CD services (forge-hosted, central)
*After platform is stable.*
1. **Harbor** (Pattern B role=forge) — needs ceph-rbd for Postgres,
   ceph-rbd or S3 (Ceph RGW) for blob store, Keycloak OIDC for auth.
2. **SonarQube** (Pattern B role=forge) — ceph-rbd for DB, Keycloak
   OIDC for auth.
3. **trivy-operator** (Pattern A) — runtime vuln scanner on every
   cluster (CRDs report back locally; Harbor's built-in Trivy scans
   images at rest, complementary).
4. **actions-runner-controller** (Pattern A) — GitHub self-hosted
   runners per cluster, if you want CI jobs to run inside k8s.
5. **argocd-devops** starts managing app deployments.

### Phase 8 — Dev clusters (dev-apps / dev-data / dev-web) [PLANNED]
*Final layer — where business workloads live.*

Same ordering pattern — cephx bootstrap + rook-ceph-external before
workload deployments.

1. **dev-<role> VMs** + **k8s bootstrap**
   (labels `tier=dev role=<apps|data|web>`).
2. **Cephx bootstrap** on each:
   `kubectl --context dev-<role> apply -f \
     storage/rook-ceph-external/generated/dev-<role>-bundle.yaml`.
3. **rook-ceph-external ApplicationSet auto-fires** (Pattern A selector
   matches `tier=dev`).
4. **Pattern-A ApplicationSets** (cert-manager, ingress-nginx, ESO,
   trivy-operator, obs agents) fire once selectors are broadened.
5. **argocd-devops picks up app-specific Applications** from the
   devops repo (different source of truth).

### One-shot visual

```
Phase 0 — oVirt + pfSense
   ↓
Phase 1 — mgmt-core (CoreDNS @ 10.10.1.200)        ← prereq for all
   ↓
Phase 2 — OKD cluster
   ↓
Phase 3 — ArgoCD SRE + DevOps on OKD               ← gate: GitOps online
   ↓
Phase 4 — mgmt-storage + Rook-Ceph server          ← gate: storage online
   ↓
Phase 5 — mgmt-forge + platform services           ← gate: platform online
         (Vault + Keycloak + ingress + CSI client)
   ↓
Phase 6 — mgmt-observability (optional)            ← gate: observability online
         (Grafana + Prometheus + Loki + Tempo)
   ↓
Phase 7 — CI/CD services on forge (Harbor, SonarQube, Trivy, ARC)
   ↓
Phase 8 — dev-apps / dev-data / dev-web + workloads
```

### What depends on what

- **Phase 1 blocks everything** — no DNS = no cluster reachability.
- **Phase 2 blocks Phases 3-8** — no OKD = no ArgoCD = no GitOps.
- **Phase 3 blocks Phases 4-8** — no ArgoCD = can't deploy anything
  declaratively.
- **Phase 4 blocks Phases 5-8 stateful workloads** — no Ceph = PVCs
  fail → Vault, Harbor, SonarQube, DBs all break.
- **Phase 5 blocks apps that need SSO** — Harbor / SonarQube / ArgoCD
  OIDC all depend on Keycloak; apps that read from Vault need ESO.
- **Phase 6 is optional** — you can run the estate without it, you
  just have no metrics/logs/traces.
- **Phase 7 & 8 are leaves** — consumers of everything above.

---

## Cluster registration & bootstrap (one-time)

Every cluster except OKD (in-cluster for ArgoCD) needs explicit
registration before any Argo ApplicationSet can target it. Runbook:
`argo-applications/sre/mgmt-storage/README.md`.

Three pieces, applied in order:

| Resource | Applied on | Lifecycle |
|---|---|---|
| `bootstrap-sa.yaml` — ServiceAccount + CRB + token Secret | target cluster (via its own kubeconfig) | one-time, not reconciled |
| `cluster-secret.yaml` — cluster registration (server URL + bearer token + CA, plus labels) | OKD, `argocd-sre` ns | one-time, not reconciled |
| `<repo>-repo-secret.yaml` — git repo creds (if private) | OKD, `argocd-sre` ns | one-time, not reconciled |

After these three, any matching ApplicationSet auto-generates an
Application on the next reconcile. Adding a new cluster = 3 YAML
applies; ApplicationSets do the rest.

### Services that also require out-of-band bootstrap

| Service | One-time manual step | Why |
|---|---|---|
| `rook-ceph-external` | `kubectl apply -f storage/rook-ceph-external/generated/<cluster>-bundle.yaml` | cephx Secrets + mon-endpoints ConfigMap can't live in Git |
| `Vault` (first time) | `vault operator init` + unseal keys | Vault storage seal/unseal ceremony is inherently interactive |
| `Keycloak` (realm setup) | import forge realm JSON / create OIDC clients | Realm config is data, not infra — lives separately |

---

## State snapshot (as of 2026-04-23, session close)

Registered clusters in argocd-sre:

```
argocd-sre-default-cluster-config  (OKD in-cluster)  labels: operator-managed
mgmt-forge                         tier=mgmt role=forge
mgmt-storage                       tier=mgmt role=storage
```

Live Applications in argocd-sre:

```
argo-workflows-okd                 Synced  Healthy
cert-manager-mgmt-forge            Synced  Healthy
external-secrets-mgmt-forge        Synced  Healthy
ingress-nginx-mgmt-forge           Synced  Healthy
jenkins-okd                        Synced  Healthy
keycloak-mgmt-forge                Synced  Healthy
rook-ceph-cluster-mgmt-storage     Synced  Healthy
rook-ceph-external-mgmt-forge      Synced  Healthy
rook-ceph-operator-mgmt-storage    Synced  Healthy
vault-mgmt-forge                   Synced  Healthy
```

Live ApplicationSets:

```
cert-manager         (selector: role=forge)
external-secrets     (selector: role=forge)
ingress-nginx        (selector: role=forge)
keycloak             (selector: role=forge)
vault                (selector: role=forge)
rook-ceph-external   (selector: tier In [mgmt,dev] AND role NotIn [storage])
```

---

## How to...

### Add a new cluster to the estate

1. Bootstrap the cluster at the VM + k8s layers (`ovirt-setup/` →
   `cluster/k8s-bstrp/`).
2. Phase 8 of `bootstrap-k8s.yml` applies `bootstrap-sa.yaml` on the
   new cluster AND creates a `cluster-secret.yaml` in argocd-sre.
3. The cluster Secret's labels come from `group_vars/<cluster>.yml` in
   k8s-bstrp. Make sure they include the right `tier` + `role`.
4. Bootstrap rook-ceph-external if the cluster needs Ceph
   (`kubectl apply -f storage/rook-ceph-external/generated/<cluster>-bundle.yaml`).
5. Within ~30s every ApplicationSet that matches the new cluster's
   labels generates an Application. No further action.

### Add a new central service (Pattern B — role=forge)

1. Author the Helm chart under `forge/<service>/`.
2. Create `argo-applications/sre/forge/<service>-appset.yaml` with
   `selector: { matchLabels: { role: forge } }`.
3. `kubectl apply -f argo-applications/sre/forge/<service>-appset.yaml`.
4. An Application is generated for mgmt-forge; Argo syncs.

### Add a new cluster-local service (Pattern A — every workload cluster)

1. Author the chart.
2. Create ApplicationSet with the Pattern A selector
   (`tier In [mgmt,dev]` AND `role NotIn [storage]`).
3. Apply. Every registered workload cluster gets the install
   automatically.

### Broaden an existing ApplicationSet's selector

Edit the selector in the ApplicationSet YAML, `kubectl apply`. On the
next reconcile, Applications are generated for any newly-matching
clusters. Be careful: anything that matches gets the install.
# argo
# argo
