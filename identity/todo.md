# identity — TODO

Backlog ordered by likely sequence.

---

## keycloak-config: drop tls_insecure_skip_verify on provider-keycloak

**Status:** open. Current value is the TLS workaround in
`control/crossplane-providers/templates/keycloak-admin-externalsecret.yaml`
(the `"tls_insecure_skip_verify": "true"` line).

Root cause: Keycloak's ingress (in `identity/keycloak/values.yaml`) has
`tls: []` and presents the default ingress.local cert. Real fix is a
cert-manager Certificate CR + valid ClusterIssuer for
`keycloak.apps.mgmt-forge.engatwork.com`. Once the ingress has a real cert,
remove the `tls_insecure_skip_verify` line + restart provider-keycloak pod.

Same fix simultaneously unblocks `oauth2-proxy` (its
`ssl-insecure-skip-verify: "true"` arg) and any future OIDC consumer.

---

## keycloak-config: realm `engatwork` UI changes will be reverted on reconcile

**Status:** behavior, not bug. Document for operators.

provider-keycloak does true upsert. Editing realm/clients/groups/scopes via
the admin UI gets reverted to spec on next reconciliation interval.
Unmanaged surfaces (users, identity providers, federation, brute-force
settings) are untouched. Add a note in the keycloak chart README about
which fields are managed vs. UI-safe.

---

## Author the canonical chart for the first service in this plane

**Status:** not started.

The plane has placeholder folders for each planned service. First step is picking ONE service to bootstrap end-to-end (chart + AppSet + cluster registration) so the pattern is established for the others.

---

## Wire the plane's AppSet pattern into argo-applications/sre/identity/

**Status:** not started.

Decide whether services in this plane are Pattern A (every workload cluster), Pattern B (single central cluster), or one-off Applications. Author the matching ApplicationSet(s) once the chart pattern is clear.

---

## Document plane-specific operational runbooks

**Status:** not started.

Each plane benefits from per-incident runbooks (e.g., "what to do when service X goes down"). Author as encountered.
