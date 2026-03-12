# ⚡ Solakon ONE Zero Export Blueprint (EN) - V208

This Home Assistant Blueprint implements **dynamic zero export** for the Solakon ONE inverter, based on a **PI controller (Proportional-Integral controller)** and intelligent **SOC zone logic** with optional **surplus export when the battery is full**.

The goal of this blueprint is to output PV energy directly without routing it through the battery. This prevents the "flickering" caused by the app's (charge one percent → discharge one percent → repeat) behavior, and protects the battery.

---

**IMPORTANT:** The implementation of remote control in the Solakon integration means there is no "disabled" as a remote control command — this actually disables remote control itself, meaning the default settings of the Solakon ONE or the app apply at that point.
For the intended functionality described below, a 0W schedule for 24h should be created and activated as the default, or in the latest version of the app the "Default Output Power" should be set to 0W. These methods are equivalent.

---

## 🚀 Installation

Install the blueprint directly via this button in your Home Assistant instance:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2FSolakon-ONE-Zero-Export%2Fsolakon_one_nulleinspeisung.yaml)

## 🛠️ Preparation: Creating the Required Helpers

The blueprint requires **two helpers** that you must create before installation:

### 1. Input Select Helper (Discharge Cycle Storage)

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Dropdown**
3. Name: e.g. `SOC Discharge Cycle Status`
4. Add **exactly** these two options:
   * `on`
   * `off`
5. Save (Entity ID: e.g. `input_select.soc_entladezyklus_status`)

### 2. Input Number Helper (Integral Storage for PI Controller)

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Number**
3. Name: e.g. `Solakon Integral`
4. **Important settings:**
   * Minimum: `-1000`
   * Maximum: `1000`
   * Step: `1`
   * Initial value: `0`
5. Save (Entity ID: e.g. `input_number.solakon_integral`)

### 3. Input Boolean Helper (Surplus State — REQUIRED when Zone 0 is active!)

Required to store the surplus export state **persistently across automation runs**.
Prevents the system from prematurely exiting Zone 0 during MPPT ramping.

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle**
3. Name: e.g. `Solakon Surplus Active`
4. Save (Entity ID: e.g. `input_boolean.solakon_surplus_aktiv`)

> ℹ️ **Note:** This helper is only required when surplus export is enabled. If Zone 0 is disabled, this field can be left empty in the blueprint.

### 4. Input Number Helper for Dynamic Offset (Optional)

If you want to dynamically adjust the zero-point offset at runtime, we recommend the **Solakon ONE — Dynamic Offset Blueprint** (see Dynamic Offset section). This creates and populates the required helpers automatically.

Alternatively, you can also create one or two helpers manually:

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Number**
3. Name: e.g. `Solakon Offset Zone 1`
4. **Settings:**
   * Minimum: `0`
   * Maximum: `500`
   * Step: `1`
   * Initial value: `30` (or your desired default offset)
5. Save (Entity ID: e.g. `input_number.solakon_offset_zone1`)
6. Optional: Repeat for Zone 2 (`input_number.solakon_offset_zone2`)


---

## 🧠 Core Functionality

### 1. PI Controller (Proportional-Integral Controller)

The blueprint uses a **PI controller** for precise zero export:

* **P-part (Proportional):**
  - Reacts immediately to current deviations
  - Configurable aggressiveness via the **P-factor** (e.g. 1.5)
  - Fast response to load changes

* **I-part (Integral):**
  - Accumulates deviations over time
  - Eliminates **steady-state errors**
  - Configurable speed via the **I-factor** (e.g. 0.05)
  - **Anti-Windup:** Limited to ±1000 points
  - **Automatic reset:** Reset to 0 at every zone change
  - **Tolerance decay:** 5% decay per cycle as long as the grid error is within tolerance and `|Integral| > 10`. Prevents creeping integral windup during stable control.

* **Measurement principle:**
  - Based on the **grid power sensor** (e.g. Shelly 3EM)
  - **Positive values = import**, **Negative values = export**
  - No delay — immediate response to sensor changes

* **Error calculation with dynamic limiting:**
  - **Zone 1:** Error = Min(available capacity, Grid Power − Offset₁)
  - **Zone 2:** Error = Min(available capacity, Grid Power − Offset₂, PV capacity)

* **Power limiting:**
  - Upper hard limit (e.g. 800 W) in Zone 1 and Zone 0
  - Dynamic PV-based limit in Zone 2
  - Lower limit: fixed 0 W

---

### 2. 🔋 SOC Zone Logic

The control is divided into up to four operating modes based on the current SOC:

| Zone | SOC Range / Condition | Mode | Max. Discharge Current | Control Target | Notes |
|:-----|:----------------------|:-----|:----------------------|:--------------|:------|
| **Zone 0: Surplus Export** | SOC ≥ Export Threshold AND Grid balanced | `INV Discharge (PV Priority)` | 2 A (Stability buffer) | Hard Limit (max. W) | **Optionally activatable.** Predominantly PV current to grid. Two-state logic for entry and stay (see below). |
| **Zone 1: Aggressive Discharge** | SOC > Zone-1-Threshold | `INV Discharge (PV Priority)` | 40 A | 0W + Offset 1 | Runs **continuously until SOC ≤ Zone-3-Threshold** (no yo-yo effect). Also active at night. Hard Limit. |
| **Zone 2: Battery-friendly** | Zone-3-Threshold < SOC ≤ Zone-1-Threshold | `INV Discharge (PV Priority)` | **0 A** | 0W + Offset 2 | Dynamic limit: **Max(0, PV − Reserve)**. Optional: Night shutdown possible. |
| **Zone 3: Safety Stop** | SOC ≤ Zone-3-Threshold | `Disabled` | 0 A | — | Output = 0 W. Full battery protection. |

**Important:**
- Zone 1 is **activated once** when the Zone-1-Threshold is exceeded and then runs through until the Zone-3-Threshold
- Zone 2 only starts from Zone 3 (when charging from below to above the Zone-3-Threshold)
- This prevents constant switching back and forth between zones
- **Max. discharge current** is controlled automatically — no manual setting needed

#### 🔄 Recovery Mechanism

If the inverter mode is externally reset (e.g. by an integration restart or manual change) while the discharge cycle is still active (`Cycle = on`), the blueprint automatically detects this state and reactivates the mode via the pulse sequence (10s → 3599s) — without a zone change or integral reset. Prerequisite: SOC is still above the Zone-3-Threshold.

---

### 3. ☀️ Surplus Export (Optional — Zone 0)

Surplus export can be optionally activated and allows the feed-in of real PV surplus to the grid when the battery is full. The transition into and out of Zone 0 follows a **two-state logic** that prevents unstable switching back and forth during variable cloud cover:

* **Activation:** Via the "Enable Surplus Export" parameter

* **Entry condition** (switch from zero export → Zone 0):
  - SOC ≥ configured export threshold
  - Grid balanced: Grid Power ≤ Offset + Tolerance
  - PV active: PV power ≥ night threshold *(protects against entry in darkness)*
  - PV surplus present: PV > current output power + grid power

* **Stay condition:**
  - surplus_hold_active (60s hold after entry protects against premature exit during MPPT ramping)
  - OR (SOC ≥ Export Threshold AND (PV > current output power + grid power OR PV ≥ Hard Limit))

* **Behavior in Zone 0:**
  - Max. discharge current is set to 0 A (no battery discharge)
  - AC output limit is set to the hard limit (e.g. 800 W)
  - Predominantly PV current is fed into the grid (2 A stability buffer allows minimal battery contribution)
  - Output is only set if it is not already at the hard limit (prevents Modbus spam)

* **Return to zero export:** As soon as one of the stay conditions is no longer met, the system automatically returns to normal PI control

* **Disabled:** The system behaves like classic zero export — no active grid feed-in

---

### 4. 🌙 Night Shutdown (Optional)

Night shutdown can be optionally activated and affects **Zone 2 only**:

* **Activation:** Via the "Enable Night Shutdown" parameter
* **Threshold:** PV power below configured value (e.g. 10 W)
* **Behavior:**
  - **Zone 0:** Not affected (requires full SOC)
  - **Zone 1:** Continues to run at night (high SOC → aggressive discharge desired)
  - **Zone 2:** Set to `Disabled` when PV < threshold (integral is reset)
  - **Zone 3:** Already disabled anyway
* **Useful for:** Battery protection at 0 PV when no base load is present

---

### 5. ⏱️ Remote Timeout Reset and Mode Change Sequence

To ensure communication stability with the Solakon ONE:

1. **Continuous timeout reset:**
   - The remote timeout is automatically reset to 3599s
   - Trigger: Countdown falls below 120s

2. **Forced reset on mode change:**
   - Two-stage pulse sequence: 10s → 3599s (with 1s delay)
   - Ensures safe mode takeover
   - Applies at Zone-1-Start, Zone-2-Start, and Recovery

---

## 📊 Input Variables and Configuration

### 🔌 Required Entities

| Category | Variable | Default Entity | Description |
|:---------|:---------|:--------------|:------------|
| **External** | Grid power sensor | *(no default)* | E.g. Shelly 3EM. **Positive = import, Negative = export** |
| **Solakon** | Solar power | `sensor.solakon_one_pv_leistung` | Current PV generation in watts |
| **Solakon** | Actual output power | `sensor.solakon_one_leistung` | Current AC output power of the inverter in watts |
| **Solakon** | Battery state of charge (SOC) | `sensor.solakon_one_batterie_ladestand` | State of charge in % |
| **Solakon** | Remote timeout countdown | `sensor.solakon_one_fernsteuerung_zeituberschreitung` | Remaining countdown |
| **Solakon** | Output power controller | `number.solakon_one_fernsteuerung_leistung` | Sets power setpoint |
| **Solakon** | Max. discharge current | `number.solakon_one_maximaler_entladestrom` | Sets discharge current limit |
| **Solakon** | Mode reset timer | `number.solakon_one_fernsteuerung_zeituberschreitung` | Sets/resets timeout (max. 3599s) |
| **Solakon** | Operating mode select | `select.solakon_one_modus_fernsteuern` | Switches operating mode |
| **Helper** | Discharge cycle storage | `input_select.soc_entladezyklus_status` | Input Select: `on`/`off` |
| **Helper** | Integral storage | `input_number.solakon_integral` | Input Number: -1000 to 1000 |
| **Helper** | Surplus state storage | `input_boolean.solakon_surplus_aktiv` | Only required when surplus export is enabled (Zone 0). If Zone 0 disabled: n/a |

---

### 🎚️ Control Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **P-Factor** | 1.5 | 0.1 | 5.0 | Proportional gain. Higher = more aggressive, faster. |
| **I-Factor** | 0.05 | 0.01 | 0.2 | Integral gain. Higher = faster error correction, but more unstable. |
| **Tolerance range** | 25 W | 0 | 200 W | Dead band around control target. No correction within this zone. |

**Tuning tips:**
- **Too slow?** → Increase P-factor (e.g. 2.0) or increase I-factor (e.g. 0.08)
- **Oscillating/overshooting?** → Reduce P-factor (e.g. 1.0) or reduce I-factor (e.g. 0.03)
- **Permanent micro-corrections?** → Increase tolerance range (e.g. 35 W)
- **Integral climbing despite stable control?** → Tolerance decay applies automatically (5% decay per cycle when `|Integral| > 10`)

---

### 🔋 SOC Zone Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Upper threshold. Exceeding activates aggressive discharge. |
| **Zone 3 Stop** | 20 % | 1 % | 49 % | Lower threshold. Falling below stops discharge completely. |
| **Max. discharge current Zone 1** | 40 A | 0 A | 40 A | Discharge current in Zone 1. Zone 2 automatically uses 0 A. |

**Important:** Zone-1-Threshold must be **greater** than Zone-3-Threshold! Blueprint validates this on start.

---

### ☀️ Surplus Export (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable Surplus Export** | false | — | — | Switch to activate Zone 0. |
| **SOC Threshold Surplus** | 90 % | 50 % | 99 % | From this SOC, PV surplus is fed into the grid. |

---

### ⚙️ Zone 1/2 - Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Zero-Point Offset 1 (Static)** | 30 W | 0 | 100 W | Static fallback value for Zone 1. |
| **Zero-Point Offset 1 (Dynamic)** | *(empty)* | — | — | Optional `input_number` entity. Overrides static value when configured. |
| **Zero-Point Offset 2 (Static)** | 30 W | 0 | 100 W | Static fallback value for Zone 2. |
| **Zero-Point Offset 2 (Dynamic)** | *(empty)* | — | — | Optional `input_number` entity. Overrides static value when configured. |
| **PV Charge Reserve** | 50 W | 0 | 1000 W | PV power reserved for charging in Zone 2. Dynamic limit: Max(0, PV − Reserve). |
---

## 🎯 Dynamic Offset Blueprint (Recommended)

The **dynamic offset** allows the zero-point offset to **automatically adapt to the current grid volatility** — without needing to change the zero export blueprint.

A dedicated blueprint is available for this:

### 👉 [Solakon ONE — Dynamic Offset (Spike Filter & Volatility Controller)](https://github.com/D4nte85/Solakon-one-dynamic-offset-blueprint)

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)

**What it does:**
- Filters out grid spikes (spike filter with configurable confirmation time)
- Analyzes grid volatility over the last 60 seconds (standard deviation)
- Automatically increases the safety buffer during unstable grid conditions (e.g. cycling compressors, washing machines)
- Writes the calculated value directly into the `input_number` helpers for Zone 1 & Zone 2

**Typical offset by grid condition:**

| Grid Condition | StdDev | Calculated Offset |
|:--------------|:------:|:-----------------:|
| Very calm | 5 W | 30 W *(Minimum)* |
| Normal | 30 W | 53 W |
| Unstable | 80 W | 128 W |
| Very unstable | 160 W | 228 W |
| Extreme | 250 W+ | 250 W *(Maximum)* |

**Interaction with this blueprint:**
1. Dynamic Offset Blueprint calculates the optimal offset and writes it to `input_number.solakon_offset_zone1` / `_zone2`
2. This blueprint reads the value from these entities via the "Dynamic Zero-Point Offset" fields
3. The offset adjusts fully automatically without manual intervention

> ⚠️ **Backwards compatible:** If no dynamic entity is selected, this blueprint works as before with the configured static offset value.
---

**Explanation of PV Charge Reserve:**
- At 300W PV generation and 50W reserve → Max. output in Zone 2: 250W
- Ensures that even at trickle charge, something is always left for the battery
- Compensates for conversion losses

> ⚠️ **Backwards compatible:** If no dynamic entity is selected, this blueprint works as before with the configured static offset value.
---

### 🔒 Safety Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Max. output power** | 800 W | 0 | 1200 W | Absolute upper limit (Hard Limit) in Zone 0 and Zone 1. |

---

### 🌙 Night Shutdown (Optional)

| Parameter | Default | Description |
|:----------|:--------|:------------|
| **Enable Night Shutdown** | false | On/off switch for the feature |
| **PV Threshold for "Night"** | 10 W | Below this PV power it is considered night |

**Note:** Only Zone 2 is disabled at night. Zone 1 continues to run!

---

## 🔧 Recommended Settings

### For conservative battery use (longevity):
```yaml
SOC Zone 1 Start: 60%
SOC Zone 3 Stop: 30%
Zero-Point Offset: 50W
PV Charge Reserve: 100W
P-Factor: 1.5
I-Factor: 0.05
Tolerance range: 30W
Surplus Export: false
```

### For maximum self-consumption optimization:
```yaml
SOC Zone 1 Start: 40%
SOC Zone 3 Stop: 15%
Zero-Point Offset: 20W
PV Charge Reserve: 30W
P-Factor: 2.0
I-Factor: 0.08
Tolerance range: 20W
Surplus Export: true
SOC Threshold Surplus: 95%
```

### For balanced operation (standard):
```yaml
SOC Zone 1 Start: 50%
SOC Zone 3 Stop: 20%
Zero-Point Offset: 30W
PV Charge Reserve: 50W
P-Factor: 1.5
I-Factor: 0.05
Max. Output Power: 800W
Tolerance range: 25W
Max. Discharge Current Zone 1: 40A
Surplus Export: false
```

---

## 📈 How It Works in Detail

### Example flow over a day:

**Morning (06:00 - SOC: 25%)**
- Zone 2 active (Zone-3-Threshold < SOC ≤ Zone-1-Threshold)
- PV rising slowly
- Max. discharge current: **0A** (battery-friendly)
- Control target: Offset 2 (slight grid import)
- Dynamic limit: Max(0, PV - 50W)
- Battery is being charged

**Midday (12:00 - SOC: 55%)**
- Zone 1 activated (SOC > Zone-1-Threshold)
- Max. discharge current: **40A** (automatically set)
- Control target: Offset 1 (near zero export)
- Hard Limit: 800W
- **Remains active even if SOC drops back below the Zone-1-Threshold!**

**Midday with full battery (SOC: 100% + Surplus Export enabled)**
- Entry condition: SOC ≥ Export Threshold, Grid ≤ Offset + Tolerance, PV active
- Zone 0 active → Max. discharge current: **2A** (stability buffer for the inverter)
- AC limit set to Hard Limit (800W) — pure PV current to grid
- Short cloud: System stays in Zone 0 during the 60s hold time; then stays as long as PV > output + grid OR PV ≥ Hard Limit
- With rising house consumption: automatic return to Zone 1

**Evening (20:00 - SOC: 22%)**
- Zone 1 still active (runs until Zone-3-Threshold)
- Max. discharge current: still 40A

**Night (22:00 - SOC: 19%)**
- Zone 3 activated (SOC ≤ Zone-3-Threshold)
- Mode: `Disabled`
- Output: 0W — battery protected

**Next morning — Cycle starts again**

---

## 🛑 Important Error Messages

The blueprint validates the configuration on start. Errors are shown in the **System Log**:

| Message | Cause | Solution |
|:--------|:------|:---------|
| **Upper SOC threshold must be greater than lower** | Thresholds incorrectly configured | Zone-1-Threshold (e.g. 50%) > Zone-3-Threshold (e.g. 20%) |
| **SOC sensor is UNKNOWN/UNAVAILABLE** | Solakon integration offline | Check connection to the inverter |
| **Timeout Countdown Sensor is UNKNOWN/UNAVAILABLE** | Sensor not available | Check Solakon integration |
| **Mode selector is UNKNOWN/UNAVAILABLE** | Select entity missing | Check Solakon integration |
| **Output power controller has no min attributes** | Number entity faulty | Check Solakon integration |

---

## ⚙️ Technical Details

### PI Controller Implementation

**Error calculation (per zone):**
```
Zone 1: error = Min(available_capacity, grid_power - offset_1)
Zone 2: error = Min(available_capacity, grid_power - offset_2, pv_capacity)
```

**Integral logic with tolerance decay:**
```
If |grid_error| > tolerance:
  integral_new = Clamp(integral_old + error, -1000, 1000)
Else if |integral_old| > 10:
  integral_new = integral_old * 0.95  # 5% decay per cycle
Else:
  integral_new = integral_old         # No change
```

**PI correction:**
```
correction = error * P_Factor + integral_new * I_Factor
new_power = current_power + correction
```

**Final power (zone-dependent limiting):**
```
Zone 0: final_power = hard_limit                       (Surplus export)
Zone 1: final_power = Min(hard_limit, new_power)
Zone 2: final_power = Min(Max(0, PV - reserve), new_power)
```

**Output update condition:**
```
Surplus mode:  Only set if current_output < hard_limit  (prevent Modbus spam)
Normal mode:   Only set if |grid_error| > tolerance
```

---

### Surplus Export State Machine

```
Entry condition:  SOC >= export_limit
              AND grid_power <= (target_offset + tolerance)
              AND solar_power >= night_threshold
              AND solar_power > (current_active_power + grid_power)

Stay condition:   surplus_hold_active (60s after entry)
              OR  (SOC >= export_limit
                  AND (solar_power > (current_active_power + grid_power)
                       OR solar_power >= hard_limit))

Exit condition:   Stay condition no longer met
```

---

### Recovery Mechanism

```
Condition:  Cycle = on
        AND Mode ≠ INV Discharge PV Priority
        AND SOC > Zone-3-Threshold

Action:     Pulse sequence 10s → (1s pause) → 3599s
            Mode → INV Discharge PV Priority
            (no integral reset, no zone change)
```

---

### Automatic Discharge Current Control

| Zone | Discharge Current | Condition |
|:-----|:-----------------|:----------|
| Zone 0 (Surplus) | 2 A (Stability buffer) | Always, while Zone 0 is active |
| Zone 1 (Aggressive) | Configured maximum value | Only when current value differs |
| Zone 2 (Battery-friendly) | 0 A | Only when current value differs |

Changes are only made when the current value differs from the setpoint (prevents unnecessary API calls).

---

## ⚠️ Important Notes

1. **Create helpers before installation:** Both helpers must exist before the blueprint is configured
2. **Solakon ONE integration:** Must be fully set up
3. **Grid power sensor:** Correct polarity (positive = import, negative = export)
4. **PI controller tuning:** Default values are conservative. Reduce I-factor if behavior is unstable.
5. **Integral helper:** Managed automatically — do not change manually!
6. **Queued mode:** Triggers are processed sequentially
7. **No trigger delay:** The blueprint reacts immediately to sensor changes
8. **Configurable wait time:** After each power change, the blueprint waits 0–30 seconds
9. **Discharge current automation:** Max. discharge current is fully automatically controlled — no manual setting needed
10. **Tolerance decay:** Automatically prevents integral windup — 5% decay per cycle when `|Integral| > 10` and grid error within tolerance
11. **Surplus export:** Persistent `input_boolean` stores the Zone-0-state across automation runs. A 60-second hold time after entry prevents premature exit during MPPT ramping. Afterwards: stay as long as SOC ≥ Threshold AND (PV > current output + grid power OR PV ≥ Hard Limit). The helper is only required when Zone 0 is enabled.
12. **Recovery:** Mode loss during an active cycle is automatically detected and corrected — no manual intervention needed

---

## 🔄 Trigger Overview

The blueprint responds to the following events:

| Trigger | ID | Description |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Immediate PI control on grid power change |
| Solar Power Change | `solar_power_change` | Immediate PI control on PV power change |
| SOC High | `soc_high` | Zone 1 start (SOC > Zone-1-Threshold) |
| SOC Low | `soc_low` | Zone 3 start (SOC ≤ Zone-3-Threshold) |
| Mode Change | `mode_change` | Responds to external mode changes, triggers recovery if applicable |

**All triggers** execute the complete logic — no separate handling needed.
