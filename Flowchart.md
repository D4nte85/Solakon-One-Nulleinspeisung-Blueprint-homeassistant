```mermaid
graph TD
    %% ----------------------------------------------------
    %% Mermaid Flowchart - Solakon ONE SOC-Logik (Final V4)
    %% ----------------------------------------------------

    %% 1. START und Validierung
    A0((<span style='font-size:120%'>START</span>)) --> A1[/<span style='font-size:120%'>Lade Vars</span>/]
    A1 --> A2{<span style='font-size:120%'>VALIDIERUNG:<br>Limits & Entitäten OK?</span>}
    A2 -- JA (Fehler) --> A3[<span style='font-size:120%'>Error Log</span>]
    A3 --> A_END((<span style='font-size:120%'>STOP</span>))
    A2 -- NEIN (Gueltig) --> B0

    %% 2. Modus-Umschaltung und Zyklus-Steuerung
    B0{<span style='font-size:120%'>CHOOSE:<br>SOC & Zyklus</span>}

    %% FALL A: Zone 1 - Start
    B0 -- FALL A: SOC > Start UND Zyklus OFF --> B1_A[<span style='font-size:120%'>Setze 'cycle_active' = 'on'</span>]
    B1_A --> B2_A[<span style='font-size:120%'>Modus-Reset Pulsfolge</span>]
    B2_A --> B3_A[<span style='font-size:120%'>Setze 'solakon_mode' = 'INV Discharge PV Priority'</span>]
    B3_A --> C0

    %% FALL B: Zone 3 - Stopp
    B0 -- FALL B: SOC <= Stop UND Zyklus ON --> B1_B[<span style='font-size:120%'>Setze 'cycle_active' = 'off'</span>]
    B1_B --> B2_B[<span style='font-size:120%'>Modus-Reset Disabled</span>]
    B2_B --> B3_B[<span style='font-size:120%'>Setze 'solakon_mode' = 'Disabled'</span>]
    B3_B --> B4_B[<span style='font-size:120%'>Setze Power Limit = 0 W</span>]
    B4_B --> C0

    %% FALL C: Zone 3 - Sicherung
    B0 -- FALL C: SOC < Stop UND Zyklus OFF UND Modus != Disabled --> B1_C[<span style='font-size:120%'>Modus-Reset Pulsfolge</span>]
    B1_C --> B2_C[<span style='font-size:120%'>Setze 'solakon_mode' = 'Disabled'</span>]
    B2_C --> B3_C[<span style='font-size:120%'>Setze Power Limit = 0 W</span>]
    B3_C --> C0

    %% FALL D: Zone 2 - Start/Stopp Logik
    B0 -- FALL D: Stop < SOC <= Start UND Zyklus OFF --> B_D0{<span style='font-size:120%'>UNTER-CHOOSE: PV-Bedingung</span>}

    B_D0 -- D1: PV > PV-Reserve (START) --> B1_D1[<span style='font-size:120%'>Modus-Reset Pulsfolge</span>]
    B1_D1 --> B2_D1[<span style='font-size:120%'>Setze 'solakon_mode' = 'INV Discharge PV Priority'</span>]
    B2_D1 --> B3_D1[<span style='font-size:120%'>Setze Power Limit = 0 W</span>]
    B3_D1 --> C0

    B_D0 -- D2: PV <= PV-Reserve (STOPP) --> B1_D2[<span style='font-size:120%'>Modus-Reset Pulsfolge</span>]
    B1_D2 --> B2_D2[<span style='font-size:120%'>Setze 'solakon_mode' = 'Disabled'</span>]
    B2_D2 --> B3_D2[<span style='font-size:120%'>Setze Power Limit = 0 W</span>]
    B3_D2 --> C0

    %% 3. Aktive P-Regelung und Timeout-Reset
    C0{<span style='font-size:120%'>Modus = 'INV Discharge PV Priority'?</span>}
    C0 -- NEIN --> A_END
    C0 -- JA --> C1{<span style='font-size:120%'>Timeout<br>< 120s?</span>}
    C1 -- JA --> C2[<span style='font-size:120%'>Modus-Reset Pulsfolge</span>]
    C2 --> C3
    C1 -- NEIN --> C3

    C3{<span style='font-size:120%'>CHOOSE: P-Regler Zone</span>}

    %% B1: Zone 2: Batterieschonend
    C3 -- B1: Zone 2 Batterieschonend --> C4_B1[/<span style='font-size:120%'>Definiere Offset/Toleranz</span>/]
    C4_B1 --> C5_B1{<span style='font-size:120%'>Netzleistung<br>ausserhalb Toleranz?</span>}
    C5_B1 -- JA --> C6_B1[<span style='font-size:120%'>Setze Power Limit Zone 2</span>]
    C6_B1 --> A_END
    C5_B1 -- NEIN --> A_END

    %% B2: Zone 1: Aggressiv
    C3 -- B2: Zone 1 Aggressiv --> C4_B2[/<span style='font-size:120%'>Definiere Offset 0 W<br>Toleranzfenster</span>/]
    C4_B2 --> C5_B2{<span style='font-size:120%'>Netzleistung<br>ausserhalb Toleranz?</span>}
    C5_B2 -- JA --> C6_B2[<span style='font-size:120%'>Setze Power Limit Zone 1</span>]
    C6_B2 --> A_END
    C5_B2 -- NEIN --> A_END

    %% Finaler Stop
    A_END((<span style='font-size:120%'>STOP</span>))
