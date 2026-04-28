# identity — current setup

## What lives here

The **identity plane** holds the OIDC IdP (Keycloak) and federation tooling (Dex). Both end-users (people logging into ArgoCD/Grafana) and machine consumers (apps requesting tokens) authenticate against this plane.

AppSets that drive deployment: `../argo-applications/sre/identity/`.

## Services

| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| **keycloak** | ✅ live (mgmt-forge); planned migration to OKD | mgmt-forge today | B today; control-plane future | Realms: forge, dev-* (per environment) |
| **dex** | 📋 placeholder | OKD | one-off | Lightweight federation; can sit in front of GitHub/Google as upstream IdP |

## Dependencies

- **Upstream**: cert-manager (`security/ca-pki/`) for OIDC issuer URL TLS; Vault (`security/secrets/`) for admin password + client secrets via ESO.
- **Downstream**: every service that authenticates users — ArgoCD UI, Grafana, Harbor, SonarQube, Backstage, etc.

## Architectural decisions

- **Keycloak migration to OKD** is planned because identity is durable / control-plane-shaped. Today it lives on mgmt-forge for historical reasons (it was set up before the planes split was clear). Migration involves: import realm via keycloak-config-cli, update OIDC client URLs, drop the mgmt-forge instance.
- **Dex as alternative or fallback**: lighter-weight than Keycloak. Useful if you want federation without a full IAM server, or as a low-overhead fallback during Keycloak maintenance.

## Known issues

- Keycloak's OIDC discovery URL must be reachable from every consumer. Until cert-manager + Vault PKI lands, the cert is self-signed; consumers need their CA bundle injected (ESO does this from Vault's CA secret).
