# CSE 160 Final Project — CDC Chronic Disease Indicators

**Team:** Scott Lenio, Amanda Novick, Fatima Tanveer, Jacob Thompson

**Goal:** Predict state-level heart disease mortality rates using chronic disease and behavioral risk factor indicators from the CDC.

**Data source:** [U.S. Chronic Disease Indicators](https://catalog.data.gov/dataset/u-s-chronic-disease-indicators)

---

## Running the Project

1. Download `U.S._Chronic_Disease_Indicators.csv` from the link above and place it in the project root (it is gitignored due to file size)
2. Open `cse160_project.Rmd` in RStudio and knit to HTML
3. Cached chunks will be saved to `cache/` — subsequent knits will be faster

**Required packages:** `dplyr`, `tidyr`, `ggplot2`

---

## Preprocessing Pipeline

The raw dataset (309,215 rows × 34 columns) is processed into a wide-format modeling dataframe through the following steps:

1. **Column reduction** — Kept 12 of 34 columns; dropped empty/sparse fields (secondary stratifications, ID columns, geographic duplicates)
2. **Suppressed cell removal** — Removed ~100k rows where `DataValue` was blank due to small sample sizes the CDC does not publish
3. **Data type filter** — Kept `"Crude Prevalence"` rows (% of population) for feature indicators, and `"Crude Rate"` (per 100k) exclusively for CVD08/CVD09. Dropped counts, means, age-adjusted variants, etc.
4. **Topic filter** — Retained five topics: Cardiovascular Disease, Tobacco, Diabetes, Alcohol, and Nutrition/Physical Activity/Weight Status
5. **Year alignment** — Multi-year rolling estimates assigned to `YearStart` as a single `Year` key
6. **Long → wide pivot** — Each row becomes one unique (LocationAbbr, Year, Stratification1) profile, with each `QuestionID` as its own column. Duplicate key combinations collapsed by mean.
7. **Missing value handling** — Dropped rows with no target value (CVD09); dropped columns with >50% NA; imputed remaining feature NAs with column median

**Final shape:** ~1,679 rows × varies columns (run the Rmd for exact output)

---

## Data Structure

Each row in the final wide dataframe represents one demographic profile:

| Column | Type | Description |
|---|---|---|
| `LocationAbbr` | character | Two-letter U.S. state/territory abbreviation (`US` = national aggregate) |
| `Year` | integer | Reporting year — 2019, 2020, or 2021 |
| `Stratification1` | character | Demographic subgroup: Overall, Male, Female, age group, or race/ethnicity |
| `CVD09` | numeric | **Target.** Heart disease mortality crude rate per 100,000 people |
| *(feature columns)* | numeric | One column per QuestionID — see data dictionary below |

---

## Target Variable

**CVD09 — Diseases of the heart mortality among all people (underlying cause)**
- Measurement: Crude Rate (deaths per 100,000 people)
- Years available: 2019, 2020, 2021
- Source: Vital records / death certificates

---

## Feature Data Dictionary

Features that may be present after preprocessing (columns with >50% NA are dropped automatically):

### Cardiovascular Disease

| QuestionID | Question | Type |
|---|---|---|
| CVD01 | High blood pressure among adults | Crude Prevalence (%) |
| CVD02 | Taking medicine to control high blood pressure among adults with high blood pressure | Crude Prevalence (%) |
| CVD03 | High cholesterol among adults who have been screened | Crude Prevalence (%) |
| CVD04 | Taking medicine for high cholesterol among adults | Crude Prevalence (%) |
| CVD08 | Coronary heart disease mortality among all people, underlying cause | Crude Rate (per 100k) |

### Diabetes

| QuestionID | Question | Type |
|---|---|---|
| DIA01 | Diabetes among adults | Crude Prevalence (%) |
| DIA02 | Gestational diabetes among women with a recent live birth | Crude Prevalence (%) |

### Tobacco

| QuestionID | Question | Type |
|---|---|---|
| TOB01 | Current tobacco use of any tobacco product among high school students | Crude Prevalence (%) |
| TOB02 | Current electronic vapor product use among high school students | Crude Prevalence (%) |
| TOB03 | Current smokeless tobacco use among high school students | Crude Prevalence (%) |
| TOB04 | Current cigarette smoking among adults | Crude Prevalence (%) |
| TOB05 | Cigarette smoking during pregnancy among women with a recent live birth | Crude Prevalence (%) |
| TOB06 | Quit attempts in the past year among adult current smokers | Crude Prevalence (%) |

### Alcohol

| QuestionID | Question | Type |
|---|---|---|
| ALC01 | Alcohol use among high school students | Crude Prevalence (%) |
| ALC06 | Binge drinking prevalence among adults | Crude Prevalence (%) |
| ALC07 | Binge drinking prevalence among high school students | Crude Prevalence (%) |

### Nutrition, Physical Activity, and Weight Status

| QuestionID | Question | Type |
|---|---|---|
| NPW01 | Consumed fruit less than one time daily among high school students | Crude Prevalence (%) |
| NPW02 | Consumed fruit less than one time daily among adults | Crude Prevalence (%) |
| NPW03 | Consumed vegetables less than one time daily among high school students | Crude Prevalence (%) |
| NPW04 | Consumed vegetables less than one time daily among adults | Crude Prevalence (%) |
| NPW05 | Consumed regular soda at least one time daily among high school students | Crude Prevalence (%) |
| NPW06 | No leisure-time physical activity among adults | Crude Prevalence (%) |
| NPW07 | Children and adolescents aged 6–13 years meeting aerobic physical activity guideline | Crude Prevalence (%) |
| NPW08 | Met aerobic physical activity guideline among high school students | Crude Prevalence (%) |
| NPW09 | Met aerobic physical activity guideline for substantial health benefits, adults | Crude Prevalence (%) |
| NPW10 | Infants who were breastfed at 12 months | Crude Prevalence (%) |
| NPW11 | Infants who were exclusively breastfed through 6 months | Crude Prevalence (%) |
| NPW12 | Obesity among WIC children aged 2 to 4 years | Crude Prevalence (%) |
| NPW13 | Obesity among high school students | Crude Prevalence (%) |
| NPW14 | Obesity among adults | Crude Prevalence (%) |

---

## Notes & Caveats for Modeling

- **CVD08 is a near-duplicate of the target.** CVD08 (coronary heart disease mortality) is a direct subset of CVD09 (all heart disease mortality) and will likely dominate any model. Consider dropping it as a feature or acknowledging its influence in your write-up.
- **Units are mixed.** Feature columns are percentages (Crude Prevalence); CVD09 is deaths per 100,000 (Crude Rate). Keep this in mind when interpreting coefficients.
- **Demographic rows are not independent.** Overall, Male, Female, and race/ethnicity rows for the same location/year are derived from the same population. Consider whether to model only `Overall` rows or account for this structure.
- **Median imputation was applied** to feature NAs after the pivot. Imputed values are population medians, not state-specific estimates.
- **Territorial data is limited.** GU, PR, and VI were dropped because CVD09 mortality data was not reported for them.


## Modeling

Two regression models are applied to predict `CVD09` (heart disease mortality crude rate per 100,000 people).

### 1. Multiple Linear Regression (OLS)

A standard multiple linear regression is fit using all feature columns retained after preprocessing. This serves as the baseline model and satisfies the original project proposal's stated method.

Required packages: No additional packages beyond base R.

Limitations: Correlated features such as tobacco use, obesity, and diabetes produce unstable coefficients. With roughly 30 features and no regularization the model is also prone to overfitting.

### 2. Ridge Regression

Ridge regression extends OLS by adding an L2 penalty term to the loss function. This shrinks correlated coefficients proportionally rather than arbitrarily, producing more stable estimates. The optimal lambda is selected via 10-fold cross-validation using `cv.glmnet`.

Required packages: `glmnet`

Key choices: `alpha = 0` specifies Ridge (vs `alpha = 1` for Lasso). `lambda.min` is used for final predictions. CVD08 is retained as a feature but its outsized influence on the target is acknowledged in the write-up.

### Model Evaluation

Both models are evaluated on the held-out test set (20% of data) using RMSE (root mean squared error, in the same units as CVD09 — deaths per 100k) and R² (proportion of variance in CVD09 explained by the model). Residual vs Fitted and Actual vs Predicted plots are generated for both models, along with a side-by-side comparison chart in the knitted HTML output.

### Required Packages (full list)

```r
install.packages(c("dplyr", "tidyr", "ggplot2", "glmnet"))
```
