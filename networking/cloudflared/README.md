# networking/cloudflared

🟢 **Chart authored 2026-08-27** — `Chart.yaml` · `values.yaml` · `templates/`
(`configmap.yaml` = ingress rules + CA bundle, `deployment.yaml`, `service.yaml` = metrics,
`tunnel-externalsecret.yaml`). AppSet: `argo-applications/sre/networking/cloudflared-appset.yaml`
(Pattern B, `role=control`, sync-wave 5). **Not live yet** — needs the tunnel created,
its decoded credentials in Vault, and `tunnel.id` set in `values.yaml` (§4).

This file is the design + rollout record for public internet access to the homelab.

Goal: reach selected homelab services from the public internet **without**
inbound port-forwards on pfSense, without a public-IP / DDNS dependency, and
without widening the hypervisor's attack surface.

**Approach: Cloudflare Tunnel** (decided 2026-08-27 — see §2 for why, and
§1 for the port-forward approach it replaces).

---

## 1. Current state — what's been done for public DNS access

### Iteration 1 — Cloudflare proxy + pfSense port-forward (SUPERSEDED)

The first attempt fronted the existing `ingress-nginx` with Cloudflare's
proxy (orange-cloud DNS) and a WAN port-forward:

| Piece | State | Notes |
|---|---|---|
| Cloudflare account + zone | ✅ done | Zone: **`engatwork.com`** — same name as the internal authoritative zone, so this is **split-horizon** (public resolvers → Cloudflare; LAN → CoreDNS via pfSense forward) |
| Proxied DNS record → WAN IP `192.168.0.101` | ✅ done | `A` / proxied (orange cloud). Hostname: `⚠️CONFIRM which` |
| Cloudflare SSL/TLS mode | ✅ set to **Full** | edge↔origin encrypted, origin cert (engatwork CA) not validated |
| pfSense NAT port-forward WAN → ingress VIP | ✅ **verified none** (2026-08-27, SSH `config.inc` dump) — `nat/rule` count = 0, no 1:1 NAT | Iteration-1 port-forward was discussed but **never created**. Nothing to remove. |
| Service exposed | Homer + (intended) everything Homer links to | Homer already behind oauth2-proxy/Keycloak |

**But** WAN filter rules are wide open: 4× `Passed via EasyRule`
(`any → any`, one scoped to port 53) + `WAN_EasyRule_Pass` (`any → any`).
With no port-forward and (presumably) no ISP-router forward to
`192.168.0.101`, nothing from the internet currently lands on an internal
host — but pfSense's own WebGUI (`:443`) and SSH (`:22`) are reachable
from the WAN side, and the moment the upstream router DMZs/forwards to
pfSense the whole `10.10.0.0/16` is exposed. **Clean these up** (see §4
pfSense checklist) — the tunnel needs none of them.

Split-horizon is workable **because the hostnames match**: Homer already
uses `*.apps.mgmt-*.engatwork.com`, so the public `CNAME`s the tunnel
creates carry the *same* names the LAN already resolves to the ingress
VIPs — no Homer edits, no new zone. Cost: every public name is a manual
entry you must not let collide with an internal-only one, and never
enable Cloudflare's bulk DNS import on this zone.

**Why the port-forward approach is being dropped:**

- Requires an open inbound port on WAN — directly internet-reachable
  origin, scanned/brute-forced within minutes, origin 0-days hit home.
- `192.168.0.101` is itself behind an upstream ISP router (double-NAT) —
  the forward also depends on the ISP box and breaks if the ISP moves to
  CGNAT or changes the WAN IP.
- `Full` (non-strict) doesn't pin the origin cert — a MITM on the
  edge↔origin path could impersonate the origin.
- Public certs / DDNS become ongoing maintenance.

### Iteration 2 — Cloudflare Tunnel (IN PROGRESS)

`cloudflared` as an outbound-only Deployment in-cluster. No WAN ports, no
DDNS, no CGNAT problem, edge-managed public certs, identity at the edge
via Cloudflare Access. Rollout checklist in §4.

---

## 2. How Cloudflare Tunnel works (foundations)

```
  Browser ──TLS──► Cloudflare edge ──QUIC/HTTP2 (outbound-only)──► cloudflared pod ──► origin service
             (1)          (2)                    (3)                       (4)
```

1. **Public DNS** for `foo.example.com` is a proxied `CNAME` to
   `<TUNNEL-UUID>.cfargotunnel.com`. Only Cloudflare edge IPs are ever
   visible on the internet — your WAN IP is never published.
2. **Edge terminates TLS** with a Cloudflare-managed certificate. The
   browser trusts it via a public CA; your internal `engatwork` CA never
   needs to be publicly trusted.
3. `cloudflared` **dials out** to the edge (443/7844 outbound) and holds
   the connection open. Nothing listens for inbound — this is why it
   works behind CGNAT / double-NAT and needs zero pfSense WAN rules.
   Auth is a **tunnel token** (or `credentials.json`).
4. `cloudflared` matches the request `Host` against its **ingress rules**
   and proxies to the mapped origin URL. It can verify the origin's TLS
   against a supplied CA bundle (`originRequest.caPool`) so the last hop
   stays authenticated — set this to the `engatwork` CA and you get
   Full-strict-equivalent end to end.

**Cloudflare Access** (recommended for every hostname): the edge refuses
to forward until the user passes an identity check (Google login, etc.),
then mints a short-lived JWT (`Cf-Access-Jwt-Assertion`). Auth sits *in
front of* the app.

**Cloudflare WARP + private routes**: instead of a public hostname,
`cloudflared` advertises a CIDR (`10.10.0.0/16`) as a route; the WARP
client then reaches `10.10.x.x` / internal hostnames as if on-LAN. No
public DNS, no public login page. This is the tool for admin planes.

---

## 3. What to expose, and how

Requested: **Homer + every service Homer links to.** Those services (from
`cplanes/control/homer/values.yaml`) split into two risk tiers. **All**
hostnames sit behind Cloudflare Access (Google `sriram.gc432@gmail.com`, that
identity only); the tiers differ in whether a public DNS name exists at
all.

### Tier 1 — public hostname + Cloudflare Access

Meant to be used off-LAN, and a stolen Access session ≠ infrastructure
takeover.

| Homer tile | Origin VIP (cluster) | `httpHostHeader` |
|---|---|---|
| Homer | `10.10.2.200` (mgmt-control) | `home.apps.mgmt-control.engatwork.com` |
| Argo Workflows | `10.10.2.200` | `workflows.apps.mgmt-control.engatwork.com` |
| Backstage | `10.10.2.200` | `backstage.apps.mgmt-control.engatwork.com` |
| Jenkins | `10.10.2.200` | `jenkins.apps.mgmt-control.engatwork.com` |
| Grafana | `10.10.3.201` (mgmt-observability) | `grafana.apps.mgmt-observability.engatwork.com` |
| SonarQube | `10.10.5.201` (mgmt-forge) | `sonarqube.apps.mgmt-forge.engatwork.com` |

### Tier 2 — WARP private route only (no public DNS)

Infrastructure control planes. A compromise here is game-over; there is no
reason for these to be resolvable on the public internet even behind
Access. Reachable at their **normal URLs** once you're on WARP.

| Homer tile | Why WARP-only |
|---|---|
| ArgoCD — SRE / DevOps | GitOps control plane, `cluster-admin` blast radius |
| Vault | every platform secret + the PKI signing engine |
| Keycloak | IdP; `/admin` = mint tokens for anything. (If a *public* app ever needs OIDC, expose **only** `/realms/**` + `/resources/**` as a Tier-1 host — never `/admin/**`.) |
| Prometheus / Alertmanager | unauth'd query API, internal topology disclosure |
| pfSense | edge firewall admin — full network control |
| oVirt Engine | hypervisor manager — see §5 |
| k8s API servers (`:6443`) | not a Homer tile, but same tier — TCP route |

> **Recommendation:** keep this split. If you specifically want a Tier-2
> service as a public hostname anyway, it must additionally carry a
> `Require → WARP` rule in its Access policy (device-gated, not just
> identity-gated). Say which and I'll move it.

### k8s routing — one rule per cluster, let nginx do the rest

`cloudflared` runs in one cluster (mgmt-control); VLANs are inter-routed
via pfSense so it can reach every cluster's ingress VIP. Point each
hostname at the relevant `ingress-nginx-controller` VIP and preserve the
`Host` header (the hostname and header are identical here — split-horizon
zone — but keep `httpHostHeader` explicit so intent is clear):

```yaml
ingress:
  - hostname: home.apps.mgmt-control.engatwork.com
    service: https://10.10.2.200
    originRequest:
      httpHostHeader: home.apps.mgmt-control.engatwork.com
      caPool: /etc/cloudflared/engatwork-ca.crt
  - hostname: grafana.apps.mgmt-observability.engatwork.com
    service: https://10.10.3.201
    originRequest:
      httpHostHeader: grafana.apps.mgmt-observability.engatwork.com
      caPool: /etc/cloudflared/engatwork-ca.crt
  # ... one block per Tier-1 host ...
  - service: http_status:404
```

pfSense keeps its `pfsense.engatwork.com` tile pointing at the LAN GUI;
it is never given a tunnel ingress rule.

---

## 4. TODO — tunnel rollout

### Decisions — locked 2026-08-27

- ✅ **Zone**: `engatwork.com` on Cloudflare — **split-horizon** accepted.
      Guardrails: only create the public records listed below; never
      enable Cloudflare bulk DNS import on this zone; when adding an
      internal-only host later, don't reuse a name that exists publicly.
- ✅ **Public hostnames (Tier 1)**: `home`, `workflows`, `backstage`,
      `jenkins` (`.apps.mgmt-control`), `grafana` (`.apps.mgmt-observability`),
      `sonarqube` (`.apps.mgmt-forge`). Everything else Homer links to is
      **Tier 2 — WARP-only** (§3).
- ✅ **Access identity**: Google `sriram.gc432@gmail.com`, that identity only,
      on every Tier-1 hostname.
- [ ] Open: confirm nobody wants a Tier-2 service promoted to a
      `Require-WARP` public hostname.

### pfSense (owner: you — see `ovirt-setup/.../pfsense/README.md` §Inbound)

- ✅ NAT port-forwards: **none exist** (verified 2026-08-27). The tunnel
      needs zero inbound WAN rules — nothing to undo here.
- ✅ **Deleted the `any → any` WAN pass rules** (2026-08-27): 4×
      `Passed via EasyRule` + `WAN_EasyRule_Pass` removed via SSH
      (`filter/rule` 41 → 36). Config backed up to
      `/conf/config.xml.bak-pre-wan-cleanup-20260827-221622` on the box.
      Remaining WAN rules — all scoped to `192.168.0.0/24`:
      `Allow oVirt mgmt network to all internal networks`, `Allow_oVirt_Mgmt`,
      `WAN_OvirtMgmt_To_Internal`. Engine host → pfSense + → internet
      verified still working.
- [ ] **Remove `WAN_EasyRule_Pass` from `pfsense_firewall_rules`** in
      `ovirt-setup/.../pfsense/vars/pfsense_config.yml`, else the next
      `configure_pfsense.yml --tags rules` re-adds it. (The 4 `Passed via
      EasyRule` rules were never in Ansible — WebGUI cruft.)
- [ ] Restrict pfSense WebGUI + SSH to LAN / `192.168.0.0/24` only
      (System → Advanced → Admin Access), or add an explicit WAN block.
- [ ] Confirm outbound 443 from the mgmt-control VLAN to the internet is
      allowed (it is — `MGMT_Allow_Out` / floating pass-out).

### Cloudflare (owner: you — dashboard + `cloudflared` CLI)

- [x] Zero Trust → Networks → Tunnels → **Create tunnel** `homelab`
      (2026-08-28). ID `37c41133-1280-4e52-9b54-5fce9b365d1e`.
- [x] 6 Tier-1 **public hostnames** as proxied `CNAME` →
      `37c41133-1280-4e52-9b54-5fce9b365d1e.cfargotunnel.com` (2026-08-28).
      No Iteration-1 `A` records existed to delete.
- [x] **Advanced Certificate Manager** ($10/mo, 2026-08-28) — one advanced
      pack (Google Trust Services) covering `*.apps.mgmt-control` /
      `*.apps.mgmt-observability` / `*.apps.mgmt-forge` `.engatwork.com`.
      Free Universal SSL is `engatwork.com` + `*.engatwork.com` only, so the
      4-label tunnel names get no edge cert without this. See §6 (incl. the
      prefill-hosts gotcha).
- [ ] SSL/TLS mode → **Full (strict)** — safe now (cloudflared validates
      the origin via `caPool` + `originServerName`). Needs `Zone Settings`
      access / dashboard.
- [ ] (WARP) Networks → Routes → add `10.10.0.0/16` via `homelab`; enrol
      laptop/phone in the Zero Trust org. This is how Tier-2 (Vault,
      ArgoCD, Prometheus, pfSense, oVirt, k8s API) is reached.
- [ ] Access → Applications → one app per Tier-1 hostname, Google-only
      policy. Session duration ~24h.

### Repo (owner: me — checklist-driver)

- [x] `Chart.yaml` / `values.yaml` — self-contained Deployment (2 replicas,
      anti-affinity, outbound-only, 64—128Mi) + `configmap.yaml` (ingress rules
      from `.Values.ingress` + `engatwork-ca.crt` from `.Values.caBundle` — same
      PEM as `security/ca-pki` / `external-secrets`) + metrics `service.yaml`.
      **Locally-managed tunnel** — ingress rules live here, not the dashboard,
      so `originRequest.caPool` (Full-strict last hop) applies.
- [x] `templates/tunnel-externalsecret.yaml` — renders `Secret/cloudflared-tunnel`
      with a `credentials.json` file, from Vault `kv/data/forge/cloudflared/tunnel`
      via `ClusterSecretStore vault-forge`. Named `tunnel-externalsecret.yaml`
      (not `*-secret.yaml` / `*-creds.yaml`) so `.gitignore` keeps it
      (repo memory `feedback_gitignore_secrets`).
- [x] **Vault** — `kv/forge/cloudflared/tunnel` seeded 2026-08-28 (via the
      Vault UI: Secrets → kv → create `forge/cloudflared/tunnel` with keys
      `account_tag` / `tunnel_id` / `tunnel_secret`, the three fields of
      `echo <TOKEN> | base64 -d`). `tunnel.id` set in
      `networking/cloudflared/values.yaml` (commit `ccb1edf`).
      ESO renders `Secret/cloudflared-tunnel` → `credentials.json`.
- [x] `argo-applications/sre/networking/cloudflared-appset.yaml` — Pattern B
      (`role=control` → mgmt-control), sync-wave 5 (after ingress-nginx / ESO /
      oauth2-proxy / Homer), ESO `ignoreDifferences` like the oauth2-proxy AppSet.
- [ ] Add the row to `networking/README.md` (left untouched — it has unrelated
      uncommitted edits); drop the "not authored" note there.

### Verify

- [x] `kubectl -n cloudflared logs deploy/cloudflared` → `Registered
      tunnel connection` ×4 per replica (2026-08-28, 8 edge connections
      total; QUIC to ewr). Pods Ready via `/ready`.
- [x] Public path verified 2026-08-28 (curl `--resolve` to a CF edge IP):
      all 6 Tier-1 hosts return valid edge TLS + correct origin response
      (home → 302 `/oauth2/start`; workflows/backstage/sonarqube 200;
      jenkins 403 own-auth; grafana 302 `/login`).
- [ ] Phone on cellular → `https://home.<public-domain>` → Access login →
      Homer.
- [ ] From WARP → `https://ovirt.engatwork.com` loads; `ping 10.10.2.101`.
- [ ] No Access session, public internet → every hostname returns the
      Access login, never the origin app.
- [ ] pfSense → NAT → Port Forward is **empty**; `nmap` of the WAN IP from
      outside shows no open 80/443.

---

## 5. Why oVirt does **not** get a public hostname

`ovirt.engatwork.com` is the Hosted Engine — control plane for every VM in
the lab (create/destroy/console/snapshot), on an appliance image not
patched on a public-exposure cadence. A login-page compromise there is
game-over for the whole homelab.

- **Reach it over WARP** as a private-network route — full web UI at its
  normal URL, zero public surface.
- Same logic for Vault, ArgoCD, the k8s API servers, Keycloak `/admin`.
  One `10.10.0.0/16` route covers all of them.

---

## 6. Edge TLS coverage for deep hostnames

**Problem found 2026-08-28.** Cloudflare's free **Universal SSL** covers only
the zone apex and a **single-label** wildcard: `engatwork.com` and
`*.engatwork.com`. Every Tier-1 tunnel hostname is 4 labels deep
(`home.apps.mgmt-control.engatwork.com`), so the edge has **no certificate**
to present for it — the TLS handshake fails before HTTP, and the site is
unreachable from the public internet even though the tunnel itself is
healthy. (Internal access is unaffected: split-horizon DNS sends LAN clients
straight to the ingress VIP, which serves the `engatwork` origin cert.)

Options considered:

| | Cost | Effort | Verdict |
|---|---|---|---|
| **A. Advanced Certificate Manager** | $10/mo/zone | ~none | **chosen.** One advanced cert with the three `*.apps.mgmt-*` wildcards. No cluster / Homer / split-horizon changes. ACM works on the Free plan. |
| B. Flatten public names to `home.engatwork.com` etc. | free | high | cloudflared would rewrite Host back to the deep internal name, but Homer's oauth2-proxy `auth-signin` redirect points at the deep name — SSO breaks unless auth also moves to Cloudflare Access. Fights the split-horizon design. |
| C. Subdomain zones (`apps.mgmt-control.engatwork.com` as its own CF zone, NS-delegated) | free | medium | makes the tunnel names 1-label within their zone — Universal SSL then covers them. Costs 3 extra zones + NS delegation + keeping the internal CoreDNS authoritative zone consistent. |

### Decision — A (2026-08-28, done)

- ACM subscribed on `engatwork.com`.
- One **advanced certificate pack** (CA = **Google Trust Services**), hosts:
  - `*.apps.mgmt-control.engatwork.com`
  - `*.apps.mgmt-observability.engatwork.com`
  - `*.apps.mgmt-forge.engatwork.com`
- DNS (TXT) validation is automatic — the zone uses Cloudflare
  nameservers, so Cloudflare adds/removes the `_acme-challenge` records
  itself. Issued in ~5 min; auto-renews at ~30 days remaining.
- **Gotcha:** the "Order Advanced Certificate" dialog prefills hosts as
  `engatwork.com` + `*.engatwork.com`. Those must be **replaced** with the
  three deep wildcards — the first order kept the prefill and covered
  nothing new. A pack's `hosts` can't be edited after creation; delete and
  re-order (`POST .../ssl/certificate_packs/order`).
- Still open: **SSL/TLS mode → Full (strict)** (needs `Zone Settings`
  scope / dashboard). Safe now — cloudflared already validates the origin
  via `caPool` + `originServerName`.

---

## 7. Origin-side fixes found during rollout (2026-08-28)

Two things broke the edge→origin hop and were fixed in-repo:

### `originServerName` per ingress rule — `configmap.yaml`

`service:` is dialed by **IP** (the ingress-nginx VIP). Without
`originServerName`, cloudflared validates the origin cert against the IP:

```
tls: failed to verify certificate: x509: cannot validate certificate for
10.10.2.200 because it doesn't contain any IP SANs        → HTTP 502
```

The engatwork PKI certs carry **DNS SANs only**. Setting
`originRequest.originServerName: <hostname>` makes cloudflared send SNI =
hostname (so ingress-nginx serves the right cert) and check the SAN against
the hostname. Fixed in `2917662`.

### grafana ingress had no TLS — `observability/grafana/values.yaml`

`grafana.apps.mgmt-observability` Ingress was **HTTP-only** (port 80, no
`tls:` block, no `cert-manager` annotation) — its four siblings
(alertmanager / loki / prometheus / tempo) all had `vault-pki` TLS. The
tunnel dials `https://` and got ingress-nginx's fake `ingress.local` cert
→ `caPool` validation fail → 502. Added `grafana-tls` via the
`vault-pki` ClusterIssuer (now present on mgmt-observability). Fixed in
`26430ba`.

**Lesson for adding a Tier-1 host later:** its origin Ingress must serve a
real engatwork-CA cert for that exact hostname, or the tunnel 502s. Check
with:
```
echo | openssl s_client -connect <vip>:443 -servername <host> 2>/dev/null \
  | openssl x509 -noout -issuer -ext subjectAltName
```

---

## References

- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Tunnel on Kubernetes](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deployment-guides/kubernetes/)
- [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/policies/access/)
- [WARP / private networks](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/private-net/)
- `../README.md` · `../todo.md` · `../../../SESSION-NOTES.md`
- `ovirt-setup/playbooks/network/pfsense/README.md` — §Inbound / NAT port-forwards
- repo memory: `feedback_gitignore_secrets`, `feedback_checklist_driver`, `feedback_explain_foundations`
