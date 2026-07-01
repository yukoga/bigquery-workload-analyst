# BigQuery Dynamic Visualization Patterns

*Licensed under the Apache License, Version 2.0 (the "License"). You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0*

---

This guide provides clean, optimized, and standalone Python templates for dynamically generating cost analytical plots from first principles. **NEVER** clone or copy static report charts or CSVs; instead, execute these scripts inside a notebook or script to render visualization outputs based on real query logs.

---

## 1. Log-Log Scatter Plot Template (Actual vs. Optimal)

This visualization maps each query's data scanned (TB) against its slot consumption (seconds) on a logarithmic scale, highlighting the break-even boundaries.

```python
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt

def generate_log_log_scatter(df, threshold_slot_sec_per_tb=375000, output_path="bq_breakeven_chart.png"):
    """
    df must contain columns:
      - 'data_tb': Data scan volume in TB (bytes_billed / 1e12)
      - 'slot_sec': Total slot-seconds (total_slot_ms / 1000)
      - 'actual_plan': 'Capacity' or 'OnDemand' representing the real execution model
    """
    # Ensure Japanese text/fonts are supported on macOS (if applicable)
    plt.rcParams["font.family"] = "Arial Unicode MS"
    
    # Filter out positive values only for log scale
    df_clean = df[(df["data_tb"] > 0) & (df["slot_sec"] > 0)].copy()
    
    # Stratified Sampling to protect memory and render readability
    df_large = df_clean[df_clean["data_tb"] > 1.0]
    df_small = df_clean[df_clean["data_tb"] <= 1.0].sample(
        n=min(10000, len(df_clean[df_clean["data_tb"] <= 1.0])), random_state=42
    )
    df_sample = pd.concat([df_large, df_small])
    
    df_cap = df_sample[df_sample["actual"] == "Capacity"]
    df_od = df_sample[df_sample["actual"] == "OnDemand"]
    
    plt.figure(figsize=(10, 8), dpi=150)
    
    # Plot Capacity actual points (Blue)
    plt.scatter(
        df_cap["slot_sec"], df_cap["data_tb"],
        color="#4285f4", alpha=0.3, s=8, label="実績：容量枠での実行 (Capacity)"
    )
    
    # Plot On-Demand actual points (Orange)
    plt.scatter(
        df_od["slot_sec"], df_od["data_tb"],
        color="#f97316", alpha=0.8, s=35, edgecolors="#ffffff", linewidths=0.5,
        label="実績：オンデマンドでの実行 (On-Demand)"
    )
    
    # Plot mathematical Break-Even Lines
    x_line = np.logspace(np.log10(df_clean["slot_sec"].min()), np.log10(df_clean["slot_sec"].max()), 100)
    y_breakeven = x_line / threshold_slot_sec_per_tb
    
    plt.loglog(
        x_line, y_breakeven,
        color="#4285f4", linestyle="-", linewidth=2,
        label=f"損益分岐点 (Enterprise: {threshold_slot_sec_per_tb:,} slots/TB)"
    )
    
    # Fill optimal zones
    plt.fill_between(x_line, y_breakeven, 1e4, color="#34a853", alpha=0.05, label="容量枠 (Capacity) 有利ゾーン")
    plt.fill_between(x_line, 1e-12, y_breakeven, color="#ea4335", alpha=0.05, label="オンデマンド有利ゾーン")
    
    plt.xscale("log")
    plt.yscale("log")
    plt.xlim(df_clean["slot_sec"].min() * 0.5, df_clean["slot_sec"].max() * 2)
    plt.ylim(df_clean["data_tb"].min() * 0.5, df_clean["data_tb"].max() * 2)
    
    plt.xlabel("消費スロット秒 (slot_sec) [対数スケール]", fontsize=11, fontweight="bold", labelpad=10)
    plt.ylabel("課金データスキャン量 (TB) [対数スケール]", fontsize=11, fontweight="bold", labelpad=10)
    plt.title("BigQuery 料金モデル実績 vs 最適判断 損益分岐分析\n(実際の実行プラン vs 理論上の最適プラン領域)", fontsize=13, fontweight="bold", pad=15)
    
    plt.legend(loc="upper left", frameon=True, facecolor="white", edgecolor="#e0e0e0", fontsize=10)
    plt.tight_layout()
    plt.savefig(output_path, dpi=200)
    plt.close()
```

---

## 2. Dynamic Confusion Matrix Heatmap Template

This script produces a clean, cell-annotated 2x2 confusion matrix that segments actual execution plans against the cost model's theoretical optimums.

```python
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

def generate_confusion_matrix(df, threshold_slot_sec_per_tb=375000, output_path="bq_confusion_matrix.png"):
    """
    df must contain:
      - 'bytes_billed': Raw bytes billed
      - 'data_tb': Bytes / 1e12
      - 'slot_sec': Slot-seconds
      - 'actual': 'Capacity' or 'OnDemand'
    """
    plt.rcParams["font.family"] = "Arial Unicode MS"
    
    # Calculate optimal plans based on math thresholds
    is_od_optimal = (df["bytes_billed"].isna()) | (df["bytes_billed"] == 0) | (df["slot_sec"] / df["data_tb"] > threshold_slot_sec_per_tb)
    df["optimal"] = np.where(is_od_optimal, "OnDemand", "Capacity")
    
    # Generate Crosstab
    cm = pd.crosstab(df["actual"], df["optimal"])
    cm = cm.reindex(index=["Capacity", "OnDemand"], columns=["Capacity", "OnDemand"], fill_value=0)
    
    total = len(df)
    pct = (cm / total * 100).values
    cnt = cm.values
    
    # Text labels mapping correctly to the 4-quadrant cost model
    cell_labels = [
        [
            f"最適：容量枠料金 (Capacity)\n\n{pct[0, 0]:.2f}%\n({cnt[0, 0]:,} 件)\n\n【適正配置】",
            f"最適：オンデマンド (OnDemand)\n\n{pct[0, 1]:.2f}%\n({cnt[0, 1]:,} 件)\n\n【スロット非効率クエリ】"
        ],
        [
            f"最適：容量枠料金 (Capacity)\n\n{pct[1, 0]:.4f}%\n({cnt[1, 0]:} 件)\n\n【オンデマンド過剰支払】",
            f"最適：オンデマンド (OnDemand)\n\n{pct[1, 1]:.4f}%\n({cnt[1, 1]:} 件)\n\n【適正配置】"
        ]
    ]
    
    fig, ax = plt.subplots(figsize=(8, 7), dpi=200)
    colors = [
        ["#e6f4ea", "#fce8e6"],  # Green (Optimal), Red (Suboptimal)
        ["#fce8e6", "#e6f4ea"]
    ]
    text_colors = [["#137333", "#c5221f"], ["#c5221f", "#137333"]]
    
    for i in range(2):
        for j in range(2):
            rect = plt.Rectangle((j, 1 - i), 1, 1, facecolor=colors[i][j], edgecolor="#bdc1c6", linewidth=1.5)
            ax.add_patch(rect)
            ax.text(
                j + 0.5, 1.5 - i, cell_labels[i][j],
                ha="center", va="center", color=text_colors[i][j],
                fontsize=10.5, fontweight="bold", linespacing=1.4
            )
            
    ax.set_xlim(0, 2)
    ax.set_ylim(0, 2)
    ax.set_xticks([0.5, 1.5])
    ax.set_xticklabels(["容量枠が有利なジョブ\n(Capacity Favorable)", "オンデマンドが有利なジョブ\n(On-Demand Favorable)"], fontsize=11, fontweight="bold")
    ax.xaxis.tick_top()
    ax.xaxis.set_label_position("top")
    ax.set_xlabel("コストモデル上の最適判断 (Optimal Pricing Model)", fontsize=12, fontweight="bold", labelpad=15)
    
    ax.set_yticks([0.5, 1.5])
    ax.set_yticklabels(["実績：オンデマンド\n(Actual On-Demand)", "実績：容量枠 / 予約枠\n(Actual Capacity-Based)"], fontsize=11, fontweight="bold", rotation=90, va="center")
    ax.set_ylabel("実際の実行プラン (Actual Pricing Plan)", fontsize=12, fontweight="bold", labelpad=20)
    
    ax.tick_params(axis="both", which="both", length=0)
    for spine in ax.spines.values():
        spine.set_visible(False)
        
    plt.axvline(x=1, color="#bdc1c6", linewidth=1.5)
    plt.axhline(y=1, color="#bdc1c6", linewidth=1.5)
    plt.title("BigQuery 料金モデル実績 vs 最適シミュレーション\n(ジョブ単位・混同行列マトリクス)", fontsize=14, fontweight="bold", pad=40)
    
    plt.tight_layout()
    plt.savefig(output_path, dpi=200)
    plt.close()
```
