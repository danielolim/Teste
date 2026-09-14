# PLAYBOOK — day-1 paper trading

For the lean desk in `TEAM.md` running the v0.1 rule in `STRATEGY.md` §4 with the breakers in `RISK.md` §3. Paper means **self-simulated**: Coinbase's sandbox returns static mock data and is not a paper venue. No exchange API keys are needed or allowed for the first 90 days.

## 0. Pre-flight (complete before day 1; do not start until every box is ticked)

**Decide and write down (one page, committed to the repo, hash noted in the ledger)**
- [ ] Numéraire: **BTC**. Benchmark: HODL of the day-0 stack. Safe state: active book at 75% BTC.
- [ ] Capital: €1,000 notional → 20% BTC core (never traded) + 80% active book. Day-0 conversion price = Coinbase BTC-EUR 00:00 UTC close of day 0 plus one taker fee (0.60%) plus half the observed spread. The core and the active book start 100% BTC.
- [ ] Rule v0.1 text, exactly as in `STRATEGY.md` §4, parameters frozen.
- [ ] KPI set and day-90 decision tree (`STRATEGY.md` §3).
- [ ] Breaker table with numbers (`RISK.md` §3).
- [ ] Expectation band for 90 days (half-out rule: p5 −9.3%, p50 0.0%, p95 +23.4%; hard limit −15%).
- [ ] Shadow rules for Revert, Macro, Pulse: one input each, one rule each, in writing.
- [ ] Fill model (see §4). Fee assumption: **0.40% maker**, with **0.60% taker** for the escalation step. Spread: observed `best_bid_ask` at ticket time; if unavailable, 0.05% (**assumption**, re-estimate weekly from observations).

**Build (spreadsheet or a small script; no exchange keys)**
- [ ] Data pull: Coinbase public candles `GET https://api.exchange.coinbase.com/products/BTC-EUR/candles?granularity=86400` (no auth). Store date, open, high, low, close, volume. Second source for the integrity check (any public BTC-EUR daily close).
- [ ] Signal sheet: SMA100, ×1.03 / ×0.97 bands, yesterday's state, today's state.
- [ ] Ledger: one row per event (fill, fee, breaker, override, note), balances in BTC and EUR, stack in BTC terms, HODL benchmark, tracking ratio, month-to-date fee spend, disposal list with EUR cost basis.
- [ ] Ticket template (§5). Daily log template (§5).
- [ ] Independent replication: a second, separate computation of the rule on the same closes (different person or script) producing the expected stack; compared to the ledger weekly.
- [ ] Coinbase facts confirmed on the day: `BTC-EUR` status `online`, `min_market_funds`, increments (from `/products/BTC-EUR`); current fee tier page screenshot.

## 1. Inputs (exact list, no others)

| Input | Source | When |
|---|---|---|
| BTC-EUR daily close (00:00 UTC) | Coinbase public candles | daily, once |
| Second-source daily close | any public BTC-EUR series | daily, integrity check only |
| Best bid / ask and spread at ticket time | Coinbase public order book / `best_bid_ask` (public product book is enough in paper) | only when a ticket exists |
| Current positions, fee spend MTD, tracking vs HODL, breaker states | ledger | daily |
| Coinbase status | status.coinbase.com | only when a ticket exists |
| Shadow inputs (one each for Revert / Macro / Pulse) | pre-registered public sources | daily, log only |

Not inputs: news, intraday charts, funding rates, social media, anyone's opinion, "how it feels".

## 2. Cadence

| Slot | Time (CET, adjust to taste but **fix it**) | Duration | Owner |
|---|---|---|---|
| Daily routine | 08:30 | 10 min | Rules → Controller → Bolt → Naka |
| Ticket window | 09:00–11:00 | only if ticket | Bolt, Naka |
| Weekly review | Sunday 18:00 | 30 min | Naka + Controller |
| Monthly review | 1st, 18:00 | 60 min | Naka + all seats |
| Day-90 decision | day 90 | — | Naka |

If the daily routine is missed, it is done the next morning with yesterday's close and logged as "late"; ≥ 95% on-time is the process KPI.

## 3. Decision flow (daily)

```
 1. Data      pull yesterday's 00:00 UTC close → age ≤ 26h? second source within 5%?
              NO  → log "no-trade: data", stop.
 2. Signal    state = ON  if close > SMA100×1.03
                      OFF if close < SMA100×0.97
                      else yesterday's state
              target(active) = 100% BTC if ON, 50% BTC if OFF
 3. Compare   current(active) from ledger. |target − current| ≥ 10pp?
              NO  → log "no-trade: on target", update ledger MTM, log shadows, stop.
 4. Breakers  Controller checks: fee budget left ≥ estimated fee? executions today < 2, rolling-30d < 4?
              tracking DD from start > −15%? no open Red/Black? weekday (if risk-increasing)?
              any NO → log "blocked: <breaker>", record counterfactual, stop.
 5. Ticket    Bolt builds: side, size (rounded to 1e-8), limit price = best bid (sell) / best ask (buy)
              from the public book at ticket time, post_only, client_order_id, estimated fee 0.40%,
              preview line: notional, fee, resulting exposure. Layer-0 checks all pass?
              NO  → reject, log, stop.
 6. Confirm   risk-increasing (buying BTC)  → Naka must write "YES <client_order_id>" in the log.
              risk-reducing (selling BTC)   → pre-authorised; Naka is informed.
              No YES by 11:00 → ticket expires; log "expired"; re-evaluate tomorrow.
 7. Fill      paper fill per §4. Record fill price, fee in EUR and in BTC, spread paid, timestamp.
 8. Ledger    update balances, stack (BTC terms), HODL, tracking ratio, MTD fees, disposal list.
 9. Shadows   log would-be Revert / Macro / Pulse actions and their hypothetical fills.
10. Log       one daily line (§5). Done.
```

Everything above step 6 is mechanical and should take under five minutes. If it regularly takes longer, the pipeline, not the market, is the problem.

## 4. Paper fill model (be pessimistic, be consistent)

- **Maker fill:** a post-only limit at the touch is assumed filled only if the next daily bar's range crosses it by ≥ 0.05% (buy: next low ≤ limit × 0.9995; sell: next high ≥ limit × 1.0005). Fee 0.40% of notional.
- **Not filled by window end (in paper: the touch was not crossed):** re-quote once at the new touch on the next routine; on the second miss and if the signal still holds, take the **taker** fill at the next close with fee 0.60% plus half the observed spread. Log every escalation.
- **Spread:** record `best_bid_ask` at ticket time; charge half the spread on taker fills.
- **Weekend:** no risk-increasing tickets Saturday/Sunday; risk-reducing tickets fill per the rule above.
- **Fees in BTC terms:** convert every EUR fee to BTC at the fill price and subtract it from the stack. The KPI is after fees in BTC.
- **No look-ahead:** the signal uses yesterday's close; the fill uses today's or tomorrow's prices. Never fill at the close that generated the signal.
- Once live, compare live fills against this model; the average deviation is the go-live-ladder metric (`RISK.md` Layer 4).

## 5. Templates

**Daily log line (one per day, append-only)**
```
date | close | sma100 | state | target% | current% | action(no-trade|ticket|blocked|expired) | reason | fee_mtd_eur | stack_btc | hodl_btc | tracking% | breakers(open) | shadow(R/M/P) | notes
```

**Ticket**
```
client_order_id: NAKA-YYYYMMDD-001
side/size: SELL 0.00593400 BTC   (active book 100% → 50%, ≈ €400 of €800)
limit: 67,410.00 EUR post_only   best_bid 67,410.00 / best_ask 67,435.00 (spread 0.037%)
est. fee: 0.40% = €1.60          preview: notional €400.01, exposure after 50.0%
layer-0: type ✓ notional ✓ price ✓ precision ✓ preview ✓ idempotency ✓ balance ✓ direction ✓ venue ✓
breakers: fee_mtd €1.60/€10 ✓  exec today 0/2 ✓  30d 1/4 ✓  trackingDD −2.1% ✓  weekday ✓
authorisation: risk-reducing → pre-authorised; Naka informed
```

**Weekly review (30 min, fixed agenda)**
1. Seven daily lines read aloud. Any missed or late?
2. Fee spend vs €10 budget. Trades and escalations.
3. Stack vs HODL, tracking DD, breaker states and counterfactuals.
4. Replication check: independent recomputation vs ledger, difference in % of equity.
5. Anomalies and pipeline bugs. **No discussion of parameters.**

**Monthly review (60 min)**
1. KPI table (`STRATEGY.md` §3), pass/fail per row.
2. Shadow logs summarised; no scoring until day 90.
3. Written parameter proposals, if any, filed for the day-90 decision only.
4. Disposal list and cost basis exported.

## 6. What NOT to do

- Do not connect exchange API keys before day 90. Paper needs none.
- Do not put keys, seed phrases or account identifiers in chat, tickets, logs or this repo.
- Do not add an indicator, input, filter or "just this once" rule during the experiment. Log the idea; file it for day 90.
- Do not change a parameter after a losing month. Do not change one after a winning month either.
- Do not let Pulse, Macro or Revert block or scale a trade. They log; they do not act.
- Do not override the rule. If you do, log it as a violation even if it made money.
- Do not use market orders or Simple Trade. Do not enter risk-increasing positions on weekends.
- Do not evaluate the desk on monthly P&L. Evaluate it on the process KPIs and the replication check.
- Do not stop the experiment early because it is going well, or badly, unless a Black breaker or a process failure says so.
- Do not count unrealised paper gains as validation. Ninety days is about one trade; it validates the pipeline, not the edge.
- Do not shorten the daily close to intraday because "the move is happening now". The rule works on 00:00 UTC closes or it does not work.
- Do not hold the 20% core inside the active book's accounting; it never moves and never counts as cash.
- Do not go live at €1,000. The ladder is €250 → €500 → €1,000.

## 7. Day 1, concretely

| Time (CET) | Step |
|---|---|
| 08:30 | Pull day-0 close (yesterday 00:00 UTC). Compute SMA100 and state. Record day-0 conversion: €1,000 → BTC at close × (1 + 0.60% + half-spread); split 20% core / 80% active; stack₀ = HODL₀. |
| 08:35 | Signal: if state = OFF, today's target is 50%; ticket SELL half the active book (risk-reducing, pre-authorised). If ON, target 100% → no trade. |
| 08:40 | Controller: breakers all green by construction; fee budget €10; log. |
| 08:45 | Bolt: ticket per template from the public book; Layer-0; paper fill per §4 tomorrow. |
| 09:00 | Shadows: log Revert / Macro / Pulse would-be actions. |
| 09:05 | Daily line written. Replication baseline computed and stored. Done. |
| Day 2 08:30 | Confirm paper fill against day-1 bar; ledger; repeat. |
