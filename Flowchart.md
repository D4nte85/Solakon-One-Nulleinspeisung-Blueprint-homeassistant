```mermaid
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
