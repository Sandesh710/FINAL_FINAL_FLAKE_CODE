# DAT File Conversion Formulas Reference

## Input Data Format

**File**: `Potsdam80-96.dat`
**Format**: 6 columns, 6210 rows (daily data, 1980-1996)

```
Column 1: Time index (sequential)
Column 2: Solar radiation [W/m²]
Column 3: Air temperature [°C]
Column 4: Humidity [mb]
Column 5: Wind speed [m/s]
Column 6: Cloudiness [0-1]
```

---

## Physical Constants Used

| Constant | Symbol | Value | Units |
|----------|--------|-------|-------|
| Stefan-Boltzmann | σ | 5.67×10⁻⁸ | W/(m²·K⁴) |
| Air pressure | P_air | 101325 | Pa |
| Molecular weight ratio | - | 0.622 | - |
| Celsius to Kelvin | - | 273.15 | K |

---

## Conversion Formulas

### 1. Solar Radiation
**DAT File**: `I_solar` [W/m²]
**Conversion**: ✅ **NONE**
**Model Variable**: `I_atm_in` [W/m²]

```python
I_atm_in = I_solar
```

---

### 2. Air Temperature
**DAT File**: `T_air_C` [°C]
**Conversion**: Celsius → Kelvin
**Model Variable**: `T_a_in` [K]

```python
T_a_in = T_air_C + 273.15
```

**Example**:
```
-0.14°C → 273.01 K
```

---

### 3. Wind Speed
**DAT File**: `U_wind` [m/s]
**Conversion**: ✅ **NONE**
**Model Variable**: `U_a_in` [m/s]

```python
U_a_in = U_wind
```

---

### 4. Humidity (TWO-STEP CONVERSION)

**DAT File**: `humidity_mb` [mb]
**Conversion**: Millibar → Pascal → Specific Humidity
**Model Variables**: `e` [Pa] and `q_a_in` [kg/kg]

#### Step 1: Millibar to Pascal
```python
e = humidity_mb × 100.0
```

**Example**:
```
5.70 mb → 570 Pa
```

#### Step 2: Vapor Pressure to Specific Humidity
```python
q_a_in = 0.622 × e / (P_air - 0.378 × e)
```

Where:
- `0.622` = ratio of molecular weights (H₂O / dry air)
- `P_air` = 101325 Pa (atmospheric pressure)
- `0.378` = empirical coefficient

**Example**:
```
570 Pa → 0.003505 kg/kg
```

**Physical Meaning**: Specific humidity = mass of water vapor / mass of moist air

---

### 5. Cloudiness (TWO-STEP CONVERSION)

**DAT File**: `cloud` [0-1] (0=clear sky, 1=overcast)
**Conversion**: Cloud fraction → Emissivity → Longwave Radiation
**Model Variables**: `emissivity` [-] and `Q_atm_lw_in` [W/m²]

#### Step 1: Cloud to Emissivity
```python
emissivity = 0.7 + 0.3 × cloud
```

**Examples**:
```
cloud = 0.0 (clear sky) → emissivity = 0.7
cloud = 0.5 (partly cloudy) → emissivity = 0.85
cloud = 1.0 (overcast) → emissivity = 1.0
```

#### Step 2: Emissivity to Downward Longwave Radiation
```python
Q_atm_lw_in = emissivity × σ × T_a_in⁴
```

Where:
- `σ` = 5.67×10⁻⁸ W/(m²·K⁴) (Stefan-Boltzmann constant)
- `T_a_in` in Kelvin (from conversion #2 above)

**Example**:
```
emissivity = 0.90, T_a_in = 273 K
→ Q_atm_lw_in = 0.90 × 5.67×10⁻⁸ × 273⁴ = 283 W/m²
```

**Physical Meaning**: Downward longwave (thermal) radiation from atmosphere to lake surface. Clouds increase atmospheric emissivity, leading to more downward radiation.

---

## Complete Conversion Flow

```
DAT FILE                    CONVERSION                          MODEL INPUT
─────────────────────────── ────────────────────────────────── ──────────────────
I_solar [W/m²]           →  (no conversion)                  → I_atm_in [W/m²]

T_air_C [°C]             →  +273.15                          → T_a_in [K]

U_wind [m/s]             →  (no conversion)                  → U_a_in [m/s]

humidity_mb [mb]         →  ×100                             → e [Pa]
                         →  0.622×e/(P-0.378×e)              → q_a_in [kg/kg]

cloud [0-1]              →  0.7 + 0.3×cloud                  → emissivity [-]
                         →  emissivity×σ×T_a_in⁴             → Q_atm_lw_in [W/m²]
```

---

## Example: First Timestep

**Input (from DAT file)**:
```
Time:       1
Solar:      17.01 W/m²
T_air:      -0.14°C
Humidity:   5.70 mb
Wind:       3.84 m/s
Cloud:      0.66
```

**Conversions**:
```python
# Solar (no change)
I_atm_in = 17.01 W/m²

# Temperature
T_a_in = -0.14 + 273.15 = 273.01 K

# Wind (no change)
U_a_in = 3.84 m/s

# Humidity (2 steps)
e = 5.70 × 100.0 = 570 Pa
q_a_in = 0.622 × 570 / (101325 - 0.378 × 570) = 0.003505 kg/kg

# Cloudiness (2 steps)
emissivity = 0.7 + 0.3 × 0.66 = 0.898
Q_atm_lw_in = 0.898 × 5.67×10⁻⁸ × 273.01⁴ = 282.92 W/m²
```

**Model Inputs (after conversion)**:
```
I_atm_in:     17.01 W/m²
T_a_in:       273.01 K
U_a_in:       3.84 m/s
e:            570 Pa
q_a_in:       0.003505 kg/kg
Q_atm_lw_in:  282.92 W/m²
```

---

## Summary Table

| DAT Column | DAT Unit | Conversion Steps | Model Variable | Model Unit |
|------------|----------|------------------|----------------|------------|
| Column 2 | W/m² | None | I_atm_in | W/m² |
| Column 3 | °C | +273.15 | T_a_in | K |
| Column 5 | m/s | None | U_a_in | m/s |
| Column 4 | mb | ×100 → formula | e, q_a_in | Pa, kg/kg |
| Column 6 | 0-1 | formula → formula | emissivity, Q_atm_lw_in | -, W/m² |

---

## Notes

1. **Air Pressure**: Constant at 101325 Pa (sea level) because it's not in the DAT file
2. **Humidity Formula**: Based on ideal gas law and molecular weight ratios
3. **Cloudiness Formula**: Empirical relationship for atmospheric emissivity
4. **Longwave Formula**: Stefan-Boltzmann law for blackbody radiation

---

## Code Location

These conversions are performed in **Cell 26** of the notebook:

```python
# Line ~146-152 (Initial conditions)
I_atm_in = I_solar[0]
T_a_in = T_air_C[0] + 273.15
U_a_in = U_wind[0]
e = hum_mb[0] * 100.0
q_a_in = 0.622 * e / (P_air - 0.378 * e)
emissivity = 0.7 + 0.3 * cloud[0]
Q_atm_lw_in = emissivity * SIGMA * T_a_in**4

# Line ~224-235 (Main loop)
I_atm_in = I_solar[k]
T_a_in = T_air_C[k] + 273.15
U_a_in = U_wind[k]
e = hum_mb[k] * 100.0
q_a_in = 0.622 * e / (P_air - 0.378 * e)
emissivity = 0.7 + 0.3 * cloud[k]
Q_atm_lw_in = emissivity * SIGMA * T_a_in**4
```

---

**Document Version**: 1.0
**Date**: December 5, 2025
**Status**: ✅ Validated against Heiligensee test case
