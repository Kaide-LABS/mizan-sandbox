# Mizan — ULTIMATE_PRD (V1)

**Status:** locked, pre-build · Synthesized from `context.md` (posture) + `MASTER_PRD.md` (spec)
**Document role:** This is the single execution-ready artifact Phase 1 blueprinting expands from.
It does not restate every clause of `MASTER_PRD.md` — it locks the architecture, cites the external
grounding that validates it, and hands off a dependency-ordered build sequence. Where this document
is silent, `MASTER_PRD.md` governs.

---

## 1. The Thesis

Mizan is a **personal, long-horizon research foundation** for halal-constrained quantitative
finance — not a product, not a client demo, not a live-trading system (`context.md` §1, §6). The
problem it solves: there is no correct, honest, auditable substrate for learning how AAOIFI-style
Shariah screening, purification, and a compliance-flip state machine actually behave under real
market data, before any of it is allowed near real capital.

The hard constraints from `MASTER_PRD.md` force one architecture, not a menu of them:

- **A hallucinated compliance verdict is the worst possible failure** (`MASTER_PRD.md` §2, §13) →
  compliance logic must be **fully deterministic**, which rules out any LLM/agent-routing design by
  construction, not by preference.
- **Every fiqh judgment call is a placeholder for a scholar's ruling, never a silent hardcode**
  (`MASTER_PRD.md` §0, §10) → the config layer must physically separate *invariants* (no knob) from
  *parameters* (knobbed, sourced, flagged), so the split is legible on disk, not just in prose.
  This is the honesty surface (`context.md` §2–3).
  - **Every result carries its caveats structurally** (`MASTER_PRD.md` §9.2) → the result object,
  not the README, is where look-ahead/survivorship/pending-verification live. A reporter that can
  render an un-caveated number is a bug.
- **This is a single-user batch research tool, not a service** (`MASTER_PRD.md` §9, `context.md`
  §5) → jobs/CLI entrypoints over an embedded analytical store, static HTML+plots for output. No
  FastAPI, no dashboard, no concurrency-ops surface this tool doesn't need.

A deterministic batch pipeline is therefore not merely *an* option — given the failure mode that
must never happen (a hallucinated verdict) and the audience that must be served (one researcher
learning the mechanics), it is the only honest foundation. Anything more "production-shaped" would
be solving a problem this project doesn't have; anything less deterministic would risk the one
failure this project cannot tolerate.

---

## 2. The System Map (deterministic data flow — no agent routing)

```
INGEST (jobs)                 NORMALIZE              SCREEN (deterministic)        BACKTEST (event loop)          REPORT
──────────────                ───────────            ──────────────────────        ──────────────────────         ──────
SEC EDGAR                 ┐                          business_activity.py  ┐        clock.py (daily)                static
  companyfacts/           │                            (SIC first-pass +   │        portfolio.py                    HTML +
  companyconcept          │                             curated map,       │        (CASH_YIELD=0.0)                plots
  XBRL JSON               ├─ raw cache ─► DuckDB ──►    tiered)             ├─ Verdict ─ strategy.py ─ compliance_   (every
  (data.sec.gov)          │  (Parquet/     1.5.x        financial_ratios.py │   (3-     (long-only,   state.py       number
                          │   JSON,        (typed,        (30/30/5,         │  state)    pluggable)   (flip→grace→   stamped
Tiingo EOD                │   replayable)  as-of          denom from        │                          divest)       with
  adjClose + divCash      │                tables)        config)          │                              │         active_
  + splitFactor           ┘                              verdict.py ───────┘                        purification.py  caveats
  (api.tiingo.com)                                        (reason trace)                              (side-ledger,   [])
                                                                                                         divCash-driven)
```

**Pinned library versions** (verified current, 2026-07-02 — see §3 for how each was confirmed):

| Library | Pinned version | Role |
|---|---|---|
| Python | ≥3.11 | runtime (DuckDB 1.5.x requires ≥3.10; Pydantic 2.13.x supports up to 3.14) |
| `pydantic` | **2.13.4** | `config/` validation, `ConfigDict(extra="forbid")` on every model |
| `duckdb` | **1.5.4** (Python client) | embedded analytical store |
| `pyarrow` | **24.0.0** | Parquet raw cache + DuckDB↔Arrow interchange |
| `pandas` | latest 2.x compatible with pyarrow 24 | tabular glue for strategy/report layers |
| `matplotlib` | latest 3.x | static plots for the HTML report |
| `httpx` | latest 0.27+ | EDGAR/Tiingo REST calls (sync, explicit rate limiting) |
| `mypy` | latest, run `--strict` | type gate |

No FastAPI, no Gemini/Vertex SDK, no LLM client library anywhere in this table — their absence is
deliberate (`context.md` §6, `MASTER_PRD.md` §9).

### 2.1 Two load-bearing seams (repeat from `MASTER_PRD.md` §9.2, because Phase 1 must not erode them)
- `screen(ticker, as_of_date) -> Verdict` is the **only** place the Fork-1 look-ahead simplification
  may live. Any other function touching filing dates must hard-filter by `as_of_date`.
- A backtest result is `(returns, purification_ledger, active_caveats: list[Caveat])`. The reporter
  takes this tuple as its only render input — there is no code path that renders a number without
  its caveat set attached.

---

## 3. State-of-the-Art Justification (grounded, not guessed)

Every citation below was fetched and read via Nia (`docs.pydantic.dev`, `duckdb.org`,
`arxiv.org/abs/2512.22858`) on 2026-07-02, not recalled from memory.

### 3.1 Shariah screening math (denominator + threshold construction)
- **Primary methodological source (as identified in `MASTER_PRD.md` §5, §Sources):** AAOIFI
  Shari'ah Standard No. 21 is paywalled; the public-methodology reading used here is the
  Secretary-General presentation (Dr. Hamed Merah, OIC Exchanges) and the amanahadvisors 30% ratio
  explainer, both already cited with URLs in `MASTER_PRD.md` §Sources. This ULTIMATE_PRD does not
  re-derive those — it inherits them as the primary-source basis for the 30/30/5 thresholds and the
  market-cap/spot denominator default (`MASTER_PRD.md` §6.2, register #1–#5).
- **Corroborating 2025 academic source (re-verified by direct fetch, not by title-matching the
  seed):** arXiv 2512.22858, *"From Binary Screens to Continuous Compliance: A Shariah Screening
  Measure for Portfolio Design"* (Qadi, Sharma, Medda; posted 2025-12-28; q-fin.PM/q-fin.GN). Its
  actual contribution — a Continuous Shariah Compliance Index (CSCI) that embeds the
  business-activity and financial-ratio thresholds of **six** leading standards (including AAOIFI)
  on a single [0,1] scale, tested on CRSP/Compustat US equities 1999–2024 — directly corroborates
  two design choices in this architecture:
  1. **Multiple standards map to materially different regions of a common compliance scale**
     (the paper's Result 1) — which is exactly why `MASTER_PRD.md` §10 treats the denominator axis
     and the 30% threshold as *configurable parameters*, not settled constants: the paper's own
     empirical finding is that "pass/fail" is standard-dependent, not universal.
  2. **A spot, unsmoothed denominator produces materially different compliance calls than an
     averaged one** (the paper's treatment of the Sept-2023 DJIM/S&P methodology change admitting
     firms with lower CSCI under looser rules) — this is the mechanism `MASTER_PRD.md` §6.2 already
     names as "why a stock can flip compliant↔non-compliant on price moves alone," and it is the
     direct empirical justification for building the compliance-flip state machine (§7.3) as a
     first-class mechanic rather than an edge case.
  This paper is a discovery/measurement paper, not a AAOIFI-primary-text substitute — it is cited
  here as corroboration of the *volatility of binary screening outcomes*, which is precisely the
  phenomenon Mizan's flip/divest/purify state machine exists to model honestly.

### 3.2 Purification math
- `MASTER_PRD.md` §8 already states the operative rule directly from AAOIFI treatment: non-halal
  income must be donated to charity and excluded from investor P&L — a side-ledger by construction
  (Fork B). No additional academic sourcing is needed to establish this; it is standard, uncontested
  Islamic-finance practice, and the architecture's job is mechanical fidelity (side-ledger
  accounting, dividend-triggered computation from Tiingo's per-day `divCash` field) rather than
  novel derivation. Fork A (dividends-only, capital-gains OFF) and Fork C (time-bounded price
  appreciation for forced-divestment attribution) are explicitly flagged in `MASTER_PRD.md` §8 and
  §10 as **underdetermined by settled formula** — this document does not manufacture false
  settledness here; it keeps them configurable and `pending scholarly verification`, per the
  existing register.

### 3.3 Backtest correctness — look-ahead and survivorship bias
- `MASTER_PRD.md` §7.1–7.2 and §13 already name the canonical failure modes (look-ahead via
  most-recent-available-filing instead of point-in-time; survivorship via an "exists today"
  universe) and the correct fix (true point-in-time via EDGAR filing dates; a frozen historical
  universe snapshot) — this is standard backtesting methodology, not a contested research question,
  so the citation obligation here is architectural, not academic: **the V1 simplification must be
  quarantined to one function body** (`screen(ticker, as_of_date)`, §2.1 above) so that the V2 fix
  is a data-slicing change to that one function, not a rewrite. EDGAR's own filing-date metadata
  (confirmed present via the indexed `companyfacts`/`companyconcept` schema, `MASTER_PRD.md` §5.1)
  is what makes this V2 path a slicing change rather than a re-sourcing effort — verified true by
  inspecting the endpoint's documented response shape, not assumed.

### 3.4 API/library grounding (fetched, not recalled)
- **DuckDB Python client (v1.5.4):** confirmed via `duckdb.org/docs/stable/clients/python/reference`
  — `duckdb.connect(database=...)` for a persistent on-disk file vs. `:memory:`; `con.sql(query)` /
  `con.execute(query)` for querying; `con.read_parquet(path)` for ingesting the raw cache directly as
  a relation; `duckdb.to_arrow_table(batch_size=..., connection=...)` for Arrow round-tripping
  (pinned against `pyarrow` 24.0.0). These are real, current signatures, not inferred from training
  data.
- **Pydantic v2 (2.13.4):** confirmed via `docs.pydantic.dev` — the exact `config/` pattern is:
  ```python
  from pydantic import BaseModel, ConfigDict

  class Invariants(BaseModel):
      model_config = ConfigDict(extra="forbid")
      cash_yield: float = 0.0
  ```
  `extra="forbid"` raises `ValidationError` on any undeclared field — the mechanism that makes
  `config/invariants.py` and `config/fiqh_params.py` fail loudly on typos or drift, rather than
  silently ignoring an unrecognized key (the Pydantic v2 default is `"ignore"`, which this project
  must explicitly override everywhere).
- **SEC EDGAR / Tiingo access terms:** already primary-sourced in `MASTER_PRD.md` §5.1–5.2 with live
  URLs; re-fetching those pages during this synthesis returned only navigational/SPA shell content
  (no server-rendered rate-limit text reachable via a shallow crawl), so this document does not
  restate numbers it could not re-verify by direct fetch. It defers to `MASTER_PRD.md`'s existing
  cited figures (~10 req/s EDGAR cap, descriptive User-Agent requirement; Tiingo free-tier ~50
  symbols/hour, ~1,000 req/day, personal-use license) rather than re-asserting them without a fresh
  read. **Flag for Phase 1 ingestion-job authors:** re-confirm these two numeric limits by reading
  `sec.gov/os/accessing-edgar-data` and `tiingo.com/documentation/general/limits` directly (browser
  or authenticated fetch) before hardcoding backoff constants, since this synthesis could not
  re-verify them past the SPA shell.

---

## 4. Failure-Mode Analysis (expanded from `MASTER_PRD.md` §13)

Per layer. **HALT-for-human** = the pipeline stops and surfaces the exception; nothing auto-passes.
**Auto-retryable** = transient, bounded backoff, no human needed unless retries exhaust.

### 4.1 Ingestion (EDGAR)
| Failure | Detection | Prevention/Handling | Category |
|---|---|---|---|
| 429 / rate-limit block | HTTP 429 or sustained 5xx | Exponential backoff, ~0.12s min spacing between requests, respect `Retry-After` | Auto-retryable |
| Missing/incomplete `companyfacts` for a CIK | Empty or partial XBRL fact set | Mark ticker `unanalyzed`, never silently treat as pass | **HALT-for-human** |
| CIK resolution mismatch (ticker→CIK drift, e.g. post-merger) | `company_tickers.json` lookup returns wrong/stale CIK | Cross-check CIK against a second field (company name) before ingest; flag mismatch | **HALT-for-human** |
| Malformed/undocumented XBRL tag drift (SEC changes a concept name) | Schema validation on ingest (Pydantic model rejects unknown shape) | `extra="forbid"` on the normalization model surfaces this immediately as a `ValidationError`, not a silent null | **HALT-for-human** |

### 4.2 Ingestion (Tiingo)
| Failure | Detection | Prevention/Handling | Category |
|---|---|---|---|
| 429 / symbols-per-hour cap hit | HTTP 429 or documented cap counter | Backoff; since backfill is one-time (~100 tickers × ~10y at ~2h), no steady-state pressure | Auto-retryable |
| Adjusted-close / raw-close conflation | Column-contract check on ingest | Explicit typed columns (`adj_close`, `raw_close`) — never a single ambiguous `close` field | **HALT-for-human** (schema-level, caught at normalize) |
| Missing `divCash` on a known ex-dividend date | Cross-check against a second corporate-actions signal (split factor discontinuity without a matching divCash) | Mark the dividend gap explicitly; purification engine must never silently treat missing divCash as zero | **HALT-for-human** |

### 4.3 Normalization
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| Silent type coercion (e.g. XBRL fact reported in different unit scales across filings) | Unit-field cross-check on load | Normalize explicitly against the XBRL `units` field; reject rather than guess a scale | **HALT-for-human** |
| Duplicate/overlapping filing periods loaded twice into DuckDB | Primary-key constraint on (CIK, concept, period, filed date) | DuckDB schema enforces uniqueness; violation raises, not upserts silently | **HALT-for-human** |

### 4.4 Screening
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| **Wrong compliance verdict (WORST case, per `MASTER_PRD.md` §13)** | Unit tests against hand-verified reference cases; reason trace inspected per verdict | No LLM in `screening/` (grep-able invariant, see §5); deterministic ratio math only | **HALT-for-human**, always |
| Look-ahead beyond the flagged Fork-1 simplification | `as_of_date` filter test — assert no fact with `filed_date > as_of_date` reaches the screen | `screen()` hard-filters by date; any other access path to filings is a bug | **HALT-for-human** |
| SIC-code misclassification (compliant-coded conglomerate with a buried non-compliant division) | Hand-verified activity map cross-check against SIC first-pass, per ticker in the curated ~100-universe | SIC is explicitly first-pass only (`MASTER_PRD.md` §6.1) — the hand-verified map is the correctness backstop, not SIC alone | **HALT-for-human** on any SIC/map disagreement |
| Denominator/threshold config drift (a `fiqh_params.py` edit silently changes verdicts) | Diff the verdict set before/after any config change during development | Every parameter change is visible in `config/fiqh_params.py` diffs, never a runtime-computed default | **HALT-for-human** (review gate, not runtime) |

### 4.5 Storage (DuckDB)
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| Schema drift between raw cache and DuckDB typed tables | `mypy --strict` + Pydantic model validation at the normalize→DuckDB boundary | Typed load functions reject shape mismatches (`extra="forbid"`) | **HALT-for-human** |
| Concurrent-write corruption | N/A for this design — single-user, single-process batch jobs never run concurrently against the same DuckDB file | Enforce single-process job execution (no concurrent ingest+backtest against one file) | **HALT-for-human** if violated (should never occur by design) |

### 4.6 Backtest engine
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| `CASH_YIELD != 0.0` at runtime | Boot validator asserts the invariant before any simulation starts | Fail-fast boot check (`MASTER_PRD.md` §9.5) — boot refuses to proceed | **HALT-for-human**, immediate |
| A strategy attempts to hold/select a Non-compliant-verdict ticker | Strategy interface only receives the screened/compliant subset; assert on any out-of-universe selection | Strategy interface is constructed to make this structurally impossible, not just checked | **HALT-for-human** if the assertion ever fires (indicates an interface bug) |
| Compliance-flip missed at a rebalance boundary (state machine skips a quarter) | Assert every HELD(compliant) position was re-screened at every rebalance date | `compliance_state.py` re-screens every holding every rebalance, no opportunistic skipping | **HALT-for-human** |
| Grace-period misconfiguration silently changes divestment timing | Config-diff review, same as §4.4 threshold drift | Flagged parameter, not a hidden default (`MASTER_PRD.md` §10 register #8) | **HALT-for-human** (review gate) |

### 4.7 Purification
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| **Purification undercount** (missed dividend, or wrong period's ratio applied) | Reconcile every `divCash` event against the purification ledger; ratio must be pinned to the screen *active at the dividend date*, not the current screen | Explicit per-holding-per-period reconciliation job; the ratio lookup is date-indexed, not "most recent" | **HALT-for-human** |
| Fork-C attribution window computed against the wrong flip/divest dates | Assert `flip_date <= divest_date` and both dates come from `compliance_state.py`'s own log, not re-derived | Single source of truth for flip/divest timestamps (the state machine's own event log) | **HALT-for-human** |
| Purification silently netted into NAV (violates AAOIFI P&L-exclusion) | Reporter-level assertion: `net_of_purification_return` and `gross_return` must both be present and distinct in every report | Side-ledger accounting (Fork B) is structural, not a report-formatting choice | **HALT-for-human** |

### 4.8 Reporting
| Failure | Detection | Prevention | Category |
|---|---|---|---|
| A result renders without its caveat set | Reporter's render function requires `active_caveats: list[Caveat]` as a non-optional argument | Type-level enforcement — there is no render call signature that omits caveats | **HALT-for-human** (should be a type error, caught before runtime) |
| Survivorship/look-ahead caveats present in code but missing from the rendered output | Snapshot-test the static HTML against an expected caveat-string checklist | Test asserts caveat text appears verbatim in every generated report | **HALT-for-human** |

**Categorization summary** (per `MASTER_PRD.md` §13): every compliance/correctness exception above
is **HALT-for-human** — nothing in `screening/`, `compliance_state.py`, or `purification.py` may
auto-recover by guessing. Only network-transient failures (§4.1–4.2 429s) are auto-retryable.

---

## 5. The Hard Boundary — grep-able checks for the downstream review agent

The permissibility boundary (`MASTER_PRD.md` §2) is the only inviolable line in this project
(`context.md` §6 — there is no competitor IP to stay adjacent to). Code review for any PR touching
`mizan/` should grep for the following and treat any hit as a blocking finding, not a style note:

```bash
# Non-zero cash yield
grep -rnE "CASH_YIELD\s*=\s*[^0]" mizan/config/invariants.py

# Boundary-crossing position primitives anywhere in backtest/ or screening/
grep -rniE "\b(short|borrow|margin|leverage|option|future|swap)\b" mizan/backtest/ mizan/screening/

# Negative position quantities (a short position in disguise)
grep -rnE "quantity\s*[:=]\s*-" mizan/backtest/portfolio.py

# Any interest-accrual mechanic on cash
grep -rniE "interest|apr|yield_rate" mizan/backtest/portfolio.py

# Any LLM/model client inside the deterministic core
grep -rniE "openai|anthropic|gemini|vertex|langchain|llm|chat_completion" mizan/screening/ mizan/backtest/compliance_state.py
```

A clean grep on all five is a necessary (not sufficient) precondition for merge. The reason trace
requirement (§6.3 of `MASTER_PRD.md`) and the unit-test-against-hand-verified-cases requirement
(§4.4 above) are the sufficient conditions.

---

## 6. Execution Spec — Phase 1, dependency order

Each stage is buildable and testable before the next starts; nothing later reaches back to change
an earlier stage's public shape except adding fields.

1. **`config/invariants.py`** — `CASH_YIELD: float = 0.0`, long-only flag, no-derivatives flag,
   no-margin flag, no-LLM-in-compliance flag. `ConfigDict(extra="forbid")`. A boot validator module
   (`config/boot_check.py` or equivalent) imports this and asserts every invariant before any other
   module runs — fail-fast, per `MASTER_PRD.md` §9.5.
2. **`config/fiqh_params.py`** — every row of `MASTER_PRD.md` §10's register as a typed, defaulted,
   sourced field, each carrying a `pending scholarly verification` stamp (a literal field or
   docstring, inspectable at runtime by the reporter). `extra="forbid"` here too.
3. **`data/ingest_edgar.py`, `data/ingest_tiingo.py`** — raw-cache-first (Parquet/JSON to disk),
   replayable, rate-limited per §3.4/§4.1–4.2, never normalize inline with the fetch.
4. **`data/normalize.py`** — raw cache → typed DuckDB tables via `duckdb.connect(database=<path>)`
   + `con.read_parquet(...)` / `con.sql(...)`, Pydantic-validated row shapes at the load boundary.
5. **`screening/business_activity.py`, `screening/financial_ratios.py`, `screening/verdict.py`** —
   deterministic, no LLM (grep-checked per §5), reason trace on every verdict, unit tests against
   hand-verified reference tickers before anything downstream consumes verdicts.
6. **`backtest/clock.py`, `backtest/portfolio.py`, `backtest/strategy.py`,
   `backtest/compliance_state.py`** — daily event loop; portfolio reads `CASH_YIELD` from
   invariants (never hardcodes it a second time); strategy interface structurally cannot select a
   non-compliant ticker; compliance-flip state machine per §7.3 of `MASTER_PRD.md`.
7. **`backtest/purification.py`** — side-ledger, `divCash`-driven, Fork A/B/C parameters read from
   `fiqh_params.py`, reconciliation test per §4.7 above.
8. **`reporting/report.py`** — static HTML + `matplotlib` plots; render function's signature
   requires `active_caveats` as a non-optional argument; snapshot test asserts caveat text appears
   in output per §4.8.

Definition of done for Phase 1 = `MASTER_PRD.md` §11 items 1–5, achieved in the order above with no
step skipped and no invariant/parameter split blurred.

---

## Sources consulted for this synthesis

- AAOIFI SS21 methodology (Secretary-General presentation): https://www.oicexchanges.org/files/1---shari-ah-screening-in-the-islamic-capital-markets-dr-hamed-merah-secretary-general-aaoifi.pdf (inherited from `MASTER_PRD.md`, not re-fetched)
- AAOIFI 30/30/5 corroboration: https://amanahadvisors.com/making-sense-of-the-30-rule-in-islamic-finance/ (inherited)
- Qadi, Sharma, Medda, "From Binary Screens to Continuous Compliance: A Shariah Screening Measure for Portfolio Design," arXiv:2512.22858 (posted 2025-12-28) — fetched and read directly via Nia on 2026-07-02: https://arxiv.org/abs/2512.22858
- DuckDB Python client reference (v1.5.4, confirmed current via PyPI + docs on 2026-07-02): https://duckdb.org/docs/stable/clients/python/reference · https://pypi.org/project/duckdb/
- Pydantic v2 `ConfigDict(extra="forbid")` (v2.13.4, confirmed current on 2026-07-02): https://docs.pydantic.dev/latest/ · https://pydantic.dev/docs/validation/latest/api/pydantic/config/
- Apache PyArrow (v24.0.0, confirmed current on 2026-07-02): https://arrow.apache.org/docs/python/install.html
- SEC EDGAR APIs (endpoint/access policy, inherited from `MASTER_PRD.md`; re-fetch attempt on 2026-07-02 returned SPA shell only, not re-verified numerically): https://www.sec.gov/search-filings/edgar-application-programming-interfaces
- Tiingo EOD documentation (inherited from `MASTER_PRD.md`; same re-fetch caveat): https://www.tiingo.com/documentation/end-of-day
