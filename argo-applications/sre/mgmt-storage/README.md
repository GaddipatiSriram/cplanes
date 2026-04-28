# argo-applications/sre/mgmt-storage

All the manifests argocd-sre needs to manage the **mgmt-storage** cluster.
Applied from the OKD `admin` context into the `argocd-sre` namespace (with
bootstrap-sa.yaml applied to mgmt-storage itself).

## What's here

| File | Where it applies | Purpose |
|---|---|---|
| `bootstrap-sa.yaml` | `--context mgmt-storage` | Creates `argocd-manager` SA + cluster-admin CRB + long-lived token Secret on mgmt-storage. |
| `cluster-secret.yaml` | `--context admin` → argocd-sre | Registers mgmt-storage as a managed cluster in argocd-sre. Contains bearer token + CA. |
| `storage-repo-secret.yaml` | `--context admin` → argocd-sre | Repository credentials for the storage repo (optional if public). |
| `rook-ceph-operator-app.yaml` | `--context admin` → argocd-sre | Argo Application: Rook operator + CRDs (sync wave 0). |
| `rook-ceph-cluster-app.yaml` | `--context admin` → argocd-sre | Argo Application: CephCluster + pools + FS + RGW (sync wave 1). |

## How to Argo-prep a new cluster (generic runbook)

Use this as a template for standing up any new managed cluster under argocd-sre.

### 1. Ensure the target cluster is up and you have a kubeconfig context

```
kubectl --context <cluster-name> get nodes    # sanity check
```

### 2. Create a cluster-admin SA on the target cluster

Edit `bootstrap-sa.yaml` only if you need to change the SA name (default
`argocd-manager` in `kube-system`):

```
kubectl --context <cluster-name> apply -f bootstrap-sa.yaml
```

This creates:
- `ServiceAccount argocd-manager` in `kube-system`.
- `ClusterRoleBinding argocd-manager` → `cluster-admin`.
- `Secret argocd-manager-token` (`kubernetes.io/service-account-token`,
  the k8s >= 1.24 explicit long-lived token pattern).

### 3. Extract the token + CA, render cluster-secret.yaml

```
CA=$(kubectl --context <cluster-name> -n kube-system get secret argocd-manager-token -o=jsonpath='{.data.ca\.crt}')
TOKEN=$(kubectl --context <cluster-name> -n kube-system get secret argocd-manager-token -o=jsonpath='{.data.token}' | base64 -d)
CONFIG=$(echo -n "{\"bearerToken\":\"$TOKEN\",\"tlsClientConfig\":{\"insecure\":false,\"caData\":\"$CA\"}}" | base64 -w 0)
# Edit cluster-secret.yaml: replace the .data.config value with $CONFIG
# Edit the server URL to match the cluster's API endpoint.
```

### 4. Register the cluster in argocd-sre

```
kubectl --context admin apply -f cluster-secret.yaml
```

Verify it landed:

```
kubectl --context admin -n argocd-sre get secrets -l argocd.argoproj.io/secret-type=cluster
```

You should see the new cluster alongside `mgmt-forge`, `mgmt-storage`,
`argocd-sre-default-cluster-config` (in-cluster OKD).

### 5. Add repo credentials if the chart repo is private

```
kubectl --context admin apply -f storage-repo-secret.yaml    # or equivalent per-repo
```

Skip if the repo is public.

### 6. Apply the Argo Applications

For mgmt-storage specifically, after steps 1–5:

```
kubectl --context admin apply -f rook-ceph-operator-app.yaml     # wave 0
kubectl --context admin apply -f rook-ceph-cluster-app.yaml      # wave 1
```

Wave 0 installs the operator + CRDs; wave 1 installs the CephCluster once
CRDs exist. `syncPolicy.automated` means Argo continuously reconciles.

For a fresh cluster (not the current mgmt-storage), write Applications
matching whatever workload the cluster should host and apply them.

## Adoption note (mgmt-storage today)

The live cluster was originally built by Ansible (`storage/rook-ceph/legacy/install.yml`).
On first sync, Argo takes ownership via server-side apply. The orphaned
Helm release Secret (`sh.helm.release.v1.rook-ceph.v1`) left behind by
the original `helm install` is harmless. Delete after Argo syncs cleanly:

```
kubectl --context mgmt-storage -n rook-ceph delete secret sh.helm.release.v1.rook-ceph.v1
```

## Future clusters

To onboard e.g. `mgmt-observability` into argocd-sre the same way:

1. Copy this directory:  `cp -r argo-applications/sre/mgmt-storage argo-applications/sre/mgmt-observability`
2. Change `server:` in cluster-secret.yaml and all the names/paths in the
   Application CRs.
3. Run the 6-step runbook above.
