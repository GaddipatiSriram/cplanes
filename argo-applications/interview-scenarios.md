# argo-applications — interview scenarios

Architecture / ops / failure / migration scenarios for practice. Each scenario is a prompt — answer it as if walking an interviewer through your design.

---

## Scenario 1: Migrate a Pattern B AppSet to Pattern A

**Context.** Vault was deployed under Pattern B (`role: forge`) — only on `mgmt-forge`. Leadership now wants Vault running on every workload cluster as a local cache / failover, so the AppSet must broaden to Pattern A.

**The ask.** Walk through the migration without losing any existing Vault state on `mgmt-forge`.

**Walk-through.**
- Audit what's cluster-singleton-y about the current install: storage backend (Raft? file?), unseal flow, KV mount paths, k8s auth roles per cluster.
- Decide whether each new cluster gets an independent Vault (simpler, more state) or whether they're performance secondaries / agents pointing at the forge cluster (one source of truth).
- Update the AppSet selector from `role: forge` to `tier In (mgmt, dev), role NotIn (storage)`. Diff the generated Applications before applying.
- Stage the rollout: pause auto-sync, apply the AppSet change, manually sync one cluster first, verify pod health and seal status, then unpause.
- Plan the per-cluster bootstrap: if independent Vaults, each needs its own init+unseal, k8s auth role config, and ESO ClusterSecretStore.

**Tradeoffs.** Independent Vaults eliminate cross-cluster blast radius but multiply operational toil (5 unseal procedures, 5 PKI roots). Performance secondaries keep one source of truth but add a cross-cluster network dependency to every secret read.

**Follow-up questions.**
- How would you handle cert-manager's dependency on Vault PKI during the cluster that hosts the forge Vault being rebuilt?
- What ArgoCD sync waves would you set so the new Vault Pod is ready before any workload tries to use it?
- How do you make this reversible if the migration goes wrong?

---

## Scenario 2: An `argocd-sre` Application is stuck in `OutOfSync` with a self-heal loop

**Context.** A platform engineer reports cert-manager on `mgmt-workload` is flapping — ArgoCD shows `OutOfSync` for 30 seconds, syncs, then immediately flips back to `OutOfSync`. CPU on the argocd-application-controller is elevated.

**The ask.** Diagnose and propose a fix.

**Walk-through.**
- Pull the diff: `argocd app diff cert-manager-mgmt-workload`. Look for fields ArgoCD wants to set but something else is overwriting.
- Common culprits: a mutating webhook (e.g., kyverno, an admission controller) is rewriting an annotation; a controller is owning a field ArgoCD also manages; `metadata.generation` differs because of a default the API server fills in.
- Check `serverSideApply` and `RespectIgnoreDifferences` — sometimes the fix is teaching ArgoCD to ignore a field rather than fight for it.
- Check if a `managed-by` label was stripped (recall: `argocd-sre` is namespace-scoped, target namespaces need the label).
- Verify no operator (e.g., the cert-manager operator itself) is reconciling the same CRDs ArgoCD is templating.

**Tradeoffs.** Adding `ignoreDifferences` is fast but brittle — it hides drift. Fixing the upstream cause (e.g., excluding the field from the chart's templating) is durable but requires a chart change.

**Follow-up questions.**
- What's the difference between `Replace=true`, `ServerSideApply=true`, and the default 3-way merge?
- How does ArgoCD's Application Controller decide a resource is `OutOfSync`?
- What metric would you alert on to catch this kind of fight loop early?
