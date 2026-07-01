# BigQuery Skill Delegation Matrix & Ecosystem Coordination

*Licensed under the Apache License, Version 2.0 (the "License"). You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0*

---

To minimize token utilization and ensure maximum system consistency, this skill acts as a **Workload & Cost Orchestrator**. When performing data engineering, querying, or visualization tasks, check if the following specialized environment skills exist in your context, and delegate the sub-tasks accordingly:

---

## 1. Core Data Engineering & Pipeline Skills

| Skill Name | Target Task / Delegation Purpose | Integration & Delegation Protocol |
| :--- | :--- | :--- |
| `data-engineer` | Designing pipelines, scheduling, and orchestrating massive datasets. | Delegate the design of recurring pipeline architectures or scheduled ingestion flows. |
| `data-engineering-data-pipeline` | Constructing robust batch/streaming data pipelines on GCP. | Use when establishing continuous audit log exports or automated ingestion of billing data. |
| `data-quality-frameworks` | Establishing tests, validation contracts, and expectations on retrieved logs. | Utilize to construct data quality checks verifying raw audit logs conform to expected schema. |
| `data-autocleaning` | Automated schema mapping, deduplication, and parsing of raw data. | Delegate cleaning steps when raw JSON payloads are extracted from audit log table fields. |

---

## 2. BigQuery & GCP Ecosystem Specific Skills

| Skill Name | Target Task / Delegation Purpose | Integration & Delegation Protocol |
| :--- | :--- | :--- |
| `developing-with-bigquery` | **Mandatory query tuning agent**. Fine-tuning heavy SQL scripts. | Delegate all low-level SQL optimization, partition/cluster analysis, and BQML. |
| `discovering-gcp-data-assets` | Locating dataset/table IDs and validating catalog schemas. | Use to resolve the exact project, location, and billing table ID of log endpoints. |
| `dataform-bigquery` / `dbt-bigquery` | Creating and managing transformed SQLX/dbt models for billing views. | Delegate the engineering of raw `INFORMATION_SCHEMA` exports into aggregated data marts. |
| `bigquery-data-transfer-service` | Automating cloud-to-cloud log exports or scheduling transfers. | Use when ingesting external SaaS or billing logs directly into BigQuery. |
| `gcloud-auth-verification` | Diagnosing authentication failures, missing ADC, or gcloud issues. | Trigger if pre-flight IAM role checks fail due to credential resolution errors. |
| `gcp-data-pipelines` | General entry point for ELT pipelines and GCP workflow orchestration. | Consult when structuring end-to-end data pipelines for analytical databases. |

---

## 3. Advanced Analytics, ML & Notebook Orchestration

| Skill Name | Target Task / Delegation Purpose | Integration & Delegation Protocol |
| :--- | :--- | :--- |
| `notebook-guidance` | **Mandatory Jupyter orchestration**. Running and validating code cells. | Delegate cell-by-cell generation, local virtualenv setup, and output validation. |
| `managing-python-dependencies` | Managing virtual environments, installing pip packages (e.g., matplotlib). | Delegate the creation of isolated virtualenvs before executing visualization scripts. |
| `data-scientist` | Advanced exploratory data analysis, forecasting, and statistical models. | Delegate when analyzing long-term workload trends or predictive cost modeling. |
| `ml-best-practices` | Model selection, predictive regression, and trend anomaly analysis. | Delegate when applying predictive algorithms or anomaly detection to slot demand. |
| `data-storytelling` | Formatting outputs, executive reporting, and creating impactful insights. | Utilize when organizing analytical output into structured reports for business leaders. |
| `building-data-apps` | Creating interactive Streamlit or React frontends for cost dashboards. | Delegate if the user requests an interactive visual web interface to explore logs. |

---

## 4. Safety & System Stability Skills

| Skill Name | Target Task / Delegation Purpose | Integration & Delegation Protocol |
| :--- | :--- | :--- |
| `accidental-data-loss-prevention` | Enforcing safety policies during database and file manipulations. | **Mandatory Gatekeeper**. Ensure zero deletion of workspace files or cloud resources. |
| `skill-repair` | Fixing broken/corrupted skill definitions or manifest mismatches. | Trigger if skill registrations, settings, or rules fail during setup or execution. |

---

## 5. Execution Routine for Dynamic Skill Detection
When this skill receives an analysis request, execute this simple discovery query:
1. Scan the environment and list active skills.
2. If `notebook-guidance` is present, immediately scaffold the analytics in a `.ipynb` format using `%%bqsql` and modular Python visualization cells.
3. If `developing-with-bigquery` is present, route the optimized extraction queries to it for cost-based verification and cluster recommendations.
4. If `accidental-data-loss-prevention` is present, attach its strict confirmation boundaries to any workspace modification tasks.
