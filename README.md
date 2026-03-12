# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V209

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku**, optionalem **AC-First-Modus** sowie optionalem **AC Laden aus externer Einspeisung**.

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie. Dies verhindert das "Flackern", das die App mit ihrer (lade ein Prozent → entlade ein Prozent → repeat) Funktionsweise verursacht, und schont die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder in der neuesten Version der APP die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2FexperimentalAC%2Fsolakon_one_nulleinspeisung.yaml)

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

### 3. Input Boolean Helper (Surplus-Zustand — ERFORDERLICH wenn Zone 0 aktiv!)

Wird benötigt, um den Überschuss-Einspeisung-Zustand **persistent über Automation-Läufe
hinweg** zu speichern. Verhindert, dass das System zwischen Zone 0 und Nulleinspeisung flackert.

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Umschalter** (Toggle)
3. Name: z.B. `Solakon Surplus Aktiv`
4. Speichern (Entity ID: z.B. `input_boolean.solakon_surplus_aktiv`)

> ℹ️ **Hinweis:** Dieser Helper wird nur benötigt wenn die Überschuss-Einspeisung aktiviert ist. Bei deaktivierter Zone 0 kann dieses Feld im Blueprint leer gelassen werden.

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
| **0. Überschuss-Einspeisung** | SOC ≥ Export-Schwelle UND Netz im Gleichgewicht UND PV ≥ Nacht-Schwelle UND PV > Ausgangsleistung + Grid | `discharge_mode` | 2 A (Stabilitätspuffer) | Hard Limit (max. W) | **Optional aktivierbar.** Überwiegend PV-Strom ins Netz. Austritt erst bei SOC < (Export-Schwelle − Hysterese). |
| **1. Aggressive Entladung** | SOC > Zone-1-Schwelle | `discharge_mode` | Konfigurierter Max-Wert (Standard: 40 A) | 0W + Offset 1 | Läuft **durchgehend bis SOC ≤ Zone-3-Schwelle** (kein Yo-Yo-Effekt). Auch nachts aktiv. Hard Limit. |
| **2. Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `discharge_mode` | **0 A** | 0W + Offset 2 | Dynamisches Limit: **Max(0, PV − Reserve)**. Optional: Nachtabschaltung möglich. |
| **3. Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `Disabled` | 0 A | — | Ausgang = 0 W. Vollständiger Schutz der Batterie. |

> `discharge_mode` ist entweder `INV Discharge (PV Priority)` (Standard) oder `INV Discharge (AC First)` je nach Konfiguration des AC-First-Parameters.

**Wichtig:**
- Zone 1 wird **einmal aktiviert** beim Überschreiten der Zone-1-Schwelle und läuft dann durch bis zur Zone-3-Schwelle
- Zone 2 startet nur wenn Zyklus = off UND Modus = Disabled (z.B. nach Zone-3-Übergang, Erststart oder manueller Deaktivierung)
- Dies verhindert ständiges Hin- und Herwechseln zwischen den Zonen
- **Max. Entladestrom** wird automatisch gesteuert — keine manuelle Einstellung nötig

#### 🔄 Recovery-Mechanismus

Falls der Modus des Wechselrichters extern zurückgesetzt wird (z.B. durch einen Neustart der Integration oder manuelle Änderung), während der Entladezyklus noch aktiv ist (`Zyklus = on`), erkennt der Blueprint diesen Zustand automatisch und reaktiviert den Modus über die Puls-Sequenz (10s → 3599s) — ohne Zonenwechsel oder Integral-Reset. Voraussetzung: SOC liegt noch über der Zone-3-Schwelle.

---

### 3. ☀️ Überschuss-Einspeisung (Optional — Zone 0)

Die Überschuss-Einspeisung kann optional aktiviert werden und ermöglicht die Einspeisung von echtem PV-Überschuss ins Netz, wenn der Akku voll ist. Der Wechsel in und aus Zone 0 folgt einer **SOC-Hysterese-Logik**, die instabiles Hin- und Herschalten bei nachlassender PV verhindert:

* **Aktivierung:** Über den Parameter "Überschuss-Einspeisung aktivieren"

* **Eintritts-Bedingung** (Wechsel von Nulleinspeisung → Zone 0):
  - SOC ≥ konfigurierte Export-Schwelle
  - Netz im Gleichgewicht: Grid Power ≤ Offset + Toleranz
  - PV aktiv: PV-Leistung ≥ Nacht-Schwelle *(schützt vor Eintritt im Dunkeln)*
  - PV-Überschuss vorhanden: PV > aktuelle Ausgangsleistung + Grid-Leistung

* **Verbleib-Bedingung:**
  - SOC ≥ (Export-Schwelle − Hysterese)
  - Beispiel: Schwelle = 90 %, Hysterese = 5 % → Austritt erst bei SOC < 85 %

* **Verhalten in Zone 0:**
  - Max. Entladestrom wird auf 2 A gesetzt (Stabilitätspuffer für den Wechselrichter)
  - AC-Output-Limit wird auf das Hard Limit gesetzt (z.B. 800 W)
  - Überwiegend PV-Strom wird ins Netz eingespeist (2 A Stabilitätspuffer ermöglicht minimalen Batteriebeitrag)
  - Output wird nur gesetzt, wenn er noch nicht auf dem Hard Limit steht (verhindert Modbus-Spam)

* **Rückkehr zur Nulleinspeisung:** Sobald SOC unter (Export-Schwelle − Hysterese) fällt, kehrt das System automatisch zur normalen PI-Regelung zurück

* **Deaktiviert:** Das System verhält sich wie klassische Nulleinspeisung — kein aktives Einspeisen ins Netz

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

### 5. 🔌 AC-First Modus (Optional)

Der AC-First Modus ändert den Entlademodus des Wechselrichters von `INV Discharge (PV Priority)` auf `INV Discharge (AC First)`. Alle Zonen, der PI-Regler und alle Limits bleiben vollständig identisch — lediglich die Priorisierung der Energiequelle ändert sich.

* **PV Priority (Standard):** PV-Energie wird bevorzugt genutzt, Netz ergänzt bei Bedarf
* **AC First:** Netzstrom wird als primäre Quelle bevorzugt; PV und Batterie ergänzen den Bedarf

Der gewählte Modus wird intern als `discharge_mode` gespeichert und in allen Zonenübergängen (Zone 1 Start, Zone 2 Start, Recovery, Exit aus AC Laden in Zone 1) konsistent verwendet.

---

### 6. ⚡ AC Laden (Optional)

Das AC Laden ermöglicht das Laden der Batterie wenn eine externe Einspeisung ins Netz erkannt wird — erkennbar daran, dass der Grid-Sensor negativ wird (Einspeisung statt Bezug). Typischer Anwendungsfall: eine externe PV-Anlage speist ihren Überschuss ins Netz, und die Solakon-Batterie soll diesen Überschuss statt des Netzes aufnehmen.

* **Aktivierung:** Über den Parameter "AC Laden aktivieren"

* **Eintritts-Bedingung:**
  - AC Laden aktiviert UND SOC < konfiguriertes Ladeziel
  - Grid < aktiver Offset (Offset₁ in Zone 1, Offset₂ in Zone 2) minus Toleranz

* **Verbleib-Bedingung (Hysterese):**
  - Grid < aktiver Offset − Toleranz + Hysterese
  - Verhindert Flackern bei wechselnder Einspeisung

* **Verhalten beim Laden:**
  - Entladestrom wird auf 0 A gesetzt
  - Modus → `INV Charge (PV Priority)` (Modus 3)
  - Ladeleistung = Min(Max-Ladeleistung, Offset − Grid) — skaliert automatisch mit dem verfügbaren Überschuss
  - Gilt sowohl in Zone 1 (Zyklus = on) als auch in Zone 2 (Zyklus = off)

* **Abbruch-Bedingung:**
  - SOC ≥ konfiguriertes Ladeziel, oder
  - Grid ≥ aktiver Offset − Toleranz + Hysterese (Überschuss weg)

* **Rückkehr nach dem Laden:**
  - Zone 1: Modus zurück zu `discharge_mode` + Puls-Sequenz (10s → 3599s)
  - Zone 2: Modus → `Disabled`, Output 0 W (Zone 2 greift beim nächsten Trigger neu ein)

* **Priorität:** Zone 3 (SOC-Schutz) evaluiert vor dem AC-Laden-Fall — der SOC-Schutz bleibt immer wirksam

---

### 7. ⏱️ Remote Timeout Reset und Moduswechsel-Sequenz

Um die Stabilität der Kommunikation mit dem Solakon ONE zu gewährleisten:

1. **Kontinuierlicher Timeout-Reset:**
   - Der Remote-Timeout wird automatisch auf 3599s zurückgesetzt
   - Trigger: Countdown fällt unter 120s

2. **Forcierter Reset bei Moduswechsel:**
   - Zweistufige Puls-Sequenz: 10s → 3599s (mit 1s Verzögerung)
   - Stellt sichere Modusübernahme sicher
   - Gilt bei Zone-1-Start, Zone-2-Start, Recovery, AC-Laden-Start und AC-Laden-Ende in Zone 1

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
| **Helper** | Surplus-Zustand-Speicher | `input_boolean.solakon_surplus_aktiv` | Nur erforderlich wenn Überschuss-Einspeisung aktiviert ist (Zone 0). Bei deaktivierter Zone 0: n/a |

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

### ☀️ Überschuss-Einspeisung (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Überschuss-Einspeisung aktivieren** | false | — | — | Schalter zum Aktivieren von Zone 0. |
| **SOC-Schwelle Überschuss** | 90 % | 50 % | 99 % | Ab diesem SOC wird bei PV-Überschuss ins Netz eingespeist. |
| **Hysterese Überschuss-Austritt** | 5 % | 1 % | 20 % | SOC muss um diesen Wert unter die Eintritts-Schwelle fallen bevor Zone 0 verlassen wird. |

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

### 🔒 Sicherheits-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Max. Ausgangsleistung** | 800 W | 0 | 1200 W | Absolute Obergrenze (Hard Limit) in Zone 0 und Zone 1. |

---

### 🔌 AC-First Modus (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **AC-First Modus aktivieren** | false | Schaltet Entlademodus auf `INV Discharge (AC First)` (Modus 13). Standard ist `INV Discharge (PV Priority)` (Modus 1). |

---

### ⚡ AC Laden (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **AC Laden aktivieren** | false | — | — | Schalter zum Aktivieren des AC Ladens. |
| **SOC-Ladeziel** | 90 % | 10 % | 99 % | Laden stoppt wenn SOC diesen Wert erreicht. Empfohlen: ≤ Zone-1-Schwelle. |
| **Max. Ladeleistung** | 800 W | 50 | 1200 W | Absolute Obergrenze der AC-Ladeleistung. |
| **Hysterese Ladeabbruch** | 50 W | 0 | 300 W | Laden endet erst wenn Grid wieder über Offset − Toleranz + Hysterese steigt. |

---

### 🌙 Nachtabschaltung (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **Nachtabschaltung aktivieren** | false | Ein/Aus-Schalter für die Funktion |
| **PV-Schwelle für "Nacht"** | 10 W | Unterhalb dieser PV-Leistung gilt es als Nacht |

**Hinweis:** Nur Zone 2 wird nachts deaktiviert. Zone 1 läuft weiter!

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

> ⚠️ **Abwärtskompatibel:** Wenn keine dynamische Entität ausgewählt wird, funktioniert dieser Blueprint wie bisher mit dem konfigurierten statischen Offset-Wert.

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

**Morgens mit externer Einspeisung (SOC: 30%, Grid negativ)**
- Externe Einspeisung erkannt: Grid < Offset₂ − Toleranz
- AC Laden startet: Modus `INV Charge (PV Priority)`, Entladestrom 0 A
- Ladeleistung = Min(Max-Ladeleistung, Offset₂ − Grid)
- SOC steigt, AC Laden läuft bis SOC-Ladeziel oder Überschuss weg

**Mittags (12:00 - SOC: 55%)**
- Zone 1 aktiviert (SOC > Zone-1-Schwelle)
- Max. Entladestrom: **40A** (automatisch gesetzt)
- Regelziel: Offset 1 (nahe Nulleinspeisung)
- Hard Limit: 800W
- **Bleibt aktiv, auch wenn SOC wieder unter die Zone-1-Schwelle fällt!**
- Falls Grid noch negativ: AC Laden bleibt aktiv (Zone 1, Offset₁ als Referenz)

**Mittags mit vollem Akku (SOC: 100% + Überschuss-Einspeisung aktiviert)**
- Eintritts-Bedingung: SOC ≥ Export-Schwelle, Netz ≤ Offset + Toleranz, PV aktiv, PV > Ausgangsleistung + Grid
- Zone 0 aktiv → Max. Entladestrom: **2A** (Stabilitätspuffer für den Wechselrichter)
- AC-Limit auf Hard Limit (800W) — reiner PV-Strom ins Netz
- Verbleib: solange SOC ≥ (Export-Schwelle − Hysterese), z.B. SOC ≥ 85% bei Schwelle 90% und Hysterese 5%
- Bei steigendem Hausverbrauch und sinkendem SOC: automatische Rückkehr zu Zone 1

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
Überschuss-Modus: Nur setzen wenn current_output < hard_limit  (Modbus-Spam verhindern)
Normal-Modus:     Nur setzen wenn |grid_error| > tolerance
```

---

### AC-First Modus

```
discharge_mode = '13'  (INV Discharge AC First)   wenn ac_first_enabled = true
discharge_mode = '1'   (INV Discharge PV Priority) wenn ac_first_enabled = false

Alle Zonenübergänge (Zone 1, Zone 2, Recovery, AC-Laden-Exit Zone 1)
verwenden discharge_mode statt hardcodierter '1'.
```

---

### AC Laden State-Machine

```
Eintritts-Bedingung:  ac_charge_enabled = true
                  UND soc < soc_ac_charge_limit
                  UND grid < active_offset - tolerance

Verbleib-Bedingung:   grid < active_offset - tolerance + hysteresis

Abbruch-Bedingung:    soc >= soc_ac_charge_limit
                  ODER grid >= active_offset - tolerance + hysteresis

Ladeleistung:         Min(ac_charge_power_limit, Max(0, active_offset - grid))

active_offset:        offset_1_int  wenn cycle = on (Zone 1)
                      offset_2_int  wenn cycle = off (Zone 2)

Exit Zone 1 → discharge_mode + Puls-Sequenz
Exit Zone 2 → Disabled + Output 0W
```

---

### Überschuss-Einspeisung State-Machine

```
Eintritts-Bedingung:  SOC >= export_limit
                  UND grid_power <= (target_offset + tolerance)
                  UND solar_power >= night_threshold
                  UND solar_power > (current_active_power + grid_power)

Verbleib-Bedingung:   SOC >= (export_limit - hysteresis)

Abbruch-Bedingung:    SOC < (export_limit - hysteresis)
```

---

### Recovery-Mechanismus

```
Bedingung:  Zyklus = on
        UND Modus ≠ discharge_mode
        UND SOC > Zone-3-Schwelle

Aktion:     Puls-Sequenz 10s → (1s Pause) → 3599s
            Modus → discharge_mode
            (kein Integral-Reset, kein Zonenwechsel)
```

---

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 0 (Überschuss) | 2 A (Stabilitätspuffer) | Immer, solange Zone 0 aktiv |
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn aktueller Wert abweicht und kein AC Laden aktiv |
| Zone 2 (Schonend) / AC Laden | 0 A | Nur wenn aktueller Wert abweicht |

Änderungen werden nur vorgenommen, wenn der aktuelle Wert vom Sollwert abweicht (verhindert unnötige API-Calls).

---

## ⚠️ Wichtige Hinweise

1. **Helper erstellen vor Installation:** Beide Pflicht-Helper müssen existieren, bevor der Blueprint konfiguriert wird
2. **Solakon ONE Integration:** Muss vollständig eingerichtet sein
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **PI-Regler Tuning:** Die Standardwerte sind konservativ. Bei instabilem Verhalten I-Faktor senken.
5. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern!
6. **Queued-Modus:** Trigger werden sequentiell abgearbeitet
7. **Keine Trigger-Verzögerung:** Der Blueprint reagiert sofort auf Sensor-Änderungen
8. **Regelbare Wartezeit:** Nach jeder Leistungsänderung wartet der Blueprint 0–30 Sekunden
9. **Entladestrom-Automatik:** Max. Entladestrom wird vollautomatisch gesteuert — keine manuelle Einstellung nötig
10. **Toleranz-Decay:** Verhindert automatisch Integral-Windup — 5% Abbau pro Zyklus wenn `|Integral| > 10` und Grid-Fehler innerhalb der Toleranz
11. **Überschuss-Einspeisung:** Persistenter `input_boolean` speichert den Zone-0-Zustand über Automation-Läufe hinweg. Austritt erfolgt erst wenn SOC unter (Export-Schwelle − Hysterese) fällt — verhindert Flackern bei nachlassender PV. Der Helper ist nur erforderlich wenn Zone 0 aktiviert ist.
12. **Recovery:** Modus-Verlust bei aktivem Zyklus wird automatisch erkannt und korrigiert — kein manueller Eingriff nötig
13. **AC-First:** Nur der Entlademodus ändert sich — alle Zonen, der PI-Regler und alle Limits bleiben identisch
14. **AC Laden:** Gilt in Zone 1 und Zone 2. Zone 3 (SOC-Schutz) hat immer Vorrang. Der Offset des aktiven Betriebsbereichs (Offset₁ in Zone 1, Offset₂ in Zone 2) dient als Referenz für den Ladestart und die Ladeleistungsberechnung.

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
Überschuss-Einspeisung: false
AC Laden: false
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
Überschuss-Einspeisung: true
SOC-Schwelle Überschuss: 95%
Hysterese Überschuss-Austritt: 5%
AC Laden: true
SOC-Ladeziel: 90%
Max. Ladeleistung: 800W
Hysterese Ladeabbruch: 50W
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
Max. Entladestrom Zone 1: 40A
Überschuss-Einspeisung: false
AC Laden: false
AC-First: false
```
