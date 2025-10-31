```mermaid
graph TD
    A[Start: Netzleistung, PV oder SOC Aenderung] --> B{"Kritische Entitaeten vorhanden?"}

    B -- Nein --> Z((STOPP: Kritischer Fehler))

    B -- Ja --> C{"SOC Groesser Obere Schwelle UND Zyklus Aus?"}

    %% --- ZONE 1 START (Aggressive Entladung) ---
    C -- Ja --> D[ZONE 1: Aggressive Entladung<br>Modus: Discharge PV Prio setzen<br>Max. Strom: Max Konfig<br>PI-Ziel: 0 W<br>Zyklus: Ein]
    D --> E

    C -- Nein --> E

    E{"Zyklus Ein ODER SOC In-Between Zone 2?"}

    %% --- ZONE 3 / Zyklus End ---
    E -- Zyklus Ein UND SOC Kleiner Untere Schwelle --> F[ZONE 3: Sicherheits-STOPP<br>Zyklus: Aus<br>Modus: Disabled<br>Active Power: 0 W]
    F --> END((Ende))

    %% --- ZONE 2 START (Batterieschonend) ---
    E -- Zyklus Aus UND SOC In-Between Zone 2 --> G{"Nachtabschaltung aktiv UND PV Kleiner Schwelle?"}

    G -- Ja --> F
    
    G -- Nein --> H[ZONE 2: Batterieschonend<br>Modus: Discharge PV Prio setzen<br>Max. Strom: 0 A<br>PI-Ziel: +Offset<br>Power Limit: Max 0, PV - Reserve]
    H --> I

    %% --- PI-Regler / Kontinuierliche Steuerung ---
    I{"Modus Discharge aktiv UND Tag?"}
    
    I -- Ja --> J[PI-REGLER-LOGIK<br>1. Fehler = Netzleistung - PI-Ziel<br>2. Neue Power = P + I Anteil<br>3. Active Power Limit anwenden<br>4. WENN Fehler Groesser Toleranz: Integral speichern & Power setzen<br>5. WENN Timeout Kleiner 120s: Timeout Reset]
    J --> END

    I -- Nein --> END

    style D fill:#c9ffc9,stroke:#333
    style H fill:#c9ffc9,stroke:#333
    style F fill:#ffcccc,stroke:#333
    style J fill:#fff7c2,stroke:#333
    style END fill:#cccccc,stroke:#333
    style Z fill:#cccccc,stroke:#333
