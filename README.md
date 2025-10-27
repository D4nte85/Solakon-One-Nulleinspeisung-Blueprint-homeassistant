# ⚡ Solakon ONE Zero Export Blueprint (EN)

This Home Assistant Blueprint implements **dynamic zero export** for the Solakon ONE inverter, based on a Proportional Controller (P-Regulator) and an intelligent **three-tiered State-of-Charge (SOC) logic**.

## 🚀 Installation

Install the Blueprint directly into your Home Assistant instance using this button:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2FSolakon-ONE-Zero-Export%2Fsolakon_one_zeroexport.yaml)

---

### 🛠️ Preparation: Creating the Input Select Helper

The Blueprint requires an **Input Select** helper to store the status of the discharge cycle.

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
* **Measurement Principle:** The controller uses the **Grid Power Sensor** entity (e.g., Shelly 3EM) as the control deviation.
* **Correction:** The controller adjusts the Solakon ONE's output power to bring the grid power into the tolerance range.
    * Positive Grid Power (Consumption) $\rightarrow$ Increase output power.
    * Negative Grid Power (Export) $\rightarrow$ Decrease output power.
* **Control:** The aggressiveness of the reaction is controlled via the **Adjustment Factor** (`anpassungs_faktor`). Power is capped at a maximum of `max_active_power_limit` and a minimum of `0 W`.

---

### 2. 🔋 Three-Stage SOC Zone Logic

The control is divided into three operating modes based on the current SOC:

| Zone | SOC Range | Mode | Goal & Control |
| :--- | :--- | :--- | :--- |
| **1. Fast Discharge** | SOC > Upper Threshold (e.g., 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Control** with a 0 W offset for exact zero export. An active discharge cycle helper maintains this state until the lower threshold is undershot. |
| **2. Battery Conservation** | Lower Threshold (e.g., 20%) < SOC $\le$ Upper Threshold | `INV Discharge (PV Priority)` | **Active P-Control** to maintain a **negative zero-point offset** (e.g., -30 W) to enforce slight grid import. Discharge power is additionally limited by a **dynamic upper limit** (PV generation minus PV charge reserve) to prioritize battery charging. |
| **3. Safety Stop** | SOC $\le$ Lower Threshold (e.g., 20%) | `Disabled` | Output power is immediately set to **0 W** to protect the battery. The discharge cycle is ended. |

---

### ⏱️ Remote Timeout Reset
To prevent the Solakon ONE from switching to `Disabled` mode due to a lack of control signal, the internal **Remote Timeout Timer** is proactively reset to a high value (3599s) in the active zones (1 and 2) as soon as it falls below a critical value (120s).

---

### 🚦 Trigger Conditions (Automation Triggers)

The automation reacts to the following five critical events to ensure immediate and stable control:

1.  **Power Changes (with 3s Delay):**
    * State change of the **Grid Power Sensor** (`shelly_grid_power_sensor`) for $\ge 3$ seconds.
    * State change of the **Solar Power Sensor** (`solakon_solar_power_sensor`) for $\ge 3$ seconds.
    * *(Purpose: Triggers the stable P-Controller regulation.)*

2.  **SOC Threshold Reached:**
    * Battery SOC (`solakon_soc_sensor`) **above** the **Upper Limit** (`soc_fast_limit`).
    * Battery SOC **below** the **Lower Limit** (`soc_conservation_limit`).
    * *(Purpose: Controls the switch between the discharge zones.)*

3.  **Mode Change:**
    * State change of the **Operating Mode Selector** (`solakon_mode_select`).
    * *(Purpose: Reacts to manual or external mode changes.)*

---

## ⚙️ Input Variables and Default Entities

> **Important:** The **Default Entities** (`default`) have been aligned with the common naming conventions of the Solakon ONE Home Assistant integration and highlighted in the description. Please adjust the values during installation if your entity names differ.

### 🔌 Required Entities (Solakon ONE & Shelly/Smart Meter)

| Variable | Default Entity | Description |
| :--- | :--- | :--- |
| **Shelly/Grid Power Sensor** | *(No default)* | Sensor for current grid power (e.g., Shelly 3EM). **Positive values = Import**, **Negative values = Export**. |
| **Solakon ONE - Solar Power** | `sensor.solakon_one_total_pv_power` | Current PV generation in Watts. |
| **Solakon ONE - Battery SOC** | `sensor.solakon_one_battery_soc` | Battery State of Charge (SOC) in %. |
| **Solakon ONE - Output Power Regulator** | `number.solakon_one_remote_active_power` | Entity for setting the power setpoint. |
| **Solakon ONE - Operating Mode Selector** | `select.solakon_one_remote_control_mode` | Entity for switching the operating mode. |
| **Mode Reset Timer Entity (Setter)** | `number.solakon_one_remote_timeout_set` | Used to set/reset the remote timeout (max. 3599 s). |
| **Remote Timeout Countdown Sensor (Reader)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor showing the remaining timeout countdown. |
| **Discharge Cycle State Storage** | `input_select.soc_discharge_cycle_status` | The created `Input Select` helper (`on`/`off`). **The default name is pre-filled but must exist!** |

---

### 🎚️ Configuration Parameters (Adjustable Values)

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **SOC Threshold "Fast Regulation"** | `50 %` | Upper threshold. Exceeding this starts the aggressive discharge cycle (Zone 1). |
| **SOC Threshold "Charge Priority"** | `20 %` | Lower threshold. Falling below this stops discharge (Zone 3). |
| **Tolerance Range (Half-Width)** | `25 W` | The allowable range in Watts around the zero point before a correction is made. |
| **Regulation Factor** | `1.5` | Defines the aggressiveness of the P-Controller. |
| **Zero-Point Offset** | `-30 W` | The target value for grid power in Zone 2. A negative value forces a slight grid import. |
| **🔋 PV Charge Reserve** | `50 W` | PV power (in Watts) reserved in Zone 2 to ensure battery charging despite discharging. |
| **Maximum Output Power (Hard Limit)**| `800 W` | The maximum AC output power the blueprint is allowed to set. Used to comply with hardware parameters. |

---

## 🛑 Important Error Messages (System Log)

The blueprint includes built-in validation that stops the automation upon critical failures and writes a clear message to the Home Assistant System Log.

| Log Message | Cause | Solution |
| :--- | :--- | :--- |
| **The upper SOC threshold (X%) must be greater than the lower SOC threshold (Y%).** | The values for **SOC Threshold "Fast Regulation"** and **SOC Threshold "Charge Priority"** are equal or swapped. | Ensure the upper threshold (e.g., 50) is always higher than the lower threshold (e.g., 20). |
| **One or more critical entities are UNAVAILABLE or have invalid values.** | One of the critical entities (SOC, Timeout Sensor, Grid Power, Output Power Regulator) is `unavailable` or providing invalid data (e.g., if the Solakon integration is disconnected). | Check the status of the Solakon ONE entities and ensure the integration is active and connected. |
