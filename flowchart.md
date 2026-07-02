````mermaid
flowchart TD

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_ENTRY ["⚡ Entry & Validation"]
        START([⚡ Trigger Grid / PV / SOC / Mode]) --> VAL
        VAL{{"🛡️ Validation SOC Limits & Entities"}}
        VAL -- Error --> STOP([🛑 Stop + Log Entry])
    end
    style SG_ENTRY fill:none,stroke:#aaaaaa,stroke-width:5,stroke-dasharray:6,color:#000

    VAL -- OK --> SURPLUS_UPDATE

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_ZONE0 ["☀️ Zone 0 — Surplus Status Update   (Cases 0A / 0B)"]
        SURPLUS_UPDATE{{"☀️ Zone 0 Status Update   (Cases 0A / 0B)"}}
        S0A["☀️ Zone 0 activate   Surplus-Bool → on"]
        S0B["☁️ Zone 0 deactivate   Surplus-Bool → off   Integral = 0"]
    end
    style SG_ZONE0 fill:none,stroke:#f0ad4e,stroke-width:5,stroke-dasharray:6,color:#000

    SURPLUS_UPDATE -- "CASE 0A   Surplus-Bool = off   AND (SOC ≥ Export Threshold AND (PV > Output+Grid+PV-Hysteresis OR PV=0)   OR Surplus-Forecast-Forced AND PV > Hard Limit)" --> S0A
    SURPLUS_UPDATE -- "CASE 0B   Surplus-Bool = on   AND (PV ≤ Output + Grid − PV-Hysteresis   OR (NOT Surplus-Forecast-Forced AND SOC < Export Threshold − SOC-Hysteresis))" --> S0B
    SURPLUS_UPDATE -- "No Surplus Update" --> ZONE_CHECK
    S0A --> ZONE_CHECK
    S0B --> ZONE_CHECK

    ZONE_CHECK{{"SOC & Cycle Status?   (Cases A – F, GT, HT, TM, G, H, I)"}}

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_SOC ["🔋 SOC Zone Control   (Cases A, B, C, D, E, F)"]

        %% ── Case A: Zone 1 ───────────────────────────────────────────────
        Z1_START["🔋 Zone 1 activate   Cycle = on   Integral = 0   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]

        %% ── Case B: Zone 3 (Cycle on) ──────────────────────────────────
        Z3_A["🛑 Zone 3 activate   Cycle = off   Integral = 0   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

        %% ── Case C: Zone 3 (Guard) ────────────────────────────────────
        Z3_B["🛑 Zone 3 Guard   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

        %% ── Case D: Recovery ─────────────────────────────────────────────
        RECOVERY["🔄 Recovery — Mode Reactivation   Timer-Toggle (3598↔3599)   AC-Charge-Bool = on OR Tariff-Charge-Bool = on → Mode '3'   otherwise → Mode '1'   (no integral reset, no zone change)"]

        %% ── Case E: Zone 2 ───────────────────────────────────────────────
        Z2_START["🔋 Zone 2 activate   Integral = 0   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]

        %% ── Case F: Night Shutdown ─────────────────────────────────────
        NIGHT["🌙 Night Shutdown   Integral = 0   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

    end
    style SG_SOC fill:none,stroke:#28a745,stroke-width:5,stroke-dasharray:6,color:#000

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_TARIFF ["💹 Tariff Arbitrage   (Cases GT, HT, TM)"]

        %% ── Case GT: Tariff Charging Entry ────────────────────────────────
        TARIFF_START["💹 Tariff Charging activate   Tariff-Charge-Bool → on   Timer-Toggle (3598↔3599)   Output → Charge Power (direct, no PI)   Mode → '3' (INV Charge PV Priority)"]

        %% ── Case HT: Tariff Charging End ─────────────────────────────────
        TARIFF_END{{"Integral = 0   Tariff-Charge-Bool → off   Current Zone?"}}
        TARIFF_END_Z1["💹 Tariff Charging end (Zone 1)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]
        TARIFF_END_Z2["💹 Tariff Charging end (Zone 2)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

        %% ── Case TM: Discharge Lock ──────────────────────────────────────
        TARIFF_MID["🔒 Discharge Lock   Integral = 0   Cycle = off (only if Zone 1)   Surplus-Bool → off (only if active)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

    end
    style SG_TARIFF fill:none,stroke:#1a7f1a,stroke-width:5,stroke-dasharray:6,color:#000

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_AC ["⚡ AC Charging & Safety   (Cases G, H, I)"]

        %% ── Case G: AC Charging Entry ────────────────────────────────────
        AC_START["⚡ AC Charging activate   AC-Charge-Bool → on   Timer-Toggle (3598↔3599)   Output → 0 W (PI starts clean)   Mode → '3' (INV Charge PV Priority)"]

        %% ── Case H: AC Charging End ─────────────────────────────────────
        AC_END{{"Integral = 0   AC-Charge-Bool → off   Current Zone?"}}
        AC_END_Z1["⚡ AC Charging end (Zone 1)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]
        AC_END_Z2["⚡ AC Charging end (Zone 2)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

        %% ── Case I: Safety — Mode '3' without active charge session ─────
        SAFETY_I{{"Integral = 0   Current Zone?"}}
        SAFETY_I_Z1["⚠️ Safety (Zone 1)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]
        SAFETY_I_Z2["⚠️ Safety (Zone 2)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '0' (Disabled)"]

    end
    style SG_AC fill:none,stroke:#0066cc,stroke-width:5,stroke-dasharray:6,color:#000

    %% ── ZONE_CHECK → Case branches ──────────────────────────────────────
    ZONE_CHECK -- "CASE A   NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on   AND NOT Discharge Lock (price < expensive)   AND SOC > Zone 1 threshold AND Cycle = off" --> Z1_START
    ZONE_CHECK -- "CASE B   NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on   AND SOC < Zone 3 threshold AND Cycle = on" --> Z3_A
    ZONE_CHECK -- "CASE C   NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on   AND SOC < Zone 3 threshold AND Cycle = off AND Mode ≠ '0'" --> Z3_B
    ZONE_CHECK -- "CASE D   Cycle = on   AND Mode ∉ {'1','3'} ← '3' explicitly excluded!   AND SOC > Zone 3 threshold" --> RECOVERY
    ZONE_CHECK -- "CASE GT   Tariff Arbitrage enabled   AND price < cheap threshold   AND SOC < tariff charge target   AND Mode ≠ '3' ← Guard!   AND NOT Surplus-Bool = on   AND NOT PV-Forecast-Suppressed" --> TARIFF_START
    ZONE_CHECK -- "CASE HT   Mode = '3'   AND Tariff-Charge-Bool = on   AND (price ≥ cheap threshold OR SOC ≥ tariff charge target)" --> TARIFF_END
    ZONE_CHECK -- "CASE TM   Tariff active   AND cheap ≤ price < expensive   AND no AC/Tariff charging   AND Mode = '1'   AND NOT PV-Forecast-Suppressed" --> TARIFF_MID
    ZONE_CHECK -- "CASE G   AC Charging enabled   AND SOC < charge target   AND Mode ≠ '3' ← Guard!   AND NOT Tariff-Charge-Bool = on   AND NOT Surplus-Bool = on   AND (Grid + Output) < −Hysteresis" --> AC_START
    ZONE_CHECK -- "CASE H   Mode = '3'   AND (SOC ≥ charge target OR (Grid ≥ AC-Offset + Hysteresis AND Output = 0 W))" --> AC_END
    ZONE_CHECK -- "CASE I   Mode = '3'   AND NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on" --> SAFETY_I
    ZONE_CHECK -- "CASE E   NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on   AND NOT Discharge Lock (price < expensive)   AND Zone 3 < SOC ≤ Zone 1 AND Cycle = off AND Mode = '0' AND NOT night" --> Z2_START
    ZONE_CHECK -- "CASE F   NOT AC-Charge-Bool = on   AND NOT Tariff-Charge-Bool = on   AND Night Shutdown active AND PV < PV Charge Reserve AND Cycle = off AND Mode active" --> NIGHT
    ZONE_CHECK -- "No zone change" --> PI_GATE

    TARIFF_END -- "Cycle = on (Zone 1)" --> TARIFF_END_Z1
    TARIFF_END -- "Cycle = off (Zone 2)" --> TARIFF_END_Z2
    AC_END -- "Cycle = on (Zone 1)" --> AC_END_Z1
    AC_END -- "Cycle = off (Zone 2)" --> AC_END_Z2
    SAFETY_I -- "Cycle = on (Zone 1)" --> SAFETY_I_Z1
    SAFETY_I -- "Cycle = off (Zone 2)" --> SAFETY_I_Z2

    Z1_START --> PI_GATE
    Z2_START --> PI_GATE
    RECOVERY --> PI_GATE

    Z3_A --> END_STOP([End])
    Z3_B --> END_STOP
    NIGHT --> END_STOP
    TARIFF_START --> END_STOP
    TARIFF_END_Z1 --> END_STOP
    TARIFF_END_Z2 --> END_STOP
    TARIFF_MID --> END_STOP
    AC_START --> END_STOP
    AC_END_Z1 --> END_STOP
    AC_END_Z2 --> END_STOP
    SAFETY_I_Z1 --> END_STOP
    SAFETY_I_Z2 --> END_STOP

    %% ══════════════════════════════════════════════════════════════════════
    subgraph SG_PI ["🧠 PI Controller Operation"]

        %% ── PI Gate ───────────────────────────────────────────────
        PI_GATE{{"Mode ∈ {'1','3'}?   AND (Cycle = on OR Night lock inactive OR PV ≥ Reserve)"}}
        PI_GATE -- No --> END_SKIP([End — no output])
        PI_GATE -- Yes --> DISCHARGE_SET

        %% ── Step 1: Discharge Current ──────────────────────────────────
        DISCHARGE_SET{{"Set discharge current?"}}
        DISCHARGE_SET -- "Surplus active → 2 A   (only if different)" --> TIMEOUT_CHECK
        DISCHARGE_SET -- "Cycle = on AND Mode ≠ '3' AND Surplus NOT active → Max value   (only if different)" --> TIMEOUT_CHECK
        DISCHARGE_SET -- "Cycle = off OR Mode = '3' AND Surplus NOT active → 0 A   (only if different)" --> TIMEOUT_CHECK

        %% ── Step 2: Timeout Reset ─────────────────────────────────────
        TIMEOUT_CHECK{{"Countdown < 120s?"}}
        TIMEOUT_CHECK -- Yes --> TIMEOUT_RESET["⏱️ Timer-Toggle (3598↔3599)"]
        TIMEOUT_CHECK -- No --> PI_DECISION
        TIMEOUT_RESET --> PI_DECISION

        %% ── Step 3: Output & Integral ─────────────────────────────────
        PI_DECISION{{"Surplus-Bool = on?   → Zone 0 active"}}
        PI_DECISION -- Yes --> CALC_SURPLUS
        PI_DECISION -- No --> TARIFF_GATE

        %% ── Zone 0 Path ──────────────────────────────────────────────
        CALC_SURPLUS["☀️ Zone 0 — Surplus Export   Output → Hard Limit   Freeze integral (integral_old unchanged)   Wait time"]

        %% ── Tariff Charging Path ─────────────────────────────────────
        TARIFF_GATE{{"Tariff-Charge-Bool = on?"}}
        TARIFF_GATE -- Yes --> CALC_TARIFF
        TARIFF_GATE -- No --> AC_GATE

        CALC_TARIFF["💹 Tariff Charging — Direct set (no PI)   Output → tariff_charge_power   Wait time"]

        %% ── AC Charging Path ────────────────────────────────────────
        AC_GATE{{"AC-Charge-Bool = on?   AND |grid error| > tolerance?"}}
        AC_GATE -- Yes --> CALC_AC
        AC_GATE -- No --> NORMAL_GATE

        CALC_AC["⚡ AC Charging — PI Script (ac_charge_mode=true)   raw_error = (ac_charge_offset − grid) × error_share   error_share: usable_i / Σ(usable_j) — set by power distribution (default 1.0)   usable_i = (SOC_i−Min-SOC_i)/100 × Cap_i   (Cap_i = Cap sensor kWh or 100 if not set)   max_power = ac_charge_power_limit   Capacity clamping   correction = P·error + I·integral_candidate   (separate ac_charge_p/i_factor)   new_power = current + correction   clamp(0, charge limit) → Output → inverter   Anti-windup: integral = (final − current − P·error) / I → clamp(±effective_max)   Wait time   (at_max / at_min guards NOT applied)"]

        %% ── Normal PI Path ─────────────────────────────────────────
        NORMAL_GATE{{"Error > tolerance?   AND no at_max / at_min limit?   (at_max = false if current > dynamic_max — PI corrects downward)"}}
        NORMAL_GATE -- Yes --> CALC_NORMAL
        NORMAL_GATE -- No --> INTEGRAL_DECAY

        CALC_NORMAL["🧠 PI Script (ac_charge_mode=false)   raw_error = (grid − target_offset) × error_share   error_share: usable_i / Σ(usable_j) — set by power distribution (default 1.0)   usable_i = (SOC_i−Min-SOC_i)/100 × Cap_i   (Cap_i = Cap sensor kWh or 100 if not set)   dynamic_max:      Zone 1 → Hard Limit      Zone 2 → Min(Hard Limit, Max(0, PV − Reserve))   Capacity clamping   correction = P·error + I·integral_candidate   new_power = current + correction   clamp(0, dynamic_max) → Output → inverter   Anti-windup: integral = (final − current − P·error) / I → clamp(±effective_max)   Wait time"]

        INTEGRAL_DECAY["📉 Integral Decay   |Integral| > 10 → Integral × 0.95   otherwise → no update"]

        CALC_SURPLUS --> END_OK([End])
        CALC_TARIFF --> END_OK
        CALC_AC --> END_OK
        CALC_NORMAL --> END_OK
        INTEGRAL_DECAY --> END_OK

    end
    style SG_PI fill:none,stroke:#004085,stroke-width:5,stroke-dasharray:6,color:#000

    %% ── Styles ───────────────────────────────────────────────────────────
    classDef zone0 fill:#fff3cd,stroke:#f0ad4e,color:#000
    classDef zone1 fill:#d4edda,stroke:#28a745,color:#000
    classDef zone2 fill:#d1ecf1,stroke:#17a2b8,color:#000
    classDef zone3 fill:#f8d7da,stroke:#dc3545,color:#000
    classDef night fill:#e2d9f3,stroke:#6f42c1,color:#000
    classDef recovery fill:#fde8d0,stroke:#e07b20,color:#000
    classDef accharge fill:#d0e8ff,stroke:#0066cc,color:#000
    classDef tariff fill:#d4f1d4,stroke:#1a7f1a,color:#000
    classDef tarifflock fill:#c8e6c9,stroke:#388e3c,color:#000
    classDef pi fill:#cce5ff,stroke:#004085,color:#000
    classDef end_node fill:#f8f9fa,stroke:#6c757d,color:#000

    class S0A,S0B,SURPLUS_UPDATE,CALC_SURPLUS,PI_DECISION zone0
    class Z1_START zone1
    class Z2_START zone2
    class Z3_A,Z3_B zone3
    class NIGHT night
    class RECOVERY recovery
    class AC_START,AC_END,AC_END_Z1,AC_END_Z2,AC_GATE,CALC_AC accharge
    class SAFETY_I,SAFETY_I_Z1,SAFETY_I_Z2 recovery
    class TARIFF_START,TARIFF_END,TARIFF_END_Z1,TARIFF_END_Z2,TARIFF_GATE,CALC_TARIFF tariff
    class TARIFF_MID tarifflock
    class CALC_NORMAL,DISCHARGE_SET,INTEGRAL_DECAY,NORMAL_GATE pi
    class END_STOP,END_SKIP,END_OK end_node
````
