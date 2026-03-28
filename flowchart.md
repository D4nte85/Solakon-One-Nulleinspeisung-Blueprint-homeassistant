````mermaid
flowchart TD
    START([⚡ Trigger Grid / PV / SOC / Mode]) --> VAL

    VAL{{"🛡️ Validation SOC Limits & Entities"}}
    VAL -- Error --> STOP([🛑 Stop + Log Entry])
    VAL -- OK --> SURPLUS_UPDATE

    %% ── Zone 0: Status Update (before main logic) ────────────────────────
    SURPLUS_UPDATE{{"☀️ Zone 0 Status Update   (Cases 0A / 0B)"}}

    SURPLUS_UPDATE -- "CASE 0A   Surplus-Bool = off   AND SOC ≥ Export Threshold   AND (PV > Output + Grid + PV-Hysteresis   OR PV = 0)" --> S0A
    S0A["☀️ Activate Zone 0   Surplus-Bool → on"]

    SURPLUS_UPDATE -- "CASE 0B   Surplus-Bool = on   AND (SOC < Export Threshold − SOC-Hysteresis   OR PV ≤ Output + Grid − PV-Hysteresis)" --> S0B
    S0B["☁️ Deactivate Zone 0   Surplus-Bool → off   Integral = 0"]

    SURPLUS_UPDATE -- "No Surplus Update" --> ZONE_CHECK
    S0A --> ZONE_CHECK
    S0B --> ZONE_CHECK

    %% ── Main Logic: Cases A – F (+ G, H, I) ─────────────────────────────
    ZONE_CHECK{{"SOC & Cycle Status?   (Cases A – F, G, H, I)"}}

    %% ── Case A: Zone 1 ───────────────────────────────────────────────────
    ZONE_CHECK -- "CASE A   NOT AC-Charge-Bool = on   AND SOC > Zone 1 Threshold AND Cycle = off" --> Z1_START
    Z1_START["🔋 Activate Zone 1   Cycle = on   Integral = 0   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]

    %% ── Case B: Zone 3 (Cycle on) ────────────────────────────────────────
    ZONE_CHECK -- "CASE B   NOT AC-Charge-Bool = on   AND SOC < Zone 3 Threshold AND Cycle = on" --> Z3_A
    Z3_A["🛑 Activate Zone 3   Cycle = off   Integral = 0   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Mode → '0' (Disabled)   Output → 0 W"]

    %% ── Case C: Zone 3 (Guard) ───────────────────────────────────────────
    ZONE_CHECK -- "CASE C   NOT AC-Charge-Bool = on   AND SOC < Zone 3 Threshold AND Cycle = off AND Mode ≠ '0'" --> Z3_B
    Z3_B["🛑 Zone 3 Guard   Surplus-Bool → off (only if active)   AC-Charge-Bool → off (only if active)   Mode → '0' (Disabled)   Output → 0 W"]

    %% ── Case D: Recovery ─────────────────────────────────────────────────
    ZONE_CHECK -- "CASE D   Cycle = on   AND Mode ∉ {'1','3'} ← '3' explicitly excluded!   AND SOC > Zone 3 Threshold" --> RECOVERY
    RECOVERY["🔄 Recovery — Mode Reactivation   Timer-Toggle (3598↔3599)   AC-Charge-Bool = on → Mode '3'   otherwise → Mode '1'   (no integral reset, no zone change)"]

    %% ── Case G: AC Charging Entry ────────────────────────────────────────
    ZONE_CHECK -- "CASE G   AC Charging enabled   AND SOC < Charge Target   AND Mode ≠ '3' ← Guard!   AND (Grid + Output) < −Hysteresis" --> AC_START
    AC_START["⚡ Activate AC Charging   AC-Charge-Bool → on   Timer-Toggle (3598↔3599)   Output → 0 W (PI starts clean)   Mode → '3' (INV Charge PV Priority)"]

    %% ── Case H: End AC Charging ──────────────────────────────────────────
    ZONE_CHECK -- "CASE H   Mode = '3'   AND (SOC ≥ Charge Target OR (Grid ≥ AC-Offset + Hysteresis AND Output = 0 W))" --> AC_END
    AC_END{{"Integral = 0   AC-Charge-Bool → off   Current Zone?"}}
    AC_END -- "Cycle = on (Zone 1)" --> AC_END_Z1
    AC_END -- "Cycle = off (Zone 2)" --> AC_END_Z2
    AC_END_Z1["⚡ End AC Charging (Zone 1)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]
    AC_END_Z2["⚡ End AC Charging (Zone 2)   Mode → '0' (Disabled)   Output → 0 W"]

    %% ── Case I: Safety — Mode '3' without active AC charging session ─────
    ZONE_CHECK -- "CASE I   Mode = '3'   AND AC Charging not active / helper off" --> SAFETY_I
    SAFETY_I{{"Integral = 0   Current Zone?"}}
    SAFETY_I -- "Cycle = on (Zone 1)" --> SAFETY_I_Z1
    SAFETY_I -- "Cycle = off (Zone 2)" --> SAFETY_I_Z2
    SAFETY_I_Z1["⚠️ Safety (Zone 1)   Output → 0 W   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]
    SAFETY_I_Z2["⚠️ Safety (Zone 2)   Mode → '0' (Disabled)   Output → 0 W"]

    %% ── Case E: Zone 2 ───────────────────────────────────────────────────
    ZONE_CHECK -- "CASE E   NOT AC-Charge-Bool = on   AND Zone 3 < SOC ≤ Zone 1 AND Cycle = off AND Mode = '0' AND NOT Night" --> Z2_START
    Z2_START["🔋 Activate Zone 2   Integral = 0   Timer-Toggle (3598↔3599)   Mode → '1' (INV Discharge PV Priority)"]

    %% ── Case F: Night Shutdown ───────────────────────────────────────────
    ZONE_CHECK -- "CASE F   NOT AC-Charge-Bool = on   AND Night Shutdown active AND PV < PV Charge Reserve AND Cycle = off AND Mode active" --> NIGHT
    NIGHT["🌙 Night Shutdown   Integral = 0   Mode → '0' (Disabled)   Output → 0 W"]

    %% ── No Zone Change ───────────────────────────────────────────────────
    ZONE_CHECK -- "No Zone Change" --> PI_GATE

    Z1_START --> PI_GATE
    Z2_START --> PI_GATE
    RECOVERY --> PI_GATE
    AC_START --> END_STOP([End])
    AC_END_Z1 --> END_STOP
    AC_END_Z2 --> END_STOP
    SAFETY_I_Z1 --> END_STOP
    SAFETY_I_Z2 --> END_STOP
    Z3_A --> END_STOP
    Z3_B --> END_STOP
    NIGHT --> END_STOP

    %% ── PI Controller Gate ───────────────────────────────────────────────
    PI_GATE{{"Mode ∈ {'1','3'}?   AND (Cycle = on OR Night Lock inactive OR PV ≥ Reserve)"}}
    PI_GATE -- No --> END_SKIP([End — no output])
    PI_GATE -- Yes --> DISCHARGE_SET

    %% ── Step 1: Discharge Current ────────────────────────────────────────
    DISCHARGE_SET{{"Set discharge current?"}}
    DISCHARGE_SET -- "Surplus active → 2 A   (only if different)" --> TIMEOUT_CHECK
    DISCHARGE_SET -- "Cycle = on AND Mode ≠ '3' AND Surplus NOT active → Max value   (only if different)" --> TIMEOUT_CHECK
    DISCHARGE_SET -- "Cycle = off OR Mode = '3' AND Surplus NOT active → 0 A   (only if different)" --> TIMEOUT_CHECK

    %% ── Step 2: Timeout Reset ────────────────────────────────────────────
    TIMEOUT_CHECK{{"Countdown < 120s?"}}
    TIMEOUT_CHECK -- Yes --> TIMEOUT_RESET["⏱️ Timer-Toggle (3598↔3599)"]
    TIMEOUT_CHECK -- No --> PI_DECISION
    TIMEOUT_RESET --> PI_DECISION

    %% ── Step 3: Output & Integral ────────────────────────────────────────
    PI_DECISION{{"Surplus-Bool = on?   → Zone 0 active"}}
    PI_DECISION -- Yes --> CALC_SURPLUS
    PI_DECISION -- No --> AC_GATE

    %% ── Zone 0 Path ──────────────────────────────────────────────────────
    CALC_SURPLUS["☀️ Zone 0 — Surplus Export   Output → Hard Limit   Freeze integral (integral_old unchanged)   Wait time"]

    %% ── AC Charging Path ─────────────────────────────────────────────────
    AC_GATE{{"AC-Charge-Bool = on?   AND |Grid error| > Tolerance?"}}
    AC_GATE -- Yes --> CALC_AC
    AC_GATE -- No --> NORMAL_GATE

    CALC_AC["⚡ AC Charging — PI Script (ac_charge_mode=true)   raw_error = ac_charge_offset − grid ← own offset!   max_power = ac_charge_power_limit   Capacity clamping   Integral += error → clamp(−1000, 1000)   Correction = P·error + I·integral   (separate ac_charge_p/i_factor)   new_power = current + Correction   clamp(0, charge limit) → Output → Inverter   Save integral   Wait time   (at_max / at_min guards NOT applied)"]

    %% ── Normal PI Path ───────────────────────────────────────────────────
    NORMAL_GATE{{"Error > Tolerance?   AND no At-Max / At-Min limit?"}}
    NORMAL_GATE -- Yes --> CALC_NORMAL
    NORMAL_GATE -- No --> INTEGRAL_DECAY

    CALC_NORMAL["🧠 PI Script (ac_charge_mode=false)   raw_error = grid − target_offset   dynamic_max:      Zone 1 → Hard Limit      Zone 2 → Max(0, PV − Reserve)   Capacity clamping   Integral += error → clamp(−1000, 1000)   Correction = P·error + I·integral   new_power = current + Correction   clamp(0, dynamic_max) → Output → Inverter   Save integral   Wait time"]

    INTEGRAL_DECAY["📉 Integral Decay   |Integral| > 10 → Integral × 0.95   otherwise → no update"]

    CALC_SURPLUS --> END_OK([End])
    CALC_AC --> END_OK
    CALC_NORMAL --> END_OK
    INTEGRAL_DECAY --> END_OK

    %% ── Styles ───────────────────────────────────────────────────────────
    classDef zone0 fill:#fff3cd,stroke:#f0ad4e,color:#000
    classDef zone1 fill:#d4edda,stroke:#28a745,color:#000
    classDef zone2 fill:#d1ecf1,stroke:#17a2b8,color:#000
    classDef zone3 fill:#f8d7da,stroke:#dc3545,color:#000
    classDef night fill:#e2d9f3,stroke:#6f42c1,color:#000
    classDef recovery fill:#fde8d0,stroke:#e07b20,color:#000
    classDef accharge fill:#d0e8ff,stroke:#0066cc,color:#000
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
    class CALC_NORMAL,DISCHARGE_SET,INTEGRAL_DECAY,NORMAL_GATE pi
    class END_STOP,END_SKIP,END_OK end_node
````
