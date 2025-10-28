```mermaid
graph TD
    %% ----------------------------------------------------
    %% Mermaid Flowchart - Solakon ONE SOC Logic (Final V4 - English)
    %% ----------------------------------------------------

    %% 1. START and Validation
    A0((<span style='font-size:120%'>START</span>)) --> A1[/<span style='font-size:120%'>Load Vars</span>/]
    A1 --> A2{<span style='font-size:120%'>VALIDATION:<br>Limits & Entities OK?</span>}
    A2 -- YES (Error) --> A3[<span style='font-size:120%'>Log Error</span>]
    A3 --> A_END((<span style='font-size:120%'>STOP</span>))
    A2 -- NO (Valid) --> B0

    %% 2. Mode Switching and Cycle Control
    B0{<span style='font-size:120%'>CHOOSE:<br>SOC & Cycle Status</span>}

    %% CASE A: Zone 1 - Start
    B0 -- CASE A: SOC > Start AND Cycle OFF --> B1_A[<span style='font-size:120%'>Set 'cycle_active' = 'on'</span>]
    B1_A --> B2_A[<span style='font-size:120%'>Mode Reset Pulse Sequence</span>]
    B2_A --> B3_A[<span style='font-size:120%'>Set 'solakon_mode' = 'INV Discharge PV Priority'</span>]
    B3_A --> C0

    %% CASE B: Zone 3 - Stop
    B0 -- CASE B: SOC <= Stop AND Cycle ON --> B1_B[<span style='font-size:120%'>Set 'cycle_active' = 'off'</span>]
    B1_B --> B2_B[<span style='font-size:120%'>Mode Reset Disabled</span>]
    B2_B --> B3_B[<span style='font-size:120%'>Set 'solakon_mode' = 'Disabled'</span>]
    B3_B --> B4_B[<span style='font-size:120%'>Set Power Limit = 0 W</span>]
    B4_B --> C0

    %% CASE C: Zone 3 - Safety
    B0 -- CASE C: SOC < Stop AND Cycle OFF AND Mode != Disabled --> B1_C[<span style='font-size:120%'>Mode Reset Pulse Sequence</span>]
    B1_C --> B2_C[<span style='font-size:120%'>Set 'solakon_mode' = 'Disabled'</span>]
    B2_C --> B3_C[<span style='font-size:120%'>Set Power Limit = 0 W</span>]
    B3_C --> C0

    %% CASE D: Zone 2 - Start/Stop Logic
    B0 -- CASE D: Stop < SOC <= Start AND Cycle OFF --> B_D0{<span style='font-size:120%'>SUB-CHOOSE: PV Condition</span>}

    B_D0 -- D1: PV > PV Reserve (START) --> B1_D1[<span style='font-size:120%'>Mode Reset Pulse Sequence</span>]
    B1_D1 --> B2_D1[<span style='font-size:120%'>Set 'solakon_mode' = 'INV Discharge PV Priority'</span>]
    B2_D1 --> B3_D1[<span style='font-size:120%'>Set Power Limit = 0 W</span>]
    B3_D1 --> C0

    B_D0 -- D2: PV <= PV Reserve (STOPP) --> B1_D2[<span style='font-size:120%'>Mode Reset Pulse Sequence</span>]
    B1_D2 --> B2_D2[<span style='font-size:120%'>Set 'solakon_mode' = 'Disabled'</span>]
    B2_D2 --> B3_D2[<span style='font-size:120%'>Set Power Limit = 0 W</span>]
    B3_D2 --> C0

    %% 3. Active P-Controller and Timeout Reset
    C0{<span style='font-size:120%'>Mode = 'INV Discharge PV Priority'?</span>}
    C0 -- NO --> A_END
    C0 -- YES --> C1{<span style='font-size:120%'>Timeout<br>< 120s?</span>}
    C1 -- YES --> C2[<span style='font-size:120%'>Mode Reset Pulse Sequence</span>]
    C2 --> C3
    C1 -- NO --> C3

    C3{<span style='font-size:120%'>CHOOSE: P-Controller Zone</span>}

    %% B1: Zone 2: Battery Gentle
    C3 -- B1: Zone 2 Battery Gentle --> C4_B1[/<span style='font-size:120%'>Define Offset/Tolerance</span>/]
    C4_B1 --> C5_B1{<span style='font-size:120%'>Grid Power<br>outside Tolerance?</span>}
    C5_B1 -- YES --> C6_B1[<span style='font-size:120%'>Set Power Limit Zone 2</span>]
    C6_B1 --> A_END
    C5_B1 -- NO --> A_END

    %% B2: Zone 1: Aggressive
    C3 -- B2: Zone 1 Aggressive --> C4_B2[/<span style='font-size:120%'>Define Offset 0 W<br>Tolerance Window</span>/]
    C4_B2 --> C5_B2{<span style='font-size:120%'>Grid Power<br>outside Tolerance?</span>}
    C5_B2 -- YES --> C6_B2[<span style='font-size:120%'>Set Power Limit Zone 1</span>]
    C6_B2 --> A_END
    C5_B2 -- NO --> A_END

    %% Final Stop
    A_END((<span style='font-size:120%'>STOP</span>))
