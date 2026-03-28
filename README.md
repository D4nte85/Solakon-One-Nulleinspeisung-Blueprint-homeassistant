# ⚡ Solakon ONE Zero Export Blueprint (EN) - V302

This Home Assistant Blueprint implements **dynamic zero export** for the Solakon ONE inverter, based on a **PI controller (Proportional-Integral Controller)** and an intelligent **SOC zone logic** with optional **surplus export when the battery is full** and optional **AC charging from external feed-in**.

The goal of this blueprint is to deliver PV energy directly to loads, bypassing the battery wherever possible.

---

**IMPORTANT:** The implementation of remote control in the Solakon integration means there is no true "disabled" remote command — this disables remote control itself, i.e. the Solakon ONE falls back to its default settings or those from the app.
For the intended behavior, a 0W schedule active for 24h should be created and activated, or the "Default Output Power" set to 0W. Both methods are equivalent.

---

## 🚀 Installation

Install the Blueprint directly via this button in your Home Assistant instance:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2FSolakon-ONE-Zero-Export%2Fsolakon_one_zeroexport.yaml)

The accompanying **PI Controller Script Blueprint** must also be imported:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2FSolakon-ONE-Zero-Export%2FPI-Controller.yaml)

---

## 🛠️ Prerequisites: Creating Required Helpers

The blueprint requires **three mandatory helpers** and a **script**, plus up to **two optional helpers**, which must be created before installation.

### 1. Input Boolean Helper (Discharge Cycle Storage) — REQUIRED

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `SOC Discharge Cycle Status`
4. Save (Entity ID: e.g. `input_boolean.soc_entladezyklus_status`)

### 2. Input Number Helper (Integral Storage for PI Controller) — REQUIRED

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Number**
3. Name: e.g. `Solakon Integral`
4. **Important settings:**
   * Minimum: `-1000`
   * Maximum: `1000`
   * Step: `1`
   * Initial value: `0`
5. Save (Entity ID: e.g. `input_number.solakon_integral`)

### 3. PI Controller Script — REQUIRED

1. Import the **PI Controller Blueprint** (link above)
2. Go to **Settings** → **Automations & Scenes** → **Scripts** → **Create Script**
3. Select the imported PI Controller Blueprint
4. Name: e.g. `PI Controller` (Script Entity ID: e.g. `script.pi_controller`)
5. Save

### 4. Input Boolean Helper (Surplus State) — ONLY if Zone 0 enabled

Persistent state storage for surplus export. Prevents flickering with variable PV.

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `Solakon Surplus Active`
4. Save (Entity ID: e.g. `input_boolean.solakon_surplus_aktiv`)

### 5. Input Boolean Helper (AC Charge State) — ONLY if AC Charging enabled

Persistent state storage for AC charging. Signals the PI script the inverted error calculation (`ac_charge_mode=true`). Also prevents the guards (at_max/at_min) from incorrectly blocking PI.

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `Solakon AC Charging Active`
4. Save (Entity ID: e.g. `input_boolean.solakon_ac_laden_aktiv`)

### 6. Input Number Helper for Dynamic Offset (Optional)

If you want to adjust the zero-point offset dynamically at runtime, the **Solakon ONE — Dynamic Offset Blueprint** is recommended. It creates and populates the required helpers automatically.

Alternatively, create manually:

1. Go to **Settings** → **Devices & Services** → **Helpers** → **Number**
2. Name: e.g. `Solakon Offset Zone 1`
3. **Settings:** Min: `0`, Max: `500`, Step: `1`, Initial: `30`
4. Save (Entity ID: e.g. `input_number.solakon_offset_zone1`)
5. Optional: repeat for Zone 2 (`input_number.solakon_offset_zone2`)

---

## 🧠 Core Functionality

### 1. PI Controller (Proportional-Integral Controller)

The blueprint uses a **PI controller** for precise zero export. The calculation logic is outsourced to a separate **Script Blueprint** (`PI Controller`) called from the main automation.

* **P Component (Proportional):**
  - Responds immediately to current deviations
  - Configurable aggressiveness via **P Factor** (e.g. 1.5)

* **I Component (Integral):**
  - Accumulates deviations over time, eliminates steady-state errors
  - **Anti-Windup:** Limited to ±1000 (in PI script)
  - **Automatic Reset:** Set to 0 on every zone change
  - **Tolerance Decay:** 5% reduction per cycle in main automation, while error ≤ tolerance and `|Integral| > 10`
  - **Zone 0 Freeze:** In Zone 0 (Surplus) the integral is kept unchanged

* **Error Calculation (mode-dependent, in PI script):**
  - **Normal (`ac_charge_mode=false`):** `raw_error = grid − target_offset`
  - **AC Charging (`ac_charge_mode=true`):** `raw_error = target_offset − grid` (inverted)
  - In both cases: capacity clamping to actually available capacity

* **Power Limit (zone-dependent, `max_power` to script):**
  - **Zone 0:** Hard Limit (PI not called)
  - **Zone 1:** Hard Limit (e.g. 800 W)
  - **Zone 2:** `Max(0, PV − Reserve)`
  - **AC Charging (Mode 3):** Configurable charge limit

* **PI Call Guard (in main automation):**
  - Zone 0 active → PI not called, integral frozen
  - AC Charging active → PI with `ac_charge_mode=true`, `at_max/at_min` guards disabled
  - Normal → PI only if `|error| > tolerance` AND no at-limit

---

### 2. 🔋 SOC Zone Logic

| Zone | SOC Range / Condition | Mode | Max. Discharge Current | Control Target | Notes |
|:-----|:----------------------|:-----|:----------------------|:--------------|:------|
| **0. Surplus Export** | SOC ≥ export threshold AND PV > Output + Grid + PV-Hysteresis | `'1'` | 2 A (stability buffer) | Hard Limit (max W) | **Optional.** Integral frozen. Persistent `input_boolean`. Exit with SOC-Hysteresis + PV-Hysteresis. |
| **1. Aggressive Discharge** | SOC > Zone 1 threshold | `'1'` | Configured max value (default: 40 A) | 0W + Offset 1 | Runs **until SOC ≤ Zone 3 threshold** (no yo-yo effect). Active at night too. |
| **2. Battery Conserving** | Zone 3 threshold < SOC ≤ Zone 1 threshold | `'1'` | **0 A** | 0W + Offset 2 | Dynamic limit: `Max(0, PV − Reserve)`. Optional: Night Shutdown. |
| **3. Safety Stop** | SOC ≤ Zone 3 threshold | `'0'` (Disabled) | 0 A | — | Output = 0 W. Full battery protection. |

#### Overview of Control Cases (choose block)

| Case | Condition | Action |
|:-----|:----------|:-------|
| **0A** | Surplus-Bool = `off` AND SOC ≥ export threshold AND (PV > Output + Grid + PV-Hysteresis OR PV = 0) | Zone 0 Start: Surplus-Bool → `on` |
| **0B** | Surplus-Bool = `on` AND (SOC < export threshold − SOC-Hysteresis OR PV ≤ Output + Grid − PV-Hysteresis) | Zone 0 End: Surplus-Bool → `off`, Integral = 0 |
| **A** | NOT AC-Charge-Bool = `on` AND SOC > Zone 1 threshold AND Cycle = `off` | Zone 1 Start: Cycle = `on`, Integral = 0, reset Surplus/AC-Bool, Timer-Toggle, Mode → `'1'` |
| **B** | NOT AC-Charge-Bool = `on` AND SOC < Zone 3 threshold AND Cycle = `on` | Zone 3 Stop: Cycle = `off`, Integral = 0, reset Surplus/AC-Bool, Mode → `'0'`, Output → 0W |
| **C** | NOT AC-Charge-Bool = `on` AND SOC < Zone 3 threshold AND Cycle = `off` AND Mode ≠ `'0'` | Zone 3 Guard: reset Surplus/AC-Bool, Mode → `'0'`, Output → 0W |
| **D** | Cycle = `on` AND Mode ∉ `{'1','3'}` AND SOC > Zone 3 threshold | Recovery: Timer-Toggle, Mode → `'3'` if AC-Charge-Bool = `on`, otherwise `'1'` (no integral reset, no zone change) |
| **G** | AC active AND SOC < charge target **AND Mode ≠ `'3'`** AND (Grid + Output) < −Hysteresis | AC Charging Start: AC-Bool = `on`, Timer-Toggle, Mode → `'3'`, Output → 0W |
| **H** | Mode = `'3'` AND (SOC ≥ charge target OR (Grid ≥ `ac_charge_offset + Hysteresis` AND Output = 0 W)) | AC Charging End: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **I** | Mode = `'3'` AND (AC Charging disabled OR AC-Charge-Bool ≠ `on`) | Safety correction: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **E** | NOT AC-Charge-Bool = `on` AND Zone 3 < SOC ≤ Zone 1 AND Cycle = `off` AND Mode = `'0'` AND NOT night | Zone 2 Start: Integral = 0, Timer-Toggle, Mode → `'1'` |
| **F** | NOT AC-Charge-Bool = `on` AND Night Shutdown active AND PV < PV Charge Reserve AND Cycle = `off` AND Mode active | Night Shutdown: Integral = 0, Mode → `'0'`, Output → 0W |

> **Order is critical:** Case D comes before Case G. This means Recovery only checks Mode ∉ `{'1','3'}` — AC charging mode `'3'` is **not** overwritten by Recovery. Case I comes after H and catches every Mode `'3'` state not legitimized by an active AC charging session.

#### 🔄 Recovery Mechanism (Case D)

If the inverter mode is externally reset (e.g. by an integration restart) while the discharge cycle is still active (Cycle = `on`), the blueprint automatically detects this and reactivates the mode via Timer-Toggle — without zone change or integral reset. Prerequisite: SOC > Zone 3 threshold AND Mode ≠ `'1'` AND Mode ≠ `'3'`.

The restored mode depends on AC charge state: if AC-Charge-Bool is `on`, Mode `'3'` is set; otherwise Mode `'1'`.

#### ⚠️ Safety Mechanism (Case I)

Catches the condition where the inverter is in Mode `'3'` but no active AC charging session exists (AC Charging disabled, helper `off` or unavailable). This can occur via external mode manipulation. Action: reset integral, Zone 1 → Mode `'1'` (Timer-Toggle), Zone 2 → Mode `'0'` + Output 0W.

---

### 3. ☀️ Surplus Export (Zone 0, Optional)

Enables active export of PV surplus when the battery is full. SOC and PV hysteresis prevent unstable toggling.

* **Activation:** Via the "Enable Surplus Export" parameter
* **Entry condition:** SOC ≥ export threshold AND (PV > (Output + Grid + PV-Hysteresis) OR PV = 0)
* **Exit condition:** SOC < (export threshold − SOC-Hysteresis) OR PV ≤ (Output + Grid − PV-Hysteresis)
* **Persistence:** State stored in `input_boolean` — survives multiple automation runs
* **Behavior:** Output to Hard Limit, discharge current 2 A (stability buffer), integral frozen (no decay, no PI call)
* **Disabled:** Classic zero export — no active export

---

### 4. ⚡ AC Charging (Optional)

Charges the battery when external grid feed-in is detected. Detection is based on `(Grid + Output) < −Hysteresis` — i.e. after subtracting the Solakon's contribution, surplus still remains. Typical use case: external PV system feeds surplus into the grid.

* **Activation:** Via the "Enable AC Charging" parameter
* **Entry condition (Case G):**
  - AC Charging enabled AND SOC < charge target
  - **Mode must not be `'3'`** (guard prevents re-entry when AC Charging already active)
  - (Grid + Output) < −Hysteresis
* **Stay condition:**
  - Mode stays `'3'` while SOC < charge target AND Grid < (ac_charge_offset + Hysteresis)
* **Exit condition (Case H):**
  - SOC ≥ charge target OR (Grid ≥ ac_charge_offset + Hysteresis AND Output = 0 W)
  - `Output = 0 W` guard prevents false trigger while PI is still actively controlling
* **PI Control:**
  - Script called with `ac_charge_mode=true` → inverted error: `target_offset − grid`
  - Positive error → increase charge power (Grid too negative → charge more)
  - `max_power` = configured charge limit
  - `at_max/at_min` guards not applied (direction inverted)
  - **Separate P/I Factors:** P small (~0.3–0.5) due to long hardware response (~25 s); I does the actual control work
* **State storage:** `input_boolean` (`ac_charge_state_helper`) persistent — signals PI script call branch
* **Return:**
  - Zone 1 → Mode `'1'` (Timer-Toggle) + Integral Reset
  - Zone 2 → Mode `'0'` + Output 0W + Integral Reset
* **Priority:** Zone 3 (SOC protection) always takes priority

---

### 5. ⏱️ Timer-Toggle and Mode Change Sequence

To ensure stable adoption of mode changes by the Solakon ONE, a **toggle between 3598 and 3599** is used instead of a fixed value:

- If current timer value = 3599 → write 3598
- Otherwise → write 3599

This creates a state change that causes the inverter to reliably adopt the new mode. No delay required. The toggle is performed at every mode change (Cases A, D, E, G, H-Zone1, I-Zone1) directly before setting the mode.

**Continuous Timeout Reset (Step 2):** Countdown < 120s → Timer-Toggle.

---

### 6. 🌙 Night Shutdown (Optional)

Applies to **Zone 2 only** (Case F). Zone 1 and AC Charging continue at night.

* **Threshold:** PV power below the **PV Charge Reserve** value (no separate parameter)
* **Behavior at night:** Mode → `'0'`, integral reset, Output 0W
* **Zone 1:** Continues running (high SOC → aggressive discharge still desired)

---

## 📊 Input Variables and Configuration

### 🔌 Required Entities

| Category | Variable | Default Entity | Description |
|:---------|:---------|:--------------|:------------|
| **External** | Grid Power Sensor | *(no default)* | E.g. Shelly 3EM. **Positive = import, Negative = export** |
| **Solakon** | Solar Power | `sensor.solakon_one_pv_leistung` | Current PV generation in Watts |
| **Solakon** | Actual Output Power | `sensor.solakon_one_leistung` | Current AC output power in Watts |
| **Solakon** | Battery SOC | `sensor.solakon_one_batterie_ladestand` | State of charge in % |
| **Solakon** | Remote Timeout Countdown | `sensor.solakon_one_fernsteuerung_zeituberschreitung` | Remaining countdown |
| **Solakon** | Output Power Controller | `number.solakon_one_fernsteuerung_leistung` | Sets power setpoint |
| **Solakon** | Max. Discharge Current | `number.solakon_one_maximaler_entladestrom` | Sets discharge current limit |
| **Solakon** | Mode Reset Timer | `number.solakon_one_fernsteuerung_zeituberschreitung` | Timer-Toggle 3598↔3599 |
| **Solakon** | Operating Mode Selection | `select.solakon_one_modus_fernsteuern` | `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority |
| **Helper** | Discharge Cycle Storage | `input_boolean.soc_entladezyklus_status` | Input Boolean: `on`/`off` |
| **Helper** | Integral Storage | `input_number.solakon_integral` | Input Number: −1000 to 1000 |
| **Script** | PI Controller Script | `script.pi_controller` | Script created from the PI Controller Blueprint |
| **Helper** | Surplus State Storage | `input_boolean.solakon_surplus_aktiv` | Only when Zone 0 active. |
| **Helper** | AC Charge State Storage | `input_boolean.solakon_ac_laden_aktiv` | Only when AC Charging active. |

---

### 🎚️ Control Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **P Factor** | 1.3 | 0.1 | 5.0 | Proportional gain. Higher = more aggressive. |
| **I Factor** | 0.05 | 0.01 | 0.2 | Integral gain. Higher = faster error correction, but less stable. |
| **Tolerance Range** | 25 W | 0 | 200 W | Deadband around setpoint. No PI correction within (integral decay instead). |
| **Wait Time** | 3 s | 0 | 30 s | Delay after power change in all modes (Zone 1, Zone 2, AC Charging). Compensates for inverter response time. |

> **Note:** P and I factors apply to Zone 1 and Zone 2. For AC Charging mode (Mode `'3'`) separate factors are used — see [AC Charging Parameters](#-ac-charging-optional).

---

### 🔋 SOC Zone Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Exceeding activates Zone 1. |
| **Zone 3 Stop** | 20 % | 1 % | 49 % | Falling below stops discharge completely. |
| **Max. Discharge Current Zone 1** | 40 A | 0 A | 40 A | Zone 2 and AC Charging automatically use 0 A. |

**Important:** Zone 1 threshold must be **greater** than Zone 3 threshold! Blueprint validates this at startup.

---

### ⚙️ Zone 1/2 Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Offset 1 (Static)** | 30 W | -100 | 100 W | Static fallback for Zone 1. |
| **Offset 1 (Dynamic)** | *(empty)* | — | — | Optional `input_number` entity. Overrides static value. |
| **Offset 2 (Static)** | 30 W | -100 | 100 W | Static fallback for Zone 2. |
| **Offset 2 (Dynamic)** | *(empty)* | — | — | Optional `input_number` entity. Overrides static value. |
| **PV Charge Reserve** | 50 W | 0 | 1000 W | Zone 2 limit: `Max(0, PV − Reserve)`. Also used as PV threshold for Night Shutdown. |

---

### 🔒 Safety Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Max. Output Power** | 800 W | 0 | 1200 W | Hard Limit in Zone 0 and Zone 1. |

---

### ☀️ Surplus Export (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable Surplus Export** | false | — | — | Toggle for Zone 0. |
| **SOC Threshold Surplus** | 90 % | 50 % | 99 % | From this SOC with PV surplus → Zone 0. |
| **Hysteresis Surplus Exit (SOC)** | 5 % | 1 % | 20 % | SOC must fall by this amount below entry threshold before Zone 0 is exited. |
| **PV Surplus Hysteresis** | 50 W | 10 W | 200 W | Deadband around house consumption for entry and exit. |

---

### ⚡ AC Charging (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable AC Charging** | false | — | — | Toggle for AC Charging. |
| **SOC Charge Target** | 90 % | 10 % | 99 % | Charging stops at this SOC. |
| **Max. Charge Power** | 800 W | 50 | 1200 W | Upper limit of AC charge power (`max_power` to script). |
| **Hysteresis AC Charging** | 50 W | 0 | 300 W | Deadband for entry and exit. Entry: (Grid + Output) < −Hysteresis. Exit: Grid ≥ (Offset + Hysteresis) AND Output = 0 W. |
| **AC Charging Offset (Static)** | -50 W | -100 | 100 W | Control target in AC charging mode. Negative = targeting export → higher charge power. |
| **AC Charging Offset (Dynamic)** | *(empty)* | — | — | Optional `input_number` entity. Overrides static value. |
| **AC Charging P Factor** | 0.5 | 0.1 | 5.0 | Proportional gain in AC charging mode. Keep small due to long hardware response (~25 s). |
| **AC Charging I Factor** | 0.07 | 0.01 | 0.2 | Integral gain in AC charging mode. Does the actual control work with long response times. |

---

### 🌙 Night Shutdown (Optional)

| Parameter | Default | Description |
|:----------|:--------|:------------|
| **Enable Night Shutdown** | false | On/Off toggle for the feature |
| **PV threshold for "Night"** | — | Uses the **PV Charge Reserve** value as threshold |

---

## 🎯 Dynamic Offset Blueprint (Recommended)

### 👉 [Solakon ONE — Dynamic Offset (Volatility Controller)](https://github.com/D4nte85/Solakon-one-dynamic-offset-blueprint)

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)

Automatically adjusts the zero-point offset based on current grid volatility (standard deviation of the last 60s).

| Grid Condition | StdDev | Calculated Offset |
|:--------------|:------:|:-----------------:|
| Very calm | 5 W | 30 W *(minimum)* |
| Normal | 30 W | 53 W |
| Noisy | 80 W | 128 W |
| Very noisy | 160 W | 228 W |
| Extreme | 250 W+ | 250 W *(maximum)* |

---

## 🔧 PI Controller Tuning

The controller is tuned in three steps.

### Step 1: Find Wait Time (P = 1, I = 0)

```yaml
P Factor: 1.0
I Factor: 0.0
```

P=1 delivers a clean, clearly visible step in output power — making the system response easy to read.

The total delay consists of several parts:

1. **Modbus write** — HA waits automatically for confirmation that the setpoint was successfully written before the automation continues (typically 0.5–2 s depending on bus load). This does **not** count toward the wait time — the wait time begins afterwards.
2. **Inverter response** — the Solakon ONE needs time after the successful write to actually adjust output power (hardware ramp). This is the main component of the wait time.
3. **Measurement latency** — Shelly and Solakon integration read new values via their own Modbus polling; this delay also belongs to the wait time.

The **wait time** covers points 2 and 3. Example for the total path from write command to stable reading:

```
number.set_value sent
  + 0.5–2.0 s   Modbus write + write confirmation   → wait time starts here
  + 0.5–1.5 s   Inverter adjusts output power (hardware ramp)
  + 0.5–1.5 s   Shelly + Solakon integration read new values via polling
─────────────────────────────────────────────────────
  = 1.5–5.0 s   Total delay
  → sensible wait time: 1–3 s
```

Timestamps are most easily read in the **activity logs** of Home Assistant (Settings → System → Logs → Activity Log), where each service call appears with a timestamp. Too short → controller reads stale values and oscillates. Too long → sluggish control.

### Step 2: Find P Factor (I = 0)

Increase P step by step (0.2 → 0.5 → 0.8 → …). Goal: find the **critical gain** — the point where output and grid power permanently oscillate around the setpoint. Then reduce P to approximately **50–60% of this value**. You deliberately seek the stability limit and then step back.

Typical working range using this method: **0.8–1.5**.

### Step 3: Add I Factor

With P alone a steady-state error remains when household load changes. The I component corrects this over time. Start small and increase gradually.

```yaml
I Factor: 0.02  # Starting point
```

Signs of too high I: system oscillates slowly with a long period. The **Anti-Windup** limits the integral to ±1000, the **Tolerance Decay** (5%/cycle when error ≤ tolerance) automatically reduces it during stable operation.

Typical working range: **0.03–0.08**. For AC Charging, tune separately — keep P especially small (~0.3–0.5) due to the long hardware response (~25 s) and let I do the actual work.

---

## 🛑 Important Error Messages

| Message | Cause | Solution |
|:--------|:------|:---------|
| **SOC limits invalid** | Zone 1 threshold ≤ Zone 3 threshold | Zone 1 (e.g. 50%) > Zone 3 (e.g. 20%) |
| **SOC sensor UNKNOWN/UNAVAILABLE** | Solakon integration offline | Check connection |
| **Timeout Countdown UNKNOWN/UNAVAILABLE** | Sensor unavailable | Check Solakon integration |

---

## ⚙️ Technical Details

### Architecture

1. **Main Automation** (`solakon_one_zeroexport.yaml`): Zone control, SOC logic, surplus state, AC charge state, discharge current management, timeout reset, PI call guard, integral decay/freeze
2. **PI Controller Script** (`PI-Controller.yaml`): Pure calculation logic with mode-dependent error calculation via `ac_charge_mode` field

### PI Controller Implementation (in Script)

**Error Calculation (mode-dependent):**
```
ac_charge_mode = false (Normal):
  raw_error = grid_power - target_offset

ac_charge_mode = true (AC Charging):
  raw_error = target_offset - grid_power   ← inverted

In both cases — Capacity Clamping:
  raw_error > 0: error = Min(raw_error, max_power - current_power)
  raw_error < 0: error = Max(raw_error, 0 - current_power)
```

**Integral and Correction:**
```
integral_new = Clamp(integral_old + error, -1000, 1000)
correction   = error * P_Factor + integral_new * I_Factor
new_power    = current_power + correction
final_power  = Clamp(new_power, 0, max_power)
```

### Integral Management (in Main Automation)
```
Zone 0 active:               integral = integral_old (frozen)
AC Charging active (Branch B): → PI Script called (ac_charge_mode=true)
                                  at_max/at_min guards NOT applied
Normal (Branch C):            → PI Script called (ac_charge_mode=false)
                                  at_max/at_min guards applied
Otherwise (Decay):
  |Integral| > 10 → integral * 0.95
  otherwise       → no change
```

### Dynamic Power Limit
```
Mode '3' (AC Charging): max_power = ac_charge_power_limit
Zone 1 (cycle = on):    max_power = hard_limit
Zone 2 (cycle = off):   max_power = Max(0, PV - pv_charge_reserve)
```

### Timer-Toggle Mechanism
```
If timer_value == 3599 → write 3598
Otherwise              → write 3599
→ Then: set mode
```

### Case D / Recovery — Mode Exclusion and Dual Restore
```
Condition: Cycle = on
       AND Mode ∉ {'1', '3'}   ← '3' (AC Charging) explicitly excluded
       AND SOC > Zone 3 threshold

Action: Timer-Toggle
        → AC-Charge-Bool = on: Mode '3'
        → otherwise:            Mode '1'
        (no integral reset, no zone change)
```

### Case G — Entry Guard
```
Condition: ac_charge_enabled
       AND SOC < soc_ac_charge_limit
       AND Mode ≠ '3' ← Guard: prevents re-entry when AC Charging already active (Mode = '3')
       AND (grid + actual_power) < -ac_charge_hysteresis
```
The mode guard prevents false triggering when `actual_power = 0` (inverter not in controlled operation).

### Case H — Exit Condition
```
Condition: Mode = '3'
       AND (soc >= soc_ac_charge_limit
            OR (grid >= ac_charge_offset + hysteresis AND actual_power == 0))

  → ac_charge_state_helper = off, integral = 0
  → Zone 1: Timer-Toggle + Mode '1'
  → Zone 2: Mode '0' + Output 0W
```
`actual_power == 0` guard prevents false triggering while PI is still actively controlling.

### Case I — Safety Guard for Mode '3'
```
Condition: Mode = '3'
       AND (ac_charge_enabled = false
            OR ac_charge_entity_valid = false
            OR ac_charge_state_helper ≠ 'on')

Action: Integral = 0
        → Cycle = on (Zone 1): Timer-Toggle + Mode '1'
        → Cycle = off (Zone 2): Output 0W + Mode '0'
```
Catches external mode manipulation to `'3'` when AC Charging is not active.

### Automatic Discharge Current Control

| Zone | Discharge Current | Condition |
|:-----|:-----------------|:----------|
| Zone 0 (Surplus) | 2 A | Only if different |
| Zone 1 (Aggressive) | Configured maximum | Only if different, no surplus, no AC Charging |
| Zone 2 / AC Charging (Mode 3) | 0 A | Only if different |

---

## ⚠️ Important Notes

1. **Create helpers and script before installation:** input_boolean (cycle), input_number (integral), PI Controller Script must exist
2. **Create optional helpers when feature is enabled:** Surplus Boolean for Zone 0; AC Charge Boolean for AC Charging
3. **Grid power sensor:** Correct polarity (positive = import, negative = export)
4. **AC Charging control target:** Its own configurable offset (`ac_charge_offset`), independent of Zone 1/2. Negative value = targeting export → PI increases charge power.
5. **AC Charging P/I tuning:** Separate P/I factors (`ac_charge_p_factor`, `ac_charge_i_factor`). Keep P factor small (~0.3–0.5) due to long hardware response (~25 s) — I does the actual control work.
6. **Integral Helper:** Managed automatically — do not change manually
7. **Tolerance Decay:** Prevents integral windup — 5% reduction when `|Integral| > 10` and error ≤ tolerance
8. **Zone 0 Integral Freeze:** No decay, no PI call during surplus phase
9. **Recovery:** Mode loss with active cycle is detected automatically (Case D) — AC charging mode `'3'` is not overwritten; if AC-Charge-Bool is `on`, `'3'` is restored
10. **Case G Guard:** AC Charging entry only when Mode ≠ `'3'` — prevents re-entry when AC Charging already active
11. **Mode Values:** `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority
12. **Case I Safety:** External Mode `'3'` state without active AC charging session is automatically corrected

---

## 🔄 Trigger Overview

| Trigger | ID | Description |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Immediate PI control on grid power change |
| Solar Power Change | `solar_power_change` | Immediate PI control on PV power change |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone 1 threshold) |
| SOC Low | `soc_low` | Zone 3 Start (SOC < Zone 3 threshold) |
| Mode Change | `mode_change` | Reacts to external mode changes, triggers Recovery (Case D) or Safety correction (Case I) if applicable |
