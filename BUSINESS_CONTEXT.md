# Unified Propensity Score — Business Context & Strategy

## 1. What Is This Project?

The **Unified Propensity Score** is a real-time machine learning inference service that predicts how likely email subscribers are to engage with marketing campaigns. For every subscriber in a client's database, it produces three scores:

| Score | What It Predicts | Business Action |
|-------|------------------|-----------------|
| `tie_predict_open` | Likelihood of opening an email | Target send timing, subject line strategy |
| `tie_predict_click` | Likelihood of clicking a link in an email | Personalize content, prioritize CTAs |
| `tie_predict_purchase` | Likelihood of making a purchase after receiving an email | Drive revenue by targeting high-intent buyers |

Each score is a **decile** (1–10), where 10 = most likely to take the action and 1 = least likely. This is standard practice in marketing analytics — it segments customers into 10 equal groups ranked by predicted behavior.

---

## 2. Business Problem We Solve

### The Challenge

E-commerce and direct-to-consumer brands send millions of emails. Sending the same message to everyone leads to:

- **Over-saturation**: Subscribers who won't open get fatigued and unsubscribe
- **Under-targeting**: High-intent buyers receive the same generic content as disengaged subscribers
- **Wasted budget**: Sending emails costs money (ESP fees, campaign design time)
- **Deliverability damage**: High spam/unsubscribe rates hurt sender reputation

### The Solution

Instead of treating all subscribers equally, this system provides per-subscriber propensity scores that enable:

1. **Smart segmentation** — Only send promotional emails to subscribers in deciles 7–10 for purchase propensity. Subscribers in deciles 1–3 get softer nurture content or reduced frequency.
2. **Spend optimization** — Allocate marketing budgets toward the subscribers most likely to convert.
3. **Churn prevention** — Identify subscribers whose engagement is dropping (decile shifts downward over time) and trigger re-engagement campaigns.
4. **Personalized frequency** — Send more to high-open-rate subscribers and less to those likely to unsubscribe.

### Quantified Impact

Propensity scoring in email marketing typically delivers measurable business outcomes. Industry research shows that machine learning propensity models outperform rule-based approaches by significant margins in terms of return on ad spend and engagement rates. For the clients using this system, the expected benefit is higher revenue per email sent and lower unsubscribe rates by aligning content and frequency to predicted engagement.

---

## 3. Who Are the Clients?

This is a **multi-tenant** system. Each client is an e-commerce brand that uses this scoring service. Examples from the codebase:

- **Caraway Home** (`carawayhome`) — cookware brand
- **Twillory** (`twillory`) — men's apparel brand

Each client has independently trained models stored in S3 under their `app_id`. The training pipeline (separate from this codebase) produces client-specific XGBoost models based on each client's historical email engagement and purchase data.

The system is designed to support **25+ active clients** simultaneously (L1 cache size = 25).

---

## 4. How Scores Are Used Downstream

```
                    ┌─────────────────────────────┐
                    │   Email Service Provider     │
                    │  (Klaviyo, Braze, Iterable)  │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Segment/Audience Builder    │
                    │  "Send to decile >= 7 for    │
                    │   purchase propensity"        │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │  Unified Propensity Score     │
                    │  API (this system)            │
                    │                              │
                    │  Input: subscriber features   │
                    │  Output: open/click/purchase  │
                    │          decile scores        │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │   Data Warehouse / CDP        │
                    │  (Subscriber profiles,        │
                    │   behavioral history)          │
                    └─────────────────────────────┘
```

### Two Consumption Patterns

1. **Real-time scoring** — When a subscriber takes an action (opens, browses, adds to cart), their profile is immediately re-scored and the updated decile feeds back into the campaign targeting logic. This uses the `/invocations` endpoint with JSON payloads.

2. **Batch scoring** — Nightly or weekly, the entire subscriber base is re-scored in bulk. This uses JSON Lines format with SageMaker Batch Transform, scoring hundreds of thousands of profiles efficiently.

---

## 5. What Data Powers the Scores?

The model uses **80+ features** organized into three categories:

### Behavioral Features (Email Engagement)
- `open_rate`, `click_rate` — raw engagement rates
- `decayed_open_rate`, `decayed_click_rate` — recency-weighted (recent activity counts more)
- `days_to_first_open`, `days_to_first_click` — how quickly they engaged initially
- `email_received_level/velocity/acceleration/consistency` — time-series signal decomposition of email receiving patterns
- `email_opened_level/velocity/acceleration/consistency` — same for opens
- `email_clicked_level/velocity/acceleration/consistency` — same for clicks
- `web_act_level/velocity/acceleration/consistency` — website browsing behavior
- `add_cart_level/velocity/acceleration/consistency` — add-to-cart behavior
- `checkout_level/velocity/acceleration/consistency` — checkout behavior
- `ordered_level/velocity/acceleration/consistency` — purchase behavior
- `amount_180d` — total revenue in last 180 days

### Demographic Features (Third-Party Enrichment)
- `state`, `gender`, `age` (computed from `birth_month_and_year`)
- `urbanicity`, `mobilecarrier`, `ethnic_group`, `education`, `occupation_type`
- `median_income`, `income_hh`, `net_worth_hh`, `credit_range`
- `home_owner`, `dwelling_type`, `home_value`, `length_of_residence`
- Lifestyle indicators: `books`, `fitness`, `pet_owner`, `cooking`, `gardening`, `travel_vacation`, `investments`

### Derived Features (Computed at Inference Time)
- `age` — calculated from `birth_month_and_year` to current date
- `last_seen_days` — days between `live_event_date` and `last_seen_date`
- `mobilecarrier` → bucketed into `premium/mid/budget/other`
- `dwelling_type` → normalized to `single/multi`
- `home_owner` → normalized to `home_owner/probable/renter`

---

## 6. Why Are We Migrating from GCP to AWS?

### The Decision

The existing production system runs on **Google Cloud Platform (Vertex AI)**. We are migrating to **AWS (SageMaker)**. This is a platform migration, not a model or business logic change.

### Business Reasons for the Migration

| Reason | Detail |
|--------|--------|
| **Cloud consolidation** | The broader organization is standardizing on AWS. Running one ML workload on GCP while everything else is on AWS creates operational overhead, cross-cloud networking complexity, and billing fragmentation. |
| **Cost optimization** | SageMaker offers Savings Plans (up to 64% off on-demand pricing) for sustained ML inference workloads. AWS's larger ecosystem of instance types provides more granular right-sizing options. |
| **Team skills alignment** | The engineering and DevOps team has deeper AWS expertise. Maintaining GCP-specific IAM, networking, and monitoring for a single service is a burden. |
| **Data locality** | Source data (subscriber profiles, engagement events) already lives in AWS (S3, Redshift). Keeping the scoring service on GCP means cross-cloud data transfers, adding latency and egress cost. |
| **Unified governance** | With MLflow on AWS + SageMaker endpoints, the full ML lifecycle (training → registry → deployment → monitoring) lives in one cloud, simplifying audit trails and access control. |

### What This Migration Is NOT

- **Not a model change** — The XGBoost models, feature engineering, and scoring logic remain identical
- **Not a performance change** — Same instance size (8 vCPU, 32 GB), same architecture (7 Gunicorn workers + Redis)
- **Not a schema change** — Same request/response JSON format for API consumers
- **Not a data migration** — Model artifacts were copied from GCS to S3 with identical folder structure

---

## 7. Model Governance with MLflow

A key addition in the AWS version is formal **model governance** using MLflow Model Registry. This was added to address enterprise requirements around traceability and approval workflows.

### Governance Flow

```
Training Pipeline              MLflow Registry              SageMaker
     │                              │                           │
     │── train models ──────────────│                           │
     │── register (Staging) ────────│                           │
     │                              │                           │
     │         Data Scientist / ML Engineer                     │
     │              reviews metrics & approves                  │
     │                              │                           │
     │── approve (Production) ──────│                           │
     │                              │                           │
     │                              │── deploy.py reads ────────│
     │                              │   approved version        │
     │                              │                           │
     │                              │── creates SageMaker ──────│
     │                              │   Model + Endpoint        │
```

### What Governance Ensures

1. **No unreviewed model goes live** — Only models explicitly transitioned to "Production" stage can be deployed
2. **Audit trail** — Every deployment event is logged with who, when, which version, and what image
3. **Rollback capability** — If a new model underperforms, one command rolls back to any previous approved version
4. **Version tracking** — Both model artifacts and Docker container images are versioned in MLflow

---

## 8. Business Unit & Ownership

| Attribute | Value |
|-----------|-------|
| Business unit | MBFT (Marketing, Business, FinTech) |
| Product category | POC (Proof of Concept → Production) |
| Application name | `rr-vertex-poc` (legacy name from GCP era) |
| Primary owner | Jagan Sivakumaran |
| Infrastructure | AWS us-east-2 (Ohio) |
| Cost class | POC tier with defined expiration |

---

## 9. Success Metrics

| Metric | Target | How Measured |
|--------|--------|--------------|
| Scoring latency (p99) | < 100ms per instance | SageMaker CloudWatch metrics |
| Endpoint availability | 99.9% uptime | SageMaker health checks |
| Model freshness | < 24 hours stale | MLflow version timestamps |
| Scoring accuracy | Maintained vs GCP baseline | Compare decile distributions pre/post migration |
| Cost efficiency | ≤ GCP equivalent | AWS billing vs historical GCP billing |
| Autoscaling responsiveness | Scale-out within 60s | CloudWatch alarms + scaling events |

---

## 10. Roadmap After Migration

1. **POC validation** — Confirm identical scoring results between GCP and AWS endpoints for the same inputs
2. **Shadow mode** — Run both endpoints in parallel, compare outputs, validate business metrics
3. **Traffic cutover** — Switch downstream consumers from GCP Vertex AI endpoint to SageMaker endpoint
4. **GCP decommission** — Shut down Vertex AI endpoint, remove GCS artifacts, delete GCP resources
5. **Production hardening** — Move from POC tags to production-tier infrastructure, implement CI/CD, add monitoring dashboards
6. **Multi-client onboarding** — Scale to additional clients beyond Caraway Home and Twillory
