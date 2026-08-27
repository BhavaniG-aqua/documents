# Unified Propensity Score — Complete Migration Process (GCP to AWS)

## Table of Contents

1. [Migration Overview](#1-migration-overview)
2. [Architecture Before and After](#2-architecture-before-and-after)
3. [What Changed and What Stayed the Same](#3-what-changed-and-what-stayed-the-same)
4. [Data Migration — GCS to S3](#4-data-migration--gcs-to-s3)
5. [Code Changes — Storage Layer](#5-code-changes--storage-layer)
6. [Code Changes — API Routes](#6-code-changes--api-routes)
7. [Code Changes — Dependencies](#7-code-changes--dependencies)
8. [Container Architecture](#8-container-architecture)
9. [Deployment Pipeline — How deploy.py Works](#9-deployment-pipeline--how-deploypy-works)
10. [MLflow Model Governance](#10-mlflow-model-governance)
11. [Infrastructure Mapping (GCP to AWS)](#11-infrastructure-mapping-gcp-to-aws)
12. [Authentication and IAM](#12-authentication-and-iam)
13. [The 3-Tier Cache System](#13-the-3-tier-cache-system)
14. [Feature Engineering at Inference Time](#14-feature-engineering-at-inference-time)
15. [Request and Response Format](#15-request-and-response-format)
16. [Autoscaling](#16-autoscaling)
17. [Testing Strategy](#17-testing-strategy)
18. [Execution Steps on EC2](#18-execution-steps-on-ec2)
19. [Rollback Procedures](#19-rollback-procedures)
20. [Troubleshooting](#20-troubleshooting)

---

## 1. Migration Overview

### What Is Being Migrated?

A real-time ML inference service that scores email subscribers on their likelihood to open, click, and purchase. The service was running on Google Cloud Platform (Vertex AI) and is being migrated to AWS (SageMaker).

### Migration Principle

**Lift-and-shift with minimal changes.** The business logic, ML models, feature engineering, caching architecture, and container design are identical. Only the cloud-specific plumbing was replaced:

- Google Cloud Storage (GCS) → Amazon S3
- Vertex AI → SageMaker
- Artifact Registry → Elastic Container Registry (ECR)
- `google-cloud-storage` Python SDK → `boto3`
- GCP IAM service accounts → AWS IAM roles

### What Triggers a Score?

```
Downstream system (ESP, CDP, or batch job)
    │
    ├── Real-time: POST /invocations with JSON body
    │       → Returns decile scores in ~50ms
    │
    └── Batch: POST /invocations with JSON Lines body
            → Returns one score per line
```

---

## 2. Architecture Before and After

### BEFORE (GCP)

```
┌───────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  API Consumer │────▶│  Vertex AI       │────▶│  Container           │
│  (ESP/CDP)    │     │  Endpoint        │     │  (Artifact Registry) │
└───────────────┘     └──────────────────┘     └──────────────────────┘
                                                        │
                                                        ▼
                                               ┌──────────────────┐
                                               │  GCS Bucket      │
                                               │  gs://tie-data.. │
                                               └──────────────────┘
```

### AFTER (AWS)

```
┌───────────────┐     ┌──────────────────┐     ┌──────────────────────┐
│  API Consumer │────▶│  SageMaker       │────▶│  Container           │
│  (ESP/CDP)    │     │  Endpoint        │     │  (ECR)               │
└───────────────┘     └──────────────────┘     └──────────────────────┘
                                                        │
                                                        ▼
                                               ┌──────────────────┐
                                               │  S3 Bucket       │
                                               │  s3://tie-data.. │
                                               └──────────────────┘
```

### Inside the Container (UNCHANGED)

```
┌─────────────────────────────────────────────────────────┐
│  Docker Container (running on SageMaker instance)       │
│                                                         │
│  supervisord                                            │
│    ├── redis-server (core 0) ← L2 cache                │
│    └── gunicorn (cores 1-7) ← 7 FastAPI workers        │
│              │                                          │
│              ├── L1 in-memory LRU cache (per worker)    │
│              ├── L2 Redis cache (shared across workers) │
│              └── L3 S3 bucket (source of truth)         │
│                                                         │
│  Port 8080 exposed                                      │
│    GET  /ping         → health check                    │
│    POST /invocations  → scoring endpoint                │
└─────────────────────────────────────────────────────────┘
```

---

## 3. What Changed and What Stayed the Same

### CHANGED (cloud plumbing only)

| Component | GCP Version | AWS Version |
|-----------|-------------|-------------|
| Object storage SDK | `google-cloud-storage` | `boto3` |
| ML platform SDK | `google-cloud-aiplatform` | `sagemaker` |
| Storage URI prefix | `gs://` | `s3://` |
| Health endpoint | `/health` | `/ping` |
| Predict endpoint | `/predict` | `/invocations` |
| Container registry | Artifact Registry | ECR |
| Container build | `gcloud builds submit` | `docker build` + `docker push` |
| Auth mechanism | Service account + ADC | IAM role (auto from instance metadata) |
| Env var for project | `GOOGLE_CLOUD_PROJECT` | Not needed (boto3 uses IAM role) |
| Fan-out config name | `GCS_FANOUT_WORKERS` | `S3_FANOUT_WORKERS` |

### UNCHANGED (all business logic)

| Component | Details |
|-----------|---------|
| ML models | XGBoost `.ubj` format, 3 models per client |
| Scoring logic | Predict → searchsorted into decile cutpoints → decile 1-10 |
| Feature engineering | Age computation, carrier bucketing, dwelling/homeowner normalization |
| Caching | 3-tier: L1 LRU (memory) → L2 Redis → L3 object storage |
| Container process model | supervisord → Redis + Gunicorn (UvicornWorker) |
| CPU pinning | Core 0 = Redis, Cores 1-N = Gunicorn workers |
| Request/response schema | Pydantic models (PredictionRequest/PredictionResponse) |
| Multi-tenancy | `client_name` parameter routes to per-client models |
| Distributed locking | Redis lock prevents parallel S3 downloads for same client |
| Error handling | All-or-nothing: all 3 models required or 404 |

---

## 4. Data Migration — GCS to S3

### What Was Migrated

Model artifacts stored in GCS were copied to S3 with **identical folder structure**:

```
Source: gs://tie-datascience/propensity-score/dev/app_id=carawayhome/assets/
Dest:   s3://tie-datascience/propensity-score/dev/app_id=carawayhome/assets/
```

### Folder Structure (same in both clouds)

```
propensity-score/dev/
└── app_id={client_name}/
    └── assets/
        ├── open-score/
        │   └── latest/
        │       ├── model.ubj              ← XGBoost binary model
        │       ├── decile_cutpoints.json  ← 9 threshold values
        │       ├── category_mappings.json ← categorical feature categories
        │       └── discrimination_metrics.json ← model quality metrics
        ├── click-score/
        │   └── latest/
        │       ├── model.ubj
        │       ├── decile_cutpoints.json
        │       ├── category_mappings.json
        │       └── discrimination_metrics.json
        └── purchase-score/
            └── latest/
                ├── model.ubj
                ├── decile_cutpoints.json
                ├── category_mappings.json
                └── discrimination_metrics.json
```

### How the Transfer Was Done

The `transfer_gcs_to_s3.py` script handles this:

```python
# It does:
# 1. Lists all objects under the GCS prefix
# 2. For each object: downloads to memory, uploads to S3
# 3. Runs in parallel (8 threads by default)

python transfer_gcs_to_s3.py              # actual transfer
python transfer_gcs_to_s3.py --dry-run    # preview only
python transfer_gcs_to_s3.py --workers 16 # more parallelism
```

**Prerequisites for running the transfer:**
- GCS auth: `GOOGLE_APPLICATION_CREDENTIALS` env var or `gcloud auth application-default login`
- S3 auth: AWS credentials configured (env vars, `~/.aws/credentials`, or IAM role)
- Both `google-cloud-storage` and `boto3` pip packages installed

### What Each File Contains

| File | Format | Content | Size |
|------|--------|---------|------|
| `model.ubj` | XGBoost binary | Trained gradient-boosted tree model | ~10-50 MB |
| `decile_cutpoints.json` | JSON array | 9 float values that divide predictions into 10 deciles | ~200 bytes |
| `category_mappings.json` | JSON dict | Maps categorical column names to their valid categories | ~2-5 KB |
| `discrimination_metrics.json` | JSON dict | AUC, KS, Gini metrics (for monitoring, not inference) | ~500 bytes |

---

## 5. Code Changes — Storage Layer

### model_manager.py — The Core Change

This is where the bulk of the GCP→AWS code changes live. The `ModelManager` class handles all object storage interactions.

#### Constructor — How Storage Is Initialized

**GCP (before):**
```python
from google.cloud import storage

if not storage_uri.startswith("gs://"):
    raise ValueError(f"Invalid MODEL_STORAGE_URI: {storage_uri!r}")

bucket_name, _, prefix = storage_uri[len("gs://"):].partition("/")
self._storage = storage_client or storage.Client()
self._bucket = self._storage.bucket(bucket_name)  # Bucket OBJECT
self._prefix = prefix.strip("/")
```

**AWS (after):**
```python
import boto3

if not storage_uri.startswith("s3://"):
    raise ValueError(f"Invalid MODEL_STORAGE_URI: {storage_uri!r}")

bucket_name, _, prefix = storage_uri[len("s3://"):].partition("/")
self._s3 = s3_client or boto3.client("s3")
self._bucket = bucket_name  # Just a STRING
self._prefix = prefix.strip("/")
```

**Key concept:** In GCS, you work with a `Bucket` object that has methods like `.list_blobs()` and `.blob()`. In S3/boto3, you work with a client object and pass the bucket name as a parameter to every API call.

#### Listing Objects — Checking What Models Exist

**GCP (before):**
```python
blobs = list(self._bucket.list_blobs(prefix=prefix))
by_suffix = {}
for b in blobs:
    tail = b.name.rsplit("/", 1)[-1]
    by_suffix[tail] = b

model_blob = by_suffix.get("model.ubj")
timestamp = model_blob.updated.timestamp()
```

**AWS (after):**
```python
response = self._s3.list_objects_v2(Bucket=self._bucket, Prefix=prefix)
contents = response.get("Contents", [])
by_suffix = {}
for obj in contents:
    tail = obj["Key"].rsplit("/", 1)[-1]
    by_suffix[tail] = obj

model_obj = by_suffix.get("model.ubj")
timestamp = model_obj["LastModified"].timestamp()
```

**Why this matters:** The code checks S3 to see if all 3 model types (open/click/purchase) have their `latest/` folder populated. If any is missing, the system refuses to serve that client (all-or-nothing rule).

#### Downloading Objects — Getting Model Bytes

**GCP (before):**
```python
model_bytes = self._bucket.blob(paths.model).download_as_bytes()
cutpoints = json.loads(
    self._bucket.blob(paths.cutpoints).download_as_bytes().decode("utf-8")
)
```

**AWS (after):**
```python
model_bytes = self._s3.get_object(Bucket=self._bucket, Key=paths.model)["Body"].read()
cutpoints = json.loads(
    self._s3.get_object(Bucket=self._bucket, Key=paths.cutpoints)["Body"].read().decode("utf-8")
)
```

**What's happening:** When the cache is cold (first request for a client, or after TTL expiry), the system downloads the model binary and metadata JSONs from S3. This takes 2-5 seconds. Subsequent requests hit the cache.

#### Error Handling

**GCP:** `from google.api_core import exceptions as gcs_exc` → catch `gcs_exc.GoogleAPIError`

**AWS:** `from botocore.exceptions import ClientError` → catch `ClientError`

Same error-handling pattern, different exception classes.

---

## 6. Code Changes — API Routes

### config.py — Route Definitions

**GCP (before):**
```python
AIP_HEALTH_ROUTE = os.environ.get("AIP_HEALTH_ROUTE", "/health")
AIP_PREDICT_ROUTE = os.environ.get("AIP_PREDICT_ROUTE", "/predict")
```

**AWS (after):**
```python
AIP_HEALTH_ROUTE = os.environ.get("AIP_HEALTH_ROUTE", "/ping")
AIP_PREDICT_ROUTE = os.environ.get("AIP_PREDICT_ROUTE", "/invocations")
```

### Why Different Routes?

SageMaker has a hard contract for custom inference containers:
- **`GET /ping`** — SageMaker calls this every 30 seconds to check container health. Must return HTTP 200.
- **`POST /invocations`** — All prediction requests go here. SageMaker routes client traffic to this path.

Vertex AI used `/health` and `/predict` for the same purposes. The variable names (`AIP_HEALTH_ROUTE`, `AIP_PREDICT_ROUTE`) were kept from the GCP version even though "AIP" stands for "AI Platform" (Google's old name). This is intentional — it avoids renaming variables across the codebase for no functional benefit.

---

## 7. Code Changes — Dependencies

### requirements.txt Changes

**Removed (GCP-specific):**
```
google-cloud-storage==3.4.1
google-cloud-aiplatform==1.133.0
grpcio-status==1.75.1
```

**Added (AWS-specific):**
```
boto3>=1.35.0
```

**Kept (framework/ML):**
```
fastapi==0.118.0
gunicorn==23.0.0
uvicorn==0.34.0
xgboost>=3.1.2
redis==6.4.0
cachetools==6.2.1
pandas==2.3.3
numpy>=1.26
pydantic (via fastapi)
```

### Why boto3 Replaces Multiple GCP Packages

On GCP, you need separate packages for storage (`google-cloud-storage`) and ML platform (`google-cloud-aiplatform`), plus gRPC dependencies. On AWS, `boto3` is the single SDK that covers S3, SageMaker, IAM, and all other AWS services. This significantly reduces dependency complexity.

---

## 8. Container Architecture

### Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1
ENV SUPERVISOR_PORT 8080

# Install Redis and process supervisor
RUN apt-get update && apt-get install -y --no-install-recommends \
    redis-server \
    supervisor \
    procps \
    && rm -rf /var/lib/apt/lists/*

# Copy application code
COPY ./src /app
COPY ./requirements.txt .
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY start.sh /start.sh
RUN chmod +x /start.sh

# Install Python dependencies
RUN pip install --upgrade pip>=26.0
RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8080

ENTRYPOINT ["/start.sh"]
```

### What Happens When the Container Starts

```
1. SageMaker pulls image from ECR
2. Runs ENTRYPOINT → /start.sh
3. start.sh detects CPU count (nproc)
4. Assigns core 0 to Redis, cores 1-N to Gunicorn
5. Launches supervisord
6. supervisord starts:
   ├── redis-server (pinned to core 0 via taskset)
   └── gunicorn with N-1 UvicornWorker processes (pinned to cores 1+)
7. Gunicorn binds to 0.0.0.0:8080
8. SageMaker health checks GET /ping
9. Once /ping returns 200 → endpoint goes "InService"
```

### start.sh — CPU Core Pinning Logic

```bash
TOTAL_CORES=$(nproc)           # e.g., 8 on ml.m5.2xlarge

if [ "$TOTAL_CORES" -le 1 ]; then
  REDIS_CORES=0
  GUNICORN_CORES=0
  GUNICORN_WORKERS=1
else
  REDIS_CORES=0                           # Redis gets core 0
  GUNICORN_CORES="1-$(($TOTAL_CORES-1))"  # Workers get cores 1-7
  GUNICORN_WORKERS=$(($TOTAL_CORES-1))    # 7 workers
fi
```

**Why CPU pinning?** Redis is single-threaded and latency-sensitive. Pinning it to its own core prevents context-switching overhead. Gunicorn workers are CPU-bound during XGBoost prediction, so giving them dedicated cores maximizes throughput.

### supervisord.conf — Process Management

```ini
[program:redis]
command=taskset -c %(ENV_REDIS_CORES)s redis-server

[program:gunicorn]
command=taskset -c %(ENV_GUNICORN_CORES)s gunicorn \
    --workers %(ENV_GUNICORN_WORKERS)s \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:%(ENV_SUPERVISOR_PORT)s \
    main:app
```

**Why Redis inside the container (not ElastiCache)?** Each SageMaker instance gets its own Redis. This avoids network latency to an external cache, simplifies deployment (no VPC/subnet config), and provides per-instance L2 caching. When autoscaling adds instances, each new instance has its own fresh Redis that warms up on first requests.

---

## 9. Deployment Pipeline — How deploy.py Works

### Overview

`deploy.py` is the single command that handles everything from model registration to live endpoint. It uses CLI flags to control which steps run:

```
python deploy.py --register        # Step 1: Register model in MLflow
python deploy.py --approve         # Step 2: Approve model for deployment
python deploy.py --build-container # Step 3: Build Docker image, push to ECR
python deploy.py --deploy-model    # Step 4: Create SageMaker Model resource
python deploy.py --deploy-endpoint # Step 5: Deploy to live endpoint
```

### Full Pipeline Diagram

```
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: --register                                                  │
│                                                                    │
│ Records model artifacts (already in S3) into MLflow Model Registry │
│ Status: "Staging" (not yet approved for deployment)                │
│                                                                    │
│ Creates: MLflow model version for "unified-propensity-{client}"    │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 2: --approve                                                   │
│                                                                    │
│ Transitions model from "Staging" → "Production" in MLflow          │
│ This is the GOVERNANCE GATE — human must explicitly approve        │
│                                                                    │
│ Archives any previous Production version automatically             │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 3: --build-container                                           │
│                                                                    │
│ 1. Creates ECR repository (if not exists)                          │
│ 2. Authenticates Docker to ECR                                     │
│ 3. docker build -t <image_uri> .                                   │
│ 4. Tags with version (propensity-unified-score-v1, v2, v3...)      │
│ 5. docker push (versioned tag + latest tag)                        │
│ 6. Registers image version in MLflow (optional tracking)           │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 4: --deploy-model                                              │
│                                                                    │
│ 1. Queries MLflow for the approved (Production) model version      │
│ 2. REFUSES to proceed if no approved version exists                │
│ 3. Creates SageMaker Model resource:                               │
│    - Links ECR image URI                                           │
│    - Sets MODEL_STORAGE_URI env var (S3 path to models)            │
│    - Attaches IAM execution role                                   │
│ 4. Logs deployment event to MLflow (audit trail)                   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                                 ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 5: --deploy-endpoint                                           │
│                                                                    │
│ 1. Creates Endpoint Configuration (model + instance type + count)  │
│ 2. Creates or Updates SageMaker Endpoint                           │
│ 3. Waits for status "InService" (~5-10 minutes)                    │
│ 4. Configures autoscaling (if max_instances > min_instances)       │
│                                                                    │
│ Result: Live HTTPS endpoint accepting prediction requests          │
└────────────────────────────────────────────────────────────────────┘
```

### ECR Image Versioning

The deploy script auto-increments container versions:

```
propensity-unified-score-v1  ← first build
propensity-unified-score-v2  ← second build
propensity-unified-score-v3  ← third build
latest                       ← always points to most recent
```

It queries ECR for existing tags, finds the highest version number, and increments. This enables easy rollback to any previous container version.

### SageMaker Model Naming

```
propensity-unified-score-model-v{mlflow_version}-image-v{ecr_version}
```

Example: `propensity-unified-score-model-v3-image-v5` means MLflow model version 3 deployed on Docker image version 5. This naming lets you trace exactly which model and which code are running.

---

## 10. MLflow Model Governance

### Why Governance?

Without governance, anyone could deploy any model at any time. MLflow Model Registry adds:

1. **Stage gates** — Models must be explicitly approved before deployment
2. **Version history** — Every model version is tracked with metadata
3. **Audit trail** — Deployment events are logged with timestamps and user info
4. **Rollback** — Revert to any previous approved version with one command

### Stage Lifecycle

```
[Training Pipeline] ──── register ────▶ [Staging]
                                            │
                         [Human reviews]    │
                         [metrics/quality]   │
                                            │
                    ──── approve ─────▶ [Production]
                                            │
                    ──── deploy.py reads ────┘
                         only Production
                         versions
```

### MLflow Commands

```bash
# Register (simulates what training pipeline does)
python3 deploy.py --register --client carawayhome
# Output: Version 1, Stage: Staging

# Approve (governance gate)
python3 deploy.py --approve --client carawayhome
# Output: Version 1, Stage: Production

# List all versions
python3 deploy.py --list-versions --client carawayhome
# Shows: version, stage, created date, source

# Rollback to previous version
python3 deploy.py --rollback --client carawayhome --version 1
# Then re-run --deploy-model --deploy-endpoint to activate
```

### What Gets Stored in MLflow

| Item | Registry Name | What It Tracks |
|------|---------------|----------------|
| Model artifacts | `unified-propensity-{client}` | S3 path to model files, version, stage |
| Docker images | `docker-image-rr-vertex-poc` | ECR image URI, tag, build timestamp |
| Deployment events | (experiment runs) | Who deployed, when, which version, which endpoint |

### MLflow Server Setup (for POC)

For this POC, MLflow runs locally on the EC2 instance:

```bash
mlflow server \
    --host 0.0.0.0 --port 5000 \
    --backend-store-uri sqlite:///mlflow.db \
    --default-artifact-root s3://tie-datascience/mlflow-artifacts/
```

In production, this would be a shared MLflow server accessible to the team.

---

## 11. Infrastructure Mapping (GCP to AWS)

### Compute

| Aspect | GCP | AWS |
|--------|-----|-----|
| Instance type | `e2-standard-8` | `ml.m5.2xlarge` |
| vCPUs | 8 | 8 |
| Memory | 32 GB | 32 GB |
| Network | Up to 16 Gbps | Up to 10 Gbps |

### Storage

| Aspect | GCP | AWS |
|--------|-----|-----|
| Object storage | Google Cloud Storage | Amazon S3 |
| Bucket name | `tie-datascience` | `tie-datascience` |
| Path structure | `propensity-score/dev/app_id={client}/assets/` | Identical |
| Access pattern | `storage.Client().bucket(name)` | `boto3.client("s3")` |

### Container Registry

| Aspect | GCP | AWS |
|--------|-----|-----|
| Service | Artifact Registry | Elastic Container Registry (ECR) |
| Build | `gcloud builds submit` (remote) | `docker build` (local) + `docker push` |
| Image URI format | `{region}-docker.pkg.dev/{project}/{repo}:{tag}` | `{account}.dkr.ecr.{region}.amazonaws.com/{repo}:{tag}` |
| Auth | `gcloud auth configure-docker` | `aws ecr get-login-password` → `docker login` |

### ML Platform

| Aspect | GCP (Vertex AI) | AWS (SageMaker) |
|--------|-----------------|-----------------|
| Model resource | `aiplatform.Model.upload()` | `sagemaker.Model().create()` |
| Endpoint resource | `aiplatform.Endpoint.create()` | Created by `model.deploy()` |
| Deploy command | `model.deploy(endpoint=..., machine_type=...)` | `model.deploy(endpoint_name=..., instance_type=...)` |
| Traffic routing | `traffic_split={"0": 100}` | Single variant (automatic) |
| Autoscaling | `min/max_replica_count` parameters | Separate `application-autoscaling` API |
| Batch inference | Vertex AI BatchPredictionJob | SageMaker Batch Transform |
| Health check path | `/health` | `/ping` |
| Inference path | `/predict` | `/invocations` |
| Port | 8080 | 8080 |

---

## 12. Authentication and IAM

### GCP Authentication (before)

```
Container runs with a GCP Service Account
    → google-cloud-storage uses Application Default Credentials (ADC)
    → ADC automatically finds credentials from:
       1. GOOGLE_APPLICATION_CREDENTIALS env var
       2. Compute Engine metadata service
       3. gcloud auth application-default login
    → Needed GOOGLE_CLOUD_PROJECT env var for storage.Client()
```

### AWS Authentication (after)

```
Container runs with an IAM Role attached to SageMaker
    → boto3 automatically uses the instance metadata service
    → No env vars needed for authentication
    → The IAM role defines what the container can access
```

### IAM Role: `rr-vertex-poc-role`

This role is attached to both:
1. The **EC2 instance** running `deploy.py` (to push images, create endpoints)
2. The **SageMaker Model** resource (so the running container can read S3)

**Required permissions for the SageMaker execution role:**
- `s3:GetObject` on `arn:aws:s3:::tie-datascience/*` (read model files)
- `s3:ListBucket` on `arn:aws:s3:::tie-datascience` (list objects in prefix)
- `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage` (pull container image)
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` (CloudWatch logs)

**Required permissions for the deployer (EC2 role or human):**
- `ecr:CreateRepository`, `ecr:PutImage` (push Docker images)
- `sagemaker:CreateModel`, `sagemaker:CreateEndpoint`, `sagemaker:CreateEndpointConfig`
- `application-autoscaling:RegisterScalableTarget`, `application-autoscaling:PutScalingPolicy`
- `iam:PassRole` (to attach the execution role to the SageMaker model)

---

## 13. The 3-Tier Cache System

This is the performance-critical part of the system. Understanding it is key to understanding how the service achieves low-latency scoring.

### Why 3 Tiers?

Loading XGBoost models from S3 takes 2-5 seconds. You can't have every prediction request wait that long. The cache hierarchy ensures that >99% of requests are served from memory in nanoseconds.

### Layer Definitions

```
┌────────────────────────────────────────────────────────────────────┐
│ L1: In-Memory LRU Cache                                            │
│                                                                    │
│ Location: Each Gunicorn worker process (7 independent caches)      │
│ Speed: Nanoseconds (direct Python dict lookup)                     │
│ Capacity: 25 clients per worker                                    │
│ Refresh: Checked every 1 hour (compares version against Redis)     │
│ Eviction: LRU (Least Recently Used) when full                      │
│ Contains: Deserialized XGBoost Booster objects (ready to predict)  │
└────────────────────────────────────────────────────────────────────┘
        │ miss or version mismatch
        ▼
┌────────────────────────────────────────────────────────────────────┐
│ L2: Redis Cache                                                     │
│                                                                    │
│ Location: Shared Redis instance (same container, port 6379)        │
│ Speed: ~1ms (localhost TCP)                                        │
│ TTL: 7 days (inactive clients expire)                              │
│ Contains: Base64-encoded model bytes + JSON metadata               │
│ Shared: All 7 Gunicorn workers read/write the same Redis           │
│ Locking: Distributed lock prevents thundering-herd on cold load    │
└────────────────────────────────────────────────────────────────────┘
        │ miss or version mismatch
        ▼
┌────────────────────────────────────────────────────────────────────┐
│ L3: Amazon S3 (Source of Truth)                                     │
│                                                                    │
│ Location: s3://tie-datascience/propensity-score/dev/               │
│ Speed: 2-5 seconds (network download)                              │
│ Contains: Raw files (model.ubj, decile_cutpoints.json, etc.)       │
│ Updated by: Training pipeline writes new files under latest/       │
│ Versioning: LastModified timestamp used as version number           │
└────────────────────────────────────────────────────────────────────┘
```

### Request Flow Through Cache

```
Request arrives: get_client_assets("carawayhome")
    │
    ├─ Read L1 entry for "carawayhome"
    ├─ Read Redis version key ("unified_client_version:carawayhome")
    │
    ├─ IF L1 exists AND L1.version == Redis version AND checked < 1hr ago:
    │      → Return L1 bundle (HOT PATH — nanoseconds)
    │
    ├─ ELSE: Check S3 metadata (list_objects_v2 × 3 model types, in parallel)
    │      │
    │      ├─ IF any model type missing from S3:
    │      │      → Return None (client not ready)
    │      │
    │      ├─ IF S3 version <= Redis version:
    │      │      → Read full bundle from Redis (L2 hit — ~1ms)
    │      │      → Write to L1
    │      │      → Return bundle
    │      │
    │      └─ IF S3 version > Redis version (new model deployed):
    │             → Acquire Redis lock ("unified_lock:carawayhome")
    │             → Re-check (another worker may have refreshed)
    │             → Download all files from S3 (parallel, 3 threads)
    │             → Build XGBoost Boosters from bytes
    │             → Write to L2 (Redis) with 7-day TTL
    │             → Write to L1 (memory)
    │             → Release lock
    │             → Return bundle
    │
    └─ Total: ~3-5s on first load, then nanoseconds for 1 hour
```

### Redis Key Structure

```
unified_client_assets:{client_name}   → JSON blob with all 3 models (base64)
unified_client_version:{client_name}  → float timestamp (latest model version)
unified_lock:{client_name}            → Redis lock (prevents parallel downloads)
```

### Cache Invalidation

```bash
# Via API parameter:
POST /invocations
{"instances": [...], "parameters": {"client_name": "carawayhome", "invalidate_cache": true}}

# What it does:
# 1. Deletes Redis keys for that client
# 2. Removes L1 entry
# 3. Next request triggers fresh S3 download
```

---

## 14. Feature Engineering at Inference Time

### What Happens Before Scoring

Raw input features are transformed before being fed to XGBoost:

```python
def prepare_features(df, categories):
    # 1. Compute age from birth date
    df['age'] = compute_age(df['birth_month_and_year'])
    
    # 2. Bucket mobile carriers into 4 groups
    df['mobilecarrier'] = clean_mobilecarrier(df['mobilecarrier'])
    
    # 3. Normalize dwelling type
    df['dwelling_type'] = clean_dwelling(df['dwelling_type'])
    
    # 4. Normalize home ownership
    df['home_owner'] = clean_homeowner(df['home_owner'])
    
    # 5. Compute days since last seen
    df['last_seen_days'] = date_diff_in_days(df['live_event_date'], df['last_seen_date'])
    
    # 6. Apply categorical encoding (from category_mappings.json)
    for col, cats in categories.items():
        df[col] = pd.Categorical(df[col], categories=cats)
    
    # 7. Select only model features (drop metadata like profile_id)
    df = df[inference_cols]
    
    # 8. Cast to correct dtypes
    df = df.astype(inference_dtypes)
    
    return df
```

### Feature Transformations Explained

**Mobile Carrier Bucketing:**
```
"Verizon Wireless" → "premium"
"AT&T"            → "premium"
"T-Mobile"        → "premium"
"Comcast"         → "mid"
"Frontier"        → "mid"
"Sprint"          → "budget"
"Metro PCS"       → "budget"
"Random Carrier"  → "other"
```

**Dwelling Type Normalization:**
```
"Single Family"         → "single"
"SINGLE FAMILY DWELLING" → "single"
"Multi-Family"          → "multi"
"MULTI UNIT"            → "multi"
"Apartment"             → (unchanged, mapped by category)
```

**Home Owner Normalization:**
```
"Home Owner"           → "home_owner"
"HOME OWNER"           → "home_owner"
"Probable Home Owner"  → "probable"
"Renter"               → "renter"
```

**Age Computation:**
```
birth_month_and_year = "1993-08-25 00:00:00"
age = (today - birth_date).days / 365.25
# → approximately 32.9 years
```

### Categorical Feature Handling

XGBoost needs categorical features to have a defined set of valid categories (like an enum). The `category_mappings.json` file for each model defines these:

```json
{
  "state": ["AL", "AK", "AZ", "AR", "CA", ...],
  "gender": ["M", "F"],
  "urbanicity": ["1. Rural", "2. Town", "3. Suburban", "4. Urban"],
  "mobilecarrier": ["premium", "mid", "budget", "other"],
  ...
}
```

If a value is not in the category list, XGBoost treats it as a missing/unknown value and routes it through the tree's default path (learned during training).

---

## 15. Request and Response Format

### Real-Time Request (JSON)

```json
{
  "instances": [
    {
      "days_since_created": 300,
      "open_rate": 0.6,
      "click_rate": 0.1,
      "state": "CT",
      "gender": "M",
      "birth_month_and_year": "1993-08-25 00:00:00",
      "live_event_date": "2026-02-04",
      "mobilecarrier": "Verizon Wireless",
      "dwelling_type": "Single Family",
      "home_owner": "Home Owner"
    }
  ],
  "parameters": {
    "client_name": "carawayhome"
  }
}
```

### Real-Time Response (JSON)

```json
{
  "predictions": [
    {
      "tie_predict_open": 7,
      "tie_predict_click": 3,
      "tie_predict_purchase": 5
    }
  ]
}
```

### Batch Request (JSON Lines)

```
{"client_name": "carawayhome", "days_since_created": 300, "open_rate": 0.6, ...}
{"days_since_created": 150, "open_rate": 0.3, ...}
{"days_since_created": 500, "open_rate": 0.9, ...}
```

### Batch Response (JSON Lines)

```
{"tie_predict_open": 7, "tie_predict_click": 3, "tie_predict_purchase": 5}
{"tie_predict_open": 4, "tie_predict_click": 2, "tie_predict_purchase": 3}
{"tie_predict_open": 9, "tie_predict_click": 8, "tie_predict_purchase": 7}
```

### Score Interpretation

| Decile | Meaning | Recommended Action |
|--------|---------|-------------------|
| 9-10 | Very high propensity | Priority targeting, premium content |
| 7-8 | Above average | Include in campaigns |
| 5-6 | Average | Standard treatment |
| 3-4 | Below average | Reduced frequency |
| 1-2 | Very low propensity | Suppression or re-engagement |

---

## 16. Autoscaling

### Configuration

```python
min_capacity = 1    # Always have at least 1 instance
max_capacity = 3    # Scale up to 3 during peaks
target_value = 70.0 # Target: 70 invocations per instance per minute
```

### How It Works

| Scenario | Invocations/min | Action |
|----------|----------------|--------|
| Low traffic | < 70 total | Stay at 1 instance |
| Medium traffic | 70-140 total | Scale to 2 instances |
| High traffic | > 140 total | Scale to 3 instances |
| Traffic drops | Sustained low | Wait 5 min cooldown, then scale down |

### Scaling Timings

- **Scale-out cooldown**: 60 seconds (can add a new instance every minute during traffic spikes)
- **Scale-in cooldown**: 300 seconds (waits 5 minutes before removing instances, prevents flapping)

### AWS Implementation

Uses Application Auto Scaling (separate service from SageMaker):

```python
aas.register_scalable_target(
    ServiceNamespace="sagemaker",
    ResourceId="endpoint/{name}/variant/AllTraffic",
    ScalableDimension="sagemaker:variant:DesiredInstanceCount",
    MinCapacity=1, MaxCapacity=3,
)
aas.put_scaling_policy(
    PolicyType="TargetTrackingScaling",
    TargetValue=70.0,
    PredefinedMetricType="SageMakerVariantInvocationsPerInstance",
)
```

---

## 17. Testing Strategy

### Test Suite Coverage

The `tests/test.py` file validates the endpoint contract with 21 tests:

| Category | Tests |
|----------|-------|
| Plumbing | Health check (`/ping`), response schema, extra fields allowed |
| Null handling | All values present, partial numeric nulls, partial categorical nulls, all categorical nulls, maximum nulls |
| Score behavior | High engagement profile, low engagement profile, score differentiation (high >= low) |
| Categorical cleaning | Different states, mobile carrier normalization, dwelling type normalization, homeowner normalization, age computation |
| Batch + cache | Batch of 5, large batch of 100, cache invalidation consistency |
| Error handling | Missing client_name (→ 400), unknown client (→ 404) |

### Running Tests

```bash
# Against local container
python3 tests/test.py --url http://localhost:8080/invocations --client carawayhome

# Against SageMaker endpoint (via port forwarding or direct)
python3 tests/test.py --url http://<endpoint-url>/invocations --client carawayhome
```

### Key Test: Score Differentiation

This test validates that the model is working correctly by ensuring a high-engagement profile scores higher than a low-engagement profile across all 3 score types. If this test fails, the model may be corrupted or loaded incorrectly.

---

## 18. Execution Steps on EC2

### Prerequisites

| Requirement | Status |
|-------------|--------|
| EC2 instance running | `i-040a5337dc1df5979` |
| IAM role attached | `rr-vertex-poc-role` |
| Docker installed | Required for `--build-container` |
| Model artifacts in S3 | `s3://tie-datascience/propensity-score/dev/app_id=carawayhome/assets/` |

### Step-by-Step

```bash
# 1. Connect to EC2
aws ec2-instance-connect ssh --instance-id i-040a5337dc1df5979 --region us-east-2

# 2. Navigate to project
cd ~/dev/"unified-score-aws migration"

# 3. Install deploy dependencies (NOT the full requirements.txt)
pip3 install mlflow boto3 sagemaker

# 4. Start MLflow (background)
mlflow server --host 0.0.0.0 --port 5000 \
    --backend-store-uri sqlite:///mlflow.db \
    --default-artifact-root s3://tie-datascience/mlflow-artifacts/ &
export MLFLOW_TRACKING_URI=http://localhost:5000

# 5. Register model
python3 deploy.py --register --client carawayhome

# 6. Approve model
python3 deploy.py --approve --client carawayhome

# 7. Build and push container
python3 deploy.py --build-container \
    --account-id 451433485314 --region us-east-2

# 8. Deploy to SageMaker
python3 deploy.py --deploy-model --deploy-endpoint \
    --client carawayhome \
    --account-id 451433485314 --region us-east-2 \
    --role-arn arn:aws:iam::451433485314:role/rr-vertex-poc-role

# 9. Wait for InService status (~5-10 min)
aws sagemaker describe-endpoint \
    --endpoint-name propensity-unified-endpoint-dev-1 \
    --region us-east-2 --query 'EndpointStatus'

# 10. Test
echo '{"instances":[{"days_since_created":300,"open_rate":0.6,"click_rate":0.1,"state":"CT","gender":"M","birth_month_and_year":"1993-08-25 00:00:00","live_event_date":"2026-02-04","mobilecarrier":"Verizon Wireless","dwelling_type":"Single Family","home_owner":"Home Owner"}],"parameters":{"client_name":"carawayhome"}}' > /tmp/payload.json

aws sagemaker-runtime invoke-endpoint \
    --endpoint-name propensity-unified-endpoint-dev-1 \
    --content-type "application/json" --region us-east-2 \
    --body fileb:///tmp/payload.json response.json

cat response.json
# Expected: {"predictions":[{"tie_predict_open":4,"tie_predict_click":9,"tie_predict_purchase":9}]}
```

### All-in-One Command

```bash
python3 deploy.py --register --approve --build-container \
    --deploy-model --deploy-endpoint \
    --client carawayhome \
    --account-id 451433485314 --region us-east-2 \
    --role-arn arn:aws:iam::451433485314:role/rr-vertex-poc-role
```

---

## 19. Rollback Procedures

### Rollback Model Version

```bash
# See what versions exist
python3 deploy.py --list-versions --client carawayhome

# Roll back to version 1
python3 deploy.py --rollback --client carawayhome --version 1

# Re-deploy with the rolled-back model
python3 deploy.py --deploy-model --deploy-endpoint \
    --client carawayhome \
    --account-id 451433485314 --region us-east-2 \
    --role-arn arn:aws:iam::451433485314:role/rr-vertex-poc-role
```

### Rollback Container Version

If the code (not the model) is the problem:

```bash
# Deploy with a specific older image tag
python3 deploy.py --deploy-model --deploy-endpoint \
    --client carawayhome \
    --image-tag propensity-unified-score-v2 \
    --account-id 451433485314 --region us-east-2 \
    --role-arn arn:aws:iam::451433485314:role/rr-vertex-poc-role
```

### Emergency: Delete Endpoint

```bash
aws sagemaker delete-endpoint \
    --endpoint-name propensity-unified-endpoint-dev-1 \
    --region us-east-2
```

This stops billing immediately. The models and images remain in S3/ECR for re-deployment.

---

## 20. Troubleshooting

### Endpoint Stuck in "Creating"

```bash
# Check failure reason
aws sagemaker describe-endpoint \
    --endpoint-name propensity-unified-endpoint-dev-1 \
    --region us-east-2 --query 'FailureReason'
```

Common causes:
- Container fails health check (`/ping` not returning 200)
- IAM role cannot pull from ECR
- IAM role cannot read from S3

### Container Health Check Failing

```bash
# View container logs
aws logs get-log-events \
    --log-group-name /aws/sagemaker/Endpoints/propensity-unified-endpoint-dev-1 \
    --region us-east-2 \
    --log-stream-name $(aws logs describe-log-streams \
        --log-group-name /aws/sagemaker/Endpoints/propensity-unified-endpoint-dev-1 \
        --region us-east-2 --order-by LastEventTime --descending \
        --query 'logStreams[0].logStreamName' --output text) \
    --query 'events[*].message' --output text
```

### "No Approved Model" Error

```bash
python3 deploy.py --list-versions --client carawayhome
# If nothing shows "Production" stage:
python3 deploy.py --approve --client carawayhome
```

### Model Files Not Found in S3

```bash
aws s3 ls s3://tie-datascience/propensity-score/dev/app_id=carawayhome/assets/ --recursive
# Must show model.ubj, decile_cutpoints.json, category_mappings.json
# for all three: open-score, click-score, purchase-score
```

### Docker Not Available on EC2

```bash
sudo yum install -y docker
sudo service docker start
sudo usermod -aG docker ec2-user
# MUST reconnect your session for group change to take effect
```

### Permission Denied

```bash
aws sts get-caller-identity
# Should show: arn:aws:sts::451433485314:assumed-role/rr-vertex-poc-role/...
```

If the role lacks permissions, contact your AWS admin to attach:
- `AmazonSageMakerFullAccess`
- `AmazonEC2ContainerRegistryFullAccess`
- `AmazonS3ReadOnlyAccess`

### Cost Control

| Resource | Cost | Stop Billing |
|----------|------|--------------|
| SageMaker Endpoint (ml.m5.2xlarge) | ~$0.46/hour | Delete endpoint |
| ECR storage | ~$0.10/GB/month | Delete images |
| S3 model artifacts | ~$0.023/GB/month | Minimal (~100MB) |
| EC2 instance | Varies by type | Stop instance |

**Always delete the endpoint when done testing.**

```bash
aws sagemaker delete-endpoint --endpoint-name propensity-unified-endpoint-dev-1 --region us-east-2
aws sagemaker delete-endpoint-config --endpoint-config-name propensity-unified-config-dev-1 --region us-east-2
```
