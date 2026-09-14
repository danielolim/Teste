# TEAM — roles, process, and a lean org chart

Scope: organisation and process quality, judged separately from edge (`STRATEGY.md`). A good process cannot create edge; a bad one reliably destroys it and, worse, hides where it went.

## 1. The pipeline as specified

```
Naka (Chief)
  └─ Atlas (CIO)
       ├─ Trend  (BTC momentum, €440)
       └─ Revert (mean reversion, €360)
            └─ Macro  (regime multiplier, 0% capital)
                 └─ Pulse (sentiment veto, 0% capital)
                      └─ Ledger (PM / KPI, merges to one BTC target + cash)
                           └─ Sentinel (risk STOP)
                                └─ Bolt (executor, human confirm)
```

Nine roles, eight sequential hand-offs, one asset, and at most one decision per day: *what fraction of €800 should be BTC tonight?* The output is a single number between 0 and 1.

## 2. Role-by-role

| Role | Stated job | What it actually contributes to tonight's number | Verdict |
|---|---|---|---|
| **Naka** (Chief) | owns the desk | the only real authority: says yes/no to tickets, owns the kill decision | **Keep.** Define the decisions Naka makes (ticket yes/no, weekly review, parameter changes, kill) or Naka is a rubber stamp for Bolt |
| **Atlas** (CIO) | allocates Trend 55 / Revert 45 | with fixed weights: nothing, Ledger merges anyway. With variable weights: an untested third strategy | **Remove.** |
| **Trend** | BTC momentum book | the one component with a coherent mechanism in BTC terms | **Keep as a signal function**, not a capital owner (`STRATEGY.md` §2.1) |
| **Revert** | mean-reversion book | fights fees and the asset's drift; catastrophic in BTC terms in the naive test | **Demote to shadow.** Zero capital, zero authority, logged daily, scored at day 90 |
| **Macro** | regime multiplier | the most powerful actor in the desk: one 0.5 halves everyone else's work, with 0% capital, no specified inputs, no accountability | **Remove as multiplier.** Shadow at most |
| **Pulse** | sentiment veto | can block, cannot initiate, is never scored because the counterfactual is not logged | **Demote to shadow** with one pre-registered input and rule |
| **Ledger** | PM / KPI | the single source of truth for positions, fees, benchmark and KPI | **Keep and enlarge**: owns BTC-terms accounting, HODL benchmark, fee budget, replication check, disposal list |
| **Sentinel** | risk STOP | today a notification service: its STOP still waits for a human yes | **Keep, merge with Ledger into "Controller"**, pre-authorised to cut to the safe state |
| **Bolt** | executor | builds and submits orders | **Keep as a deterministic order builder** with the Layer-0 checklist (`RISK.md` §3); no discretion |

## 3. Structural problems

1. **Attribution is impossible.** When a month goes badly, the loss could be Trend's signal, Revert's offsetting signal, Macro's 0.5, Pulse's veto, Ledger's merge, or Naka's 14-hour delay confirming the ticket. Nobody owns it; everybody will explain it.
2. **Authority without capital.** Macro and Pulse can override capital-bearing agents while owning none. Zero skin in the game plus unspecified inputs equals unfalsifiable. Their track record can never be computed, so they can never be fired.
3. **Two books, one asset.** Trend and Revert will regularly disagree by construction (one buys strength, the other sells it). The merged target churns, fees are paid on the churn, and the "books" are fiction because only the net position exists on the exchange.
4. **Serial latency.** Eight hand-offs between a 00:00 UTC close and an order. In a 24/7 market with a 3.4% daily σ that is a real cost, and it makes every stop conditional on a human being awake.
5. **Swarm envy.** Agent-portfolio swarms earn their overhead through breadth: many assets, many uncorrelated rules, specialisation. This desk has one asset and, after review, one rule. The swarm is process theater around a moving average.
6. **Monthly "review" on a 77% miss rate** (`STRATEGY.md` §3) guarantees a review almost every month, which guarantees parameter drift. The org has no rule against changing rules.
7. **Human confirm is applied to the wrong side.** Entries can wait for Naka. Exits and risk cuts cannot.

## 4. Recommended lean org chart

```
Naka — Human Principal
  approves risk-increasing tickets, chairs weekly/monthly review,
  signs parameter changes and diagnoses, owns kill and size ladder
      │
      ├─ Rules (Signal)            one pre-registered rule → target exposure
      │                             (absorbs Trend; nothing else has authority)
      │
      ├─ Controller (Ledger+Sentinel)
      │      positions, fees, HODL benchmark, KPI, replication check,
      │      Layers 1–3 breakers; PRE-AUTHORISED to cut to neutral
      │
      ├─ Bolt (Executor)          deterministic ticket builder + Layer-0 checklist;
      │                             executes Controller risk-cuts automatically,
      │                             executes risk-increasing tickets only on Naka's yes
      │
      └─ Shadow Research (no authority, no capital)
             Revert / Macro / Pulse would-be signals logged daily; scored at day 90
```

Four seats with authority, one without. Every daily decision has exactly one owner.

### RACI

| Decision / artefact | Naka | Rules | Controller | Bolt | Shadow |
|---|---|---|---|---|---|
| Daily target exposure | I | **R/A** | C | I | I |
| Order ticket (build, preview, Layer-0) | A (risk-up only) | I | C | **R** | — |
| Risk-reducing cut to neutral | I (after) | — | **A** | **R** | — |
| Breaker states, fee budget | I | — | **R/A** | I | — |
| Ledger, KPI, benchmark, disposal list | I | — | **R/A** | C | — |
| Replication check (independent recompute) | **A** | — | R | — | — |
| Weekly / monthly review | **R/A** | C | C | C | C |
| Parameter or rule change | **A** | R | C | I | — |
| Shadow scoring at day 90 | **A** | — | C | — | **R** |
| Go-live, size ladder, kill | **R/A** | — | C | — | — |

R = does it, A = accountable, C = consulted, I = informed.

### Guardrails on the org itself

- **Rule changes only at the monthly review, in writing, never within 30 days of a Red breaker, never mid-month.** Each change restarts the 90-day experiment clock.
- **Every override is logged as a rule violation**, even if it made money. Three in 90 days fails the process KPI.
- **Shadow agents get authority only by scoring**: at day 90, a shadow signal earns a *proposal* (not a promotion) if its logged counterfactual would have improved BTC-terms results after fees without increasing the −5% month count. Naka decides; a further 90 days shadow-with-teeth before capital.
- **No new roles.** If a job needs doing, it goes to one of the four seats.

## 5. Cadence

| When | Who | Minutes | Content |
|---|---|---|---|
| Daily, fixed time after 00:00 UTC close | Rules → Controller → Bolt → Naka | 10 | data freshness, signal, target vs current, breaker states, ticket or "no trade", shadow logs |
| Weekly (e.g., Sunday) | Naka + Controller | 30 | week's log, fee spend vs budget, tracking vs HODL, open breakers, any anomalies; **no parameter talk** |
| Monthly (1st) | Naka + all seats | 60 | KPI table, replication check, breaker counterfactuals, shadow scores, written parameter proposals if any |
| Day 90 | Naka | — | decision per `STRATEGY.md` §3 |

## 6. What good looks like after 90 days

- Every position change traces to a rule id, a timestamp, a ticket, a fill model and a ledger line.
- Zero unlogged overrides; ≤ 2 logged ones.
- The independent recomputation matches the ledger within 0.5% of equity.
- Each shadow agent has a scored counterfactual, in BTC terms after fees.
- Naka can state the desk's numéraire, benchmark, safe state and kill rule in four sentences without notes.
