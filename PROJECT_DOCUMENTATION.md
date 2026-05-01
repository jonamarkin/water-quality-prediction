# Leveaniemi Water-Quality Prediction Project Documentation

## 1. Plain-Language Summary

This project predicts future process-water concentrations for selected contaminants at the Leveaniemi iron ore mine. The notebook now models all consultant `Process water` blocks that contain final recurrence outputs:

- `Cu`, `Ni`, `Zn`, `Co`, `Mo`, `As`, and `Cr` in `ug/l`
- `NH4`, `Cl`, `SO4`, `Ca`, `NO3`, and `PO4P` in `mg/l`

Fluoride, `F`, appears in the workbook as an input-style block, but it does not contain the same final `AI` recurrence output, so it is not included as a modelled target.

The project is based on an Excel workbook originally created by an environmental consultant. That workbook contains a year-by-year water-quality model. The consultant model calculates future concentrations using mine production volumes, ore leaching values, water flows, pit-pump water, and the previous stored process-water concentration.

The consultant's practical question was whether Leveaniemi water and planned ore processing could be used in mining activities while staying within element-specific concentration limits. The current thesis question extends that work: the mine may now use Leveaniemi-Kiruna, LK, with ratios such as 50/50, 40/60, or other mixes that vary over time.

If the element-specific limits are known, they should be entered into the notebook in `ELEMENT_LIMITS`. This allows the forecast plots and exported tables to show whether predicted concentrations approach or exceed the relevant thresholds.

The goal of this codebase is not to blindly copy the consultant's old prediction. The goal is to:

1. Learn how the consultant's model behaves.
2. Reproduce that behaviour directly using the consultant's deterministic formula.
3. Reproduce that behaviour using a machine-learning surrogate.
4. Run future forecasts under new or alternative ore-input assumptions.
5. Quantify uncertainty using Monte Carlo simulation.
6. Produce thesis-ready figures and Excel tables.

The most important thing to understand is this:

> The deterministic formula and machine-learning model can both reproduce the consultant's calculation pattern, but both still need assumptions about future ore production and leaching. If confirmed future ore inputs are unavailable, the output should be interpreted as scenario-based or sensitivity-based prediction, not as a confirmed operational forecast.

The only data treated as actual observed monitoring data is the 2020-2025 data in `parameters_used.xlsx`. Pre-2020 rows in the consultant workbook are not treated as observations; they are consultant model/formula rows used only to reconstruct or learn the consultant recurrence and to seed sequential predictions.

## 2. Files in This Project

The current project contains:

| File | Purpose |
|---|---|
| `code.ipynb` | Main Jupyter notebook. It reads the Excel workbook, runs the deterministic formula method, trains ML models, runs forecasts, creates plots, and exports results. |
| `Leveaniemi_data.xlsx` | Input Excel workbook containing the consultant's original process-water model and supporting sheets. |
| `parameters_used.xlsx` | New validation workbook containing 2020-2025 observed/seasonal data, LK mix ratios, production references, and supporting parameters. |
| `requirements.txt` | Python packages needed to run the notebook. |
| `PROJECT_DOCUMENTATION.md` | This documentation file. |
| `CONSULTANT_MATHEMATICAL_MODEL.md` | Detailed extraction of the consultant's original mass-balance recurrence and Excel formula logic. |

The notebook expects the main Excel workbook to be named:

```text
Leveaniemi_data.xlsx
```

For the 2020-2025 validation section, it also expects:

```text
parameters_used.xlsx
```

If running in Google Colab, both files must be uploaded into the Colab runtime or placed in Google Drive.

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

Each contaminant has its own block inside the `Process water` sheet. The notebook extracts the modelled blocks that have final recurrence outputs:

| Parameter | Unit | First Excel Data Row Used |
|---|---:|---:|
| `Cu` | `ug/l` | 43 |
| `NH4` | `mg/l` | 82 |
| `Cl` | `mg/l` | 122 |
| `Ni` | `ug/l` | 161 |
| `Zn` | `ug/l` | 200 |
| `Co` | `ug/l` | 240 |
| `Mo` | `ug/l` | 279 |
| `SO4` | `mg/l` | 321 |
| `Ca` | `mg/l` | 361 |
| `NO3` | `mg/l` | 397 |
| `PO4P` | `mg/l` | 433 |
| `As` | `ug/l` | 507 |
| `Cr` | `ug/l` | 544 |

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

The consultant originally predicted future water quality using a specific ore-combination assumption. In the consultant-style mixes, the workbook uses a fixed 40/60 structure for the helper ore-combination calculations. The mine's actual or planned ore combinations have since changed, and current LK operations may vary between ratios such as 50/50, 40/60, and other schedules.

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

### Approach 1: Deterministic Consultant Formula Model

This means translating the key Excel recurrence into Python and running it directly as a prediction model.

Advantages:

- Most transparent.
- Closest to the original model.
- Good for auditing the consultant workbook.
- Produces an independent, non-ML baseline.

Disadvantages:

- Time-consuming.
- Easy to make mistakes because Excel formulas differ by row and contaminant.
- Less flexible if future input structures change.

This approach is now implemented in the notebook as its own complete method. The notebook checks whether the Python formula reproduces the consultant workbook's GM output, then uses the same formula for LK hindcast and future scenario prediction.

### Approach 2: Machine-Learning Surrogate Model

This is the second approach used in the notebook.

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
- Deterministic consultant-formula modelling
- Random Forest surrogate modelling
- Sequential recurrence forecasting
- Monte Carlo uncertainty simulation
- GK/GL/LK proxy sensitivity analysis

### Step 1: Extract Consultant Rows

The notebook reads the `Process water` sheet and extracts rows for all modelled parameters listed in `PARAM_BLOCKS`, including the original four focus parameters and the additional blocks such as Zn, Co, Mo, SO4, Ca, NO3, PO4P, As, and Cr.

It uses the consultant's calculated rows from 2014 to 2025 as the source data. The deterministic formula section uses these rows to check whether the Python recurrence reproduces the workbook, while the ML section uses them for training.

### Step 2: Implement the Deterministic Consultant Formula

The notebook directly implements the consultant recurrence:

1. Calculate contaminant load from ore leaching and water sources.
2. Divide by the effective water volume.
3. Mix that incoming concentration with the previous stored concentration.
4. Feed the new concentration into the next row.

This gives a complete non-ML method. It is useful because it is transparent and close to the consultant's original Excel approach.

### Step 3: Build Features for ML

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

### Step 4: Train Separate ML Models Per Parameter

The notebook trains one model per contaminant.

This is important because the contaminants have very different scales. For example:

- Several metals are in `ug/l`
- NH4, Cl, SO4, Ca, NO3, and PO4P are in `mg/l`
- Some leaching values, especially mg/kg-style blocks such as Cl and SO4, can be very large compared with metal blocks

Training separately avoids one parameter dominating another.

### Step 5: Use Random Forest

Random Forest was selected because:

- It handles nonlinear relationships.
- It can work with small to medium tabular datasets.
- It is robust to feature scaling after preprocessing.
- It can capture interactions between flow, leaching, and previous concentration.

The notebook still uses scaling and imputation inside each parameter model.

### Step 6: Validate the ML Surrogate Using Cross-Validation

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

### Step 7: Forecast Sequentially

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

## 8. Important Note About GK, GL, and LK Values

Recent feedback says that the GK and GL values inside the consultant workbook are also consultant predictions or visualized scenario values.

This means:

> They are not confirmed current or future mine-plan truth.

The newest requested scenario is LK, meaning Leveaniemi-Kiruna. The consultant did not directly include LK as its own process-water scenario. The notebook therefore infers an LK proxy from the available Leveaniemi and Kiruna leaching-rate structure already present in the workbook.

Unlike the consultant's fixed-style helper mixes, current LK operation can vary over time. The notebook supports this through:

```python
TIME_VARYING_ORE_MIX
```

For example:

```python
TIME_VARYING_ORE_MIX = pd.DataFrame([
    {"year": 2026.0, "lk": 1.0, "lk_leveaniemi": 0.50, "lk_kiruna": 0.50},
    {"year": 2026.5, "lk": 1.0, "lk_leveaniemi": 0.40, "lk_kiruna": 0.60},
])
```

This allows the forecast to ask:

> What happens to each modelled process-water parameter over time under each changing ore combination?

The notebook therefore treats them as:

```text
consultant_proxy_sensitivity
```

not as confirmed future data.

If real future ore inputs are unavailable, it is acceptable to use the workbook GK/GL values and inferred LK values as proxy assumptions, but the thesis must describe the output as scenario-based or sensitivity-based.

Suggested thesis wording:

> Because confirmed updated ore-plan inputs were unavailable, future predictions were generated using proxy GK/GL scenarios and an inferred LK scenario from the consultant workbook structure. These results should be interpreted as scenario-based forecasts rather than confirmed operational predictions.

## 9. What New Ore Inputs Would Look Like

If the thesis owner or LKAB provides confirmed future ore inputs, they should be entered in the notebook as `NEW_ORE_INPUTS`.

The required columns are:

| Column | Meaning |
|---|---|
| `parameter` | Any modelled parameter in `PARAM_BLOCKS`, for example `Cu`, `NH4`, `Cl`, `Ni`, `Zn`, `SO4`, or `As` |
| `year` | Half-year row, for example `2026.0` or `2026.5` |
| `production_mton` | Production volume for that row |
| `process_leach` | Process-leaching input for that parameter and row |

Optional columns:

| Column | Meaning |
|---|---|
| `ore_frac_gm` | Fraction of GM, if relevant |
| `ore_frac_gk` | Fraction of GK, if relevant |
| `ore_frac_gl` | Fraction of GL, if relevant |
| `ore_frac_lk` | Fraction of LK, if relevant |
| `lk_leveaniemi_frac` | Leveaniemi share within LK, if LK is used |
| `lk_kiruna_frac` | Kiruna share within LK, if LK is used |

Example structure:

```python
NEW_ORE_INPUTS = [
    {"parameter": "Cu", "year": 2026.0, "production_mton": 2.1, "process_leach": 18.5, "ore_frac_lk": 1.0},
    {"parameter": "Cu", "year": 2026.5, "production_mton": 3.2, "process_leach": 30.0, "ore_frac_lk": 1.0},
    {"parameter": "NH4", "year": 2026.0, "production_mton": 2.1, "process_leach": 1200.0, "ore_frac_lk": 1.0},
]
```

In practice, this table must cover every forecast year and every parameter.

## 10. How to Ask for Missing Ore Information

If speaking to the thesis owner, use simple language:

> I understand that the GK and GL values in the consultant workbook were also scenario values, not confirmed current data. For the 2026-2030 forecast, do we have confirmed updated production volumes and process-leaching values for the current ore combinations?

More specific:

> For each modelled parameter, do we have planned production volume and leaching input for each year or half-year from 2026 to 2030?

If not:

> Is it acceptable to use the workbook GK/GL values and an inferred LK value as proxy scenarios and present the result as sensitivity analysis rather than a confirmed forecast?

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
- Optional element-specific concentration limits

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
- If `TIME_VARYING_ORE_MIX` is provided, the notebook uses a year-by-year or season-by-season proxy ore schedule.
- If both `NEW_ORE_INPUTS = None` and `TIME_VARYING_ORE_MIX = None`, the notebook uses fixed consultant-proxy GK/GL sensitivity assumptions and an inferred LK proxy.

The displayed table shows the selected ore mix scenario and input mode.

### 11.5 Deterministic Consultant Formula Method

This section runs the direct formula-based approach.

Expected result:

- A reproduction metrics table comparing the Python formula against the consultant workbook's GM output.
- A deterministic formula forecast table with P10/P50/P90 uncertainty values.
- A deterministic formula forecast figure.

Important interpretation:

> This is the closest Python version of the consultant's own method. It is separate from machine learning.

For the workbook-reproduction check, the code uses the workbook's own `AF` storage-state column so it can verify the Excel calculation accurately. For hindcast and future forecast, the same formula is then run sequentially, meaning each predicted concentration becomes the next storage state.

### 11.6 Model Training and Diagnostics

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

### 11.7 Sequential Forecast and Monte Carlo

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

### 11.8 Proxy Ore-Combination Sensitivity

This section tests different GK, GL, and LK proxy mixes. It includes fixed LK examples such as 50/50, 40/60, and 60/40 Leveaniemi-Kiruna.

Expected result:

- A table showing annual forecast results under different GK fractions.

Use this to answer:

> How much does the prediction change if the future ore mix is more GK-heavy or more GL-heavy?

### 11.9 Thesis Figure

This section creates a multi-panel figure.

Expected result:

- One panel for each modelled parameter.
- Historical consultant model values.
- Forecast median line.
- P10-P90 uncertainty band.
- Optional horizontal limit lines if `ELEMENT_LIMITS` is filled.

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
| `Element_limits` | Optional element limits used for threshold comparison |
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

> Under different assumed GK/GL/LK mixes, how much do predicted concentrations change?

For the current operational question, the most important interpretation is:

> Under a changing LK schedule, how do the modelled concentrations evolve over time, and do any of them approach or exceed relevant limits?

This is useful for discussion because it shows whether results are stable or highly dependent on ore assumptions.

## 13. Possible Questions and Suggested Answers

### Question 1: Is this model predicting real measured water quality?

Answer:

> Not directly. The model first learns the consultant's workbook logic. It predicts what the consultant-style model would output under selected future input assumptions. To predict real measured monitoring data, the model would need calibration or validation against monitoring observations.

### Question 2: Why use machine learning if the consultant already had formulas?

Answer:

> The deterministic formula method gives a transparent reproduction of the consultant's approach. The machine-learning method is added as a second method to test whether a data-driven surrogate can learn the same workbook behaviour and support scenario comparison. Presenting both gives the thesis a stronger comparison.

### Question 3: Why do we need future ore input values?

Answer:

> Future concentration depends on what ore is processed, how much is processed, and how much contaminant leaches from that ore. The model can learn the calculation logic, but it cannot know the real future ore plan unless that information is supplied.

### Question 4: If we do not have future ore inputs, can we still predict?

Answer:

> Yes, but only as a scenario or sensitivity forecast. We can use inferred or proxy values from the workbook and clearly state that the results are conditional on those assumptions.

### Question 5: Are the workbook GK/GL/LK values actual updated mine data?

Answer:

> Based on feedback, no. GK and GL are consultant-derived scenario/proxy values. LK was not directly included by the consultant and is inferred from the available Leveaniemi and Kiruna rates. These values can be useful for sensitivity analysis but should not be presented as confirmed future mine-plan values.

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

> Yes, if framed correctly. It now contains two complete methods: a deterministic consultant-formula method and a machine-learning surrogate method. It should not be described as a confirmed operational forecast unless confirmed future ore inputs and monitoring validation are included.

## 14. How to Get Better Results

### 14.1 Get Confirmed Future Ore Inputs

This is the most important improvement.

Needed data:

- Future production volume by year or half-year
- Future ore mix
- Future leaching values for each modelled parameter

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

### 14.3 Improve the Direct Formula Inputs

The key mass-balance recurrence has now been translated directly into Python. The next improvement is to replace proxy future inputs with confirmed operational values.

This allows comparison between:

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

> This study implemented two versions of the consultant's process-water mass-balance model for Leveaniemi. First, the consultant's deterministic recurrence was translated into Python and checked against the original workbook output. Second, a Random Forest surrogate was trained on consultant-calculated rows to learn the relationship between production, ore leaching, water-balance inputs, previous storage concentration, and modelled concentration. Both methods were run sequentially so each predicted concentration became the next storage state. Uncertainty was quantified using Monte Carlo perturbation of leaching inputs. Where confirmed future ore-plan inputs were unavailable, GK/GL forecasts and the inferred LK forecast were treated as proxy scenario and sensitivity results rather than confirmed operational predictions.

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

> Replace proxy GK/GL/LK assumptions with confirmed future ore production and leaching inputs, then compare predictions against monitoring data.
