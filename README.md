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
    * **Positive Netzleistung (Bezug)** $\rightarrow$ Erhöhung der Ausgangsleistung.
    * **Negative Netzleistung (Einspeisung)** $\rightarrow$ Verringerung der Ausgangsleistung.
* **Steuerung:** Die Aggressivität der Reaktion wird über den **Anpassungs-Faktor** gesteuert. Die Leistung wird auf max. `max_active_power_limit` begrenzt, mindestens jedoch auf `0 W`.

---

### 2. 🔋 Dreistufige SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in drei Betriebsmodi unterteilt:

| Zone | SOC-Bereich | Modus | Ziel & Regelung |
| :--- | :--- | :--- | :--- |
| **1. Schnell-Entladung** | SOC > Obere Schwelle (z.B. 50%) | `INV Discharge (PV Priority)` | **Aggressive P-Regelung** mit 0 W-Offset für exakte Nulleinspeisung. Ein aktiver Entladezyklus-Helfer hält diesen Zustand bis zum Unterschreiten der unteren Schwelle. |
| **2. Batterieschonend** | Untere Schwelle (z.B. 20%) < SOC $\le$ Obere Schwelle | `INV Discharge (PV Priority)` (nur bei PV > Reserve) | **Aktive P-Regelung** mit **positivem Nullpunkt-Offset** (z.B. +30 W) für leichten Netzbezug (Ladepriorität). Die **Ausgangsleistung** wird dynamisch durch (**PV-Erzeugung minus PV-Ladereserve**) begrenzt, um die Reserve für die Ladung zu sichern. **START:** Nur wenn PV > Reserve. **STOPP:** Automatisch wenn PV ≤ Reserve. |
| **3. Sicherheitsstopp** | SOC $\le$ Untere Schwelle (z.B. 20%) | `Disabled` | Ausgangsleistung wird sofort auf **0 W** gesetzt, um die Batterie zu schonen. Der Entladezyklus wird beendet. |

---

### ⏱️ Remote Timeout Reset und Moduswechsel-Sequenz

Um die Stabilität der Kommunikation mit dem Solakon ONE zu gewährleisten, werden zwei Timer-Mechanismen genutzt:

1.  **Kontinuierlicher Timeout-Reset (Refresh):** Der interne **Remote-Timeout-Timer** wird in den aktiven Zonen (1 und 2) proaktiv auf einen hohen Wert (3599s) zurückgesetzt, sobald er einen kritischen Wert (**120s**) unterschreitet.
2.  **Forcierter Reset vor Moduswechsel (Puls-Sequenz):** Bei jedem **kritischen Moduswechsel** (`Disabled` $\leftrightarrow$ `INV Discharge (PV Priority)`) wird zusätzlich eine **zweistufige Puls-Sequenz** (`1s` Puls, dann `3599s` Setzen) ausgeführt. Dies stellt sicher, dass der Solakon den folgenden Modusbefehl zuverlässig annimmt und Timeout-Fehler verhindert werden.

---

### 🚦 Trigger-Bedingungen (Automatisierungs-Auslöser)

Die Automatisierung reagiert auf folgende kritische Ereignisse, um eine sofortige und stabile Regelung zu gewährleisten:

1.  **Leistungsänderungen (mit 3s Verzögerung):**
    * Zustandsänderung des **Netz-Leistungssensors** (`shelly_grid_power_sensor`) für $\ge 3$ Sekunden.
    * Zustandsänderung des **Solarleistungssensors** (`solakon_solar_power_sensor`) für $\ge 3$ Sekunden.
    * *(Zweck: Löst die stabile P-Regelung aus.)*
2.  **SOC-Schwellwert-Erreichung:**
    * Batterie-SOC (`solakon_soc_sensor`) **über** der **Oberen Schwelle** (`soc_fast_limit`).
    * Batterie-SOC **unter** der **Unteren Schwelle** (`soc_conservation_limit`).
    * *(Zweck: Steuert den Wechsel zwischen den Entladezonen.)*
3.  **Moduswechsel:**
    * Zustandsänderung der **Betriebsmodus-Auswahl** (`solakon_mode_select`).
    * *(Zweck: Reagiert auf manuelle oder externe Modusänderungen.)*
4.  **Timeout-Countdown kritisch:**
    * Der **Remote Timeout Countdown** (`solakon_timeout_countdown_sensor`) fällt unter **150 Sekunden**.
    * *(Zweck: Ermöglicht proaktives Timeout-Management.)*

---

## ⚙️ Input-Variablen und Standard-Entitäten

> **Wichtig:** Die **Standard-Entitäten** (`default`) wurden an die gängigen Bezeichnungen der Solakon ONE Home Assistant Integration angepasst und in der Beschreibung hervorgehoben. Passen Sie die Werte bei der Installation an, falls Ihre Entitätsnamen abweichen.

### 🔌 Erforderliche Entitäten (Solakon ONE & Shelly/Smart Meter)

| Variable | Standard-Entität | Beschreibung |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | *(kein Standard)* | Sensor für die aktuelle Netzleistung (z.B. Shelly 3EM). **Positive Werte = Bezug**, **Negative Werte = Einspeisung**. Muss `device_class: power` haben. |
| **Solakon ONE - Solarleistung** | `sensor.solakon_one_total_pv_power` | Aktuelle PV-Erzeugung in Watt. |
| **Solakon ONE - Batterieladestand** | `sensor.solakon_one_battery_soc` | Batterieladestand (State of Charge) in %. |
| **Solakon ONE - Ausgangsleistungsregler** | `number.solakon_one_remote_active_power` | Entität zum Setzen des Leistungs-Sollwerts. |
| **Solakon ONE - Betriebsmodus-Auswahl** | `select.solakon_one_remote_control_mode` | Entität zum Umschalten des Betriebsmodus. |
| **Modus-Reset-Timer-Entität (Setter)** | `number.solakon_one_remote_timeout_set` | Dient zum Setzen/Zurücksetzen des Remote-Timeouts (max. 3600 s). |
| **Remote Timeout Countdown Sensor (Ausleser)** | `sensor.solakon_one_remote_timeout_countdown` | Sensor oder Number-Entität, die den verbleibenden Timeout-Countdown anzeigt. |
| **Entladezyklus-Zustandsspeicher** | `input_select.soc_entladezyklus_status` | Der erstellte `Input Select` Helfer (`on`/`off`). **Der Standardname wird automatisch eingetragen, muss aber existieren!** |

---

### 🎚️ Konfigurationsparameter (Einstellwerte)

| Parameter | Standardwert | Beschreibung |
| :--- | :--- | :--- |
| **SOC-Schwelle "Schnelle Regelung"** | `50 %` | Obere Schwelle. Überschreiten startet den aggressiven Entladezyklus (Zone 1). Min: 21%, Max: 100% |
| **SOC-Schwelle "Lade-Priorität"** | `20 %` | Untere Schwelle. Unterschreiten stoppt die Entladung (Zone 3). Min: 0%, Max: 49% |
| **Toleranzbereich (Halbbreite)** | `25 W` | Der zulässige Bereich in Watt um den Nullpunkt, bevor eine Korrektur vorgenommen wird. Min: 10W, Max: 200W |
| **Regelungs-Faktor** | `1.5` | Definiert die Aggressivität des P-Reglers. Min: 0.5, Max: 3.0 |
| **Nullpunkt-Offset (Netzbezug)** | `30 W` | Der Zielwert für die Netzleistung in Zone 2. Positiver Wert verschiebt Regelziel in Richtung Netzbezug (Ladepriorität). Min: 0W, Max: 100W |
| **🔋 PV-Ladereserve** | `50 W` | Die PV-Leistung (in Watt), die reserviert wird. Dieser Puffer gleicht **interne Wandlerverluste** aus und wird in Zone 2 zur **dynamischen Begrenzung der Ausgangsleistung** genutzt, um die Batterieladung zu gewährleisten. Min: 0W, Max: 500W |
| **Maximale Ausgangsleistung (Hard Limit)**| `800 W` | Die maximale AC-Ausgangsleistung, die das Blueprint setzen darf. Dient zur Einhaltung der Hardware-Parameter. Min: 0W, Max: 1200W |

---

## 🛑 Wichtige Fehlermeldungen (System-Log)

Der Blueprint enthält eine integrierte Validierung, die bei kritischen Fehlern die Automatisierung stoppt und eine klare Meldung in das Home Assistant System-Log schreibt (`Logger: automation.solakon_zero_export`).

| Meldung im Log | Ursache | Lösung |
| :--- | :--- | :--- |
| **Der obere SOC-Schwellenwert (X%) muss größer sein als der untere SOC-Schwellenwert (Y%).** | Die Werte für **SOC-Schwelle "Schnelle Regelung"** und **SOC-Schwelle "Lade-Priorität"** sind gleich oder vertauscht. | Stellen Sie sicher, dass die obere Schwelle (z.B. 50) immer höher ist als die untere Schwelle (z.B. 20). |
| **SOC-Sensor (entity_id) ist UNKNOWN/UNAVAILABLE oder hat ungültige Werte.** | Der SOC-Sensor liefert keine gültigen Daten. | Prüfen Sie die Solakon ONE Integration und Verbindung. |
| **Remote Timeout Countdown Sensor (entity_id) ist UNKNOWN/UNAVAILABLE oder hat ungültige Werte.** | Der Countdown-Sensor ist nicht verfügbar. | Prüfen Sie, ob die Entität existiert und die Integration verbunden ist. |
| **Netzleistungs-Sensor (entity_id) ist UNKNOWN/UNAVAILABLE.** | Der Shelly/Netzleistungssensor liefert keine Daten. | Prüfen Sie die Shelly-Verbindung und Konfiguration. |
| **Ausgangsleistungsregler (entity_id) ist UNKNOWN/UNAVAILABLE.** | Die Number-Entität zur Leistungsregelung ist nicht verfügbar. | Prüfen Sie die Solakon ONE Integration. |
| **Betriebsmodus-Auswahl (entity_id) ist UNKNOWN/UNAVAILABLE.** | Die Select-Entität für den Betriebsmodus ist nicht verfügbar. | Prüfen Sie die Solakon ONE Integration. |

---

## 📊 Funktionsweise im Detail

### Zone 1: Schnell-Entladung (SOC > 50%)
- **Aktivierung:** Automatisch beim Überschreiten der oberen SOC-Schwelle
- **Modus:** `INV Discharge (PV Priority)`
- **Regelung:** P-Regler mit 0W-Offset (exakte Nulleinspeisung)
- **Entladezyklus-Helfer:** Wird auf `on` gesetzt
- **Deaktivierung:** Erst beim Unterschreiten der unteren SOC-Schwelle (20%)

### Zone 2: Batterieschonend (20% < SOC ≤ 50%)
- **Aktivierung:** Nur wenn PV-Leistung > PV-Ladereserve
- **Modus:** `INV Discharge (PV Priority)` (bei ausreichend PV)
- **Regelung:** P-Regler mit +30W-Offset (leichter Netzbezug = Ladepriorität)
- **Leistungslimit:** Dynamisch begrenzt auf max(PV-Leistung - PV-Reserve, 0)
- **Entladezyklus-Helfer:** Bleibt auf `off`
- **Automatischer Stopp:** Bei PV-Leistung ≤ PV-Reserve → Modus auf `Disabled`

### Zone 3: Sicherheitsstopp (SOC ≤ 20%)
- **Aktivierung:** Automatisch beim Unterschreiten der unteren SOC-Schwelle
- **Modus:** `Disabled`
- **Aktion:** Ausgangsleistung = 0W
- **Entladezyklus-Helfer:** Wird auf `off` gesetzt
- **Batterieschutz:** Keine Entladung mehr möglich

---

## 🔧 Empfohlene Einstellungen

### Für konservative Batterienutzung:
- SOC-Schwelle "Schnelle Regelung": 60%
- SOC-Schwelle "Lade-Priorität": 30%
- Nullpunkt-Offset: 50W
- PV-Ladereserve: 100W

### Für maximale Eigenverbrauchsoptimierung:
- SOC-Schwelle "Schnelle Regelung": 40%
- SOC-Schwelle "Lade-Priorität": 15%
- Nullpunkt-Offset: 20W
- PV-Ladereserve: 30W

### Für ausgewogenen Betrieb (Standard):
- SOC-Schwelle "Schnelle Regelung": 50%
- SOC-Schwelle "Lade-Priorität": 20%
- Nullpunkt-Offset: 30W
- PV-Ladereserve: 50W

---

## ⚠️ Wichtige Hinweise

1. **Entladezyklus-Helfer:** Muss vor der Installation des Blueprints erstellt werden
2. **Solakon ONE Integration:** Muss vollständig eingerichtet und verbunden sein
3. **Netzleistungssensor:** Muss korrekt kalibriert sein (positiv = Bezug, negativ = Einspeisung)
4. **Timeout-Management:** Der Blueprint setzt den Timeout automatisch zurück - keine manuelle Intervention nötig
5. **Queued-Modus:** Der Blueprint läuft im `queued`-Modus, d.h. Trigger werden sequentiell abgearbeitet
