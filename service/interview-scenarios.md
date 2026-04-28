# data — interview scenarios

Architecture / ops / failure / migration scenarios for practice. Each scenario is a prompt — answer it as if walking an interviewer through your design.

---

## Scenario 1: Postgres primary fails mid-traffic in a CNPG cluster

**Context.** CNPG is running a 3-instance `Cluster` for Harbor's metadata: one primary, two replicas with synchronous replication. At 14:00 on a weekday the primary's underlying node hard-fails.

**The ask.** Walk through detection, failover, and recovery.

**Walk-through.**
- **Detection**: CNPG's operator notices the primary's leader-lease expires; replicas notice the WAL stream stopped. Health checks on the Service start failing.
- **Failover**: the operator picks the most up-to-date replica (lowest replication lag) and promotes it. Promotion writes a `pg_ctl promote`, the new primary opens for writes, the operator updates the read-write Service endpoint to point at the new Pod.
- **Reconciliation**: surviving Pods reconfigure their replication source to the new primary. The original primary's PVC is left intact for forensics.
- **Application impact**: any in-flight writes during the failover window get connection errors; Harbor's connection pool reconnects within seconds. Reads hitting the read-only Service may get stale data briefly until replication catches up.
- **Recovery**: when the failed node returns, the operator re-attaches the dead Pod's PVC as a new replica, replicates from the current primary's WAL, and rejoins the cluster. If WAL has rotated past what the dead replica needs, the operator rebuilds it from a base backup + WAL.

**Tradeoffs.** Synchronous replication gives RPO=0 but adds latency to every write. Async gives lower latency but a small window of data loss on primary failure. For metadata stores like Harbor's, sync is the right call.

**Follow-up questions.**
- What metric tells you replication lag is growing past acceptable bounds?
- How does CNPG's `minSyncReplicas` setting change failure behavior?
- If the object-store WAL archive is unreachable, what breaks?

---

## Scenario 2: A Kafka topic's schema needs a backwards-incompatible change

**Context.** A `payments` topic carries Avro-encoded messages. The schema currently has a required `currency` field. A downstream consumer wants the field renamed to `currency_code`. There are 6 producers and 12 consumers in flight.

**The ask.** Roll out the change without breaking anyone.

**Walk-through.**
- The change as proposed is a *breaking* schema evolution — Apicurio's compatibility checks should reject it under any compatibility level except NONE.
- Standard play: add a new optional field `currency_code` alongside `currency` (additive change, backwards-compatible). Producers start writing both. Consumers update one at a time to read the new field, falling back to the old. Once all consumers are migrated, deprecate `currency` (mark as deprecated in the schema, eventually drop in a major-version bump).
- During rollout, monitor the registry's compatibility errors and the Kafka consumer-group lag — a consumer that breaks will lag and surface the problem fast.
- Coordinate with the team owning the topic: schema changes need a documented owner, not just a registry write.

**Tradeoffs.** Forward-and-backward compatible evolution is slower (months instead of a deploy window) but doesn't require coordinating downtime across 18 services. A breaking change with a coordinated cutover is faster but requires every team to be ready at the same instant.

**Follow-up questions.**
- What's the difference between BACKWARD, FORWARD, and FULL compatibility in a schema registry?
- How would Kafka Streams consumers behave differently than raw consumers during the transition?
- How do you detect a producer that's still writing the old schema after you thought migration was complete?
