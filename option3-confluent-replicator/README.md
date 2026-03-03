# Option 3: Confluent for Kubernetes (CFK) Replicator

This option uses the **Confluent for Kubernetes (CFK)** operator to deploy Kafka Connect
with the Confluent Replicator connector for migrating data from AWS MSK to Confluent Cloud.

## Approach
- Uses Confluent's own `Connect` CRD (not Strimzi)
- Deploys a Kafka Connect cluster with Replicator pre-installed
- Supports partition reduction (50 → 10) via pre-created target topics

## Prerequisites
1. Confluent for Kubernetes (CFK) operator installed
2. Target topics pre-created on Confluent Cloud with 10 partitions
3. Confluent Platform license
4. Network connectivity to MSK and Confluent Cloud

## Deployment Order
```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-msk-credentials.yaml
kubectl apply -f 02-confluent-cloud-credentials.yaml
kubectl apply -f 03-connect-cluster.yaml
kubectl apply -f 04-replicator-connector.yaml
```

## Key Partition Reduction Settings
| Setting | Value | Purpose |
|---------|-------|---------|
| `topic.creation.enable` | `false` | Prevents auto-creating topics with source partition count |
| `topic.config.sync` | `false` | Don't sync partition count from source |
| `topic.preserve.partitions` | `false` | Don't preserve source partition assignment |
| `producer.override.partitioner.class` | `DefaultPartitioner` | Redistributes by key across 10 partitions |
