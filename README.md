# Loop engineering, tested against real money

In July 2026 I ran a study on a popular thesis: that you can wrap an AI agent in a self-improving loop and it will discover profitable trading strategies while you sleep.

I built the loop. I ran it on real market data. Then I built a fleet of trading agents as the physical embodiment of what the study found — and kept revising the study when the fleet proved parts of it wrong.

This is the write-up of both: [the research document](#the-research-document) that guided every design decision, and the systems that tested it.

**The short version: the infrastructure claim is true. The alpha claim is marketing.** A loop amplifies whatever edge already exists — it cannot manufacture edge that isn't there. What a loop *can* do, extremely well, is tell you cheaply and quickly that an idea doesn't work.

---

## The decisive principle

One line from the study governs everything downstream:

> **A loop can only safely automate what it can objectively verify.**

*"Did these trades settle profitably, net of costs?"* is a fact. *"Is this signal alpha?"* is a judgment. Loops automate the first and must never be trusted with the second.

Every architectural decision in the fleet follows from taking that seriously — and from a second principle that I now think is the more important of the two:

> **No single output should be trusted without independent verification — mine or a model's.**

So one model proposes a change, a **separate** model audits it independently, and disagreements are settled with evidence — a test, a measurement, a receipt — never by picking the answer I preferred. It works in both directions, which is the part I'd point to: the auditing model has caught real errors in the builder's work, and the builder has caught errors in the auditor's. I've watched both be wrong and be corrected by a measurement in the same week.

That process is the skill. The systems are what it produced.

---

## The four conditions — and why my first loop was structurally doomed

The most useful thing in the study is a test for whether a ratchet loop can work at all. A loop needs all four:

| Condition | Meaning |
|---|---|
| **Verifiable** | A measurable result exists |
| **Reversible** | You can revert to the last good state |
| **Short horizon** | Feedback arrives *frequently* |
| **Bounded** | The action space is narrow |

My Kalshi weather-market bot satisfied three of them and **catastrophically failed the third.** Trading settlements arrive over days and carry heavy noise, so the loop got 154 data points total. Reference implementations of this pattern retain ~20 optimizations from ~700 experiments in *two days*, because their feedback arrives in five-minute cycles.

I had derived statistically that my evaluation gate was underpowered. The study explained *why* in one line: **the horizon was never short enough to ratchet.**

That single insight reframed the whole project. The constraint was never rigor — it was **feedback speed.**

---

## What the process caught

**A 97% win rate that was fiction.** My BTC event-contract bot's backtest reported a 97% win rate and thousands in paper profit. The audit found it was scoring trades against the model's *own* estimated prices rather than real market prices — a system grading its own homework. That became a fleet-wide rule: any backtest scored against model-own prices is inadmissible evidence. The bot was rebuilt measurement-first, and a daily tripwire now verifies captured entry prices stay real.

**A strategy retired by a criterion I wrote before looking.** On the weather bot I pre-committed a kill rule in writing: after 100 held-out settled trades, if the best achievable expected value net of all costs was still at or below zero, the strategy was done. No extensions. The verdict came at 154 settled trades: **−$0.023 per contract.** Cost of that answer: under $5 of compute and zero capital at risk — versus the six weeks and real money the same lesson cost me on an earlier bot I let run past the evidence.

**A precise verdict instead of a slow fade.** My SEC-filings agent ran sixteen days, placed zero live trades, and never proved its thesis. I retired it with a deliberately exact verdict: *never proven — retired at cost-of-proof, not disproven.* Knowing the difference between "this is dead" and "proving this would cost more than it could return" is the same muscle, applied before money moved.

**Discipline under a real loss.** A small 13F-mirror agent traded real money on a ring-fenced account and went against me. I closed it at the pre-committed stop rather than on a story I told myself. What I do when a system I built loses money is diagnose and constrain it — not rationalize it, and not raise the stakes to prove it wrong.

**The system overruled a result I wanted.** The one I'm proudest of. A strategy cleared the statistical bar on its own numbers — a genuine "this works" signal. The gate then asked a second question: *out of how many things did we try?* Across 17 registered experiments, a single winner is what luck looks like, so the verdict was automatically demoted from CONTINUE to **INSUFFICIENT**. I built a machine that tells me no, and it was right.

---

## Verification lives in the code, not in my judgment

- **Models propose; they never execute.** Every trade clears a deterministic guardrail layer — position limits, concentration caps, drawdown rules, a kill switch — before it can reach a broker.
- **Order state is an append-only event log**, not a value I overwrite. On restart the system replays the log and reconciles against the broker as source of truth, so a crash mid-trade doesn't leave me guessing.
- **Order submission is idempotent** on a client-side ID, so a retried network call can't silently double an order.
- **Two-leg trades use a saga pattern.** If the second leg fails after the first fills, the system runs a compensating transaction to unwind the exposure, with an escalation path if the unwind itself fails.

Once several agents could touch the same brokerage account, a problem appeared that no single bot's code could solve. So I built a coordination layer — deliberately **read-only, it cannot place a single trade.** It reads a standardized snapshot from every agent and enforces a fail-closed ownership rule: an agent that can't confirm it owns a position is blocked from selling it, full stop. It tracks fleet-wide capital and AI spend no individual agent can see, and runs continuously behind **337 of its own tests.**

The pattern: when directing multiple autonomous agents created a *new* category of risk, I built the governance for that risk before scaling further — not after something went wrong.

---

## The diagnosis that's driving the next phase

Late in the study I added research on self-improving and co-evolutionary systems. It produced the sharpest criticism of my own fleet, and I wrote it into the document verbatim:

> Umbrella already has **selection pressure** — strategies get floored, the graveyard buries them, the registry tracks who is alive. Things die on evidence. **What it lacks is birth.** Nothing spawns. Without variation it is a very disciplined hospice.

That was uncomfortable and correct. It also matched something I found in the running system a week later: the research engine caps how many experiments run at once, and **two of its three slots were occupied by a bot I'd shut down days earlier** — experiments that could never produce another observation. Real discovery capacity was ~1–2 attempts per year against a design capacity of ~24. Clearing them took an afternoon and multiplied throughput across the whole operation, without loosening a single standard.

Theory predicted the constraint. The system hit it. The fix followed the theory. That loop — between the written study and the running fleet — is the actual product here.

So the roadmap is now organized around one idea: **more honest shots per month, never a lower bar.** Shorten time-to-verdict so dead ideas free capacity faster. Score the decisions a strategy *considered* but didn't take — free evidence, no capital. Forecast markets the fleet already watches at zero cost. Reuse measurement machinery so a new idea takes a day to test instead of a month.

The study is also clear about where automation must stop, and I've adopted the line as policy: **automate variation only where the horizon is short and the check is objective** — execution and cost parameters, never strategy theses. Automated variation over slow, noisy, expensive-to-evaluate strategies is how you build a very sophisticated overfitting machine.

---

## The research document

`LOOP_ENGINEERING_FINDINGS.md` is a ~17,000-word working reference in 20 parts, maintained privately alongside the fleet. It is not a summary written after the fact — it's the live design document, and it's been revised repeatedly by adversarial review *against the running codebase*, including corrections that invalidated its own earlier conclusions:

- The evaluation gate's statistical power had never been computed. Individually reasonable thresholds were jointly so strict the gate would almost never fire — which reads as "nothing left to improve" when it actually means "the test can't see at this sample size."
- A gate tested only against a correctly specified noise model will pass a dishonest one.
- "Graph" had been three different things the whole time — orchestration graph, commit lineage, and knowledge graph — and collapsing them was hiding a real design decision about *where memory belongs*.
- Twenty sections had treated accumulated memory as a pure benefit. The adaptive-systems literature is clear that accumulation is also a decay vector: a living system needs **pruning** as much as recording.

The document's restated thesis is better than the one I started with:

> **The bottleneck is not the next model call. It is the placement of memory and evaluation.**

Applying maker/checker to the research itself — not just the code — is why I trust it. One model compiled the findings; a second reviewed them adversarially and found five real errors, including the statistical-power gap above.

---

## What this is proof of

A specific, verifiable capability: I can take an unproven technical thesis, design an experiment that could actually falsify it, build the system that runs the experiment, and report the result honestly — including when the result costs me the outcome I wanted.

Alongside that: directing multiple AI systems toward a goal, designing independent verification into the parts that matter, and keeping hard guardrails around anything touching real risk.

That maps to roles built around operating and scaling AI-driven workflows rather than writing production code: AI operations, AI enablement, or revenue/GTM operations where the automation layer is native to the job.

The repos are public. The process that produced them is what I'd want to talk about.

---

## The fleet

| Repo | What it is | Status |
|---|---|---|
| [multi-agent-llm-trading-platform](https://github.com/brooksmoore/multi-agent-llm-trading-platform) | Four Claude models running differentiated mandates behind a deterministic risk layer; 844 tests | Paper trading |
| [btc-kalshi-contract-trader](https://github.com/brooksmoore/btc-kalshi-contract-trader) | Short-horizon BTC event contracts, rebuilt measurement-first after the 97% artifact | Paper; forward test running |
| [kalshi-market-maker](https://github.com/brooksmoore/kalshi-market-maker) | Weather strategy retired by the pre-committed test above; repo now hosts a separately pre-registered maker-side experiment with frozen kill bars | New experiment in simulation |
| music-market-trader *(private)* | Paper Kalshi music-market trader with a pre-registered research window; no order-placing code by construction | Paper, window running |
| umbrella-fleet-brain *(private)* | The read-only governance layer — ownership ledger, fleet exposure, AI-spend metering, experiment registry, scored analyst; 337 tests | Running; never trades |
| [pure-arb-bot](https://github.com/brooksmoore/pure-arb-bot) | Cross-venue Kalshi/Polymarket structural arbitrage; ~200K markets canonicalized per cycle | Concluded |
| [hood-ai-trading-agent](https://github.com/brooksmoore/hood-ai-trading-agent) | LLM reasoning on small-cap SEC filings behind an adversarial auditor | Retired at cost-of-proof; zero live trades (postmortem in repo) |
| [portfolio-mirror-agent](https://github.com/brooksmoore/portfolio-mirror-agent) | Deterministic 13F mirror-basket agent; append-only ownership ledger, fail-closed broker safety | Concluded at a pre-set stop |

Strategies are paper-traded until the evidence earns capital. The reusable rules, separated from the bots, are in [the verification playbook](VERIFICATION_PLAYBOOK.md).

---

*Last updated 2026-08-02. Every number above is verifiable in the repos' commit histories, test suites, and audit ledgers. Two coordination and research repos run privately and are noted without links.*
