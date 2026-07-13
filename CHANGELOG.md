# Changelog

Alle nennenswerten Änderungen am Solakon-ONE-Nulleinspeisung-Blueprint (DE).
Format angelehnt an [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).

## [Unreleased]

### Hinzugefügt
- Surplus-Austritts-Sperre (optional, Issue #11 im Integration-Repo): Neue Inputs `surplus_lock_enabled`, `surplus_lock_sensor`, `surplus_lock_factor`. Solange die aktuell prognostizierte PV-Leistung ≥ Sperr-Faktor × Hard Limit (Standard 1,5) UND SOC > Zone-3-Schwelle, ist in Fall 0B nur der PV-Austritt gesperrt — kurze PV-Einbrüche (Wolken) werden in Zone 0 durchgeritten statt auszutreten. Hintergrund: Der Austritt bei vollem Akku führt in einen Zustand, in dem die Hardware die PV auf den Eigenbedarf drosselt und der Überschuss nicht mehr messbar ist; da die Batterie während einer Wolke nicht entladen wird, bleibt der SOC am Maximum gepinnt und der Wiedereintritt verzögert sich um Minuten. Der SOC-Austritt bleibt ungesperrt, Sensor nicht verfügbar → Sperre inaktiv (Parität mit der Integration)

### Geändert
- Empfehlung für die Export-SOC-Schwelle präzisiert (README, Input-Beschreibung, Header): ~5 % unter der App-Ladeobergrenze, mit Begründung — der Eintritt (PV > Verbrauch + Hysterese) ist nur messbar solange der Akku noch lädt; am Vollladepunkt drosselt der WR die PV auf den Eigenbedarf und der Überschuss wird unsichtbar

## [V308] – 2026-07-08

### Behoben
- AC-Lade-Fehler-Anteil von der Nulleinspeisungs-Verteilung entkoppelt: Leistungsverteilungs-Blueprint
  berechnet jetzt zwei getrennte Pools — Pool 1 (Nulleinspeisung, nur Instanzen in Modus '1', unverändert)
  und der neue Pool 2 (`_compute_ac_distribution`-Äquivalent: `aw1`–`aw4`, nur unter gleichzeitig
  AC-ladenden Instanzen). Vorher fror der AC-Lade-PI einer ladenden Instanz auf 0 W ein, weil sie
  im gemeinsamen Pool als "inaktiv" (Modus ≠ '1') galt und `error_share = 0` bekam. Neue optionale
  Inputs pro Instanz: AC-Lade-Zustand-Helfer (`inst{N}_ac_charge_helper`) und AC-Lade Fehler-Anteil
  Helfer (`inst{N}_ac_share_number`); neuer Input `ac_error_share_entity` im Hauptblueprint, in
  Zweig B (AC-Laden-PI) statt des bisherigen gemeinsamen `error_share_entity` verwendet (aus der
  Integration v2.1.2 portiert)
- Fall TM (Discharge-Lock) verschont aktiven Überschuss: neuer Guard `NICHT Surplus-Bool = on` — vorher stoppte die mittlere Preiszone die Zone-0-Einspeisung bei vollem Speicher (PV wurde abgeregelt statt eingespeist), entgegen der dokumentierten Priorität Zone 0 > Tarif; der damit tote Surplus-Reset im TM-Body wurde entfernt (Parität mit der Integration)
- Fall F (Nachtabschaltung) verschont aktiven Überschuss: neuer Guard `NICHT Surplus-Bool = on` plus Surplus-Reset als Absicherung — vorher konnte F bei tarifgesperrtem Fall A den Modus deaktivieren und das Surplus-Bool hängen lassen
- Surplus-Forecast-Forcierung zusätzlich an `SOC > Zone-3-Schwelle` gekoppelt — verhindert Modus-Flattern 0A ↔ C, wenn die Forcierung bei tiefentladener Batterie gegen den Zone-3-Sicherheitsstopp ankämpft (aus der Integration portiert)
- Neue Konfigurationsprüfung: bei aktivierter Überschuss-Einspeisung muss die Export-Schwelle über der Zone-1-Schwelle liegen — sonst Abbruch mit Fehler-Log, analog zur bestehenden Zone1-/Zone3-Prüfung
- Surplus-Forecast-Forcierung an `solar > hard_limit` gekoppelt statt am rohen Vorhersagewert allein — verhindert, dass die Forcierung bei einer Tages-/Morgen-Prognose den ganzen Tag über aktiv bleibt. Exit-Sperre (Fall 0B) blockiert jetzt SOC- **und** PV-Austritt symmetrisch, aber nur solange PV real das Hard Limit übersteigt (aus der Integration portiert, dort behoben in `3ea1d7f`)

## [V307] – 2026-07-03

### Behoben
- Timer-Toggle vor jedem Moduswechsel zu `'0'` erzwungen — AC-Laden-Exit (Fall H/I), Fall HT und alle verbleibenden Falls (identische Lücke wie Integration-Issue #10)
- Zone-2-PI-Bugs aus der Integration portiert: Overshoot am Hard-Limit, sinkende PV wird nachgeführt
- PI-Anti-Windup via Back-Calculation; Integral-Helper-Bereich auf ±1200 erhöht
- Forecast blockiert nur noch den SOC-Austritt aus Surplus, der PV-Austritt greift immer
- Output auf WR-Entity-Max gedeckelt, wenn Gesamt-Limit > Einzel-WR-Max (#74)
- Leistungsverteilung: Interval-Guard nutzt `i1_share` statt `i1_limit`

### Geändert
- Leistungsverteilung: PV-Gewichtung entfernt — reine SOC-Gewichtung, `pv_influence`-Parameter entfernt
- Doku: Entity-ID-Hinweis am Kapazitätssensor, veraltete SOC/PV-Slider-Referenz entfernt, Hinweis zur offiziell deaktivierten Max-Entladestrom-Entität, source_url/Import-Buttons auf main-Branch

## [V306] – 2026-05-22
- Multi-Instancing in main gemerged: Leistungsverteilungs-Blueprint, instanzbezogene Limits/Shares
- kW→W-Guards, adaptive Wartezeit (Self-Adjusting Wait), PV-Forecast-Unterdrückung, Surplus-Forecast-Eintritt

## [V304] – 2026-05 und früher
Ältere Versionen sind nicht rückwirkend dokumentiert.
