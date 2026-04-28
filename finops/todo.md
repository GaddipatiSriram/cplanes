# finops — TODO

Backlog ordered by likely sequence.

---

## Author the canonical chart for the first service in this plane

**Status:** not started.

The plane has placeholder folders for each planned service. First step is picking ONE service to bootstrap end-to-end (chart + AppSet + cluster registration) so the pattern is established for the others.

---

## Wire the plane's AppSet pattern into argo-applications/sre/finops/

**Status:** not started.

Decide whether services in this plane are Pattern A (every workload cluster), Pattern B (single central cluster), or one-off Applications. Author the matching ApplicationSet(s) once the chart pattern is clear.

---

## Document plane-specific operational runbooks

**Status:** not started.

Each plane benefits from per-incident runbooks (e.g., "what to do when service X goes down"). Author as encountered.
