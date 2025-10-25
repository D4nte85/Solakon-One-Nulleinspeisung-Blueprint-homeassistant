# Solakon ONE Nulleinspeisung & SOC-Management ⚡

Dieser Home Assistant Automations-Blueprint ermöglicht eine präzise, dynamische Nulleinspeisungsregelung (P-Regler) für den **Solakon ONE Hybrid-Wechselrichter** unter Berücksichtigung definierter Batterie-Ladestandszonen (SOC). Das Ziel ist es, die Batterielebensdauer durch intelligente Entladezyklen zu schonen, während gleichzeitig der Eigenverbrauch maximiert und die Netzeinspeisung präzise auf den Nullpunkt geregelt wird.

---

## ✨ Features

* **Dreistufige SOC-Zonen-Logik:** Intelligente Steuerung des Entladeverhaltens basierend auf dem aktuellen Batterieladestand (SOC).
* **Dynamische P-Regelung:** Kontinuierliche Anpassung der AC-Ausgangsleistung (`AC-Output-Limit`) in Abhängigkeit von der gemessenen Netzleistung.
* **Batterieschonende Regelung (Zone 2):** Anwendung eines **negativen Offsets** (`nulleinspeisung_offset`), um die Entladung zu unterdrücken und die Batterie vorrangig zu laden.
* **Aggressive Entladung (Zone 1):** Start eines **Entladezyklus** bei hohem SOC mit strikter Nulleinspeisung (`Offset 0 W`).
* **Sicherheitsstopp (Zone 3):** Automatisches Umschalten auf Modus `Disabled` bei Unterschreitung einer kritischen SOC-Schwelle.
* **Remote Timeout Reset:** Automatisches Zurücksetzen des Solakon ONE Remote Timeout Timers, um den eingestellten Modus zu halten.

---

## 🛠️ Installation

Dieser Code ist als **Home Assistant Blueprint** konzipiert. Verwenden Sie den folgenden Button oder die manuelle URL, um ihn in Ihre Home Assistant-Instanz zu importieren.

### 📥 Blueprint Import

Klicken Sie auf den Button, um den Blueprint direkt in Ihre Home Assistant Instanz zu importieren. Stellen Sie sicher, dass Sie alle erforderlichen Entitäten vorher eingerichtet haben.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleispeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

#### Manuelle Import-URL

Sollte der Button nicht funktionieren, verwenden Sie diese **RAW-URL** im Blueprint-Import-Dialog:

\`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleispeisung-Blueprint-homeassistant/main/solakon_one_nulleinspeisung.yaml\`

---

## ⚙️ Verwendung

### 💡 Wichtiger Schritt: Erstellung des Entladezyklus-Helfers

Dieser Blueprint erfordert einen **Dropdown-Menü-Helfer (`input_select`)**, um den Zustand des aggressiven Entladezyklus zu speichern.

#### Vorgehensweise zur Erstellung:

1.  Gehen Sie in Home Assistant zu: **Einstellungen** → **Geräte & Dienste** → Reiter **Helfer**.
2.  Klicken Sie auf **Helfer erstellen** und wählen Sie den Typ **Dropdown-Menü (`Input Select`)**.
3.  Geben Sie ihm einen Namen, z. B. `Entladezyklus Zustand`.
4.  Fügen Sie **exakt** die folgenden beiden Optionen hinzu:
    * `on`
    * `off`
5.  Speichern Sie den Helfer. Die resultierende Entität (z. B. `input_select.entladezyklus_zustand`) muss im Blueprint unter **Entladezyklus-Zustandsspeicher** ausgewählt werden.

### 🧠 Detaillierte Funktionsbeschreibung: Dynamische Nulleinspeisung mit SOC-Zonen-Logik

Der Blueprint steuert den **AC-Ausgangsleistungsregler** (`AC-Output-Limit`) des Solakon ONE, um eine präzise Nulleinspeisung zu erreichen, deren Verhalten durch drei vordefinierte SOC-Zonen gesteuert wird:

| Zone | Modus | Ziel und Funktion | P-Regler Parameter |
| :--- | :--- | :--- | :--- |
| **1. Schnelle Regelung / Entladezyklus** | \`INV Discharge (PV Priority)\` | **Aggressive Entladung** und exakte Nulleinspeisung (SOC > Obere Schwelle oder Zyklus aktiv). | **Nullpunkt-Offset:** \`0 W\` (Strikte Nulleinspeisung). |
| **2. Batterieschonende Regelung** | \`INV Discharge (PV Priority)\` | **Aktive Entladung verhindern** und **PV-Überschuss speichern** (Zwischen den SOC-Schwellen). | **Nullpunkt-Offset:** Negativ (\`nulleinspeisung_offset\`). **Logik:** Begrenzung auf \`min(PV-Leistung, Netzüberschuss)\`. |
| **3. Sicherheitsstopp** | \`Disabled\` | **Sofortiges Stoppen** jeglicher Entladung (SOC < untere Schwelle). | Setzt Modus auf \`Disabled\` und Limit auf 0 W. |

### ⚙️ Erforderliche Entitäten und Konfigurationsparameter

#### I. Erforderliche Entitäten (Sensoren und Steuerungen)

| Parameter Name | Entitätstyp | Standard-Entität (Beispiel) | Beschreibung |
| :--- | :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | \`sensor\` (power) | z.B. \`sensor.shelly\_3em\_power\` | Der Sensor, der die **Netzleistung** misst. **Wichtig: Positiv = Einspeisung, Negativ = Bezug.** |
| **Solakon ONE - Solarleistung (PV-Erzeugung)** | \`sensor\` (power) | \`sensor.solakon\_one\_pv\_power\` | Der Sensor, der die aktuelle **Solarerzeugung** in Watt anzeigt. |
| **Solakon ONE - Batterieladestand (SOC)** | \`sensor\` | \`sensor.solakon\_one\_battery\_soc\` | Der **SOC-Sensor** des Solakon ONE (%). |
| **Solakon ONE - Ausgangsleistungsregler (AC-Output)** | \`number\` | \`number.solakon\_one\_remote\_active\_power\` | Die Entität zur Steuerung des **AC-Output-Limits** (Soll-Wert). |
| **Solakon ONE - Betriebsmodus-Auswahl** | \`select\` | \`select.solakon\_one\_remote\_mode\` | Die Entität zur Steuerung des **Betriebsmodus**. |
| **Modus-Reset-Timer-Entität (Setter)** | \`number\` | \`number.solakon\_one\_remote\_timeout\_control\` | Die Solakon ONE **Number** Entität, die die Modus-Haltezeit (in Sekunden) steuert. |
| **Remote Timeout Countdown Sensor (Ausleser)** | \`sensor\` | \`sensor.solakon\_one\_remote\_mode\_countdown\` | Der **Sensor**, der den aktuell verbleibenden Countdown-Wert des Remote Timeouts anzeigt. |
| **Entladezyklus-Zustandsspeicher** | \`input\_select\` | z.B. \`input\_select.entladezyklus\_zustand\` | Ein Helfer mit Optionen \`on\` und \`off\` zur Speicherung des Zyklus-Status. |

#### II. Konfigurationsparameter (Number Inputs)

| Parameter Name | Standardwert | Erklärung der Funktion |
| :--- | :--- | :--- |
| **\`soc\_fast\_limit\`** | **50%** | Die **Obere SOC-Schwelle**. Überschreiten = Start **aggressiver Entladezyklus**. |
| **\`soc\_conservation\_limit\`** | **20%** | Die **Untere SOC-Schwelle**. Unterschreiten = Start **Sicherheitsstopp** (\`Disabled\`). |
| **\`nulleinspeisung\_toleranz\`** | **50 W** | Die **Toleranzgrenze (Halbbreite)**. Korrektur erfolgt nur außerhalb dieses Bereichs. |
| **\`anpassungs\_faktor\`** | **1.5** | Der **Regelungs-Faktor**. Definiert die **Aggressivität** der Limit-Anpassung in Zone 1. |
| **\`nulleinspeisung\_offset\`** | **-30 W** | Der **Negative Nullpunkt-Offset** für Zone 2. Erlaubt einen leichten Netzbezug, um die **aktive Entladung zu verhindern**. |

---
