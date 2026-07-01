# BigQuery Cost & Pricing Analysis Guidelines

*Licensed under the Apache License, Version 2.0 (the "License"). You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0*

---

## 1. Timeline Correction & Job-Level Aggregation

When analyzing raw BigQuery logs, execute the correct SQL de-duplication rules. Raw timeline records are often sampled every few seconds, which can lead to a **30.8x overestimation** of data scanned if aggregated using simple row-level sums.

### The Aggregation Rule
To convert timeline samples into correct, job-level metrics, group by the unique `job_id`:
- **Bytes Billed**: Use `MAX(total_bytes_billed)` per job.
- **Slot Time (Seconds)**: Sum up the individual slot usage time. Use `SUM(total_slot_ms) / 1000`.

### Reference SQL: Dynamic JOBS Extractor
The following query shows how to dynamically query `INFORMATION_SCHEMA` based on selected scope:

```sql
-- Replace 'JOBS_BY_PROJECT' with JOBS_BY_USER, JOBS_BY_FOLDER, or JOBS_BY_ORGANIZATION depending on scope.
SELECT
  job_id,
  user_email,
  reservation_id,
  -- Correct de-duplicated bytes billed (MAX per job)
  MAX(total_bytes_billed) AS bytes_billed,
  -- Correct accumulated slot usage in seconds (SUM of ms / 1000)
  SUM(total_slot_ms) / 1000.0 AS slot_sec,
  -- Derived metrics
  SAFE_DIVIDE(SUM(total_slot_ms) / 1000.0, MAX(total_bytes_billed) / 1e12) AS slot_sec_per_tb,
  TIMESTAMP_DIFF(MAX(end_time), MIN(start_time), SECOND) AS duration_sec
FROM
  -- Dynamic view selection depending on scope
  `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE
  job_type = 'QUERY'
  AND statement_type != 'SCRIPT'
  AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY
  job_id,
  user_email,
  reservation_id;
```

---

## 2. Mathematical Break-Even Pricing Model

To evaluate whether a query or workload is more cost-effective under **On-Demand** pricing or **Capacity-Based Editions**, apply the following pricing formulas.

### Parameter Definitions
- **On-Demand Price**: $P_{od} = \$6.25 \text{ per TB}$ (or $\$5.00/\text{TB}$ for legacy plans).
- **Edition Slot Cost ($C$)**:
  - **Standard Edition**: $\$0.04 \text{ per slot-hour}$ (Standard baseline)
  - **Enterprise Edition**: $\$0.06 \text{ per slot-hour}$ (Enterprise baseline, including autoscale)
  - **Enterprise Plus Edition**: $\$0.10 \text{ per slot-hour}$ (Plus baseline, high security/compliance)

### The Break-Even Formula
The point where the cost of On-Demand matches the cost of Capacity-Based pricing is calculated as follows:

$$\text{Break-Even Cost}_{\text{On-Demand}} = \text{Break-Even Cost}_{\text{Capacity}}$$

$$P_{od} \times \text{Data Scanned (TB)} = C \times \left( \frac{\text{Slot Seconds}}{3600} \right)$$

$$\text{Break-Even Slot-Seconds per TB} = \frac{P_{od} \times 3600}{C}$$

### Break-Even Thresholds by Edition
Assuming On-Demand price $P_{od} = \$6.25 \text{ / TB}$:

1. **Standard Edition ($C = \$0.04$)**:
   $$\text{Break-Even Threshold} = \frac{6.25 \times 3600}{0.04} = 562,500 \text{ slot-seconds / TB}$$
2. **Enterprise Edition ($C = \$0.06$)**:
   $$\text{Break-Even Threshold} = \frac{6.25 \times 3600}{0.06} = 375,000 \text{ slot-seconds / TB}$$
3. **Enterprise Plus Edition ($C = \$0.10$)**:
   $$\text{Break-Even Threshold} = \frac{6.25 \times 3600}{0.10} = 225,000 \text{ slot-seconds / TB}$$

*Interpretation:* If a query consumes **fewer** slot-seconds per TB than the break-even threshold, it is **Capacity Favorable** (more efficient on a Reservation). If it consumes **more**, it is **On-Demand Favorable** (computational bottleneck, cheaper on On-Demand).

---

## 3. The 4-Quadrant Mismatch Interpretation Matrix & Localization

A confusion matrix comparing **Actual Execution Plan** (Capacity vs. On-Demand) and **Cost Model's Optimal Plan** is used to classify query logs. When generating reports or charts, dynamically localize the labels based on the target output language using the following mapping:

| Actual \ Optimal | Capacity (Optimal) | On-Demand (Optimal) |
| :--- | :--- | :--- |
| **Capacity (Reservation)** | **[1] Localized: [Optimal / 適正配置]**<br>Low-slot-ratio large queries executed on slot pools. | **[2] Localized: [Slot-Inefficient / スロット非効率クエリ]**<br>High-slot-ratio small queries on reservation. |
| **On-Demand** | **[3] Localized: [On-Demand Overpayment / オンデマンド過剰支払]**<br>Heavy queries running on On-Demand billing. | **[4] Localized: [Optimal / 適正配置]**<br>Lightweight high-intensity compute queries. |

### Classification Actions & Remedies (Multilingual Explanations)

#### Quadrant [1]: Capacity Actual & Capacity Optimal (JP: 【適正配置】 / EN: 【Optimal Allocation】)
* **Diagnosis**: High-volume, high-throughput queries efficiently routed.
* **Action**: Maintain current routing setup.

#### Quadrant [2]: Capacity Actual & On-Demand Optimal (JP: 【スロット非効率クエリ】 / EN: 【Slot-Inefficient Queries】)
* **Diagnosis**: High slot consumption relative to bytes scanned, but often zero-byte (cache hits, dry runs) or small queries (averaging < 500MB).
* **Action**: **No Action Required**. Even though they seem "inefficient" on paper, they run within the pre-purchased reservation capacity pool, incurring $\$0$ marginal cost. Switching them to On-Demand would expose the project to scan-billing risks for no net gain.

#### Quadrant [3]: On-Demand Actual & Capacity Optimal (JP: 【オンデマンド過剰支払】 / EN: 【On-Demand Overpayment】)
* **Diagnosis**: Massive queries scanning terabytes but using few slots, billed per-TB on On-Demand. Highly expensive.
* **Action**: **Immediate Routing Action**. Move these project assignments into the reservation capacity pool. Routing to reservations eliminates the per-scan billing and handles these heavy queries within the existing budget.

#### Quadrant [4]: On-Demand Actual & On-Demand Optimal (JP: 【適正配置】 / EN: 【Optimal Allocation】)
* **Diagnosis**: Computational queries with tiny data scans and short durations.
* **Action**: Continue on On-Demand or place inside a lower-priority reservation if concurrency constraints are tight.
