# Changelog

All notable changes to the Solakon ONE Zero Export blueprint (EN).
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Fixed
- Surplus forecast forcing additionally tied to `SOC > zone-3 limit` — prevents 0A ↔ C mode flapping when the forcing fights the zone-3 safety stop on a deeply discharged battery (ported from the integration)
- New configuration check: with surplus export enabled, the export threshold must be above the zone-1 limit — otherwise the automation aborts with an error log, mirroring the existing zone-1/zone-3 check
- Surplus forecast forcing tied to `solar > hard_limit` instead of the raw forecast value alone — prevents forcing from staying active all day on a daily/morning forecast sensor. Exit lock (Case 0B) now blocks SOC **and** PV exit symmetrically, but only while PV actually exceeds the hard limit (ported from the integration, fixed there in `3ea1d7f`)

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
