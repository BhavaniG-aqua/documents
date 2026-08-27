# Soundry AI — Business Context Documentation

## 1. Project Overview

Soundry AI is building a fully managed, end-to-end machine learning pipeline for **text-to-music generation** using the **LeVo2 model**. The system takes lyrics and style descriptions as input and generates full songs (vocals + instruments, up to 4.5 minutes of 48kHz stereo audio).

The project involves scaling the existing model from **500 million to 7 billion parameters** and migrating the entire workflow from NVIDIA GPU hardware (CUDA) onto **AWS Trainium** (for training) and **AWS Inferentia** (for inference) using the **AWS Neuron SDK**.

---

## 2. Parties Involved

| Role | Organization |
|------|-------------|
| **Client** | Soundry AI (Remote Intelligence Solutions) |
| **Service Provider** | Zeb AI, Inc. |

---

## 3. Engagement Details

| Item | Detail |
|------|--------|
| Duration | ~12 weeks |
| Total Cost | $106,000 |
| Net Cost (after AWS funding) | $6,000 |
| AWS Funding | $100,000 (projected) |
| Start Date | December 12, 2025 |
| Methodology | Fixed-scope, fixed-fee |

---

## 4. Business Objective

The goal is to deliver a complete, production-ready platform that allows **non-technical users** at Soundry AI to:

1. Upload training data to train/retrain the music generation model
2. Configure and launch training jobs without touching the AWS console
3. Run inference to generate music from genre tags and lyrics
4. Listen to generated results in a web player
5. Browse generation history and download audio files

The entire system is **AWS-native**, **infrastructure-as-code driven**, and designed so Soundry AI owns every artifact at the end of the engagement.

---

## 5. Business Entities

### 5.1 Users
- Authenticate via AWS Cognito
- Roles: ML Engineer, Admin, Non-technical User
- Access the platform through a lightweight web interface
- No AWS console access required for day-to-day operations

### 5.2 Training Jobs
- Lifecycle states: `queued` → `running` → `completed` / `failed` / `stopped` / `cancelled`
- Each job has a unique run ID, hyperparameter configuration, dataset reference, and S3 checkpoint path
- Jobs are tracked in PostgreSQL with commit SHA for code traceability

### 5.3 Models
- Registered checkpoints stored in S3
- Versions tracked by training run ID and step number
- Can be deployed as SageMaker endpoints for inference
- Types: Fine-tuned (LoRA), Full Retrain, Base

### 5.4 Inference Jobs (Songs)
- Generated outputs with metadata (lyrics, genre tags, model version, timestamps)
- Audio stored in S3, accessible via CDN (CloudFront)
- Tracks available: mixed (full song), vocal-only, BGM-only, or separated
- Status tracked: `pending` → `analyzing` → `composing` → `generating_vocals` → `finalizing` → `completed`

### 5.5 Datasets
- Pre-encoded audio token tar shards stored in S3
- Contain: audio tokens, lyrics tokens, genre features, style features, melody features, structure labels
- Uploaded via the web UI or referenced by S3 path

---

## 6. Business Flow — Training

```
User uploads training data (audio + metadata)
        │
        ▼
Data stored in S3 (s3://soundry-training-data/{dataset-name}/)
        │
        ▼
User configures hyperparameters via Web UI
(learning rate, batch size, epochs, LoRA rank, sequence length, etc.)
        │
        ▼
System pulls latest code from GitHub (commit SHA recorded)
        │
        ▼
Docker container rebuilt if code changed (cached otherwise)
        │
        ▼
SageMaker Training Job launched on Trainium2 instance
        │
        ▼
Training runs with periodic checkpointing (every 500 steps)
  ├── Metrics emitted to CloudWatch (loss, throughput, utilization)
  ├── Status polled every 30 seconds for UI updates
  └── Checkpoints saved to S3 (s3://soundry-checkpoints/{run-id}/)
        │
        ▼
Training completes → Model registered → Ready for deployment
```

---

## 7. Business Flow — Inference

```
User inputs lyrics + genre/style tags (optional: reference audio)
        │
        ▼
Web UI sends request to backend API
        │
        ▼
Backend invokes SageMaker real-time inference endpoint
        │
        ▼
Two-stage generation pipeline runs:
  Stage 1: Primary transformer (7B) generates mixed audio tokens
  Stage 2: Secondary transformer (1B) generates vocal + BGM tokens
        │
        ▼
Audio tokens decoded via diffusion model + VAE → waveform
        │
        ▼
Generated audio stored in S3 (s3://soundry-inference/{user-id}/{timestamp}/)
        │
        ▼
Audio playable in web UI within 10 seconds of completion
  ├── Full song (mixed)
  ├── Vocal track (optional)
  └── BGM track (optional)
        │
        ▼
Generation metadata stored in PostgreSQL (for history browsing)
```

---

## 8. Business Flow — Deployment

```
Training completes successfully
        │
        ▼
Model artifact packaged as model.tar.gz
(Neuron-compiled model + inference handler + tokenizer assets)
        │
        ▼
User selects model from S3 via Deployment Module
        │
        ▼
SageMaker Endpoint provisioned (inf2.xlarge or trn1.2xlarge)
  ├── Deployment completes in under 15 minutes
  └── Health check confirms endpoint is InService
        │
        ▼
Endpoint ready to serve inference requests
  ├── Version updates: zero-downtime blue/green deployment
  └── Safe deletion: removes all SageMaker resources
```

---

## 9. Functional Requirements Summary

| REQ ID | Requirement | Key Acceptance Criteria |
|--------|------------|------------------------|
| REQ-F001 | Optimized Neuron Training Pipeline (500M → 7B) | NeuronCore >70% utilization, BF16 verified, no OOM |
| REQ-F002 | Trainium EC2 Performance Validation | 100-step correctness, 1000-step stability, performance benchmarked |
| REQ-F003 | SageMaker-Compatible Training Container | ECR push, S3 integration, checkpoint recovery works |
| REQ-F004 | Training Control Module | All 5 operations functional, status within 30s |
| REQ-F005 | Automatic GitHub Code Integration | Commit SHA in metadata, cache reuse verified |
| REQ-F006 | SageMaker Endpoint Deployment Module | Deploy <15 min, update <30s downtime |
| REQ-F007 | Neuron-Optimized Inference Pipeline | 2-segment CoT <5 min on inf2.xlarge, no audio artifacts |
| REQ-F008 | Inference Application Module | Audio retrievable within 10s, history paginated |
| REQ-F009 | Full CUDA-to-Neuron Migration | Loss convergence ±5% vs CUDA baseline at 1000 steps |
| REQ-F010 | Lightweight Web Interface | Cross-browser: Chrome, Firefox, Safari desktop |

---

## 10. Non-Functional Requirements

### 10.1 Infrastructure & Deployment
- VPC-based networking with security groups
- IAM least-privilege access control
- ECR lifecycle policies for image cleanup
- Infrastructure-as-Code (IaC) for all resources

### 10.2 Security & Compliance
- JWT-based authentication via Cognito
- S3 bucket policies limiting access
- Encrypted data at rest and in transit
- No secrets in code repositories

### 10.3 Performance
- NeuronCore utilization >70% during training
- Inference response within 5 minutes for 2-segment songs
- UI status refresh within 30 seconds

### 10.4 Cost Efficiency
- SageMaker spot instance support for training
- Only 3 most recent checkpoints retained locally
- ECR image caching to avoid redundant Docker builds
- Endpoint auto-scaling based on demand

### 10.5 Observability
- CloudWatch metrics: loss, throughput, NeuronCore utilization, HBM usage, step time, gradient norm
- Structured JSON logging to CloudWatch Logs
- Custom namespace: `SoundryAI/Training`
- System monitoring API for CPU/memory/GPU metrics

---

## 11. Project Timeline

| Phase | Weeks | Activities |
|-------|-------|------------|
| EPIC-0: Discovery & Validation | 1-2 | Kickoff, open questions, technical research, design alignment |
| EPIC-1: CUDA-to-Neuron Migration | 2-5 | Port training code, scale model to 7B, NKI kernel development |
| EPIC-2: SageMaker Training | 3-6 | Docker container, S3 integration, Training Control Module |
| EPIC-3: CI/CD & GitHub Integration | 5-7 | Automated code pull, ECR caching, version tracking |
| EPIC-4: Deployment & Inference | 5-9 | Endpoint deployment, two-stage inference pipeline |
| EPIC-5: Web Interface | 8-11 | Training dashboard, inference UI, history page |
| EPIC-6: Infrastructure (parallel) | 1-11 | VPC, IAM, monitoring, security hardening |

---

## 12. Technology Stack

| Layer | Technology |
|-------|-----------|
| Compute (Training) | AWS Trainium2 (trn1.32xlarge / trn2.3xlarge) |
| Compute (Inference) | AWS Inferentia2 (inf2.xlarge) or Trainium |
| ML Framework | PyTorch 2.10 (Native) + AWS Neuron SDK |
| Orchestration | Amazon SageMaker (Training Jobs + Endpoints) |
| Container Registry | Amazon ECR |
| Storage | Amazon S3 |
| Database | PostgreSQL |
| Authentication | AWS Cognito |
| API | REST (FastAPI/Flask on ECS Fargate or Lambda + API Gateway) |
| Frontend | React or Vue.js SPA (S3 + CloudFront) |
| Monitoring | Amazon CloudWatch + custom System Monitor API |
| CI/CD | GitHub Actions + ECR |
| Version Control | GitHub |

---

## 13. Data Storage Architecture

| Bucket / Store | Purpose |
|---------------|---------|
| `s3://soundry-training-data/{dataset-name}/` | Raw and preprocessed training data |
| `s3://soundry-checkpoints/{run-id}/` | Training checkpoints (model weights) |
| `s3://soundry-inference/{user-id}/{timestamp}/` | Generated audio files + metadata |
| PostgreSQL | User accounts, job metadata, checkpoint registry, song catalog |
| CloudWatch | Real-time training metrics and logs |

---

## 14. Key Business Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Neuron SDK incompatibility with model architecture | Technical feasibility research in EPIC-0 before building |
| OOM on large model training | Mixed-precision BF16, FSDP sharding, gradient checkpointing |
| SageMaker spot interruptions | Checkpoint-resume from latest S3 checkpoint |
| Code drift between training runs | Automatic GitHub pull + commit SHA tagging per run |
| Long inference latency | Bucketed compilation, micro-offloading, prefetch optimization |
| Non-technical users unable to use system | Lightweight web UI with no AWS knowledge required |

---

## 15. Deliverables at Engagement End

1. Fully migrated CUDA-to-Neuron training codebase (Soundry AI owns)
2. SageMaker training container in ECR
3. CI/CD pipeline for automated builds
4. Deployed inference endpoint on Inferentia/Trainium
5. Web interface for training management and song generation
6. System monitoring infrastructure
7. Complete documentation and knowledge transfer
8. All model artifacts and checkpoints in S3
