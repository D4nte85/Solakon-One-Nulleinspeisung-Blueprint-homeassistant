# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE)

Dieses Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem Proportional-Regler (P-Regler) und einer intelligenten **dreistufigen Batterieladestands-Logik (SOC)**.

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fexperimental_battery_discharge_limit%2Fsolakon_one_nulleinspeisung.yaml)

---

### 🛠️ Vorbereitung: Erstellung des Input Select Helfers

Der Blueprint benötigt einen `Input Select` Helfer, um den Status des Entladezyklus zu speichern.

1.  Gehen Sie in Home Assistant zu **Einstellungen** (Settings) -> **Geräte & Dienste** (Devices & Services) -> **Helfer** (Helpers).
2.  Klicken Sie auf **Helfer erstellen** (Create Helper).
3.  Wählen Sie den Typ **Auswahl** (**Input Select**).
4.  Geben Sie einen Namen ein, z.B. `SOC Entladezyklus Status`.
5.  Fügen Sie unter **Optionen** (Options) genau diese beiden Werte hinzu:
    * `on`
    * `off`
6.  Speichern Sie den Helfer. Die resultierende Entität (z.B. `input_select.soc_entladezyklus_status`) muss dann im Blueprint unter **Entladezyklus-Zustandsspeicher** ausgewählt werden.

---

## 🧠 Kernfunktionalität

### 1. Proportional-Regler (P-Regler)
* **Messprinzip:** Die Steuerung nutzt die Entität des **Netz-Leistungssensors** (z.B. Shelly 3EM) als Regelabweichung.
* **Korrektur:** Der Regler passt die Ausgangsleistung des Solakon ONE an, um die Netzleistung in den Toleranzbereich zu bringen.
    * **Positive Netzleistung (Bezug)** $\rightarrow$ Erhöhung der Ausgangsleistung.
    * **Negative Netzleistung (Einspeisung)** $\rightarrow$ Verringerung der Ausgangsleistung.
* **Steuerung:** Die Aggressivität der Reaktion wird über den **Anpassungs-Faktor** gesteuert. Die Leistung wird auf max. `max_active_power_limit` begrenzt, mindestens jedoch auf `0 W`.

---

### 2. 🔋 Dreistufige SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in drei Betriebsmodi unterteilt:

| Zone | SOC-Bereich | Modus | Ziel & Regelung (Aktualisierte Logik V166) |
| :--- | :--- | :--- | :--- |
| **1. Schnell-Entladung** | SOC > Obere Schwelle (z.B. 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Regelung** mit 0 W-Offset für exakte Nulleinspeisung. Ein aktiver Entladezyklus-Helfer hält diesen Zustand bis zum Unterschreiten der unteren Schwelle. |
| **2. Batterieschonend** | Untere Schwelle (z.B. 20%) < SOC $\le$ Obere Schwelle | `INV Discharge (PV Priority)` | **Aktion:** Entladestrom auf **0 A** limitiert (keine Batterieentnahme). **Aktive P-Regelung** mit **negativem Nullpunkt-Offset** (z.B. -30 W) für leichten Netzbezug. Die **Ausgangsleistung** wird dynamisch durch (**PV-Erzeugung minus PV-Ladereserve**) begrenzt, um die Reserve für die Ladung zu sichern. |
| **3. Sicherheitsstopp** | SOC $\le$ Untere Schwelle (z.B. 20%) | `Disabled` | Ausgangsleistung wird sofort auf **0 W** gesetzt, um die Batterie zu schonen. Der Entladezyklus wird beendet. |

---

### ⏱️ Remote Timeout Reset und Moduswechsel-Sequenz

Um die Stabilität der Kommunikation mit dem Solakon ONE zu gewährleisten, werden zwei Timer-Mechanismen genutzt:

1.  **Kontinuierlicher Timeout-Reset (Refresh):** Der interne **Remote-Timeout-Timer** wird in den aktiven Zonen (1 und 2) proaktiv auf einen hohen Wert (3599s) zurückgesetzt, sobald er einen kritischen Wert (**120s**) unterschreitet.
2.  **Forcierter Reset vor Moduswechsel (Puls-Sequenz):** Bei jedem **kritischen Moduswechsel** (`Disabled` $\leftrightarrow$ `INV Discharge (PV Priority)`) wird zusätzlich eine **zweistufige Puls-Sequenz** (`1s` Puls, dann `3599s` Setzen) ausgeführt. Dies stellt sicher, dass der Solakon den folgenden Modusbefehl zuverlässig annimmt und Timeout-Fehler verhindert werden.

---

### 🚦 Trigger-Bedingungen (Automatisierungs-Auslöser)

Die Automatisierung reagiert auf folgende fünf kritische Ereignisse, um eine sofortige und stabile Regelung zu gewährleisten:

1.  **Leistungsänderungen (mit 3s Verzögerung):**
    * Zustandsänderung des **Netz-Leistungssensors** (`shelly_grid_power_sensor`) für $\ge 3$ Sekunden.
    * Zustandsänderung des **Solarleistungssensors** (`solakon_solar_power_sensor`) für $\ge 3$ Sekunden.
    * *(Zweck: Löst die stabile P-Regelung aus.)*
2.  **SOC-Schwellwert-Erreichung:**
    * Batterie-SOC (`solakon_soc_sensor`) **über** der **Oberen Schwelle** (`soc_fast_limit`).
    * Batterie-SOC **unter** der **Unteren Schwelle** (`soc_conservation_limit`).
    * *(Zweck: Steuert den Wechsel zwischen den Entladezonen.)*
3.  **Moduswechsel:**
    * Zustandsänderung der **Betriebsmodus-Auswahl** (`solakon_mode_select`).
    * *(Zweck: Reagiert auf manuelle oder externe Modusänderungen.)*

---

## ⚙️ Input-Variablen und Standard-Entitäten

> **Wichtig:** Die **Standard-Entitäten** (`default`) wurden an die gängigen Bezeichnungen der Solakon ONE Home Assistant Integration angepasst und in der Beschreibung hervorgehoben. Passen Sie die Werte bei der Installation an, falls Ihre Entitätsnamen abweichen.

### 🔌 Erforderliche Entitäten (Solakon ONE & Shelly/Smart Meter)

| Variable | Standard-Entität | Beschreibung |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | *(kein Standard)* | Sensor für die aktuelle Netzleistung (z.B. Shelly 3EM). **Positive Werte = Bezug**, **Negative Werte = Einspeisung**. |
| **Solakon ONE - Solarleistung** | `sensor.solakon_one_total_pv_power` | Aktuelle PV-Erzeugung in Watt. |
| **Solakon ONE - Batterieladestand** | `sensor.solakon_one_battery_soc` | Batterieladestand (State of Charge) in %. |
| **Solakon ONE - Ausgangsleistungsregler** | `number.solakon_one_remote_active_power` | Entität zum Setzen des Leistungs-Sollwerts. |
| **Solakon ONE - Betriebsmodus-Auswahl** | `select.solakon_one_remote_control_mode` | Entität zum Umschalten des Betriebsmodus. |
| **Modus-Reset-Timer-Entität (Setter)** | `number.solakon_one_remote_timeout_set` | Dient zum Setzen/Zurücksetzen des Remote-Timeouts (max. 3599 s). |
| **Remote Timeout Countdown Sensor (Ausleser)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor, der den verbleibenden Timeout-Countdown anzeigt. |
| **Entladezyklus-Zustandsspeicher** | `input_select.soc_entladezyklus_status` | Der erstellte `Input Select` Helfer (`on`/`off`). **Der Standardname wird automatisch eingetragen, muss aber existieren!** |

---

### 🎚️ Konfigurationsparameter (Einstellwerte)

| Parameter | Standardwert | Beschreibung |
| :--- | :--- | :--- |
| **SOC-Schwelle "Schnelle Regelung"** | `50 %` | Obere Schwelle. Überschreiten startet den aggressiven Entladezyklus (Zone 1). |
| **SOC-Schwelle "Lade-Priorität"** | `20 %` | Untere Schwelle. Unterschreiten stoppt die Entladung (Zone 3). |
| **Toleranzbereich (Halbbreite)** | `25 W` | Der zulässige Bereich in Watt um den Nullpunkt, bevor eine Korrektur vorgenommen wird. |
| **Regelungs-Faktor** | `1.5` | Definiert die Aggressivität des P-Reglers. |
| **Nullpunkt-Offset** | `-30 W` | Der Zielwert für die Netzleistung in Zone 2. Negativer Wert erzwingt leichten Netzbezug. |
| **🔋 PV-Ladereserve** | `50 W` | Die PV-Leistung (in Watt), die reserviert wird. Dieser Puffer gleicht **interne Wandlerverluste** aus und wird in Zone 2 zur **dynamischen Begrenzung der Ausgangsleistung** genutzt, um die Batterieladung zu gewährleisten. |
| **Maximale Ausgangsleistung (Hard Limit)**| `800 W` | Die maximale AC-Ausgangsleistung, die das Blueprint setzen darf. Dient zur Einhaltung der Hardware-Parameter. |

---

## 🛑 Wichtige Fehlermeldungen (System-Log)

Der Blueprint enthält eine integrierte Validierung, die bei kritischen Fehlern die Automatisierung stoppt und eine klare Meldung in das Home Assistant System-Log schreibt.

| Meldung im Log | Ursache | Lösung |
| :--- | :--- | :--- |
| **Die obere SOC-Schwelle (X%) muss größer sein als die untere SOC-Schwelle (Y%).** | Die Werte für **SOC-Schwelle "Schnelle Regelung"** und **SOC-Schwelle "Lade-Priorität"** sind gleich oder vertauscht. | Stellen Sie sicher, dass die obere Schwelle (z.B. 50) immer höher ist als die untere Schwelle (z.B. 20). |
| **Eine oder mehrere kritische Entitäten sind UNVERFÜGBAR oder haben ungültige Werte.** | Eine der kritischen Entitäten (SOC, Timeout-Sensor, Netzleistung, Ausgangsleistungsregler) ist `unavailable` oder liefert ungültige Daten (z.B. wenn die Solakon-Integration nicht verbunden ist). | Prüfen Sie den Status der Solakon ONE Entitäten und stellen Sie sicher, dass die Integration aktiv und verbunden ist. |
