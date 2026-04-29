# control — TODO

Backlog ordered by likely sequence.

---

## crossplane-providers: replace skip_tls_verify with proper CA bundle

**Status:** Phase 1 done (cert-manager flow), Phase 2 open (drop the flags).

Phase 1 (2026-04-29 ✅) — `vault-pki` ClusterIssuer now exists on every
cert-manager cluster (mgmt-forge, mgmt-control, mgmt-observability) and
issues engatwork-CA-signed certs. ClusterIssuer template moved from
`security/secrets/vault/` to `security/ca-pki/cert-manager/`; AppSet
sets per-cluster `vaultPki.kubernetesAuthMount=kubernetes-{{name}}`.
Vault-side: `cert-manager-issuer` Policy + 3 AuthBackendRoles (one per
cluster's auth mount) declared via Crossplane vault-config. Crossplane-
admin policy expanded to include `auth/kubernetes-mgmt-observability/role/*`.

Phase 2 (open) — drop the `skip_tls_verify` / `tls_insecure_skip_verify`
flags on provider-vault and provider-keycloak. Blocked on mounting the
engatwork CA into the provider pods (their default trust stores don't
include it).

Plan:
1. Author a `DeploymentRuntimeConfig` for each provider that mounts a
   ConfigMap or Secret holding the engatwork Root CA at e.g.
   `/etc/ssl/certs/engatwork-ca.crt` (appended to system bundle).
2. provider-vault: set `VAULT_CACERT=/etc/ssl/certs/engatwork-ca.crt`
   env var via the runtime config (provider-vault's ProviderConfig spec
   has no `ca_cert_file` field, so pod-level env is the path).
3. provider-keycloak: same volume mount; provider reads
   `tls_insecure_skip_verify` from the JSON credentials but also
   honors system trust roots.
4. Drop the `skip_tls_verify` lines from both ProviderConfig templates.
5. Restart provider pods → confirm Healthy without skip-verify.

CA source-of-truth question: today the engatwork CA PEM is duplicated
inline in `security/secrets/external-secrets/values.yaml` and
`security/ca-pki/cert-manager/values.yaml`. Phase 2 should consolidate
to one ConfigMap (or ESO-rendered Secret) all four consumers reference.

Cookie secret seeding for oauth2-proxy is a separate manual residue —
see security/todo.md.

---

## crossplane-providers: pin provider-vault SA name via DeploymentRuntimeConfig

**Status:** open. Today the `crossplane` Vault role binds `bound_service_account_names='*'`
because Crossplane's package manager assigns hash-suffixed SA names we
can't predict. Authoring a `DeploymentRuntimeConfig` to pin the SA name
to e.g. `provider-vault` would let us tighten the bind to exactly one
SA. Same opportunity for provider-keycloak.

---

## Author the canonical chart for the first service in this plane

**Status:** not started.

The plane has placeholder folders for each planned service. First step is picking ONE service to bootstrap end-to-end (chart + AppSet + cluster registration) so the pattern is established for the others.

---

## Wire the plane's AppSet pattern into argo-applications/sre/control/

**Status:** not started.

Decide whether services in this plane are Pattern A (every workload cluster), Pattern B (single central cluster), or one-off Applications. Author the matching ApplicationSet(s) once the chart pattern is clear.

---

## Document plane-specific operational runbooks

**Status:** not started.

Each plane benefits from per-incident runbooks (e.g., "what to do when service X goes down"). Author as encountered.
