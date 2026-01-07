# Data Lake Quick Start Guide

This guide will help you get a minimal data lake up and running locally for development and testing.

---

## Prerequisites

- Docker Desktop (with Kubernetes enabled) or Minikube
- kubectl installed
- Helm 3.x installed
- Python 3.9+
- 16GB+ RAM recommended
- 100GB+ free disk space

---

## Local Development Setup

### Step 1: Start Kubernetes Cluster

```bash
# If using Docker Desktop
# Enable Kubernetes in Docker Desktop settings

# Or if using Minikube
minikube start --memory=8192 --cpus=4 --disk-size=100g
```

### Step 2: Install MinIO (Object Storage)

```bash
# Add MinIO Helm repo
helm repo add minio https://charts.min.io/
helm repo update

# Install MinIO in standalone mode (for development)
kubectl create namespace datalake
helm install minio minio/minio \
  --namespace datalake \
  --set mode=standalone \
  --set replicas=1 \
  --set persistence.size=50Gi \
  --set resources.requests.memory=2Gi \
  --set rootUser=admin \
  --set rootPassword=minio123

# Port-forward to access UI
kubectl port-forward -n datalake svc/minio 9000:9000 9001:9001
# Access UI at http://localhost:9001
```

Create buckets:
```bash
# Install mc (MinIO Client)
brew install minio/stable/mc

# Configure alias
mc alias set local http://localhost:9000 admin minio123

# Create buckets
mc mb local/bronze
mc mb local/silver
mc mb local/gold
mc mb local/warehouse
```

### Step 3: Install PostgreSQL (Metadata Store)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql bitnami/postgresql \
  --namespace datalake \
  --set auth.postgresPassword=postgres123 \
  --set primary.persistence.size=10Gi

# Get password
export POSTGRES_PASSWORD=$(kubectl get secret --namespace datalake postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)

# Port-forward
kubectl port-forward -n datalake svc/postgresql 5432:5432

# Connect
psql -h localhost -U postgres -d postgres
```

Create database:
```sql
CREATE DATABASE datalake_metadata;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```

### Step 4: Install Kafka (Event Streaming)

```bash
# Install Strimzi operator
kubectl create namespace kafka
kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Wait for operator to be ready
kubectl wait --for=condition=ready pod -l name=strimzi-cluster-operator -n kafka --timeout=300s

# Deploy Kafka cluster (minimal for dev)
cat <<EOF | kubectl apply -f -
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: datalake-kafka
  namespace: kafka
spec:
  kafka:
    version: 3.6.0
    replicas: 1
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
    storage:
      type: ephemeral
  zookeeper:
    replicas: 1
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
EOF

# Wait for Kafka to be ready
kubectl wait kafka/datalake-kafka --for=condition=Ready --timeout=300s -n kafka

# Port-forward
kubectl port-forward -n kafka svc/datalake-kafka-kafka-bootstrap 9092:9092
```

### Step 5: Install Spark Operator

```bash
# Install Spark operator
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator
helm install spark-operator spark-operator/spark-operator \
  --namespace datalake \
  --set webhook.enable=true

# Build custom Spark image with dependencies
cat > Dockerfile.spark <<EOF
FROM apache/spark:3.5.0

USER root

# Install Python dependencies
RUN pip install --no-cache-dir \
    pyspark==3.5.0 \
    pyiceberg[s3fs,glue]==0.5.1 \
    pandas \
    numpy \
    boto3

USER spark
EOF

docker build -f Dockerfile.spark -t datalake/spark:3.5.0 .
```

### Step 6: Install Trino (Query Engine)

```bash
helm repo add trino https://trinodb.github.io/charts
helm install trino trino/trino \
  --namespace datalake \
  --set server.workers=2 \
  --set server.coordinator.resources.memory=4Gi \
  --set server.worker.resources.memory=4Gi

# Port-forward
kubectl port-forward -n datalake svc/trino 8080:8080
# Access UI at http://localhost:8080
```

Configure Iceberg catalog:
```bash
kubectl exec -it -n datalake trino-coordinator-0 -- bash
cat > /etc/trino/catalog/iceberg.properties <<EOF
connector.name=iceberg
iceberg.catalog.type=hadoop
iceberg.catalog.warehouse=s3a://warehouse/
s3.endpoint=http://minio:9000
s3.path-style-access=true
s3.access-key-id=admin
s3.secret-access-key=minio123
EOF
```

### Step 7: Set Up Python Development Environment

```bash
cd /Users/richardmunyemana/Desktop/projects/businessResearch/datalake

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
cat > requirements.txt <<EOF
# Data processing
pyspark==3.5.0
pandas>=2.0.0
numpy>=1.24.0
pyarrow>=12.0.0

# Data lake
pyiceberg[s3fs,glue]==0.5.1
delta-spark==3.0.0

# Storage
boto3>=1.28.0
minio>=7.1.0

# Databases
psycopg2-binary>=2.9.0
sqlalchemy>=2.0.0

# Kafka
confluent-kafka>=2.2.0

# API
fastapi>=0.100.0
uvicorn[standard]>=0.23.0
pydantic>=2.0.0

# Data quality
great-expectations>=0.17.0

# Utilities
python-dotenv>=1.0.0
pyyaml>=6.0
click>=8.1.0

# Development
pytest>=7.4.0
black>=23.7.0
flake8>=6.1.0
mypy>=1.5.0
EOF

pip install -r requirements.txt
```

### Step 8: Configure Environment

```bash
# Create .env file
cat > .env <<EOF
# MinIO
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=minio123
MINIO_SECURE=false

# PostgreSQL
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_DB=datalake_metadata
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres123

# Kafka
KAFKA_BOOTSTRAP_SERVERS=localhost:9092

# Trino
TRINO_HOST=localhost
TRINO_PORT=8080
TRINO_CATALOG=iceberg
TRINO_SCHEMA=default
EOF
```

### Step 9: Initialize Data Lake Structure

```bash
# Create initial directories in MinIO
python3 <<EOF
from minio import Minio

client = Minio(
    "localhost:9000",
    access_key="admin",
    secret_key="minio123",
    secure=False
)

# Create bucket structure
buckets = ['bronze', 'silver', 'gold', 'warehouse']
for bucket in buckets:
    if not client.bucket_exists(bucket):
        client.make_bucket(bucket)
        print(f"Created bucket: {bucket}")

print("Data lake structure initialized!")
EOF
```

### Step 10: Run Sample Data Pipeline

```bash
# Create sample data generator
cat > src/datalake/samples/generate_sample_data.py <<EOF
import json
from datetime import datetime
from confluent_kafka import Producer
import random
import time

# Kafka producer config
conf = {
    'bootstrap.servers': 'localhost:9092',
    'client.id': 'sample-producer'
}

producer = Producer(conf)

# Sample data generator
def generate_event():
    return {
        'event_id': f"evt_{random.randint(1000000, 9999999)}",
        'timestamp': datetime.utcnow().isoformat(),
        'user_id': f"user_{random.randint(1, 10000)}",
        'event_type': random.choice(['page_view', 'click', 'purchase', 'signup']),
        'properties': {
            'page': random.choice(['/home', '/products', '/cart', '/checkout']),
            'device': random.choice(['mobile', 'desktop', 'tablet']),
            'browser': random.choice(['chrome', 'firefox', 'safari', 'edge'])
        }
    }

# Send events
print("Generating sample events...")
for i in range(100):
    event = generate_event()
    producer.produce('events', json.dumps(event).encode('utf-8'))
    if i % 10 == 0:
        print(f"Sent {i} events")
        producer.flush()
    time.sleep(0.1)

producer.flush()
print("Sample data generation complete!")
EOF

# Run generator
python3 src/datalake/samples/generate_sample_data.py
```

### Step 11: Create Spark Job to Process Data

```bash
cat > jobs/kafka_to_bronze.py <<EOF
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *

# Create Spark session
spark = SparkSession.builder \
    .appName("Kafka to Bronze") \
    .config("spark.sql.extensions", "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions") \
    .config("spark.sql.catalog.iceberg", "org.apache.iceberg.spark.SparkCatalog") \
    .config("spark.sql.catalog.iceberg.type", "hadoop") \
    .config("spark.sql.catalog.iceberg.warehouse", "s3a://warehouse/") \
    .config("spark.hadoop.fs.s3a.endpoint", "http://minio:9000") \
    .config("spark.hadoop.fs.s3a.access.key", "admin") \
    .config("spark.hadoop.fs.s3a.secret.key", "minio123") \
    .config("spark.hadoop.fs.s3a.path.style.access", "true") \
    .config("spark.hadoop.fs.s3a.impl", "org.apache.hadoop.fs.s3a.S3AFileSystem") \
    .getOrCreate()

# Define schema
schema = StructType([
    StructField("event_id", StringType(), True),
    StructField("timestamp", TimestampType(), True),
    StructField("user_id", StringType(), True),
    StructField("event_type", StringType(), True),
    StructField("properties", MapType(StringType(), StringType()), True)
])

# Read from Kafka
df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "datalake-kafka-kafka-bootstrap.kafka:9092") \
    .option("subscribe", "events") \
    .option("startingOffsets", "earliest") \
    .load()

# Parse JSON
parsed = df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

# Add metadata
enriched = parsed \
    .withColumn("ingestion_time", current_timestamp()) \
    .withColumn("date", to_date(col("timestamp")))

# Write to Bronze (Iceberg)
query = enriched \
    .writeStream \
    .format("iceberg") \
    .outputMode("append") \
    .option("path", "iceberg.bronze.events") \
    .option("checkpointLocation", "s3a://warehouse/checkpoints/bronze_events") \
    .partitionBy("date", "event_type") \
    .start()

query.awaitTermination()
EOF
```

### Step 12: Query Data with Trino

```bash
# Install Trino CLI
curl -o trino https://repo1.maven.org/maven2/io/trino/trino-cli/427/trino-cli-427-executable.jar
chmod +x trino
./trino --server http://localhost:8080

# Run queries
SELECT * FROM iceberg.bronze.events LIMIT 10;
SELECT event_type, COUNT(*) as count FROM iceberg.bronze.events GROUP BY event_type;
```

---

## Sample API Service

Create a simple FastAPI service:

```bash
cat > src/datalake/api/main.py <<EOF
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import trino
import os
from typing import List, Dict, Any

app = FastAPI(title="Data Lake API", version="1.0.0")

# Trino connection
def get_trino_connection():
    return trino.dbapi.connect(
        host=os.getenv('TRINO_HOST', 'localhost'),
        port=int(os.getenv('TRINO_PORT', 8080)),
        catalog='iceberg',
        schema='bronze'
    )

class QueryRequest(BaseModel):
    sql: str
    limit: int = 100

class QueryResponse(BaseModel):
    columns: List[str]
    data: List[List[Any]]
    row_count: int

@app.get("/")
async def root():
    return {
        "service": "Data Lake API",
        "version": "1.0.0",
        "status": "healthy"
    }

@app.get("/health")
async def health_check():
    try:
        conn = get_trino_connection()
        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.fetchone()
        return {"status": "healthy", "database": "connected"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Database unhealthy: {str(e)}")

@app.post("/query", response_model=QueryResponse)
async def execute_query(request: QueryRequest):
    try:
        conn = get_trino_connection()
        cursor = conn.cursor()

        # Add LIMIT if not present
        sql = request.sql
        if "LIMIT" not in sql.upper():
            sql += f" LIMIT {request.limit}"

        cursor.execute(sql)
        columns = [desc[0] for desc in cursor.description]
        rows = cursor.fetchall()

        return QueryResponse(
            columns=columns,
            data=rows,
            row_count=len(rows)
        )
    except Exception as e:
        raise HTTPException(status_code=400, detail=f"Query failed: {str(e)}")

@app.get("/datasets")
async def list_datasets():
    try:
        conn = get_trino_connection()
        cursor = conn.cursor()
        cursor.execute("SHOW TABLES")
        tables = [row[0] for row in cursor.fetchall()]
        return {"datasets": tables, "count": len(tables)}
    except Exception as e:
        raise HTTPException(status_code=500, detail=f"Failed to list datasets: {str(e)}")

@app.get("/datasets/{table_name}/schema")
async def get_schema(table_name: str):
    try:
        conn = get_trino_connection()
        cursor = conn.cursor()
        cursor.execute(f"DESCRIBE {table_name}")
        schema = [
            {"column": row[0], "type": row[1]}
            for row in cursor.fetchall()
        ]
        return {"table": table_name, "schema": schema}
    except Exception as e:
        raise HTTPException(status_code=404, detail=f"Table not found: {str(e)}")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF

# Run API
cd src/datalake/api
uvicorn main:app --reload

# Access docs at http://localhost:8000/docs
```

---

## Testing the Setup

### 1. Test MinIO
```bash
mc ls local/bronze/
```

### 2. Test Kafka
```bash
kubectl run kafka-producer -ti --image=quay.io/strimzi/kafka:latest-kafka-3.6.0 --rm=true --restart=Never -n kafka -- bin/kafka-console-producer.sh --bootstrap-server datalake-kafka-kafka-bootstrap:9092 --topic events
```

### 3. Test Trino
```bash
curl http://localhost:8080/v1/statement -X POST -H "X-Trino-User: admin" -d "SELECT 1"
```

### 4. Test API
```bash
curl http://localhost:8000/health
curl http://localhost:8000/datasets
```

---

## Monitoring

### View Kubernetes Resources
```bash
kubectl get all -n datalake
kubectl get all -n kafka
```

### View Logs
```bash
# MinIO
kubectl logs -n datalake -l app=minio

# Trino
kubectl logs -n datalake -l app=trino

# Spark jobs
kubectl get sparkapplications -n datalake
kubectl logs -n datalake <spark-driver-pod>
```

### Resource Usage
```bash
kubectl top nodes
kubectl top pods -n datalake
```

---

## Cleanup

```bash
# Delete all resources
helm uninstall minio -n datalake
helm uninstall postgresql -n datalake
helm uninstall trino -n datalake
helm uninstall spark-operator -n datalake
kubectl delete kafka datalake-kafka -n kafka
kubectl delete namespace datalake
kubectl delete namespace kafka

# Stop minikube (if using)
minikube stop
```

---

## Next Steps

1. Read [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) for full architecture
2. Review [TECHNOLOGY_STACK.md](TECHNOLOGY_STACK.md) for component details
3. Implement Bronze → Silver → Gold transformations
4. Add data quality checks with Great Expectations
5. Set up Airflow for orchestration
6. Deploy monitoring with Prometheus + Grafana
7. Scale to production with multi-node clusters

---

## Troubleshooting

### MinIO connection issues
```bash
# Check service
kubectl get svc -n datalake minio

# Check logs
kubectl logs -n datalake -l app=minio
```

### Kafka not ready
```bash
# Check operator
kubectl get pods -n kafka

# Check Kafka resource
kubectl get kafka -n kafka
kubectl describe kafka datalake-kafka -n kafka
```

### Trino catalog errors
```bash
# Restart Trino
kubectl rollout restart statefulset/trino-coordinator -n datalake
kubectl rollout restart statefulset/trino-worker -n datalake
```

### Out of memory
```bash
# Increase Docker Desktop memory (16GB+)
# Or reduce replicas
helm upgrade minio minio/minio --reuse-values --set resources.requests.memory=1Gi
```

---

**For questions or issues, check the logs and Kubernetes events:**
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```
