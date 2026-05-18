# ICU Mortality Prediction
### eICU → MIMIC-III External Validation

Predict in-hospital mortality for ICU patients using clinical data from the **eICU Collaborative Research Database**, with external validation on **MIMIC-III**.

---

## Clinical Motivation

Early and accurate mortality prediction in the ICU enables clinicians to:
- Prioritise high-risk patients for closer monitoring and intervention
- Allocate scarce ICU resources more efficiently
- Benchmark ICU performance against risk-adjusted expected mortality

---

## Datasets

| Dataset | Role | Patients | Source |
|---|---|---|---|
| eICU Collaborative Research Database | Training | 2,520 | PhysioNet |
| MIMIC-III | External Validation | 135 | PhysioNet |

Both datasets require credentialed access via [PhysioNet](https://physionet.org). They are **not included** in this repository.

### Features (73 shared columns)
- **Demographics:** age, gender, ethnicity, care unit, admission source
- **Lab values:** albumin, bicarbonate, BUN, creatinine, glucose, hematocrit, hemoglobin, INR, lactate, platelets, potassium, sodium, WBC, ALT — min / mean / max per stay
- **Vital signs:** heart rate, respiration rate, SpO2, temperature, systolic BP, diastolic BP, mean arterial pressure — min / mean / max per stay
- **Diagnosis:** ICD-9 chapter
- **Target:** `mortality` (0 = survived, 1 = died in-hospital)

> **Class imbalance:** ~5% mortality rate in eICU. Accuracy is not a valid metric — use AUROC, AUPRC, or Sensitivity.

---

## Project Structure

```
ICU Mortality Prediction/
├── data/
│   ├── eicu_raw/               # Raw eICU tables (not tracked)
│   ├── mimic_3_raw/            # Raw MIMIC-III tables (not tracked)
│   └── output_data/
│       ├── eicu_train/         # eicu_features.csv
│       └── mimic_val/          # mimic_features.csv
└── notebooks/
    ├── 00_data_collection.ipynb          # Overview of data collection
    ├── 00_data_collection_eICU.ipynb     # Feature extraction from eICU
    ├── 00_data_collection_MIMIC.ipynb    # Feature extraction from MIMIC-III
    ├── 01_data_exploration.ipynb         # EDA version Sara
    ├── 01_data_exploration_02.ipynb      # EDA version Kimia
    └── 02_data_preprocessing.ipynb       # Imputation, encoding, feature alignment
```

---

## Notebook Pipeline

```
00_data_collection_eICU   ──┐
                             ├──► 01_data_exploration_02 ──► 02_data_preprocessing
00_data_collection_MIMIC  ──┘
```

### 00 — Data Collection
Extracts and aggregates features from raw eICU and MIMIC-III tables:
- Aggregates lab results and vital signs (min / mean / max per ICU stay)
- Harmonises ICD-9 diagnosis chapters
- Outputs `eicu_features.csv` and `mimic_features.csv`

### 01 — Exploratory Data Analysis
- Missingness analysis per feature for both datasets
- Class imbalance assessment
- Demographic distributions (age, gender, ethnicity, care unit)
- Lab value and vital sign distributions with outlier detection
- Bivariate analysis: feature vs mortality (KDE, box plots, Mann-Whitney U)
- Pearson correlation matrix and top predictors of mortality
- Dataset shift analysis: KS test comparing eICU vs MIMIC distributions

### 02 — Preprocessing
- Column harmonisation (eICU ↔ MIMIC naming)
- Missing value imputation
- Categorical encoding
- Feature selection

---

## Key Findings (EDA)

- **Creatinine, BUN, lactate, bicarbonate** show the strongest correlation with mortality
- **Min systolic BP, min SpO2, max respiration rate** are the most discriminative vitals
- **Albumin, lactate, ALT** have >50% missingness — require careful imputation
- Significant distributional shift exists between eICU and MIMIC for several features — external validation performance is expected to be lower than internal

---

## Requirements

Notebooks are designed for **Google Colab** with data stored on Google Drive.

```
pandas · numpy · matplotlib · seaborn · scikit-learn
imbalanced-learn · xgboost · lightgbm · shap · missingno
```

---

## Access & Ethics

Both eICU and MIMIC-III are de-identified datasets available under data use agreements via [PhysioNet](https://physionet.org). Credentialed access is required. This repository contains no patient data.

---

## Author

**Alper Karayaz** — MSc AI in Data Science and Engineering, Politecnico di Torino
