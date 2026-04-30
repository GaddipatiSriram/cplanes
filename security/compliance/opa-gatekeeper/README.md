# opa-gatekeeper

OPA Gatekeeper — Rego-based policy admission for Kubernetes. CNCF
graduated. Sister to Kyverno (sibling chart); both deployed for cert
breadth.

## What this chart deploys

- **gatekeeper-controller-manager** — the admission webhook
- **gatekeeper-audit** — periodic background scan
- **CRDs**: `ConstraintTemplate`, `Constraint`, `Config`, plus
  `MutatingPolicy` (beta), `Provider`, `ConstraintPodStatus`

## The mental model — ConstraintTemplate vs Constraint

This is unique to Gatekeeper and trips beginners:

- **ConstraintTemplate** — a Rego program that *defines* a policy class
  (e.g., "require labels"). Generic. Reusable.
- **Constraint** — an *instance* of the template, scoped to specific
  resources, with specific parameters (e.g., "require label `owner` on
  Deployments").

Two-step thinking compared to Kyverno's all-in-one ClusterPolicy.

## Example: require labels

### Step 1 — ConstraintTemplate (the policy logic)

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata: { name: k8srequiredlabels }
spec:
  crd:
    spec:
      names: { kind: K8sRequiredLabels }
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels
        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("missing label: %v", [missing])
        }
```

### Step 2 — Constraint (the instance)

```yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels       # name from the template above
metadata: { name: deploy-must-have-owner }
spec:
  match:
    kinds: [{ apiGroups: [apps], kinds: [Deployment] }]
  parameters:
    labels: [owner]
```

## Why it's here (cert + role mapping)

| Cert | What it tests |
|---|---|
| **CKS** | Required: ConstraintTemplate + Constraint authorship |
| **CKAD** | Lighter — concept-aware |
| **CNPE** | Capstone — policy-as-code |

| Role | What you'd do here |
|---|---|
| **Security Platform Engineer** | Author baseline ConstraintTemplates per CIS controls |
| **Edge / Gateway Engineer** (the JD) | Use OPA *not* Gatekeeper at the gateway via Envoy ext_authz; same Rego language |

## Cert prep — concrete exercises

1. **Install Gatekeeper Library** — github.com/open-policy-agent/gatekeeper-library
   has 50+ ready ConstraintTemplates. `kubectl apply -k <subdir>`.
2. **Apply 3 starter Constraints** from the library:
   - `K8sPSPAllowedUsers` (no root containers)
   - `K8sRequiredLabels` (require `app.kubernetes.io/name`)
   - `K8sBlockNodePort` (no NodePort Services)
3. **Try to deploy a violating resource** — verify the webhook blocks it.
4. **Author one custom ConstraintTemplate** — e.g., "Deployments must
   set `securityContext.runAsNonRoot: true`."
5. **Use OPA CLI offline** — `opa eval -d policy.rego -i input.json
   "data.k8srequiredlabels.violation"` for fast iteration.

## Coexistence with Kyverno

Both register admission webhooks on **different paths**:
- Kyverno: webhooks under `kyverno` ns
- Gatekeeper: webhooks under `gatekeeper-system` ns

Both run simultaneously without interference. Real orgs typically pick
ONE; we run both for learning + cert breadth.

## Bridging to gateway-level OPA (the JD use case)

Gatekeeper teaches Rego in the k8s admission context. The same Rego
language drives Envoy `ext_authz` filters in the Edge/Gateway tier:

```
k8s admission                         Envoy ext_authz at gateway
─────────────                         ──────────────────────────
ConstraintTemplate (Rego)        ↔    same Rego policy
input = AdmissionReview          ↔    input = HTTP request + headers
violation[msg] returns blocks    ↔    `allow` returns gates request
```

Mastering Gatekeeper Rego = on-ramp to OPA at the gateway = the JD's
"Implement and maintain OPA policies for gateway-level authorization"
requirement.

## Cross-references

- [`/root/learning/certifications/cks.md`](../../../../certifications/cks.md) — CKS prep
- [`../kyverno/`](../kyverno/) — sister policy engine (YAML-based)
- [gatekeeper-library](https://github.com/open-policy-agent/gatekeeper-library) — copy-paste templates
- [OPA Rego playground](https://play.openpolicyagent.org/) — interactive Rego REPL
