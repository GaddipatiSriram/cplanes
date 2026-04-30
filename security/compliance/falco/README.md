# falco

Runtime security threat detection. eBPF-based syscall monitoring on
every node. Alerts when a container does something suspicious — reading
`/etc/shadow`, spawning `nc`, writing to `/proc`, network scanning, etc.

CNCF graduated. The OSS equivalent of commercial runtime-security
products (Sysdig Secure — built on Falco; Aqua; Prisma Cloud).

## What this chart deploys

- **DaemonSet** running falco on every node
- **Driver pod** (init container) loads the eBPF probe
- **ConfigMap** with rule files (default + custom)
- **ServiceMonitor** for Prometheus

## How Falco works (at a glance)

```
Container does syscall  →  eBPF probe in kernel  →  falco userspace
                                                    │
                                                    ↓
                                          Match against rules
                                                    │
                                          ┌─────────┴─────────┐
                                          ▼                   ▼
                                       Match!             No match
                                          │                   │
                                          ↓                   ↓
                                Emit alert (stdout       Drop (no overhead)
                                / syslog / HTTP)
```

eBPF probes are kernel-loaded but JIT-compiled and verified — safe;
they can read syscalls but can't modify them. Falco is detect-only by
default. Pair with Tetragon (Cilium) if you want kernel-level enforcement.

## Why it's here (cert + role mapping)

| Cert | What it tests |
|---|---|
| **CKS** | ~20% — runtime security domain. Custom rule authoring. |
| **KCSA** | Concept-level — what runtime security is. |
| **CNPE** | Capstone — security operations integration. |

| Role | What you'd do here |
|---|---|
| **Security Platform Engineer** | Tune rules per LOB; integrate with SIEM |
| **SRE** | Runtime alerts as part of incident triage |
| **Compliance Engineer** | Map Falco events to MITRE ATT&CK matrix |

## The default Falco ruleset (what comes built-in)

~150 rules covering:
- **Privileged container detected** — pod with `privileged: true`
- **Sensitive mount by container** — `/proc`, `/sys`, `/etc`
- **Container shell spawned** — interactive shell in a container
- **Write below /etc** — modification of system config
- **Read sensitive files** — `/etc/shadow`, `/etc/sudoers`
- **Outbound network connection from container**
- **Execve from script (interactive)** — bash spawning curl/wget
- **Kubernetes audit log events** — RBAC misuse, secret access

Read `/etc/falco/falco_rules.yaml` inside the pod to see the full set.

## Cert prep — concrete exercises

1. **Install + verify** — `kubectl logs -n falco -l app.kubernetes.io/name=falco -f`
   then `kubectl exec -it <some-pod> -- cat /etc/shadow` from a tainted
   namespace; watch the alert appear.

2. **Author a custom rule** — `customRules` block in values.yaml:
   ```yaml
   customRules:
     custom-rules.yaml: |-
       - rule: Suspicious sleep
         desc: A container ran sleep with a duration > 1 hour
         condition: >
           spawned_process and proc.name = "sleep" and
           proc.cmdline contains " 3600"
         output: >
           Long sleep detected (proc.cmdline=%proc.cmdline
           container=%container.name image=%container.image.repository)
         priority: NOTICE
   ```

3. **Test the rule** — `kubectl exec -it pod -- sleep 3600` in another
   shell; watch the alert in falco logs.

4. **Memorize Falco rule fields**:
   - `rule:` (name)
   - `desc:` (description)
   - `condition:` (the matching expression)
   - `output:` (what the alert message says)
   - `priority:` (EMERGENCY | ALERT | CRITICAL | ERROR | WARNING | NOTICE | INFORMATIONAL | DEBUG)
   - `tags:` (e.g., MITRE ATT&CK tactic IDs)

5. **Falco condition macros** — pre-defined patterns:
   - `container` — any container event
   - `spawned_process` — process started
   - `open_read` / `open_write` — file access
   - `sensitive_files` — list of sensitive paths
   - `inbound` / `outbound` — network direction

## Common operations

```bash
# tail Falco events live
kubectl logs -n falco daemonset/falco -f

# count alerts by priority
kubectl logs -n falco daemonset/falco | jq -r '.priority' | sort | uniq -c

# search for specific rule firings
kubectl logs -n falco daemonset/falco | jq 'select(.rule | contains("shell"))'

# verify eBPF driver loaded
kubectl exec -n falco -it <falco-pod> -- ls /sys/fs/bpf/
```

## Falco vs Tetragon (Cilium)

| | Falco | Tetragon |
|---|---|---|
| Detection | ✅ syscalls | ✅ syscalls + tracepoints + kprobes |
| Enforcement | ❌ detect-only | ✅ can kill processes |
| Rule format | Falco DSL (custom) | YAML TracingPolicy |
| Status | CNCF graduated | CNCF incubating |
| When to pick | Ubiquitous, mature | Cilium ecosystem alignment |

## Forward integration plan

- **Now**: Falco logs to stdout → Promtail → Loki → query in Grafana
- **Later**: Wire `falcosidekick` (chart subcomponent) → Slack /
  PagerDuty / ELK / OpenSearch SIEM (when stood up)
- **Future**: Add Tetragon for kernel-level enforcement; let Falco stay
  as detection layer

## Cross-references

- [`/root/learning/certifications/cks.md`](../../../../certifications/cks.md) — CKS prep
- [`/root/learning/certifications/kcsa.md`](../../../../certifications/kcsa.md) — KCSA prep
- [Falco Rules library](https://github.com/falcosecurity/rules) — community-curated rules
- [MITRE ATT&CK for Containers](https://attack.mitre.org/matrices/enterprise/containers/)
