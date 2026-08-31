# Grind Advisor — Changelog

## v3.13.1 (a cleaning profile no longer blanks the recommendation) — 2026-08-31 — TABLET-VERIFIED 2026-08-31

**Safety status: read-only, unchanged. No write behavior exists beyond the
v3.9.0 SDB resync button. One identity guard in `current_bag_key`; no
recommendation math touched.**

Owner report: running the **"Cleaning/Forward Flush x5"** profile made the
skin's grind tile drop to "-" / "Pull a shot on this bag". Cause: with
profile segmentation on, `current_bag_key` composed *bean + cleaning
profile* — a "bag" with no shots — so `recommendation_for_current_bag`
returned nothing. The cleaning rows themselves never touched the math
(`_row_is_valid_espresso` already rejects them); this was purely the live
identity key.

* `current_bag_key` now answers `""` (the documented identity-indeterminate
  value) when segmentation is on and the live `profile_title` matches the
  same `_text_is_nonespresso` keyword gate that keeps cleaning/rinse/flush
  rows out of the evidence — no second keyword list. Every consumer then
  falls back to its existing fail-safe: `last_recommendation_is_current`
  answers 1, `recommendation_for_current_bag` hands back the saved
  recommendation (the number stays on the tile through the cleaning
  session), and `starting_estimate` stays silent. Switching back to an
  espresso profile restores the normal per-bag path.
* Guarded to segmentation-ON only: with `segment_by_profile` off, the
  bean-only key still matches the bag's shots and there was no bug.
* Known cosmetic corner: pressing Recalculate while a cleaning profile is
  loaded reports "no bean set, nothing to recompute" — accurate enough
  (identity is indeterminate), recompute after switching back.
* Harness grew to 58 checks: the tablet's exact profile string, the other
  gate keywords (backflush, rinse, descale, hot water), an espresso title
  still composing the full key, segmentation-off behavior, the saved-rec
  fail-safes — and a repro check proving v3.12.0 builds the blank-tile key
  the fix removes. No Lumen change needed: the tile follows
  `recommendation_for_current_bag`.

## v3.13.0 (new-bag starting estimate) — 2026-08-29 — TABLET-VERIFIED 2026-08-29

**Safety status: read-only, unchanged. No write behavior exists beyond the
v3.9.0 SDB resync button; the estimate path issues only the existing
Bag Stats SELECTs.**

**Owner-decision reversal, recorded:** this pass deliberately reverses the
2026-08-15 decision *"reset only, never seed a guessed grind for a new bag"*
(previously recorded at the `show_last_recommendation` comment). Reversed by
the owner in the v3.13.0 pass spec (2026-08-29). The v3.7.0 bag-key reset
itself stays — the estimate is layered on top of it, not a replacement.

A freshly scanned bag used to show only "-" and *"New bag - pull a shot to
calibrate"*, making the first shot a blind guess. Now:

* **`starting_estimate` (new public proc)** returns a display-only starting
  grind for a bag with **no shots yet**, borrowed from bags whose Bag Stats
  Theil-Sen fit **converged** (the v3.5.0 trust gate: 4+ shots, ≥ 1.0 grind
  spread, |slope| ≥ 0.5) **on the same profile**. First rung that yields
  data wins: `same_coffee` (same bean + roaster, i.e. a re-bought coffee →
  the most recent such bag's ideal grind), `same_roaster` (median of that
  roaster's bags' ideals), `same_profile` (median of the 6 most recently
  used bags' ideals — internal constant, documented in Help), else empty and
  everything behaves exactly as v3.12.0. Median, not mean; rounded to the
  grind rounding increment and clamped to the grinder range.
* **The estimate is not evidence.** It never enters the regression, never
  counts as a shot, and never touches n, the slope, or Calibration
  Accuracy. Its only callers are display paths (`show_last_recommendation`,
  `show_bag_stats`) — the engine
  (`analyze_latest_shot → _forecast_rec → _compute_forecast/_ladder_small`)
  has no path to it, so shot 1 on a new bag still runs the unchanged
  first-shot rung, and the ladder math (damping 0.5, decay 0.85, dose
  sensitivity 1.8 s/g, outlier 2.0 g) is untouched.
* **Popup**: for a bag with no shots, the result card now leads with
  `Start ~13.5 (est. from 4 bags)`, keeps the new-bag note as the second
  line, then the reason (`Starting estimate: same coffee` / `same roaster
  (4 bags)` / `this profile (6 bags)`). The word "Recommended" never
  appears next to an estimate (estimates render only on the ok-0 card,
  which has no "Recommended Next Grind" line, and `present_result` never
  saves ok-0 dicts, so the estimate cannot overwrite the saved
  recommendation).
* **Bag Stats**: when the loaded bag has no shots and an estimate exists, a
  leading card names the rung and the source bags; the "Bags x–y of N"
  counter excludes that card from the bag count.
* **Help** gained a paragraph explaining the ladder, the 6-bag window, and
  that an estimate is not a calibration.
* Plumbing: the per-bag grouping + Theil-Sen ideal computation moved
  verbatim from `_bag_cards_data` into a structured `_bag_data` helper
  (cards render from it unchanged — the offline harness asserts v3.12.0
  and v3.13.0 produce byte-identical Bag Stats cards); `current_bag_key`
  split into `_current_bag_values`/`_current_profile` so the estimate can
  match bean/roaster field-by-field (composed keys drop empty segments and
  cannot be parsed back). Skins keep reading
  `recommendation_for_current_bag` exactly as before — it still returns
  `{}` for a shotless bag; the Lumen tile picks up `starting_estimate` in
  its own later pass.
* Offline harness `verify_ga_v3130.tcl`: 47 checks — refactor equivalence
  against the archived v3.12.0, the estimate ladder/rounding/clamping/
  guards, both display paths, byte-compile of all 206 procs, and
  nothing-new sweeps for color literals, SQL write keywords, and file
  writes.

## v3.12.0 (page dark mode) — 2026-08-28

**Safety status: read-only, unchanged.** The one new persisted value is
`theme` (light | dark) in the plugin's own settings, saved on each
explicit toggle tap.

- A sun/moon button in the settings page's top-right corner switches
  all seven dui pages between a light and a dark palette instantly; the
  choice persists across restarts. This is separate from **Popup
  theme**, which keeps its own light/dark for the after-shot popup and
  the History / Bag Stats overlays exactly as before — the two can
  disagree on purpose (e.g. dark popup over light pages).
- Every color the pages paint moved from scattered literals into one
  `_apply_palette` proc (the offline harness enforces that no palette
  literal appears anywhere else — the popup's independent `_colors`
  dict excluded). The repaint walk covers page backgrounds, section
  cards, all label/value/help texts, the nine numeric entries, the ten
  History Display Options checkboxes (box + label), the calibration
  gauge (its fill/empty colors are palette tokens, recolored live by
  the existing refresh), and every button face. A restart in dark
  runs one retheme pass after page creation so the checkbox colors
  (framework defaults at creation) come up dark too.

## v3.11.1 (normalization only trusts plausible actuals) — 2026-08-24

**Safety status: read-only, unchanged. No write behavior exists beyond the
v3.9.0 SDB resync button.**

Caught live on the tablet minutes after v3.11.0 went on, during its
verification: the recommendation came back **5.5 → 23.7**.

The engine contradicted itself. Its own yield-source logic looked at a
13.9g actual against a 19.1g dose, called it noise — *"set 38.0g (actual
13.9g out of ratio 0.7)"* — and fell back to the set yield. `_norm_time`
never applied that judgement: it normalized by the raw row value anyway,
inflating a real 50.1s shot into a fictitious **137.0s** one
(50.1 × 38/13.9). v3.11.0 had just put the n=1 rung on normalized time, so
that single inflated point WAS the recommendation: +18 steps, unclamped,
because one distinct grind is not an evidence range (the documented
pass-through). **The regression rung has carried this same exposure since
normalization was introduced** — merely diluted by the other shots and the
outlier exclusion. v3.11.0 did not create the bug; it removed the dilution.

* **`_norm_time` now gates the actuals through the same plausibility rules
  as `_resolve_dose`/`_resolve_yield`**: an actual dose outside
  `dose_min..dose_max`, or an actual yield whose ratio against the best
  dose figure falls outside `ratio_min..ratio_max`, contributes nothing —
  the corresponding correction is skipped, and with no other actual the
  shot normalizes to its raw time. A rejected weigh is a scale mishap or an
  aborted shot, not a measurement. Deliberately independent of
  `dose_yield_mode`: these are sanity bounds, not source preferences.
* `check_guards.tcl` pins both real rows from the incident bag (13.9g
  rejected → t_norm = raw 50.1; 22.2g trusted → 16.43), an absurd-high
  ratio, a zero weigh, an out-of-range dose, and the end-to-end case:
  **the 50.1s/13.9g shot alone answers 9.2, not 23.7.**

Tablet-verified the same hour: Recalculate from History reports
*"SDB resynced, recomputed: 9.2"* and the home card shows 9.2.

## v3.11.0 (the 1- and 2-shot rungs use normalized time) — 2026-08-24

**Safety status: read-only, unchanged. No write behavior was added in this
version. Still nothing but `SELECT`, and the one database write in this
plugin remains the SDB resync behind the Recalculate button (v3.9.0).**

Owner-requested, from a real shot on the tablet: 9.6 seconds on the clock,
but only **22.2g of a 38g target**. Had that shot been allowed to run to
yield it would have taken ~16.4s — and 16.4 vs the 28s target is the error
that says how far off the grind is. The raw clock answered a different
question ("when was the cup pulled away") and recommended a 3-step move,
5.5 → 2.4, where the yield-corrected error justifies half that, 5.5 → **3.6**.

The regression rung has always fitted normalized time (`ynorm=1` since the
normalization was introduced). This release brings the n=1/n=2 ladder —
which also answers for `regression_fallback` and `regression_untrusted` —
onto the same series, so the two halves of the engine no longer measure
different things.

* **`_ladder_small` consumes `t_norm`** for the latest shot's time and for
  the 2-shot slope. **Shots without scale data behave exactly as before**:
  `_norm_time` returns `t_norm == t_raw` when there are no actuals, the same
  convention the regression has always relied on. All six historical bag
  cases in `check_guards.tcl` answer byte-identically.
* **The `normalized` flag is now honest on ladder recs** — set from the
  shots the rung actually consumed (latest; plus the previous shot when the
  2-shot slope was used) instead of hardcoded 0.
* **The reason names the series when it matters**: *"First shot on
  normalized time 16.4s (s/step 3.0)"*. Without that, "9.6s against a 28s
  target, so go coarser" reads as a contradiction. The Why? dialog gains a
  **Normalized time** row on ladder rungs for the same reason — the
  arithmetic stays checkable by hand.
* **The Curve view keys its y series on the rec's `normalized` flag** rather
  than hardcoding raw-for-ladder: a recommendation saved by 3.10.x was
  computed from raw time and carries `normalized 0`, so it still plots in
  the series that produced its number. The axis caption ("time (normalized),
  s" vs "shot time, s") follows the same flag automatically.

`check_guards.tcl` grew a v3.11.0 section pinning: the tablet case
(9.6 raw / 16.43 norm → **3.6**, method `first_shot`, normalized 1), the
no-scale-data identity with 3.10.x (same shot, no actuals → **2.4**), the
2-shot slope from t_norm, the normalized flag when only the *previous* shot
carried scale data, and that the reason names the normalized time exactly
when it was used and never otherwise.

## v3.10.2 (a rounded grind was not a clean decimal) — 2026-08-23

**Safety status: read-only, unchanged. No write behavior was added in this
version. Still nothing but `SELECT`, and the one database write in this plugin
remains the SDB resync behind the Recalculate button (v3.9.0).**

Seen in the tablet log immediately after a Recalculate from History:

```
GrindAdvisor: refresh_from_history: SDB resynced, recomputed: 2.4000000000000004
```

`_round_grind` rounds to the configured increment by dividing, rounding, and
multiplying back. With the increment at 0.1, `round(2.43 / 0.1)` is 24, and
**24 × 0.1 is the next representable double above 2.4** — it prints as
`2.4000000000000004`.

**This was not only cosmetic, which is why it is fixed at the source.** That
value becomes `next` in the recommendation dict, so it was written into
`last_recommendation.tdb`, handed to the Lumen skin, and handed to
ShotHistoryEditor. Every *display* path formats numbers (`_fmt_num`, the
skin's `_num`), which is exactly why nobody had ever seen it — until
`refresh_from_history`'s summary string, which interpolated `next` raw, began
being printed on ShotHistoryEditor's Edit Result and Delete Result pages in
that plugin's v0.6.0 / v0.6.3.

* **`_round_grind` snaps through a fixed-precision string**, returning the
  closest double to the decimal actually asked for. No grinder increment is
  remotely as fine as 1e-6, so nothing real is lost. It also collapses the
  `-0.0` that a tiny negative would otherwise carry into every downstream
  format as "-0.0".
* **`refresh_from_history`'s summary goes through `_fmt_num`**, like every
  other user-facing number here. With the source fixed there is no artifact
  left for it to hide — it formats because it is read by people.
* **`tools/check_guards.tcl`** sweeps 0.0–50.0 at every real increment and
  asserts on the **printed form**, since it is the string that leaked, plus
  that the result is still within half an increment of the input. It carries
  the tablet's exact case, `2.43 → 2.4`, as a named check.

**No recommendation moved.** All six v3.10.0 bag cases return byte-identical
results. Verified against the old implementation, where the check fails at
increment 0.1 — the increment this tablet is actually set to. At 0.25, 0.5 and
1.0 the old arithmetic happens to be artifact-free over this sweep, so that
one increment is doing the work.

## v3.10.1 (the untrusted rung had no label) — 2026-08-23

**Safety status: read-only, unchanged. No write behavior was added in this
version. Still nothing but `SELECT`, and the one database write in this plugin
remains the SDB resync behind the Recalculate button (v3.9.0).**

Owner spotted it on the tablet: the method chip on the home screen read
**`regression_unt...`** — a raw internal dict key, truncated to fit.

v3.10.0 added a fifth forecast rung, `regression_untrusted` (the R² guard
handing a scattered fit back to the ladder), and never added a matching arm to
`_forecast_method_label`. That proc's `default` arm returned the key verbatim,
so the identifier went straight to the screen anywhere the method is named:
the Why? dialog's **Method** row, the shot-list diagnostics, and the
calculation diagnostics dump. The Lumen skin keeps its own copy of the same
map, with the same gap, and that is the one in the screenshot.

* **`regression_untrusted` → "Regression not trusted (ladder)"**, worded to
  match the existing `regression_fallback` → "Regression fallback (pairwise)"
  and the same length as it, so it fits the Why? dialog's value column exactly
  as the current longest label already does.
* **The `default` arm no longer returns the key.** It prettifies whatever it
  is handed (`regression_untrusted` → "Regression untrusted"), so a rung added
  later degrades to readable words instead of leaking an internal identifier
  again. It is a net, not a substitute for adding the arm.
* **`tools/check_guards.tcl` now proves every rung has a label**, and does not
  do it from a hand-written list — a stale hand-written list is exactly what
  caused this. It reads the rung names back out of `GrindAdvisor.tcl` (every
  literal assigned to `method`) and asserts each one comes back from the label
  proc as words: non-empty, not equal to the key, no underscore. Verified
  against the v3.10.0 proc, where it fails on `regression_untrusted` — the
  check reproduces the screenshot rather than merely passing.

**No engine change.** Nothing about the forecast moved: every `eq "regression"`
branch already treated the untrusted rung correctly as a ladder result (raw
time on the Curve view, seconds-per-step in Why?), because those are string
equality tests, not prefix matches. The v3.10.0 guard cases in
`check_guards.tcl` return byte-identical results — the two broken bags still
blocked, the four working bags still unchanged.

**The Lumen skin still shows the truncated key**; its `::lumen::data::grind_method`
map has to be fixed in the skin, in its own pass.

## v3.10.0 (two guards on the regression) — 2026-08-19

**Safety status: read-only, unchanged. Still nothing but `SELECT`. The one
database write in this plugin remains the SDB resync behind the Recalculate
button (v3.9.0).**

Owner-reported: the Sure Shot bag's recommendations were "very inconsistent".
They were. The engine recommended **4.0** for a bag that had been pulled 13
times between **7.5 and 8.2**.

Rebuilt as live formulas in a spreadsheet over the real shot database, the
fit behind that number was:

```
m = -0.62128    b = 30.49171    R2 = -0.0544
ideal = (28 - b) / m = 4.0106  ->  4.0
```

**R² below zero: the fitted line describes the data worse than a flat line
through the mean.** The engine solved it anyway, because its only test was
`|m| >= GA_M_MIN` — and that rejects a *flat* line, not a *meaningless* one.
Noise produces a steep slope just as readily as signal does.

### GA_R2_MIN 0.30 — is the fit worth using?

Below it, the regression is discarded and the small-sample ladder answers
instead, with the reason saying so in as many words: *"Shot times are not
tracking grind on this bag (R2 -0.05 over 13 shots), so the fit was not
used"*.

The threshold was chosen by measuring, not by taste. Every bag in the real
history:

| bag | R² | outcome |
|---|---|---|
| Jorge Diaz Campos | 0.80 | unchanged |
| Origin Colombia | 0.65 | unchanged |
| Chelchele | 0.57 | unchanged |
| Hamasho Anaerobic | 0.42 | unchanged |
| *— 0.30 sits in the gap —* | | |
| Guji Hambela | 0.06 | **blocked** (its fit claimed −15.3 s/grind) |
| Sure Shot | −0.05 | **blocked** |

Four working bags keep their exact recommendations; the two that were
producing nonsense stop.

### GA_EXTRAP_MARGIN 0.5 — is the answer on ground we have stood on?

`_limit_to_evidence` holds any recommendation to the grind range the bag has
actually been pulled at, plus half a step. A fitted line is only evidence
*between* the points that made it; solving it far outside them is arithmetic,
not calibration.

This is applied to the ladder as well, which could bolt for a different
reason: a 2-shot slope is allowed down to `GA_SLOPE_MIN` (0.1 s/grind), and
dividing a 5-second error by 0.1 asks for a 25-step move.

When the clamp bites, the reason says *"held to the grind range actually
tried"* rather than quietly returning a different number than was computed.

### What this does not fix

Neither guard makes a noisy bag calibratable. To dial in a bag like Sure
Shot you have to make a **deliberately large grind move** — a full step or
two — so the change clears the ±3s scatter. The engine damps every
correction by half and will never suggest that on its own.

### tools/check_guards.tcl

New. Sources the real plugin with the framework stubbed and drives
`_compute_forecast` with all six bags' actual shot lists, asserting both
halves of the claim: the guards fire on the two bad bags, and **change
nothing on the four good ones**. Plus `_limit_to_evidence` directly, including
that a bag pulled at a single grind setting has no range and passes through
untouched.

It earned its place twice while being written: it caught `_compute_forecast`
missing a `variable GA_R2_MIN` declaration, and it caught a fixture whose
numbers had been typed by hand rather than generated from `shots.db` — which
"failed" a bag that was fine. The fixtures are generated now.

## v3.9.0 (Recalculate from History, after editing a shot) — 2026-08-18

**Safety status: this plugin still issues no SQL but SELECT and still opens
the shot database read-only. One thing changed and it must be stated plainly:
the new button asks *SDB* to resync itself, and SDB writes its own database
when it does.** That is SDB's own public entry point — the one behind its
"Resync database to history" button — called with SDB's own arguments. It runs
only when you press the new button. Nothing else in Grind Advisor writes
anywhere, and no history file is touched by either plugin here.

### The problem

Owner-reported from the tablet:

> *"After I edited the grind setting from Shot History it worked, but it
> didn't update the Grind Advisor recommended value."*

The edit had worked. The Shot History Editor writes `history/<file>.shot` and
deliberately nothing else. Grind Advisor reads **SDB**, so the correction was
invisible to it — and four separate things kept it that way:

1. **SDB had not re-read the file.** It does re-read a `.shot` whose mtime is
   newer than its stored one (`SDB.tcl:2060`) — but only when `populate` runs:
   on load if `sync_on_startup` is set, and it is `0` by default (and on this
   tablet), or from SDB's own resync button.
2. **`bag_rec_cache` memoizes per bag** and is only dropped when a NEW shot
   lands, via `save_last_recommendation`.
3. **`recommendation_for_current_bag` prefers `last_recommendation`** while it
   matches the loaded bag, so an empty cache would not have helped: the saved
   answer would still win.
4. **`last_recommendation.tdb` reloads that saved answer at startup**, so
   restarting the app did not clear it either.

Any fix that addressed fewer than all four would have looked like it worked
and changed nothing.

### `refresh_from_history`

A new public proc walks all four in order:

1. `::plugins::SDB::populate "" "" 1` if SDB is loaded — its own call, copied
   from its own settings page, guarded by `info procs` and `catch`.
2. `invalidate_bag_rec_cache`.
3. Recompute through **`recommendation_for_bag`**, not
   `recommendation_for_current_bag`. That distinction is the point: the latter
   returns the saved recommendation, which is the stale answer being replaced.
4. `save_last_recommendation` on the result, so the corrected figure is what
   `last_recommendation.tdb` holds and a restart cannot resurrect the old one.

It returns a summary of what it did, and logs the same line.

### Where it lives

**Advanced → Shot Data → Recalculate from History**, a new card in the empty
top-right quadrant beside Popup Tuning. Nothing else on the page moves. A
caption under the button explains what it does and is replaced by the result
("SDB resynced, recomputed: 8.5") after a run, then reset on the next visit so
a stale status can never read as something that just happened.

It is deliberately **not automatic**: step 1 rescans the whole history folder,
which is expensive, and it is the only thing in this plugin that causes a
database write. A button you press is the honest way to spend that.

### Verified

The file parses (`info complete`), and the changed procs
(`refresh_from_history`, the page's `recalculate` / `_default_note` / `show`,
and the whole `Advanced` `setup`) byte-compile. The new card's geometry was
computed from the plugin's own `_init_layout` tokens rather than by eye: card
`208..600`, **56px clear** of the Tools card at `656` (md is 32), right column
ending exactly on the page's content edge, and 112px of note area for a
two-line caption. Not yet tablet-tested.

## v3.8.1 (fix: a keyless saved recommendation masked every per-bag answer) — 2026-08-15

Safety status: read-only, unchanged.

Owner-reported from the tablet: cycling bags still did not change the
recommended grind. v3.8.0's `recommendation_for_current_bag` opened with

```tcl
if {[last_recommendation_is_current]} { return $last_recommendation }
```

and `last_recommendation_is_current` **fails safe by answering "current"
whenever bag identity is unknown**. That is right for the question it was
written for — *"should the tile blank?"* — and exactly wrong for *"should I
bother computing?"*. A recommendation saved before v3.7.0 carries no
`bag_key`, so it claimed to match every bag and the same number came back
forever, which is precisely what the tablet showed.

The saved recommendation is now preferred only on a **positive** match: a
non-empty current key AND a stored `bag_key` equal to it. Otherwise the bag's
own recommendation is computed. With no bean identity at all there is no
per-bag answer to be had, so the saved rec is still the fallback rather than
blanking a machine with no bean fields filled in.

Regression test added: a keyless saved rec must not mask the per-bag answer,
and cycling between two bags must change the number.

## v3.8.0 (per-bag recommendations) — NOT YET TABLET-VERIFIED, 2026-08-15

Safety status: **still read-only. No write behavior is added.** SDB is opened
read-only and only `SELECT`s are run; nothing in `history/` or `history_v2/`
is read, written, renamed or deleted. The only file written remains
`last_recommendation.tdb`, unchanged in shape from v3.7.0.

### The card now knows what to say about a bag you switch back to

v3.7.0 let a skin detect that the saved recommendation belonged to a different
bag — but that was only half a solution, because the tile then went blank. A
bag you switch back to already has its own shots and its own regression, so
blanking threw away a recommendation the data fully supports.

`recommendation_for_bag {bag_key}` computes for any bag, from that bag's own
shots, by running the **same engine at that bag's most recent shot** instead of
the newest shot overall. No math is duplicated: `_forecast_rec` already takes a
position and `_bag_forecast_shots` already filters same-bag from it.

Against the real history this produces, for each bag: Morgon 4.4, Pirates 8.9,
Chelchele 7.3, JIVA Colombia 12.7 and 10.3, Saraya 8.2 — every one from a
regression over that bag's own shots.

This does **not** contradict the owner's "reset only, no seeded number"
decision (2026-08-15). That governs a bag with NO history, which still returns
nothing here. This only ever shows numbers a bag's own shots justify.

`recommendation_for_current_bag` prefers the saved recommendation when it
already describes the loaded bag — that is the freshest possible answer, since
it was computed from the shot just pulled — and otherwise computes.

### Decoration was extracted, not duplicated

Everything the popup shows beyond the bare forecast (bag shot number, channel
warning, confidence, identity, dose/yield source, stable id) moved from
`analyze_latest_shot` into `_decorate_rec {rec rows fields pos}`, so a per-bag
rec is decorated identically. A second copy would have drifted the moment
either changed.

Two call sites became position-relative (`_channel_warning` start and the row
slice handed to `_calibration_confidence`). At `pos` 0 both reduce to exactly
the previous values, and the harness asserts that `analyze_latest_shot` returns
a result **identical** to a direct `_decorate_rec` at position 0.

### Caching

The skin's grind tile is re-evaluated on the app's update tick (~200 ms), so an
uncached lookup would mean reopening SDB and re-running the engine several
times a second. Results are memoized per bag key, and the whole cache is
dropped in `save_last_recommendation` — the single point where a new shot
exists — so a fresh shot can never be masked by a stale cached rec. Only
successful lookups are cached; caching a failure would keep returning nothing
after a transient database problem had cleared.

Verified offline: all changed procs byte-compile; per-bag selection, the
decoration set, cache hit/miss, invalidation on a new shot, unknown bag, empty
key, and both branches of `recommendation_for_current_bag` all pass; the
per-bag numbers were cross-checked against the real `shots.db` by an
independent Python replication.

## v3.7.0 (profile-aware calibration; bag identity on the recommendation) — NOT YET TABLET-VERIFIED, 2026-08-15

Safety status: **still read-only. No write behavior is added in this version.**
The plugin opens SDB read-only and runs only `SELECT`s; it never writes, moves,
renames or deletes anything in `history/`, `history_v2/` or SDB. The only file
it writes remains its own `last_recommendation.tdb`, which gains three small
identity fields (`bag_key`, `bag_label`, `profile`) and no new write path.

### Profile is part of the calibration key

Switching profile now starts a fresh calibration, exactly like opening a new
bag — a different profile is a different extraction, so earlier shots are not
evidence about it. The key is bean · roaster · origin · roast date · profile.

A profile on its own never forms a key: with no bean fields there is no bag
identity, and treating the profile as one would make every shot on such a
machine look like one endless bag.

New setting `segment_by_profile` (default 1) turns it off without a code change.

**Be honest about the evidence for this, and do not let it drift into folklore.**
Replaying the real engine over the author's history produces **7 segments either
way, with identical ideal grinds**. It changes nothing about any shot on record.
The two JIVA "Origin Colombia" bags that look like a profile-driven 10.2 / 12.7
split were **already separate bags** — their `roast_date` differs (one empty, one
`19.09.2025`) — so they are not evidence for profile segmentation. This is a
forward-looking feature with no backward validation, and correspondingly no
regression risk. Verified in the offline harness: with `segment_by_profile` 0 the
key is byte-identical to v3.6.3's for every case tested, including whitespace
and empty-field edge cases.

### Recommendations know which bag they are about

`_forecast_rec` attaches `bag_key`; `analyze_latest_shot` adds `bag_label` and
`profile`. Identity and display only — no recommendation math reads them back.

This fixes a real bug in skin integration: Lumen's grind tile reads
`last_recommendation` directly, and because the recommendation carried no bag
identity, switching bags left the **previous** bag's number on screen until the
next shot was pulled.

Two public procs for skins, neither of which opens the database or touches the
filesystem (safe to call from a 200 ms refresh path):

* `current_bag_key` — the key for whatever is loaded right now, built from live
  `::settings` through the *detected* column names cached by
  `_resolve_bag_fields`, so it can never normalize differently from the key
  built from a stored row. The harness checks this equivalence directly; if the
  two ever diverged, a skin would treat every recommendation as stale.
* `last_recommendation_is_current` — 1 when the saved recommendation is about
  the loaded bag and profile. **Fails safe:** a recommendation saved before
  v3.7.0 has no key, and a machine with no bean fields yields an empty key; in
  both cases it returns 1 rather than blanking a usable number.

`show_last_recommendation` now refuses to present a recommendation belonging to
a different bag, saying which bag it was for instead. Per the owner's decision
(2026-08-15) this is **reset only** — no seeded or guessed grind for a new bag,
and the grinder is left where it is.

### Shot window 40 → 200 rows

The window must reach far enough back to hold every shot of the current bag, and
with several bags in rotation those shots are interleaved with other bags'.
Measured on the real history: the deepest single-bag span is 13 shots, so a
3-bag rotation spans ~39 rows and a 5-bag one ~65 — and on a machine where
rinses and flushes are not purged they consume the same limit (the SQL filters
`removed=1`, not rinses). 40 was tight for two bags and short for three.

Cost measured against the real 1071-row database: `LIMIT 40` = 0.20 ms,
`LIMIT 200` = 0.34 ms. Bag Stats already runs 600 on every open. Applied at both
call sites (the recommendation path and the Calibration Accuracy gauge, which
counts same-bag shots and had the same problem) via one `GA_FETCH_LIMIT`
constant instead of two hardcoded 40s.

### Bag Stats labels

Cards are labelled through the new shared `_bag_label`, which appends the
profile when segmentation is on. Without this, one bean run on two profiles
renders as two identical-looking cards.

### Diagnostics

Now reports the detected profile column, whether segmentation is on, the current
bag key, the saved recommendation's key with a `(matches current)` /
`(STALE …)` verdict, and the shot window size — so none of this is silent.

## v3.6.3 (display-name consistency: "Grind Advisor" in all prose) — docs only, 2026-08-10

Safety status: **no write behavior exists in this version.** Docs-only pass;
the only code change is the version string in `plugin.tcl`.

Owner request (raised while announcing the plugin on the dcamp forum): the
display name is always written "Grind Advisor" (with a space), consistent
with "Bean Scanner" and "Shot History Editor". Every user-visible UI string
and `plugin.tcl`'s `name` metadata already used the spaced form; this pass
aligns the remaining prose mentions in README, CHANGELOG, and PROJECT_STATE.
Technical identifiers are deliberately unchanged: the `GrindAdvisor`
folder/namespace, `GrindAdvisor_*` page names, filenames, install paths,
`msg` log prefixes, and the GitHub repo name `de1app-plugin-GrindAdvisor`.
No tablet verification needed (no behavior change).

## v3.6.2 (Dose/Yield page: Next button inset from the card border) — TABLET-VERIFIED 2026-08-09

Safety status: **no write behavior exists in this version.** One-geometry
bugfix, display only.

Owner-reported: on Advanced → Dose / Yield Source, the "Next" button's right
edge sat flush on the card border. It was drawn to the content edge (`$rx`)
instead of being inset by the card's inner padding. Now `btn_x2 = rx −
sec_pad`, matching the rule every other in-card button already follows
(card action button right inner edge = card right − padding). Nothing else
changed.

## v3.6.1 (theme hardening — self-painted page background, themed button style) — TABLET-VERIFIED 2026-08-09

Safety status: **no write behavior exists in this version.** Display-only
hardening; no math, navigation, or data changes.

Prompted by ShotHistoryEditor v0.5.4's root-cause analysis: Lumen's DYE
integration switches the current dui theme to DYE_Lumen mid-load
(skins/Lumen/skin.tcl:1831). Grind Advisor's un-themed `ga_btn` aspect only
kept working because this plugin happens to load *before* that switch, and
its fpdialog pages were never painting their own background — the grey seen
behind them was whatever page lay beneath. Both latent defects fixed with
the BeanScanner v0.1.2 pattern, copied verbatim: `ga_btn` registered with
`-theme default` plus explicit fill/disabledfill and label fills (stock
periwinkle/white — identical rendered look), and `_page_bg` paints an
explicit full-page grey (#d5d6e3) as the first item of all 7 dui pages.
Color tokens added to the layout block.

## v3.6.0 (Bag Stats card list + Actions entry; long-text clipping fixed) — TABLET-VERIFIED 2026-08-09

Safety status: **no write behavior exists in this version.** Read-only SDB
access unchanged; all changes are display/navigation. Still the one written
file, `last_recommendation.tdb`.

### Bag Stats is now a card list, opened from Actions too

Owner request: Bag Stats under Actions as well, and "similar to history,
neatly in cards". Bag Stats now uses the **same overlay card-list mechanism
as the History list** — 5 cards per page, ◀ Prev / Next ▶, Done — instead of
a wall of text on a dui page:

* card line 1 (bold): bean · roaster + first–last shot dates,
* line 2: Ideal grind · Slope s/grind · Drift s/day (or "Fit not reliable
  (reason)" when the v3.5.0 trust gate fails),
* line 3 (muted): On target in N shots · shot count, outliers excluded,
* header: summary line (target, median slope over reliable bags, average
  R²), toolbar count "Bags 1–5 of N".
* Now lists up to 25 bags (5 pages) instead of 8.

Opened from **both** the main page's Actions card (new third button; the
card grew one row) and Advanced → Tools (same slot as before). The old
`GrindAdvisor_bag_stats` dui page, its registration, and `_bag_stats_text`
are removed — the data logic moved intact into `_bag_cards_data`, and the
overlay procs (`show_bag_stats`, `_bag_page`, `_render_bag_stats_dialog`)
mirror the History dialog's structure verbatim.

### Long text no longer clips (Help, Calculation Details, Diagnostics)

Owner-reported bug: Help / Guide and Calculation Details ran past the card
and clipped under the bottom bar. Fix: `_paginate_text` splits content on
newlines, charges each line its estimated wrapped-line count, and packs
conservative pages (19 body lines / 23 caption lines); a shared
`_add_pager` puts ◀ Prev / Next ▶ plus a "Page 1 / N" indicator in the
bottom bar of all three text pages. Single-page content shows no indicator
and the buttons no-op. Glyphs use the proven `[format %c 0x25C0]` pattern.

### Verified offline

Whole file parses; `_bag_cards_data` exercised against the real tablet
shots.db (9 bags render, numbers identical to v3.5.0's — Pirates ideal
8.9 / slope −3.4 / drift −1.6 s/day); `_paginate_text` unit-checked
(short → 1 page, 50 lines → 3+, wrapped-line costing); every new/changed
proc byte-compiles, including all three paginated page namespaces and the
reworked settings/advanced setups. Geometry re-checked from the token
math: Actions card now ends at y=1384 (48 above the bottom bar), all
other pages unchanged or verified.

## v3.5.0 (Bag Stats becomes a bag comparison view) — TABLET-VERIFIED 2026-08-09

Safety status: **no write behavior exists in this version.** Same read-only
SDB fetch path as v3.4.0 (SELECTs only, 600-row window); everything on the
page is display-only and nothing feeds back into any recommendation. Still
the one written file, `last_recommendation.tdb`.

### Bag Stats now answers "how did each bag behave?"

Per the owner's goal (2026-08-09): fastest dial-in plus per-bag comparison.
Backtesting (see v3.4.0's PROJECT_STATE notes) demoted R² — it gauges fit
quality but predicts nothing — so each bag now shows the numbers that
actually describe its behaviour:

* **ideal** — the grind this bag's robust fit predicts for the target time
  (clamped to the grinder range).
* **slope** — the bean's seconds-per-grind from a **Theil–Sen** fit (median
  of pairwise slopes, median-residual intercept), so one channeled shot
  cannot drag it the way least-squares was dragged (Saraya: LS −26.2 vs
  TS −19.4 s/grind on a 0.4 grind spread — both now correctly refused).
* **drift** — s/day from a 3-parameter fit `t = b + m·grind + c·days`
  (Cramer's rule), shown only with 6+ dated shots spanning 3+ days. This is
  the backtest's real discovery (Pirates −1.6 s/day, R² 0.54→0.83): beans
  speed up as the bag ages, which is why recommendations "chase" late-bag.
* **on target in N shots** — first raw shot within ±2s of target.
* **trust gate** — a bag needs 4+ eligible shots AND ≥1.0 grind spread AND
  |slope| ≥ 0.5, else it shows "fit not reliable (reason)" instead of
  numbers; shots all at one setting are arithmetic, not calibration.

Header: median bean slope across reliable bags (the transferable number),
plus the average R² retained as an overall gauge. New procs `_median`,
`_theil_sen`, `_drift_fit`; `_bag_stats_text` rewritten; page geometry,
navigation, and every other feature untouched.

### Verified offline

Whole file parses in tclsh; all new procs byte-compile; output exercised
against the real tablet shots.db and cross-checked against the independent
Python replication: Pirates ideal 8.9/slope −3.4/drift −1.6 s/day,
Chelchele 7.2/−3.1/−0.8, Morgon 3.0/−2.2 — exact match; Saraya and
LANGBIANG correctly gated as unreliable.

## v3.4.0 (Bag Stats page: per-bag R² and average) — TABLET-VERIFIED 2026-08-09

Safety status: **no write behavior exists in this version.** The new page is
read-only and display-only: it reuses the existing read-only SDB fetch path
(`locate_shot_source` + `_fetch_recent`, SELECTs only, one larger 600-row
window) and nothing on it feeds back into any recommendation. Still the one
written file, `last_recommendation.tdb`.

### New: Advanced → Tools → Bag Stats

The R² shown on the popup, Curve, and Calculation Details is **per bag** — it
is the fit of the current bag's own time-vs-grind regression and resets when
the bag changes. Until now there was nowhere to see it across bags. The new
page lists the latest 8 bags (newest first), each with:

* the bag label (bean · roaster) and its first–last shot dates,
* **R²** and the learned **slope** (s/grind) from the exact same
  recency-weighted regression the forecast uses (normalized time, outliers
  excluded), computed over the whole bag,
* eligible shot count and how many outliers were excluded,
* bags with fewer than 3 eligible shots or no grind spread show "R² n/a"
  with the reason instead of a number.

A summary block on top shows the **average R² across all fitted bags** and
the **median bean slope** of good fits (R² ≥ 0.5) — the slope, not R², is the
number that could seed a new bag's first recommendations in a future pass
(analysis 2026-08-08: the n=1/n=2 rungs assume 3.0 s/step with 0.5 damping,
an effective 6.0 s/step; measured bean slopes ran 2.5–4.9 s/grind, which is
why first-shot corrections under-step and bags take ~3 shots to reach
target). This version only displays the statistics; the recommendation
engine is untouched.

### Implementation notes

* New `_bag_stats_text` (text builder) + `_bag_date_short`, new
  `GrindAdvisor_bag_stats` fpdialog page copied from the Diagnostics page
  pattern (full-width section card, caption font, Done → `_exit_subpage`),
  registered in `preload_settings_page`, launched from the free (row 2,
  col 2) slot of the Advanced Tools card — no geometry changes.
* Verified offline: whole file parses in tclsh, all new procs byte-compile,
  and the page text was exercised against the real tablet `shots.db`
  (600-row window, 9 bags) with per-bag R² values cross-checked against an
  independent Python replication of the engine (0.94 / 0.42 / 0.12 for the
  three most recent bags — exact match).

## v3.3.0 (grind axis labelled; Curve openable from a skin tile) — TABLET-VERIFIED 2026-08-02

Safety status: **no write behavior is added in this version.** Read-only SDB
access, no history-file writes, raw sensor/chart data untouched. Still the one
written file, `last_recommendation.tdb`.

### The grind axis is now labelled

The curve had no x-axis numbers, so you could see the shape but not read off
which grind setting a point belonged to. Both panels now carry a labelled
grind axis:

* Ticks at "nice" grind values, chosen by `_nice_step`, which rounds to the
  **nearest** 1/2/5 × a power of ten (breakpoints at the geometric midpoints
  1.5/3/7). Rounding *up* — the first cut — halved the tick count: a typical
  2.5-wide grind range came out as three lonely whole numbers instead of the
  0.5 steps grinders are actually marked in. Measured across six ranges:
  8.0–10.5 → 0.5, 8.0–14.0 → 1, 8.0–8.4 → 0.1, 5–45 → 10, 9.0–9.2 → 0.05.
* Each tick runs a faint gridline through **both** panels, so a point or its
  residual traces straight down to the grind that produced it — which is the
  reason the two panels share an x scale in the first place.
* Tick marks on both baselines, numbers under the residual strip, and the
  axis caption split into `residuals, s` (left) and `grind setting →` (right).

The grid is drawn immediately after the axes and **before** the data. Tk
stacks later items on top, so drawing it last — as the first version did —
would have laid gridlines over the points.

### Curve opens from a skin tile

New public `show_calibration_curve`, so a skin can open the calibration plot
directly from its own grind tile without the after-shot popup being on screen
first (Lumen v0.17.0 uses it). Same source ladder as
`show_last_recommendation` — in-memory rec, then the saved one, then a fresh
read-only analysis — and `_curve_rec` fills in the bag's shots either way.
`_last_rec_shown` is seeded so the curve's Back button lands on the normal
popup. If there is genuinely nothing to plot it shows the normal error card
rather than empty axes.

### Verified before shipping

All changed procs byte-compile; tick steps checked across six grind ranges;
card geometry re-checked at 1340x800, 1280x800 and 2560x1600 with the added
tick row — no content overruns the button row.

**Tablet-verified 2026-08-02:** the Curve screen opens from Lumen's grind tile
via `show_calibration_curve` and renders with the labelled grind axis.

## v3.2.0 (Curve reads the bag from SDB; optimal grind labelled) — TABLET-VERIFIED 2026-08-02

Safety status: **no write behavior is added in this version.** The SDB re-read
below goes through `analyze_latest_shot`, the same read-only `SELECT` path the
popup already uses — no new query, no new connection mode, no writes. Still
the one written file, `last_recommendation.tdb`.

### Curve no longer asks for a shot you have already pulled

v3.1.0 drew the curve only from shots carried in the recommendation dict, so a
recommendation saved by an older version showed "pull a shot to build the
curve" — even though the bag's shots were already sitting in SDB. That was a
self-imposed limit, not a real one.

`_curve_rec` now fills the gap: if the rec has no `shots`, it re-runs
`analyze_latest_shot` (read-only) and uses that result. The **whole** rec is
replaced, not just its shots — captioning a stale R² and n against freshly
gathered points would report two different fits side by side. It falls back to
the original rec, and to a clear message, only if SDB genuinely yields nothing.

### The recommended grind is labelled on the plot

The dashed vertical at the recommended grind now carries its value. On the
regression rung that vertical crosses the dashed target-time guide exactly
where the fitted line solves for the target, so the crossing of the two guides
is the recommendation itself. The label flips to the other side of the line
when the marker sits near the right edge, so it cannot run off the card.

### Verified before shipping

Re-ran the off-tablet checks: all touched procs byte-compile; card geometry
still clean at 1340x800, 1280x800 and 2560x1600; and the new recompute path
was exercised end to end — a legacy rec with no shots now produces a full
5-shot plot with a caption computed from the recomputed fit (`R² 0.994 · bias
−0.20s · spread ±0.24s · n=5`) rather than a dead end.

**Tablet-verified 2026-08-02:** the Curve screen renders and plots the current
bag on the Samsung Tab A9 (1340x800) without pulling a new shot.

## v3.1.0 (feature: Calibration Curve view) — superseded by v3.2.0, never tablet-verified

Safety status: **no write behavior is added in this version.** SDB stays
read-only; no history-file writes; raw sensor/chart data untouched. The only
file this plugin has ever written, `last_recommendation.tdb`, is still the
only one — it now also carries the eligible shot list (grind + time pairs the
model already used), which grows the file by roughly 60 bytes per shot on the
bag. Hooks, popup trigger conditions, rinse/flush/steam event filtering,
popup guards, and Done/Back navigation are all unchanged.

### The Calibration Curve screen

The after-shot popup gains a fourth button — **OK / Why? / Curve / History**.
**Curve** replaces the overlay with a calibration view and **Back** returns to
the popup, exactly like the existing Why? explainer. No new dui page, no new
navigation machinery.

The view shows, on one shared grind axis:

* **Scatter + fitted line.** Every eligible shot on the current bag, with the
  latest shot drawn larger and in the accent colour. The line is the one the
  model actually solved, plus a dashed guide at the target time and a dashed
  vertical at the recommended grind.
* **Residual strip.** Each shot's deviation from that line, drawn as a stem to
  zero so a one-sided (biased) pattern is visible at a glance rather than
  inferred from a number.
* **Caption:** `R² · bias · spread · n`.
* **A plain-language verdict** — including the case this view exists for: a
  high R² sitting on top of large residuals is called out as "the trend is
  right, the individual predictions are not," instead of being left to look
  like a good result.

### Why bias and spread, not just R²

R² measures correlation, not accuracy. A line can track the trend closely and
still sit consistently off the points, and with two points it is 1.000 by
construction while saying nothing at all. So the caption reports the mean
residual (bias) and the RMS residual (spread) beside it, and the residual
strip shows the pattern R² collapses into a single number.

**Below 3 shots, bias and spread are deliberately not shown.** At n=1 and n=2
the ladder solves a line *through* the latest shot, so those statistics
describe the construction rather than the calibration — at n=1 they are
identically zero. The caption drops them and the verdict says plainly that
there is no fit quality to judge yet.

### Matching the model exactly

The plotted y series is the series that was actually fitted, or the drawn
residuals and the reported R² would describe different things:
`_weighted_regression` is always called with `ynorm=1`, so the `regression`
rung plots **normalized** time against its own `m`/`b` (and the axis says so);
every other rung comes from `_ladder_small`, which works in **raw** time, so
those plot raw time against the line the ladder implies — through the latest
shot with slope `-s_per_step`, because it solves
`next = grind + (t - target)/s_per_step`.

Everything is derived from the rec dict already computed and shown. No new SDB
read, no re-analysis, no recomputation — the same rule the Why? explainer
follows.

### Changed

* `_forecast_rec` now carries `shots` in the rec dict (the eligible list it
  already built). This is what makes the curve drawable, and persisting it
  means the view survives an app restart.
* New: `_curve_model`, `_curve_caption`, `_curve_verdict`, `_curve_and_stay`,
  `_show_curve_dialog`, `_draw_curve_panels`.
* The popup button row is now four buttons, so its gap tightens from 5% to 3%
  of card width; at the old gap the four labels would not fit.

A recommendation saved by v3.0.0 has no stored shots. **Curve** says so and
asks for one shot, rather than failing or drawing an empty plot.

### Verified before shipping

Off-tablet, since this is drawing code: all new and touched procs
byte-compile; the model was exercised across every ladder rung (`first_shot`,
`two_shot`, `regression` tight, `regression` high-R²-loose,
`regression_fallback`, and a pre-3.1.0 rec with no shots); and the card
geometry was checked at 1340x800, 1280x800 and 2560x1600 with no content
overrunning the button row or leaving the card. **Not yet run on the tablet.**

## v3.0.0 (major: single Regression Forecast method; modes removed) — TABLET-VERIFIED 2026-07-10

Safety status: **no write behavior exists in this version.** SDB stays
read-only; no history-file writes; raw sensor/chart data untouched. Hooks,
popup trigger conditions, rinse/flush/steam event filtering, popup guards,
and Done/Back navigation are all unchanged.

The four recommendation modes (Conservative / Normal / Dynamic Barista /
Aggressive New Bean) and their per-mode caps are **removed**. There is no
mode setting. One built-in, non-selectable method replaces them.

### The method — a ladder by eligible shot count n on the current bag

n resets when the bag changes. Eligibility = the existing espresso guards
plus exclusion of any shot whose weighed dose or yield is more than 2.0 g
off the set value.

* **n=1:** `change = (shot_time − target) / 3.0 × 0.5`, `next = grind + change`.
* **n=2:** same, with `s/step = |(t2−t1)/(g2−g1)|` when the grind moved and
  |slope| ≥ 0.1, else the 3.0 default.
* **n≥3:** recency-weighted (`w = 0.85^(n−i)`, newest = 1.0) least-squares of
  normalized time vs grind; `m,b` from the five weighted sums; ideal grind =
  `(target − b) / m`; rounded (sign-safe `round(x/incr)*incr`) and clamped to
  grinder min/max. Guards (all shots at one grind, or |m| < 0.5) fall back to
  the n=2 method and say so in the reason. The grinder direction is the sign
  of m, learned from the data — no direction setting.

Normalization (guarded on yield_actual > 0):
`t_norm = shot_time × (yield_set/yield_actual) − (dose_actual − dose_set) × 1.8`.
With no scale columns the fit runs on raw time and Diagnostics notes it.

Internal constants (documented in Help, never settings): s/step default 3.0,
damping 0.5, slope floor 0.1, m floor 0.5, decay 0.85, dose sensitivity
1.8 s/g, outlier 2.0 g, min shots 3.

### Validation (headless tclsh)

Reproduces the reference `GrindAdvisor_Shot_Calculation2.xlsx` "Forecast
Method" sheet EXACTLY: the five weighted sums, m = −4.0977, b = 81.139, ideal
grind 12.968 → rounded **13.0**, predicted 27.87 s. Single fast shot
(grind 13.5, time 19.1, target 28) → **12.0**. Edge cases verified: all shots
at one grind → regression fallback with no divide-by-zero; missing scale
columns → unnormalized regression runs; a 3.5 g dose-error shot is excluded
(n drops, excluded count rises). Model-fit R² is the ordinary goodness-of-fit
(≈0.92 on the reference), and Diagnostics reports normalized-vs-raw R² so the
value of normalization is visible per dataset. (Note: the spec's "R² ≈0.97
normalized vs ≈0.81 raw" is dataset-dependent and does not reproduce on the
reference sheets — the recommendation numbers all match exactly; the plugin
reports the actual computed R² rather than a fixed figure.)

### Reason + Diagnostics

Reason states the rung ("First shot", "2-shot calibration", "Regression over
N shots" with the learned slope and predicted time, or "Regression fallback:
<why>"). The Why? popup and Calculation Details show method, eligible n,
outliers excluded, m, b, R² (normalized vs raw), normalization state, ideal
grind, and the constants.

### Settings / migration

Removed settings: `recommendation_mode`, `default_seconds_per_step`,
`first_cap`, `later_cap`. On first load of v3.0.0 any stored values are
ignored (they are simply not read; `apply_defaults` no longer defines them),
no error, no orphaned UI. Kept: target time, rounding increment, grinder
min/max, popup + theme + font-scale, and the v2.3.0 dose/yield source mode.

### UI

Main-page Recommendation card dropped the mode row (now rounding control +
Calibration Accuracy gauge, 2 rows); the Popup card moved up accordingly.
Advanced dropped the now-empty Calculation Tuning card (Popup Tuning + Tools
remain). Help rewritten: mode explanations replaced by a 3-line ladder
description and the reason-string meanings.

### Removed / added code

Removed `compute_recommendation`, `_recommendation_reason`, `_mode_label`,
`_cap_for_mode`, `_dynamic_cap`, `_find_calibration_shot` (now unused), and
`cycle_mode`. Added `_forecast_rec`, `_compute_forecast`,
`_weighted_regression`, `_ladder_small`, `_bag_forecast_shots`, `_norm_time`,
`_is_outlier`, `_forecast_round`, `_forecast_reason`, and display helpers.
The v2.x Calibration Accuracy gauge and dose/yield source mode are unchanged.

## v2.3.0 (feature: dose/yield source mode)

Safety status: **no write behavior exists in this version.** SDB stays
read-only; no history-file writes. The grind recommendation math and
calibration shot-matching are byte-identical to v2.2.0 (`compute_recommendation`,
`_recipe_matches`, `_shot_from_row` untouched).

New "Dose / Yield Source" mode (Advanced → Dose / Yield Source) with three
options, default **Auto**:
* **Fixed** — always the set/target dose and yield (prior behavior).
* **Actual** — the measured dose and final yield; a 0/empty/missing
  measurement always falls back to set (with a label). No bounds applied.
* **Auto** — measured values only when plausible: dose within
  [dose_min, dose_max], and ratio (yield/dose) within [ratio_min, ratio_max];
  otherwise set. A 0/empty/missing measurement always falls back.

Plausibility bounds are settings, not constants: `dose_min` 12.0, `dose_max`
22.0, `ratio_min` 1.0, `ratio_max` 4.0 (all editable on the sub-page, kept in
the top half of the screen for the Android keyboard).

**Scope decision (user-confirmed):** the grind recommendation is computed
from shot *time* vs target only — dose/yield never enter that formula, and
their only other effect (calibration shot-matching) deliberately keeps using
set/target values for stability. So this mode governs **display only**: the
after-shot popup, the reason line, the ratio, the Calculation Details
summary + trace, and Diagnostics. It never changes the recommended grind.

**Transparency (never silent):** the popup reason line and the Diagnostics /
Calculation Details dumps always state the source used, e.g.
"dose: actual 18.4g" or "dose: set 18.0g (actual 0.0g rejected)" or
"(actual 25.0g out of range)" / "(actual 5.0g out of ratio 0.3)".

Field detection: added an `actual_dose` logical field (patterns
`bean_weight`, `actual_dose`, `measured_dose`, `dose_in`, `scale_dose`,
`dry_weight`, `ground_weight`), resolved after the set-dose column so it
never steals it. If no measured-dose column exists (common on the DE1, where
dose is user-entered), dose gracefully stays on set and Diagnostics shows
"actual dose column: not detected". Measured yield already came from
`drink_weight`.

Display changes: the popup's "Set Yield" row is now "Yield" (source is on the
reason line) and a "Ratio 1:x.x" row was added; the Advanced Tools grid
gained a "Dose / Yield Source" button (now a 2x3 grid); Help documents the
mode.

Verified in tclsh across all three modes and edge cases (good measured
values, zero/missing measurement, out-of-dose-range, out-of-ratio, and a
missing actual-dose column) — every case resolves and labels correctly.

## v2.2.0 (read-only diagnostics: per-shot calculation trace) — TABLET-VERIFIED 2026-07-10

Safety status: **no write behavior exists in this version.** Display-only
diagnostics addition; zero changes to recommendation math, hooks, popups,
gates, navigation, or settings behavior.

* New `_shot_trace_text`: for the latest 5 valid shots, recomputes each
  recommendation through the exact procs the popup/history already use and
  dumps every input and intermediate -- date/time, grind, shot time, target,
  signed error, dose, set yield, bag shot number, seconds-per-step with its
  calibration source (same bag / recipe / default estimate), the cap tier
  and whether it was hit, the raw next grind before cap/rounding, the final
  recommendation, and the reason. Two compact lines per shot, newest first.
* Shown at the bottom of the Calculation Details page (its text switched to
  caption size so the summary block plus 5-shot trace fit the card without
  overflow); the Diagnostics page gained a pointer line to it.
* Verified via a stubbed-data tclsh run: spp derivation, cap tiers engaging
  (e.g. "cap 0.25 HIT"), and source switching (default est -> same bag)
  all render correctly.

## v2.1.1 (bugfix: deleted shots no longer appear) — TABLET-VERIFIED 2026-07-10

Safety status: **no write behavior exists in this version.** The fix adds
read-only SELECT filters and file-existence checks only.

Bug (tablet-reported): a shot soft-deleted via ShotHistoryEditor vanished
from `history/` but kept appearing in Grind Advisor's history, calibration,
and confidence score — even after an SDB resync and app relaunch.

Root cause (confirmed in `plugins/SDB/SDB.tcl:2205-2216`): deleting a shot
removes the history file but never touches SDB (by design — SDB is
read-only for editor plugins). SDB's own resync then *flags* the orphaned
row with `removed=1` rather than deleting it. Grind Advisor's fetch only
filtered on grind/duration being present and ignored both the `removed`
flag and file existence, so the deleted shot's row stayed visible forever.
(ShotHistoryEditor hides it via its own manifest — the per-plugin
tolerance the workspace data-model rules prescribe.)

Fix — two defensive layers, all consumers covered via the shared pipeline
(popup, history cards, "Show Latest", calibration search, bag counts,
confidence gauge):
* `_fetch_recent` adds `AND (removed IS NULL OR removed=0)` to the WHERE
  when the schema has a `removed` column (exact-name detection) — covers
  everything after any resync.
* `_row_is_valid_espresso` rejects rows whose `filename` no longer exists
  in `history/` or `history_archive/` (with or without the `.shot`
  extension) — covers the window before a resync runs. Skipped entirely
  when the schema has no filename column or the history folders can't be
  located (e.g. desktop testing), so nothing is ever over-filtered.
* Diagnostics page now shows the detected filename/removed columns and
  whether the file check is active.

Verified in tclsh with a simulated history folder: deleted shot filtered
both pre-resync (file gone, flag 0) and post-resync (flag 1); schemas
without the columns keep all rows; survivors exactly match remaining files.

Recommendation math, navigation, popup logic, and all v2.x behavior
otherwise unchanged.

## v2.1.0 (feature: Calibration Accuracy confidence gauge) — TABLET-VERIFIED 2026-07-10

Safety status: **no write behavior exists in this version.** Adds a
display-only calibration confidence gauge; recommendation math outputs are
unchanged (the score is attached to the result dict after the
recommendation is fully computed and is never read back by any math).

Formula, in plain language (also on the Help page): relevant shots are the
valid shots sharing the current shot's bag — or matching its recipe when no
bag fields exist, the same fallback order the calibration search uses, so a
new bag resets the score exactly like it resets calibration.

* evidence = min(relevant_count, 5) / 5
* consistency = 1 − (average |shot time − target| over the most recent
  up-to-4 relevant shots) / 10 s, floored at 0
* score = round(100 × (0.4·evidence + 0.6·consistency))
* fewer than 2 relevant shots → "Not enough data" instead of a score

Bands (thresholds in the token block): 0–39 Poor, 40–64 Fair, 65–84 Good,
85–100 Excellent.

Verified against concrete histories (target 28 s):
* 1 shot → Not enough data
* 3 consistent shots (27 / 29 / 26.5 s) → 77% Good
* 4 erratic shots (20 / 35 / 22 / 36 s) → 49% Fair
* 6 consistent shots (±1 s) → 96% Excellent
* 2 shots, one poor (28 / 36 s) → 52% Fair

Display (all token-driven, shared fonts, no new patterns):
* Main page: third row of the Recommendation section card — label
  "Calibration Accuracy", a 10-segment rounded bar (one accent fill
  `#4e85f4`, neutral empty `#e3e6ee` — no rainbow) and "82% — Good" in
  caption size beside it. Card grew to three rows (208–808); the Popup card
  moved down (856–1304), still 128 above the bottom bar; no overlaps
  (segments end at x=2130, text column 2178–2420).
* After-shot popup: one caption line under the Reason block,
  "Calibration: 82% — Good"; the card height accounts for it. Older saved
  recommendations without the fields simply omit the line.
* Help page: new paragraph explaining the score, what raises/lowers it,
  and that it never changes recommendations.

New procs: `_calibration_confidence` (pure scorer), `_confidence_band`,
`calibration_confidence` (settings-page fetch through the existing
read-only pipeline), `refresh_confidence` (gauge painter). Navigation,
gates, and the v2.0.2 fixes untouched.

## v2.0.2 (bugfix: history-button glyphs + Done ping-pong) — TABLET-VERIFIED 2026-07-07

Safety status: **no write behavior exists in this version.**

Two tablet-reported bugs fixed:

* **History overlay showed "25C0 Prev" / "Next 25B6" instead of arrow
  glyphs.** Root cause: the v2.0.1 build step converted literal Unicode
  characters to escape sequences with `sed`, whose replacement syntax treats
  a backslash-u specially — it silently ate the backslashes, leaving bare
  hex digits as visible text in six strings (the two paging buttons, the
  history card "Grind X → Y" arrow, the middle-dot separator, the
  "Shots M–N of T" dash, and the Advanced subtitle's em dash). All six are
  restored as proper `\uXXXX` escapes (written via a Tcl rewrite, verified
  byte-exact; the file is pure ASCII again). Lesson recorded: never use sed
  for backslash-escape edits.
* **Done ping-pong: Advanced → Done went to main settings, then Done went
  back to Advanced instead of leaving to Extensions.** Root cause: the
  return-page capture in the settings page's `show{}` only skipped
  transient machine-state pages. Returning from a sub-page re-shows
  settings with `page_to_hide = GrindAdvisor_advanced`, which passed the
  filter and overwrote the genuine entry page. `_capture_return_page` now
  also rejects the plugin's own pages (`GrindAdvisor_*`), so the entry
  target captured when the user first opened the plugin survives sub-page
  round-trips. (This latent bug existed since v1.8.8; it only became
  reachable in practice once v2.0.0 moved sub-pages behind Advanced.)

No changes to recommendation math, SDB reads, gates, popup logic, or any
control's command; the navigation mechanism is unchanged apart from the one
extra capture filter.

## v2.0.1 (visual grouping into section cards; behavior identical)

Safety status: **no write behavior exists in this version.** Layout and
grouping only — zero changes to logic, navigation, SDB reading, validation
gates, or what any control does (all commands byte-identical to v2.0.0).

Adopted the stock DE1app "App tab" visual pattern: related controls grouped
inside white rounded section cards (via the existing `rounded_rect` helper —
the core's own `dui::item::rounded_rectangle` is an internal post-rescale
button primitive, so the design-system helper is the correct outward-facing
equivalent) with a section title, on the grey page background.

New layout tokens (all values virtual, scale 2.0 from the 1340x800
reference): `sec_top` 208, `sec_gutter` 48, `sec_col_w` 1164, `sec_col2_x`
1304, `sec_pad` 48, `sec_gap` 48, `sec_title_h` 48, `sec_label_w` 400,
`sec_value_dx` 472, `sec_btn_w` 280, `entry_row_h` 96, `entry_row_pitch`
128, `sec_radius` = card radius, plus card fill/outline colors. New
`_sec_card` helper draws the card + title and returns the first row's y.

Main settings page, two balanced columns of cards:
* LEFT: **Shot Settings** (208–736: Target shot time, Grinder min/max as
  entry rows — placed first so all numeric inputs sit at y centers
  384/512/640, comfortably in the top half for the Android keyboard; the
  spec's Actions-first order was swapped for this reason), then **Actions**
  (784–1232: Show Latest Recommendation + History as stacked full-width
  buttons — side-by-side would truncate the long label at this column
  width).
* RIGHT: **Recommendation** (208–656: rounding + mode rows), **Popup**
  (704–1152: theme + automatic-popup rows). Rows use the card-relative
  grid: labels at card+48, values at card+472 (width 292, wraps within the
  row if long), buttons (280 wide) right-aligned to card inner edge.
* Bottom bar unchanged and outside any card; Done/Advanced commands
  byte-identical. The v2.0.0 toolbar action buttons moved into the Actions
  card.

Advanced page: **Calculation Tuning** (left, 3 entries, 208–736) and
**Popup Tuning** (right, 2 entries, 208–608) cards — all numeric inputs in
the top half — plus a full-width **Tools** card (784–1232) holding the 2x2
button grid (same `open_settings_dialog` targets).

Help / Diagnostics / Calculation Details: body text now sits inside a
full-width section card ("Guide" / "Detected Fields" / "Latest
Calculation") spanning 208 to 32 above the bottom bar. History Display
Options left unchanged (not required; avoids crowding). Sub-page Done/Back
untouched.

Overlap audit: title-to-first-row gap = md inside every card; value columns
end ≥ lg before buttons; cards separated by 48 (columns) / 48 (vertical);
lowest card ends 1232, bar starts 1432. Zero hardcoded coordinates outside
the token block (grep-audited); zero SQL write keywords; file remains pure
ASCII.

## v2.0.0 (complete UI redesign; behavior identical to v1.8.8) — TABLET-VERIFIED 2026-07-07

Safety status: **no write behavior exists in this version.** Complete UI
redesign only: recommendation math, SDB reads, hooks, event validation
gates, popup_active guards, and the v1.8.8 Done/Back navigation mechanism
are all unchanged (navigation targets and procs byte-identical). The two
temporary `DIAG` log lines from the navigation fix were removed
(tablet-verified).

### Design system (mirrored from ShotHistoryEditor v0.3.1–v0.3.3)

* New `_init_layout` token array `L`: all page coordinates in the app's
  virtual 2560x1600 canvas space (confirmed against `de1app-core/dui.tcl`),
  fonts as real Tk font objects in physical pixels from the detected screen
  (`GA_title` 40 bold, `GA_section` 24 bold, `GA_primary` 22 bold,
  `GA_body` 19, `GA_caption` 16, `GA_button` 20 bold; 16px floor). Shared
  `ga_btn` rounded button style (radius token); every button label passes
  `-label_font $L(font_button)`. `rounded_rect` helper copied verbatim.
* Zero hardcoded coordinates below the token block (audited by grep).

### Pages

* **Main settings**: centered 40px title, caption subtitle; toolbar row
  holds [Show Latest Recommendation] (left) and [History] (right) — same
  commands as before; General Options as 7 label/value rows (labels at
  left_x, values/controls at value_x, lg gap): the 3 numeric entries first
  (top half of screen, rows end at y=760 of 1600, for the Android
  keyboard), then rounding / mode / theme / automatic-popup rows, each with
  a value text and a standard [Next]/[Toggle] button ending at right_x.
  The old checkbox became an On/Off value + Toggle button flipping the same
  enable_popup 0/1 setting. Bottom bar: [Done] left / [Advanced] right.
* **Advanced** (regrouped per spec): the 5 tuning entries in two columns in
  the top half, plus a 2x2 button grid gathering History Display Options,
  Diagnostics, Calculation Details, and Help / Guide (all moved off the
  main page; open calls unchanged: `open_settings_dialog <page>`).
* **History Display Options**: 2x5 checkbox grid on the token layout.
* **Help / Diagnostics / Calculation Details**: standard header + body +
  bottom-left Done.
* All six Done buttons still call the untouched `_exit_settings` /
  `_exit_subpage` navigation.

### After-shot popup (raw-canvas overlay, physical pixels as before)

* Rounded card (~65% width, content-driven height) centered on the theme
  scrim; "✓ Shot Saved" section header; hero line "13.5 → 12.0" in
  title-size bold accent with the direction ("finer by 1.5") in body size
  under it; detail label/value grid (Time/target, Dose, Set Yield,
  Bag Shot #); caption "Reason" label + wrapped body text; channel warning
  as caption line when present.
* Button row [OK] [Why?] [History], identical sizes, evenly spaced. NEW
  [Why?] opens a display-only explainer card built purely from the rec
  dict already computed and shown (mode, seconds-per-step + calibration
  source, raw next grind, cap, rounding, grinder range, reason) — no new
  SDB reads or analysis; [Back] returns to the popup, [OK] closes.
* Error popup: same card style with warning header + [OK] [History].
* Light/dark themes: one layout, two color sets (added a "muted" secondary
  text color to both).
* popup_active guard flow unchanged: present_result sets it, _close_dialog
  clears it; Why?/History replace the overlay content so the flag stays
  correctly set while any overlay is up.

### History overlay

* Rebuilt as the standard card list: 5 rounded cards per page with three
  baselines (primary: date/time + "Grind X → Y"; secondary: dose, ratio,
  yields, time; caption: reason, bag shot, channel warning — all honoring
  the history_show_* options), toolbar with "Shots M–N of T" left and
  [◀ Prev]/[Next ▶] right, [Done] bottom-left. Ellipsis truncation via
  font measurement so nothing collides. Same read-only data pipeline
  (fetch → valid-espresso filter → per-shot recommendation), fetched once
  per open and paged from memory.

### Layout audit (virtual 2560x1600; scale 2.0 from the 1340x800 reference)

* margin 92; left_x 92; right_x 2468; content_w 2376; value_x 980;
  col2_x 1328; col2_value_x 2216.
* Spacing tokens xs 12, sm 20, md 32, lg 48, xl 64, xxl 96.
* Buttons: std 400x120, wide 480x120, radius 24. Cards (overlay, physical):
  h 96, gap 12, radius 12, pad 18, baselines +30/+56/+80.
* Header title y 56, subtitle y 144; toolbar 208–304; list_top 336;
  row pitch 152; bottom bar 1432–1552.
* Main page: 7 rows, last ends 1368 → 64px gap to the bar. Advanced:
  entries end 760 (numeric inputs all in top half), tools grid 824–1096.
* Compromises: toolbar-zone buttons are 96 virtual (48 physical) tall,
  matching ShotHistoryEditor's proven toolbar Prev/Next (the 60px touch
  minimum applies to body/bar buttons, which are all 120 virtual).

## v1.8.8 addendum

Tablet-verified: the v1.8.8 captured-return-page navigation fix works
(settings → flush → stop → Done exits correctly, no crash, no popup).

## v1.8.8 (final navigation fix, root cause confirmed from de1app-core source)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched; this pass only changes Done/Back navigation.

### Root cause (confirmed, not guessed, from `de1app-core/dui.tcl`)

`proc ::dui::page::load` (the "Handle page stack" block, ~line 6462) resets
`page_stack` to a single entry, `[dict create $page_to_show {}]`, every time
a page of type `"default"` is shown -- e.g. the flow-monitor screen shown
while a flush/rinse/steam runs. This happens even while a `fpdialog` page
(Grind Advisor's settings pages) is current, because the "only one dialog
page can be visible" guard a few lines above (~line 6415) only checks page
type `"dialog"`, not `"fpdialog"`. When the app re-shows
`GrindAdvisor_settings` after the flow ends, it gets pushed onto that
freshly-wiped stack, so `page_stack` becomes
`{<flow_page>: {}, GrindAdvisor_settings: <callback>}`.
`::dui::page::close_dialog` (~line 6780) navigates to `previous`, defined as
the second-to-last key of `page_stack` (`::dui::page::previous`, ~line 5964)
-- i.e. the flow page. **This is exactly why Done lands on the flush
screen**, and it is not specific to Grind Advisor: it is how this DE1app
build's `page_stack` behaves for any `fpdialog` left open across a
state-driven page change, with no self-healing.

`plugins/Graphical_Flow_Calibrator` survives the identical interruption
because its "GFC" page is plain (`type "default"`, never `fpdialog`) and its
Exit never consults `page_stack`/`close_dialog` at all: it renames the
global `::page_show`, captures `[dui page current]` into `::gfc_start_page`
only when the wrapped call is genuinely `page_show GFC`, and its `exit`
proc calls `dui page load $::gfc_start_page` directly.

### Fix (convergence, grounded in the confirmed mechanism)

* Every Grind Advisor page's `show{page_to_hide page_to_show}` callback
  (called by `::dui::page::load` itself on every single page-show, genuine
  entry or not) now calls `_capture_return_page $page_to_hide`, which
  records the real previous page as `_settings_return_page` -- but skips the
  update whenever `page_to_hide` looks like a machine-state flow/monitor
  page (`_is_transient_name`, matching the real state names confirmed in
  `de1app-core/machine.tcl`'s `::de1_num_state` array: Espresso, Steam,
  HotWater, HotWaterRinse, SteamRinse, Descale, Clean, AirPurge, etc.). This
  means a flush/rinse/steam interruption's re-show can never overwrite the
  last legitimate return target.
* `_navigate_done $target`: if `$target` is non-empty, not transient-looking,
  and `dui page exists $target` confirms it is a real, currently-registered
  page, navigates with `dui page load $target` -- confirmed in `dui.tcl` to
  be the same underlying call `open_dialog` itself uses
  (`open_dialog` is literally `dui page load $page {*}$args`), so this is
  not a different/riskier mechanism than opening the dialog, just reusing
  it. Otherwise (or if that call errors) it falls back to
  `dui page close_dialog`, matching every reference plugin's normal-case
  behavior exactly. Errors are logged via `msg`, never swallowed by a bare
  `catch`.
* `_exit_settings` (main settings Done) uses the captured, filtered
  `_settings_return_page`. `_exit_subpage` (Advanced, Help, History Display
  Options, Diagnostics, Calculation Details) always targets
  `GrindAdvisor_settings` -- a real page this plugin itself registers, not a
  guess.
* Removed all v1.8.7 exploratory diagnostic logging (per-state-change and
  per-page-display-change log lines) now that the real mechanism is
  confirmed; kept exactly one diagnostic log line on Done press showing
  which path was taken (`DIAG Done -> dui page load $target (captured)` or
  `DIAG Done -> dui page close_dialog (no valid captured target)`), to be
  removed once tablet-verified.

Not changed: recommendation math, SDB reads, popup content/layout, settings
semantics, or the v1.8.2 popup guards (`popup_active`, `_overlay_exists`,
`_popup_blocked`, `_flow_active`, `_on_settings_page`, `_install_nav_watch`,
`_nav_state_change`, `_nav_page_change`).

## v1.8.7 (diagnostic only — no behavior change)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched. No navigation behavior was touched either:
Done still uses the plain `dui page close_dialog` from v1.8.6.

Confirmed on-tablet: v1.8.6 fixed the crash (Done no longer errors), but the
original, milder bug is back exactly as expected -- after starting and
stopping a flush while in Grind Advisor settings, Done lands on the flush
screen instead of leaving settings. Three prior attempts to fix this
(v1.8.3, v1.8.4, v1.8.5) each guessed at a different navigation mechanism
and each guess was wrong in a different way. Rather than guess a fourth
time, this version adds read-only diagnostic logging (via the existing
`msg` convention, e.g. `catch { msg "GrindAdvisor: DIAG ..." }`) so the
actual page/state bookkeeping during the bug can be read from the DE1app
log after reproducing it on the tablet:

* `preload_settings_page` logs `dui page current` at genuine entry.
* `_nav_state_change` (already traces `::de1(state)`) logs the state name
  and `dui page current` whenever the state actually changes (deduplicated
  so it isn't spammed for repeat writes of the same state).
* `_nav_page_change` (already traces `page_display_change`) logs its
  arguments and `dui page current`.
* `_exit_settings`/`_exit_subpage` log `dui page current` immediately before
  and immediately after `dui page close_dialog`.

To help find the real fix: reproduce the bug (settings → flush → stop →
Done lands on flush), then pull the DE1app log and look for the
`GrindAdvisor: DIAG` lines covering that sequence. That will show what page
name the framework considers "current" at each step, which is the piece of
information needed to pick a real fix instead of a fourth guess. Remove
this logging once the real fix is implemented.

## v1.8.6 (bugfix: revert Done navigation to the reference-plugin mechanism)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched; this pass only changes the Done/Back
navigation call.

v1.8.5's fix made Done crash with the same `A_Flow` plugin-load error on
*every* press, not just after a flush interruption. Root cause: v1.8.5
copied Graphical_Flow_Calibrator's `dui page load $return_page` pattern, but
GFC's pages are plain top-level pages, never `-type fpdialog`. Grind Advisor's
settings pages are all registered `-type fpdialog`. On this DE1app build,
`dui page load` does not correctly exit an fpdialog page -- regardless of
what name it is given (guessed or captured), it falls through to the app's
plugin-loading resolver, which is what produces the `A_Flow` error. That
mechanism simply does not transfer from GFC's page architecture to
Grind Advisor's.

This is the fourth navigation attempt (v1.8.3 crashed via a captured
fallback, v1.8.4 silently no-op'd, v1.8.5 crashed on every press). Rather
than guess a fifth mechanism, Done is reverted to the one call proven to
close an fpdialog page correctly: a bare `dui page close_dialog`, exactly as
used by every reference plugin (`plugins/SDB/SDB.tcl:3464-3467`,
`plugins/visualizer_upload/plugin.tcl:558-561`), with no fallback page, no
capture, and no post-hoc check of any kind -- restoring the exact
pre-v1.8.2 Done behavior.

Trade-off, stated plainly: this reintroduces the original, much milder bug
this whole chain was trying to fix -- after a flush/rinse/steam
interruption while sitting in Grind Advisor settings, Done may occasionally
land on that transient screen instead of leaving settings cleanly. That is
a stale-page navigation nuisance, not a crash or data-safety issue, and it
matches the exact behavior every other plugin using `-type fpdialog` +
`dui page close_dialog` has on this tablet (since none of them special-case
it either). Fixing that original issue for real would require finding the
correct fpdialog-aware "return to a specific page" call for this DE1app
build (not `dui page load`), which needs on-tablet investigation or a
DE1app source reference this workspace doesn't have -- not another guess.

Not changed: recommendation math, SDB reads, popup content/layout, settings
semantics, or the v1.8.2 popup guards.

## v1.8.5 (diagnostic + bugfix, converges v1.8.2-v1.8.4 navigation attempts)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched; this pass only replaces the Done/Back
navigation mechanism on every Grind Advisor settings page.

### Comparative diagnosis

`plugins/Graphical_Flow_Calibrator` (GFC) handles the identical
flush-interruption-while-in-settings scenario correctly on the same tablet.
Reading its code in full:

* **Page creation:** GFC's pages ("GFC", "gfc_reset_page") are plain
  top-level skin pages (`dui add ... $page_name ...`). They are never
  registered `-type fpdialog`, so GFC has no "dialog" concept at all -- no
  `open_dialog`/`close_dialog`.
* **Return-page capture:** GFC renames the global `::page_show` and wraps
  it: `if {$page_to_show == "GFC"} { set ::gfc_start_page [dui page current] }`.
  This fires only on a genuine `page_show GFC` call (the actual button
  press), never on a machine-state interruption replay. It starts from a
  literal default, `set ::gfc_start_page extensions`, set once at file load.
* **Done ("Exit") implementation:** `proc exit {} { ...; dui page load
  $::gfc_start_page; ... }` -- one bare, uncaught call to `dui page load`
  on a plain variable. No dialog-close call, no post-hoc "is this page
  transient" re-check, no guard machinery of any kind.
* **Catch blocks:** none around GFC's navigation call -- failures would
  surface visibly rather than being swallowed.

Grind Advisor's Done, by contrast, used `dui page close_dialog` (an
fpdialog-specific primitive GFC's plain pages never touch) and then layered
recovery logic on top of it across three patches:

* v1.8.2: no return-page handling; Done could land on whatever transient
  flow screen (e.g. flush) the framework's fpdialog bookkeeping considered
  "current" after an interruption.
* v1.8.3: added a captured return-page plus a transient-page-rejection
  fallback (`dui page close_dialog` then, if still transient,
  `dui page load $fallback`). The captured fallback value was not
  guaranteed to be a real static page; on this tablet it made the app's
  page-name resolver fall through to its plugin-loading machinery, which
  then errored trying to load an unrelated plugin, `A_Flow`.
* v1.8.4: replaced the fallback target with a static string but wrapped the
  call in a bare `catch {...}` with no error variable and no logging. The
  underlying `dui page load GrindAdvisor_settings` call -- reaching an
  fpdialog-type page the wrong way -- apparently failed, and the failure
  was invisible: Done appeared to do nothing.

**Root cause:** Grind Advisor never adopted the one thing that makes GFC's
Done reliable -- a plain, unconditional `dui page load $return_page` call
with no dialog-close call and no post-hoc filtering. Instead it kept patching
around `dui page close_dialog`, and its final iteration hid the real error
behind a swallowing catch.

### Fix (convergence, not another patch)

* Removed all v1.8.3/v1.8.4 navigation-guard machinery: `_transient_page_re`,
  `_is_transient_page`, `_recover_from_transient_page`,
  `_close_settings_dialog`, `_close_subpage_dialog`. The v1.8.2 popup
  validation gates and `popup_active` guard (`_overlay_exists`,
  `_popup_blocked`, `_flow_active`, `_on_settings_page`, `_install_nav_watch`,
  `_nav_state_change`, `_nav_page_change`) are untouched -- they are separate
  and were never broken.
* Added `_settings_return_page`, defaulting to the literal static page
  `"extensions"` -- same style of page reference as `::gfc_start_page`.
  `preload_settings_page` captures it unconditionally (`dui page current`,
  no transient filtering) at genuine plugin entry, matching GFC's wrapped
  `::page_show` capture.
* Added `_exit_settings` (main settings Done) and `_exit_subpage` (every
  sub-page Done): each does exactly one `dui page load <page>` call -- the
  same proc GFC's Exit uses -- with a narrow catch that captures the error
  and logs it via `msg` (matching the `msg "ERROR ...: $err"` logging
  convention already used elsewhere in this codebase and in
  plugins/SDB/SDB.tcl), so a navigation failure is now visible in the log
  instead of silently swallowed.
* All six `page_done` procs now call `_exit_settings` or `_exit_subpage`
  instead of `dui page close_dialog`.

Not changed: recommendation math, SDB reads, popup content/layout, settings
semantics, or the v1.8.2 popup guards.

## v1.8.4 (bugfix)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched; this pass only replaces the fallback
navigation target used by the top-level settings page's Done button.

Root cause: v1.8.3's `_close_settings_dialog` fell back, when the page left
by `dui page close_dialog` still looked transient, to a page name captured
by `_capture_settings_return_page` from `dui page current` at plugin preload
time. That captured value is not guaranteed to be an ordinary static page --
on at least one tablet it produced a value that, when passed to
`dui page load`, made the app's page-name resolver fall through to its
plugin-loading machinery (treating the unrecognized name as a plugin to
enable). That machinery then tried to load an unrelated, already-installed
plugin, `A_Flow` (apparently the first match it iterates to), which failed
with "parent namespace doesn't exist" because nothing had actually asked for
that plugin. Grind Advisor's own sub-page Done buttons (`_close_subpage_dialog`)
were never affected because their fallback was already the literal static
string `GrindAdvisor_settings`, never a captured value -- confirming the
captured/computed page name was the defect, not the recovery mechanism
itself.

Reference check: `SDB_settings::page_done` (plugins/SDB/SDB.tcl:3464-3467)
and `visualizer_settings::page_done` (plugins/visualizer_upload/plugin.tcl:
558-561) both leave their settings dialog with exactly one call,
`dui page close_dialog`, and never compute or store a return-page value at
all.

Fix:
* Removed the v1.8.3 dynamic-capture mechanism entirely: deleted the
  `_settings_return_page` variable, `_capture_settings_return_page` proc, and
  its call from `preload_settings_page`.
* `_close_settings_dialog` (main settings Done) now calls
  `dui page close_dialog` -- the exact same call the reference plugins use,
  with no arguments -- and, only if the revealed page still matches
  `_is_transient_page`, force-navigates to the literal static page name
  `GrindAdvisor_settings` via `dui page load GrindAdvisor_settings`. This
  string is hardcoded (never a variable), already registered by this plugin
  in `preload_settings_page`, and is the same fallback `_close_subpage_dialog`
  already used safely for every sub-page. No empty or computed page name can
  ever reach `dui page load`, and no call passes through any plugin
  enable/load/settings proc.
* `_recover_from_transient_page` no longer takes a fallback argument; it is
  now hardcoded to the one known-safe target.

Not changed: recommendation math, SDB reads, popup content/layout, settings
semantics, or the v1.8.2 popup guards (`popup_active`, `_nav_page_change`,
`_nav_state_change`), which were re-verified end to end (settings → flush →
stop → Done leaves settings cleanly, no plugin loading, no popup).

## v1.8.3 (bugfix)

Safety status: **no write behavior exists in this version.** No SDB or
history-file code was touched at all in this pass; it only changes how the
plugin's own Done/Back buttons navigate after closing.

Root cause: every Done/Back button calls the framework's generic
`dui page close_dialog`, which reveals whatever page the framework currently
considers "current." The plugin never captured its own notion of "the page I
should return to" — it fully trusted the framework's bookkeeping. When a
flush/rinse/etc. starts and stops while a Grind Advisor settings dialog is
open, the framework's page-show bookkeeping ends up pointing at that
transient flow screen, so `dui page close_dialog` correctly hands the dialog
back control, but the page it reveals is the flush screen instead of wherever
the user actually came from.

Fix (navigation only, no SDB/history changes):
* Added `_capture_settings_return_page`, called once from
  `preload_settings_page` — the genuine-entry hook the framework calls only
  when the user actually opens Grind Advisor from Extensions, never when a
  flow interruption re-shows the already-open settings page. It records
  `dui page current` at that moment into `_settings_return_page`, guarded by
  `_is_transient_page` so a machine-state/flow page can never be captured.
* Added `_close_settings_dialog` (used by the main `GrindAdvisor_settings`
  Done button): closes the dialog normally, then checks `dui page current`;
  if it still looks transient (espresso, steam, hot water, water, rinse,
  flush, clean/cleaning, descale, purge), it force-navigates to the captured
  `_settings_return_page` instead. No page name is hardcoded/guessed — the
  fallback is always a real page the user was actually on.
* Added `_close_subpage_dialog` (used by Advanced, Help, History Display
  Options, Diagnostics, Calculation Details): same recovery, but the fallback
  is `GrindAdvisor_settings` itself — a page this plugin owns, not a guess —
  since every sub-page's logical parent is the main settings page.
* Confirmed the v1.8.2 popup guards are unaffected: `_nav_page_change` and
  `_nav_state_change` still trace the same variables/procs, and the extra
  `dui page load` call in the recovery path only fires when navigation was
  already going to happen, so `popup_active` still self-heals exactly as
  before. A flush during settings still produces no popup.

Not changed: recommendation math, SDB reads, popup content/layout, or
settings semantics.

## v1.8.2 (bugfix)

Safety status: **no write behavior exists in this version.** SDB is opened
read-only / SELECT-only, exactly as in v1.8.1; this pass adds read-side
filtering only.

Root causes:
* **Bug 1 (rinse/flush/steam/water triggered the popup):** the shot-complete
  hooks (`after_flow_complete_add` / `on_major_state_change_add` / the
  machine-state trace) fire on *any* completed flow, not just espresso, and
  `analyze_latest_shot` treated the newest SDB row as a valid shot without
  checking what kind of flow produced it.
* **Bug 2 (stuck between rinse and the popup):** there was no popup re-entry
  guard and no reset on navigation/state changes, so a popup left open (or a
  pending delayed popup) could still be showing or firing when a rinse
  started, and nothing tore it down or cancelled it.

Fix (read-only, no SDB write changes):
* Added a real-espresso-shot validation gate (`_row_is_valid_espresso`) used
  by both the automatic path and history: rejects rinse, flush, backflush,
  clean/cleaning, descale, hot water, water, steam, skip, dummy, calibration
  rows (via profile title / beverage type columns, when present), shots under
  5 seconds, and rows missing grind, dose, or set/target yield (only checked
  when those columns exist in the schema).
* Added a `popup_active` re-entry guard: `present_result`/`_show_overlay_dialog`
  set it, `_close_dialog` clears it, and it self-heals to 0 whenever no popup
  widget is actually on screen — so a stuck flag can never permanently block
  the UI.
* Added a navigation/state watch (`_install_nav_watch`) that traces
  `::de1(state)` and any `page_display_change`-style proc: starting a flow
  (rinse/steam/water/clean/etc.) cancels a pending auto-popup and force-closes
  any open popup; any page change also resets a stuck guard flag.
* The automatic popup is suppressed while the current page/context looks like
  a settings/extension page (`_on_settings_page`); the recommendation is still
  computed and saved silently so "Show Latest Recommendation" reflects it.
* `show_latest_recommendation` and `test_latest_shot` now search the filtered
  (valid-espresso-only) row list, so they always find the latest real shot and
  skip rinse/flush/steam rows, and they seed the dedup id so the same shot
  cannot pop again from a later flow event.
* `load_last_recommendation` seeds `last_shown_id` at startup so a fresh
  restart cannot re-pop an old saved recommendation on the next rinse/state
  change.

Not changed: recommendation math, settings UI (aside from the version display
value), SDB connection/table/column discovery, or the popup layout/theme.
