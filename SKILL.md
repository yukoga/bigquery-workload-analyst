---
name: bigquery-workload-analyst
description: |
  Analyze BigQuery cost consumption, simulate pricing models (On-Demand vs. Capacity/Editions), validate resourceViewer permissions, dynamically query INFORMATION_SCHEMA.JOBS across project/user/folder/org scopes, and architect workload management solutions (reservations, queueing, BI Engine, cost-safeguards).
license: Apache-2.0
metadata:
  version: v1.1.0
  publisher: yukoga
---

# BigQuery Workload Analyst Skill

This standalone skill handles BigQuery log diagnostics, Edition vs. On-Demand pricing simulations, permission verification, and advanced Workload Management (WLM) designs under Apache License 2.0.

---

## 1. Mandatory Workflow & Permission Check

### Step 1: Pre-flight Permission Gate
Verify permissions BEFORE running queries.
- **Required**: `bigquery.jobs.list` / `bigquery.metadata.viewer` (satisfied by `roles/bigquery.resourceViewer` or higher like `admin`).
- **Failure Handling**: If missing, halt immediately. Output these instructions and wait:
  ```bash
  gcloud projects add-iam-policy-binding [PROJECT_ID] --member="user:[USER_EMAIL]" --role="roles/bigquery.resourceViewer"
  ```

### Step 2: Dynamic Scope & View Selection
Select target metadata view based on query context:
- **Project**: `INFORMATION_SCHEMA.JOBS_BY_PROJECT` (or standard `INFORMATION_SCHEMA.JOBS`)
- **User**: `INFORMATION_SCHEMA.JOBS_BY_USER`
- **Folder**: `INFORMATION_SCHEMA.JOBS_BY_FOLDER`
- **Organization**: `INFORMATION_SCHEMA.JOBS_BY_ORGANIZATION`

### Step 3: Timeline Log De-duplication
Timeline logs are sampled every few seconds. Aggregating rows directly overestimates data scanned by **up to 30.8x**.
- **Rule**: Always group by `job_id`. Calculate `bytes_billed` as `MAX(total_bytes_billed)` and `slot_sec` as `SUM(total_slot_ms)/1000`.

### Step 4: Dynamic Language & Localization Tuning
- **Language Detection**: Automatically output reports, comments, and plots in the user's preferred language (e.g. English `'en'`, Japanese `'ja'`).
- **Term Localization Map**:
  - `Capacity Favorable` ⇄ `容量枠が有利なジョブ`
  - `On-Demand Favorable` ⇄ `オンデマンドが有利なジョブ`
  - `[Optimal]` ⇄ `【適正配置】`
  - `[Slot-Inefficient]` ⇄ `【スロット非効率クエリ】`
  - `[On-Demand Overpay]` ⇄ `【オンデマンド過剰支払】`
- **Tofu Prevention**: Set standard OS fonts for plots dynamically: `'Arial Unicode MS'` on macOS, `'MS Gothic'` on Windows, `'IPAexGothic'` on Linux.
- **Mandatory Chart Linkage in Reports**: When generating a final analytical or cost optimization report (in Markdown or HTML), the agent **MUST** ensure that the code blocks from this skill are executed to generate the plots, and both resulting chart images (`bq_breakeven_chart.png` and `bq_confusion_matrix.png`) are physically embedded and linked as visual evidence in the report (using correct relative image links `![](./bq_breakeven_chart.png)` and `![](./bq_confusion_matrix.png)`).

---

## 2. Mathematical Break-Even Pricing Model

Compare On-Demand ($P_{od} = \$6.25 \text{ / TB}$) vs. Capacity Editions hourly slot pricing ($C$).

$$\text{Break-Even Slot-Seconds per TB} = \frac{P_{od} \times 3600}{C} = \frac{22,500}{C}$$

- **Standard Edition ($C = \$0.04$)**: $562,500 \text{ slot-seconds / TB}$
- **Enterprise Edition ($C = \$0.06$)**: $375,000 \text{ slot-seconds / TB}$
- **Enterprise Plus Edition ($C = \$0.10$)**: $225,000 \text{ slot-seconds / TB}$

---

## 3. Workload Management (WLM) Architecture

1. **Workload Isolation (Reservations)**:
   - `prod-reservation` (Baseline: 300 slots, Autoscaling, Priority: HIGH) for customer APIs & dashboards.
   - `batch-reservation` (Baseline: 100 slots, Autoscaling, Priority: LOW) for scheduled ETL.
   - `adhoc-reservation` (Baseline: 0 slots, pure Autoscaling, max 200) for development.
2. **Dynamic Query Queuing**: Set low timeouts (`queue_timeout_ms=120000`) for dashboards to fail fast, and generous timeouts (`14400000` / 4 hours) for batch reservations to buffer concurrency peaks.
3. **BI Engine Memory Allocation**: Allocate up to 100GB of memory for fast dashboard rendering. Ensure queries avoid non-accelerated operators to consume **0 slots** from the reservation pool.
4. **Safeguards**: Enforce `maximum_bytes_billed` limits on sandbox connections and set daily project-level scanning quotas.

---

## 4. Standalone Plot Generation Code (With Anti-Workspace Pollution)

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

def generate_log_log_scatter(df, threshold_slot_sec_per_tb=375000, output_path="bq_breakeven_chart.png", lang="ja", os_platform="macOS"):
    font_families = {"macOS": ["Arial Unicode MS", "AppleGothic", "sans-serif"], "Windows": ["MS Gothic", "sans-serif"], "Linux": ["IPAexGothic", "sans-serif"]}
    plt.rcParams["font.family"] = font_families.get(os_platform, ["sans-serif"])
    labels = {
        "en": {
            "title": "BigQuery Cost Model Actual vs. Optimal Break-Even Analysis", "xlabel": "Slot Consumption (slot_sec) [Log Scale]", "ylabel": "Scanned Data Volume (TB) [Log Scale]",
            "cap_actual": "Actual: Capacity Pool (Reservation)", "od_actual": "Actual: On-Demand Execution", "breakeven_ent": f"Break-Even (Enterprise: {threshold_slot_sec_per_tb:,} slots/TB)",
            "cap_zone": "Capacity Favorable Zone", "od_zone": "On-Demand Favorable Zone"
        },
        "ja": {
            "title": "BigQuery 料金モデル実績 vs 最適判断 損益分岐分析", "xlabel": "消費スロット秒 (slot_sec) [対数スケール]", "ylabel": "課金データスキャン量 (TB) [対数スケール]",
            "cap_actual": "実績：容量枠での実行 (Capacity)", "od_actual": "実績：オンデマンドでの実行 (On-Demand)", "breakeven_ent": f"損益分岐点 (Enterprise: {threshold_slot_sec_per_tb:,} slots/TB)",
            "cap_zone": "容量枠 (Capacity) 有利ゾーン", "od_zone": "オンデマンド有利ゾーン"
        }
    }
    lang_labels = labels.get(lang, labels["en"])
    df_clean = df[(df["data_tb"] > 0) & (df["slot_sec"] > 0)].copy()
    df_sample = pd.concat([df_clean[df_clean["data_tb"] > 1.0], df_clean[df_clean["data_tb"] <= 1.0].sample(n=min(10000, len(df_clean[df_clean["data_tb"] <= 1.0])), random_state=42)])
    df_cap, df_od = df_sample[df_sample["actual"] == "Capacity"], df_sample[df_sample["actual"] == "OnDemand"]
    
    plt.figure(figsize=(10, 8), dpi=150)
    plt.scatter(df_cap["slot_sec"], df_cap["data_tb"], color="#4285f4", alpha=0.3, s=6, label=lang_labels["cap_actual"])
    plt.scatter(df_od["slot_sec"], df_od["data_tb"], color="#f97316", alpha=0.8, s=35, edgecolors="#ffffff", linewidths=0.5, label=lang_labels["od_actual"])
    
    x_line = np.logspace(np.log10(df_clean["slot_sec"].min()), np.log10(df_clean["slot_sec"].max()), 100)
    y_enterprise, y_standard = x_line / threshold_slot_sec_per_tb, x_line / 562500
    plt.loglog(x_line, y_enterprise, color="#4285f4", linestyle="-", linewidth=2, label=lang_labels["breakeven_ent"])
    standard_label = "Break-Even (Standard: 562,500 slots/TB)" if lang == "en" else "損益分岐点 (Standard: $0.04/hr)"
    plt.loglog(x_line, y_standard, color="#fbbc05", linestyle="--", linewidth=1.5, label=standard_label)
    
    plt.fill_between(x_line, y_enterprise, 1e4, color="#34a853", alpha=0.05, label=lang_labels["cap_zone"])
    plt.fill_between(x_line, 1e-12, y_enterprise, color="#ea4335", alpha=0.05, label=lang_labels["od_zone"])
    plt.xscale("log"); plt.yscale("log")
    plt.xlim(df_clean["slot_sec"].min() * 0.5, df_clean["slot_sec"].max() * 2)
    plt.ylim(df_clean["data_tb"].min() * 0.5, df_clean["data_tb"].max() * 2)
    plt.xlabel(lang_labels["xlabel"], fontsize=11, fontweight="bold")
    plt.ylabel(lang_labels["ylabel"], fontsize=11, fontweight="bold")
    plt.title(lang_labels["title"], fontsize=13, fontweight="bold")
    
    handles, labels_list = plt.gca().get_legend_handles_labels()
    by_label = dict(zip(labels_list, handles))
    legend_order = [lang_labels["cap_actual"], lang_labels["od_actual"], lang_labels["breakeven_ent"], standard_label, lang_labels["od_zone"], lang_labels["cap_zone"]]
    plt.legend([by_label[lbl] for lbl in legend_order if lbl in by_label], [lbl for lbl in legend_order if lbl in by_label], loc="upper left", frameon=True, facecolor="white", edgecolor="#e0e0e0", fontsize=10)
    plt.tight_layout(); plt.savefig(output_path, dpi=200); plt.close()

def generate_confusion_matrix(df, threshold_slot_sec_per_tb=375000, output_path="bq_confusion_matrix.png", lang="ja", os_platform="macOS"):
    font_families = {"macOS": ["Arial Unicode MS", "AppleGothic", "sans-serif"], "Windows": ["MS Gothic", "sans-serif"], "Linux": ["IPAexGothic", "sans-serif"]}
    plt.rcParams["font.family"] = font_families.get(os_platform, ["sans-serif"])
    is_od_optimal = (df["bytes_billed"].isna()) | (df["bytes_billed"] == 0) | (df["slot_sec"] / df["data_tb"] > threshold_slot_sec_per_tb)
    df["optimal"] = np.where(is_od_optimal, "OnDemand", "Capacity")
    cm = pd.crosstab(df["actual"], df["optimal"]).reindex(index=["Capacity", "OnDemand"], columns=["Capacity", "OnDemand"], fill_value=0)
    pct, cnt = (cm / len(df) * 100).values, cm.values
    
    labels = {
        "en": {
            "title": "BigQuery Pricing Model Actual vs. Optimal Simulation", "xlabel": "Optimal Pricing Model (Simulated)", "ylabel": "Actual Pricing Plan",
            "xticklabels": ["Capacity Favorable\nJobs", "On-Demand Favorable\nJobs"], "yticklabels": ["Actual: On-Demand\n", "Actual: Capacity\n(Reservation)"],
            "cells": [
                [f"Optimal: Capacity\n\n{pct[0, 0]:.2f}%\n({cnt[0, 0]:,} jobs)\n\n[Optimal]", f"Optimal: On-Demand\n\n{pct[0, 1]:.2f}%\n({cnt[0, 1]:,} jobs)\n\n[Slot-Inefficient]"],
                [f"Optimal: Capacity\n\n{pct[1, 0]:.4f}%\n({cnt[1, 0]:} jobs)\n\n[On-Demand Overpay]", f"Optimal: On-Demand\n\n{pct[1, 1]:.4f}%\n({cnt[1, 1]:} jobs)\n\n[Optimal]"]
            ]
        },
        "ja": {
            "title": "BigQuery 料金モデル実績 vs 最適シミュレーション", "xlabel": "コストモデル上の最適判断 (Optimal Pricing Model)", "ylabel": "実際の実行プラン (Actual Pricing Plan)",
            "xticklabels": ["容量枠が有利なジョブ\n(Capacity Favorable)", "オンデマンドが有利なジョブ\n(On-Demand Favorable)"], "yticklabels": ["実績：オンデマンド\n(Actual On-Demand)", "実績：容量枠 / 予約枠\n(Actual Capacity-Based)"],
            "cells": [
                [f"最適：容量枠料金 (Capacity)\n\n{pct[0, 0]:.2f}%\n({cnt[0, 0]:,} 件)\n\n【適正配置】", f"最適：オンデマンド (OnDemand)\n\n{pct[0, 1]:.2f}%\n({cnt[0, 1]:,} 件)\n\n【スロット非効率クエリ】"],
                [f"最適：容量枠料金 (Capacity)\n\n{pct[1, 0]:.4f}%\n({cnt[1, 0]:} 件)\n\n【オンデマンド過剰支払】", f"最適：オンデマンド (OnDemand)\n\n{pct[1, 1]:.4f}%\n({cnt[1, 1]:} 件)\n\n【適正配置】"]
            ]
        }
    }
    lang_labels = labels.get(lang, labels["en"])
    
    fig, ax = plt.subplots(figsize=(8, 7), dpi=200)
    colors, text_colors = [["#e6f4ea", "#fce8e6"], ["#fce8e6", "#e6f4ea"]], [["#137333", "#c5221f"], ["#c5221f", "#137333"]]
    for i in range(2):
        for j in range(2):
            rect = plt.Rectangle((j, 1 - i), 1, 1, facecolor=colors[i][j], edgecolor="#bdc1c6", linewidth=1.5)
            ax.add_patch(rect)
            ax.text(
                j + 0.5, 1.5 - i, lang_labels["cells"][i][j],
                ha="center", va="center", color=text_colors[i][j],
                fontsize=10.5, fontweight="bold", linespacing=1.4
            )
            
    ax.set_xlim(0, 2); ax.set_ylim(0, 2); ax.set_xticks([0.5, 1.5]); ax.set_xticklabels(lang_labels["xticklabels"], fontsize=11, fontweight="bold")
    ax.xaxis.tick_top(); ax.xaxis.set_label_position("top"); ax.set_xlabel(lang_labels["xlabel"], fontsize=12, fontweight="bold")
    ax.set_yticks([0.5, 1.5]); ax.set_yticklabels(lang_labels["yticklabels"], fontsize=11, fontweight="bold", rotation=90, va="center")
    ax.set_ylabel(lang_labels["ylabel"], fontsize=12, fontweight="bold")
    ax.tick_params(axis="both", which="both", length=0)
    for spine in ax.spines.values():
         spine.set_visible(False)
    plt.axvline(x=1, color="#bdc1c6", linewidth=1.5); plt.axhline(y=1, color="#bdc1c6", linewidth=1.5)
    plt.title(lang_labels["title"], fontsize=14, fontweight="bold", pad=40)
    plt.tight_layout(); plt.savefig(output_path, dpi=200); plt.close()
```

### ⚠️ Workspace Clutter Prevention Rule (STRICT)
To prevent creating standalone helper files (like `generate_charts.py` or temporary test scripts) in the user's workspace directory, the agent **MUST** execute the Python visualization code using one of the following cleaner methods. Writing or saving `.py` script files to the active repository or workspace directories is strictly prohibited:
1. **Interactive Notebook Cell (Strongly Preferred)**: Run the plotting code cell-by-cell directly inside a Jupyter Notebook (.ipynb) using the `@skill:notebook-guidance` framework. This keeps all execution logic contained inside the notebook file.
2. **Standard Python `tempfile` Module (No Workspace Writing)**: Instead of manually writing to hardcoded directories, use Python's built-in `tempfile.NamedTemporaryFile` or `tempfile.TemporaryDirectory` module. This ensures scripts are saved to the OS standard hidden temporary folders (completely invisible to the user) and are automatically destroyed by Python as soon as execution completes.
   ```python
   import tempfile
   import os
   # Example pattern for running and immediately self-deleting script:
   with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
       f.write(python_code_block)
       temp_path = f.name
   # Execute temp_path then...
   os.unlink(temp_path)
   ```
3. **Inline Terminal Execution (Heredoc)**: Run the script inline through the terminal without writing any file to disk by piping a heredoc directly to Python:
   ```bash
   python3 - <<'EOF'
   # matplotlib plotting code goes here...
   EOF
   ```

---

## 5. Ecosystem & Environmental Skill Routing

Coordinate seamlessly with active workspace skills. When any of these are present, delegate specific tasks to minimize token consumption:
- `data-engineer` | `data-engineering-data-pipeline` | `data-quality-frameworks` | `data-autocleaning`
- `developing-with-bigquery` | `discovering-gcp-data-assets` | `dataform-bigquery` | `dbt-bigquery` | `bigquery-data-transfer-service`
- `notebook-guidance` | `managing-python-dependencies` | `data-scientist` | `ml-best-practices` | `data-storytelling` | `building-data-apps`
- `accidental-data-loss-prevention` | `skill-repair`

---

## Strict Safety Guardrails
- **Zero-Deletion Policy**: Under no circumstances shall this skill or any agent using it delete, truncate, or overwrite local reference scripts, reports, or remote BigQuery tables, datasets, schemas, or billing logs. Always apply safety checks matching `@skill:accidental-data-loss-prevention`.
- **Zero-Static-Copy Policy**: Never clone or hardcode static copies of analytical reports, spreadsheets, or charts inside this repository. All artifacts must be generated dynamically based on active telemetry and customer environments.
- **Zero-Local-Pollution Policy (STRICT)**: To prevent polluting the user's workspace with untracked code files:
  - Do **NOT** write or commit temporary chart generation Python scripts (like `plot_job_level_charts.py`, `generate_charts.py` or other visualization tests) into the active workspace or repository folder. Creating `.py` helper files inside the repository is strictly forbidden.
  - Dynamically run plotting logic *only* through self-cleaning OS standard temporary directory structures (via the Python `tempfile` module), run directly inside a Jupyter Notebook cell (via `@skill:notebook-guidance`), or pipe inline scripts directly using terminal heredoc inputs (`python3 - <<'EOF'`). Any temporary code file must be completely invisible to the user and self-deleted immediately upon image generation.
