# Bliss — a trading research system engineered so it cannot lie to its operator

**Built by one person and Claude, in public view of its own failures.**

Bliss is an options-intelligence platform and autonomous paper-trading agent:
it reads market structure, grades trade setups Prime / Valid / Watch / Avoid,
executes the best of them against a broker (Interactive Brokers, paper), and
keeps an append-only journal of everything that happens. The working system —
grading engine, backtest harness, multi-horizon scanner, execution daemon,
API, and website — lives in a private repository.

This public repo documents the part that matters to anyone outside the
strategy: **the honesty architecture.** Bliss is designed around one governing
question — *how do you build a self-evaluating system that cannot flatter
itself?* — which turns out to be an alignment question wearing a finance
costume.

---

## The problem: evaluation pipelines want to lie

Any system that grades its own ideas and reports its own performance sits on
a gradient toward self-flattery. In trading research the canonical failures
are lookahead (the model peeks at the future), survivorship and selection
bias, overfit backtests, and — subtlest of all — **evaluation assumptions
that quietly favor the system**. Nobody writes `flatter_me=True`; it arrives
disguised as a reasonable default.

Bliss treats these as structural problems, not discipline problems. The rules
are enforced by code and tests, never by intention.

## The honesty architecture

- **The cardinal no-lookahead rule.** At bar *i*, every read is computed from
  bars `[:i+1]` only. This is pinned by tests: appending future bars to a
  series must not change any earlier read.
- **Point-in-time replay as the single ruler.** Every idea is graded by the
  same harness: walk history, grade each moment blind, follow each setup
  forward to a labeled outcome, calibrate per grade (win rates with Wilson
  intervals, Brier score, expected calibration error), and test the ladder's
  monotonicity (Prime must actually beat Avoid).
- **Grade-before-hardcode.** Every candidate signal ships default-OFF and
  byte-identical to baseline when off. It earns influence only by winning an
  A/B on the ruler, across symbols and regimes. Rejections are recorded and
  kept. The system's own liquidity gate was graded NO-GO twice and remains
  off — the process working as designed.
- **No gate may rescue an Avoid.** Confluence can promote good setups; nothing
  may promote one the base rules called untradable.
- **Real data or no data.** `require_real_data` gates the record: synthetic
  or fallback data can exercise the plumbing but can never enter the journal,
  the hit rate, or any reported number.
- **Cancels are not trades.** An entry that never filled is excluded from the
  hit rate — crediting phantom fills once produced a fake 100% win rate in
  early development, and the fix became a permanent rule.
- **Overfit surveillance.** Probabilistic and Deflated Sharpe Ratio and
  Probability of Backtest Overfitting are first-class outputs, so "we found
  an edge" always travels with "and here is the probability we fooled
  ourselves."

## Case study: the phantom-fill subsidy

For months, the harness labeled setups under a standard, innocent-looking
assumption: *the planned entry always fills.* In July 2026 we replaced it
with an honest fill model that mirrors live execution exactly — an entry is
pending until price actually trades through it; a target struck while the
order is still pending cancels the trade (it never existed); a bar that fills
and stops you in the same breath is a loss.

Measured on real broker data, the results were uncomfortable and immediately
adopted:

| metric (Prime setups, honest fills) | assumed fills | honest fills |
|---|---|---|
| fill rate | 100% (by assumption) | ~30% |
| expectancy per attempt | +0.65R | ≈ 0.00R (statistically zero) |
| dominant cancel reason | — | price ran to target without filling (~67%) |

The evaluation assumption had been subsidizing every baseline by roughly
+0.65R per attempt. No parameter had been overfit; no rule had been broken.
The *measurement itself* had been optimistic, which is the failure mode that
matters: it rewards exactly the strategies that exploit it (deep pullback
entries that look brilliant when fills are free and rarely fill in reality).

The system's response, in order: record the number, re-baseline everything on
the honest ruler, mark the machine-learning layer's earlier passing grade as
suspect pending re-evaluation on honest labels, and redirect the roadmap from
"add signals" to "fix entries." The same investigation surfaced a silent data
bug — intraday requests falling back to daily bars — caught precisely because
no component is a black box, and pinned by a full-coverage test the same
night.

## The learning layer: trust is rented, never owned

Bliss's ML component is deliberately boring: a transparent meta-labeler that
learns p(win) per setup from harness outcomes, evaluated on calibration (does
"70%" win 70% of the time?) rather than accuracy theater. Its designed role
in sizing is a **conviction ladder**: position size may scale up only with
the *realized, out-of-sample* hit rate of the model's confidence bucket —
keyed to the Wilson lower bound, capped, and wrapped in automatic
self-distrust: if a bucket's live reliability decays below its band, the
ladder collapses to base size fleet-wide and each tier must re-earn
activation. Authority is granted by verified track record and revoked by the
same mechanism, without appeal.

## Guardrails against everyone — including the operator

Live trading is not disabled by configuration; it is disabled *in code* — the
live broker constructor raises. Enabling it is a deliberate code change
gated on a rigor bar (calibrated, non-overfit, positive expectancy on the
honest ruler) plus explicit human sign-off, per market. Below that sit a
fleet-wide daily kill switch, per-trade risk caps, a correlation rule that
counts correlated positions as one, and account-law awareness (pattern-day-
trading budgets, settled-funds ledgers) so the paper record can never
outperform what is legally replicable live.

## How it was built: a human-AI collaboration protocol

Bliss is a working case study in delegated agency. One non-professional
developer directs; Claude instances build, measure, and challenge:

- **Specs as prompts.** Features arrive as staged, testable scripts with
  non-negotiables at the top; an agent session implements one stage, runs the
  ruler, prints a verdict, and stops for human review.
- **Shared memory.** A single `SESSION_STATE.md` handoff document makes any
  fresh session fully operational in one read and is updated whenever
  operational reality changes.
- **The AI as skeptic, not cheerleader.** The measurement layer exists partly
  to referee the humans: when "make every trade fill" was proposed, the
  harness had already priced that wish — guaranteed fills lost more to chasing
  than misses cost — and the design conversation moved from philosophy to
  arithmetic.

## Why this might interest AI-safety-minded readers

Bliss is small, but it is an end-to-end exercise in problems safety research
cares about: specification gaming by evaluation pipelines (the phantom-fill
subsidy is reward hacking's quiet cousin), honest reporting under incentive
to look good, calibrated confidence as a condition for delegated authority,
trust mechanisms with automatic revocation, and hard action-space guardrails
around an increasingly autonomous agent. None of it is claimed as novel
research — it is practiced engineering against those failure modes, with
receipts.

## Status and honesty note

As of July 2026: paper trading a $1,000 forward test through Interactive
Brokers; measured edge under honest fills is currently **zero** — stated
plainly because that is the point of the system. The strategy layer,
research dossiers, and edge work remain private.

*Bliss is a research project. Nothing here is financial advice, and no
returns are promised or implied.*
