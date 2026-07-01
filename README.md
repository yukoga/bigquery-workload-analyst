# BigQuery Workload Analyst (`bigquery-workload-analyst`)

*Licensed under the Apache License, Version 2.0 (the "License"). You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0*

---

A repository-level custom skill for Visual Studio Code (GitHub Copilot / Claude Code) that empowers AI coding assistants with advanced BigQuery cost analysis, pricing model simulation, and workload management (WLM) capabilities.

## Key Features

1. **Mandatory IAM Check**: Ensures `roles/bigquery.resourceViewer` or higher is verified before running queries, preventing permission errors.
2. **Dynamic Scope Selection**: Automatically targets the correct `INFORMATION_SCHEMA` metadata views (`JOBS_BY_PROJECT`, `JOBS_BY_USER`, `JOBS_BY_FOLDER`, `JOBS_BY_ORGANIZATION`) based on user queries.
3. **Timeline Log De-duplication**: Aggregates sampling logs correctly at the `job_id` level using `MAX(bytes_billed)` and `SUM(slot_sec)` to prevent up to **30.8x overestimation** of scanned data.
4. **Mathematical Break-Even Pricing Model**: Simulates On-Demand vs. capacity-based Editions ($375,000 \text{ slot-seconds / TB}$ break-even threshold for Enterprise Edition).
5. **Workload Management (WLM) Architecture**: Delivers best-practice patterns for reservation mapping, query queuing, BI Engine acceleration, and runaway query cost limits.
6. **Ecosystem Delegation Matrix**: Automatically detects and leverages 27+ specialized environmental skills (e.g., `data-engineer`, `notebook-guidance`) to reduce token consumption.
7. **Strict Safety Guardrails**: Enforces a non-destructive zero-deletion rule for both local workspace scripts and cloud databases.

---

## Directory Structure

```
bigquery-workload-analyst/
├── LICENSE                    # Apache License 2.0
├── README.md                  # This file
├── SKILL.md                   # Main custom skill entry point and YAML frontmatter
├── skill-rules.json           # VS Code / Claude Code automatic trigger rules
└── references/                # Sub-guides (progressive disclosure)
    ├── ANALYSIS_GUIDELINES.md # De-duplication queries & math pricing formulas
    ├── DELEGATION_MATRIX.md   # Routing guide for environmental skills
    ├── VISUALIZATION_PATTERNS.md # Standard code blocks for charts (Scatter, Matrix)
    └── WLM_ARCHITECTURE.md    # Reservation, queueing, & cost control solutions
```

---

## Installation & Activation

To register this custom skill inside your active AI terminal or workspace:

1. **Claude Code Settings**: Add this folder's absolute path to your `.claude/settings.json` or custom skill load path.
2. **Copilot Settings**: Include this folder as part of your custom prompt/skill directory configuration.
3. **Automatic Triggering**: The triggers in `skill-rules.json` will automatically load this skill when keywords like `INFORMATION_SCHEMA.JOBS`, `bytes_billed`, `slot_sec`, or `workload management` are matched in your prompts.

---

## Contributing & License

Contributions are welcome! Please ensure any code added is properly formatted and contains Apache-2.0 License notices.

This project is licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file for the full text.
