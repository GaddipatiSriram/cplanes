# data — TODO

Backlog ordered by likely sequence.

---

## Deploy CloudNativePG operator on `mgmt-forge`
**Status:** not started.

First thing to land. Operator-only deploy (no Cluster CRs yet) so it's ready when the first consumer (likely Keycloak) wants a managed PG. "Done" means the operator is healthy and can accept a `Cluster` CR.

---

## Migrate Keycloak's PG from the Bitnami subchart to a CNPG `Cluster`
**Status:** blocked on operator deploy.

Once CNPG is up, Keycloak is the obvious first consumer — its bundled PG is the weakest link in the identity stack. Plan needs `pg_dump` from the subchart and `pg_restore` into the CNPG cluster, then swap Keycloak's connection string. "Done" means Keycloak runs against CNPG and the subchart values are removed.

---

## Deploy Strimzi (Kafka KRaft) for audit log shipping
**Status:** not started.

First Kafka use case is shipping Vault audit logs and ArgoCD event streams to a topic that observability/security can consume. "Done" means Vault audit logs land in a Kafka topic and a downstream consumer can replay them.

---

## Stand up Apicurio Schema Registry alongside Kafka
**Status:** not started.

Pairs with Kafka — once any topic carries Avro-encoded events, schemas need a registry. "Done" means producer/consumer libraries can resolve schemas from the registry and reject mismatches.

---

## Decide Redis vs KeyDB for caching tier
**Status:** not started.

Redis is the obvious default; KeyDB earns its slot if multi-master writes are needed. Decision should consider memory model, persistence requirements, and operator availability. "Done" means a written ADR plus a deploy of the chosen one.
