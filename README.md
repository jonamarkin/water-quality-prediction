# Leveaniemi Water-Quality Prediction

This project supports a master's thesis on process-water quality prediction for the Leveaniemi iron ore mine in northern Sweden. It uses the consultant's original Excel water-balance model in two ways: first as a deterministic consultant-formula model, and second as a machine-learning surrogate model. Both methods can produce LK scenario hindcasts and 2026-2030 forecasts with uncertainty bands.

The model currently covers:

- Cu, copper, in `ug/l`
- NH4, ammonium, in `mg/l`
- Cl, chloride, in `mg/l`
- Ni, nickel, in `ug/l`

## Project Goal

The original consultant model predicted future process-water concentrations using an old ore-combination assumption. The consultant's purpose was to check whether Leveaniemi water and planned ore processing could remain within element-specific concentration limits.

This project extracts the consultant's calculation logic from the workbook, implements it directly as a deterministic formula, and also trains a Random Forest model to learn the same behaviour. The current operational question is different: the mine may use Leveaniemi-Kiruna, LK, with ratios such as 50/50, 40/60, or other mixes that vary over time. The notebook therefore supports both fixed proxy scenarios and a time-varying ore-mix schedule.

Where confirmed future ore inputs are unavailable, GK/GL/LK forecasts should be interpreted as proxy scenario or sensitivity analysis, not as confirmed operational predictions. LK means Leveaniemi-Kiruna. Because LK was not directly included in the consultant workbook, the notebook infers an LK proxy from the available Leveaniemi and Kiruna leaching-rate structure.

## Repository Contents

| File | Description |
|---|---|
| `code.ipynb` | Main notebook for data extraction, deterministic formula modelling, ML model training, 2020-2025 hindcast validation, forecasting, Monte Carlo simulation, plotting, and Excel export. |
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

If running in Google Colab, install packages inside the notebook:

```python
%pip install pandas openpyxl scikit-learn matplotlib xlsxwriter
```

Then upload `Leveaniemi_data.xlsx` and `parameters_used.xlsx` to the Colab runtime or place them in Google Drive.

## How to Run

Open and run:

```text
code.ipynb
```

Run the notebook from top to bottom.

The notebook will:

1. Read the `Process water` sheet from the Excel workbook.
2. Extract contaminant blocks for Cu, NH4, Cl, and Ni.
3. Run the deterministic consultant-formula method and check that it reproduces the workbook's GM output.
4. Train Random Forest models to reproduce the consultant's calculated concentrations.
5. Load `parameters_used.xlsx` and run a 2020-2025 LK hindcast validation for both methods.
6. Run sequential forecasts for 2026-2030, including the selected proxy ore-combination scenario or time-varying ore-mix schedule.
7. Apply Monte Carlo uncertainty analysis.
8. Create four-panel validation and forecast figures, with optional element-limit lines if limits are provided.
9. Export forecast and hindcast tables to Excel.

## Outputs

After running the notebook, outputs are written to:

```text
outputs/
```

Expected files:

| Output | Description |
|---|---|
| `leveaniemi_forecast_bands.png` | Four-panel plot with historical model values and forecast uncertainty bands. |
| `leveaniemi_consultant_formula_forecast_bands.png` | Four-panel forecast plot from the direct deterministic consultant formula. |
| `leveaniemi_forecast_values.xlsx` | Excel workbook containing ML diagnostics, deterministic-formula checks, forecasts, sensitivity results, and input notes. |
| `leveaniemi_hindcast_validation_consultant_formula_2020_2025.png` | Four-panel plot comparing deterministic consultant-formula LK hindcast predictions with observed seasonal concentrations. |
| `leveaniemi_hindcast_validation_ml_2020_2025.png` | Four-panel plot comparing ML LK hindcast predictions with observed seasonal concentrations. |
| `leveaniemi_hindcast_validation_2020_2025.xlsx` | Excel workbook containing LK mix schedule, observed data, deterministic and ML hindcast predictions, and error metrics. |

## Important Modelling Note

The deterministic formula reproduces the consultant workbook logic directly. The machine-learning model learns that same logic as a surrogate. Neither method automatically proves that the consultant model matches real monitoring data; that is why the 2020-2025 hindcast validation section is included.

For final operational prediction, confirmed future ore inputs are needed, especially:

- production volume
- ore mix
- process-leaching values for Cu, NH4, Cl, and Ni

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
