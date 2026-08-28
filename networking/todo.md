# networking — TODO

Backlog ordered by likely sequence.

---

## cloudflared — Cloudflare Tunnel for public internet access

**Status:** in progress (decided 2026-08-27). Design, current state, and
the full rollout checklist live in [`cloudflared/README.md`](./cloudflared/README.md).

Short form:

- [ ] **pfSense** — locate and **remove** any WAN NAT port-forward from
      Iteration 1 (Firewall → NAT → Port Forward) + its associated filter
      rule; tighten `WAN_EasyRule_Pass` (any→any on WAN). Codify "zero
      inbound port-forwards" in `ovirt-setup/.../pfsense/`. Tunnel is
      outbound-only.
- [ ] **Cloudflare** — create tunnel `homelab`; swap the Iteration-1
      proxied `A` record(s) for 6 Tier-1 `CNAME`s → `*.cfargotunnel.com`;
      SSL mode → Full (strict); add `10.10.0.0/16` WARP route; Access
      apps (Google-only) per Tier-1 host.
- [ ] **Repo** — author `cloudflared/` chart (self-contained Deployment +
      ConfigMap ingress rules + `engatwork-ca.crt` mount), ExternalSecret
      for the tunnel token (Vault `kv/forge/cloudflared/tunnel`; do NOT
      name the file `*-secret.yaml`), and a Pattern-B AppSet at
      `../argo-applications/sre/networking/cloudflared-appset.yaml`.
- [ ] Update this plane's README service table status → ✅ live.

Iteration 1 (Cloudflare proxy + pfSense port-forward, SSL Full) is
**superseded** — see `cloudflared/README.md` §1 for why.

---

## Author the canonical chart for the first service in this plane

**Status:** not started.

The plane has placeholder folders for each planned service. First step is picking ONE service to bootstrap end-to-end (chart + AppSet + cluster registration) so the pattern is established for the others.

---

## Wire the plane's AppSet pattern into argo-applications/sre/networking/

**Status:** not started.

Decide whether services in this plane are Pattern A (every workload cluster), Pattern B (single central cluster), or one-off Applications. Author the matching ApplicationSet(s) once the chart pattern is clear.

---

## Document plane-specific operational runbooks

**Status:** not started.

Each plane benefits from per-incident runbooks (e.g., "what to do when service X goes down"). Author as encountered.
