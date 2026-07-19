# Grind Advisor — Changelog

All database and history-file access is strictly read-only in every version.

## v3.0.0 — 2026-07-10

- New recommendation engine: a single built-in **Regression Forecast**
  method replaces the old Conservative / Normal / Dynamic Barista /
  Aggressive New Bean modes (there is no mode setting anymore). It uses
  every shot of the current bag: one shot nudges the grind, two shots
  measure your bean's seconds-per-grind, and three or more fit a
  recency-weighted line through your shots and solve it for the grind that
  hits your target time. It normalizes out dose/yield weighing error when
  your shots carry scale data, excludes obvious outliers, and learns your
  grinder's direction from the data — so it converges on the right grind as
  shots accumulate. The Why? button and Calculation Details page show the
  learned slope, model fit (R²), shot count, and every intermediate.
- Upgrading from 2.x discards the old mode / seconds-per-step / cap settings
  gracefully. Recommendation math verified against the reference spreadsheet.

## v2.3.0 — 2026-07-10

- New: **Dose / Yield Source** (Advanced) — choose whether recommendations
  report your set dose/yield, the measured (actual) values, or Auto
  (measured when plausible, else set — the default). The popup, reason line,
  and Diagnostics always state which source was used. Display-only; it does
  not change the recommended grind.

## v2.2.0 — 2026-07-10

- New: the Calculation Details page (under Advanced) now shows a trace of
  your latest 5 shots, listing every value behind each recommendation —
  shot time, target, error, dose, yield, bag shot number, seconds-per-grind-
  step and where it came from (same-bag calibration, recipe match, or
  default estimate), the adjustment cap, the raw grind before rounding, the
  final recommendation, and the reason. Useful for understanding why a
  recommendation came out the way it did. Display-only; the recommendation
  math is unchanged.

## v2.1.1 — 2026-07-10

- Fixed: shots deleted from the history folder (e.g. by a shot-editor
  plugin's soft delete) kept appearing in Grind Advisor's history,
  calibration, and confidence score. Grind Advisor now honors the shot
  database's `removed` flag and also skips rows whose history file no
  longer exists, so deleted shots disappear immediately — no resync
  needed. Both checks auto-disable on databases that don't have the
  relevant columns; the Diagnostics page shows their status.

## v2.1.0 — 2026-07-10

- New: Calibration Accuracy — a display-only confidence gauge (0–100) for
  the current recommendation, shown on the settings page (10-segment bar)
  and as a line in the after-shot popup, explained on the Help page. It
  weighs how many valid same-bag/recipe shots feed the calibration against
  how consistently recent shots hit your target time. Fewer than 2 relevant
  shots shows "Not enough data"; a new bag resets it. Recommendation math
  is unchanged — the score never feeds back.

## v2.0.2 — 2026-07-07

- Fixed the history page's Prev/Next buttons showing escape text instead of
  the ◀ / ▶ glyphs (also affected the grind arrow on history cards).
- Fixed Done on the main settings page returning to Advanced after visiting
  a sub-page instead of leaving the plugin.

## v2.0.1

- Regrouped the main settings and Advanced pages into white rounded section
  cards with titles (matching the stock app's App-tab look), in two balanced
  columns. Visual change only.

## v2.0.0

- Complete UI redesign on a shared layout token system: card-style
  after-shot popup with the grind change as the hero number, a new
  display-only "Why?" explainer for each recommendation, and a paged
  history card list (5 shots per page). Behavior unchanged.

## v1.8.x

- Hardened event safety: rinse, flush, steam, and hot water can no longer
  trigger the after-shot popup; added popup re-entry guards and a
  valid-espresso-shot filter for history and recommendations.
- Fixed Done/Back navigation after a flush/rinse/steam interrupts an open
  settings page.

## v1.0 – v1.7

- Initial development: recommendation engine with calibration and
  recommendation modes, read-only SDB integration with dynamic schema
  detection, settings pages, after-shot popup with light/dark themes,
  recent-shot history.
