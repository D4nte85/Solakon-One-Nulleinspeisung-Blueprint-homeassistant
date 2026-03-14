```mermaid
graph TD
    A[Start: Grid Power, PV or SOC Change] --> B{Critical Entities Available?<br/>SOC Limits Correct?}
    B -- No --> Z((STOP: Critical Error<br/>System Log Entry))
    B -- Yes --> C{SOC > Upper Threshold<br/>AND Cycle = off?}
    
    %% --- ZONE 1 START (Aggressive Discharge) ---
    C -- Yes --> D[ZONE 1 START: Aggressive Discharge<br/>Logbook Entry<br/>Integral Reset to 0<br/>Cycle: set to on<br/>Mode: Discharge PV Prio<br/>Timeout: 10s → 3599s Pulse]
    D --> E
    
    C -- No --> E{SOC ≤ Lower Threshold<br/>AND Cycle = on?}
    
    %% --- ZONE 3 START (Safety STOP) ---
    E -- Yes --> F[ZONE 3 START: Safety STOP<br/>Logbook Entry<br/>Integral Reset to 0<br/>Cycle: set to off<br/>Mode: Disabled<br/>Active Power: 0 W]
    F --> END((End))
    
    %% --- ZONE 3 Additional Safety ---
    E -- No --> G{SOC < Lower Threshold<br/>AND Cycle = off<br/>AND Mode ≠ Disabled?}
    G -- Yes --> H[ZONE 3 SAFETY<br/>Mode: Disabled<br/>Active Power: 0 W]
    H --> END
    
    %% --- ZONE 2 START (Battery Protection) ---
    G -- No --> I{SOC Between Thresholds<br/>AND Cycle = off<br/>AND Mode ≠ Discharge<br/>AND Day OR Night-OFF?}
    I -- Yes --> J[ZONE 2 START: Battery Protection<br/>Logbook Entry<br/>Integral Reset to 0<br/>Mode: Discharge PV Prio<br/>Timeout: 10s → 3599s Pulse]
    J --> K
    
    %% --- NIGHT SHUTDOWN Zone 2 ---
    I -- No --> L{Night Shutdown = on<br/>AND PV < Threshold<br/>AND Cycle = off<br/>AND Mode = Discharge?}
    L -- Yes --> M[NIGHT SHUTDOWN Zone 2<br/>Logbook Entry<br/>Integral Reset to 0<br/>Mode: Disabled<br/>Active Power: 0 W]
    M --> END
    
    %% --- PI CONTROLLER BLOCK ---
    L -- No --> K{Mode = Discharge<br/>AND Cycle = on<br/>OR Day?}
    
    K -- Yes --> N[DISCHARGE CURRENT CONTROL<br/>Zone 1: 40A if ≠ 40A<br/>Zone 2: 0A if ≠ 0A]
    N --> O[TIMEOUT REFRESH<br/>If Countdown < 120s:<br/>10s → 1s Delay → 3599s]
    
    O --> P[PI CONTROLLER CALCULATION<br/>1. Error Zone 1: Min avail. capacity, Grid<br/>   Error Zone 2: Min avail. capacity, Grid-Offset, PV-Capacity<br/>2. Integral: Clamp -1000, +1000 with Tolerance Decay<br/>3. Correction: error×P + integral×I<br/>4. New Power: current + Correction<br/>5. Final Power Zone 1: Min Hard Limit, new Power<br/>   Final Power Zone 2: Min PV-Reserve, new Power]
    
    P --> Q[SAVE INTEGRAL<br/>set input_number]
    
    Q --> R{Error > Tolerance?}
    R -- Yes --> S[SET ACTIVE POWER<br/>Max 0, round final Power]
    S --> END
    R -- No --> END
    
    K -- No --> END
    
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
