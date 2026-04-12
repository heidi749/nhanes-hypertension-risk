# Predicting Hypertension Risk in U.S. Adults
### A Case Study in Why Model Governance Matters

---

## The Core Argument

A model that looks good on paper can still cause serious harm when deployed.

This project builds a hypertension prediction model using real public health data from the CDC. On the surface the results are reasonable: 70% accuracy, AUC of 0.755, and the model correctly identifies 87% of hypertensive cases overall. Most data science projects would stop there and call it a success.

This one does not.

A governance analysis revealed that the same model misses 77.5% of hypertensive young adults, performs meaningfully worse for Black and Asian participants, and initially produced probability scores that were statistically untrustworthy despite an acceptable AUC. None of these failures are visible in a top-line summary. All of them matter enormously if the model is ever used to inform real decisions.

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

**Source:** NHANES Cycle L (2021-2023), National Center for Health Statistics, CDC. Publicly available at https://www.cdc.gov/nchs/nhanes/

**Files used:**
| File | Contents |
|---|---|
| DEMO_L | Age, sex, race/ethnicity, education, income |
| TCHOL_L | Total cholesterol |
| BMX_L | BMI, waist circumference |
| BPXO_L | Blood pressure (3 readings per participant) |
| BPQ_L | Blood pressure diagnosis and medication history |
| SMQ_L | Smoking history |
| MCQ_L | Heart disease, stroke, diabetes, arthritis |

**Final dataset:** 7,809 adults aged 18 and older

Note: Survey weights were not applied. Results reflect the unweighted sample and should not be treated as nationally representative estimates.

---

## How Hypertension Was Defined

A participant was classified as hypertensive if any of the following were true:

- Measured systolic BP at or above 130 mmHg
- Measured diastolic BP at or above 80 mmHg
- Currently taking blood pressure medication (this captures patients whose BP appears controlled because of treatment, not because they are healthy)

Self-reported diagnosis was excluded because it is prone to recall bias and may reflect historical rather than current status. Resulting prevalence was 53.6%, slightly above the CDC national estimate of 47%, which is expected given the stricter ACC/AHA 2017 threshold used and the fact that survey weights were not applied.

---

## Model Development

**Why logistic regression?**

Both logistic regression and random forest were built and compared. Random forest achieved marginally higher AUC (0.763 vs 0.755) but the difference was small enough that the near-identical results confirmed the data has mostly linear relationships. Logistic regression was selected as the final model because its coefficients directly explain the direction and size of each variable's contribution. In a clinical context, interpretability is not just a preference, it is an ethical requirement.

**Three rounds of improvement:**

Round 1: Class weight balancing. By default the model treats all mistakes equally. Telling the model to penalize missing a hypertensive case more heavily than a false positive shifted its behavior toward catching more true cases, which is the right tradeoff for screening.

Round 2: Threshold adjustment. The default cutoff of 0.5 was lowered to 0.4, meaning the model flags someone as hypertensive if their predicted probability exceeds 40% rather than 50%. This further reduced missed cases.

Round 3: Probability calibration. After class weight balancing the model's probability scores were distorted. A score of 0.3 did not actually mean a 30% chance of hypertension. Platt scaling corrected this by fitting a small adjustment on top of the raw scores to make them statistically accurate. This matters because clinicians and decision makers who see a probability score need to be able to trust that number.

**False negative progression:**
| Version | Missed Hypertensive Cases |
|---|---|
| Baseline | 192 |
| After class weights and threshold | 152 |
| After calibration (final) | 111 |

---

## Where the Model Looks Good

| Metric | Value |
|---|---|
| AUC | 0.755 |
| Overall recall | 87.2% |
| Overall accuracy | 69% |
| False negatives reduced | 192 to 111 |

Top statistically significant predictors: age, total cholesterol, waist circumference, prior stroke, coronary heart disease, BMI, arthritis, income-to-poverty ratio, prior heart attack, smoking history, sex.

Notably, race and education were not statistically significant after controlling for other variables (p = 0.21 and p = 0.31). The model is not using race to make decisions. The real drivers are age, income, and clinical measurements.

---

## Where Governance Changes Everything

This is the section that matters most.

**The age problem is severe:**
| Age Group | Cases Missed (FNR) |
|---|---|
| 18-29 | 77.5% |
| 30-39 | 57.8% |
| 40-49 | 29.0% |
| 50-59 | 3.2% |
| 60-69 | 0.0% |
| 70-79 | 0.0% |

For young adults, the model misses 77.5% of hypertensive cases. That means if 100 hypertensive 18-29 year olds were screened, 77 of them would be told they are fine. This is not a threshold problem. AUC is also low for younger groups, meaning the model genuinely does not understand hypertension risk in young people. This is likely because age dominates the model and early-onset hypertension has different drivers that are not captured in the available features.

**The racial gap is a threshold problem, not a comprehension problem:**
| Group | AUC | Cases Missed (FNR) |
|---|---|---|
| NH White | 0.750 | 8.6% |
| Mexican American | 0.743 | 12.2% |
| Other Hispanic | 0.794 | 19.5% |
| NH Black | 0.748 | 21.3% |
| NH Asian | 0.715 | 21.3% |

AUC is fairly consistent across racial groups. The model ranks hypertensive individuals above normotensive individuals at similar rates across groups. But some groups receive systematically lower raw probability scores, meaning fewer cross the 0.4 threshold even when the model correctly sees them as higher risk. One potential fix is group-specific thresholds. However, using different cutoffs by race is ethically and legally contested and would require review by ethicists, clinicians, and legal counsel before any implementation.

**The probability scores were initially untrustworthy:**

Before calibration, the model's probability scores were distorted by class weight balancing. A patient assigned a 30% probability of hypertension was not actually at 30% risk. After Platt scaling the scores were corrected. This matters because a model that produces accurate binary predictions but unreliable probability scores cannot be safely used in any context where the actual probability number informs decisions.

---

## What This Means for Deployment

This model is appropriate for:
- Population-level research and risk stratification
- Identifying demographic groups for targeted public health outreach
- Demonstrating end-to-end data science methodology on public health data

This model should not be used for:
- Clinical diagnosis of hypertension
- Individual medical decision making
- Adults under 40 years old (recall is too low to be useful)
- Any deployment without supplemental screening protocols for NH Black and NH Asian populations
- Any context where decisions are made based on model output alone without clinician review

---

## Key Findings Summary

- Age is by far the strongest predictor of hypertension risk
- Income-to-poverty ratio ranked third overall, ahead of BMI, smoking, and most disease history variables. Structural socioeconomic factors matter more than individual behaviors in this data
- Race is not a statistically significant predictor after controlling for other variables
- The model looks reasonable in aggregate but fails specific populations in ways that would cause real harm if deployed without governance guardrails
- Governance is not a formality. It is what separates a model that is technically functional from one that is safe to use

---

## How to Run

**1. Download NHANES data** from https://www.cdc.gov/nchs/nhanes/
Get the Cycle L (2021-2023) files: DEMO_L, TCHOL_L, BMX_L, BPXO_L, BPQ_L, SMQ_L, MCQ_L

**2. Install dependencies:**
```bash
pip install pandas numpy matplotlib seaborn scikit-learn statsmodels
```

**3. Update file paths** in Cell 1 of the notebook to point to your downloaded .xpt files

**4. Run:**
```bash
jupyter notebook PROJECThypertension_clean.ipynb
```
Select Kernel, then Restart and Run All

---

## Technologies

Python 3.12, pandas, numpy, scikit-learn, statsmodels, matplotlib, seaborn, Jupyter Notebook

---

## About

This project was built as a portfolio piece demonstrating end-to-end data science with a focus on responsible modeling. The goal was not to build the most accurate model possible. It was to build a model that is honest about what it can and cannot do.

BS in Statistics. Background in tutoring and applied statistical analysis.
