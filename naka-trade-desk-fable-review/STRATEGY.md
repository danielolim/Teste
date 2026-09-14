# STRATEGY — Trend + Mean Reversion + Macro multiplier + Sentiment veto

Scope: edge realism only. Org and process are in `TEAM.md`; breakers in `RISK.md`. Every number here is computed in `METHOD.md` (Coinbase BTC-EUR daily closes, 2016-01 to 2026-09, fees 0.40% and 0.60% per side, next-close execution, no slippage). Anything not in `METHOD.md` is labelled **assumption**.

## 1. What the KPI actually demands

The KPI is denominated in BTC. That fixes three things the desk design ignores:

1. **The benchmark is HODL.** Holding BTC has a BTC-terms return of exactly 0. "≥3% more BTC per month" means *outperform holding by 3%/month after fees*. Any EUR the desk holds is a short position against its own benchmark.
2. **3%/month compounds to +42.6%/year relative to BTC.** No invented comparison is needed: that is a relative return that would put a single-asset, two-indicator, daily-bar timing rule among the best-performing strategies anywhere. It is demanded of a desk that has not shown one backtest.
3. **Fees at €1k are the highest tier, permanently.** Entry tier is 0.40% maker / 0.60% taker (verified 2026-09-14). One full round trip of the €800 active book costs €6.40–€9.60 = 0.64%–0.96% of equity. Monthly volume of €1.6k ≈ $1.9k never reaches the $10K tier threshold, so the desk cannot trade its way to cheaper fees without destroying the book. Gross timing gain required per month with a single round trip: **~3.7%–4.0%**.

How often has the underlying even moved enough? Monthly BTC-EUR since 2016 (127 months): median absolute move 11.0%, standard deviation 20.9%. The raw material for 3–4%/month of *relative* return exists in principle, but only if the desk is on the right side of double-digit monthly moves far more often than not.

## 2. Component critique

### 2.1 Trend (BTC momentum) — KEEP, as the only active rule

**What it is in BTC terms.** A trend rule that flips to EUR when price is below a moving average is a *crash-avoidance* rule. It grows the stack only in one way: sell BTC before a decline, buy more BTC back later. It pays for that with (a) whipsaw fees and (b) missed upside during every false exit and every re-entry lag. In BTC terms the strategy loses in rallies it sat out and in chop; it wins in sustained declines.

**Sanity check (in-sample, full grid disclosed, from `METHOD.md`).**

| Variant | Fee/side | Stack vs HODL (10.7y) | Per year | Max DD vs HODL | Trades/yr | Months ≥ +3% | Months ≤ −5% |
|---|---|---|---|---|---|---|---|
| SMA50, no band | 0.40% | ×0.45 | −7.2% | −73% | 19.9 | 32/127 | 39 |
| SMA50, ±3% band | 0.40% | ×0.92 | −0.8% | −70% | 7.6 | 28/127 | 36 |
| SMA100, no band | 0.40% | ×1.17 | +1.6% | −59% | 13.8 | 28/125 | 31 |
| **SMA100, ±3% band** | 0.40% | **×1.44** | **+3.5%** | −58% | 4.6 | 29/125 | 28 |
| SMA200, no band | 0.40% | ×0.53 | −6.1% | −81% | 7.7 | 27/122 | 31 |
| SMA200, ±3% band | 0.40% | ×0.95 | −0.5% | −72% | 2.8 | 26/122 | 29 |
| **SMA100, ±3% band** | 0.60% | **×1.31** | **+2.6%** | −60% | 4.6 | 29/125 | 28 |
| all other variants | 0.60% | ×0.30–×0.90 | −10.8% to −1.0% | −65% to −84% | | | |

Read this honestly:

- The *best* variant in a 6-cell grid delivers **+2.6% to +3.5% per year** relative to HODL. The KPI demands +42.6% per year. Gap: 12–16×.
- Even that variant sits **58–60% below HODL at its worst point** (it was in EUR while BTC rallied). A BTC-denominated principal experiences that as "doing nothing would have left me with 2.5× the coins."
- It hits ≥ +3% in a month **23% of the time** (29/125). It posts ≤ −5% in **22% of months** (28/125). Under the locked kill rule it is shut down from 100% of start months, 90% of them within 3 months, all within 6.
- Hysteresis (the ±3% band) matters more than the lookback: it cuts executions from 13.8 to 4.6 per year. On the desk's €800 active book each execution costs €3.20–€4.80, so that is the difference between ~€44–€66/yr (4.4–6.6% of equity) and ~€15–€22/yr (1.5–2.2%) in fees (**assumption**: executions × per-side fee; the backtest already nets fees, this is only to size the drag).
- 4 of 6 variants lose to HODL. Trend on BTC is *fragile to parameterisation*, and I did not optimise. Expect out-of-sample to be worse than the best in-sample cell.

**All-out vs half-out.** The desk's design flips the whole active book between BTC and EUR. I also tested the same SMA100 ±3% rule with the OFF state at 50% BTC instead of 0%:

| OFF-state exposure | Fee/side | Stack vs HODL | Per year | Max DD vs HODL | Months ≤ −5% | 90-day rel. return p5 / p50 / p95 | 90-day windows breaching −10% / −15% DD from start |
|---|---|---|---|---|---|---|---|
| 0% BTC (all-out) | 0.40% | ×1.44 | +3.5% | −58% | 28/125 | −18.0% / 0.0% / +45.9% | 36% / 18% |
| 0% BTC (all-out) | 0.60% | ×1.31 | +2.6% | −60% | 28/125 | −18.4% / 0.0% / +45.9% | 36% / 18% |
| **50% BTC (half-out)** | 0.40% | **×1.48** | **+3.8%** | **−36%** | **13/125** | −9.3% / 0.0% / +23.4% | **6% / 1%** |
| **50% BTC (half-out)** | 0.60% | **×1.41** | **+3.3%** | **−37%** | **15/125** | −9.4% / 0.0% / +23.3% | **6% / 1%** |

Half-out did not give up relative return in-sample (fees on half-size flips are half as large, and half the rally is never missed) while cutting the worst relative drawdown by ~40% and halving the number of ≤ −5% months. It is the less heroic bet against the benchmark, which is what a €1k desk with no track record should be making.

**Verdict.** Keep trend as the single active rule because it is the only component with a coherent mechanism in BTC terms and the only one that did not lose to HODL in every configuration. Keep it slow (100-day class), with hysteresis, evaluated daily, traded rarely, and **half-out rather than all-out**. Pre-register it as a crash-avoidance rule with low single-digit annual relative edge and multi-year relative drawdowns, and stop calling it a momentum engine.

### 2.2 Mean Reversion — REPLACE (remove from capital)

Three independent reasons, each sufficient:

1. **Fee economics.** MR earns small edges per trade. At 0.80%–1.20% per round trip the average daily-bar mispricing it captures must exceed ~1.2% *after* the mean-reversion actually occurs. On daily BTC-EUR bars at €1k that margin is not there, and there is no volume path to cheaper fees.
2. **Direction problem in BTC terms.** "Buy the dip" requires holding EUR most of the time (a permanent short against HODL). "Sell the rip" requires selling BTC into strength, which is fighting the asset's dominant drift; in strong trends z-scores stay elevated for weeks and you buy back higher.
3. **Measured result.** Naive z-score(20) MR at ±2, two bases, 2016–2026:

| Variant | Fee/side | Stack vs HODL | Per year | Max DD vs HODL | Trades/yr |
|---|---|---|---|---|---|
| BTC base (sell tops, buy back at mean) | 0.40% | ×0.01 | −34.6% | −99% | 13.5 |
| EUR base (buy dips, sell at mean) | 0.40% | ×0.00 | −43.0% | −100% | 10.8 |
| both, at 0.60% | | ×0.01 / ×0.00 | −36.4% / −44.2% | | |

I also tested the charitable version — MR used only as *entry timing inside the trend rule* (on trend-ON, wait up to 10 days for a z<0 pullback before buying). It made the trend rule **worse**: ×1.12 vs ×1.44 at 0.40%, ×1.01 vs ×1.31 at 0.60%. Waiting for a pullback in a fresh BTC uptrend mostly means buying higher.

These are one naive parameterisation each. They do not prove no MR rule can work; they prove the desk's MR book is unproven in the one denomination that matters, and that its default failure mode is total in BTC terms. **Verdict:** remove Revert from capital. If Naka wants it, it runs as a shadow signal with pre-registered rules and zero authority for 90 days, then is scored (see `TEAM.md`).

Consequence: the 55/45 Trend/Revert split and Atlas's allocation job disappear. Ledger already merges to a single BTC target, so nothing downstream changes.

### 2.3 Macro regime multiplier — REMOVE as a multiplier

- **Any multiplier below 1.0 is a short against HODL.** Scaling exposure to 0.5 in "risk-off" is exactly the trade that loses in BTC terms when the regime call is wrong, and BTC's highest-volatility periods include its biggest rallies (16 of 127 months were ≥ +33%).
- **Measured proxy.** The most defensible rule-based multiplier is volatility targeting. `exposure = trend × min(1, 60% / realised 30-day vol)` on top of the best trend variant: **×0.71 vs ×1.44** at 0.40%, **×0.63 vs ×1.31** at 0.60%, with more trades (10.9/yr) and deeper relative drawdown (−73%). Vol-targeting is a sound EUR-risk tool and a structural BTC-terms drag.
- **Discretionary macro (rates, DXY, ETF flows) at monthly horizon on €1k is narrative.** No inputs are specified, so the multiplier cannot be wrong, which means it cannot be right either. It is also the single most powerful actor in the pipeline: one 0.5 halves everyone else's work, with zero capital and zero accountability.

**Verdict.** Remove. If Naka insists on a regime concept, it must be (a) binary, not continuous, (b) inside the trend rule (e.g., "trend-ON requires price above the 200-day"), (c) pre-registered and tested before it gets authority. I did **not** test a binary filter; it is an open item, not a recommendation.

### 2.4 Sentiment veto (Pulse) — REMOVE authority, keep as shadow

A veto with unspecified inputs is unfalsifiable: when it blocks a trade nobody records what would have happened. Its best case is "prevented a loss we cannot show"; its worst case is silently killing the trend rule's few winning entries. Convert to a shadow signal: pre-register one public input (e.g., a published fear/greed index) and one rule (e.g., "would veto buys when index > 85, would veto sells when < 15"), log the would-be veto daily, and score the counterfactual at day 90. Zero authority until then.

## 3. Is ≥3% BTC MoM realistic? No. Replacement KPI for a 30–90 day experiment

### Why the current KPI cannot be kept

- Best in-sample relative edge found: **+2.6–3.5%/yr**, not per month.
- Under the locked rules a strategy with a *true* 3%/month mean alpha and 5%/month dispersion survives 12 months with **6.2%** probability; with 8% dispersion, **2.7%** (iid-normal model; `METHOD.md` §2 and script section 3). Every empirical variant is killed within 8 months from every start month.
- "Soft miss → review" fires on **~77% of months** for the best variant. A review nearly every month is not governance; it is an invitation to change parameters after every loss.

### What 90 days can and cannot measure

A slow trend rule trades ~3–5 times per year. Ninety days is roughly one trade. **No 90-day paper run can validate edge for this strategy class.** It can validate the pipeline: data, signals, tickets, fills modelling, ledger, breakers, human cadence. Rule choice must therefore rest on long-sample evidence (with the caveats in `METHOD.md`), and the experiment must measure process, with performance reported but not targeted.

### Proposed KPI set (30–90 days, paper)

| Layer | KPI | Pass |
|---|---|---|
| Process (primary) | Decisions logged *before* execution, with timestamp + rule id | 100% |
| Process | Unlogged discretionary overrides | 0 |
| Process | Daily routine completed | ≥ 95% of days; weekly and monthly 100% |
| Process | Ledger reconciles to simulated fills, fee recorded in EUR and BTC | 100% |
| Cost | Fee + modelled spread spend | ≤ 1.0% of equity per calendar month (~€10) |
| Performance (measured, not targeted) | Cumulative BTC-terms return vs HODL after modelled costs | reported with trade count against the pre-registered 90-day band (half-out rule: p5 −9.3%, p50 0.0%, p95 +23.4%; historical min −24.6%, max +47.5%) |
| Replication | Independent recomputation of the rule on the same closes vs the ledger | match within 0.5% of equity, else it is a pipeline bug |
| Hard limit | Tracking drawdown vs HODL from experiment start | half-out rule: never below **−15%** (breached in ~1% of historical 90-day windows) → stop, diagnose, redesign (not "shutdown"). If Naka insists on all-out, the equivalent level is −25% (2% of windows); −10% would have fired in 36% of windows and is not a usable limit |
| Shadow | Revert / Macro / Pulse would-be signals logged daily | 100%, scored at day 90 |

### Decision at day 90

- All process KPIs pass, replication matches, tracking DD within limit → **eligible** for live at €250 (25% of capital), same rules, no changes for 60 days.
- Any process KPI fails or replication mismatches → fix, extend paper 30 days. Do not go live.
- Performance outside [−15%, +25%] vs HODL over 90 days → investigate the pipeline first (a paper run that beats HODL by 30% in 90 days is more likely a fill-model bug than an edge; the rule's historical 95th percentile is +23%).

## 4. Recommended v0.1 rule set to pre-register

Unoptimised. Parameters come from the disclosed grid, not from a search. Items marked **assumption** were not tested.

**Portfolio**
- 20% BTC core (~€200 equivalent), never traded, never counted as "cash". In BTC terms the risk-free asset is BTC; the 20% EUR floor was a permanent 20% short (static 80/20 alone: ×0.70 vs HODL, −3.2%/yr, and it hard-fails in any ≥ +33.3% month — 16 of 127). Once live, the core's job is custody isolation (self-custody, never on the exchange), not market risk.
- 80% active book, exposure ∈ {50%, 100%} of the active book (half-out). Total BTC exposure of the desk is therefore always between 60% and 100% of equity.

**Signal (daily, on the 00:00 UTC close)**
- Trend-ON if close > SMA100 × 1.03. Trend-OFF if close < SMA100 × 0.97. Otherwise keep yesterday's state (hysteresis).
- Target exposure of the active book = 100% BTC if ON, 50% BTC / 50% EUR if OFF.

**Execution (next day, fixed window, e.g., 08:00–10:00 CET)**
- Post-only limit at or inside the best bid/ask; re-quote at most 3 times at 15-minute intervals; if still unfilled and the signal still holds at the window's end, cross the spread once and log the taker fee. **Assumption**: this escalation is untested; it exists to bound slippage, not to add edge.
- Never use Simple Trade; never use market orders except the final escalation step.

**Turnover brakes**
- Fee budget ≤ 1.0% of equity per calendar month. If a flip would exceed it, log the blocked flip; it executes on the 1st of next month if still signalled. **Assumption**: untested; caps whipsaw cost at the price of occasional delayed exits.

**Expectations to pre-register (half-out rule, from in-sample; expect worse live)**
- Months ≥ +3% stack: ~19% (24/125). Months ≤ −5% stack: ~10–12% (13–15/125). Annual relative return: low single digits (+3.3–3.8% in-sample). Relative drawdown vs HODL: −36% at worst in-sample, and relative drawdowns last years. Executions: ~5 per year, i.e. roughly one per 90-day experiment.
- 90-day relative return band: p5 −9.3%, p50 0.0%, p95 +23.4%. Outside [−15%, +25%] → suspect the pipeline before the market.
- Under the desk's *current* kill rule this configuration is still shut down from 100% of start months (86% within 3 months). The rule set only makes sense together with the KPI change in §3.

## 5. What was not tested (honest gaps)

- Walk-forward / out-of-sample splits; parameter stability across halving cycles.
- Only one OFF level (50%) for half-out. Scanning OFF levels would be optimisation; the point is structural, not the number.
- Spread, slippage, partial fills; weekend execution.
- Breakout, dual-MA, or channel definitions of trend; binary 200-day regime filter.
- Intraday data; funding or basis signals; any sentiment dataset.
- Taxes (every BTC→EUR sale is a taxable disposal in most EU jurisdictions; see `RISK.md`).
