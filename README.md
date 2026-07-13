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

- **The generalization gap is real**: internal (eICU) performance is noticeably higher than external (MIMIC) performance — expected given the case-mix and prevalence differences between the two cohorts.
- **Missingness pattern shift is the biggest driver**, not feature distribution shift. Blood pressure is measured in eICU only ~16% of the time vs ~97% in MIMIC — the model never learned to use it, so MIMIC's real BP signal is wasted.
- **Label shift correction (prevalence re-calibration) is the simplest and most reliable fix.** More elaborate domain adaptation didn't help further:
  - Importance weighting collapsed to near-uniform weights (domain classifier AUC ≈ 1.0 — the two cohorts are *too* separable for density-ratio weighting to work).
  - CORAL alignment reduced covariance distance substantially but didn't improve AUC — confirming the gap isn't about feature distribution shape.
- All bootstrap confidence intervals across methods overlap — no method significantly beats the label-shift-corrected baseline on this external test set.

## Results

*(fill in with your final run's numbers)*

| Method | Val AUC | MIMIC AUC | MIMIC Sens | MIMIC Spec |
|---|---|---|---|---|
| RF calibrated + label shift (baseline) | | | | |
| LR-L1 (sparse) + label shift | | | | |
| RF, no BP/INR features + label shift | | | | |
| CORAL + label shift | | | | |
| CORAL + importance weighting + label shift | | | | |

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
