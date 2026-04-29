# identity — TODO

Backlog ordered by likely sequence.

---

## keycloak-config: drop tls_insecure_skip_verify on provider-keycloak

**Status:** Phase 1 done, Phase 2 open.

Phase 1 (2026-04-29 ✅) — Keycloak ingress now serves a vault-pki cert:
`identity/keycloak/values.yaml` has `cert-manager.io/cluster-issuer: vault-pki`
+ a tls block; the ingress at `https://keycloak.apps.mgmt-forge.engatwork.com`
presents an `engatwork.com Root CA`-signed cert.

Phase 2 (open) — drop the `"tls_insecure_skip_verify": "true"` line in
`control/crossplane-providers/templates/keycloak-admin-externalsecret.yaml`.
provider-keycloak pod's trust store doesn't include engatwork CA today,
so dropping the flag would break HTTPS verify. Real fix: mount the
engatwork CA into the provider pod via a `DeploymentRuntimeConfig`
(volume + volumeMount), then drop the JSON flag and restart the pod.

Same Phase 2 cleanup applies to:
- `oauth2-proxy` (`ssl-insecure-skip-verify: "true"`) — needs CA mounted via extraVolumes.
- `provider-vault` (`skip_tls_verify: true`) — same DeploymentRuntimeConfig pattern.
- `promtail` (`tls_config.insecure_skip_verify`) — extraVolumes on the DaemonSet.

All four drop together. Tracked in SESSION-NOTES "Phase 2".

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
