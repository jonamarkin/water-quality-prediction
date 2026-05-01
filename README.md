# Leveaniemi Water-Quality Prediction

This project supports a master's thesis on process-water quality prediction for the Leveaniemi iron ore mine in northern Sweden. It uses the consultant's original Excel water-balance model in two ways: first as a deterministic consultant-formula model, and second as a machine-learning surrogate model. Both methods can produce LK scenario hindcasts and 2026-2030 forecasts with uncertainty bands.

The model now covers all consultant `Process water` blocks that have final recurrence outputs:

- `Cu`, `Ni`, `Zn`, `Co`, `Mo`, `As`, and `Cr` in `ug/l`
- `NH4`, `Cl`, `SO4`, `Ca`, `NO3`, and `PO4P` in `mg/l`

Fluoride, `F`, appears in the workbook as an input-style block, but it does not contain the same final `AI` recurrence output, so it is not included as a modelled target.

## Project Goal

The original consultant model predicted future process-water concentrations using an old ore-combination assumption. The consultant's purpose was to check whether Leveaniemi water and planned ore processing could remain within element-specific concentration limits.

This project extracts the consultant's calculation logic from the workbook, implements it directly as a deterministic formula, and also trains a Random Forest model to learn the same behaviour. The current operational question is different: the mine may use Leveaniemi-Kiruna, LK, with ratios such as 50/50, 40/60, or other mixes that vary over time. The notebook therefore supports both fixed proxy scenarios and a time-varying ore-mix schedule.

Where confirmed future ore inputs are unavailable, GK/GL/LK forecasts should be interpreted as proxy scenario or sensitivity analysis, not as confirmed operational predictions. LK means Leveaniemi-Kiruna. Because LK was not directly included in the consultant workbook, the notebook infers an LK proxy from the available Leveaniemi and Kiruna leaching-rate structure.

## Repository Contents

| File | Description |
|---|---|
| `code.ipynb` | Main notebook. It is split into Method 1, deterministic consultant formula, and optional Method 2, ML surrogate. Method 1 can be run and exported without running any ML cells. |
| `Leveaniemi_data.xlsx` | Input workbook containing the consultant's original process-water model. |
| `parameters_used.xlsx` | New workbook containing 2020-2025 observed/seasonal data, LK ore-mix ratios, production references, and validation inputs. |
| `PROJECT_DOCUMENTATION.md` | Detailed explanation of the dataset, modelling choices, assumptions, results, and thesis interpretation. |
| `CONSULTANT_MATHEMATICAL_MODEL.md` | Extracted explanation of the consultant's original mass-balance recurrence and Excel formulas. |
| `requirements.txt` | Python dependencies needed to run the notebook. |

## Setup

Install the required Python packages:

```bash
pip install -r requirements.txt
```

If running in Google Colab and you only want Method 1, install:

```python
%pip install pandas numpy openpyxl matplotlib xlsxwriter
```

If you also want the optional ML method, install:

```python
%pip install scikit-learn
```

Then upload `Leveaniemi_data.xlsx` and `parameters_used.xlsx` to the Colab runtime or place them in Google Drive.

## How to Run

Open and run:

```text
code.ipynb
```

Run the notebook from the top through **Method 1 Hindcast Validation and Export** if you only want the consultant-formula approach. Stop there if you do not want any ML results.

Method 1 will:

1. Read the `Process water` sheet from the Excel workbook.
2. Extract all modelled contaminant blocks from the consultant workbook.
3. Run the deterministic consultant-formula method and check that it reproduces the workbook's GM output.
4. Run a deterministic 2026-2030 forecast.
5. Load `parameters_used.xlsx`.
6. Run a 2020-2025 LK hindcast validation against actual observed data.
7. Create Method 1 forecast and validation figures.
8. Export the Method 1 workbook.

The optional Method 2 section then:

1. Imports `scikit-learn`.
2. Trains Random Forest models to reproduce the consultant's calculated concentrations.
3. Runs the ML 2026-2030 forecast and sensitivity analysis.
4. Runs the ML 2020-2025 hindcast validation.
5. Exports ML and combined comparison workbooks.

## Outputs

After running the notebook, outputs are written to:

```text
outputs/
```

Expected files:

| Output | Description |
|---|---|
| `leveaniemi_consultant_formula_forecast_bands.png` | Multi-panel forecast plot from the direct deterministic consultant formula. |
| `leveaniemi_hindcast_validation_consultant_formula_2020_2025.png` | Multi-panel plot comparing deterministic consultant-formula LK hindcast predictions with observed seasonal concentrations. |
| `leveaniemi_method1_consultant_formula_outputs.xlsx` | Complete Method 1 workbook with formula reproduction checks, forecast values, observed 2020-2025 data, formula hindcast values, and error metrics. |
| `leveaniemi_forecast_bands.png` | Optional Method 2 multi-panel plot with historical model values and ML forecast uncertainty bands. |
| `leveaniemi_method2_ml_outputs.xlsx` | Optional Method 2 workbook containing ML diagnostics, ML forecasts, sensitivity results, formula forecast references, and input notes. |
| `leveaniemi_hindcast_validation_ml_2020_2025.png` | Multi-panel plot comparing ML LK hindcast predictions with observed seasonal concentrations. |
| `leveaniemi_hindcast_validation_2020_2025.xlsx` | Optional combined workbook containing LK mix schedule, observed data, deterministic and ML hindcast predictions, and side-by-side error metrics. |

## Important Modelling Note

The deterministic formula reproduces the consultant workbook logic directly. The machine-learning model learns that same logic as a surrogate. Neither method automatically proves that the consultant model matches real monitoring data; that is why the 2020-2025 hindcast validation section is included.

Only the 2020-2025 data from `parameters_used.xlsx` is treated as actual observed monitoring data. Rows before 2020 in the consultant workbook are model/formula rows, not actual observations; they are used only to reconstruct or learn the consultant recurrence and to seed sequential predictions.

For final operational prediction, confirmed future ore inputs are needed, especially:

- production volume
- ore mix
- process-leaching values for each modelled parameter

If those inputs are unavailable, the results should be presented as scenario-based forecasts or sensitivity analysis.

If element-specific limits are known, enter them in `ELEMENT_LIMITS` in `code.ipynb`. The notebook will add limit lines to the forecast figure and export them to the results workbook.

The default proxy scenario in the notebook is currently:

```text
LK50_50
```

This represents an inferred 50/50 Leveaniemi-Kiruna sensitivity case, not confirmed operational mine-plan data. To model changing mixes over time, fill `TIME_VARYING_ORE_MIX` in `code.ipynb`.

## Documentation

For a full explanation of the dataset, code, assumptions, possible thesis questions, and interpretation of results, see:

```text
PROJECT_DOCUMENTATION.md
```

For the extracted mathematical model used by the consultant, see:

```text
CONSULTANT_MATHEMATICAL_MODEL.md
```
