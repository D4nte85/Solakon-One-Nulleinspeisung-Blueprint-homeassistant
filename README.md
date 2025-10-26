# 🔋 Solakon ONE Zero Export Blueprint (V113)

[![Home Assistant Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-41bdf5.svg?style=for-the-badge)](https://my.home-assistant.io/redirect/blueprint_import?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

This blueprint dynamically controls the Solakon ONE inverter using a **three-stage, SOC-dependent control mechanism** for dynamic zero export. It employs a Proportional Controller (P-Controller) to manage grid power and switches to a battery-conserving charge priority zone in the middle SOC range.

### 📥 Direct Import into Home Assistant

Click the button below to directly import the blueprint into your Home Assistant instance:

[![Open your Home Assistant instance and start setting up this blueprint.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

***

## ⚙️ How It Works

The core of the control is a **P-Controller** which uses the difference between the measured grid power and a target offset (usually 0 W) to dynamically adjust the AC Output Limit.

The logic is divided into three SOC zones:

1.  **🚀 Fast Control ($\text{SOC} > 50\%$ or Cycle Active):** Aggressive P-Control with $\text{Offset} = 0\text{ W}$ for fast discharge and precise zero export.
2.  **🐢 Battery-Conserving Control ($\text{20\%} < \text{SOC} \le 50\%$):** **Passive Mode** with Charge Priority. The **Zero Point Offset** is set negative to enforce slight grid import. The discharge power is reduced by the **PV Charge Reserve** to compensate for internal converter losses and guarantee battery charging.
3.  **🛑 Safety Stop ($\text{SOC} \le 20\%$):** Switches to **`Disabled`** and sets $\text{AC-Limit} = 0\text{ W}$ to protect the battery.

***

## 🛠️ Prerequisites

### 1. Helper: Discharge Cycle State Helper

This blueprint requires an **Input Select Helper** to track whether the aggressive discharge cycle is active.

| Parameter | Value |
| :--- | :--- |
| **Name** | e.g., `Solakon Discharge Cycle Status` |
| **Type** | **Input Select / Dropdown** |
| **Options** | `on`, `off` |

***

## 📝 Variables (User Input)

| Variable Name | Description | Default Value |
| :--- | :--- | :--- |
| **Grid Power Sensor** | Sensor measuring current grid power. (Must have `power` device_class) | (No Default) |
| **Solakon ONE - Solar Power (PV Generation)** | Sensor for current PV power in Watts. | (No Default) |
| **Solakon ONE - Battery State of Charge (SOC)** | SOC sensor of the Solakon ONE (%). | (No Default) |
| **Solakon ONE - Output Power Controller (AC-Output)** | The **Number** entity for controlling the AC Output Limit. | (No Default) |
| **Solakon ONE - Operating Mode Select** | The **Select** entity for controlling the operating mode. | (No Default) |
| **Mode Reset Timer Entity** | The Solakon **Number** entity (`remote_timeout_control`) used to **set/reset** the Remote Timeout. | (No Default) |
| **Remote Timeout Countdown Sensor** | Sensor/Number entity displaying the remaining Remote Timeout countdown in seconds. | (No Default) |
| **Discharge Cycle State Helper** | The previously created **Input Select Helper** (`on`/`off`). | (No Default) |
| **SOC Threshold "Fast Control"** | Upper SOC value, above which the aggressive discharge cycle starts. | `50 %` |
| **SOC Threshold "Charge Priority"** | Lower SOC value, below which the system switches to Disabled mode. | `20 %` |
| **Tolerance Range (Half-Width)** | Allowable range in Watts around the zero point. | `50 W` |
| **Adjustment Factor (Correction Speed)** | Aggressiveness of the P-Controller. | `1.5` |
| **Zero Point Offset** | Negative Watt value to shift the zero point into grid import in Zone 2 (Charge Priority). | `-30 W` |
| **🔋 PV Charge Reserve (Intermediate-SOC)** | Reserved PV power (Watts) to compensate for internal converter losses in Zone 2. | `15 W` |
