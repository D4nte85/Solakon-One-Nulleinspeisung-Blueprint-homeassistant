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

| Zone | SOC Range | Mode | Goal & Control Logic |
| :--- | :--- | :--- | :--- |
| **1. Fast Discharge** | SOC > Upper Threshold (e.g., 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Control** with a 0 W offset for exact zero export. An active discharge cycle helper maintains this state until the lower threshold is undershot. |
| **2. Battery Conservation**| Lower Threshold (e.g., 20%) < SOC $\le$ Upper Threshold | `INV Discharge (PV Priority)` | **Charging Priority** forced by a **negative zero-point offset** (e.g., -30 W) to ensure slight grid consumption. Control is not P-based (threshold-based). Discharge power is additionally reduced by a **PV Charge Reserve** to secure charging. |
| **3. Safety Stop** | SOC $\le$ Lower Threshold (e.g., 20%) | `Disabled` | Output power is immediately set to **0 W** to protect the battery. The discharge cycle is ended. |

---

### ⏱️ Remote Timeout Reset
To prevent the Solakon ONE from switching to `Disabled` mode due to a lack of control signal, the internal **Remote Timeout Timer** is proactively reset to a high value (3599s) in the active zones (1 and 2) as soon as it falls below a critical value (120s).

## ⚙️ Input Variables and Default Entities

### 🔌 Required Entities (Solakon ONE & Shelly/Smart Meter)

| Variable | Default Entity | Description |
| :--- | :--- | :--- |
| **Shelly/Grid Power Sensor** | *(No Default)* | Sensor for current grid power. **Positive values = Consumption**, **Negative values = Export**. |
| **Solakon ONE - Solar Power** | `sensor.solakon_one_pv_power` | Current PV generation in Watts. |
| **Solakon ONE - Battery SOC** | `sensor.solakon_one_battery_soc` | Battery State of Charge (%) |
| **Solakon ONE - Output Power Controller** | `number.solakon_one_remote_active_power` | The entity to set the power target value. |
| **Solakon ONE - Operating Mode Selection** | `select.solakon_one_remote_mode` | The entity to switch the operating mode. |
| **Mode Reset Timer Entity (Setter)** | `number.solakon_one_remote_timeout_control` | Used to set/reset the remote timeout (to 3599 s). |
| **Remote Timeout Countdown Sensor (Reader)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor displaying the remaining timeout countdown. |
| **Discharge Cycle Status Storage** | *(See above)* | The created `Input Select` helper (`on`/`off`). |

### 🎚️ Configuration Parameters (Setting Values)

| Parameter | Default Value | Description |
| :--- | :--- | :--- |
| **SOC Threshold "Fast Control"** | `50 %` | Upper threshold. Exceeding this starts the aggressive discharge cycle (Zone 1). |
| **SOC Threshold "Charge Priority"** | `20 %` | Lower threshold. Falling below this stops discharge (Zone 3). |
| **Tolerance Range (Half-Width)** | `50 W` | The allowed range in Watts around the zero point before correction. |
| **Adjustment Factor** | `1.5` | Defines the aggressiveness of the P-Regulator in Zone 1. |
| **Zero-Point Offset** | `-30 W` | The target value for grid power in Zone 2 (battery conservation). Negative value forces slight grid consumption. |
| **🔋 PV Charge Reserve** | `15 W` | PV power reserved in Zone 2 to allow battery charging despite discharge. |
| **Maximum Output Power (Hard Limit)**| `800 W` | **The absolute maximum AC output power that the Blueprint is allowed to set.** This is used to respect hardware limits and allows for additional throttling of power (e.g., to 600 W), even if the device is technically capable of more. |
