# argo-applications/ TODOs

Planning doc for pending ArgoCD-managed platform work. Items here are pre-
implementation; once a task is in-flight, the actual manifests live under
`argo-applications/sre/<cluster>/` or `argo-applications/devops/` and this doc gets updated/closed.

---

## D3a — Fan ESO out to the OKD cluster

**Status:** not started
**Roadmap ref:** `/root/learning/README.md:162` (D3a)
**Depends on:** ESO on mgmt-forge (D3, done) · Vault post-install (done)
**Blocks:** any ArgoCD Application on OKD that needs Vault-sourced secrets
(ARC GitHub App creds, Harbor cross-cluster pulls, Keycloak OIDC secrets
consumed from OKD, etc.)

### Why

Today ESO runs only on `mgmt-forge` (`argo-applications/sre/forge/external-secrets-appset.yaml`
selector = `tier In (mgmt, dev), role NotIn (storage)`). The
`ClusterSecretStore` template hardcodes in-cluster Vault DNS
(`http://vault.vault.svc:8200`) and the Vault role `forge-reader`, so the
AppSet technically *would* match OKD if OKD had the `tier=mgmt` label, but the
resulting pod would crashloop — it can't resolve Vault and wouldn't be
authorized if it could.

Getting ESO onto OKD unlocks secret delivery to every Application that lives
on the management cluster, starting with ARC (see B-series below).

### The four changes (ordered by dependency)

#### 1. Vault auth role for OKD  *(one-time, NOT GitOps)*

Vault's k8s auth method validates SA JWTs against a specific cluster's API +
CA. The existing mount `auth/kubernetes/` is bound to forge. OKD needs its
own mount or role with OKD's API server + CA + reviewer JWT.

**Where:** extend `forge/vault/playbooks/configure.yml` (or add
`configure-okd-auth.yml`).

**What to run:**

```bash
# On a host with Vault CLI + a token with root policy
vault auth enable -path=kubernetes-okd kubernetes

vault write auth/kubernetes-okd/config \
  kubernetes_host="https://api.mgmt-devops-okd.engatwork.com:6443" \
  kubernetes_ca_cert=@okd-ca.crt \
  token_reviewer_jwt=@okd-reviewer-token \
  disable_iss_validation=true

vault write auth/kubernetes-okd/role/okd-reader \
  bound_service_account_names=external-secrets \
  bound_service_account_namespaces=external-secrets \
  policies=forge-reader \
  ttl=1h
```

**How to get the CA + reviewer JWT:**

```bash
# From OKD
oc --context mgmt-devops-okd create sa vault-reviewer -n kube-system
oc --context mgmt-devops-okd adm policy add-cluster-role-to-user \
    system:auth-delegator -z vault-reviewer -n kube-system

oc --context mgmt-devops-okd create token vault-reviewer \
    -n kube-system --duration=87600h > okd-reviewer-token

oc --context mgmt-devops-okd get cm kube-root-ca.crt -n kube-system \
    -o jsonpath='{.data.ca\.crt}' > okd-ca.crt
```

**Policy** `forge-reader` already grants `read` on `kv/data/forge/*`. If OKD
needs a disjoint path (e.g. `kv/data/okd/*`), author `okd-reader` as a
separate policy and bind it to the role instead.

**Verify:**
```bash
vault read auth/kubernetes-okd/role/okd-reader   # role exists
# Later, after ESO is on OKD:
vault read sys/internal/counters/tokens          # count bumps when ESO auths
```

#### 2. ESO chart — parameterize `ClusterSecretStore`  *(GitOps)*

**Where:** `forge/external-secrets/` chart.

**File:** `forge/external-secrets/values.yaml` — add:

```yaml
vaultStore:
  name: vault-forge                            # CR name
  server: http://vault.vault.svc:8200          # in-cluster default (forge)
  authMountPath: kubernetes                    # Vault auth mount
  role: forge-reader                           # Vault role
```

**File:** `forge/external-secrets/templates/cluster-secret-store.yaml` —
swap hardcoded values for template references:

```yaml
metadata:
  name: {{ .Values.vaultStore.name }}
spec:
  provider:
    vault:
      server: {{ .Values.vaultStore.server | quote }}
      path: "kv"
      version: "v2"
      auth:
        kubernetes:
          mountPath: {{ .Values.vaultStore.authMountPath | quote }}
          role: {{ .Values.vaultStore.role | quote }}
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

**Smoke-test ExternalSecret** — `smoke-test-externalsecret.yaml` currently
references `vault-forge` by name; bind it to `{{ .Values.vaultStore.name }}`
too, or accept that only the forge store exercises the smoke test.

#### 3. ESO chart — OpenShift SCC binding (OKD only)  *(GitOps)*

OKD's default `restricted-v2` SCC will block the ESO pod unless bound to
something compatible. Gate an SCC `RoleBinding` behind a flag so forge stays
unchanged.

**File:** `forge/external-secrets/values.yaml` — add:

```yaml
openshift:
  enabled: false
```

**File:** `forge/external-secrets/templates/scc-binding.yaml` — new:

```yaml
{{- if .Values.openshift.enabled }}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: external-secrets-scc
  namespace: external-secrets
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:nonroot-v2
subjects:
  - kind: ServiceAccount
    name: external-secrets
{{- end }}
```

#### 4. OKD cluster registration + AppSet parameter override  *(GitOps)*

**File:** `argo-applications/sre/okd/cluster-secret.yaml` — new. Mirrors the
`mgmt-storage` shape but uses in-cluster kubeconfig since argocd-sre runs on
OKD itself.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mgmt-devops-okd
  namespace: argocd-sre
  labels:
    argocd.argoproj.io/secret-type: cluster
    tier: mgmt
    role: okd
    env: nonprod
    managed-by: argo-sre-bootstrap
type: Opaque
stringData:
  name: mgmt-devops-okd
  server: https://kubernetes.default.svc
  config: '{"tlsClientConfig":{"insecure":false}}'
```

**First — audit the operator-managed default.** `argo-applications/README.md:489` notes an
existing `argocd-sre-default-cluster-config` for the in-cluster destination.
Before creating a parallel cluster Secret, check:

```bash
oc --context mgmt-devops-okd get secret -n argocd-sre \
    -l argocd.argoproj.io/secret-type=cluster -o yaml
```

If the operator-managed Secret is already there, **patch labels onto it
instead** of adding a duplicate registration (two cluster Secrets for the
same server URL = AppSet generates two Applications with clashing names).

**File:** `argo-applications/sre/forge/external-secrets-appset.yaml` — the existing
selector already matches `tier=mgmt, role=okd`, so no selector edit is
required.

What *does* need editing: the AppSet must pass per-cluster Helm values so
OKD's ClusterSecretStore uses the external Vault URL + `kubernetes-okd` mount
+ `okd-reader` role, while forge keeps the in-cluster defaults. Cleanest
approach is a **matrix generator** (clusters × list) or inline
`helm.parameters` keyed on `{{metadata.labels.role}}`:

```yaml
spec:
  # ... existing clusters generator ...
  template:
    spec:
      source:
        path: external-secrets
        helm:
          releaseName: external-secrets
          valuesObject:
            vaultStore:
              name: 'vault-{{metadata.labels.role}}'
              # these two overridden below per cluster via a second generator
              server: http://vault.vault.svc:8200
              authMountPath: kubernetes
              role: forge-reader
            openshift:
              enabled: '{{metadata.labels.role == "okd"}}'
```

The cleanest shape is a **merge generator** combining the `clusters`
selector with a `list` of overrides keyed by cluster name — draft this
shape in the PR rather than trying to template it inline in this doc.

### Verification

After all four changes are in place and ArgoCD has synced:

```bash
# ESO pod up on OKD
oc --context mgmt-devops-okd get pods -n external-secrets

# ClusterSecretStore ready (points at forge's Vault ingress)
oc --context mgmt-devops-okd get clustersecretstore vault-okd -o yaml | \
    grep -A2 conditions:

# End-to-end smoke: write a KV, create an ExternalSecret, read the Secret
vault kv put kv/okd/hello message="hello from okd"

oc --context mgmt-devops-okd apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: hello-from-vault-okd
  namespace: default
spec:
  refreshInterval: 1m
  secretStoreRef:
    name: vault-okd
    kind: ClusterSecretStore
  target:
    name: hello-from-vault-okd
  data:
    - secretKey: message
      remoteRef:
        key: okd/hello
        property: message
EOF

oc --context mgmt-devops-okd -n default get secret hello-from-vault-okd \
    -o jsonpath='{.data.message}' | base64 -d
# expected: "hello from okd"
```

### Risks / open questions

- **Vault ingress TLS.** Cross-cluster traffic goes through
  `https://vault.apps.mgmt-forge.engatwork.com`. If that endpoint uses a
  self-signed cert from cert-manager's local CA, ESO on OKD will reject it
  unless the CA bundle is injected into the `ClusterSecretStore`
  (`.spec.provider.vault.caBundle` or `caProvider`). Confirm what issuer
  Vault's ingress uses before assuming trust works.
- **MetalLB vs ingress for Vault.** If Vault is exposed via a MetalLB
  LoadBalancer rather than ingress-nginx, the URL changes. Check
  `forge/vault/values.yaml` / the Service type.
- **Network reachability forge ↔ OKD.** VLAN 14 (forge, 10.10.5.0/24) and
  VLAN 12 (OKD, 10.10.2.0/24) must route with pfSense rules permitting OKD →
  forge:443 (or whichever port Vault ingress listens on).
- **Dual registration.** If the OKD operator already registered in-cluster
  via its own cluster Secret, adding `cluster-secret.yaml` manually creates
  a duplicate. Audit first.

---

## B-series — ARC on OKD (depends on D3a)

Placeholder. Once D3a is green, the ARC plan (controller chart +
RunnerScaleSet + GitHub App secret sourced via ESO) gets filled in here.
Current sketch captured in conversation; formalize before implementation.

---

## L1 — "Hello from platform" consumer service (learning exercise)

**Status:** not started.
**Why:** every platform layer in this repo (Vault, Keycloak, ESO, cert-manager, ingress-nginx, Loki, Prometheus, ArgoCD) was built without a real consumer exercising the whole chain end-to-end. Each layer was tested in isolation. Building one tiny app that touches every layer is the highest-leverage way to (a) validate the platform actually works for app teams and (b) close the foundations gap captured in `/root/learning/LEARNING-NOTES.md`.

**Reference:** `LEARNING-NOTES.md` §5 (App-builder consumer view).

### Done = a 50-200 line service that does all of:

1. **OIDC client registered via Crossplane** — add a `Client` CR in `cplanes/identity/keycloak-config/values.yaml` named `learning-app`. Client secret auto-generated by Keycloak, mirrored to Vault by PushSecret (existing pattern).
2. **Pull `client_secret` via ESO** — `ExternalSecret` in the app's namespace pulling `kv/forge/keycloak/clients/learning-app` → `Secret/learning-app-oidc`.
3. **Validate JWTs from Keycloak** — fetch JWKS from `https://keycloak.apps.mgmt-forge.engatwork.com/auth/realms/engatwork/protocol/openid-connect/certs`; verify `iss`, `aud`, `exp`, signature.
4. **Expose via ingress with vault-pki cert** — ingress annotation `cert-manager.io/cluster-issuer: vault-pki`; cert auto-issued.
5. **Structured stdout logs** — JSON, single line per event. Promtail picks up automatically; verify in Grafana → Explore → Loki via `{cluster="mgmt-control",namespace="learning-app"}`.
6. **Prometheus metrics endpoint** — `/metrics` with at minimum `http_requests_total{path,method,status}` and `http_request_duration_seconds`. ServiceMonitor CR scrapes it.
7. **Helm chart in `cplanes/`** — small umbrella, no subchart, just hand-rolled templates (Deployment, Service, Ingress, ServiceMonitor, ExternalSecret).
8. **AppSet wiring it up** — Pattern A or single Application in `argo-applications/sre/control/` or wherever fits.

### What this exercises (tied back to LEARNING-NOTES)

- Identity §1 — OIDC flow, JWT, JWKS, custom claims if you add `groups`-based auth in the app.
- Secrets §2 — full ESO chain from Vault path → mounted env var.
- Certs §3 — vault-pki ClusterIssuer minting a real cert end-to-end for your own service.
- K8s §4 — Deployment/Service/Ingress/ServiceMonitor/ExternalSecret authored from scratch (no umbrella subchart hiding things).
- App-side §5 — 12-factor config, JWT validation in code, structured logs, metrics endpoint.
- Observability §6 — once data flows, query LogQL + PromQL for your own service and confirm what you sent shows up.

### Open design choice (decide before starting)

Language. Pick what you'll keep using long-term:
- **Go** — single static binary, smallest container, idiomatic for k8s ecosystem (controller-runtime, prometheus client are first-class). Bigger learning curve if not already comfortable.
- **Python (FastAPI)** — fastest to write, great OIDC libs (`authlib`), `prometheus_client` is solid. Slower runtime, larger image.
- **TypeScript (Fastify or Hono)** — if you want to keep the option of building a Backstage plugin later (Backstage is TS).

No wrong answer; pick one and stick with it across the platform.

### Follow-up — Backstage software template

Once L1 is working manually, the natural next exercise is wrapping it as a Backstage software template so the next service is `npx`-scaffolded. That's the moment the platform becomes self-service.
