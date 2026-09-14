# RISK — kill-switch critique and circuit breakers for a €1k EUR↔BTC spot desk on Coinbase Advanced Trade

Numbers are from `METHOD.md` (Coinbase BTC-EUR daily closes, 2016-01 to 2026-09). Venue facts were checked against Coinbase's public API and docs on 2026-09-14. Items marked **assumption** were not computed.

## 1. The rules as specified

| Rule | Denomination | Trigger | Action |
|---|---|---|---|
| KPI | BTC | < +3% stack growth in a month | "soft miss" → review |
| Hard fail A | BTC | 2 consecutive soft misses | freeze + propose shutdown |
| Hard fail B | BTC | any month ≤ −5% stack | freeze + propose shutdown |
| Circuit breaker | EUR | −8% month-to-date MTM equity | "stop new risk" |
| Structural | EUR | 20% cash floor | always held |
| Execution | — | every order | human "yes" required |

## 2. Critique

### 2.1 The desk has two numéraires and they fight

The KPI is in BTC; the breaker and the cash floor are in EUR. The safe position in BTC terms is 100% BTC (zero tracking error). The safe position in EUR terms is 100% EUR. No position is safe in both, so the two rule sets pull in opposite directions:

| Scenario | Position | BTC KPI | EUR breaker | Outcome |
|---|---|---|---|---|
| BTC −20% in month | 100% long | stack unchanged (0%) | fires at −8% | breaker fires when the KPI is *unaffected*; "stop new risk" does nothing useful or sells the bottom |
| BTC −20% in month | in EUR | stack +25% | silent | desk is a hero; nothing checks whether the flip was luck |
| BTC +30% in month | in EUR (all-out) | stack −23% → hard fail | silent | the KPI is destroyed and the only breaker is asleep |
| BTC +30% in month | 100% long | 0% (miss) | silent | a *miss* for doing exactly what the benchmark did |

The EUR breaker fires when it is irrelevant to the KPI and stays silent when the KPI is being destroyed. It is not a bad rule; it is a rule for a different desk.

### 2.2 The kill switch is a shutdown timer

- "2 consecutive misses" with a per-month hit rate of ~20–23% (best trend variants, in-sample) means P(two misses in a row) ≈ 0.6 per pair of months. Empirically, applying the locked rule to every strategy tested from every start month since 2016: killed from **~100% of start months, ~90% within 3 months, 100% within 8**. That includes the variants that beat HODL over the decade.
- Analytically (iid-normal monthly alpha; `METHOD.md` §2 and script section 3): a strategy with a *true* +3%/month mean alpha and 5%/month dispersion survives 12 months with **6.2%** probability. At +5%/month and 3% dispersion — an annualised Sharpe of roughly 5.8, which nobody has — **54%**. Real returns are fat-tailed and autocorrelated, which makes these numbers optimistic.
- Per-month noise dominates: BTC's median absolute monthly move is 11%. A rule that judges a 3–5-trades-a-year strategy on monthly buckets is measuring the market, not the desk.

### 2.3 The cash floor guarantees hard fail B

A permanent 20% EUR position loses in BTC terms whenever BTC rises. Static 80/20 rebalanced monthly: stack change `= 0.8 + 0.2/(1+r) − 1`. It hits ≤ −5% whenever BTC's month is **≥ +33.3%** — **16 of 127 months** since 2016 (2017-05/08/10/11/12, 2018-04, 2019-05, 2020-04/11/12, 2021-02/03/10, 2023-01, 2024-02, 2024-11). The desk hard-fails in those months with zero trading error, and hits the KPI only when BTC falls ≥ 13% (19 months). Over the sample the floor alone cost 48.5% of the stack (monthly rebalance) or 30% (rebalance on 10pp drift).

### 2.4 The −8% MTD EUR breaker fires on noise and after the fact

Applied to a fully long BTC book on daily closes: reached in **57 of 127 months (45%)**; −12% MTD in 37 (29%). In **42** of those months the breach came from a **single-day** drop ≤ −8% (68 such days since 2016; daily σ = 3.44%), so "stop new risk" arrives after the loss. After breaching, the month closed at or above the breach level **34** times vs below it **23** times: within the month, acting on the breach was wrong more often than right. Intraday breaches would be more frequent (median daily high–low range 3.71%; 311 days with ≥ 10% range).

### 2.5 "Stop new risk" and "freeze" are undefined

Freeze *where*? If the desk is in EUR when it freezes, in BTC terms it is frozen at maximum risk. If it is 100% BTC, it is frozen at zero risk vs its benchmark. The safe state must be named once; today every agent will assume a different one.

### 2.6 Human confirmation on risk *reduction*

"No auto-execute without explicit human yes" is correct for risk-increasing orders. Applied to Sentinel's STOP it turns the risk function into a notification service. Coinbase BTC-EUR trades 24/7; a −8% day at 03:00 on a Sunday (weekend volume is ~1/3 of weekday) will not wait for Naka to wake up.

### 2.7 No operational breakers

Nothing covers stale data, API errors, order rejects, fill-price deviation, ledger/exchange balance mismatch, exchange incidents, key permission drift, or clock drift. At €1k these are the failures that actually happen.

### 2.8 No go-live gates and no size ladder

"Paper until live" has no exit criteria and live starts at 100% of capital.

### 2.9 Custody and key hygiene incomplete

"No API keys in chat" is necessary and not sufficient: no statement on key permissions, portfolio scoping, IP allow-listing, 2FA, or withdrawal allow-list. EUR held at Coinbase's EU entity is a claim on that entity under e-money safeguarding rules, not an insured bank deposit; the 20% "cash" carries the same counterparty as the BTC (**assumption**: entity terms vary by country; verify).

### 2.10 Tax and reporting

Every BTC→EUR sale is a disposal. A BTC-denominated desk can be *down* in BTC and still owe tax on realised EUR gains. Example (**verify with an adviser**): Portugal taxes gains on crypto held under 365 days at 28% since 2023. EU DAC8 reporting by exchanges applies from 2026, so the trades will be visible to the tax authority regardless.

## 3. Better circuit breakers

### Design rules

1. **One numéraire.** The KPI says BTC. Benchmark = HODL. Risk = tracking error vs HODL. Safe state = **the rule's own neutral exposure**, not 0% and not a discretionary guess. The "how much EUR am I willing to have in BTC at all" question is answered *once*, by sizing the desk, outside the trading rules.
2. **Breakers act on process and pipeline failure, plus a tracking limit set where normal strategy behaviour rarely reaches it.** They do not act on BTC's own volatility. (A −10% tracking limit on an all-out trend rule would have fired in 36% of 90-day windows; that is a shutdown timer, not a breaker.)
3. **Risk-reducing actions are pre-authorised.** Risk-increasing actions need Naka's yes.
4. **Every breaker has a reset condition and an owner.**
5. **Every breaker is logged with the counterfactual** (what the rule would have done had the breaker not fired), so breakers can be scored.

### Layer 0 — per-order guards (Bolt, deterministic, every order)

| Guard | Setting | On failure |
|---|---|---|
| Order type | `limit_limit_gtc` with `post_only=true`; taker only as the final, logged escalation step | reject ticket |
| Notional cap | ≤ 50% of the active book (€400) per order; ≤ 100% per day | reject |
| Price sanity | limit price within ±1.0% of `best_bid_ask` mid at ticket time | reject, re-quote |
| Precision | size rounded to `base_increment` 1e-8 BTC, price to `quote_increment` €0.01; notional ≥ `min_market_funds` €0.84 | reject |
| Preview | `POST /orders/preview` must succeed and report commission; commission ≤ 0.60% | reject |
| Idempotency | unique `client_order_id` per ticket; duplicate id → refuse | reject |
| Balance | EUR/BTC available ≥ order size + fee | reject |
| Direction | order side must match today's logged signal and ticket | reject |
| Venue | never Simple Trade; never market order except escalation; never a product other than BTC-EUR | reject |

### Layer 1 — turnover and cadence (Ledger/Controller, daily)

| Guard | Setting | Action |
|---|---|---|
| Fee budget | ≤ 1.0% of equity per calendar month, fees + modelled spread | block further flips this month; log blocked flip; execute on the 1st if still signalled |
| Executions | ≤ 2 per day, ≤ 4 per rolling 30 days | block |
| Exposure change | ≤ 50pp of the active book per day (half-out design implies 50pp max anyway) | block |
| Signal freshness | last close used ≤ 26h old; data source reachable | no new orders |
| Trading window | orders only in the fixed daily window | no orders outside |

### Layer 2 — tracking drawdown vs HODL (BTC terms, Controller, daily)

For the half-out rule (`STRATEGY.md` §4). Levels are set from the rule's own historical 90-day behaviour so that normal operation rarely triggers them.

| Level | Trigger (from experiment/quarter start) | Action | Reset |
|---|---|---|---|
| Amber | −7.5% | no discretionary changes allowed; weekly review becomes daily until back above −5% | back above −5% |
| Red | **−15%** (≈1% of historical 90-day windows) | pre-authorised: move active book to **neutral = 75% BTC** (midpoint of the rule's two states); stop new orders; written diagnosis before any resumption; Naka decides resume/redesign | Naka signs a diagnosis |
| Black | −25% or a Red breach within 30 days of a previous Red | freeze at neutral; experiment ends; redesign required | new pre-registration |

If Naka insists on an all-out rule, the equivalent levels are −15% / −25% / −35% (all-out breaches −25% in ~2% of 90-day windows). Do not use −10% with all-out.

### Layer 3 — operational and venue guards (Controller, continuous)

| Guard | Setting | Action |
|---|---|---|
| API errors | ≥ 3 consecutive errors or any 5xx on order placement | halt orders; alert |
| Order rejected | any reject (`INSUFFICIENT_FUND`, precision, post-only cross, etc.) | halt; human review |
| Fill deviation | fill price > 0.5% from ticket limit, or partial fill leaving > 10pp exposure gap after the window | halt; reconcile |
| Reconciliation | ledger vs `/accounts` balances mismatch > €1 or > 0.00001 BTC | halt until explained |
| Exchange status | Coinbase status page reports a BTC-EUR/Advanced Trade incident | no new orders; existing orders cancelled if unfilled |
| Data integrity | daily candle missing or close deviates > 5% from a second public source at the same timestamp | no new orders |
| Key permissions | startup check: key has `view` + `trade` only, no transfer; key scoped to the desk's dedicated portfolio; IP allow-list active | refuse to start |
| Clock | system clock within 5s of NTP | refuse to start |
| Weekend | no *new* risk-increasing orders Sat/Sun (thin book); risk-reducing allowed | defer to Monday window |

### Layer 4 — experiment and go-live gates (Naka, monthly)

| Gate | Requirement |
|---|---|
| Paper → live eligibility | ≥ 90 days paper; all process KPIs pass; replication check within 0.5%; no unresolved Red; shadow signals scored |
| Live size ladder | €250 → €500 → €1,000. Step up only after 60 days at the current step with live fills within 0.3% of the paper fill model on average |
| Step down | any Red at live size → back one step; any Black → paper |
| Parameter changes | only at monthly review, in writing, never within 30 days of a Red, never mid-month; each change restarts the 90-day clock |
| Custody | before the first live order: 20% core moved to self-custody; exchange holds only the active book |
| Key hygiene | dedicated portfolio; `view`+`trade` only; IP allow-list; 2FA hardware key; withdrawal allow-list on the Coinbase account; keys rotated at each size step; keys never in chat, tickets, or logs |

### Replacement for the kill rule

Delete "≥3% BTC MoM or die". Replace with:

- **Paper:** the experiment stops on (a) Black, (b) any unresolved process failure > 7 days, or (c) replication mismatch. It does **not** stop on monthly performance.
- **Live:** as paper, plus the size ladder, plus a **personal solvency cap** set *outside* the desk: the maximum EUR value Naka is willing to hold in BTC at all (this is the honest home of the EUR worry). If BTC's EUR price falls such that the whole desk is below that cap, the *desk size* is reduced by Naka, not the rule.

### If Naka actually cares about EUR

Then say so and change the KPI: EUR-denominated return with a max drawdown and a fee budget, benchmark = 60/40 BTC/EUR or similar, and vol-targeting becomes a legitimate tool (it was a drag only in BTC terms). What does not work is a BTC KPI with EUR breakers. Pick one before day 1.

## 4. Coinbase Advanced Trade specifics (checked 2026-09-14)

| Topic | Fact | Consequence for the desk |
|---|---|---|
| Pair | `BTC-EUR` online; `base_increment` 1e-8, `quote_increment` €0.01, `min_market_funds` €0.84 | sizes are not a constraint at €1k; precision errors are (round before preview) |
| Fees | entry tier (< $10K trailing 30-day volume) 0.60% taker / 0.40% maker; tiers update hourly | the desk lives in the top tier permanently; maker-only is worth 0.20% per side |
| Liquidity | last 90 days median 217 BTC/day (~€14.6M) vs 6,212 BTC/day on BTC-USD (ratio 0.03); weekday median 267 vs weekend 92 BTC/day | fine for €400 clips; spreads and wicks matter; avoid weekend entries; always read `best_bid_ask` and log the spread |
| Stop orders | `stop_limit_stop_limit_gtc/gtd` trigger on **last trade price**; `trigger_bracket_*` place the stop-limit **5% away** when triggered | on a thin EUR book a single small print can trigger a stop; a stop-limit can be gapped through and never fill. For a daily-bar rule, prefer exposure rules to resting stops; if resting stops are used, they are catastrophe insurance with a wide limit, not P&L tools |
| Order preview | `POST /orders/preview` returns commission and explicit failure reasons (`PREVIEW_INSUFFICIENT_FUND`, `PREVIEW_INVALID_SIZE_PRECISION`, `PREVIEW_INVALID_LIMIT_PRICE_POST_ONLY`, …) | make preview mandatory in Bolt |
| Paper venue | sandbox (`api-sandbox.coinbase.com`) is **static mocked responses**, Accounts/Orders only, no live prices | there is no real paper venue; "paper" means self-simulated fills with an explicit fee + spread model, reconciled by replication |
| Portfolios / keys | `retail_portfolio_id` on orders; API keys can be created with `view`/`trade` permissions | run the desk in a dedicated portfolio; keys without transfer permission |
| Rate limits | WebSocket 8/s per IP; REST 429 with back-off | irrelevant at daily cadence; still implement back-off |
| 24/7 market | daily close is a convention; MTD breakers breach intraday | define "close" as 00:00 UTC and evaluate once; do not stare at intraday |
| EUR funding | SEPA; EUR held with Coinbase's EU entity under e-money safeguarding, not deposit insurance (**assumption**: verify entity and terms for the account's country) | the "cash" bucket has exchange counterparty risk; only the self-custodied core does not |
| Tax | each BTC→EUR sale is a disposal; DAC8 reporting from 2026 (**verify jurisdiction**) | ledger must produce a disposal list with EUR cost basis per lot |
