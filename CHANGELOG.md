# Changelog

All notable changes to the Solakon ONE Zero Export blueprint (EN).
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Changed
- Adaptive wait time (introduced in V304/V306) is now optional instead of hardwired — new input `adaptive_wait_time` (default: off). With Zero Export active and a high-latency external grid sensor (e.g. an IR reading head on the meter, see DE-repo Discussion #82), the adaptive exit (actual power ≈ the storage's own setpoint, independent of the grid reading) could shrink the control cycle below the meter's update rate — the configured wait time then had no effect, since it only served as a rarely-reached timeout. Default behavior is the fixed wait (`delay`) again, as before V304

### Fixed
- Case H exit (AC Charging End): comparison `Output ≤ 0 W` (from V309) changed to `|Output| ≤ Tolerance`. The one-sided `≤ 0` was meant to catch isolated one-second zero readings from Modbus noise, but made the guard more sensitive than before: any tiny negative outlier (e.g. `-0.1 W`) now satisfies the condition too, where the previous exact `== 0` comparison would never have matched such an outlier. On setups with frequent AC charging (in particular a pure AC connection without PV) observed as a periodic mode switch 3↔1 every ~15–20 s during active AC charging, with a hard output reset to 0 W on every false trigger (DE-repo Discussion #58, field report 2026-08-27). Fix: symmetric tolerance band around 0 W (`|Output| ≤ tolerance`, the same setting used by the PI convergence check) instead of a one-sided comparison — catches noise in both directions without triggering on every tiny negative outlier during genuine ongoing charging
- `TypeError: unsupported operand type(s) for -: 'str' and 'float'` on `grid_power_float`/`solar_power_float`/`actual_power_float` (DE-repo Issue #78, PR #83, credit `bitfoo1`; catch-up port from the DE blueprint): Home Assistant's template parser deliberately skips the auto-conversion to `float` when the rendered text contains `e`/`E`/`j`/`J`/`inf`/`nan` (guard against misreading scientific notation) — values below 1e-4 render in Python as `"1e-05"` instead of `0.0001`. A 3-phase summing sensor (e.g. Shelly 3EM) produces exactly such residues when the phases cancel almost exactly (`-5.68e-14`), silently turning the affected variable into a string for downstream arithmetic. Fix: `round(3)` on the three producer variables at all 5 locations (top-level `variables` + the block before the PI calculation) — removes the exponent at the source; 0.001 W resolution is irrelevant to the control loop
- The four multi-instancing fields (`max_power_entity`, `error_share_entity`, `ac_error_share_entity`, `total_actual_power_entity`) were placed in the config UI as standalone, non-collapsible top-level fields directly after the "🔒 SAFETY PARAMETERS" section instead of inside a section of their own — visually they appeared attached to the safety section instead of being what they are (optional, relevant only for multi-instance setups). New collapsible section "🔀 MULTI-INSTANCING (Optional)" groups all four fields. Pure UI/structure change, no behavior or formula difference
- Surplus Exit Lock (`surplus_lock_sensor`) compared the raw sensor value against `Factor × Hard Limit` (W) regardless of its unit — unlike Grid/Solar/Actual (already kW→W normalized), no unit detection existed here. A sensor reporting in `kW` (e.g. Solcast `power_now`) would almost never trigger the lock. New variable `surplus_lock_power_float` (kW→W normalized, analog to `grid_power_float`) replaces the raw sensor value in the comparison
- Same bug in Surplus Forecast Entry (`surplus_forecast_sensor`, section 7) — also missing kW→W normalization, even though the sensor is clearly Watt-scale per its own threshold label "(W)" and its `AND` with `Solar > Hard Limit`. New variable `surplus_forecast_power_float` replaces the raw sensor value in the comparison
- `TypeError: cannot use 'dict' as a dict key` when `surplus_forecast_sensor`/`surplus_lock_sensor` is left empty (Issue #81): both optional fields were unguarded when unconfigured — the selector then yields `{}` (a dict) instead of an empty string, and `states({})` triggers an internal dict-key lookup with a dict as the key, aborting the entire variable rendering (automation trace stayed empty, no meaningful log entry). Affected every installation without these optional Solcast forecast fields. Fix: `surplus_forecast_power_float`/`surplus_lock_power_float` are now only computed when the respective sensor entity is a non-empty string, otherwise `0`
- The critical-error check at automation start validated `unknown`/`unavailable` only for the SOC sensor and timeout countdown, not for `grid_power_sensor` and `actual_power_sensor` — the two most frequently read control values. `states(...) | float(0)` silently reads a sensor dropout as a hard zero, indistinguishable from a real reading. At default configuration (`ac_charge_offset` −50 + `ac_charge_hysteresis` 50 = 0), a single external sensor dropout (Modbus timeout/reconnect) on `grid_power_sensor` could trigger Case H (AC charging end) regardless of the true grid value — observed as a periodic mode flip during active AC charging (DE-repo Discussion #58). Fix: both sensors added to the existing critical-error check, automation now skips the cycle on dropout instead of proceeding with a false zero reading

## [V309] – 2026-08-21

### Fixed
- Case H exit (AC charging end): comparison changed from `Output = 0 W` to `Output ≤ 0 W` — more robust against isolated one-second zero readings caused by Modbus noise in power sensors, matching the comparison operator already used at other guard points in the blueprint (e.g. the Zone-0 `solar==0` branch)
- **Catch-up port from the DE blueprint (this repo's variants had drifted out of sync):**
  - Case 0A entry (Zone 0, PV=0 branch) could oscillate at night with a full battery when PV reads 0 persistently (integration Issue #17 — the DE blueprint had the identical, even less protected weakness). Fix: new optional helper `surplus_zero_entry_armed_helper` (input_boolean) as a latch — armed as soon as PV > 0 is measured, disarmed on exit (Case 0B) while PV = 0. The PV=0 entry branch (Case 0A) only fires while the latch is armed, staying locked after a nightly exit until real PV > 0 is measured again. Purely additive, falls back to previous (unprotected) behavior if the helper is not set up
  - Case G entry (AC charging start) in multi-instance operation: condition `(Grid + Output) < −Hysteresis` compared the checking instance's own output instead of the sum across all instances in discharge mode (integration Issue #16 — same pattern inherited in the DE blueprint). Could cause an instance to read a sibling instance's discharge as external grid surplus and then charge back from the grid exactly that amount — battery-to-battery pumping with double conversion losses. Fix: new optional helper `total_actual_power_entity` (populated by `solakon_power_distribution.yaml` via new per-instance actual-power sensors), Case G uses it instead of the own share; Case H (exit) stays unchanged, referencing the instance's own output. Purely additive — existing multi-instancing configs without the new inputs behave unchanged
  - `PI-Controller.yaml`: anti-windup back-calculation (`integral_new`) now rounds to 2 decimal places instead of writing the unrounded division result (surfaced in the context of integration Issue #78 — the actual crash mechanism is unconfirmed, but the rounding reliably prevents `input_number.solakon_integral` from settling in scientific notation, e.g. `5.68e-13`; 0.01 W resolution is irrelevant to the control loop)
  - Case E (Zone 2 entry) now explicitly resets `active_power_number` to 0 W, matching Cases B/C/F (integration Issue #79). Previously the output power could randomly remain within PI tolerance across a zone change — the PI controller would then never trigger again and power would freeze at the old Zone 1 level instead of ramping down to `dynamic_max_power` (Zone 2)

### Added
- Dynamic Zone 1 start threshold (optional, integration Issue #80 — catch-up port from the DE blueprint): new input `soc_fast_limit_entity` (analogous to `offset_1_entity`/`cheap_threshold_entity`). Instead of the fixed `soc_fast_limit` percentage, an `input_number` entity can override the value — e.g. lowered in the evening by a separate automation when the battery didn't reach the default threshold on a cloudy day but tomorrow's PV forecast is high. Prevents unused storage potential from being wasted overnight. Purely additive, falls back to the static value when the entity is empty/unavailable
- Logbook warning on degraded distribution (`solakon_power_distribution.yaml`): if an active instance's SOC sensor becomes unavailable in SOC-weighted mode, it previously fell to weight 0 silently — now also surfaced as a logbook entry. Message-only feature, distribution behavior itself is unchanged
- Surplus exit lock (optional, Issue #11 in the integration repo): new inputs `surplus_lock_enabled`, `surplus_lock_sensor`, `surplus_lock_factor`. While the currently forecast PV power is ≥ lock factor × Hard Limit (default 1.5) AND SOC > zone-3 limit, only the PV exit in Case 0B is blocked — short PV dips (clouds) are ridden out in Zone 0 instead of exiting. Background: exiting with a full battery enters a state where the hardware throttles PV down to consumption and the surplus becomes unmeasurable; since the battery only discharges through the 2 A stability buffer during a cloud, the SOC stays practically pinned at the maximum and re-entry is delayed by minutes. The SOC exit stays unblocked; sensor unavailable → lock inactive (parity with the integration)

### Changed
- Recommendation for the export SOC threshold clarified (README, input description, header): ~5% below the app charge limit, with rationale — entry (PV > consumption + hysteresis) is only measurable while the battery is still charging; at the full-charge point the inverter throttles PV down to consumption and the surplus becomes invisible
- README (sections 5 + 12): corrected claim that the "battery stays untouched during a cloud" — contradicted the already-documented 2 A stability buffer in Zone 0, which discharges the battery continuously at ~1.5–2 A regardless of PV (reporter measurement, integration repo Discussion #18). The argument itself (SOC barely moves during a cloud, exit lock still needed) is unchanged, only the incorrect zero-discharge claim was replaced
- README (section 3): clarified why the 2 A discharge allowance (ceiling, not a fixed setpoint) exists and can't be 0 — with a full battery no current can flow in, and the Solakon can only regulate PV while battery current flows; without this current flow the device shuts down completely, the 2 A is deliberately kept low to minimize battery load

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
