# data — learning Q&A

Study log for stateful-data-plane concepts. Use as a study reference; revisit when abstractions stop making sense.

---

## Q1. Why CloudNativePG over a StatefulSet + self-managed Postgres?

**A.** A StatefulSet gives you ordered Pods with stable network identities and PVCs. It does not give you streaming replication, automatic failover, backup, PITR, or version upgrades — those are application-level concerns the operator handles for you.

CloudNativePG is a kubernetes-native operator that owns the `Cluster` CRD and handles:
- promoting a replica when the primary fails (no Patroni, no Stolon)
- streaming replication setup between Pods
- continuous archive to object storage (WAL shipping for PITR)
- in-place minor-version upgrades and orchestrated major-version upgrades
- TLS between Pods using a generated CA

You can do all of this with a StatefulSet plus enough bash, but the operator turns it into declarative YAML. For a homelab where there's no full-time DBA, that tradeoff is heavily in CNPG's favor.

---

## Q2. When pick KRaft Kafka over ZooKeeper-Kafka?

**A.** Always, for new deploys. ZooKeeper-managed Kafka is being removed upstream — Kafka 4.0 drops ZK support entirely. Adopting ZK in a new cluster is signing up for a migration in the near future.

KRaft (Kafka Raft) replaces ZooKeeper with a Raft-based consensus layer running inside the Kafka brokers themselves. Operationally this means: one process to deploy instead of two, one set of configs, one set of metrics, no ZK ensemble to size and tune.

The reasons you'd still pick ZK-mode are narrow: an existing migration where the rest of your tooling assumes ZK, or a Kafka feature that hadn't reached KRaft parity at the time of decision. For a fresh homelab deploy in 2026, neither applies.

---

## Q3. What does Apicurio Schema Registry do, and why is it needed alongside Kafka?

**A.** Kafka is schema-agnostic — it ships bytes. Producer encodes some object, consumer decodes some bytes; if the two disagree about the schema, the consumer crashes or (worse) silently misinterprets data.

A schema registry stores schemas (Avro, Protobuf, JSON Schema), assigns each a version, and producers/consumers check schemas against the registry instead of hard-coding them. The producer's serializer registers the schema if needed and prepends a schema-ID to each message; the consumer's deserializer reads the ID, fetches the matching schema, and decodes accordingly.

This buys you: schema evolution rules (the registry rejects breaking changes), one source of truth for what's on each topic, and the ability for new consumers to come online and discover what to expect.

Apicurio is the Red Hat / open-source registry; Confluent's Schema Registry is the other big one. Apicurio supports more schema formats and ships under Apache 2.0.
