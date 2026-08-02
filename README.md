# How I run a multi-model AI build process as a non-engineer

I'm a sales consultant with a Ross MM, not a software engineer. Working around a full-time commercial job, I've used Claude, Grok, and other coding assistants to build and run a fleet of seven automated trading-research agents plus a governance layer that supervises them.

None of them has made money. The fleet currently holds **$0.00 in live capital**, on purpose.

That sounds like a failure, so let me be precise about what I think I've actually built: not a profitable trading system, but a research process that reliably tells me the truth — including when the truth is "this doesn't work" or "you don't have enough evidence to say yet." Every strategy I've tested has been killed by that process. Several of them were killed *before* they could cost me anything.

The honest constraint I started from: **I can direct these models, but I can't personally read every line they write.** Everything below follows from taking that seriously instead of pretending otherwise.

---

## The problem I had to solve

Trusting a single model's output on anything touching real money is a bad idea — not because models are untrustworthy in the abstract, but because I have no way to independently check a claim like *"this fixes the race condition"* if I can't read the diff closely enough to know.

So I stopped treating any one model as the source of truth and built a process instead:

- One model proposes a change. A **separate** model audits it independently.
- Disagreements get **arbitrated with evidence** — a test, a measurement, a receipt — not resolved by picking whichever answer I liked better.
- Every project keeps a running status file and an append-only ledger, so context survives between sessions instead of living in my head. I genuinely cannot track eight codebases from memory.

**That process is the skill. The bots are the artifact it produced.**

---

## Verification lives in the code, not in my judgment

Because I can't personally vouch for correctness, I pushed the checking into the system itself:

- **Models propose; they never execute.** Every trade clears a deterministic guardrail layer — position limits, concentration caps, drawdown rules, a kill switch — before it can reach a broker.
- **Order state is an append-only event log**, not a value I overwrite. On restart the system replays the log and reconciles against the broker as source of truth, so a crash mid-trade doesn't leave me guessing.
- **Order submission is idempotent** on a client-side ID, so a retried network call can't silently double an order.
- **Two-leg trades use a saga pattern.** If the second leg fails after the first fills, the system runs a defined compensating transaction to unwind the exposure, with an escalation path if the unwind itself fails.

None of this required me to write the implementation by hand. It required knowing what to insist on before I'd trust an answer.

---

## It's a fleet, so it needed its own governance layer

Once more than one agent could touch the same brokerage account, a problem appeared that no single bot's code could solve: nothing stopped one agent from acting on a position it didn't own, and nothing gave me one place to see total exposure.

So I built a separate coordination layer — deliberately **read-only, it cannot place a single trade**. It reads a standardized state snapshot from every agent and enforces a fail-closed ownership rule: an agent that can't confirm it owns a position is blocked from selling it, full stop. It tracks fleet-wide capital and AI spend that no individual agent can see, and it runs continuously behind **334 of its own tests**.

The pattern I'd point to: when directing multiple autonomous agents created a *new* category of risk, I built the governance for that risk before scaling further — not after something went wrong.

---

## What the process has actually caught

I think the catches are better evidence than any win would be.

**A 97% win rate that was fiction.** My BTC event-contract bot's backtest reported a 97% win rate and thousands in paper profit. The audit found it was scoring trades against the model's *own* estimated prices rather than real market prices — a system grading its own homework. That became a fleet-wide rule: any backtest scored against model-own prices is an inadmissible class of evidence. The bot was rebuilt measurement-first, and a daily tripwire now checks that captured entry prices stay real.

**A strategy killed by a criterion I wrote before looking.** On a Kalshi weather trader I pre-committed a kill rule in writing: after 100 held-out settled trades, if the best achievable expected value net of all costs was still at or below zero, the strategy was dead. No extensions. The verdict came at 154 settled trades: **−$0.023 per contract.** Total cost of that answer: under $5 of compute and zero capital at risk — versus the six weeks and real money the same lesson cost me on an earlier bot that I let run past the evidence.

**A strategy that got a grave instead of a slow fade.** My SEC-filings agent ran sixteen days, placed zero live trades, and never proved its thesis. I retired it with a deliberately precise verdict: *never proven — retired at cost-of-proof, not disproven.* Knowing the difference between "this is dead" and "proving this would cost more than it could return" is the same muscle, applied before money moved.

**A live position closed at a pre-set stop.** A small 13F-mirror agent traded real money on a ring-fenced account. It went against me. I closed it at −32.6% on roughly $70, on the pre-committed terms rather than on a story I told myself. The fleet has held zero live capital since. What I do when a system I built loses real money is diagnose and constrain it — not rationalize it, and not double the stakes to prove it wrong.

**The system overruled a result I would have liked.** This is the one I'm proudest of. A btc strategy cleared the statistical bar on its own numbers — a genuine "this works" signal. The gate then asked a second question: *out of how many things did we try?* With 17 registered experiments and zero prior survivors, one winner is what luck looks like. The verdict was automatically demoted from CONTINUE to **INSUFFICIENT**. The machine told me no, and it was right to.

---

## Where it stands, honestly

**17 experiments registered. 0 survivors. $0.00 live capital.**

I want to be straight about that number rather than dress it up, because the arithmetic is genuinely reassuring once you look at it:

If roughly 5% of tested strategy ideas have real durable edge — a reasonable assumption in this domain — then 17 attempts should produce **fewer than one** survivor on average. There's about a 42% chance of seeing exactly zero even if the bar is calibrated perfectly and real edges exist. **Zero survivors from 17 attempts isn't evidence the bar is too strict. It's evidence I haven't taken many shots yet.**

That distinction is the whole game, and it points at what's actually limiting me. It isn't rigor. It's throughput.

---

## What's next: the constraint moved

The interesting recent discovery was diagnostic. The research engine caps how many experiments can run at once, and I found that **two of its three slots were occupied by a bot that had been shut down a week earlier** — experiments that could never produce another observation. Real discovery capacity was ~1–2 attempts per year against a design capacity of ~24.

Clearing those slots cost an afternoon and multiplied the throughput of the entire operation. Nothing was loosened to do it.

That reframed the roadmap around a single idea: **more honest shots per month, never a lower bar.** The levers are all speed and volume, not permissiveness —

- shortening time-to-verdict, so dead ideas free capacity faster
- scoring the decisions a strategy *considered* but didn't take, which costs nothing and multiplies evidence
- forecasting markets the fleet already watches, at zero capital and zero LLM cost, to build calibration without waiting on trades
- reusing measurement machinery across strategies so a new idea takes a day to test, not a month

The honest ambition: not "find a winning strategy," but **compress the cost of finding out from six weeks and real money down to days and a few dollars** — and then do it often enough that the base rate works in my favor. That's a solvable engineering problem, which is exactly why I find it a more interesting one than picking trades.

---

## What this is proof of

I'm not pitching myself as an engineer. I'm pitching a specific, verifiable capability:

I can direct multiple AI systems toward a goal, build independent verification for the parts I can't personally check, keep hard guardrails around anything touching real risk, and hold that discipline when no one is making me — including when the discipline costs me the answer I wanted.

That maps to roles built around operating and scaling AI-driven workflows rather than writing production code from scratch: AI operations, AI enablement, or revenue/GTM operations where the automation layer is native to the job rather than a side project.

The repos are public. The process that produced them is what I'd want to talk about.

---

## The fleet

| Repo | What it is | Status |
|---|---|---|
| [multi-agent-llm-trading-platform](https://github.com/brooksmoore/multi-agent-llm-trading-platform) | Four Claude models running differentiated mandates behind a deterministic risk layer; 844 tests | Paper trading |
| [pure-arb-bot](https://github.com/brooksmoore/pure-arb-bot) | Cross-venue Kalshi/Polymarket structural arbitrage; ~200K markets canonicalized per cycle | Concluded — no live path |
| [hood-ai-trading-agent](https://github.com/brooksmoore/hood-ai-trading-agent) | LLM reasoning on small-cap SEC filings behind an adversarial auditor | Retired at cost-of-proof; zero live trades (postmortem in repo) |
| [btc-kalshi-contract-trader](https://github.com/brooksmoore/btc-kalshi-contract-trader) | Short-horizon BTC event contracts, rebuilt measurement-first after the 97% artifact | Paper; retrospective edge floored at −1.27¢/contract over 167 fills, forward test running |
| [portfolio-mirror-agent](https://github.com/brooksmoore/portfolio-mirror-agent) | Deterministic 13F mirror-basket agent; append-only ownership ledger, fail-closed broker safety | Concluded — closed at a pre-set stop, −32.6%; no live capital |
| [kalshi-market-maker](https://github.com/brooksmoore/kalshi-market-maker) | Weather strategy retired by the pre-committed test above; repo now hosts a separately pre-registered maker-side experiment with frozen kill bars | Weather retired (verdict in repo); new experiment in simulation |
| music-market-trader *(private)* | Paper Kalshi music-market trader with a pre-registered research window; no order-placing code by construction | Paper, window running |
| umbrella-fleet-brain *(private)* | The read-only governance layer — ownership ledger, fleet exposure, AI-spend metering, experiment registry, scored analyst; 334 tests | Running; never trades |

The reusable rules, separated from my bots, are in [the verification playbook](VERIFICATION_PLAYBOOK.md).

---

*Last updated 2026-08-02. Every number above is verifiable in the repos' commit histories, test suites, and audit ledgers. Two coordination and research repos run privately and are noted without links.*
