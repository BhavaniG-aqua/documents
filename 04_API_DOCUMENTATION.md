# Soundry AI — API Documentation

## 1. Overview

The SoundryAI LeVo2 ML Pipeline API provides a RESTful interface for the complete text-to-music generation platform. It covers authentication, model training lifecycle management, inference (song generation), song library management, and deployment operations.

**Base URL:** `https://api.soundryai.com` (production)  
**API Version:** 1.0.0  
**Protocol:** HTTPS only  
**Authentication:** JWT Bearer token (all endpoints except login, forgot-password, and reset-password)

---

## 2. Authentication

All authenticated endpoints require the header:
```
Authorization: Bearer <JWT_TOKEN>
```

Tokens are obtained via the login endpoint and are invalidated on logout or expiration.

---

## 3. Error Response Format

All error responses follow a consistent structure:

| Field | Type | Description |
|-------|------|-------------|
| `status_code` | integer | HTTP status code |
| `status` | string | Always "error" |
| `message` | string | Human-readable error description |
| `error_code` | string | Machine-readable error identifier |
| `validation_errors` | array (optional) | List of specific validation failures |
| `timestamp` | string | ISO 8601 timestamp of the error |

**Common Error Codes:**
- `UNAUTHENTICATED` — Invalid or expired token (401)
- `INVALID_CREDENTIALS` — Wrong email/password (401)
- `VALIDATION_ERROR` — Invalid request parameters (400)
- `INVALID_STATE` — Operation not valid for current state (400)
- `INVALID_TOKEN` — Expired password reset token (400)
- `NOT_FOUND` — Requested resource doesn't exist (404)

---

## 4. API Endpoints

### 4.1 Authentication Endpoints

#### POST /api/auth/login
**Purpose:** Authenticate a user with email and password.

**Request Body:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `email` | string (email) | Yes | User's email address |
| `password` | string | Yes | User's password |

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `token` | string | JWT authentication token |
| `user` | object | User profile (user_id, email, first_name, last_name, role, created_at, last_login) |

**Errors:** 401 INVALID_CREDENTIALS

---

#### POST /api/auth/logout
**Purpose:** Invalidate the current user's authentication token.

**Authentication:** Required

**Success Response (200):**
```json
{ "status": "logged_out" }
```

---

#### GET /api/auth/me
**Purpose:** Get the current authenticated user's full profile.

**Authentication:** Required

**Success Response (200):**
Returns user object with: user_id, email, first_name, last_name, phone, role, date_of_birth, country, city, postal_code, created_at, last_login.

---

#### PUT /api/auth/profile
**Purpose:** Update the current user's profile. All fields are optional — only provided fields are updated.

**Authentication:** Required

**Updatable Fields:** first_name, last_name, phone, date_of_birth, country, city, postal_code

---

#### POST /api/auth/forgot-password
**Purpose:** Request a password reset link. Always returns 200 (prevents email enumeration).

**Request Body:**
| Field | Type | Required |
|-------|------|----------|
| `email` | string (email) | Yes |

**Response (always 200):**
```json
{ "status": "ok", "message": "If the email exists, a reset link has been sent." }
```

---

#### POST /api/auth/reset-password
**Purpose:** Reset password using a token received via email.

**Request Body:**
| Field | Type | Required |
|-------|------|----------|
| `token` | string | Yes |
| `new_password` | string | Yes |

**Errors:** 400 INVALID_TOKEN (expired or invalid)

---

### 4.2 Dashboard Endpoints

#### GET /api/dashboard/summary
**Purpose:** Get aggregated statistics for the overview dashboard page.

**Authentication:** Required

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `jobs.running` | integer | Currently running training jobs |
| `jobs.completed` | integer | Completed training jobs |
| `jobs.failed` | integer | Failed training jobs |
| `jobs.queued` | integer | Queued training jobs |
| `avg_hw_utilization` | number | Average hardware utilization % |
| `success_rate` | number | completed / (completed + failed) × 100 |
| `avg_train_loss` | number (nullable) | Average loss across running jobs |
| `platform_uptime` | string | Platform uptime percentage |
| `total_songs` | integer | Total generated songs |

---

#### GET /api/dashboard/health
**Purpose:** Real-time health snapshot across all active training runs. Used by Training Health panel and charts.

**Authentication:** Required

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `active_jobs` | integer | Currently running jobs |
| `avg_loss` | number (nullable) | Average training loss |
| `loss_trend` | string | stable_decline / improving / converging / no_active_runs |
| `avg_hw_util` | number | Average hardware utilization % |
| `avg_memory_pct` | number | Average memory usage % |
| `hw_efficiency` | string | efficient / moderate / below_target |
| `loss_sparkline` | array[number] | Recent loss values for chart |
| `hw_util_bars` | array[number] | Recent utilization values (last 7 intervals) |

---

### 4.3 Training Endpoints

#### POST /api/training/start
**Purpose:** Start a new training or retraining job on Trainium2. Validates S3 paths and hyperparameters, generates a unique run ID.

**Authentication:** Required

**Request Body:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `s3_model_path` | string | Yes | S3 path to base model checkpoint |
| `s3_train_data` | string | Yes | S3 path to training data tar shards |
| `s3_val_data` | string | No | S3 path to validation data |
| `learning_rate` | float | Yes | Learning rate for AdamW |
| `batch_size` | integer | Yes | Training batch size |
| `num_steps` | integer | Yes | Total training steps |
| `checkpoint_interval` | integer | Yes | Save checkpoint every N steps |
| `stage` | integer | Yes | Training stage: 1 (full conditioning) or 2 (no genre) |
| `num_cores` | integer | No | NeuronCores for FSDP |
| `use_compile` | boolean | No | Enable torch.compile with neuron backend |
| `use_nki_rope` | boolean | No | Enable NKI fused RoPE kernel |

**Success Response (200):**
```json
{ "run_id": "run-20260421-001", "status": "starting" }
```

**Errors:** 400 VALIDATION_ERROR (invalid params or S3 paths)

---

#### POST /api/training/stop
**Purpose:** Gracefully stop a running training job. Current step completes, checkpoint is saved and uploaded to S3.

**Authentication:** Required

**Request Body:**
| Field | Type | Required |
|-------|------|----------|
| `run_id` | string | Yes |

**Success Response (200):**
```json
{
  "run_id": "run-20260421-001",
  "stopped_at_step": 4250,
  "checkpoint_s3_path": "s3://bucket/runs/run-20260421-001/checkpoints/step_4250_stopped.pt"
}
```

**Errors:** 400 INVALID_STATE (job not running)

---

#### POST /api/training/resume
**Purpose:** Resume a stopped/failed training job from a checkpoint. Creates a new run that continues from a saved checkpoint.

**Authentication:** Required

**Request Body:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Job name for resumed run |
| `mode` | string | Yes | "fine-tuning" or "full-retraining" |
| `dataset_mode` | string | No | "existing" or "upload" |
| `dataset_id` | string | Conditional | Required when dataset_mode=existing |
| `learning_rate` | string | Yes | Learning rate |
| `batch_size` | integer | Yes | Batch size |
| `grad_accum_steps` | integer | Yes | Gradient accumulation steps |
| `epochs` | integer | Yes | Total epochs |
| `warmup_steps` | integer | Yes | Warmup steps |
| `lora_rank` | integer | Conditional | Required for fine-tuning mode |
| `lora_alpha` | integer | Conditional | Required for fine-tuning mode |
| `sequence_length` | integer | Yes | Fixed sequence length |
| `starting_checkpoint` | string | Yes | S3 path to checkpoint to resume from |

**Success Response (200):**
```json
{
  "job_id": "exp-h4i5",
  "name": "soundry-finetune-v4-resume",
  "resumed_from_checkpoint": "s3://soundry-checkpoints/exp-f2a8/step-500/",
  "status": "queued"
}
```

---

#### POST /api/training/cancel
**Purpose:** Cancel a queued (not yet running) training job. Cannot cancel jobs already running — use `/stop` instead.

**Authentication:** Required

**Request Body:**
| Field | Type | Required |
|-------|------|----------|
| `job_id` | string | Yes |

**Success Response (200):**
```json
{ "job_id": "exp-d5f0", "status": "cancelled" }
```

**Errors:** 400 INVALID_STATE (not in queued state), 404 NOT_FOUND

---

#### GET /api/training/status/{run_id}
**Purpose:** Get current status, step progress, loss, throughput, and checkpoint info.

**Authentication:** Required

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `run_id` | string | Run identifier |
| `status` | string | starting / running / stopped / completed / failed / cancelled |
| `current_step` | integer | Current training step |
| `total_steps` | integer | Total planned steps |
| `current_loss` | number | Latest training loss |
| `tokens_per_sec` | number | Current throughput |
| `estimated_time_remaining` | string | Human-readable ETA |
| `last_checkpoint_step` | integer | Last saved checkpoint step |
| `last_checkpoint_s3_path` | string | S3 path to last checkpoint |

---

#### GET /api/training/metrics/{run_id}
**Purpose:** Get per-step training metrics for live chart rendering in the UI.

**Authentication:** Required

**Success Response (200) — `steps` array containing per-step objects:**
| Field | Type | Description |
|-------|------|-------------|
| `step` | integer | Training step number |
| `loss` | number | Training loss at this step |
| `lr` | number | Current learning rate |
| `throughput` | number | Tokens per second |
| `neuroncore_util` | number | NeuronCore utilization % |
| `hbm_usage` | number | HBM memory usage % (of 96 GB) |
| `cpu_util` | number | Host CPU utilization % |
| `step_time` | number | Step duration in seconds |
| `grad_norm` | number | Gradient norm |

---

#### GET /api/training/checkpoints/{run_id}
**Purpose:** List all checkpoints saved during a training run.

**Authentication:** Required

**Success Response (200) — `checkpoints` array:**
| Field | Type | Description |
|-------|------|-------------|
| `step` | integer | Step number |
| `loss` | number | Loss at checkpoint |
| `s3_path` | string | S3 path to checkpoint |
| `timestamp` | string | Save timestamp |

---

#### GET /api/training/jobs
**Purpose:** Paginated list of all training jobs with filtering.

**Authentication:** Required

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `page` | integer | Page number (default 1) |
| `per_page` | integer | Items per page (default 10) |
| `status` | string | Filter: running / completed / failed / queued / cancelled |
| `search` | string | Search by job name, ID, or mode |

**Success Response (200):**
Returns `jobs` array with: id, name, mode, status, epochs_current/total, steps_current/total, train_loss, val_loss, hw_util, memory_pct, started_at, completed_at, duration, eta, checkpoint_path, commit_hash, code_version.

Plus pagination: `total`, `page`, `per_page`.

---

#### GET /api/training/jobs/{job_id}
**Purpose:** Get full details of a single training job, including hyperparameters, dataset info, and failure reason.

**Authentication:** Required

**Additional fields beyond list endpoint:**
- `commit_msg` — Git commit message for this run's code
- `failure_reason` — Error message if job failed
- `params` — Full hyperparameter object (learning_rate, batch_size, grad_accum_steps, epochs, warmup_steps, lora_rank, lora_alpha, sequence_length, starting_checkpoint)
- `dataset` — Dataset info (name, size, samples, s3_path)

---

### 4.4 Dataset Endpoints

#### GET /api/datasets
**Purpose:** List available training datasets.

**Authentication:** Required

**Success Response (200) — `datasets` array:**
| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Dataset identifier |
| `name` | string | Display name |
| `size` | string | Human-readable size |
| `samples` | string | Sample count |
| `s3_path` | string | S3 storage path |
| `created_at` | string | Upload timestamp |

---

#### POST /api/datasets/upload
**Purpose:** Upload a new training dataset (`.tar.gz` or `.zip` containing audio + metadata).

**Authentication:** Required  
**Content-Type:** multipart/form-data

**Request Fields:**
| Field | Type | Required |
|-------|------|----------|
| `file` | binary | Yes |
| `name` | string | No (optional display name) |

**Success Response (200):**
```json
{
  "dataset_id": "ds-007",
  "name": "custom-vocals-2025",
  "size": "4.2 GB",
  "s3_path": "s3://soundry-datasets/custom-vocals-2025/",
  "status": "uploaded"
}
```

---

### 4.5 Model Endpoints

#### GET /api/models
**Purpose:** List available trained models for inference.

**Authentication:** Required

**Success Response (200) — `models` array:**
| Field | Type | Description |
|-------|------|-------------|
| `model_id` | string | Model identifier |
| `display_name` | string | Display name |
| `type` | string | fine-tuned / full-retrain / base |
| `status` | string | deployed / available / training |
| `checkpoint_path` | string | S3 path to model weights |
| `created_at` | string | Creation timestamp |

---

### 4.6 Inference Endpoints

#### POST /api/inference/generate
**Purpose:** Submit a song generation request.

**Authentication:** Required  
**Content-Type:** multipart/form-data

**Request Fields:**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | No | Song title |
| `lyrics` | string | Yes | Lyrics with section markers ([verse], [chorus], [bridge], etc.) |
| `tags` | array[string] | Yes | Genre/mood/style descriptors (min 3, max 50) |
| `model_id` | string | Yes | Model to use for generation |
| `reference_audio_file` | binary | No | Reference audio for style transfer (MP3, WAV, FLAC, M4A, OGG; max 50 MB) |
| `output_mode` | string | Yes | "full" (vocals + instruments) or "instrumental" |
| `duration_minutes` | number | Yes | Desired duration (1-5 minutes) |
| `quality` | string | Yes | "draft", "balanced", or "high" |

**Success Response (200):**
```json
{
  "song_id": "b1c2d3e4-f5a6-7890-bcde-fa1234567890",
  "job_id": "j1k2l3m4-n5o6-7890-pqrs-tu1234567890",
  "status": "pending"
}
```

**Errors:** 400 VALIDATION_ERROR (bad tags count, invalid duration, unsupported format)

---

#### GET /api/inference/status/{job_id}
**Purpose:** Track generation job progress.

**Authentication:** Required

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `job_id` | string | Job identifier |
| `status` | string | pending / analyzing / composing / generating_vocals / finalizing / completed / failed |
| `progress_pct` | integer | Progress percentage (0-100) |
| `current_stage` | string | Human-readable stage description |

**Generation Stages Logic:**
1. `pending` (0%) — Job queued
2. `analyzing` (10%) — Processing lyrics, parsing structure
3. `composing` (20-60%) — LM token generation (autoregressive)
4. `generating_vocals` (60-85%) — Diffusion decode + VAE decode
5. `finalizing` (85-99%) — Audio post-processing, S3 upload
6. `completed` (100%) — Ready for playback

---

#### GET /api/inference/result/{song_id}
**Purpose:** Get audio URLs for a generated song.

**Authentication:** Required

**Success Response (200):**
| Field | Type | Description |
|-------|------|-------------|
| `song_id` | string | Song identifier |
| `audio_url` | string | CDN URL to full audio (48kHz stereo FLAC) |
| `vocal_url` | string | URL to vocal-only track (if available) |
| `bgm_url` | string | URL to BGM-only track (if available) |
| `duration` | number | Actual duration in seconds |
| `generation_time` | number | Total generation time in seconds |

---

### 4.7 Tags Endpoint

#### GET /api/tags
**Purpose:** List curated style tags for song generation (autocomplete support).

**Authentication:** Required

**Query Parameters:**
| Param | Type | Description |
|-------|------|-------------|
| `search` | string | Filter tags by prefix match |
| `limit` | integer | Max tags to return (default 200) |

**Success Response (200):**
```json
{
  "tags": ["pop", "rock", "hip-hop", "electronic", "jazz", "classical", "uplifting", "melancholic"],
  "total": 200
}
```

**Tag Categories:** genre, mood, vocal, instrument, production, structure, tempo, theme, era

---

## 5. Deployment Endpoints

#### POST /api/deployment/deploy
**Purpose:** Deploy a model to a SageMaker inference endpoint.

**Logic:**
1. Select model artifact (`.tar.gz`) from S3
2. Create SageMaker Model object
3. Create Endpoint Configuration (instance type: `inf2.xlarge` or `trn1.2xlarge`)
4. Provision real-time endpoint
5. Wait for `InService` status (< 15 minutes)

#### GET /api/deployment/status
**Purpose:** Get current endpoint status and deployed model.

**Returns:** endpoint status (Creating / InService / Updating / Failed), deployed model version, endpoint URL, instance type.

---

## 6. Data Flow Diagrams

### Training Data Flow
```
POST /api/datasets/upload → S3 upload
POST /api/training/start → Validate params → Pull GitHub code → Build Docker → Submit SageMaker Job
GET /api/training/status → Poll SageMaker → Return metrics
GET /api/training/metrics → Read CloudWatch → Return time-series
POST /api/training/stop → SageMaker StopTrainingJob → Save checkpoint
```

### Inference Data Flow
```
POST /api/inference/generate → Validate → Invoke SageMaker Endpoint → Return job_id
GET /api/inference/status → Poll endpoint → Return progress
GET /api/inference/result → Read S3 → Return CDN URLs
```

---

## 7. Database Schema (PostgreSQL)

### Users Table
| Column | Type | Description |
|--------|------|-------------|
| user_id | UUID (PK) | Unique identifier |
| email | varchar (unique) | Login email |
| password_hash | varchar | Hashed password |
| first_name | varchar | First name |
| last_name | varchar | Last name |
| phone | varchar | Phone number |
| role | varchar | ml_engineer, admin, user |
| date_of_birth | date | Birth date |
| country | varchar | Country |
| city | varchar | City |
| postal_code | varchar | Postal code |
| created_at | timestamp | Account creation |
| last_login | timestamp | Last login time |

### Training Jobs Table
| Column | Type | Description |
|--------|------|-------------|
| job_id | varchar (PK) | Unique job identifier |
| name | varchar | Human-readable name |
| mode | varchar | fine-tuning / full-retraining |
| status | varchar | queued / running / completed / failed / stopped / cancelled |
| params | jsonb | Full hyperparameter configuration |
| dataset_id | varchar (FK) | Reference to dataset |
| commit_hash | varchar | GitHub commit SHA used |
| started_at | timestamp | Job start time |
| completed_at | timestamp | Job completion time |
| failure_reason | text | Error message if failed |
| checkpoint_path | varchar | Latest checkpoint S3 path |

### Songs Table
| Column | Type | Description |
|--------|------|-------------|
| song_id | UUID (PK) | Unique identifier |
| user_id | UUID (FK) | Owner |
| title | varchar | Song title |
| lyrics | text | Full lyrics text |
| tags | text[] | Style tags used |
| model_id | varchar | Model used for generation |
| audio_s3_path | varchar | S3 path to audio file |
| vocal_s3_path | varchar | S3 path to vocal track |
| bgm_s3_path | varchar | S3 path to BGM track |
| duration_seconds | float | Audio duration |
| generation_time_seconds | float | How long generation took |
| output_mode | varchar | full / instrumental / separate |
| quality | varchar | draft / balanced / high |
| status | varchar | pending / completed / failed |
| created_at | timestamp | Generation request time |

### Checkpoints Table
| Column | Type | Description |
|--------|------|-------------|
| id | serial (PK) | Auto-increment |
| job_id | varchar (FK) | Parent training job |
| step | integer | Training step number |
| loss | float | Loss at checkpoint |
| s3_path | varchar | S3 storage path |
| timestamp | timestamp | Save time |

---

## 8. Authentication Flow

```
1. User submits email + password → POST /api/auth/login
2. Server validates credentials against PostgreSQL
3. Server generates JWT token (with user_id, role, expiration)
4. Token returned to client
5. Client includes token in all subsequent requests: Authorization: Bearer <token>
6. Server validates token on every request (middleware)
7. Expired/invalid token → 401 UNAUTHENTICATED
8. Logout → POST /api/auth/logout → token invalidated server-side
```

---

## 9. Rate Limiting and Constraints

| Constraint | Value |
|-----------|-------|
| Max tags per generation request | 50 |
| Min tags per generation request | 3 |
| Max song duration | 5 minutes |
| Min song duration | 1 minute |
| Max reference audio file size | 50 MB |
| Supported audio formats | MP3, WAV, FLAC, M4A, OGG |
| Max dataset upload size | Defined per deployment |
| Pagination default page size | 10 |
| Status refresh interval (UI) | 30 seconds |

---

## 10. System Monitor API (Sidecar Service)

A separate FastAPI service for hardware monitoring:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/cpu` | GET | CPU usage, per-core stats, frequency, load averages |
| `/memory` | GET | RAM total/used/available, swap usage |
| `/gpu` | GET | Per-GPU memory, utilization, temperature |
| `/gpu/{index}` | GET | Stats for specific GPU by index |
| `/overview` | GET | Combined hardware + OS snapshot |

**Usage:** `python monitor_api.py --host 0.0.0.0 --port 8000`

This service is deployed alongside training/inference workloads to provide real-time hardware telemetry for the dashboard and alerting systems.
