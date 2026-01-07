# Technology Stack - Detailed Breakdown

## Overview
This document provides detailed information about each open-source technology in the data lake stack, including installation, configuration, and cost analysis.

---

## Storage Layer

### MinIO
**Purpose**: S3-compatible object storage for petabyte-scale data

**Why MinIO?**
- 100% open source (AGPLv3 license)
- S3 API compatible (drop-in replacement)
- Production deployments handling 100+ PB
- Erasure coding for data protection
- Active-active replication

**Deployment**:
```bash
# Using Helm
helm repo add minio https://charts.min.io/
helm install minio minio/minio \
  --set mode=distributed \
  --set replicas=16 \
  --set persistence.size=1Ti

# Or using Operator for production
kubectl apply -f https://github.com/minio/operator/releases/latest/download/minio-operator.yaml
```

**Cost Analysis**:
- Hardware: ~$10/TB one-time (commodity servers)
- No licensing fees
- Electricity: ~$0.10/TB/month
- Total: ~$0.40/TB/month (amortized over 3 years)

**Performance**:
- Throughput: 10+ GB/s per node
- IOPS: 100k+ per node
- Latency: <10ms for small objects

---

### Apache Iceberg
**Purpose**: Table format with ACID transactions, schema evolution, and time travel

**Why Iceberg?**
- ACID guarantees without compromising performance
- Time travel (query historical data)
- Schema evolution without rewrites
- Hidden partitioning (automatic optimization)
- Works with Spark, Flink, Trino, etc.

**Configuration**:
```python
# Spark configuration
spark.sql.catalog.my_catalog = org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.my_catalog.type = hadoop
spark.sql.catalog.my_catalog.warehouse = s3a://datalake/warehouse
spark.sql.catalog.my_catalog.io-impl = org.apache.iceberg.aws.s3.S3FileIO

# Create table
spark.sql("""
  CREATE TABLE my_catalog.db.events (
    event_id STRING,
    timestamp TIMESTAMP,
    user_id STRING,
    event_type STRING,
    properties MAP<STRING, STRING>
  )
  USING iceberg
  PARTITIONED BY (days(timestamp), event_type)
  TBLPROPERTIES (
    'format-version' = '2',
    'write.parquet.compression-codec' = 'zstd'
  )
""")
```

**Cost**: Zero (library, not a service)

---

### PostgreSQL + pgvector
**Purpose**: Metadata catalog, relational data, vector search

**Why PostgreSQL?**
- Most popular open-source RDBMS
- pgvector extension for vector similarity search
- Full ACID compliance
- Rich ecosystem (replication, monitoring, etc.)

**Deployment**:
```bash
# Using Zalando Postgres Operator
kubectl apply -f https://raw.githubusercontent.com/zalando/postgres-operator/master/manifests/postgres-operator.yaml

# Install pgvector
CREATE EXTENSION vector;
```

**Cost**: Free, hardware costs only

---

## Ingestion Layer

### Apache Kafka
**Purpose**: Distributed event streaming platform

**Why Kafka?**
- Industry standard for event streaming
- Handles millions of events per second
- Durable, fault-tolerant
- Rich ecosystem (Connect, Streams, Schema Registry)

**Deployment**:
```bash
# Using Strimzi operator
kubectl create namespace kafka
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Deploy cluster
kubectl apply -f - <<EOF
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: datalake-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    storage:
      type: jbod
      volumes:
      - id: 0
        type: persistent-claim
        size: 1Ti
        deleteClaim: false
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 100Gi
      deleteClaim: false
EOF
```

**Sizing**:
- 3 brokers: 1M msgs/sec
- 10 brokers: 5M msgs/sec
- 50 brokers: 20M+ msgs/sec

**Cost**: ~$200/broker/month (compute + storage)

---

### Apache NiFi
**Purpose**: Visual data flow management and ETL

**Why NiFi?**
- User-friendly web UI for building pipelines
- 300+ processors (connectors)
- Built-in backpressure and prioritization
- Data provenance tracking

**Deployment**:
```bash
helm repo add cetic https://cetic.github.io/helm-charts
helm install nifi cetic/nifi \
  --set replicaCount=3 \
  --set persistence.enabled=true
```

**Cost**: ~$150/node/month

---

### Debezium
**Purpose**: Change Data Capture (CDC) for databases

**Why Debezium?**
- Real-time streaming of database changes
- Supports MySQL, PostgreSQL, MongoDB, SQL Server, Oracle
- Exactly-once semantics
- Schema evolution handling

**Deployment**:
```bash
# Deploy as Kafka Connect connectors
# Example for PostgreSQL
curl -X POST http://kafka-connect:8083/connectors -H "Content-Type: application/json" -d '{
  "name": "postgres-source",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres.example.com",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "secret",
    "database.dbname": "production",
    "database.server.name": "prod-db",
    "table.include.list": "public.orders,public.customers",
    "plugin.name": "pgoutput"
  }
}'
```

**Cost**: Free (runs on Kafka Connect)

---

## Processing Layer

### Apache Spark
**Purpose**: Unified analytics engine for batch processing

**Why Spark?**
- De facto standard for big data processing
- Processes 10+ TB/hour
- Rich APIs (SQL, DataFrame, ML)
- Works with all major file formats

**Deployment**:
```bash
# Using Spark Operator
kubectl apply -f https://github.com/GoogleCloudPlatform/spark-on-k8s-operator/releases/latest/download/spark-operator.yaml

# Submit job
kubectl apply -f - <<EOF
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication
metadata:
  name: bronze-to-silver
spec:
  type: Python
  pythonVersion: "3"
  mode: cluster
  image: "datalake/spark-jobs:latest"
  mainApplicationFile: "s3a://datalake/jobs/bronze_to_silver.py"
  sparkVersion: "3.5.0"
  driver:
    cores: 2
    memory: "4g"
  executor:
    cores: 4
    instances: 10
    memory: "8g"
EOF
```

**Cost**:
- Small cluster (10 executors): ~$500/month
- Large cluster (100 executors): ~$5,000/month

---

### Apache Flink
**Purpose**: Real-time stream processing

**Why Flink?**
- True stream processing (not micro-batching)
- Exactly-once semantics
- Sub-second latency
- Advanced windowing and state management

**Deployment**:
```bash
helm repo add flink-operator https://downloads.apache.org/flink/flink-kubernetes-operator-1.6.0/
helm install flink-operator flink-operator/flink-kubernetes-operator

# Deploy job
kubectl apply -f - <<EOF
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: realtime-processor
spec:
  image: flink:1.18
  flinkVersion: v1_18
  jobManager:
    resource:
      memory: "2048m"
      cpu: 1
  taskManager:
    resource:
      memory: "4096m"
      cpu: 2
    replicas: 5
EOF
```

**Cost**: ~$300/task manager/month

---

### Apache Airflow
**Purpose**: Workflow orchestration

**Why Airflow?**
- Python-based DAGs (easy to version control)
- Rich UI for monitoring
- 200+ operators
- Large community

**Deployment**:
```bash
helm repo add apache-airflow https://airflow.apache.org
helm install airflow apache-airflow/airflow \
  --set executor=KubernetesExecutor \
  --set postgresql.enabled=true \
  --set redis.enabled=true
```

**Example DAG**:
```python
from airflow import DAG
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'datalake',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'daily_etl',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False
)

bronze_to_silver = SparkSubmitOperator(
    task_id='bronze_to_silver',
    application='s3a://datalake/jobs/bronze_to_silver.py',
    dag=dag
)

silver_to_gold = SparkSubmitOperator(
    task_id='silver_to_gold',
    application='s3a://datalake/jobs/silver_to_gold.py',
    dag=dag
)

bronze_to_silver >> silver_to_gold
```

**Cost**: ~$200/month (small deployment)

---

## Query Layer

### Trino (formerly PrestoSQL)
**Purpose**: Distributed SQL query engine

**Why Trino?**
- Queries data where it lives (no ETL needed)
- Federates across multiple sources
- Sub-second latency for TB-scale data
- ANSI SQL support

**Deployment**:
```bash
helm repo add trino https://trinodb.github.io/charts
helm install trino trino/trino \
  --set server.workers=10 \
  --set server.coordinator.resources.memory=32Gi

# Configure catalogs
# s3://bucket/trino/catalog/iceberg.properties
connector.name=iceberg
iceberg.catalog.type=hadoop
iceberg.catalog.warehouse=s3a://datalake/warehouse
```

**Performance**:
- 10 workers: 1-10 TB queries
- 50 workers: 10-100 TB queries
- 200 workers: 100+ TB queries

**Cost**: ~$400/worker/month

---

### Apache Druid
**Purpose**: Real-time OLAP database

**Why Druid?**
- Sub-second queries on event data
- Real-time ingestion (1M+ events/sec)
- Automatic rollup and retention
- Time-series optimized

**Deployment**:
```bash
helm repo add druid https://apache.github.io/druid/
helm install druid druid/druid
```

**Use Cases**:
- Real-time dashboards
- User behavior analytics
- Time-series metrics
- Clickstream analysis

**Cost**: ~$500/month (small cluster)

---

### Elasticsearch
**Purpose**: Full-text search and log analytics

**Why Elasticsearch?**
- Best-in-class full-text search
- Rich query DSL
- Aggregations and analytics
- Visualization with Kibana

**Deployment**:
```bash
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch \
  --set replicas=3 \
  --set minimumMasterNodes=2

helm install kibana elastic/kibana
```

**Cost**: ~$300/node/month

---

### Milvus
**Purpose**: Vector database for AI/ML embeddings

**Why Milvus?**
- Built for billion-scale vector search
- 10+ indexing algorithms (HNSW, IVF, etc.)
- GPU acceleration support
- Hybrid search (vector + scalar filtering)

**Deployment**:
```bash
helm repo add milvus https://milvus-io.github.io/milvus-helm/
helm install milvus milvus/milvus \
  --set cluster.enabled=true \
  --set etcd.replicaCount=3 \
  --set pulsar.enabled=true
```

**Use Cases**:
- Semantic search
- Recommendation systems
- Image similarity search
- Question answering (RAG)

**Cost**: ~$600/month (cluster mode)

---

## API Layer

### FastAPI
**Purpose**: High-performance REST API framework

**Why FastAPI?**
- Fastest Python framework (async)
- Auto-generated OpenAPI docs
- Type checking with Pydantic
- WebSocket support

**Example**:
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import trino

app = FastAPI()

class QueryRequest(BaseModel):
    sql: str
    limit: int = 1000

@app.post("/query")
async def execute_query(request: QueryRequest):
    conn = trino.dbapi.connect(
        host='trino-coordinator',
        port=8080,
        catalog='iceberg',
        schema='default'
    )

    cursor = conn.cursor()
    cursor.execute(request.sql + f" LIMIT {request.limit}")

    columns = [desc[0] for desc in cursor.description]
    rows = cursor.fetchall()

    return {
        "columns": columns,
        "data": rows,
        "row_count": len(rows)
    }
```

**Performance**: 10,000+ req/sec per instance

**Cost**: ~$50/instance/month

---

### Hasura
**Purpose**: Auto-generated GraphQL APIs

**Why Hasura?**
- Instant GraphQL over PostgreSQL
- Role-based access control
- Real-time subscriptions
- Remote schema stitching

**Deployment**:
```bash
helm repo add hasura https://hasura.github.io/helm-charts
helm install hasura hasura/hasura \
  --set postgresql.enabled=false \
  --set env.HASURA_GRAPHQL_DATABASE_URL=postgres://user:pass@postgres:5432/metadata
```

**Cost**: Free (open source)

---

### Redis
**Purpose**: In-memory cache and rate limiting

**Why Redis?**
- Extremely fast (sub-millisecond latency)
- Rich data structures
- Pub/sub messaging
- Cluster mode for HA

**Deployment**:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install redis bitnami/redis \
  --set architecture=replication \
  --set auth.password=secretpassword
```

**Cost**: ~$100/month (small cluster)

---

## Monitoring & Governance

### Prometheus + Grafana
**Purpose**: Metrics collection and visualization

**Deployment**:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kube-prometheus prometheus-community/kube-prometheus-stack
```

**Cost**: ~$200/month

---

### OpenSearch
**Purpose**: Centralized logging

**Deployment**:
```bash
helm repo add opensearch https://opensearch-project.github.io/helm-charts/
helm install opensearch opensearch/opensearch
helm install opensearch-dashboards opensearch/opensearch-dashboards
```

**Cost**: ~$300/month

---

### Apache Atlas
**Purpose**: Data catalog and governance

**Deployment**:
```bash
# Requires HBase, Kafka, Solr
kubectl apply -f atlas-deployment.yaml
```

**Cost**: ~$400/month

---

## Total Cost Summary

### Minimal Production Setup (1 PB data)
```
Storage (MinIO): $400/month
Compute (K8s): $3,000/month
Kafka: $600/month
Spark: $500/month
Trino: $1,200/month
Monitoring: $500/month
---
Total: ~$6,200/month
```

### Medium Scale (5 PB data, 50k users)
```
Storage: $2,000/month
Compute: $8,000/month
Kafka: $2,000/month
Spark: $2,000/month
Flink: $1,500/month
Trino: $4,000/month
Druid: $1,000/month
Elasticsearch: $900/month
Milvus: $600/month
APIs: $500/month
Monitoring: $1,000/month
---
Total: ~$23,500/month
```

### Large Scale (10 PB data, 100k users)
```
Storage: $4,000/month
Compute: $12,000/month
Kafka: $3,000/month
Spark: $5,000/month
Flink: $3,000/month
Trino: $8,000/month
Druid: $2,000/month
Elasticsearch: $1,500/month
Milvus: $1,200/month
APIs: $1,000/month
Monitoring: $2,000/month
---
Total: ~$42,700/month
```

**Cost per TB**: $4.27/month (large scale)
**Cost per user**: $0.43/month (100k users)

---

## Licensing Summary

All technologies are open source with permissive licenses:

- **Apache 2.0**: Kafka, Spark, Flink, Airflow, Druid, Iceberg, Atlas
- **AGPLv3**: MinIO, Elasticsearch (OpenSearch is Apache 2.0)
- **MIT**: Redis, FastAPI
- **PostgreSQL License**: PostgreSQL

No licensing costs, full control, no vendor lock-in.
