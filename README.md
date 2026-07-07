# Grind Advisor v2.0.2

Current plugin version: **v2.0.2**.

A DE1app (Decent Espresso) plugin. After every completed espresso shot it
reads your latest shot from **SDB** and shows a popup recommending your next
grind setting. You enter nothing by hand.

```
┌───────────────────────────────────┐
│            ✓ Shot Saved           │
│                                   │
│           16.0 → 15.0             │
│           finer by 1.0            │
│                                   │
│   Time        22.4s (target 28s)  │
│   Dose        19.0g               │
│   Set Yield   38.2g               │
│   Bag Shot    #4                  │
│                                   │
│   Reason                          │
│   Fast shot, using default        │
│   estimate.                       │
│                                   │
│  [ OK ]   [ Why? ]   [ History ]  │
└───────────────────────────────────┘
```

Dose and Yield lines appear automatically when those columns exist in SDB; if
they aren't stored, those lines are simply omitted.

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

## How the recommendation is computed

Convention: **higher grind number = coarser = faster shot; lower = finer =
slower shot.**

```
next_grind = current_grind - ((target_time - actual_time) / seconds_per_step)
```

* `target_time` defaults to **28.0 s**.
* **First shot / no calibration:** `seconds_per_step = 5.0`, adjustment capped
  to **±1.0**.
* **With a previous shot:** `seconds_per_step = abs(time_change / grind_change)`,
  adjustment capped to **±0.75**. (If the grind didn't change between the two
  shots, calibration isn't possible, so it falls back to the default step and
  the ±1.0 cap.)
* The result is rounded to the nearest **0.5**.

Worked example (no previous shot): grind 16.0, time 22.4 s, target 28.0 s.

```
adjustment = (28.0 - 22.4) / 5.0 = 1.12  → capped to 1.0
next       = 16.0 - 1.0       = 15.0     → rounds to 15.0
```

The shot was faster than target, so it recommends a **finer** (lower) grind.

> Note: the layout sketch in the original brief shows `15.5` for these inputs.
> With the specified caps and 0.5 rounding the same inputs yield `15.0`; the
> plugin follows the stated formula and rules exactly. Adjust the caps in
> `settings` if you want a different feel.

Not implemented (by design): prediction model, smart history, Actual Dose.
The possible-channeling warning is display-only and does not override the grind
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
| `default_seconds_per_step` | 5.0     | assumed s/grind-step before calibration  |
| `first_cap`                | 1.0     | max first adjustment, ± grind units      |
| `later_cap`                | 0.75    | max calibrated adjustment, ± grind units |
| `popup_delay_ms`           | 1500    | delay after a shot before reading SDB    |
| `popup_theme`              | dark    | `dark` or `light` popup colors           |
| `popup_font_scale`         | 1.0     | multiplier if text looks too big/small   |
| `enable_popup`             | 1       | automatic popup after completed shots    |
| `recommendation_mode`      | dynamic_barista | Conservative, Normal, Dynamic Barista, or Aggressive New Bean |
| `grinder_min`              | 0       | minimum allowed recommended grind        |
| `grinder_max`              | 50      | maximum allowed recommended grind        |
| `grind_rounding_increment` | 0.5     | rounding increment for recommended grind |
| `history_show_reason`      | 1       | show recommendation reason in history    |
| `history_show_bag_shot`    | 1       | show bag shot number in history          |



See [CHANGELOG.md](CHANGELOG.md) for version history.

## License

MIT — see [LICENSE](LICENSE).
