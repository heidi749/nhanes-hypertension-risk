# Hypertension Classification with NHANES

## A case study in model evaluation and governance

This project uses seven CDC NHANES datasets to classify current hypertension status and examine how model performance varies across demographic groups. It combines data cleaning, exploratory analysis, logistic regression, random forest, threshold adjustment, sigmoid calibration, and subgroup auditing.

The primary calibrated logistic regression achieved **0.802 AUC and 91.3% sensitivity** on the test sample. However, it missed **70.8% of hypertensive participants aged 18–29** and incorrectly flagged **47.7% of participants without hypertension overall**. These results illustrate why an aggregate score is insufficient for assessing a model's usefulness.

**Scope:** This is an exploratory portfolio and research demonstration. It is not a validated diagnostic, clinical screening, or future-risk prediction tool.

## Data

Source: [NHANES August 2021–August 2023, CDC/NCHS](https://wwwn.cdc.gov/nchs/nhanes/continuousnhanes/default.aspx?Cycle=2021-2023).

| File | Variables used |
|---|---|
| DEMO_L | Age, recorded sex, race/ethnicity, education, income-to-poverty ratio |
| TCHOL_L | Total cholesterol |
| BMX_L | BMI and waist circumference |
| BPXO_L | Three systolic and three diastolic blood-pressure readings |
| BPQ_L | Hypertension diagnosis history and current BP-medication use |
| SMQ_L | Ever smoked at least 100 cigarettes |
| MCQ_L | Arthritis, heart failure, coronary heart disease, heart attack, stroke |

The seven files are left-merged on respondent ID (`SEQN`). The adult dataset contains **8,153 participants aged 18+**. Of these, **6,704 have a known outcome**; **1,449 (17.8%)** are excluded from modeling because their outcome cannot be established.

Survey weights, strata, and clusters are not incorporated. Percentages, performance metrics, and regression results describe the analyzed sample and are not nationally representative estimates.

## Outcome definition

Blood-pressure readings are averaged over available measurements. A positive outcome requires at least one of:

- Mean systolic blood pressure ≥130 mmHg.
- Mean diastolic blood pressure ≥80 mmHg.
- Reported current use of prescribed blood-pressure medication (`BPQ150 = 1`).

A negative outcome requires both BP averages below these thresholds and either explicit nonuse of BP medication or an expected medication-question skip following a report of no previous hypertension diagnosis. Remaining outcomes stay unknown. BP measurements are not imputed to construct the outcome.

There are **3,829 positive** and **2,875 negative** outcomes: **57.1% positive among adults with known outcomes**. Diagnosis history alone does not establish a positive outcome; it helps interpret questionnaire skips. See the [CDC BPQ codebook](https://wwwn.cdc.gov/Nchs/Data/Nhanes/Public/2021/DataFiles/BPQ_L.htm) for questionnaire definitions.

## Modeling approach

- Stratified 80/20 split with `random_state=42`: **5,363 training** and **1,341 test** participants.
- Fifteen original predictors become **19 model columns** after one-hot encoding race/ethnicity. Mexican American is the reference category.
- Blood pressure, hypertension diagnosis history, BP-medication use, and respondent ID are excluded from predictors.
- Baseline and tuned models use training-only median imputation for continuous predictors and mode imputation for binary predictors and adult education. Education is centered around the training mode; participants under 20 receive a neutral placeholder and a separate indicator.
- Logistic regression uses standardized inputs; random forest uses unscaled inputs.
- Tuned versions use `class_weight='balanced'` and a 0.4 classification threshold.
- The final sigmoid-calibrated logistic regression uses five-fold training-set cross-validation, with imputation and scaling fitted inside each fold.

Class balancing weights classes inversely to their training frequency. Hypertension is the majority class here, so balancing gives greater weight to negative cases. Lowering the threshold is what explicitly shifts the decision rule toward flagging more participants.

**Calibration comparison detail:** The calibrated pipeline uses uncentered education categories, a zero placeholder for participants under 20, and fold-specific mode imputation. The earlier models use centered education and a neutral placeholder. The comparison therefore includes preprocessing and cross-validation differences as well as calibration; the change in recall cannot be attributed exclusively to calibration.

## Results

| Model | Test AUC | Hypertension recall |
|---|---:|---:|
| Baseline logistic regression | 0.802 | 83.6% |
| Tuned logistic regression, threshold 0.4 | 0.802 | 86.0% |
| Baseline random forest | 0.819 | 82.1% |
| Tuned random forest, threshold 0.4 | 0.819 | About 90% |
| Calibrated logistic regression, threshold 0.4 | 0.802 | 91.3% |

Random forest has the higher observed AUC. Logistic regression is retained as the primary model for interpretability, with random forest as a comparison. This is a project choice, not evidence that logistic regression is the best model for deployment or that the relationships are linear.

### Primary calibrated model

| Metric | Value |
|---|---:|
| AUC | 0.802 |
| Sensitivity/recall | 91.3% |
| Specificity | 52.3% |
| Precision, hypertensive class | 71.8% |
| Accuracy | 74.6% |
| F1, hypertensive class | 0.804 |

| Actual outcome | Predicted negative | Predicted positive |
|---|---:|---:|
| Negative | 301 | 274 |
| Positive | 67 | 699 |

Missed hypertensive cases decreased from **126 → 107 → 67** across baseline, tuned, and calibrated logistic regression: **59 fewer missed cases (46.8%)**. The final model's 274 false positives show the cost of prioritizing sensitivity at this threshold.

Reliability diagrams compare predicted probabilities with observed outcome frequencies. Higher recall alone does not establish better probability calibration, and the scores should not be presented as validated individual clinical risks.

## Subgroup audit

### Age

| Age | Test N | AUC | Missed hypertensive cases, FNR |
|---|---:|---:|---:|
| 18–29 | 176 | 0.812 | 70.8% |
| 30–39 | 196 | 0.740 | 34.9% |
| 40–49 | 153 | 0.629 | 31.0% |
| 50–59 | 210 | 0.711 | 3.4% |
| 60–69 | 305 | 0.679 | 0.4% |
| 70–79 | 214 | 0.719 | 0.0% |
| 80+ | 87 | 0.460 | 0.0% |

The 18–29 group has reasonable observed ranking performance but very low recall at the selected threshold. AUC and threshold-based recall measure different aspects of performance. Conversely, zero false negatives in older groups does not establish good discrimination: AUC is only 0.460 for participants aged 80+. Age-specific specificity and uncertainty estimates would be necessary for a fuller assessment.

### Race/ethnicity

| Group | Test N | AUC | Recall | FNR |
|---|---:|---:|---:|---:|
| Mexican American | 117 | 0.811 | 90.9% | 9.1% |
| Other Hispanic | 133 | 0.853 | 84.0% | 16.0% |
| Non-Hispanic White | 769 | 0.781 | 91.2% | 8.8% |
| Non-Hispanic Black | 176 | 0.865 | 97.5% | 2.5% |
| Non-Hispanic Asian | 72 | 0.838 | 94.9% | 5.1% |
| Other/Multiracial | 74 | 0.694 | 84.4% | 15.6% |

All groups exceed the project's 80% recall benchmark. That benchmark is not a clinical standard or proof of fairness. Test N includes both outcome classes; recall denominators are smaller. Estimates for small groups are uncertain, and no subgroup confidence intervals were calculated.

### Recorded sex

| Group | Test N | AUC | FNR |
|---|---:|---:|---:|
| Male | 594 | 0.778 | 6.9% |
| Female | 747 | 0.815 | 10.4% |

Observed subgroup differences do not establish their causes. Different positive prediction rates also do not, by themselves, account for outcome prevalence differences or demonstrate discrimination. No group-specific thresholds were implemented.

## Interpretation

Age, waist circumference, BMI, cholesterol, and income-to-poverty ratio have the largest random-forest impurity importances. These importances are descriptive and can favor continuous predictors; they do not establish causal importance.

A separate unweighted, unpenalized Statsmodels logistic regression reports exploratory adjusted associations. Predictors with p < 0.05 are age, BMI, waist circumference, prior stroke, total cholesterol, education, recorded sex, and the Non-Hispanic Black indicator relative to Mexican American participants. The intercept is not a substantive predictor.

These results are not coefficients or p-values for the final cross-validated calibrated ensemble. Standard errors do not account for the survey design or imputation uncertainty. Race/ethnicity is a social/demographic category, not an established biological cause; statistical significance does not establish causality.

## Limitations and governance

- Cross-sectional current-status classification does not predict future onset or establish causal relationships.
- Unknown outcomes and missing predictors may introduce selection bias and information loss.
- Survey weights and the complex design are not incorporated.
- The same test split was inspected across model variants. This is an exploratory comparison, not a final independent validation after model selection.
- The 0.4 threshold was not independently optimized or clinically validated.
- Small subgroup estimates lack confidence intervals, and there is no external validation.
- Smoking is represented by lifetime 100-cigarette history; current use and intensity are not modeled. Lipid-lowering medication use is also not included.
- Education is treated as ordinal; its spacing is a modeling assumption.

Practical use would require an independent validation dataset, evaluation of calibration and both error types by subgroup, review of the role of race in predictions, and appropriate clinical and privacy review. Any eventual system would need an accountable owner, monitoring, and a controlled process for model updates. This version remains a portfolio demonstration.

## How to run

1. Download the seven Cycle L `.xpt` files listed above from the CDC. The complete BMX file contains **8,860 rows**. Check download integrity if pandas reports corruption.
2. Install dependencies:

   ```bash
   pip install pandas numpy matplotlib seaborn scikit-learn statsmodels jupyterlab
   ```

3. In the notebook's **Data Cleaning & Preparation → STEP 1**, update the seven file paths. The author's working paths point to `/Users/heidisaldana/Downloads/`. The validated body-measurement download is named `BMX_L.xpt.txt`; this is still read using `format='xport'`. Use your actual filename.
4. Open the notebook with JupyterLab, then choose **Kernel → Restart Kernel and Run All Cells**. Save when it finishes.

The notebook exports `nhanes_clean.csv` to its working directory. This is an adult analysis table that intentionally retains missing predictors and unknown outcomes; predictor imputation occurs later for modeling.

The accompanying notebook retains the author's executed results. Dependencies are not version-pinned, so results may vary across software versions.

## Technologies

Python, pandas, NumPy, scikit-learn, Statsmodels, Matplotlib, Seaborn, and Jupyter.

## About

An applied statistics portfolio project demonstrating data preparation, predictive modeling, and transparent evaluation of model limitations.
