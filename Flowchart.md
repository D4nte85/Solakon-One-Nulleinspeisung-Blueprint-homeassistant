```mermaid
graph TD
    %% -----------------------------------------------------------------
    %% Mermaid Flowchart - Solakon ONE SOC-Logik (V166)
    %% -----------------------------------------------------------------

    %% 1. START und Validierung
    A0((START)) --> A1[/Lade Vars/]
    A1 --> A2{VALIDIERUNG:<br>Limits & Entitäten OK?}
    A2 -- JA (Fehler) --> A3[Error Log]
    A3 --> A_END((STOP))
    A2 -- NEIN (Gueltig) --> B0

    %% 2. Modus-Umschaltung und Zyklus-Steuerung
    B0{CHOOSE:<br>SOC & Zyklus}

    %% FALL A: Zone 1 - Aggressiv Start
    B0 -- FALL A: SOC > Start UND Zyklus OFF --> B1_A[Setze 'cycle_active' = 'on']
    B1_A --> B2_A[Setze Discharge Current = Max]
    B2_A --> B3_A[Modus-Wechsel zu 'INV Discharge PV Priority']
    B3_A --> C0

    %% FALL B: Zone 3 - Stopp
    B0 -- FALL B: SOC <= Stop UND Zyklus ON --> B1_B[Setze 'cycle_active' = 'off']
    B1_B --> B2_B[Modus-Wechsel zu 'Disabled' und Limit auf 0 W]
    B2_B --> C0

    %% FALL C: Zone 3 - Sicherung
    B0 -- FALL C: SOC < Stop UND Zyklus OFF --> B1_C[Modus-Wechsel zu 'Disabled' und Limit auf 0 W]
    B1_C --> C0

    %% FALL D: Zone 2 - Schonend Start (P-Regler aktiv, 0A Entladung)
    B0 -- FALL D: Stop < SOC <= Start UND Zyklus OFF --> B1_D[Setze Discharge Current = 0 A]
    B1_D --> B2_D[Modus-Wechsel zu 'INV Discharge PV Priority']
    B2_D --> C0

    %% 3. Aktive P-Regelung und Timeout-Reset
    C0{Modus = 'INV Discharge PV Priority'?}
    C0 -- NEIN --> A_END
    C0 -- JA --> C1{Timeout < 120s?}
    C1 -- JA --> C2[Führe Modus-Reset Pulsfolge aus]
    C2 --> C3
    C1 -- NEIN --> C3

    C3{CHOOSE: P-Regler Zone}

    %% B1: Zone 2: Batterieschonend
    C3 -- B1: Zone 2 Schonend UND Zyklus OFF --> C4_B1[/Definiere Offset/Toleranz/]
    C4_B1 --> C5_B1{Netzleistung<br>ausserhalb Toleranz?}
    C5_B1 -- JA --> C6_B1[Power Limit Z2: Setze Minimum von P-Regler oder PV abzl. Reserve]
    C6_B1 --> A_END
    C5_B1 -- NEIN --> A_END

    %% B2: Zone 1: Aggressiv
    C3 -- B2: Zone 1 Aggressiv ODER Zyklus ON --> C4_B2[/Definiere Offset 0 W/Toleranzfenster/]
    C4_B2 --> C5_B2{Netzleistung<br>ausserhalb Toleranz?}
    C5_B2 -- JA --> C6_B2[Power Limit Z1: Setze Minimum von P-Regler oder Hard Limit]
    C6_B2 --> A_END
    C5_B2 -- NEIN --> A_END

    %% Finaler Stop
    A_END((STOP))
