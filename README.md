# MSK to Confluent Cloud Migration using Strimzi

Kubernetes YAML manifests for migrating Apache Kafka topics from **AWS MSK** to **Confluent Cloud** using the **Strimzi Kafka Operator**, with support for **partition reduction** (e.g., 50 source partitions → 10 target partitions).

## Options

### Option 1: KafkaConnect + Confluent Replicator

Uses Strimzi's `KafkaConnect` CRD with the Confluent Replicator connector plugin.

| File | Description |
|------|-------------|
| `option1-kafkaconnect-replicator/00-secrets.yaml` | Kubernetes Secrets for MSK & Confluent Cloud credentials |
| `option1-kafkaconnect-replicator/01-kafkaconnect.yaml` | KafkaConnect cluster with Replicator plugin build |
| `option1-kafkaconnect-replicator/02-replicator-connector.yaml` | Replicator KafkaConnector with partition reduction config |

> **Note:** Confluent Replicator requires a Confluent Platform license.

### Option 2: KafkaMirrorMaker2 (No License Required)

Uses Strimzi's native `KafkaMirrorMaker2` CRD built on Apache Kafka MirrorMaker 2.

| File | Description |
|------|-------------|
| `option2-mirrormaker2/00-secrets.yaml` | Kubernetes Secrets for MSK & Confluent Cloud credentials |
| `option2-mirrormaker2/01-mirrormaker2.yaml` | MirrorMaker2 with partition reduction & offset sync |

## Prerequisites

1. **Strimzi Kafka Operator** installed on your Kubernetes cluster
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
3. Apply the secrets: `kubectl apply -f <option>/00-secrets.yaml`
4. Apply the main resource:
   - **Option 1:** `kubectl apply -f option1-kafkaconnect-replicator/01-kafkaconnect.yaml` then `kubectl apply -f option1-kafkaconnect-replicator/02-replicator-connector.yaml`
   - **Option 2:** `kubectl apply -f option2-mirrormaker2/01-mirrormaker2.yaml`

## Recommendation

**Option 2 (KafkaMirrorMaker2)** is recommended for most use cases:
- No license cost (fully open source)
- Single CRD — simpler operations
- Built-in consumer offset sync for easier cutover
- No custom container image build required
