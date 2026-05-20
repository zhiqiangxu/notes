# Post-Quantum Signatures in leanSpec

> Snapshot pinned to commit [`d9d2e67`](https://github.com/leanEthereum/leanSpec/tree/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9). All code links resolve to that commit.

## Scheme: Hash-based Generalized XMSS

leanSpec signs attestations and blocks with a **Generalized XMSS** hash-based signature scheme, implemented in [`src/lean_spec/subspecs/xmss/`](https://github.com/leanEthereum/leanSpec/tree/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss). It is quantum-safe because security reduces to preimage resistance of a hash function (Poseidon1 over the KoalaBear field `P = 2^31 − 2^24 + 1`) rather than to lattice or discrete-log assumptions.

- Main class: [`GeneralizedXmssScheme`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/interface.py#L37) (interface.py:37–554)
- Field arithmetic: [`koalabear/field.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/koalabear/field.py)
- Production / test parameters: [`PROD_CONFIG`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/constants.py#L128) and [`TEST_CONFIG`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/constants.py#L147)
- Academic references: ePrint [2025/055](https://eprint.iacr.org/2025/055) ("Hash-Based Multi-Signatures for Post-Quantum Ethereum"), [2025/1332](https://eprint.iacr.org/2025/1332) ("LeanSig"); canonical Rust reference [`b-wagn/hash-sig`](https://github.com/b-wagn/hash-sig).

## Key generation — stateful, lifetime-based

[`GeneralizedXmssScheme.key_gen()`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/interface.py#L60) (interface.py:60–205):

1. Draws a master 32-byte `prf_key` and a public `parameter`.
2. Conceptually builds a Merkle tree over `2^LOG_LIFETIME` WOTS+ one-time public keys (production uses `LOG_LIFETIME = 32`).
3. Memory trick: keeps the **top tree** plus a sliding window of two **bottom trees**, achieving `O(√lifetime)` memory instead of `O(lifetime)` — ~2^16 hashes for prod instead of hundreds of GiB.

Data structures in [`containers.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/containers.py):

- [`PublicKey`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/containers.py#L27) = `(root, parameter)`
- [`SecretKey`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/containers.py#L67) carries `prf_key`, `parameter`, `activation_slot`, `num_active_slots`, `top_tree`, and the `left_bottom_tree` / `right_bottom_tree` window.

## Signing — one key per slot

[`GeneralizedXmssScheme.sign(sk, slot, message)`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/interface.py#L207) (interface.py:207–339):

1. **Slot validity check** — slot must be within the key's active range *and* covered by a currently-loaded bottom tree.
2. **Message → codeword** — hash `(message, parameter, epoch(slot), rho)` with Poseidon1 and rejection-sample randomness `rho` until the resulting base-`BASE` digits sum to `TARGET_SUM`. See [`message_hash.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/message_hash.py) and [`target_sum.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/target_sum.py).
3. **Winternitz chain release** — for each of `DIMENSION` chains, derive the chain start via [`PRF`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/prf.py) (SHAKE128-based) seeded by `(prf_key, slot, chain_index)`, then hash `x_i` steps. Each step is domain-separated via a [`ChainTweak`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/tweak_hash.py#L64).
4. **Merkle path** — emit the authentication path from this slot's leaf (the OTS public key) up to the lifetime root.
5. Output [`Signature`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/containers.py#L37) = `{path, rho, hashes}`.

Domain separation lives in [`tweak_hash.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/tweak_hash.py): prefixes `TWEAK_PREFIX_CHAIN = 0x00`, `TWEAK_PREFIX_TREE = 0x01`, `TWEAK_PREFIX_MESSAGE = 0x02` (see [constants.py:166–169](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/constants.py#L166)).

Signing is **deterministic** (same `(sk, slot, message)` → identical signature) and **stateful**: signing two different messages at the same slot reuses a WOTS+ key and breaks security.

## Verification

[`GeneralizedXmssScheme.verify(pk, slot, message, sig)`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/interface.py#L341) (interface.py:341–429):

1. Re-encode the message with `sig.rho` and check the codeword's digit sum equals `TARGET_SUM`.
2. For each chain `i`, take `sig.hashes[i]` (the chain endpoint at step `x_i`) and hash forward the remaining `BASE − 1 − x_i` steps. The resulting `DIMENSION` endpoints form the OTS public key.
3. Hash them into a Merkle leaf and walk `sig.path` upward, combining with siblings under [`TreeTweak(level, index)`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/tweak_hash.py#L50). Final node must equal `pk.root`. See [`verify_path()`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/subtree.py#L541) in subtree.py:541–617.

## Aggregation

Because XMSS is one-time-per-slot, signatures cannot be linearly combined the way BLS signatures can. leanSpec instead aggregates them with succinct **proofs** (Rust `lean-multisig-py` library) in [`aggregation.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/aggregation.py):

- **Type-1** ([`TypeOneMultiSignature`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/aggregation.py#L55), lines 55–210) — a single proof aggregating many validators' XMSS signatures on the **same** message at the same slot (e.g., one `AttestationData`). Stored as `{participants: AggregationBits, proof: ByteList512KiB}`. Aggregation calls `aggregate_type_1()` and verification calls `verify_type_1()`.
- **Type-2** ([`TypeTwoMultiSignature`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/aggregation.py#L212), lines 212–339) — merges multiple Type-1 proofs over **different** messages into one block-level proof, used to bundle every attestation plus the proposer's own signature into a single blob carried on the wire. Helpers `merge_many_type_1()`, `split_type_2_by_msg()`, and `verify_type_2_with_messages()`.
- Greedy participant selection for block production is in [`aggregation.py` lines 68–101](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/aggregation.py#L68).

## Consensus integration

- [`forks/lstar/containers/attestation/attestation.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/forks/lstar/containers/attestation/attestation.py) — `SignedAttestation` (raw XMSS, one validator) and `SignedAggregatedAttestation` (Type-1 multi-signature). The signed message is the SSZ root of `AttestationData`.
- [`forks/lstar/containers/block/block.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/forks/lstar/containers/block/block.py) — `SignedBlock` carries one Type-2 proof covering every attestation in the block plus the proposer signature over the block root.
- Consensus state-transition logic that consumes these proofs lives in [`forks/lstar/spec.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/forks/lstar/spec.py).

## Why XMSS instead of Dilithium / Falcon?

- Security rests on hash preimage resistance only — a well-understood, conservative assumption.
- Compact aggregate proofs (Type-1 / Type-2) over thousands of validators.
- Deterministic, reproducible signing.
- `O(√lifetime)` signer memory via the top/bottom tree split.

Trade-off: **stateful** (per-slot one-time keys, key state must advance via `advance_preparation()`) versus the stateless lattice schemes.

## File map

| File | Purpose |
| --- | --- |
| [`xmss/interface.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/interface.py) | `key_gen`, `sign`, `verify`, scheme instances |
| [`xmss/containers.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/containers.py) | `PublicKey`, `SecretKey`, `Signature`, `ValidatorKeyPair` |
| [`xmss/constants.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/constants.py) | `PROD_CONFIG`, `TEST_CONFIG`, domain prefixes |
| [`xmss/aggregation.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/aggregation.py) | Type-1 / Type-2 multi-signature wrappers |
| [`xmss/subtree.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/subtree.py) | Merkle subtree, `combined_path`, `verify_path` |
| [`xmss/tweak_hash.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/tweak_hash.py) | `TreeTweak`, `ChainTweak`, tweakable hasher |
| [`xmss/message_hash.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/message_hash.py) | Message-to-codeword encoding |
| [`xmss/target_sum.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/target_sum.py) | Target-sum codeword check |
| [`xmss/prf.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/xmss/prf.py) | SHAKE128-based PRF for chain starts and randomness |
| [`koalabear/field.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/subspecs/koalabear/field.py) | KoalaBear prime field |
| [`forks/lstar/containers/attestation/attestation.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/forks/lstar/containers/attestation/attestation.py) | Attestation containers |
| [`forks/lstar/containers/block/block.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/src/lean_spec/forks/lstar/containers/block/block.py) | Block containers |
| [`tests/consensus/lstar/ssz/test_xmss_containers.py`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/tests/consensus/lstar/ssz/test_xmss_containers.py) | SSZ serialization tests |
| [`lean_consensus.pdf`](https://github.com/leanEthereum/leanSpec/blob/d9d2e6737d6c6d416815f1ea9b1c4b21c9973cc9/lean_consensus.pdf) | Reference write-up |
