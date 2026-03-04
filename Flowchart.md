```mermaid
    flowchart TD
        START([⚡ Trigger Grid / PV / SOC / Modus]) --> VAL

        VAL{{"🛡️ Validierung SOC-Limits & Entitäten"}}
        VAL -- Fehler --> STOP([🛑 Stop + Log-Eintrag])
        VAL -- OK --> ZONE_CHECK

        ZONE_CHECK{{"SOC & Zyklus-Status?"}}

        %% ── Zone 1 ──────────────────────────────────────────────────
        ZONE_CHECK -- "SOC > Zone-1-Schwelle UND Zyklus = off" --> Z1_START
        Z1_START["🔋 Zone 1 aktivieren   Zyklus = on   Integral = 0   Modus → INV Discharge PV Priority   (Puls-Sequenz: 10s → 3599s)"]

        %% ── Zone 3 (Zyklus on) ──────────────────────────────────────
        ZONE_CHECK -- "SOC ≤ Zone-3-Schwelle UND Zyklus = on" --> Z3_A
        Z3_A["🛑 Zone 3 aktivieren   Zyklus = off   Integral = 0   Modus → Disabled   Output → 0 W"]

        %% ── Zone 3 (Absicherung) ────────────────────────────────────
        ZONE_CHECK -- "SOC < Zone-3-Schwelle UND Zyklus = off UND Modus ≠ Disabled" --> Z3_B
        Z3_B["🛑 Zone 3 Absicherung   Modus → Disabled   Output → 0 W"]

        %% ── Zone 2 ──────────────────────────────────────────────────
        ZONE_CHECK -- "Zone-3 < SOC ≤ Zone-1 UND Zyklus = off UND Modus = Disabled UND NICHT Nacht" --> Z2_START
        Z2_START["🔋 Zone 2 aktivieren   Integral = 0   Modus → INV Discharge PV Priority   (Puls-Sequenz: 10s → 3599s)"]

        %% ── Nachtabschaltung ────────────────────────────────────────
        ZONE_CHECK -- "Nachtabschaltung aktiv UND PV < Schwelle UND Zyklus = off UND Modus aktiv" --> NIGHT
        NIGHT["🌙 Nachtabschaltung   Integral = 0   Modus → Disabled   Output → 0 W"]

        %% ── Kein Zonenwechsel ───────────────────────────────────────
        ZONE_CHECK -- "Kein Zonenwechsel" --> PI_GATE

        Z1_START --> PI_GATE
        Z2_START --> PI_GATE
        Z3_A --> END_STOP([Ende])
        Z3_B --> END_STOP
        NIGHT --> END_STOP

        %% ── PI-Regler Gate ──────────────────────────────────────────
        PI_GATE{{"Modus = INV Discharge? UND Zone 1 ODER Tag?"}}
        PI_GATE -- Nein --> END_SKIP([Ende — kein Output])
        PI_GATE -- Ja --> SURPLUS_CHECK

        %% ── Überschuss-Prüfung ──────────────────────────────────────
        SURPLUS_CHECK{{"☀️ Überschuss-Einspeisung aktiviert?"}}
        SURPLUS_CHECK -- Nein --> CALC_NORMAL
        SURPLUS_CHECK -- Ja --> SURPLUS_COND

        SURPLUS_COND{{"SOC ≥ Export-Schwelle UND PV > Haus + 50W?"}}
        SURPLUS_COND -- Nein --> CALC_NORMAL
        SURPLUS_COND -- Ja --> CALC_SURPLUS

        %% ── Zone-0-Pfad ─────────────────────────────────────────────
        CALC_SURPLUS["☀️ Zone 0 — Überschuss-Einspeisung   Entladestrom → 0 A   final_power = Hard Limit   Integral gespeichert"]
        CALC_SURPLUS --> TIMEOUT_CHECK

        %% ── Normaler PI-Pfad ────────────────────────────────────────
        CALC_NORMAL["🧠 PI-Regler 1. Fehler berechnen (zonabhängig) 2. Integral aktualisieren (Anti-Windup) 3. Korrektur = P-Teil + I-Teil 4. new_power = current + Korrektur 5. Begrenzen: Zone 1 → Hard Limit               Zone 2 → Max(0, PV-Reserve)"]
        CALC_NORMAL --> DISCHARGE_SET

        %% ── Entladestrom setzen ─────────────────────────────────────
        DISCHARGE_SET{{"Zyklus = on?"}}
        DISCHARGE_SET -- "Ja (Zone 1)" --> SET_40A["🔋 Entladestrom → konfigurierter Max-Wert (nur wenn Wert abweicht)"]
        DISCHARGE_SET -- "Nein (Zone 2)" --> SET_0A["🔋 Entladestrom → 0 A (nur wenn Wert abweicht)"]
        SET_40A --> TIMEOUT_CHECK
        SET_0A --> TIMEOUT_CHECK

        %% ── Timeout ─────────────────────────────────────────────────
        TIMEOUT_CHECK{{"Countdown < 120s?"}}
        TIMEOUT_CHECK -- Ja --> TIMEOUT_RESET["⏱️ Timeout-Reset 10s → (1s Pause) → 3599s"]
        TIMEOUT_CHECK -- Nein --> INTEGRAL_SAVE
        TIMEOUT_RESET --> INTEGRAL_SAVE

        %% ── Integral & Output ───────────────────────────────────────
        INTEGRAL_SAVE["💾 Integral-Wert speichern"]
        INTEGRAL_SAVE --> TOL_CHECK

        TOL_CHECK{{"Toleranz überschritten? ODER Surplus-Status geändert?"}}
        TOL_CHECK -- Nein --> END_TOL([Ende — kein Output keine unnötigen API-Calls])
        TOL_CHECK -- Ja --> SET_OUTPUT

        SET_OUTPUT["⚙️ Ausgangsleistung setzen Max(0, final_power) → Wechselrichter"]
        SET_OUTPUT --> WAIT["⏳ Wartezeit (0–30s)"]
        WAIT --> END_OK([Ende])

        %% ── Styles ──────────────────────────────────────────────────
        classDef zone0 fill:#fff3cd,stroke:#f0ad4e,color:#000
        classDef zone1 fill:#d4edda,stroke:#28a745,color:#000
        classDef zone2 fill:#d1ecf1,stroke:#17a2b8,color:#000
        classDef zone3 fill:#f8d7da,stroke:#dc3545,color:#000
        classDef night fill:#e2d9f3,stroke:#6f42c1,color:#000
        classDef pi fill:#cce5ff,stroke:#004085,color:#000
        classDef end_node fill:#f8f9fa,stroke:#6c757d,color:#000

        class CALC_SURPLUS,SURPLUS_COND zone0
        class Z1_START zone1
        class Z2_START zone2
        class Z3_A,Z3_B zone3
        class NIGHT night
        class CALC_NORMAL,DISCHARGE_SET,SET_40A,SET_0A pi
        class END_STOP,END_SKIP,END_TOL,END_OK end_node
