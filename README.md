# How I run a multi-model AI build process as a non-engineer

I'm a sales consultant with a Ross MM, not a software engineer. Working around a full-time commercial job, I've used Claude, Grok, and other coding assistants to build and run a fleet of seven automated trading-research agents, plus a governance layer that supervises all of them and can overrule any one of them.

The honest constraint I started from: **I can direct these models, but I can't personally read every line they write.**

Everything I've built follows from taking that seriously. Instead of trusting my own review, I built a process where the models check each other — and then pushed the checking down into the code itself, so correctness doesn't depend on me noticing.

The result is a research operation that can tell me whether an idea works in **days, for a few dollars** — where the same answer used to cost me six weeks and real money.

---

## The process

Trusting a single model's output on anything touching money is a bad idea — not because models are untrustworthy in the abstract, but because I can't independently check a claim like *"this fixes the race condition"* if I can't read the diff closely enough to know.

So I stopped treating any one model as the source of truth:

- One model proposes a change. A **separate** model audits it independently.
- Disagreements are **settled with evidence** — a test, a measurement, a receipt — never by picking the answer I liked.
- Every project keeps a status file and an append-only ledger, so context survives between sessions instead of living in my head. I can't track eight codebases from memory, so I don't try.

**That process is the skill. The bots are what it produced.**

It works in both directions, which is the part I'd point to. The auditing model has caught real errors in the builder's work — and the builder has caught errors in the auditor's. When they disagree, a measurement decides. I've watched both models be wrong and be corrected by data in the same week.

---

## Verification lives in the code, not in my judgment

- **Models propose; they never execute.** Every trade clears a deterministic guardrail layer — position limits, concentration caps, drawdown rules, a kill switch — before it can reach a broker.
- **Order state is an append-only event log**, not a value I overwrite. On restart the system replays the log and reconciles against the broker as source of truth, so a crash mid-trade doesn't leave me guessing.
- **Order submission is idempotent** on a client-side ID, so a retried network call can't silently double an order.
- **Two-leg trades use a saga pattern.** If the second leg fails after the first fills, the system runs a compensating transaction to unwind the exposure, with an escalation path if the unwind itself fails.

None of this required writing the implementation by hand. It required knowing what to insist on before trusting an answer.

---

## The fleet needed its own governance layer

Once more than one agent could touch the same brokerage account, a problem appeared that no single bot's code could solve: nothing stopped one agent from acting on a position it didn't own, and nothing gave me one place to see total exposure.

So I built a coordination layer — deliberately **read-only, it cannot place a single trade**. It reads a standardized snapshot from every agent and enforces a fail-closed ownership rule: an agent that can't confirm it owns a position is blocked from selling it, full stop. It tracks fleet-wide capital and AI spend that no individual agent can see, and runs continuously behind **334 of its own tests**.

The pattern: when directing multiple autonomous agents created a *new* category of risk, I built the governance for that risk before scaling further — not after something went wrong.

---

## What the process catches

**A 97% win rate that was fiction.** My BTC event-contract bot's backtest reported a 97% win rate and thousands in paper profit. The audit found it was scoring trades against the model's *own* estimated prices rather than real market prices — a system grading its own homework. That became a fleet-wide rule: any backtest scored against model-own prices is inadmissible evidence. The bot was rebuilt measurement-first, and a daily tripwire now verifies that captured entry prices stay real.

**A strategy retired by a criterion I wrote before looking.** On a Kalshi weather trader I pre-committed a kill rule in writing: after 100 held-out settled trades, if the best achievable expected value net of all costs was still at or below zero, the strategy was done. No extensions. The verdict came at 154 settled trades: **−$0.023 per contract.** Cost of that answer: under $5 of compute and zero capital at risk — versus the six weeks and real money the same lesson cost me on an earlier bot I let run past the evidence.

**A precise verdict instead of a slow fade.** My SEC-filings agent ran sixteen days, placed zero live trades, and never proved its thesis. I retired it with a deliberately exact verdict: *never proven — retired at cost-of-proof, not disproven.* Knowing the difference between "this is dead" and "proving this would cost more than it could return" is the same muscle, applied before money moved.

**Discipline under a real loss.** A small 13F-mirror agent traded real money on a ring-fenced account and went against me. I closed it at the pre-committed stop rather than on a story I told myself, and moved back to paper. What I do when a system I built loses money is diagnose and constrain it — not rationalize it, and not raise the stakes to prove it wrong.

**The system overruled a result I wanted.** This is the one I'm proudest of. A strategy cleared the statistical bar on its own numbers — a genuine "this works" signal. The gate then asked a second question: *out of how many things did we try?* Across 17 registered experiments, a single winner is what luck looks like, so the verdict was automatically demoted from CONTINUE to **INSUFFICIENT**. I built a machine that tells me no, and it was right.

---

## What I'm building toward

The recent breakthrough was diagnostic rather than strategic, and it's the most useful thing I've learned all year.

The research engine caps how many experiments run at once. I found **two of its three slots were occupied by a bot I'd shut down a week earlier** — experiments that could never produce another observation. Real discovery capacity was roughly 1–2 attempts per year against a design capacity of ~24. Clearing those slots took an afternoon and multiplied the throughput of the entire operation, without loosening a single standard.

That reframed everything around one idea: **more honest shots per month, never a lower bar.** The levers are all speed and volume —

- shortening time-to-verdict, so dead ideas free capacity faster
- scoring the decisions a strategy *considered* but didn't take — free evidence, no capital
- forecasting markets the fleet already watches at zero cost, building calibration without waiting on trades
- reusing measurement machinery so a new idea takes a day to test instead of a month

The ambition isn't to guess a winning strategy. It's to **compress the cost of finding out** until testing ideas is cheap enough that the base rate works in my favor. That's an engineering problem with a clear solution path, which is why I find it more interesting than picking trades.

---

## What this is proof of

I'm not pitching myself as an engineer. I'm pitching a specific, verifiable capability:

I can direct multiple AI systems toward a goal, build independent verification for the parts I can't personally check, keep hard guardrails around anything touching real risk, and hold that discipline when no one is making me — including when it costs me the answer I wanted.

That maps directly to roles built around operating and scaling AI-driven workflows rather than writing production code: AI operations, AI enablement, or revenue/GTM operations where the automation layer is native to the job rather than a side project.

The repos are public. The process that produced them is what I'd want to talk about.

---

## The fleet

| Repo | What it is | Status |
|---|---|---|
| [multi-agent-llm-trading-platform](https://github.com/brooksmoore/multi-agent-llm-trading-platform) | Four Claude models running differentiated mandates behind a deterministic risk layer; 844 tests | Paper trading |
| [btc-kalshi-contract-trader](https://github.com/brooksmoore/btc-kalshi-contract-trader) | Short-horizon BTC event contracts, rebuilt measurement-first after the 97% artifact | Paper; forward test running |
| [kalshi-market-maker](https://github.com/brooksmoore/kalshi-market-maker) | Weather strategy retired by the pre-committed test above; repo now hosts a separately pre-registered maker-side experiment with frozen kill bars | New experiment in simulation |
| music-market-trader *(private)* | Paper Kalshi music-market trader with a pre-registered research window; no order-placing code by construction | Paper, window running |
| umbrella-fleet-brain *(private)* | The read-only governance layer — ownership ledger, fleet exposure, AI-spend metering, experiment registry, scored analyst; 334 tests | Running; never trades |
| [pure-arb-bot](https://github.com/brooksmoore/pure-arb-bot) | Cross-venue Kalshi/Polymarket structural arbitrage; ~200K markets canonicalized per cycle | Concluded |
| [hood-ai-trading-agent](https://github.com/brooksmoore/hood-ai-trading-agent) | LLM reasoning on small-cap SEC filings behind an adversarial auditor | Retired at cost-of-proof; zero live trades (postmortem in repo) |
| [portfolio-mirror-agent](https://github.com/brooksmoore/portfolio-mirror-agent) | Deterministic 13F mirror-basket agent; append-only ownership ledger, fail-closed broker safety | Concluded at a pre-set stop |

Strategies are paper-traded until the evidence earns capital. The reusable rules, separated from my bots, are in [the verification playbook](VERIFICATION_PLAYBOOK.md).

---

*Last updated 2026-08-02. Every number above is verifiable in the repos' commit histories, test suites, and audit ledgers. Two coordination and research repos run privately and are noted without links.*
