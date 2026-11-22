# MACD with HTF Panels v2: Pine Script and Dhan ChartScript

This is a MACD indicator with higher-timeframe (HTF) MACD panels, a signal table, a confluence count and alerts. The original is Pine Script v6 for TradingView; the port is ChartScript for Dhan / TradingCharts.

| File | What it is |
|---|---|
| `pine/MACD_HTF_Panels_v2.pine` | Original Pine Script v6 (TradingView) |
| `chartscript/MACD_HTF_Panels_v2.chartscript.txt` | **Main Dhan script.** MACD pane, signal table and alerts. The HTF panels are drawn on the price chart (`force_overlay=true`) |
| `chartscript/MACD_HTF_Panels_v2_pane.chartscript.txt` | Fallback: the same script with the panels drawn inside the MACD pane |
| `chartscript/MACD_HTF_Panels_v2_overlay_companion.chartscript.txt` | Fallback companion (`overlay=true`) that draws only the HTF panels on the price chart. Use it with the pane version, with that version's panels turned off |
| `REVIEW.md` | Review of the Pine script and a list of every change made in the ChartScript port |

## Using it on Dhan
1. Paste `chartscript/MACD_HTF_Panels_v2.chartscript.txt` into the ChartScript editor and run **Verify**.
2. If the panels don't show on the price chart (scroll right first, because they start 20 bars after the last candle), use the pane version with **Enable HTF Side Panels** turned off, plus the overlay companion. Keep the MACD, timeframe and closed-bar inputs the same in both scripts.

## Styling
The MACD pane follows the CM MACD style: the MACD line is green above Signal and red below it, Signal is blue, a dot marks each cross, and the histogram uses four shades. All of these colours can be changed in the "Style (CM MACD)" input group.

## Licence
Based on "MACD with HTF Panels" © TheUltimator5, Mozilla Public License 2.0 (https://mozilla.org/MPL/2.0/).
