# Changelog

All notable changes to the Solakon ONE Zero Export blueprint (EN).
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [V307] – 2026-07-03

### Fixed
- Timer toggle forced on every mode change to `'0'` — AC charging exit (Case H/I), Case HT and all remaining cases (same gap as integration issue #10)

### Added
- Power Distribution blueprint for multi-instancing

### Changed
- Power distribution: PV weighting removed — pure SOC-weighted
- Docs: full multi-instancing README section with corrected formula, entity ID hint on capacity sensor, outdated SOC/PV slider reference removed

## [V306] – 2026-06-29
- Full feature parity with the German main branch (V302 → V306): multi-instancing, kW→W guards, self-adjusting wait, PV forecast suppression, surplus forecast entry, plus all bug fixes up to 2026-06-29

## [V302] – and earlier
Older versions are not documented retroactively.
