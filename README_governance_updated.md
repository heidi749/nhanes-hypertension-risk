# Predicting Hypertension Risk in U.S. Adults

### A Case Study in Why Model Governance Matters

---

## The Core Argument

A model that looks good on paper can still cause serious harm when deployed.

This project builds a hypertension prediction model using real public health data from the CDC. On the surface the results are reasonable: 74.6% accuracy, AUC of 0.802, and the model correctly identifies 91.3% of hypertensive cases overall. Most data science projects would stop there and call it a success.

This one does not.

A governance analysis revealed that the same model misses 70.8% of hypertensive young adults and incorrectly flags 47.7% of participants without hypertension despite an acceptable AUC. None of these failures are visible in a top-line summary. All of them matter enormously if the model is ever used to inform real decisions.

This project documents what responsible data science looks like: not just building a model, but stress-testing it, understanding who it fails, and being honest about what it should and should not be used for.

---

## What This Project Covers

1. Merging and cleaning 7 real CDC health datasets
2. Exploratory analysis of cardiovascular risk across demographic groups
3. Building and comparing logistic regression and random forest models
4. Iterative model improvement through class weighting, threshold tuning, and probability calibration
5. Subgroup fairness analysis across race, age, and sex
6. A full governance and ethics framework grounded in the actual findings

---

## The Data

**Source:** NHANES Cycle L (2021-2023), National Center for Health Statistics, CDC. Publicly available at [https://www.cdc.gov/nchs/nhanes/](https://www.cdc.gov/nchs/nhanes/)

**Files used:**

| File    | Contents                                        |
| ------- | ----------------------------------------------- |
| DEMO_L  | Age, sex, race/ethnicity, education, income     |
| TCHOL_L | Total cholesterol                               |
| BMX_L   | BMI, waist circumference                        |
| BPXO_L  | Blood pressure (3 readings per participant)     |
| BPQ_L   | Blood pressure diagnosis and medication history |
| SMQ_L   | Smoking history                                 |
| MCQ_L   | Heart disease, stroke, arthritis      |

**Final dataset:** 8,153 adults aged 18 and older; 6,704 with known outcomes were included in modeling. The remaining 1,449 (17.8%) had unknown outcomes and were excluded.

Note: Survey weights were not applied. Results reflect the unweighted sample and should not be treated as nationally representative estimates.

---

## How Hypertension Was Defined

A participant was classified as hypertensive if any of the following were true:

- Measured systolic BP at or above 130 mmHg
- Measured diastolic BP at or above 80 mmHg
- Currently taking blood pressure medication (this captures patients whose BP appears controlled because of treatment, not because they are healthy)

Self-reported diagnosis alone did not establish a positive outcome; it was used to interpret medication-question skips. A negative outcome required both BP averages below the thresholds and either reported medication nonuse or an expected skip after reporting no previous diagnosis. Otherwise, status remained unknown. Blood pressure was not imputed to create the outcome. Resulting prevalence was 57.1% among adults with known outcomes, an unweighted sample percentage rather than a national estimate.

The model classifies current hypertension status rather than predicting future onset.

---

## Model Development

**Why logistic regression?**

Both logistic regression and random forest were built and compared. Random forest achieved a higher AUC (0.819 vs 0.802). Logistic regression was retained as the primary model for its interpretable coefficients, with random forest as a comparison. In a model that could affect health decisions, understanding how predictions are made matters alongside predictive performance.

**Three rounds of improvement:**

Round 1: Class weight balancing. Classes were weighted inversely to their training frequency. Because hypertension was the majority class, balancing gave more weight to participants without hypertension; it did not specifically increase the penalty for missed hypertensive cases.

Round 2: Threshold adjustment. The default cutoff of 0.5 was lowered to 0.4, meaning the model flags someone as hypertensive if their predicted probability is at least 40% rather than 50%. This trades more false positives for fewer missed cases.

Round 3: Probability calibration. Sigmoid (Platt) calibration was fitted using five-fold cross-validation within the training set, with imputation and scaling learned inside each fold. Reliability diagrams compare predicted probabilities with observed outcomes. Calibration does not guarantee trustworthy individual probabilities. The calibrated implementation also handles education differently, so the before/after comparison includes preprocessing differences and cannot isolate calibration's effect.

**False negative progression:**

| Version                           | Missed Hypertensive Cases |
| --------------------------------- | ------------------------- |
| Baseline                          | 126                       |
| After class weights and threshold | 107                       |
| After calibration (final)         | 67                        |

---

## Where the Model Looks Good

| Metric                  | Value      |
| ----------------------- | ---------- |
| AUC                     | 0.802      |
| Overall recall          | 91.3%      |
| Overall accuracy        | 74.6%      |
| False negatives reduced | 126 to 67  |

In a separate exploratory Statsmodels regression, statistically significant predictors included age, BMI, waist circumference, prior stroke, total cholesterol, education, sex, and the Non-Hispanic Black indicator relative to Mexican American participants.

Race/ethnicity was included as a categorical predictor. These associations do not establish causality, and the regression's p-values do not account for survey design or imputation uncertainty. They are not p-values for the final calibrated model.

---

## Where Governance Changes Everything

This is the section that matters most.

**The age problem is severe:**

| Age Group | Cases Missed (FNR) |
| --------- | ------------------ |
| 18-29     | 70.8%              |
| 30-39     | 34.9%              |
| 40-49     | 31.0%              |
| 50-59     | 3.4%               |
| 60-69     | 0.4%               |
| 70-79     | 0.0%               |
| 80+       | 0.0%               |

For young adults, the model misses 70.8% of hypertensive cases. That means roughly 71 out of every 100 hypertensive 18-29 year olds would not be flagged at the selected threshold. Their AUC is 0.812: the model can rank cases reasonably well while still missing most of them at a fixed cutoff. Overall AUC and recall conceal this age-specific problem.

**Performance still varies across racial groups:**

| Group            | AUC   | Cases Missed (FNR) |
| ---------------- | ----- | ------------------ |
| NH White         | 0.781 | 8.8%               |
| Mexican American | 0.811 | 9.1%               |
| Other Hispanic   | 0.853 | 16.0%              |
| NH Black         | 0.865 | 2.5%               |
| NH Asian         | 0.838 | 5.1%               |
| Other/Multiracial | 0.694 | 15.6%              |

All evaluated racial/ethnic groups exceed the project's 80% recall target, including Black and Asian participants. However, the model still misses 16.0% of hypertensive Other Hispanic participants and 15.6% of Other/Multiracial participants. Small subgroup sizes mean these estimates are uncertain. Meeting a recall target does not, by itself, establish that a model is fair or safe to deploy.

**Fewer missed cases come with a cost:**

The final model identifies 699 of 766 hypertensive test participants but also flags 274 of 575 participants without hypertension. Its false-positive rate is 47.7%. Governance must consider unnecessary follow-up alongside the harm of missed cases.

**Probability scores need scrutiny too:**

An acceptable AUC does not guarantee calibrated probabilities. Reliability diagrams provide a separate check, but neither calibration nor the increase in recall establishes that an individual probability can be trusted in clinical practice.

---

## What This Means for Deployment

This model is appropriate for:

- Exploratory research on associations within the analyzed sample
- Demonstrating how subgroup auditing exposes limitations hidden by overall scores
- Demonstrating end-to-end data science methodology on public health data

This model should not be used for:

- Clinical diagnosis of hypertension
- Individual medical decision making
- Reassuring younger adults that they do not have hypertension based on a negative prediction
- Any deployment without independent validation and subgroup assessment
- Any context where decisions are made based on model output alone without clinician review

---

## Key Findings Summary

- Age is by far the strongest predictor of hypertension risk
- Waist circumference and BMI follow age in random-forest feature importance; these rankings describe model behavior and do not establish causal importance
- Updated racial subgroup results are more reassuring on recall, but age disparities and a high false-positive rate remain substantial
- The model looks reasonable in aggregate but fails specific populations in ways that would cause real harm if deployed without governance guardrails
- Governance is not a formality. It is what separates a model that is technically functional from one that is safe to use

---

## How to Run

**1. Download NHANES data** from [https://www.cdc.gov/nchs/nhanes/](https://www.cdc.gov/nchs/nhanes/) Get the Cycle L (2021-2023) files: DEMO_L, TCHOL_L, BMX_L, BPXO_L, BPQ_L, SMQ_L, MCQ_L

**2. Install dependencies:**

```
pip install pandas numpy matplotlib seaborn scikit-learn statsmodels notebook
```

**3. Update file paths** in the Data Cleaning & Preparation cell (STEP 1) to point to your downloaded files. The author’s validated BMX download is named `BMX_L.xpt.txt`; use your actual filename.

**4. Run:**

```
jupyter notebook PROJECThypertension_final.ipynb
```

Select Kernel, then Restart and Run All

---

## Technologies

Python, pandas, numpy, scikit-learn, statsmodels, matplotlib, seaborn, Jupyter Notebook

---

## About

This project was built as a portfolio piece demonstrating end-to-end data science with a focus on responsible modeling. The goal was not to build the most accurate model possible. It was to build a model that is honest about what it can and cannot do.

BS in Statistics. Background in tutoring and applied statistical analysis.