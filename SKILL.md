---
name: bigquery-workload-analyst
description: |
  Analyze BigQuery cost consumption, simulate pricing models (On-Demand vs. Capacity/Editions), validate mandatory resourceViewer permissions, dynamically query INFORMATION_SCHEMA.JOBS across project/user/folder/org scopes, and architect advanced workload management solutions (reservations, queueing, BI Engine, cost-safeguards).
license: Apache-2.0
metadata:
  version: v1.0.0
  publisher: yukoga
---

# BigQuery Workload Analyst Skill

## Purpose
This skill assists in analyzing BigQuery query logs, simulating optimal pricing models, and designing comprehensive workload management (WLM) strategies to maximize performance and minimize cost in Google Cloud.

## When to Use This Skill
Activate this skill when the user asks to:
- Optimize BigQuery costs, budgets, or pricing plans (On-Demand vs. Enterprise/Standard Editions).
- Retrieve execution statistics, slot consumption, or bytes scanned from `INFORMATION_SCHEMA.JOBS*`.
- Analyze performance bottlenecks, query slots, concurrent queues, or BI Engine usage.
- Establish workload isolation, query resource limits, or custom reservations.

---

## Mandatory Operational Workflow

### Step 1: Mandatory IAM Role & Permission Verification
Before executing any queries or analyzing database assets, verify that the current user or service account has the necessary permissions.
- **Required Permission**: `bigquery.jobs.list` and `bigquery.metadata.viewer` (satisfied by `roles/bigquery.resourceViewer` or higher roles such as `roles/bigquery.admin`, `roles/bigquery.metadataViewer`).
- **Pre-flight Validation Action**:
  - Direct the agent to prompt the user or run a lightweight test query to ensure they can select from `INFORMATION_SCHEMA`.
  - **Fallback / Failure Gate**: If permissions are missing, immediately halt the operation. Provide the following copy-pasteable instructions to grant the role, and ask the user to re-run after authorization is complete:
    ```bash
    # Grant resourceViewer role at the project level
    gcloud projects add-iam-policy-binding [PROJECT_ID] \
        --member="user:[USER_EMAIL]" \
        --role="roles/bigquery.resourceViewer"
    ```

### Step 2: Dynamic Scope and View Selection
Choose the correct `INFORMATION_SCHEMA` metadata view dynamically depending on the user's intent and organizational scope:
- **Project-Level** (Default/Ad-hoc): `INFORMATION_SCHEMA.JOBS_BY_PROJECT` (or standard `INFORMATION_SCHEMA.JOBS` within a specific dataset/project).
- **User-Level** (Specific developer/service account analysis): `INFORMATION_SCHEMA.JOBS_BY_USER`.
- **Folder-Level** (Shared workspace/folders): `INFORMATION_SCHEMA.JOBS_BY_FOLDER`.
- **Organization-Level** (Enterprise-wide billing and audit analysis): `INFORMATION_SCHEMA.JOBS_BY_ORGANIZATION`.

### Step 3: Job-Level Aggregation & De-duplication Rule
- **Timeline Correction Rule**: If the input data is sample-based timeline logs, **NEVER** perform direct raw count/sum billing calculations on rows. Direct row-level summing overestimates cost by **up to 30.8x** due to sampling frequency duplication.
- **SQL Aggregation Protocol**:
  - Always group by `job_id` (and optionally `reservation_id` / `user_email`).
  - Calculate `bytes_billed` as `MAX(total_bytes_billed)`.
  - Calculate `slot_sec` as `SUM(total_slot_ms) / 1000`.

### Step 4: Dynamic Language & Localization Tuning
Customize all outputs (reports, suggestions, instructions, charts, code comments, and annotations) dynamically based on the user's preferred language, matching their environment or query language.
- **Language Detection**: Automatically inspect the user's prompt, git settings, active OS locale, or requested output format. Default to Japanese (`'ja'`) if the user prompts in Japanese, and English (`'en'`) if in English.
- **Term & Label Mapping**: Translate key metrics and structural definitions correctly across languages (e.g. Japanese `【適正配置】` ⇄ English `【Optimal Allocation】`, `【スロット非効率クエリ】` ⇄ `【Slot-Inefficient Queries】`, `【オンデマンド過剰支払】` ⇄ `【On-Demand Overpayment】`).
- **Font & Rendering Configuration**: Ensure chart-drawing code dynamically selects standard localized font families supported by the user's OS platform (e.g., `'Arial Unicode MS'` or `'AppleGothic'` on macOS, `'IPAexGothic'` or `'MS Gothic'` on Windows/Linux) to avoid rendering errors or broken characters (tofu) in localized charts.
- **Mandatory Chart Linkage in Reports**: When generating a final analytical or cost optimization report (in Markdown or HTML), the agent **MUST** ensure that the code blocks from `references/VISUALIZATION_PATTERNS.md` are executed to generate the plots, and both resulting chart images (`bq_breakeven_chart.png` and `bq_confusion_matrix.png`) are physically embedded and linked as visual evidence in the report (using correct relative image links `![](./bq_breakeven_chart.png)` and `![](./bq_confusion_matrix.png)`).

---

## Key Reference Guidelines (Progressive Disclosure)

To maintain a clean and token-efficient workspace, this skill utilizes modular reference guides. Refer to these sub-files for comprehensive, step-by-step guidance:

### 1. Cost & Pricing Analysis Guidelines
Detailed de-duplication SQL queries, mathematical break-even models ($375,000\text{ slot-seconds/TB}$ for Enterprise Edition at $\$0.06/\text{slot-hour}$), and the 4-quadrant mismatch interpretation guidelines.
- **See**: [references/ANALYSIS_GUIDELINES.md](references/ANALYSIS_GUIDELINES.md)

### 2. Workload Management (WLM) Architecture
Standard guidelines for mapping workloads to dedicated reservations (ETL, ad-hoc, BI), setting autoscaling slot ceilings, query queuing, BI Engine allocations, and setting query `maximum_bytes_billed` safeguards.
- **See**: [references/WLM_ARCHITECTURE.md](references/WLM_ARCHITECTURE.md)

### 3. Dynamic Visualizations & Coding Patterns
Instructions for dynamically constructing log-log scatter plots and Actual-vs-Optimal confusion matrix heatmaps using matplotlib, pandas, and BigFrames from first principles.
- **See**: [references/VISUALIZATION_PATTERNS.md](references/VISUALIZATION_PATTERNS.md)

### 4. Interactive Skill Delegation Matrix
How to dynamically detect and delegate sub-tasks to other installed environmental skills (such as `data-engineer`, `developing-with-bigquery`, `notebook-guidance`, etc.) to minimize token consumption.
- **See**: [references/DELEGATION_MATRIX.md](references/DELEGATION_MATRIX.md)

---

## Strict Safety Guardrails
- **Zero-Deletion Policy**: Under no circumstances shall this skill or any agent using it delete, truncate, or overwrite local reference scripts, reports, or remote BigQuery tables, datasets, schemas, or billing logs. Always apply safety checks matching `@skill:accidental-data-loss-prevention`.
- **Zero-Static-Copy Policy**: Never clone or hardcode static copies of analytical reports, spreadsheets, or charts inside this repository. All artifacts must be generated dynamically based on active metadata and customer environments.
- **Zero-Local-Pollution Policy (STRICT)**: To prevent polluting the user's workspace with untracked code files:
  - Do **NOT** write or commit temporary chart generation Python scripts (like `plot_job_level_charts.py`, `generate_charts.py` or other visualization tests) into the active workspace or repository folder. Creating `.py` helper files inside the repository is strictly forbidden.
  - Dynamically run plotting logic *only* through self-cleaning OS standard temporary directory structures (via the Python `tempfile` module), run directly inside a Jupyter Notebook cell (via `@skill:notebook-guidance`), or pipe inline scripts directly using terminal heredoc inputs (`python3 - <<'EOF'`). Any temporary code file must be completely invisible to the user and self-deleted immediately upon image generation.
