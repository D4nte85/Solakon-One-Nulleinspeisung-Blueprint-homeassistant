# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V303

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku**, optionalem **AC Laden aus externer Einspeisung** und optionaler **Tarif-Arbitrage** (günstig laden, Entladesperre bei niedrigem Tarif).

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fexperimental-AC-tarif%2Fsolakon_one_nulleinspeisung.yaml)

Der zugehörige **PI-Regler Script-Blueprint** muss ebenfalls importiert werden:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fexperimental-AC-tarif%2FPI-Regler.yaml)

---

## 🛠️ Vorbereitung: Erstellung der erforderlichen Helper

Der Blueprint benötigt **drei Pflicht-Helper** und ein **Script** sowie bis zu **drei optionale Helper**, die Sie vor der Installation erstellen müssen.

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

### 6. Input Boolean Helper (Tarif-Lade-Zustand) — NUR wenn Tarif-Arbitrage aktiv

Persistenter Zustandsspeicher für das Tarif-Laden. Steuert den direkten Leistungs-Setz-Zweig (BT) im PI-Gate und schützt Fall I vor fälschlicher Korrektur.

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer**
2. Klicken Sie auf **Helfer erstellen** → **Schalter** (Input Boolean)
3. Name: z.B. `Solakon Tarif Laden Aktiv`
4. Speichern (Entity ID: z.B. `input_boolean.solakon_tarif_laden_aktiv`)

### 7. Input Number Helper für Dynamischen Offset (Optional)

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
  - **Tarif-Laden (Modus 3):** Direkt gesetzter Wert — kein PI-Aufruf

* **PI-Aufruf-Guard (in Hauptautomatisierung):**
  - Zone 0 aktiv → PI nicht aufgerufen, Integral eingefroren
  - Tarif-Laden aktiv → direkt `tariff_charge_power` setzen, kein PI
  - AC Laden aktiv → PI mit `ac_charge_mode=true`, `at_max/at_min`-Guards deaktiviert
  - Normal → PI nur wenn `|Fehler| > Toleranz` UND kein At-Limit

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
| **0A** | Surplus-Bool = `off` UND SOC ≥ Export-Schwelle UND (PV > Output + Grid + PV-Hysterese ODER PV = 0) | Zone 0 Start: Surplus-Bool → `on` |
| **0B** | Surplus-Bool = `on` UND (SOC < Export-Schwelle − SOC-Hysterese ODER PV ≤ Output + Grid − PV-Hysterese) | Zone 0 Ende: Surplus-Bool → `off`, Integral = 0 |
| **A** | NICHT AC-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Surplus/AC-Bool zurücksetzen, Timer-Toggle, Modus → `'1'` |
| **B** | NICHT AC-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **C** | NICHT AC-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **D** | Zyklus = `on` UND Modus ∉ `{'1','3'}` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'3'` wenn AC-Lade-Bool **oder** Tarif-Lade-Bool = `on`, sonst `'1'` |
| **GT** | Tarif-Arbitrage aktiv UND Preis < Günstig-Schwelle UND SOC < Tarif-Ladeziel **UND Modus ≠ `'3'`** | Tarif-Laden Start: Tarif-Bool = `on`, Timer-Toggle, Output → Ladeleistung (direkt), Modus → `'3'` |
| **G** | AC aktiv UND SOC < Ladeziel **UND Modus ≠ `'3'`** UND NICHT Tarif-Lade-Bool = `on` UND (Grid + Output) < −Hysterese | AC Laden Start: AC-Bool = `on`, Timer-Toggle, Modus → `'3'`, Output → 0W |
| **HT** | Modus = `'3'` UND Tarif-Bool = `on` UND (Preis ≥ Günstig-Schwelle ODER SOC ≥ Tarif-Ladeziel) | Tarif-Laden Ende: Tarif-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **H** | Modus = `'3'` UND (SOC ≥ Ladeziel ODER (Grid ≥ `ac_charge_offset + Hysterese` UND Output = 0 W)) | AC Laden Ende: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **I** | Modus = `'3'` UND NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` | Safety-Korrektur: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **E** | NICHT AC-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Timer-Toggle, Modus → `'1'` |
| **F** | NICHT AC-Lade-Bool = `on` UND Nachtabschaltung aktiv UND PV < PV-Ladereserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Modus → `'0'`, Output → 0W |

> **Reihenfolge ist entscheidend:** Fall D liegt vor GT/G. Recovery erkennt `'3'` als legitimen Zustand und überschreibt ihn nicht. Fall GT liegt vor G — Tarif-Laden hat Vorrang, Fall G ist zusätzlich per Guard `NOT tariff_charge_bool = on` gesichert. Fall HT liegt vor H. Fall I liegt nach HT/H und prüft nun beide Bools — AC-Lade-Bool UND Tarif-Lade-Bool.

#### 🔄 Recovery-Mechanismus (Fall D)

Falls der Modus des Wechselrichters extern zurückgesetzt wird, während der Entladezyklus noch aktiv ist, erkennt der Blueprint diesen Zustand automatisch und reaktiviert den Modus — ohne Zonenwechsel oder Integral-Reset. Der wiederhergestellte Modus richtet sich nach dem Lade-Zustand: AC-Lade-Bool **oder** Tarif-Lade-Bool = `on` → Modus `'3'`; andernfalls Modus `'1'`.

#### ⚠️ Safety-Mechanismus (Fall I)

Fängt den Zustand ab, in dem der Wechselrichter Modus `'3'` hat, aber **keine** aktive Lade-Session vorliegt (weder AC-Laden noch Tarif-Laden aktiv). Aktion: Integral zurücksetzen, Zone 1 → Modus `'1'` (Timer-Toggle), Zone 2 → Modus `'0'` + Output 0W.

---

### 3. ☀️ Überschuss-Einspeisung (Zone 0, Optional)

Ermöglicht aktives Einspeisen von PV-Überschuss wenn der Akku voll ist. SOC- und PV-Hysterese verhindern instabiles Hin- und Herschalten.

* **Aktivierung:** Über den Parameter "Überschuss-Einspeisung aktivieren"
* **Eintritts-Bedingung:** SOC ≥ Export-Schwelle UND (PV > (Output + Grid + PV-Hysterese) ODER PV = 0)
* **Austritts-Bedingung:** SOC < (Export-Schwelle − SOC-Hysterese) ODER PV ≤ (Output + Grid − PV-Hysterese)
* **Persistenz:** Zustand wird in `input_boolean` gespeichert — überlebt mehrere Automation-Läufe
* **Verhalten:** Output auf Hard Limit, Entladestrom 2 A (Stabilitätspuffer), Integral eingefroren (kein Decay, kein PI-Aufruf)
* **Deaktiviert:** Klassische Nulleinspeisung — kein aktives Einspeisen

---

### 4. ⚡ AC Laden (Optional)

Laden der Batterie wenn eine externe Einspeisung ins Netz erkannt wird. Die Erkennung basiert auf `(Grid + Ausgangsleistung) < −Hysterese`. Typischer Anwendungsfall: externe PV-Anlage speist Überschuss ins Netz.

* **Eintritts-Bedingung (Fall G):**
  - AC Laden aktiviert UND SOC < Ladeziel
  - **Modus darf nicht `'3'` sein** (Guard verhindert Re-Eintritt wenn AC/Tarif-Laden bereits aktiv)
  - **NICHT Tarif-Lade-Bool = `on`** (Tarif-Laden hat Vorrang)
  - (Grid + Ausgangsleistung) < −Hysterese
* **PI-Regelung:** Script mit `ac_charge_mode=true` → invertierte Fehlerberechnung
* **Separate P/I-Faktoren:** P klein (~0.3–0.5), I macht die eigentliche Regelarbeit
* **Rückkehr:** Zone 1 → `'1'` / Zone 2 → `'0'`
* **Priorität:** Zone 3 (SOC-Schutz) hat immer Vorrang

---

### 5. 💹 Tarif-Arbitrage (Optional)

Lädt die Batterie bei günstigem Strompreis und sperrt die Entladung bei günstigem oder neutralem Tarif. Vollständig unabhängig von den AC-Laden-Parametern konfigurierbar.

#### Preiszonen

| Preis | Verhalten |
|:------|:----------|
| `Preis < günstig_threshold` | Tarif-Laden aktiv (Fall GT) + Entladesperre (Falls A/E blockiert) |
| `günstig ≤ Preis < teuer_threshold` | Nur Entladesperre — Akku passiv, keine Entladung, kein aktives Laden |
| `Preis ≥ teuer_threshold` | Normale SOC-Zonen-Logik — kein Eingriff der Tarif-Logik |

#### Tarif-Laden (Fall GT)

* **Eintritts-Bedingung:** Tarif-Arbitrage aktiviert UND Preis < Günstig-Schwelle UND SOC < Tarif-Ladeziel UND Modus ≠ `'3'`
* **Verhalten:** Direkt konfigurierte Ladeleistung setzen (`tariff_charge_power`) — kein PI-Regler, kein Toleranz-Check
* **Abbruch (Fall HT):** Preis steigt über Günstig-Schwelle ODER SOC-Ladeziel erreicht
* **Rückkehr:** Zone 1 → Timer-Toggle + Modus `'1'` / Zone 2 → Modus `'0'` + Output 0W
* **Priorität:** Tarif-Laden (GT) liegt vor AC-Laden (G) im choose-Block

#### Entladesperre (Guards A/E)

* Falls A und E enthalten den Guard `NOT (tariff_arbitrage_enabled AND Preis < expensive_threshold)`
* Solange der Preis unter der Teuer-Schwelle liegt, können Zone 1 und Zone 2 nicht betreten werden
* Zone 3 (SOC-Schutz) ist davon unabhängig — greift immer

#### Sensor-Flexibilität

Der Blueprint nimmt einen generischen `sensor` und vergleicht dessen Zustand direkt mit den konfigurierten Schwellwerten — ohne Einheitskonvertierung. Tibber (EUR/kWh), Awattar (ct/kWh) oder eigene `input_number` können verwendet werden, solange Sensor-Einheit und Schwellwerte übereinstimmen. Bei nicht verfügbarem Sensor (`unknown`/`unavailable`) werden weder Laden noch Sperrung aktiv.

* **Zustandsspeicher:** `input_boolean` (`tariff_charge_state_helper`) persistent
* **Recovery:** Fall D erkennt Tarif-Lade-Bool = `on` und stellt Modus `'3'` wieder her
* **Safety:** Fall I prüft nun beide Bools — greift nur wenn weder AC- noch Tarif-Laden aktiv

---

### 6. ⏱️ Timer-Toggle und Moduswechsel-Sequenz

Um die stabile Übernahme von Moduswechseln durch den Solakon ONE zu gewährleisten, wird statt eines festen Werts ein **Toggle zwischen 3598 und 3599** gesendet. Dies erzeugt eine Zustandsänderung, die den Wechselrichter zur sicheren Übernahme des neuen Modus veranlasst. Der Toggle wird bei jedem Moduswechsel (Falls A, D, E, GT, G, HT-Zone1, H-Zone1, I-Zone1) direkt vor dem Setzen des Modus durchgeführt.

**Kontinuierlicher Timeout-Reset (Schritt 2):** Countdown < 120s → Timer-Toggle.

---

### 7. 🌙 Nachtabschaltung (Optional)

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
| **Helper** | Tarif-Lade-Zustand-Speicher | `input_boolean.solakon_tarif_laden_aktiv` | Nur wenn Tarif-Arbitrage aktiv. |

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.3 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine PI-Korrektur innerhalb (stattdessen Integral-Decay). |
| **Wartezeit** | 3 s | 0 | 30 s | Verzögerung nach Leistungsänderung in allen Modi. Kompensiert die Reaktionszeit des Wechselrichters. |

> **Hinweis:** P- und I-Faktor gelten für Zone 1 und Zone 2. Für den AC-Lade-Modus werden separate Faktoren verwendet. Für Tarif-Laden wird kein PI-Regler verwendet.

---

### 🔋 SOC-Zonen-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Überschreiten aktiviert Zone 1. |
| **Zone 3 Stopp** | 20 % | 1 % | 49 % | Unterschreiten stoppt Entladung komplett. |
| **Max. Entladestrom Zone 1** | 40 A | 0 A | 40 A | Zone 2 und AC/Tarif-Laden nutzen automatisch 0 A. |

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
| **Hysterese Ladeabbruch** | 50 W | 0 | 300 W | Totband für Ein- und Austritt. |
| **AC Laden Offset (Statisch)** | -50 W | -100 | 100 W | Regelziel im AC-Lade-Modus. Negativ = Einspeisung angestrebt. |
| **AC Laden Offset (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **AC Laden P-Faktor** | 0.5 | 0.1 | 5.0 | Proportional-Verstärkung im AC-Lade-Modus. Klein halten (~0.3–0.5). |
| **AC Laden I-Faktor** | 0.07 | 0.01 | 0.2 | Integral-Verstärkung im AC-Lade-Modus. Macht die eigentliche Regelarbeit. |

---

### 💹 Tarif-Arbitrage (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Tarif-Arbitrage aktivieren** | false | — | — | Schalter für die gesamte Tarif-Logik. |
| **Strompreis-Sensor** | *(leer)* | — | — | Generischer Sensor. Einheit muss zu den Schwellwerten passen. |
| **Günstig-Schwelle** | 10 | -100 | 100 | Unter diesem Wert: Laden + Entladesperre. |
| **Teuer-Schwelle** | 25 | -100 | 100 | Ab diesem Wert: Entladesperre aufgehoben, normale SOC-Logik. |
| **SOC-Ladeziel Tarif-Laden** | 90 % | 10 % | 99 % | Tarif-Laden stoppt bei diesem SOC. Unabhängig vom AC-Laden-Ladeziel. |
| **Ladeleistung Tarif-Laden** | 800 W | 50 | 1200 W | Direkt gesetzter Wert — kein PI-Regler. Z.B. Geräte-Maximum 1200 W. |

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

---

## 🔧 PI-Regler Einstellung

Der Regler wird in drei Schritten eingestellt. Ziel ist ein System, das schnell und präzise auf Änderungen reagiert, aber nicht dauerhaft hin- und herschwingt.

### Schritt 1: Wartezeit finden (P = 1, I = 0)

Die **Wartezeit** deckt Wechselrichter-Reaktion und Messlatenz ab. Sinnvolle Wartezeit: 1–3 s.

### Schritt 2: P-Faktor finden (I = 0)

```yaml
P-Faktor: 0.3   # Startpunkt
I-Faktor: 0.0
```

Schrittweise erhöhen bis System leicht anfängt zu zittern — dann einen Schritt zurück. Typischer Arbeitsbereich: **0.8–1.5**.

### Schritt 3: I-Faktor hinzufügen

```yaml
I-Faktor: 0.02   # Startpunkt
```

Typischer Arbeitsbereich: **0.03–0.08**. Für AC Laden separat tunen — P besonders klein halten (~0.3–0.5). Tarif-Laden verwendet keinen PI-Regler.

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

1. **Hauptautomatisierung** (`solakon_one_nulleinspeisung.yaml`): Zonen-Steuerung, SOC-Logik, Surplus/AC/Tarif-Lade-Zustand, Entladestrom-Verwaltung, Timeout-Reset, PI-Aufruf-Guard, Integral-Decay/-Einfrieren
2. **PI-Regler Script** (`PI-Regler.yaml`): Reine Berechnungslogik — wird für AC-Laden genutzt, nicht für Tarif-Laden

### Tarif-Arbitrage State-Machine

```
Eintritts-Bedingung (Fall GT):
  tariff_arbitrage_enabled UND Preis < cheap_threshold
  UND soc < tariff_soc_charge_target
  UND Modus ≠ '3' ← Guard: verhindert Re-Eintritt
  → tariff_charge_state_helper = on
  → Output = tariff_charge_power (direkt, kein PI)
  → Modus = '3', Timer-Toggle

Direktes Setzen (Zweig BT, jeder Trigger):
  tariff_charge_mode_active = true
  → number.set_value → tariff_charge_power
  → delay wait_time

Abbruch-Bedingung (Fall HT):
  Modus = '3' UND tariff_charge_bool = on
  UND (soc >= tariff_soc_charge_target ODER Preis >= cheap_threshold)
  → tariff_charge_state_helper = off, integral = 0
  → Zone 1: Timer-Toggle + Modus '1'
  → Zone 2: Modus '0' + Output 0W

Entladesperre (Guards A/E):
  price_discharge_locked = tariff_arbitrage_enabled
                           UND price_sensor_valid
                           UND Preis < expensive_threshold
  → Fall A blockiert (Zone 1 kann nicht starten)
  → Fall E blockiert (Zone 2 kann nicht starten)

Fallback bei Sensor unavailable:
  current_price_float = expensive_threshold + 1
  → price_is_cheap = false (kein Laden)
  → price_discharge_locked = false (keine Sperre)
```

### Fall D / Recovery — erweiterter Modus-Ausschluss

```
Bedingung: Zyklus = on
       UND Modus ∉ {'1', '3'}
       UND SOC > Zone-3-Schwelle

Aktion: Timer-Toggle
        → AC-Lade-Bool = on ODER Tarif-Lade-Bool = on: Modus '3'
        → sonst: Modus '1'
```

### Fall I — erweiterter Safety-Guard

```
Bedingung: Modus = '3'
       UND (ac_charge deaktiviert ODER ac_charge_entity_valid = false
            ODER ac_charge_state_helper ≠ 'on')
       UND (tariff_arbitrage deaktiviert ODER tariff_charge_entity_valid = false
            ODER tariff_charge_state_helper ≠ 'on')
```

### Dynamisches Power-Limit

```
Modus '3' + tariff_charge_mode_active:  max_power = tariff_charge_power (Zweig BT, kein PI)
Modus '3' + ac_charge_mode_active:      max_power = ac_charge_power_limit (PI mit ac_charge_mode=true)
Zone 1 (cycle = on):                    max_power = hard_limit
Zone 2 (cycle = off):                   max_power = Max(0, PV - pv_charge_reserve)
```

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 0 (Überschuss) | 2 A | Nur wenn abweichend |
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn abweichend und kein Surplus, kein AC/Tarif-Laden |
| Zone 2 / AC/Tarif-Laden (Modus 3) | 0 A | Nur wenn abweichend |

---

## ⚠️ Wichtige Hinweise

1. **Helper und Script vor Installation erstellen:** input_boolean (Zyklus), input_number (Integral), PI-Regler Script müssen existieren
2. **Optionale Helper erstellen wenn Funktion aktiviert:** Surplus-Boolean für Zone 0; AC-Lade-Boolean für AC Laden; Tarif-Lade-Boolean für Tarif-Arbitrage
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **Tarif-Sensor Einheit:** Sensor-Wert und Schwellwerte müssen dieselbe Einheit verwenden — kein Blueprint-seitiges Umrechnen
5. **Tarif-Laden kein PI:** Direkt gesetzter Wert — kein Toleranz-Check, kein Integral-Einfluss
6. **Entladesperre Schwelle:** `expensive_threshold` steuert die Sperre (unterhalb = gesperrt), `cheap_threshold` steuert zusätzlich das aktive Laden
7. **Fallback bei Sensor-Ausfall:** Weder Laden noch Sperre — System verhält sich wie ohne Tarif-Arbitrage
8. **AC Laden vs. Tarif-Laden:** Beide vollständig unabhängig konfigurierbar (eigene SOC-Ziele, eigene Leistungen). Tarif-Laden hat Vorrang wenn beide gleichzeitig zutreffen würden
9. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern
10. **Recovery:** Modus-Verlust bei aktivem Zyklus wird automatisch erkannt (Fall D) — sowohl AC- als auch Tarif-Lade-Modus `'3'` wird korrekt wiederhergestellt
11. **Fall I Safety:** Jetzt doppelt gesichert — greift nur wenn weder AC-Laden noch Tarif-Laden aktiv ist

---

## 🔄 Trigger-Übersicht

| Trigger | ID | Beschreibung |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Sofortige Regelung bei Netzleistungsänderung |
| Solar Power Change | `solar_power_change` | Sofortige Regelung bei PV-Leistungsänderung |
| SOC High | `soc_high` | Zone 1 Start (SOC > Zone-1-Schwelle) |
| SOC Low | `soc_low` | Zone 3 Start (SOC < Zone-3-Schwelle) |
| Mode Change | `mode_change` | Reagiert auf externe Modusänderungen, löst ggf. Recovery (Fall D) oder Safety-Korrektur (Fall I) aus |

> **Hinweis Tarif-Trigger:** Ein separater Preis-Trigger ist in Blueprint-Automationen mit optionalen Entitäten nicht sauber realisierbar. Die Tarif-Logik greift beim nächsten regulären Grid/PV-Trigger. Da diese bei aktiver PV sehr häufig feuern, ist die Reaktionsverzögerung bei Preisübergängen vernachlässigbar.
