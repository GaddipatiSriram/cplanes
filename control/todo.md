# control — TODO

Backlog ordered by likely sequence.

---

## crossplane-providers: replace skip_tls_verify with proper CA bundle

**Status:** open. Both ProviderConfigs currently bypass TLS verification:
- `templates/provider-config-vault.yaml`: `skip_tls_verify: true`
- `templates/keycloak-admin-externalsecret.yaml`: `"tls_insecure_skip_verify": "true"` in the credentials JSON

Root cause: provider pods don't trust the engatwork Root CA (Vault PKI).
Real fix: mount the engatwork CA into provider pods via
`DeploymentRuntimeConfig` (Crossplane v1.14+), then pass the CA path
or content via the provider's TLS-config field.

Open question: provider-vault's ProviderConfig spec has no `ca_cert_file`
field exposed (schema only has address/credentials/headers/etc). May need
to set `VAULT_CACERT` env var in the runtime config, or add CA to the
Secret's JSON as `ca_cert_pem`.

Cookie secret seeding for oauth2-proxy is the other manual residue — see
security/todo.md.

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
