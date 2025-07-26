# WikiSentry

**Real-time ML platform for analyzing Wikipedia edit streams with streaming classification and automated model retraining**

WikiSentry demonstrates modern data engineering and MLOps practices by processing the live Wikimedia edit feed through a cloud-native architecture deployed locally.

## Prerequisites

This section assumes you are on macOS.

### Docker

> **Note**: If you already have Docker installed on your system, you can skip this step.

I recommend installing [OrbStack](https://orbstack.dev) for its lightweight runtime. Since the project involves resource-intensive tools like Kafka, Flink and Spark, every bit of efficiency will go a long way.

```bash
brew install --cask orbstack

orb  # Select Docker when a menu opens.
```

**Alternative**: You can use Docker Desktop if you prefer the official distribution, but OrbStack typically uses fewer system resources.

**Verify installation:**
```bash
docker --version
docker compose version
```

### Kubernetes

Install k3d -- a minimal Kubernetes distribution that will allow you to spin up a lightweight cluster locally -- and kubectl -- a CLI for managing Kubernetes clusters.

```bash
brew install k3d kubectl
```

Additionally, I recommend to install `k9s` -- a terminal-based UI to interact with Kubernetes clusters that most of the time is more convenient than `kubectl`.

```bash
brew install k9s
```

**Verify installation:**
```bash
k3d version
kubectl version --client
k9s version
```

### MinIO CLI (Optional)

MinIO is an S3-compatible object storage system we'll use for data lake functionality. To interact with it manually for testing purposes, install the MinIO CLI:

```bash
brew install minio-mc
```

#### Configure MinIO CLI

Run the following to configure MinIO CLI credentials. Make sure the values match whatever is configured in `compose.yaml`.

```bash
mc alias set local http://localhost:9000 minioadmin minioadmin
```

#### Test - create a bucket and upload a file

Create a test bucket and upload a file to it.

```bash
mc mb local/test-bucket
echo "Hello MinIO" > test.txt
mc cp test.txt local/test-bucket/
mc ls local/test-bucket/
rm test.txt
```

## Quick Start

TODO: Add step-by-step deployment commands

## Overview

WikiSentry provides:

- **Real-time edit classification** using Apache Flink and ML models
- **S3-compatible data lake** with MinIO for historical data persistence  
- **Automated analytics** via Apache Spark and Airflow orchestration
- **MLOps pipeline** with continuous model retraining and deployment

## Architecture

The platform uses a two-layer architecture separating persistent data services from ephemeral compute workloads:

### Data Layer (Docker Compose)
- **MinIO**: S3-compatible object storage for data lake
- **Apache Kafka**: Event streaming platform (KRaft mode, no Zookeeper)
- **Kafka Connect**: Ingestion connectors for external data sources
- Provides durable storage and messaging that persists across application restarts

### Application Layer (Kubernetes)
- **Apache Flink**: Stream processing for real-time edit classification
- **Apache Spark**: Batch processing for analytics and model training
- **Apache Airflow**: Workflow orchestration for ETL and MLOps pipelines
- **PostgreSQL**: Airflow metadata store
- **FastAPI**: Model serving endpoints for inference

### Data Flows

#### Stream Processing Pipeline
1. Kafka Connect ingests Wikimedia EventStreams API into Kafka topics
2. Events routed by user type (registered/anonymous users)
3. Flink jobs process streams and invoke ML models for classification
4. Classified events archived to MinIO; alerts triggered for anomalies

#### Batch Processing Pipeline
1. Daily Airflow DAG processes historical data with Spark
2. Model retraining DAG periodically updates models using full dataset
3. New model versions automatically deployed to serving layer

## Technology Stack

| Component | Purpose | Deployment |
|-----------|---------|------------|
| **Docker Compose** | Container orchestration for data services | Local host |
| **MinIO** | S3-compatible object storage (data lake) | Docker container |
| **Apache Kafka** | Event streaming platform (KRaft mode) | Docker container |
| **Kafka Connect** | Data ingestion connectors | Docker container |
| **k3d/K3s** | Lightweight Kubernetes distribution | Local cluster |
| **Apache Flink** | Stream processing framework | Kubernetes |
| **Apache Spark** | Unified analytics engine | Kubernetes |
| **Apache Airflow** | Workflow orchestration platform | Kubernetes |
| **PostgreSQL** | Relational database (Airflow metadata) | Kubernetes |
| **FastAPI** | Modern Python web framework for APIs | Kubernetes |

## Prerequisites

- **Hardware**: 32GB+ RAM recommended for full stack
- **Software**: Docker, kubectl, k3d
- **Network**: Internet access for Wikimedia EventStreams API

## Quick Start

### 1. Start Data Layer
```bash
# Start persistent storage and messaging
docker compose up -d
```
Services available:
- **MinIO Console**: `http://localhost:9001` 
- **MinIO API**: `localhost:9000`
- **Kafka**: `localhost:9092`

### 2. Create Kubernetes Cluster
```bash
# Create local k3d cluster
k3d cluster create wikisentry-dev --port "8080:80@loadbalancer"
```

### 3. Deploy Application Stack
```bash
# Add Helm repositories
helm repo add apache-airflow https://airflow.apache.org
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator

# Deploy processing applications
helm install airflow apache-airflow/airflow -f k8s/airflow/values.yaml
helm install spark spark-operator/spark-operator -f k8s/spark/values.yaml

# Deploy Flink and model APIs
kubectl apply -f k8s/flink/
kubectl apply -f k8s/models/
```

### Accessing Services

- **Airflow UI**: `http://localhost:8080`
- **MinIO Console**: `http://localhost:9001`
- **Flink Dashboard**: `http://localhost:8081`

### Network Configuration

- **Inter-cluster communication**: Kubernetes DNS (`service.namespace.svc.cluster.local`)
- **Cluster-to-host**: Docker gateway (`host.docker.internal:9092` for Kafka, `:9000` for MinIO)
- **External access**: k3d load balancer forwards to `localhost`
