# kyverno

Kubernetes-native policy engine. YAML-based policies — no Rego required —
supporting validate / mutate / generate / cleanup / verifyImages rule
types. Sister project to OPA Gatekeeper; both can coexist.

## What this chart deploys

- **admission controller** — runs as a ValidatingWebhookConfiguration +
  MutatingWebhookConfiguration; enforces policies on every API call
- **background controller** — periodic scan of existing resources against
  policies (catches drift)
- **cleanup controller** — runs CleanupPolicy CRs (newer feature)
- **reports controller** — emits `PolicyReport` + `ClusterPolicyReport`
- **CRDs**: `Policy`, `ClusterPolicy`, `PolicyException`, `CleanupPolicy`,
  `ClusterCleanupPolicy`, `PolicyReport`, `ClusterPolicyReport`

## Why it's here (cert + role mapping)

| Cert | What it tests |
|---|---|
| **KCA** (Kyverno Certified Associate) | Entire exam is this chart |
| **CKS** | Tests admission policies generally; usually Gatekeeper but Kyverno is acceptable |
| **CNPE** | Policy-as-code is a capstone capability |

| Role | What you'd do here |
|---|---|
| **Security Platform Engineer** | Author baseline policies (no privileged, require labels) |
| **Cloud Platform Engineer** | Enforce platform conventions (resource limits, naming) |
| **Compliance Engineer** | CIS / NIST controls expressed as policies |

## The 5 rule types — one example each

### 1. Validate (most common)

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: { name: require-labels }
spec:
  validationFailureAction: Audit   # Audit = warn; Enforce = block
  rules:
    - name: require-owner-label
      match:
        any:
          - resources:
              kinds: [Deployment]
      validate:
        message: "Deployments must have an 'owner' label."
        pattern:
          metadata:
            labels:
              owner: "?*"
```

### 2. Mutate (auto-fix)

```yaml
spec:
  rules:
    - name: add-default-resources
      match:
        any: [{ resources: { kinds: [Pod] } }]
      mutate:
        patchStrategicMerge:
          spec:
            containers:
              - (name): "*"
                resources:
                  requests:
                    cpu: "100m"
                    memory: "128Mi"
```

### 3. Generate (auto-create)

```yaml
spec:
  rules:
    - name: default-network-policy
      match:
        any: [{ resources: { kinds: [Namespace] } }]
      generate:
        kind: NetworkPolicy
        name: default-deny
        namespace: "{{ request.object.metadata.name }}"
        synchronize: true
        data:
          spec:
            podSelector: {}
            policyTypes: [Ingress]
```

### 4. Cleanup (delete on schedule)

```yaml
apiVersion: kyverno.io/v2alpha1
kind: ClusterCleanupPolicy
metadata: { name: cleanup-failed-pods }
spec:
  schedule: "0 0 * * *"   # daily
  match:
    any: [{ resources: { kinds: [Pod] } }]
  conditions:
    any:
      - key: "{{ target.status.phase }}"
        operator: Equals
        value: Failed
```

### 5. VerifyImages (supply chain)

```yaml
spec:
  rules:
    - name: verify-cosign
      match:
        any: [{ resources: { kinds: [Pod] } }]
      verifyImages:
        - imageReferences: ["ghcr.io/myorg/*"]
          attestors:
            - entries:
                - keys: { publicKeys: "<cosign pubkey>" }
```

## Cert prep — concrete exercises

1. **Install policies/best-practices**: Kyverno publishes a curated set —
   `kubectl apply -f https://raw.githubusercontent.com/kyverno/policies/main/best-practices/...` — install 5-10 starter policies.
2. **Deploy a Pod that violates one** — see PolicyReport reflect it.
3. **Switch one policy from Audit → Enforce** — try to deploy violating
   resource, verify it's blocked.
4. **Author a custom mutation** — auto-inject `securityContext.runAsNonRoot: true`.
5. **Author a custom generation** — namespace creation triggers default
   ResourceQuota.
6. **Use Kyverno CLI** — `kyverno apply policy.yaml --resource=pod.yaml`
   to test offline.

## Coexistence with OPA Gatekeeper

Both register webhooks on different paths:
- Kyverno: `/validate/fail` and `/mutate/fail` (in `kyverno` ns)
- Gatekeeper: `/v1/admit` (in `gatekeeper-system` ns)

Both can run simultaneously. Use Kyverno for YAML-friendly policies,
Gatekeeper for organizations already using Rego elsewhere (OPA, Conftest).

## Cross-references

- [`/root/learning/certifications/kca.md`](../../../../certifications/kca.md) — KCA prep
- [`../opa-gatekeeper/`](../opa-gatekeeper/) — sister policy engine (Rego-based)
- [Kyverno Policy Library](https://kyverno.io/policies/) — copy-paste starter pack
