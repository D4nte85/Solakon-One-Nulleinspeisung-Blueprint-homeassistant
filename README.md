# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V302

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku** sowie optionalem **AC Laden aus externer Einspeisung**.

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie. Dies verhindert das "Flackern", das die App mit ihrer (lade ein Prozent → entlade ein Prozent → repeat) Funktionsweise verursacht, und schont die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder in der neuesten Version der APP die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

Der zugehörige **PI-Regler Script-Blueprint** muss ebenfalls importiert werden:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fmain%2FPI-Regler.yaml)

---

## 🛠️ Vorbereitung: Erstellung der erforderlichen Helper

Der Blueprint benötigt **drei Pflicht-Helper** und ein **Script** sowie bis zu **zwei optionale Helper**, die Sie vor der Installation erstellen müssen.

### 1. Input Boolean Helper (Entladezyklus-Speicher) — ERFORDERLICH

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Schalter** (Input Boolean)
3. Name: z.B. `SOC Entladezyklus Status`
4. Speichern (Entity ID: z.B. `input_boolean.soc_entladezyklus_status`)

### 2. Input Number Helper (Integral-Speicher für PI-Regler) — ERFORDERLICH

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Number**
3. Name: z.B. `Solakon Integral`
4. **Wichtige Einstellungen:**
   * Minimum: `-1000`
   * Maximum: `1000`
   * Step: `1`
   * Initialwert: `0`
5. Speichern (Entity ID: z.B. `input_number.solakon_integral`)

### 3. PI-Regler Script — ERFORDERLICH

1. Importieren Sie den **PI-Regler Blueprint** (Link oben)
2. Gehen Sie zu **Einstellungen** → **Automationen & Szenen** → **Scripts** → **Script erstellen**
3. Wählen Sie den importierten PI-Regler Blueprint
4. Name: z.B. `PI-Regler` (Script Entity ID: z.B. `script.pi_regler`)
5. Speichern

### 4. Input Boolean Helper (Surplus-Zustand) — NUR wenn Zone 0 aktiv

Persistenter Zustandsspeicher für die Überschuss-Einspeisung. Verhindert Flackern bei schwankender PV.

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Schalter** (Input Boolean)
3. Name: z.B. `Solakon Surplus Aktiv`
4. Speichern (Entity ID: z.B. `input_boolean.solakon_surplus_aktiv`)

### 5. Input Boolean Helper (AC-Lade-Zustand) — NUR wenn AC Laden aktiv

Persistenter Zustandsspeicher für das AC Laden. Signalisiert dem PI-Script die invertierte Fehlerberechnung (`ac_charge_mode=true`). Verhindert außerdem, dass die Guards (at_max/at_min) den PI fälschlich blockieren.

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Schalter** (Input Boolean)
3. Name: z.B. `Solakon AC Laden Aktiv`
4. Speichern (Entity ID: z.B. `input_boolean.solakon_ac_laden_aktiv`)

### 6. Input Number Helper für Dynamischen Offset (Optional)

Wenn Sie den Nullpunkt-Offset zur Laufzeit dynamisch anpassen möchten, empfehlen wir den **Solakon ONE — Dynamischer Offset Blueprint**. Dieser erstellt und befüllt die benötigten Helper automatisch.

Alternativ manuell:

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer** → **Number**
2. Name: z.B. `Solakon Offset Zone 1`
3. **Einstellungen:** Min: `0`, Max: `500`, Step: `1`, Initialwert: `30`
4. Speichern (Entity ID: z.B. `input_number.solakon_offset_zone1`)
5. Optional: Wiederholen für Zone 2 (`input_number.solakon_offset_zone2`)

---

## 🧠 Kernfunktionalität

### 1. PI-Regler (Proportional-Integral-Regler)

Der Blueprint nutzt einen **PI-Regler** für präzise Nulleinspeisung. Die Rechenlogik ist in ein separates **Script-Blueprint** (`PI-Regler`) ausgelagert, das aus der Hauptautomatisierung heraus aufgerufen wird.

* **P-Anteil (Proportional):**
  - Reagiert sofort auf aktuelle Abweichungen
  - Konfigurierbare Aggressivität über den **P-Faktor** (z.B. 1.5)

* **I-Anteil (Integral):**
  - Summiert Abweichungen über die Zeit auf, eliminiert bleibende Regelabweichungen
  - **Anti-Windup:** Begrenzt auf ±1000 Punkte (im PI-Script)
  - **Automatischer Reset:** Bei jedem Zonenwechsel auf 0 zurückgesetzt
  - **Toleranz-Decay:** 5% Abbau pro Zyklus in der Hauptautomatisierung, solange Fehler ≤ Toleranz und `|Integral| > 10`
  - **Zone-0-Einfrieren:** In Zone 0 (Überschuss) wird das Integral unverändert beibehalten

* **Fehlerberechnung (modusabhängig, im PI-Script):**
  - **Normal (`ac_charge_mode=false`):** `raw_error = grid − target_offset`
  - **AC Laden (`ac_charge_mode=true`):** `raw_error = target_offset − grid` (invertiert)
  - In beiden Fällen: Kapazitäts-Clamping auf tatsächlich verfügbare Kapazität

* **Leistungsbegrenzung (zonenabhängig, `max_power` ans Script):**
  - **Zone 0:** Hard Limit (PI nicht aufgerufen)
  - **Zone 1:** Hard Limit (z.B. 800 W)
  - **Zone 2:** `Max(0, PV − Reserve)`
  - **AC Laden (Modus 3):** Konfigurierbares Lade-Limit

* **PI-Aufruf-Guard (in Hauptautomatisierung):**
  - Zone 0 aktiv → PI nicht aufgerufen, Integral eingefroren
  - AC Laden aktiv → PI mit `ac_charge_mode=true`, `at_max/at_min`-Guards deaktiviert
  - Normal → PI nur wenn `|Fehler| > Toleranz` UND kein At-Limit

* **PI-Aufruf-Guard (in Hauptautomatisierung):**
  - Wird nur ausgeführt wenn `|Fehler| > Toleranz`
  - Zusätzlich: kein Aufruf wenn `at_max_limit` (Fehler positiv UND Output bereits am Limit) oder `at_min_limit` (Fehler negativ UND Output bereits bei 0W)

---

### 2. 🔋 SOC-Zonen-Logik

| Zone | SOC-Bereich / Bedingung | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:------------------------|:------|:-----------------|:---------|:--------------|
| **0. Überschuss-Einspeisung** | SOC ≥ Export-Schwelle UND PV > Output + Grid + PV-Hysterese | `'1'` | 2 A (Stabilitätspuffer) | Hard Limit (max. W) | **Optional.** Integral eingefroren. Persistenter `input_boolean`. Austritt mit SOC-Hysterese + PV-Hysterese. |
| **1. Aggressive Entladung** | SOC > Zone-1-Schwelle | `'1'` | Konfigurierter Max-Wert (Standard: 40 A) | 0W + Offset 1 | Läuft **bis SOC ≤ Zone-3-Schwelle** (kein Yo-Yo-Effekt). Auch nachts aktiv. |
| **2. Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `'1'` | **0 A** | 0W + Offset 2 | Dynamisches Limit: `Max(0, PV − Reserve)`. Optional: Nachtabschaltung. |
| **3. Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `'0'` (Disabled) | 0 A | — | Output = 0 W. Vollständiger Batterieschutz. |

#### Übersicht der Steuer-Falls (choose-Block)

| Fall | Bedingung | Aktion |
|:-----|:----------|:-------|
| **A** | SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Surplus/AC-Bool zurücksetzen, Timer-Toggle, Modus → `'1'` |
| **B** | SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **C** | SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **D** | Zyklus = `on` UND Modus ∉ `{'1','3'}` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'3'` wenn AC-Lade-Bool = `on`, sonst `'1'` (kein Integral-Reset, kein Zonenwechsel) |
| **G** | AC aktiv UND SOC < Ladeziel **UND Modus ≠ `'3'`** UND (Grid + Output) < −Hysterese | AC Laden Start: AC-Bool = `on`, Timer-Toggle, Modus → `'3'`, Output → 0W |
| **H** | Modus = `'3'` UND (SOC ≥ Ladeziel ODER (Grid ≥ `ac_charge_offset + Hysterese` UND Output = 0 W)) | AC Laden Ende: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **I** | Modus = `'3'` UND (AC Laden deaktiviert ODER AC-Lade-Bool ≠ `on`) | Safety-Korrektur: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **E** | Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Timer-Toggle, Modus → `'1'` |
| **F** | Nachtabschaltung aktiv UND PV < PV-Ladereserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Modus → `'0'`, Output → 0W |

> **Reihenfolge ist entscheidend:** Fall D liegt vor Fall G. Dadurch prüft Recovery nur Modus ∉ `{'1','3'}` — AC-Laden-Modus `'3'` wird **nicht** durch Recovery überschrieben. Fall I liegt nach H und fängt jeden Modus-`'3'`-Zustand ab, der nicht durch eine aktive AC-Lade-Session legitimiert ist.

#### 🔄 Recovery-Mechanismus (Fall D)

Falls der Modus des Wechselrichters extern zurückgesetzt wird (z.B. durch einen Neustart der Integration), während der Entladezyklus noch aktiv ist (Zyklus = `on`), erkennt der Blueprint diesen Zustand automatisch und reaktiviert den Modus über den Timer-Toggle — ohne Zonenwechsel oder Integral-Reset. Voraussetzung: SOC > Zone-3-Schwelle UND Modus ≠ `'1'` UND Modus ≠ `'3'`.

Der wiederhergestellte Modus richtet sich nach dem AC-Lade-Zustand: Wenn der AC-Lade-Bool `on` ist, wird Modus `'3'` gesetzt; andernfalls Modus `'1'`.

#### ⚠️ Safety-Mechanismus (Fall I)

Fängt den Zustand ab, in dem der Wechselrichter Modus `'3'` hat, aber keine aktive AC-Lade-Session vorliegt (AC Laden deaktiviert, Helper `off` oder nicht verfügbar). Dies kann durch externe Modussetzung entstehen. Aktion: Integral zurücksetzen, Zone 1 → Modus `'1'` (Timer-Toggle), Zone 2 → Modus `'0'` + Output 0W.

| Fall | Bedingung | Aktion |
|:-----|:----------|:-------|
| **A** | SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Timer-Toggle, Modus → `'1'` |
| **B** | SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Modus → `'0'`, Output → 0W |
| **C** | SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Modus → `'0'`, Output → 0W |
| **D** | Zyklus = `on` UND Modus ≠ `'1'` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'1'` (kein Integral-Reset) |
| **E** | Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Timer-Toggle, Modus → `'1'` |
| **F** | Nachtabschaltung aktiv UND PV < Reserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Modus → `'0'`, Output → 0W |

#### 🔄 Recovery-Mechanismus (Fall D)

Ermöglicht die Einspeisung von echtem PV-Überschuss ins Netz, wenn der Akku voll ist. SOC- und PV-Hysterese verhindern instabiles Hin- und Herschalten.

* **Aktivierung:** Über den Parameter "Überschuss-Einspeisung aktivieren"
* **Eintritts-Bedingung:** SOC ≥ Export-Schwelle UND PV > (Output + Grid + PV-Hysterese)
* **Austritts-Bedingung:** SOC < (Export-Schwelle − SOC-Hysterese) ODER PV ≤ (Output + Grid − PV-Hysterese)
* **Persistenz:** Zustand wird in `input_boolean` gespeichert — überlebt mehrere Automation-Läufe
* **Verhalten:** Output auf Hard Limit, Entladestrom 2 A, Integral eingefroren (kein Decay, kein PI-Aufruf)
* **Deaktiviert:** Klassische Nulleinspeisung — kein aktives Einspeisen

---

### 4. ⚡ AC Laden (Optional)

Laden der Batterie wenn eine externe Einspeisung ins Netz erkannt wird. Die Erkennung basiert auf `(Grid + Ausgangsleistung) < −Hysterese` — d.h. nach Abzug des Solakon-Beitrags liegt noch Überschuss an. Typischer Anwendungsfall: externe PV-Anlage speist Überschuss ins Netz.

* **Aktivierung:** Über den Parameter "AC Laden aktivieren"
* **Eintritts-Bedingung (Fall G):**
  - AC Laden aktiviert UND SOC < Ladeziel
  - **Modus darf nicht `'3'` sein** (Guard verhindert Re-Eintritt wenn AC Laden bereits aktiv)
  - (Grid + Ausgangsleistung) < −Hysterese
* **Verbleib-Bedingung:**
  - Modus bleibt `'3'` solange SOC < Ladeziel UND Grid < (ac_charge_offset + Hysterese)
* **Abbruch-Bedingung (Fall H):**
  - SOC ≥ Ladeziel ODER (Grid ≥ ac_charge_offset + Hysterese UND Output = 0 W)
  - `Output = 0 W`-Guard verhindert Fehlauslösung während PI noch regelt
* **PI-Regelung:**
  - Script wird mit `ac_charge_mode=true` aufgerufen → invertierte Fehlerberechnung: `target_offset − grid`
  - Positiver Fehler → Ladeleistung erhöhen (Grid zu negativ → mehr laden)
  - `max_power` = konfiguriertes Lade-Limit
  - `at_max/at_min`-Guards werden nicht angewendet (Richtung invertiert)
  - **Separate P/I-Faktoren:** P klein (~0.3–0.5) wegen langer Hardware-Flanke (~25 s); I-Anteil macht die eigentliche Regelarbeit
* **Zustandsspeicher:** `input_boolean` (`ac_charge_state_helper`) persistent — signalisiert PI-Script-Aufrufzweig
* **Rückkehr:**
  - Zone 1 → Modus `'1'` (Timer-Toggle) + Integral Reset
  - Zone 2 → Modus `'0'` + Output 0W + Integral Reset
* **Priorität:** Zone 3 (SOC-Schutz) hat immer Vorrang

---

### 5. ⏱️ Timer-Toggle und Moduswechsel-Sequenz

Um die stabile Übernahme von Moduswechseln durch den Solakon ONE zu gewährleisten, wird statt eines festen Werts ein **Toggle zwischen 3598 und 3599** gesendet:

- Wenn aktueller Timer-Wert = 3599 → schreibe 3598
- Sonst → schreibe 3599

Dies erzeugt eine Zustandsänderung, die den Wechselrichter zur sicheren Übernahme des neuen Modus veranlasst. Kein Delay erforderlich. Der Toggle wird bei jedem Moduswechsel (Falls A, D, E, G, H-Zone1, I-Zone1) direkt vor dem Setzen des Modus durchgeführt.

**Kontinuierlicher Timeout-Reset (Schritt 2):** Countdown < 120s → Timer-Toggle.

---

### 6. 🌙 Nachtabschaltung (Optional)

Betrifft **nur Zone 2** (Fall F). Zone 1 und AC Laden laufen auch nachts weiter.

* **Schwelle:** PV-Leistung unter dem Wert der **PV-Ladereserve** (kein separater Parameter)
* **Verhalten bei Nacht:** Modus → `'0'`, Integral reset, Output 0W
* **Zone 1:** Läuft durch (hoher SOC → aggressive Entladung weiterhin gewünscht)

---

## 📊 Input-Variablen und Konfiguration

### 🔌 Erforderliche Entitäten

| Kategorie | Variable | Standard-Entität | Beschreibung |
|:----------|:---------|:----------------|:-------------|
| **Extern** | Netz-Leistungssensor | *(kein Standard)* | Z.B. Shelly 3EM. **Positiv = Bezug, Negativ = Einspeisung** |
| **Solakon** | Solarleistung | `sensor.solakon_one_pv_leistung` | Aktuelle PV-Erzeugung in Watt |
| **Solakon** | Tatsächliche Ausgangsleistung | `sensor.solakon_one_leistung` | Aktuelle AC-Ausgangsleistung in Watt |
| **Solakon** | Batterieladestand (SOC) | `sensor.solakon_one_batterie_ladestand` | Ladestand in % |
| **Solakon** | Remote Timeout Countdown | `sensor.solakon_one_fernsteuerung_zeituberschreitung` | Verbleibender Countdown |
| **Solakon** | Ausgangsleistungsregler | `number.solakon_one_fernsteuerung_leistung` | Setzt Leistungs-Sollwert |
| **Solakon** | Max. Entladestrom | `number.solakon_one_maximaler_entladestrom` | Setzt Entladestrom-Limit |
| **Solakon** | Modus-Reset-Timer | `number.solakon_one_fernsteuerung_zeituberschreitung` | Timer-Toggle 3598↔3599 |
| **Solakon** | Betriebsmodus-Auswahl | `select.solakon_one_modus_fernsteuern` | `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority |
| **Helper** | Entladezyklus-Speicher | `input_boolean.soc_entladezyklus_status` | Input Boolean: `on`/`off` |
| **Helper** | Integral-Speicher | `input_number.solakon_integral` | Input Number: −1000 bis 1000 |
| **Script** | PI-Regler Script | `script.pi_regler` | Aus dem PI-Regler Blueprint erstelltes Script |
| **Helper** | Surplus-Zustand-Speicher | `input_boolean.solakon_surplus_aktiv` | Nur wenn Zone 0 aktiv. |
| **Helper** | AC-Lade-Zustand-Speicher | `input_boolean.solakon_ac_laden_aktiv` | Nur wenn AC Laden aktiv. |

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.3 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine PI-Korrektur innerhalb (stattdessen Integral-Decay). |
| **Wartezeit** | 3 s | 0 | 30 s | Verzögerung nach Leistungsänderung in allen Modi (Zone 1, Zone 2, AC Laden). Kompensiert die Reaktionszeit des Wechselrichters. |

> **Hinweis:** P- und I-Faktor gelten für Zone 1 und Zone 2. Für den AC-Lade-Modus (Modus `'3'`) werden separate Faktoren verwendet — siehe [AC Laden Parameter](#-ac-laden-optional).

---

### 🔋 SOC-Zonen-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Überschreiten aktiviert Zone 1. |
| **Zone 3 Stopp** | 20 % | 1 % | 49 % | Unterschreiten stoppt Entladung komplett. |
| **Max. Entladestrom Zone 1** | 40 A | 0 A | 40 A | Zone 2 und AC Laden nutzen automatisch 0 A. |

**Wichtig:** Zone-1-Schwelle muss **größer** als Zone-3-Schwelle sein! Blueprint validiert dies beim Start.

---

### ⚙️ Zone 1/2 - Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Nullpunkt-Offset-1 (Statisch)** | 30 W | -100 | 100 W | Statischer Fallback für Zone 1. |
| **Nullpunkt-Offset-1 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **Nullpunkt-Offset-2 (Statisch)** | 30 W | -100 | 100 W | Statischer Fallback für Zone 2. |
| **Nullpunkt-Offset-2 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Zone-2-Limit: `Max(0, PV − Reserve)`. Dient auch als PV-Schwelle für Nachtabschaltung. |

---

### 🔒 Sicherheits-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Max. Ausgangsleistung** | 800 W | 0 | 1200 W | Hard Limit in Zone 0 und Zone 1. |

---

### ☀️ Überschuss-Einspeisung (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Überschuss-Einspeisung aktivieren** | false | — | — | Schalter für Zone 0. |
| **SOC-Schwelle Überschuss** | 90 % | 50 % | 99 % | Ab diesem SOC bei PV-Überschuss → Zone 0. |
| **Hysterese Überschuss-Austritt (SOC)** | 5 % | 1 % | 20 % | SOC muss um diesen Wert unter die Eintritts-Schwelle fallen bevor Zone 0 verlassen wird. |
| **Hysterese PV-Überschuss** | 50 W | 10 W | 200 W | Totband um Hausverbrauch für Ein- und Austritt. |

---

### ⚡ AC Laden (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **AC Laden aktivieren** | false | — | — | Schalter für AC Laden. |
| **SOC-Ladeziel** | 90 % | 10 % | 99 % | Laden stoppt bei diesem SOC. |
| **Max. Ladeleistung** | 800 W | 50 | 1200 W | Obergrenze der AC-Ladeleistung (`max_power` ans Script). |
| **Hysterese Ladeabbruch** | 50 W | 0 | 300 W | Totband für Ein- und Austritt. Eintritt: (Grid + Output) < −Hysterese. Austritt: Grid ≥ (Offset + Hysterese) UND Output = 0 W. |
| **AC Laden Offset (Statisch)** | -50 W | -100 | 100 W | Regelziel im AC-Lade-Modus. Negativ = Einspeisung angestrebt → höhere Ladeleistung. |
| **AC Laden Offset (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **AC Laden P-Faktor** | 0.5 | 0.1 | 5.0 | Proportional-Verstärkung im AC-Lade-Modus. Klein halten wegen langer Hardware-Flanke (~25 s). |
| **AC Laden I-Faktor** | 0.07 | 0.01 | 0.2 | Integral-Verstärkung im AC-Lade-Modus. Macht bei langen Wartezeiten die eigentliche Regelarbeit. |

---

### 🌙 Nachtabschaltung (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **Nachtabschaltung aktivieren** | false | Ein/Aus-Schalter für die Funktion |
| **PV-Schwelle für "Nacht"** | — | Verwendet den Wert der **PV-Ladereserve** als Schwelle |

---

## 🎯 Dynamischer Offset Blueprint (Empfohlen)

### 👉 [Solakon ONE — Dynamischer Offset (Volatilitäts-Regler)](https://github.com/D4nte85/Solakon-one-dynamic-offset-blueprint)

[![Blueprint importieren](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/D4nte85/Solakon-one-dynamic-offset-blueprint/main/solakon_dynamic_offset_blueprint.yaml)

Passt den Nullpunkt-Offset vollautomatisch an die aktuelle Netz-Volatilität an (Standardabweichung der letzten 60s).

| Netz-Zustand | StdDev | Berechneter Offset |
|:------------|:------:|:------------------:|
| Sehr ruhig | 5 W | 30 W *(Minimum)* |
| Normal | 30 W | 53 W |
| Unruhig | 80 W | 128 W |
| Sehr unruhig | 160 W | 228 W |
| Extrem | 250 W+ | 250 W *(Maximum)* |

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
```

---

## 📈 Funktionsweise im Detail

### Beispiel-Ablauf über einen Tag:

**Morgens (06:00 - SOC: 25%)**
- Zone 2 aktiv; PV steigt langsam an
- Max. Entladestrom: **0A** (batterieschonend)
- Dynamisches Limit: `Max(0, PV - 50W)`; Batterie wird geladen

**Morgens mit externer Einspeisung (SOC: 30%)**
- (Grid + Output) < −Hysterese, Modus ≠ `'3'` → Fall G greift
- AC Laden startet: `ac_charge_state_helper` = `on`, Modus → `'3'`
- PI-Script mit `ac_charge_mode=true` → invertierte Fehlerberechnung
- Ladeleistung geregelt auf `ac_charge_offset`, begrenzt auf Max. Ladeleistung

**Mittags (12:00 - SOC: 55%)**
- Zone 1 aktiviert (Fall A); Max. Entladestrom: **40A**
- Regelziel: Offset 1; Hard Limit: 800W
- **Bleibt aktiv bis SOC ≤ Zone-3-Schwelle!**

**Mittags mit vollem Akku (SOC: 92%, Überschuss-Einspeisung aktiviert)**
- Zone 0 aktiv; Entladestrom: **2A** (Stabilitätspuffer)
- Output → Hard Limit; Integral eingefroren
- Verbleib: solange SOC ≥ (90% − 5%) = 85%

**Abends (20:00 - SOC: 22%)**
- Zone 1 weiterhin aktiv; Max. Entladestrom: 40A

**Nacht (22:00 - SOC: 19%)**
- Zone 3 aktiviert (Fall B); Modus: `'0'`; Output: 0W

---

## 🛑 Wichtige Fehlermeldungen

| Meldung | Ursache | Lösung |
|:--------|:--------|:-------|
| **SOC-Limits ungültig** | Zone-1-Schwelle ≤ Zone-3-Schwelle | Zone-1 (z.B. 50%) > Zone-3 (z.B. 20%) |
| **SOC-Sensor UNKNOWN/UNAVAILABLE** | Solakon Integration offline | Verbindung prüfen |
| **Timeout Countdown UNKNOWN/UNAVAILABLE** | Sensor nicht verfügbar | Solakon Integration prüfen |

---

## ⚙️ Technische Details

### Architektur

1. **Hauptautomatisierung** (`solakon_one_nulleinspeisung.yaml`): Zonen-Steuerung, SOC-Logik, Surplus-Zustand, AC-Lade-Zustand, Entladestrom-Verwaltung, Timeout-Reset, PI-Aufruf-Guard, Integral-Decay/-Einfrieren
2. **PI-Regler Script** (`PI-Regler.yaml`): Reine Berechnungslogik mit modusabhängiger Fehlerberechnung via `ac_charge_mode`-Feld

### PI-Regler Implementierung (im Script)

**Fehlerberechnung (modusabhängig):**
```
ac_charge_mode = false (Normal):
  raw_error = grid_power - target_offset

ac_charge_mode = true (AC Laden):
  raw_error = target_offset - grid_power   ← invertiert

In beiden Fällen — Kapazitäts-Clamping:
  raw_error > 0: error = Min(raw_error, max_power - current_power)
  raw_error < 0: error = Max(raw_error, 0 - current_power)
```

**Integral und Korrektur:**
```
integral_new = Clamp(integral_old + error, -1000, 1000)
correction   = error * P_Factor + integral_new * I_Factor
new_power    = current_power + correction
final_power  = Clamp(new_power, 0, max_power)
```

### Integral-Management (in Hauptautomatisierung)
```
Zone 0 aktiv:              integral = integral_old (eingefroren)
AC Laden aktiv (Zweig B):  → PI-Script aufgerufen (ac_charge_mode=true)
                              at_max/at_min Guards NICHT angewendet
Normal (Zweig C):          → PI-Script aufgerufen (ac_charge_mode=false)
                              at_max/at_min Guards angewendet
Sonst (Decay):
  |Integral| > 10 → integral * 0.95
  sonst           → keine Änderung
```

### Dynamisches Power-Limit
```
Modus '3' (AC Laden): max_power = ac_charge_power_limit
Zone 1 (cycle = on):  max_power = hard_limit
Zone 2 (cycle = off): max_power = Max(0, PV - pv_charge_reserve)
```

### Timer-Toggle Mechanismus
```
Wenn timer_value == 3599 → schreibe 3598
Sonst                    → schreibe 3599
→ Anschließend: Modus setzen
```
Kein Delay erforderlich (keine 1s-Pause wie in V209).

### Fall D / Recovery — Modus-Ausschluss und Dual-Restore
```
Bedingung: Zyklus = on
       UND Modus ∉ {'1', '3'}   ← '3' (AC Laden) explizit ausgeschlossen
       UND SOC > Zone-3-Schwelle

Aktion: Timer-Toggle
        → AC-Lade-Bool = on: Modus '3'
        → sonst:              Modus '1'
        (kein Integral-Reset, kein Zonenwechsel)
```

### Fall G — Eintritts-Guard
```
Bedingung: ac_charge_enabled
       UND SOC < soc_ac_charge_limit
       UND Modus ≠ '3' ← Guard: verhindert Re-Eintritt wenn AC Laden bereits aktiv (Modus = '3')
       UND (grid + actual_power) < -ac_charge_hysteresis
```
Der Modus-Guard verhindert Fehlauslösung wenn `actual_power = 0` (Wechselrichter nicht im geregelten Betrieb).

### Fall H — Abbruch-Bedingung
```
Bedingung: Modus = '3'
       UND (soc >= soc_ac_charge_limit
            ODER (grid >= ac_charge_offset + hysteresis UND actual_power == 0))

  → ac_charge_state_helper = off, integral = 0
  → Zone 1: Timer-Toggle + Modus '1'
  → Zone 2: Modus '0' + Output 0W
```
`actual_power == 0`-Guard verhindert Fehlauslösung während der PI noch aktiv regelt.

### Fall I — Safety-Guard für Modus '3'
```
Bedingung: Modus = '3'
       UND (ac_charge_enabled = false
            ODER ac_charge_entity_valid = false
            ODER ac_charge_state_helper ≠ 'on')

Aktion: Integral = 0
        → Zyklus = on (Zone 1): Timer-Toggle + Modus '1'
        → Zyklus = off (Zone 2): Output 0W + Modus '0'
```
Fängt externe Modussetzung auf `'3'` ab wenn AC Laden nicht aktiv ist.

### AC-Lade-Zustand State-Machine
```
Eintritts-Bedingung (Fall G):
  ac_charge_enabled UND soc < soc_ac_charge_limit
  UND Modus ≠ '3' ← Guard: verhindert Re-Eintritt wenn AC Laden bereits aktiv (Modus = '3')
  → ac_charge_state_helper = on
  → Modus = '3', Output = 0W, Timer-Toggle

PI-Aufruf (Zweig B, jeder Trigger):
  ac_charge_mode_active = true UND |grid_error_abs| > tolerance
  → PI-Script mit ac_charge_mode=true, max_power=ac_charge_power_limit
  → separate P/I-Faktoren (ac_charge_p_factor, ac_charge_i_factor)

Abbruch-Bedingung (Fall H):
  mode = '3' UND (soc >= soc_ac_charge_limit
                  ODER (grid >= ac_charge_offset + hysteresis UND actual_power == 0))
  → ac_charge_state_helper = off, integral = 0
  → Zone 1: Timer-Toggle + Modus '1'
  → Zone 2: Modus '0' + Output 0W

Safety-Korrektur (Fall I):
  mode = '3' UND ac_charge nicht aktiv/legitimiert
  → integral = 0
  → Zone 1: Timer-Toggle + Modus '1'
  → Zone 2: Modus '0' + Output 0W
```

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 0 (Überschuss) | 2 A | Nur wenn abweichend |
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn abweichend und kein Surplus, kein AC Laden |
| Zone 2 / AC Laden (Modus 3) | 0 A | Nur wenn abweichend |

---

## ⚠️ Wichtige Hinweise

1. **Helper und Script vor Installation erstellen:** input_boolean (Zyklus), input_number (Integral), PI-Regler Script müssen existieren
2. **Optionale Helper erstellen wenn Funktion aktiviert:** Surplus-Boolean für Zone 0; AC-Lade-Boolean für AC Laden
3. **Kein input_select:** Ab V301 wird `input_boolean` für den Entladezyklus-Speicher verwendet
4. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
5. **AC Laden Regelziel:** Eigener konfigurierbarer Offset (`ac_charge_offset`), unabhängig von Zone 1/2. Negativer Wert = Einspeisung angestrebt → PI erhöht Ladeleistung.
6. **AC Laden P/I-Tuning:** Separate P/I-Faktoren (`ac_charge_p_factor`, `ac_charge_i_factor`). P-Faktor klein halten (~0.3–0.5) wegen langer Hardware-Flanke (~25 s) — der I-Anteil macht die eigentliche Regelarbeit.
7. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern
8. **Toleranz-Decay:** Verhindert Integral-Windup — 5% Abbau wenn `|Integral| > 10` und Fehler ≤ Toleranz
9. **Zone-0-Integral-Einfrieren:** In der Überschuss-Phase kein Decay, kein PI-Aufruf
10. **Recovery:** Modus-Verlust bei aktivem Zyklus wird automatisch erkannt (Fall D) — AC-Laden-Modus `'3'` wird dabei nicht überschrieben; bei aktivem AC-Lade-Bool wird `'3'` wiederhergestellt
11. **Fall G Guard:** Eintritt in AC Laden nur wenn Modus ≠ `'3'` — verhindert Re-Eintritt bei bereits aktivem AC Laden
12. **Modus-Werte:** `'0'` = Disabled, `'1'` = INV Discharge PV Priority, `'3'` = INV Charge PV Priority
13. **Fall I Safety:** Externer Modus-`'3'`-Zustand ohne aktive AC-Lade-Session wird automatisch korrigiert

---

## 🔄 Trigger-Übersicht

| Trigger | ID | Beschreibung |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Sofortige PI-Regelung bei Netzleistungsänderung |
| Solar Power Change | `solar_power_change` | Sofortige PI-Regelung bei PV-Leistungsänderung |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone-1-Schwelle) |
| SOC Low | `soc_low` | Zone 3 Start (SOC < Zone-3-Schwelle) |
| Mode Change | `mode_change` | Reagiert auf externe Modusänderungen, löst ggf. Recovery (Fall D) oder Safety-Korrektur (Fall I) aus |
