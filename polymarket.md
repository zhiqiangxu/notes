# Introduction

[Polymarket CTFExchange](https://github.com/Polymarket/ctf-exchange) is a hybrid-decentralized exchange for binary prediction markets: off-chain order matching, on-chain non-custodial settlement using [Gnosis Conditional Tokens](https://github.com/gnosis/conditional-tokens-contracts). It only supports trading between binary outcome tokens (ERC1155) and collateral (ERC20, e.g., USDC), not between different outcome tokens directly.

# Complementary Tokens

In a binary market (e.g., "Trump wins?"), two outcome tokens exist:
- **A** = YES token (redeemable for $1 if event happens, $0 otherwise)
- **A'** = NO token (redeemable for $1 if event doesn't happen, $0 otherwise)

The fundamental invariant: **1 YES + 1 NO = $1 USDC** (always).

This enables:
- **MINT**: $1 USDC → 1 YES + 1 NO (new tokens enter market)
- **MERGE**: 1 YES + 1 NO → $1 USDC (tokens exit market)

# Economically Equivalent Trades

Different trades can represent the same bet:

Using the "Trump wins?" market (YES = Trump wins, NO = Trump loses):

| Bet "Trump loses" | If Trump wins | If Trump loses |
|-------------------|---------------|----------------|
| SELL 100 YES @ $0.90 | -$10 | +$90 |
| BUY 100 NO @ $0.10 | -$10 | +$90 |

Same bet, same payoff → should pay same fee.

# Order Matching

Three [match types](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/mixins/Trading.sol#L275-L279):

| Type | Orders | Action |
|------|--------|--------|
| COMPLEMENTARY | BUY YES vs SELL YES (same token) | Direct swap |
| MINT | BUY YES vs BUY NO | Collect collateral → mint tokens → distribute |
| MERGE | SELL YES vs SELL NO | Collect tokens → merge → distribute collateral |

Two functions for trade execution:

1. [`fillOrder`](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/CTFExchange.sol#L61): Operator is counterparty (must hold tokens to trade against user), only supports COMPLEMENTARY
2. [`matchOrders`](https://github.com/Polymarket/ctf-exchange/blob/main/src/exchange/CTFExchange.sol#L82): Operator matches users against each other, supports all three match types

# Fee System

Fees go to the **operator** who executes trades.

## Symmetric Fee Formula

```
fee = baseRate * min(price, 1-price) * outcomeTokens
```

The `min(price, 1-price)` ensures complementary operations pay equal fees:

```
SELL 100 YES @ $0.90 → fee = 2% × min(0.90, 0.10) × 100 = $0.20
BUY 100 NO @ $0.10   → fee = 2% × min(0.10, 0.90) × 100 = $0.20
```

Without symmetry, economically equivalent operations (e.g., SELL YES @ $0.90 vs BUY NO @ $0.10) would have different fees, allowing traders to always pick the cheaper route for the same bet.
