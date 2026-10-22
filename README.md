# LMIC-CRIS: analysis documentation

Prepared from the cleaned original dataset. Every number below comes from one frozen run of `LMIC-CRIS_final_analysis.ipynb` on `df.xlsx`. If the analysis changes, rerun and replace all numbers together.

## 1. Files

| File | Purpose |
|---|---|
| df.xlsx | De-identified source data, sheet Cleaning, 346 records. Patient names replaced by Patient_ID. This is not the original file with names. |
| LMIC-CRIS_final_analysis.ipynb | Full pipeline with saved outputs. No names in code or outputs. |
| record_level_audit.csv | One row per raw record: why it is in or out. Excel row numbers only. |
| pending_records.csv | The 21 records with no definitive histology, for clinical review. |
| results_summary.csv | Model results table. |
| threshold_analysis.csv | Referral threshold analysis (patient level). |
| baseline_comparison.csv | AUC of the model and of single PSA measures, with bootstrap intervals. |

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

Patient-level analysis keeps the first record per patient: 139 records (77 malignant, 62 benign). Of the 182 primary records, 82 belong to patients with more than one record.

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

## 5. Prostate volume missingness

16 of 182 primary records have no usable volume: 8.8%. Of these, 13 have no number in the imaging text (for example ENLARGED) and 3 contain a number without an mL or cm3 unit. Missing values are left missing. XGBoost handles them natively, so no imputation (mean, median or MICE) is used.

## 6. Model and evaluation

- XGBoost classifier: learning rate 0.1, max depth 3, 50 trees, scale_pos_weight equal to benign count divided by malignant count, seed 42.
- Features: age, psa1, psa2, PSA velocity, PSA doubling time, prostate volume, imaging suspicious, HTN, T2DM.
- 5-fold stratified cross-validation, seed 42. Nothing is fitted outside the training folds.
- The headline is the one-record-per-patient analysis, because record-level folds can place the same patient in both training and test data.
- Bounding sensitivity: the 21 non-definitive records forced to malignant, then to benign.

## 7. Results

| Analysis | n (malignant/benign) | CV AUC (mean ± SD) |
|---|---|---|
| Primary, all records | 182 (107/75) | 0.837 ± 0.067 |
| Primary, one record per patient (headline) | 139 (77/62) | 0.751 ± 0.105 |
| Non-definitive as malignant, all records | 203 (128/75) | 0.808 ± 0.081 |
| Non-definitive as malignant, per patient | 157 (95/62) | 0.716 ± 0.120 |
| Non-definitive as benign, all records | 203 (107/96) | 0.851 ± 0.059 |
| Non-definitive as benign, per patient | 157 (77/80) | 0.787 ± 0.065 |

Across the bounding analyses the patient-level AUC ranges from 0.716 to 0.787, and fold-to-fold standard deviations are 0.07 to 0.12.

### Baseline comparison (139 patients, one record per patient)

| Score | AUC | 95% bootstrap CI |
|---|---|---|
| Model (pooled out-of-fold) | 0.745 | 0.661 to 0.826 |
| PSA 1 alone | 0.745 | 0.658 to 0.825 |
| PSA 2 alone | 0.659 | 0.569 to 0.756 |
| PSA velocity alone (negated) | 0.706 | 0.617 to 0.79 |

The model's discrimination is the same as the first PSA reading alone. A first PSA of 4 ng/mL or more has sensitivity 0.909 and specificity 0.145.

### Referral threshold analysis (per 100 patients)

| Target sensitivity | Threshold | Sensitivity | Specificity | Referred | Benign not referred | Cancers missed |
|---|---|---|---|---|---|---|
| 95% | 0.065 | 0.961 | 0.048 | 95.7 | 2.2 | 2.2 |
| 90% | 0.129 | 0.909 | 0.113 | 89.9 | 5.0 | 5.0 |
| 80% | 0.292 | 0.805 | 0.468 | 68.3 | 20.9 | 10.8 |

Referring everyone sends 44.6 benign patients per 100 for unnecessary evaluation. Thresholds are chosen on the same out-of-fold predictions that are scored, so these figures are optimistic and descriptive only.

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

## 8. Decisions recorded and points to state in the manuscript

Resolved:
- Source row 154 excluded because its PSA dates are placeholders (see section 3).
- The 21 non-definitive records and the six benign records with NB notes mentioning malignancy (rows 48, 58, 91, 103, 155, 193) were treated as confirmed on the clinical side. No reclassifications were supplied, so none were made. If any are changed, rerun the notebook and replace every number in this document.

To state or acknowledge in the manuscript:
1. **Timing of PSA 2.** PSA fell between the two readings in 82 of 107 malignant and 44 of 75 benign records, with a median gap of 43 days. State when each PSA was taken relative to biopsy and any treatment. If PSA 2 follows treatment, velocity is not a valid predictor.
2. **PSA cap.** psa1 is capped at 100 in 47 of 182 records (26%), which limits what velocity and doubling time can show.
3. **Comorbidities** are coded 0 when blank, so absent and not recorded are not distinguished.
4. **Precision.** One site, 139 patients, wide fold-to-fold variation. Report intervals and avoid strong claims.
5. **Repository privacy.** Publish from a new repository, or purge the old history, because earlier output printed patient names.

## 9. Reproducibility record

- Python 3.12.3, xgboost 3.4.1, scikit-learn 1.8.0, pandas 3.0.2, numpy 2.4.4.
- Original raw file with names (kept private, not published) MD5: 2c4acddfdb7f473ebc9718457893209e
- df.xlsx (de-identified) MD5: dfeeda69c4f74d5e24798a50a36d7443
- LMIC-CRIS_final_analysis.ipynb MD5: b805aa2fd7c95ed375b753ebc7e38931
- Rerunning the notebook on df.xlsx reproduces the original-data run, apart from the date rule added for row 154.
