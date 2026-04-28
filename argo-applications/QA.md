# argo-applications — learning Q&A

Study log for the GitOps layer that controls every other domain. Use as a study reference; revisit when abstractions stop making sense.

---

## Q1. Why two ArgoCD instances (`argocd-sre` and `argocd-devops`) on OKD instead of one?

**A.** Separation of concerns and blast radius. `argocd-sre` owns platform / infra workloads (cert-manager, ingress, Vault, Keycloak, Prometheus) — things that, if broken, take down the cluster. `argocd-devops` owns application-tier deploys (CI runners, dev workflows, app namespaces) where churn is high and failures are recoverable.

Splitting them means an SRE-side RBAC/AppProject change doesn't risk every dev pipeline, and a misconfigured devops Application can't reconcile itself into deleting `kube-system` resources. It also lets the two instances run different sync windows, RBAC, and notifications.

The `argocd-sre` instance is namespace-scoped (see auto-memory note) — target namespaces need a `managed-by` label for it to write into them.

---

## Q2. What's the difference between an `Application` and an `ApplicationSet`?

**A.** An `Application` is a single deployable unit: one source (Git path / Helm chart), one destination (cluster + namespace), one sync policy. You write it directly when you have a one-off deployment that doesn't fan out — e.g. the rook-ceph server on `mgmt-storage`.

An `ApplicationSet` is a generator that produces N `Application` objects from a template. We use the cluster generator with label selectors to express the deployment Patterns:

- **Pattern A** — `tier In (mgmt, dev), role NotIn (storage)` produces one Application per workload cluster. Used for cert-manager, ingress-nginx, ESO, trivy-operator, the rook-ceph CSI consumer.
- **Pattern B** — `role: forge` targets only `mgmt-forge`. Used for Keycloak, Vault, Harbor.
- **Pattern C** — `role: observ` targets only `mgmt-observability`. Used for Prometheus, Grafana, Alertmanager.

An AppSet is the right shape when "this thing deploys the same way on every cluster matching X."

---

## Q3. How do `argocd-sre` and `argocd-devops` discover the workload clusters they target?

**A.** Each ArgoCD instance has cluster Secrets registered (typically via `argocd-cluster-secrets` controllers or manually by the bootstrap process). The Secret carries the cluster's API endpoint, CA bundle, and a service-account bearer token plus the labels (`tier`, `role`) that AppSet generators select on. Removing or relabeling a cluster Secret is how you take a cluster out of an AppSet's scope without editing the AppSet itself.
