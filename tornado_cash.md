# Introduction

[`Tornado cash`](https://github.com/tornadocash/tornado-core) is the most famous mixer in the decentralized world. It has several pools, each with a different fixed amount of eth unit to deposit and withdraw from.(There're also erc20 pools but let's ignore that for now.) What it achieves in the end is that there's no way for outsiders monitoring the blockchain contracts to link the depositor and withdrawer from the same pool. So if there're `N` accounts that interacted with a specific `Tornado cash` pool, outsiders will know nothing more about any withdrawer except that it's one of the `N` accounts.



# Principle

`Tornado cash` is essentially a game of two secret numbers, let's name them `nullifier` and `secret`.  Only the actual user knows the exact value. `nullifier` serves as a unique identifier, the hash of which, named `nullifierHash`, is posted on the blockchain when withdraw, so that one can not withdraw from the pool twice for the same deposit. `secret` serves as the key for withdraw, nothing about it is actually posted on the blockchain. How is it actually implemented? By `zkp`.

1. Each deposit will [submit](https://github.com/tornadocash/tornado-core/blob/1ef6a263ac6a0e476d063fcb269a9df65a1bd56a/contracts/Tornado.sol#L55) its own `Commitment(secret || nullifier)`.
2. The `Tornado cash` contracts [maintain](https://github.com/tornadocash/tornado-core/blob/1ef6a263ac6a0e476d063fcb269a9df65a1bd56a/contracts/Tornado.sol#L58) a merkle tree of `Commitment(secret || nullifier)`, obtaining the merkle root as `root`.
3. When withdraw, the user will [submit](https://github.com/tornadocash/tornado-core/blob/1ef6a263ac6a0e476d063fcb269a9df65a1bd56a/contracts/Tornado.sol#L76) `Commitment(secret || nullifier)` and `nullifierHash`, and the zk proof that:
    1. The user knows both `nullifier` and `secret`
    2. `Commitment(secret || nullifier)` is actually a leaf of the above merkle tree.
4. In the end, only `Commitment(secret || nullifier)` and `nullifierHash` are publicly available on the blockchain, even if someone brute forces the `nullifier` from `nullifierHash`, nothing is known about `secret`. So only the user has the key to withdraw and that's it.