# The Verification Playbook

*The reusable rules behind the case study — how I keep a fleet of AI agents honest when I can't personally read every line they write. Distilled from months of running these systems, two full experiments (one of which killed its own strategy), and a lot of 2026 writing on "loop engineering" and agent orchestration — including Anthropic's own guidance on agent loops and Andrej Karpathy's field notes — tested against my own results rather than taken on faith. Everything with a number behind it comes from my own repos.*

---

## 1. Automate only what you can objectively verify

"Did these trades settle profitably, net of all costs?" is a fact. "Is this signal alpha?" is a judgment. An autonomous loop may only be trusted with the first kind of question. Every failure I've had, and most of the failures I read about, come from letting a system self-certify a judgment as if it were a fact.

## 2. Maker/checker: the model that builds never grades its own work

One model proposes; a separate model — different instructions, ideally a different vendor — audits. Where possible the checker isn't a model at all: it's deterministic code with zero model calls, because a script can't hallucinate a verdict. In my fleet, one model writes implementations, a second writes the acceptance tests *first* and locks them against edits by the builder, and a human gates anything touching real money. The pattern also runs on documents: before acting on a model-compiled research summary, I have a second model attack it. The best catch so far was a promotion gate whose thresholds were individually reasonable but jointly so strict it would almost never fire — a test that "can't see" masquerading as a system with nothing left to find.

## 3. Ground truth or nothing

A backtest scored against the model's own estimated prices is inadmissible — not weak evidence, *inadmissible*. I learned this when a bot of mine reported a 97% win rate that evaporated the moment real market prices replaced the model's own. A network of agents confirming each other while none of them touches reality fails exactly like a single deluded agent, just later and more expensively. The anchors that count: real settlements from the venue, real captured entry prices at signal time, accounting net of every fee and spread, and a holdout the proposing model never sees.

## 4. Walk-forward, effect size, and paired metrics

- **Walk-forward:** the data that justifies a change must never include the data the change was derived from. Train on the past, score on strictly later held-out results, advance the cut each cycle.
- **Effect size, not sign:** with enough proposals, one will beat a small holdout by luck. Require the improvement to exceed roughly twice its standard error, or you are promoting noise on a schedule.
- **Paired metrics (anti-gaming):** every metric you optimize travels with a counter-metric that catches the cheap way to win. EV-per-contract improves trivially if the system just trades less — so it's paired with trade count and total P&L, with floors. A change that improves the headline while collapsing the pair is rejected.

## 5. Pre-commit the kill criterion — dated, in writing, before the window opens

Decide in advance what result kills the strategy, write it down with a date, and forbid extensions. Mine read: after 100 held-out settled trades, if best-achievable EV net of costs is still ≤ 0 with every fixable execution leak closed, the strategy is dead. The verdict arrived at 154 settled trades: **−$0.023 per contract.** The strategy was retired that day. The first version of that same strategy, run without a pre-committed kill, cost me six weeks and real money learning the identical lesson — because every incremental fix produces a plausible local improvement, and the urge to keep iterating on a falsified thesis is the most expensive bias in this whole domain.

A negative verdict delivered cheaply, in paper, is the system *succeeding*.

## 6. Retirement is a verdict, not a fade — and graves are machine-readable

Projects that quietly stop running get rebuilt from scratch a year later by someone (usually me) who forgot why they died. So every kill gets a written postmortem with a precise verdict — including the honest middle category: *never proven, retired at cost-of-proof, not disproven*, which is what happened to a second agent of mine that ran sixteen days without its adversarial auditor ever being able to establish edge. The verdicts live in a machine-readable graveyard that the fleet's tooling checks, so a dead thesis can't be re-proposed without tripping over its own grave.

## 7. Score your advisors, and ban gimmes

Any model whose job is advice gets its predictions logged append-only (immutable IDs, tamper-evident), then graded by deterministic code against an honesty baseline: not a coin flip, but **"beats assuming nothing changes"** — because most confident-sounding predictions about a slow-moving system are just persistence dressed up. Predictions that were already true when made get flagged as gimmes and don't count. An advisor that can't beat "the current state persists" is billing you for weather reports that say yesterday.

## 8. Standing goals: a goal you verify once is an assumption with a timestamp

Every finished invariant graduates into a cheap, read-only shell predicate re-verified on a schedule, forever: the captured prices are still real, the append-only files are still append-only, the test suites still pass, the canonical database still has its rows. Two rules make this honest: a **timeout counts as a violation, not a pass**, and the sentinel only *detects* — it wakes a human, it never auto-fixes, because an auto-fixing watchdog is just another unaudited writer.

## 9. Frozen nodes: name what the optimizer must never touch

Some rules exist precisely because an optimizing system would be tempted to weaken them: the kill switch, the kill criterion, the holdout window, the live-trading gate. Freeze them explicitly — the loop has no authority over them, full stop — and ask one design question on every cycle: *what would the optimizer weaken if it could?* That answer is the next thing to freeze. Corollary: never accept a proposal that softens a gate, raises a budget, or relaxes a "never." Those are the system asking to be allowed to fail more comfortably.

## 10. A verdict is an opinion; a row is evidence

An agent saying "done" or "refreshed" or "fixed" is a claim. A test that ran, a settlement row, a diff between the dashboard and the source of truth — those are evidence. Every loop in my fleet is being converted to the same standard: after it acts, something *other than the agent's own claim* must confirm the action landed. Catch errors at the cheapest layer that can catch them: a shell check before a model call, a model call before a human review.

## 11. Know what these systems cannot do

A loop with memory genuinely compounds at eliminating self-inflicted losses — execution slippage, taker fees in maker-paid markets, calibration drift. Those leaks are objectively verifiable, so a loop closes them one by one. What a loop cannot do is manufacture a price-beating signal from information every counterparty already has; memory refines the *use* of information, it cannot add information that isn't there. Point the machinery at the layer where improvement is verifiable, let it tell you where the floor is, and believe the floor when it's negative. The honest sales pitch for this whole architecture isn't "it prints money while you sleep." It's: **it tells you the truth about a strategy cheaply, in paper, with the leaks actually removed rather than assumed** — and the truth is sometimes that there's nothing there, delivered for $5 instead of six weeks.

---

*Companion to the [case study](README.md). The numbers cited (154 settled trades, −$0.023/contract, the 97% artifact, sixteen days / zero live trades) are from my own repos' ledgers and postmortems.*
