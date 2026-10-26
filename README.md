# LMIC-CRIS: analysis documentation

Prepared from the cleaned original dataset. Every number below comes from one frozen run of `LMIC-CRIS_final_analysis.ipynb` on `df.xlsx`. If the analysis changes, rerun and replace all numbers together.

## 1. Files

| File | Purpose |
|---|---|
| df.xlsx | De-identified source data, sheet Cleaning, 346 records. Patient names replaced by Patient_ID. This is not the original file with names. |
| LMIC-CRIS_final_analysis.ipynb | Full pipeline with saved outputs. No names in code or outputs. |
| record_level_audit.csv | One row per raw record: why it is in or out. Excel row numbers only. |
| pending_records.csv | The 21 records with no definitive histology, for clinical review. |
| results_summary.csv | Primary, secondary and bounding analyses. |
| baseline_comparison.csv | Model and single PSA measures, with patient-clustered bootstrap intervals and differences. |
| psa_band_analysis.csv, psa1_band_counts_records.csv | Model and PSA 1 within PSA strata, and record counts by PSA band. |
| threshold_analysis.csv | Referral threshold analysis. |
| missing_data_comparison.csv, predictor_missingness.csv | Missing-data strategies compared, and missing counts per predictor. |
| psa_timing_vs_imaging.csv, reduced_feature_models.csv | PSA dates against the ultrasound date, and the model without PSA 2, velocity and doubling time. |
| psa_density_analysis.csv | PSA density alone and as an added predictor (exploratory). |
| validation_sample_for_clinician.csv, validation_sample_for_clinician.xlsx | Random sample of 50 records for clinician verification of the automated extraction, with a ready agreement calculator. |

## 2. Source data and de-identification

- Source: df.xlsx, sheet Cleaning, 346 rows, 11 data columns plus a running index.
- The Patients' Name column was replaced by a pseudonymous Patient_ID (P001 to P211, one per exact name). The name-to-ID key was not saved.
- Free-text columns were scanned for reference numbers, phone numbers, emails and clinician names. None were found.
- Dates are retained because PSA intervals are used. Whether dates may be shared is a question for the ethics approval, not for this analysis.
- Source row means the Excel row number in df.xlsx (header is row 1).

## 3. Cohort flow

| Step | Rule | Removed | Remaining |
|---|---|---|---|
| 1 | Raw records | | 346 |
| 2 | Exact duplicates (identical on all columns except the running index) | 74 | 272 |
| 3 | Invalid PSA, date or age (a PSA value or date does not parse, PSA date 1 before 2000-01-01, PSA date 2 not after PSA date 1, or age outside 30 to 110) | 69 | 203 |
| 4 | No definitive histology (uncertain, inadequate or qualified diagnosis) | 21 | 182 |
| Primary cohort | 107 malignant, 75 benign | | 182 records, 139 patients |

Dates before 2000 are treated as spreadsheet placeholders. Source row 154 had PSA dates of 1970-01-01 and 1970-02-13 (imaging dated 2016), so its true PSA dates are unknown and it was excluded under this rule.

The secondary analysis keeps the first record per patient: 139 records (77 malignant, 62 benign). Of the 182 primary records, 82 belong to patients with more than one record.

The 21 excluded rows are: 10, 108, 121, 131, 158, 181, 208, 211, 269, 272, 276, 284, 292, 295, 313, 323, 331, 332, 334, 335, 345.

## 4. Variable definitions

| Variable | Source column | Definition |
|---|---|---|
| Outcome | Pathological Diagnosis | Malignant if the primary diagnosis line (text before any NB note) contains ADENOCARCINOMA, CARCINOMA, MALIGN or SARCOMA. Benign if it contains HYPERPLASIA, PROSTATITIS or BENIGN. Otherwise no definitive histology. |
| Uncertain histology | Pathological Diagnosis | Wording such as READ TEXT, INADEQUATE, INSUFFICIENT, NOT REPRESENTATIVE, COULD NOT BE MADE, CANNOT BE RULED, SUSPICIOUS, INTRA-EPITHELIAL, a leading "?" or any "?" in the diagnosis line. Inadequacy wording is read from the full text including NB notes. A record with ADENOCARCINOMA and a Gleason score stays malignant. |
| Age | AGE | First number in the field. |
| psa1, psa2 | PSA_Value_1, PSA_Value_2 | Numeric value after removing units and ">". Capped at 100 ng/mL. |
| PSA velocity | PSA values and dates | (psa2 minus psa1) divided by years between the two dates. |
| PSA doubling time | PSA values and dates | 0.693 divided by (velocity divided by psa1), only when velocity is positive. |
| Prostate volume | Imaging_Findings (free text) | Number followed by mL, mLs, cm3 or cm³. cm2 is an area and is not accepted. |
| Imaging suspicious | Imaging_Findings | 1 if the text mentions enlarged, lesion, nodule, hypoechoic or remarkable. |
| HTN, T2DM | Comorbidities | 1 if the text mentions HTN or HHDx (HTN), or T2DM, DM or diabetes (T2DM). A blank field is coded 0. |

No structured prostate-volume field exists. Volume is extracted from free text.

## 5. Missing data

PSA doubling time is undefined when PSA did not rise between the two readings. It is missing in 134 of 182 records (73.6%) and 100 of 139 patients (71.9%). Prostate volume is missing in 16 of 182 records (8.8%) and 11 of 139 patients (7.9%): 13 have no number in the imaging text and 3 contain a number without an mL or cm3 unit. No other predictor has missing values. Missing values are left missing. XGBoost handles them natively, so no imputation is used in the primary analysis.

### Missing-data handling comparison (grouped cross-validation, 182 records)

| Strategy | Records (malignant/benign) | CV AUC | Pooled AUC |
|---|---|---|---|
| Native handling (primary) | 182 (107/75) | 0.799 ± 0.074 | 0.802 |
| Median imputation inside training folds | 182 (107/75) | 0.805 ± 0.078 | 0.811 |
| Volume omitted | 182 (107/75) | 0.776 ± 0.079 | 0.784 |
| Complete cases for volume | 166 (98/68) | 0.816 ± 0.088 | 0.825 |

Native handling and median imputation differ by 0.005 on paired folds, and omitting volume lowers the AUC by 0.024, against a fold-to-fold spread of about 0.08. Complete cases are a different sample. No strategy is shown to be better.

## 6. Model and evaluation

- XGBoost classifier: learning rate 0.1, max depth 3, 50 trees, seed 42. Class weight is the benign-to-malignant ratio computed inside each training fold. No tuning.
- Features: age, psa1, psa2, PSA velocity, PSA doubling time, prostate volume, imaging suspicious, HTN, T2DM.
- **Primary validation:** five-fold stratified cross-validation grouped by patient (all records kept, no patient split across folds), repeated 20 times. Imputation, where used, is fitted inside training folds.
- **Secondary:** first record per patient (139 patients).
- **Bounding:** the 21 non-definitive records forced to malignant, then to benign (203 records).
- Intervals for AUC and for AUC differences are percentile bootstrap (2,000 resamples) that resamples patients.

## 7. Results

### Model performance

| Analysis | Records (patients) | Malignant/benign | CV AUC (mean ± SD) | Pooled AUC |
|---|---|---|---|---|
| PRIMARY: all records, grouped by patient | 182 (139) | 107/75 | 0.799 ± 0.074 | 0.802 |
| SECONDARY: first record per patient | 139 (139) | 77/62 | 0.767 ± 0.082 | 0.773 |
| BOUND: uncertain records as malignant | 203 (157) | 128/75 | 0.752 ± 0.080 | 0.760 |
| BOUND: uncertain records as benign | 203 (157) | 107/96 | 0.772 ± 0.080 | 0.779 |

### Baseline comparison (182 records, pooled out-of-fold, 95% CI)

| Score | AUC | Difference from PSA 1 |
|---|---|---|
| Model (pooled out-of-fold) | 0.802 (0.720 to 0.877) | 0.044 (-0.035 to 0.122) |
| PSA 1 alone | 0.758 (0.666 to 0.838) |  |
| PSA 2 alone | 0.660 (0.553 to 0.758) | -0.098 (-0.199 to -0.009) |
| PSA velocity alone (negated) | 0.724 (0.630 to 0.807) | -0.034 (-0.089 to 0.021) |

A first PSA of 4 ng/mL or more has sensitivity 0.907 and specificity 0.133. The model is 0.044 above PSA 1, but the interval includes zero, so the data show neither a gain nor its absence.

### By first PSA level (exploratory, cut-point chosen after seeing the data)

| Stratum | Records (malignant/benign) | Model AUC | PSA 1 AUC | Difference |
|---|---|---|---|---|
| All records | 182 (107/75) | 0.802 (0.720 to 0.877) | 0.758 (0.666 to 0.838) | 0.044 (-0.035 to 0.122) |
| PSA 1 below 20 ng/mL | 83 (31/52) | 0.598 (0.447 to 0.747) | 0.431 (0.281 to 0.592) | 0.167 (-0.067 to 0.400) |
| PSA 1 at or above 20 ng/mL | 99 (76/23) | 0.856 (0.758 to 0.935) | 0.823 (0.696 to 0.923) | 0.033 (-0.052 to 0.133) |

Of 71 records with PSA 1 of 50 ng/mL or more, 65 are malignant. Discrimination is poor below 20 ng/mL, where triage is hardest.

### Referral threshold analysis (per 100 patients)

| Target sensitivity | Threshold | Sensitivity (95% CI) | Specificity | Referred | Benign not referred | Cancers missed |
|---|---|---|---|---|---|---|
| 95% | 0.147 | 0.953 (0.895 to 0.980) | 0.160 | 90.7 | 6.6 | 2.7 |
| 90% | 0.170 | 0.907 (0.836 to 0.948) | 0.187 | 86.8 | 7.7 | 5.5 |
| 80% | 0.348 | 0.804 (0.719 to 0.868) | 0.533 | 66.5 | 22.0 | 11.5 |

Referring everyone sends 41.2 benign patients per 100 for unnecessary evaluation. Thresholds are chosen on the same out-of-fold predictions that are scored, and class weighting distorts the probabilities, so these figures are optimistic and descriptive only.

### PSA timing sensitivity

The timing of the second PSA relative to biopsy or treatment is not recorded and could not be obtained. PSA dates were compared with the ultrasound date (138 patients with a date; first-record patients). The ultrasound is not the biopsy, so this is indicative only.

| PSA | Before ultrasound | On or after ultrasound | Median days after (IQR) | More than 30 days after |
|---|---|---|---|---|
| PSA 1 | 61 | 77 | 3.0 (-223.0 to 171.5) | 62 |
| PSA 2 | 45 | 93 | 81.5 (-93.2 to 310.8) | 90 |

| Model (182 records) | CV AUC | Pooled AUC |
|---|---|---|
| All nine predictors (primary) | 0.799 ± 0.074 | 0.802 |
| Without PSA 2, PSA velocity and PSA doubling time | 0.783 ± 0.071 | 0.781 |

### PSA density (exploratory, added after the primary results)

| Score (records with volume) | Records | AUC (95% CI) |
|---|---|---|
| PSA density alone | 166 | 0.787 (0.698 to 0.867) |
| PSA 1 alone | 166 | 0.758 (0.664 to 0.845) |

| Model | CV AUC | Pooled AUC |
|---|---|---|
| Primary model plus PSA density (all records) | 0.787 ± 0.076 | 0.794 |
| Primary model (all records) | 0.799 ± 0.074 | 0.802 |

### Cohort characteristics (median, interquartile range)

| | All (182) | Malignant (107) | Benign (75) |
|---|---|---|---|
| Age, years | 70.0 (62.0 to 75.0) | 70.0 (62.0 to 75.0) | 67.0 (64.0 to 73.0) |
| PSA 1, ng/mL | 24.2 (9.6 to 100.0) | 86.5 (17.0 to 100.0) | 13.2 (5.7 to 23.6) |
| PSA 2, ng/mL | 13.4 (7.5 to 28.1) | 25.2 (8.1 to 29.7) | 11.4 (6.9 to 15.3) |
| Years between PSA readings | 0.1 (0.1 to 0.3) | 0.1 (0.1 to 0.2) | 0.1 (0.1 to 0.4) |
| Prostate volume, mL (observed only) | 68.3 (45.0 to 131.0) | 60.9 (39.2 to 116.8) | 85.3 (56.5 to 151.5) |
| Volume missing, n | 16 | 9 | 7 |
| Imaging suspicious, n | 135 | 78 | 57 |
| HTN, n | 172 | 102 | 70 |
| T2DM, n | 49 | 28 | 21 |

All imaging in the primary cohort is ultrasound (recorded as US or USS). PSA 1 dates run from 2008-08-07 to 2023-07-21.

## 8. What this replaces in the earlier manuscript

| Earlier claim | Source of that claim | Final analysis |
|---|---|---|
| 246 records, 104 benign / 142 malignant | Thesis section 5 | 182 records, 75 benign / 107 malignant (139 patients) |
| 52% prostate volume missing | Thesis section 5 and Table 5.1 | 8.8% (16/182) |
| MICE failed, median imputation | Thesis section 5 and reflection | No imputation. Missing values handled natively. |
| AUC 0.87, test set 49 | Thesis section 5 | AUC 0.80 (95% CI 0.72 to 0.88), grouped cross-validation; 0.04 above PSA 1 (95% CI −0.04 to 0.12) |

The thesis appendices (268 records, 9.7%, AUC 0.81) and the earlier notebook (266 records, AUC 0.943, mean imputation, imputation before the split) described other runs and are not used.

## 9. Decisions recorded and points to state in the manuscript

Resolved:
- Source row 154 excluded because its PSA dates are placeholders (see section 3).
- The 21 non-definitive records and the six benign records with NB notes mentioning malignancy (rows 48, 58, 91, 103, 155, 193) were treated as confirmed on the clinical side. No reclassifications were supplied, so none were made. If any are changed, rerun the notebook and replace every number in this document.

To state or acknowledge in the manuscript:
1. **Timing of PSA 2.** The hospital could not supply it. PSA fell between the two readings in 82 of 107 malignant and 44 of 75 benign records, with a median gap of 43 days. State when each PSA was taken relative to biopsy and any treatment. The model without PSA 2, velocity and doubling time performs about as well.
2. **PSA cap.** psa1 is capped at 100 in 47 of 182 records. Uncapped values give the same AUC at the patient level.
3. **Comorbidities** are coded 0 when blank, so absent and not recorded are not distinguished.
4. **Precision.** One site, 139 patients, and wide intervals. The AUC difference from PSA 1 has a 95% interval of −0.04 to 0.12.
5. **Extraction not yet clinically validated.** The outcome and volume rules were not checked against a clinician-reviewed sample. A 50-record sample sheet is provided (validation_sample_for_clinician.xlsx).
6. **Repository privacy.** Publish from a new repository, or purge the old history, because earlier output printed patient names.

## 10. Reproducibility record

- Python 3.12.3, xgboost 3.4.1, scikit-learn 1.8.0, pandas 3.0.2, numpy 2.4.4.
- Original raw file with names (kept private, not published) MD5: 2c4acddfdb7f473ebc9718457893209e
- df.xlsx (de-identified) MD5: dfeeda69c4f74d5e24798a50a36d7443
- LMIC-CRIS_final_analysis.ipynb MD5: 32a8a31a5beea12e9345848b83128782
- Rerunning the notebook on df.xlsx reproduces every number in this document.
