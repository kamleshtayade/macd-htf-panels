# Performance review: MACD HTF Panels v2 (ChartScript)

Scope: `MACD_HTF_Panels_v2.chartscript.txt` (main), the pane version and the overlay companion, as on GitHub commit `d0eaf82`.
Reference: ChartScript Complete Documentation (26 Oct 2026). Sections are cited as [Doc: section].
This is review only; no code has been changed.

## Where the cost is today

| Phase | What runs | Cost |
|---|---|---|
| Full compute (history, ≤5,000 bars) | 4 × `request.security`, 1 chart MACD, 1 ATR, 4 ring-buffer feeds per bar | Low. Everything is O(1) per bar |
| **Every live tick** on the last bar | Delete every line, label and box, then recreate up to ~480 lines, ~120 labels and 4 boxes; rewrite 30 table cells | **High.** This dominates |

The documentation sets the budgets that matter here: 2 s per live tick, ticks slower than 150 ms are paced to 5 s / 30 s / 60 s, 256 KB per live output delta and a 4 MB snapshot. "Every live drawing is sent to every viewer" [Doc: Limitations; Compute Budget Exceeded; Drawing-Object Caps].

## Prioritised changes

### P1 (high impact): stop rebuilding ~600 drawings on every tick
- **Now:** on each tick on `barstate.islast`, the script runs `array.copy(line.all)` and deletes everything, then calls `line.new` ~480 times.
- **Change:** in closed-HTF mode the panel data only changes when an HTF bar closes. Redraw only when something changed: `barstate.isnew`, a new entry in any ring buffer, or a different `sk`. On the other ticks, do nothing.
- **Catch:** the panels are anchored to `close` (y) and `bar_index` (x), so they move every tick. Anchor y to `close[1]` and `ta.atr(...)[1]` (the last confirmed bar) so the geometry is stable within a bar. This is a small visual change: the panels sit relative to the previous close.
- **Expected impact:** drawing work and network deltas drop from every tick to about once per chart bar. This is the largest win and keeps live ticks well under the 150 ms pacing threshold [Doc: Compute Budget Exceeded, "A slow script is not killed on the first slow tick…"].

### P2 (high): draw the MACD and Signal lines as polylines, not hundreds of segments
- **Now:** each panel draws ~29 MACD and ~29 Signal `line.new` segments, about 232 lines across 4 panels.
- **Change:** one `polyline.new(points, …, force_overlay=true)` per Signal line. For the trend-coloured MACD, draw one polyline per same-colour run (usually 2–8 per panel). `polyline.new` is the one drawing call that documents `force_overlay` [Doc: Built-In Reference → polyline.*; Lines and Boxes → polyline.new]. The cap is 100 polylines [Doc: Limitations].
- **Expected impact:** ~232 drawing objects become ~20–40. The output payload is about 5–10× smaller, and it frees the line budget so `render_bars` no longer has to shrink when candles are on.

### P3 (high, with P1): update an object pool instead of delete and create
- **Change:** create the lines, labels and boxes once (for example at `barstate.islastconfirmedhistory`, whose state survives into realtime [Doc: Bar States]). Afterwards move or recolour them with `line.set_xy1/set_xy2/set_color`, `box.set_lefttop/set_rightbottom` and `label.set_xy/set_text` [Doc: Lines and Boxes → Updating a line]. Hide unused objects with a transparent colour.
- **Expected impact:** no object churn and no `array.copy(line.all)` scan per redraw. It also avoids the oldest-object-dropped behaviour if the cap is ever hit [Doc: Drawing-Object Caps].

### P4 (medium): don't fetch timeframes that aren't shown
- **Now:** all 4 `request.security` calls always run, even with 1–3 panels, or with the panels and table off.
- **Change:** request.security can't be wrapped in a condition, but its timeframe can be an input-derived value. For inactive slots pass `timeframe.period` (or repeat HTF1). "The executor fetches each unique (symbol, timeframe) pair once per computation" [Doc: Other Timeframes and Data → Dynamic requests], so the duplicate pair costs no extra fetch.
- **Expected impact:** with 2 panels, about 50% less HTF fetch and compute on full recalculation. Recalculation runs on start, after input changes and after history reloads.

### P5 (medium): don't run the HTF work twice in the two-script fallback
- **Now:** the pane script and the overlay companion both make the same 4 requests and keep the same buffers. They also use 2 of the 5 custom-script slots per chart [Doc: Limitations → Executions per user].
- **Change:** if `force_overlay` on lines works in Dhan, keep only the main script. Otherwise set the pane script's panel timeframes to the chart timeframe (see P4), so it skips the HTF work it doesn't draw.
- **Expected impact:** halves HTF cost in the fallback setup and frees one script slot.

### P6 (medium): refresh the table only when it changes
- **Now:** the table rewrites 30 cells and builds every string (`str.tostring`, formatting) on every tick of the last bar.
- **Change:** in closed mode, write the cells only on `barstate.isnew` or when an `sk` or a buffer head changed. The docs specifically advise building formatted strings only when needed [Doc: Profiling → "Minimise str.format on every bar"].
- **Expected impact:** small CPU win per tick, and smaller live deltas.

### P7 (low): fit the declaration caps to real usage
- **Now:** `max_lines_count=500` and `max_labels_count=300`. The real need after P2 is roughly lines ≤ 130 (hist + zero + candles), labels ≤ 124 and boxes 4.
- **Change:** lower the caps, for example lines 200, labels 130, boxes 8, polylines 50. The docs: "makes the pruning intentional and keeps the snapshot small" [Doc: Drawing-Object Caps → Fix 1].
- **Expected impact:** small. Mainly a safety margin against the 4 MB snapshot and 256 KB delta limits.

### P8 (low): use plots for cross markers where possible
- **Now:** one `label.new` per panel cross. The docs: "Prefer plots over labels … use plotchar/plotshape instead of label.new" [Doc: Profiling → Prefer plots over labels].
- **Limit:** plots can't sit in the future area where the panels are, so this applies only to the chart-TF dots, which are already a `plot`. For the panels, drawing the dots as markers on the MACD polyline (P2) removes the extra labels.
- **Expected impact:** up to ~120 fewer label objects when there are many crosses.

### P9 (low): compute constant helpers once
- **Now:** `f_ladderStart`, `f_autoHtf` and `f_tfLabel` (string building) run on every bar, although their inputs never change during a run.
- **Change:** compute them once with `var string htf1_tf = …` or inside `if barstate.isfirst`.
- **Expected impact:** small, roughly 5,000 × 4 string builds saved per full compute.

### P10 (low): ring-buffer feed
- `f_feed` does nine `unshift`/`pop` calls on arrays of ≤32 elements, once per new HTF bar. That's fine. Consider one `matrix<float>` per HTF only if P1–P3 are done and the profiler still shows the feed.

## Already done right (no change needed)
- One `request.security` per timeframe, returning a tuple [Doc: Profiling → "Avoid request.security for the same thing twice"].
- Every `ta.*` call (macd, atr, cross) runs at top level every bar. None sits inside a conditional [Doc: Repainting → Series-mode functions].
- No loop grows with `bar_index`. The panel loop is ≤40 iterations, only on the last bar [Doc: Compute Budget Exceeded → cause 2].
- No cumulative `ta.cum`/`obv`-style functions, so no full recompute on every tick [Doc: Compute Budget Exceeded → cause 3].
- History depth: the buffers need only 32 HTF bars, and 5,000 chart bars covers that on every timeframe in the auto ladder.

## How to verify
Use **Profiler mode** (Script menu → Profiler mode) before and after each change. It shows each line's share of runtime and its execution count on the live chart [Doc: Profiling → Profiler mode]. Expect the `renderPanel` and `line.new` lines to dominate today. After P1–P3 they should fall to a few percent.

## Suggested order
P1 + P3 together (biggest win, one refactor), then P2, then P4/P5, then P6–P9.

## Applied to the main script (22 Nov 2026)
The pane and overlay backups are unchanged.

| Item | Status |
|---|---|
| P1 redraw gating | Done. Geometry is anchored to the last confirmed bar (its close and ATR). A `var` key (anchor bar plus the newest usable HTF bar of each slot) decides when to redraw. In closed-HTF mode the panels and table rebuild only when that key changes; live-HTF mode still redraws every tick. Visible change: panels sit relative to the previous bar's close while a bar is forming |
| P2 polylines | Done. Signal is one polyline per panel; the trend-coloured MACD is one polyline per colour run, capped at 18 runs per panel. Lines now cover only the histogram, zero line and candles, so 40 bars per panel fit even with candles on |
| P3 object pool | Not needed after P1. Redraws now happen only on change, and every redraw still deletes all of this script's drawings first, so there are no leftovers whatever Dhan's live-tick behaviour is |
| P4 unused HTF slots | Done. HTF2–4 request the chart timeframe when hidden |
| P5 two-script fallback | Skipped. The backups aren't used |
| P6 table refresh | Done. It's gated by the same key |
| P7 caps | Done: labels 170, boxes 8, polylines 100; lines stay at 500 |
| P8 cross markers | Kept as labels, at most one per HTF cross |
| P9 constant labels | Done. Timeframe labels are `var` and computed once |
