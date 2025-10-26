# 🔋 Solakon ONE Nulleinspeisung Blueprint (V113)

[![Home Assistant Blueprint](https://img.shields.io/badge/Home%20Assistant-Blueprint-41bdf5.svg?style=for-the-badge)](https://my.home-assistant.io/redirect/blueprint_import?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

Dieser Blueprint steuert den Solakon ONE Wechselrichter über einen **dreistufigen, SOC-abhängigen Regelmechanismus** zur dynamischen Nulleinspeisung. Er nutzt einen Proportional-Regler (P-Regler), um die Netzleistung zu steuern und schaltet in eine batterieschonende Lade-Prioritätszone im mittleren Ladestand.

### 📥 Direktimport in Home Assistant

Klicke auf den Button, um den Blueprint direkt in deiner Home Assistant Instanz zu importieren:

[![Open your Home Assistant instance and start setting up this blueprint.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import?blueprint_url=https%3A%2F%2Fgithub.com%2FD4nte85%2FSolakon-One-Nulleinspeisung-Blueprint-homeassistant%2Fblob%2Fmain%2Fsolakon_one_nulleinspeisung.yaml)

***

## ⚙️ Funktionsweise

Der Kern der Steuerung ist ein **P-Regler**, der die Differenz zwischen der gemessenen Netzleistung und einem Ziel-Offset (oft 0 W) zur dynamischen Anpassung des AC-Output-Limits nutzt.

Die Logik ist in drei SOC-Zonen unterteilt:

1.  **🚀 Schnelle Regelung ($\text{SOC} > 50\%$ oder Zyklus aktiv):** Aggressive P-Regelung mit $\text{Offset} = 0\text{ W}$ zur schnellen Entladung und exakten Nulleinspeisung.
2.  **🐢 Batterieschonende Regelung ($\text{20\%} < \text{SOC} \le 50\%$):** **Passiver Modus** mit Lade-Priorität. Der **Nullpunkt-Offset** wird negativ gesetzt, um leichten Netzbezug zu erzwingen. Die Entladeleistung wird zusätzlich um die **PV-Ladereserve** reduziert, um interne Wandlerverluste zu kompensieren und die Batterieladung zu garantieren.
3.  **🛑 Sicherheitsstopp ($\text{SOC} \le 20\%$):** Umschaltung auf **`Disabled`** und $\text{AC-Limit} = 0\text{ W}$ zum Schutz der Batterie.

***

## 🛠️ Voraussetzungen

### 1. Helfer: Entladezyklus-Zustandsspeicher

Dieser Blueprint benötigt einen **Input Select Helfer**, um zu verfolgen, ob der aggressive Entladezyklus aktiv ist.

| Parameter | Wert |
| :--- | :--- |
| **Name** | z.B. `Solakon Entladezyklus Status` |
| **Typ** | **Input Select / Auswahl** |
| **Optionen** | `on`, `off` |

***

## 📝 Variablen (User Input)

| Name der Variable | Beschreibung | Standardwert |
| :--- | :--- | :--- |
| **Shelly/Netz-Leistungssensor** | Sensor, der die aktuelle Netzleistung misst. (Muss `power` device_class haben) | (Kein Standard) |
| **Solakon ONE - Solarleistung (PV-Erzeugung)** | Sensor für die aktuelle PV-Leistung in Watt. | (Kein Standard) |
| **Solakon ONE - Batterieladestand (SOC)** | SOC-Sensor des Solakon ONE (%). | (Kein Standard) |
| **Solakon ONE - Ausgangsleistungsregler (AC-Output)** | Die **Number** Entität zur Steuerung des AC-Output-Limits. | (Kein Standard) |
| **Solakon ONE - Betriebsmodus-Auswahl** | Die **Select** Entität zur Steuerung des Betriebsmodus. | (Kein Standard) |
| **Modus-Reset-Timer-Entität** | Die Solakon **Number** Entität (`remote_timeout_control`) zum Zurücksetzen des Remote Timeouts. | (Kein Standard) |
| **Remote Timeout Countdown Sensor** | Sensor/Number Entität, die den aktuell verbleibenden Countdown anzeigt. | (Kein Standard) |
| **Entladezyklus-Zustandsspeicher** | Der zuvor erstellte **Input Select Helfer** (`on`/`off`). | (Kein Standard) |
| **SOC-Schwelle "Schnelle Regelung"** | Oberer SOC-Wert, ab dem der aggressive Entladezyklus startet. | `50 %` |
| **SOC-Schwelle "Lade-Priorität"** | Unterer SOC-Wert, unter dem in den Disabled-Modus gewechselt wird. | `20 %` |
| **Toleranzbereich (Halbbreite)** | Zulässiger Bereich in Watt um den Nullpunkt. | `50 W` |
| **Regelungs-Faktor (Korrektur-Geschw.)** | Aggressivität des P-Reglers. | `1.5` |
| **Nullpunkt-Offset** | Negative Watt-Zahl, um den Nullpunkt in Zone 2 in den Netzbezug zu verschieben (Lade-Priorität). | `-30 W` |
| **🔋 PV-Ladereserve (Zwischen-SOC)** | Reservierte PV-Leistung (Watt) zum Kompensieren interner Wandlerverluste in Zone 2. | `15 W` |
