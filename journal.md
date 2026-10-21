# Engineering Log: Home Credit Default Risk

## 1. Project Setup & Validation Architecture
* **Objective:** Predict probability of loan default (binary classification, highly imbalanced at 8.07% positive class rate).
* **Validation Strategy:** Stratified 5-Fold Cross-Validation. Stratification locks the exact 8.07% target distribution across all folds, eliminating fold-to-fold drift.
* **Evaluation Metrics:**
  * **ROC-AUC:** Primary metric for discrimination and rank-ordering across all thresholds.
  * **Kolmogorov-Smirnov (KS) Statistic:** Primary operational banking metric measuring the maximum percentage point separation between cumulative caught defaulters and cumulative approved repayers.
* **Initial Prior:** LightGBM logs confirmed initial baseline log-odds:
  $$\ln\left(\frac{0.080729}{1 - 0.080729}\right) = -2.432486$$

---

## 2. Root Table Preprocessing & Feature Engineering (`application_train`)

### Data Sanitation & Artifact Isolation
* **`DAYS_EMPLOYED = 365,243`:** Identified this 1,000-year figure as an administrative sentinel encoding non-working applicants (retirees/homemakers). 
* **Mitigation:** Rather than dropping rows or allowing extreme values to distort continuous tree thresholds, flagged rows using a binary indicator (`DAYS_EMPLOYED_ANOM = 1`) and replaced the sentinel in the continuous column with `NaN`. This isolated the retiree risk pool (5.40% default rate) while restoring monotonic risk decay for active workers.

### Baseline Domain Ratios
* `RATIO_CREDIT_TO_INCOME`: Measures gross leverage (median: 3.27, 99th percentile: 13.0).
* `RATIO_ANNUITY_TO_INCOME`: Measures payment burden (median: 16.3% of annual earnings).
* `RATIO_CREDIT_TO_ANNUITY`: Measures expected repayment duration / installment horizon (peaked at 20 cycles). Ranked #3 in overall feature importance, outperforming over 115 raw native features.

---

## 3. The Root Table Ceiling Experiment
* **Hypothesis:** Composite interactions across credit bureau scores (`EXT_SOURCES_MEAN`, `STD`, `MIN`, `MAX`, `PROD`, `COUNT`, `EXT_SOURCE_TO_AGE`) would boost application table signal.
* **Result:** AUC increased from **0.76694** to **0.76699** (+0.00005 AUC; KS dropped from 39.74% to 39.65%).
* **Architectural Decision:** Recognized that `application_train` had reached its predictive ceiling. To avoid feature bloat and collinearity under tree column subsampling (`colsample_bytree = 0.8`), abandoned root-table feature stacking and transitioned to relational tables.

---

## 4. Relational Data Engineering (`bureau.csv`)

### Architecture & Integrity
* **Table Grain:** 1,716,428 external credit records mapped to 307,511 applicants (1:N relationship, ~5.6 loans/applicant).
* **Join Rule:** Enforced `how='left'` on `SK_ID_CURR` after performing `groupby` aggregations. Prevented row explosion (fan-out) and protected 44,020 "credit invisible" applicants (14.31% thin-file rate) from deletion. Filled loan count missingness with `0` while preserving informative `NaN` values for conditional metrics (e.g., days overdue).

### Iteration 1: Unsegmented Bureau Aggregations
* Grouped by `SK_ID_CURR` across debt balances, overdue days, prolongations, and inquiry recency.
* `BUREAU_DAYS_CREDIT_MAX` emerged as the #1 bureau feature, confirming that **credit hunger** (recency of seeking credit) provides higher split gain than raw debt volume.

### Iteration 2: Active vs. Closed Segmentation
* Segmented records on `CREDIT_ACTIVE == 'Active'` to prevent settled historical loans from dampening open risk exposure.
* **Engineered Ratios:**
  * `RATIO_BUREAU_ACTIVE_LOANS`: Active loans relative to total bureau history (exposure rate).
  * `RATIO_TOTAL_SYSTEM_DEBT_TO_INCOME`: System-wide debt leverage:
    $$\frac{\text{AMT\_CREDIT (Current)} + \text{BUREAU\_ACTIVE\_AMT\_CREDIT\_SUM\_DEBT\_SUM}}{\text{AMT\_INCOME\_TOTAL}}$$

---

## 5. Scorecard & Progression Matrix

| Iteration | Description | OOF ROC-AUC | OOF KS (%) | Delta (AUC) |
|---|---|---|---|---|
| **V0** | Baseline Model (Application + Core Ratios) | 0.76694 | 39.74% | Reference Floor |
| **V1** | Root Table EXT_SOURCE Interactions | 0.76699 | 39.65% | +0.00005 (Plateau) |
| **V2** | Bureau Aggregations (All Loans) | 0.76974 | 40.20% | +0.00280 |
| **V3** | Bureau Segmented (Active Debt + Systemic DTI) | **0.77126** | **40.39%** | **+0.00432** |

---

## 6. Next Actions (Upcoming Session)
1. Ingest `previous_application.csv` to capture internal Home Credit institutional memory.
2. Engineer internal refusal rate features (`NAME_CONTRACT_STATUS == 'Refused'`).
3. Compute lender haircut metrics: $\frac{\text{AMT\_CREDIT}}{\text{AMT\_APPLICATION}}$ (requested vs. approved amount).