# CRASH 1000 Spike Early-Warning System (Pine Script v6)

File: `crash1000_spike_ews.pine`. It is a TradingView **indicator**: it places no orders and does no automated trading.

## 1. Installation

1. Open TradingView and load **CRASH 1000 INDEX** (Deriv feed), on the **1-minute** timeframe.
2. Open the **Pine Editor** at the bottom of the chart, then choose **Open → New blank indicator**.
3. Select everything in the editor and delete it. Paste the full contents of `crash1000_spike_ews.pine`.
4. Click **Save** and give it a name. Then click **Add to chart**.
5. To see more history in the evaluation panel, scroll the chart back so more bars load, or use a plan that loads more bars. All statistics are computed only on the bars that are loaded.

### Setting up alerts
Open **Alerts → Create alert** and set the condition to *C1000 Spike EWS*. Then choose one of these:

| Choice | What it does |
|---|---|
| `1. Potential spike developing` | Yellow SPIKE WATCH starts |
| `2. High-probability spike warning` | Red HIGH SPIKE ALERT starts |
| `3. Confirmed downward spike` | A spike bar is detected |
| `Any alert() function call` | Sends all enabled alerts with detailed messages (score, number of aligned factors, price). The timing comes from the input **8. Alerts → alert() timing**. |

Frequency: **Once Per Bar Close** gives confirmed signals that never repaint. **Once Per Bar** gives the earliest provisional warning, which may be withdrawn before the bar closes.

## 2. Default settings (CRASH 1000, 1m)

| Setting | Default | Notes |
|---|---|---|
| Detection sensitivity | 1.0 | Higher values give more signals |
| Early-warning sensitivity | 1.0 | Above 1 lowers the WATCH threshold and the number of factors needed for RED |
| SPIKE WATCH score | 50 | |
| HIGH ALERT score | 70 | Also needs at least 4 aligned factors |
| Momentum lookback | 14 | |
| Volatility / statistics lookback | 50 | |
| Structure lookback | 30 | |
| Spike confirmation threshold | 4.0 × baseline range | **Calibrate this one first (see below)** |
| Post-spike cooldown | 3 bars | |
| MTF confirmation | 5m | |
| Evaluation window | 5 bars | |

**Calibrating spike detection.** Crash 1000 averages about one spike per 1000 ticks, which is roughly every 16–17 one-minute bars. Look at the dashboard's **Mean spike interval**:
- If it is much higher than ~17 bars, small spikes are being missed. Lower *Spike confirmation threshold*.
- If it is much lower, normal noise is being counted as spikes. Raise the threshold.

Every other statistic depends on spikes being detected correctly, so do this step before anything else.

These defaults are reasonable starting values for the instrument's structure. They have **not** been fitted to data, and no accuracy figure is claimed. Use the evaluation panel to measure how they perform on your chart.

## 3. How the early-warning algorithm works

On every bar, ten factors are computed. Each factor is normalised to a value from 0 to 1. Spike bars are taken out of the "normal behaviour" statistics so that one spike does not distort the baselines.

| # | Factor | What it measures |
|---|---|---|
| F1 | Momentum & acceleration | z-score of the spike-free drift over the momentum lookback (how stretched it is), plus deceleration: short-term velocity falling below long-term velocity while the drift is still positive |
| F2 | Consecutive up-moves | Length of the current up-run, and the share of up bars in the lookback |
| F3 | Candle body & wick | Shrinking bodies compared with their average (exhaustion), upper-wick rejection, and growing lower-wick dips |
| F4 | Volatility regime | Short ATR (5 bars) divided by long ATR. Scores both compression (<0.85) and expansion (>1.2) |
| F5 | Tick volume | Absolute z-score of volume. Turned off automatically when the feed has no volume |
| F6 | Short-term structure | Failure right after a new high, close below the prior low, close below EMA9, and a lower-high sequence |
| F7 | Stretch from highs | z-score of price against its structure mean, and distance from the recent high in baseline-range units |
| F8 | Abnormal movement | Bar-change and range z-scores above 1.5σ |
| F9 | Statistical hazard | Empirical P(spike \| bars since last spike), compared with the average per-bar rate |
| F10 | MTF confirmation | Pressure on the higher timeframe (position in range, up-bar ratio, RSI), read from the **last closed HTF bar only** |

**Score.** `raw = Σ(wᵢ·fᵢ) / Σwᵢ`, then `score = 100·(1 − e^(−2.2·sensitivity·raw))`, smoothed by an EMA.
- **YELLOW**: score ≥ the WATCH threshold.
- **RED**: score ≥ the ALERT threshold **and** at least N factors are ≥ the "aligned" level.
- Warnings are switched off on the spike bar itself and during the cooldown.

**Probability estimate.** The script keeps a running table of the score at each past bar (in deciles) and whether a spike followed within the evaluation window. The figure *P(spike ≤ N bars)* on the dashboard is the hit rate for the current score decile, smoothed towards the overall base rate. It uses past bars only.

**Why this is not a countdown.** Spikes on a random-number-driven index behave close to a memoryless (geometric) process:
- The chance of a spike on the next bar is about the same whether the last spike was 2 bars ago or 40 bars ago.
- The script therefore does **not** raise risk just because time has passed. F9 stays near 0 unless the chart's own history shows that risk depends on age.
- The dashboard shows the interval **CV**. A value near 1.0 is what a random process produces.
- The dashboard also shows the theoretical base rate `1 − (1 − p)^N`.

### Repainting and future data
- **Confirmed** markers (WATCH, SPIKE ALERT, SPIKE) are drawn only when `barstate.isconfirmed` is true. They never change after the bar closes.
- The **live bar** shows a single label marked *provisional*, which can disappear before the bar closes. The dashboard shows both the live state and the state at the last close.
- The HTF data uses `request.security(..., expr[1], lookahead_on)`, the standard non-repainting pattern.
- All baselines use previous bars (for example, the spike baseline uses `[1]`).

## 4. Historical evaluation panel

| Metric | Definition |
|---|---|
| Warnings resolved | Number of warning starts. Overlapping warnings are merged into one |
| Followed by spike | A confirmed spike occurred within N bars after the warning started |
| No spike (false) | The window ended without a spike |
| Precision | hits / resolved warnings |
| Recall (spikes caught) | Share of spikes that had a warning active in the N bars before them |
| Missed spikes | Spikes with no prior warning in the window |
| Avg lead time | Bars (and minutes) from the start of the warning to the spike, for hits |
| **Lift vs random** | precision / base rate. **1.0 means no edge**: a warning raised at random times would score the same |

On 1m, with a spike every ~17 bars, a random 5-bar window already contains a spike about 25% of the time. Judge precision against that base rate, not against 0%. Treat the indicator as useful only if lift stays clearly above 1 across different stretches of history.

## 5. Disclaimer
Deriv synthetic indices are generated by a random number generator. No indicator can guarantee spike prediction. This tool shows conditions and measures its own track record honestly. It is not financial advice.
