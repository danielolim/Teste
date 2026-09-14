# CHANGES — ranked

P0 = before paper day 1 (the desk is not viable without it). P1 = during paper, before any live order. P2 = after live at €250. Evidence references `METHOD.md`.

## P0 — before paper day 1

| # | Change | Why (evidence) | Owner | Done when |
|---|---|---|---|---|
| P0-1 | **Pick one numéraire and write it down: BTC.** Benchmark = HODL. Safe state = active book at 75% BTC. | KPI in BTC vs breaker/cash floor in EUR pull in opposite directions; the EUR breaker fires when the KPI is unaffected and sleeps when it is destroyed (`RISK.md` §2.1). | Naka | one page committed; every doc uses the same three words |
| P0-2 | **Delete the "≥3% BTC MoM or two misses = shutdown" rule.** Replace with the process-first KPI set and day-90 decision tree (`STRATEGY.md` §3). | Every tested strategy is killed from ~100% of start months, ~90% within 3 months, including ones that beat HODL over a decade. A true +3%/mo, 5% σ strategy survives 12 months with 6.2% probability. | Naka | new KPI table signed |
| P0-3 | **Replace the 20% EUR cash floor with a 20% BTC core** (never traded; self-custody once live). | Static 80/20 hard-fails in every ≥ +33.3% month (16 of 127) and bled 30–48% of the stack over the sample. In BTC terms the risk-free asset is BTC. | Naka / Controller | ledger shows core separately, never counted as cash |
| P0-4 | **Remove Revert from capital; remove Macro as a multiplier; remove Pulse's veto.** All three become shadow logs with one pre-registered input and rule each. | Naive MR: ×0.00–×0.01 vs HODL; MR as trend-entry timing made trend worse (×1.12 vs ×1.44). Vol-target multiplier: ×0.71 vs ×1.44. Vetoes are unscorable without logged counterfactuals. | Naka | shadow rules written; zero authority in the decision flow |
| P0-5 | **One active rule, pre-registered: SMA100 ±3% hysteresis, half-out (OFF = 50% BTC of the active book), daily on 00:00 UTC close, next-day execution.** | Only trend variant class that did not lose to HODL in every cell; half-out matched all-out return in-sample (×1.41–1.48) with −36% vs −58% relative drawdown and half the ≤ −5% months. | Rules | rule text + parameters frozen and hashed |
| P0-6 | **Replace the −8% MTD EUR breaker with the tracking-DD ladder** (−7.5% amber / −15% red / −25% black from experiment start, BTC terms). | −8% MTD would fire in 45% of months for a long book, 42 of 57 times from a single day; −10% tracking on all-out fires in 36% of 90-day windows; −15% on half-out fires in ~1%. | Controller | ladder in the ledger with reset rules |
| P0-7 | **Pre-authorise risk-reducing actions; require Naka's yes only for risk-increasing orders.** | Human confirm on STOP makes Sentinel a notification service in a 24/7 market with thin weekend liquidity (92 vs 267 BTC/day). | Naka | RACI signed (`TEAM.md` §4) |
| P0-8 | **Collapse to four seats + shadow** (Naka, Rules, Controller = Ledger+Sentinel, Bolt; Shadow Research with no authority). Remove Atlas. | Nine roles / eight hand-offs for one number per day; attribution impossible; overlay agents with authority and no capital. | Naka | org chart replaced |
| P0-9 | **Build the paper pipeline yourself** (public candles, signal sheet, ledger, fill model, replication). Coinbase's sandbox is static mocks. | Sandbox returns pre-defined responses, Accounts/Orders only, no prices (checked 2026-09-14). | Controller / Bolt | day-1 dry run produces a daily line and a ticket |
| P0-10 | **Pessimistic fill model in the rulebook**: 0.40% maker, 0.60% taker escalation, observed spread, no look-ahead, fees converted to BTC. | Entry fee tier is permanent at €1k volume; the KPI is after fees in BTC. | Controller | `PLAYBOOK.md` §4 adopted verbatim or amended in writing |
| P0-11 | **Fee budget ≤ 1% of equity per month, hard.** | One flip of €400 costs €1.60–€2.40; without a cap, the whipsaw variants (14–20 executions/yr on the full €800 book) would spend roughly 4–10% of equity per year on fees (**assumption**: executions × per-side fee). | Controller | Layer-1 guard active |
| P0-12 | **No-parameter-change rule**: changes only at monthly review, in writing, never within 30 days of a Red, each change restarts the 90-day clock. | "Soft miss → review" would have fired ~77% of months, guaranteeing drift. | Naka | written into the rulebook |

## P1 — during paper, before the first live order

| # | Change | Why | Owner | Done when |
|---|---|---|---|---|
| P1-1 | **Replication check weekly**: independent recomputation of the rule on the same closes must match the ledger within 0.5% of equity. | A 90-day run cannot validate edge (≈1 trade); it can validate the pipeline. Big paper wins are more likely bugs. | Naka (A), Controller (R) | 12 consecutive weekly matches |
| P1-2 | **Layer-0 order checklist implemented in Bolt** (post-only, notional cap, ±1% price sanity, precision, preview, idempotent `client_order_id`, balance, direction, venue). | `POST /orders/preview` and explicit reject reasons exist; use them. Precision rejects are the common €1k failure. | Bolt | every paper ticket shows all nine checks |
| P1-3 | **Layer-3 operational guards** designed and dry-run: API error halt, reject halt, fill deviation, reconciliation, status page, data integrity vs second source, key-permission startup check, clock, weekend rule. | No operational breakers exist today; these are the failures that actually happen at €1k. | Controller | each guard triggered once in a drill and logged |
| P1-4 | **Shadow scoring at day 90** for Revert / Macro / Pulse, BTC terms after fees, with the −5% month count. | Authority is earned by a scored counterfactual, not by narrative. | Shadow (R), Naka (A) | scorecard filed; proposals, not promotions |
| P1-5 | **Custody and key plan written**: dedicated Coinbase portfolio, `view`+`trade` key only (no transfer), IP allow-list, hardware 2FA, withdrawal allow-list, rotation at each size step; 20% core to self-custody before first live order. | "No keys in chat" is necessary, not sufficient. EUR at the exchange carries the same counterparty as the BTC. | Naka | checklist executed and screenshot-free confirmation logged |
| P1-6 | **Tax ledger**: disposal list with EUR cost basis per lot; jurisdiction rule confirmed with an adviser (e.g., Portugal 28% on < 365-day holdings — verify). | Every BTC→EUR sale is a disposal; DAC8 reporting from 2026. A BTC-up month can still be a taxable EUR gain. | Controller | export works from paper ledger |
| P1-7 | **Second data source and integrity rule** (close deviation > 5% → no trade). | Single-venue daily closes; thin EUR book; wick risk on "last trade price" triggers. | Controller | in the daily flow |
| P1-8 | **Decide about resting stops**: default none (exposure rule instead); if used, wide-limit catastrophe insurance only, documented. | Stop-limit triggers on last trade price; bracket auto-limit is 5% away; thin EUR book can print through. | Rules / Naka | decision recorded |

## P2 — after live at €250

| # | Change | Why | Owner | Done when |
|---|---|---|---|---|
| P2-1 | **Size ladder €250 → €500 → €1,000**, step only after 60 days with live fills within 0.3% of the paper model on average; step down on Red, back to paper on Black. | Live starts at 100% of capital today; fill-model error is unknown. | Naka | ladder log |
| P2-2 | **Live-vs-paper fill study**: spread paid, maker fill rate, escalation count, weekend behaviour; re-estimate the fill model monthly. | Spread and slippage were not modelled in the review; the EUR book is 3% of the USD book's volume. | Controller | monthly table |
| P2-3 | **Out-of-sample rule check** at 6 and 12 months live: compare realised monthly BTC-terms distribution to the pre-registered expectations (≈19% months ≥ +3%, ≈10–12% months ≤ −5%). | In-sample is all there is today; expect worse. | Naka | written comparison at each checkpoint |
| P2-4 | **Consider a binary regime filter inside the trend rule** (e.g., trend-ON requires price > 200-day) — as a *tested* proposal, not a multiplier. | Untested here; the only form of "Macro" that is falsifiable. | Rules | backtest with the `METHOD.md` mechanics, full grid disclosed, before any authority |
| P2-5 | **Revisit EUR-terms objectives honestly**: if Naka finds the −36% relative drawdowns intolerable in EUR reality, change the KPI to EUR terms and re-run the review; do not bolt EUR breakers onto a BTC KPI. | The two numéraires cannot both be primary. | Naka | decision recorded, review re-run |
| P2-6 | **Reassess venue** only if fees remain the binding constraint at €1k; the desk's volume cannot reach the $10K tier. | Entry tier 0.60/0.40 is permanent at this size. | Naka | one-page comparison, if ever |

## Explicitly not recommended

- Adding leverage, alts, perpetuals, or a second venue to "make the 3% achievable". The KPI is the problem, not the toolset.
- Intraday signals or faster moving averages: the 50-day variants lost 8–70% to HODL in-sample and traded 8–20 times a year.
- Vol-targeting or any continuous exposure multiplier while the KPI is in BTC.
- Keeping the 20% EUR cash floor "for safety". It is a 20% short against the benchmark and the mechanical cause of hard fail B.
- Going live at €1,000 on day 91.
