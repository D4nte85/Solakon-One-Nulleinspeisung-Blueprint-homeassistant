# Solakon ONE Nulleinspeisung & SOC-Management (V68) ⚡

Dieser Home Assistant Automations-Blueprint ermöglicht eine präzise, dynamische Nulleinspeisungsregelung (P-Regler) für den **Solakon ONE Hybrid-Wechselrichter** unter Berücksichtigung definierter Batterie-Ladestandszonen (SOC).

---

### 📥 Blueprint Import

Klicken Sie auf den Button, um den Blueprint direkt in Ihre Home Assistant Instanz zu importieren:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleispeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

#### Manuelle Import-URL

Sollte der Button nicht funktionieren, verwenden Sie diese RAW-URL im Blueprint-Import-Dialog (`Einstellungen` -> `Automatisierungen & Szenen` -> `Blueprints`):

`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleispeisung-Blueprint-homeassistant/main/solakon_one_nulleinspeisung.yaml`

---

### ⚙️ Funktionsweise der SOC-Zonen-Logik

Der Blueprint steuert den Solakon-Modus und das AC-Output-Limit, um die Nulleinspeisung in drei verschiedenen Batteriezuständen zu optimieren:

| Zone | SOC-Bereich | Ziel | Regelung |
| :--- | :--- | :--- | :--- |
| **1. Schnelle Regelung / Entladung** | SOC > Obere Schwelle (z.B. 50%) | Aggressive Entladung der Batterie und exakte Nulleinspeisung. | **Strikte Nulleinspeisung** (Offset 0 W) über P-Regler. |
| **2. Batterieschonung** | Untere Schwelle < SOC < Obere Schwelle | **Aktive Entladung verhindern**, Nulleinspeisung beibehalten und PV-Überschuss speichern. | **Passive Regelung** (Negativer Offset). Die Logik nutzt `min(PV-Leistung, Netzüberschuss)` zur Begrenzung, um Entladung zu stoppen und Ladung zu ermöglichen. |
| **3. Sicherheitsstopp** | SOC ≤ Untere Schwelle (z.B. 20%) | Sofortiges und sicheres Stoppen jeglicher Entladung. | Umschaltung auf den Modus **`Disabled`** und Limit 0 W. |

---

### 🎛️ Erforderliche Entitäten (Inputs)

Stellen Sie sicher, dass alle notwendigen Entitäten korrekt in Home Assistant eingerichtet sind:

| Name | Typ | Beschreibung |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | Sensor (`device_class: power`) | Der Sensor, der die **Netzleistung** misst (z.B. Shelly 3EM). |
| **Solakon ONE - Batterieladestand (SOC)** | Sensor | Der SOC-Sensor des Solakon ONE (%). |
| **Solakon ONE - Ausgangsleistungsregler** | Number | Die Entität zur Steuerung des AC-Output-Limits. |
| **Entladezyklus-Zustandsspeicher** | Input Select | Ein **Input Select Helper** mit den Optionen `on` und `off`. |
| *Weitere Parameter* | Number | Benutzerdefinierte Grenzwerte (SOC-Schwellen, Toleranzen, Offset und Faktor). |
