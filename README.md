# ⚡ Solakon ONE Zero Export Control Blueprint (V149)

This Home Assistant Blueprint implements a **dynamic zero export control** solution for the Solakon ONE inverter, utilizing a Proportional Controller (P-Controller) and an intelligent **three-stage State of Charge (SOC) logic**.

## 🚀 Installation

Install the Blueprint directly in your Home Assistant instance using this button:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_zero_export.yaml)

---

### 🛠️ Prerequisite: Creating the Input Select Helper

The Blueprint requires an `Input Select` helper to store the status of the discharge cycle.

1.  In Home Assistant, navigate to **Settings** -> **Devices & Services** -> **Helpers**.
2.  Click **Create Helper**.
3.  Select the type **Dropdown** (**Input Select**).
4.  Enter a name, e.g., `SOC Discharge Cycle Status`.
5.  Under **Options**, add exactly these two values:
    * `on`
    * `off`
6.  Save the helper. The resulting entity (e.g., `input_select.soc_discharge_cycle_status`) must then be selected in the Blueprint under **Discharge Cycle Status Storage**.

---

## 🧠 Core Functionality

### 1. Proportional Controller (P-Controller)
* **Measurement Principle:** Control is based on the deviation measured by the **Grid Power Sensor** (e.g., Shelly 3EM).
* **Correction:** The controller adjusts the AC output power of the Solakon ONE to bring the grid power within the tolerance range.
    * **Positive Grid Power (Import)** $\rightarrow$ Increase AC output.
    * **Negative Grid Power (Export)** $\rightarrow$ Decrease AC output.
* **Control:** The aggressiveness of the reaction is managed by the **Adjustment Factor**. Power is limited to a maximum of `max_active_power_limit` and a minimum of `0 W`.

---

### 2. 🔋 Three-Stage SOC Zone Logic

The control system is divided into three operating modes based on the current SOC:

| Zone | SOC Range | Mode | Target & Control (Updated Logic V149) |
| :--- | :--- | :--- | :--- |
| **1. Fast Discharge** | SOC > Upper Threshold (e.g., 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Control** with a 0 W offset for exact zero export. An active discharge cycle helper maintains this state until the SOC drops below the lower threshold. |
| **2. Battery Conservation** | Lower Threshold (e.g., 20%) < SOC $\le$ Upper Threshold | `INV Discharge (PV Priority)` / `Disabled` | **Start/Stop Logic:** Discharge **starts** (`INV Discharge`) only if **PV generation strictly exceeds the PV Charge Reserve**. Discharge **stops** (`Disabled` and 0 W limit) immediately if PV generation no longer covers the reserve. **Active P-Control** with a **negative zero-point offset** (e.g., -30 W) to favor slight grid import. Discharge power is dynamically limited by (**PV generation minus PV Charge Reserve**). |
| **3. Safety Stop** | SOC $\le$ Lower Threshold (e.g., 20%) | `Disabled` | AC output power is immediately set to **0 W** to protect the battery. The discharge cycle ends. |

---

### ⏱️ Remote Timeout Reset and Mode Change Sequence

To ensure stable communication with the Solakon ONE, two timer mechanisms are used:

1.  **Continuous Timeout Reset (Refresh):** The internal **Remote Timeout Timer** is proactively reset to a high value (3599s) in the active zones (1 and 2) as soon as it drops below a critical value (**120s**).
2.  **Forced Reset before Mode Change (Pulse Sequence):** A **two-stage pulse sequence** (`1s` pulse, then `3599s` set) is executed before every **critical mode change** (`Disabled` $\leftrightarrow$ `INV Discharge (PV Priority)`). This ensures the Solakon reliably accepts the subsequent mode command and prevents timeout errors.

---

### 🚦 Trigger Conditions (Automation Triggers)

The automation reacts to the following five critical events to ensure immediate and stable control:

1.  **Power Changes (with 3s delay):**
    * State change of the **Grid Power Sensor** (`shelly_grid_power_sensor`) for $\ge 3$ seconds.
    * State change of the **Solar Power Sensor** (`solakon_solar_power_sensor`) for $\ge 3$ seconds.
    * *(Purpose: Triggers stable P-Control.)*
2.  **SOC Threshold Reached:**
    * Battery SOC (`solakon_soc_sensor`) **above** the **Upper Threshold** (`soc_fast_limit`).
    * Battery SOC **below** the **Lower Threshold** (`soc_conservation_limit`).
    * *(Purpose: Controls the transition between discharge zones.)*
3.  **Mode Change:**
    * State change of the **Operating Mode Select** (`solakon_mode_select`).
    * *(Purpose: Reacts to manual or external mode changes.)*

---

## ⚙️ Input Variables and Default Entities

> **Important:** **Default entities** (`default`) have been adapted to the common names used by the Solakon ONE Home Assistant integration and are highlighted in the description. Adjust the values during installation if your entity names differ.

### 🔌 Required Entities (Solakon ONE & Smart Meter)

| Variable | Default Entity | Description |
| :--- | :--- | :--- |
| **Shelly/Grid Power Sensor** | *(No default)* | Sensor for current grid power (e.g., Shelly 3EM). **Positive values = Import**, **Negative values = Export**. |
| **Solakon ONE - Solar Power** | `sensor.solakon_one_total_pv_power` | Current PV generation in Watts. |
| **Solakon ONE - Battery SOC** | `sensor.solakon_one_battery_soc` | Battery State of Charge (%) |
| **Solakon ONE - AC Output Limit** | `number.solakon_one_remote_active_power` | Entity for setting the power setpoint. |
| **Solakon ONE - Operating Mode Select** | `select.solakon_one_remote_control_mode` | Entity for switching the operating mode. |
| **Mode Reset Timer Entity (Setter)** | `number.solakon_one_remote_timeout_set` | Used to **set/reset** the remote timeout time (max. 3599 s). |
| **Remote Timeout Countdown Sensor (Reader)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor that displays the remaining remote timeout countdown value. |
| **Discharge Cycle Status Storage** | `input_select.soc_discharge_cycle_status` | The created `Input Select` helper (`on`/`off`). **The default name is entered automatically, but the helper must exist!** |

---

### 🎚️ Configuration Parameters (Settings)

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **SOC Threshold "Fast Control"** | `50 %` | Upper threshold. Exceeding this starts the aggressive discharge cycle (Zone 1). |
| **SOC Threshold "Charge Priority"** | `20 %` | Lower threshold. Falling below this triggers the safety stop (Zone 3). |
| **Tolerance Range (Half-Width)** | `25 W` | The allowed power range around the zero or offset point before a correction is made. |
| **Adjustment Factor** | `1.5` | Defines the aggressiveness/speed of the P-Controller. |
| **Zero Point Offset** | `-30 W` | The target value for grid power in Zone 2. A negative value enforces slight grid import. |
| **🔋 PV Charge Reserve** | `50 W` | The PV power (in Watts) reserved. This buffer compensates for **internal inverter losses** and ensures the battery can charge despite discharging. **Also used for the Start/Stop logic in Zone 2!** |
| **Max AC Output Power (Hard Limit)**| `800 W` | The absolute maximum AC output power the Blueprint is allowed to set. Used to comply with hardware parameters. |

---

## 🛑 Important Error Messages (System Log)

The Blueprint includes integrated validation that stops the automation and writes a clear message to the Home Assistant system log in the event of critical errors.

| Log Message | Cause | Solution |
| :--- | :--- | :--- |
| **The upper SOC threshold (X%) must be greater than the lower SOC threshold (Y%).** | The values for **SOC Threshold "Fast Control"** and **SOC Threshold "Charge Priority"** are equal or swapped. | Ensure the upper threshold (e.g., 50) is always higher than the lower threshold (e.g., 20). |
| **One or more critical entities are UNAVAILABLE or have invalid values.** | One of the critical entities (SOC, Timeout sensor, Grid Power, AC Output Limit) is `unavailable` or providing invalid data (e.g., if the Solakon integration is not connected). | Check the status of the Solakon ONE entities and ensure the integration is active and connected. |
