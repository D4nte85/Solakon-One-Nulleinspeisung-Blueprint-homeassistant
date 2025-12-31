# ⚡ Solakon ONE Nulleinspeisung Blueprint (DE) - V205

Dieses Home Assistant Blueprint implementiert eine **dynamische Nulleinspeisung** für den Solakon ONE Wechselrichter, basierend auf einem **PI-Regler (Proportional-Integral-Regler)** und einer intelligenten **dreistufigen Batterieladestands-Logik (SOC)**.

## 🚀 Installation

Installieren Sie den Blueprint direkt über diesen Button in Ihrer Home Assistant Instanz:

[![Open your Home Assistant instance and show the blueprint import dialog with a pre-filled URL.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

---

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

---

## 🧠 Kernfunktionalität

### 1. PI-Regler (Proportional-Integral-Regler)

Der Blueprint nutzt einen modernen **PI-Regler** statt eines einfachen P-Reglers für präzisere Nulleinspeisung:

* **P-Anteil (Proportional):** 
  - Reagiert sofort auf aktuelle Abweichungen
  - Konfigurierbare Aggressivität über den **P-Faktor** (z.B. 1.5)
  - Schnelle Reaktion auf Laständerungen

* **I-Anteil (Integral):**
  - Summiert Abweichungen über die Zeit auf
  - Eliminiert **bleibende Regelabweichungen** (die ein reiner P-Regler nicht korrigieren kann)
  - Konfigurierbare Geschwindigkeit über den **I-Faktor** (z.B. 0.05)
  - **Anti-Windup:** Begrenzt auf ±1000 Punkte
  - **Automatischer Reset:** Bei jedem Zonenwechsel auf 0 zurückgesetzt
  - **Toleranz-Decay:** 5% Abbau wenn keine Korrektur nötig (verhindert unnötiges Aufaddieren)

* **Messprinzip:** 
  - Basiert auf dem **Netz-Leistungssensor** (z.B. Shelly 3EM)
  - **Positive Werte = Bezug**, **Negative Werte = Einspeisung**
  - Keine Verzögerung - sofortige Reaktion auf Sensor-Änderungen

* **Fehlerberechnung mit dynamischer Begrenzung:**
  - **Zone 1:** Fehler = Min(verfügbare Kapazität, Grid Power)
  - **Zone 2:** Fehler = Min(verfügbare Kapazität, Grid Power - Offset, PV-Kapazität)
  - Verhindert Integral-Windup durch intelligente Begrenzung

* **Leistungsbegrenzung:**
  - Oberes Hard Limit (z.B. 800 W) in Zone 1
  - Dynamisches PV-basiertes Limit in Zone 2
  - Unteres Limit: fest 0 W

---

### 2. 🔋 Dreistufige SOC-Zonen-Logik

Die Regelung wird anhand des aktuellen SOC in drei Betriebsmodi unterteilt:

| Zone | SOC-Bereich | Modus | Max. Entladestrom | Regelziel | Besonderheiten |
|:-----|:-----------|:------|:-----------------|:---------|:--------------|
| **1. Aggressive Entladung** | SOC > 50% | `INV Discharge (PV Priority)` | 40 A | 0 W (exakte Nulleinspeisung) | Läuft **durchgehend bis SOC ≤ 20%** (kein Yo-Yo-Effekt). Auch nachts aktiv. Hard Limit 800W. |
| **2. Batterieschonend** | 20% < SOC ≤ 50% | `INV Discharge (PV Priority)` | **0 A** (nur AC-Limit) | 30 W (leichter Netzbezug = Laden) | Dynamisches Limit: **Max(0, PV - Reserve)**. Optional: Nachtabschaltung möglich. |
| **3. Sicherheitsstopp** | SOC ≤ 20% | `Disabled` | 0 A | - | Ausgang = 0 W. Vollständiger Schutz der Batterie. |

**Wichtig:** 
- Zone 1 wird **einmal aktiviert** beim Überschreiten von 50% SOC und läuft dann durch bis 20% erreicht wird
- Zone 2 startet nur aus Zone 3 (beim Laden von unter 20% auf über 20%)
- Dies verhindert ständiges Hin- und Herwechseln zwischen den Zonen
- **Max. Entladestrom** wird automatisch gesteuert: 40A in Zone 1, 0A in Zone 2

---

### 3. 🌙 Nachtabschaltung (Optional)

Die Nachtabschaltung kann optional aktiviert werden und betrifft **nur Zone 2**:

* **Aktivierung:** Über den Parameter "Nachtabschaltung aktivieren"
* **Schwelle:** PV-Leistung unter konfiguriertem Wert (z.B. 10 W)
* **Verhalten:**
  - **Zone 1:** Läuft auch nachts weiter (hoher SOC → aggressive Entladung gewünscht)
  - **Zone 2:** Wird bei PV < Schwelle auf `Disabled` gesetzt (Integral wird zurückgesetzt)
  - **Zone 3:** Ohnehin deaktiviert
* **Sinnvoll für:** Batterieschonung bei 0 PV, wenn keine Grundlast vorhanden ist

---

### 4. ⏱️ Remote Timeout Reset und Moduswechsel-Sequenz

Um die Stabilität der Kommunikation mit dem Solakon ONE zu gewährleisten:

1. **Kontinuierlicher Timeout-Reset:** 
   - Der Remote-Timeout wird automatisch auf 3599s zurückgesetzt
   - Trigger: Countdown fällt unter 120s
   
2. **Forcierter Reset bei Moduswechsel:**
   - Zweistufige Puls-Sequenz: 10s → 3599s (mit 1s Verzögerung)
   - Stellt sichere Modusübernahme sicher
   - Verhindert Timeout-Fehler

---

## 📊 Input-Variablen und Konfiguration

### 🔌 Erforderliche Entitäten

| Kategorie | Variable | Standard-Entität | Beschreibung |
|:----------|:---------|:----------------|:-------------|
| **Extern** | Netz-Leistungssensor | *(kein Standard)* | Z.B. Shelly 3EM. **Positiv = Bezug, Negativ = Einspeisung** |
| **Solakon** | Solarleistung | `sensor.solakon_one_total_pv_power` | Aktuelle PV-Erzeugung in Watt |
| **Solakon** | Batterieladestand (SOC) | `sensor.solakon_one_battery_soc` | Ladestand in % |
| **Solakon** | Remote Timeout Countdown | `sensor.solakon_one_remote_timeout_countdown` | Verbleibender Countdown |
| **Solakon** | Ausgangsleistungsregler | `number.solakon_one_remote_active_power` | Setzt Leistungs-Sollwert |
| **Solakon** | Max. Entladestrom | `number.solakon_one_battery_max_discharge_current` | Setzt Entladestrom-Limit |
| **Solakon** | Modus-Reset-Timer | `number.solakon_one_remote_timeout_set` | Setzt/Reset Timeout (max. 3599s) |
| **Solakon** | Betriebsmodus-Auswahl | `select.solakon_one_remote_control_mode` | Schaltet Betriebsmodus |
| **Helper** | Entladezyklus-Speicher | `input_select.soc_entladezyklus_status` | Input Select: `on`/`off` |
| **Helper** | Integral-Speicher | `input_number.solakon_integral` | Input Number: -1000 bis 1000 |

---

### 🎚️ Regelungs-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **P-Faktor** | 1.5 | 0.1 | 5.0 | Proportional-Verstärkung. Höher = aggressiver, schneller. |
| **I-Faktor** | 0.05 | 0.01 | 0.2 | Integral-Verstärkung. Höher = schnellere Fehlerkorrektur, aber instabiler. |
| **Toleranzbereich** | 25 W | 0 | 200 W | Totband um Regelziel. Keine Korrektur innerhalb dieser Zone. |

**Tuning-Tipps:**
- **Zu langsam?** → P-Faktor erhöhen (z.B. 2.0) oder I-Faktor erhöhen (z.B. 0.08)
- **Schwingt/Überschwingt?** → P-Faktor senken (z.B. 1.0) oder I-Faktor senken (z.B. 0.03)
- **Permanente Mini-Korrekturen?** → Toleranzbereich erhöhen (z.B. 35 W)
- **Integral läuft hoch obwohl stabil?** → Toleranz-Decay greift automatisch (5% Abbau)

---

### 🔋 SOC-Zonen-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Zone 1 Start** | 50 % | 1 % | 99 % | Obere Schwelle. Überschreiten aktiviert aggressive Entladung. |
| **Zone 3 Stopp** | 20 % | 1 % | 49 % | Untere Schwelle. Unterschreiten stoppt Entladung komplett. |
| **Max. Entladestrom Zone 1** | 40 A | 0 A | 40 A | Entladestrom in Zone 1. Zone 2 nutzt automatisch 0 A. |

**Wichtig:** Obere Schwelle muss **größer** als untere Schwelle sein! Blueprint validiert dies beim Start.

---

### ⚙️ Zone 2 - Parameter (Batterieschonender Modus)

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Nullpunkt-Offset** | 30 W | 0 | 100 W | Regelziel in Zone 2. Positiv = leichter Netzbezug (Batterie wird geladen). 0W = exakte Nulleinspeisung. |
| **PV-Ladereserve** | 50 W | 0 | 1000 W | Reservierte PV-Leistung für Ladung. Dynamisches Limit: Max(0, PV - Reserve). |

**Erklärung PV-Ladereserve:**
- Bei 300W PV-Erzeugung und 50W Reserve → Max. Ausgang in Zone 2: 250W
- Stellt sicher, dass auch bei Trickle-Charge immer etwas für die Batterie übrig bleibt
- Gleicht Wandlerverluste aus
- Wird automatisch in der Fehlerberechnung des PI-Reglers berücksichtigt

---

### 🔒 Sicherheits-Parameter

| Parameter | Standard | Min | Max | Beschreibung |
|:----------|:---------|:----|:----|:-------------|
| **Max. Ausgangsleistung** | 800 W | 0 | 1200 W | Absolute Obergrenze (Hard Limit) in Zone 1 zum Schutz des Wechselrichters. |

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
Max. Entladestrom Zone 1: 40A
```

---

## 📈 Funktionsweise im Detail

### Beispiel-Ablauf über einen Tag:

**Morgens (06:00 - SOC: 25%)**
- Zone 2 aktiv (20% < SOC ≤ 50%)
- PV steigt langsam an
- Max. Entladestrom: **0A** (automatisch gesetzt - batterieschonend)
- Regelziel: 30W (leichter Netzbezug)
- Dynamisches Limit: Max(0, PV - 50W)
- Batterie wird geladen

**Mittags (12:00 - SOC: 55%)**
- Zone 1 aktiviert (SOC > 50%)
- Max. Entladestrom: **40A** (automatisch gesetzt)
- Regelziel: 0W (exakte Nulleinspeisung)
- Hard Limit: 800W
- Aggressive Entladung bei Lastspitzen
- **Bleibt aktiv, auch wenn SOC wieder unter 50% fällt!**

**Abends (20:00 - SOC: 22%)**
- Zone 1 immer noch aktiv (läuft bis 20%)
- Max. Entladestrom: weiterhin 40A
- Regelziel: weiterhin 0W
- Optional: Nachtabschaltung nicht aktiv (Zone 1 läuft weiter)

**Nacht (22:00 - SOC: 19%)**
- Zone 3 aktiviert (SOC ≤ 20%)
- Modus: `Disabled`
- Max. Entladestrom: **0A** (automatisch gesetzt)
- Ausgang: 0W
- Batterie geschützt

**Nächster Morgen - Zyklus beginnt von vorne**

---

## 🛑 Wichtige Fehlermeldungen

Der Blueprint validiert die Konfiguration beim Start. Fehler werden im **System-Log** angezeigt:

| Meldung | Ursache | Lösung |
|:--------|:--------|:-------|
| **Die obere SOC-Schwelle muss größer sein als die untere** | Schwellenwerte falsch konfiguriert | Obere Schwelle (z.B. 50%) > Untere Schwelle (z.B. 20%) |
| **SOC-Sensor ist UNKNOWN/UNAVAILABLE** | Solakon Integration offline | Prüfen Sie die Verbindung zum Wechselrichter |
| **Timeout Countdown Sensor ist UNKNOWN/UNAVAILABLE** | Sensor nicht verfügbar | Prüfen Sie die Solakon Integration |
| **Modus-Selektor ist UNKNOWN/UNAVAILABLE** | Select-Entity fehlt | Prüfen Sie die Solakon Integration |
| **Ausgangsleistungsregler hat keine min-Attribute** | Number-Entity fehlerhaft | Prüfen Sie die Solakon Integration |

---

## ⚙️ Technische Details

### PI-Regler Implementierung

**Fehlerberechnung (je nach Zone):**
```yaml
Zone 1: error = Min(verfügbare_Kapazität, grid_power)
Zone 2: error = Min(verfügbare_Kapazität, grid_power - offset, pv_kapazität)
```

**Integral-Logik mit Toleranz-Decay:**
```yaml
Wenn |grid_error| > tolerance:
  integral_new = Clamp(integral_old + error, -1000, 1000)
Sonst wenn |integral_old| > 10:
  integral_new = integral_old * 0.95  # 5% Abbau
Sonst:
  integral_new = integral_old  # Keine Änderung
```

**PI-Korrektur:**
```yaml
correction = error * P_Factor + integral_new * I_Factor
new_power = current_power + correction
```

**Finale Leistung (mit zonabhängiger Begrenzung):**
```yaml
Zone 1: final_power = Min(hard_limit, new_power)
Zone 2: final_power = Min(max(0, PV - reserve), new_power)
```

### Automatische Entladestrom-Steuerung

Der Blueprint setzt den **Max. Entladestrom** automatisch:
- **Zone 1 (Aggressive Entladung):** Prüfung ob aktueller Wert ≠ 40A → Setzen auf 40A
- **Zone 2 (Batterieschonend):** Prüfung ob aktueller Wert ≠ 0A → Setzen auf 0A
- **Zone 3 (Sicherheit):** Entladestrom bleibt auf letztem Wert (meist 0A aus Zone 2)

**Wichtig:** Änderungen werden nur vorgenommen, wenn der aktuelle Wert vom Sollwert abweicht (verhindert unnötige API-Calls).

---

## ⚠️ Wichtige Hinweise

1. **Helper erstellen vor Installation:** Beide Helper müssen existieren, bevor der Blueprint konfiguriert wird
2. **Solakon ONE Integration:** Muss vollständig eingerichtet sein
3. **Netzleistungssensor:** Korrekte Polarität (positiv = Bezug, negativ = Einspeisung)
4. **PI-Regler Tuning:** Die Standardwerte sind konservativ. Bei instabilem Verhalten I-Faktor senken.
5. **Integral-Helper:** Wird automatisch verwaltet - nicht manuell ändern!
6. **Queued-Modus:** Trigger werden sequentiell abgearbeitet
7. **Keine Trigger-Verzögerung:** Der Blueprint reagiert sofort auf Sensor-Änderungen
8.  **Regelbare-Wartezeit:** Nach jeder Leistungsänderung wartet der Blueprint 0 - 30 Sekunden
9. **Entladestrom-Automatik:** Max. Entladestrom wird vollautomatisch gesteuert - keine manuelle Einstellung nötig
10. **Toleranz-Decay:** Verhindert automatisch Integral-Windup bei stabiler Regelung

---

## 🔄 Trigger-Übersicht

Der Blueprint reagiert auf folgende Events:

| Trigger | ID | Beschreibung |
|:--------|:---|:------------|
| Grid Power Change | `grid_power_change` | Sofortige PI-Regelung bei Netzleistungsänderung |
| Solar Power Change | `solar_power_change` | Sofortige PI-Regelung bei PV-leistungsänderung |
| SOC High | `soc_high` | Zone 1 Start (SOC > 50%) |
| SOC Low | `soc_low` | Zone 3 Start (SOC ≤ 20%) |
| Mode Change | `mode_change` | Reagiert auf externe Modusänderungen |

**Alle Trigger** führen die komplette Logik aus - keine separate Behandlung nötig.
