# ICU Mortality Prediction — eICU → MIMIC-III External Validation

Binary in-hospital mortality prediction for ICU patients, trained on the **eICU Collaborative Research Database** and externally validated on **MIMIC-III** — two datasets from different hospital systems, used here to test whether a model trained on one generalizes to real, unseen clinical data from another.

## Why this project

Most ICU mortality models in coursework/tutorials report performance on a held-out split of the *same* dataset they were trained on. That number is often optimistic — it doesn't tell you what happens when the model meets a different hospital, a different patient population, a different documentation style. This project trains on eICU and evaluates on MIMIC-III as a **true external test set**, then investigates and tries to close the resulting generalization gap.

## Data

| Dataset | Role | Patients | Mortality rate |
|---|---|---|---|
| eICU | Training + internal validation (80/20 split) | 2,520 | ~5% |
| MIMIC-III | External test (never used for training or model selection) | 136 | ~34% |

Features: demographics, ICU stay metadata, first-24h lab summary statistics, first-24h vital sign summary statistics, primary diagnosis (ICD-9 chapter). Full extraction logic in `notebooks/00_data_collection_*.ipynb`.

The two datasets differ substantially in case mix, base mortality rate, and missingness patterns (e.g. blood pressure is ~84% missing in eICU but only ~3% in MIMIC) — this shift is a central theme of the project, not an afterthought.

## Pipeline

| # | Notebook | What it does |
|---|---|---|
| 00 | `data_collection_eicu` / `data_collection_mimic` | Extract & harmonize raw features from both databases |
| 01 | `data_exploration` | Full EDA: missingness, class imbalance, distributions, dataset shift (KS tests), informative missingness |
| 02 | `data_preprocessing` | 80/20 split (train/val on eICU), outlier capping, imputation, encoding, scaling, SMOTE, feature stability filtering |
| 03 | `baseline_model` | Logistic Regression, Random Forest (GridSearch + calibration), XGBoost — evaluated on val, then MIMIC (single look) |
| 04 | `model_extensions` | Domain adaptation: label shift correction, importance weighting via domain classifier |
| 05 | `extended_baselines` | L1/L2 Logistic Regression, feature-selective RF (drops high-missingness BP/INR features), ensembling |
| 06 | `coral_adaptation` | CORAL covariance alignment, combined with importance weighting |
| 07 | `final_comparison` | Full method comparison with bootstrap 95% CIs, subgroup analysis (ICD-9, care unit), Decision Curve Analysis, honest summary |

## Key findings

- **The generalization gap is real**: the RF baseline reaches AUC 0.826 on the eICU validation split but only 0.734 on MIMIC — expected given the large prevalence gap (5% vs 34% mortality) and case-mix differences between the two cohorts.
- **Label shift correction (prevalence re-calibration) is the simplest and most reliable fix**, and it's doing essentially all of the useful work here. More elaborate domain adaptation didn't help further:
  - **Importance weighting** collapsed to near-uniform weights — the domain classifier separating eICU from MIMIC reached AUC 1.000 (near-perfect), so the resulting weights had an effective sample size of 2,009 out of 2,016 (99.7% retention) — i.e. essentially no reweighting happened.
  - **CORAL** alignment reduced the covariance distance between eICU and MIMIC by ~100% but MIMIC AUC didn't improve (0.733 vs 0.734 baseline) — confirming the gap isn't about feature distribution *shape*.
  - Combining CORAL with importance weighting *hurt* performance slightly (AUC 0.715).
- **Simpler isn't better here**: LR-L1/L2 had higher validation AUC (0.857–0.861 vs RF's 0.826) but *lower* MIMIC AUC (0.708–0.719) — the RF generalizes better despite being outperformed internally.
- **Dropping the highest-missingness features (BP/INR) didn't help** (MIMIC AUC 0.726, essentially unchanged) — removing features the model barely learned from doesn't unlock signal it never had.
- All bootstrap 95% CIs overlap across every method tested — none is statistically distinguishable from the label-shift-corrected baseline on this external test set (n=136).
- Subgroup analysis shows the model is weakest on SICU patients (AUC 0.675, n=34) and strongest on Respiratory admissions (AUC 0.875, n=19) — though sample sizes are small enough that these should be read as directional, not definitive.

## Results

All figures are on the **MIMIC-III external test set (n=136, 34% mortality)**, after label-shift correction. Sens/Spec use per-method thresholds tuned on the eICU validation set (Youden's J, shifted via the same log-odds transform) rather than one threshold borrowed from the RF baseline — see notebook 07 for why that distinction matters here.

| Method | Val AUC | MIMIC AUC (95% CI) | MIMIC PR-AUC | MIMIC Sens | MIMIC Spec |
|---|---|---|---|---|---|
| RF calibrated + label shift (baseline) | 0.826 | 0.734 (0.642–0.819) | 0.642 | 0.435 | 0.867 |
| + Importance weighting + label shift | 0.829 | 0.734 (0.645–0.817) | 0.642 | 0.435 | 0.878 |
| LR-L2 + label shift | 0.857 | 0.719 (0.619–0.811) | 0.621 | 0.478 | 0.844 |
| LR-L1 (sparse) + label shift | 0.861 | 0.708 (0.613–0.799) | 0.615 | 0.739 | 0.478 |
| RF, no BP/INR features + label shift | 0.834 | 0.726 (0.632–0.809) | 0.638 | 0.457 | 0.822 |
| CORAL + label shift | 0.814 | 0.733 (0.641–0.815) | 0.629 | 0.478 | 0.844 |
| CORAL + importance weighting + label shift | 0.824 | 0.715 (0.616–0.810) | 0.635 | 0.326 | 0.933 |

**Bottom line:** none of the domain adaptation techniques beat plain label-shift correction. That's a legitimate, useful negative result — it says the RF baseline's 0.734 external AUC is close to the practical ceiling for this feature set and sample size, not that more exotic methods weren't tried.

## Repo structure

```
notebooks/    — 00 through 07, run in order
src/          — shared helper functions (if any)
requirements.txt
```

## Running it

Developed in Google Colab (`google.colab.drive.mount`, paths under `/content/drive/MyDrive/...`). To run locally:
1. Download eICU and MIMIC-III (both require completing PhysioNet's credentialing process — not included in this repo).
2. Replace the Colab drive-mount cells with local paths.
3. Run notebooks 00 → 07 in order; each stage reads the previous stage's saved outputs.

`SEED = 42` throughout for reproducibility.

## Honest limitations

- MIMIC-III external test set is small (n=136) — bootstrap CIs are wide, and subgroup breakdowns with very small n (e.g. certain care units) are noted as unreliable rather than hidden.
- Sensitivity/specificity depend on a chosen decision threshold; AUC and PR-AUC (threshold-independent) are treated as the primary comparison metrics for this reason.
- This is a research/portfolio project, not a validated clinical tool.
