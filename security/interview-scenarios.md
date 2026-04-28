# security — interview scenarios

---

## Scenario 1: A new CRITICAL CVE was published for an image running in production

**Context.** A new CVE (let's call it CVE-2026-1234) drops Friday afternoon, affecting `nginx:1.25.x` — running in 14 pods across mgmt-forge and dev-* clusters. trivy-operator's hourly scan picks it up.

**The ask.** Walk through the response.

**Walk-through.**
- **Triage** (T+0): pull the `VulnerabilityReport` CRDs to confirm scope. `kubectl get vulnerabilityreport -A -o yaml | yq '.items[] | select(.report.vulnerabilities[].vulnerabilityID == "CVE-2026-1234") | .metadata.name'`. Confirm: 14 reports.
- **Severity assessment** (T+15m): Read the CVE. Is the vulnerable code path actually reachable from outside? (e.g., a parser bug only triggered by malformed Host header — exploitable via ingress.) Or is it a corner case?
- **Patch availability** (T+30m): Does the `nginx:1.25.x` image have a fix? Check Docker Hub. If yes → bump tag in chart values, ArgoCD auto-syncs.
- **Mitigation** (T+1h, if no patch): Add an OPA admission policy that blocks new deploys of the vulnerable tag. Existing pods: deploy a NetworkPolicy that restricts the affected ingress paths.
- **Verification** (T+24h): trivy-operator's next scan should show 0 reports for CVE-2026-1234.

**Tradeoffs.**
- Force-reschedule pods now (rolling restart) vs. wait for natural rollout: faster patching vs. risk to availability.
- Block at OPA vs. mark images "deprecated" in Harbor: OPA blocks new deploys; Harbor marker is informational.

**Follow-up questions.**
- "What if the patched image has a different sha256 than what's pinned in your Application?" → ArgoCD's auto-sync will swap. If you pin sha256 instead of tag, you have to manually update values.yaml — better security hygiene, more friction.
- "How would you communicate the impact?" → Slack the security channel with the report list, the proposed mitigation, and the planned timeline. Stakeholders include: app owners (their workload), platform team (chart maintenance), security (audit trail).

---

## Scenario 2: An OPA policy you authored is rejecting a legitimate Helm chart upgrade

**Context.** You wrote `disallow-privileged.rego` to block `privileged: true`. The Bitnami PostgreSQL chart's init container (`bitnami-shell`) sets `privileged: true` for one specific permission setup. The chart upgrade fails admission.

**The ask.** Resolve.

**Walk-through.**
- Don't loosen the policy globally. Instead, exempt only this specific init container.
- Three approaches:
  - **Namespace exemption**: add `excludedNamespaces: [postgres-system]` to the `Constraint`. Coarse — exempts ALL privileged containers in that namespace.
  - **Label exemption**: match on a `policy/exemption=postgres-init` label. Workloads must opt in by labeling. Fine-grained.
  - **Image exemption**: match on `image: docker.io/bitnami/bitnami-shell:*`. Very specific; a malicious init container with the same image still bypasses.
- I'd pick **label exemption** + a process: any team requesting an exemption files an issue, security reviews, label gets added to the workload's manifest.

**Tradeoffs.**
- Tight policy + lots of exemptions = bureaucratic friction. Loose policy + fewer exemptions = real risk. The label approach makes the exemptions visible (`kubectl get pods -A -l policy/exemption -o wide`).
- Could replace Bitnami's init with a custom one that doesn't need `privileged`. Time-consuming; couples your upgrade to chart-internal changes.

**Follow-up questions.**
- "How would you audit how often each exemption is used?" → Prometheus metric exposed by Gatekeeper for `auditViolations` + a per-exemption counter. Alert if an exemption is unused for >30 days (stale → remove).
- "What's the difference between a `Constraint` and a `ConstraintTemplate`?" → Template defines the schema + Rego logic; Constraint is an instance with parameters. Like CRD-and-CR.
