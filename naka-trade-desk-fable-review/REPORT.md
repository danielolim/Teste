# REPORT — Naka BTC desk review

Reviewer: Fable (Cursor cloud agent). Date: 2026-09-14. Basis: the desk specification as provided; Coinbase public API data and documentation fetched on the review date; computations disclosed in full in `METHOD.md`. No account, no live data feed, no proprietary backtest. Where a statement is not backed by `METHOD.md` it is labelled **assumption**.

Package: `REPORT.md` (this) · `TEAM.md` · `STRATEGY.md` · `RISK.md` · `PLAYBOOK.md` · `CHANGES.md` · `METHOD.md`.

## 1. Executive verdict

**NO-GO for live with €1,000 under the current design. Conditional GO for a 90-day paper experiment, only after the twelve P0 changes in `CHANGES.md`.**

The desk as specified cannot survive its own rules, independently of whether it has edge:

1. **The KPI is a shutdown timer.** "≥3% more BTC per month" is +42.6%/year relative to simply holding BTC, after fees, demanded from two indicators on daily bars. Applying the locked kill rule (two consecutive misses or any −5% month) to every strategy I tested on Coinbase BTC-EUR since 2016 — including the one that beat HODL by ~3%/year over the decade — the desk is shut down from ~100% of possible start months, ~90% of them within three months, all within eight. A hypothetical strategy with a *true* +3%/month mean and 5% monthly dispersion survives twelve months with 6.2% probability.
2. **The 20% EUR cash floor guarantees the −5% hard fail.** In BTC terms, EUR is a short against the benchmark. A static 80/20 book loses ≥ 5% of its stack whenever BTC rises ≥ 33.3% in a month: 16 of the last 127 months. Zero trading error required.
3. **The desk has two numéraires that fight.** The KPI is in BTC; the −8% MTD breaker and the cash floor are in EUR. The EUR breaker fires when the BTC KPI is unaffected (long, BTC drops) and sleeps when the KPI is being destroyed (in EUR, BTC rallies). For a fully long book it would have fired in 45% of months, 42 of 57 times from a single day's drop.

Separately from those fatal items: the nine-role serial pipeline is process theater around one number a day; three of the four signal components (mean reversion, macro multiplier, sentiment veto) have no falsifiable specification and, where I could test a proxy, were destructive in BTC terms; and "paper" has no venue — Coinbase's sandbox is static mock data.

## 2. Scorecard — org/process quality vs edge realism, kept separate

| Dimension | Grade | One line |
|---|---|---|
| Mandate clarity (spot, BTC-only, no leverage, human confirm, paper first) | **A** | The constraints are right and rare |
| Numéraire consistency | **F** | BTC KPI with EUR breaker and EUR floor |
| KPI realism | **F** | 12–16× the best in-sample relative edge found |
| Kill-switch design | **F** | Fires on noise; safe state undefined; risk cuts wait for a human |
| Team structure | **D** | 9 roles, 8 hand-offs, overlay agents with authority and no capital, no attribution |
| Strategy evidence | **D** | None supplied; my sanity checks: trend marginal and fragile, MR toxic, multipliers a drag |
| Venue readiness (Coinbase Advanced Trade, BTC-EUR) | **C** | Sizes fine; fees permanent at top tier; thin EUR book; no paper venue; good order tooling |
| Operational safety | **B−** | Human confirm and key hygiene stated; no operational breakers; no size ladder |

## 3. Strengths

- Spot only, no leverage, one asset, explicit human confirmation for execution, paper before live. Most retail "desks" fail on these before anything else.
- A BTC-denominated KPI is honest: it forces the desk to beat *holding*, which is the only meaningful bar for a BTC-only book.
- Ledger merging to a single BTC target + cash is correct and makes the sub-books redundant in a good way.
- Explicit kill criteria exist at all. They are wrong, but a desk that writes them down can fix them.
- "No API keys in chat" and a human executor are the right instincts; they need completing, not replacing.

## 4. Weaknesses, ranked

1. **KPI mathematically unachievable and kill rule statistically doomed** (§1.1; `STRATEGY.md` §3, `RISK.md` §2.2).
2. **Cash floor is a structural short against the benchmark** (§1.2; `RISK.md` §2.3). Over the sample it alone cost 30–48% of the stack.
3. **Numéraire conflict** (§1.3; `RISK.md` §2.1). "Stop new risk" and "freeze" have no defined safe state.
4. **Edge unproven and, for three of four components, unfalsifiable.** Best trend variant: +2.6–3.5%/yr vs HODL in-sample with −58–60% relative drawdowns; 4 of 6 variants lose to HODL. Naive MR: ×0.00–0.01. Vol-target multiplier: ×0.71 vs ×1.44. Sentiment veto: unscorable (`STRATEGY.md` §2).
5. **Org is over-built and unattributable** (`TEAM.md` §3). Nobody can own a loss; overlay agents can never be fired because their record cannot be computed.
6. **Risk reduction waits for a human** in a 24/7 market with weekend volume a third of weekday (`RISK.md` §2.6).
7. **No operational breakers, no go-live gates, no size ladder** (`RISK.md` §2.7–2.8).
8. **No real paper venue and no fill model.** Coinbase sandbox is static; paper results without an explicit fee + spread model will flatter (`RISK.md` §4).
9. **Fees are permanent at the top tier** (0.60% taker / 0.40% maker below $10K trailing volume); a €1k desk cannot trade its way down (`STRATEGY.md` §1).
10. **Tax and custody unaddressed**: every BTC→EUR sale is a disposal; EUR at the exchange carries the same counterparty as the BTC (`RISK.md` §2.9–2.10).

## 5. Go / no-go for live with €1,000

**No-go.** Conditions under which the answer becomes "go", in order:

1. P0-1 to P0-12 in `CHANGES.md` adopted in writing (numéraire, KPI, core instead of floor, one rule, breakers, org, pipeline, fill model, fee budget, no-tinkering rule).
2. 90 days paper with all process KPIs passed, weekly replication check within 0.5% of equity, no unresolved Red, shadow signals scored (`STRATEGY.md` §3).
3. Custody and key plan executed; 20% core in self-custody (`RISK.md` Layer 4).
4. Live starts at **€250**, not €1,000; ladder €250 → €500 → €1,000 with 60 days per step and live fills within 0.3% of the paper model.

Even then, go live with the written expectation that the rule's edge is low single digits per year relative to BTC, that relative drawdowns of −36% (half-out) are normal, and that ninety days proved the pipeline, not the edge.

## 6. Recommendations in one paragraph each

**Team** (`TEAM.md`): four seats — Naka, Rules, Controller (Ledger + Sentinel, pre-authorised to cut to neutral), Bolt — plus a Shadow Research log with no authority. Remove Atlas. Demote Revert, Macro, Pulse to shadow. Rule changes only monthly, in writing, never after a Red.

**Strategy** (`STRATEGY.md`): keep trend as the only active rule — SMA100 with ±3% hysteresis, daily on the 00:00 UTC close, next-day execution, **half-out** (OFF state = 50% BTC of the active book). Replace the 20% EUR floor with a 20% BTC core. Replace Revert (remove from capital), remove the Macro multiplier, remove Pulse's veto. Replace the 3% MoM KPI with process KPIs, a reported (not targeted) BTC-terms return band, a replication check, and a −15% tracking hard limit.

**Risk** (`RISK.md`): one numéraire (BTC), benchmark HODL, safe state 75% BTC of the active book. Layer-0 order guards (post-only, caps, preview, idempotency), Layer-1 turnover guards (fee budget ≤ 1%/month), Layer-2 tracking ladder (−7.5/−15/−25%), Layer-3 operational guards, Layer-4 go-live gates and size ladder. Pre-authorised risk cuts. EUR worry answered once by sizing the desk, not by trading-time breakers.

**Playbook** (`PLAYBOOK.md`): ten-minute daily routine at a fixed time, one ticket window, weekly and monthly reviews with fixed agendas, pessimistic self-simulated fills, one append-only daily line, and a list of things not to do that starts with "no exchange keys before day 90".

## 7. What I computed, and what I did not

Computed (`METHOD.md`): monthly BTC-EUR return distribution 2016-02 to 2026-08; the analytic effect of a 20% EUR floor; kill-rule survival analytically and empirically; a fully disclosed 6-cell trend grid, two naive mean-reversion rules, two hybrids, a static 80/20 book, and an all-out vs half-out comparison including 90-day window statistics; Coinbase BTC-EUR vs BTC-USD liquidity over the last 90 days; daily gap statistics; the −8% MTD breaker's firing frequency.

Not computed: out-of-sample or walk-forward tests, spread and slippage, intraday behaviour, any sentiment or macro dataset, a binary regime filter, taxes. All backtest numbers are in-sample and unoptimised, and should be expected to degrade live.

Verified against Coinbase docs and API on 2026-09-14: `BTC-EUR` online with `min_market_funds` €0.84; fee tiers; order types (`limit_limit_gtc` with `post_only`, `stop_limit_*` triggering on last trade price, `trigger_bracket_*` placing a stop-limit 5% away, attached TP/SL, `orders/preview`); sandbox being static; WebSocket rate limits.

## 8. Naka should do next — one page

**Decide (this week, in writing)**
- [ ] Numéraire = BTC. Benchmark = HODL. Safe state = active book at 75% BTC. One page, committed.
- [ ] Delete "≥3% BTC MoM / two misses / −5% month" as kill criteria. Adopt the process-first KPI set and day-90 decision tree (`STRATEGY.md` §3).
- [ ] Convert the 20% EUR cash floor into a 20% BTC core; plan self-custody for it before any live order.
- [ ] Remove Revert from capital, Macro as multiplier, Pulse as veto. Write one shadow rule (one input, one threshold) for each.
- [ ] Freeze rule v0.1: SMA100, ±3% band, half-out, daily 00:00 UTC close, next-day execution. Hash the text.
- [ ] Collapse the org to Naka / Rules / Controller / Bolt + Shadow. Remove Atlas. Sign the RACI (`TEAM.md` §4).
- [ ] Pre-authorise risk-reducing orders; Naka's "YES" only for buys.
- [ ] Adopt the breaker table (`RISK.md` §3): fee budget ≤ 1%/month, tracking ladder −7.5/−15/−25%, Layer-0 order guards, weekend rule.
- [ ] Adopt the no-tinkering rule: parameter changes only at monthly review, never within 30 days of a Red, each change restarts the 90-day clock.

**Build (before day 1)**
- [ ] Public-candle data pull + second source; signal sheet; ledger with BTC-terms stack, HODL, tracking, fee MTD, disposal list.
- [ ] Fill model exactly as `PLAYBOOK.md` §4 (0.40% maker, 0.60% taker escalation, observed spread, no look-ahead, fees in BTC).
- [ ] Ticket and daily-line templates; independent replication script or second person.
- [ ] Confirm on the day: `BTC-EUR` status, increments, `min_market_funds`, current fee tier.

**Run (days 1–90)**
- [ ] Daily 10-minute routine at a fixed time; ≥ 95% on time.
- [ ] Weekly: fee spend, tracking, breakers, replication check within 0.5%. No parameter talk.
- [ ] Monthly: KPI table pass/fail; shadow logs summarised; proposals filed for day 90 only.
- [ ] Zero unlogged overrides. Every override logged as a violation.
- [ ] No exchange API keys. No new indicators. No intraday. No market orders. No weekend buys.

**Decide again (day 90)**
- [ ] Process KPIs all pass + replication matches + no unresolved Red → eligible for live at **€250**.
- [ ] Any failure → fix, extend paper 30 days.
- [ ] Score shadows in BTC terms after fees; proposals, not promotions.
- [ ] If the −36% relative drawdowns are intolerable in EUR reality, change the KPI to EUR terms and re-run this review. Do not bolt EUR breakers onto a BTC KPI.

**Before the first live order**
- [ ] Dedicated Coinbase portfolio; API key `view` + `trade` only, no transfer; IP allow-list; hardware 2FA; withdrawal allow-list.
- [ ] 20% core in self-custody. Exchange holds only the active book.
- [ ] Tax treatment confirmed with an adviser for your jurisdiction; disposal export tested.
- [ ] Size ladder written: €250 → €500 → €1,000, 60 days per step, live fills within 0.3% of the paper model.
