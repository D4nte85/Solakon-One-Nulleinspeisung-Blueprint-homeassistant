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

## 🧠 Functionality: Three-Tiered SOC Zone Logic

The Blueprint controls the AC output limit of the Solakon ONE to achieve the most accurate zero export possible. The logic dynamically adapts the behavior to the current battery State of Charge (SOC) to protect the battery while maximizing self-sufficiency.

| Zone | SOC Range | Mode | Control Type | Objective |
| :--- | :--- | :--- | :--- | :--- |
| **1. Fast Control** | **> Upper Threshold (e.g., 50%)** | `INV Discharge (PV Priority)` | **Aggressive P-Regulator** with 0 W Offset | Maximum discharge and precise zero export. |
| **2. Battery-Conserving**| **Between Thresholds (e.g., 20%-50%)**| `INV Discharge (PV Priority)` | **Passive Threshold Control** with Negative Offset | Shifts the target point to slight **grid consumption** (e.g., -30 W) to prioritize battery charging (Charge Priority). |
| **3. Safety Stop** | **<= Lower Threshold (e.g., 20%)** | `Disabled` | **Fixed Limit of 0 W** | Immediate cessation of discharge to protect the battery. |

### P-Regulator Principle (Zone 1)
In the **Fast Control** zone, the difference between the measured **Grid Power** and the target point (0 W) is used as the correction value.
* **Positive Grid Power** (Consumption/Import): The inverter decreases its output power.
* **Negative Grid Power** (Export/Feed-in): The inverter increases its output power.

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
