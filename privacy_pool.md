# Introduction

The [privacy pool](https://github.com/ameensol/privacy-pools) is an extension of [tornado cash](https://github.com/tornadocash/tornado-core) so that users can self-prove that the funds is **not** from laundered funds when **withdraw**.

This post only tries to clarify the extension part.

# Protocol

1. Users [`deposit`](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L37) funds into the privacy pool, with a `commitment` to a `secret` no one else knows.
    1. The `commitment` together with the deposit information is [hashed into a poseidon hash](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L49), resulting in `leaf`.
    2. The `leaf` is [inserted](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L50) into a merkle tree with fixed capacity `1<<20`.
        1. Every time a new `leaf` is inserted, the merkle root is calculated accordingly, and the contract [records](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/MerkleTree.sol#L81) up to `30` of the recent merkle roots.
2. When [`withdraw`](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L55) from the privacy pool, the user has to specify a recent merkle root([`root`](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L57) in code) and corresponding labeled merkle root([`subsetRoot`](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L58) in code, the construction process will be explained subsequently), and pass [this circuit](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/circuits/withdraw_from_subset.circom#L122).
    1. Essentially, this circuit verifies merkle proofs for two merkle trees:
        1. The one that's directly maintained in the contract when [`deposit`](https://github.com/ameensol/privacy-pools/blob/974cdccdf66c83c724b0b30074c7d895c361fa84/contracts/PrivacyPool.sol#L37).
        2. The labeled merkle tree constructed offchain by user, from exactly the same amount of leaves, and the leaf value is calculated this way:
            1. If the leaf of the deposit tree at the same index is allowed(e.g., not from a hacker address), the value is `keccak256("allowed") % p`
            2. Otherwise the value is `keccak256("blocked") % p`
    2. This circuit will only pass if the corresponding label for the target leaf is `keccak256("allowed") % p`.
    3. As the valid `subsetRoot`(there're many) can actually be calculated from the deposit transactions, a hacker whose deposit tx has been marked laundering will only pass the circuit by cheating, thus the `subsetRoot` must be different from valid ones.

# Soundness

If the user withdraws before the hacker is marked, false positive happens.