# POC Code – Detailed Notebook-by-Notebook Walkthrough

> This document explains every single notebook in the POC at a level where you can confidently explain to anyone what happens inside each one, step by step, including what data goes in, what comes out, and what happens in between.

---

## Table of Contents

1. [Utility.py](#1-utilitypy)
2. [Load Balancing Helper.py](#2-load-balancing-helperpy)
3. [Column_Description_Gen_And_Upd_Orchestrator.py](#3-column_description_gen_and_upd_orchestratorpy)
4. [Column Description Generation.py](#4-column-description-generationpy)
5. [Column Description Updation.py](#5-column-description-updationpy)
6. [Mapping Document Generation.py](#6-mapping-document-generationpy)
7. [Mapping Document Updations.py](#7-mapping-document-updationspy)
8. [Code Generation.py](#8-code-generationpy)
9. [Code Generation Updation.py](#9-code-generation-updationpy)
10. [Dbt Project Initialization.py](#10-dbt-project-initializationpy)
11. [Ground Rules Validation.py](#11-ground-rules-validationpy)
12. [Ground Rule Updation.py](#12-ground-rule-updationpy)
13. [Issue Fixing Workflow.py](#13-issue-fixing-workflowpy)

---

## 1. Utility.py

**Purpose**: Shared helper functions used across ALL other notebooks. Every notebook runs this first via `%run`.

### Functions Explained:

#### `log_error(error_message, error_function, error_flow, notebook_name)`
- Generates a UUID for the error
- Gets current US/Eastern timestamp
- Creates a single-row Spark DataFrame with error details
- Appends it to `havasu_local.integration_assistant.error_logs` Delta table
- If even logging fails, it just prints to stdout (fail-safe)

#### `log_ai_response(ai_audit_id, input_payload, ai_response, input_tokens_used, output_tokens_used, response_time_ms)`
- Takes the full input prompt and full LLM response
- Records token usage (input + output) and response latency
- Inserts into `havasu_local.integration_assistant.ai_response_audit` table
- Uses `repr()` for safe string escaping in SQL
- Every single LLM call in the entire system gets logged here — this is the audit trail

#### `profile_source_raw_columns(schema_path, source_raw_columns_df, client_id, created_by, notebook_name)`
- Iterates over every row in the input DataFrame (each row = one column from source schema)
- For each column, it:
  - Queries the actual source table to get a non-null sample value
  - Determines the data type (string, numeric, boolean, datetime) from the sample
  - Runs appropriate statistical queries:
    - **Numeric**: count, null_count, avg, min, max
    - **String**: count, null_count, max_length, min_length
    - **Boolean**: true_count, false_count, null_count
    - **Datetime**: count, null_count
  - Appends tuple of metrics to results list
- Creates a Spark DataFrame from all results
- Writes to `havasu_local.integration_assistant.source_raw_column_metrics`
- This gives you a statistical profile of every column — useful for understanding data quality

#### `validate_document(document_df, overview_df, document_name)`
- Checks that all required columns exist in the document
- Checks that all required columns exist in the overview sheet
- Checks mandatory columns have no null values
- Different validation rules for different document types:
  - `source_raw_input_sheet`: needs table_name, column_name, data_type (mandatory)
  - `code_updation`: needs integ_table_name, dbt_code (mandatory)
  - `mapping_document_updation`: needs integ_column_id, mapping_row_id, etc.
  - `overview`: needs customer_name, customer_raw_schema_path, targeted_schema_path
- Returns `True` if valid, `False` if any check fails

#### `fetch_prompts(table_name, prompt_name)`
- Queries the prompts table filtered by `prompt_name`
- Separates into `system_prompt` and `user_prompt` by filtering on `prompt_type`
- Returns both as a tuple
- Every notebook fetches its own prompt pair — prompts are NOT hardcoded anywhere

#### `find_latest_file_from_volume(volume_url)`
- Lists all files in a Databricks volume
- Filters for `.xlsx` files
- Sorts by modification time (newest first)
- Reads the binary content of the latest Excel file
- Parses both sheets (Sheet 0 = document, Sheet 1 = overview) using pandas
- Converts both to Spark DataFrames
- Returns `(document_df, overview_df)`

---

## 2. Load Balancing Helper.py

**Purpose**: The backbone infrastructure that ensures no LLM call ever permanently fails. Every notebook that calls LLMs imports this.

### The Problem It Solves

When you're making hundreds of LLM calls (one per column, one per table), you hit:
- Rate limits (429 errors)
- Timeouts
- Transient server errors
- Token capacity limits

Without this helper, a single rate limit would crash the entire pipeline.

### Classes Explained:

#### `TokenBucket`
- Implements a **sliding window** rate limiter
- Tracks how many tokens were consumed in the last 60 seconds
- Uses a `deque` (double-ended queue) of `(timestamp, token_count)` entries
- `can_consume(tokens)` → checks if adding these tokens would exceed the per-minute limit
- `consume(tokens)` → records the consumption
- `wait_for_capacity(tokens, max_wait)` → blocks until capacity is available (polls every 0.5s)
- Old entries (>60s) are automatically cleaned up

#### `CircuitBreaker`
- Tracks consecutive failures per model
- After `failure_threshold` (default: 3) consecutive failures → opens the circuit (disables model)
- After `timeout` (default: 60s) passes → allows one retry attempt
- If retry succeeds → closes circuit (model is back)
- If retry fails → circuit stays open for another timeout period

#### `ModelEndpoint`
- Wraps a single LLM endpoint URL with:
  - `input_bucket` (TokenBucket) — tracks input tokens per minute
  - `output_bucket` (TokenBucket) — tracks output tokens per minute
  - `circuit_breaker` — health tracking
  - Usage statistics (total calls, successes, failures)
- `is_available()` → checks circuit breaker + both token budgets have capacity
- `reserve_capacity(input, output)` → pre-reserves tokens before making the call
- After the call, actual token usage is reconciled (adjusts if estimate was off)

#### `LoadBalancer`
- Manages an array of `ModelEndpoint` objects
- `select_model()` → picks the healthiest model with least usage that has capacity
- `get_active_model_count()` → how many models are currently healthy
- `reset_all_circuit_breakers()` → emergency reset when everything is down
- `print_stats()` → displays a summary table of model health and usage

### Key Functions:

#### `make_llm_call(endpoint, system_prompt, user_prompt, headers)`
- Estimates input tokens as `(len(system_prompt) + len(user_prompt)) / 4`
- Waits for both input and output token capacity (up to 30s)
- Reserves capacity
- Makes POST request with `timeout=45s`
- Parses response (handles multiple formats: OpenAI-style, list format for GPT-OSS)
- Reconciles actual vs. estimated token usage
- Records success/failure on the endpoint
- Returns: `{ success, content, model, input_tokens, output_tokens, response_time_ms }`

#### `process_single_row(row, sample_data_dict, load_balancer, headers, system_prompt, user_prompt_template)`
- Used by Column Description Generation
- Formats the user prompt with column-specific data
- **Infinite retry loop**:
  - Selects best model → makes call → returns result on success
  - On failure: retries up to 3 times per cycle
  - If all models fail: resets circuit breakers, waits 15s, starts over
  - **NEVER returns failure** — loops forever until success
- Returns enhanced column description

#### `process_batch(batch_rows, ...)`
- Takes a list of rows
- Spawns parallel threads (up to 10 per batch)
- Each thread calls `process_single_row`
- Collects results as futures complete
- If even a future fails (shouldn't happen due to infinite retry), re-attempts inline

#### `process_single_table(table_info, ...)` / `process_table_batch(batch_tables, ...)`
- Same pattern as above but for table descriptions
- Formats prompt with all column metadata + sample data for the table
- Returns table-level description

#### `process_single_mapping_group(group_info, ...)` 
- Used by Mapping Document Generation
- For each integration column, handles the mapping logic generation
- If no candidates found → returns null mapping
- Otherwise builds a candidate block with all source column details
- LLM selects the best match and generates mapping logic

---

## 3. Column_Description_Gen_And_Upd_Orchestrator.py

**Purpose**: The entry point for the entire column description pipeline. Decides whether to run Generation or Updation.

### Step-by-Step:

1. **Installs openpyxl** (for Excel reading)
2. **Imports libraries** + runs Utility notebook
3. **Calls `column_description_generation_orchestrator()`**:
   - Reads the latest Excel file from `/Volumes/havasu_local/integration_assistant/staging/source_raw_input_documents`
   - Validates the document (checks headers and mandatory fields)
   - Extracts from the overview sheet:
     - `customer_name`
     - `client_description`
     - `customer_raw_schema_path` (where source data lives, e.g., `havasu_local.zeb_bgc_raw`)
     - `targeted_schema_path` (where transformed data goes, e.g., `havasu_local.zeb_bgc_integ`)
   - Queries `customer_info` table: does this customer already exist?
     - **YES** → `COLUMN_DESCRIPTION_UPDATION_FLAG = True` (incremental update)
     - **NO** → `COLUMN_DESCRIPTION_GENERATION_FLAG = True` (first-time generation)
   - Returns all metadata + flags

4. **Converts source DataFrame to JSON** (because Databricks widget parameters only accept strings)
5. **Routes to appropriate notebook**:
   - If GENERATION → runs `Column Description Generation` notebook with arguments
   - If UPDATION → runs `Column Description Updation` notebook with arguments (includes `client_id`)
   - Uses `dbutils.notebook.run()` with `timeout_seconds=0` (no timeout — runs until complete)

### What Goes In:
- An Excel file with source columns (table_name, column_name, data_type, constraint, description, is_promotion_used, promotion_description)
- An overview sheet with customer info

### What Comes Out:
- Triggers the appropriate downstream notebook

---

## 4. Column Description Generation.py

**Purpose**: For a brand-new customer, generates AI-enhanced descriptions for every single source column, then generates table-level descriptions.

### Step-by-Step:

1. **Installs `databricks-vectorsearch`**, restarts Python
2. **Receives parameters** via widgets:
   - `source_input_raw_sheet_json` — the source columns as JSON
   - `client_name`, `client_description`, `source_raw_schema_path`, `target_schema_path`
3. **Generates UUIDs**:
   - `source_raw_table_id` — one unique UUID per distinct table name
   - `source_raw_column_id` — UUID5 based on `table_name:column_name` (deterministic)
4. **Profiles all columns**: Calls `profile_source_raw_columns()` to get stats (nulls, min/max, etc.)
5. **Fetches sample data**: For each distinct source table, runs `SELECT * FROM <table> LIMIT 5` and stores as list of dictionaries
6. **Fetches prompts**: Gets "Column Description Generation" system + user prompts from database
7. **Runs the Load Balancing Helper** (`%run`)
8. **Main execution — `generate_column_descriptions()`**:
   - Collects all rows (one per column)
   - Splits into batches of 20
   - For each batch, calls `process_batch()` which:
     - Spawns 10 parallel threads
     - Each thread takes one column, formats the prompt:
       ```
       Table: {table_name}
       Column: {column_name}
       Data Type: {data_type}
       Constraints: {constraints}
       Description: {user_provided_description}
       Sample Rows: {first 5 rows of data}
       ```
     - LLM returns a JSON: `{ "column_description": "..." }`
     - Result stored with metadata (model used, processing time)
   - After all batches: creates a Spark DataFrame with enhanced descriptions
   - Prints model usage distribution and performance stats
9. **Table Description Generation**:
   - Groups all columns by table
   - For each table, sends all its column metadata + sample data to LLM
   - LLM generates: `{ "table_description": "..." }`
   - Stores in `source_raw_table_descriptions`
10. **Final storage**: Both column and table descriptions are persisted to Delta tables with metadata (client_id, timestamps, created_by, is_active=true)

### What Goes In:
- Source column metadata (name, type, constraints) from Excel
- Sample data from actual source tables

### What Comes Out:
- `source_raw_column_descriptions` table populated (one row per column with AI description)
- `source_raw_table_descriptions` table populated (one row per table with AI description)
- `source_raw_column_metrics` table populated (statistics per column)
- Vector search index updated (descriptions become searchable)

---

## 5. Column Description Updation.py

**Purpose**: When source schema changes for an existing customer — handles additions, deletions, and modifications.

### Step-by-Step:

1. **Receives parameters**: Same as generation but also gets `client_id` (existing customer)
2. **Fetches existing records** from `source_raw_column_descriptions` where `client_id` matches and `is_active = true`
3. **Compares incoming vs. existing** using Spark joins:
   - **New rows** (`left_anti` join): columns in Excel that don't exist in DB
     - Gets existing `table_id` if the table already exists, otherwise generates new UUID
     - Generates new `column_id` for every new column
   - **Deleted rows** (`left_anti` reverse): columns in DB that aren't in new Excel
   - **Updated rows** (`inner` join + value comparison): same column but different data_type, constraint, or description
4. **If nothing changed** → exits the notebook immediately
5. **Soft-deletes removed columns**: Sets `is_active = false` + records `modified_timestamp` and `modified_by`
6. **Fetches sample data** from source tables (same as generation)
7. **Enhances new rows**: Calls `enhance_with_ai()` which:
   - For each new column, calls `get_column_description_ai()` 
   - Tries each model in priority order (no load balancer — simpler fallback)
   - Returns enhanced description
   - Adds metadata columns (client_id, timestamps, is_active=true)
   - Appends to `source_raw_column_descriptions`
8. **Enhances updated rows**: Same AI description generation
   - Deactivates old versions (sets `is_active = false`)
   - Inserts new versions with fresh UUIDs (versioning via new column_id)
9. **Regenerates table descriptions** for affected tables:
   - Gets all active columns for the affected tables
   - Groups by table → calls LLM for new table description
   - Deactivates old table descriptions
   - Inserts new ones
10. **Updates vector search index** (implicitly — Delta Sync keeps the index up to date)

### What Goes In:
- New Excel with potentially changed source columns
- Existing state in the database

### What Comes Out:
- Updated `source_raw_column_descriptions` (new versions for changes, deactivated for deletions)
- Updated `source_raw_table_descriptions` (regenerated for affected tables)
- Updated metrics for new columns
- Full audit trail of what changed

---

## 6. Mapping Document Generation.py

**Purpose**: The most complex notebook. Creates the source→target column mapping using vector search (semantic similarity) + LLM reranking + mapping logic generation.

### Step-by-Step:

1. **Loads integration column descriptions** (`integ_column_descriptions`) — these are Afresh's canonical target columns
2. **Loads source column + table descriptions** for this client (joining the two tables)
3. **Filters out excluded tables** (hotspots, in_store_prepared_items, orderable_item_transformations — handled differently)
4. **Separates default vs. non-default columns**:
   - **Default columns**: things like `created_at`, `modified_at`, `is_active` → these get auto-populated, no source mapping needed
   - **Non-default columns**: actual business columns that need a source → target mapping
5. **Fetches table descriptions** for all integration tables
6. **Handles promotion columns**: Identifies source tables that have promotion-related data (special handling)

#### Phase 1: Vector Search (Finding Candidates)

7. For each **non-default integration column**:
   - Takes the column's `integ_column_description` as query text
   - Calls Databricks Vector Search API:
     ```
     POST /api/2.0/vector-search/indexes/{INDEX_NAME}/query
     Body: { query_text, columns, num_results: 20, filters_json: { client_id, source_raw_table_ids } }
     ```
   - Returns top 20 source columns most semantically similar to the target column
   - Each result includes: table_name, column_name, data_type, constraints, IDs, description, similarity_score
   - All results stored in a `results` list

8. Converts to Spark DataFrame with explicit schema (17 columns including similarity_score)
9. Fetches "Reranking Similarity Search" prompts from database

#### Phase 2: LLM Reranking (Refining Scores)

10. Groups results by `integ_column_id` (all 20 candidates per target column)
11. For each target column:
    - Builds a **candidate block** with all 20 source columns + their metadata
    - Includes the target column's info (name, type, constraints, table description)
    - Sends to LLM with reranking prompt
    - LLM returns JSON array: `[{ source_raw_column_id, enhanced_similarity_score }, ...]`
    - Scores are enhanced based on: semantic alignment, data type compatibility, business logic understanding
12. Joins enhanced scores back to the original DataFrame
13. For each target column, keeps only the top candidate(s) based on enhanced score

#### Phase 3: Mapping Logic Generation

14. For each target column with its best-matched source column(s):
    - Sends target info + matched source info to LLM
    - LLM generates **mapping_logic** — a natural language description of the transformation
    - Example outputs:
      - "Direct mapping, cast from INT to TEXT"
      - "Concatenate street_address, city, state, zip with comma separator"
      - "Use CASE WHEN to map status codes to boolean is_active"
      - "NULL - no suitable source column found"

15. **Stores final results** in `mapping_documents` table with all metadata
16. **Exports to Excel** in staging volume for human review

### What Goes In:
- Integration column definitions (target schema)
- Source column descriptions (from Step 4/5 above)
- Vector search index (pre-built on source descriptions)

### What Comes Out:
- `mapping_documents` table: one row per target column with the best source match + mapping logic
- Excel export for data engineer review
- Rows with `source_raw_column_id = NULL` where no suitable match was found

---

## 7. Mapping Document Updations.py

**Purpose**: When a data engineer reviews the mapping document and provides corrections via review comments, this notebook processes those corrections intelligently.

### Step-by-Step:

1. **Reads the review Excel** from staging volume (mapping document with `review_comments` column filled in)
2. **Validates** document structure and client existence
3. **Filters rows with review comments** (only processes rows that have feedback)

#### Phase 1: Intent Detection

4. For each row with review comments:
   - Sends the full context (current mapping + review comment) to LLM
   - LLM detects the **intent** — what does the reviewer want?
   - Returns a JSON with boolean flags:
     ```json
     {
       "intent_by_table_name": true/false,    // Reviewer specified a different source table
       "intent_to_delete": true/false,         // Reviewer wants this mapping removed
       "intent_to_make_as_default": true/false, // Reviewer wants a hardcoded default value
       "intent_by_column_name": true/false,    // Reviewer specified a different source column
       "table_names": ["sales", "items"],      // If by table name
       "row_ids_to_delete": [...],             // If delete
       "default_value": "...",                 // If default
       "column_names": ["store_id"],           // If by column name
       "row_id": "uuid-xxx"                    // Which mapping row this applies to
     }
     ```
   - This runs in parallel across multiple threads

#### Phase 2: Table Name Resolution

5. If intent includes `intent_by_table_name = true`:
   - The reviewer might have written "use the sales table" — but what's the exact table ID?
   - Calls LLM with:
     - The fuzzy table names from the intent
     - Full list of available source tables with IDs and descriptions
     - Review comment for context
   - LLM returns: `{ "matched_table_ids": ["uuid-1", "uuid-2"] }`
   - This maps human-readable references to actual database IDs

#### Phase 3: Apply Changes

6. **For "by table name" intents**:
   - Deletes old mapping rows from `mapping_documents`
   - Joins intent results with source column descriptions for the matched tables
   - Fetches table descriptions for enrichment
   - Inserts new mapping rows pointing to the correct source columns
   - Calls LLM to generate updated `mapping_logic` for the new mappings

7. **For "delete" intents**:
   - Removes the specified rows from `mapping_documents`

8. **For "make as default" intents**:
   - Updates the mapping to have no source column but a fixed `mapping_logic` value (e.g., "NULL", "current_timestamp()", "'USD'")

9. **For "by column name" intents**:
   - Similar to table name resolution but at column level
   - Finds the specific column across all source tables

10. **Exports updated mapping** to Excel for further review if needed

### What Goes In:
- Previously generated mapping document (with review comments added)
- Source column/table descriptions

### What Comes Out:
- Updated `mapping_documents` table reflecting reviewer corrections
- New Excel export with corrections applied

---

## 8. Code Generation.py

**Purpose**: Takes the finalized mapping documents and generates complete dbt (SQL) transformation code for each integration table.

### Step-by-Step:

1. **Gets client name** from widget
2. **Runs Load Balancing Helper + Utility**
3. **Sets up LLM endpoints** (4 models in priority order)
4. **Fetches client metadata** from `customer_info`:
   - `client_id`
   - `customer_raw_schema_path` (e.g., `havasu_local.zeb_bgc_raw`)
   - `target_schema_path` (e.g., `havasu_local.zeb_bgc_integ`)
5. **Deactivates any existing code** for this client (sets `is_active = false`)
6. **Loads active mapping documents** for this client from `mapping_documents`
7. **Fetches "Code Generation" prompts** from database (system + user)

#### Main Generation Loop:

8. **`generate_dbt_code_with_llm()` function**:
   - Gets distinct `integ_table_id` values from mappings
   - For each table:
     
     a. **Collects all column mappings** for that table:
     ```
     Target Column: store_id (TEXT)
     Description: Unique store identifier
     Source Column: id (VARCHAR)
     Source Table: stores
     Mapping Logic: Direct mapping, cast to TEXT
     ```
     
     b. **Gets source table schemas** — for every source table referenced in the mappings:
     - Queries actual Databricks table to get column names + types
     - Fetches 5 sample rows
     - Identifies potential JOIN key columns (anything with 'id', 'key', 'code' in name)
     - Builds a rich context string
     
     c. **Formats the user prompt**:
     ```
     Generate DBT transformation code for table: {name} ({id})
     Target Schema: havasu_local.zeb_bgc_integ
     Source Schema: havasu_local.zeb_bgc_raw
     Table Description: {description}
     
     Source Table Schemas:
     {all source tables with columns and sample data}
     
     Column Mappings:
     {each target column with its source and mapping logic}
     ```
   
   - Splits all tables into batches of 20
   - Processes each batch in parallel (10 workers)
   - Each table goes through `process_single_dbt_table()`:
     - Selects best model via LoadBalancer
     - Makes LLM call
     - Cleans response:
       - Strips markdown code fences (` ```sql `, ` ``` `)
       - Removes control characters
     - **Validates**:
       - Code must be >10 characters
       - Must contain at least one SQL/dbt keyword (SELECT, FROM, WITH, {{ config }}, etc.)
       - If validation fails → retries with different model
     - On success: returns `{ integ_table_id, integ_table_name, dbt_code, model_used, processing_time_ms }`
     - On failure: infinite retry loop (same pattern as all other notebooks)

9. **Stores results**:
   - Creates Spark DataFrame from all results
   - Adds metadata: `is_active = true`, `created_at`, `created_by`, `client_id`
   - Writes to `havasu_local.integration_assistant.code_outputs`

10. **Exports to Excel** for human review (with `integ_table_name`, `dbt_code`, `review_comments` column blank for engineer to fill)

### What Goes In:
- Mapping documents (source-target column pairs with mapping logic)
- Actual source table schemas + sample data
- Integration table descriptions

### What Comes Out:
- `code_outputs` table with complete dbt SQL per integration table
- Excel export for review
- Example generated code:
```sql
{{ config(materialized='view', schema='zeb_bgc_integ') }}

WITH source_stores AS (
    SELECT * FROM {{ source('zeb_bgc_raw', 'stores') }}
),

final AS (
    SELECT
        CAST(id AS STRING) AS id,
        CAST(name AS STRING) AS name,
        CAST(address AS STRING) AS address,
        CAST(latitude AS FLOAT) AS latitude,
        CAST(longitude AS FLOAT) AS longitude,
        CAST(timezone AS STRING) AS timezone,
        CAST(is_active AS BOOLEAN) AS is_active
    FROM source_stores
)

SELECT * FROM final
```

---

## 9. Code Generation Updation.py

**Purpose**: After a data engineer reviews generated code and adds comments in the Excel, this regenerates code incorporating the feedback.

### Step-by-Step:

1. **Reads latest Excel** from `/Volumes/.../staging/code_updations/`
   - Sheet: `code_outputs` — has columns: integ_table_id, integ_table_name, dbt_code, review_comments
   - Sheet: `overview` — has customer_name, client_id
2. **Validates** the Excel structure (correct headers, non-null mandatory fields)
3. **Validates client**:
   - Checks customer exists in `customer_info`
   - Checks client has existing active code in `code_outputs`
4. **Filters rows with review comments** (only processes tables that got feedback)
5. **Loads prompts**: "Code Generation Review" (system + user), "Code Validation", "Code Enhancement"
6. **For each table with review comments**:
   - Fetches mapping data for that table (from `mapping_documents`)
   - Fetches the table description
   - Gets current active dbt code
   - Constructs prompt:
     ```
     Current DBT Code: {existing code}
     Review Comments: {what the engineer wants changed}
     Table Description: {context}
     Mapping Data: {all column mappings with logic}
     
     Please regenerate the code addressing the review comments.
     ```
   - Calls LLM with "Code Generation Review" prompts
   - Cleans and validates response (same SQL keyword check)
7. **Deactivates old code** (sets `is_active = false` for that table)
8. **Inserts new version** with fresh metadata

### What Goes In:
- Excel with review comments on previously generated code
- Current code + mapping context

### What Comes Out:
- Updated `code_outputs` with new code version addressing reviewer feedback
- Old version preserved (is_active = false) for audit

---

## 10. Dbt Project Initialization.py

**Purpose**: Takes all generated code and builds a complete deployable dbt project — files, configs, GitHub push, Databricks jobs.

### Step-by-Step:

1. **Gets client info** from widget + `customer_info` table
2. **Loads active code** from `code_outputs` for the client
3. **Exits if empty** (no code to deploy)
4. **Sets up constants**: Databricks URL, token, GitHub token, repo details, warehouse ID

#### Project Setup:

5. **`setup_dbt_project()`**:
   - Runs `dbt init {project_name}` → creates standard dbt folder structure
   - Creates a `custom_models/` temp directory
   - For each row in `code_outputs`: writes `{table_name}.sql` file with the dbt code
   - Deletes the default `models/` folder created by dbt init
   - Moves `custom_models/` → `models/` (replaces default with our generated models)

6. **Creates custom schema macro** (`macros/get_custom_schema.sql`):
   - Overrides dbt's default schema naming behavior
   - Ensures custom schema names are used exactly as specified (no prefix like `dbt_username_`)

#### Configuration:

7. **`setup_dbt_configs()`**:
   - Creates `profiles.yml`:
     ```yaml
     bgc:
       target: dev
       outputs:
         dev:
           type: databricks
           catalog: havasu_local
           schema: zeb_bgc_integ
           host: adb-xxx.azuredatabricks.net
           http_path: /sql/1.0/warehouses/xxx
           token: dapi...
           threads: 1
     ```
   - Creates `sources.yml`:
     - Runs `SHOW TABLES IN {source_catalog}.{source_schema}`
     - Lists every source table as a dbt source
     ```yaml
     version: 2
     sources:
       - name: zeb_bgc_raw
         catalog: havasu_local
         schema: zeb_bgc_raw
         tables:
           - name: stores
           - name: sales
           - name: shipments
           ...
     ```

#### GitHub Push:

8. **`push_file_to_github(local_path, repo_path)`**:
   - Reads file content, base64 encodes it
   - Checks if file already exists in GitHub (GET)
   - If exists → updates with SHA (PUT with sha)
   - If new → creates (PUT without sha)
   - Pushes to `models/{customer_name}/` folder on `dev` branch

9. **`push_project_to_github()`**:
   - Walks through all `.sql` files in models/
   - Pushes each to GitHub under `models/{customer_name}/`

#### Databricks Jobs:

10. **`create_dbt_jobs()`**:
    - For each `.sql` model file:
      - Creates a Databricks Job with:
        - Serverless compute environment
        - dbt-databricks dependency
        - Commands: `dbt deps`, `dbt seed`, `dbt run --select {model_name}`
        - SQL warehouse connection
        - Descriptive name with timestamp
      - Sets permissions:
        - `CAN_MANAGE` for Kevin (Afresh)
        - `IS_OWNER` for the deploying user
    - Returns a DataFrame of job_id ↔ model mappings

11. **`trigger_dbt_job_runs()`**:
    - For each created job:
      - Calls `/api/2.1/jobs/run-now` with parameters:
        - `integ_table_id`
        - `client_id`
        - `client_name`
      - The job runs serverlessly — no cluster management needed

### What Goes In:
- Active code in `code_outputs`
- Client configuration (schema paths)

### What Comes Out:
- A complete dbt project on the Databricks workspace
- SQL model files pushed to GitHub
- Databricks jobs created and triggered (one per model)
- Each job runs independently and can be monitored

---

## 11. Ground Rules Validation.py

**Purpose**: Validates whether the customer's source data actually satisfies Afresh's business requirements (ground rules).

### Step-by-Step:

1. **Receives** `client_id` and `customer_raw_schema_path` from widgets
2. **Loads mapping documents** for the client (active only)
3. **Loads ground rules** (active only, from `ground_rules` table)
4. **Splits ground rules** into two categories:
   - **With integration table** (has `integ_table_id` + `integ_table_name`) — can be validated against specific mappings
   - **Without integration table** (null `integ_table_id`) — needs similarity search to find relevant data

#### Validation Path 1: Direct Validation (Rules WITH Integration Table)

5. **Filters for `validation_type = 'direct_validation'`**
6. **Calls `validate_ground_rules_with_llm()`**:
   - Groups mapping data by `integ_table_id`
   - For each ground rule:
     - Collects all mapped source columns for that table
     - Builds a context string:
       ```
       source_raw_column_name: store_id
       source_raw_column_data_type: VARCHAR
       source_raw_column_description: Unique identifier for each store
       mapping_row_id: uuid-xxx
       ```
     - Formats prompt:
       ```
       Ground Rule: "Store ID must be provided for each shipment"
       Subject Area: Shipments
       Mapped Source Columns: {context}
       ```
     - LLM evaluates and returns:
       ```json
       { "validation_status": true/false, "validation_result_reason": "..." }
       ```
   - Runs in parallel (3 workers, rate-limited)
   - Results include: ground_rule_id, integ_table_id, mapping_row_id, validation_result, reasoning

#### Validation Path 2: Without Integration Table (Similarity Search)

7. **Calls `validation_without_integ_table()`**:
   - For each ground rule without a table:
     - Uses **vector search** to find top 10 relevant source columns
     - Builds context from search results
     - Sends to LLM with prompt that asks:
       "Is this rule a query validation (can be checked with SQL) or a direct validation (needs reasoning)?"
     - LLM returns:
       ```json
       {
         "is_query_validation": true/false,
         "is_direct_validation": "true"/"false",
         "query": "SELECT COUNT(*) FROM ...",
         "is_direct_validation_result": "true"/"false",
         "reasoning": "..."
       }
       ```
     - **If query validation**:
       - Executes the SQL query against actual source data
       - If result > 0 → validation passes
       - If query has syntax error → records the error (doesn't crash)
     - **If direct validation**:
       - Takes the LLM's reasoning-based verdict

8. **Adds metadata** (validation_id, created_by, created_at, client_id)
9. **Stores results** in validation tables
10. **Exports to Excel** for review

### What Goes In:
- Ground rules (business requirements)
- Mapping documents (what's mapped where)
- Actual source data (for SQL validation queries)

### What Comes Out:
- Validation results per rule: pass/fail + reasoning
- Query validation results: the SQL used + whether it passed + any syntax errors
- Direct validation results: LLM reasoning for each rule

---

## 12. Ground Rule Updation.py

**Purpose**: Manages the ground rules themselves — inserting new rules, updating existing ones, and soft-deleting removed rules from an Excel source.

### Step-by-Step:

1. **Gets latest Excel** from `Ground_Rules_Updation/` volume
2. **Reads and validates** the Excel:
   - Expected columns: subject_area, ground_rule, integ_table_name, validation_type, is_feasible, is_active, integ_table_description, integ_table_id, ground_rule_id
   - Generates UUIDs for any rules missing `ground_rule_id`
   - Converts boolean columns (TRUE/FALSE/1/0/YES/NO → Python bool)
   - Removes duplicates
3. **Fetches existing rules** from database
4. **Identifies changes** using composite key (`ground_rule_id` + `integ_table_name`):
   - **Insertions**: Rules in Excel but not in DB
   - **Updates**: Same composite key but different subject_area, ground_rule text, or validation_type
   - **Soft Deletions**: Rules in DB but not in Excel
5. **Applies changes**:
   - **Insertions**: Direct append to table
   - **Updates**: Uses SQL CASE WHEN join to overwrite changed fields while preserving created_at/created_by
   - **Soft Deletions**: Sets `is_active = false` via SQL join
6. **Exports final state** to a timestamped Excel in `ground_rules/` volume (backup/audit)

### What Goes In:
- Excel with the desired state of all ground rules

### What Comes Out:
- `ground_rules` table updated (new rules added, changed rules updated, removed rules deactivated)
- Timestamped backup Excel exported

---

## 13. Issue Fixing Workflow.py

**Purpose**: Automated self-healing — when a dbt job fails during execution, this notebook extracts the error, asks LLM to fix the code, and re-deploys.

### Step-by-Step:

1. **Receives `run_id`** of the failed job (passed as widget parameter)
2. **Fetches run info** via Databricks REST API:
   - `GET /api/2.1/jobs/runs/get?run_id={run_id}`
   - Extracts: job_id, job_parameters (integ_table_id, client_id, client_name)
3. **Gets source schema path** from `customer_info`
4. **Downloads job artifacts**:
   - `GET /api/2.1/jobs/runs/get-output?run_id={run_id}`
   - Downloads the artifact tarball (`.tar.gz`)
   - Saves with unique filename: `job{id}_run{id}_attempt{n}_{timestamp}.tar.gz`
5. **Extracts the archive** → looks for error in:
   - First: `target/run_results.json` → `results[0].message`
   - Fallback: `logs/dbt.log` → entire log content
6. **Loads current code + mappings**:
   - Gets active `code_outputs` for the failed `integ_table_id`
   - Gets active `mapping_documents` for context
   - Gets `integ_table_description` for context
7. **Fetches "Issue Fixing" prompts** from database
8. **Calls `regenerate_dbt_code_with_llm()`**:
   - Formats prompt with:
     ```
     Table: {name}
     Description: {description}
     Current DBT Code: {the code that failed}
     Error Message: {the actual error from dbt}
     Source Schema: {source_schema_path}
     Column Mappings: {all mapping details}
     
     Fix this code to resolve the error.
     ```
   - Uses same load-balanced, infinite-retry approach
   - Validates output (SQL keywords, minimum length)
   - Returns regenerated code
9. **Updates database**:
   - Deactivates old code (`is_active = false`)
   - Inserts new version with metadata
10. **Writes fixed SQL to workspace**:
    - `write_rephrased_sql()` → finds or creates `{table_name}.sql` in the models directory
    - Overwrites with new code
11. **Pushes to GitHub**:
    - Same `push_file_to_github()` function
    - Updates the `.sql` file in `models/{customer_name}/`

### What Goes In:
- A failed job's `run_id`
- Error message extracted from job artifacts
- Current (broken) code + mapping context

### What Comes Out:
- Fixed code stored in `code_outputs` (new version)
- Updated `.sql` file in local workspace
- Pushed to GitHub
- Ready for job re-trigger

### The Self-Healing Loop:
```
Job runs → Fails → Issue Fixing triggered → Error extracted →
LLM fixes code → New code deployed → Job re-triggered →
(If fails again → loop repeats with new error context)
```

---

## Summary: How They All Connect

```
                    ┌─────────────────────────────┐
                    │   Excel Upload (Source Data) │
                    └──────────────┬──────────────┘
                                   │
                                   ▼
                    ┌─────────────────────────────┐
                    │    ORCHESTRATOR (3)          │
                    │  Decides: Generate or Update │
                    └──────────┬────────┬─────────┘
                               │        │
                    ┌──────────┘        └──────────┐
                    ▼                               ▼
        ┌───────────────────┐           ┌───────────────────┐
        │ COL DESC GEN (4)  │           │ COL DESC UPD (5)  │
        │ First-time flow   │           │ Incremental flow  │
        └─────────┬─────────┘           └─────────┬─────────┘
                  │                                │
                  └───────────────┬────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────────┐
                    │  MAPPING DOC GEN (6)         │
                    │  Vector Search + LLM Rerank  │
                    └──────────────┬──────────────┘
                                   │
                          ┌────────┴────────┐
                          ▼                 ▼
              ┌─────────────────┐  ┌─────────────────┐
              │  Human Review   │  │ MAPPING UPD (7)  │
              │  (Excel review) │──│ Process feedback │
              └─────────────────┘  └────────┬────────┘
                                            │
                                            ▼
                    ┌─────────────────────────────┐
                    │     CODE GENERATION (8)      │
                    │  LLM generates dbt SQL       │
                    └──────────────┬──────────────┘
                                   │
                          ┌────────┴────────┐
                          ▼                 ▼
              ┌─────────────────┐  ┌─────────────────┐
              │  Human Review   │  │ CODE UPD (9)     │
              │  (Excel review) │──│ Process feedback │
              └─────────────────┘  └────────┬────────┘
                                            │
                                            ▼
                    ┌─────────────────────────────┐
                    │   DBT PROJECT INIT (10)      │
                    │  Build + Push + Deploy       │
                    └──────────────┬──────────────┘
                                   │
                          ┌────────┴────────┐
                          ▼                 ▼
              ┌─────────────────┐  ┌─────────────────┐
              │  GROUND RULES   │  │  JOBS RUN        │
              │  VALIDATION (11)│  │  (Databricks)    │
              └─────────────────┘  └────────┬────────┘
                                            │
                                            ▼ (on failure)
                                   ┌─────────────────┐
                                   │ ISSUE FIXING (13)│
                                   │ Auto-heal + retry│
                                   └─────────────────┘

Supporting Notebooks (used by all):
  • Utility.py (1) — Logging, validation, prompt fetching
  • Load Balancing Helper.py (2) — Resilient LLM calls
  • Ground Rule Updation.py (12) — Rules lifecycle management
```

---

## Key Patterns Used Everywhere

| Pattern | Where Used | What It Does |
|---------|-----------|--------------|
| Infinite Retry | All LLM calls | Guarantees 100% task completion |
| Batch + Parallel | Description gen, mapping, code gen | Processes 20 items at a time with 10 threads |
| Soft Delete | Descriptions, mappings, code | Never physically deletes — sets `is_active = false` |
| Version History | Column descriptions, code outputs | New versions get fresh UUIDs; old deactivated |
| Prompts from DB | Every notebook | All LLM prompts stored in `prompts` table, not hardcoded |
| Excel as Interface | All flows | Data engineers interact via Excel uploads/downloads |
| Audit Logging | Every LLM call | Full input/output/tokens recorded in `ai_response_audit` |
| Error Logging | Every function | All exceptions captured in `error_logs` table |
| Widget Parameters | Notebook-to-notebook | Data passed as JSON strings via `dbutils.widgets` |

---

*This document covers every notebook at a depth where you can walk someone through exactly what happens at each step, why it happens, and how it connects to the rest of the pipeline.*
