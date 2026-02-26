# Introduction

Polymarket's CLOB (Central Limit Order Book) is a hybrid system: off-chain order matching with on-chain settlement via [CTFExchange](https://github.com/Polymarket/ctf-exchange). SDKs: [py-clob-client](https://github.com/Polymarket/py-clob-client) (Python), [clob-client](https://github.com/Polymarket/clob-client) (TypeScript).

# Architecture

```
1. Sign EIP-712 message with wallet (L1)
        ↓
2. POST /auth/api-key → get api_key, secret, passphrase
        ↓
3. Sign order (EIP-712) + POST /order with HMAC headers (L2)
        ↓
4. CLOB engine matches against orderbook
        ↓
5. CTFExchange settles on-chain (Polygon)
```

Off-chain: order submission, matching, orderbook management. On-chain: settlement, token transfers.

# Authentication Levels

| Level | Auth Method | Use Case |
|-------|-------------|----------|
| L0 | None | Market data (orderbook, prices, tick sizes) |
| L1 | EIP-712 signature | Create/derive API credentials |
| L2 | HMAC-SHA256 + API key | Order management, trading |

## L1: EIP-712 Signature

Used to create API credentials:

```python
from py_clob_client.client import ClobClient

client = ClobClient(host="https://clob.polymarket.com", key=private_key, chain_id=137)
creds = client.create_or_derive_api_creds()  # Returns ApiCreds(api_key, api_secret, api_passphrase)
client.set_api_creds(creds)
```

Domain and types:

```javascript
domain = { name: "ClobAuthDomain", version: "1", chainId: 137 }
types = { ClobAuth: [
    { name: "address", type: "address" },
    { name: "timestamp", type: "string" },
    { name: "nonce", type: "uint256" },
    { name: "message", type: "string" }
]}
```

## L2: HMAC-SHA256 Signature

For authenticated requests:

```
message = timestamp + method + requestPath + body
signature = base64(hmac_sha256(api_secret, message))
```

Headers:

```
POLY_ADDRESS: 0x...              # Signer address (not funder)
POLY_SIGNATURE: <hmac_signature>
POLY_TIMESTAMP: 1234567890
POLY_API_KEY: <api_key>
POLY_PASSPHRASE: <passphrase>
```

# REST API Endpoints

## Market Data (L0 - No Auth)

| Endpoint | Description |
|----------|-------------|
| `GET /book?token_id=<id>` | Order book |
| `GET /price?token_id=<id>&side=BUY` | Best bid/ask |
| `GET /midpoint?token_id=<id>` | Midpoint price |
| `GET /spread?token_id=<id>` | Bid-ask spread |
| `GET /tick-size?token_id=<id>` | Minimum tick size |
| `GET /neg-risk?token_id=<id>` | Is negative-risk market |
| `GET /markets` | All markets |

## Credential Management (L1 - EIP-712 Required)

| Endpoint | Description |
|----------|-------------|
| `POST /auth/api-key` | Create API credentials |
| `GET /auth/api-key` | Derive existing credentials |
| `DELETE /auth/api-key` | Revoke credentials |

## Order Management (L2 - Auth Required)

| Endpoint | Description |
|----------|-------------|
| `POST /order` | Submit order |
| `POST /orders` | Batch submit |
| `DELETE /order` | Cancel by ID |
| `DELETE /orders` | Cancel multiple |
| `DELETE /cancel-all` | Cancel all |
| `GET /data/orders` | Open orders |
| `GET /data/trades` | Trade history |

## Session Management

| Endpoint | Description |
|----------|-------------|
| `POST /v1/heartbeats` | Keep session alive |

**Critical**: Orders cancelled if no heartbeat for 10 seconds during active trading.

# Order Types

| Type | Behavior |
|------|----------|
| `GTC` | Good-til-Cancel (default) |
| `GTD` | Good-til-Date (with expiration) |
| `FOK` | Fill-or-Kill (full fill or reject) |
| `FAK` | Fill-and-Kill (partial fill, cancel rest) |

Post-only orders (maker-only, no immediate match) supported with GTC/GTD.

# Order Structure

```python
order = {
    "salt": 123456,                    # Random nonce
    "maker": "0x...",                  # Funder address (holds assets)
    "signer": "0x...",                 # Signing address (may differ for proxy wallets)
    "taker": "0x0...0",                # Zero = public order
    "tokenId": "...",                  # Conditional token ID
    "makerAmount": "6000000",          # 6 USDC (6 decimals)
    "takerAmount": "10000000",         # 10 shares (6 decimals)
    "expiration": "0",                 # 0 = no expiration
    "nonce": "0",
    "feeRateBps": "100",               # 1% fee
    "side": "BUY",
    "signatureType": 1,                # 0=EOA, 1=POLY_PROXY, 2=POLY_GNOSIS_SAFE
    "signature": "0x..."
}
```

**Signer vs Maker**: For proxy wallets (Magic/email users), `signer` is the EOA that signs, `maker` is the proxy wallet holding assets. For EOA users, they're the same.

# Amount Calculations

Prices are probabilities in [0, 1]. Amounts use 6 decimals.

| Side | makerAmount | takerAmount |
|------|-------------|-------------|
| BUY | USDC spent | Shares received |
| SELL | Shares sold | USDC received |

Example: BUY 100 shares @ $0.60

```
makerAmount = 100 * 0.60 * 1e6 = 60000000  (60 USDC)
takerAmount = 100 * 1e6 = 100000000        (100 shares)
```

# Tick Size

Must fetch per-market before creating orders:

```python
tick_size = client.get_tick_size(token_id)  # e.g., "0.01"
```

Price constraints: `tick_size ≤ price ≤ 1 - tick_size`

Precision varies by tick size:

| Tick Size | Price Decimals | Size Decimals |
|-----------|----------------|---------------|
| 0.1 | 1 | 2 |
| 0.01 | 2 | 2 |
| 0.001 | 3 | 2 |
| 0.0001 | 4 | 2 |

# Python SDK Usage

```python
from py_clob_client.client import ClobClient
from py_clob_client.clob_types import OrderArgs

# Initialize
client = ClobClient(
    host="https://clob.polymarket.com",
    key=private_key,
    chain_id=137,
    signature_type=1,      # 0=EOA, 1=POLY_PROXY, 2=POLY_GNOSIS_SAFE
    funder=funder_address  # Proxy wallet address (if using proxy)
)

# Setup credentials
creds = client.create_or_derive_api_creds()
client.set_api_creds(creds)

# First-time setup (one-time, not needed for every trade)
client.update_balance_allowance()

# Market data
orderbook = client.get_order_book(token_id)
tick_size = client.get_tick_size(token_id)
neg_risk = client.get_neg_risk(token_id)

# Create and submit limit order
order_args = OrderArgs(
    token_id=token_id,
    price=0.45,
    size=100,
    side="BUY",
    fee_rate_bps=100
)
signed_order = client.create_order(order_args)
response = client.post_order(signed_order, orderType="GTC")

# Market order (FOK)
from py_clob_client.clob_types import MarketOrderArgs
market_order = MarketOrderArgs(
    token_id=token_id,
    amount=500,       # USDC for BUY, shares for SELL
    side="BUY"
)
signed = client.create_market_order(market_order)
response = client.post_order(signed, orderType="FOK")

# Order management
open_orders = client.get_orders()
client.cancel(order_id)
client.cancel_all()
```

# TypeScript SDK Usage

```typescript
import { ClobClient, OrderType, Side } from "@polymarket/clob-client";
import { Wallet } from "@ethersproject/wallet";

const signer = new Wallet(privateKey);
let client = new ClobClient("https://clob.polymarket.com", 137, signer);

// Setup credentials
const creds = await client.createOrDeriveApiKey();
client = new ClobClient(
    "https://clob.polymarket.com",
    137,
    signer,
    creds,
    1,              // SignatureType
    funderAddress
);

// First-time setup (one-time, not needed for every trade)
await client.updateBalanceAllowance();

// Market data
const orderbook = await client.getOrderBook(tokenID);
const tickSize = await client.getTickSize(tokenID);

// Create and submit order
const order = await client.createOrder(
    { tokenID, price: 0.45, size: 100, side: Side.BUY, feeRateBps: 100 },
    { tickSize, negRisk: false }
);
const response = await client.postOrder(order, OrderType.GTC);

// Heartbeat (required during active trading)
const heartbeatId = await client.postHeartbeat();
setInterval(() => client.postHeartbeat(heartbeatId), 5000);
```

# WebSocket Feeds

```
wss://ws-subscriptions-clob.polymarket.com/ws/market  # Public market data
wss://ws-subscriptions-clob.polymarket.com/ws/user    # Private user data
```

Subscribe to market feed:

```json
{
    "type": "market",
    "assets_ids": ["token_id_1", "token_id_2"]
}
```

Send `PING` every 50 seconds to keep connection alive.

# Token Allowances

Users must approve before trading:

1. **USDC** → Exchange, NegRisk Exchange, NegRisk Adapter
2. **Conditional Tokens** → Exchange, NegRisk Exchange, NegRisk Adapter

Check and update via API:

```python
allowances = client.get_balance_allowance()
client.update_balance_allowance()
```
