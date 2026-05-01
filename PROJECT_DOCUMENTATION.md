# Leveaniemi Water-Quality Prediction Project Documentation

## 1. Plain-Language Summary

This project predicts future process-water concentrations for selected contaminants at the Leveaniemi iron ore mine. The contaminants currently modelled are:

- Copper, `Cu`, in `ug/l`
- Ammonium, `NH4`, in `mg/l`
- Chloride, `Cl`, in `mg/l`
- Nickel, `Ni`, in `ug/l`

The project is based on an Excel workbook originally created by an environmental consultant. That workbook contains a year-by-year water-quality model. The consultant model calculates future concentrations using mine production volumes, ore leaching values, water flows, pit-pump water, and the previous stored process-water concentration.

The goal of this codebase is not to blindly copy the consultant's old prediction. The goal is to:

1. Learn how the consultant's model behaves.
2. Reproduce that behaviour using a machine-learning surrogate.
3. Run future forecasts under new or alternative ore-input assumptions.
4. Quantify uncertainty using Monte Carlo simulation.
5. Produce thesis-ready figures and Excel tables.

The most important thing to understand is this:

> The machine-learning model can learn the calculation pattern, but it still needs assumptions about future ore production and leaching. If confirmed future ore inputs are unavailable, the output should be interpreted as scenario-based or sensitivity-based prediction, not as a confirmed operational forecast.

## 2. Files in This Project

The current project contains:

| File | Purpose |
|---|---|
| `code.ipynb` | Main Jupyter notebook. It reads the Excel workbook, trains models, runs forecasts, creates plots, and exports results. |
| `Leveaniemi_data.xlsx` | Input Excel workbook containing the consultant's original process-water model and supporting sheets. |
| `requirements.txt` | Python packages needed to run the notebook. |
| `PROJECT_DOCUMENTATION.md` | This documentation file. |

The notebook expects the Excel workbook to be named:

```text
Leveaniemi_data.xlsx
```

If running in Google Colab, the file must be uploaded into the Colab runtime or placed in Google Drive.

## 3. Dataset Explanation

### 3.1 Source Workbook

The dataset is an Excel workbook based on the consultant's original 2013 water-balance and process-water-quality model. The workbook contains multiple sheets, including:

- `Process water`
- `Back Data Lenvean Pit Lake`
- `Hist modellering`
- `Modell During mining`
- `Mod after closure`
- `Nitrogen`
- `Flow-Production`
- `ore-tail leach calc`
- `Lev-GW`
- `PhreeC-InputPW`
- `PhreeC-outputPW`
- `Copper`
- Alternative-scenario sheets such as `Alt 0`, `Alt 1`, `Alt 2A`, `Alt 2B`, `Alt 3`

The most important sheet for this codebase is:

```text
Process water
```

That sheet contains the process-water mass-balance calculations for each contaminant.

### 3.2 Process Water Sheet Structure

Each contaminant has its own block inside the `Process water` sheet. The notebook extracts four blocks:

| Parameter | Unit | First Excel Data Row Used |
|---|---:|---:|
| `Cu` | `ug/l` | 43 |
| `NH4` | `mg/l` | 82 |
| `Cl` | `mg/l` | 122 |
| `Ni` | `ug/l` | 161 |

The data uses half-year rows:

- `2015.0` means the annual or winter/non-processing-season row.
- `2015.5` means the summer or processing-season row.

This half-year pattern matters because the model behaves differently in `.0` and `.5` rows. The code therefore creates a feature called:

```python
half_year
```

where:

- `0` means `.0` row
- `1` means `.5` row

### 3.3 Important Columns Used

The notebook extracts fixed columns from each contaminant block. The most important are:

| Notebook Field | Workbook Meaning |
|---|---|
| `year` | Half-year timestamp, for example `2025.0` or `2025.5` |
| `production_mton_gm` | Production volume under the consultant's GM scenario |
| `process_leach_gm` | Process-leaching load/rate under the consultant's GM scenario |
| `production_mton_gk` | Production volume under the consultant's GK scenario/proxy |
| `process_leach_gk` | Process-leaching load/rate under the consultant's GK scenario/proxy |
| `process_leach_gl` | Process-leaching load/rate under the consultant's GL scenario/proxy |
| `pit_pump_volume` | Leveaniemi pit-pump water volume |
| `pit_pump_conc` | Pit-pump concentration/inflow concentration, from workbook column `J` |
| `storage_conc_state_col_af` | Storage concentration/state column, from workbook column `AF` |
| `gm_output_conc` | Consultant's GM recurrence output, from workbook column `AI` |

Important clarification:

> Workbook column `J` is treated in this notebook as an inflow/pit-pump concentration, not as the final process-water recurrence target. The notebook uses column `AI`, stored as `gm_output_conc`, as the consultant recurrence output to learn.

The recurrence state is critical. The prediction for a row depends partly on the concentration from the previous row. The notebook therefore creates:

```python
prev_storage_conc
```

This is the previous row's modelled concentration, fed forward into the next prediction.

## 4. What the Consultant Model Is Doing

The consultant model is a mass-balance recurrence.

In simple terms, each row asks:

> Given the previous concentration in storage, the new incoming water, the ore-related contaminant leaching, gains, losses, and storage volume, what should the new process-water concentration be?

This means the model is not just a normal table of independent rows. Each row depends on the row before it.

That is why the notebook forecasts sequentially:

1. Predict 2026.0.
2. Use the predicted 2026.0 concentration as input for 2026.5.
3. Predict 2026.5.
4. Use that value as input for 2027.0.
5. Continue until 2030.0.

This is called a recurrence or state-dependent model.

## 5. The Main Challenge

The consultant originally predicted future water quality using a specific ore-combination assumption. The mine's actual or planned ore combinations have since changed.

The thesis question is therefore not just:

> Can we reproduce the consultant's old forecast?

It is:

> Can we learn the consultant's calculation logic, then use updated ore assumptions to produce new future predictions?

The difficulty is that future prediction requires future input assumptions. For this project, those assumptions are mainly:

- Future production volume
- Future ore mix
- Future process-leaching value for each contaminant

If those values are known from LKAB, the thesis can produce stronger operational forecasts.

If those values are not known, the thesis can still produce useful scenario or sensitivity forecasts.

## 6. Possible Modelling Approaches

There are several reasonable approaches to this problem.

### Approach 1: Rebuild the Consultant Formula Exactly

This means manually translating every Excel formula into Python.

Advantages:

- Most transparent.
- Closest to the original model.
- Good for auditing the consultant workbook.

Disadvantages:

- Time-consuming.
- Easy to make mistakes because Excel formulas differ by row and contaminant.
- Less flexible if future input structures change.

This approach is good when the thesis goal is formula reproduction.

### Approach 2: Machine-Learning Surrogate Model

This is the approach used in the notebook.

The idea is:

> Train a machine-learning model on the consultant's calculated rows so it learns the relationship between inputs and outputs.

The model used is a Random Forest Regressor.

Advantages:

- Can learn nonlinear relationships.
- Does not require manually copying every Excel formula.
- Useful for scenario testing.
- Provides feature-based prediction while preserving the recurrence structure.

Disadvantages:

- Training data is small.
- The model learns the consultant workbook, not necessarily real-world monitoring data.
- Results must be interpreted carefully.

This approach is good when the thesis goal is to reproduce the consultant model behaviour and then run alternative input scenarios.

### Approach 3: Time-Series Extrapolation

This would predict future concentration from past concentration trends only.

Examples:

- Linear trend
- ARIMA
- Exponential smoothing

Advantages:

- Simple.
- Does not require detailed ore inputs.

Disadvantages:

- Ignores the mine-process mechanism.
- Cannot properly represent changed ore combinations.
- Weak for explaining cause and effect.

This approach is not ideal for this thesis because ore mix and leaching are central to the research question.

### Approach 4: Direct Calibration to Monitoring Data

This would train or calibrate the model using daily or monthly observed water-quality monitoring data.

Advantages:

- More connected to real observed water quality.
- Can reduce the gap between consultant predictions and reality.

Disadvantages:

- Requires reliable monitoring data.
- Monitoring data may include operational events not represented in the consultant workbook.
- Needs careful aggregation, for example daily to seasonal or annual values.

This is a good future improvement.

### Approach 5: Scenario and Sensitivity Analysis

This approach tests multiple possible future ore assumptions.

Example:

- 100% GK
- 75% GK and 25% GL
- 50% GK and 50% GL
- 25% GK and 75% GL
- 100% GL

Advantages:

- Honest when true future ore inputs are uncertain.
- Shows how sensitive the prediction is to ore assumptions.
- Useful for thesis discussion.

Disadvantages:

- Does not give one confirmed forecast unless the scenario is confirmed.

This is included in the notebook.

## 7. Our Approach and Why We Chose It

The notebook uses a hybrid of:

- Consultant-model reproduction
- Random Forest surrogate modelling
- Sequential recurrence forecasting
- Monte Carlo uncertainty simulation
- GK/GL sensitivity analysis

### Step 1: Extract Consultant Rows

The notebook reads the `Process water` sheet and extracts rows for:

- Cu
- NH4
- Cl
- Ni

It uses the consultant's calculated rows from 2014 to 2025 for training.

### Step 2: Build Features

The model uses features such as:

- Year
- Half-year indicator
- Production volume
- Process leaching
- Pit-pump volume
- Pit-pump concentration
- Water gains
- Water losses
- Storage volume
- Previous storage concentration
- Derived load and mass-balance proxy variables

The previous concentration is especially important because the original model is recurrent.

### Step 3: Train Separate Models Per Parameter

The notebook trains one model per contaminant.

This is important because the contaminants have very different scales. For example:

- Cu and Ni are in `ug/l`
- NH4 and Cl are in `mg/l`
- Cl leaching values can be very large compared with Cu and Ni

Training separately avoids one parameter dominating another.

### Step 4: Use Random Forest

Random Forest was selected because:

- It handles nonlinear relationships.
- It can work with small to medium tabular datasets.
- It is robust to feature scaling after preprocessing.
- It can capture interactions between flow, leaching, and previous concentration.

The notebook still uses scaling and imputation inside each parameter model.

### Step 5: Validate Using Cross-Validation

The notebook checks whether the Random Forest can reproduce the consultant's calculated values.

The main metric is:

```text
CV R2
```

Interpretation:

| CV R2 | Meaning |
|---:|---|
| Above 0.7 | Good reproduction of consultant logic |
| 0.5 to 0.7 | Moderate reproduction; usable with caution |
| Below 0.5 | Weak reproduction; results should be treated carefully |

Ni may be more difficult because the half-year pattern can dominate the signal.

### Step 6: Forecast Sequentially

The forecast is not done row-by-row independently.

Instead:

1. Predict a concentration.
2. Feed that prediction into the next row as the previous storage concentration.
3. Continue forward.

This matches the structure of the original consultant model.

### Step 7: Add Monte Carlo Uncertainty

The notebook perturbs leaching inputs by plus or minus 15%.

This is repeated many times, currently:

```python
N_MONTE_CARLO = 500
```

The output gives:

- `P10`: lower uncertainty bound
- `P50`: median forecast
- `P90`: upper uncertainty bound

## 8. Important Note About GK and GL Values

Recent feedback says that the GK and GL values inside the consultant workbook are also consultant predictions or visualized scenario values.

This means:

> They are not confirmed current or future mine-plan truth.

The notebook therefore treats them as:

```text
consultant_proxy_sensitivity
```

not as confirmed future data.

If real future ore inputs are unavailable, it is acceptable to use the workbook GK/GL values as proxy assumptions, but the thesis must describe the output as scenario-based or sensitivity-based.

Suggested thesis wording:

> Because confirmed updated ore-plan inputs were unavailable, future predictions were generated using proxy GK/GL scenarios inferred from the consultant workbook. These results should be interpreted as scenario-based forecasts rather than confirmed operational predictions.

## 9. What New Ore Inputs Would Look Like

If the thesis owner or LKAB provides confirmed future ore inputs, they should be entered in the notebook as `NEW_ORE_INPUTS`.

The required columns are:

| Column | Meaning |
|---|---|
| `parameter` | `Cu`, `NH4`, `Cl`, or `Ni` |
| `year` | Half-year row, for example `2026.0` or `2026.5` |
| `production_mton` | Production volume for that row |
| `process_leach` | Process-leaching input for that parameter and row |

Optional columns:

| Column | Meaning |
|---|---|
| `ore_frac_gm` | Fraction of GM, if relevant |
| `ore_frac_gk` | Fraction of GK, if relevant |
| `ore_frac_gl` | Fraction of GL, if relevant |

Example structure:

```python
NEW_ORE_INPUTS = [
    {"parameter": "Cu", "year": 2026.0, "production_mton": 2.1, "process_leach": 18.5, "ore_frac_gk": 0.6, "ore_frac_gl": 0.4},
    {"parameter": "Cu", "year": 2026.5, "production_mton": 3.2, "process_leach": 30.0, "ore_frac_gk": 0.6, "ore_frac_gl": 0.4},
    {"parameter": "NH4", "year": 2026.0, "production_mton": 2.1, "process_leach": 1200.0, "ore_frac_gk": 0.6, "ore_frac_gl": 0.4},
]
```

In practice, this table must cover every forecast year and every parameter.

## 10. How to Ask for Missing Ore Information

If speaking to the thesis owner, use simple language:

> I understand that the GK and GL values in the consultant workbook were also scenario values, not confirmed current data. For the 2026-2030 forecast, do we have confirmed updated production volumes and process-leaching values for the current ore combinations?

More specific:

> For each parameter, Cu, NH4, Cl, and Ni, do we have planned production volume and leaching input for each year or half-year from 2026 to 2030?

If not:

> Is it acceptable to use the workbook GK/GL values as proxy scenarios and present the result as sensitivity analysis rather than a confirmed forecast?

## 11. Notebook Sections and What to Expect

### 11.1 Dependencies

This section imports Python packages.

Expected result:

- If packages are installed, it runs silently.
- If packages are missing, it tells you to install them.

Required packages are:

```text
pandas
numpy
openpyxl
scikit-learn
matplotlib
xlsxwriter
```

In Colab or Jupyter, install with:

```python
%pip install pandas openpyxl scikit-learn matplotlib xlsxwriter
```

### 11.2 Project Configuration

This section defines:

- Workbook filename
- Forecast years
- Monte Carlo settings
- Ore scenarios
- Parameter blocks

Expected result:

- It should print the workbook path being used.

Example:

```text
Using workbook: /content/Leveaniemi_data.xlsx
```

If it cannot find the workbook, upload `Leveaniemi_data.xlsx`.

### 11.3 Workbook Extraction

This section reads the `Process water` sheet and extracts the contaminant blocks.

Expected result:

- A summary table showing rows per parameter.
- A preview of extracted model data.

If row extraction is wrong, the summary table will look strange, for example missing years or missing parameters.

### 11.4 Ore-Input Substitution

This section decides what forecast inputs to use.

Expected result:

- If `NEW_ORE_INPUTS` is provided, the notebook uses those values.
- If `NEW_ORE_INPUTS = None`, the notebook uses consultant-proxy GK/GL sensitivity assumptions.

The displayed table shows the selected ore mix scenario and input mode.

### 11.5 Model Training and Diagnostics

This section trains the Random Forest models.

Expected result:

- A diagnostics table with one row per parameter.

Important columns:

| Column | Meaning |
|---|---|
| `parameter` | Contaminant |
| `training_rows` | Number of rows used for training |
| `full_cv_r2` | Cross-validated R2 for the main model |
| `full_cv_mae` | Cross-validated mean absolute error |
| `half_year_cv_r2` | R2 if separate half-year models are tested |
| `forecast_model` | Whether the notebook uses one model or separate half-year models |

Good result:

```text
CV R2 > 0.7
```

Weak result:

```text
CV R2 < 0.5
```

If a parameter has weak R2, interpret its forecast carefully.

### 11.6 Sequential Forecast and Monte Carlo

This section predicts 2026-2030.

Expected result:

- A forecast table with P10/P50/P90 values.
- An annual subset table for `.0` years.

Important columns:

| Column | Meaning |
|---|---|
| `p10` | Lower uncertainty estimate |
| `p50` | Median forecast |
| `p90` | Upper uncertainty estimate |
| `input_mode` | Whether forecast used real ore inputs or proxy sensitivity inputs |

### 11.7 GK/GL Sensitivity

This section tests different GK/GL mixes.

Expected result:

- A table showing annual forecast results under different GK fractions.

Use this to answer:

> How much does the prediction change if the future ore mix is more GK-heavy or more GL-heavy?

### 11.8 Thesis Figure

This section creates a four-panel figure.

Expected result:

- One panel each for Cu, NH4, Cl, and Ni.
- Historical consultant model values.
- Forecast median line.
- P10-P90 uncertainty band.

The figure is saved as:

```text
outputs/leveaniemi_forecast_bands.png
```

### 11.9 Excel Export

This section writes the output tables to Excel.

Expected result:

```text
outputs/leveaniemi_forecast_values.xlsx
```

Sheets include:

| Sheet | Meaning |
|---|---|
| `Diagnostics` | Model fit statistics |
| `CV_predictions` | Cross-validation predictions versus consultant values |
| `Forecast_half_year` | Forecasts for all half-year rows |
| `Forecast_annual` | Forecasts for `.0` annual rows |
| `Sensitivity_annual` | Annual sensitivity results |
| `Selected_proxy_mix` | Selected proxy ore-mix scenario |
| `Forecast_input_note` | Important note on whether forecast inputs are proxy or user-supplied |

### 11.10 Optional Monitoring Data

This section is only a scaffold.

It can later be used to compare model forecasts with daily monitoring data.

## 12. How to Interpret the Results

### 12.1 Diagnostics

The diagnostics table tells you whether the machine-learning model has learned the consultant's workbook logic.

Use this interpretation:

| CV R2 | Interpretation |
|---:|---|
| `> 0.7` | Good reproduction |
| `0.5-0.7` | Moderate reproduction |
| `< 0.5` | Weak reproduction |

Important:

> A good CV R2 does not mean the consultant model matches real-world monitoring data. It only means the ML model reproduces the consultant workbook calculations well.

### 12.2 Forecast Values

The forecast table gives:

- `P10`: lower-end forecast
- `P50`: median or central forecast
- `P90`: upper-end forecast

Example interpretation:

> For Cu in 2027, P50 is the central predicted concentration. P10 and P90 describe uncertainty from leaching-input variation.

### 12.3 Forecast Figure

The figure shows:

- Historical modelled values from the consultant workbook.
- Future forecast median.
- Uncertainty band.

If the band is wide, the prediction is sensitive to leaching uncertainty.

If the line changes strongly between `.0` and `.5` years, the half-year pattern is important.

### 12.4 Sensitivity Table

The sensitivity table should be used when real future ore inputs are unknown.

It answers:

> Under different assumed GK/GL mixes, how much do predicted concentrations change?

This is useful for discussion because it shows whether results are stable or highly dependent on ore assumptions.

## 13. Possible Questions and Suggested Answers

### Question 1: Is this model predicting real measured water quality?

Answer:

> Not directly. The model first learns the consultant's workbook logic. It predicts what the consultant-style model would output under selected future input assumptions. To predict real measured monitoring data, the model would need calibration or validation against monitoring observations.

### Question 2: Why use machine learning if the consultant already had formulas?

Answer:

> The consultant formulas are embedded across a complex Excel workbook. A Random Forest surrogate allows us to learn the relationship between inputs and outputs without manually rewriting every formula, while still preserving the important recurrence structure.

### Question 3: Why do we need future ore input values?

Answer:

> Future concentration depends on what ore is processed, how much is processed, and how much contaminant leaches from that ore. The model can learn the calculation logic, but it cannot know the real future ore plan unless that information is supplied.

### Question 4: If we do not have future ore inputs, can we still predict?

Answer:

> Yes, but only as a scenario or sensitivity forecast. We can use inferred or proxy values from the workbook and clearly state that the results are conditional on those assumptions.

### Question 5: Are the workbook GK/GL values actual updated mine data?

Answer:

> Based on feedback, no. They are consultant-derived scenario/proxy values. They can be useful for sensitivity analysis but should not be presented as confirmed future mine-plan values.

### Question 6: What does a high CV R2 mean?

Answer:

> It means the machine-learning model reproduced the consultant workbook calculations well. It does not automatically mean the model predicts real-world monitoring data accurately.

### Question 7: What if Ni has poor model performance?

Answer:

> Ni may be strongly affected by half-year alternation or by small values that are harder to learn from limited data. If Ni CV R2 is low, its forecast should be interpreted more cautiously, and separate half-year models or formula-based modelling may be considered.

### Question 8: Why use Monte Carlo simulation?

Answer:

> Leaching inputs are uncertain. Monte Carlo simulation repeatedly varies leaching values within a reasonable range and shows how much the forecast changes. This gives uncertainty bands instead of a single overconfident line.

### Question 9: What does P10/P50/P90 mean?

Answer:

> P50 is the median forecast. P10 is a lower estimate and P90 is an upper estimate. Roughly speaking, most simulation results fall between P10 and P90.

### Question 10: Can this be used in a thesis?

Answer:

> Yes, if framed correctly. It is suitable as a consultant-model surrogate and scenario-forecasting workflow. It should not be described as a confirmed operational forecast unless confirmed future ore inputs and monitoring validation are included.

## 14. How to Get Better Results

### 14.1 Get Confirmed Future Ore Inputs

This is the most important improvement.

Needed data:

- Future production volume by year or half-year
- Future ore mix
- Future leaching values for Cu, NH4, Cl, and Ni

With this, the forecast becomes much more defensible.

### 14.2 Add Monitoring Data

Daily or monthly monitoring data can be used to:

- Compare predictions with observed concentrations.
- Understand model bias.
- Calibrate the forecast.

The monitoring data should be aggregated to match the model time scale, for example:

- Annual median
- Summer/winter median
- Half-year average

### 14.3 Rebuild Key Mass-Balance Formula Directly

For maximum transparency, the most important Excel recurrence formulas could be translated directly into Python.

This would allow comparison between:

- Excel formula result
- Python formula result
- Random Forest surrogate result

### 14.4 Improve Ni Modelling

If Ni performance is weak:

- Train separate `.0` and `.5` models.
- Use simpler models such as linear regression for comparison.
- Check whether the target column is correct.
- Check whether Ni values are too close to zero or strongly seasonal.

### 14.5 Tune Monte Carlo Assumptions

The current leaching uncertainty is:

```python
LEACH_PERTURBATION = 0.15
```

This means plus or minus 15%.

If expert knowledge suggests a different uncertainty, change it.

Examples:

- `0.10` for plus or minus 10%
- `0.25` for plus or minus 25%

### 14.6 Increase Monte Carlo Runs

The current number of runs is:

```python
N_MONTE_CARLO = 500
```

This is already above the 200-run minimum. For smoother uncertainty bands, use:

```python
N_MONTE_CARLO = 1000
```

This will take longer to run.

### 14.7 Compare Several Model Types

For thesis robustness, compare Random Forest against:

- Linear regression
- Gradient boosting
- Formula-based recurrence
- Seasonal baseline model

This helps justify why Random Forest was selected.

## 15. Recommended Thesis Framing

A safe and accurate way to describe the work is:

> This study developed a machine-learning surrogate of the consultant's process-water mass-balance model for Leveaniemi. The model was trained on consultant-calculated rows from 2014-2025 and used previous storage concentration, production volume, leaching inputs, pit-pump water, and water-balance variables as predictors. Forecasts for 2026-2030 were generated sequentially so that each predicted concentration was fed into the next row as the storage state. Uncertainty was quantified using Monte Carlo perturbation of leaching inputs. Where confirmed future ore-plan inputs were unavailable, GK/GL forecasts were treated as proxy scenario and sensitivity results rather than confirmed operational predictions.

## 16. Practical Checklist Before Presenting Results

Before using the outputs in the thesis, check:

- The workbook file is correctly loaded.
- The extracted years and rows look correct.
- CV R2 is acceptable for each parameter.
- Weak parameters are discussed cautiously.
- The forecast input mode is clear.
- If `NEW_ORE_INPUTS = None`, results are labelled as proxy/sensitivity forecasts.
- The figure axes and units are correct.
- The Excel output includes `Forecast_input_note`.
- Monitoring data comparison is added if available.

## 17. Final Interpretation

This codebase is a good modelling framework for a thesis because it is transparent about:

- What data is used.
- What is learned from the consultant model.
- How future scenarios are created.
- How uncertainty is represented.
- What assumptions limit the conclusions.

Its strongest current use is:

> Scenario-based prediction and sensitivity analysis based on the consultant model structure.

Its strongest future improvement is:

> Replace proxy GK/GL assumptions with confirmed future ore production and leaching inputs, then compare predictions against monitoring data.

