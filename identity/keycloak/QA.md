# Keycloak / IAM Learning Q&A

Study log for understanding Identity and Access Management before diving
deeper into Keycloak configuration. Kept so the conceptual foundation is
searchable and can be revisited when the abstractions in the admin UI stop
making sense.

Reading-time budget to solid mental model: ~5 hours. Plus 2-3 hours hands-on.

---

## Q1. Why IAM exists at all — what problem is Keycloak solving?

**A.** Without centralized identity, every application owns its own user
table, password storage, and user lifecycle. Consequences:

1. **Security** — each app is a breach surface with its own password hashing,
   its own rate limiting, its own (often missing) MFA. One sloppy app leaks
   credentials that users reused elsewhere.
2. **UX** — users juggle N passwords, N login forms.
3. **Ops / HR** — adding a new hire or offboarding a leaver means touching N
   systems.

A central identity provider (IdP) like Keycloak consolidates this: apps stop
storing credentials and delegate login to the IdP. MFA, password policy, user
lifecycle, and audit all live in one place.

The four pillars of a full IAM system:

- **Identity store** — where user records live (DB, LDAP, federated).
- **Authentication** — login UX, MFA, brute-force protection.
- **SSO / federation** — log in once, access many apps; or accept identity
  from Google/GitHub/AD.
- **Authorization** — roles, groups, policies that decide what the user can do.

Keycloak covers all four.

Recommended intro reads (each ~15 min): Okta "What is IAM" overview, Auth0
"Identity 101". Either one is enough to get the vocabulary.

---

## Q2. AuthN vs AuthZ — why do protocols keep these separate?

**A.**

- **Authentication (AuthN)**: proves *who you are*. Protocols: OIDC, SAML.
- **Authorization (AuthZ)**: decides *what you can do*. Mechanisms: OAuth 2.0
  scopes, roles, policies (XACML, OPA, Keycloak Authorization Services).

They are distinct because a system may need one without the other:

- A background job accessing an API needs AuthZ (access token) but has no
  human user to authenticate.
- A corporate directory lookup may need AuthN (is this really Alice?)
  without any specific resource being authorized.

Keep these mentally separate. Every modern "login" protocol is actually
both glued together: OIDC is "OAuth 2.0 (authorization) + an identity
layer (authentication)".

---

## Q3. Where should I start with protocols — OAuth, OIDC, or SAML?

**A.** Start with **OAuth 2.0**. It's the foundation that everything else
builds on (OIDC sits on top of it; even some modern SAML integrations look
OAuth-ish). SAML 2.0 is older, XML-heavy, and still everywhere in the
enterprise SaaS world — but you'll understand it much faster once you know
OAuth + OIDC.

**The one OAuth resource worth reading**: `oauth.com` by Aaron Parecki. Free
online book, ~100 pages, clearest explanation on the internet. Minimum
required reading:

- "The Protocol Flow"
- "Authorization Code Grant"
- "Access Tokens"
- "Refresh Tokens"
- "PKCE"

Concepts to fully internalize before moving on:

- **Four roles**: resource owner (user), client (app), authorization server
  (Keycloak), resource server (API).
- **Authorization Code flow + PKCE** — the default for web apps and SPAs.
  90% of what you'll ever configure.
- **Client credentials flow** — machine-to-machine; no user involved.
- **Device code flow** — for devices without a browser (e.g., a TV or CLI).
- **Scopes** — what access the user is consenting to.
- **Why implicit flow is deprecated** — PKCE solves the problems it tried to.
- **Refresh tokens** — how to stay logged in without storing passwords.

Rough time: 2 hours.

---

## Q4. What does OpenID Connect (OIDC) add on top of OAuth 2.0?

**A.** OAuth 2.0 by itself only handles *authorization* — it gets you an
opaque access token that grants access to some resource. It does NOT tell
the client who the user is.

OIDC adds an identity layer:

- **ID token** — a JWT (signed, self-contained) asserting who the user is.
  The client reads the claims directly (`sub`, `email`, `preferred_username`,
  etc.) without needing to call another endpoint.
- **`/userinfo` endpoint** — get additional user attributes.
- **`/.well-known/openid-configuration`** — the discovery document clients
  use to auto-configure. You saw this today when we debugged the issuer URL
  mismatch.

Resources: OpenID Foundation's "OIDC Core" spec overview, or Auth0's "Intro
to OpenID Connect" (easier read). Focus on:

- Difference between **access token** (bearer, sent to APIs) and **ID token**
  (identity proof, read by the client).
- **JWT structure** — header.payload.signature, base64url-encoded.
- **JWT validation** — signature check, `iss`, `aud`, `exp`, `nbf` claims.
- **`jwt.io`** — paste a real token and look inside. Nothing demystifies
  JWTs faster.

Rough time: 1 hour.

---

## Q5. How does SAML fit in, and do I need to learn it now?

**A.** SAML 2.0 is an older (2005) XML-based SSO protocol. It's dominant in
enterprise SaaS (Salesforce, ServiceNow, many Atlassian-era apps) but
painful to work with — XML, XML signatures, SP-initiated vs IdP-initiated
flows.

For a homelab platform focused on modern apps (ArgoCD, Vault, Harbor, Argo
Workflows, SonarQube), **OIDC is enough**. All of those speak OIDC natively.
Skip SAML until you actually need to federate with an enterprise SaaS that
forces it.

When you do learn it later: the mental model is similar to OIDC (IdP + SP,
assertions instead of tokens) but the wire format and signing rules are
wildly different.

---

## Q6. What are Keycloak's core abstractions I need to understand?

**A.** Read the **Keycloak Server Administration Guide** at `keycloak.org/guides`.
Focus on these chapters first:

- **Realms** — top-level tenant boundary. Users, clients, roles, and
  sessions are realm-scoped. `master` is for administering Keycloak itself;
  apps should use their own realm (which is why we're standing up `forge`).
  Users in one realm cannot see or authenticate into another.
- **Clients** — each application that authenticates users or consumes tokens.
  Client types:
  - **Public** — browser-delivered code (SPA, mobile). No client secret.
    Uses PKCE.
  - **Confidential** — server-side app that can safely hold a secret.
  - **Bearer-only** — APIs that only validate incoming tokens and never
    initiate login.
- **Users, roles, groups** — authorization primitives.
  - Roles: realm-level or client-level.
  - Groups: containers of users that can have roles attached.
  - Composite roles: roles that include other roles.
- **Protocol mappers** — transformations from user/session data into JWT
  claims. E.g., "put the user's group membership into a `groups` claim" so
  your app can read it.
- **Authentication flows** — the ordered steps Keycloak runs during login
  (username → password → optional OTP → conditional MFA). Fully
  configurable per realm.
- **Identity providers (federation)** — Keycloak delegating authentication
  to Google, GitHub, LDAP, AD, or another OIDC/SAML IdP. "Login with
  Google" is Keycloak talking OIDC to Google on your behalf.
- **Client scopes** — reusable bundles of protocol mappers + role mappings
  assigned to clients.

Rough time: 1 hour.

---

## Q7. What hands-on exercises will make it stick?

**A.** Don't just read. On the existing mgmt-forge Keycloak:

1. **Create a playground realm and test user.**
   - Realm: `playground`
   - User: `alice` with a known password
   - No impact on any real app; can be deleted freely.

2. **Drive Authorization Code flow by hand.**
   - Create a **public** client in `playground` with redirect URI
     `http://localhost:8080/callback`.
   - Open the auth URL in a browser:
     `https://keycloak.apps.mgmt-forge.engatwork.com/auth/realms/playground/protocol/openid-connect/auth?client_id=...&response_type=code&redirect_uri=http://localhost:8080/callback&scope=openid&code_challenge=...&code_challenge_method=S256`
   - Log in as alice. Browser redirects to `localhost:8080/callback?code=...` —
     copy the code out of the URL bar (no server needed).
   - Exchange the code for tokens via `curl` to the `/token` endpoint.
   - Paste the resulting `id_token` into `jwt.io` and inspect claims.

3. **Client credentials flow.**
   - Create a **confidential** client (service account enabled).
   - `curl -d "grant_type=client_credentials&client_id=...&client_secret=..."`
     to the `/token` endpoint.
   - No user involved — this is how a backend service authenticates itself.
   - Inspect the access token; note `sub` is the service account, not a user.

4. **Decode tokens obsessively.** Every time you touch an OIDC interaction,
   paste the tokens into `jwt.io` and read the claims. Over a few sessions
   you'll stop thinking of tokens as opaque.

5. **Break things deliberately.**
   - Set redirect URI to the wrong value — see the error Keycloak returns.
   - Let a token expire — observe the behavior.
   - Revoke a session from the admin console — see what happens on next
     API call.

Today (during Keycloak bring-up) you already did one of these inadvertently:
the `curl -X POST grant_type=password` that fetched a JWT for `admin` was an
OAuth 2.0 password grant against the master realm. Password grant is
deprecated but illustrative.

Rough time: 1-2 hours to complete all five.

---

## Q8. Why is this ordering — problem space → OAuth → OIDC → Keycloak — better than jumping straight to Keycloak?

**A.** The Keycloak admin UI is a thin wrapper over OIDC, SAML, and OAuth
2.0 concepts. Without the protocol layer underneath, every checkbox is
magic: "what's a 'client scope'?", "what's 'Direct Access Grants Enabled'?",
"what does 'implicit flow' mean?"

With the protocol layer, the UI becomes a direct mapping: pick your flow,
declare redirects, pick your claims. Configuration that seemed arbitrary
starts reading as "this exposes OIDC parameter X".

Trying to learn Keycloak without OAuth/OIDC is like reading nginx docs
without knowing HTTP. You can copy-paste your way to something that works
but you can't debug it and you can't design for it.

---

## Q9. After the basics — what are the next layers to study?

**A.** In rough priority, for what this platform will eventually need:

1. **Token validation in consuming apps** — every app that accepts tokens
   has to verify the signature using Keycloak's public keys from the JWKS
   endpoint, check `iss`, `aud`, `exp`, `nbf`. Libraries do this but
   understanding what they do matters when something breaks.
2. **Front-channel vs back-channel logout** — how SSO logout propagates
   across apps. Relevant when multiple apps share a Keycloak realm.
3. **Keycloak Authorization Services** — fine-grained resource + permission
   model inside Keycloak (UMA 2.0). Probably skip until needed; for most
   platforms, role-based access on the client side is enough.
4. **mTLS, token binding, DPoP** — proof-of-possession for access tokens.
   Advanced; needed for sensitive APIs.
5. **Admin REST API + keycloak-config-cli** — how we'll manage realms
   declaratively from Git. Once you understand the concepts, reading the
   admin API is mostly "match concept X to endpoint Y".
6. **SAML** — when you need to federate with an enterprise IdP that
   insists on it.

---

## Q10. Glossary — terms that will show up everywhere.

- **IdP** — Identity Provider. The thing that authenticates users. Keycloak
  in our stack.
- **SP** — Service Provider (SAML term). The app that trusts the IdP.
- **RP** — Relying Party (OIDC term). Same idea as SP.
- **Client** — OIDC/OAuth term for the app that asks for tokens. Can be an
  RP and/or a consumer of access tokens.
- **Bearer token** — a token that whoever holds it can use. Possession =
  authority. Hence: don't leak them.
- **JWT** — JSON Web Token. Self-contained, signed. Used for ID tokens,
  sometimes access tokens, API keys, etc.
- **JWKS** — JSON Web Key Set. Public keys published at
  `/.well-known/jwks.json` used to verify JWT signatures.
- **Claims** — fields inside a JWT (`sub`, `iss`, `aud`, `exp`, custom ones).
- **Scope** — OAuth concept: a named permission the client is asking for.
- **Consent** — user-visible screen where they approve the scopes a client
  is requesting.
- **Federation** — IdP delegating login to another IdP.
- **Realm** — Keycloak-specific: tenant boundary.

---

## Resources cheat sheet

- **oauth.com** — Aaron Parecki's free book on OAuth 2.0.
- **keycloak.org/guides** — official Keycloak docs (surprisingly good).
- **openid.net/developers/how-connect-works** — OIDC primer.
- **jwt.io** — decode/inspect JWTs interactively.
- **jose.readthedocs.io** — deeper dive into JWT signing/encryption.
- Auth0 blog and Okta developer blog — both have high-quality explainers
  on specific concepts (search by topic).

---

## Q11. When wiring an app (e.g. ArgoCD) to Keycloak as an OIDC client, the client secret has to land in two places: inside Keycloak's client config AND as a k8s Secret the consuming app references. What's the clean way to automate that?

**A.** Short answer: the clean answer is **"both sides read the secret from a single source of truth"** — typically HashiCorp Vault, consumed via External Secrets Operator (ESO) on each side. The manual `kubectl create secret` approach is fine for a demo but creates drift (two copies, no rotation story).

Five patterns teams actually use, in order of how often you see them:

### Pattern 1 — keycloak-config-cli + Vault + ESO (most common in mature k8s orgs)

```
  Vault: kv/forge/oidc/argocd-sre/clientSecret = <random 32-byte hex>
         │                                       │
   ExternalSecret                         ExternalSecret
   (mgmt-forge, keycloak ns)              (OKD, argocd-sre ns)
         │                                       │
         ▼                                       ▼
   k8s Secret mounted as                 k8s Secret referenced by
   env into keycloak-config-cli          ArgoCD CR's oidcConfig via
   Job (imports realm YAML with          `$argocd-sre-oidc-secret:clientSecret`
   `clientSecret: "$(env:...)"`)         syntax
```

Realm config lives in Git at `forge/keycloak/realms/forge.yaml` (declarative
YAML or JSON — the "infra as code" for SSO). The client secret is a
placeholder `$(env:SECRET_NAME)` that keycloak-config-cli substitutes at
import time from mounted env vars. Same Vault secret flows to both sides.

Rotation: `vault kv put kv/forge/oidc/argocd-sre clientSecret=<new>`. Both
ESO instances refresh within their `refreshInterval` (usually 1h). Both
sides converge.

### Pattern 2 — Keycloak Operator (CR-driven)

Use the Keycloak Operator (separate from Helm chart). It provides
`KeycloakRealm`, `KeycloakClient`, `KeycloakUser` CRDs. The operator
auto-creates a k8s Secret named `keycloak-client-secret-<clientId>`
holding the generated secret. Consumers reference that Secret.

Trade-off: clean in-cluster, but cross-cluster (ArgoCD on OKD consuming
a Secret on mgmt-forge) still needs bridging (Replicator, ESO-via-Vault,
etc). And you're fully committed to the operator ecosystem.

### Pattern 3 — Terraform + kubernetes provider

```hcl
resource "keycloak_openid_client" "argocd_sre" { ... }
resource "kubernetes_secret" "argocd_sre_oidc" {
  data = { clientSecret = keycloak_openid_client.argocd_sre.client_secret }
}
```

Terraform state holds the secret; one `apply` creates Keycloak client
AND k8s Secret atomically. Common in infra-team-runs-TF orgs. Rotation
= tainted resource + reapply.

### Pattern 4 — Crossplane

Same shape as Terraform but as k8s CRs managed by Crossplane controllers.
"Kubernetes-native IaC." Providers for Keycloak + Kubernetes both exist
but the Keycloak provider isn't as mature as Terraform's.

### Pattern 5 — Custom bootstrap Job

Post-install Job that uses Keycloak admin API → creates client → writes
Secret to both sides. Works, but loses the declarative "reviewable in
Git" property of the first four.

---

### Recommendation for THIS stack

**Pattern 1** — we already have Vault + ESO on mgmt-forge. Adding a
keycloak-config-cli sidecar Job under `forge/keycloak/` is a bounded
piece of work. Concrete design:

```
forge/keycloak/
├── Chart.yaml                  (already exists)
├── values.yaml                 (already exists)
├── templates/
│   ├── admin-externalsecret.yaml       (already exists — admin creds from Vault)
│   ├── db-externalsecret.yaml          (already exists)
│   ├── argocd-oidc-externalsecret.yaml (NEW — pulls client secret from Vault
│   │                                     into a Secret in keycloak ns, mounted
│   │                                     into the config-cli Job as env)
│   └── keycloak-config-cli-job.yaml    (NEW — runs on Helm hook post-sync;
│                                         imports realm YAML from configmap
│                                         below, substituting env vars)
└── realms/
    └── forge.yaml                      (NEW — declarative realm: clients,
                                          users, roles, protocol mappers;
                                          client secrets are $(env:...))
```

And on the OKD side:
```
argo-applications/sre/forge/argocd-sre-oidc-externalsecret.yaml   (NEW — materializes
                                                      the same Vault path
                                                      into argocd-sre ns)
```

Plus a one-time bootstrap:
```
vault kv put kv/forge/oidc/argocd-sre clientSecret=$(openssl rand -hex 16)
```

### Cross-cluster trust — the asymmetry to watch

ArgoCD runs on OKD; Vault runs on mgmt-forge. For ESO on OKD to read from
mgmt-forge's Vault over HTTPS:

- Vault's ingress is already reachable from OKD (pfSense routes between
  the VLANs).
- Vault needs an **auth method** OKD can use. Options:
  - **Kubernetes auth**: Vault trusts JWTs issued by OKD's API server.
    Requires configuring Vault with OKD's JWKS URL + CA + a role tying
    a specific SA to a policy. One-time Vault config.
  - **AppRole auth**: OKD bootstraps a role_id + secret_id (stored as a
    k8s Secret on OKD), ESO uses those to authenticate. Simpler but the
    initial `secret_id` is itself a bootstrap secret.

Either way, this is a one-time Vault configuration step, done by hand
or a bootstrap playbook — part of Phase 5 of the estate build.

### Homelab-acceptable shortcut

For now we use the manual flow (kubectl create secret + referenced in
ArgoCD CR). It proves OIDC works end-to-end. Pattern 1 is the migration
target, separately tracked.

> **Update — what we actually shipped.** The implementation diverged from
> Pattern 1. See **"Current setup — engatwork SSO architecture"** below. We
> used a hybrid: realm-import ConfigMap (one-shot declarative) + post-sync
> Job harvesting auto-generated secrets to Vault (closer to Pattern 5) +
> Crossplane provider-vault for the Vault config itself (Pattern 4 for the
> *infrastructure* policies/roles, not the client secrets). Trade-offs and
> rationale below.

### Open items to revisit when migrating

- Where does the bootstrap cephx-style "generate the initial Vault
  secrets once" step live? Likely a standalone playbook invoked during
  Phase 5 of the estate build, analogous to Vault init + unseal.
- Keycloak-config-cli handles realm imports; does it handle **updates**
  idempotently? What about deletions? Currently it supports
  `IMPORT_FORCE=true` which overwrites unmanaged changes — good for
  GitOps (drift correction), painful for anyone who edited the realm
  via UI expecting their changes to stick.
- How do you rotate the client secret without downtime? Answer: Vault's
  KV v2 keeps old versions; ESO refreshes to latest; ArgoCD picks up
  the new Secret on next pod restart; Keycloak accepts both old + new
  for a grace window (the realm import rotates in a new credential but
  keeps old active for N minutes). Details depend on keycloak-config-cli
  version.

---

# Current setup — engatwork SSO architecture

What was actually shipped, why it diverged from Pattern 1, and how a
request flows through it. README.md covers ops (table of clients, runbook
commands); this section covers the *why*.

## Q12. What pattern did we end up shipping, and why not Pattern 1 (keycloak-config-cli)?

**A.** A hybrid, captured here:

| Concern | Pattern 1 (recommended in Q11) | What we shipped |
|---|---|---|
| Realm + clients declared in Git | YAML processed by keycloak-config-cli | JSON inside a ConfigMap, mounted at `/opt/keycloak/data/import`, read on Keycloak startup via `--import-realm` |
| Client secrets generated | keycloak-config-cli substitutes `$(env:VAR)` from a Vault-backed Secret | Keycloak auto-generates on import; a PostSync Job harvests via REST API into Vault |
| Vault policies + auth-backend roles | Imperative `vault policy write …` | Declarative Crossplane CRs (`Policy`, `AuthBackendRole`) reconciled by provider-vault |
| Drift between Git and live realm | keycloak-config-cli forces re-import (IMPORT_FORCE) | IGNORE_EXISTING — UI edits survive; Git changes don't propagate to a live realm |

**Why not pure Pattern 1.** The keycloak-config-cli sidecar adds a Java
runtime, an extra image, and a templating language we'd otherwise skip.
For a homelab with four OIDC clients evolving slowly, the realm-import
ConfigMap is simpler — one boot, one import. The trade-off is the
IGNORE_EXISTING gotcha (see Q15).

**Why a Job for secret harvest, not config-cli's `$(env:VAR)`.** Avoids
needing the secret in Vault *before* Keycloak boots — Vault was empty for
these paths until the Job ran the first time. Removes the pre-population
step from the bootstrap dance.

**Why Crossplane for Vault policies.** Vault config (policies + auth roles)
used to be a separate Ansible playbook (`vault/playbooks/configure.yml`).
Bringing it under Crossplane means *adding a new policy is a values-file
edit* — same workflow as everything else. The single chicken-and-egg —
minting the Crossplane admin token in Vault — is documented in
`control/crossplane-providers/values.yaml` and only happens once per
Vault server.

---

## Q13. Walk through the boot sequence — what depends on what?

**A.** ArgoCD sync waves enforce ordering. Wave 0 is the foundation, each
subsequent wave assumes prior waves are healthy.

```
wave 0  cert-manager, ingress-nginx, external-secrets CRDs, crossplane core
wave 1  external-secrets controller, vault server, ClusterSecretStore vault-forge
wave 2  keycloak (realm-import on first boot)
        crossplane-providers (Provider/ProviderConfig for vault)
PostSync keycloak-client-secret-sync Job runs after keycloak StatefulSet healthy
        → harvests apps-launcher / backstage / argocd-sre / argocd-devops secrets to Vault
        → seeds oauth2-proxy cookie-secret if absent
wave 3  oauth2-proxy (reads Secret/oauth2-proxy-oidc via ESO from Vault)
        argocd-sre, argocd-devops (read Secret/argocd-oidc via ESO)
        backstage (Secret/backstage-oidc rendered, env wiring still commented
        until custom image lands)
wave 4  vault-config CRs apply (Policy + AuthBackendRole — provider-vault
        translates to Vault HTTP API calls)
        homer (gated by oauth2-proxy via nginx auth-url subrequest)
```

**Cross-cluster note.** Keycloak + Vault run on **mgmt-forge**.
oauth2-proxy / Homer / argocd-* / Backstage / Crossplane all run on
**mgmt-control**. The two clusters are stitched together by:
1. ESO's `ClusterSecretStore vault-forge` on each cluster, hitting Vault's
   public ingress URL (`https://vault.apps.mgmt-forge.engatwork.com`).
2. CoreDNS on mgmt-core resolving the wildcard `*.apps.<cluster>.engatwork.com`
   to each cluster's ingress VIP.

---

## Q14. Trace one user login end-to-end (Homer + ArgoCD).

**A.** Two flows; both terminate at the same Keycloak realm.

### A. Apps launcher (Homer behind oauth2-proxy)

```
1. Browser → GET https://home.apps.mgmt-control.engatwork.com/
2. ingress-nginx fires auth-subrequest to /oauth2/auth
   → oauth2-proxy returns 401 (no session cookie)
3. ingress-nginx redirects browser to /oauth2/start?rd=/...
4. oauth2-proxy → 302 to
     https://keycloak.apps.mgmt-forge.../auth/realms/engatwork/protocol/openid-connect/auth
     ?client_id=apps-launcher&redirect_uri=.../oauth2/callback&...
5. Keycloak presents login (or skips if SSO cookie present)
6. User authenticates → 302 back to /oauth2/callback?code=…
7. oauth2-proxy exchanges code for tokens via the apps-launcher client_secret
   (read from Secret/oauth2-proxy-oidc, sourced from Vault)
8. oauth2-proxy sets _oauth2_proxy session cookie, redirects to original /
9. Browser → GET / again. ingress-nginx auth-subrequest → 202 (cookie valid)
   → request reaches Homer pod, tile page renders.
```

### B. ArgoCD-SRE (direct OIDC, no oauth2-proxy)

```
1. Browser → GET https://argocd-sre.apps.mgmt-control.engatwork.com/login
2. User clicks "LOGIN VIA KEYCLOAK"
3. argocd-server → 302 to Keycloak with client_id=argocd-sre
4. Keycloak — already SSO'd (cookie from step 5 above) → silent success
5. 302 back to /auth/callback?code=…
6. argocd-server fetches OIDC config from issuer's well-known endpoint,
   validates code, reads ID token. clientSecret pulled from
   Secret/argocd-oidc (rendered by ESO from kv/forge/keycloak/clients/argocd-sre).
7. ID token contains `groups` claim — Keycloak's `groups` clientScope put
   it there via the oidc-group-membership-mapper.
8. argocd-server matches groups against policy.csv:
     g, platform-sre, role:admin
   User is admin → full UI.
```

The "single login" experience is real because steps 4–5 in flow B reuse
the Keycloak session cookie set during flow A. Both clients live in the
same realm, so the SSO session crosses applications.

---

## Q15. The realm-import IGNORE_EXISTING gotcha — what is it, and how do we work around it?

**A.** Keycloak's `--import-realm` flag has three strategies (controlled by
`KC_SPI_IMPORT_SINGLE_FILE_STRATEGY`):

- **IGNORE_EXISTING** (default) — if the realm already exists, do nothing.
- **OVERWRITE_EXISTING** — re-import every boot, clobbering UI edits.
- **NO_IMPORT** — never import.

We run with default IGNORE_EXISTING. Consequence: editing
`templates/realm-import-cm.yaml` (e.g., adding a new client) does *not*
update a running realm. The new client only appears if the realm is
re-created.

**Why we accept this.** Most Keycloak operators want UI edits to stick —
imagine fine-tuning a brute-force-detection setting in the UI and having
it overwritten on the next pod restart. IGNORE_EXISTING is the "safe"
default the chart maintainers picked.

**Workarounds for adding a new client to a live realm:**

1. **Manual UI creation** matching the JSON spec — tedious but correct.
2. **Delete the realm** in admin UI → next pod restart re-imports — only
   safe on a fresh install (loses users, sessions, tokens).
3. **One-off OVERWRITE_EXISTING boot** — set the env var, restart, revert.
   Risky.
4. **Future improvement (open work):** extend `client-secret-sync-job.yaml`
   into a client-*upsert* Job. Already has the admin token and REST
   access; adding "POST the client config if it doesn't exist" is a few
   lines of bash. Would make new client provisioning truly git-push-only.

Tracked under todo.md. Not a blocker today because the four currently
defined clients cover the immediate scope.

---

## Q16. What does the vault-config chart actually do?

**A.** It declares the *Vault server's* policies and auth-backend roles
as Kubernetes CRs, reconciled by Crossplane's provider-vault.

Files:

```
security/secrets/vault-config/
├── values.yaml                          ← maps of policies + roles
└── templates/
    ├── policies.yaml                    ← renders Policy CRs
    └── kubernetes-auth-roles.yaml       ← renders AuthBackendRole CRs
```

`values.yaml` is the source of truth — adding a new Vault policy is one
entry in `policies:` + `git push`. Crossplane's provider-vault pod (on
mgmt-control, in `crossplane-system`) sees the new CR, calls the Vault
API using its admin token, creates the policy. From there everything
downstream just works.

**Why the chart deploys to mgmt-control, not mgmt-forge.** The Crossplane
controller — which actually calls the Vault API — runs on mgmt-control.
The Crossplane CRs (the *managed resources*) only have meaning where the
controller can see them. So the chart targets `role: control` even
though, conceptually, it's configuring something on mgmt-forge. This is
a recurring pattern with Crossplane: the *target of management* and the
*location of the managed resource CR* are decoupled.

**Bootstrap sequence.** The Crossplane → Vault bridge needs one
manually-minted Vault token with the `crossplane-admin` policy, written
to `kv/forge/crossplane/vault-token`. That's the only manual Vault step
left — see `control/crossplane-providers/values.yaml` header. After that
token exists, every other Vault role/policy is reconciled by Crossplane
on git push.

---

## Q17. What's NOT done yet?

**A.**

- **Backstage SSO is scaffolded but not active.** The `backstage`
  client + ExternalSecret + values.yaml block exist, but the demo image
  (`ghcr.io/backstage/backstage:latest`) doesn't bundle the OIDC plugin.
  Activating requires bootstrapping a Backstage TS app source in a sibling
  repo, adding `@backstage/plugin-auth-backend-module-oidc-provider`,
  building/pushing a custom image, then flipping the chart values.
- **Vault UI not behind SSO.** Vault has its own auth (token / userpass /
  OIDC). Could be added as a 5th client in the realm with `vault auth enable
  oidc` configured against Keycloak.
- **Grafana / Prometheus / Alertmanager not behind SSO.** They sit at
  *.apps.mgmt-observability.engatwork.com today with chart-default auth.
  Each is a one-client-per-app addition once the pattern is comfortable.
- **forge-reader Vault policy not yet under Crossplane management.** The
  existing policy (used by ESO across all clusters) is still provisioned
  via `vault/playbooks/configure.yml`. Bringing it under vault-config is
  one entry in `values.yaml` — just needs an unmanaged → managed import.
- **Realm-import client upsert.** See Q15. Currently adding a new client
  to a live realm is manual.

Each of these is a one-session piece of work. The *infrastructure* (realm,
Vault paths, ESO ClusterSecretStore, Crossplane provider) is in place;
new consumers just plug in.
