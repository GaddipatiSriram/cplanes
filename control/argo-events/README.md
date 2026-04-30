# argo-events

Event-driven workflow automation. Bridges external event producers
(webhooks, GitHub, Kafka, k8s API, S3, calendar, etc.) to Kubernetes
actions (Argo Workflows, raw k8s resources, AWS Lambda, HTTP).

## What this chart deploys

- **EventBus controller** — manages NATS-based event brokers
- **EventSource controller** — manages subscriptions to external producers
- **Sensor controller** — evaluates dependencies + fires triggers
- **CRDs**: `EventBus`, `EventSource`, `Sensor`
- **ServiceMonitor** for Prometheus

The controllers are tiny; the actual brokers and event-source pods are
created on-demand via the CRs you author.

## The three building blocks

```yaml
# 1. EventBus — the broker (NATS JetStream by default)
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata: { name: default }
spec:
  jetstream: { version: latest }
---
# 2. EventSource — subscribes to external producer
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata: { name: github-webhook }
spec:
  github:
    pull-request:
      repositories: [{ owner: GaddipatiSriram, names: [cplanes] }]
      webhook:
        endpoint: /push
        port: "12000"
        method: POST
      events: [pull_request]
---
# 3. Sensor — listens on EventBus, fires triggers
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata: { name: pr-build }
spec:
  dependencies:
    - name: pr
      eventSourceName: github-webhook
      eventName: pull-request
  triggers:
    - template:
        name: trigger-workflow
        argoWorkflow:
          operation: submit
          source:
            resource: { ... ArgoWorkflow CR ... }
```

## Why it's here (cert + role mapping)

| Cert | What it tests |
|---|---|
| **CAPA** | Events is 1 of 4 Argo pillars; ~10% of exam |
| **CNPE** | Event-driven self-service patterns; webhook-driven onboarding |

| Role | What you'd do here |
|---|---|
| **DevOps Engineer** | Wire GitHub PR events → Argo Workflow build pipelines |
| **DevEx / IDP Engineer** | Webhook → Backstage scaffolder template |
| **SRE** | Calendar event → maintenance window automation |

## Cert prep — concrete exercises

1. **Author an EventBus** with NATS JetStream (single-replica for homelab):
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: argoproj.io/v1alpha1
   kind: EventBus
   metadata: { name: default, namespace: argo-events }
   spec:
     jetstream: { version: latest, replicas: 1 }
   EOF
   ```

2. **Wire a webhook EventSource** that listens on a Service:
   ```bash
   kubectl apply -f - <<EOF
   apiVersion: argoproj.io/v1alpha1
   kind: EventSource
   metadata: { name: webhook, namespace: argo-events }
   spec:
     service: { ports: [{ port: 12000, targetPort: 12000 }] }
     webhook:
       hello: { port: "12000", endpoint: /hello, method: POST }
   EOF
   ```

3. **Test it** — `curl http://<svc>:12000/hello -d '{}'` and watch a Sensor fire.

4. **Memorize the dependency model** — Sensor `dependencies[].eventSourceName`
   + `.eventName` is how it ties back to the EventSource's named event.

## Cross-references

- [`/root/learning/certifications/capa.md`](../../../certifications/capa.md) — CAPA prep
- [`../argo-workflows/`](../argo-workflows/) — Argo Workflows (often the trigger target)
