# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V300

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik**.

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie. Dies verhindert das "Flackern", das die App mit ihrer (lade ein Prozent → entlade ein Prozent → repeat) Funktionsweise verursacht, und schont die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder in der neuesten Version der APP die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fcomplete-rework-%2F-PI-script%2Fsolakon_one_nulleinspeisung.yaml)

Der zugehörige **PI-Regler Script-Blueprint** muss ebenfalls importiert werden:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fcomplete-rework-%2F-PI-script%2FPI-Regler.yaml)

## 🛠️ Vorbereitung: Erstellung der erforderlichen Helper

Der Blueprint benötigt **zwei Helper** und ein **Script**, die Sie vor der Installation erstellen bzw. einrichten müssen:

### 1. Input Boolean Helper (Entladezyklus-Speicher)

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Schalter** (Input Boolean)
3. Name: z.B. `SOC Entladezyklus Status`
4. Speichern (Entity ID: z.B. `input_boolean.soc_entladezyklus_status`)

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

### 3. PI-Regler Script (ERFORDERLICH)

1. Importieren Sie den **PI-Regler Blueprint** (Link oben)
2. Gehen Sie zu **Einstellungen** → **Automationen & Szenen** → **Scripts** → **Script erstellen**
3. Wählen Sie den importierten PI-Regler Blueprint
4. Name: z.B. `PI-Regler` (Script Entity ID: z.B. `script.pi_regler`)
5. Speichern

### 4. Input Number Helper für Dynamischen Offset (Optional)

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

Der Blueprint nutzt einen **PI-Regler** für präzise Nulleinspeisung. Die Rechenlogik ist in ein separates **Script-Blueprint** (`PI-Regler`) ausgelagert, das aus der Hauptautomatisierung heraus aufgerufen wird:

* **P-Anteil (Proportional):**
  - Reagiert sofort auf aktuelle Abweichungen
  - Konfigurierbare Aggressivität über den **P-Faktor** (z.B. 1.5)
  - Schnelle Reaktion auf Laständerungen

* **I-Anteil (Integral):**
  - Summiert Abweichungen über die Zeit auf
  - Eliminiert **bleibende Regelabweichungen**
  - Konfigurierbare Geschwindigkeit über den **I-Faktor** (z.B. 0.05)
  - **Anti-Windup:** Begrenzt auf ±1000 Punkte (im PI-Script)
  - **Automatischer Reset:** Bei jedem Zonenwechsel auf 0 zurückgesetzt
  - **Toleranz-Decay:** 5% Abbau pro Zyklus in der Hauptautomatisierung, solange der Grid-Fehler innerhalb der Toleranz liegt und `|Integral| > 10`. Verhindert schleichendes Integral-Windup bei stabiler Regelung.

* **Messprinzip:**
  - Basiert auf dem **Netz-Leistungssensor** (z.B. Shelly 3EM)
  - **Positive Werte = Bezug**, **Negative Werte = Einspeisung**
  - Keine Verzögerung — sofortige Reaktion auf Sensor-Änderungen

* **Fehlerberechnung mit Kapazitäts-Clamping (im PI-Script):**
  - Fehler = Grid Power − Target Offset
  - Positiver Fehler (Bezug): begrenzt auf verfügbare Kapazität nach oben (`dynamic_max − current`)
  - Negativer Fehler (Überschuss): begrenzt auf `0 − current` nach unten

* **Leistungsbegrenzung:**
  - **Zone 1:** `dynamic_max` = Hard Limit (z.B. 800 W)
  - **Zone 2:** `dynamic_max` = Max(0, PV − Reserve)
  - Unteres Limit: fest 0 W

* **PI-Aufruf-Guard (in Hauptautomatisierung):**
  - Wird nur ausgeführt wenn `|Fehler| > Toleranz`
  - Zusätzlich: kein Aufruf wenn `at_max_limit` (Fehler positiv UND Output bereits am Limit) oder `at_min_limit` (Fehler negativ UND Output bereits bei 0W)

---

### 2. 🔋 SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in drei Betriebsmodi unterteilt. Die Zonen-Steuerung erfolgt über die **Falls A–F** in einem `choose`-Block:

| Zone | SOC-Bereich / Bedingung | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:------------------------|:------|:-----------------|:---------|:--------------|
| **1. Aggressive Entladung** | SOC > Zone-1-Schwelle | `'1'` (INV Discharge PV Priority) | Konfigurierter Max-Wert (Standard: 40 A) | 0W + Offset 1 | Läuft **durchgehend bis SOC ≤ Zone-3-Schwelle** (kein Yo-Yo-Effekt). Auch nachts aktiv. Hard Limit. |
| **2. Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `'1'` (INV Discharge PV Priority) | **0 A** | 0W + Offset 2 | Dynamisches Limit: **Max(0, PV − Reserve)**. Optional: Nachtabschaltung möglich. |
| **3. Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `'0'` (Disabled) | 0 A | — | Ausgang = 0 W. Vollständiger Schutz der Batterie. |

**Wichtig:**
- Zone 1 wird **einmal aktiviert** beim Überschreiten der Zone-1-Schwelle (Zyklus = `off` → `on`) und läuft dann durch bis zur Zone-3-Schwelle
- Zone 2 startet nur wenn Zyklus = `off` UND Modus = `'0'` (z.B. nach Zone-3-Übergang, Erststart oder manueller Deaktivierung) UND NICHT Nacht
- Dies verhindert ständiges Hin- und Herwechseln zwischen den Zonen
- **Max. Entladestrom** wird automatisch gesteuert — keine manuelle Einstellung nötig (Änderung nur wenn aktueller Wert vom Sollwert abweicht)

#### Übersicht der Steuer-Falls (choose-Block)

| Fall | Bedingung | Aktion |
|:-----|:----------|:-------|
| **A** | SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Timer-Toggle, Modus → `'1'` |
| **B** | SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Modus → `'0'`, Output → 0W |
| **C** | SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Modus → `'0'`, Output → 0W |
| **D** | Zyklus = `on` UND Modus ≠ `'1'` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'1'` (kein Integral-Reset) |
| **E** | Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Timer-Toggle, Modus → `'1'` |
| **F** | Nachtabschaltung aktiv UND PV < Reserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Modus → `'0'`, Output → 0W |

#### 🔄 Recovery-Mechanismus (Fall D)

Falls der Modus des Wechselrichters extern zurückgesetzt wird (z.B. durch einen Neustart der Integration oder manuelle Änderung), während der Entladezyklus noch aktiv ist (Zyklus = `on`), erkennt der Blueprint diesen Zustand automatisch und reaktiviert den Modus über den Timer-Toggle — ohne Zonenwechsel oder Integral-Reset. Voraussetzung: SOC liegt noch über der Zone-3-Schwelle.

---

### 3. ⏱️ Timer-Toggle und Moduswechsel-Sequenz

Um die stabile Übernahme von Moduswechseln durch den Solakon ONE zu gewährleisten, wird statt eines festen Werts ein **Toggle zwischen 3598 und 3599** gesendet:

- Wenn aktueller Timer-Wert = 3599 → schreibe 3598
- Sonst → schreibe 3599

Dies erzeugt eine Zustandsänderung, die den Wechselrichter zur sicheren Übernahme des neuen Modus veranlasst. Der Toggle wird bei jedem Moduswechsel (Falls A, D, E) direkt vor dem Setzen des Modus durchgeführt.

**Kontinuierlicher Timeout-Reset (Schritt 2):**
- Der Remote-Timeout wird automatisch per Timer-Toggle zurückgesetzt
- Trigger: Countdown fällt unter 120s

---

### 4. 🌙 Nachtabschaltung (Optional)

Die Nachtabschaltung kann optional aktiviert werden und betrifft **nur Zone 2** (Fall F):

* **Aktivierung:** Über den Parameter "Nachtabschaltung aktivieren"
* **Schwelle:** PV-Leistung unter konfiguriertem Wert (PV-Ladereserve-Wert)
* **Verhalten:**
  - **Zone 1:** Läuft auch nachts weiter (hoher SOC → aggressive Entladung gewünscht)
  - **Zone 2:** Wird bei PV < Schwelle auf `'0'` (Disabled) gesetzt (Integral wird zurückgesetzt)
  - **Zone 3:** Ohnehin deaktiviert
* **Sinnvoll für:** Batterieschonung bei 0 PV, wenn keine Grundlast vorhanden ist

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
| **Solakon** | Modus-Reset-Timer | `number.solakon_one_fernsteuerung_zeituberschreitung` | Setzt/Reset Timeout (Timer-Toggle 3598↔3599) |
| **Solakon** | Betriebsmodus-Auswahl | `select.solakon_one_modus_fernsteuern` | Schaltet Betriebsmodus (`'0'` = Disabled, `'1'` = INV Discharge PV Priority) |
| **Helper** | Entladezyklus-Speicher | `input_boolean.soc_entladezyklus_status` | Input Boolean: `on`/`off` |
| **Helper** | Integral-Speicher | `input_number.solakon_integral` | Input Number: −1000 bis 1000 |
| **Script** | PI-Regler Script | `script.pi_regler` | Aus dem PI-Regler Blueprint erstelltes Script |

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.5 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver, schneller. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine PI-Korrektur innerhalb dieser Zone (stattdessen Integral-Decay). |
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
| **Zone 1 Start** | 50 % | 1 % | 99 % | Obere Schwelle. Überschreiten aktiviert Zone 1 (aggressive Entladung). |
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
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Reservierte PV-Leistung für Ladung in Zone 2. Dynamisches Limit: Max(0, PV − Reserve). Dient auch als PV-Schwelle für Nachtabschaltung. |

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

---

### 🔒 Sicherheits-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Max. Ausgangsleistung** | 800 W | 0 | 1200 W | Absolute Obergrenze (Hard Limit) in Zone 1. |

---

### 🌙 Nachtabschaltung (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **Nachtabschaltung aktivieren** | false | Ein/Aus-Schalter für die Funktion |
| **PV-Schwelle für "Nacht"** | — | Verwendet den Wert der **PV-Ladereserve** als Schwelle |

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
- Zone 1 aktiviert (SOC > Zone-1-Schwelle, Fall A)
- Max. Entladestrom: **40A** (automatisch gesetzt)
- Regelziel: Offset 1 (nahe Nulleinspeisung)
- Hard Limit: 800W
- **Bleibt aktiv, auch wenn SOC wieder unter die Zone-1-Schwelle fällt!**

**Abends (20:00 - SOC: 22%)**
- Zone 1 immer noch aktiv (läuft bis zur Zone-3-Schwelle)
- Max. Entladestrom: weiterhin 40A

**Nacht (22:00 - SOC: 19%)**
- Zone 3 aktiviert (SOC ≤ Zone-3-Schwelle, Fall B)
- Modus: `'0'` (Disabled)
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

---

## ⚙️ Technische Details

### Architektur

Der Blueprint besteht aus **zwei Komponenten**:

1. **Hauptautomatisierung** (`solakon_one_nulleinspeisung.yaml`): Zonen-Steuerung, SOC-Logik, Entladestrom-Verwaltung, Timeout-Reset, PI-Aufruf-Guard, Integral-Decay
2. **PI-Regler Script** (`PI-Regler.yaml`): Reine Berechnungslogik — Fehler, Integral-Update (Anti-Windup), Korrektur, Output-Setzen

### PI-Regler Implementierung (im Script)

**Fehlerberechnung mit Kapazitäts-Clamping:**
```
raw_error = grid_power - target_offset

Wenn raw_error > 0 (Bezug):
  error = Min(raw_error, dynamic_max - current_power)   # nicht über Kapazität
Wenn raw_error < 0 (Einspeisung):
  error = Max(raw_error, 0 - current_power)             # nicht unter 0
```

**Integral-Logik (Anti-Windup im Script):**
```
integral_new = Clamp(integral_old + error, -1000, 1000)
```

**PI-Korrektur:**
```
correction = error * P_Factor + integral_new * I_Factor
new_power = current_power + correction
final_power = Clamp(new_power, 0, dynamic_max)
```

### Integral-Decay (in Hauptautomatisierung)

Wird im `default`-Zweig ausgeführt, wenn der PI-Regler **nicht** aufgerufen wird (Fehler ≤ Toleranz oder At-Limit):

```
Wenn |Integral| > 10:
  integral_new = integral_old * 0.95   # 5% Abbau pro Zyklus
Sonst:
  integral_new = integral_old          # Keine Änderung
```

### PI-Aufruf-Bedingung

```
Nur ausführen wenn:
  |grid_error| > tolerance
  UND NICHT (raw_error > 0 UND current >= dynamic_max)   # at_max_limit
  UND NICHT (raw_error < 0 UND current <= 0)             # at_min_limit
```

### Dynamisches Power-Limit (zonenabhängig)

```
Zone 1 (cycle = on):  dynamic_max = hard_limit
Zone 2 (cycle = off): dynamic_max = Max(0, PV - pv_charge_reserve)
```

### Timer-Toggle Mechanismus

```
Wenn timer_value == 3599:
  → Schreibe 3598
Sonst:
  → Schreibe 3599
→ Anschließend: Modus setzen
```

Erzeugt zuverlässig eine Zustandsänderung für sichere Modusübernahme durch den Solakon ONE.

### Recovery-Mechanismus (Fall D)

```
Bedingung:  Zyklus = on
        UND Modus ≠ '1'
        UND SOC > Zone-3-Schwelle

Aktion:     Timer-Toggle (3598↔3599)
            Modus → '1' (INV Discharge PV Priority)
            (kein Integral-Reset, kein Zonenwechsel)
```

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn aktueller Wert abweicht |
| Zone 2 (Schonend) | 0 A | Nur wenn aktueller Wert abweicht |

Änderungen werden nur vorgenommen, wenn der aktuelle Wert vom Sollwert abweicht (verhindert unnötige Modbus-Calls).

---

## ⚠️ Wichtige Hinweise

1. **Helper und Script vor Installation erstellen:** Input Boolean, Input Number (Integral) und PI-Regler Script müssen existieren, bevor der Blueprint konfiguriert wird
2. **Solakon ONE Integration:** Muss vollständig eingerichtet sein
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **PI-Regler Tuning:** Die Standardwerte sind konservativ. Bei instabilem Verhalten I-Faktor senken.
5. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern!
6. **Queued-Modus:** Trigger werden sequentiell abgearbeitet
7. **Keine Trigger-Verzögerung:** Der Blueprint reagiert sofort auf Sensor-Änderungen
8. **Regelbare Wartezeit:** Nach jeder Leistungsänderung wartet der Blueprint 0–30 Sekunden
9. **Entladestrom-Automatik:** Max. Entladestrom wird vollautomatisch gesteuert — keine manuelle Einstellung nötig
10. **Toleranz-Decay:** Verhindert automatisch Integral-Windup — 5% Abbau pro Zyklus wenn `|Integral| > 10` und Grid-Fehler innerhalb der Toleranz
11. **Recovery:** Modus-Verlust bei aktivem Zyklus wird automatisch erkannt und korrigiert (Fall D) — kein manueller Eingriff nötig
12. **Modus-Werte:** Der Solakon ONE verwendet `'0'` für Disabled und `'1'` für INV Discharge PV Priority

---

## 🔄 Trigger-Übersicht

Der Blueprint reagiert auf folgende Events:

| Trigger | ID | Beschreibung |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Sofortige PI-Regelung bei Netzleistungsänderung |
| Solar Power Change | `solar_power_change` | Sofortige PI-Regelung bei PV-Leistungsänderung |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone-1-Schwelle) |
| SOC Low | `soc_low` | Zone 3 Start (SOC ≤ Zone-3-Schwelle) |
| Mode Change | `mode_change` | Reagiert auf externe Modusänderungen, löst ggf. Recovery aus (Fall D) |

**Alle Trigger** führen die komplette Logik aus — keine separate Behandlung nötig.
