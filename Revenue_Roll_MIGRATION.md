# Unified Score — GCP to AWS Migration Summary

## Service Mapping

| GCP Component | AWS Equivalent | Notes |
|---------------|---------------|-------|
| GCS (`gs://tie-datascience/...`) | S3 (`s3://tie-datascience/...`) | Same bucket/prefix structure |
| Artifact Registry | ECR | Docker container registry |
| Vertex AI Model | SageMaker Model | Model resource |
| Vertex AI Endpoint | SageMaker Real-Time Endpoint | Inference endpoint |
| `e2-standard-8` | `ml.m5.2xlarge` (8 vCPU, 32 GB) | Instance type |
| `google-cloud-storage` SDK | `boto3` S3 client | Object storage SDK |
| `google-cloud-aiplatform` SDK | `sagemaker` SDK | ML platform SDK |
| `gcloud builds submit` | `docker build` + `docker push` | Container build |
| Vertex AI BatchPredictionJob | SageMaker Batch Transform | Batch inference |

## Route Changes

| GCP (Vertex AI) | AWS (SageMaker) | Purpose |
|-----------------|-----------------|---------|
| `/health` | `/ping` | Health check |
| `/predict` | `/invocations` | Prediction endpoint |

## Environment Variable Changes

| Variable | GCP Value | AWS Value |
|----------|-----------|-----------|
| `MODEL_STORAGE_URI` | `gs://tie-datascience/propensity-score/dev/` | `s3://tie-datascience/propensity-score/dev/` |
| `AIP_HEALTH_ROUTE` | `/health` | `/ping` |
| `AIP_PREDICT_ROUTE` | `/predict` | `/invocations` |
| `GCS_FANOUT_WORKERS` | 3 | renamed to `S3_FANOUT_WORKERS` = 3 |
| `GOOGLE_CLOUD_PROJECT` | (removed) | N/A — boto3 uses IAM role |

## Dependency Changes

| Removed (GCP) | Added (AWS) |
|----------------|-------------|
| `google-cloud-storage==3.4.1` | `boto3==1.35.0` |
| `google-cloud-aiplatform==1.133.0` | `sagemaker==2.232.0` |
| `grpcio-status==1.75.1` | — |
| `fastapi-cloud-cli==0.2.1` | — |

## What Stayed the Same

- All inference logic (XGBoost predict, decile scoring, feature engineering)
- 3-tier caching architecture (L1 in-memory LRU → L2 Redis → L3 object storage)
- Containerized Redis (co-located in same container)
- Supervisor process management (Redis + Gunicorn)
- CPU core pinning strategy (start.sh)
- Pydantic schemas (request/response models)
- All feature cleaning functions (mobilecarrier, dwelling, homeowner, age)
- Distributed locking via Redis for model refresh
- All-or-nothing model loading (3 models required per client)
- Logging format and error handling patterns

## Deploy Usage (AWS)

```bash
# First-time deployment (everything new):
python deploy.py --build-container --deploy-model --deploy-endpoint

# Push new container version to existing endpoint:
python deploy.py --build-container --deploy-model \
    --endpoint-name propensity-unified-endpoint-dev-1

# Only rebuild the container (no SageMaker changes):
python deploy.py --build-container
```

## S3 Artifact Structure (same layout as GCS)

```
s3://tie-datascience/propensity-score/dev/
└── app_id=twillory/
     └── assets/
          ├── open-score/
          │    └── latest/
          │         ├── model.ubj
          │         ├── decile_cutpoints.json
          │         └── category_mappings.json
          ├── click-score/
          │    └── latest/
          │         └── ...
          └── purchase-score/
               └── latest/
                    └── ...
```

## IAM Requirements

The SageMaker execution role needs:
- `s3:GetObject`, `s3:ListBucket` on `arn:aws:s3:::tie-datascience/*`
- `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage` for pulling container
- `sagemaker:*` for endpoint management
- `application-autoscaling:*` for autoscaling configuration
