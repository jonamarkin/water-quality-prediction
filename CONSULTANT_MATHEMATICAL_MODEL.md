# Consultant Mathematical Model

This document extracts and explains the mathematical model used by the consultant in the original Leveaniemi process-water Excel workbook.

The main source is the `Process water` sheet in:

```text
Leveaniemi_data.xlsx
```

The consultant model is not a black-box statistical model. It is a deterministic mass-balance recurrence. Each period's concentration is calculated from:

- ore production and leaching load
- water inflows and inflow concentrations
- selected water losses
- storage volume
- previous process-water/storage concentration

The important point is:

> The consultant's final concentration column uses the GM ore assumption. GK and GL are calculated as helper/scenario leaching columns, but the final recurrence formula uses the GM process-leaching column. LK, Leveaniemi-Kiruna, was not directly included by the consultant.

The consultant's practical purpose was to check whether planned mining activity could use Leveaniemi water and process ore while staying within element-specific water-quality limits. The current thesis task is different: current operations may use Leveaniemi-Kiruna, LK, with ratios such as 50/50 or 40/60 that can vary over time.

## 1. Contaminant Blocks

The `Process water` sheet contains repeated blocks for contaminants. The current project focuses on:

| Parameter | Unit | Approximate Excel Data Start Row |
|---|---:|---:|
| Cu | `ug/l` | 43 |
| NH4 | `mg/l` | 82 |
| Cl | `mg/l` | 122 |
| Ni | `ug/l` | 161 |

Each block has rows for half-year periods:

- `.0` rows represent annual/winter/non-processing periods.
- `.5` rows represent summer/processing-season periods.

For example:

```text
2015.0
2015.5
2016.0
2016.5
```

The formulas are copied through the block, but Excel sometimes stores them as "shared formulas", so only the first visible formula appears in the XML while later rows store calculated values.

## 2. Main Columns in the Process-Water Blocks

The most important columns are:

| Excel Column | Meaning in This Explanation |
|---|---|
| `A` | Year / half-year period |
| `B` | Tailings load or tailings indicator, depending on block |
| `C` | Base roll-leach rate, generally the GM/Mertainen-Gruvberget rate |
| `D` | Production volume for GM |
| `E` | Process leaching for GM |
| `F` | Production volume for GK |
| `G` | Process leaching for GK |
| `H` | Process leaching for GL |
| `I` | Leveaniemi pit-pump volume |
| `J` | Leveaniemi pit-pump concentration |
| `K` | Ditch SW flow |
| `L` | Ditch SW concentration |
| `M` | Ditch SE flow |
| `N` | Ditch SE concentration |
| `O` | Gruvberget flow |
| `P` | Gruvberget concentration |
| `Q` | Surface-water flow |
| `R` | Surface-water concentration, usually zero in the formula context |
| `S` | Process-loss flow |
| `T` | Process-loss concentration |
| `AE` | Total storage volume |
| `AF` | Storage concentration used as recurrence state |
| `AH` | Total gains volume |
| `AI` | Consultant modelled concentration, final GM output |

Columns `U` to `AD` track additional internal flows, discharge, tailings storage, and leakage. They are visible in the sheet, but the core `AI` recurrence formula mainly uses the inflows `I`, `K`, `M`, `O`, `Q`, process loss `S`, storage `AE`, and state concentration `AF`.

## 3. Ore-Leaching Submodel

Before calculating the process-water concentration, the workbook calculates ore-related contaminant loads for different ore combinations.

Let:

- `r_GM` = base GM leaching rate from column `C`
- `r_K` = Kiruna leaching rate from the block header, for example `G$41`, `G$80`, `G$120`, or `G$159`
- `r_L` = Leveaniemi leaching rate from the block header, for example `H$41`, `H$80`, `H$120`, or `H$159`
- `D_t` = GM production volume in period `t`, column `D`
- `F_t` = GK production volume in period `t`, column `F`

### 3.1 GM Process Leaching

The GM process-leaching load is:

```text
E_t = r_GM,t * D_t
```

Excel example from the Cu block:

```text
E43 = C43 * D43
```

This is the process-leaching term that the consultant's final `AI` recurrence uses.

### 3.2 GK Process Leaching

The workbook also calculates a GK helper/scenario leaching value:

```text
G_t = r_GM,t * 0.4 * D_t + r_K * 0.6 * F_t
```

Excel example:

```text
G43 = (C43*0.4*D43 + G$41*0.6*F43)
```

Interpretation:

- 40% contribution from the GM/base leaching rate
- 60% contribution from Kiruna
- production volumes are taken from `D` and `F`

This fixed 40/60-style structure belongs to the consultant workbook. It should not be assumed to describe every current operating LK mix unless confirmed.

### 3.3 GL Process Leaching

The workbook also calculates a GL helper/scenario leaching value:

```text
H_t = (r_GM,t * 0.4 + r_L * 0.6) * D_t
```

Excel example:

```text
H43 = (C43*0.4 + H$41*0.6) * D43
```

Interpretation:

- 40% contribution from the GM/base leaching rate
- 60% contribution from Leveaniemi

Again, this is the consultant workbook's helper-scenario structure. The current operational LK mix may use different ratios over time.

### 3.4 Important: Consultant Final Output Uses GM

Although the workbook calculates `G_t` and `H_t`, the final recurrence formula in column `AI` uses `E_t`, the GM process-leaching load.

This means:

> The consultant's final process-water output was based on GM, not on GK, GL, or LK.

This is visible in the formulas because the recurrence includes `E`, not `G` or `H`.

## 4. Water-Balance Terms

For each period `t`, define the inflow/gain volume:

```text
Gains_t = I_t + K_t + M_t + O_t + Q_t
```

In the workbook, this is column `AH`.

Excel example:

```text
AH43 = I43 + K43 + M43 + O43 + Q43
```

The concentration-carrying load terms are:

```text
Pit-pump load      = I_t * J_t
Ditch SW load      = K_t * L_t
Ditch SE load      = M_t * N_t
Gruvberget load    = O_t * P_t
Process-loss load  = S_t * T_t
```

Surface water `Q_t` is included in the gains volume. Its concentration term `Q_t * R_t` is not explicitly present in the main Excel recurrence formulas, likely because `R_t` is zero in these blocks.

The effective water volume used for the incoming concentration calculation is:

```text
EffectiveVolume_t = I_t + K_t + M_t + O_t + Q_t + I_t - S_t
```

This can also be written as:

```text
EffectiveVolume_t = Gains_t + I_t - S_t
```

The extra `I_t` appears directly in the consultant's Excel formula. It means the pit-pump volume is counted once inside `Gains_t` and then again in the effective-volume denominator.

## 5. Core Recurrence Formula

The consultant calculates a new process-water/storage concentration by combining:

1. the concentration of the new incoming/process water for the period
2. the concentration already stored from the previous period

Let:

- `C_prev,t` = previous storage concentration, column `AF`
- `V_store,t` = storage volume, column `AE`
- `C_new,t` = new modelled concentration, column `AI`
- `OreLoad_t` = ore/tailings leaching load for the parameter
- `WaterLoad_t` = concentration load from external waters

### 5.1 Load Term

The general load term is:

```text
TotalLoad_t =
    OreLoad_t
  + I_t * J_t
  + K_t * L_t
  + M_t * N_t
  + O_t * P_t
  - S_t * T_t
```

Surface water load `Q_t * R_t` is omitted in the actual formulas because the surface-water concentration is effectively zero in the relevant rows.

### 5.2 Incoming Concentration

The incoming/process concentration for the period is:

```text
C_in,t = TotalLoad_t / EffectiveVolume_t
```

where:

```text
EffectiveVolume_t = I_t + K_t + M_t + O_t + Q_t + I_t - S_t
```

### 5.3 Mixing With Existing Storage

The new concentration is then:

```text
C_new,t = (C_in,t * Gains_t + V_store,t * C_prev,t) / (Gains_t + V_store,t)
```

where:

```text
Gains_t = I_t + K_t + M_t + O_t + Q_t
```

This is a weighted average between:

- the new period's incoming/process-water concentration
- the previous concentration already stored in the process-water system

The weights are:

- `Gains_t` for the new incoming water
- `V_store,t` for the already stored water

## 6. Full Formula in One Line

For a normal GM row, the consultant recurrence can be written as:

```text
C_new,t =
(
  (
    OreLoad_t
    + I_t*J_t
    + K_t*L_t
    + M_t*N_t
    + O_t*P_t
    - S_t*T_t
  )
  /
  (I_t + K_t + M_t + O_t + Q_t + I_t - S_t)
  *
  (I_t + K_t + M_t + O_t + Q_t)
  + AE_t*AF_t
)
/
(
  (I_t + K_t + M_t + O_t + Q_t) + AE_t
)
```

This is the mathematical form of the repeated Excel formulas in column `AI`.

## 7. Example Excel Formula

For Cu, one of the key formulas is:

```text
AI45 =
((B44+E44+I44*J44+K44*L44+M44*N44+O44*P44-S44*T44)
 /(I44+K44+M44+O44+Q44+I44-S44)
 *AH44
 +AE44*AF44)
 /(AH44+AE44)
```

This corresponds to:

```text
new concentration =
(
  incoming concentration * total gains
  + stored water mass
)
/
(
  total gains + storage volume
)
```

## 8. Parameter-Specific OreLoad Differences

The recurrence structure is the same across parameters, but the workbook uses slightly different unit conversions.

### 8.1 Cu

For Cu, the ore/tailings term is generally:

```text
OreLoad_t = B_t + E_t
```

where:

- `B_t` is a tailings-related term
- `E_t` is GM process leaching

### 8.2 Ni

For Ni, the structure is similar to Cu:

```text
OreLoad_t = B_t + E_t
```

### 8.3 NH4

For NH4, the workbook divides process leaching by `1000`:

```text
OreLoad_t = E_t / 1000
```

Example:

```text
E83/1000
```

This is a unit conversion inside the consultant workbook.

### 8.4 Cl

For Cl, the workbook also divides process leaching by `1000`:

```text
OreLoad_t = E_t / 1000
```

Some Cl rows also include a tailings term divided by `1000`:

```text
OreLoad_t = B_t/1000 + E_t/1000
```

This is because the chloride block has an extra `mg/kg` column and larger leaching values.

## 9. Time Indexing and Recurrence State

The conceptual model is:

```text
C_next = f(period inputs, C_previous)
```

In the Excel sheet, the exact row placement differs slightly by contaminant block:

- Cu, Cl, and Ni often place the calculated result one row after the period inputs.
- NH4 starts with a same-row calculation, then carries the result forward.

However, the mathematical idea is the same:

1. Start with an initial storage concentration.
2. Use the current period's flows and leaching to calculate a new concentration.
3. Carry that new concentration forward as the storage concentration for the next period.

Column `AF` is the recurrence state. Column `AI` is the calculated output.

## 10. What the Consultant Did Not Do

Based on the extracted formulas:

### 10.1 He Did Not Use LK

LK, Leveaniemi-Kiruna, does not appear as its own final process-water scenario in the consultant recurrence.

The workbook contains helper calculations for:

- GM
- GK
- GL

but not LK.

### 10.2 He Did Not Use GK/GL in the Final AI Recurrence

Even though the workbook calculates GK and GL process-leaching columns, the final recurrence formula uses column `E`, the GM process-leaching value.

So the consultant's final process-water prediction is GM-based.

### 10.3 He Did Not Directly Calibrate to Daily Monitoring Data in This Formula

The recurrence is based on workbook assumptions and water-balance inputs. It is not a statistical fit to daily monitoring observations.

## 11. Pseudocode Version

The consultant model can be represented as:

```python
storage_conc = initial_storage_concentration

for each period:
    gm_process_leach = gm_leach_rate * gm_production

    gains = pit_pump_volume + ditch_sw_flow + ditch_se_flow + gruvberget_flow + surface_water_flow
    effective_volume = gains + pit_pump_volume - process_loss_flow

    total_load = (
        ore_load
        + pit_pump_volume * pit_pump_conc
        + ditch_sw_flow * ditch_sw_conc
        + ditch_se_flow * ditch_se_conc
        + gruvberget_flow * gruvberget_conc
        - process_loss_flow * process_loss_conc
    )

    incoming_conc = total_load / effective_volume

    new_storage_conc = (
        incoming_conc * gains
        + storage_volume * storage_conc
    ) / (gains + storage_volume)

    storage_conc = new_storage_conc
```

The parameter-specific part is the `ore_load` calculation.

## 12. How This Relates to the Current Python Notebook

The current notebook does not manually rewrite every Excel formula as the final prediction engine. Instead, it trains a Random Forest surrogate to learn the relationship between:

- production/leaching inputs
- pit-pump water
- water-balance flows
- storage volume
- previous storage concentration
- consultant output concentration

The reason this works is that the consultant model is deterministic and repeated across rows. If the machine-learning model has good cross-validated `R2`, it means the model has learned the consultant's recurrence behaviour well.

The notebook also preserves the recurrence idea by feeding each predicted concentration forward as the next period's storage concentration.

## 13. Thesis-Ready Explanation

A thesis explanation could say:

> The consultant model is a deterministic process-water mass-balance recurrence. For each half-year period, the model first calculates contaminant load from ore leaching and external water sources. This load is divided by an effective process-water volume to estimate the incoming concentration for that period. The incoming concentration is then mixed with the previous stored process-water concentration using total inflow volume and storage volume as weights. The resulting concentration is carried forward as the recurrence state for the next period. The final consultant output uses the GM process-leaching column, while GK and GL are present as helper scenario columns but are not used in the final GM recurrence. LK was not included directly by the consultant.
