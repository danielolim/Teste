# METHOD — data, code, and caveats behind every number in this review

All quantitative statements in `REPORT.md`, `STRATEGY.md`, `RISK.md`, `PLAYBOOK.md` and `CHANGES.md` come from the two scripts below, run on **2026-09-14**. Nothing else was computed. Where a document says "assumption" or "qualitative", no computation backs it.

## 1. Data

| Item | Value |
|---|---|
| Source | Coinbase Exchange public REST API, `GET https://api.exchange.coinbase.com/products/{BTC-EUR,BTC-USD}/candles?granularity=86400` (no auth) |
| Bars | Daily, UTC. Fields: time, low, high, open, close, volume (base units) |
| BTC-EUR range downloaded | 2015-04-23 .. 2026-09-14 (4,160 bars) |
| BTC-EUR range **used** | 2016-01-01 .. 2026-09-14 (3,909 bars). 2015 excluded: launch-year liquidity was negligible (avg 402 BTC/day in 2015 but the first weeks print 0.01–0.03 BTC/day and single-print candles) |
| Full calendar months used | 2016-02 .. 2026-08 (n = 127). 2026-09 is partial and excluded |
| BTC-USD | Used only for the liquidity comparison (last 90 days) |
| Product metadata | `GET /products/BTC-EUR` on 2026-09-14: `base_increment 0.00000001`, `quote_increment 0.01`, `min_market_funds 0.84` (EUR), `status online`, `trading_disabled false` |

### Venue facts checked on 2026-09-14 (public docs, not computed)

- Advanced Trade fee tiers: entry tier ($0–$10K trailing 30-day USD volume) = **0.60% taker / 0.40% maker**; next tier ($10K–$50K) 0.40% / 0.25%. Tiers update hourly; non-USD volume converted to USD; stable-pair volume excluded. Source: Coinbase Help "Advanced Trade fees" / "Exchange fees" pages.
- Order types available via `POST /api/v3/brokerage/orders`: `market_market_ioc`, `limit_limit_gtc/gtd` (with `post_only`), `limit_limit_fok`, `sor_limit_ioc`, `twap_limit_gtd`, `stop_limit_stop_limit_gtc/gtd` (trigger = **last trade price**, `STOP_DIRECTION_STOP_UP/DOWN`), `trigger_bracket_gtc/gtd`, and attached TP/SL via `attached_order_configuration`. Bracket trigger, when hit, places a stop-limit with the limit price **5% away** (higher for buys, lower for sells). `POST /orders/preview` exists and returns commission and failure reasons.
- Sandbox: `https://api-sandbox.coinbase.com` returns **static, pre-defined responses**, Accounts and Orders endpoints only, no authentication, no live pricing. It is **not** a paper-trading venue.
- Rate limits: WebSocket 8 connections and 8 unauthenticated messages per second per IP. REST limits are documented per-hour at the Coinbase App level; irrelevant at this desk's cadence but 429 back-off must be implemented.

## 2. Definitions

- **Stack (BTC terms)**: `stack_t = BTC_t + EUR_t / close_t`. HODL has stack growth 0 by construction; therefore "stack growth" ≡ outperformance vs HODL, after fees.
- **Static 80/20 book** (analytic, monthly rebalance): monthly stack change `= 0.8 + 0.2/(1+r) − 1` where `r` is BTC-EUR's monthly return. KPI hit (≥ +3%) requires `r ≤ −13.0%`; hard fail (≤ −5%) requires `r ≥ +33.3%`.
- **Locked kill rule (as specified by the desk)**: hard fail on 2 consecutive months < +3%, or any month ≤ −5%, all in BTC terms.
- **Kill survival (empirical)**: apply the locked rule to a strategy's monthly BTC-terms series starting from *every* month in the sample; report the share of start months killed within 3 and 6 months, and the maximum survival time.
- **Kill survival (analytic)**: two-state Markov chain over months with iid normal monthly alpha `N(μ, σ)`; P(miss) = Φ((3% − μ)/σ), P(catastrophe) = Φ((−5% − μ)/σ). Illustrative only; real returns are fat-tailed and autocorrelated, which makes survival worse, not better.
- **90-day window statistics** (§8 of the script): for every start day after warm-up, the rule's BTC-terms return over the next 90 days and the worst drawdown from the window start; used to set the experiment's tracking limit where normal behaviour rarely reaches it.
- **Backtest mechanics**: target BTC fraction decided on close *t*, executed at close *t+1* (next bar); fee charged on traded notional each side at 0.40% and 0.60%; no spread or slippage modelled; rebalance only when |target − current| > 10pp; sub-€1 orders ignored; strategies start 100% BTC so stack starts at 1.00 = HODL. Warm-up bars excluded from the reported period.

## 3. Caveats (read before quoting any number)

1. **In-sample, single path, no optimisation** — but also no cherry-picking: every parameter tried is reported (trend: SMA 50/100/200 × band 0%/3%; mean reversion: z-score(20) at ±2 with two bases; static 80/20; two hybrids; the SMA100 ±3% rule with OFF-state exposure 0% and 50%). 2 of 6 trend variants beat HODL at 0.40%, 1 of 6 at 0.60%. That is the whole grid; the half-out OFF level (50%) was tested once, not scanned.
2. **No spread, slippage or partial-fill modelling.** Real BTC-EUR on Coinbase will be worse than shown, especially on weekends (median weekend volume is ~1/3 of weekday volume).
3. **Daily UTC closes are a convention.** BTC trades 24/7. "Month-to-date" breakers were evaluated on daily closes; intraday breaches would be more frequent (median intraday range is 3.71%).
4. **Venue survivorship**: Coinbase BTC-EUR since 2016 is one venue's price series; it includes 2017 illiquid-ish months (still hundreds of BTC/day).
5. **No taxes.** Every BTC→EUR sale is a taxable disposal in most EU jurisdictions.
6. **Mean-reversion results are for one naive parameterisation.** They demonstrate the direction-and-fee problem in BTC terms; they do not prove that no MR rule can work. The burden of proof is on the desk.
7. **The analytic survival table assumes iid normality.** Real BTC-terms alpha is fat-tailed and regime-dependent; that lowers survival further.
8. The 90-day liquidity window is 2026-06-16 .. 2026-09-13 and reflects current, not historical, depth.

## 4. Scripts

### 4.1 `fetch_candles.py`

```python
"""Fetch full daily candle history for a Coinbase Exchange product (public API, no auth).
Coinbase returns max 300 candles per request; we page backwards in 300-day windows.
Output: CSV with columns date,open,high,low,close,volume (UTC daily bars)."""
import csv, json, sys, time, urllib.request
from datetime import datetime, timedelta, timezone

product = sys.argv[1] if len(sys.argv) > 1 else "BTC-EUR"
out = sys.argv[2] if len(sys.argv) > 2 else f"/tmp/review/{product}_daily.csv"
start = datetime(2015, 1, 1, tzinfo=timezone.utc)
end = datetime.now(timezone.utc).replace(hour=0, minute=0, second=0, microsecond=0)

rows = {}
cur_end = end
while cur_end > start:
    cur_start = max(start, cur_end - timedelta(days=299))
    url = (f"https://api.exchange.coinbase.com/products/{product}/candles"
           f"?granularity=86400&start={cur_start.isoformat()}&end={cur_end.isoformat()}")
    req = urllib.request.Request(url, headers={"User-Agent": "desk-review/1.0"})
    for attempt in range(5):
        try:
            with urllib.request.urlopen(req, timeout=30) as r:
                data = json.loads(r.read().decode())
            break
        except Exception as e:  # noqa: BLE001
            time.sleep(1.5 * (attempt + 1))
            data = None
    if data is None:
        print("failed window", cur_start, cur_end, file=sys.stderr)
        sys.exit(1)
    for t, low, high, o, c, v in data:
        d = datetime.fromtimestamp(t, tz=timezone.utc).date().isoformat()
        rows[d] = (o, high, low, c, v)
    cur_end = cur_start - timedelta(days=1)
    time.sleep(0.35)  # public rate limit is ~10 req/s; be polite

with open(out, "w", newline="") as f:
    w = csv.writer(f)
    w.writerow(["date", "open", "high", "low", "close", "volume"])
    for d in sorted(rows):
        w.writerow([d, *rows[d]])
print(f"{product}: {len(rows)} daily bars, {min(rows)} .. {max(rows)} -> {out}")
```

### 4.2 `analyze.py`

```python
"""Desk-review computations on Coinbase BTC-EUR daily closes (UTC). Stdlib only.

Everything here is in BTC terms unless stated: stack_t = BTC held + EUR held / close_t.
HODL has BTC-terms return 0 by construction, so "stack growth" == outperformance vs HODL.
"""
import csv, math, statistics
from collections import defaultdict
from datetime import date

START = "2016-01-01"   # 2015 launch months are illiquid (see volume table) -> excluded
rows = [r for r in csv.DictReader(open("/tmp/review/BTC-EUR_daily.csv")) if r["date"] >= START]
dates = [r["date"] for r in rows]
close = [float(r["close"]) for r in rows]
vol = [float(r["volume"]) for r in rows]
N = len(close)
print(f"bars: {N}  {dates[0]} .. {dates[-1]}")

# ---------- 1. monthly BTC-EUR returns ----------
month_last = {}
for i, d in enumerate(dates):
    month_last[d[:7]] = i
months = sorted(month_last)
m_ret = []
for a, b in zip(months, months[1:]):
    m_ret.append((b, close[month_last[b]] / close[month_last[a]] - 1))
# drop current partial month
m_ret = [x for x in m_ret if x[0] < dates[-1][:7]]
rets = [r for _, r in m_ret]
print(f"\n== Monthly BTC-EUR returns, {m_ret[0][0]} .. {m_ret[-1][0]}  (n={len(rets)})")
print(f"median abs monthly move: {statistics.median(abs(r) for r in rets):.1%}")
print(f"std of monthly returns:  {statistics.pstdev(rets):.1%}")
for thr in (0.10, 0.20, 0.25, 0.333, 0.40):
    print(f"months >= +{thr:.1%}: {sum(r >= thr for r in rets):3d}   months <= -{thr:.1%}: {sum(r <= -thr for r in rets):3d}")
big_up = [(m, r) for m, r in m_ret if r >= 0.333]
print("months >= +33.3%:", ", ".join(f"{m} {r:+.0%}" for m, r in big_up))

# ---------- 2. structural effect of a 20% EUR cash floor, in BTC terms ----------
# static 80% BTC / 20% EUR, rebalanced monthly: stack_ret = 0.8 + 0.2/(1+r) - 1
s = [0.8 + 0.2 / (1 + r) - 1 for r in rets]
print("\n== Static 80/20 (BTC/EUR) book, BTC-terms monthly stack change")
print(f"months >= +3% (KPI hit):        {sum(x >= 0.03 for x in s):3d} / {len(s)}  (needs BTC <= -13.0% in the month)")
print(f"months <= -5% (hard fail):      {sum(x <= -0.05 for x in s):3d} / {len(s)}  (needs BTC >= +33.3% in the month)")
print(f"months in (-5%, +3%) (miss):    {sum(-0.05 < x < 0.03 for x in s):3d} / {len(s)}")
print(f"mean monthly stack change:      {statistics.mean(s):+.2%}   cumulative: {math.prod(1 + x for x in s) - 1:+.1%}")

# ---------- 3. kill-switch survival, analytic Markov model ----------
def Phi(x):
    return 0.5 * (1 + math.erf(x / math.sqrt(2)))

def p_survive(mu, sd, horizon=12, thr=0.03, cat=-0.05):
    """P(no hard fail within `horizon` months). Hard fail = 2 consecutive misses (<thr) OR any month <= cat.
    Assumes iid normal monthly BTC-terms alpha. Illustrative only."""
    p_cat = Phi((cat - mu) / sd)
    p_miss_ok = Phi((thr - mu) / sd) - p_cat  # miss but not catastrophic
    p_hit = 1 - p_cat - p_miss_ok
    # states: 0 = no pending miss, 1 = one miss pending
    st = {0: 1.0, 1: 0.0}
    for _ in range(horizon):
        nx = {0: 0.0, 1: 0.0}
        nx[0] += st[0] * p_hit + st[1] * p_hit
        nx[1] += st[0] * p_miss_ok
        st = nx
    return st[0] + st[1]

print("\n== P(desk survives 12 months under locked kill rules), iid-normal monthly BTC-terms alpha")
sds = (0.03, 0.05, 0.08, 0.12)
print("mean alpha/mo  " + "  ".join(f"sd={sd:.0%}" for sd in sds))
for mu in (0.0, 0.01, 0.02, 0.03, 0.04, 0.05):
    print(f"   {mu:+.0%}        " + "  ".join(f"{p_survive(mu, sd):5.1%}" for sd in sds))
print("P(survive 3 months), same grid:")
for mu in (0.0, 0.01, 0.02, 0.03, 0.04, 0.05):
    print(f"   {mu:+.0%}        " + "  ".join(f"{p_survive(mu, sd, horizon=3):5.1%}" for sd in sds))

# ---------- 4. sanity-check backtests, BTC terms, with fees ----------
def sma(x, n):
    out = [None] * len(x); s_ = 0.0
    for i, v in enumerate(x):
        s_ += v
        if i >= n: s_ -= x[i - n]
        if i >= n - 1: out[i] = s_ / n
    return out

def run(target_fn, fee, warmup):
    """target_fn(i) -> desired BTC fraction of book in [0,1] decided on close i; executed at close i+1 (next bar).
    Returns daily stack (BTC terms, start 1.0) and trade count. Fee charged on traded notional, each side."""
    btc, eur = 1.0, 0.0  # start fully in BTC so BTC-terms start = 1 (benchmark = HODL)
    stack = [1.0] * len(close); trades = 0; pending = None
    for i in range(len(close)):
        px = close[i]
        if pending is not None:
            cur_val_btc = btc + eur / px
            want_btc = pending * cur_val_btc
            delta = want_btc - btc
            if abs(delta) * px > 1.0:  # ignore sub-euro rebalances
                if delta > 0:  # buy BTC with EUR
                    cost_eur = delta * px
                    if cost_eur > eur: delta = eur / px; cost_eur = eur
                    btc += delta * (1 - fee); eur -= cost_eur
                else:           # sell BTC for EUR
                    btc += delta; eur += -delta * px * (1 - fee)
                trades += 1
            pending = None
        stack[i] = btc + eur / px
        if i >= warmup:
            t = target_fn(i)
            cur = btc / (btc + eur / px)
            if abs(t - cur) > 0.10:  # only act on material exposure change
                pending = t
    return stack, trades

def monthly_from_stack(stack):
    out = []
    for a, b in zip(months, months[1:]):
        if b < dates[-1][:7]:
            out.append((b, stack[month_last[b]] / stack[month_last[a]] - 1))
    return out

def kill_survival(mrets):
    """From every possible start month, months until locked kill rule fires (None = survived to end)."""
    lives = []
    for s0 in range(len(mrets)):
        consec = 0; fired = None
        for k, (_, r) in enumerate(mrets[s0:]):
            if r <= -0.05: fired = k + 1; break
            if r < 0.03:
                consec += 1
                if consec >= 2: fired = k + 1; break
            else:
                consec = 0
        lives.append(fired)
    fired = [x for x in lives if x is not None]
    within3 = sum(1 for x in fired if x <= 3) / len(lives)
    within6 = sum(1 for x in fired if x <= 6) / len(lives)
    return len(fired), len(lives), statistics.median(fired) if fired else None, max(fired) if fired else None, within3, within6

def max_dd(stack):
    peak = stack[0]; dd = 0.0
    for v in stack:
        peak = max(peak, v); dd = min(dd, v / peak - 1)
    return dd

def report(name, stack, trades, warmup_idx):
    mr = monthly_from_stack(stack)
    mr = [(m, r) for m, r in mr if month_last[m] > warmup_idx]  # ignore warmup months
    rs = [r for _, r in mr]
    years = (len(close) - warmup_idx) / 365.25
    final = stack[-1] / stack[warmup_idx]
    n_fired, n_starts, med, mx, w3, w6 = kill_survival(mr)
    print(f"{name:40s} stack x{final:5.2f} ({(final ** (1 / years) - 1):+.1%}/yr BTC-terms) "
          f"maxDD-vs-HODL {max_dd(stack[warmup_idx:]):+.0%}  trades/yr {trades / years:4.1f}  "
          f"mo>=+3%: {sum(r >= 0.03 for r in rs):3d}/{len(rs)}  mo<=-5%: {sum(r <= -0.05 for r in rs):3d}  "
          f"kill: {n_fired}/{n_starts} starts, within 3mo {w3:.0%}, within 6mo {w6:.0%}, max survival {mx} mo")

print("\n== Sanity-check strategies, BTC terms, next-close execution, fee per side as stated. IN-SAMPLE, NOT OPTIMISED.")
print("   (HODL = 1.00x by definition. 'kill fires' applies the locked rule from every possible start month.)")
for fee in (0.004, 0.006):
    print(f"\n-- fee {fee:.1%} per side --")
    for n in (50, 100, 200):
        for band in (0.0, 0.03):
            S = sma(close, n)
            state = {"x": 1.0}
            def tf(i, S=S, band=band, state=state):
                if close[i] > S[i] * (1 + band): state["x"] = 1.0
                elif close[i] < S[i] * (1 - band): state["x"] = 0.0
                return state["x"]
            st, tr = run(tf, fee, n)
            report(f"Trend SMA{n} band±{band:.0%}", st, tr, n)
    # mean reversion, two bases. z = (close - SMA20)/std20
    S20 = sma(close, 20)
    def z(i):
        w = close[i - 19:i + 1]; sd = statistics.pstdev(w)
        return (close[i] - S20[i]) / sd if sd > 0 else 0.0
    st_b = {"x": 1.0}
    def mr_btc_base(i):   # default hold BTC; sell to EUR when stretched up, buy back when back to mean
        zz = z(i)
        if zz > 2.0: st_b["x"] = 0.0
        elif zz < 0.0: st_b["x"] = 1.0
        return st_b["x"]
    st, tr = run(mr_btc_base, fee, 20); report("MeanRev z20 (BTC base, sell tops)", st, tr, 20)
    st_e = {"x": 0.0}
    def mr_eur_base(i):   # default hold EUR; buy BTC when stretched down, sell when back to mean
        zz = z(i)
        if zz < -2.0: st_e["x"] = 1.0
        elif zz > 0.0: st_e["x"] = 0.0
        return st_e["x"]
    st, tr = run(mr_eur_base, fee, 20); report("MeanRev z20 (EUR base, buy dips)", st, tr, 20)
    # static 80/20 rebalanced when drift > 10pp (the desk's cash floor, on its own)
    st, tr = run(lambda i: 0.8, fee, 1); report("Static 80/20 (cash floor alone)", st, tr, 1)
    # hybrid: SMA100 band 3% trend; on trend-ON wait for a pullback (z20 < 0) up to 10 days before buying
    S100 = sma(close, 100)
    hy = {"trend": 1.0, "x": 1.0, "wait": 0}
    def hybrid(i):
        if close[i] > S100[i] * 1.03: new = 1.0
        elif close[i] < S100[i] * 0.97: new = 0.0
        else: new = hy["trend"]
        if new != hy["trend"]:
            hy["trend"] = new; hy["wait"] = 0
        if hy["trend"] == 0.0:
            hy["x"] = 0.0
        elif hy["x"] == 0.0:
            hy["wait"] += 1
            if z(i) < 0.0 or hy["wait"] > 10: hy["x"] = 1.0
        return hy["x"]
    st, tr = run(hybrid, fee, 100); report("Trend SMA100 ±3% + pullback entry (z<0)", st, tr, 100)
    # vol-target multiplier on top of SMA100 ±3% trend: exposure = trend * min(1, 60% / realised vol 30d)
    vt = {"x": 1.0}
    def voltarget(i):
        if close[i] > S100[i] * 1.03: vt["x"] = 1.0
        elif close[i] < S100[i] * 0.97: vt["x"] = 0.0
        w = [math.log(close[k] / close[k - 1]) for k in range(i - 29, i + 1)]
        rv = statistics.pstdev(w) * math.sqrt(365)
        return vt["x"] * min(1.0, 0.60 / rv) if rv > 0 else vt["x"]
    st, tr = run(voltarget, fee, 100); report("Trend SMA100 ±3% x vol-target 60%", st, tr, 100)

# ---------- 5. liquidity: BTC-EUR vs BTC-USD on Coinbase, last 90 full days ----------
usd = {r["date"]: float(r["volume"]) for r in csv.DictReader(open("/tmp/review/BTC-USD_daily.csv"))}
last90 = dates[-91:-1]
ve = statistics.median(vol[dates.index(d)] for d in last90)
vu = statistics.median(usd[d] for d in last90 if d in usd)
print(f"\n== Coinbase liquidity, median daily volume last 90 days ({last90[0]}..{last90[-1]}):")
print(f"BTC-EUR {ve:8.1f} BTC/day (~EUR {ve * close[-1] / 1e6:.1f}M)   BTC-USD {vu:8.1f} BTC/day   ratio EUR/USD = {ve / vu:.2f}")
# weekend vs weekday
wk = [vol[dates.index(d)] for d in last90 if date.fromisoformat(d).weekday() < 5]
we = [vol[dates.index(d)] for d in last90 if date.fromisoformat(d).weekday() >= 5]
print(f"BTC-EUR weekday median {statistics.median(wk):.1f}  weekend median {statistics.median(we):.1f}  (BTC/day)")

# ---------- 6. daily gap stats (for stop-order realism) ----------
dr = [close[i] / close[i - 1] - 1 for i in range(1, N)]
print(f"\n== Daily close-to-close moves since {START}: sd {statistics.pstdev(dr):.2%}, "
      f"days <= -8%: {sum(r <= -0.08 for r in dr)}, days <= -5%: {sum(r <= -0.05 for r in dr)}, days >= +8%: {sum(r >= 0.08 for r in dr)}")
lows = [float(r["low"]) for r in rows]; highs = [float(r["high"]) for r in rows]
intraday = [(highs[i] - lows[i]) / close[i] for i in range(N)]
print(f"median intraday high-low range: {statistics.median(intraday):.2%}; days with range >= 10%: {sum(x >= 0.10 for x in intraday)}")

# ---------- 7. the desk's -8% MTD EUR breaker, applied to a fully long BTC book ----------
by = defaultdict(list)
for d, c in zip(dates, close):
    by[d[:7]].append(c)
full_months = sorted(by)[:-1]
prev = None; n_m = 0; hit8 = 0; hit12 = 0; single_day = 0; closed_below = 0; closed_at_or_above = 0
for m in full_months:
    cl = by[m]
    if prev is None:
        prev = cl[-1]; continue
    n_m += 1
    mtd = [c / prev - 1 for c in cl]
    if min(mtd) <= -0.08:
        hit8 += 1
        breach = next(c for c in cl if c / prev - 1 <= -0.08)
        if cl[-1] < breach: closed_below += 1
        else: closed_at_or_above += 1
    if min(mtd) <= -0.12: hit12 += 1
    seq = [prev] + cl
    if any(seq[i] / seq[i - 1] - 1 <= -0.08 for i in range(1, len(seq))): single_day += 1
    prev = cl[-1]
print(f"\n== -8% MTD (EUR terms) breaker on a 100% BTC book, {n_m} months:")
print(f"months reaching -8% MTD: {hit8} ({hit8 / n_m:.0%});  -12% MTD: {hit12} ({hit12 / n_m:.0%});  "
      f"months containing a single-day drop <= -8%: {single_day}")
print(f"after the -8% breach, month closed below breach level: {closed_below};  at/above breach level: {closed_at_or_above}")

# ---------- 8. 90-day window behaviour of the recommended rule (for setting experiment limits) ----------
def pct(xs, q):
    xs = sorted(xs); k = (len(xs) - 1) * q; f = math.floor(k); c = min(f + 1, len(xs) - 1)
    return xs[f] + (xs[c] - xs[f]) * (k - f)

S100 = sma(close, 100)
for label, off_exposure in (("all-out (OFF = 0% BTC)", 0.0), ("half-out (OFF = 50% BTC)", 0.5)):
    for fee in (0.004, 0.006):
        stt = {"x": 1.0}
        def rule(i, stt=stt, off=off_exposure):
            if close[i] > S100[i] * 1.03: stt["x"] = 1.0
            elif close[i] < S100[i] * 0.97: stt["x"] = off
            return stt["x"]
        st, tr = run(rule, fee, 100)
        report(f"SMA100 ±3% {label}", st, tr, 100)
        W = 90; rets90 = []; dd_breach = {0.10: 0, 0.15: 0, 0.20: 0, 0.25: 0, 0.30: 0}; nwin = 0
        for s0 in range(100, len(st) - W):
            seg = st[s0:s0 + W + 1]
            rets90.append(seg[-1] / seg[0] - 1)
            worst = min(v / seg[0] - 1 for v in seg)  # drawdown from window start (experiment start)
            nwin += 1
            for k in dd_breach:
                if worst <= -k: dd_breach[k] += 1
        print(f"   90-day windows (n={nwin}, fee {fee:.1%}): rel. return p5 {pct(rets90, .05):+.1%}  p25 {pct(rets90, .25):+.1%}  "
              f"p50 {pct(rets90, .5):+.1%}  p75 {pct(rets90, .75):+.1%}  p95 {pct(rets90, .95):+.1%}  min {min(rets90):+.1%}  max {max(rets90):+.1%}")
        print("   share of windows whose DD-from-start breached: " +
              "  ".join(f"-{k:.0%}: {v / nwin:.0%}" for k, v in dd_breach.items()))
```

Run: `python3 fetch_candles.py BTC-EUR && python3 fetch_candles.py BTC-USD && python3 analyze.py`. Python 3.12, standard library only.

## 5. Raw output (2026-09-14)

```text
bars: 3909  2016-01-01 .. 2026-09-14

== Monthly BTC-EUR returns, 2016-02 .. 2026-08  (n=127)
median abs monthly move: 11.0%
std of monthly returns:  20.9%
months >= +10.0%:  45   months <= -10.0%:  23
months >= +20.0%:  29   months <= -20.0%:   7
months >= +25.0%:  23   months <= -25.0%:   5
months >= +33.3%:  16   months <= -33.3%:   4
months >= +40.0%:   9   months <= -40.0%:   0
months >= +33.3%: 2017-05 +63%, 2017-08 +66%, 2017-10 +52%, 2017-11 +61%, 2017-12 +35%, 2018-04 +36%, 2019-05 +63%, 2020-04 +35%, 2020-11 +39%, 2020-12 +44%, 2021-02 +37%, 2021-03 +34%, 2021-10 +40%, 2023-01 +38%, 2024-02 +43%, 2024-11 +41%

== Static 80/20 (BTC/EUR) book, BTC-terms monthly stack change
months >= +3% (KPI hit):         19 / 127  (needs BTC <= -13.0% in the month)
months <= -5% (hard fail):       16 / 127  (needs BTC >= +33.3% in the month)
months in (-5%, +3%) (miss):     92 / 127
mean monthly stack change:      -0.45%   cumulative: -48.5%

== P(desk survives 12 months under locked kill rules), iid-normal monthly BTC-terms alpha
mean alpha/mo  sd=3%  sd=5%  sd=8%  sd=12%
   +0%         0.0%   0.1%   0.2%   0.2%
   +1%         0.2%   0.6%   0.5%   0.3%
   +2%         1.7%   2.1%   1.2%   0.6%
   +3%         9.0%   6.2%   2.7%   1.1%
   +4%        27.4%  14.4%   5.2%   1.8%
   +5%        54.1%  27.3%   9.3%   3.0%
P(survive 3 months), same grid:
   +0%        16.4%  23.7%  23.8%  21.8%
   +1%        28.7%  33.4%  30.0%  25.6%
   +2%        44.7%  44.4%  36.8%  29.7%
   +3%        62.0%  55.8%  43.9%  34.1%
   +4%        77.5%  66.8%  51.3%  38.7%
   +5%        88.8%  76.4%  58.5%  43.4%

== Sanity-check strategies, BTC terms, next-close execution, fee per side as stated. IN-SAMPLE, NOT OPTIMISED.
   (HODL = 1.00x by definition. 'kill fires' applies the locked rule from every possible start month.)

-- fee 0.4% per side --
Trend SMA50 band±0%                      stack x 0.45 (-7.2%/yr BTC-terms) maxDD-vs-HODL -73%  trades/yr 19.9  mo>=+3%:  32/127  mo<=-5%:  39  kill: 127/127 starts, within 3mo 91%, within 6mo 100%, max survival 5 mo
Trend SMA50 band±3%                      stack x 0.92 (-0.8%/yr BTC-terms) maxDD-vs-HODL -70%  trades/yr  7.6  mo>=+3%:  28/127  mo<=-5%:  36  kill: 126/127 starts, within 3mo 92%, within 6mo 99%, max survival 5 mo
Trend SMA100 band±0%                     stack x 1.17 (+1.6%/yr BTC-terms) maxDD-vs-HODL -59%  trades/yr 13.8  mo>=+3%:  28/125  mo<=-5%:  31  kill: 125/125 starts, within 3mo 91%, within 6mo 100%, max survival 6 mo
Trend SMA100 band±3%                     stack x 1.44 (+3.5%/yr BTC-terms) maxDD-vs-HODL -58%  trades/yr  4.6  mo>=+3%:  29/125  mo<=-5%:  28  kill: 125/125 starts, within 3mo 90%, within 6mo 100%, max survival 6 mo
Trend SMA200 band±0%                     stack x 0.53 (-6.1%/yr BTC-terms) maxDD-vs-HODL -81%  trades/yr  7.7  mo>=+3%:  27/122  mo<=-5%:  31  kill: 122/122 starts, within 3mo 92%, within 6mo 99%, max survival 7 mo
Trend SMA200 band±3%                     stack x 0.95 (-0.5%/yr BTC-terms) maxDD-vs-HODL -72%  trades/yr  2.8  mo>=+3%:  26/122  mo<=-5%:  29  kill: 122/122 starts, within 3mo 91%, within 6mo 98%, max survival 7 mo
MeanRev z20 (BTC base, sell tops)        stack x 0.01 (-34.6%/yr BTC-terms) maxDD-vs-HODL -99%  trades/yr 13.5  mo>=+3%:  26/127  mo<=-5%:  37  kill: 127/127 starts, within 3mo 86%, within 6mo 100%, max survival 6 mo
MeanRev z20 (EUR base, buy dips)         stack x 0.00 (-43.0%/yr BTC-terms) maxDD-vs-HODL -100%  trades/yr 10.8  mo>=+3%:  50/127  mo<=-5%:  53  kill: 127/127 starts, within 3mo 82%, within 6mo 97%, max survival 8 mo
Static 80/20 (cash floor alone)          stack x 0.70 (-3.2%/yr BTC-terms) maxDD-vs-HODL -44%  trades/yr  1.6  mo>=+3%:  18/127  mo<=-5%:  11  kill: 126/127 starts, within 3mo 84%, within 6mo 98%, max survival 7 mo
Trend SMA100 ±3% + pullback entry (z<0)  stack x 1.12 (+1.1%/yr BTC-terms) maxDD-vs-HODL -64%  trades/yr  4.6  mo>=+3%:  30/125  mo<=-5%:  27  kill: 125/125 starts, within 3mo 88%, within 6mo 98%, max survival 7 mo
Trend SMA100 ±3% x vol-target 60%        stack x 0.71 (-3.3%/yr BTC-terms) maxDD-vs-HODL -73%  trades/yr 10.9  mo>=+3%:  30/125  mo<=-5%:  35  kill: 125/125 starts, within 3mo 88%, within 6mo 100%, max survival 6 mo

-- fee 0.6% per side --
Trend SMA50 band±0%                      stack x 0.30 (-10.8%/yr BTC-terms) maxDD-vs-HODL -82%  trades/yr 19.9  mo>=+3%:  29/127  mo<=-5%:  42  kill: 127/127 starts, within 3mo 92%, within 6mo 100%, max survival 5 mo
Trend SMA50 band±3%                      stack x 0.78 (-2.3%/yr BTC-terms) maxDD-vs-HODL -71%  trades/yr  7.6  mo>=+3%:  27/127  mo<=-5%:  36  kill: 126/127 starts, within 3mo 92%, within 6mo 99%, max survival 5 mo
Trend SMA100 band±0%                     stack x 0.88 (-1.2%/yr BTC-terms) maxDD-vs-HODL -65%  trades/yr 13.8  mo>=+3%:  28/125  mo<=-5%:  32  kill: 125/125 starts, within 3mo 92%, within 6mo 100%, max survival 6 mo
Trend SMA100 band±3%                     stack x 1.31 (+2.6%/yr BTC-terms) maxDD-vs-HODL -60%  trades/yr  4.6  mo>=+3%:  29/125  mo<=-5%:  28  kill: 125/125 starts, within 3mo 90%, within 6mo 100%, max survival 6 mo
Trend SMA200 band±0%                     stack x 0.45 (-7.5%/yr BTC-terms) maxDD-vs-HODL -84%  trades/yr  7.7  mo>=+3%:  27/122  mo<=-5%:  31  kill: 122/122 starts, within 3mo 92%, within 6mo 99%, max survival 7 mo
Trend SMA200 band±3%                     stack x 0.90 (-1.0%/yr BTC-terms) maxDD-vs-HODL -73%  trades/yr  2.8  mo>=+3%:  26/122  mo<=-5%:  29  kill: 122/122 starts, within 3mo 91%, within 6mo 98%, max survival 7 mo
MeanRev z20 (BTC base, sell tops)        stack x 0.01 (-36.4%/yr BTC-terms) maxDD-vs-HODL -99%  trades/yr 13.5  mo>=+3%:  24/127  mo<=-5%:  39  kill: 127/127 starts, within 3mo 87%, within 6mo 100%, max survival 6 mo
MeanRev z20 (EUR base, buy dips)         stack x 0.00 (-44.2%/yr BTC-terms) maxDD-vs-HODL -100%  trades/yr 10.8  mo>=+3%:  48/127  mo<=-5%:  53  kill: 127/127 starts, within 3mo 82%, within 6mo 97%, max survival 8 mo
Static 80/20 (cash floor alone)          stack x 0.70 (-3.3%/yr BTC-terms) maxDD-vs-HODL -44%  trades/yr  1.6  mo>=+3%:  18/127  mo<=-5%:  11  kill: 126/127 starts, within 3mo 84%, within 6mo 98%, max survival 7 mo
Trend SMA100 ±3% + pullback entry (z<0)  stack x 1.01 (+0.1%/yr BTC-terms) maxDD-vs-HODL -65%  trades/yr  4.6  mo>=+3%:  29/125  mo<=-5%:  27  kill: 125/125 starts, within 3mo 88%, within 6mo 98%, max survival 7 mo
Trend SMA100 ±3% x vol-target 60%        stack x 0.63 (-4.3%/yr BTC-terms) maxDD-vs-HODL -74%  trades/yr 10.9  mo>=+3%:  30/125  mo<=-5%:  35  kill: 125/125 starts, within 3mo 88%, within 6mo 100%, max survival 6 mo

== Coinbase liquidity, median daily volume last 90 days (2026-06-16..2026-09-13):
BTC-EUR    216.9 BTC/day (~EUR 14.6M)   BTC-USD   6212.4 BTC/day   ratio EUR/USD = 0.03
BTC-EUR weekday median 267.1  weekend median 91.6  (BTC/day)

== Daily close-to-close moves since 2016-01-01: sd 3.44%, days <= -8%: 68, days <= -5%: 217, days >= +8%: 87
median intraday high-low range: 3.71%; days with range >= 10%: 311

== -8% MTD (EUR terms) breaker on a 100% BTC book, 127 months:
months reaching -8% MTD: 57 (45%);  -12% MTD: 37 (29%);  months containing a single-day drop <= -8%: 42
after the -8% breach, month closed below breach level: 23;  at/above breach level: 34
SMA100 ±3% all-out (OFF = 0% BTC)        stack x 1.44 (+3.5%/yr BTC-terms) maxDD-vs-HODL -58%  trades/yr  4.6  mo>=+3%:  29/125  mo<=-5%:  28  kill: 125/125 starts, within 3mo 90%, within 6mo 100%, max survival 6 mo
   90-day windows (n=3719, fee 0.4%): rel. return p5 -18.0%  p25 -9.8%  p50 +0.0%  p75 +6.4%  p95 +45.9%  min -44.2%  max +100.7%
   share of windows whose DD-from-start breached: -10%: 36%  -15%: 18%  -20%: 5%  -25%: 2%  -30%: 1%
SMA100 ±3% all-out (OFF = 0% BTC)        stack x 1.31 (+2.6%/yr BTC-terms) maxDD-vs-HODL -60%  trades/yr  4.6  mo>=+3%:  29/125  mo<=-5%:  28  kill: 125/125 starts, within 3mo 90%, within 6mo 100%, max survival 6 mo
   90-day windows (n=3719, fee 0.6%): rel. return p5 -18.4%  p25 -10.1%  p50 -0.0%  p75 +6.2%  p95 +45.9%  min -44.3%  max +100.3%
   share of windows whose DD-from-start breached: -10%: 36%  -15%: 18%  -20%: 5%  -25%: 2%  -30%: 1%
SMA100 ±3% half-out (OFF = 50% BTC)      stack x 1.48 (+3.8%/yr BTC-terms) maxDD-vs-HODL -36%  trades/yr  5.3  mo>=+3%:  24/125  mo<=-5%:  13  kill: 125/125 starts, within 3mo 86%, within 6mo 99%, max survival 7 mo
   90-day windows (n=3719, fee 0.4%): rel. return p5 -9.3%  p25 -4.6%  p50 +0.0%  p75 +3.7%  p95 +23.4%  min -24.6%  max +47.5%
   share of windows whose DD-from-start breached: -10%: 6%  -15%: 1%  -20%: 0%  -25%: 0%  -30%: 0%
SMA100 ±3% half-out (OFF = 50% BTC)      stack x 1.41 (+3.3%/yr BTC-terms) maxDD-vs-HODL -37%  trades/yr  5.3  mo>=+3%:  24/125  mo<=-5%:  15  kill: 125/125 starts, within 3mo 86%, within 6mo 99%, max survival 7 mo
   90-day windows (n=3719, fee 0.6%): rel. return p5 -9.4%  p25 -4.7%  p50 +0.0%  p75 +3.7%  p95 +23.3%  min -24.7%  max +47.3%
   share of windows whose DD-from-start breached: -10%: 6%  -15%: 1%  -20%: 0%  -25%: 0%  -30%: 0%
```
