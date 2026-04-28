# data — current setup

## What lives here
The stateful-data plane for the homelab — currently planned, not yet deployed. Will host CloudNativePG (Postgres), Strimzi (Kafka in KRaft mode), Apicurio Schema Registry, and a Redis/KeyDB cache. AppSet will live under `argo-applications/sre/data/` and follow Pattern B (`role: forge`) — co-located with the rest of the forge tooling that consumes these.

## Services
| Service | Status | Cluster target | Pattern | Notes |
|---|---|---|---|---|
| CloudNativePG | planned | mgmt-forge | B | Operator + per-app `Cluster` CRDs; Keycloak and Harbor will migrate off bundled PG. |
| Strimzi (Kafka KRaft) | planned | mgmt-forge | B | KRaft mode (no ZooKeeper); for event-driven workloads and audit pipelines. |
| Apicurio Schema Registry | planned | mgmt-forge | B | Pairs with Kafka; Avro/JSON Schema validation on produce/consume. |
| Redis or KeyDB | planned | mgmt-forge | B | Cache and ephemeral state; KeyDB if multi-master is needed, Redis otherwise. |

## Dependencies
- **Upstream** (this domain assumes is already running): rook-ceph CSI from `mgmt-storage` for persistent volumes; cert-manager for internal mTLS; Vault for credentials; ESO for syncing them into Pods.
- **Downstream** (depends on this domain): Keycloak (PG), Harbor (PG, Redis), SonarQube (PG), GitLab when it lands (PG, Redis, optional Kafka), any app needing event streaming or cache.

## Architectural decisions
- **CNPG over StatefulSet+self-managed PG** because the operator handles failover, backup, PITR, and PG-version upgrades declaratively. We don't want to be the people running Patroni by hand.
- **KRaft Kafka over ZooKeeper-Kafka** because ZooKeeper is being deprecated upstream; new deploys should not adopt it.
- **Centralized data plane** rather than each forge service running its own PG: one operator to operate, one backup story, one failover story.

## Known issues
- Nothing yet — domain is planned, not deployed.
