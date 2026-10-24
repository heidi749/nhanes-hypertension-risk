# CVD Classification with NHANES: Model Governance Review

**A model can rank participants well and still miss most cases at its chosen cutoff.**

This project audits a collaborator’s cardiovascular disease (CVD) classification notebook using NHANES August 2021–August 2023 data. The original analysis compared logistic regression, random forest, and XGBoost. My contribution is the governance review: examining who was excluded, whether coding matched the stated definitions, how errors differed across groups, whether probabilities were calibrated, and whether the evaluation supported the model’s claims.

The original logistic regression achieved a test ROC-AUC of **0.844**, yet missed **107 of 135 reported CVD cases (79.3%)** at its 0.50 cutoff. An exploratory 0.10 cutoff detected substantially more cases, but increased false positives and left important subgroup limitations.

**Scope:** Classification of self-reported CVD history in the analyzed sample. This is not a model of future CVD events or a validated clinical screening tool.

## Project Overview

| Component | Description |
|---|---|
| Main notebook | `Final_CVD_Dataset_Breakdown_governance.ipynb` |
| Data | NHANES August 2021–August 2023, Cycle L |
| Population analyzed | Adults aged 20+ with known CVD status and complete predictors |
| Original models | Logistic regression, random forest, XGBoost |
| Governance focus | Selection bias, coding, leakage checks, errors, subgroup uncertainty, calibration, threshold tradeoffs, and reproducibility |
| Contribution boundary | Original model development belongs to the collaborator; appended audits and threshold experiments constitute the governance contribution |

## Data and Outcome Definition

The notebook merges eight public NHANES files on respondent identifier `SEQN`: `DEMO_L`, `BMX_L`, `BPXO_L`, `TCHOL_L`, `SMQ_L`, `MCQ_L`, `BPQ_L`, and `DIQ_L`.

CVD is a composite of five self-reported conditions:

| Variable | Condition |
|---|---|
| `MCQ160B` | Congestive heart failure |
| `MCQ160C` | Coronary heart disease |
| `MCQ160D` | Angina |
| `MCQ160E` | Heart attack |
| `MCQ160F` | Stroke |

A positive outcome requires at least one Yes response. A negative outcome requires No to all five conditions. Other combinations remain unknown. Independent reconstruction matched all **6,615** stored outcome classifications. This verifies implementation of the specified rule, not the accuracy of self-reports.

The 11 original predictors are age, sex, race/ethnicity, income-to-poverty ratio, BMI, measured total cholesterol, lifetime smoking history, diagnosed diabetes, the derived hypertension indicator, diagnosed hypertension, and diagnosed high cholesterol. Race/ethnicity is one-hot encoded. Other numeric and binary predictors are standardized within the modeling pipeline.

Diabetes uses `DIQ010`; the hypertension indicator incorporates averaged blood pressure and reported medication use from `BPQ150`. In this notebook, diabetes is not derived from the asthma question.

## Participant Flow and Selection Bias

| Stage | Participants |
|---|---:|
| After eight-file inner merge | 6,615 |
| Adults aged 20+ | 6,064 |
| Adults with unknown CVD status | 28 |
| Adults with known CVD status | 6,036 |
| Excluded for missing predictors | 1,595 |
| Complete-case modeling sample | 4,441 |
| Training sample | 3,330 |
| Test sample | 1,111 |

The modeling sample contains **541 CVD cases** and **3,900 participants without reported CVD**. The stratified 75/25 split uses `random_state=42`.

Complete-case cleaning excluded **26.4%** of eligible adults. Exclusion was uneven:

| Group | Excluded |
|---|---:|
| Non-Hispanic Black | 37.2% |
| Non-Hispanic Asian | 32.3% |
| Other Hispanic | 30.4% |
| Mexican American | 29.5% |
| Other/Multiracial | 27.0% |
| Non-Hispanic White | 22.5% |
| Ages 80+ | 37.7% |
| Ages 20–29 | 30.5% |
| Income-to-poverty ratio ≤1 | 23.1% |
| Income-to-poverty ratio >3 | 11.6% |

Sex-based exclusion rates were similar: 26.4% for female and 26.5% for male participants. All **782** eligible adults missing income were excluded. Reported CVD prevalence was **14.55% among excluded adults**, compared with **12.18% among included adults**.

These findings flag possible selection bias; they do not establish its cause or quantify performance among excluded participants. The audit begins after the inner merge, so losses caused by requiring membership in all eight source files have not been quantified.

### Why participants were excluded

Of the 1,595 excluded adults, **1,378 (86.4%)** were missing just one predictor.

| Sole missing predictor | Participants excluded |
|---|---:|
| Income-to-poverty ratio | 643 |
| Total cholesterol | 425 |
| Diabetes indicator | 166 |
| Hypertension indicator | 71 |
| BMI | 40 |
| High cholesterol diagnosis | 24 |
| Smoking history | 7 |
| Hypertension diagnosis | 2 |

Missing income or cholesterol alone accounted for **67.0% of exclusions**, without overlap.

The diabetes recoding keeps Yes and No, while borderline, uncertain, and missing responses become `NaN`. The subsequent `.dropna()` removes the participant’s entire record. **166 participants were excluded solely because the derived diabetes predictor was missing.** The audit did not separately count the original diabetes response categories within those 166 records, so they should not all be described as borderline.

This binary coding is a defensible modeling choice, not an identified coding error. A future comparison could retain borderline and unknown responses as separate categories and evaluate training-only imputation for genuinely missing predictors. Such changes were **not implemented** in this governance review.

## Original Model Performance

| Model | Test ROC-AUC | Training five-fold mean ROC-AUC |
|---|---:|---:|
| Logistic regression | 0.844 | 0.852 |
| Random forest | 0.827 | 0.834 |
| XGBoost | 0.817 | 0.831 |

The original notebook selects the best model by test AUC. Training cross-validation also favors logistic regression, but does not undo the use of test data in the original selection process. A stronger future design would select models within training validation and reserve an independent final evaluation.

The governance audit cloned the logistic pipeline and refitted it using only the original training participants. It exactly reproduced the original test confusion matrix:

| Actual status | Predicted negative | Predicted positive |
|---|---:|---:|
| No reported CVD | 957 | 19 |
| Reported CVD | 107 | 28 |

At 0.50, sensitivity was **20.7%**, specificity **98.1%**, precision **59.6%**, and accuracy **88.7%**. Predicting negative for everyone would already achieve **87.8% accuracy** in this test sample. High accuracy therefore provides limited reassurance about case detection.

## Threshold Tradeoffs

Thresholds were explored using five-fold out-of-fold predictions from the training sample. Each participant’s probability came from a pipeline that did not train on that participant.

| Cutoff | Training CV sensitivity | Training CV specificity | False positives |
|---|---:|---:|---:|
| 0.50 | 20.9% | 97.5% | 74 |
| 0.40 | 33.5% | 95.3% | 136 |
| 0.30 | 48.0% | 90.7% | 271 |
| 0.20 | 67.0% | 84.2% | 463 |
| 0.15 | 74.9% | 79.1% | 611 |
| 0.10 | 83.3% | 71.1% | 845 |
| 0.05 | 93.1% | 57.1% | 1,253 |

A 0.10 cutoff was investigated as an exploratory candidate for approximately 80% sensitivity, not as a clinically established optimum.

| Test result | Original 0.50 | Candidate 0.10 |
|---|---:|---:|
| CVD cases detected | 28 | 110 |
| CVD cases missed | 107 | 25 |
| False positives | 19 | 289 |
| True negatives | 957 | 687 |
| Sensitivity | 20.7% | 81.5% |
| Specificity | 98.1% | 70.4% |
| Precision | 59.6% | 27.6% |
| Participants flagged | 4.2% | 35.9% |
| ROC-AUC | 0.844 | 0.844 |

Lowering the cutoff detected **82 additional CVD cases** and produced **270 additional false positives**. The probabilities and AUC did not change. Whether this tradeoff is useful depends on an explicitly defined use and the consequences of each error.

The test set had already been examined during the review. These results remain exploratory rather than a pristine, prospective final validation.

## Subgroup Performance and Uncertainty

At 0.10, improved overall sensitivity did not establish dependable performance for every group.

### Race/ethnicity

| Group | Test N | CVD cases | Sensitivity | Approximate 95% interval |
|---|---:|---:|---:|---:|
| Mexican American | 64 | 2 | 100.0% | 34.2%–100.0% |
| Non-Hispanic Asian | 54 | 6 | 83.3% | 43.6%–97.0% |
| Non-Hispanic Black | 122 | 14 | 85.7% | 60.1%–96.0% |
| Non-Hispanic White | 677 | 90 | 80.0% | 70.6%–87.0% |
| Other Hispanic | 118 | 15 | 80.0% | 54.8%–93.0% |
| Other/Multiracial | 76 | 8 | 87.5% | 52.9%–97.8% |

All observed sensitivities reached 80%, but wide intervals prevent a strong claim of consistent performance. Test N includes both outcome classes; sensitivity is calculated only among CVD cases. The 80% benchmark is exploratory and does not establish fairness.

### Age, sex, and income

- **Age:** Sensitivity was 68.0% for ages 50–59, 84.8% for ages 60–69, and 85.7% for ages 70–79. Both CVD cases aged 30–39 were missed. No CVD cases occurred among test participants aged 20–29, so their sensitivity could not be evaluated.
- **Older adults:** The model detected all 15 cases aged 80+, but flagged **42 of 43 participants** in that age group. Specificity was only **3.6%**. Perfect observed sensitivity here does not mean strong overall classification.
- **Sex:** Sensitivity was 73.2% for female participants (95% interval 60.4%–83.0%) and 87.3% for male participants (78.2%–93.0%). These intervals are not formal tests of the difference.
- **Income:** Sensitivity was 94.1% at PIR ≤1, 83.9% at PIR >1–3, and 75.8% at PIR >3. These estimates apply only to participants retained in the complete-case sample.

Wilson intervals describe approximate uncertainty in observed sensitivity. They do not incorporate NHANES survey design, model-training uncertainty, or adjustment for repeated comparisons. No group-specific thresholds were implemented, and the review does not establish the cause of subgroup differences.

## Calibration and Probability Accuracy

The original training-only logistic model showed reasonable overall agreement between probabilities and observed CVD frequency:

| Measure | Test result |
|---|---:|
| Mean predicted probability | 12.10% |
| Observed CVD frequency | 12.15% |
| Brier score | 0.0861 |
| Constant training-prevalence reference Brier score | 0.1067 |
| Log loss | 0.2808 |

Ten approximately equal-sized probability bins and a reliability diagram provided additional descriptive checks. Bin-average predictions fell within the approximate intervals for observed frequencies, but broad bins and uncertainty do not prove exact calibration. Brier score and log loss reflect probability accuracy, not calibration alone.

Average agreement differed across groups:

| Group | Mean prediction | Observed frequency | Prediction minus observed |
|---|---:|---:|---:|
| Non-Hispanic Asian | 6.97% | 11.11% | −4.15 percentage points |
| Non-Hispanic Black | 15.77% | 11.48% | +4.29 percentage points |
| Ages 50–59 | 9.65% | 13.59% | −3.94 percentage points |
| Ages 70–79 | 26.89% | 23.33% | +3.56 percentage points |

These are descriptive average differences, not definitive evidence of persistent miscalibration. Small case counts add uncertainty, and mean agreement can conceal differences across probability ranges within a group. Recalibration was not implemented merely to increase sensitivity; the cutoff experiment addressed a different issue.

## Coding, Leakage, and Reproducibility Checks

The audit found:

- Zero mismatches in the checked sex, smoking, diabetes, diagnosed hypertension, and diagnosed cholesterol recodings.
- Zero source-copy mismatches for age, race/ethnicity, income, BMI, and total cholesterol.
- No direct overlap by column name between model predictors and the CVD outcome or its five components.
- No duplicate participant IDs within either split and no participant overlap between training and testing.

These checks support implementation consistency but do not rule out every form of leakage or measurement error. The clinical hypertension construction and questionnaire skip assumptions were reviewed in code; this audit did not independently validate them against raw measurements and current external codebooks. Cross-sectional timing also does not establish whether measured cholesterol, diagnoses, or treatment preceded CVD. Reverse causation and treatment effects remain possible explanations for associations.

### Shared preprocessing object

Original model pipelines and the full-data final logistic pipeline reuse the same preprocessing object. Fitting the final model therefore refits preprocessing held by the earlier model objects. The audit observed a maximum absolute change of **0.001807** in test probabilities from the earlier logistic object, approximately **0.18 percentage points**.

Earlier saved probability arrays were not retroactively changed. The governance evaluations use a separately cloned, training-only model. Future code should give each pipeline an independent preprocessor, such as through `sklearn.base.clone`.

### Full-data score evaluation

The final logistic model was fitted on all **4,441** complete-case participants and scored those same participants, including all **1,111** original test participants. Its apparent full-data AUC was **0.858** and Brier score **0.0838**.

These are development-data summaries, not independent validation. Refitting a final model on all available development data can be reasonable; using its own training participants to establish performance is the limitation. The risk-quintile table should be described accordingly, and its scores should not replace held-out predictions in the governance audit.

## Interpretation, Survey Design, and Intended Use

NHANES uses a complex survey design. The original models and governance tables do not incorporate survey weights, strata, or clusters. Counts, percentages, metrics, and comparisons therefore describe the analyzed sample rather than nationally representative estimates.

The separate Statsmodels regression uses HC3 robust standard errors. HC3 does not substitute for survey-design adjustments. Its odds ratios and p-values describe exploratory adjusted associations; they are not causal effects, validated future-risk estimates, or coefficients from the separate penalized prediction pipeline. Race/ethnicity should not be interpreted as a biological cause of CVD.

The “risk score” is a predicted probability of the specified self-reported CVD-history outcome. Predicting future events would require a suitable longitudinal design with predictors measured before subsequent outcomes.

This project is intended for portfolio, academic, and research demonstration. It does not establish suitability for clinical diagnosis, screening, treatment, insurance, employment, or access-to-care decisions. Deployment monitoring and operational safeguards were discussed as future requirements, not implemented.

## Proposed Improvements — Not Implemented

1. Compare complete-case modeling with methods that retain more participants, including appropriate diabetes categories and training-fold imputation of genuinely missing predictors. Do not impute unknown CVD outcomes to create evaluation labels.
2. Quantify sample losses from the initial inner merges and examine questionnaire eligibility and missingness mechanisms.
3. Use independent preprocessing objects and select algorithms, hyperparameters, and thresholds within training validation.
4. Investigate nonlinear age effects and interactions, then reassess calibration, errors, and subgroup uncertainty rather than relying on overall AUC alone.
5. Incorporate appropriate survey-design methods when making population-level estimates or inferential claims.
6. Validate independently on new data with adequate subgroup case counts before considering practical use. Define error costs, accountability, privacy protections, and monitoring requirements for any future deployment.

## Reproducing the Notebook

The notebook contains original model development followed by appended governance checks. Run cells from top to bottom; audit cells depend on variables created earlier.

1. Obtain the eight matching Cycle L `.xpt` files from the [NHANES August 2021–August 2023 data portal](https://wwwn.cdc.gov/nchs/nhanes/continuousnhanes/default.aspx?Cycle=2021-2023).
2. Update the first cell’s local file paths. Some saved paths end in `.xpt.txt`; point them at the actual local filenames. Filenames alone do not verify file contents.
3. Install the packages used by the notebook, for example:

   ```bash
   python -m pip install pandas numpy scipy scikit-learn xgboost statsmodels matplotlib jupyter
   ```

4. Restart the kernel and run all cells. The governance reconstruction should reproduce the original logistic confusion matrix `[[957, 19], [107, 28]]`.
5. Confirm 6,615 merged records, 4,441 complete cases, and 1,111 test participants before comparing results.

Exact package versions are not pinned, so results may vary across environments. Saved notebook outputs document the author’s local execution. This README was checked against the submitted code and saved outputs; the source datasets were not supplied for an independent full rerun.

### Notebook presentation notes

- The training cutoff table uses one-decimal rounding, displaying 0.15 as 0.2 and 0.05 as 0.0. The underlying tested cutoffs and the table in this README use the correct values. Display cutoffs to two decimals in a future cleanup.
- An original CVD-response summary says “All 4 responses” although the code uses five conditions. This is a label issue; the reconstructed outcome had zero mismatches.
- Governance step labels skip Step 9 because that proposed check was not added. This does not indicate an unexecuted analysis.

## Key Conclusion

The model has useful discrimination and reasonable overall probability agreement, but its original cutoff misses most reported CVD cases. A lower cutoff improves sensitivity while adding false positives and leaving subgroup gaps. Unequal exclusions, uncertain subgroup estimates, shared preprocessing, and reuse of development participants limit the claims supported by the notebook.

**Governance makes these limitations visible: a good AUC does not establish that a model’s decisions are reliable, equitable, or appropriate for its intended use.**
