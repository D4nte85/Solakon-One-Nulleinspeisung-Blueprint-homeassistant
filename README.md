# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V207

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku**.

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie. Dies verhindert das "Flackern", das die App mit ihrer (lade ein Prozent → entlade ein Prozent → repeat) Funktionsweise verursacht, und schont die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder in der neuesten Version der APP die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

## 🛠️ Vorbereitung: Erstellung der erforderlichen Helper

Der Blueprint benötigt **zwei Helper**, die Sie vor der Installation erstellen müssen:

### 1. Input Select Helper (Entladezyklus-Speicher)

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Auswahl** (Dropdown)
3. Name: z.B. `SOC Entladezyklus Status`
4. Fügen Sie **genau** diese beiden Optionen hinzu:
   * `on`
   * `off`
5. Speichern (Entity ID: z.B. `input_select.soc_entladezyklus_status`)

### 2. Input Number Helper (Integral-Speicher für PI-Regler)

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Number**
3. Name: z.B. `Solakon Integral`
4. **Wichtige Einstellungen:**
   * Minimum: `-1000`
   * Maximum: `1000`
   * Step: `1`
   * Initialwert: `0`
5. Speichern (Entity ID: z.B. `input_number.solakon_integral`)

### 3. Input Number Helper für Dynamischen Offset (Optional)

Wenn Sie den Nullpunkt-Offset zur Laufzeit dynamisch anpassen möchten, empfehlen wir den **Solakon ONE — Dynamischer Offset Blueprint** (siehe Abschnitt Dynamischer Offset). Dieser erstellt und befüllt die benötigten Helper automatisch.

Alternativ können Sie auch manuell einen oder zwei Helper erstellen:

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Number**
3. Name: z.B. `Solakon Offset Zone 1`
4. **Einstellungen:**
   * Minimum: `0`
   * Maximum: `500`
   * Step: `1`
   * Initialwert: `30` (oder Ihr gewünschter Standard-Offset)
5. Speichern (Entity ID: z.B. `input_number.solakon_offset_zone1`)
6. Optional: Wiederholen für Zone 2 (`input_number.solakon_offset_zone2`)


---

## 🧠 Kernfunktionalität

### 1. PI-Regler (Proportional-Integral-Regler)

Der Blueprint nutzt einen **PI-Regler** für präzise Nulleinspeisung:

* **P-Anteil (Proportional):**
  - Reagiert sofort auf aktuelle Abweichungen
  - Konfigurierbare Aggressivität über den **P-Faktor** (z.B. 1.5)
  - Schnelle Reaktion auf Laständerungen

* **I-Anteil (Integral):**
  - Summiert Abweichungen über die Zeit auf
  - Eliminiert **bleibende Regelabweichungen**
  - Konfigurierbare Geschwindigkeit über den **I-Faktor** (z.B. 0.05)
  - **Anti-Windup:** Begrenzt auf ±1000 Punkte
  - **Automatischer Reset:** Bei jedem Zonenwechsel auf 0 zurückgesetzt
  - **Toleranz-Decay:** 5% Abbau pro Zyklus, solange der Grid-Fehler innerhalb der Toleranz liegt und `|Integral| > 10`. Verhindert schleichendes Integral-Windup bei stabiler Regelung.

* **Messprinzip:**
  - Basiert auf dem **Netz-Leistungssensor** (z.B. Shelly 3EM)
  - **Positive Werte = Bezug**, **Negative Werte = Einspeisung**
  - Keine Verzögerung — sofortige Reaktion auf Sensor-Änderungen

* **Fehlerberechnung mit dynamischer Begrenzung:**
  - **Zone 1:** Fehler = Min(verfügbare Kapazität, Grid Power − Offset₁)
  - **Zone 2:** Fehler = Min(verfügbare Kapazität, Grid Power − Offset₂, PV-Kapazität)

* **Leistungsbegrenzung:**
  - Oberes Hard Limit (z.B. 800 W) in Zone 1 und Zone 0
  - Dynamisches PV-basiertes Limit in Zone 2
  - Unteres Limit: fest 0 W

---

### 2. 🔋 SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in bis zu vier Betriebsmodi unterteilt:

| Zone | SOC-Bereich / Bedingung | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:------------------------|:------|:-----------------|:---------|:--------------|
| **1. Aggressive Entladung** | SOC > Zone-1-Schwelle | `INV Discharge (PV Priority)` | Konfigurierter Max-Wert (Standard: 40 A)| 0W + Offset 1 | Läuft **durchgehend bis SOC ≤ Zone-3-Schwelle** (kein Yo-Yo-Effekt). Auch nachts aktiv. Hard Limit. |
| **2. Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `INV Discharge (PV Priority)` | **0 A** | 0W + Offset 2 | Dynamisches Limit: **Max(0, PV − Reserve)**. Optional: Nachtabschaltung möglich. |
| **3. Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `Disabled` | 0 A | — | Ausgang = 0 W. Vollständiger Schutz der Batterie. |

**Wichtig:**
- Zone 1 wird **einmal aktiviert** beim Überschreiten der Zone-1-Schwelle und läuft dann durch bis zur Zone-3-Schwelle
- Zone 2 startet nur wenn Zyklus = off UND Modus = Disabled (z.B. nach Zone-3-Übergang, Erststart oder manueller Deaktivierung)
- Dies verhindert ständiges Hin- und Herwechseln zwischen den Zonen
- **Max. Entladestrom** wird automatisch gesteuert — keine manuelle Einstellung nötig

#### 🔄 Recovery-Mechanismus

Falls der Modus des Wechselrichters extern zurückgesetzt wird (z.B. durch einen Neustart der Integration oder manuelle Änderung), während der Entladezyklus noch aktiv ist (`Zyklus = on`), erkennt der Blueprint diesen Zustand automatisch und reaktiviert den Modus über die Puls-Sequenz (10s → 3599s) — ohne Zonenwechsel oder Integral-Reset. Voraussetzung: SOC liegt noch über der Zone-3-Schwelle.

---

### 4. 🌙 Nachtabschaltung (Optional)

Die Nachtabschaltung kann optional aktiviert werden und betrifft **nur Zone 2**:

* **Aktivierung:** Über den Parameter "Nachtabschaltung aktivieren"
* **Schwelle:** PV-Leistung unter konfiguriertem Wert (z.B. 10 W)
* **Verhalten:**
  - **Zone 0:** Nicht betroffen (setzt SOC-Vollstand voraus)
  - **Zone 1:** Läuft auch nachts weiter (hoher SOC → aggressive Entladung gewünscht)
  - **Zone 2:** Wird bei PV < Schwelle auf `Disabled` gesetzt (Integral wird zurückgesetzt)
  - **Zone 3:** Ohnehin deaktiviert
* **Sinnvoll für:** Batterieschonung bei 0 PV, wenn keine Grundlast vorhanden ist

---

### 5. ⏱️ Remote Timeout Reset und Moduswechsel-Sequenz

Um die Stabilität der Kommunikation mit dem Solakon ONE zu gewährleisten:

1. **Kontinuierlicher Timeout-Reset:**
   - Der Remote-Timeout wird automatisch auf 3599s zurückgesetzt
   - Trigger: Countdown fällt unter 120s

2. **Forcierter Reset bei Moduswechsel:**
   - Zweistufige Puls-Sequenz: 10s → 3599s (mit 1s Verzögerung)
   - Stellt sichere Modusübernahme sicher
   - Gilt bei Zone-1-Start, Zone-2-Start und Recovery

---

## 📊 Input-Variablen und Konfiguration

### 🔌 Erforderliche Entitäten

| Kategorie | Variable | Standard-Entität | Beschreibung |
|:----------|:---------|:----------------|:-------------|
| **Extern** | Netz-Leistungssensor | *(kein Standard)* | Z.B. Shelly 3EM. **Positiv = Bezug, Negativ = Einspeisung** |
| **Solakon** | Solarleistung | `sensor.solakon_one_pv_leistung` | Aktuelle PV-Erzeugung in Watt |
| **Solakon** | Tatsächliche Ausgangsleistung | `sensor.solakon_one_leistung` | Aktuelle AC-Ausgangsleistung des Wechselrichters in Watt |
| **Solakon** | Batterieladestand (SOC) | `sensor.solakon_one_batterie_ladestand` | Ladestand in % |
| **Solakon** | Remote Timeout Countdown | `sensor.solakon_one_fernsteuerung_zeituberschreitung` | Verbleibender Countdown |
| **Solakon** | Ausgangsleistungsregler | `number.solakon_one_fernsteuerung_leistung` | Setzt Leistungs-Sollwert |
| **Solakon** | Max. Entladestrom | `number.solakon_one_maximaler_entladestrom` | Setzt Entladestrom-Limit |
| **Solakon** | Modus-Reset-Timer | `number.solakon_one_fernsteuerung_zeituberschreitung` | Setzt/Reset Timeout (max. 3599s) |
| **Solakon** | Betriebsmodus-Auswahl | `select.solakon_one_modus_fernsteuern` | Schaltet Betriebsmodus |
| **Helper** | Entladezyklus-Speicher | `input_select.soc_entladezyklus_status` | Input Select: `on`/`off` |
| **Helper** | Integral-Speicher | `input_number.solakon_integral` | Input Number: -1000 bis 1000 |

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.5 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver, schneller. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine Korrektur innerhalb dieser Zone. |
| **Wartezeit** | 3 s | 0 | 30 s | Verzögerung nach Leistungsänderung. Gibt Wechselrichter und Sensoren Zeit zum Aktualisieren. |

**Tuning-Tipps:**
- **Zu langsam?** → P-Faktor erhöhen (z.B. 2.0) oder I-Faktor erhöhen (z.B. 0.08)
- **Schwingt/Überschwingt?** → P-Faktor senken (z.B. 1.0) oder I-Faktor senken (z.B. 0.03)
- **Permanente Mini-Korrekturen?** → Toleranzbereich erhöhen (z.B. 35 W)
- **Integral läuft hoch obwohl stabil?** → Toleranz-Decay greift automatisch (5% Abbau pro Zyklus wenn `|Integral| > 10`)

---

### 🔋 SOC-Zonen-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Obere Schwelle. Überschreiten aktiviert aggressive Entladung. |
| **Zone 3 Stopp** | 20 % | 1 % | 49 % | Untere Schwelle. Unterschreiten stoppt Entladung komplett. |
| **Max. Entladestrom Zone 1** | 40 A | 0 A | 40 A | Entladestrom in Zone 1. Zone 2 nutzt automatisch 0 A. |

**Wichtig:** Zone-1-Schwelle muss **größer** als Zone-3-Schwelle sein! Blueprint validiert dies beim Start.

---

### ⚙️ Zone 1/2 - Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Nullpunkt-Offset-1 (Statisch)** | 30 W | 0 | 100 W | Statischer Fallback-Wert für Zone 1. |
| **Nullpunkt-Offset-1 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert wenn konfiguriert. |
| **Nullpunkt-Offset-2 (Statisch)** | 30 W | 0 | 100 W | Statischer Fallback-Wert für Zone 2. |
| **Nullpunkt-Offset-2 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert wenn konfiguriert. |
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Reservierte PV-Leistung für Ladung in Zone 2. Dynamisches Limit: Max(0, PV − Reserve). |

---

## 🎯 Dynamischer Offset Blueprint (Empfohlen)

Der **dynamische Offset** ermöglicht es, den Nullpunkt-Offset **vollautomatisch an die aktuelle Netz-Volatilität anzupassen** — ohne den Nulleinspeisung-Blueprint ändern zu müssen.

Dafür steht ein eigener Blueprint zur Verfügung:

### 👉 [Solakon ONE — Dynamischer Offset (Spike-Filter & Volatilitäts-Regler)](https://github.com/D4nte85/Solakon-one-dynamic-offset-blueprint)

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)

**Was er macht:**
- Filtert Netzspitzen heraus (Spike-Filter mit konfigurierbarer Bestätigungszeit)
- Analysiert die Netz-Volatilität der letzten 60 Sekunden (Standardabweichung)
- Erhöht den Sicherheitspuffer automatisch bei unruhigem Netz (z.B. taktende Kompressoren, Waschmaschinen)
- Schreibt den berechneten Wert direkt in die `input_number`-Helper für Zone 1 & Zone 2

**Typischer Offset je nach Netzsituation:**

| Netz-Zustand | StdDev | Berechneter Offset |
|:------------|:------:|:------------------:|
| Sehr ruhig | 5 W | 30 W *(Minimum)* |
| Normal | 30 W | 53 W |
| Unruhig | 80 W | 128 W |
| Sehr unruhig | 160 W | 228 W |
| Extrem | 250 W+ | 250 W *(Maximum)* |

**Zusammenspiel mit diesem Blueprint:**
1. Dynamic Offset Blueprint berechnet den optimalen Offset und schreibt ihn in `input_number.solakon_offset_zone1` / `_zone2`
2. Dieser Blueprint liest den Wert aus diesen Entitäten über die "Dynamischer Nullpunkt-Offset" Felder
3. Der Offset passt sich so vollautomatisch und ohne manuelle Eingriffe an

> ⚠️ **Abwärtskompatibel:** Wenn keine dynamische Entität ausgewählt wird, funktioniert dieser Blueprint wie bisher mit dem konfigurierten statischen Offset-Wert.

---

**Erklärung PV-Ladereserve:**
- Bei 300W PV-Erzeugung und 50W Reserve → Max. Ausgang in Zone 2: 250W
- Stellt sicher, dass auch bei Trickle-Charge immer etwas für die Batterie übrig bleibt
- Gleicht Wandlerverluste aus

> ⚠️ **Abwärtskompatibel:** Wenn keine dynamische Entität ausgewählt wird, funktioniert dieser Blueprint wie bisher mit dem konfigurierten statischen Offset-Wert.

---

### 🔒 Sicherheits-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Max. Ausgangsleistung** | 800 W | 0 | 1200 W | Absolute Obergrenze (Hard Limit) in Zone 0 und Zone 1. |

---

### 🌙 Nachtabschaltung (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **Nachtabschaltung aktivieren** | false | Ein/Aus-Schalter für die Funktion |
| **PV-Schwelle für "Nacht"** | 10 W | Unterhalb dieser PV-Leistung gilt es als Nacht |

**Hinweis:** Nur Zone 2 wird nachts deaktiviert. Zone 1 läuft weiter!

---

## 🔧 Empfohlene Einstellungen

### Für konservative Batterienutzung (Langlebigkeit):
```yaml
SOC Zone 1 Start: 60%
SOC Zone 3 Stopp: 30%
Nullpunkt-Offset: 50W
PV-Ladereserve: 100W
P-Faktor: 1.5
I-Faktor: 0.05
Toleranzbereich: 30W
```

### Für maximale Eigenverbrauchsoptimierung:
```yaml
SOC Zone 1 Start: 40%
SOC Zone 3 Stopp: 15%
Nullpunkt-Offset: 20W
PV-Ladereserve: 30W
P-Faktor: 2.0
I-Faktor: 0.08
Toleranzbereich: 20W
```

### Für ausgewogenen Betrieb (Standard):
```yaml
SOC Zone 1 Start: 50%
SOC Zone 3 Stopp: 20%
Nullpunkt-Offset: 30W
PV-Ladereserve: 50W
P-Faktor: 1.5
I-Faktor: 0.05
Max. Ausgangsleistung: 800W
Toleranzbereich: 25W
```

---

## 📈 Funktionsweise im Detail

### Beispiel-Ablauf über einen Tag:

**Morgens (06:00 - SOC: 25%)**
- Zone 2 aktiv (Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle)
- PV steigt langsam an
- Max. Entladestrom: **0A** (batterieschonend)
- Regelziel: Offset 2 (leichter Netzbezug)
- Dynamisches Limit: Max(0, PV - 50W)
- Batterie wird geladen

**Mittags (12:00 - SOC: 55%)**
- Zone 1 aktiviert (SOC > Zone-1-Schwelle)
- Max. Entladestrom: **40A** (automatisch gesetzt)
- Regelziel: Offset 1 (nahe Nulleinspeisung)
- Hard Limit: 800W
- **Bleibt aktiv, auch wenn SOC wieder unter die Zone-1-Schwelle fällt!**

**Abends (20:00 - SOC: 22%)**
- Zone 1 immer noch aktiv (läuft bis zur Zone-3-Schwelle)
- Max. Entladestrom: weiterhin 40A

**Nacht (22:00 - SOC: 19%)**
- Zone 3 aktiviert (SOC ≤ Zone-3-Schwelle)
- Modus: `Disabled`
- Ausgang: 0W — Batterie geschützt

**Nächster Morgen — Zyklus beginnt von vorne**

---

## 🛑 Wichtige Fehlermeldungen

Der Blueprint validiert die Konfiguration beim Start. Fehler werden im **System-Log** angezeigt:

| Meldung | Ursache | Lösung |
|:--------|:--------|:-------|
| **Die obere SOC-Schwelle muss größer sein als die untere** | Schwellenwerte falsch konfiguriert | Zone-1-Schwelle (z.B. 50%) > Zone-3-Schwelle (z.B. 20%) |
| **SOC-Sensor ist UNKNOWN/UNAVAILABLE** | Solakon Integration offline | Prüfen Sie die Verbindung zum Wechselrichter |
| **Timeout Countdown Sensor ist UNKNOWN/UNAVAILABLE** | Sensor nicht verfügbar | Prüfen Sie die Solakon Integration |
| **Modus-Selektor ist UNKNOWN/UNAVAILABLE** | Select-Entity fehlt | Prüfen Sie die Solakon Integration |
| **Ausgangsleistungsregler hat keine min-Attribute** | Number-Entity fehlerhaft | Prüfen Sie die Solakon Integration |

---

## ⚙️ Technische Details

### PI-Regler Implementierung

**Fehlerberechnung (je nach Zone):**
```
Zone 1: error = Min(verfügbare_Kapazität, grid_power - offset_1)
Zone 2: error = Min(verfügbare_Kapazität, grid_power - offset_2, pv_kapazität)
```

**Integral-Logik mit Toleranz-Decay:**
```
Wenn |grid_error| > tolerance:
  integral_new = Clamp(integral_old + error, -1000, 1000)
Sonst wenn |integral_old| > 10:
  integral_new = integral_old * 0.95  # 5% Abbau pro Zyklus
Sonst:
  integral_new = integral_old         # Keine Änderung
```

**PI-Korrektur:**
```
correction = error * P_Factor + integral_new * I_Factor
new_power = current_power + correction
```

**Finale Leistung (zonenabhängige Begrenzung):**
```
Zone 0: final_power = hard_limit                       (Überschuss-Einspeisung)
Zone 1: final_power = Min(hard_limit, new_power)
Zone 2: final_power = Min(Max(0, PV - reserve), new_power)
```

**Ausgangs-Update-Bedingung:**
```
Nur setzen wenn |grid_error| > tolerance
```

---

### Recovery-Mechanismus

```
Bedingung:  Zyklus = on
        UND Modus ≠ INV Discharge PV Priority
        UND SOC > Zone-3-Schwelle

Aktion:     Puls-Sequenz 10s → (1s Pause) → 3599s
            Modus → INV Discharge PV Priority
            (kein Integral-Reset, kein Zonenwechsel)
```

---

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn aktueller Wert abweicht |
| Zone 2 (Schonend) | 0 A | Nur wenn aktueller Wert abweicht |

Änderungen werden nur vorgenommen, wenn der aktuelle Wert vom Sollwert abweicht (verhindert unnötige API-Calls).

---

## ⚠️ Wichtige Hinweise

1. **Helper erstellen vor Installation:** Beide Helper müssen existieren, bevor der Blueprint konfiguriert wird
2. **Solakon ONE Integration:** Muss vollständig eingerichtet sein
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **PI-Regler Tuning:** Die Standardwerte sind konservativ. Bei instabilem Verhalten I-Faktor senken.
5. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern!
6. **Queued-Modus:** Trigger werden sequentiell abgearbeitet
7. **Keine Trigger-Verzögerung:** Der Blueprint reagiert sofort auf Sensor-Änderungen
8. **Regelbare Wartezeit:** Nach jeder Leistungsänderung wartet der Blueprint 0–30 Sekunden
9. **Entladestrom-Automatik:** Max. Entladestrom wird vollautomatisch gesteuert — keine manuelle Einstellung nötig
10. **Toleranz-Decay:** Verhindert automatisch Integral-Windup — 5% Abbau pro Zyklus wenn `|Integral| > 10` und Grid-Fehler innerhalb der Toleranz
11. **Recovery:** Modus-Verlust bei aktivem Zyklus wird automatisch erkannt und korrigiert — kein manueller Eingriff nötig

---

## 🔄 Trigger-Übersicht

Der Blueprint reagiert auf folgende Events:

| Trigger | ID | Beschreibung |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Sofortige PI-Regelung bei Netzleistungsänderung |
| Solar Power Change | `solar_power_change` | Sofortige PI-Regelung bei PV-Leistungsänderung |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone-1-Schwelle) |
| SOC Low | `soc_low` | Zone 3 Start (SOC ≤ Zone-3-Schwelle) |
| Mode Change | `mode_change` | Reagiert auf externe Modusänderungen, löst ggf. Recovery aus |

**Alle Trigger** führen die komplette Logik aus — keine separate Behandlung nötig.
