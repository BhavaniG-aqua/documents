# POC Code – Integration Assistant: Complete Technical Documentation

## 1. Executive Summary

The POC (Proof of Concept) code in this workspace implements an **AI-powered Customer Data Integration Automation Framework** — a system that uses Large Language Models (LLMs), vector search, and parallel processing to automate the most labor-intensive parts of onboarding a new grocery retailer onto the Afresh platform.

The system runs entirely on **Databricks** (Azure-hosted) as a collection of interactive notebooks. It automates:
1. Understanding what customer source data means (column/table descriptions)
2. Figuring out how source data maps to Afresh's canonical model (mapping generation)
3. Writing the actual transformation code in dbt (code generation)
4. Validating that the data meets business rules (ground rules validation)
5. Fixing issues when code fails in production (issue fixing workflow)

**The core innovation**: Instead of data engineers spending weeks manually analyzing schemas, writing mappings, and coding transformations, LLMs handle the bulk of this work with human review at key checkpoints.

---

## 2. System Architecture

### High-Level Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────────┐
│                         DATABRICKS WORKSPACE                           │
│                                                                        │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────────┐  │
│  │  Orchestrator│───▶│  Column     │───▶│  Table Description       │  │
│  │  Notebook    │    │  Description│    │  Generation              │  │
│  └─────────────┘    │  Generation │    └──────────────────────────┘  │
│         │            └─────────────┘                                   │
│         ▼                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              Mapping Document Generation                          │  │
│  │  (Vector Search + LLM Reranking → Source-to-Target Mappings)     │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              Code Generation                                      │  │
│  │  (LLM generates dbt SQL transformation code per table)           │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              dbt Project Initialization                           │  │
│  │  (Creates project, pushes to GitHub, creates Databricks jobs)    │  │
│  └──────────────────────────────┬────────────────────────────────────┘  │
│                                 ▼                                       │
│  ┌──────────────────┐    ┌──────────────────┐                         │
│  │  Ground Rules    │    │  Issue Fixing     │                         │
│  │  Validation      │    │  Workflow         │                         │
│  └──────────────────┘    └──────────────────┘                         │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              SHARED INFRASTRUCTURE                                 │  │
│  │  • Load Balancing Helper (circuit breaker, rate limiting)        │  │
│  │  • Utility Functions (logging, validation, prompts)              │  │
│  │  • Vector Search Index (semantic column matching)                │  │
│  │  • Delta Lake Tables (state management, audit trail)             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘

External Services:
  • Claude Sonnet 4 (Primary LLM)
  • Claude 3.7 Sonnet (Secondary LLM)
  • GPT-4 OSS 120B (Tertiary LLM)
  • GPT-4 OSS 20B (Quaternary LLM)
  • Databricks Vector Search
  • GitHub (afresh-technologies/integration-assistant)
```

### Database Schema (Internal Tables)

All internal state is stored in `havasu_local.integration_assistant.*`:

| Table | Purpose |
|-------|---------|
| `customer_info` | Client registry (name, IDs, schema paths) |
| `source_raw_column_descriptions` | Enhanced descriptions for each source column |
| `source_raw_table_descriptions` | Enhanced descriptions for each source table |
| `source_raw_column_metrics` | Statistical profiles for each column (nulls, min, max, etc.) |
| `mapping_documents` | Column-to-column mappings between source and target |
| `code_outputs` | Generated dbt transformation SQL code |
| `ground_rules` | Business validation rules (50+ rules) |
| `prompts` | All LLM prompts stored in database (system + user prompt pairs) |
| `integ_table_description` | Descriptions of Afresh's canonical tables |
| `integ_column_descriptions` | Column-level descriptions of the canonical model |
| `error_logs` | Centralized error tracking |
| `ai_response_audit` | Full audit trail of every LLM call (input, output, tokens, latency) |
| `source_raw_column_descriptions_index` | Vector search index on column descriptions |
| `source_raw_table_descriptions_index` | Vector search index on table descriptions |

---

## 3. The Notebooks (Workflow Components)

### 3.1 Column_Description_Gen_And_Upd_Orchestrator.py

**Role**: Entry point / orchestrator for the column description pipeline.

**What It Does**:
1. Reads the latest Excel file from a staging volume (`/Volumes/havasu_local/integration_assistant/staging/source_raw_input_documents`)
2. The Excel has two sheets: one with source column metadata, one with client overview info
3. Validates the document structure (required headers, mandatory fields)
4. Checks if the client already exists in `customer_info`:
   - **New client** → Routes to "Column Description Generation" notebook
   - **Existing client** → Routes to "Column Description Updation" notebook
5. Passes data as JSON widgets to the appropriate downstream notebook

**Key Decision Logic**:
```
IF client_id is NULL (not in customer_info) → GENERATION flow (first-time)
IF client_id exists → UPDATION flow (incremental changes)
```

---

### 3.2 Column Description Generation.py

**Role**: For a brand-new client, generates AI-enhanced descriptions for every source column.

**What It Does**:

1. **Receives input** via widgets: source schema info as JSON, client name, schema paths
2. **Generates IDs**: Creates UUIDs for each table and column (deterministic via uuid5 for columns)
3. **Profiles columns**: Runs statistical profiling on each column in the source schema:
   - Numeric: count, null_count, avg, min, max
   - String: count, null_count, max_length, min_length
   - Boolean: true_count, false_count
   - DateTime: count, null_count
4. **Fetches sample data**: Gets first 5 rows from each source table
5. **Generates descriptions via LLM**: For each column, calls an LLM with:
   - Column name, data type, constraints
   - Any user-provided description
   - Sample data rows from the table
   - A prompt asking: "What does this column represent in a data context?"
6. **Generates table descriptions**: Groups columns by table, calls LLM to produce a summary of what each table represents
7. **Stores results** in `source_raw_column_descriptions` and `source_raw_table_descriptions`

**Processing Pattern**: Batch processing with parallel execution (ThreadPoolExecutor), load-balanced across 4 LLM endpoints.

---

### 3.3 Column Description Updation.py

**Role**: When a client already exists, handles incremental changes (new columns, updated columns, deleted columns).

**What It Does**:

1. **Compares incoming Excel with existing database records** using joins:
   - `left_anti` join → identifies **new** columns (in Excel but not in DB)
   - `left_anti` reverse → identifies **deleted** columns (in DB but not in Excel)
   - `inner` join with value comparison → identifies **updated** columns
2. **Soft-deletes** removed columns (sets `is_active = false`)
3. **Generates AI descriptions** for new and updated columns (same LLM flow as generation)
4. **Creates new versions** for updated columns (old version deactivated, new version inserted with new UUID)
5. **Regenerates table descriptions** for affected tables
6. **Updates the vector search index** (new descriptions get indexed for future similarity searches)

**Key Design**: Uses versioning (soft delete + new insert) rather than in-place updates, providing full audit history.

---

### 3.4 Mapping Document Generation.py

**Role**: Creates the source-to-target column mapping document using vector search + LLM reranking.

**What It Does**:

1. **Loads integration schema** (`integ_column_descriptions`) – these are Afresh's canonical target columns
2. **Separates default vs. non-default columns**:
   - Default columns (like `created_at`, `is_active`) get auto-filled
   - Non-default columns need actual source mapping
3. **For each non-default integration column**:
   - **Vector Search**: Queries the vector index with the integration column's description to find the top-20 most semantically similar source columns
   - Returns results with similarity scores
4. **LLM Reranking**: For each integration column's top-20 candidates:
   - Sends all candidates + target column info to an LLM
   - LLM assigns **enhanced_similarity_scores** considering:
     - Semantic meaning alignment
     - Data type compatibility
     - Constraint alignment
     - Table context (what table it belongs to)
   - Returns reranked scores as JSON
5. **LLM Mapping Logic Generation**: For the best-matched source columns:
   - Calls LLM to generate the **mapping_logic** – a natural language description of how to transform the source column to the target
   - Example: "Cast `item_id` from VARCHAR to TEXT, no transformation needed" or "Combine `street`, `city`, `state`, `zip` into a single `address` field"
6. **Stores results** in `mapping_documents` table
7. **Exports to Excel** in a staging volume for human review

**The Two-Phase Approach**:
```
Phase 1: Vector Search (fast, approximate) → Top 20 candidates per column
Phase 2: LLM Reranking (precise, slower) → Enhanced scores + mapping logic
```

---

### 3.5 Code Generation.py

**Role**: Generates dbt (Data Build Tool) SQL transformation code for each integration table.

**What It Does**:

1. **For each integration table** in the mapping document:
   - Collects all column-level mappings
   - Retrieves the table description
   - Fetches source table schemas and sample data
   - Gets the source schema path for proper SQL references
2. **Constructs a rich prompt** including:
   - Target table name and description
   - Column-by-column mapping logic
   - Source table schemas with data types
   - Sample source data (first 5 rows)
   - The canonical model expectations
3. **Calls LLM** to generate a complete dbt SQL file:
   - Includes `{{ config(...) }}` block
   - Source references using `{{ source('schema', 'table') }}`
   - All column transformations (casts, renames, logic)
   - Proper SQL structure (WITH clauses, JOINs, etc.)
4. **Validates the output**:
   - Checks for SQL keywords (SELECT, FROM, etc.)
   - Ensures non-empty result
   - Strips markdown code fences
   - Removes control characters
5. **Stores** in `code_outputs` table with `is_active = true`

**Key Feature**: Uses the same infinite-retry load-balanced pattern. Every table will eventually get code generated – guaranteed.

---

### 3.6 Code Generation Updation.py

**Role**: Handles human review feedback and regenerates code based on review comments.

**What It Does**:

1. **Reads an Excel file** from a staging volume containing:
   - `code_outputs` sheet: current dbt code + `review_comments` column
   - `overview` sheet: client name and ID
2. **Validates** the client exists and has existing active code
3. **Filters** rows that have non-empty review comments (these are the tables needing updates)
4. **For each table with comments**:
   - Fetches the current dbt code
   - Fetches the mapping data for context
   - Gets the table description
   - Constructs a prompt saying: "Here's the current code, here's the review feedback, please regenerate with these fixes"
5. **Calls LLM** with the "Code Generation Review" prompt set
6. **Deactivates** old code (soft delete)
7. **Inserts** new version of code

**Workflow**:
```
Data Engineer reviews code → Adds comments in Excel → Uploads to volume →
Notebook processes → LLM regenerates → New version stored
```

---

### 3.7 Dbt Project Initialization.py

**Role**: Takes generated code and creates a full, deployable dbt project.

**What It Does**:

1. **Gets client info** from widgets and `customer_info` table
2. **Retrieves active dbt code** from `code_outputs`
3. **Initializes a dbt project**:
   - Runs `dbt init <project_name>`
   - Creates `.sql` files for each integration table in the `models/` folder
   - Removes default dbt scaffolding
4. **Creates configuration files**:
   - `profiles.yml` – Databricks connection config (host, token, warehouse, catalog, schema)
   - `sources.yml` – Declares all source tables from the customer's raw schema
   - `get_custom_schema.sql` macro – Ensures custom schema names are used without prefix
5. **Pushes to GitHub**:
   - Each `.sql` model file → `models/{customer_name}/` in the `integration-assistant` repo
   - Uses GitHub API (create or update via PUT)
6. **Creates Databricks Jobs**:
   - One job per dbt model (serverless compute)
   - Each job runs: `dbt deps`, `dbt seed`, `dbt run --select <model>`
   - Sets permissions (CAN_MANAGE for Kevin at Afresh, IS_OWNER for the deploying user)
7. **Triggers job runs** with parameters (client_id, integ_table_id)

---

### 3.8 Issue Fixing Workflow.py

**Role**: Automated self-healing pipeline – when a dbt job fails, this notebook regenerates the code.

**What It Does**:

1. **Receives a `run_id`** from a failed Databricks job run
2. **Fetches job run info** via Databricks REST API (`/api/2.1/jobs/runs/get`)
3. **Downloads job artifacts** (tar.gz containing dbt output)
4. **Extracts the error message**:
   - Checks `run_results.json` for error messages
   - Falls back to `dbt.log` if needed
5. **Loads current code and mappings** for the failed table
6. **Calls LLM** with:
   - The current (broken) dbt code
   - The error message
   - The mapping document (for context)
   - The source schema path
   - A prompt: "This code failed with this error. Fix it while maintaining the business logic."
7. **Validates** the regenerated code (SQL keyword check, minimum length)
8. **Deactivates old code** and stores new version
9. **Writes the fixed SQL** back to the local dbt project workspace
10. **Pushes to GitHub** (updates the `.sql` file in the repo)

**This creates a feedback loop**: Job fails → Error extracted → Code regenerated → Re-deployed → Job retried.

---

### 3.9 Ground Rules Validation.py

**Role**: Validates customer data against Afresh's business rules using two complementary approaches.

**What It Does**:

1. **Loads ground rules** from the `ground_rules` table (50+ business rules)
2. **Separates rules** into:
   - Rules **with** integration table mapping (can be validated against specific columns)
   - Rules **without** integration table mapping (need similarity search to find relevant data)

3. **For rules WITH integration tables** ("Direct Validation"):
   - Gets the mapping document for that table
   - Sends to LLM with the ground rule text + mapped source columns
   - LLM returns: `{ validation_status: true/false, validation_result_reason: "..." }`
   - For each rule, the LLM determines whether the source data satisfies the requirement
   - Runs in parallel across multiple LLM endpoints

4. **For rules WITHOUT integration tables** ("Query/Similarity-based Validation"):
   - Uses vector search to find relevant source columns for the rule text
   - Sends the rule + found columns to LLM
   - LLM decides whether to:
     - **Generate a SQL query** to validate (query_validation) – then executes it
     - **Validate directly** using reasoning (direct_validation)
   - SQL queries are executed against the actual source data
   - Results include whether validation passed, any syntax errors, and reasoning

5. **Stores results** with metadata (who ran it, when, client_id)

**Example Validation Flow**:
```
Rule: "Orderable items must have a unique identifier"
Table: ORDERABLE_ITEMS
→ LLM examines mapped source columns
→ Finds `item_id` column mapped
→ Returns: { validation_status: true, reason: "item_id column exists and is non-null" }

Rule: "Delivery schedules must be provided for at least 14 days into the future"
→ LLM generates SQL: SELECT COUNT(*) FROM schedules WHERE order_date > CURRENT_DATE + 14
→ Executes query → Returns count → Validates
```

---

### 3.10 Ground Rule Updation.py

**Role**: Manages the lifecycle of ground rules (insert/update/delete) from Excel input.

**What It Does**:

1. Reads latest Excel from `Ground_Rules_Updation/` volume
2. Validates structure and generates UUIDs for new rules
3. Compares with existing database records using composite keys (`ground_rule_id` + `integ_table_name`)
4. Identifies insertions, updates (comparing `subject_area`, `ground_rule`, `validation_type`), and soft deletions
5. Applies changes safely using DataFrame operations (not string interpolation SQL)
6. Exports updated rules back to a volume as a timestamped Excel file

---

### 3.11 Mapping Document Updations.py

**Role**: Handles review feedback on mapping documents and resolves changes using LLM intent detection.

**What It Does**:

1. Reads an Excel file with review comments on existing mappings
2. **Intent Detection**: Calls LLM to understand what the reviewer wants:
   - `intent_by_table_name`: Reviewer wants to change the source table
   - `intent_to_delete`: Reviewer wants to remove a mapping
   - `intent_to_make_as_default`: Reviewer wants to set a default value
   - `intent_by_column_name`: Reviewer wants a specific column
3. **Table Name Resolution**: If intent involves table names, calls LLM to match fuzzy table references to actual source table IDs
4. **Applies changes**: Deletes old mappings, inserts corrected ones with new source columns

---

### 3.12 Load Balancing Helper.py

**Role**: Shared infrastructure notebook providing resilient LLM access across all other notebooks.

**What It Does**:

Implements a production-grade load balancing system with:

#### Classes

| Class | Responsibility |
|-------|---------------|
| `TokenBucket` | Sliding-window rate limiter tracking tokens/minute per model |
| `CircuitBreaker` | Disables models after repeated failures, auto-recovers after timeout |
| `ModelEndpoint` | Wraps a single model with rate limiting + health tracking |
| `LoadBalancer` | Orchestrates model selection, health monitoring, and failover |

#### Key Patterns

1. **Token Bucket Rate Limiting**: Tracks both input tokens per minute (ITPM) and output tokens per minute (OTPM) for each model, respecting actual API limits:
   - Claude Sonnet 4: 2M ITPM, 150K OTPM
   - Claude 3.7 Sonnet: 2M ITPM, 150K OTPM
   - GPT-4 OSS 120B: 1M ITPM, 100K OTPM
   - GPT-4 OSS 20B: 1M ITPM, 100K OTPM

2. **Circuit Breaker Pattern**: After 3 consecutive failures, a model is "opened" (disabled). After 60 seconds cooldown, it's automatically retried.

3. **Infinite Retry with Cooldown**: If ALL models are down, the system:
   - Resets all circuit breakers
   - Waits 15 seconds
   - Retries from scratch
   - **Guarantees 100% success** – no task ever fails permanently

4. **Parallel Processing**: Uses `ThreadPoolExecutor` with configurable batch sizes (default: 20 rows per batch, 10 parallel workers per batch)

5. **Smart Model Selection**: Picks the model with the least usage/errors that currently has available capacity

#### Configuration

```python
CONFIG = {
    "max_parallel_models": 4,
    "min_parallel_models": 1,
    "circuit_breaker_threshold": 3,      # failures before disabling
    "circuit_breaker_timeout": 60,       # seconds before retry
    "all_models_down_cooldown": 15,      # seconds to wait when all fail
    "retry_attempts_per_cycle": 3,       # retries before cycling
    "request_timeout": 45,              # seconds per API call
    "batch_size": 20,                   # rows per batch
    "max_workers_per_batch": 10         # parallel threads per batch
}
```

---

### 3.13 Utility.py

**Role**: Shared utility functions used by all notebooks.

**Key Functions**:

| Function | Purpose |
|----------|---------|
| `log_error(...)` | Appends errors to `error_logs` Delta table with UUID, timestamp, context |
| `log_ai_response(...)` | Audit trail for every LLM call (input, output, tokens, latency) |
| `profile_source_raw_columns(...)` | Statistical profiling of source columns |
| `validate_document(...)` | Validates Excel documents against expected headers/mandatory fields |
| `fetch_prompts(...)` | Retrieves system+user prompt pairs from the `prompts` table |
| `find_latest_file_from_volume(...)` | Finds most recently modified Excel in a volume |

---

## 4. End-to-End Data Flow

### Complete Pipeline Sequence

```
STEP 1: COLUMN DESCRIPTION GENERATION
═══════════════════════════════════════
Input:  Excel with source columns (table_name, column_name, data_type, constraints)
Action: LLM generates enhanced business descriptions for each column
Output: source_raw_column_descriptions table (indexed for vector search)

    ↓

STEP 2: TABLE DESCRIPTION GENERATION
═══════════════════════════════════════
Input:  All columns grouped by table + sample data
Action: LLM generates a summary description of each source table
Output: source_raw_table_descriptions table (indexed for vector search)

    ↓

STEP 3: MAPPING DOCUMENT GENERATION
═══════════════════════════════════════
Input:  Integration columns (target) + Source column descriptions (indexed)
Action: 
  a) Vector search finds top-20 similar source columns per target column
  b) LLM reranks candidates with enhanced similarity scores
  c) LLM generates mapping logic (transformation description)
Output: mapping_documents table + Excel for human review

    ↓ (Human review checkpoint)

STEP 4: CODE GENERATION
═══════════════════════════════════════
Input:  Mapping documents + source schemas + sample data
Action: LLM generates complete dbt SQL transformation code per table
Output: code_outputs table

    ↓ (Human review checkpoint)

STEP 5: DBT PROJECT INITIALIZATION
═══════════════════════════════════════
Input:  Active code_outputs for client
Action:
  a) Creates dbt project structure
  b) Writes .sql model files
  c) Generates profiles.yml and sources.yml
  d) Pushes to GitHub
  e) Creates Databricks jobs (one per model)
  f) Triggers job runs
Output: Running dbt jobs in Databricks

    ↓

STEP 6: GROUND RULES VALIDATION
═══════════════════════════════════════
Input:  Ground rules + mapping documents + source data
Action:
  a) Direct validation: LLM checks if mapped columns satisfy rules
  b) Query validation: LLM generates SQL, system executes it
Output: Validation results with pass/fail + reasoning

    ↓ (If jobs fail)

STEP 7: ISSUE FIXING WORKFLOW
═══════════════════════════════════════
Input:  Failed job run_id → error message + current code
Action: LLM regenerates code incorporating the error context
Output: Fixed code → re-deployed → job retried
```

---

## 5. Key Design Decisions

### Why Multiple LLM Models?

The system uses **4 models** in a priority cascade:
1. **Claude Sonnet 4** – Best quality, primary choice
2. **Claude 3.7 Sonnet** – Fallback with similar quality
3. **GPT-4 OSS 120B** – Open-source alternative (lower token limit: 20K)
4. **GPT-4 OSS 20B** – Lightest fallback

This ensures high availability. If Claude is rate-limited, the system transparently falls back without any task failing.

### Why Vector Search + LLM Reranking?

A two-phase approach balances speed and accuracy:
- **Vector search** (Phase 1): Fast embeddings-based retrieval – narrows hundreds of source columns to 20 candidates in milliseconds
- **LLM reranking** (Phase 2): Deep semantic understanding – evaluates each candidate considering business context, data types, and table relationships

Neither alone is sufficient. Vector search can't understand complex business logic. LLM can't efficiently scan hundreds of columns without a pre-filter.

### Why Infinite Retry?

In data engineering, partial completion is often worse than no completion. If 18 out of 20 tables get code generated but 2 fail, the entire project is blocked. The infinite retry pattern ensures:
- **100% task completion** – every column gets described, every table gets code
- **Automatic recovery** from transient errors (rate limits, timeouts, server errors)
- **No human intervention** needed for API stability issues

### Why Store Prompts in Database?

All LLM prompts live in the `prompts` table rather than being hardcoded. This allows:
- **Prompt versioning** without code changes
- **A/B testing** different prompt strategies
- **Role separation** – prompt engineers can iterate without touching code
- **Audit trail** – every prompt is tracked alongside its results

### Why dbt (Data Build Tool)?

dbt is the industry standard for SQL-based data transformation. Benefits:
- **Declarative** – each model is a SELECT statement
- **Version controlled** – SQL files live in Git
- **Testable** – dbt has built-in testing framework
- **Incremental** – supports incremental materialization for large datasets
- **Databricks native** – `dbt-databricks` adapter provides seamless integration

---

## 6. Human Touchpoints

The system is **not fully autonomous**. Critical human review points include:

| Step | Human Action |
|------|-------------|
| After mapping generation | Data engineer reviews mappings in Excel, adds corrections |
| After code generation | Data engineer reviews dbt SQL, adds review_comments |
| After validation | Data engineer analyzes failed rules, determines if data issue or mapping issue |
| Ground rules management | Business analysts update/add rules via Excel |
| Schema changes | Updation flows handle changes, but engineer confirms intent |

---

## 7. Monitoring & Observability

### Audit Trail

Every LLM call is logged to `ai_response_audit`:
- Input payload (full prompt)
- Output response (full LLM output)
- Input/output token counts
- Response time in milliseconds
- Unique audit ID

### Error Logging

Every exception is captured in `error_logs`:
- Error message
- Function name where it occurred
- Flow name (which pipeline)
- Notebook name
- Timestamp

### Load Balancer Statistics

After each batch, the system prints:
- Total calls per model
- Success/failure counts
- Success rate percentage
- Current ITPM/OTPM usage vs. limits
- Health status (healthy/unhealthy)

---

## 8. File Structure Summary

```
POC code/
├── Column_Description_Gen_And_Upd_Orchestrator.py  ← Entry point
├── Column Description Generation.py                ← First-time description generation
├── Column Description Updation.py                  ← Incremental description updates
├── Mapping Document Generation.py                  ← Vector search + LLM mapping
├── Mapping Document Updations.py                   ← Review-based mapping corrections
├── Code Generation.py                              ← dbt SQL code generation
├── Code Generation Updation.py                     ← Review-based code regeneration
├── Ground Rules Validation.py                      ← Business rule validation
├── Ground rule updation.py                         ← Ground rules lifecycle management
├── Issue Fixing Workflow.py                        ← Auto-healing failed jobs
├── Load Balancing Helper.py                        ← Shared LLM infrastructure
├── Utility.py                                      ← Shared utility functions
├── Development/
│   ├── Dbt Project Initialization.py              ← dbt project setup + deployment
│   └── github link mail testing.py                ← Email notification testing
├── Sample/
│   └── Sequence Diagram 1.puml                    ← Architecture diagram
└── integration-assistant/                          ← Generated dbt project output
    ├── README.md
    └── models/bgc/                                ← Example customer models
        ├── profiles.yml
        ├── sources.yml
        ├── orderable_items.sql
        ├── retail_items.sql
        ├── shipments.sql
        ├── stores.sql
        └── ... (18 more .sql files)
```

---

## 9. How to Run the Pipeline

### Prerequisites
- Access to the Databricks workspace (`adb-5688001054324596.16.azuredatabricks.net`)
- Valid API token for LLM serving endpoints
- Source data loaded in `havasu_local.<customer>_raw` schema
- Integration schema definitions in `integ_column_descriptions` and `integ_table_description`
- Ground rules populated in the `ground_rules` table
- Prompts configured in the `prompts` table

### Execution Order

1. **Prepare**: Upload source column Excel to the staging volume
2. **Run**: `Column_Description_Gen_And_Upd_Orchestrator` → triggers generation or updation
3. **Run**: `Mapping Document Generation` (pass `client_id` as widget)
4. **Review**: Download mapping Excel, add corrections, re-upload
5. **Run**: `Mapping Document Updations` (if corrections were made)
6. **Run**: `Code Generation` (pass `client_id` as widget)
7. **Review**: Download code Excel, add review comments, re-upload
8. **Run**: `Code Generation Updation` (if comments were added)
9. **Run**: `Dbt Project Initialization` (pass `client_name` as widget)
10. **Monitor**: Jobs run automatically; if they fail, `Issue Fixing Workflow` is triggered
11. **Run**: `Ground Rules Validation` (pass `client_id` and `customer_raw_schema_path`)

---

## 10. Key Metrics & Performance

Based on the configuration and code analysis:

- **Batch size**: 20 items per batch
- **Parallelism**: Up to 10 concurrent LLM calls per batch
- **Rate limits respected**: Sliding window token accounting per model
- **Recovery time**: 15 seconds cooldown when all models are exhausted
- **Request timeout**: 45 seconds per individual LLM call
- **Success guarantee**: Infinite retry ensures 0% permanent failure rate

---

## 11. Limitations & Known Gaps

1. **No automated accuracy measurement** – Generated code quality depends on LLM capability and prompt quality
2. **Manual review still required** – Mappings and code need human validation for business edge cases
3. **Token cost tracking** – While tokens are logged, there's no cost optimization or budget enforcement
4. **No monitoring dashboard** – Observability is via print statements and Delta tables, not a real-time dashboard
5. **Single-client serial processing** – The pipeline processes one client at a time
6. **Prompt sensitivity** – Quality of outputs heavily depends on prompt engineering stored in the `prompts` table
7. **Vector search quality** – Depends on the quality of enhanced descriptions for embeddings

---

*This documentation is based on analysis of all POC code notebooks in the workspace. The system is actively being developed and refined as part of the ZEB × Afresh engagement.*
