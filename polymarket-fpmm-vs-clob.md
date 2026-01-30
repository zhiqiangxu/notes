# Introduction

Two models for prediction market trading:
- **FPMM** (Fixed Product Market Maker): [Gnosis AMM](https://github.com/gnosis/conditional-tokens-market-makers/blob/master/contracts/FixedProductMarketMaker.sol) - pool determines price
- **CLOB** (Central Limit Order Book): [Polymarket](https://github.com/Polymarket/ctf-exchange) - traders determine price

Both use [Gnosis Conditional Tokens](https://github.com/gnosis/conditional-tokens-contracts) underneath.

# Price Discovery

**FPMM**: Price from pool ratio via `YES × NO = k`

```
YES_price = NO_balance / (YES_balance + NO_balance)
```

**CLOB**: Price from order book spread

```
Mid_price = (Best_bid + Best_ask) / 2
```

| | FPMM | CLOB |
|---|------|------|
| Who sets price | Formula (passive) | Traders (active) |
| When price updates | After trade | Before trade (quote update) |

**Core difference**: FPMM reacts to trades. CLOB anticipates via quote updates.

# LP Risk

**FPMM**: Pool takes opposite side of every trade (forced exposure)

```
Traders buy YES → Pool accumulates NO → Event resolves YES → NO = $0
LP loss: near-total (pool can't fully empty due to constant product curve)
```

**CLOB**: Makers choose exposure, can cancel orders before adverse events.

# Capital Efficiency

**FPMM**: Liquidity spread across entire price curve ($0.01 to $0.99)

**CLOB**: Liquidity concentrated at current price

# Manipulation Resistance

| Attack | FPMM | CLOB |
|--------|------|------|
| Move price | Must spend capital | Can spoof (free) |
| Wash trading | Costs fees + slippage | Easier |

FPMM harder to manipulate — every price move costs real money.

# Summary

| | FPMM | CLOB |
|---|------|------|
| Price discovery | Reactive (slow) | Proactive (fast) |
| LP risk | High (forced) | Controlled |
| Capital efficiency | Low | High |
| Manipulation resistance | Higher | Lower |
| Decentralization | Full on-chain | Hybrid |

Polymarket migrated FPMM → CLOB in 2023 for better price discovery and capital efficiency, trading off decentralization.
