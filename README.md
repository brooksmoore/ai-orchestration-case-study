# How I run a multi-model AI build process as a non-engineer

I'm a sales consultant with a Ross MM, not a software engineer. Over the past several months, working around a full-time commercial job, I've used Claude, Grok, and other coding assistants to design and build two public multi-agent systems: a four-agent trading platform (github.com/brooksmoore/multi-agent-llm-trading-platform) and a deterministic cross-venue arbitrage bot (github.com/brooksmoore/pure-arb-bot). Neither has made money. That's not the point of this write-up. The point is what building them required me to figure out, because the honest constraint I started from was: I can direct these models, but I can't personally read every line they write.

## The actual problem I had to solve

Early on it was obvious that trusting a single model's output on anything touching real capital was a bad idea, not because the model is untrustworthy in some abstract sense, but because I have no way to independently verify a claim like "this fixes the race condition" if I can't read the diff closely enough to know. So I stopped treating any one model as the source of truth and built a process instead: one model proposes a change, a second, separate model audits it independently, and disagreements get arbitrated rather than resolved by picking whichever answer I liked better. I keep a running memory file and a ledger per project so context survives between sessions instead of living in my head, because I genuinely cannot track the state of five-plus codebases by memory alone.

That process is the actual skill. The bots are the artifact it produced.

## What the systems themselves demonstrate

Because I couldn't personally vouch for correctness, I pushed the verification into the code itself rather than into my own judgment:

- Every trade a model proposes clears a deterministic guardrail layer (a `RiskGate` class) before it reaches a broker — position limits, concentration caps, drawdown rules, a kill switch. The model proposes; it never executes directly.
- Order state is stored in an append-only event log, not a value I overwrite. On restart, the system replays that log and reconciles against the broker, which is treated as the source of truth, so a crash mid-trade doesn't leave me guessing what actually happened.
- Order submission is idempotent on a client-side order ID, so a retried network call can't silently double an order. I audited this myself recently and found it's solid where it matters most (the live-broker paths) and had a real, unfixed gap in an older, now-shelved project — which is itself the kind of thing this process is built to catch.
- The arbitrage bot's two-leg execution follows what's formally called a saga pattern: if the second leg of a trade fails after the first one filled, the system doesn't just log an error, it runs a defined compensating transaction to unwind the exposure, with an explicit escalation path if the unwind itself fails.

None of this required me to write the implementation by hand. It required me to know what questions to ask, what to insist on before I'd trust an answer, and how to structure a process where an error in one model's output gets caught by another step rather than by me happening to notice.

## It's a fleet, not one bot, and it needed its own governance layer

The trading platform above is one piece of a wider set of agents I run across two brokers, Alpaca and Robinhood. One of them (a smaller, quarterly rebalancing agent that mirrors public 13F filings) trades with real money on a shared Robinhood account; the rest run in paper mode. Once more than one agent could touch the same brokerage account, a new problem showed up that no single bot's code could solve: nothing stopped one agent from acting on a position it didn't actually own, and nothing gave me one place to see the whole fleet's exposure at once.

So I'm building a separate coordination layer for exactly that problem, deliberately read-only: it can't place a single trade. It reads a standardized state snapshot from every agent, enforces a fail-closed ownership rule (an agent that can't confirm it owns a position is blocked from selling it, full stop), and tracks whole-fleet exposure and AI spend that no individual agent can see on its own. It's early, about a week old as of this writing, with its own test suite passing alongside the two bots' suites it verifies against. I'm not claiming it's finished. I'm pointing to it because it's the clearest evidence of the actual skill: when directing multiple autonomous agents created a new risk that didn't exist when I was running one, I built the governance layer for that risk before scaling further, instead of after something went wrong.

## Where it got tested for real

In the spring I moved a small live position and lost money on it. I wrote an honest postmortem, tightened the gates that let that happen, and moved back to paper testing rather than either quitting or doubling the stakes to "prove it wrong." That's the actual evidence I'd point to over a hypothetical backtest: what I do when a system I built loses real money is diagnose and constrain it, not rationalize it or walk away.

## The strongest single piece of evidence: killing my own strategy with a pre-committed test

That postmortem taught me something specific: my earlier bot ran six weeks past the point the evidence said its thesis was wrong, because every incremental fix produced a plausible local improvement. So on the rebuilt version of that strategy (a Kalshi weather-market trader), I did it differently. Before evaluating anything, I pre-committed a kill criterion in writing: after 100 held-out settled trades, if the best achievable expected value — net of fees, spread, and slippage, with every fixable execution leak closed — was still at or below zero, the strategy would be declared dead. No extensions, no "keep watching."

The evaluation harness itself was built maker/checker: one model diagnoses losses on a training window and proposes one change; a separate, pure-Python checker scores that change on strictly later, held-out data, with an effect-size bar so a proposal can't get promoted by noise. The rule underneath it all: the data that justifies a change must never include the data the change was derived from.

The verdict came back at 154 settled paper trades: net expected value of −$0.023 per contract. The edge was dead, and no execution improvement could rescue it — closing the biggest cost leak recovered about a quarter of the bleed without flipping the sign. Total cost of that answer: under $5 of compute and zero capital at risk, versus the six weeks and real money the same lesson cost me the first time. I consider that negative result the most valuable output of the whole project, because it proves the process tells me the truth rather than what I'd like to hear. The strategy is retired with the verdict written into its repo, instead of quietly running in the background on hope.

## What this is actually proof of

I'm not pitching myself as an engineer. I'm pitching a specific, verifiable capability: I can direct multiple AI systems toward a goal, build in independent verification for the parts I can't personally check, keep guardrails around anything touching real risk, and hold that discipline even when no one's making me. That's a close match for roles built around operating and scaling AI-driven workflows rather than writing production code from scratch — AI operations, AI enablement, or revenue/GTM operations roles where the automation layer is native to the job, not a side project.

The repos are public. The process that produced them is the thing I'd actually want to talk about in an interview.

## The fleet

| Repo | What it is | Status |
|---|---|---|
| [multi-agent-llm-trading-platform](https://github.com/brooksmoore/multi-agent-llm-trading-platform) | Four Claude models running differentiated mandates behind a deterministic risk layer | Paper trading |
| [pure-arb-bot](https://github.com/brooksmoore/pure-arb-bot) | Cross-venue Kalshi/Polymarket structural arbitrage; ~200K markets canonicalized per cycle; 480+ tests | Paper, live-gated |
| [hood-ai-trading-agent](https://github.com/brooksmoore/hood-ai-trading-agent) | LLM reasoning on small-cap SEC filings with a forward-only calibration gate before any live decision | Paper, OOS clock running |
| [truleo-13f-mirror](https://github.com/brooksmoore/truleo-13f-mirror) | Deterministic 13F mirror-basket agent on a ring-fenced account | Live since June 2026 |
| [kalshi-market-maker](https://github.com/brooksmoore/kalshi-market-maker) | Weather-market trader, retired by the pre-committed test described above | Retired (verdict in repo) |
| [bar-generator](https://github.com/brooksmoore/bar-generator) | A free web toy on Cloudflare Workers — proof the process generalizes beyond trading | Deployed |

*Written July 2026. Everything above is verifiable in the repos' commit histories, test suites, and audit ledgers.*
