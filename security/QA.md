# security — learning Q&A

Cluster-wide security tooling: scanning, policy, runtime detection.

---

## Q1. Why trivy-operator (per-cluster) and Harbor's Trivy (registry-side)?

**A.** Different threat models, different timing.

- **Harbor's Trivy** scans an image when it's pushed. Catches known CVEs at supply-chain time. Doesn't catch CVEs published *after* the image was scanned.
- **trivy-operator** scans every *running* pod's image, periodically (default 1h). Catches CVEs that became known after deploy. Generates `VulnerabilityReport` CRDs you can query.

Together: Harbor blocks bad images entering the system; trivy-operator catches images that became bad after entering. Run both.

---

## Q2. OPA/Gatekeeper vs Kyverno?

**A.**
- **OPA + Gatekeeper**: policies in Rego (a domain-specific language). Steep learning curve; very expressive. Same engine can enforce policies for non-k8s things (Terraform, CI/CD, AWS APIs). Industry standard.
- **Kyverno**: policies in YAML. Easy to learn; k8s-native. Limited expressiveness (no general programming language). Easier for teams new to policy-as-code.

For a platform team that'll author policies for multiple systems, OPA is the long-term pick. For "I just need to block privileged pods," Kyverno is simpler.

---

## Q3. What's the difference between admission policy and runtime detection?

**A.**
- **Admission policy** (OPA/Gatekeeper, Kyverno): the API server checks every CREATE/UPDATE against policies. Bad pods are rejected before they ever run. *Prevention.*
- **Runtime detection** (Falco): a daemon watches syscalls / network / filesystem on the running node. Fires alerts when behavior deviates from the rule set (e.g., shell in container, cryptominer). *Detection.*

Both matter. Admission catches misconfigurations; runtime catches active compromise. A privileged container blocked at admission can't be exploited; a runtime-only stance lets the bad config in and detects abuse after the fact.

---

## Q4. Why are CRDs (VulnerabilityReport, ConfigAuditReport) better than just Slack alerts?

**A.** CRDs are *data* — queryable, versioned, joinable. Slack alerts are *events* — ephemeral, hard to aggregate.

With trivy-operator's CRDs you can:
- `kubectl get vulnerabilityreport -A -o jsonpath='{.items[?(@.report.summary.criticalCount > 0)].metadata.name}'` — list all reports with criticals.
- Build a Grafana dashboard via prometheus-trivy-operator-exporter that surfaces vuln counts over time.
- Export to DefectDojo for triage.

Slack alerts are fine for *fresh* events ("a new critical was just found"), but they don't replace the persistent data store.
