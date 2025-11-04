# Solakon ONE Zero Export Blueprint for Home Assistant

This Home Assistant Blueprint implements dynamic zero export (zero grid feed-in) for the Solakon ONE inverter, based on a PI controller (Proportional-Integral controller) and intelligent three-stage battery state-of-charge (SOC) logic.

## Quick Installation

Install the Blueprint directly via this button in your Home Assistant instance:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_zero_export.yaml)

## Prerequisites

The Blueprint requires two helpers that you must create before installation:

### 1. Input Select: SOC Discharge Cycle Status

- Go to Settings → Devices & Services → Helpers
- Click Create Helper → Dropdown
- Name: e.g., `SOC Discharge Cycle Status`
- Add exactly these two options:
  - `on`
  - `off`
- Save (Entity ID: e.g., `input_select.soc_discharge_cycle_status`)

### 2. Input Number: Solakon Integral

- Go to Settings → Devices & Services → Helpers
- Click Create Helper → Number
- Name: e.g., `Solakon Integral`
- Important settings:
  - Minimum: `-1000`
  - Maximum: `1000`
  - Step: `1`
  - Initial value: `0`
- Save (Entity ID: e.g., `input_number.solakon_integral`)

## Key Features

### Modern PI Controller

The Blueprint uses a modern PI controller instead of a simple P controller for more precise zero export:

**P Component (Proportional):**
- Reacts immediately to current deviations
- Configurable aggressiveness via P factor (e.g., 1.5)
- Fast response to load changes

**I Component (Integral):**
- Accumulates deviations over time
- Eliminates steady-state errors (which a pure P controller cannot correct)
- Configurable speed via I factor (e.g., 0.05)
- Anti-Windup: Limited to ±1000 points
- Automatic Reset: Reset to 0 at every zone change
- Tolerance Decay: 5% decay when no correction needed (prevents unnecessary accumulation)

### Measurement Principle

- Based on grid power sensor (e.g., Shelly 3EM)
- Positive values = grid import, Negative values = grid export
- No delay - immediate response to sensor changes

### Error Calculation with Dynamic Limiting

- Zone 1: error = Min(available capacity, grid power)
- Zone 2: error = Min(available capacity, grid power - offset, PV capacity)
- Prevents integral windup through intelligent limiting

### Power Limiting

- Upper hard limit (e.g., 800 W) in Zone 1
- Dynamic PV-based limit in Zone 2
- Lower limit: fixed at 0 W

## Three-Stage SOC Logic

The control system is divided into three operating modes based on current SOC:

| Zone | SOC Range | Mode | Max. Discharge Current | Control Target | Special Features |
|------|-----------|------|------------------------|----------------|------------------|
| 1. Aggressive Discharge | SOC > 50% | INV Discharge (PV Priority) | 40 A | 0 W (exact zero export) | Runs continuously until SOC ≤ 20% (no yo-yo effect). Active at night too. Hard limit 800W. |
| 2. Battery Protection | 20% < SOC ≤ 50% | INV Discharge (PV Priority) | 0 A (AC limit only) | 30 W (slight grid import = charging) | Dynamic limit: Max(0, PV - Reserve). Optional: Night shutdown possible. |
| 3. Safety Stop | SOC ≤ 20% | Disabled | 0 A | - | Output = 0 W. Complete battery protection. |

**Important:**
- Zone 1 is activated once when exceeding 50% SOC and then runs through until 20% is reached
- Zone 2 only starts from Zone 3 (when charging from below 20% to above 20%)
- This prevents constant switching between zones
- Max. discharge current is automatically controlled: 40A in Zone 1, 0A in Zone 2

## Night Shutdown (Optional)

Night shutdown can be optionally activated and only affects Zone 2:

- **Activation:** Via the "Enable Night Shutdown" parameter
- **Threshold:** PV power below configured value (e.g., 10 W)
- **Behavior:**
  - Zone 1: Continues running at night (high SOC → aggressive discharge desired)
  - Zone 2: Set to `Disabled` when PV < threshold (Integral is reset)
  - Zone 3: Already disabled anyway
- **Useful for:** Battery protection at 0 PV when no base load is present

## Reliable Communication

To ensure stable communication with the Solakon ONE:

**Continuous Timeout Reset:**
- Remote timeout automatically reset to 3599s
- Trigger: Countdown falls below 120s

**Forced Reset on Mode Change:**
- Two-stage pulse sequence: 10s → 3599s (with 1s delay)
- Ensures safe mode adoption
- Prevents timeout errors

## Required Entities

| Category | Variable | Default Entity | Description |
|----------|----------|----------------|-------------|
| External | Grid Power Sensor | (no default) | E.g., Shelly 3EM. Positive = import, Negative = export |
| Solakon | Solar Power | `sensor.solakon_one_total_pv_power` | Current PV generation in watts |
| Solakon | Battery State of Charge (SOC) | `sensor.solakon_one_battery_soc` | Charge level in % |
| Solakon | Remote Timeout Countdown | `sensor.solakon_one_remote_timeout_countdown` | Remaining countdown |
| Solakon | Output Power Controller | `number.solakon_one_remote_active_power` | Sets power setpoint |
| Solakon | Max. Discharge Current | `number.solakon_one_battery_max_discharge_current` | Sets discharge current limit |
| Solakon | Mode Reset Timer | `number.solakon_one_remote_timeout_set` | Sets/Resets timeout (max. 3599s) |
| Solakon | Operating Mode Selection | `select.solakon_one_remote_control_mode` | Switches operating mode |
| Helper | Discharge Cycle Memory | `input_select.soc_discharge_cycle_status` | Input Select: on/off |
| Helper | Integral Memory | `input_number.solakon_integral` | Input Number: -1000 to 1000 |

## Configuration Parameters

### PI Controller Tuning

| Parameter | Default | Min | Max | Description |
|-----------|---------|-----|-----|-------------|
| P Factor | 1.5 | 0.1 | 5.0 | Proportional gain. Higher = more aggressive, faster. |
| I Factor | 0.05 | 0.01 | 0.2 | Integral gain. Higher = faster error correction, but less stable. |
| Tolerance Range | 25 W | 0 | 200 W | Dead band around control target. No correction within this zone. |

**Tuning Tips:**
- Too slow? → Increase P factor (e.g., 2.0) or increase I factor (e.g., 0.08)
- Oscillating/Overshooting? → Lower P factor (e.g., 1.0) or lower I factor (e.g., 0.03)
- Permanent mini-corrections? → Increase tolerance range (e.g., 35 W)
- Integral running up despite stable? → Tolerance decay automatically applies (5% reduction)

### SOC Thresholds

| Parameter | Default | Min | Max | Description |
|-----------|---------|-----|-----|-------------|
| Zone 1 Start | 50 % | 1 % | 99 % | Upper threshold. Exceeding activates aggressive discharge. |
| Zone 3 Stop | 20 % | 1 % | 49 % | Lower threshold. Falling below stops discharge completely. |
| Max. Discharge Current Zone 1 | 40 A | 0 A | 40 A | Discharge current in Zone 1. Zone 2 automatically uses 0 A. |

**Important:** Upper threshold must be greater than lower threshold! Blueprint validates this at startup.

### Zero Point & PV Reserve

| Parameter | Default | Min | Max | Description |
|-----------|---------|-----|-----|-------------|
| Zero Point Offset | 30 W | 0 | 100 W | Control target in Zone 2. Positive = slight grid import (battery charges). 0W = exact zero export. |
| PV Charging Reserve | 50 W | 0 | 1000 W | Reserved PV power for charging. Dynamic limit: Max(0, PV - Reserve). |

**PV Charging Reserve Explanation:**
- With 300W PV generation and 50W reserve → Max. output in Zone 2: 250W
- Ensures something is always left for the battery even with trickle charge
- Compensates for converter losses
- Automatically considered in PI controller error calculation

### Power Limits

| Parameter | Default | Min | Max | Description |
|-----------|---------|-----|-----|-------------|
| Max. Output Power | 800 W | 0 | 1200 W | Absolute upper limit (hard limit) in Zone 1 for inverter protection. |

### Night Shutdown

| Parameter | Default | Description |
|-----------|---------|-------------|
| Enable Night Shutdown | false | On/Off switch for the function |
| PV Threshold for "Night" | 10 W | Below this PV power it's considered night |

**Note:** Only Zone 2 is disabled at night. Zone 1 continues running!

## Usage Examples

### Conservative Home Setup
```
SOC Zone 1 Start: 60%
SOC Zone 3 Stop: 30%
Zero Point Offset: 50W
PV Charging Reserve: 100W
P Factor: 1.5
I Factor: 0.05
Tolerance Range: 30W
```

### Aggressive Apartment Setup
```
SOC Zone 1 Start: 40%
SOC Zone 3 Stop: 15%
Zero Point Offset: 20W
PV Charging Reserve: 30W
P Factor: 2.0
I Factor: 0.08
Tolerance Range: 20W
```

### Balanced Setup (Recommended)
```
SOC Zone 1 Start: 50%
SOC Zone 3 Stop: 20%
Zero Point Offset: 30W
PV Charging Reserve: 50W
P Factor: 1.5
I Factor: 0.05
Max. Output Power: 800W
Tolerance Range: 25W
Max. Discharge Current Zone 1: 40A
```

## Daily Operation Example

**Morning (06:00 - SOC: 25%)**
- Zone 2 active (20% < SOC ≤ 50%)
- PV slowly rising
- Max. discharge current: 0A (automatically set - battery protection)
- Control target: 30W (slight grid import)
- Dynamic limit: Max(0, PV - 50W)
- Battery is charging

**Noon (12:00 - SOC: 55%)**
- Zone 1 activated (SOC > 50%)
- Max. discharge current: 40A (automatically set)
- Control target: 0W (exact zero export)
- Hard limit: 800W
- Aggressive discharge during load peaks
- Remains active even when SOC falls below 50% again!

**Evening (20:00 - SOC: 22%)**
- Zone 1 still active (runs until 20%)
- Max. discharge current: still 40A
- Control target: still 0W
- Optional: Night shutdown not active (Zone 1 continues running)

**Night (22:00 - SOC: 19%)**
- Zone 3 activated (SOC ≤ 20%)
- Mode: `Disabled`
- Max. discharge current: 0A (automatically set)
- Output: 0W
- Battery protected

**Next Morning - Cycle starts over**

## Error Messages & Troubleshooting

The Blueprint validates configuration at startup. Errors are displayed in the system log:

| Message | Cause | Solution |
|---------|-------|----------|
| Upper SOC threshold must be greater than lower threshold | Threshold values incorrectly configured | Upper threshold (e.g., 50%) > Lower threshold (e.g., 20%) |
| SOC sensor is UNKNOWN/UNAVAILABLE | Solakon integration offline | Check connection to inverter |
| Timeout Countdown Sensor is UNKNOWN/UNAVAILABLE | Sensor not available | Check Solakon integration |
| Mode selector is UNKNOWN/UNAVAILABLE | Select entity missing | Check Solakon integration |
| Output power controller has no min attributes | Number entity faulty | Check Solakon integration |

## Technical Details

### Error Calculation (per zone):

```
Zone 1: error = Min(available_capacity, grid_power)
Zone 2: error = Min(available_capacity, grid_power - offset, pv_capacity)
```

### Integral Logic with Tolerance Decay:

```
If |grid_error| > tolerance:
    integral_new = Clamp(integral_old + error, -1000, 1000)
Else if |integral_old| > 10:
    integral_new = integral_old * 0.95  # 5% decay
Else:
    integral_new = integral_old  # No change
```

### PI Correction:

```
correction = error * P_Factor + integral_new * I_Factor
new_power = current_power + correction
```

### Final Power (with zone-dependent limiting):

```
Zone 1: final_power = Min(hard_limit, new_power)
Zone 2: final_power = Min(max(0, PV - reserve), new_power)
```

### Automatic Discharge Current Control

The Blueprint sets max. discharge current automatically:
- **Zone 1 (Aggressive Discharge):** Check if current value ≠ 40A → Set to 40A
- **Zone 2 (Battery Protection):** Check if current value ≠ 0A → Set to 0A
- **Zone 3 (Safety):** Discharge current remains at last value (usually 0A from Zone 2)

**Important:** Changes are only made when the current value differs from the target value (prevents unnecessary API calls).

## Important Notes

- **Create helpers before installation:** Both helpers must exist before configuring the Blueprint
- **Solakon ONE Integration:** Must be fully set up
- **Grid power sensor:** Correct polarity (positive = import, negative = export)
- **PI Controller Tuning:** Default values are conservative. Lower I factor if behavior is unstable.
- **Integral Helper:** Automatically managed - do not change manually!
- **Queued Mode:** Triggers are processed sequentially
- **No Trigger Delay:** Blueprint reacts immediately to sensor changes
- **3-Second Wait Time:** After each power change, the Blueprint waits 3 seconds
- **Discharge Current Automatic:** Max. discharge current is fully automatically controlled - no manual setting needed
- **Tolerance Decay:** Automatically prevents integral windup during stable control

## Trigger Overview

The Blueprint reacts to the following events:

| Trigger | ID | Delay | Description |
|---------|----|----|-------------|
| Grid Power Change | `grid_power_change` | none | Immediate PI control on grid power change |
| Solar Power Change | `solar_power_change` | none | Dynamic PV limit in Zone 2 |
| SOC High | `soc_high` | none | Zone 1 start (SOC > 50%) |
| SOC Low | `soc_low` | none | Zone 3 start (SOC ≤ 20%) |
| Mode Change | `mode_change` | none | Reacts to external mode changes |

All triggers execute the complete logic - no separate handling needed.

## License

This Blueprint is provided as-is without warranty. Use at your own risk.

## Contributing

Contributions, bug reports, and feature requests are welcome! Please open an issue or pull request on GitHub.

## Credits

Created for the Solakon ONE inverter community. Special thanks to all testers and contributors.
