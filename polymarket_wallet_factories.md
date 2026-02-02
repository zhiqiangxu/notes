# Introduction

Polymarket uses two wallet factory contracts to create user wallets: [ProxyWalletFactory](https://polygonscan.com/address/0xaB45c5A4B0c941a2F231C04C3f49182e1A254052) for Magic (email) users and [SafeProxyFactory](https://polygonscan.com/address/0xaacFeEa03eb1561C4e67d661e40682Bd20E3541b) for browser wallet (MetaMask, Coinbase, etc.) users. Source code: [proxy-factories](https://github.com/Polymarket/proxy-factories). Both enable gasless transactions and improved UX, but differ in complexity and features.

# Why Proxy Wallets?

Users don't trade directly from their EOA. Instead, a smart contract wallet holds their assets:

- **Gasless transactions**: Relayers pay gas, users don't need MATIC
- **Atomic multi-step operations**: Approve + trade in one transaction
- **Pre-configured approvals**: No approval popups for each trade

# Two Wallet Types

| Login Method | Wallet Type | SignatureType | Factory |
|--------------|-------------|---------------|---------|
| Email/Magic | Poly Proxy (EIP-1167) | `POLY_PROXY` | `proxyFactory` |
| MetaMask/Coinbase Wallet | Poly Safe (Gnosis Safe variant) | `POLY_GNOSIS_SAFE` | `safeFactory` |

**Why two types?**

- **Poly Proxy**: Minimal (~100 bytes), cheap to deploy, sufficient for Magic users who only need basic execution (see [Appendix](#appendix-magic-link-onboarding))
- **Poly Safe**: Full Gnosis Safe with multi-sig capability, modules, guards - more features for self-custody users

# Address Derivation (CREATE2)

Both factories use deterministic CREATE2 deployment ([PolyProxyLib](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/libraries/PolyProxyLib.sol#L8-L14), [PolySafeLib](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/libraries/PolySafeLib.sol#L13-L21)). Same EOA always gets the same wallet address:

```
walletAddress = keccak256(0xff, factory, keccak256(eoa), bytecodeHash)
```

This allows pre-computing wallet addresses before deployment.

# Signature Verification

In [CTFExchange](https://github.com/Polymarket/ctf-exchange), orders contain [`signer`](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/libraries/OrderStructs.sol#L14) (EOA that signs) and [`maker`](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/libraries/OrderStructs.sol#L12) (wallet holding assets). Verification flow ([Signatures.sol](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/mixins/Signatures.sol#L21-L25)):

```solidity
// validateOrderSignature passes order.maker as "associated"
isValidSignature(order.signer, order.maker, orderHash, order.signature, order.signatureType)

// isValidSignature routes to appropriate verifier, passing "associated" as proxyWallet/safeAddress
verifyPolyProxySignature(signer, associated, structHash, signature)
verifyPolySafeSignature(signer, associated, structHash, signature)
```

Each verifier checks that computed wallet address matches the passed address:

```solidity
getPolyProxyWalletAddress(signer) == proxyWallet  // Poly Proxy
getSafeAddress(signer) == safeAddress              // Poly Safe
```

# Flow

```
User signs order with EOA
        ↓
Order submitted to operator
        ↓
CTFExchange verifies:
  1. Signature valid?
  2. Computed wallet address == order.maker?
        ↓
Assets transferred from wallet (maker), not EOA
```

---

# Appendix: Magic Link Onboarding

For web2 users unfamiliar with crypto wallets:

1. **Email login**: User enters email on Polymarket → Magic sends login link → User clicks link → AWS Cognito verifies → Magic SDK issues a [DID token](https://magic.link/docs/authentication/features/decentralized-id) (session credential, default 7 days)
2. **EOA generation** (first login only): Magic generates EOA keypair in browser, encrypts private key with AWS KMS, stores encrypted shards
3. **Proxy deployment**: ProxyWalletFactory deploys a Poly Proxy wallet via [`makeWallet`](https://github.com/Polymarket/proxy-factories/blob/7137c021e6954d671095f77c94afc3d083d10a84/packages/proxy-factory/contracts/ProxyWallet/ProxyWalletFactory.sol#L88-L91)
4. **Order signing**: User clicks "Buy 10 YES @ $0.60" → frontend constructs order → Magic verifies DID token → retrieves encrypted key shards → recombines inside TEE → signs [order hash](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/mixins/Hashing.sol#L19-L39) inside TEE → returns signature (key never leaves TEE)
5. **Gasless trading**: Relayers submit transactions on behalf of the user, paying gas fees

The user never sees seed phrases, gas fees, or wallet addresses. They just log in with email and trade.

Note: Magic is semi-custodial — AWS KMS holds encrypted key shards. For full self-custody, users connect MetaMask/Coinbase Wallet and get a Poly Safe wallet instead.

## Example Code

([Magic SDK docs](https://docs.magic.link/embedded-wallets/authentication/login/magic-links), [Magic Ethereum docs](https://docs.magic.link/embedded-wallets/blockchains/ethereum/javascript), [Order struct](https://github.com/Polymarket/ctf-exchange/blob/1354de68752a065b7af8c0b004aeba2b16cf25a1/src/exchange/libraries/OrderStructs.sol#L8-L37))

```javascript
import { Magic } from 'magic-sdk';
import { ethers } from 'ethers';

const magic = new Magic('API_KEY');

// 1. Email login - returns DID token, also establishes authenticated session in SDK
const didToken = await magic.auth.loginWithMagicLink({ email: 'user@example.com' });

// 2. Send DID token to backend for verification (https://magic.link/posts/secure-auth-on-client-and-server-guide)
await fetch('/api/login', { headers: { Authorization: `Bearer ${didToken}` } });

// 3. Now magic.rpcProvider can sign - it uses the authenticated session
const provider = new ethers.BrowserProvider(magic.rpcProvider);
const signer = await provider.getSigner();
const eoaAddress = await signer.getAddress();

// 4. User clicks "Buy 10 YES @ $0.60" → frontend constructs order
const order = {
  salt: 123456,
  maker: proxyWalletAddress,  // Poly Proxy wallet
  signer: eoaAddress,         // EOA from Magic
  taker: '0x0000000000000000000000000000000000000000',
  tokenId: 12345,
  makerAmount: 6000000,       // 6 USDC
  takerAmount: 10000000,      // 10 outcome tokens
  expiration: 1735689600,
  nonce: 0,
  feeRateBps: 100,
  side: 0,                    // BUY
  signatureType: 1            // POLY_PROXY
};

// 5. Sign via Magic (TEE signing happens here, using the authenticated session)
const signature = await signer.signTypedData(domain, types, order);
```
