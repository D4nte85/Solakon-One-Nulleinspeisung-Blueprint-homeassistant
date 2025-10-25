# Solakon ONE Nulleinspeisung & SOC-Management ⚡

Dieser Home Assistant Automations-Blueprint ermöglicht eine präzise, dynamische Nulleinspeisungsregelung (P-Regler) für den **Solakon ONE Hybrid-Wechselrichter** unter Berücksichtigung definierter Batterie-Ladestandszonen (SOC).

---

### 📥 Blueprint Import

Klicken Sie auf den Button, um den Blueprint direkt in Ihre Home Assistant Instanz zu importieren. Stellen Sie sicher, dass Sie alle erforderlichen Entitäten vorher eingerichtet haben.

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleispeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

#### Manuelle Import-URL

Sollte der Button nicht funktionieren, verwenden Sie diese **RAW-URL** im Blueprint-Import-Dialog:

`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleispeisung-Blueprint-homeassistant/main/solakon_one_nulleinspeisung.yaml`

---

### 💡 Wichtiger Schritt: Erstellung des Entladezyklus-Helfers

Dieser Blueprint erfordert einen **Dropdown-Menü-Helfer (`input_select`)**, um den Zustand des aggressiven Entladezyklus zu speichern.

#### Vorgehensweise zur Erstellung:

1.  Gehen Sie in Home Assistant zu: **Einstellungen** → **Geräte & Dienste** → Reiter **Helfer**.
2.  Klicken Sie auf **Helfer erstellen** und wählen Sie den Typ **Dropdown-Menü (`Input Select`)**.
3.  Geben Sie ihm einen Namen, z. B. `Entladezyklus Zustand`.
4.  Fügen Sie exakt die folgenden beiden Optionen hinzu:
    * `on`
    * `off`
5.  Speichern Sie den Helfer. Die resultierende Entität (z. B. `input_select.entladezyklus_zustand`) muss im Blueprint unter **Entladezyklus-Zustandsspeicher** ausgewählt werden.

---

### 🧠 Detaillierte Funktionsbeschreibung: Dynamische Nulleinspeisung mit SOC-Zonen-Logik

Der Blueprint steuert den **AC-Ausgangsleistungsregler** (`AC-Output-Limit`) des Solakon ONE, um eine präzise Nulleinspeisung zu erreichen, deren Verhalten durch drei vordefinierte SOC-Zonen gesteuert wird:

| Zone | Modus | Ziel und Funktion | P-Regler Parameter |
| :--- | :--- | :--- | :--- |
| **1. Schnelle Regelung / Entladezyklus** | `INV Discharge (PV Priority)` | **Aggressive Entladung** und exakte Nulleinspeisung (SOC > Obere Schwelle). | **Nullpunkt-Offset:** `0 W` (Strikte Nulleinspeisung). |
| **2. Batterieschonende Regelung** | `INV Discharge (PV Priority)` | **Aktive Entladung verhindern** und **PV-Überschuss speichern** (Zwischen den SOC-Schwellen). | **Nullpunkt-Offset:** Negativ (`nulleinspeisung_offset`). **Logik:** Begrenzung auf `min(PV-Leistung, Netzüberschuss)`. |
| **3. Sicherheitsstopp** | `Disabled` | **Sofortiges Stoppen** jeglicher Entladung (SOC < untere Schwelle). | Setzt Modus auf `Disabled` und Limit auf 0 W. |

---

### ⚙️ Erforderliche Entitäten und Konfigurationsparameter

#### I. Erforderliche Entitäten (Sensoren und Steuerungen)

| Parameter Name | Typ | Beschreibung |
| :--- | :--- | :--- |
| **Netz-Leistungssensor** | `sensor` (power) | Der Sensor, der die **Netzleistung** misst (z.B. Shelly 3EM). **Wichtig: Positiv = Einspeisung, Negativ = Bezug.** |
| **Solakon - Solarleistung (PV)** | `sensor` (power) | Der Sensor, der die aktuelle **Solarerzeugung** in Watt anzeigt. |
| **Solakon - Batterieladestand (SOC)** | `sensor` | Der **SOC-Sensor** des Solakon ONE (%). |
| **Solakon - Ausgangsleistungsregler** | `number` | Die Entität zur Steuerung des **AC-Output-Limits**. |
| **Solakon - Betriebsmodus-Auswahl** | `select` | Die Entität zur Steuerung des **Betriebsmodus**. |
| **Modus-Reset-Timer-Entität** | `number` | Die Solakon ONE Entität, die die Modus-Haltezeit (in Sekunden, max. 3600) steuert. |
| **Entladezyklus-Zustandsspeicher** | `input_select` | Ein Helfer mit Optionen `on` und `off` zur Speicherung des Zyklus-Status. |

#### II. Konfigurationsparameter (Number Inputs)

| Parameter Name | Standardwert | Erklärung der Funktion |
| :--- | :--- | :--- |
| **`soc_fast_limit`** | 50% | Die **Obere SOC-Schwelle**. Überschreiten = Start **aggressiver Entladezyklus**. |
| **`soc_conservation_limit`** | 20% | Die **Untere SOC-Schwelle**. Unterschreiten = Start **Sicherheitsstopp** (`Disabled`). |
| **`nulleinspeisung_toleranz`** | 50 W | Die **Toleranzgrenze (Halbbreite)**. Korrektur erfolgt nur außerhalb dieses Bereichs. |
| **`anpassungs_faktor`** | 1.5 | Der **Regelungs-Faktor**. Definiert die **Aggressivität** der Limit-Anpassung in Zone 1. |
| **`nulleinspeisung_offset`** | -30 W | Der **Negative Nullpunkt-Offset** für Zone 2. Erlaubt einen leichten Netzbezug, um die **aktive Entladung zu verhindern**. |
