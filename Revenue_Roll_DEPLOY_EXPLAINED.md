# deploy.py — AWS SageMaker Deployment Explained

## Overview

`deploy.py` is the deployment script that:
1. Builds a Docker image and pushes it to **ECR**
2. Creates a **SageMaker Model** resource pointing to that image
3. Deploys the model to a **SageMaker Real-Time Endpoint**
4. Configures **autoscaling** (1–3 instances)

---

## Full Deployment Flow

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Build & Push to ECR (--build-container)             │
│                                                             │
│  aws ecr create-repository (if not exists)                  │
│  aws ecr get-login-password → docker login                  │
│  docker build -t <image_uri> .                              │
│  docker push <dated_tag>                                    │
│  docker push <latest_tag>                                   │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: Create SageMaker Model (--deploy-model)             │
│                                                             │
│  sagemaker.Model(                                           │
│    image_uri = <ECR image from step 1>                      │
│    role = arn:aws:iam::...:role/SageMakerExecutionRole      │
│    env = {"MODEL_STORAGE_URI": "s3://..."}                  │
│  )                                                          │
│  model.create(instance_type="ml.m5.2xlarge")                │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3: Deploy to Endpoint (--deploy-endpoint)              │
│                                                             │
│  model.deploy(                                              │
│    initial_instance_count = 1                               │
│    instance_type = "ml.m5.2xlarge"                          │
│    endpoint_name = "propensity-unified-endpoint-dev-1"      │
│  )                                                          │
│                                                             │
│  → SageMaker pulls the Docker image from ECR                │
│  → Starts 1 instance running the container on port 8080     │
│  → Routes traffic to /invocations and /ping                 │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: Configure Autoscaling (automatic if max > min)      │
│                                                             │
│  application-autoscaling:                                   │
│    min_capacity = 1                                         │
│    max_capacity = 3                                         │
│    metric = SageMakerVariantInvocationsPerInstance           │
│    target_value = 70.0                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## Code Block Breakdown

### 1. Configuration Defaults

```python
DEFAULT_AWS_ACCOUNT_ID = os.environ.get("AWS_ACCOUNT_ID", "123456789012")
DEFAULT_REGION = "us-east-1"
DEFAULT_S3_BUCKET = "tie-datascience"
DEFAULT_ECR_REPO = "datascience-propensity"
DEFAULT_ROLE_ARN = os.environ.get(
    "SAGEMAKER_ROLE_ARN",
    "arn:aws:iam::123456789012:role/SageMakerExecutionRole"
)
DEFAULT_IMAGE_NAME = "propensity-unified-v1"
DEFAULT_MODEL_NAME = "propensity-unified-score-v1"
DEFAULT_ENDPOINT_NAME = "propensity-unified-endpoint-dev-1"
DEFAULT_INSTANCE_TYPE = "ml.m5.2xlarge"
DEFAULT_MIN_INSTANCES = 1
DEFAULT_MAX_INSTANCES = 3
```

**What it does:**
- Sets all the AWS-specific defaults that can be overridden via CLI args or env vars.
- `AWS_ACCOUNT_ID` — your 12-digit AWS account number (needed to construct ECR URIs).
- `SAGEMAKER_ROLE_ARN` — the IAM role that SageMaker assumes when running your container. This role needs S3 read access and ECR pull access.
- `ml.m5.2xlarge` — 8 vCPU, 32 GB RAM. Equivalent to the GCP `e2-standard-8`. Gives you 7 Gunicorn workers + 1 Redis core.

**How to customize:**
```bash
export AWS_ACCOUNT_ID=111122223333
export SAGEMAKER_ROLE_ARN=arn:aws:iam::111122223333:role/MyCustomRole
python deploy.py --build-container --deploy-model --deploy-endpoint
```

---

### 2. Shell Helpers

```python
def _require_tool(name: str) -> None:
    if shutil.which(name) is None:
        sys.exit(f"Required tool not on PATH: {name}")

def run_command(command, can_fail: bool = False) -> int:
    print(f"\nExecuting: {' '.join(command)}")
    process = subprocess.Popen(
        command,
        stdout=subprocess.PIPE,
        stderr=subprocess.STDOUT,
        text=True,
        shell=(os.name == "nt"),
    )
    # ... streams output line by line ...
```

**What it does:**
- `_require_tool` — pre-flight check that `docker` and `aws` CLI are installed before attempting anything.
- `run_command` — runs a shell command, streams its output to the console in real-time, and raises an error if it fails (unless `can_fail=True`).
- `shell=(os.name == "nt")` — on Windows, uses `shell=True` so `cmd.exe` can resolve `.cmd` shims (like `aws.cmd`). On Linux/Mac it doesn't need this.

---

### 3. ECR Image URI Construction

```python
def ecr_image_uri(account_id, region, repo, tag="latest"):
    return f"{account_id}.dkr.ecr.{region}.amazonaws.com/{repo}:{tag}"
```

**What it does:**
- Constructs the full ECR image URI that Docker and SageMaker need.

**Example output:**
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/datascience-propensity:20260709-143022
```

**How ECR URIs work:**
```
{account_id}.dkr.ecr.{region}.amazonaws.com/{repo_name}:{tag}
     │                    │                      │         │
     │                    │                      │         └── version tag
     │                    │                      └── repository name
     │                    └── AWS region
     └── your AWS account
```

---

### 4. Build and Push Container

```python
def build_and_push_container(account_id, region, repo, image_name, dated_tag):
    _require_tool("docker")
    _require_tool("aws")

    # 1. Create ECR repo (idempotent — fails silently if exists)
    run_command(["aws", "ecr", "create-repository",
                 "--repository-name", repo, "--region", region], can_fail=True)

    # 2. Login to ECR
    login_password = subprocess.check_output(
        ["aws", "ecr", "get-login-password", "--region", region], ...
    ).strip()
    # ... pipes password to: docker login --username AWS --password-stdin <registry>

    # 3. Build the Docker image
    run_command(["docker", "build", "-t", dated_uri, "."])
    run_command(["docker", "tag", dated_uri, latest_uri])

    # 4. Push both tags
    run_command(["docker", "push", dated_uri])
    run_command(["docker", "push", latest_uri])
```

**What it does step by step:**

| Step | Command | Purpose |
|------|---------|---------|
| 1 | `aws ecr create-repository` | Creates the ECR repo if it doesn't exist. `can_fail=True` because it's fine if it already exists. |
| 2 | `aws ecr get-login-password` | Gets a temporary auth token from ECR, then pipes it to `docker login`. This token expires in 12 hours. |
| 3 | `docker build -t <uri> .` | Builds the Dockerfile in the current directory and tags it with the dated URI. |
| 4 | `docker push` | Pushes both the dated tag (for rollback) and `latest` (for convenience). |

**Image tagging strategy:**
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/datascience-propensity:20260709-143022  ← rollback
123456789012.dkr.ecr.us-east-1.amazonaws.com/datascience-propensity:latest           ← convenience
```

---

### 5. Container Environment Variables

```python
def _container_env(storage_uri):
    return {
        "MODEL_STORAGE_URI": storage_uri,
    }
```

**What it does:**
- Defines the env vars injected into the running container by SageMaker.
- `MODEL_STORAGE_URI` tells the container which S3 path to read models from.

**Why only one var (vs GCP's two)?**
- GCP needed `GOOGLE_CLOUD_PROJECT` because `storage.Client()` can't always infer the project from Application Default Credentials alone.
- AWS doesn't need this — `boto3` automatically picks up credentials from the **IAM role** attached to the SageMaker endpoint. No explicit account/project config needed.

---

### 6. Create SageMaker Model

```python
def get_or_create_model(image_uri_dated, model_name, role_arn, container_env, sagemaker_session):
    model = Model(
        image_uri=image_uri_dated,
        role=role_arn,
        name=model_name,
        env=container_env,
        sagemaker_session=sagemaker_session,
    )
    model.create(instance_type=DEFAULT_INSTANCE_TYPE)
    return model
```

**What it does:**
- Creates a **SageMaker Model** resource. This is a metadata object that says: "this Docker image + these env vars = a model that can serve predictions."
- `image_uri` — the ECR image to pull.
- `role` — the IAM role SageMaker assumes. This role gives the container permission to read from S3.
- `env` — environment variables injected at runtime.
- `model.create()` registers it with SageMaker.

**What SageMaker Model is NOT:**
- It does NOT start any instances. It's just a definition.
- It does NOT download or store the XGBoost `.ubj` files — your container handles that itself via the 3-tier cache.

**Model naming:**
```python
model_name = f"{args.model_name}-{dated_tag}"
# Example: "propensity-unified-score-v1-20260709-143022"
```
Each deployment gets a unique name so you can track versions and rollback.

---

### 7. Deploy Model to Endpoint

```python
def deploy_model(model, endpoint_name, endpoint_config_name,
                 instance_type, min_instances, max_instances, sagemaker_session):
    model.deploy(
        initial_instance_count=min_instances,
        instance_type=instance_type,
        endpoint_name=endpoint_name,
    )
```

**What it does:**
- `model.deploy()` does 3 things behind the scenes:
  1. Creates an **Endpoint Configuration** — defines which model + instance type + instance count.
  2. Creates (or updates) an **Endpoint** — the actual network-accessible inference URL.
  3. Provisions instances — pulls the Docker image from ECR, starts the container, waits for `/ping` to return 200.

**What happens on the instance:**
```
SageMaker pulls image from ECR
    → Runs ENTRYPOINT ["/start.sh"]
        → start.sh detects 8 cores (ml.m5.2xlarge)
        → Launches supervisord
            → Redis on core 0
            → Gunicorn (7 workers) on cores 1-7
                → Each worker serves /invocations and /ping
```

**How SageMaker routes traffic:**
- Health checks: `GET /ping` every 30s
- Predictions: `POST /invocations` with JSON body
- SageMaker handles load balancing across instances automatically

---

### 8. Autoscaling Configuration

```python
def _configure_autoscaling(endpoint_name, variant_name, min_capacity, max_capacity):
    client = boto3.client("application-autoscaling")
    resource_id = f"endpoint/{endpoint_name}/variant/{variant_name}"

    # Register the endpoint as a scalable target
    client.register_scalable_target(
        ServiceNamespace="sagemaker",
        ResourceId=resource_id,
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        MinCapacity=min_capacity,
        MaxCapacity=max_capacity,
    )

    # Set the scaling policy
    client.put_scaling_policy(
        PolicyName=f"{endpoint_name}-scaling-policy",
        ServiceNamespace="sagemaker",
        ResourceId=resource_id,
        ScalableDimension="sagemaker:variant:DesiredInstanceCount",
        PolicyType="TargetTrackingScaling",
        TargetTrackingScalingPolicyConfiguration={
            "TargetValue": 70.0,
            "PredefinedMetricSpecification": {
                "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance",
            },
            "ScaleInCooldown": 300,
            "ScaleOutCooldown": 60,
        },
    )
```

**What it does:**
- Uses AWS Application Auto Scaling to dynamically adjust the number of instances.
- Only runs if `max_instances > min_instances` (i.e., autoscaling is desired).

**How the scaling works:**

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `MinCapacity` | 1 | Never go below 1 instance |
| `MaxCapacity` | 3 | Never exceed 3 instances |
| `TargetValue` | 70.0 | Target 70 invocations per instance per minute |
| `ScaleOutCooldown` | 60s | After adding an instance, wait 60s before adding another |
| `ScaleInCooldown` | 300s | After removing an instance, wait 5 min before removing another |

**Scaling behavior:**
```
Traffic low (< 70 invocations/min/instance):
    → Stay at 1 instance

Traffic medium (70-140 invocations/min total):
    → Scale to 2 instances

Traffic high (> 140 invocations/min total):
    → Scale to 3 instances (max)

Traffic drops:
    → Wait 5 minutes (cooldown), then scale back down
```

---

### 9. Main Function

```python
def main(args):
    dated_tag = args.image_tag or datetime.now(timezone.utc).strftime("%Y%m%d-%H%M%S")
    storage_uri = f"s3://{args.s3_bucket}/{args.compiled_models_dir.strip('/')}/"
    container_env = _container_env(storage_uri)

    if args.build_container:
        image_uri_dated = build_and_push_container(...)

    boto_session = boto3.Session(region_name=region)
    sagemaker_session = sagemaker.Session(boto_session=boto_session)

    if args.deploy_model:
        model_name = f"{args.model_name}-{dated_tag}"
        model = get_or_create_model(...)

    if args.deploy_endpoint and model:
        deploy_model(...)
```

**What it does:**
- Orchestrates the full pipeline based on CLI flags.
- Each step is optional — you can just build the container, or just deploy to endpoint, or do everything.

---

## CLI Usage Examples

### First-time full deployment:
```bash
python deploy.py --build-container --deploy-model --deploy-endpoint
```
This builds the image → pushes to ECR → creates SageMaker Model → creates Endpoint → configures autoscaling.

### Update container on existing endpoint:
```bash
python deploy.py --build-container --deploy-model \
    --endpoint-name propensity-unified-endpoint-dev-1
```
Builds new image → creates new Model version → deploys to existing endpoint (blue/green swap).

### Only rebuild the container (no deployment):
```bash
python deploy.py --build-container
```
Just builds and pushes the Docker image. Useful for CI/CD pipelines where deployment is a separate step.

### Use a different instance type:
```bash
python deploy.py --build-container --deploy-model --deploy-endpoint \
    --instance-type ml.m5.xlarge
```
Uses `ml.m5.xlarge` (4 vCPU, 16 GB) instead of `2xlarge`. Good for dev/staging.

---

## GCP vs AWS Deploy Comparison

| Aspect | GCP (Vertex AI) | AWS (SageMaker) |
|--------|-----------------|-----------------|
| Container registry | Artifact Registry (`gcloud builds submit`) | ECR (`docker build` + `docker push`) |
| Auth to registry | `gcloud auth configure-docker` | `aws ecr get-login-password` → `docker login` |
| Model resource | `aiplatform.Model.upload()` | `sagemaker.Model().create()` |
| Endpoint resource | `aiplatform.Endpoint.create()` | Created implicitly by `model.deploy()` |
| Deploy to endpoint | `model.deploy(endpoint=..., machine_type=...)` | `model.deploy(endpoint_name=..., instance_type=...)` |
| Traffic split | `traffic_split={"0": 100}` | Handled automatically (single variant) |
| Autoscaling | Built into `min/max_replica_count` | Separate `application-autoscaling` API |
| Service account | Explicit `service_account=` param | IAM role attached to Model (`role=`) |
| Health route | `/health` | `/ping` |
| Predict route | `/predict` | `/invocations` |
| Port | 8080 (configured via `serving_container_ports`) | 8080 (SageMaker default for BYO containers) |

---

## IAM Role Permissions Required

The `SageMakerExecutionRole` needs these permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::tie-datascience",
        "arn:aws:s3:::tie-datascience/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

The **user/CI running deploy.py** also needs:
- `ecr:CreateRepository`, `ecr:PutImage` (to push images)
- `sagemaker:CreateModel`, `sagemaker:CreateEndpoint`, `sagemaker:CreateEndpointConfig`
- `application-autoscaling:RegisterScalableTarget`, `application-autoscaling:PutScalingPolicy`
- `iam:PassRole` on the SageMaker execution role
