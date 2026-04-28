# security — TODO

Backlog ordered by likely sequence.

---

## Trivy DB mirror in Harbor

**Status:** not started. Blocks reliable cluster bootstrapping at scale.

Trivy fetches its CVE DB from `ghcr.io/aquasecurity/trivy-db` on every cluster install. GitHub rate-limits unauth requests at ~60/hr, which is fine for one cluster but dies on bulk rebuilds. Set up Harbor as a pull-through cache for `ghcr.io/aquasecurity/*`. Done = trivy-operator on a fresh cluster reads from `harbor.apps.mgmt-forge.engatwork.com/aquasecurity/trivy-db` instead of GitHub.

---

## Author OPA/Gatekeeper baseline policy bundle

**Status:** not started.

Start with the basics: no `hostPath` volumes, no `privileged: true` containers, no `runAsUser: 0`, image must come from Harbor (`*.apps.mgmt-forge.engatwork.com/*`). Per-namespace exemptions for kube-system / metallb-system / etc. Done = a deliberately-bad Pod is rejected at admission with a clear message.

---

## Add Falco DaemonSet via Pattern A AppSet

**Status:** not started. Lower priority than OPA (admission > runtime for prevention).

Author `forge/falco/` chart (or use the upstream Helm chart umbrella) with default ruleset, output forwarding to Wazuh manager (when search tier is up). Done = `kubectl exec` into a pod and run `bash` → Falco fires "shell spawned" alert visible in Wazuh dashboard.

---

## Migrate trivy-operator → broader scope (Pattern A → also dev clusters)

**Status:** small. Selector already matches dev clusters — they just need to be registered.

When dev-{web,apps,data} clusters are bootstrapped, Pattern A picks them up automatically. Verify on first dev cluster: `kubectl get vulnerabilityreport -A` shows reports within 5 minutes.
