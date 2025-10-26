# Solakon ONE Nulleinspeisung Steuerung ⚡

Dieser Home Assistant Blueprint implementiert eine dynamische, dreistufige State of Charge (SOC)-Logik und einen Proportional-Regler (P-Regler), um die AC-Ausgangsleistungsbegrenzung (AC Output Limit) des Solakon ONE Wechselrichters zu steuern. Das Ziel ist die **präzise, dynamische Nulleinspeisung** bei gleichzeitiger batterieschonender Priorisierung des Ladevorgangs im mittleren SOC-Bereich.

---

## 🚀 Kernfunktionen

* **Dynamische Nulleinspeisung (P-Regler):** Erreicht eine präzise Nulleinspeisung, indem die AC-Ausgangsleistung kontinuierlich basierend auf der Echtzeit-Netzleistung angepasst wird.
* **Dreistufige SOC-Logik:** Implementiert drei unterschiedliche Zonen für das Batteriemanagement:
    1.  **Schnelle Entladung (SOC Hoch, z. B. >50%):** Aggressiver P-Regler mit einem 0-W-Ziel, um einen hohen SOC schnell zu reduzieren.
    2.  **Lade-Priorität (SOC Mittel, z. B. 20-50%):** Verwendet einen einstellbaren **negativen Offset** (Standard: -30 W) um leichten Netzbezug zu erzwingen und so die Batterieladung zu priorisieren, wenn der SOC moderat ist.
    3.  **Sicherheitsstopp (SOC Niedrig, z. B. <20%):** Deaktiviert die Entladung vollständig, um die Batterie zu schonen.
* **Remote-Timeout-Reset:** Setzt den Timeout des Solakon ONE für die Fernsteuerung automatisch zurück, um eine unterbrechungsfreie Regelung zu gewährleisten.

---

## 📥 Blueprint Importieren

Du kannst diesen Blueprint direkt über den folgenden Button in deine Home Assistant Instanz importieren:

[![Öffne deine Home Assistant Instanz und zeige den Blueprint-Import-Dialog mit einem bestimmten, vorausgefüllten Blueprint an.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

**Manueller Link:**
`https://raw.githubusercontent.com/D4nte85/Solakon-One-Nulleinspeisung-Blueprint-homeassistant/main/solakon_one_nulleinspeisung.yaml`

---

## 🛠️ Voraussetzungen (Helfer erstellen)

Bevor du den Blueprint verwenden kannst, musst du **einen Helfer** erstellen, um den Status des Entladezyklus zu speichern.

| Helfer-Name | Typ | Optionen | Zweck |
| :--- | :--- | :--- | :--- |
| `input_select.solakon_entladezyklus_aktiv` | **Dropdown (Input Select)** | `on`, `off` | Speichert, ob der aggressive Entladezyklus aktiv (`on`) oder inaktiv (`off`) ist. |

### **So erstellst du den Helfer:**

1.  Navigiere in Home Assistant zu **Einstellungen** -> **Geräte & Dienste** -> **Helfer**.
2.  Klicke auf **"Helfer erstellen"**.
3.  Wähle **"Dropdown"** (Input Select).
4.  **Name:** `Solakon Entladezyklus Aktiv`
5.  **Optionen:** Gib die beiden Optionen `on` und `off` ein (jeweils in einer neuen Zeile).
6.  Verwende diese neue Entität (`input_select.solakon_entladezyklus_aktiv`) für den Blueprint-Input **"Entladezyklus-Zustandsspeicher"**.

---

## 🧠 Template-Variablen und Standardeinstellungen

Mit den folgenden Variablen kannst du die Regelungslogik an deine Batterie und dein Nutzungsverhalten anpassen.

| Variablenname | Beschreibung | Standardwert | Einheit |
| :--- | :--- | :--- | :--- |
| **SOC-Schwelle "Schnelle Regelung"** | Der obere SOC-Wert. Bei Überschreitung startet der aggressive Entladezyklus. | **50** | `%` |
| **SOC-Schwelle "Lade-Priorität"** | Der untere SOC-Wert. Bei Unterschreitung wird auf "Disabled" (Sicherheitsstopp) umgeschaltet. | **20** | `%` |
| **Toleranzbereich (Halbbreite)** | Der zulässige Leistungsbereich um den Nullpunkt, bevor der P-Regler eine Korrektur vornimmt. | **50** | `W` |
| **Regelungs-Faktor (Korrektur-Geschw.)** | Definiert die Aggressivität/Geschwindigkeit der Reaktion des P-Reglers. | **1.5** | (Faktor) |
| **Nullpunkt-Offset** | Negativer Watt-Wert, der **nur** in der konservativen SOC-Mittelzone (Zone 2) verwendet wird, um leichten Netzbezug zu erzwingen. | **-30** | `W` |

---

## 📥 Blueprint-Inputs (Entitäten)

Du musst die folgenden Entitäten aus deiner Solakon ONE Integration und deinem Energiemessgerät (z. B. Shelly 3EM) verknüpfen.

| Input-Name | Erforderlicher Entitätstyp | Standard-Entitätsbeispiel |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | Sensor (`device_class: power`) | `sensor.shelly_3em_channel_a_power` |
| **Solakon ONE - Solarleistung** | Sensor | `sensor.solakon_one_pv_power` |
| **Solakon ONE - Batterieladestand (SOC)** | Sensor | `sensor.solakon_one_battery_soc` |
| **Solakon ONE - Ausgangsleistungsregler**| Number | `number.solakon_one_remote_active_power` |
| **Solakon ONE - Betriebsmodus-Auswahl**| Select | `select.solakon_one_remote_mode` |
| **Modus-Reset-Timer-Entität (Setter)** | Number | `number.solakon_one_remote_timeout_control` |
| **Remote Timeout Countdown Sensor (Ausleser)** | Sensor | `sensor.solakon_one_remote_mode_countdown` |
| **Entladezyklus-Zustandsspeicher** | Input Select | (Siehe Helfer oben) | Der von dir erstellte Helfer zur Verfolgung des Entladezyklus. |
