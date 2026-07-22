# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V307

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku**, optionalem **AC Laden aus externer Einspeisung** und optionaler **Tarif-Arbitrage** (günstig laden, Entladesperre bei niedrigem Tarif).

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

Der zugehörige **PI-Regler Script-Blueprint** muss ebenfalls importiert werden:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fmain%2FPI-Regler.yaml)

Für das Multi instancing:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fmain%2Fsolakon_leistungsverteilung.yaml)

---
| Artefakt | Instanz 1 | Instanz 2 |
| :--- | :--- | :--- |
| `input_boolean` Zyklus | `...entladezyklus_status_i1` | `...entladezyklus_status_i2` |
| `input_number` Integral | `...integral_i1` | `...integral_i2` |
| `input_boolean` Surplus | `...surplus_aktiv_i1` | `...surplus_aktiv_i2` |
| `input_boolean` AC Laden | `...ac_laden_aktiv_i1` | `...ac_laden_aktiv_i2` |
| `input_boolean` Tarif Laden | `...tarif_laden_aktiv_i1` | `...tarif_laden_aktiv_i2` |
| PI-Regler Script | `script.pi_regler_i1` | `script.pi_regler_i2` |
| `input_number` Hard Limit | `...instanz1_limit` | `...instanz2_limit` |
| **NEU** `input_number` Fehler-Anteil | `...instanz1_share` | `...instanz2_share` |

> **Hinweis:** Der Netz-Sensor (Shelly) wird von beiden Instanzen gleichzeitig gelesen — kein Problem, da nur lesend.
...

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

Persistenter Zustandsspeicher für das AC Laden. Signalisiert dem PI-Script die invertierte Fehlerberechnung (`ac_charge_mode=true`). Verhindert außerdem, dass die Guards (at\_max/at\_min) den PI fälschlich blockieren.

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

### 7. Input Number Helper (Leistungslimit Dynamisch) — NUR für Multi-Instancing

Wird von der Leistungsverteilungs-Automation beschrieben und begrenzt den Ausgangsleistungs-Hardlimit der Instanz dynamisch.

1. **Einstellungen:** Min: `0`, Max: `≥ Global-Max`, Step: `1`, Initialwert: `800`
2. In der Leistungsverteilung als „Max. Leistung (input_number)" eintragen
3. In der Instanz-Automation als „Max. Ausgangsleistung — Dynamisch" eintragen

### 8. Input Number Helper (Fehler-Anteil) — NUR für Multi-Instancing

Enthält den Anteil des Netzfehlers (0.0–1.0), den diese Instanz über ihren PI-Regler übernimmt.
Wird von der Leistungsverteilungs-Automation berechnet: `usable_i / Σ usable_j` mit `usable_i = (SOC_i − Min-SOC_i) / 100 × Kap_i`. Ohne Kapazitätssensor gilt `Kap_i = 100` — reine SOC-%-Gewichtung.
Gilt nur für Zone 1/Zone 2 (Nulleinspeisung, Modus '1'). Für AC-Laden siehe Punkt 8b.

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer** → **Number**
2. Name: z.B. `Solakon Instanz 1 Share`
3. **Einstellungen:** Min: `0`, Max: `1`, Step: `0.001`, Initialwert: `1`
4. Speichern (Entity ID: z.B. `input_number.solakon_instanz1_share`)
5. Wiederholen für jede weitere Instanz
6. In der Leistungsverteilung als „Fehler-Anteil Helfer" eintragen
7. In der Instanz-Automation als „Fehler-Anteil Helfer" eintragen

### 8b. Input Number Helper (AC-Lade Fehler-Anteil) — NUR für Multi-Instancing MIT AC Laden

Eigener Fehler-Anteil-Pool für AC-Laden (Modus '3'), getrennt vom Helper aus Punkt 8. Notwendig,
weil eine ladende Instanz nicht in Modus '1' steht und im Nulleinspeisungs-Pool sonst
`error_share = 0` bekäme — das würde den AC-Lade-PI auf 0 W einfrieren, obwohl aktiv Ladebedarf
besteht. Die Leistungsverteilungs-Automation berechnet den Anteil nur unter den gerade
gleichzeitig AC-ladenden Instanzen (eigener Pool, unabhängig von Punkt 8).

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer** → **Number**
2. Name: z.B. `Solakon Instanz 1 AC Share`
3. **Einstellungen:** Min: `0`, Max: `1`, Step: `0.001`, Initialwert: `1`
4. Speichern (Entity ID: z.B. `input_number.solakon_instanz1_ac_share`)
5. Wiederholen für jede weitere Instanz mit AC-Laden
6. In der Leistungsverteilung als „AC-Lade Fehler-Anteil Helfer" eintragen (zusammen mit dem
   AC-Lade-Zustand-Helfer aus Punkt 5)
7. In der Instanz-Automation als „AC-Lade Fehler-Anteil Helfer" eintragen

### 9. Input Number Helper für Dynamischen Offset (Optional)

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

* **P-Anteil:** Reagiert sofort auf aktuelle Abweichungen. Konfigurierbare Aggressivität über den P-Faktor.
* **I-Anteil:** Summiert Abweichungen über die Zeit auf, eliminiert bleibende Regelabweichungen. Anti-Windup via Back-Calculation: Integral wird nach jedem Eingriff auf den Wert korrigiert, der den tatsächlichen (ggf. geklemmten) Ausgang produziert — Clamp auf ±effective_max, gerundet auf 2 Nachkommastellen. Automatischer Reset bei Zonenwechsel. Toleranz-Decay 5%/Zyklus wenn Fehler ≤ Toleranz und |Integral| > 10. Zone-0-Einfrieren bei aktivem Überschuss.
* **Fehlerberechnung:** Normal (`ac_charge_mode=false`): `raw_error = (grid − target_offset) × error_share`. AC Laden (`ac_charge_mode=true`): `raw_error = (target_offset − grid) × error_share` (invertiert). `error_share` skaliert den Fehler auf den Anteil dieser Instanz (Standard 1.0 = voller Fehler). Zwei unabhängige Pools im Multi-Instanz-Betrieb: Nulleinspeisung (`error_share_entity`) und AC-Laden (`ac_error_share_entity`) — siehe Multi-Instancing-Abschnitt.
* **Dynamisches Power-Limit:** Zone 1 → Hard Limit. Zone 2 → `Min(Hard Limit, Max(0, PV − Reserve))`. AC Laden → konfigurierbares Lade-Limit. Tarif-Laden → kein PI (direkter Wert).
* **PI-Aufruf-Guard:** Zone 0 aktiv → PI nicht aufgerufen, Integral eingefroren. Tarif-Laden aktiv → direkt setzen. AC Laden aktiv → PI mit `ac_charge_mode=true`. Normal → PI nur wenn `|Fehler| > Toleranz` UND kein At-Limit. `at_max_limit` = false wenn `current > dynamic_max` (PV-Einbruch) → PI kann nach unten korrigieren.

---

### 2. 🔋 SOC-Zonen-Logik

| Zone | SOC-Bereich / Bedingung | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:------------------------|:------|:-----------------|:---------|:--------------|
| **0 — Überschuss-Einspeisung** | SOC ≥ Export-Schwelle UND PV > Output + Grid + PV-Hysterese | `'1'` | 2 A | Hard Limit | **Optional.** Integral eingefroren. Blockiert GT und G. |
| **1 — Aggressive Entladung** | SOC > Zone-1-Schwelle | `'1'` | Konfigurierter Max-Wert | 0W + Offset 1 | Läuft **bis SOC ≤ Zone-3-Schwelle**. Auch nachts aktiv. |
| **2 — Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `'1'` | **0 A** | 0W + Offset 2 | Dynamisches Limit: `Min(Hard Limit, Max(0, PV − Reserve))`. Optional: Nachtabschaltung. |
| **3 — Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `'0'` | 0 A | — | Output = 0 W. Vollständiger Batterieschutz. Absoluter Vorrang. |

---

### 3. ⚡ Priorität der optionalen Module

Die optionalen Module überschreiben die SOC-Zonen-Logik in folgender Reihenfolge:

```
Zone 3 (Sicherheit, absolut) > Zone 0 (Überschuss) > Tarif-Laden > AC-Laden > SOC-Zonen
```

**Zone 3** hat absoluten Vorrang — greift unabhängig von allen anderen Modi.

**Zone 0 (Überschuss)** blockiert Tarif-Laden (Fall GT), den Discharge-Lock (Fall TM), AC-Laden (Fall G) und die Nachtabschaltung (Fall F) solange aktiv. Zone 0 kann nur starten wenn kein Ladevorgang aktiv ist.

**Tarif-Laden (Fall GT)** startet nur wenn Zone 0 NICHT aktiv ist. Sperrt Zone 1 **und** Zone 2: Für die mittlere Preiszone (günstig ≤ Preis < teuer) wird laufende Entladung sowohl aus Zone 1 als auch Zone 2 gestoppt (Fall TM) — außer Zone 0 ist aktiv, dann läuft die Überschuss-Einspeisung ungestört weiter. Der Zyklus-Helper wird dabei zurückgesetzt; beim nächsten teuren Tarif starten Falls A/E neu. Für günstige Preise gilt zusätzlich aktives Laden (Fall GT).

**AC-Laden (Fall G)** startet nur wenn weder Zone 0 noch Tarif-Laden aktiv ist.

**SOC-Zonen (Falls A/E)** starten nur wenn keine der obigen Prioritäten greift und die Entladesperre (Preis < teuer) nicht aktiv ist.

---

### 4. Übersicht der Steuer-Falls (choose-Block)

Die Reihenfolge ist entscheidend — der erste zutreffende Fall wird ausgeführt.

| Fall | Bedingung | Aktion |
|:-----|:----------|:-------|
| **0A** | Surplus-Bool = `off` UND (SOC ≥ Export-Schwelle UND (PV > Output + Grid + PV-Hysterese ODER PV = 0) **ODER** Surplus-Forecast-Forced) | Zone 0 Start: Surplus-Bool → `on` |
| **0B** | Surplus-Bool = `on` UND **NICHT Surplus-Forecast-Forced** UND (SOC < Export-Schwelle − SOC-Hysterese ODER (PV ≤ Output + Grid − PV-Hysterese UND **NICHT Austritts-Sperre**)) | Zone 0 Ende: Surplus-Bool → `off`, Integral = 0 |
| **A** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Surplus/AC-Bool zurücksetzen, Timer-Toggle, Modus → `'1'` |
| **B** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Surplus/AC-Bool zurücksetzen, Output → 0W, Timer-Toggle, Modus → `'0'` |
| **C** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Surplus/AC-Bool zurücksetzen, Output → 0W, Timer-Toggle, Modus → `'0'` |
| **D** | Zyklus = `on` UND Modus ∉ `{'1','3'}` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'3'` wenn AC-Lade-Bool **oder** Tarif-Lade-Bool = `on`, sonst `'1'` |
| **GT** | Tarif-Arbitrage aktiv UND Preis < Günstig-Schwelle UND SOC < Tarif-Ladeziel UND **Modus ≠ `'3'`** UND **NICHT Surplus-Bool = `on`** UND **NICHT PV-Forecast-Suppressed** | Tarif-Laden Start: Tarif-Bool = `on`, Timer-Toggle, Output → Ladeleistung (direkt), Modus → `'3'` |
| **HT** | Modus = `'3'` UND Tarif-Bool = `on` UND (Preis ≥ Günstig-Schwelle ODER SOC ≥ Tarif-Ladeziel) | Tarif-Laden Ende: Tarif-Bool = `off`, Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` (Timer-Toggle) |
| **TM** | Tarif aktiv UND Günstig ≤ Preis < Teuer-Schwelle UND kein AC/Tarif-Laden UND **NICHT Surplus-Bool = `on`** UND **Modus = `'1'`** UND **NICHT PV-Forecast-Suppressed** | Discharge-Lock: Integral = 0, Zyklus = `off` (wenn aktiv), Output → 0W, Timer-Toggle, Modus → `'0'` |
| **G** | AC aktiv UND SOC < Ladeziel UND **Modus ≠ `'3'`** UND NICHT Tarif-Lade-Bool = `on` UND **NICHT Surplus-Bool = `on`** UND (Grid + Output) < −Hysterese | AC Laden Start: AC-Bool = `on`, Timer-Toggle, Modus → `'3'`, Output → 0W |
| **H** | Modus = `'3'` UND (SOC ≥ Ladeziel ODER (Grid ≥ `ac_charge_offset + Hysterese` UND Output = 0 W)) | AC Laden Ende: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` (Timer-Toggle) |
| **I** | Modus = `'3'` UND NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` | Safety-Korrektur: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W (Timer-Toggle) |
| **E** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Output → 0W, Timer-Toggle, Modus → `'1'` |
| **F** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND NICHT Surplus-Bool = `on` UND Nachtabschaltung aktiv UND PV < PV-Ladereserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Output → 0W, Timer-Toggle, Modus → `'0'` |

---

### 5. ☀️ Überschuss-Einspeisung (Zone 0, Optional)

Ermöglicht aktives Einspeisen von PV-Überschuss wenn der Akku voll ist. SOC- und PV-Hysterese verhindern instabiles Hin- und Herschalten.

* **Aktivierung:** Über den Parameter "Überschuss-Einspeisung aktivieren"
* **Eintritts-Bedingung:** SOC ≥ Export-Schwelle UND (PV > (Output + Grid + PV-Hysterese) ODER PV = 0)
* **Austritts-Bedingung:** (PV ≤ (Output + Grid − PV-Hysterese) UND NICHT Austritts-Sperre) ODER SOC < (Export-Schwelle − SOC-Hysterese) — beide Terme werden blockiert solange Surplus-Forecast forciert (siehe Abschnitt 11); die optionale Austritts-Sperre (siehe Abschnitt 12) blockiert nur den PV-Term
* **Blockiert:** Tarif-Laden (Fall GT) und AC-Laden (Fall G) können nicht starten solange Zone 0 aktiv ist.
* **Verhalten:** Output auf Hard Limit, Entladestrom 2 A (Stabilitätspuffer), Integral eingefroren.
* **Deaktiviert:** Klassische Nulleinspeisung — kein aktives Einspeisen.

**Warum die Export-Schwelle unter dem Vollladepunkt liegen muss:** Der Eintritt prüft `PV > Verbrauch + Hysterese`. Das ist nur messbar, solange der Akku noch lädt — dann läuft die PV ungedrosselt und zeigt `Verbrauch + Ladeleistung`. Am Vollladepunkt (App-Ladeobergrenze) drosselt der Wechselrichter die PV exakt auf den Eigenbedarf herunter; der Überschuss ist dann unsichtbar und der Eintritt hängt von zufälligen Verbrauchsschwankungen ab — minutenlange Verzögerung möglich. Eine Schwelle ~5 % unter der App-Ladeobergrenze (z.B. 90–95 % bei Max 100 %) legt den Eintritt sicher in die Ladephase, wo der Überschuss zuverlässig messbar ist. Aus demselben Grund kann der Wiedereintritt nach einer Wolke verzögert sein, wenn der SOC bereits am Maximum gepinnt ist — während der Wolke wird die Batterie nicht entladen (solange Solar existiert, bleibt sie unangetastet), der SOC bewegt sich nicht. Dagegen hilft die Austritts-Sperre (Abschnitt 12).

---

### 6. ⚡ AC Laden (Optional)

Laden der Batterie wenn eine externe Einspeisung ins Netz erkannt wird. Eintritts-Erkennung: `(Grid + Ausgangsleistung) < −Hysterese`.

* **Blockiert durch:** Zone 0 (Überschuss-Bool = `on`) und Tarif-Laden (Tarif-Bool = `on`).
* **Eintritts-Bedingung (Fall G):** AC Laden aktiviert UND SOC < Ladeziel UND Modus ≠ `'3'` UND NICHT Tarif-Lade-Bool = `on` UND **NICHT Surplus-Bool = `on`** UND (Grid + Output) < −Hysterese.
* **PI-Regelung:** `ac_charge_mode=true` → invertierte Fehlerberechnung: `target_offset − grid`. Separate P/I-Faktoren. P klein halten (~0.3–0.5), I auf 0 belassen (Hardware zu träge).
* **Rückkehr:** Zone 1 → Modus `'1'` (Timer-Toggle) + Integral Reset. Zone 2 → Modus `'0'` (Timer-Toggle) + Output 0W + Integral Reset.

---

### 7. 💹 Tarif-Arbitrage (Optional)

Optimiert die Speichernutzung basierend auf deinem dynamischen Stromtarif. Diese Logik lädt die Batterie bei extrem günstigen Preisen gezielt aus dem Netz und sperrt die Entladung in "neutralen" Preisphasen, um die Energie für teure Spitzenzeiten aufzusparen.

#### Preis-Steuerung (Statisch & Dynamisch)

Du kannst für beide Schwellenwerte entweder feste Zahlenwerte nutzen oder **dynamische Helfer** (`input_number`) hinterlegen. Ein gesetzter Helfer überschreibt immer den statischen Wert.

| Schwelle | Verhalten bei Preis-Unterschreitung | Override-Möglichkeit |
|:---------|:------------------------------------|:---------------------|
| **Günstig-Schwelle** | **Tarif-Laden aktiv** (GT) + Entladesperre | `input_number` (z.B. Tages-Min + 2ct) |
| **Teuer-Schwelle** | **Entladesperre** — Zone 1 & 2 werden gestoppt (TM) | `input_number` (z.B. Tages-Schnitt) |

#### Die Preiszonen im Detail

| Aktueller Preis | Verhalten der Automatisierung |
|:----------------|:------------------------------|
| `Preis < Günstig-Schwelle` | **Fall GT (Tarif-Laden):** Akku lädt mit `tariff_charge_power`. Entladung blockiert. |
| `Günstig ≤ Preis < Teuer` | **Fall TM (Entladesperre):** Akku passiv. Zone 1 & 2 werden gestoppt. |
| `Preis ≥ Teuer-Schwelle` | **Normalbetrieb:** Die Standard-SOC-Zonen-Logik regelt die Einspeisung. |

#### Tarif-Laden (Fall GT)

* **Eintritts-Bedingung:** Tarif-Arbitrage aktiviert **UND** Preis < Günstig-Schwelle **UND** SOC < Ziel-SOC **UND** Modus ≠ `'3'` **UND** kein Überschuss-Laden aktiv.
* **Verhalten:** Setzt die konfigurierte Ladeleistung (`tariff_charge_power`) — kein PI-Regler, kein Toleranz-Check.
* **Abbruch:** Preis steigt über Günstig-Schwelle **ODER** SOC-Ladeziel erreicht.
* **Rückkehr:** Zone 1 → Timer-Toggle + Modus `'1'` / Zone 2 → Timer-Toggle + Modus `'0'` + Output 0W.
* **Priorität:** Tarif-Laden (GT) liegt vor AC-Laden (G) im choose-Block.

#### Entladesperre / Discharge-Lock (Fall TM)

* **Bedingung:** Tarif aktiv **UND** Günstig ≤ Preis < Teuer **UND** kein AC/Tarif-Laden **UND** kein Überschuss (Zone 0 hat Vorrang) **UND** Modus = `'1'`.
* **Wirkung:** Stoppt sofort jede Entladung in Zone 1 und Zone 2. Der Wechselrichter wird auf 0W (Modus `'0'`) gesetzt.
* **Zone-1-Besonderheit:** Zyklus-Helper wird auf `off` zurückgesetzt. Fall D kann den Modus dadurch nicht sofort wiederherstellen.
* **Reaktivierung:** Wenn der Preis die Teuer-Schwelle überschreitet, können Falls A und E wieder feuern (Entladesperre aufgehoben).

#### ⚠️ Wichtiger Hinweis zu Sensoren & Einheiten

Der Blueprint ist **sensor-agnostisch**. Er führt keine automatische Skalierung durch (z. B. von Euro in Cent).
* **Einheitlichkeit:** Die Einheit deines Preissensors (z. B. von Tibber oder Nordpool) **muss** mit der Einheit deiner Schwellenwerte und Helfer identisch sein.
* **Beispiel:** Liefert dein Sensor `0.34` (€), nutze als Schwelle `0.20`. Liefert er `34.0` (ct), nutze `20.0`.
* **Ausfallsicherheit:** Bei Status `unavailable` oder `unknown` des Preissensors wird die Tarif-Logik komplett ignoriert (Sicherheits-Fallback).

---

### 8. ⏱️ Timer-Toggle und Moduswechsel-Sequenz

Statt eines festen Werts wird ein **Toggle zwischen 3598 und 3599** gesendet. Dies erzeugt eine Zustandsänderung, die den Wechselrichter zur sicheren Übernahme des neuen Modus veranlasst. Der Toggle wird bei jedem Moduswechsel (Falls A, D, E, GT, G, HT-Zone1, H-Zone1, I-Zone1) direkt vor dem Setzen des Modus durchgeführt.

**Kontinuierlicher Timeout-Reset (Schritt 2):** Countdown < 120s → Timer-Toggle.

---

### 9. 🌙 Nachtabschaltung (Optional)

Betrifft **nur Zone 2** (Fall F). Zone 1, AC Laden und Überschuss-Einspeisung (Zone 0) laufen auch nachts weiter — Zone 0 hat Vorrang vor der Nachtabschaltung.

* **Schwelle:** PV-Leistung unter dem Wert der **PV-Ladereserve** (kein separater Parameter)
* **Verhalten:** Output 0 W, Timer-Toggle, Modus → `'0'`, Integral reset
* **Zone 2 Reaktivierung:** Sobald PV wieder über die PV-Ladereserve steigt, greift Fall E

---

### 10. 🌤️ PV-Forecast Tarif-Unterdrückung (Optional)

Verhindert Tarif-Laden und Discharge-Lock an Tagen, an denen die PV-Prognose ausreichend ist.

* **Voraussetzung:** Tarif-Arbitrage muss ebenfalls aktiviert sein.
* **Funktion:** Wenn der konfigurierte PV-Forecast-Sensor ≥ Schwelle → Fall GT und Fall TM werden übersprungen. Der Akku kann an sonnigen Tagen normal entladen und muss nicht durch Tarifsignale blockiert werden.
* **Sensor:** Z.B. Solcast `energy_production_today` oder ähnlicher Tages-/Stunden-Forecast in W.
* **Fallback:** Sensor unavailable/unknown → Unterdrückung inaktiv (Sicherheits-Fallback: Tarif-Logik greift normal).

### 11. 🌤️ Surplus-Forecast erzwungener Eintritt (Optional)

Erzwingt frühzeitigen Zone-0-Eintritt auf Basis einer PV-Überschuss-Prognose.

* **Voraussetzung:** Überschuss-Einspeisung muss ebenfalls aktiviert sein.
* **Forcierungs-Flag:** `surplus_forecast_forced = (Forecast ≥ Schwelle) UND (Solar > Hard Limit) UND (SOC > Zone-3-Schwelle)` — an ein echtes Abregel-Risiko gekoppelt, nicht am rohen Vorhersagewert allein. So bleibt die Forcierung nur so lange aktiv, wie tatsächlich mehr PV anliegt als das Hard Limit zulässt. Die SOC-Untergrenze verhindert, dass die Forcierung bei tiefentladener Batterie gegen den Zone-3-Sicherheitsstopp ankämpft (Flattern 0A ↔ C).
* **Eintritt (Fall 0A, OR-Branch):** Wenn `surplus_forecast_forced` → Zone-0-Eintritt **ohne Export-Schwelle** (die Export-SOC-Schwelle wird ignoriert; der SOC muss nur über der Zone-3-Schwelle liegen).
* **Exit-Sperre (Fall 0B):** Solange `surplus_forecast_forced = true` werden **SOC-** und **PV-basierter** Austritt gleichermaßen blockiert. Fällt Solar unter das Hard Limit (auch nachts, PV = 0), die Vorhersage unter die Schwelle oder der SOC unter die Zone-3-Schwelle, wird das Flag sofort `false` — normale Austrittslogik (SOC- oder PV-Term) greift ohne Sonderfall.
* **Sensor:** Z.B. Solcast `power_now_1h` oder stündlicher Überschuss-Forecast in W.
* **Fallback:** Sensor unavailable/unknown → Forecast inaktiv, normale Export-Schwelle gilt.

---

### 12. ⛅ Surplus Austritts-Sperre (Optional)

Hält Zone 0 bei kurzen PV-Einbrüchen (Wolken), statt auszutreten.

* **Voraussetzung:** Überschuss-Einspeisung muss ebenfalls aktiviert sein.
* **Sperr-Flag:** `surplus_exit_locked = (Vorhersage ≥ Sperr-Faktor × Hard Limit) UND (SOC > Zone-3-Schwelle)` — der Faktor (Standard 1,5) ist die Sicherheitsmarge gegen Vorhersagefehler: Selbst wenn die Vorhersage deutlich daneben liegt, liegt das reale PV-Potenzial noch über dem Ausgabelimit, der gemessene Einbruch muss also transient sein.
* **Wirkung (Fall 0B):** Sperrt **nur** den PV-Austritt. Der SOC-Austritt bleibt ungesperrt und beendet Surplus immer — fällt der SOC real unter die Austrittsschwelle, greift der Exit trotz Sperre. Zone 3 (Fall C) beendet Surplus zusätzlich jederzeit.
* **Hintergrund:** Der Austritt bei vollem Akku führt in einen Zustand, in dem der Wechselrichter die PV auf den Eigenbedarf drosselt — der Überschuss ist danach nicht mehr messbar, und der Wiedereintritt hängt an zufälligen Verbrauchsschwankungen (minutenlange Verzögerung). Da die Batterie während einer Wolke nicht entladen wird (solange Solar existiert, bleibt sie unangetastet), bleibt der SOC am Maximum gepinnt und der Zustand löst sich nicht von selbst. Die Sperre vermeidet ihn, indem transiente Einbrüche gar nicht erst zum Austritt führen.
* **Risiko:** Meldet der Sensor dauerhaft einen zu hohen Wert, bleibt Zone 0 entsprechend lange aktiv — bis der SOC-Austritt eingreift.
* **Sensor:** Aktuell prognostizierte PV-Leistung in W, z.B. Solcast `power_now`.
* **Fallback:** Sensor unavailable/unknown → Sperre inaktiv, normale Austrittslogik gilt.

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
| **Helper** | Integral-Speicher | `input_number.solakon_integral` | Input Number: −1200 bis 1200 |
| **Script** | PI-Regler Script | `script.pi_regler` | Aus dem PI-Regler Blueprint erstelltes Script |
| **Helper** | Surplus-Zustand-Speicher | `input_boolean.solakon_surplus_aktiv` | Nur wenn Zone 0 aktiv. |
| **Helper** | AC-Lade-Zustand-Speicher | `input_boolean.solakon_ac_laden_aktiv` | Nur wenn AC Laden aktiv. |
| **Helper** | Tarif-Lade-Zustand-Speicher | `input_boolean.solakon_tarif_laden_aktiv` | Nur wenn Tarif-Arbitrage aktiv. |

> **Hinweis zum „Maximalen Entladestrom":** In der offiziellen Solakon Home-Assistant-Integration ist die Entität `number.solakon_one_maximaler_entladestrom` **standardmäßig deaktiviert**. Wird sie nicht gefunden bzw. als „unbekannte Entität" angezeigt, muss sie zunächst in HA unter dem Solakon-Gerät bei den **Konfigurations-Entitäten** aktiviert werden (Gerät → Entität → Zahnrad → „Aktivieren"). Ohne diese Entität kann der Entladestrom nicht gesteuert werden.

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.3 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine PI-Korrektur innerhalb (stattdessen Integral-Decay). |
| **Wartezeit** | 3 s | 0 | 30 s | Maximale Wartezeit nach Leistungsänderung. Adaptiv: bricht früh ab wenn Ist-Leistung ≈ Sollwert ± Toleranz. |

> **Hinweis:** P- und I-Faktor gelten für Zone 1 und Zone 2. Für den AC-Lade-Modus werden separate Faktoren verwendet.

---

### 🔋 SOC-Zonen-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Überschreiten aktiviert Zone 1. |
| **Zone 3 Stopp** | 20 % | 1 % | 49 % | Unterschreiten stoppt Entladung komplett. |
| **Max. Entladestrom Zone 1** | 40 A | 0 A | 40 A | Zone 2 und AC/Tarif-Laden nutzen automatisch 0 A. |
| **Nullpunkt-Offset 1 (Statisch)** | 30 W | -100 | 100 W | Statischer Fallback für Zone 1. |
| **Nullpunkt-Offset 1 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **Nullpunkt-Offset 2 (Statisch)** | 30 W | -100 | 100 W | Statischer Fallback für Zone 2. |
| **Nullpunkt-Offset 2 (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Zone-2-Limit: `Min(Hard Limit, Max(0, PV − Reserve))`. Auch PV-Schwelle für Nachtabschaltung. |

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
| **SOC-Schwelle Überschuss** | 90 % | 50 % | 99 % | Ab diesem SOC bei PV-Überschuss → Zone 0. Empfehlung: ~5 % unter der App-Ladeobergrenze (Begründung siehe Abschnitt 5). |
| **Hysterese Überschuss-Austritt (SOC)** | 5 % | 1 % | 20 % | SOC muss um diesen Wert unter die Eintritts-Schwelle fallen bevor Zone 0 verlassen wird. |
| **Hysterese PV-Überschuss** | 50 W | 10 W | 200 W | Totband um Hausverbrauch für Ein- und Austritt. |

---

### ⚡ AC Laden (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **AC Laden aktivieren** | false | — | — | Schalter für AC Laden. |
| **SOC-Ladeziel** | 90 % | 10 % | 99 % | Laden stoppt bei diesem SOC. Empfohlen: > Zone-1-Schwelle, damit Zone 1 direkt nach dem Laden übernimmt. |
| **Max. Ladeleistung** | 800 W | 50 | 1200 W | Obergrenze der AC-Ladeleistung. |
| **Hysterese Ladeabbruch** | 50 W | 0 | 300 W | Totband für Ein- und Austritt. |
| **AC Laden Offset (Statisch)** | -50 W | -100 | 100 W | Regelziel im AC-Lade-Modus. Negativ = Einspeisung angestrebt. |
| **AC Laden Offset (Dynamisch)** | *(leer)* | — | — | Optionale `input_number` Entität. Überschreibt statischen Wert. |
| **AC Laden P-Faktor** | 0.5 | 0.1 | 5.0 | Klein halten wegen langer Hardware-Flanke (~25 s). |
| **AC Laden I-Faktor** | 0 | 0 | 0.2 | Wegen träger Hardware (~25 s) kaum wirksam — Standardwert 0 belassen. |

---

### 💹 Tarif-Arbitrage (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Tarif-Arbitrage aktivieren** | false | — | — | Schalter für die gesamte Tarif-Logik. |
| **Strompreis-Sensor** | *(leer)* | — | — | Generischer Sensor. Einheit muss zu den Schwellwerten passen. |
| **Günstig-Schwelle** | 0.20 | 0 | 1 | Unter diesem Wert: Laden + Entladesperre (Zone 1 und Zone 2). |
| **Teuer-Schwelle** | 0.25 | 0 | 1 | Ab diesem Wert: Entladesperre aufgehoben, normale SOC-Logik. |
| **SOC-Ladeziel Tarif-Laden** | 90 % | 10 % | 99 % | Tarif-Laden stoppt bei diesem SOC. Unabhängig vom AC-Laden-Ladeziel. |
| **Ladeleistung Tarif-Laden** | 800 W | 50 | 1200 W | Direkt gesetzter Wert — kein PI-Regler. |

---

### 🌙 Nachtabschaltung (Optional)

| Parameter | Standard | Beschreibung |
|:----------|:---------|:-------------|
| **Nachtabschaltung aktivieren** | false | Ein/Aus-Schalter für die Funktion |
| **PV-Schwelle für "Nacht"** | — | Verwendet den Wert der **PV-Ladereserve** als Schwelle |

---

### 🌤️ PV-Forecast Tarif-Unterdrückung (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **PV-Forecast Unterdrückung aktivieren** | false | — | — | Schalter für die Funktion. Nur wirksam wenn Tarif-Arbitrage aktiv. |
| **PV-Forecast Sensor** | *(leer)* | — | — | PV-Ertragsprognose in W (z.B. Solcast). Leer lassen wenn nicht genutzt. |
| **PV-Forecast Schwelle** | 5000 W | 0 | 20000 W | Forecast muss diesen Wert erreichen um GT/TM zu unterdrücken. |

---

### 🌤️ Surplus-Forecast Eintritt (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Surplus-Forecast aktivieren** | false | — | — | Schalter für die Funktion. Nur wirksam wenn Überschuss-Einspeisung aktiv. |
| **Surplus-Forecast Sensor** | *(leer)* | — | — | PV-Überschuss-Prognose in W (z.B. Solcast). Leer lassen wenn nicht genutzt. |
| **Surplus-Forecast Schwelle** | 5000 W | 0 | 20000 W | Forecast muss diesen Wert erreichen um erzwungenen Zone-0-Eintritt auszulösen. |

---

### ⛅ Surplus Austritts-Sperre (Optional)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Austritts-Sperre aktivieren** | false | — | — | Schalter für die Funktion. Nur wirksam wenn Überschuss-Einspeisung aktiv. |
| **Austritts-Sperre Vorhersage-Sensor** | *(leer)* | — | — | Aktuell prognostizierte PV-Leistung in W (z.B. Solcast `power_now`). Leer lassen wenn nicht genutzt. |
| **Austritts-Sperre Faktor** | 1.5 | 1.0 | 3.0 | Sperre aktiv solange Vorhersage ≥ Faktor × Hard Limit. Höher = mehr Sicherheitsmarge gegen Vorhersagefehler. |

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

## 🔧 PI-Regler Einstellung

### Schritt 1: Wartezeit finden (P = 1, I = 0)

Die **Wartezeit** deckt Wechselrichter-Reaktion und Messlatenz ab. Sinnvolle Wartezeit: 1–3 s.

### Schritt 2: P-Faktor finden (I = 0)

```
P-Faktor: 0.3   # Startpunkt
I-Faktor: 0.0
```

Schrittweise erhöhen bis System leicht anfängt zu zittern — dann einen Schritt zurück. Typischer Arbeitsbereich: **0.8–1.5**.

### Schritt 3: I-Faktor hinzufügen

```
I-Faktor: 0.02   # Startpunkt
```

Typischer Arbeitsbereich: **0.03–0.08**. Für AC Laden separat tunen — P besonders klein halten (~0.3–0.5), I-Faktor auf 0 lassen (Hardware zu träge). Tarif-Laden verwendet keinen PI-Regler.

---

## 🛑 Wichtige Fehlermeldungen

| Meldung | Ursache | Lösung |
|:--------|:--------|:-------|
| **SOC-Limits ungültig** | Zone-1-Schwelle ≤ Zone-3-Schwelle | Zone-1 (z.B. 50%) > Zone-3 (z.B. 20%) |
| **SOC-Limits ungültig** | Überschuss aktiviert UND Export-Schwelle ≤ Zone-1-Schwelle | Export-Schwelle (z.B. 90%) > Zone-1 (z.B. 50%) |
| **SOC-Sensor UNKNOWN/UNAVAILABLE** | Solakon Integration offline | Verbindung prüfen |
| **Timeout Countdown UNKNOWN/UNAVAILABLE** | Sensor nicht verfügbar | Solakon Integration prüfen |

---

## ⚙️ Technische Details

### Architektur

1. **Hauptautomatisierung** (`solakon_one_nulleinspeisung.yaml`): Zonen-Steuerung, SOC-Logik, Surplus/AC/Tarif-Lade-Zustand, Entladestrom-Verwaltung, Timeout-Reset, PI-Aufruf-Guard, Integral-Decay/-Einfrieren
2. **PI-Regler Script** (`PI-Regler.yaml`): Reine Berechnungslogik — skaliert `raw_error` mit `error_share` vor der PI-Berechnung
3. **Leistungsverteilung** (`solakon_leistungsverteilung.yaml`): Multi-Instanz-Koordination — berechnet Leistungslimits und Fehler-Anteile; nur für Multi-Instancing erforderlich

### Tarif-Arbitrage State-Machine

```
Eintritts-Bedingung (Fall GT):
  tariff_arbitrage_enabled UND Preis < cheap_threshold
  UND soc < tariff_soc_charge_target
  UND Modus ≠ '3' ← Guard: verhindert Re-Eintritt
  UND NICHT surplus_active ← Guard: Zone 0 hat Vorrang
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

Discharge-Lock (Fall TM):
  price_discharge_locked = tariff_arbitrage_enabled
                           UND price_sensor_valid
                           UND Preis < expensive_threshold
  Modus = '1' (Entladung läuft, egal ob Zone 1 oder Zone 2)
  UND kein Überschuss (Zone 0 hat Vorrang)
  → Integral = 0
  → Zone 1: cycle_active → off
  → Output = 0W, Modus = '0'
  (Reaktivierung über Fall A/E wenn Preis ≥ expensive_threshold)

Entladesperre-Guards (Falls A/E):
  → Fall A blockiert (Zone 1 kann nicht starten)
  → Fall E blockiert (Zone 2 kann nicht starten)
```

### AC-Lade-Zustand State-Machine

```
Eintritts-Bedingung (Fall G):
  ac_charge_enabled UND soc < soc_ac_charge_limit
  UND Modus ≠ '3' ← Guard: verhindert Re-Eintritt
  UND NICHT tariff_charge_active ← Guard: Tarif-Laden hat Vorrang
  UND NICHT surplus_active ← Guard: Zone 0 hat Vorrang
  → ac_charge_state_helper = on
  → Modus = '3', Output = 0W, Timer-Toggle
```

### Dynamisches Power-Limit

```
Modus '3' + tariff_charge_mode_active:  tariff_charge_power (Zweig BT, kein PI)
Modus '3' + ac_charge_mode_active:      ac_charge_power_limit (PI mit ac_charge_mode=true)
Zone 1 (cycle = on):                    hard_limit
Zone 2 (cycle = off):                   Min(Hard Limit, Max(0, PV - pv_charge_reserve))
```

### Automatische Entladestrom-Steuerung

| Zone | Entladestrom | Bedingung |
|:-----|:------------|:----------|
| Zone 0 (Überschuss) | 2 A | Nur wenn abweichend |
| Zone 1 (Aggressiv) | Konfigurierter Maximalwert | Nur wenn abweichend und kein Surplus, kein AC/Tarif-Laden |
| Zone 2 / AC/Tarif-Laden (Modus 3) | 0 A | Nur wenn abweichend |

---

## 🔀 Multi-Instancing (Mehrere Batterien)

### Funktionsweise

Jede Instanz übernimmt nur ihren proportionalen Anteil des Netzfehlers (`error_share`).
Die Leistungsverteilungs-Automation berechnet diesen Anteil pro Instanz aus der nutzbaren
Kapazität — also dem SOC-Bereich über dem konfigurierten Min-SOC (Zone 3 Stopp):

```
usable_i    = (SOC_i − Min-SOC_i) / 100 × Kap_i
error_share_i = usable_i / Σ usable_j
```

Ohne Kapazitätssensor gilt `Kap_i = 100` — Gewichtung nach SOC-Prozentpunkten (identisch zum Vorgänger).

Beispiel mit zwei Instanzen (Min-SOC jeweils 20 %, Kapazitäten 10 kWh / 5 kWh):

```
Instanz 1: SOC=60% → (40/100) × 10 = 4,0 kWh nutzbar
Instanz 2: SOC=40% → (20/100) ×  5 = 1,0 kWh nutzbar
→ share1 = 4.0/5.0 = 0.80, share2 = 1.0/5.0 = 0.20
```

Der berechnete Anteil wird von der Leistungsverteilung in einen `input_number`-Helfer
geschrieben, den der PI-Regler jeder Instanz als `error_share` liest und auf `raw_error`
anwendet. Im Einzelbetrieb (kein Helfer konfiguriert) gilt `error_share = 1.0`.

Da die Gewichtung auf der nutzbaren Kapazität basiert, gleichen sich die Ladezustände
mehrerer Batterien automatisch an: eine Instanz mit mehr nutzbarer Kapazität übernimmt
mehr Last und entlädt sich entsprechend stärker, bis beide wieder auf gleichem Stand sind.

### Zwei getrennte Pools: Nulleinspeisung und AC-Laden

Die Verteilung läuft über **zwei unabhängige Pools**, nicht einen gemeinsamen:

- **Pool 1 (Nulleinspeisung):** nur Instanzen in Modus `'1'` (Zone 1/Zone 2). Bestimmt sowohl
  Leistungslimit als auch `error_share_entity`.
- **Pool 2 (AC-Laden):** nur Instanzen mit aktivem AC-Lade-Zustand-Helfer (Modus `'3'`). Bestimmt
  nur `ac_error_share_entity` — kein eigenes Leistungslimit, das AC-Ladelimit bleibt unabhängig.

Grund für die Trennung: Eine Instanz, die gerade per AC lädt, steht in Modus `'3'` und zählt damit
nicht zu Pool 1. Gäbe es nur einen gemeinsamen Fehler-Anteil, bekäme sie dort `error_share = 0`
zugewiesen — und genau dieser Wert würde auch ihrem AC-Lade-PI übergeben, der dadurch bei aktivem
Ladebedarf auf 0 W einfriert. Mit zwei getrennten Pools bekommt jede Instanz für jeden Modus einen
eigenen, korrekt berechneten Anteil. Beide Pools verwenden dieselbe Gewichtungslogik
(Gleichverteilung oder SOC-gewichtet, je nach globalem Umschalter).

Die beiden Pools überschneiden sich nie — eine Instanz ist zu jedem Zeitpunkt entweder in Pool 1,
in Pool 2 oder in keinem von beiden (z. B. Zone 3 gestoppt, Tarif-Laden).

### Erforderliche Helper pro Instanz (zusätzlich zu Einzelinstanz)

| Helper / Sensor | Typ | Einstellungen | Verwendung |
|:----------------|:----|:--------------|:-----------|
| `...instanz_N_limit` | `input_number` | min:0, max:≥Global-Max, step:1 | Leistungslimit von Leistungsverteilung → Instanz (Pool 1) |
| `...instanz_N_share` | `input_number` | min:0, max:1, step:0.001 | Fehler-Anteil Nulleinspeisung von Leistungsverteilung → PI-Regler (Pool 1) |
| Kapazitätssensor (optional) | `sensor` | kWh — von Solakon-Integration bereitgestellt | kWh-genaue Gewichtung bei unterschiedlichen Batteriekapazitäten |
| `...instanz_N_ac_share` (nur bei AC-Laden) | `input_number` | min:0, max:1, step:0.001 | Fehler-Anteil AC-Laden von Leistungsverteilung → PI-Regler (Pool 2) |

Für Pool 2 wird zusätzlich derselbe AC-Lade-Zustand-Helfer (`input_boolean`, siehe Punkt 5 der
Helper-Liste) in der Leistungsverteilung eingetragen — er zeigt an, welche Instanzen gerade
gleichzeitig laden.

### Konfigurationsschritte

1. Alle Instanz-Blueprints wie gewohnt einrichten
2. Pro Instanz die zwei neuen Helper (`limit`, `share`) erstellen — bei AC-Laden zusätzlich `ac_share`
3. In jeder Instanz-Automation eintragen: „Max. Ausgangsleistung — Dynamisch", „Fehler-Anteil Helfer" und (bei AC-Laden) „AC-Lade Fehler-Anteil Helfer"
4. Leistungsverteilungs-Blueprint (`solakon_leistungsverteilung.yaml`) als Automation anlegen:
   - Min-SOC pro Instanz eintragen — identisch mit dem Wert „Zone 3 Stopp" der jeweiligen Instanz
   - `limit`- und `share`-Helper pro Instanz zuordnen
   - Optional: Kapazitätssensor der Solakon-ONE-Integration pro Instanz eintragen — empfohlen bei unterschiedlichen Batteriekapazitäten
   - Bei AC-Laden zusätzlich: AC-Lade-Zustand-Helfer und `ac_share`-Helfer pro ladender Instanz zuordnen

---

## ⚠️ Wichtige Hinweise

1. **Helper und Script vor Installation erstellen:** input_boolean (Zyklus), input_number (Integral), PI-Regler Script müssen existieren
2. **Optionale Helper erstellen wenn Funktion aktiviert:** Surplus-Boolean für Zone 0; AC-Lade-Boolean für AC Laden; Tarif-Lade-Boolean für Tarif-Arbitrage
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **Tarif-Sensor Einheit:** Sensor-Wert und Schwellwerte müssen dieselbe Einheit verwenden — kein Blueprint-seitiges Umrechnen
5. **Entladesperre Schwelle:** `expensive_threshold` steuert die Sperre (unterhalb = gesperrt). TM greift für **Zone 1 und Zone 2** — `cycle_active` spielt keine Rolle mehr.
6. **Integral-Helper:** Wird automatisch verwaltet — nicht manuell ändern
7. **Sensor-Einheit kW:** Grid-, Solar- und Ist-Leistungssensoren dürfen W **oder** kW liefern — der Blueprint normalisiert automatisch auf W. Tarif- und Preis-Sensoren werden nicht automatisch umgerechnet (siehe Hinweis 4).

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
