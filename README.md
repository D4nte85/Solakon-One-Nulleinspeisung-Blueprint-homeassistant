# Solakon ONE Zero Export Control ⚡

This Home Assistant Blueprint provides a dynamic, three-stage State of Charge (SOC) logic and a Proportional Controller (P-Controller) to manage the AC Output Power Limit of the Solakon ONE Inverter. The goal is to achieve **precise dynamic zero export** while implementing battery-conserving charge prioritization in the mid-range SOC zone.

---

## 🚀 Key Features

* **Dynamic Zero Export (P-Controller):** Achieves precise zero export by adjusting the AC output limit based on real-time grid power feedback (P-Controller).
* **Three-Stage SOC Logic:** Implements three distinct zones for efficient battery management:
    1.  **Fast Discharge (SOC High, e.g., >50%):** Aggressive P-Control to quickly reduce high SOC.
    2.  **Charge Priority (SOC Mid-Range, e.g., 20-50%):** Uses an adjustable negative offset (default: -30W) to favor slight grid import, prioritizing battery charging over discharge when SOC is moderate.
    3.  **Safety Stop (SOC Low, e.g., <20%):** Disables discharge completely to conserve the battery.
* **Remote Timeout Reset:** Automatically resets the Solakon ONE's remote control timeout when necessary to maintain continuous control.

---

## 📥 Import the Blueprint

You can import this Blueprint directly into your Home Assistant instance using the button below:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2FSolakon-ONE-Zero-Export%2Fsolakon_one_zeroexport.yaml)

**Manual Link:**
`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleinspeisung-Blueprint-homeassistant/Solakon-ONE-Zero-Export/solakon_one_zeroexport.yaml`

---

## 🛠️ Prerequisites (Helper Creation)

Before using the Blueprint, you must create one Helper entity to store the status of the discharge cycle.

| Helper Name | Type | Options | Purpose |
| :--- | :--- | :--- | :--- |
| `input_select.solakon_discharge_cycle_active` | **Input Select (Dropdown)** | `on`, `off` | Stores whether the aggressive discharge cycle is currently active (`on`) or inactive (`off`). |

### **How to Create the Helper:**

1.  In Home Assistant, navigate to **Settings** -> **Devices & Services** -> **Helpers**.
2.  Click **"Create Helper"**.
3.  Select **"Dropdown"** (Input Select).
4.  **Name:** `Solakon Discharge Cycle Active`
5.  **Icon (Optional):** `mdi:battery-arrow-down`
6.  Under **"Options"**, enter the following two options, one per line:
    * `on`
    * `off`
7.  Click **"Create"** (or **"Finish"**).
8.  Use this new entity (`input_select.solakon_discharge_cycle_active`) for the Blueprint input **"Discharge Cycle Status Storage"**.

---

## 🧠 Template Variables and Default Settings

The following variables allow you to fine-tune the control logic.

| Variable Name | Description | Standard/Default Value | Unit |
| :--- | :--- | :--- | :--- |
| **SOC Threshold "Fast Control"** | The upper SOC value. If exceeded, the aggressive discharge cycle starts. | **50** | `%` |
| **SOC Threshold "Charge Priority"** | The lower SOC value. If fallen below, the inverter switches to "Disabled" (Safety Stop). | **20** | `%` |
| **Tolerance Range (Half-Width)** | The allowed power range around the zero point before the P-Controller makes an adjustment. | **50** | `W` |
| **Control Factor (Correction Speed)** | Defines the aggressiveness/speed of the P-Controller's reaction. | **1.5** | (Factor) |
| **Zero Point Offset** | Negative Watt value used **only** in the mid-range SOC zone (Zone 2) to enforce slight grid import, prioritizing charging. | **-30** | `W` |

---

## 📥 Blueprint Inputs (Entities)

You must link the following entities from your Solakon ONE integration and your energy meter (e.g., Shelly 3EM).

| Input Name | Required Entity Type | Standard Entity | Description |
| :--- | :--- | :--- | :--- |
| **Shelly/Grid Power Sensor** | Sensor (`device_class: power`) | - | Sensor measuring the power flowing to/from the grid (positive = export). |
| **Solakon ONE - Solar Power** | Sensor (`device_class: power`) | `sensor.solakon_one_pv_power` | Current solar generation in Watts. |
| **Solakon ONE - Battery SOC** | Sensor | `sensor.solakon_one_battery_soc` | Battery State of Charge in %. |
| **Solakon ONE - Output Power Regulator**| Number | `number.solakon_one_remote_active_power` | Target value for charge/discharge power. |
| **Solakon ONE - Operating Mode Selector**| Select | `select.solakon_one_remote_mode` | Entity for controlling the operating mode. |
| **Mode Reset Timer Entity (Setter)** | Number | `number.solakon_one_remote_timeout_control` | Used to set/reset the Remote Timeout value (in seconds). |
| **Remote Timeout Countdown Sensor (Reader)** | Sensor | `sensor.solakon_one_remote_mode_countdown` | Displays the current remaining countdown value of the Remote Timeout. |
| **Discharge Cycle Status Storage** | Input Select | (See Helper above) | The Helper entity you created to track the discharge cycle. |
