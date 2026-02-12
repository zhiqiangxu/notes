# NegRisk CTF Adapter

[neg-risk-ctf-adapter](https://github.com/Polymarket/neg-risk-ctf-adapter) extends Gnosis CTF for **mutually exclusive multi-outcome markets** (e.g., elections where exactly one candidate wins).

## Core Idea

```
NO(A) + NO(B) → YES(C) + USDC    (atomic conversion)
```

Standard CTF can't do this directly. The adapter enables it.

## Linked Markets

All outcomes share one collateral pool with enforced mutual exclusivity — only one can win.

Without linking, separate binary markets would be broken:
- Prices don't sum to 100% (Trump 50% + Biden 45% + RFK 25% = 120%?)
- No solvency guarantee if multiple resolve YES
- Fragmented liquidity, higher capital requirements

Linked markets enforce mutual exclusivity → consistent prices, shared liquidity, guaranteed payout.

## Architecture

```
NegRiskOperator (oracle/admin) → NegRiskAdapter (core logic) → CTF (ERC1155)
                                       ↓
                              WrappedCollateral + Vault
```

| Contract | Purpose |
|----------|---------|
| `NegRiskAdapter` | Split, merge, convert, redeem positions |
| `NegRiskOperator` | Oracle integration, dispute handling |
| `WrappedCollateral` | Collateral wrapper with synthetic minting |
| `Vault` | Fee collection |

## Conversion Economics

Converting K NO positions in N-outcome market yields:
- **(N - K)** complementary YES tokens
- **(K - 1)** USDC

Example: Convert NO-A + NO-B (K=2) in 3-outcome market → 1 YES-C + 1 USDC

## Synthetic Minting

`convertPositions` mints wcol **without depositing USDC** — safe because:
1. User's NO tokens (backed by real USDC) get burned
2. Byproduct NO tokens from splits also burned
3. Net claims on collateral remain balanced

## Key Invariants

1. **One Winner**: Exactly one outcome resolves YES (multiple YES = insolvency)
2. **NO Burning**: Converted NO tokens burned to prevent double-redemption
3. **1:1 Backing**: WrappedCollateral always fully backed

## ID System

```
MASK       = 0xFFFFFFFF...FFFFFF00  (last byte zeroed)
MarketId   = keccak256(oracle, feeBips, metadata) & MASK
QuestionId = MarketId + index (0-255)
```

The MASK clears the last 8 bits, reserving space for up to 256 questions per market. Questions share the same 31-byte prefix as their market.

## Lifecycle

```
prepareMarket → prepareQuestion → [trading] → reportPayouts → resolveQuestion → redeem
```

---

*Powers Polymarket's multi-outcome prediction markets.*
