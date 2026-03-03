# MSK to Confluent Cloud Migration

Kubernetes YAML manifests for migrating Apache Kafka topics from **AWS MSK** to **Confluent Cloud** with support for **partition reduction** (e.g., 50 source partitions → 10 target partitions).

Three approaches are provided using different Kubernetes operators.

## Options

### Option 1: Strimzi KafkaConnect + Confluent Replicator

Uses Strimzi's `KafkaConnect` CRD with the Confluent Replicator connector plugin.

| File | Description |
|------|-------------|
| `option1-kafkaconnect-replicator/00-secrets.yaml` | Kubernetes Secrets for MSK & Confluent Cloud credentials |
| `option1-kafkaconnect-replicator/01-kafkaconnect.yaml` | KafkaConnect cluster with Replicator plugin build |
| `option1-kafkaconnect-replicator/02-replicator-connector.yaml` | Replicator KafkaConnector with partition reduction config |

> **Note:** Requires Strimzi operator + Confluent Platform license.

### Option 2: Strimzi KafkaMirrorMaker2 (No License Required)

Uses Strimzi's native `KafkaMirrorMaker2` CRD built on Apache Kafka MirrorMaker 2.

| File | Description |
|------|-------------|
| `option2-mirrormaker2/00-secrets.yaml` | Kubernetes Secrets for MSK & Confluent Cloud credentials |
| `option2-mirrormaker2/01-mirrormaker2.yaml` | MirrorMaker2 with partition reduction & offset sync |

> **Note:** Requires Strimzi operator. No Confluent license needed.

### Option 3: Confluent for Kubernetes (CFK) Replicator

Uses the **Confluent for Kubernetes (CFK)** operator with the `Connect` and `Connector` CRDs to deploy Confluent Replicator natively.

| File | Description |
|------|-------------|
| `option3-confluent-replicator/00-namespace.yaml` | Namespace for Confluent Replicator |
| `option3-confluent-replicator/01-msk-credentials.yaml` | MSK source cluster credentials & TLS |
| `option3-confluent-replicator/02-confluent-cloud-credentials.yaml` | Confluent Cloud credentials & license |
| `option3-confluent-replicator/03-connect-cluster.yaml` | CFK Connect cluster with Replicator image |
| `option3-confluent-replicator/04-replicator-connector.yaml` | Replicator Connector with partition reduction & performance tuning |
| `option3-confluent-replicator/05-monitoring-and-scaling.yaml` | PodMonitor, HPA autoscaling & PodDisruptionBudget |

> **Note:** Requires CFK operator + Confluent Platform license.

## Prerequisites

1. **Kubernetes operator** installed:
   - Options 1 & 2: [Strimzi Kafka Operator](https://strimzi.io/)
   - Option 3: [Confluent for Kubernetes (CFK)](https://docs.confluent.io/operator/current/overview.html)
2. **Target topics pre-created** on Confluent Cloud with the desired partition count (e.g., 10)
3. Network connectivity from K8s cluster to both MSK and Confluent Cloud

### Pre-create Target Topics

```bash
confluent kafka topic create my-topic \
  --partitions 10 \
  --cluster <CCLOUD_CLUSTER_ID> \
  --environment <CCLOUD_ENV_ID>
```

## Key Configuration for Partition Reduction (50 → 10)

| Setting | Purpose |
|---------|---------|
| `topic.auto.create: "false"` | Prevents auto-creating topics with source partition count |
| `topic.preserve.partitions: "false"` | (Replicator) Disables source partition preservation |
| `producer.override.partitioner.class` | Redistributes records by key across target partitions |
| `tasksMax: 10` | Matches target partition count for optimal throughput |

## Usage

1. Replace all placeholder values (`<MSK_BOOTSTRAP_SERVERS>`, `<CCLOUD_API_KEY>`, etc.)
2. Pre-create target topics with desired partition count on Confluent Cloud
3. Deploy:

   **Option 1 (Strimzi + Replicator):**
   ```bash
   kubectl apply -f option1-kafkaconnect-replicator/00-secrets.yaml
   kubectl apply -f option1-kafkaconnect-replicator/01-kafkaconnect.yaml
   kubectl apply -f option1-kafkaconnect-replicator/02-replicator-connector.yaml
   ```

   **Option 2 (Strimzi MirrorMaker2):**
   ```bash
   kubectl apply -f option2-mirrormaker2/00-secrets.yaml
   kubectl apply -f option2-mirrormaker2/01-mirrormaker2.yaml
   ```

   **Option 3 (CFK Replicator):**
   ```bash
   kubectl apply -f option3-confluent-replicator/00-namespace.yaml
   kubectl apply -f option3-confluent-replicator/01-msk-credentials.yaml
   kubectl apply -f option3-confluent-replicator/02-confluent-cloud-credentials.yaml
   kubectl apply -f option3-confluent-replicator/03-connect-cluster.yaml
   kubectl apply -f option3-confluent-replicator/04-replicator-connector.yaml
   kubectl apply -f option3-confluent-replicator/05-monitoring-and-scaling.yaml  # Optional
   ```

## Comparison

| Feature | Option 1 (Strimzi + Replicator) | Option 2 (Strimzi MM2) | Option 3 (CFK Replicator) |
|---|---|---|---|
| **Operator** | Strimzi | Strimzi | CFK |
| **License** | Confluent license | Open source | Confluent license |
| **CRDs** | KafkaConnect + KafkaConnector | KafkaMirrorMaker2 | Connect + Connector |
| **Custom Image** | Required (plugin build) | Not required | Not required (uses CP image) |
| **Offset Sync** | Manual | Built-in checkpoint | Manual |
| **Monitoring** | Manual setup | Manual setup | Built-in metrics + HPA |
| **Best For** | Strimzi shops needing Replicator | Open source / simple migrations | Confluent Platform customers |
