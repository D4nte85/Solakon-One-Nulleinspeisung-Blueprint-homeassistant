# Changelog

Alle nennenswerten Änderungen am Solakon-ONE-Nulleinspeisung-Blueprint (DE).
Format angelehnt an [Keep a Changelog](https://keepachangelog.com/de/1.1.0/).

## [Unreleased]

### Behoben
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
