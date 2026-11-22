# MACD with HTF Panels v2: Pine review and ChartScript port

Files: `MACD_HTF_Panels_v2.chartscript.txt` (paste into the Dhan ChartScript editor, then run Verify).

The port was written against the ChartScript documentation (compiled 26 Oct 2026). It has **not** been compiled; no ChartScript compiler was available here. Run Verify first and send back any errors.

## 1. Review of the Pine v6 script

The overall design is solid: closed-bar mode with `lookahead_on` and skipping index 0 is the correct non-repainting idiom, drawing budgets are respected, and everything is redrawn on the last bar.

Bugs and weak spots found:

| # | Issue | Effect | In the port |
|---|-------|--------|-------------|
| 1 | 1-panel auto height is `panel_height * 6` while 2/3/4 panels share `3*H + 2*gap` | With defaults, one panel is about 1.9x taller than the whole 3-panel stack | Fixed: 1 panel uses the same stack budget |
| 2 | In 4-panel mode the HTF3/HTF4 titles are right-aligned at the left edge of the right column | With a small column gap the titles cover the left-column panels | Fixed: right-column titles sit to the right of their panel |
| 3 | `math.min(0.0, mS.min(), sS.min(), xS.min())`: if any array min is `na`, the whole range falls back to -1..1 | The histogram can spill out of the panel when the HTF has little history | Fixed with `nz()` |
| 4 | Zone levels are fixed at ±20 / ±5 in raw MACD units | They mean something only where MACD sits near that size (e.g. NIFTY intraday). On a ₹100 stock they are off-scale; on BANKNIFTY daily they are inside the noise | Kept as is; consider normalising MACD (for example as % of price or in ATR units) |
| 5 | Panels are anchored to `close` | They move up and down with every tick | Not applicable: the port anchors them to the MACD zero line |
| 6 | Live mode (`Closed HTF Bars Only` off) | Crosses can fire and then undo within the HTF bar, so `alert()` can fire more than once per HTF bar (once per chart bar). This is documented in the tooltip | Same behaviour |
| 7 | All four `request.security` calls run even when fewer panels are shown | Wasted fetch and compute only | Same, kept simple |

Repainting: closed-bar mode does not repaint. Live mode repaints by design, as the tooltip says. No other repaint sources were found. Every `ta.*` call runs unconditionally.

## 2. What changed in the ChartScript port

| Pine feature | ChartScript port |
|---|---|
| `//@version=6`, `const` declarations | `//@version=1`; `const` is not a ChartScript keyword, so typed / plain declarations are used |
| `line.new` / `label.new` / `box.new` with `force_overlay=true` (panels on the price chart) | **Approximated.** ChartScript documents `force_overlay` only for plots, shapes, bgcolor, tables and polylines. The panels are drawn in the MACD pane, to the right of the last bar. The size unit is now the chart MACD's peak absolute value over 100 bars (instead of ATR), and the vertical centre is measured from zero. New defaults: height 0.6, gap 0.1, centre 0 |
| `request.security` returning 7 arrays, with `calc_bars_count` | **Rebuilt.** ChartScript `request.security` returns series and tuples, with no documented array returns and no `calc_bars_count`. HTF time, close time, OHLC and MACD come back as a 9-value tuple, and chart-side typed arrays (one set of 9 per HTF) add one entry per HTF bar. A UDT holding the arrays failed Verify ("Cannot infer the type of 'b.t'"), so every array is now declared with an explicit type and used through `array.*()` calls |
| `lookahead_on` + skip index 0 for closed bars | **Re-done.** ChartScript `lookahead_off` already returns only confirmed HTF values on history; the live bar shows the forming HTF bar. Closed mode drops the newest entry until its HTF `time_close` is at or before the chart bar's open, so history and live behave the same way |
| `var` drawing arrays inside `renderPanel` (one copy per call site) | Replaced: on each last-bar run the script deletes all of its own lines, labels and boxes (`line.all`, etc.) and redraws. This works whether or not drawings roll back on live ticks |
| Zone `hline()`s with input value, colour, style and width | Changed to `plot()` + `fill()`, because `hline` arguments are documented as const. The value and colour inputs are kept; **style and width per zone are now set in the Style tab** (the 8 style/width inputs were removed) |
| Signal table with `force_overlay=true` | Same (documented for tables) |
| `alertcondition()` ×4 and dynamic `alert()` | Same; `{{ticker}}` is supported |
| HTF ladder 5…12M, labels via `timeframe.in_seconds` | Same, with "D/W/M" instead of "1D/1W/1M". Added the input **Allow 3M / 6M / 12M** in case the platform rejects multi-month timeframes |

## 3. Things to check on Dhan

1. **Verify compile.** Fixed so far: UDT array fields (type inference). Still unconfirmed: `array.copy(line.all)` and `time_close` inside `request.security`.
2. **Timeframes.** The docs only show "5", "60", "D", "W", "M". If "240", "3M", "6M" or "12M" are rejected, turn off the multi-month input or set the HTFs manually. NSE has about 1.6 four-hour bars per session, so a 240-minute panel looks uneven on NSE.
3. **History depth.** ChartScript keeps 5,000 chart bars, and the ring buffer only collects HTF bars that fall inside them. A 1H chart (HTF up to M) gets about 34 monthly bars, which is enough. If chart history is short, the higher panels show fewer bars. The MACD values are still correct, because they are computed on the HTF series itself.
4. **Panel placement.** If you want the panels back on the price chart: when Verify accepts `force_overlay=true` on `line.new`/`label.new`/`box.new`, the panels can go back there with ATR sizing. I can make that version.

## 4. Panels on the price chart (update)

There are now three files:

- `MACD_HTF_Panels_v2.chartscript.txt` (**try this first**): one script. The panels are drawn on the price chart with `force_overlay=true` on `line.new` / `label.new` / `box.new`, sized in ATR units around the last close, as in TradingView. The defaults are back to the originals: height 6, centre +2, gap 0.6, ATR 14. Dhan accepted `force_overlay` on the table, so it may accept it here too, but the docs don't list it for lines, labels or boxes.
- If Verify rejects `force_overlay` on those calls, use these two together:
  - `MACD_HTF_Panels_v2_pane.chartscript.txt`: the previous version (MACD pane, table, alerts). Turn **Enable HTF Side Panels** off.
  - `MACD_HTF_Panels_v2_overlay_companion.chartscript.txt`: an `overlay=true` script that draws only the 4 HTF panels on the price chart. Keep its MACD, HTF and closed-bar inputs the same as the main script's. It uses one more of the 5 custom-script slots.
