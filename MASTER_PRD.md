# Mizan — Halal-Constrained Market Simulation & Research Sandbox

**Master PRD (V1)** · Working name: *Mizan* (rename freely) · Status: scope locked, pre-build
**Document role:** This is the consolidated context + master spec. It is the single source of
truth the Step 7 synthesis run ingests to produce `ULTIMATE_PRD.md`. There are no lateral PRD
variants for this build — this document folds context and master into one.

---

## 0. Reading Guide (the one thing to internalize first)

Every design decision in this project sorts into exactly one of two bins, and confusing them is
the single worst failure mode:

- **INVARIANTS** — the permissibility boundary. Hardcoded, no config knob, never tunable "for
  realism." These are not preferences; they are the reason the project exists.
- **PARAMETERS** — *fiqh judgment calls* where scholarly bodies legitimately differ. Every one is
  a visible, configurable value, defaulted to the AAOIFI-faithful reading, and stamped
  `pending scholarly verification`. These are engineering placeholders for a muamalat scholar's
  future ruling — never silent hardcoded positions dressed up as settled law.

The codebase must make this split legible: a reader opens one config surface and sees every
judgment call the system embeds, each labeled as boundary or placeholder. This is the project's
"honesty surface." See §10 for the full register.

---

## 1. Purpose & Problem Statement

Build the **foundation and learning substrate** for long-horizon, halal/ethical-finance
quantitative research: a simulation environment that ingests real, public, stale (1–2 week delay
acceptable) US equity data and lets a single researcher screen a Shariah-permitted universe and
backtest long-only strategies against it.

This is **not** a live-trading system. No real capital, no live execution, no order routing. The
deliverable is a correct data pipeline, an honest screening engine, and a legible backtest
substrate — so that later, after a scholar rules on the harder questions, there is a solid,
auditable foundation to build on.

The value is **pipeline correctness and screening honesty**, not throughput, not UI polish, not
concurrency. Architecture decisions are judged against that, not against production-service
instincts.

---

## 2. The Hard Permissibility Boundary (the inviolable line)

Treated as a compile-time invariant. Anything that appears to require a boundary-crossing feature
is flagged and routed to a halal-permitted alternative — never quietly implemented for realism.

### INVARIANTS — hardcoded, no knob
- **Long-only.** Action space is limited to instruments actually owned. No selling what is not held.
- **Fully-funded.** No borrowing, no margin, no leverage of any kind.
- **No interest earned on cash.** `CASH_YIELD = 0.0`. Idle cash earns exactly zero — no T-bill,
  money-market, or risk-free sweep. Idle-cash yield is riba. The resulting cash drag is *correct*
  and must be visible in results.
- **No derivatives.** No options, futures, swaps, or any derivative contract.
- **No riba modeling.** The interest-based portion of the market is not modeled.
- **Compliance verdicts are deterministic.** No compliance decision may route through an LLM. A
  hallucinated compliance verdict is the worst possible failure in this system.

### DEFERRED — pending a muamalat (structural-feasibility) ruling; out of scope for V1
Short-selling, derivatives/options/futures/swaps, margin/leverage, any interest-bearing mechanic,
and modeling of the riba-based market. These are not "V2 features" in the ordinary sense — they do
not enter the roadmap until a scholar rules on structural feasibility.

---

## 3. Scope — V1

### IN SCOPE (build freely)
- Long-only positions in screened equities (buy / hold / sell-owned / hold-cash).
- AAOIFI-style Shariah screening: business-activity screen + financial-ratio screens, implemented
  from **public** AAOIFI methodology (no paid screening API).
- Real EOD price/volume + fundamentals + dividends. Total-return series (dividends included).
- Backtesting long-only strategies over the screened universe.
- Purification tracking (computing the non-compliant income portion to purify).

### OUT OF SCOPE — V1 (see §2 DEFERRED, plus engineering deferrals in §12)
Short-selling, derivatives, margin/leverage, interest mechanics, riba modeling; and (engineering,
not fiqh) true point-in-time screening, survivorship-corrected universe, full S&P 500, news/
sentiment, any LLM component, non-US markets, any dashboard/frontend, capital-gains purification.

---

## 4. Asset Universe

- **Market:** US only for V1.
- **Size:** curated ~100-ticker subset (scales to full S&P 500 in V2).
- **No universe-level sector pre-pruning.** Alcohol, tobacco, conventional finance, etc. stay in
  the *raw* universe deliberately, so the business-activity screen has real cases to reject.
  Pre-pruning would hide whether the screen actually works.
- **Survivorship note:** the universe is "companies that exist today," which bakes in survivorship
  bias (see §7). Accepted for V1, flagged loudly, fixed in V2.

---

## 5. Data Sources (verified against primary sources)

Two clean, free, license-appropriate legs, split by data type. No `yfinance`, no ToS-gray
redistribution.

### 5.1 Fundamentals (for screening ratios) → SEC EDGAR
- **Endpoint:** XBRL `companyfacts` / `companyconcept` on `data.sec.gov`. Returns every
  structured XBRL fact a company has filed (revenue, total assets, shares outstanding, debt, etc.)
  as JSON per CIK.
- **Access:** free, **no API key**. Requires (a) a descriptive `User-Agent` header naming the app
  + a contact email, and (b) staying under **10 requests/second per IP**. Exceeding → 429 +
  temporary block. Use ~0.12s spacing.
- **Bonus fields:** SIC codes (a free first-pass business-activity signal) and filing dates (which
  make true point-in-time screening a V2 data-slicing change rather than a re-sourcing effort).
- **CIK resolution:** `company_tickers.json` maps ticker → CIK (zero-pad to 10 digits).

### 5.2 Prices + Dividends (for backtest + purification) → Tiingo EOD, free tier
- **Why Tiingo:** its adjusted prices follow the CRSP methodology, which incorporates **both split
  and dividend adjustments** → clean total-return series. The daily response also carries a
  **per-day dividend cash field** (on the ex-date) plus a split/distribution adjustment factor.
  One clean source gives both the total-return series *and* the raw per-event dividend cash the
  purification engine needs.
- **Free-tier limits (approximate; Tiingo states limits can change):** internal/personal use only,
  ~50 unique symbols/hour, ~1,000 requests/day, ~1 GB/month bandwidth. Fundamentals and news are
  **not** on the free tier — irrelevant here, since fundamentals come from EDGAR.
- **Backfill math:** ~100 tickers × ~10y daily bars is a **one-time** historical pull (~2 hours at
  the symbols/hour cap), cached to Parquet and never re-fetched. The cap never bites steady-state
  because a stale sandbox has no steady state. Ingest jobs handle 429s with backoff regardless.
- **License caveat (carried):** free tier is personal-use, no redistribution. Fine for a private
  sandbox; if this ever becomes shared/commercial the tier changes. Flagged so it can't ambush.

### 5.3 Replayable raw cache (both legs)
Raw EDGAR JSON and raw Tiingo pulls are cached to disk (Parquet/JSON) before normalization, so
ingestion is replayable and auditable — the "I can prove what the data said" property. Normalized,
typed tables live in DuckDB (§9).

---

## 6. Screening Engine (deterministic)

Anchored to **AAOIFI Shari'ah Standard No. 21 ("Financial Papers — Shares and Bonds")**,
implemented from public methodology, verified against primary sources (see §Sources). Every
threshold flagged `as-implemented from public AAOIFI methodology, pending scholarly verification`.
No LLM anywhere in this engine.

### 6.1 Business-activity screen (Layer 1 — a fail here cannot be purified)

Tiered, because AAOIFI's *own explicit* enumerated list is narrower than the list informally
attributed to it. The items AAOIFI does not enumerate fall under its catch-all "any activities
deemed unlawful," which is a scholar's interpretation — so they are flagged, not hardcoded as
settled.

| Tier | Default | Contents | Basis |
|------|---------|----------|-------|
| **Tier 0** | ON (firm) | conventional financial institutions, conventional insurance, alcohol, gambling, **pork and non-halal food**, tobacco | AAOIFI-explicit (SS21 industry screen) |
| **Tier 0.5** | ON (flagged as interpretation) | adult content/pornography, weapons/defence | AAOIFI catch-all "any unlawful activity"; also in index-provider lists — *not* SS21-enumerated |
| **Tier 1** | OFF | media/music/entertainment, hotels | Genuinely contested across scholarly bodies |

Notes:
- **"Non-halal food," not just "pork."** AAOIFI names non-halal food generally — the screen
  rejects non-halal meat processing, not only swine. (Correction to earlier "pork"-only framing.)
- **Weapons is an interpretation, not SS21-explicit.** Kept default-on, but demoted out of the
  hardcoded core into the flagged tier; a scholar rules on scope (all defence vs. offensive-only).
- **SIC is first-pass only.** SIC codes seed the classifier but are coarse (they miss a compliant-
  coded conglomerate with a buried alcohol/financing division). For a 100-ticker universe the
  honest implementation is **SIC first-pass + a hand-verified activity map**, not pretend-
  automation. Deeper automation (parsing 10-K business-description text) is a V2 NLP item and must
  not drag an LLM into this deterministic screen.

### 6.2 Financial-ratio screen (Layer 2 — only runs if Layer 1 passes)

AAOIFI **30 / 30 / 5**:

1. **Interest-bearing debt ÷ market cap < 30%.** (Tighter than the 33% used by DJIM / S&P Shariah
   / FTSE Yasaar.)
2. **Interest-bearing investments/deposits ÷ market cap < 30%.**
3. **Non-permissible income ÷ total revenue < 5%.**

**Denominator is two-axis (both configurable, both flagged):**
- Axis A: **market-cap** (AAOIFI) vs total-assets (MSCI-style).
- Axis B: **spot** vs averaged. AAOIFI specifies *no averaging window* (SS21 refers only to "the
  last budget or verified financial position"), so spot market cap is the faithful reading; index
  providers bolt on 12/24/36-month smoothing.
- **Default: market-cap + spot** (AAOIFI-faithful). Toggle to total-assets and/or averaged to
  observe the stability difference firsthand. A spot-market-cap denominator is *why* a stock can
  flip compliant↔non-compliant on price moves alone — the exact event §7's state machine catches.

### 6.3 Verdict taxonomy (three-state, not boolean)

AAOIFI's Pure / Mixed / Non-compliant maps directly onto the output. `Verdict` is a **three-state
enum**:
- **Pure** — compliant business, no interest-based deposits/loans. No purification needed.
- **Mixed** — compliant business and ratios within thresholds, but some incidental non-halal
  income. **Purification applies.**
- **Non-compliant** — fails an activity or ratio screen. Not investable; if held, triggers the
  flip → forced-divestment path.

Every verdict carries a **reason trace** (which screen, which ratio, what value vs. threshold) so
any pass/fail is inspectable.

---

## 7. Backtest Engine (small, explicit event loop — built from scratch)

Not vectorbt/backtrader/zipline. The halal mechanics are a custom state machine no off-the-shelf
engine models, and an explicit daily loop is legible for learning. From-scratch engines breed
look-ahead and survivorship bugs, so both are treated as first-class concerns.

- **Granularity:** daily EOD bars.
- **Action space:** `{buy, hold, sell-owned, hold-cash}`. Fully-funded. No leverage.
- **Cash:** `CASH_YIELD = 0.0` invariant (§2). Idle cash is a real drag, shown.
- **Rebalance cadence:** quarterly (screening data only updates with filings anyway).
- **Strategy interface:** pluggable, long-only. A "strategy" selects among screened, compliant
  tickers and allocates fully-funded capital; it can never short, lever, or hold a non-compliant
  name by choice.

### 7.1 Look-ahead (Fork 1) — documented V1 simplification
Screening is called **as-of a date**: `screen(ticker, as_of_date) -> Verdict`. V1 uses
most-recent-available filing rather than true filing-lag-corrected point-in-time. This introduces
**look-ahead bias** and is flagged **loudly, in code and on every result output** as a known V1
limitation — never to be mistaken for an honest backtest. The simplification lives **entirely
inside that one function body**; V2 true-PIT changes only that body (EDGAR filing dates make it a
data-slicing change).

### 7.2 Survivorship (Fork trap) — accept + flag for V1
The "exists today" universe silently excludes anything delisted or gone to zero. Accepted for V1,
flagged with the same loudness as look-ahead. V2 freezes a historical as-of universe snapshot.

### 7.3 Compliance-flip state machine (the distinctive mechanic — V1, non-negotiable)
This is the single most distinctive halal-specific mechanic and the reason the project is more than
a generic screened-equity backtester. State machine per holding:

```
HELD(compliant)
   │  quarterly rebalance re-screens holding as-of date
   ▼
flip detected? ── no ──► stay HELD(compliant)
   │ yes
   ▼
HELD(non-compliant, grace)  ── forced-divestment scheduled at next rebalance ──►
   ▼
DIVEST  ──► purify gains attributable to the non-compliant holding window (Fork C)
```

- **Grace-period length is a configurable, flagged parameter** (default: divest at next rebalance
  after flip), per common Islamic-fund practice, pending scholarly verification.
- **Degradation rule:** if dividend sourcing ever becomes the long pole, the state machine still
  ships on price-return — flip-detection + forced-divestment run regardless; only *purification*
  fidelity degrades to a documented stub. State machine is always in V1; only purification flexes.
  (Per §5.2, dividends are sourced, so this degradation is not expected to trigger.)

---

## 8. Purification Engine (side-ledger)

AAOIFI treatment confirms the design: non-halal income must be **donated to charity and cannot be
included in the investor's P&L**. That is a side-ledger by definition — purification is carried
*outside* the return, never netted into NAV.

- **Trigger:** purification is a **dividend mechanic**. For each dividend received from a **Mixed**
  holding, purify `non_compliant_income_ratio × dividend_amount`, where the ratio comes from that
  holding's most-recent screen and the dividend cash comes from Tiingo's per-day divCash field.
- **Fork A — what gets purified:** **dividends only** for V1. Capital-gains purification is a
  configurable, **default-OFF**, `pending scholarly ruling` flag.
- **Fork B — accounting:** **side-ledger.** Accrue a running `purification_liability` separately;
  report **two numbers** — gross return and net-of-purification return — so the drag is its own
  visible figure, not a silent NAV distortion. (Confirmed faithful to AAOIFI's P&L-exclusion rule.)
- **Fork C — forced-divestment attribution:** for a holding that flipped and was divested, the
  amount to purify from the non-compliant window is **underdetermined by settled formula**. V1
  default: **price appreciation during the non-compliant window, time-bounded flip-date →
  divestment-date**, as a visible, configurable, `pending scholarly ruling` parameter. No pretense
  of a settled formula.

---

## 9. Architecture — batch pipeline, stateful single-user

Not a service constellation. Ingestion and backtest are **jobs / CLI entrypoints**, not endpoints.
No FastAPI service in the V1 core. **No LLM in the V1 core at all** — deterministic data → ratios →
verdicts → backtest, end to end. Interface is **notebook/CLI-first, results as generated static
HTML + plots**. No Next.js dashboard in V1.

### 9.1 Pipeline

```
INGEST (jobs)          NORMALIZE          SCREEN (deterministic)      BACKTEST (event loop)         REPORT
────────────           ─────────          ──────────────────────      ─────────────────────         ──────
EDGAR companyfacts ─┐                     business_activity ─┐        daily clock                    static HTML
                    ├─ raw cache ─ DuckDB ─ financial_ratios ─┼ verdict ─ strategy ─ portfolio ─────  + plots
Tiingo EOD (adj)   ─┤   (parquet/  (typed,  (denom+thresholds │         │          │ CASH_YIELD=0     (caveats
Tiingo divCash     ─┘    json,     as-of     from config)     │         │          │                  stamped on
                         replayable)         )                │   compliance_state ─┤                  every output)
                                                              │   (flip → divest)   │
                                                              └─ purification ◄──────┘
                                                                 (side-ledger)
```

### 9.2 Two load-bearing seams
- **Caveats are a property of the result object, not a footnote.** A backtest result is
  `(returns, purification_ledger, active_caveats[])`. The reporter **refuses to render** a number
  without stamping its caveat set (look-ahead, survivorship, pending-verification). This makes the
  "every output caveated" requirement structural — you cannot emit an un-caveated result.
- **Screening is called as-of a date.** `screen(ticker, as_of_date) -> Verdict`; the V1 look-ahead
  simplification is quarantined inside that one body (§7.1).

### 9.3 Storage
- **DuckDB** as the primary analytical store (embedded, zero-ops, columnar — matches the backtest's
  column-scan workload; ~250k price rows + a few thousand fundamentals rows is trivial). No
  Postgres/Cloud SQL — that is concurrency ops this single-user tool does not need.
- **Raw file cache** (Parquet/JSON) upstream of DuckDB, for replayable/auditable ingestion.

### 9.4 Module boundaries

```
mizan/
  config/
    invariants.py        # permissibility boundary — constants, NO knobs (CASH_YIELD=0.0, etc.)
    fiqh_params.py       # every judgment call; each defaulted + source + pending-verification flag
  data/
    ingest_edgar.py      # companyfacts XBRL → raw cache
    ingest_tiingo.py     # EOD adj-close + divCash → raw cache
    normalize.py         # raw → typed DuckDB tables
  screening/
    business_activity.py # SIC first-pass + curated activity map (tiered)
    financial_ratios.py  # 30/30/5; denominator (two-axis) from config
    verdict.py           # combine → three-state Verdict + reason trace
  backtest/
    clock.py             # daily event loop
    portfolio.py         # holdings + cash (reads CASH_YIELD invariant)
    strategy.py          # pluggable long-only interface
    compliance_state.py  # flip-detect → forced-divest state machine
    purification.py      # side-ledger, divCash-driven (Fork A/B/C params)
  reporting/
    report.py            # static HTML + plots; refuses to render without caveat set
  cli/                   # ingest | screen | backtest | report entrypoints
```

### 9.5 Engineering invariants
Pydantic v2 with `ConfigDict(extra="forbid")`; `mypy --strict`; fail-fast boot validators that
assert the permissibility invariants at startup (e.g. boot fails if `CASH_YIELD != 0.0`). Commits
stay neutral — no AI attribution, configured git user only.

---

## 10. The Fiqh Judgment-Call Register (the honesty surface)

Every entry is a configurable parameter in `config/fiqh_params.py`, defaulted to the AAOIFI-
faithful reading, and stamped `pending scholarly verification`. Nothing here is a silent hardcode.

| # | Judgment call | V1 default | Configurable? | Note |
|---|---------------|-----------|---------------|------|
| 1 | Ratio denominator — asset base | market cap | yes | market-cap (AAOIFI) vs total-assets (MSCI) |
| 2 | Ratio denominator — smoothing | spot | yes | AAOIFI specifies no averaging window |
| 3 | Debt threshold | 30% | yes | AAOIFI SS21 |
| 4 | Interest-investments threshold | 30% | yes | AAOIFI SS21 |
| 5 | Non-permissible income threshold | 5% | yes | AAOIFI SS21 |
| 6 | Business-activity Tier 0.5 (adult, weapons) | ON | yes | catch-all interpretation, not SS21-explicit |
| 7 | Business-activity Tier 1 (media/music, hotels) | OFF | yes | genuinely contested |
| 8 | Grace period (flip → divest) | next rebalance | yes | common fund practice |
| 9 | Purify capital gains (Fork A) | OFF | yes | dividends-only default |
| 10 | Divestment attribution (Fork C) | time-bounded price appreciation | yes | no settled formula |
| 11 | Purification accounting (Fork B) | side-ledger | yes | faithful to AAOIFI P&L-exclusion |

**Standing rule:** a muamalat scholar with structural-feasibility expertise rules on these before
any of it informs real capital. This sandbox learns the mechanics; it does not rule on them.

---

## 11. V1 Definition of Done

1. Ingestion pulls and caches EDGAR fundamentals + EOD **total-return** price data (incl.
   dividends) for the ~100-ticker universe, **replayably**.
2. The screening engine computes business-activity + configurable financial-ratio screens from
   public AAOIFI methodology, emits **three-state** per-ticker verdicts with **every fiqh judgment
   call exposed as a flagged parameter** and a reason trace, and any pass/fail is inspectable.
3. The backtest engine runs a long-only quarterly-rebalance strategy over the screened universe
   with the **compliance-flip → forced-divestment → purification** state machine working end to
   end.
4. **Every honesty caveat** (look-ahead, survivorship, pending-scholarly-verification) is stamped
   on **every result output**, structurally (via the result object), not buried in a README.
5. Results come out as a **generated static report** (HTML + plots).

---

## 12. Deferred to V2

True point-in-time screening; survivorship-corrected (as-of snapshot) universe; full S&P 500;
news/sentiment ingestion; any LLM component (re-enters only if V2 news/NLP needs it, and never for
compliance verdicts); non-US markets; the Next.js dashboard; capital-gains purification; deeper
10-K-text business-activity classification.

---

## 13. Failure-Mode Analysis (seed — silent-correctness focus)

The dangerous failures here are **silent-correctness** failures, not crashes. Non-exhaustive seed
for the Step 7 expansion:

- **Wrong compliance verdict (WORST).** Cause: LLM in the path, or a ratio bug, or stale
  fundamentals. Prevention: no LLM in screening (invariant); deterministic ratio engine with unit
  tests against hand-verified cases; reason trace on every verdict. **HALT-for-human** on any
  screening exception; never auto-pass.
- **Look-ahead leakage** beyond the documented Fork-1 simplification (e.g. using a filing dated
  after `as_of_date`). Prevention: `screen()` hard-filters by date; the only permitted look-ahead
  is the flagged Fork-1 one.
- **Adjusted-close misuse.** Using adjusted close for share-count/market-cap math, or raw close for
  total return. Prevention: explicit column contracts; raw vs adjusted never interchanged.
- **Purification undercount.** Missing a dividend, or applying the wrong period's ratio.
  Prevention: divCash reconciled per holding per period; ratio pinned to the screen active at the
  dividend date.
- **Survivorship** (see §7.2) — flagged, not prevented, in V1.
- **Silent ingest gaps.** A ticker with missing filings screened as if complete. Prevention: mark
  "unanalyzed" rather than guess; never let a data gap read as a pass.

Categorization for the build loop: correctness/compliance exceptions **HALT-for-human**; transient
network/429s are **auto-retryable** with backoff.

---

## Sources (primary-source URLs preserved)

- SEC EDGAR APIs (endpoints, XBRL, access policy): https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- SEC EDGAR fair-access / rate limit / User-Agent: https://www.sec.gov/about/developer-resources · https://www.sec.gov/search-filings/edgar-search-assistance/accessing-edgar-data
- Tiingo EOD adjusted-price (CRSP) + dividend field: https://www.tiingo.com/documentation/end-of-day
- Tiingo free-tier limits / personal-use license: https://www.tiingo.com/ (pricing) · limits corroborated at product pages
- AAOIFI SS21 methodology (Secretary-General presentation): https://www.oicexchanges.org/files/1---shari-ah-screening-in-the-islamic-capital-markets-dr-hamed-merah-secretary-general-aaoifi.pdf
- AAOIFI 30/30/5 + denominator corroboration: https://amanahadvisors.com/making-sense-of-the-30-rule-in-islamic-finance/ · https://arxiv.org/html/2512.22858v1 (survey of standards & denominators; seed for the academic sweep)

*All AAOIFI thresholds are as-implemented from public methodology, pending scholarly verification.
The definitive text is AAOIFI Shari'ah Standard No. 21 itself.*
