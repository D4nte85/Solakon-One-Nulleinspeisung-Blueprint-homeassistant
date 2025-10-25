# Solakon ONE Nulleinspeisung & SOC-Management (V68) ⚡

Dieser Home Assistant Automations-Blueprint ermöglicht eine präzise, dynamische Nulleinspeisungsregelung (P-Regler) für den **Solakon ONE Hybrid-Wechselrichter** unter Berücksichtigung definierter Batterie-Ladestandszonen (SOC).

---

### 📥 Blueprint Import

Klicken Sie auf den Button, um den Blueprint direkt in Ihre Home Assistant Instanz zu importieren:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleispeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

#### Manuelle Import-URL

Sollte der Button nicht funktionieren, verwenden Sie diese RAW-URL im Blueprint-Import-Dialog:

`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleispeisung-Blueprint-homeassistant/main/solakon_one_nulleinspeisung.yaml`

---

### 🧠 Detaillierte Funktionsbeschreibung: Dynamische Nulleinspeisung mit SOC-Zonen-Logik

Der Blueprint steuert den **AC-Ausgangsleistungsregler** (`AC-Output-Limit`) des Solakon ONE, um eine präzise Nulleinspeisung zu erreichen. Die Steuerung basiert auf einem **Proportional-Regler (P-Regler)**, dessen Verhalten durch drei vordefinierte SOC-Zonen gesteuert wird:

| Zone | Modus | Ziel und Funktion | P-Regler Parameter |
| :--- | :--- | :--- | :--- |
| **1. Schnelle Regelung / Entladezyklus** | `INV Discharge (PV Priority)` | **Aggressive Entladung** und exakte Nulleinspeisung. | **Nullpunkt-Offset:** `0 W` (Strikte Nulleinspeisung). |
| **2. Batterieschonende Regelung** | `INV Discharge (PV Priority)` | **Aktive Entladung verhindern** und **PV-Überschuss speichern**. | **Nullpunkt-Offset:** Negativ (`nulleinspeisung_offset`). **Logik:** Begrenzung auf `min(PV-Leistung, Netzüberschuss)`. |
| **3. Sicherheitsstopp** | `Disabled` | **Sofortiges Stoppen** jeglicher Entladung (SOC < untere Schwelle). | Deaktiviert die Regelung und setzt den Modus auf `Disabled`. |

---

### ⚙️ Konfigurierbare Parameter

| Parameter Name | Standardwert | Erklärung der Funktion |
| :--- | :--- | :--- |
| **`soc_fast_limit`** | 50% | Die **Obere SOC-Schwelle**. Wird dieser Ladestand überschritten, startet der **aggressive Entladezyklus** (Zone 1). |
| **`soc_conservation_limit`** | 20% | Die **Untere SOC-Schwelle**. Wird dieser Ladestand unterschritten, schaltet das System sofort in den **Sicherheitsstopp** (Zone 3). |
| **`nulleinspeisung_toleranz`** | 50 W | Die **Toleranzgrenze (Halbbreite)** um den Nullpunkt-Offset. Der Regler korrigiert nur außerhalb dieses Bereichs. |
| **`anpassungs_faktor`** | 1.5 | Der **Regelungs-Faktor (Korrektur-Geschwindigkeit)**. Definiert die Aggressivität der Limit-Anpassung. |
| **`nulleinspeisung_offset`** | -30 W | Der **Negative Nullpunkt-Offset** für Zone 2. Erlaubt einen leichten Netzbezug, um die **aktive Entladung zu verhindern**. |

---
