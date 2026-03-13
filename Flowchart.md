```mermaid
    flowchart TD
        START([⚡ Trigger Grid / PV / SOC / Modus]) --> VAL

        VAL{{"🛡️ Validierung SOC-Limits & Entitäten"}}
        VAL -- Fehler --> STOP([🛑 Stop + Log-Eintrag])
        VAL -- OK --> ZONE_CHECK

        ZONE_CHECK{{"SOC & Zyklus-Status?"}}

        %% ── Zone 1 ──────────────────────────────────────────────────
        ZONE_CHECK -- "SOC > Zone-1-Schwelle UND Zyklus = off" --> Z1_START
        Z1_START["🔋 Zone 1 aktivieren   Zyklus = on   Integral = 0   Surplus-Boolean → off (nur wenn Zone 0 aktiv)   Modus → INV Discharge PV Priority   (Puls-Sequenz: 10s → 3599s)"]

        %% ── Zone 3 (Zyklus on) ──────────────────────────────────────
        ZONE_CHECK -- "SOC < Zone-3-Schwelle UND Zyklus = on" --> Z3_A
        Z3_A["🛑 Zone 3 aktivieren   Zyklus = off   Integral = 0   Surplus-Boolean → off (nur wenn Zone 0 aktiv)   Modus → Disabled   Output → 0 W"]

        %% ── Zone 3 (Absicherung) ────────────────────────────────────
        ZONE_CHECK -- "SOC < Zone-3-Schwelle UND Zyklus = off UND Modus ≠ Disabled" --> Z3_B
        Z3_B["🛑 Zone 3 Absicherung   Surplus-Boolean → off (nur wenn Zone 0 aktiv)   Modus → Disabled   Output → 0 W"]

        %% ── Recovery ────────────────────────────────────────────────
        ZONE_CHECK -- "Zyklus = on UND Modus ≠ INV Discharge UND SOC > Zone-3-Schwelle" --> RECOVERY
        RECOVERY["🔄 Recovery — Modus-Reaktivierung   Modus → INV Discharge PV Priority   (Puls-Sequenz: 10s → 3599s)"]

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
        RECOVERY --> PI_GATE
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
        SURPLUS_CHECK -- Ja --> SURPLUS_STATE

        %% ── Surplus Zwei-Zustands-Logik ─────────────────────────────
        SURPLUS_STATE{{"Aktuell im Überschuss-Modus?   (input_boolean = on)"}}
        SURPLUS_STATE -- "Ja — Bleiben:   SOC ≥ (Export-Schwelle − SOC-Hysterese)   UND PV > Output + Grid − PV-Hysterese" --> CALC_SURPLUS
        SURPLUS_STATE -- "Nein — Eintreten:   SOC ≥ Export-Schwelle   UND Grid ≤ Offset + Toleranz   UND PV ≥ Nacht-Schwelle   UND PV > Output + Grid + PV-Hysterese" --> CALC_SURPLUS
        SURPLUS_STATE -- "Bedingung nicht erfüllt" --> CALC_NORMAL

        %% ── Zone-0-Pfad ─────────────────────────────────────────────
        CALC_SURPLUS["☀️ Zone 0 — Überschuss-Einspeisung   final_power = Hard Limit"]
        CALC_SURPLUS --> DISCHARGE_SET

        %% ── Normaler PI-Pfad ────────────────────────────────────────
        CALC_NORMAL["🧠 PI-Regler   1. Fehler berechnen (zonenabhängig)      Zone 1: Min(Kapazität, Grid − Offset₁)      Zone 2: Min(Kapazität, Grid − Offset₂, PV-Kap.)   2. Integral aktualisieren (Anti-Windup)      |Grid-Fehler| > Toleranz → Integral += Fehler (±1000)      |Grid-Fehler| ≤ Toleranz UND |Integral| > 10 → Integral × 0,95      sonst → kein Update   3. Korrektur = P-Teil + I-Teil   4. new_power = current + Korrektur   5. Begrenzen:      Zone 1 → Hard Limit      Zone 2 → Max(0, PV − Reserve)"]
        CALC_NORMAL --> DISCHARGE_SET

        %% ── Entladestrom setzen ─────────────────────────────────────
        DISCHARGE_SET{{"Entladestrom setzen?"}}
        DISCHARGE_SET -- "Surplus aktiv → 2 A" --> TIMEOUT_CHECK
        DISCHARGE_SET -- "Zyklus = on → Max-Wert" --> TIMEOUT_CHECK
        DISCHARGE_SET -- "Sonst → 0 A" --> TIMEOUT_CHECK

        %% ── Timeout ─────────────────────────────────────────────────
        TIMEOUT_CHECK{{"Countdown < 120s?"}}
        TIMEOUT_CHECK -- Ja --> TIMEOUT_RESET["⏱️ Timeout-Reset 10s → (1s Pause) → 3599s"]
        TIMEOUT_CHECK -- Nein --> INTEGRAL_SAVE
        TIMEOUT_RESET --> INTEGRAL_SAVE

        %% ── Integral & Output ───────────────────────────────────────
        INTEGRAL_SAVE["💾 Integral-Wert speichern   Zone 0: integral_old (eingefroren)   Sonst: integral_new"]
        INTEGRAL_SAVE --> TOL_CHECK

        TOL_CHECK{{"Überschuss-Modus aktiv?   → Output < Hard Limit?   SONST Normal-Modus   → |Grid-Fehler| > Toleranz?"}}
        TOL_CHECK -- Nein --> BOOL_UPDATE["🔁 Surplus-Boolean aktualisieren   (on/off je nach is_surplus_mode)"]
        BOOL_UPDATE --> END_TOL([Ende — kein Output keine unnötigen API-Calls])
        TOL_CHECK -- Ja --> SET_OUTPUT

        SET_OUTPUT["⚙️ Ausgangsleistung setzen   Überschuss-Modus: Hard Limit + Boolean → on   Normal-Modus: Max(0, final_power) + Boolean → off   → Wechselrichter"]
        SET_OUTPUT --> WAIT["⏳ Wartezeit (0–30s)"]
        WAIT --> BOOL_UPDATE

        %% ── Styles ──────────────────────────────────────────────────
        classDef zone0 fill:#fff3cd,stroke:#f0ad4e,color:#000
        classDef zone1 fill:#d4edda,stroke:#28a745,color:#000
        classDef zone2 fill:#d1ecf1,stroke:#17a2b8,color:#000
        classDef zone3 fill:#f8d7da,stroke:#dc3545,color:#000
        classDef night fill:#e2d9f3,stroke:#6f42c1,color:#000
        classDef recovery fill:#fde8d0,stroke:#e07b20,color:#000
        classDef pi fill:#cce5ff,stroke:#004085,color:#000
        classDef end_node fill:#f8f9fa,stroke:#6c757d,color:#000

        class CALC_SURPLUS,SURPLUS_STATE zone0
        class Z1_START zone1
        class Z2_START zone2
        class Z3_A,Z3_B zone3
        class NIGHT night
        class RECOVERY recovery
        class CALC_NORMAL,DISCHARGE_SET pi
        class END_STOP,END_SKIP,END_TOL,END_OK,BOOL_UPDATE end_node
``````mermaid
graph TD
    A[Start: Netzleistung, PV oder SOC Änderung] --> B{Kritische Entitäten vorhanden?<br/>SOC Limits korrekt?}
    B -- Nein --> Z((STOPP: Kritischer Fehler<br/>System Log Eintrag))
    B -- Ja --> C{SOC > Obere Schwelle<br/>UND Zyklus = off?}

    %% --- ZONE 1 START (Aggressive Entladung) ---
    C -- Ja --> D[ZONE 1 START: Aggressive Entladung<br/>Logbook-Eintrag<br/>Integral Reset auf 0<br/>Zyklus: on setzen<br/>Modus: Discharge PV Prio<br/>Timeout: 10s → 3599s Puls]
    D --> E

    C -- Nein --> E{SOC ≤ Untere Schwelle<br/>UND Zyklus = on?}

    %% --- ZONE 3 START (Sicherheits-STOPP) ---
    E -- Ja --> F[ZONE 3 START: Sicherheits-STOPP<br/>Logbook-Eintrag<br/>Integral Reset auf 0<br/>Zyklus: off setzen<br/>Modus: Disabled<br/>Active Power: 0 W]
    F --> END((Ende))

    %% --- ZONE 3 Zusatz-Absicherung ---
    E -- Nein --> G{SOC < Untere Schwelle<br/>UND Zyklus = off<br/>UND Modus ≠ Disabled?}
    G -- Ja --> H[ZONE 3 ABSICHERUNG<br/>Modus: Disabled<br/>Active Power: 0 W]
    H --> END

    %% --- ZONE 2 START (Batterieschonend) ---
    G -- Nein --> I{SOC zwischen Schwellen<br/>UND Zyklus = off<br/>UND Modus ≠ Discharge<br/>UND Tag ODER Nacht-AUS?}
    I -- Ja --> J[ZONE 2 START: Batterieschonend<br/>Logbook-Eintrag<br/>Integral Reset auf 0<br/>Modus: Discharge PV Prio<br/>Timeout: 10s → 3599s Puls]
    J --> K

    %% --- NACHTABSCHALTUNG Zone 2 ---
    I -- Nein --> L{Nachtabschaltung = on<br/>UND PV < Schwelle<br/>UND Zyklus = off<br/>UND Modus = Discharge?}
    L -- Ja --> M[NACHTABSCHALTUNG Zone 2<br/>Logbook-Eintrag<br/>Integral Reset auf 0<br/>Modus: Disabled<br/>Active Power: 0 W]
    M --> END

    %% --- PI-REGLER BLOCK ---
    L -- Nein --> K{Modus = Discharge<br/>UND Zyklus = on<br/>ODER Tag?}

    K -- Ja --> N[ENTLADESTROM-STEUERUNG<br/>Zone 1: 40A wenn ≠ 40A<br/>Zone 2: 0A wenn ≠ 0A]
    N --> O[TIMEOUT-REFRESH<br/>Wenn Countdown < 120s:<br/>10s → 1s Delay → 3599s]

    O --> P[PI-REGLER BERECHNUNG<br/>1. Fehler Zone 1: Min verfüg. Kapazität, Grid<br/>   Fehler Zone 2: Min verfüg. Kapazität, Grid-Offset, PV-Kapazität<br/>2. Integral: Clamp-1000, +1000 mit Toleranz-Decay<br/>3. Korrektur: error×P + integral×I<br/>4. Neue Power: current + Korrektur<br/>5. Finale Power Zone 1: Min Hard Limit, neue Power<br/>   Finale Power Zone 2: Min PV-Reserve, neue Power]

    P --> Q[INTEGRAL SPEICHERN<br/>input_number setzen]

    Q --> R{Fehler > Toleranz?}
    R -- Ja --> S[ACTIVE POWER SETZEN<br/>Max 0, finale Power runden]
    S --> END
    R -- Nein --> END

    K -- Nein --> END

    %% --- STYLING ---
    style D fill:#c9ffc9,stroke:#333,stroke-width:2px
    style J fill:#c9ffc9,stroke:#333,stroke-width:2px
    style F fill:#ffcccc,stroke:#333,stroke-width:2px
    style H fill:#ffcccc,stroke:#333,stroke-width:2px
    style M fill:#ffd699,stroke:#333,stroke-width:2px
    style P fill:#fff7c2,stroke:#333,stroke-width:2px
    style N fill:#d4e6f1,stroke:#333,stroke-width:2px
    style O fill:#d4e6f1,stroke:#333,stroke-width:2px
    style Q fill:#d4e6f1,stroke:#333,stroke-width:2px
    style S fill:#d4e6f1,stroke:#333,stroke-width:2px
    style END fill:#cccccc,stroke:#333,stroke-width:2px
    style Z fill:#cccccc,stroke:#333,stroke-width:2px
