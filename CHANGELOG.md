# Changelog

All notable changes to the Solakon ONE Zero Export blueprint (EN).
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added
- Surplus exit lock (optional, Issue #11 in the integration repo): new inputs `surplus_lock_enabled`, `surplus_lock_sensor`, `surplus_lock_factor`. While the currently forecast PV power is ≥ lock factor × Hard Limit (default 1.5) AND SOC > zone-3 limit, only the PV exit in Case 0B is blocked — short PV dips (clouds) are ridden out in Zone 0 instead of exiting. Background: exiting with a full battery enters a state where the hardware throttles PV down to consumption and the surplus becomes unmeasurable; since the battery is not discharged during a cloud, the SOC stays pinned at the maximum and re-entry is delayed by minutes. The SOC exit stays unblocked; sensor unavailable → lock inactive (parity with the integration)

### Changed
- Recommendation for the export SOC threshold clarified (README, input description, header): ~5% below the app charge limit, with rationale — entry (PV > consumption + hysteresis) is only measurable while the battery is still charging; at the full-charge point the inverter throttles PV down to consumption and the surplus becomes invisible

## [V308] – 2026-07-08

### Fixed
- AC-charging error share decoupled from the Zero-Export distribution: the power distribution
  blueprint now computes two separate pools — Pool 1 (Zero-Export, only instances in mode '1',
  unchanged) and new Pool 2 (`aw1`–`aw4`, only among instances currently AC charging at the same
  time). Previously, an AC-charging instance's PI froze at 0 W because it was counted as
  "inactive" (mode ≠ '1') in the shared pool and got `error_share = 0`. New optional inputs per
  instance: AC charge state helper (`inst{N}_ac_charge_helper`) and AC charge error share helper
  (`inst{N}_ac_share_number`); new input `ac_error_share_entity` in the main blueprint, used in
  Branch B (AC-charging PI) instead of the previously shared `error_share_entity` (ported from
  integration v2.1.2)
- Case TM (discharge lock) now spares active surplus: new guard `NOT Surplus-Bool = on` — previously the mid-price zone stopped Zone 0 export on a full battery (PV was curtailed instead of exported), contradicting the documented priority Zone 0 > tariff; the now-dead surplus reset inside the TM body was removed (parity with the integration)
- Case F (night shutdown) now spares active surplus: new guard `NOT Surplus-Bool = on` plus a surplus reset as a safety net — previously F could disable the mode while Case A was tariff-locked, leaving the surplus bool stuck
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
