# Insights from `agnes_predictions.xlsx`

This workbook appears to be Agnes's manually adjusted version of the consultant model. It contains a full consultant-style workbook, including `Process water`, `Decimal date`, `Flow-Production`, `ore-tail leach calc`, and related sheets.

## Main Finding

Her results are useful because they show the same broad modelling direction as the consultant workbook, but they do not consistently match the observed 2020-2025 monitoring data.

Using the `Process water` output column and the observed table in `Decimal date`, the closest matches are:

| Parameter | Pattern |
|---|---|
| `Mo` | Closest overall; predicted mean is about 82% of observed mean. |
| `NO3` | Reasonable order of magnitude, but still low. |
| `Co` | Similar average magnitude, but seasonal points still differ. |

The largest mismatches are:

| Parameter | Pattern |
|---|---|
| `As` | Very large overprediction; model is around 78 times observed on average. |
| `Cr` | Strong overprediction; around 7 times observed on average. |
| `Zn` | Strong overprediction; around 3.5 times observed on average. |
| `Cu` | Strong overprediction; around 4 times observed on average. |
| `SO4` | Strong underprediction; model is only about 21% of observed mean. |
| `Ca` | Strong underprediction; model is only about 21% of observed mean. |

## Approximate Validation Summary

The comparison below uses 2020-2025/2025.5 rows where both modelled and observed values exist.

| Parameter | Predicted mean | Observed mean | Direction | Mean absolute percent error |
|---|---:|---:|---|---:|
| `Mo` | 12.17 | 14.78 | Low | 18% |
| `NO3` | 4.50 | 5.97 | Low | 28% |
| `Co` | 0.44 | 0.37 | Slightly high | 32% |
| `Cl` | 70.52 | 134.90 | Low | 46% |
| `Ni` | 0.53 | 1.16 | Low | 54% |
| `Ca` | 74.49 | 348.80 | Very low | 78% |
| `SO4` | 194.70 | 940.00 | Very low | 78% |
| `Cu` | 7.45 | 1.89 | Very high | 311% |
| `Zn` | 5.57 | 1.56 | Very high | 571% |
| `Cr` | 0.40 | 0.058 | Very high | 769% |
| `As` | 23.29 | 0.30 | Extremely high | 7860% |

`PO4P` has too few observed comparison points in this table to interpret confidently.

## What This Suggests

1. **Major ions are probably missing a water-source contribution.**  
   `SO4`, `Ca`, and `Cl` are underpredicted. This often points to missing or underestimated pit-water, Gruvberget, process-return, or background concentration inputs rather than only ore leaching.

2. **Several trace metals look like unit or leaching-rate problems.**  
   `As`, `Cr`, `Cu`, and `Zn` are much higher than observed. This may indicate wrong unit conversion, for example confusing `g/ton`, `kg/Mton`, `mg/l`, and `ug/l`, or using the wrong leaching column.

3. **Her changes are not applied consistently across all blocks.**  
   In the `Process water` sheet, `Cu`, `NH4`, and parts of `Cl` mostly change pit-pump volume and recurrence outputs, while many later blocks also change production, leaching loads, and pit-water concentrations. That inconsistency can make some parameters appear close and others very far off.

4. **The observed-data sheet has multiple tables.**  
   The `Decimal date` sheet contains more than one table-like area. Comparing against the wrong table can produce misleading conclusions. The observed table starting with columns like `Ca`, `Klorid`, `Sulfat`, `NO3-N`, `P_tot`, `As`, `Co`, `Cr`, `Cu`, `Mo`, `Ni`, and `Zn` is the more useful one for this comparison.

5. **The workbook is best treated as an intermediate/manual reconstruction.**  
   It is valuable for understanding Agnes's thinking and checking where her manual reconstruction agrees with observations, but it should not replace the cleaner notebook workflow unless the formulas and unit conversions are verified parameter by parameter.

## Recommended Next Step

Use `agnes_predictions.xlsx` as a diagnostic reference:

- compare Agnes's prediction line against observed data;
- compare the notebook Method 1 line against observed data;
- identify which input columns differ between Agnes's workbook and the notebook;
- focus first on `SO4`, `Ca`, `As`, `Cr`, `Cu`, and `Zn`, because those are the clearest mismatch cases.

