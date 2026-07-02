# Mizan — Project Context

**Document role:** This is orienting context, not a build spec. It holds the durable truth about
*why* this project exists and *how* it must be approached — the things that color every decision in
`MASTER_PRD.md` but are not themselves buildable instructions. When context here and spec in
`MASTER_PRD.md` both apply, the PRD governs *what* to build; this file governs *the posture* you
build it with. Read this first, then the PRD.

---

## 1. What this is, and what it is not

Mizan is the **foundation and learning substrate** for a long-horizon (years-out) goal: halal /
ethical-finance quantitative research. V1 is deliberately *not* the goal — it is the first concrete
brick under it.

- **This is a research/learning sandbox.** No real capital. No live execution. No order routing. No
  broker connection. The point is to build a correct data pipeline, an honest screening engine, and
  a legible backtest substrate — so that later, after a scholarly ruling on the harder questions,
  there is something solid and auditable to build on.
- **The deliverable is understanding, not returns.** Success is "I can see exactly why this ticker
  passed, exactly what the cash drag costs, exactly how much to purify, and exactly which of my
  assumptions are engineering defaults vs. placeholders for a scholar." Success is *not* a
  good-looking equity curve.
- **V1 is a deliberate, long-held plan** — not a pivot, not drift. It descends from a documented
  personal manifesto with an explicit long-horizon window and anti-drift disciplines. Treat it as
  intentional foundation-laying, and do not reframe it as a sudden change of direction.

## 2. The permissibility posture (the stance behind the invariants)

The enforceable, grep-able invariant list lives in `MASTER_PRD.md` §2. This section is the *why*.

- **The permissibility boundary is the reason the project exists, not a constraint on it.** It is
  never relaxed "for realism." If a feature seems to need a boundary-crossing mechanic (shorting,
  derivatives, margin/leverage, interest on cash, riba modeling), that is a signal to flag it and
  find the halal-permitted alternative — never to quietly implement it.
- **Honesty over realism, always.** A less-realistic-but-honest simulation beats a more-realistic
  one that hides a compliance compromise or an un-flagged bias. Every simplification (look-ahead,
  survivorship) is surfaced loudly on every output, not buried.
- **A hallucinated compliance verdict is the worst possible failure in this system.** This is why
  no compliance decision may ever route through an LLM. The screen is deterministic, end to end,
  and that is non-negotiable regardless of how convenient an LLM shortcut looks.

## 3. The scholarly-ruling discipline (why everything is "pending")

- Every embedded fiqh judgment call — denominator choice, business-activity tiers, ratio
  thresholds, grace-period length, purification method, divestment attribution — is an
  **engineering placeholder for a muamalat scholar's future ruling**, defaulted to the
  AAOIFI-faithful reading and stamped `pending scholarly verification`. See `MASTER_PRD.md` §10 for
  the register.
- **The standing rule:** a scholar with structural-feasibility (muamalat) expertise must rule on
  these before any of it informs real capital. This sandbox learns the mechanics; it does not rule
  on them, and it must never present itself as having ruled.
- **AAOIFI thresholds are as-implemented from public methodology**, verified against primary
  sources, never quoted from memory. The definitive text is AAOIFI Shari'ah Standard No. 21 itself,
  which is paywalled — so the implementation is a good-faith public-methodology reading awaiting
  ratification, and it says so wherever it matters.

## 4. Sourcing ethic

- **Clean / legal-only data.** SEC EDGAR (fundamentals) + Tiingo EOD (prices + dividends), both
  used within their stated terms. No `yfinance`, no ToS-gray scraping, no redistribution of
  personal-tier data. Auditability matters: raw pulls are cached so the system can *prove what the
  data said*, not just re-pull and hope it matches.
- **Fabricated citations are a known, recurring failure mode.** The mitigation is loading and
  reading primary sources before making a factual or numeric claim — never relying on memory or
  prior-session context for a threshold, an API limit, or a licensing term. If a source can't be
  confirmed, the claim is omitted, not guessed.

## 5. The shape of the system (why production reflexes are wrong here)

- This is a **stateful, single-user, data-heavy batch research tool** — not a service. The value is
  pipeline correctness and screening honesty, not concurrent request-serving. That is why V1 is a
  batch pipeline of jobs/CLI entrypoints with a local analytical store and static reports — no
  FastAPI service, no dashboard, no LLM in the core.
- **Resist the build-instinct time-sinks.** A pretty Next.js dashboard is exactly the kind of thing
  that eats a sprint and delays the actual learning. It is explicitly V2. If a "nice to have"
  starts pulling the build toward service/UI complexity, that is drift away from the point.

## 6. This is a personal build — the Kaide FDE apparatus does NOT apply

Kaide Labs' Forward Deployed Engineering operating system — the Anti-Replication Principle, the
5-Pillar Demo Standard, the Native-Environment "theater," founder-psychology teardowns, the
Google-native-LLM (Gemini/Vertex) routing mandate, the 48-72h client-demo sprint framing — is the
house style for **client teardowns and prospect demos**. **None of it governs this project.**

Mizan is a personal, long-horizon research foundation, not a client demo. Concretely, for this repo:
- **No anti-replication / adjacency framing.** There is no prospect's core IP to stay modular
  around. The only inviolable line is the permissibility boundary.
- **No 5-Pillar standard, no "Magic Moment," no native-environment UI theater.** Correctness and
  honesty are the standard, not instant visible ROI.
- **No Google-native LLM mandate. In fact, no LLM in the V1 core at all.** The default Kaide
  reflex to route everything through Gemini/Vertex is explicitly overridden here — a deterministic
  screen is a hard requirement, and any LLM use is a V2 news/NLP concern, out of scope for V1.

State this plainly so no synthesis or build step pattern-matches the Kaide house style and starts
designing a Vertex-routed sidecar with a Slack-native front end. That would be a correct instinct
for a client teardown and exactly the wrong one here.

## 7. Working posture for the build agent

- **Deterministic-first.** When a choice is between a deterministic rule and an LLM/heuristic for
  anything that must be *correct*, choose deterministic. The LLM is not in this loop.
- **Flag, don't smuggle.** Any fiqh judgment call, any bias simplification, any licensing caveat
  gets surfaced explicitly — in code and on output — never silently defaulted.
- **Commit discipline is absolute.** Neutral commits only. No AI attribution, no "Co-Authored-By,"
  no "Generated with." Author is the configured git user, full stop.
