# Loop engineering, tested against real money

Can you wrap an AI agent in a self-improving loop and have it find profitable trading strategies? Rather than argue about it, I've spent many months building the answer.

I built the loop, ran it on real market data, and grew a fleet of trading agents as the working embodiment of what the research found — revising the research whenever the fleet taught me something it had missed.

This is the write-up of both: [the research document](#the-research-document) that guides every design decision, and the systems that test it.

**What's established so far:** a well-built loop is exceptional at identifying which ideas deserve capital — quickly, cheaply, and before anything is at risk. That's the foundation, and it's working. Turning that engine toward edge discovery at scale is the work in front of me, and the path to it is now a concrete engineering problem rather than an open question.

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

## The four conditions — and the insight that reset my roadmap

The most useful thing in the research is a test for whether a ratchet loop can work at all. A loop needs all four:

| Condition | Meaning |
|---|---|
| **Verifiable** | A measurable result exists |
| **Reversible** | You can revert to the last good state |
| **Short horizon** | Feedback arrives *frequently* |
| **Bounded** | The action space is narrow |

My Kalshi weather-market bot satisfied three of them cleanly. The third is where the real lever turned out to be: trading settlements arrive over days, so the loop accumulated 154 data points. Reference implementations of this pattern retain ~20 optimizations from ~700 experiments in *two days*, because their feedback arrives in five-minute cycles.

I had already derived statistically that my evaluation gate was underpowered. The research explained *why* in one line: **the horizon wasn't short enough to ratchet.**

That single insight reframed the entire project, and it's the most valuable thing I've learned so far. The constraint was never rigor — it's **feedback speed.** That's a tractable engineering target, and nearly everything I'm building now aims at it.

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

## The frontier: from selection to variation

Adding research on self-improving and co-evolutionary systems produced the sharpest read on where my fleet goes next:

> Umbrella already has **selection pressure** — strategies are evaluated, retired on evidence, and recorded so they're never rebuilt by accident. Things resolve on data rather than opinion. **The half still to build is variation:** systematically generating candidates, not only judging them.

Selection is the hard half and it's solved. Variation is the frontier — and it's a build problem, which is the good kind.

The running system confirmed the same thing from a different direction. The research engine caps how many experiments run at once, and I found **two of its three slots held by a bot I'd retired days earlier**. Clearing them took an afternoon and multiplied discovery throughput across the whole operation — from roughly 1–2 viable attempts a year to the ~24 the design supports, without loosening a single standard.

Theory pointed at the constraint. The system confirmed it. The fix followed the theory. That loop — between the written research and the running fleet — is the real product here, and it's compounding.

So the roadmap is now organized around one idea: **more honest shots per month, never a lower bar.** Shorten time-to-verdict so dead ideas free capacity faster. Score the decisions a strategy *considered* but didn't take — free evidence, no capital. Forecast markets the fleet already watches at zero cost. Reuse measurement machinery so a new idea takes a day to test instead of a month.

The research is equally clear about where automation pays off first, and I've adopted it as policy: **automate variation where the horizon is short and the check is objective** — execution and cost parameters to start, with the scope widening as feedback speed improves. Sequencing it that way is what keeps a fast system an accurate one.

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

A specific, verifiable capability: I can take an unproven technical thesis, design an experiment that genuinely tests it, build the system that runs it, and act on what comes back — including when the answer isn't the one I was hoping for. That last part is what makes the rest of it worth trusting.

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
