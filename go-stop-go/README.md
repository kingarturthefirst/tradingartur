# Go-Stop-Go: Asia Session Strategy (Pine Script v6)

`GoStopGo_Asia.pine` is a single-file TradingView `strategy()` that mechanically backtests the Go-Stop-Go entry model on NQ1! / MNQ1! 1-minute charts during the 20:00–22:30 ET window, and records research statistics for every trade. Those stats cover depth, expansion, MFE and shadow management results.

## Quick start

1. Open a **1-minute** NQ1! or MNQ1! chart. On any other timeframe the script shows a red warning label and takes no trades.
2. Paste the file into the Pine Editor, save, and click "Add to chart".
3. **Settings → Properties.** Pine does not allow `input.*()` values inside `strategy()`, so these settings live here:
   - Initial capital: default 100,000.
   - Commission: default 2.25 USD per contract per side, which approximates NQ. **Set about 0.62 for MNQ.**
   - Slippage: default 1 tick.
   - Bar Magnifier: optional. It improves real fills; see assumption 3.
4. A few weeks of 1m history is a thin sample. Use Deep Backtesting, if your plan has it, for a meaningful number of trades.

## How it works

```
every bar ─► GO-leg trackers (long + short, always running)
               │  leg completes in the bias direction, flat, nothing pending
               ▼
            Setup: STOP bars collected → re-evaluated every bar
               │  valid STOP + entry window open + limits OK + size OK
               ▼
            stop-entry order + bracket (re-submitted each bar while valid)
               │  fill
               ▼
            Trade record ─► engine manages exits (BE move on 1R)
                         └► research tracker: MFE, expansion, shadow 2R, shadow 1R+BE
```

All decisions are made on bar close. An order submitted on the close of bar *t* can only fill during bar *t+1*.

## Ambiguities in the spec and what I assumed

| # | Spec item | Decision |
|---|-----------|----------|
| 1 | Bias: "`lookahead_off`, apply `[1]`" | These two instructions conflict. `lookahead_off` with `expr[1]` doesn't leak future data, but on **historical** bars it returns the 15m bar *before* the last closed one, while on realtime bars it returns the last closed one. So the backtest and live trading would disagree. I used `request.security(..., [close[1], ema[1]], lookahead = barmerge.lookahead_on)`, which is TradingView's documented non-repainting idiom. It always gives "the last fully closed 15m bar", which is the spec's stated goal. |
| 2 | Commission, slippage and initial capital "(input)" | `strategy()` only accepts constants. These are declaration defaults that you edit in the Properties tab. |
| 3 | "If both stop and target are inside one bar, assume stop first" | A script can't change how TradingView's broker emulator orders prices inside a bar. Without Bar Magnifier it assumes open → nearer extreme → farther extreme → close. **Real fills** (strategy P&L and "actual net R") follow the emulator. **All research simulations** (MFE, expansion, shadow 2R, shadow 1R+BE) apply the stop-first rule. A trade gets the "ambiguous" flag when any bar contains both its stop and a target, or when the stop falls inside the entry bar. The table shows the count. |
| 4 | Contracts formula `(entry − stop distance in points × pointvalue)` | Read as `floor(equity × risk% ÷ (|entry − stop| × pointvalue))`, minimum 1. The setup is skipped if one contract risks more than 2 × budget. Commission is not part of the risk budget. "Max contracts" (default 0 = off) is optional. |
| 5 | Which ATR | Every ATR-normalised test for a setup uses the ATR(14) of the bar **just before the GO leg started**. That ATR is what "recent price action" means, and it isn't inflated by the leg's own big candles. This covers leg size, the small opposite candle, STOP tolerance and STOP height. |
| 6 | GO leg mechanics | A leg starts on a same-direction candle. Every leg bar must make a new extreme. One small opposite candle (body < 0.30 × ATR, dojis included) is allowed if it still makes a new extreme. The leg ends on the first bar that fails to make a new extreme, or on a disallowed opposite candle. **That bar is STOP bar 1.** The leg's size is its own extreme low to extreme high. |
| 7 | "Every high used to define the top level is within k × ATR" | A **touch** is a separate visit to the band within k × ATR of the STOP's highest high (or, on the other side, its lowest low). Consecutive bars inside the band count as one touch, so "touched twice" means two separate visits. Horizontality is enforced by the regression test: `|slope| × bars ≤ k × ATR`, run separately for highs and lows. The spec requires only the breakout level to be touched twice. "Min touches of opposite level" defaults to 1, which turns that filter off. |
| 8 | STOP that stops being horizontal while an order is working | The order is pulled. If the STOP qualifies again before it dies, the order is re-submitted. The spec's cancel rules end the setup for good: opposite side broken, bias flip, 22:30, or limits. |
| 9 | Setup lifetime | A STOP that has not yet confirmed can extend in both directions. A **confirmed** STOP dies if its opposite side breaks. A setup also dies if price retraces through the GO leg's start (depth > 100 %), if it exceeds max STOP bars, or if a newer GO leg replaces it (only when no order is working). Pattern detection runs at all hours, so a GO + STOP that forms at 19:50 can get an order at 20:00. Only setups that are valid while the window is open count as "armed" in the skip statistics. |
| 10 | Stop-loss swing | Uses the most recent pivot inside the STOP. Left bars must be strictly higher than the pivot low (long), right bars not lower. If no pivot exists, the STOP extreme is used. The stop tracks the newest pivot while the order is working, so the order size can change between bars. |
| 11 | Session limits | Trades are counted when they fill. A loss counts toward the session in which the trade was **opened**. A trade from a previous window that closes later doesn't count. "Win" means net R > 0 and "loss" means net R < 0, both after commission and slippage. |
| 12 | Gap "filled" | Treated as filled when a bar reaches Friday's close (touch). "Friday's close" is the close of the last bar before the Sunday reopen, so holiday-shortened Fridays work. An unfilled gap keeps overriding the EMA until it fills or the next Sunday. You can also set it to expire after N sessions (default 0 = never). |
| 13 | Day type with short history | The 18:00–20:00 range average uses the previous `lookback` sessions. Before at least 5 sessions exist, days count as Safe unless a gap is active. |
| 14 | 50 % partial with 1 contract | Half of 1 contract rounds to 0. In that case the whole position goes to the runner target, and the stop still moves to breakeven once 1R trades. The shadow 1R+BE result always assumes an exact 50/50 split. |
| 15 | Breakeven timing | The stop moves to breakeven after a bar **closes** having traded through 1R, so the new stop is active from the next bar. The research simulation uses the same timing. BE is the actual average fill price (live) or the planned entry (simulation). |
| 16 | MFE and expansion horizon | Both are measured from entry until the **original** stop is hit or 23:59 ET, whichever comes first. A bar that hits the stop doesn't contribute its favourable extreme (stop-first rule). Expansion = (best price − STOP low) ÷ GO leg size for longs, and the mirror image for shorts. |
| 17 | Shadow horizon | Each shadow runs until its own stop or target is hit. With "Flatten at window end" on, an unresolved shadow is marked to market at the close of the last window bar, matching what the real position does. |
| 18 | Margin | `margin_long` and `margin_short` are set to 0. The v6 default of 100 % would reject a single NQ contract on a 100k account. Real futures margin isn't modelled; use "Max contracts" as a ceiling if you need one. |

## Inputs

### Session
| Input | Default | What it does |
|---|---|---|
| Entry window (ET) | 2000-2230 | Entry orders can only be working inside this window. Anything unfilled is cancelled at the end. |
| Day-type range window (ET) | 1800-2000 | Range used for the Momentum / Safe classification. |
| Flatten at window end | false | Off: open positions run to their stop or target. On: closed at market at window end. |
| Max trades per session | 3 | Counts fills. |
| Losing trades that end the session | 2 | No new orders after this many losses with net R < 0. |

### Risk
| Input | Default | What it does |
|---|---|---|
| Risk per trade (% of equity) | 1.0 | Risk budget = `strategy.equity × %`. |
| Skip if 1 contract risks more than × budget | 2.0 | Size guard. Skips are logged and counted. |
| Max contracts (0 = no cap) | 0 | Optional hard ceiling on size. |

### Bias
| Input | Default | What it does |
|---|---|---|
| Bias timeframe | 15 | Higher-timeframe bar used for the bias. |
| Bias EMA length | 50 | Last closed 15m close above the EMA = longs only, below = shorts only. |
| Sunday-gap override | true | Enables gap mode. |
| Major gap threshold (points) | 100 | Sunday open vs Friday close. |
| Gap override expires after N sessions | 0 | 0 = the override lasts until the gap fills or the next Sunday. |
| Plot bias EMA | false | Draws the closed-bar 15m EMA as a step line, for checking. |

### GO Leg
| Input | Default | What it does |
|---|---|---|
| ATR length (1m) | 14 | ATR used for all normalisation. |
| Min leg size (× ATR) | 2.5 | The spec suggests testing 2–4. |
| Min same-direction candles | 3 | |
| Max small opposite candles | 1 | |
| Small opposite candle: body < × ATR | 0.30 | |
| Min average body / range (%) | 60 | Average across all leg candles. |

### STOP
| Input | Default | What it does |
|---|---|---|
| Min STOP bars | 3 | |
| Max STOP bars | 30 | Past this, the setup is invalid. |
| Flatness tolerance k (× ATR) | 0.25 | Both the touch band and the maximum regression drift. |
| Min touches of breakout level | 2 | Top for longs, bottom for shorts. |
| Min touches of opposite level | 1 | 1 = no filter. |
| STOP tolerance uses ATR of | Pre-leg | Pre-leg = ATR of the bar before the GO leg (strict). Current bar = live ATR(14), usually inflated by the impulse (looser). |
| STOP starts on | Leg-ending bar | Leg-ending bar = the first bar that failed to make a new extreme is STOP bar 1. "Bar after leg ends" leaves it out; its high is usually near the leg high, so leaving it out helps sagging pauses pass the drift test. |

### Entry
| Input | Default | What it does |
|---|---|---|
| Entry offset beyond STOP (ticks) | 1 | Buy stop at STOP high + n ticks (sell stop below the STOP low for shorts). |
| Stop-loss offset beyond swing (ticks) | 1 | |
| Swing pivot left / right bars | 1 / 1 | Pivot used for the stop. |

### Management
| Input | Default | What it does |
|---|---|---|
| Management override | Auto (day type) | Auto: Momentum day = full exit at target; Safe day = partial + BE. Or force "Always 2R" / "Always 1R+BE". |
| Full-exit target (R) | 2.0 | Also the target of the "Always 2R" shadow. |
| Partial exit at (R) | 1.0 | |
| Partial size (%) | 50 | Implemented with `strategy.exit(qty_percent = …)`. |
| Runner target (R) | 2.0 | |
| Momentum day: range ≥ × average | 1.5 | |
| Momentum day: average lookback | 20 | Sessions. |

### Visuals
Each group can be turned on or off: window shading, gap tint and Friday-close line, GO leg line, STOP box, entry/stop/target lines, trade labels (direction · depth % · day type · gap · final R), skip markers (an ✕ whose tooltip gives the reason), and whether to keep drawings of cancelled setups.

### Stats
Stats table on/off and its text size, the diagnostics funnel (bottom-left, on by default), an optional trade log table (bottom-right, newest first), how many rows the log shows, and a cap on how many trades the stats arrays keep.

## Reading the stats table (top-right)

- **Overall.** Trades, win %, average and median net R, expectancy (win% × avg win + loss% × avg loss), profit factor, max drawdown in R (peak to trough of cumulative R), longest losing streak, total R, and the number of ambiguous trades.
- **Breakdown rows.** All trades, the 5 depth buckets, Momentum / Safe, Gap / No gap, Longs / Shorts. Each row shows N, win %, average and median R, and the mean / P25 / P50 / P75 of **expansion** and **MFE (R)**. Percentiles come from `array.percentile_linear_interpolation`. Expansion is right-skewed, so read P50 before the mean.
- **Management.** Only trades where every shadow has resolved are included. Actual (net) uses engine fills after costs. Auto (sim), Always 2R (sim) and Always 1R+BE (sim) all come from the same stop-first simulation with no costs, so the three are directly comparable on identical trades. The gap between "Actual" and "Auto (sim)" is costs plus the intrabar rule.
- **Skips.** Setups armed, setups taken, and every non-taken armed setup by reason. Examples: `Skipped: Size too large`, `Skipped: Limit hit (losses)`, `Cancelled: STOP low broken`, `Cancelled: Bias flipped`, `Cancelled: Entry window ended`.

Each fill, close and skip is also written to **Pine Logs**, with the setup ID that appears as the order comment in the List of Trades.

## Too few trades? Read the diagnostics funnel

The bottom-left table counts every GO leg that ended between (window start − max STOP bars) and the window end, then shows how many survive each stage and which filter removed the rest:

```
GO legs ≥ X×ATR → ✕ same-direction candles / ✕ body ratio → valid GO legs → in bias direction
→ setups started → never a valid STOP (✕ touches, ✕ highs drift, ✕ lows drift, death reasons)
→ valid STOP → armed in window → order placed → filled
```

What to change depends on where the count collapses:

| Collapses at | Likely cause | What to try |
|---|---|---|
| The title says only ~10 sessions | TradingView only loads about 2–3 weeks of 1m history without Deep Backtesting | Use Deep Backtesting, or judge the filters only by the funnel ratios |
| GO legs ≥ X×ATR | Legs too small | `Min leg size` 2.0 |
| ✕ body ratio or ✕ same-direction candles | 1m candles are wickier than the 60 % / 3-candle defaults | `Min average body / range` 50, `Min same-direction candles` 2 |
| ✕ highs drift / ✕ lows drift | k × pre-leg ATR is only about 1–2 NQ points | `STOP tolerance uses ATR of` = Current bar, `STOP starts on` = Bar after leg ends, or `k` 0.4–0.5 |
| ✕ breakout level touched | Breakouts often come after a single touch | `Min touches of breakout level` 1 (for comparison only) |
| died: Bias flipped | The 15m bias flips while the STOP is forming | Expected; leave it |
| armed → order placed | Size or limit skips | Check the Skips section of the stats table. On NQ with a small account, "Size too large" dominates: switch to MNQ or raise the cap |

## Validation checklist: confirm these on the chart before trusting any statistic

Turn on *Skip / cancel markers*, *Keep drawings of cancelled setups*, *Plot bias EMA* and the *trade log*. Open the Data Window: it shows Bias, ATR, STOP drift highs/lows and STOP tolerance k×ATR for the bar under your cursor.

1. **A drifting STOP is rejected.** Find a pause after a GO leg whose highs or lows step up or down. While it forms, the Data Window's "STOP drift highs/lows" should be above "STOP tolerance k×ATR", and no box or order should appear. On a flat range, both drifts should be below the tolerance when the box first appears. That's the bar the order goes in.
2. **Entry and stop prices are exact.** On a few trades, zoom in and check: entry line = STOP box top + 1 tick (long), stop line = the most recent 1/1 pivot low inside the box − 1 tick (or the box low if there's no pivot), target = entry + 2R. Then open the List of Trades and check the fill matches entry + 1 tick of slippage.
3. **The bias doesn't repaint.** With the bias EMA plotted, the step line should only change on the first 1m bar after each 15m close. Check that no trade's direction disagrees with "Bias" in the Data Window on the bar before the fill. Then find a Sunday where the open is ≥ 100 pts from Friday's close. You should see the orange tint, the dashed Friday-close line, only counter-gap trades until the line is touched, and the tint stop on the bar that touches it.
4. **Session limits and the window behave.** No fill timestamp outside 20:00–22:29 ET. Never a 4th trade in a session, and no new trades after the second loss. A setup that is still pending at 22:29 shows `Cancelled: Entry window ended`.
5. **Safe-day management.** On a SAFE-labelled trade with ≥ 2 contracts, the List of Trades should show two exits: about 50 % at "TP1" (1R), then the rest at "TP2" (2R) or "BE" at the entry price. For a trade that goes to 1R and back, the label's final R should be about +0.5R minus costs, counted as a win. MOM trades should show a single exit.

More spot checks worth doing:

- Pick a trade from the log and recompute depth % = (GO high − STOP low) ÷ (GO high − GO low) by hand from the drawn line and box.
- Compare the ambiguous count with the total trades. If it's a large share, the shadow results lean conservative; compare them with a Bar Magnifier run.
- Switch the chart to 5m: the warning label should appear and the trade count should be 0.

## Known limitations

- **Not compiled in TradingView.** The TradingView compile endpoint isn't reachable from the environment this was written in. The file was parsed with `pynescript`, an open-source Pine grammar parser, which only checks syntax, and was reviewed by hand against v6 rules: no `na` bools, no implicit casts, lazy `and`/`or`, loop direction guards. Types and runtime behaviour are only checked when TradingView compiles it. Please report any compiler message and it will be fixed.
- Order quantity follows the newest pivot while pending. If a setup's quantity goes from 1 to ≥ 2 after its first order, the partial exit is created after the runner exit and may receive no quantity. This is rare.
- Shadows assume fractional sizes and no costs. Actual R uses engine fills after costs.
- The 1m history available on TradingView limits the sample size. Check the trade count before drawing conclusions from any bucket.
