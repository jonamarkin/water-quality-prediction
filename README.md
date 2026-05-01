# Leveaniemi Water-Quality Prediction

This project supports a master's thesis on process-water quality prediction for the Leveaniemi iron ore mine in northern Sweden. It uses the consultant's original Excel water-balance model to train machine-learning surrogate models for selected contaminants, then produces 2026-2030 scenario forecasts with uncertainty bands.

The model currently covers:

- Cu, copper, in `ug/l`
- NH4, ammonium, in `mg/l`
- Cl, chloride, in `mg/l`
- Ni, nickel, in `ug/l`

## Project Goal

The original consultant model predicted future process-water concentrations using an old ore-combination assumption. The consultant's purpose was to check whether Leveaniemi water and planned ore processing could remain within element-specific concentration limits.

This project learns the consultant's calculation logic from the workbook and then allows future predictions under alternative ore-input assumptions. The current operational question is different: the mine may use Leveaniemi-Kiruna, LK, with ratios such as 50/50, 40/60, or other mixes that vary over time. The notebook therefore supports both fixed proxy scenarios and a time-varying ore-mix schedule.

Where confirmed future ore inputs are unavailable, GK/GL/LK forecasts should be interpreted as proxy scenario or sensitivity analysis, not as confirmed operational predictions. LK means Leveaniemi-Kiruna. Because LK was not directly included in the consultant workbook, the notebook infers an LK proxy from the available Leveaniemi and Kiruna leaching-rate structure.

## Repository Contents

| File | Description |
|---|---|
| `code.ipynb` | Main notebook for data extraction, model training, forecasting, Monte Carlo simulation, plotting, and Excel export. |
| `Leveaniemi_data.xlsx` | Input workbook containing the consultant's original process-water model. |
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

Then upload `Leveaniemi_data.xlsx` to the Colab runtime or place it in Google Drive.

## How to Run

Open and run:

```text
code.ipynb
```

Run the notebook from top to bottom.

The notebook will:

1. Read the `Process water` sheet from the Excel workbook.
2. Extract contaminant blocks for Cu, NH4, Cl, and Ni.
3. Train Random Forest models to reproduce the consultant's calculated concentrations.
4. Run sequential forecasts for 2026-2030, including the selected proxy ore-combination scenario or time-varying ore-mix schedule.
5. Apply Monte Carlo uncertainty analysis.
6. Create a four-panel forecast figure, with optional element-limit lines if limits are provided.
7. Export forecast tables to Excel.

## Outputs

After running the notebook, outputs are written to:

```text
outputs/
```

Expected files:

| Output | Description |
|---|---|
| `leveaniemi_forecast_bands.png` | Four-panel plot with historical model values and forecast uncertainty bands. |
| `leveaniemi_forecast_values.xlsx` | Excel workbook containing diagnostics, forecasts, sensitivity results, and input notes. |

## Important Modelling Note

The machine-learning model learns the consultant workbook logic. It does not automatically prove that the consultant model matches real monitoring data.

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
