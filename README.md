# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V306

Dieser Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **SOC-Zonen-Logik** mit optionaler **Überschuss-Einspeisung bei vollem Akku**, optionalem **AC Laden aus externer Einspeisung** und optionaler **Tarif-Arbitrage** (günstig laden, Entladesperre bei niedrigem Tarif).

Ziel dieses Blueprints ist es, PV-Energie direkt auszugeben ohne den Umweg über die Batterie.

---

**WICHTIG:** Die Implementierung der Fernsteuerung der Solakon Integration führt dazu, dass es kein "disabled" gibt als Fernsteuerbefehl — dies schaltet die Fernsteuerung an sich ab, d.h. die Standardeinstellungen des Solakon ONE bzw. aus der APP greifen zu diesem Zeitpunkt.
Für eine wie im Folgenden gewollte Funktion sollte als Standard ein 0W für 24std Zeitplan erstellt und aktiviert werden, oder die "Standart-Ausgangsleistung" auf 0W gestellt werden. Diese Methoden sind äquivalent.

---

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fexperimental_multi_instancing%2Fsolakon_one_nulleinspeisung.yaml)

Der zugehörige **PI-Regler Script-Blueprint** muss ebenfalls importiert werden:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fexperimental_multi_instancing%2FPI-Regler.yaml)

Für das Multi instancing:

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fraw.githubusercontent.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fexperimental_multi_instancing%2Fsolakon_leistungsverteilung.yaml)

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

1. Gehen Sie zu **Einstellungen** → **Geräte & Dienste** → **Helfer** → **Number**
2. Name: z.B. `Solakon Instanz 1 Share`
3. **Einstellungen:** Min: `0`, Max: `1`, Step: `0.001`, Initialwert: `1`
4. Speichern (Entity ID: z.B. `input_number.solakon_instanz1_share`)
5. Wiederholen für jede weitere Instanz
6. In der Leistungsverteilung als „Fehler-Anteil Helfer" eintragen
7. In der Instanz-Automation als „Fehler-Anteil Helfer" eintragen

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
* **I-Anteil:** Summiert Abweichungen über die Zeit auf, eliminiert bleibende Regelabweichungen. Anti-Windup auf ±1000. Automatischer Reset bei Zonenwechsel. Toleranz-Decay 5%/Zyklus wenn Fehler ≤ Toleranz und |Integral| > 10. Zone-0-Einfrieren bei aktivem Überschuss.
* **Fehlerberechnung:** Normal (`ac_charge_mode=false`): `raw_error = (grid − target_offset) × error_share`. AC Laden (`ac_charge_mode=true`): `raw_error = (target_offset − grid) × error_share` (invertiert). `error_share` skaliert den Fehler auf den Anteil dieser Instanz (Standard 1.0 = voller Fehler).
* **Dynamisches Power-Limit:** Zone 1 → Hard Limit. Zone 2 → `Max(0, PV − Reserve)`. AC Laden → konfigurierbares Lade-Limit. Tarif-Laden → kein PI (direkter Wert).
* **PI-Aufruf-Guard:** Zone 0 aktiv → PI nicht aufgerufen, Integral eingefroren. Tarif-Laden aktiv → direkt setzen. AC Laden aktiv → PI mit `ac_charge_mode=true`. Normal → PI nur wenn `|Fehler| > Toleranz` UND kein At-Limit.

---

### 2. 🔋 SOC-Zonen-Logik

| Zone | SOC-Bereich / Bedingung | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:------------------------|:------|:-----------------|:---------|:--------------|
| **0 — Überschuss-Einspeisung** | SOC ≥ Export-Schwelle UND PV > Output + Grid + PV-Hysterese | `'1'` | 2 A | Hard Limit | **Optional.** Integral eingefroren. Blockiert GT und G. |
| **1 — Aggressive Entladung** | SOC > Zone-1-Schwelle | `'1'` | Konfigurierter Max-Wert | 0W + Offset 1 | Läuft **bis SOC ≤ Zone-3-Schwelle**. Auch nachts aktiv. |
| **2 — Batterieschonend** | Zone-3-Schwelle < SOC ≤ Zone-1-Schwelle | `'1'` | **0 A** | 0W + Offset 2 | Dynamisches Limit: `Max(0, PV − Reserve)`. Optional: Nachtabschaltung. |
| **3 — Sicherheitsstopp** | SOC ≤ Zone-3-Schwelle | `'0'` | 0 A | — | Output = 0 W. Vollständiger Batterieschutz. Absoluter Vorrang. |

---

### 3. ⚡ Priorität der optionalen Module

Die optionalen Module überschreiben die SOC-Zonen-Logik in folgender Reihenfolge:

```
Zone 3 (Sicherheit, absolut) > Zone 0 (Überschuss) > Tarif-Laden > AC-Laden > SOC-Zonen
```

**Zone 3** hat absoluten Vorrang — greift unabhängig von allen anderen Modi.

**Zone 0 (Überschuss)** blockiert den Eintritt in Tarif-Laden (Fall GT) und AC-Laden (Fall G) solange aktiv. Zone 0 kann nur starten wenn kein Ladevorgang aktiv ist.

**Tarif-Laden (Fall GT)** startet nur wenn Zone 0 NICHT aktiv ist. Sperrt Zone 1 **und** Zone 2: Für die mittlere Preiszone (günstig ≤ Preis < teuer) wird laufende Entladung sowohl aus Zone 1 als auch Zone 2 gestoppt (Fall TM). Der Zyklus-Helper wird dabei zurückgesetzt; beim nächsten teuren Tarif starten Falls A/E neu. Für günstige Preise gilt zusätzlich aktives Laden (Fall GT).

**AC-Laden (Fall G)** startet nur wenn weder Zone 0 noch Tarif-Laden aktiv ist.

**SOC-Zonen (Falls A/E)** starten nur wenn keine der obigen Prioritäten greift und die Entladesperre (Preis < teuer) nicht aktiv ist.

---

### 4. Übersicht der Steuer-Falls (choose-Block)

Die Reihenfolge ist entscheidend — der erste zutreffende Fall wird ausgeführt.

| Fall | Bedingung | Aktion |
|:-----|:----------|:-------|
| **0A** | Surplus-Bool = `off` UND (SOC ≥ Export-Schwelle UND (PV > Output + Grid + PV-Hysterese ODER PV = 0) **ODER** Surplus-Forecast-Forced UND PV > Hard Limit) | Zone 0 Start: Surplus-Bool → `on` |
| **0B** | Surplus-Bool = `on` UND **NICHT Surplus-Forecast-Forced** UND (SOC < Export-Schwelle − SOC-Hysterese ODER PV ≤ Output + Grid − PV-Hysterese) | Zone 0 Ende: Surplus-Bool → `off`, Integral = 0 |
| **A** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND SOC > Zone-1-Schwelle UND Zyklus = `off` | Zone 1 Start: Zyklus = `on`, Integral = 0, Surplus/AC-Bool zurücksetzen, Timer-Toggle, Modus → `'1'` |
| **B** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `on` | Zone 3 Stop: Zyklus = `off`, Integral = 0, Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **C** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND SOC < Zone-3-Schwelle UND Zyklus = `off` UND Modus ≠ `'0'` | Zone 3 Absicherung: Surplus/AC-Bool zurücksetzen, Modus → `'0'`, Output → 0W |
| **D** | Zyklus = `on` UND Modus ∉ `{'1','3'}` UND SOC > Zone-3-Schwelle | Recovery: Timer-Toggle, Modus → `'3'` wenn AC-Lade-Bool **oder** Tarif-Lade-Bool = `on`, sonst `'1'` |
| **GT** | Tarif-Arbitrage aktiv UND Preis < Günstig-Schwelle UND SOC < Tarif-Ladeziel UND **Modus ≠ `'3'`** UND **NICHT Surplus-Bool = `on`** UND **NICHT PV-Forecast-Suppressed** | Tarif-Laden Start: Tarif-Bool = `on`, Timer-Toggle, Output → Ladeleistung (direkt), Modus → `'3'` |
| **HT** | Modus = `'3'` UND Tarif-Bool = `on` UND (Preis ≥ Günstig-Schwelle ODER SOC ≥ Tarif-Ladeziel) | Tarif-Laden Ende: Tarif-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **TM** | Tarif aktiv UND Günstig ≤ Preis < Teuer-Schwelle UND kein AC/Tarif-Laden UND **Modus = `'1'`** UND **NICHT PV-Forecast-Suppressed** | Discharge-Lock: Integral = 0, Zyklus = `off` (wenn aktiv) + Surplus-Bool zurücksetzen, Output → 0W, Modus → `'0'` |
| **G** | AC aktiv UND SOC < Ladeziel UND **Modus ≠ `'3'`** UND NICHT Tarif-Lade-Bool = `on` UND **NICHT Surplus-Bool = `on`** UND (Grid + Output) < −Hysterese | AC Laden Start: AC-Bool = `on`, Timer-Toggle, Modus → `'3'`, Output → 0W |
| **H** | Modus = `'3'` UND (SOC ≥ Ladeziel ODER (Grid ≥ `ac_charge_offset + Hysterese` UND Output = 0 W)) | AC Laden Ende: AC-Bool = `off`, Integral = 0, Zone 1 → `'1'` / Zone 2 → `'0'` |
| **I** | Modus = `'3'` UND NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` | Safety-Korrektur: Integral = 0, Zone 1 → `'1'` (Timer-Toggle) / Zone 2 → `'0'` + 0W |
| **E** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND NICHT Entladesperre (Preis < teuer) UND Zone-3 < SOC ≤ Zone-1 UND Zyklus = `off` UND Modus = `'0'` UND NICHT Nacht | Zone 2 Start: Integral = 0, Timer-Toggle, Modus → `'1'` |
| **F** | NICHT AC-Lade-Bool = `on` UND NICHT Tarif-Lade-Bool = `on` UND Nachtabschaltung aktiv UND PV < PV-Ladereserve UND Zyklus = `off` UND Modus aktiv | Nachtabschaltung: Integral = 0, Modus → `'0'`, Output → 0W |

---

### 5. ☀️ Überschuss-Einspeisung (Zone 0, Optional)

Ermöglicht aktives Einspeisen von PV-Überschuss wenn der Akku voll ist. SOC- und PV-Hysterese verhindern instabiles Hin- und Herschalten.

* **Aktivierung:** Über den Parameter "Überschuss-Einspeisung aktivieren"
* **Eintritts-Bedingung:** SOC ≥ Export-Schwelle UND (PV > (Output + Grid + PV-Hysterese) ODER PV = 0)
* **Austritts-Bedingung:** SOC < (Export-Schwelle − SOC-Hysterese) ODER PV ≤ (Output + Grid − PV-Hysterese)
* **Blockiert:** Tarif-Laden (Fall GT) und AC-Laden (Fall G) können nicht starten solange Zone 0 aktiv ist.
* **Verhalten:** Output auf Hard Limit, Entladestrom 2 A (Stabilitätspuffer), Integral eingefroren.
* **Deaktiviert:** Klassische Nulleinspeisung — kein aktives Einspeisen.

---

### 6. ⚡ AC Laden (Optional)

Laden der Batterie wenn eine externe Einspeisung ins Netz erkannt wird. Eintritts-Erkennung: `(Grid + Ausgangsleistung) < −Hysterese`.

* **Blockiert durch:** Zone 0 (Überschuss-Bool = `on`) und Tarif-Laden (Tarif-Bool = `on`).
* **Eintritts-Bedingung (Fall G):** AC Laden aktiviert UND SOC < Ladeziel UND Modus ≠ `'3'` UND NICHT Tarif-Lade-Bool = `on` UND **NICHT Surplus-Bool = `on`** UND (Grid + Output) < −Hysterese.
* **PI-Regelung:** `ac_charge_mode=true` → invertierte Fehlerberechnung: `target_offset − grid`. Separate P/I-Faktoren. P klein halten (~0.3–0.5), I auf 0 belassen (Hardware zu träge).
* **Rückkehr:** Zone 1 → Modus `'1'` (Timer-Toggle) + Integral Reset. Zone 2 → Modus `'0'` + Output 0W + Integral Reset.

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
* **Rückkehr:** Zone 1 → Timer-Toggle + Modus `'1'` / Zone 2 → Modus `'0'` + Output 0W.
* **Priorität:** Tarif-Laden (GT) liegt vor AC-Laden (G) im choose-Block.

#### Entladesperre / Discharge-Lock (Fall TM)

* **Bedingung:** Tarif aktiv **UND** Günstig ≤ Preis < Teuer **UND** kein AC/Tarif-Laden **UND** Modus = `'1'`.
* **Wirkung:** Stoppt sofort jede Entladung in Zone 1 und Zone 2. Der Wechselrichter wird auf 0W (Modus `'0'`) gesetzt.
* **Zone-1-Besonderheit:** Zyklus-Helper wird auf `off` zurückgesetzt und Surplus-Bool ggf. bereinigt. Fall D kann den Modus dadurch nicht sofort wiederherstellen.
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

Betrifft **nur Zone 2** (Fall F). Zone 1 und AC Laden laufen auch nachts weiter.

* **Schwelle:** PV-Leistung unter dem Wert der **PV-Ladereserve** (kein separater Parameter)
* **Verhalten:** Modus → `'0'`, Integral reset, Output 0 W
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
* **Eintritt (Fall 0A, OR-Branch):** Wenn Forecast ≥ Schwelle UND Solar > Hard Limit → Zone-0-Eintritt **ohne SOC-Gate** (SOC-Schwelle wird ignoriert).
* **Exit-Sperre (Fall 0B):** Solange `surplus_forecast_forced = true` wird Fall 0B **blockiert** — Zone 0 bleibt aktiv auch wenn SOC oder PV sonst den Austritt auslösen würden.
* **Sensor:** Z.B. Solcast `power_now_1h` oder stündlicher Überschuss-Forecast in W.
* **Fallback:** Sensor unavailable/unknown → Forecast inaktiv, normales SOC-Gate gilt.

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
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Zone-2-Limit: `Max(0, PV − Reserve)`. Auch PV-Schwelle für Nachtabschaltung. |

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
  → Integral = 0
  → Zone 1: cycle_active → off, Surplus-Bool → off
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
Zone 2 (cycle = off):                   Max(0, PV - pv_charge_reserve)
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

### Erforderliche Helper pro Instanz (zusätzlich zu Einzelinstanz)

| Helper / Sensor | Typ | Einstellungen | Verwendung |
|:----------------|:----|:--------------|:-----------|
| `...instanz_N_limit` | `input_number` | min:0, max:≥Global-Max, step:1 | Leistungslimit von Leistungsverteilung → Instanz |
| `...instanz_N_share` | `input_number` | min:0, max:1, step:0.001 | Fehler-Anteil von Leistungsverteilung → PI-Regler |
| Kapazitätssensor (optional) | `sensor` | kWh — von Solakon-Integration bereitgestellt | kWh-genaue Gewichtung bei unterschiedlichen Batteriekapazitäten |

### Konfigurationsschritte

1. Alle Instanz-Blueprints wie gewohnt einrichten
2. Pro Instanz die zwei neuen Helper (`limit`, `share`) erstellen
3. In jeder Instanz-Automation eintragen: „Max. Ausgangsleistung — Dynamisch" und „Fehler-Anteil Helfer"
4. Leistungsverteilungs-Blueprint (`solakon_leistungsverteilung.yaml`) als Automation anlegen:
   - Min-SOC pro Instanz eintragen — identisch mit dem Wert „Zone 3 Stopp" der jeweiligen Instanz
   - `limit`- und `share`-Helper pro Instanz zuordnen
   - Optional: Kapazitätssensor der Solakon-ONE-Integration pro Instanz eintragen — empfohlen bei unterschiedlichen Batteriekapazitäten

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
