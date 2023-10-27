# Introduction

[IZAR bridge](https://izar.xyz/) is a trial of decentralized bridge harnessing zkSNARK and a novel governance scheme. 

Most existing bridges are either very centralized, or claiming to be decentralized by using MPC for signatures. Every participator holds a keyshare, and each time new participators join the system, all existing keyshares have to be regenerated. The problem is that the old keyshares are still effective, there's no way to revoke the old keyshares, which may pose significant security risks to the system. They mitigate this problem by transfering assets to another fresh address every few months, but this is coordinated manually, a process far from decentralization.

IZAR bridge tries to replace the MPC part with zkSNARK. Furthermore, IZAR bridge also employs a POS+POA hybrid consensus mechanism to ensure both security and decentralization.

# Design Rationale

To be decentralized, IZAR bridge first employs the POS (Proof of Stake) mechanism to allow permissionless participation for external validators. However, to prevent malicious validators from gaining control of 2/3+ of the voting power at a low cost during the initial stage, the system also introduces the POA (Proof of Authority) mechanism. Through a deterministic algorithm, it ensures that 1/3+ of the voting power is allocated to POA, thus leaving POS with 2/3- of the voting power.

```mermaid
pie title Voting Power
    "POA" : 34
    "POS" : 66
```

In this way, even if all members of POS are malicious, as long as the members of POA act in good faith, the system remains secure. The inclusion of POA is permissioned, and only well-recognized teams within the industry will be allowed to join.

However, there is still a possibility of the system becoming inactive. To address this concern, the system has adjusted its multisig accepting criterion to be: either with a voting power of 2/3+, or voted by all POA members.


```mermaid
---
title: Multisig Accepting Criterion
---
stateDiagram-v2
    direction LR
    state if_state <<choice>>
    Event --> ValidatorSet
    ValidatorSet --> if_state
    if_state --> Yes : Either with a voting power of 2/3+, or voted by all POA members
    if_state --> No: Otherwise
```

This concludes the rationale for hybrid POS+POA consensus.

Another aspect to concern is the gas cost of onchain verification. 

Traditionally most projects have chosen to use MPC to reduce gas cost. However, due to the aforementioned drawbacks, IZAR bridge has opted to use zkSNARK to address this problem.

# Protocol

There're mainly 3 parts in this protocol:

    1. POS+POA: consensus and governance
    2. zkSNARK: multisig verification
    3. Tokenomics: incentives for the system 

## POS+POA

The basic model is that time is divided into multiple periods, and each period has a predetermined validator set responsible for confirming cross-chain events and the entry and exit of validators. Under normal circumstances, prior to the end of an period, the validator set for the next period is settled, thereby achieving a smooth transition. Each period lasts approximately 2 weeks and is called an `Epoch`. 

In order to avoid changes in the validator set while settlement is taking place, each `Epoch` is further divided into an `Active Period` at the beginning and a `Frozen Period` at the end. The `Frozen Period` lasts approximately 1 day, during which no validator entry or exit are allowed on the blockchain. Additionally, settlement is conducted for the interval [`the_Frozen_Period_of_the_previously_settled_Epoch`, `the_Active_Period_of_the_current_Epoch`], which is called the `settlement range`.


```mermaid
timeline
    title Under normal circumstances
    section Epoch
        Active Period : At this stage, the previous Epoch has been successfully settled, and validators can join or exit through an on-chain contract.
        Frozen Period : At this stage, validators cannot join or exit. The settlement of the current Epoch begins, covering the interval from the Frozen Period of the previously settled Epoch to the Active Period of the current Epoch.
```

In the event of exceptional circumstances where settlement is not successfully completed during the `Frozen Period`, the settlement process will be extended to the `Active Period` of the next `Epoch`. During this period, no validator additions or exits will be accepted until the settlement is finalized. In other words, there is always a principle followed that at any given moment, either the settlement phase or the mutable phase is active, but they do not occur in parallel.

```mermaid
timeline
    title Under exceptional circumstances
    section Epoch
        Active Period : At this stage, if the previous Epoch has not been successfully settled, validators cannot join or exit through an on-chain contract until the settlement of the previous Epoch is completed.
        Frozen Period : At this stage, validators cannot join or exit. The settlement of the current Epoch begins, covering the interval from the Frozen Period of the previously settled Epoch to the Active Period of the current Epoch.
```

Next, let's focus on describing the settlement process.

The input for the settlement process consists of the events of validator entry and exit, as well as cross-chain events that occur within the `settlement range` on the blockchain. 
The output of the settlement process is the balance and status of each validator.

In order to ensure the determinism of the settlement process, the rules are as follows:

1. Calculate the principal and status of POS validators based on on-chain events of validator entry and exit.
    1. Validator status includes: `NotJoint`, `PendingActivation`, `Activated`, `PendingExit`, and `Exitable`.
    2. Status transition:
        1. `NotJoint` → deposit → `PendingActivation` → settlement → `Activated`
        2. `Activated` → apply for exit → `PendingExit` → settlement → `Exitable` → withdraw → `NotJoint`
2. Rewards and penalties for validators are settled based on their voting behavior off-chain.
    1. Votes recorded within a time duration `T` (approximately 1 hour) after the cross-chain event finalization will be attributed to the current `Epoch`; otherwise, they will be attributed to subsequent Epochs.
    2. The settlement for a single `Epoch` begins at the Active Period plus the longest finalize time plus `T`. By this time, under normal circumstances, all cross-chain votes should be completed. Therefore, based on the determined voting set, definitive results can be settled. In exceptional cases, if the settlement is not successfully completed by the end of the `Frozen Period` (considering the result being updated on-chain), it will be extended to the `Active Period` of the next `Epoch`. If it still cannot be settled successfully, it will be settled together with the next `Epoch`.
3. Calculate the effective voting power of POS validators based on the balance(`balance = principal + accumulated_reward_or_penalty`) and status.
4. Calculate the addition or removal of POA validators.
    1. Addition needs approval of 2/3+ existing POA validators.
    2. Removal needs approval of 2/3+ existing POA validators, or initiated by oneself.
5. Calculate the voting power of each POA validator based on the effective voting power of POS validators. 

```mermaid
---
title: Status Transition
---
stateDiagram-v2
    direction LR
    
    NotJoint --> PendingActivation : deposit
    PendingActivation --> Activated : settlement
    Activated --> PendingExit : apply for exit
    PendingExit --> Exitable : settlement
    Exitable --> NotJoint : withdraw
```



The settlement process for rewards and penalties essentially involves two rounds of voting. The first round is a vote on on-chain events within the `settlement range`. The second round is a vote on the deterministic calculation results derived from the first round of voting.

```mermaid
---
title: Settlement process for rewards and penalties
---
flowchart LR
    subgraph Round1
    event(cross chain event)-->|after finalization|a2(collect votes)
    end
    subgraph Round2
    a2-->|after ActivePeriod+TheLongestFinalizeTime+T|b2(compute rewards and penalties)
    end
```    

To avoid having a large size when directly putting the results on-chain, only a merkle root is actually stored on the blockchain.

There is another underlying issue here, which is that the settlement results need to be updated first in the cross-chain contract (present on each supported blockchain) and then in the governance contract (present only on a specific blockchain). In extreme cases, it is possible that the update to the cross-chain contract occurs but misses the timing to update the governance contract. This situation may result in the cross-chain contract recognizing a newer set of validators while the governance contract recognizes an older set of validators.

However, this does not pose a problem because the older validators have not exited and will continue to vote and receive rewards. The votes of the new validators will also be included by the older validators and they will receive rewards as well. Eventually, both the cross-chain contract and the governance contract will be updated to a consistent result.

## zkSNARK

We use poseidon hash to commit each validator's public key and voting power:

```rust
validators_hash = PoseidonHash(validator_pk_1, power_1, validator_pk_2, power_2, ..., validator_pk_1024, power_1024)
```

If the number of validators is less than 1024, zero values are used instead.

The onchain part is:

```solidity
// verify whether 2/3+ voting power voted for msg_hash
function verifyMultiSigSingle(bytes32 validators_hash, bytes32 msg_hash, bytes proof) returns (bool)
// batch verification for 2 instances
function verifyMultiSigBatch2(bytes32 validators_hash, bytes32[2] msg_hashes, bytes proof) returns (bool)
// batch verification for 4 instances
function verifyMultiSigBatch4(bytes32 validators_hash, bytes32[4] msg_hashes, bytes proof) returns (bool)
// batch verification for 8 instances
function verifyMultiSigBatch8(bytes32 validators_hash, bytes32[8] msg_hashes, bytes proof) returns (bool)
```

The core parts of offchain circuit is `hash_to_curve` and `bls_signature_aggregate_verify`.

The circuit for the `hash_to_curve`, as described in Section 3: "Encoding Byte Strings to Elliptic Curves" of [IETF RFC 9380](https://datatracker.ietf.org/doc/rfc9380/), implements the conversion of a message to a point on the G2 group of the BN254 curve.

More specifically,

```
1. u = hash_to_field(msg, 2)
2. Q0 = map_to_curve(u[0])
3. Q1 = map_to_curve(u[1])
4. R = Q0 + Q1
5. P = clear_cofactor(R)
```

The design requirements for the `hash_to_field` function are that it should be an indistinguishable random oracle and should prevent information leakage through side-channel attacks, such as timing attacks. In our circuit, we choose to use `expand_message_xmd` and the SHA-2 hash function to ensure a constant running time and avoid timing attacks.

The `map_to_curve` function can use different methods depending on the curve. In our circuit, we choose to use the BN254 curve and employ the [Shallue-van de Woestijne method](https://datatracker.ietf.org/doc/html/draft-irtf-cfrg-hash-to-curve-12#name-shallue-van-de-woestijne-me).

`clear_cofactor` can be implemented by scalar multiplication with `h`. In some cases, the cofactor can be cleared by methods that are faster than the scalar multiplication with `h`. These methods are equivalent to multiplying by some scalar `h_eff` whose value is determined by the method and the curve.
In the BN254 curve condition, we chose to use the [Lattice-based method](https://cacr.uwaterloo.ca/techreports/2011/cacr2011-26.pdf) to clear the cofactor.

The `bls_signature_aggregate_verify` part of the circuit follows the requirements of section 3.3.4: " FastAggregateVerify" of the [IETF BLS Signature Standard](https://datatracker.ietf.org/doc/draft-irtf-cfrg-bls-signature/05/), which implements BLS signature aggregation and verification based on BN254 curves.

```
1. aggregate = pubkey_to_point(PK_1)
2. for i in 2, ..., n:
3.   next = pubkey_to_point(PK_i)
4.   aggregate = aggregate + next
5. PK = point_to_pubkey(aggregate)
6. return CoreVerify(PK, message, signature)
```

Based on the above two parts, we have implemented the following rules in the circuit:
1. Use `validators_hash`, `msg_hash` as public input.
2. Use each validator's public key `pk`, corresponding `power`, and signature submitted in this round as private inputs. For the validators who did not submit their signatures in this round, we just generate an empty signature to populate the circuit.
3. Verify the correctness of `pk` and `power` in the circuit: compute `PoseidonHash(validator_pk_1, power_1, validator_pk_2, power_2, ..., validator_pk_1024, power_1024)`, and constrain that the result is equal to public input `validators_hash`.
4. Set a flag for each validator in the circuit, when a certain flag is 1, it means that the validator has submitted its own signature on `msg_hash` in this round, and 0 is the opposite. Constrain the flags in the circuit to be either 0 or 1.
5. Multiply the power of all validators and the corresponding flags, and then add them up, constraining the result to be greater than 2/3 of the sum of the power of all validators.

After completing the above validation, we can be confident that each validator's public keys `pk` and corresponding `power` have been set correctly, since their poseidon hash is equal to the publicly passed `validators_hash`. Next, we verify that all validators whose flag is set to 1 have provided the correct BLS signature for `msg_hash`. We then verify that all validators whose flag is set to 1 have signed for `msg_hash`correctly.
1. Validate `M = hash_to_curve(msg_hash)` in the circuit.
2. Aggregate all signatures and pick out the correct `pk` and signature by flag: `bls_signature_aggregate_verify(pk_1 * flag_1 + pk_2 * flag_2 + .... , M, signature_1 * flag_1 + signature_2 * flag2...)`

After the above verification, Prover can generate the proof and submit it to the online contract.

## Tokenomics

(in progress)

# Conclusion

In summary, the core of this protocol consists of on-chain smart contracts and off-chain deterministic calculations and zkSNARK circuit. The smart contracts govern the protocol and leverage a `zkSNARK Verifier` for efficient verification. The off-chain deterministic calculations are based on on-chain events and off-chain voting. They compute deterministic incentives and the latest validator set for each deterministic interval and update the results to the smart contracts, thereby passing the baton to the next `Epoch`. The zkSNARK circuit essentially compresses all signatures into a single succinct proof.