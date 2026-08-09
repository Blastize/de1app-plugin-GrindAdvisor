# Grind Advisor v3.6.0

Current plugin version: **v3.6.0**.

A DE1app (Decent Espresso) plugin. After every completed espresso shot it
reads your latest shot from **SDB** and shows a popup recommending your next
grind setting. You enter nothing by hand.

<img src="images/popup.png" alt="Grind Advisor after-shot popup" width="620">

Dose, Yield, and Ratio lines appear automatically when those columns exist in
SDB; if they aren't stored, those lines are simply omitted.

## Calibration Curve

**Curve** shows what the recommendation is actually standing on. **Back**
returns to the popup.

<img src="images/calibration_curve.png" alt="Calibration Curve view" width="740">

* **Top panel** — every eligible shot on the current bag, the line the model
  solved, a dashed guide at your target time, and a labelled dashed vertical
  at the recommended grind. On the regression rung those two dashed guides
  cross exactly where the fitted line hits your target time — that crossing
  *is* the recommendation. The latest shot is the larger accent dot.
* **Residual strip** — how far each shot sits from that line. Stems to zero,
  so residuals stacked on one side (a biased fit) are obvious.
* **Grind axis** — ticks at readable grind values (0.5 steps on a typical
  range, 0.1 on a tight one), each running a faint gridline through *both*
  panels, so any point or residual traces straight down to the grind that
  produced it.
* **Caption** — `R² · bias · spread · n`, then a plain-language verdict.

**Opening it:** the **Curve** button on the after-shot popup, or — in skins
that support it, such as Lumen — the **Curve** control on the grind tile,
which goes straight there without the popup. Back returns to the popup either
way.

**Why bias and spread, and not just R²?** R² is correlation, not accuracy. A
line can follow the trend closely and still sit consistently off the points,
and through two points it is 1.000 by construction while telling you nothing.
So a high R² with large residuals is called out rather than left to look like
a good result.

**Under 3 shots on a bag, bias and spread are not shown.** The line is solved
*through* the latest shot at that point, so those numbers would describe the
arithmetic rather than your calibration. The view says so instead of printing
a reassuring zero.

The y axis reads *time (normalized)* when the regression is running — that is
the series it actually fits, with dose and yield differences divided out — and
*shot time* on the 1- and 2-shot rungs, which work in raw time.

**You do not need to pull a shot first.** If the stored recommendation has no
shot data attached (anything saved before v3.1.0), Curve re-reads the current
bag from SDB using the same read-only query the popup itself uses, and plots
the shots you have already pulled. It only reports a problem if SDB genuinely
has no espresso shots for this bag.

## Install

Copy this folder to:

```
de1plus/plugins/GrindAdvisor/
```

so you have:

```
de1plus/plugins/GrindAdvisor/plugin.tcl
de1plus/plugins/GrindAdvisor/GrindAdvisor.tcl
de1plus/plugins/GrindAdvisor/filelist.txt
de1plus/plugins/GrindAdvisor/README.md
```

Then in the app, enable it under **Settings → App → Extensions** (the exact
menu name varies by skin) and make sure **SDB** is also enabled. Pull a shot.

To trigger the popup manually for testing (e.g. from the Tcl console without
pulling a shot):

```tcl
::plugins::GrindAdvisor::test
```

## How the recommendation is computed (v3.0.0 — Regression Forecast)

There is one built-in method, chosen automatically by how many shots of the
**current bag** you have (it resets when the bag changes). There are no
recommendation modes to pick.

* **1 shot:** nudge the grind from the time error —
  `change = (shot_time − target) / 3.0 × 0.5`, then `next = grind + change`.
* **2 shots:** the same, but the seconds-per-grind-step is measured from the
  two shots (`|Δtime / Δgrind|`), falling back to the default 3.0 when the
  grind didn't move.
* **3+ shots:** a recency-weighted least-squares line is fit through every
  shot of the bag (shot time vs grind, recent shots weighted more), then
  solved for the grind that predicts your target time. The grinder's
  direction (does a higher number mean finer or coarser?) is learned from
  the data — you never set it. If there isn't enough grind spread to fit a
  line, it falls back to the 2-shot method and says so.

Before fitting, shot times are **normalized** to remove dose/yield weighing
error (using your scale data when present), and shots whose weighed dose or
yield was more than 2 g off are excluded as outliers. When your shots carry
no scale data, the fit runs on raw times.

Verified against the reference spreadsheet's "Forecast Method" sheet: for its
10-shot bag the plugin reproduces the exact weighted slope (−4.10 s/grind),
intercept, ideal grind (12.97 → **13.0**), and predicted time (27.9 s). A
single fast shot (grind 13.5, time 19.1 s, target 28) recommends **12.0**.

The **Why?** button on the popup and the **Calculation Details** page show
the method used, the learned slope, the model fit (R²), the eligible shot
count, and every intermediate. The possible-channeling warning is
display-only and does not override the recommendation.

## Bag Stats — compare how each bag behaved (v3.5.0, card list v3.6.0)

**Bag Stats** — on the main page's Actions card and under Advanced → Tools —
lists your bags newest first as a card list (same look and paging as the
History list: 5 cards per page, ◀ Prev / Next ▶, up to 25 bags), so bags can
be compared at a glance. Each bag card shows:

* **ideal** — the grind its robust fit predicts for your target time,
* **slope** — the bean's seconds-per-grind (Theil–Sen: the median of all
  pairwise slopes, so one channeled shot can't drag the number),
* **drift** — how the bean's shot time moved per day of bag life (beans
  speed up as they degas; shown with 6+ dated shots over 3+ days),
* **on target in N shots** — how quickly the bag was dialed in (±2s),
* shot count and excluded outliers.

A bag only gets numbers when the fit is trustworthy (4+ eligible shots and
at least 1.0 of grind spread); otherwise it says "fit not reliable" with the
reason — shots pulled all at one grind setting can't calibrate anything.
The header shows the **median bean slope** across reliable bags (the number
that carries between bags) and the average R² as an overall fit-quality
gauge. R² itself is per bag and resets when the bag changes.

Long text pages (Help / Guide, Calculation Details, Diagnostics) paginate
with ◀ Prev / Next ▶ in the bottom bar instead of clipping when the content
outgrows the card (fixed in v3.6.0).

Everything is computed read-only from your saved shots (normalized time,
outliers excluded) and is display-only — it never changes any
recommendation.

## Defensive SDB handling (read-only)

SDB schemas vary, so the plugin discovers everything at runtime and only ever
runs `SELECT`s:

1. **Connection.** It first reuses an already-open sqlite connection if one
   exists (e.g. `::plugins::SDB::db`, `::db`, …). If none is found it opens its
   **own read-only** connection (`sqlite3 … -readonly true`) to a likely SDB
   file discovered by scanning the data directory for `*.sqlite`, `*.sqlite3`,
   `*.db`, `*.sdb` and ranking by filename.
2. **Table.** It lists tables/views and picks the one whose columns best match
   a shot record (must have a grind column and a shot-time column; a timestamp
   is preferred for ordering).
3. **Columns.** It maps logical fields to real column names with ordered
   regexes:
   * grind → `grinder_setting`, `grind…`
   * time/duration → `extraction_time`, `shot_time`, `espresso_elapsed`,
     `elapsed`, `duration`, … (if the value is the full elapsed-time *series*,
     the maximum is used as the total time)
   * dose → `grinder_dose`, `dose…`
   * yield → `drink_weight`, `final_weight`, `yield`, …
   * timestamp → `clock`, `timestamp`, `date`, …
4. **Query.** It selects the latest one or two shots (ordered by timestamp, or
   rowid if there's no timestamp), ignoring rows missing grind or time.

If no usable table or no valid shot is found, it shows a clear error popup
instead of guessing. It never inserts, updates, deletes, or alters SDB.

## Integration notes (the parts most likely to need tweaking)

These are handled with fallbacks, but if something doesn't fire on your setup,
this is where to look:

* **Shot-complete detection.** It tries, in order: the DE1app event system
  (`::de1::event::listener::after_flow_complete_add` /
  `on_major_state_change_add`), then a trace on `save_this_espresso_shot`, then
  a trace on the machine-state variable. Duplicate triggers are de-duplicated
  by the latest shot's id, so it pops once per new shot.
* **Popup rendering.** Primary path draws an overlay on the skin's shared root
  canvas (portable across skins); it falls back to a Tk toplevel, then to an
  Android toast / log line. The recommendation is always written to the log via
  `msg` regardless.
* **History button.** Tries common skin history-page names
  (`shot_history`, `history_viewer`, `DSx_past`, …). If yours differs, set the
  right page name in `_open_history` in `GrindAdvisor.tcl`.

## Settings

Defaults live in `plugin.tcl` under `::plugins::GrindAdvisor::settings`:

| key                        | default | meaning                                  |
|----------------------------|---------|------------------------------------------|
| `target_time`              | 28.0    | target shot time, seconds                |
| `grind_rounding_increment` | 0.5     | rounding increment for recommended grind |
| `grinder_min`              | 0       | minimum allowed recommended grind        |
| `grinder_max`              | 50      | maximum allowed recommended grind        |
| `popup_delay_ms`           | 1500    | delay after a shot before reading SDB    |
| `popup_theme`              | dark    | `dark` or `light` popup colors           |
| `popup_font_scale`         | 1.0     | multiplier if text looks too big/small   |
| `enable_popup`             | 1       | automatic popup after completed shots    |
| `dose_yield_mode`          | auto    | which dose/yield to report: fixed/actual/auto |

The Regression Forecast method has no tuning knobs — its constants (3.0 s/step
default, 0.5 damping, 0.85 recency decay, 1.8 s/g dose sensitivity, 2.0 g
outlier threshold, 3-shot minimum) are fixed and documented on the Help page.
(Upgrading from v2.x: the old `recommendation_mode`, `default_seconds_per_step`,
`first_cap`, and `later_cap` settings are gone and any stored values are
ignored.)


## v1.2 safe layout patch

This build changes only the popup layout and History button safety:

* Uses a fixed tablet-safe layout for 1340x800-style screens.
* Draws title, stats, recommendation, and buttons in separate vertical areas to avoid overlap.
* Keeps the History button visible but disables navigation to prevent the blank/white screen until the exact skin history page is identified.
* Does not change the recommendation formula.


## v1.3 notes

This build fixes the Android/AndroWish popup sizing issue by using negative Tk font sizes, which makes fonts pixel-sized instead of point-sized. It also replaces the History button behavior: it no longer guesses or opens the skin's history page. Instead, it shows a safe read-only recent-shot list inside the Grind Advisor overlay.

## v1.6 notes

This build keeps the v1.3 safe-history popup and hook lineage, adds Pass 2
Dynamic Barista recommendation modes, and adds Pass 3 UI polish:

* Settings title/version showed v1.6 in this pass; v1.7 moved version text out of the main title.
* Long recommendation mode values have more room on the main settings page.
* History Display Options can hide/show Recommendation Reason and Bag Shot Number.
* Light popup theme uses a light outer overlay.
* Advanced diagnostics show display-only calculation details for the latest recommendation.

## v1.7 notes

This build keeps the v1.6 behavior and focuses on settings usability:

* Main settings title is exactly `Grind Advisor Settings`.
* Numeric settings are ordered near the top of General Options.
* Help / Guide is available from the Actions column.
* Advanced contains editable tuning settings plus buttons for Diagnostics and Calculation Details.
* Diagnostics and Calculation Details open as separate internal pages.

## v1.8 notes

This build keeps the v1.7 behavior and only polishes main settings spacing:

* General Options no longer crowds the Target shot time row.
* General Options rows use more vertical space on 1340x800-style layouts.
* Numeric fields remain near the top for Android keyboard usability.

## v1.8.1 notes

This tiny patch keeps v1.8 behavior and only adds more vertical spacing below
the General Options header on the main settings page.

## v3.0.0 notes — Regression Forecast method (major behavior change)

The four recommendation modes (Conservative / Normal / Dynamic Barista /
Aggressive New Bean) are **removed** and replaced by one built-in Regression
Forecast ladder — see "How the recommendation is computed" above. The mode
selector and the `default_seconds_per_step` / `first_cap` / `later_cap`
settings are gone; upgrading discards any stored values gracefully. The
method uses every shot of the current bag, normalizes out dose/yield weighing
error, learns the grinder direction from the data, and converges as shots
accumulate. Reproduces the reference spreadsheet's Forecast Method numbers
exactly (rec 13.0, slope −4.10). SDB stays strictly read-only; the popup,
navigation, rinse/flush guards, and the v2.x dose/yield source mode and
Calibration Accuracy gauge are unchanged.

## v2.3.0 notes — dose/yield source mode

New Advanced → "Dose / Yield Source": choose whether the plugin reports your
**set** dose/yield, the **measured** (actual) values, or **Auto** (measured
when plausible, else set — the default). Plausibility bounds (dose min/max,
ratio min/max) are editable settings. The popup, reason line, and Diagnostics
always state which source was used ("dose: actual 18.4g", or
"set 18.0g (actual 0.0g rejected)"). Because the grind recommendation is based
on shot time only, this choice changes what's reported and the ratio — never
the recommended grind. If your shots don't record a measured dry dose, dose
gracefully stays on the set value (Diagnostics shows what was detected).

## v2.2.0 notes — per-shot calculation trace (read-only diagnostics)

Calculation Details now ends with a trace of the latest 5 shots showing
every input and intermediate the recommendation used: time/target/error,
dose, yield, bag shot, seconds-per-step + calibration source, cap tier (and
whether it was hit), raw next grind, final recommendation, and reason.
Nothing about the math changed — the trace recomputes through the same
procs the popup uses.

## v2.1.0 notes — Calibration Accuracy gauge (display-only, no write behavior)

Adds a confidence score (0–100) for the current recommendation, shown as a
10-segment bar with "82% — Good" in the Recommendation card, a caption line
in the after-shot popup, and an explanation on the Help page. The score
combines evidence (how many valid same-bag/recipe shots feed the
calibration, full credit at 5) and consistency (average distance of recent
shot times from target; 0s = full, ≥10s = zero), weighted 40/60. Bands:
0–39 Poor, 40–64 Fair, 65–84 Good, 85–100 Excellent. Fewer than 2 relevant
shots shows "Not enough data"; a new bag resets the score like it resets
calibration. Recommendation math outputs are unchanged — the score is
display-only and never feeds back.

## v2.0.2 notes — glyph + Done ping-pong bugfixes (no write behavior)

Fixes two v2.0.1 tablet findings: the history page's Prev/Next buttons (and
a few other strings) showed bare hex like "25C0" because a build-step `sed`
ate the backslashes out of `\uXXXX` escapes — all six strings restored; and
pressing Done on the main page after visiting Advanced bounced back to
Advanced instead of leaving to Extensions — the return-page capture now
ignores the plugin's own pages, so the genuine entry page always wins.
Everything else identical to v2.0.1.

## v2.0.1 notes — section-card visual grouping (behavior identical, no write behavior)

Adopts the stock DE1app "App tab" look: related controls now sit inside
white rounded section cards with titles, on the grey page background, in two
balanced columns. Main page: Shot Settings (numeric entries, kept in the top
half for the Android keyboard) + Actions on the left; Recommendation + Popup
on the right. Advanced: Calculation Tuning + Popup Tuning cards plus a
full-width Tools card. Help/Diagnostics/Calculation Details text sits inside
a full-width card. Every control keeps its exact v2.0.0 command; Done /
Advanced / all navigation calls are byte-identical.

## v2.0.0 notes — complete UI redesign (behavior identical, no write behavior)

Full facelift on the design system proven in ShotHistoryEditor: one layout
token block (virtual 2560x1600 coordinates, physical-pixel fonts, shared
button font/style, rounded cards). Main settings is now a clean label/value
grid with Done/Advanced in the bottom bar and the two actions (Show Latest
Recommendation, History) in the toolbar; History Display Options,
Diagnostics, Calculation Details, and Help moved under Advanced. The
after-shot popup is a centered rounded card with the grind change as the
hero number, detail rows, reason block, and [OK] [Why?] [History] — Why? is
a new display-only explainer of the already-computed recommendation. The
history list is a standard 5-card page with Prev/Next. Light/dark popup
themes share one layout. Behavior is identical to v1.8.8: recommendation
math, SDB reads (read-only), hooks, validation gates, popup guards, and
Done/Back navigation are unchanged; the navigation fix's temporary DIAG log
lines are removed.

## v1.8.8 notes — Done navigation fixed, root cause confirmed from app source

The actual DE1app source (`de1app-core/`) finally made the bug explainable
instead of guessable. In `de1app-core/dui.tcl`, `::dui::page::load` resets
the internal `page_stack` to a single entry whenever a normal (`default`
type) page loads -- which is exactly what happens when a flush/rinse/steam
flow screen appears -- even while an `fpdialog` settings page is open (the
"only one dialog visible" guard only checks type `dialog`, not `fpdialog`).
When the app re-shows GrindAdvisor's settings after the flow ends, the stack
reads `{flow_page, GrindAdvisor_settings}`, so `dui page close_dialog`
"returns" to the flow page. That is the whole bug.

The fix mirrors how Graphical_Flow_Calibrator survives the same scenario,
adapted to fpdialog pages: every GrindAdvisor page's `show{}` callback
captures the real page it was opened from (ignoring machine-state flow pages
so an interruption can never overwrite it), and Done navigates to that
captured, existence-verified page via `dui page load` -- the same underlying
call the framework's own `open_dialog` uses -- falling back to plain
`dui page close_dialog` whenever nothing valid was captured. Navigation
errors are logged, never swallowed. One `DIAG Done -> ...` log line remains
for tablet verification and will be removed next pass. Recommendation math,
SDB reads, popup logic, and settings semantics are unchanged.

## v1.8.7 notes — diagnostic logging only, no behavior change

Confirmed on-tablet: v1.8.6 fixed the crash, and the original, milder bug is
back as expected -- after a flush interruption while in settings, Done lands
on the flush screen instead of leaving settings. Since three fix attempts in
a row each guessed wrong, this version adds read-only `msg` diagnostic
logging (`GrindAdvisor: DIAG ...`) at plugin entry, at every machine-state /
page-display change, and immediately before/after `dui page close_dialog` in
Done. No navigation behavior changed. **To help find the real fix:**
reproduce the bug (settings → flush → stop → Done lands on flush), then
check the DE1app log for the `GrindAdvisor: DIAG` lines around that sequence
and share them -- that will show what page the framework considers
"current" at each step, which is what's needed to fix this for real instead
of guessing again.

## v1.8.6 notes — Done reverted to plain `dui page close_dialog` (no SDB write behavior)

v1.8.5's fix crashed with the same DE1app "A_Flow" plugin-load error on
*every* Done press, not just after a flush. Root cause: `dui page load`
does not correctly exit an fpdialog-type page on this DE1app build --
GrindAdvisor's settings pages are all `-type fpdialog`, while
Graphical_Flow_Calibrator's pages (whose pattern v1.8.5 copied) are plain
top-level pages, so its `dui page load` approach doesn't transfer here.

Since three different navigation "fixes" in a row (v1.8.3, v1.8.4, v1.8.5)
each broke Done in a new way, this version reverts to the one mechanism
proven to work reliably for fpdialog pages: a bare `dui page close_dialog`,
exactly as every reference plugin (SDB, visualizer_upload) uses it, with no
fallback page, no capture, and no extra logic. This restores the original
(pre-v1.8.2) Done behavior: normal use works cleanly again, though the
original narrow stale-page-after-a-flush-interruption issue may resurface in
that specific case. That is a minor navigation nuisance, not a crash, and is
far preferable to the last three attempts. Fixing the original interruption
case for real needs on-tablet investigation of the correct fpdialog-aware
navigation call before trying again. Recommendation math, SDB reads, popup
content, and the v1.8.2 popup guards are unchanged.

## v1.8.5 notes — Done/Back navigation converged with a working reference (no SDB write behavior)

v1.8.2 through v1.8.4 each patched Done/Back navigation differently (stale
flow screen, then a plugin-load crash, then a silent no-op). This pass
throws all of that away and instead copies the exact pattern already proven
to work for this scenario in `plugins/Graphical_Flow_Calibrator`: a plain
variable holding the page to return to (defaulting to the literal page
`extensions`, captured unconditionally when the user genuinely opens Grind
Advisor), and Done doing one unconditional `dui page load <page>` call --
no `dui page close_dialog`, no post-hoc "is this page transient" check, no
silently swallowed errors (navigation failures are now logged). Every
Done/Back button in the plugin (main settings, Advanced, Help, History
Display Options, Diagnostics, Calculation Details) uses this same mechanism.
Recommendation math, SDB reads, popup content, and the v1.8.2 popup guards
are unchanged.

## v1.8.4 notes — Done fallback plugin-load bugfix (no SDB write behavior)

This patch fixes a bug introduced by v1.8.3: pressing Done on the main
settings page after a flush interruption could show a DE1app error about an
unrelated plugin ("A_Flow") failing to load.

Root cause: the v1.8.3 fallback navigated, in the rare case where closing
the dialog still left a transient flow screen on view, to a page name it had
captured from the app at plugin-preload time. That captured value was not
guaranteed to be an ordinary page, and on at least one tablet it made the
app's page loader fall through to its plugin-enable machinery instead.

Fix: that captured/computed fallback is gone. The main settings Done button
now closes exactly like every reference plugin's Done button
(`dui page close_dialog`, no arguments), and only force-navigates, if still
needed, to the literal static page name `GrindAdvisor_settings` -- the exact
same known-safe fallback already used by every sub-page's Done button. No
page name is ever captured, computed, or empty; no call can reach plugin
enable/load machinery. Recommendation math, SDB reads, popup content, and
the v1.8.2 popup guards are unchanged.

## v1.8.3 notes — Done/Back stale-page bugfix (no SDB write behavior)

This patch fixes a bug where, after starting and stopping a flush (likely
also rinse/steam/hot water) while sitting on any Grind Advisor settings page,
pressing Done would land on the flush screen instead of leaving settings.

Root cause: Done/Back buttons relied entirely on the framework's generic
`dui page close_dialog` to reveal the right page. That framework bookkeeping
gets pointed at the transient flow screen by the interruption, so closing
the dialog correctly closes it but reveals the wrong page underneath.

Fix: the main settings page now captures the real page it was opened from
(only at genuine entry from Extensions, never re-captured mid-visit), and
every Done/Back button checks the page left on screen after closing; if it
still looks like a machine-state/flow screen, it force-navigates back to the
correct page instead — either the captured entry page (main settings) or the
main settings page itself (every sub-page). No page name is guessed; the
fallback is always either a real page the user was actually on or a page
this plugin owns. Recommendation math, SDB reads, popup content, and the
v1.8.2 popup guards are unchanged.

## v1.8.2 notes — rinse/flush bugfix (no SDB write behavior)

This patch fixes two bugs and does not touch recommendation math, settings
semantics, or SDB write behavior (still read-only / SELECT-only):

* **Non-espresso rows no longer trigger anything.** A validation gate rejects
  rinse, flush, backflush, clean/cleaning, descale, hot water, water, steam,
  skip, dummy, and calibration rows (detected from profile title / beverage
  type columns when the schema has them), shots under 5 seconds, and rows
  missing grind, dose, or target yield. This applies to the automatic popup,
  history, and "Show Latest Recommendation".
* **No more stuck screens.** A popup re-entry guard (`popup_active`) stops a
  second popup from opening, self-heals if the popup widget disappears, and
  is force-reset whenever the machine starts a flow (rinse/steam/water/etc.)
  or the user navigates to a different page — so a stuck flag can never
  permanently block the UI.
* **Automatic popup is suppressed on settings pages.** If a shot completes
  while you're in a settings/extension page, the recommendation is computed
  and saved silently; open "Show Latest Recommendation" to see it.
* **"Show Latest Recommendation" now really means latest.** It searches
  backward through the filtered, valid-espresso-only shot list, so it always
  skips rinse/flush/steam rows instead of grabbing the newest SDB row
  regardless of type.
