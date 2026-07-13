# ⚡ Solakon ONE Zero Export Blueprint (EN) - V307

This Home Assistant Blueprint implements **dynamic zero export** for the Solakon ONE inverter, based on a **PI controller (Proportional-Integral Controller)** with back-calculation anti-windup and an intelligent **SOC zone logic** with optional **surplus export** (incl. surplus forecast entry), optional **AC charging** from external feed-in, optional **tariff arbitrage** (incl. PV forecast suppression), and **multi-instancing** support.

The goal of this blueprint is to deliver PV energy directly to loads, bypassing the battery wherever possible.

---

**IMPORTANT:** The implementation of remote control in the Solakon integration means there is no true "disabled" remote command — this disables remote control itself, i.e. the Solakon ONE falls back to its default settings or those from the app.
For the intended behavior, a 0W schedule active for 24h should be created and activated, or the "Default Output Power" set to 0W. Both methods are equivalent.

---

## 🚀 Installation

Install the Blueprint directly via this button in your Home Assistant instance:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2FSolakon-ONE-Zero-Export%2Fsolakon_one_zeroexport.yaml)

The accompanying **PI Controller Script Blueprint** must also be imported:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2FSolakon-ONE-Zero-Export%2FPI-Controller.yaml)

---

## 🛠️ Prerequisites: Creating Required Helpers

The blueprint requires **three mandatory helpers** and a **script**, plus optional helpers depending on which features are enabled.

### 1. Input Boolean Helper (Discharge Cycle Storage) — REQUIRED

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `SOC Discharge Cycle Status`
4. Save (Entity ID: e.g. `input_boolean.soc_discharge_cycle_status`)

### 2. Input Number Helper (Integral Storage for PI Controller) — REQUIRED

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Number**
3. Name: e.g. `Solakon Integral`
4. **Important settings:**
   * Minimum: `-1200`
   * Maximum: `1200`
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
4. Save (Entity ID: e.g. `input_boolean.solakon_surplus_active`)

### 5. Input Boolean Helper (AC Charge State) — ONLY if AC Charging enabled

Persistent state storage for AC charging. Signals the PI script the inverted error calculation (`ac_charge_mode=true`). Also prevents the guards (at_max/at_min) from incorrectly blocking PI.

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `Solakon AC Charging Active`
4. Save (Entity ID: e.g. `input_boolean.solakon_ac_charging_active`)

### 6. Input Boolean Helper (Tariff Charge State) — ONLY if Tariff Arbitrage enabled

Persistent state storage for tariff charging sessions.

1. Go to **Settings** → **Devices & Services** → **Helpers**
2. Click **Create Helper** → **Toggle** (Input Boolean)
3. Name: e.g. `Solakon Tariff Charging Active`
4. Save (Entity ID: e.g. `input_boolean.solakon_tariff_charging_active`)

### 7. Input Number Helper (Power Limit Dynamic) — MULTI-INSTANCING ONLY

Written by the power distribution automation and dynamically limits the instance's output power Hard Limit.

1. Go to **Settings** → **Devices & Services** → **Helpers** → **Number**
2. Name: e.g. `Solakon Instance 1 Limit`
3. **Settings:** Min: `0`, Max: `≥ Global Max`, Step: `1`, Initial: `800`
4. Save (Entity ID: e.g. `input_number.solakon_instance1_limit`)
5. In the power distribution automation: enter as "Max. Power (input_number)"
6. In this instance automation: enter as "Max. Output Power — Dynamic"
7. Repeat for each additional instance

### 8. Input Number Helper (Error Share) — MULTI-INSTANCING ONLY

Contains the share of grid error (0.0–1.0) this instance handles via its PI controller.
Calculated by the power distribution automation: `usable_i / Σ usable_j` with `usable_i = (SOC_i − Min-SOC_i) / 100 × Cap_i`. Without capacity sensor: `Cap_i = 100` — weighting by SOC percentage points.

1. Go to **Settings** → **Devices & Services** → **Helpers** → **Number**
2. Name: e.g. `Solakon Instance 1 Share`
3. **Settings:** Min: `0`, Max: `1`, Step: `0.001`, Initial: `1`
4. Save (Entity ID: e.g. `input_number.solakon_instance1_share`)
5. Repeat for each additional instance
6. In the power distribution automation: enter as "Error Share Helper"
7. In this instance automation: enter as "Error Share Helper"

### 9. Input Number Helper for Dynamic Offset (Optional)

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
  - Configurable aggressiveness via **P Factor** (e.g. 1.3)

* **I Component (Integral):**
  - Accumulates deviations over time, eliminates steady-state errors
  - **Anti-Windup:** Back-calculation — after each control action, integral is reset to the value that exactly produces the actual (possibly clamped) output. Clamped to ±effective_max.
  - **Automatic Reset:** Set to 0 on every zone change
  - **Tolerance Decay:** 5% reduction per cycle in main automation, while error ≤ tolerance and `|Integral| > 10`
  - **Zone 0 Freeze:** In Zone 0 (Surplus) the integral is kept unchanged

* **Error Calculation (mode-dependent, in PI script):**
  - **Normal (`ac_charge_mode=false`):** `raw_error = (grid − target_offset) × error_share`
  - **AC Charging (`ac_charge_mode=true`):** `raw_error = (target_offset − grid) × error_share` (inverted)
  - In both cases: capacity clamping to actually available capacity
  - `error_share` = share of grid error this instance handles (0–1, default 1.0)
  - Two independent multi-instance pools: Zero-Export (`error_share_entity`) and AC charging (`ac_error_share_entity`) — see Multi-Instancing section

* **Power Limit (zone-dependent, `max_power` to script):**
  - **Zone 0:** Hard Limit (PI not called)
  - **Zone 1:** Hard Limit (e.g. 800 W)
  - **Zone 2:** `Min(Hard Limit, Max(0, PV − Reserve))`
  - **AC Charging (Mode 3):** Configurable charge limit

* **PI Call Guard (in main automation):**
  - Zone 0 active → PI not called, integral frozen
  - Tariff Charging active → direct power set, no PI
  - AC Charging active → PI with `ac_charge_mode=true`, `at_max/at_min` guards disabled
  - Normal → PI only if `|error| > tolerance` AND no at-limit
  - `at_max_limit = false` if `current > dynamic_max` (PV drop) → PI can correct downward

---

### 2. 🔋 SOC Zone Logic

| Zone | SOC Range / Condition | Mode | Max. Discharge Current | Control Target | Notes |
|:-----|:----------------------|:-----|:----------------------|:--------------|:------|
| **0. Surplus Export** | SOC ≥ export threshold AND PV > Output + Grid + PV-Hysteresis | `'1'` | 2 A (stability buffer) | Hard Limit (max W) | **Optional.** Integral frozen. Persistent `input_boolean`. Exit with SOC-Hysteresis + PV-Hysteresis. |
| **1. Aggressive Discharge** | SOC > Zone 1 threshold | `'1'` | Configured max value (default: 40 A) | 0W + Offset 1 | Runs **until SOC ≤ Zone 3 threshold** (no yo-yo effect). Active at night too. |
| **2. Battery Conserving** | Zone 3 threshold < SOC ≤ Zone 1 threshold | `'1'` | **0 A** | 0W + Offset 2 | Dynamic limit: `Min(Hard Limit, Max(0, PV − Reserve))`. Optional: Night Shutdown. |
| **3. Safety Stop** | SOC ≤ Zone 3 threshold | `'0'` (Disabled) | 0 A | — | Output = 0 W. Full battery protection. |

#### Overview of Control Cases (choose block)

| Case | Condition | Action |
|:-----|:----------|:-------|
| **0A** | Surplus-Bool = `off` AND ((SOC ≥ export threshold AND (PV > Output + Grid + PV-Hysteresis OR PV = 0)) OR Surplus-Forecast-Forced) | Zone 0 Start: Surplus-Bool → `on` |
| **0B** | Surplus-Bool = `on` AND NOT Surplus-Forecast-Forced AND ((PV ≤ Output + Grid − PV-Hysteresis AND **NOT exit lock**) OR SOC < export threshold − SOC-Hysteresis) | Zone 0 End: Surplus-Bool → `off`, Integral = 0 |
| **A** | NOT AC-Charge-Bool = `on` AND NOT Tariff-Charge-Bool = `on` AND NOT discharge lock AND SOC > Zone 1 threshold AND Cycle = `off` | Zone 1 Start: Cycle = `on`, Integral = 0, reset Surplus/AC-Bool, Timer-Toggle, Mode → `'1'` |
| **B** | NOT AC-Charge-Bool = `on` AND NOT Tariff-Charge-Bool = `on` AND SOC < Zone 3 threshold AND Cycle = `on` | Zone 3 Stop: Cycle = `off`, Integral = 0, reset Surplus/AC-Bool, Output → 0W, Timer-Toggle, Mode → `'0'` |
| **C** | NOT AC-Charge-Bool = `on` AND NOT Tariff-Charge-Bool = `on` AND SOC < Zone 3 threshold AND Cycle = `off` AND Mode ≠ `'0'` | Zone 3 Guard: reset Surplus/AC-Bool, Output → 0W, Timer-Toggle, Mode → `'0'` |
| **D** | Cycle = `on` AND Mode ∉ `{'1','3'}` AND SOC > Zone 3 threshold | Recovery: Timer-Toggle, Mode → `'3'` if AC-Bool or Tariff-Bool = `on`, otherwise `'1'` (no integral reset, no zone change) |
| **GT** | Tariff enabled AND price < cheap threshold AND NOT PV-forecast-suppressed AND SOC < tariff target AND Mode ≠ `'3'` AND NOT Surplus-Bool = `on` | Tariff Charging Start: Tariff-Bool = `on`, Timer-Toggle, Output → charge power (direct), Mode → `'3'` |
| **HT** | Mode = `'3'` AND Tariff-Bool = `on` AND (price ≥ cheap threshold OR SOC ≥ target) | Tariff Charging End: Integral = 0, Tariff-Bool = `off`, Zone 1 → Timer-Toggle + `'1'` / Zone 2 → `'0'` + 0W |
| **TM** | Tariff active AND cheap ≤ price < expensive AND NOT PV-forecast-suppressed AND no charging AND NOT Surplus-Bool = `on` AND Mode = `'1'` | Discharge Lock: Integral = 0, Cycle = `off` (if Zone 1), Output → 0W, Timer-Toggle, Mode → `'0'` |
| **G** | AC active AND SOC < charge target AND **Mode ≠ `'3'`** AND NOT Tariff-Bool = `on` AND NOT Surplus-Bool = `on` AND (Grid + Output) < −Hysteresis | AC Charging Start: AC-Bool = `on`, Timer-Toggle, Mode → `'3'`, Output → 0W |
| **H** | Mode = `'3'` AND NOT Tariff-Bool = `on` AND (SOC ≥ charge target OR (Grid ≥ `ac_charge_offset + Hysteresis` AND Output = 0 W)) | AC Charging End: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **I** | Mode = `'3'` AND NOT AC-Bool = `on` AND NOT Tariff-Bool = `on` | Safety correction: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **E** | NOT AC-Bool = `on` AND NOT Tariff-Bool = `on` AND NOT discharge lock AND Zone 3 < SOC ≤ Zone 1 AND Cycle = `off` AND Mode = `'0'` AND NOT night | Zone 2 Start: Integral = 0, Timer-Toggle, Mode → `'1'` |
| **F** | NOT AC-Bool = `on` AND NOT Tariff-Bool = `on` AND NOT Surplus-Bool = `on` AND Night Shutdown active AND PV < PV Charge Reserve AND Cycle = `off` AND Mode active | Night Shutdown: Integral = 0, Output → 0W, Timer-Toggle, Mode → `'0'` |

> **Order is critical:** Case D comes before Cases GT/G. This means Recovery only checks Mode ∉ `{'1','3'}` — AC/Tariff charging mode `'3'` is **not** overwritten by Recovery. Case I comes after H and catches every Mode `'3'` state not legitimized by an active charging session.

#### 🔄 Recovery Mechanism (Case D)

If the inverter mode is externally reset (e.g. by an integration restart) while the discharge cycle is still active (Cycle = `on`), the blueprint automatically detects this and reactivates the mode via Timer-Toggle — without zone change or integral reset. Prerequisite: SOC > Zone 3 threshold AND Mode ∉ `{'1','3'}`.

The restored mode depends on active charging state: if Tariff-Charge-Bool or AC-Charge-Bool is `on`, Mode `'3'` is set; otherwise Mode `'1'`.

#### ⚠️ Safety Mechanism (Case I)

Catches the condition where the inverter is in Mode `'3'` but no active charging session exists (AC Charging disabled/helper `off`, Tariff Charging `off`). This can occur via external mode manipulation. Action: reset integral, Zone 1 → Mode `'1'` (Timer-Toggle), Zone 2 → Mode `'0'` + Output 0W (Timer-Toggle).

---

### 3. ☀️ Surplus Export (Zone 0, Optional)

Enables active export of PV surplus when the battery is full. SOC and PV hysteresis prevent unstable toggling.

* **Standard Entry:** SOC ≥ export threshold AND (PV > (Output + Grid + PV-Hysteresis) OR PV = 0)
* **Forecast Entry (optional):** Surplus forecast sensor ≥ threshold AND PV > Hard Limit AND SOC > zone-3 limit → Zone 0 entry **without export threshold**
* **Exit:** (PV ≤ (Output + Grid − PV-Hysteresis) AND NOT exit lock) OR SOC < (export threshold − SOC-Hysteresis) — both terms are blocked while surplus forecast is forced (see section 7); the optional exit lock (see section 11) blocks only the PV term
* **Persistence:** State stored in `input_boolean` — survives multiple automation runs
* **Behavior:** Output to Hard Limit, discharge current 2 A (stability buffer), integral frozen (no decay, no PI call)
* **Disabled:** Classic zero export — no active export

**Why the export threshold must sit below the full-charge point:** Entry checks `PV > consumption + hysteresis`. That is only measurable while the battery is still charging — the PV then runs unthrottled and shows `consumption + charge power`. At the full-charge point (app charge limit) the inverter throttles PV down to exactly the consumption; the surplus becomes invisible and entry depends on random consumption transients — delays of minutes are possible. A threshold ~5% below the app charge limit (e.g. 90–95% with max 100%) places entry safely inside the charging phase where the surplus is reliably measurable. For the same reason, re-entry after a cloud can be delayed when the SOC is already pinned at the maximum — the battery is not discharged during the cloud (as long as solar exists it stays untouched), so the SOC does not move. The exit lock (section 11) addresses exactly this.

---

### 4. ⚡ AC Charging (Optional)

Charges the battery when external grid feed-in is detected. Detection is based on `(Grid + Output) < −Hysteresis` — i.e. after subtracting the Solakon's contribution, surplus still remains. Typical use case: external PV system feeds surplus into the grid.

* **Entry condition (Case G):**
  - AC Charging enabled AND SOC < charge target
  - **Mode must not be `'3'`** (guard prevents re-entry when AC Charging already active)
  - NOT Tariff charging active (Tariff has priority)
  - NOT Surplus-Bool = `on` (Zone 0 blocks charging)
  - (Grid + Output) < −Hysteresis
* **Stay condition:** Mode stays `'3'` while SOC < charge target AND Grid < (ac_charge_offset + Hysteresis)
* **Exit condition (Case H):** SOC ≥ charge target OR (Grid ≥ ac_charge_offset + Hysteresis AND Output = 0 W)
  - `Output = 0 W` guard prevents false trigger while PI is still actively controlling
* **PI Control:** `ac_charge_mode=true` → inverted error: `(target_offset − grid) × error_share`
  - Positive error → increase charge power (Grid too negative → charge more)
  - `at_max/at_min` guards not applied (direction inverted)
  - **Separate P/I Factors:** P small (~0.3–0.5) due to long hardware response (~25 s); I barely effective — leave at default 0
* **Return:**
  - Zone 1 → Mode `'1'` (Timer-Toggle) + Integral Reset
  - Zone 2 → Mode `'0'` (Timer-Toggle) + Output 0W + Integral Reset
* **Priority:** Zone 3 (SOC protection) always takes priority. Tariff charging has priority over AC charging.

---

### 5. 💹 Tariff Arbitrage (Optional)

Charges the battery at cheap prices and locks discharge during neutral price phases.

* **Case GT (price < cheap threshold):** Charges with `tariff_charge_power` directly (no PI) until SOC target. Returns to Zone 1 (Timer-Toggle) or Zone 2 (Timer-Toggle, 0W). **Skipped when PV-forecast-suppressed.**
* **Case TM (cheap ≤ price < expensive):** Stops Zone 1 & 2 immediately (Mode `'0'`). Resets cycle helper. Preserves battery for expensive peaks. **Skipped when PV-forecast-suppressed and while surplus export (Zone 0) is active.**
* **Normal operation (price ≥ expensive):** Discharge lock lifted, standard zone logic (Cases A/E) takes over.
* **Dynamic thresholds:** Both cheap and expensive thresholds can be overridden by `input_number` entities.
* **⚠️ Sensor unit must match thresholds** — no conversion in blueprint (e.g. all in EUR/kWh).

---

### 6. 🌤️ PV Forecast Tariff Suppression (Optional)

Prevents tariff charging (GT) and discharge lock (TM) on sunny days.

* Forecast sensor ≥ threshold → `pv_forecast_suppressed = true` → GT and TM are completely skipped
* Only active when Tariff Arbitrage is also enabled
* Sensor `unknown`/`unavailable` → suppression inactive, tariff logic applies normally

---

### 7. 🌤️ Surplus Forecast Entry (Optional)

Forces early Zone 0 entry based on a PV surplus forecast.

* **Forcing flag:** `surplus_forecast_forced` = (forecast ≥ threshold) AND (PV > Hard Limit) AND (SOC > zone-3 limit) — tied to real clipping risk, not the raw forecast value alone; the SOC floor keeps the forcing from fighting the zone-3 safety stop (0A ↔ C flapping)
* **Entry:** `surplus_forecast_forced` → Zone 0 entry without export threshold (SOC only needs to be above the zone-3 limit)
* **Exit lock:** `surplus_forecast_forced` blocks SOC **and** PV exit symmetrically. Once PV drops below Hard Limit (including at night), the forecast drops below the threshold or the SOC drops below zone 3, the flag turns false immediately — normal exit logic resumes with no special case
* Only active when Surplus Export is also enabled

---

### 8. ⚖️ Multi-Instancing (Optional)

For operation of multiple Solakon ONE inverters in one household.

* `max_power_entity`: `input_number` for per-instance power limit — overrides static Hard Limit
* `error_share_entity`: `input_number` (0.0–1.0) for per-instance share of grid error — Pool 1 (Zero-Export, mode '1') only
  - `usable_i = (SOC_i − Min-SOC_i) / 100 × Cap_i`, `error_share_i = usable_i / Σ usable_j`
  - Without capacity sensor: `Cap_i = 100` — weighting by SOC percentage points
  - Written by a parent power distribution automation; leave empty in single-instance operation
* `ac_error_share_entity`: `input_number` (0.0–1.0) for per-instance share of grid error while AC charging — Pool 2 (mode '3'), independent from Pool 1
  - Prevents the AC-charge PI from freezing at 0 W: a charging instance isn't in mode '1' and would get `error_share = 0` from Pool 1 if both pools were shared
  - Leave empty in single-instance operation, or when not using AC charging with multi-instancing
* Single instance: always leave all three empty (error_share = 1.0, Hard Limit applies)

---

### 9. ⏱️ Timer-Toggle and Mode Change Sequence

To ensure stable adoption of mode changes by the Solakon ONE, a **toggle between 3598 and 3599** is used instead of a fixed value:

- If current timer value = 3599 → write 3598
- Otherwise → write 3599

This creates a state change that causes the inverter to reliably adopt the new mode. No delay required. The toggle is performed at every mode change (Cases A, D, E, G, GT, H-Zone1, HT-Zone1, I-Zone1) directly before setting the mode.

**Continuous Timeout Reset (Step 2):** Countdown < 120s → Timer-Toggle.

---

### 10. 🌙 Night Shutdown (Optional)

Applies to **Zone 2 only** (Case F). Zone 1, AC Charging and Surplus Export (Zone 0) continue at night — Zone 0 takes priority over Night Shutdown.

* **Threshold:** PV power below the **PV Charge Reserve** value (no separate parameter)
* **Behavior at night:** Output 0W, Timer-Toggle, Mode → `'0'`, integral reset
* **Zone 1:** Continues running (high SOC → aggressive discharge still desired)
* **Zone 2 Reactivation:** As soon as PV rises above PV Charge Reserve again, Case E applies

---

### 11. ⛅ Surplus Exit Lock (Optional)

Keeps Zone 0 alive through short PV dips (clouds) instead of exiting.

* **Prerequisite:** Surplus Export must also be enabled.
* **Lock flag:** `surplus_exit_locked = (forecast ≥ lock factor × Hard Limit) AND (SOC > zone-3 limit)` — the factor (default 1.5) is the safety margin against forecast errors: even a substantially wrong forecast still leaves real PV potential above the output limit, so a measured dip must be transient.
* **Effect (Case 0B):** Blocks **only** the PV exit. The SOC exit stays unblocked and always ends surplus — if the SOC genuinely falls below the exit threshold, the exit applies despite the lock. Zone 3 (Case C) ends surplus at any time as well.
* **Background:** Exiting with a full battery enters a state where the inverter throttles PV down to consumption — the surplus becomes unmeasurable afterwards, and re-entry depends on random consumption transients (delays of minutes). Since the battery is not discharged during a cloud (as long as solar exists it stays untouched), the SOC stays pinned at the maximum and the state does not resolve itself. The lock avoids it by not letting transient dips trigger the exit in the first place.
* **Sensor:** Currently forecast PV power in W, e.g. Solcast `power_now`.
* **Fallback:** Sensor unavailable/unknown → lock inactive, normal exit logic applies.

---

## 📊 Input Variables and Configuration

### 🔌 Required Entities

| Category | Variable | Default Entity | Description |
|:---------|:---------|:--------------|:------------|
| **External** | Grid Power Sensor | *(no default)* | E.g. Shelly 3EM. **Positive = import, Negative = export** |
| **Solakon** | Solar Power | `sensor.solakon_one_pv_power` | Current PV generation in Watts |
| **Solakon** | Actual Output Power | `sensor.solakon_one_active_power` | Current AC output power in Watts |
| **Solakon** | Battery SOC | `sensor.solakon_one_battery_state_of_charge` | State of charge in % |
| **Solakon** | Remote Timeout Countdown | `sensor.solakon_one_remote_timeout_countdown` | Remaining countdown |
| **Solakon** | Output Power Controller | `number.solakon_one_remote_control_power` | Sets power setpoint |
| **Solakon** | Max. Discharge Current | `number.solakon_one_maximum_discharge_current` | Sets discharge current limit |
| **Solakon** | Mode Reset Timer | `number.solakon_one_remote_control_timeout` | Timer-Toggle 3598↔3599 |
| **Solakon** | Operating Mode Selection | `select.solakon_one_remote_control_mode` | `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority |
| **Helper** | Discharge Cycle Storage | `input_boolean.solakon_soc_discharge_cycle_status` | Input Boolean: `on`/`off` |
| **Helper** | Integral Storage | `input_number.solakon_integral` | Input Number: −1200 to 1200, Step 1 |
| **Script** | PI Controller Script | `script.pi_controller` | Script created from the PI Controller Blueprint |
| **Helper** | Surplus State Storage | `input_boolean.solakon_surplus_active` | Only when Zone 0 active |
| **Helper** | AC Charge State Storage | `input_boolean.solakon_ac_charging_active` | Only when AC Charging active |
| **Helper** | Tariff Charge State Storage | `input_boolean.solakon_tariff_charging_active` | Only when Tariff Arbitrage active |

> **Note on "Max. Discharge Current":** In the official Solakon Home Assistant integration the entity `number.solakon_one_maximum_discharge_current` is **disabled by default**. Enable it first in HA under the Solakon device among the **configuration entities** (Device → Entity → gear icon → "Enable"). Without this entity the discharge current cannot be controlled.

---

### 🎚️ Control Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **P Factor** | 1.3 | 0.1 | 5.0 | Proportional gain. Higher = more aggressive. |
| **I Factor** | 0.05 | 0 | 0.2 | Integral gain. Higher = faster error correction, but less stable. |
| **Tolerance Range** | 25 W | 0 | 200 W | Deadband around setpoint. No PI correction within (integral decay instead). |
| **Wait Time** | 3 s | 0 | 30 s | Maximum wait time. Adaptive: exits early once actual power ≈ setpoint ± tolerance. Compensates for inverter response time. |

> **Note:** P and I factors apply to Zone 1 and Zone 2. For AC Charging mode (Mode `'3'`) separate factors are used — see [AC Charging Parameters](#-ac-charging-optional-1).

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
| **PV Charge Reserve** | 50 W | 0 | 1000 W | Zone 2 limit: `Min(Hard Limit, Max(0, PV − Reserve))`. Also used as PV threshold for Night Shutdown. |

---

### 🔒 Safety Parameters

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Max. Output Power (Hard Limit)** | 800 W | 0 | 1200 W | Hard Limit in Zone 0 and Zone 1. Overridable by `max_power_entity` for multi-instancing. |
| **Max. Output Power — Dynamic** | *(empty)* | — | — | Optional `input_number` entity for multi-instancing. Overrides static Hard Limit when set. |
| **Error Share Helper** | *(empty)* | — | — | Optional `input_number` (0.0–1.0) for multi-instancing, Pool 1 (Zero-Export). Leave empty in single-instance operation. |
| **AC Charge Error Share Helper** | *(empty)* | — | — | Optional `input_number` (0.0–1.0) for multi-instancing, Pool 2 (AC charging) — independent from the helper above. Leave empty unless using AC charging with multi-instancing. |

---

### ☀️ Surplus Export (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable Surplus Export** | false | — | — | Toggle for Zone 0. |
| **SOC Threshold Surplus** | 90 % | 50 % | 99 % | From this SOC with PV surplus → Zone 0. Recommendation: ~5% below the app charge limit (rationale in section 3). |
| **Hysteresis Surplus Exit (SOC)** | 5 % | 1 % | 20 % | SOC must fall by this amount below entry threshold before Zone 0 is exited. |
| **PV Surplus Hysteresis** | 50 W | 10 W | 200 W | Deadband around house consumption for entry and exit. |
| **Enable Surplus Forecast Entry** | false | — | — | Forces Zone 0 entry on high forecast without export threshold (SOC must stay above zone 3). |
| **Surplus Forecast Sensor** | *(empty)* | — | — | PV surplus forecast in W (e.g. Solcast). |
| **Surplus Forecast Threshold** | 5000 W | 0 | 20000 W | Minimum forecast value for forced Zone 0 entry. |
| **Enable Surplus Exit Lock** | false | — | — | Blocks the PV exit while forecast ≥ factor × Hard Limit (rides out clouds). |
| **Surplus Exit Lock Forecast Sensor** | *(empty)* | — | — | Currently forecast PV power in W (e.g. Solcast `power_now`). |
| **Surplus Exit Lock Factor** | 1.5 | 1.0 | 3.0 | Lock active while forecast ≥ factor × Hard Limit. Higher = more safety margin. |

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
| **AC Charging I Factor** | 0 | 0 | 0.2 | Integral gain in AC charging mode. Barely effective due to sluggish hardware (~25 s) — leave at default 0. |

---

### 💹 Tariff Arbitrage (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable Tariff Arbitrage** | false | — | — | Toggle for Tariff Arbitrage. |
| **Electricity Price Sensor** | *(empty)* | — | — | Current price sensor. Unit must match thresholds. |
| **Cheap Threshold (Static)** | 0.20 | 0 | 1 | Below this price charging starts. |
| **Cheap Threshold (Dynamic)** | *(empty)* | — | — | Optional `input_number` override. |
| **Expensive Threshold (Static)** | 0.25 | 0 | 1 | Below this price discharge is locked. |
| **Expensive Threshold (Dynamic)** | *(empty)* | — | — | Optional `input_number` override. |
| **Tariff Charging SOC Target** | 90 % | 10 % | 99 % | Charging stops at this SOC. |
| **Tariff Charge Power** | 1200 W | 50 | 1200 W | Direct charge power (no PI). |

---

### 🌤️ PV Forecast Tariff Suppression (Optional)

| Parameter | Default | Min | Max | Description |
|:----------|:--------|:----|:----|:------------|
| **Enable PV Forecast Suppression** | false | — | — | Skips GT and TM when forecast ≥ threshold. |
| **PV Forecast Sensor** | *(empty)* | — | — | PV yield forecast in W (e.g. Solcast). |
| **PV Forecast Threshold** | 5000 W | 0 | 20000 W | Minimum forecast value for suppression. |

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

The **wait time** covers points 2 and 3. Adaptive mode exits early once actual power ≈ setpoint ± tolerance — no unnecessary waiting when the inverter responds quickly.

```
number.set_value sent
  + 0.5–2.0 s   Modbus write + write confirmation   → wait time starts here
  + 0.5–1.5 s   Inverter adjusts output power (hardware ramp)
  + 0.5–1.5 s   Shelly + Solakon integration read new values via polling
─────────────────────────────────────────────────────
  = 1.5–5.0 s   Total delay
  → sensible wait time: 1–3 s
```

Timestamps are most easily read in the **activity logs** of Home Assistant (Settings → System → Logs → Activity Log). Too short → controller reads stale values and oscillates. Too long → sluggish control.

### Step 2: Find P Factor (I = 0)

Increase P step by step (0.2 → 0.5 → 0.8 → …). Goal: find the **critical gain** — the point where output and grid power permanently oscillate around the setpoint. Then reduce P to approximately **50–60% of this value**. You deliberately seek the stability limit and then step back.

Typical working range using this method: **0.8–1.5**.

### Step 3: Add I Factor

With P alone a steady-state error remains when household load changes. The I component corrects this over time. Start small and increase gradually.

```yaml
I Factor: 0.02  # Starting point
```

Signs of too high I: system oscillates slowly with a long period. The **Back-Calculation Anti-Windup** resets the integral after every clamped output, preventing runaway accumulation. The **Tolerance Decay** (5%/cycle when error ≤ tolerance) automatically reduces it during stable operation.

Typical working range: **0.03–0.08**. For AC Charging, tune separately — keep P especially small (~0.3–0.5) due to the long hardware response (~25 s); I is barely effective and can be left at 0.

---

## 🛑 Important Error Messages

| Message | Cause | Solution |
|:--------|:------|:---------|
| **SOC limits invalid** | Zone 1 threshold ≤ Zone 3 threshold | Zone 1 (e.g. 50%) > Zone 3 (e.g. 20%) |
| **SOC limits invalid** | Surplus export enabled AND export threshold ≤ zone-1 threshold | Export threshold (e.g. 90%) > Zone 1 (e.g. 50%) |
| **SOC sensor UNKNOWN/UNAVAILABLE** | Solakon integration offline | Check connection |
| **Timeout Countdown UNKNOWN/UNAVAILABLE** | Sensor unavailable | Check Solakon integration |

---

## ⚙️ Technical Details

### Architecture

1. **Main Automation** (`solakon_one_zeroexport.yaml`): Zone control, SOC logic, surplus state, tariff state, AC charge state, discharge current management, timeout reset, PI call guard, integral decay/freeze
2. **PI Controller Script** (`PI-Controller.yaml`): Pure calculation logic with back-calculation anti-windup, mode-dependent error calculation via `ac_charge_mode` field, `error_share` for multi-instancing

### PI Controller Implementation (in Script)

**Error Calculation (mode-dependent):**
```
ac_charge_mode = false (Normal):
  raw_error = (grid_power - target_offset) × error_share

ac_charge_mode = true (AC Charging):
  raw_error = (target_offset - grid_power) × error_share   ← inverted

In both cases — Capacity Clamping:
  raw_error > 0: error = Min(raw_error, max_power - current_power)
  raw_error < 0: error = Max(raw_error, 0 - current_power)
```

**Integral and Correction (Back-Calculation Anti-Windup):**
```
integral_candidate = integral_old + error
power_correction   = error × P_Factor + integral_candidate × I_Factor
new_power          = current_power + power_correction
final_power        = Clamp(new_power, 0, effective_max)

# Anti-windup: back-calculate integral from actual output
# When I_Factor ≠ 0:
  integral_new = Clamp((final_power - current_power - error × P_Factor) / I_Factor,
                       -effective_max, effective_max)
# When I_Factor = 0:
  integral_new = Clamp(integral_candidate, -effective_max, effective_max)
```

### Integral Management (in Main Automation)
```
Zone 0 active (Surplus):     integral = integral_old (frozen)
Tariff Charging active:      direct power set (no PI called)
AC Charging active (Branch B): → PI Script called (ac_charge_mode=true)
                                  at_max/at_min guards NOT applied
Normal (Branch C):            → PI Script called (ac_charge_mode=false)
                                  at_max/at_min guards applied
                                  at_max_limit = false when current > dynamic_max
                                  (allows downward correction after PV drop)
Otherwise (Decay):
  |Integral| > 10 → integral × 0.95
  otherwise       → no change
```

### Dynamic Power Limit
```
Mode '3' (AC Charging):  max_power = Min(ac_charge_power_limit, inverter_entity_max)
Zone 1 (cycle = on):     max_power = Min(hard_limit, inverter_entity_max)
Zone 2 (cycle = off):    max_power = Min(Max(0, PV - pv_charge_reserve), hard_limit, inverter_entity_max)
```

The `inverter_entity_max` reads the `max` attribute of the output power entity — prevents setting values the hardware cannot accept.

### Timer-Toggle Mechanism
```
If timer_value == 3599 → write 3598
Otherwise              → write 3599
→ Then: set mode
```

### Case D / Recovery — Mode Exclusion and Dual Restore
```
Condition: Cycle = on
       AND Mode ∉ {'1', '3'}   ← '3' (charging) explicitly excluded
       AND SOC > Zone 3 threshold

Action: Timer-Toggle
        → Tariff-Charge-Bool = on OR AC-Charge-Bool = on: Mode '3'
        → otherwise:                                        Mode '1'
        (no integral reset, no zone change)
```

### Case G — Entry Guard
```
Condition: ac_charge_enabled
       AND SOC < soc_ac_charge_limit
       AND Mode ≠ '3' ← Guard: prevents re-entry when AC Charging already active
       AND NOT tariff_charge_mode_active
       AND NOT surplus_active
       AND (grid + actual_power) < -ac_charge_hysteresis
```

### Case H — Exit Condition
```
Condition: Mode = '3'
       AND NOT tariff_charge_mode_active
       AND (soc >= soc_ac_charge_limit
            OR (grid >= ac_charge_offset + hysteresis AND actual_power == 0))

  → ac_charge_state_helper = off, integral = 0
  → Zone 1: Timer-Toggle + Mode '1'
  → Zone 2: Timer-Toggle + Mode '0' + Output 0W
```

### Case GT/HT/TM — Tariff Logic
```
GT Entry:
  tariff_arbitrage_enabled AND price < cheap_threshold
  AND NOT pv_forecast_suppressed
  AND soc < tariff_soc_charge_target AND Mode ≠ '3'
  AND NOT surplus_active
  → Tariff-Bool = on, Timer-Toggle, Output = tariff_charge_power, Mode '3'
  → stop (no PI)

HT End:
  Mode = '3' AND Tariff-Bool = on
  AND (price >= cheap_threshold OR soc >= tariff_soc_charge_target)
  → Integral = 0, Tariff-Bool = off
  → Zone 1: Output 0W, Timer-Toggle, Mode '1'
  → Zone 2: Output 0W, Timer-Toggle, Mode '0'

TM Lock:
  tariff_arbitrage_enabled AND price_discharge_locked AND NOT price_is_cheap
  AND NOT pv_forecast_suppressed
  AND NOT surplus_active
  AND Mode = '1' AND NOT charging active
  → Integral = 0, Cycle = off (if Zone 1), Output 0W, Mode '0'
```

### Automatic Discharge Current Control

| Zone | Discharge Current | Condition |
|:-----|:-----------------|:----------|
| Zone 0 (Surplus) | 2 A | Only if different |
| Zone 1 (Aggressive) | Configured maximum | Only if different, no surplus, no charging mode |
| Zone 2 / Charging (Mode 3) | 0 A | Only if different |

### kW → W Normalization

All power sensor readings are automatically normalized to Watts. Sensors reporting in kW (unit_of_measurement = `kW`) are multiplied by 1000. This ensures correct operation with sensors from different integrations regardless of their unit.

---

## 🔀 Multi-Instancing (Multiple Batteries)

### How It Works

Each instance handles only its proportional share of the grid error (`error_share`).
The power distribution automation calculates this share per instance from usable capacity —
the SOC range above the configured Min-SOC (Zone 3 Stop):

```
usable_i      = (SOC_i − Min-SOC_i) / 100 × Cap_i
error_share_i = usable_i / Σ usable_j
```

Without capacity sensor: `Cap_i = 100` — weighting by SOC percentage points (equal capacity assumed).

Example with two instances (Min-SOC 20% each, capacities 10 kWh / 5 kWh):

```
Instance 1: SOC=60% → (40/100) × 10 = 4.0 kWh usable
Instance 2: SOC=40% → (20/100) ×  5 = 1.0 kWh usable
→ share1 = 4.0/5.0 = 0.80, share2 = 1.0/5.0 = 0.20
```

The calculated share is written by the power distribution automation into an `input_number` helper,
which the PI controller of each instance reads as `error_share` and applies to `raw_error`.
In single-instance operation (no helper configured): `error_share = 1.0`.

Because weighting is based on usable capacity, the state of charge of multiple batteries
equalises automatically: an instance with more usable capacity takes more load and
discharges more strongly until both are at the same level again.

### Two Separate Pools: Zero-Export and AC Charging

Distribution runs across **two independent pools**, not one shared pool:

- **Pool 1 (Zero-Export):** only instances in mode `'1'` (Zone 1/Zone 2). Determines both the
  power limit and `error_share_entity`.
- **Pool 2 (AC Charging):** only instances with an active AC charge state helper (mode `'3'`).
  Determines only `ac_error_share_entity` — no power limit of its own, the AC charge limit
  stays independent.

Reason for the split: an instance currently AC charging is in mode `'3'` and therefore does
not count toward Pool 1. With a single shared error share, it would be assigned
`error_share = 0` there — and that same value would be handed to its AC-charge PI, freezing
it at 0 W despite active charging demand. With two separate pools, every instance gets a
correctly calculated share for whichever mode it's in. Both pools use the same weighting
logic (equal split or SOC-weighted, per the global toggle).

The two pools never overlap — an instance is at any time either in Pool 1, in Pool 2, or in
neither (e.g. Zone 3 stopped, tariff charging).

### Required Helpers per Instance (in addition to single-instance)

| Helper / Sensor | Type | Settings | Purpose |
|:----------------|:-----|:---------|:--------|
| `...instance_N_limit` | `input_number` | min:0, max:≥Global-Max, step:1 | Power limit from distribution → instance (Pool 1) |
| `...instance_N_share` | `input_number` | min:0, max:1, step:0.001 | Zero-Export error share from distribution → PI controller (Pool 1) |
| Capacity sensor (optional) | `sensor` | kWh — from Solakon integration | Accurate kWh weighting for different battery capacities |
| `...instance_N_ac_share` (AC charging only) | `input_number` | min:0, max:1, step:0.001 | AC-charging error share from distribution → PI controller (Pool 2) |

For Pool 2, the same AC charge state helper (`input_boolean`, see helper list item 5) is also
entered in the power distribution automation — it identifies which instances are currently
charging at the same time.

### Configuration Steps

1. Set up all instance blueprints as usual
2. Create the two new helpers (`limit`, `share`) per instance — plus `ac_share` for AC charging
3. In each instance automation enter: "Max. Output Power — Dynamic", "Error Share Helper" and (for AC charging) "AC Charge Error Share Helper"
4. Create the power distribution blueprint (`solakon_power_distribution.yaml`) as an automation:
   - Enter Min-SOC per instance — identical to the "Zone 3 Stop" value of the respective instance
   - Assign `limit` and `share` helpers per instance
   - Optional: enter the capacity sensor from the Solakon ONE integration per instance — recommended for different battery capacities
   - For AC charging: additionally assign the AC charge state helper and `ac_share` helper per charging instance

---

## ⚠️ Important Notes

1. **Create helpers and script before installation:** input_boolean (cycle), input_number (integral −1200 to 1200), PI Controller Script must exist
2. **Create optional helpers when feature is enabled:** Surplus Boolean for Zone 0; AC Charge Boolean for AC Charging; Tariff Charge Boolean for Tariff Arbitrage
3. **Grid power sensor:** Correct polarity (positive = import, negative = export)
4. **Tariff price sensor:** Unit must match thresholds (no conversion in blueprint)
5. **AC Charging control target:** Its own configurable offset (`ac_charge_offset`), independent of Zone 1/2. Negative value = targeting export → PI increases charge power
6. **AC Charging P/I tuning:** Keep P factor small (~0.3–0.5) due to long hardware response (~25 s); I barely effective — leave at 0
7. **Integral Helper bounds:** −1200 to 1200 (matches effective_max clamping in back-calculation anti-windup)
8. **Integral Helper:** Managed automatically — do not change manually
9. **Tolerance Decay:** Prevents integral accumulation during stable operation — 5% reduction when `|Integral| > 10` and error ≤ tolerance
10. **Zone 0 Integral Freeze:** No decay, no PI call during surplus phase
11. **Recovery:** Mode loss with active cycle is detected automatically (Case D) — charging mode `'3'` is not overwritten; if Tariff-Bool or AC-Bool is `on`, `'3'` is restored
12. **Case G Guard:** AC Charging entry only when Mode ≠ `'3'` — prevents re-entry when AC Charging already active
13. **Mode Values:** `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority
14. **Case I Safety:** External Mode `'3'` state without active charging session is automatically corrected
15. **Multi-Instancing:** Leave `max_power_entity` and `error_share_entity` empty in single-instance operation

---

## 🔄 Trigger Overview

| Trigger | ID | Description |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Immediate PI control on grid power change |
| Solar Power Change | `solar_power_change` | Immediate PI control on PV power change |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone 1 threshold) |
| SOC Low | `soc_low` | Zone 3 Start (SOC < Zone 3 threshold) |
| Mode Change | `mode_change` | Reacts to external mode changes, triggers Recovery (Case D) or Safety correction (Case I) if applicable |
