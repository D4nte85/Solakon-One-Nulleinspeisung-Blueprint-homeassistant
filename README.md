# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE)

Dieses Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem Proportional-Regler (P-Regler) und einer intelligenten **dreistufigen Batterieladestands-Logik (SOC)**.

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

---

### 🛠️ Vorbereitung: Erstellung des Input Select Helfers

Der Blueprint benötigt einen `Input Select` Helfer, um den Status des Entladezyklus zu speichern.

1.  Gehen Sie in Home Assistant zu **Einstellungen** (Settings) -> **Geräte & Dienste** (Devices & Services) -> **Helfer** (Helpers).
2.  Klicken Sie auf **Helfer erstellen** (Create Helper).
3.  Wählen Sie den Typ **Auswahl** (**Input Select**).
4.  Geben Sie einen Namen ein, z.B. `SOC Entladezyklus Status`.
5.  Fügen Sie unter **Optionen** (Options) genau diese beiden Werte hinzu:
    * `on`
    * `off`
6.  Speichern Sie den Helfer. Die resultierende Entität (z.B. `input_select.soc_entladezyklus_status`) muss dann im Blueprint unter **Entladezyklus-Zustandsspeicher** ausgewählt werden.

---

## 🧠 Kernfunktionalität

### 1. Proportional-Regler (P-Regler)
* **Messprinzip:** Die Steuerung nutzt die Entität des **Netz-Leistungssensors** (z.B. Shelly 3EM) als Regelabweichung.
* **Korrektur:** Der Regler passt die Ausgangsleistung des Solakon ONE an, um die Netzleistung in den Toleranzbereich zu bringen.
    * Positive Netzleistung (Einspeisung) $\rightarrow$ Erhöhung der Ausgangsleistung.
    * Negative Netzleistung (Bezug) $\rightarrow$ Verringerung der Ausgangsleistung.
* **Steuerung:** Die Aggressivität der Reaktion wird über den **Anpassungs-Faktor** gesteuert. Die Leistung wird auf max. `max_active_power_limit` begrenzt, mindestens jedoch auf `0 W`.

---

### 2. 🔋 Dreistufige SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in drei Betriebsmodi unterteilt:

| Zone | SOC-Bereich | Modus | Ziel & Regelung |
| :--- | :--- | :--- | :--- |
| **1. Schnell-Entladung** | SOC > Obere Schwelle (z.B. 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Regelung** mit 0 W-Offset für exakte Nulleinspeisung. Ein aktiver Entladezyklus-Helfer hält diesen Zustand bis zum Unterschreiten der unteren Schwelle. |
| **2. Batterieschonend** | Untere Schwelle (z.B. 20%) < SOC $\le$ Obere Schwelle | `INV Discharge (PV Priority)` | **Lade-Priorität** durch einen **negativen Nullpunkt-Offset** (z.B. -30 W), der einen leichten Netzbezug erzwingt. Steuerung ohne P-Regler (Schwellwert-basiert). Die Entladeleistung wird zusätzlich um eine **PV-Ladereserve** reduziert, um die Ladung zu sichern. |
| **3. Sicherheitsstopp** | SOC $\le$ Untere Schwelle (z.B. 20%) | `Disabled` | Ausgangsleistung wird sofort auf **0 W** gesetzt, um die Batterie zu schonen. Der Entladezyklus wird beendet. |

---

### ⏱️ Remote Timeout Reset
Um zu verhindern, dass der Solakon ONE in den `Disabled`-Modus wechselt, da er keine Steuerung mehr erhält, wird der interne **Remote-Timeout-Timer** in den aktiven Zonen (1 und 2) proaktiv auf einen hohen Wert (3599s) zurückgesetzt, sobald er einen kritischen Wert (120s) unterschreitet.

## ⚙️ Input-Variablen und Standard-Entitäten

### 🔌 Erforderliche Entitäten (Solakon ONE & Shelly/Smart Meter)

| Variable | Standard-Entität | Beschreibung |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | *(kein Standard)* | Sensor für die aktuelle Netzleistung. **Positive Werte = Bezug**, **Negative Werte = Einspeisung**. |
| **Solakon ONE - Solarleistung** | `sensor.solakon_one_pv_power` | Aktuelle PV-Erzeugung in Watt. |
| **Solakon ONE - Batterieladestand** | `sensor.solakon_one_battery_soc` | Batterieladestand (State of Charge) in %. |
| **Solakon ONE - Ausgangsleistungsregler** | `number.solakon_one_remote_active_power` | Entität zum Setzen des Leistungs-Sollwerts. |
| **Solakon ONE - Betriebsmodus-Auswahl** | `select.solakon_one_remote_mode` | Entität zum Umschalten des Betriebsmodus. |
| **Modus-Reset-Timer-Entität (Setter)** | `number.solakon_one_remote_timeout_control` | Dient zum Setzen/Zurücksetzen des Remote-Timeouts (auf 3599 s). |
| **Remote Timeout Countdown Sensor (Ausleser)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor, der den verbleibenden Timeout-Countdown anzeigt. |
| **Entladezyklus-Zustandsspeicher** | *(siehe oben)* | Der erstellte `Input Select` Helfer (`on`/`off`). |

### 🎚️ Konfigurationsparameter (Einstellwerte)

| Parameter | Standardwert | Beschreibung |
| :--- | :--- | :--- |
| **SOC-Schwelle "Schnelle Regelung"** | `50 %` | Obere Schwelle. Überschreiten startet den aggressiven Entladezyklus (Zone 1). |
| **SOC-Schwelle "Lade-Priorität"** | `20 %` | Untere Schwelle. Unterschreiten stoppt die Entladung (Zone 3). |
| **Toleranzbereich (Halbbreite)** | `50 W` | Der zulässige Bereich in Watt um den Nullpunkt, bevor eine Korrektur vorgenommen wird. |
| **Regelungs-Faktor** | `1.5` | Definiert die Aggressivität des P-Reglers in Zone 1. |
| **Nullpunkt-Offset** | `-30 W` | Der Zielwert für die Netzleistung in Zone 2. Negativer Wert erzwingt leichten Netzbezug. |
| **🔋 PV-Ladereserve** | `15 W` | PV-Leistung, die in Zone 2 reserviert wird, um die Batterie trotz Entladung zu laden. |
| **Maximale Ausgangsleistung (Hard Limit)**| `800 W` | **Die maximale AC-Ausgangsleistung, die das Blueprint setzen darf.** Dient zur Einhaltung der Hardware-Parameter und ermöglicht eine zusätzliche Drosselung der Leistung (z.B. auf 600 W), selbst wenn das Gerät mehr könnte. |
