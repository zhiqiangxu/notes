# Polymarket Order Book Griefing Attack

A race condition in Polymarket's hybrid architecture allows an attacker to wipe market maker orders from the order book for less than $0.1 in gas. One tracked address profited $16,427 in under a day.

## Background: Hybrid Architecture

Polymarket uses off-chain matching + on-chain settlement:

```
User places order → Off-chain engine matches → Relayer submits tx on-chain → Settlement
```

There's a time gap (seconds to tens of seconds) between off-chain match confirmation and on-chain settlement. The attack exploits this gap.

## Attack Triggers

Two methods to cause on-chain settlement failure (both enable the same profit paths):

### Trigger V1: Balance Drain Race

```
T0: Attacker places BUY order via API
T1: Off-chain engine validates balance ✓, matches with market maker orders
T2: Attacker sends USDC transfer on-chain with HIGH gas (drains wallet)
T3: Drain tx confirms first (higher gas priority)
T4: Relayer submits matched trade → fails (insufficient balance)
T5: Off-chain system REMOVES ALL matched market maker orders from order book
```

The critical flaw is at T5: on-chain failure triggers off-chain order removal for **all parties involved**, not just the attacker. Innocent market maker orders are permanently cleared.

### Trigger V2: Ghost Fills

Improved version — no need to race the transfer:

```
T0: Attacker places orders across multiple markets via API
T1: Off-chain engine matches orders
T2: Attacker calls contract cancel function before on-chain settlement
T3: Matched orders invalidated, market maker orders cleared
```

This variant doubles as a **free option**: place orders in many markets, observe price movement, keep favorable fills, cancel unfavorable ones.

### V1 vs V2

| | V1: Balance Drain | V2: Ghost Fills |
|---|---|---|
| Mechanism | Transfer USDC on-chain before settlement | Call cancel function before settlement |
| Selectivity | All-or-nothing (all orders fail) | Selective (cancel only unfavorable fills) |
| Key advantage | Simpler to execute | **Free option** — bet first, decide later |

V2 is strictly more powerful. It enables a "trade-or-fade" strategy that should be impossible: place orders, observe price movement during the settlement delay, then cancel losers while keeping winners. Zero-cost optionality from the off-chain/on-chain time gap.

## Profit Paths

Either trigger (V1 or V2) enables these profit strategies:

### Profit Path 1: Monopoly Spread

```
Normal order book:          After attack:
  SELL $0.51 (Maker A)        SELL $0.65 (Attacker)
  SELL $0.52 (Maker B)
  --- spread $0.02 ---        --- spread $0.30 ---
  BUY  $0.49 (Maker A)
  BUY  $0.48 (Maker B)        BUY  $0.35 (Attacker)
```

1. Clear all competing orders (~$0.1 gas per cycle)
2. Place own orders with wide spread
3. Users forced to trade at attacker's price (no alternatives)
4. Repeat: clear → monopolize → profit → clear

Attacker earns the spread differential. Normal spread ~$0.02, attack spread $0.20-0.30.

### Profit Path 2: Hedging Bot Hunt

```
T0: Attacker places $10,000 BUY YES order
T1: Off-chain matches with market maker bot (bot "sells" 20,000 YES)
T2: Bot receives fill confirmation via API
T3: Bot hedges by buying 20,000 NO in related market (real on-chain trade)
T4: Attacker's original BUY YES order fails on-chain (rollback)

Result:
- Bot thinks: sold YES + bought NO = hedged ✓
- Reality: YES sale rolled back, NO purchase is real = naked NO position
```

The bot now holds unprotected NO tokens. The attacker can profit by buying the discounted NO tokens when the bot panic-sells to close its naked position.

**Note**: Path 2 is less reliable than Path 1. It requires predicting bot behavior, the bot may not unwind immediately, and profiting from the dislocation needs additional capital. On-chain data suggests the tracked attacker primarily used **Path 1 (monopoly spread)** as the main revenue source. Bot hunting is more collateral damage than primary strategy.

## Economics

| Metric | Value |
|--------|-------|
| Cost per attack cycle | < $0.1 (Polygon gas) |
| Cycle time | ~50 seconds |
| Theoretical throughput | ~72 attacks/hour |
| Tracked address profit | $16,427 |
| Time to profit | < 1 day |
| Markets traded | 7 |
| Max single profit | $4,415 |

The attacker used a dual-wallet rotation system (Cycle A Hub / Cycle B Hub) for automated high-frequency execution.

## Root Cause

The off-chain system treats settlement failure as a signal to **purge all related orders**, without distinguishing which party caused the failure.

```
Current logic:    on-chain fail → remove ALL matched orders
Correct logic:    on-chain fail → identify failing party → remove ONLY their orders
```

This is an architectural issue — the off-chain system cannot know real-time on-chain balances with zero latency. But at minimum, it should only penalize the party whose balance was insufficient, not the counterparties.

## Why Isn't It Fixed?

The most obvious fix — only remove the failing party's orders, not the counterparties' — is not technically hard. So why no action?

The likely answer: **Polymarket doesn't see it as a platform bug, but as a market maker performance issue.**

The attacker's edge is not speed, but **information asymmetry** — they know the exact moment orders will be cleared (because they triggered it), while bots must detect and react. But this window is small. A well-built bot that detects order removal and re-quotes in sub-second time can compress the attack window to near zero.

The $16,427 profit suggests the affected bots (Negrisk, ClawdBots, MoltBot) were simply **too slow to react** — polling instead of streaming, seconds of delay instead of milliseconds.

From Polymarket's perspective: fast market makers are unaffected, slow ones get punished — that's just competition. Fixing the root cause (order purge logic) is low priority when the "fix" is market makers upgrading their infrastructure.

## Community Response

- Polymarket: **silent** as of Feb 2026. Reports filed months prior, ignored.
- Community tool: [Nonce Guard](https://github.com/) — monitors on-chain cancellations, builds attacker blacklists, warns bots. Monitoring patch only, not a fix.
- Affected bots include Negrisk, ClawdBots, MoltBot.

## Implications

For a $9B prediction market platform, having its liquidity foundation attackable for $0.1 per cycle reveals a fundamental tension in the hybrid off-chain/on-chain architecture. Market makers lose willingness to quote, bots can't trust API fill confirmations, and order book depth deteriorates — a negative spiral.

The broader lesson: **any system where off-chain state assumes on-chain state stability is vulnerable to state desynchronization attacks**. The cost is just the gas to change on-chain state; the damage scales with the off-chain system's trust assumptions.

---

*Feb 2026. References: [PANews](https://www.chaincatcher.com/en/article/2248033), [DeFi Rate](https://defirate.com/news/polymarket-traders-allege-settlement-exploit-targeting-liquidity-providers/), community disclosures by BuBBliK, GoPlus security team monitoring.*
