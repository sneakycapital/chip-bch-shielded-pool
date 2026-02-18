# CHIP-2026-02-ShieldedBCH: Mandatory Shielded Pool via Zero-Knowledge Proofs

        Title: Mandatory Shielded Pool via Zero-Knowledge Proofs
        Type: Standards
        Layer: Consensus
        Maintainer: Sneaky Capital
        Status: Draft
        Initial Publication Date: 2026-02-18
        Latest Revision Date: 2026-02-19
        Version: 0.1.1-draft

<details>

<summary><strong>Table of Contents</strong></summary>

- [Summary](#summary)
  - [Terms](#terms)
- [Deployment](#deployment)
  - [Phase 1: Activation](#phase-1-activation)
  - [Phase 2: Migration Window](#phase-2-migration-window)
  - [Phase 3: Mandatory Shielding](#phase-3-mandatory-shielding)
  - [Phase 4: Transparent Pool Closure](#phase-4-transparent-pool-closure)
- [Motivation](#motivation)
  - [Transparent Blockchains Undermine Financial Privacy](#transparent-blockchains-undermine-financial-privacy)
  - [Optional Privacy Fails](#optional-privacy-fails)
  - [Existing BCH Privacy Tools Are Insufficient](#existing-bch-privacy-tools-are-insufficient)
  - [Mandatory Privacy Maximizes the Anonymity Set](#mandatory-privacy-maximizes-the-anonymity-set)
  - [Complementary to Zcash, Not a Replacement](#complementary-to-zcash-not-a-replacement)
- [Benefits](#benefits)
  - [Universal Anonymity Set](#universal-anonymity-set)
  - [True Fungibility](#true-fungibility)
  - [Censorship Resistance](#censorship-resistance)
  - [Commercial Privacy](#commercial-privacy)
  - [No Trusted Setup](#no-trusted-setup)
  - [Hardware Wallet Compatibility](#hardware-wallet-compatibility)
- [Technical Specification](#technical-specification)
  - [1. Proof System: Halo 2](#1-proof-system-halo-2)
  - [2. Elliptic Curve Cycle: Pallas and Vesta](#2-elliptic-curve-cycle-pallas-and-vesta)
  - [3. Shielded Note Structure](#3-shielded-note-structure)
  - [4. Note Commitment Scheme](#4-note-commitment-scheme)
  - [5. Note Commitment Tree](#5-note-commitment-tree)
  - [6. Nullifier Set](#6-nullifier-set)
  - [7. Action Model](#7-action-model)
  - [8. Transaction Format (TXv6)](#8-transaction-format-txv6)
  - [9. Key Hierarchy](#9-key-hierarchy)
  - [10. Value Balance and Fee Handling](#10-value-balance-and-fee-handling)
  - [11. Consensus Rule Changes](#11-consensus-rule-changes)
  - [12. Block Header Changes](#12-block-header-changes)
  - [13. Migration Mechanism](#13-migration-mechanism)
  - [14. Cryptographic Primitives](#14-cryptographic-primitives)
  - [15. Address Format](#15-address-format)
- [Rationale](#rationale)
  - [Why Halo 2 Over Groth16](#why-halo-2-over-groth16)
  - [Why Halo 2 Over zk-STARKs](#why-halo-2-over-zk-starks)
  - [Why Mandatory Over Optional](#why-mandatory-over-optional)
  - [Why Orchard-Style Actions](#why-orchard-style-actions)
  - [Why Flag-Day Activation](#why-flag-day-activation)
  - [Why a 6-Month Migration Window](#why-a-6-month-migration-window)
- [Performance and Scalability](#performance-and-scalability)
  - [Transaction Size Impact](#transaction-size-impact)
  - [Throughput Analysis](#throughput-analysis)
  - [Proof Generation Performance](#proof-generation-performance)
  - [Block Validation](#block-validation)
  - [State Growth](#state-growth)
  - [Scalability Roadmap](#scalability-roadmap)
- [Backwards Compatibility](#backwards-compatibility)
- [Risk Assessment](#risk-assessment)
  - [Unprecedented Migration](#unprecedented-migration)
  - [Regulatory Risk](#regulatory-risk)
  - [Ecosystem Coordination](#ecosystem-coordination)
  - [State Growth](#state-growth-1)
  - [Cryptographic Assumptions](#cryptographic-assumptions)
  - [Quantum Resistance](#quantum-resistance)
  - [ZK Circuit Implementation Bugs](#zk-circuit-implementation-bugs)
  - [Performance and Block Propagation](#performance-and-block-propagation)
  - [Tooling and Compliance Breakage](#tooling-and-compliance-breakage)
- [Prior Art and Alternatives](#prior-art-and-alternatives)
  - [Zcash (Sapling and Orchard)](#zcash-sapling-and-orchard)
  - [Monero (RingCT)](#monero-ringct)
  - [Iron Fish](#iron-fish)
  - [Penumbra](#penumbra)
  - [CashFusion (BCH)](#cashfusion-bch)
- [Stakeholder Responses and Statements](#stakeholder-responses-and-statements)
- [Test Vectors](#test-vectors)
- [Implementations](#implementations)
- [Feedback and Reviews](#feedback-and-reviews)
- [Acknowledgements](#acknowledgements)
- [Changelog](#changelog)
- [Copyright](#copyright)

</details>

## Summary

This proposal specifies a hard fork upgrade to the Bitcoin Cash network that introduces **mandatory zero-knowledge privacy** for all transactions. The upgrade replaces Bitcoin Cash's transparent UTXO model with a fully shielded pool based on the **Halo 2** proving system, using an architecture derived from Zcash's Orchard protocol.

After activation, all new transactions MUST be shielded. Existing transparent UTXOs are migrated into the shielded pool during a phased transition period. At the conclusion of the transition, Bitcoin Cash becomes a **privacy-only** cryptocurrency where transaction amounts, sender identities, and receiver identities are cryptographically hidden from all observers, including miners and full node operators.

This is a **consensus-layer** change requiring a coordinated hard fork. All node implementations, wallets, exchanges, and services interacting with the Bitcoin Cash network must upgrade.

### Terms

- **Shielded Pool**: The set of all unspent shielded notes, represented as a note commitment tree. The shielded pool replaces the transparent UTXO set.
- **Note**: The shielded equivalent of a UTXO. A note encodes a value (in satoshis) and ownership information, encrypted so that only the owner can identify and spend it.
- **Note Commitment**: A Pedersen commitment to a note's contents, appended to the global note commitment tree. Commitments reveal nothing about the note's value or owner.
- **Nullifier**: A unique, deterministic identifier derived from a note when it is spent. Publishing a nullifier marks the corresponding note as spent without revealing which note commitment it corresponds to.
- **Action**: A combined spend-and-output operation. Each action consumes one note (by revealing its nullifier) and creates one new note (by publishing its commitment). Transactions consist of one or more actions.
- **Value Commitment**: A Pedersen commitment to a note's value using a random blinding factor. Value commitments enable balance verification without revealing amounts.
- **Binding Signature**: A signature that proves the transaction's value commitments balance correctly (inputs equal outputs plus fees), without revealing any individual values.
- **Halo 2**: A zero-knowledge proof system based on the PLONK arithmetization with an Inner Product Argument, operating over the Pallas/Vesta elliptic curve cycle. It requires no trusted setup.
- **Turnstile**: An accounting mechanism that tracks the total value flowing between the transparent and shielded pools during migration, ensuring supply integrity.

## Deployment

This upgrade uses Bitcoin Cash's established **flag-day hard fork** activation mechanism, where new consensus rules activate when the Median Time Past (MTP) of the most recent 11 blocks exceeds a specified UNIX timestamp.

The upgrade proceeds in four phases:

### Phase 1: Activation

**Chipnet activation**: 6 months before mainnet, following standard BCH practice.

**Mainnet activation**: At the designated MTP timestamp, the following consensus changes take effect simultaneously:

- The shielded pool is created with an empty note commitment tree (depth 32) and an empty nullifier set.
- Transaction version 6 (TXv6) becomes valid, supporting shielded bundles.
- Both transparent (legacy) and shielded transactions are accepted.
- Coinbase outputs MUST be shielded from this point forward. Blocks containing transparent coinbase outputs are invalid.
- The `note_commitment_tree_root` and `shielded_value_pool_balance` fields become mandatory in block headers.

### Phase 2: Migration Window

**Duration**: 6 months after Phase 1 activation.

During the migration window:

- Users migrate their transparent UTXOs to the shielded pool by creating **shielding transactions**: transactions that spend one or more transparent inputs and produce one or more shielded outputs.
- Both transparent-to-transparent and shielded-to-shielded transactions remain valid.
- An **escalating fee multiplier** applies to transactions that produce transparent outputs:
  - Months 1-2: Transparent output fee multiplier = **2x** the standard fee rate.
  - Months 3-4: Transparent output fee multiplier = **5x** the standard fee rate.
  - Months 5-6: Transparent output fee multiplier = **20x** the standard fee rate.
- Wallet software SHOULD automatically migrate user funds to the shielded pool during this period, batching migrations over time to reduce timing correlation.
- The **turnstile mechanism** is active: the block header's `shielded_value_pool_balance` field tracks the cumulative net value transferred into the shielded pool, and at all times `total_transparent_supply + shielded_value_pool_balance = total_supply`.

### Phase 3: Mandatory Shielding

**Activation**: 6 months after Phase 1.

At Phase 3 activation:

- Transactions that create new transparent outputs are **invalid**. No new transparent UTXOs may be created.
- The ONLY valid use of transparent inputs is in **shielding transactions**, i.e. transactions that spend transparent UTXOs and produce exclusively shielded outputs.
- All change outputs MUST be shielded.
- Pure shielded-to-shielded transactions continue to operate normally.

### Phase 4: Transparent Pool Closure

**Activation**: 6 months after Phase 3 (12 months after Phase 1).

At Phase 4 activation:

- Only shielded-to-shielded transactions are valid for normal operation.
- **Exception**: Transparent inputs from UTXOs created before Phase 1 activation MAY still be spent into the shielded pool indefinitely. This accommodation ensures that users with lost-and-recovered keys, cold storage, or long-dormant wallets can always recover their funds. However, these legacy shielding transactions may ONLY produce shielded outputs.
- The transparent UTXO set is effectively frozen: it can only shrink (via shielding), never grow.
- Any BCH remaining in transparent UTXOs that are never claimed effectively reduces the circulating supply.

```
Timeline (T+0 = Mainnet Activation):

  T+0              T+2m             T+4m             T+6m             T+12m
  |                |                |                |                |
  |  Phase 1       |                |                |  Phase 3       |  Phase 4
  |  Activation    |                |                |  Mandatory     |  Pool
  |  (both valid)  |  2x fee mult   |  5x fee mult   |  Shielding     |  Closure
  |                |                |                |                |
  |<-------------- Phase 2: Migration Window ------->|                |
  |                (6 months)                        |<-- 6 months -->|

  T+0:   Hard fork activates. Shielded pool created. Coinbase must be shielded.
  T+6m:  No new transparent outputs. Transparent inputs valid only for shielding.
  T+12m: Shielded-to-shielded only. Legacy UTXOs may still be swept indefinitely.
```

## Motivation

### Transparent Blockchains Undermine Financial Privacy

Bitcoin Cash, like Bitcoin, operates on a fully transparent ledger. Every transaction, including the sender address, receiver address, and amount, is permanently recorded on a public blockchain visible to anyone in the world. This transparency creates severe privacy problems:

- **Surveillance**: Governments, corporations, and malicious actors can monitor all financial activity. Blockchain analytics firms routinely deanonymize users by linking addresses to real-world identities through exchange KYC data, IP address correlation, and transaction graph analysis.
- **Financial discrimination**: Merchants, employers, or service providers who can see a user's full transaction history may discriminate based on purchasing patterns, income levels, or associations.
- **Front-running and MEV**: Visible pending transactions enable front-running by miners or other observers who can see transaction details before confirmation.
- **Physical security risks**: Publicly visible large balances make holders targets for theft, extortion, or coercion.
- **Commercial espionage**: Businesses transacting on a transparent chain expose their supplier relationships, pricing structures, payment volumes, and cash flow to competitors.

Financial privacy is not merely a convenience. It is a requirement for a functioning monetary system. Traditional cash transactions are private by default. A digital cash system that fails to provide equivalent privacy falls short of physical cash in this respect.

### Optional Privacy Fails

The most prominent privacy-focused cryptocurrency, Zcash, has operated with **optional** shielded transactions since its launch in 2016. After nearly a decade of operation, the results demonstrate that optional privacy does not work:

- As of February 2026, only approximately **30% of the ZEC supply** (~5.03 million ZEC out of ~16.6 million) resides in shielded pools.
- The majority of Zcash transactions remain transparent, creating a small and easily analyzable anonymity set for shielded users.
- Users who do shield their funds stand out precisely because they opted in, making them targets for enhanced scrutiny (the "privacy is suspicious" problem).
- Exchange and wallet support for shielded transactions has been inconsistent, creating friction that discourages adoption.

The fundamental insight is that **privacy is a network effect**. A privacy system where only ~30% of users participate provides far less protection than one where 100% participate. When privacy is optional, the default transparent behavior becomes the norm, and the small minority who opt in receive diminished protection.

### Existing BCH Privacy Tools Are Insufficient

Bitcoin Cash currently offers two main privacy tools:

- **CashShuffle**: A coin-mixing protocol based on CoinJoin. It requires active participation from multiple users simultaneously, provides limited anonymity sets (typically 5-10 participants per round), and is vulnerable to timing analysis and Sybil attacks.
- **CashFusion**: An improvement on CashShuffle that uses larger, more complex transactions. While CashFusion provides better anonymity than CashShuffle, it still requires opt-in participation, has limited adoption, and creates a distinct on-chain fingerprint that identifies fusion participants.

Both tools operate at the application layer, attempting to add privacy on top of a transparent protocol. This approach has hard limits because:

1. The underlying transparent UTXO model leaks information that mixing cannot fully eliminate.
2. Participation is opt-in, creating small anonymity sets.
3. The mixing transactions themselves are identifiable, marking participants as "privacy seekers."
4. Neither tool hides transaction amounts.

### Mandatory Privacy Maximizes the Anonymity Set

By making all transactions shielded at the protocol level, this proposal achieves several critical properties:

- **The anonymity set is the entire network**. Every BCH user contributes to every other user's privacy, regardless of technical sophistication.
- **No opt-in stigma**. When all transactions are private, using privacy features is not suspicious, it is simply how the protocol works.
- **Defense in depth**. Protocol-level privacy cannot be circumvented by application-layer surveillance or chain analysis. There is no transparent transaction graph to analyze.
- **Alignment with BCH's mission**. Bitcoin Cash positions itself as "peer-to-peer electronic cash." Cash transactions in the physical world are private by default. This proposal brings the same property to digital cash.

### Complementary to Zcash, Not a Replacement

This proposal is **not** an attempt to replace Zcash. While this CHIP borrows heavily from Zcash's cryptographic research, and acknowledges the Electric Coin Company's years of work in this area, Bitcoin Cash and Zcash serve different roles and can thrive side by side.

Zcash is purpose-built as a privacy-first cryptocurrency with a research-driven development culture, an opt-in shielding model, and a developer fund structure. It occupies a unique and valuable position in the cryptocurrency ecosystem as the origin of production zk-SNARK deployments and continues to push the frontier of applied cryptography.

Bitcoin Cash occupies a different niche. It was born from the original Bitcoin scaling debate with a singular mission: **peer-to-peer electronic cash for the world**. BCH has spent nearly a decade building merchant adoption infrastructure, maintaining low fees, scaling on-chain with larger blocks, and preserving the proof-of-work mining model that Satoshi Nakamoto described in the Bitcoin whitepaper. What BCH lacks, and what this proposal addresses, is the financial privacy that physical cash provides by default.

The broader context matters. Bitcoin (BTC) has drifted far from its original purpose as peer-to-peer cash. It has been captured by institutional interests, rebranded as "digital gold," wrapped in ETFs, and increasingly subject to the same surveillance and control mechanisms that cryptocurrency was invented to circumvent. The Lightning Network has not delivered on the promise of cheap, private payments. The cypherpunk vision of censorship-resistant money for ordinary people has been largely abandoned in favor of Wall Street speculation.

Bitcoin Cash kept that original vision alive, but without privacy, it remains incomplete. A transparent ledger cannot serve as censorship-resistant money when every transaction is visible to the very institutions users seek independence from. This proposal completes what Bitcoin was always meant to be: **private, peer-to-peer electronic cash that works for everyone, not just institutions**.

BCH and Zcash can coexist and reinforce each other. More privacy-focused chains strengthen the broader ecosystem by:

- Increasing the total population using privacy-preserving cryptocurrency, which benefits all privacy chains through network effects and normalization.
- Providing users with choices that match their specific needs (BCH's emphasis on merchant payments and low fees vs. Zcash's emphasis on cutting-edge cryptographic research).
- Diversifying the implementation risk. If a critical bug is found in one chain's privacy implementation, the other remains unaffected.
- Demonstrating to regulators and the public that financial privacy is a mainstream value, not a fringe concern.

The goal is not one privacy chain to rule them all. The goal is a world where private digital cash is the norm, not the exception.

## Benefits

### Universal Anonymity Set

Every transaction on the network contributes to the anonymity set. Unlike optional privacy systems where shielded users form a small, identifiable subset, mandatory shielding means that an observer cannot distinguish any transaction from any other. The anonymity set grows with every block, and every user benefits from every other user's activity.

### True Fungibility

With transparent transactions, individual coins can be traced and "tainted", i.e. associated with specific activities, addresses, or entities. This creates a two-tier system where some coins are considered "clean" and others "dirty," undermining the fundamental property that every unit of currency should be interchangeable with every other.

Mandatory shielding eliminates coin tracing entirely. Once a UTXO is migrated into the shielded pool, its history is cryptographically severed. All shielded BCH is indistinguishable, restoring true fungibility.

### Censorship Resistance

Transparent blockchains enable selective censorship: miners or validators who can see transaction details can choose to exclude specific senders, receivers, or transaction types. With shielded transactions, miners verify validity through zero-knowledge proofs without learning the identities of participants or the amounts transferred. Selective censorship based on transaction content becomes infeasible.

### Commercial Privacy

Businesses require financial privacy to operate competitively. Transparent blockchains expose:

- Supplier and vendor relationships
- Pricing and payment terms
- Revenue and cash flow patterns
- Customer identities and purchasing patterns
- Payroll information

Mandatory shielding protects all of this information, making Bitcoin Cash viable for commercial use without the privacy compromises inherent in transparent ledgers.

### No Trusted Setup

The Halo 2 proving system eliminates the trusted setup ceremony required by earlier zero-knowledge proof systems (e.g. Groth16, used in Zcash's Sapling protocol). A trusted setup requires a multi-party computation ceremony where participants generate cryptographic parameters and then destroy their individual inputs ("toxic waste"). If any participant's toxic waste is compromised or not properly destroyed, an attacker could forge proofs and create counterfeit coins undetectably.

By using Halo 2's Inner Product Argument over the Pallas/Vesta curve cycle, this proposal requires no trusted setup whatsoever. The proving system's security relies solely on standard cryptographic assumptions (the discrete logarithm problem on elliptic curves), with no ceremony-dependent trust. This aligns with Bitcoin Cash's core values of trustlessness and verifiability.

### Hardware Wallet Compatibility

The key hierarchy separates the **proof authorizing key** from the **spending key**. This enables a workflow where:

1. A host device (computer or smartphone) generates the zero-knowledge proof using the proof authorizing key.
2. The hardware wallet holds only the spending key and signs the spend authorization.
3. The hardware wallet never needs to perform expensive proof generation.

This architecture, proven in Zcash's Sapling and Orchard protocols, ensures that hardware wallet support remains practical despite the computational demands of zero-knowledge proof generation.

## Technical Specification

### 1. Proof System: Halo 2

This proposal adopts the **Halo 2** proving system for all zero-knowledge proofs. Halo 2 combines:

- **PLONK arithmetization**: An efficient representation of the computation to be proven, using UltraPLONK with custom gates and lookup tables. UltraPLONK allows the circuit designer to define specialized gates for common operations (elliptic curve addition, hash function rounds, range checks), reducing the number of constraints required.
- **Inner Product Argument (IPA)**: The polynomial commitment scheme used to construct the proof. Unlike KZG commitments (used in original PLONK), the IPA requires no structured reference string and therefore no trusted setup. The IPA operates over Pedersen commitments to polynomial coefficient vectors.
- **Recursive proof composition**: Halo 2's IPA can be efficiently verified inside another Halo 2 circuit via the Pallas/Vesta curve cycle, enabling proof aggregation and future scalability improvements.

**Circuit specification**: The Halo 2 circuit for this proposal verifies the following statements for each action in a transaction:

1. **Note existence**: The spent note's commitment exists as a leaf in the note commitment tree at the anchor (tree root) specified by the transaction. This is proven via a Merkle path authentication.
2. **Nullifier correctness**: The revealed nullifier is correctly derived from the spent note using the spender's nullifier deriving key.
3. **Spend authorization**: The spender possesses the spending key corresponding to the spent note's owner.
4. **Value commitment integrity**: The value commitment for the action correctly encodes the difference between the spent note's value and the created note's value.
5. **Note well-formedness**: The created note's commitment is correctly constructed from its component fields.

A single aggregated Halo 2 proof covers ALL actions within a transaction bundle, amortizing proof overhead.

### 2. Elliptic Curve Cycle: Pallas and Vesta

Halo 2 operates over the **Pallas/Vesta** elliptic curve 2-cycle, where both curves share the equation `y^2 = x^3 + 5` but operate over different prime fields:

| Property | Pallas | Vesta |
|----------|--------|-------|
| Equation | y^2 = x^3 + 5 | y^2 = x^3 + 5 |
| Base field size (p) | 2^254 + 45560315531506369815346746415080538113 | 2^254 + 45560315531419706090280762371685220353 |
| Scalar field size (q) | 2^254 + 45560315531419706090280762371685220353 | 2^254 + 45560315531506369815346746415080538113 |
| Relationship | Scalar field = Vesta's base field | Scalar field = Pallas's base field |

The critical property of this 2-cycle is that the scalar field of each curve equals the base field of the other. This enables **native recursion**: a proof generated over Pallas can be efficiently verified inside a circuit over Vesta (and vice versa), without expensive non-native field arithmetic.

All note commitments, value commitments, and the primary proof system operate over the **Pallas** curve. Recursive verification (for future proof aggregation) uses the **Vesta** curve.

### 3. Shielded Note Structure

A shielded note is a tuple of six fields:

```
note = (d, pk_d, v, rho, psi, rcm)
```

| Field | Size | Description |
|-------|------|-------------|
| `d` | 11 bytes | **Diversifier**. An arbitrary 11-byte value that, when valid, maps to a unique point on the Pallas curve via the `DiversifyHash` function. Each diversifier produces a distinct payment address from a single spending key, enabling users to give unique addresses to each counterparty without revealing common ownership. |
| `pk_d` | 32 bytes | **Diversified transmission key**. The public key derived from the diversifier `d` and the recipient's incoming viewing key `ivk`: `pk_d = DiversifyHash(d) * ivk`. This key is used by the sender to encrypt the note for the recipient. |
| `v` | 8 bytes | **Value**. The amount in satoshis (an unsigned 64-bit integer). Valid range: 0 to 2,100,000,000,000,000 (21 million BCH in satoshis). |
| `rho` | 32 bytes | **Nullifier randomness input**. For a note created by spending an existing note, `rho` is set to the nullifier of the spent note. For notes created without a corresponding spend (e.g., coinbase, shielding transactions), `rho` is derived from a random seed. This construction, borrowed from Zcash's Orchard, eliminates the need to know the note's position in the commitment tree when computing its nullifier. |
| `psi` | 32 bytes | **Sender randomness**. Derived from `rho` and a sender-selected random seed `rseed` via a PRF: `psi = PRF_psi(rho, rseed)`. Used as additional randomness in the nullifier derivation to prevent related-key attacks. |
| `rcm` | 32 bytes | **Commitment randomness**. Derived from `rseed`: `rcm = PRF_rcm(rho, rseed)`. The blinding factor used in the Pedersen commitment to this note. |

The sender selects a random 32-byte `rseed` for each note and derives both `psi` and `rcm` from it. Only `rseed` needs to be transmitted to the recipient (encrypted); the recipient recomputes `psi` and `rcm` independently.

### 4. Note Commitment Scheme

The commitment to a note is computed as:

```
cm = PedersenCommit_rcm(d || pk_d || v || rho || psi)
```

where `PedersenCommit_rcm` is a Pedersen commitment over the Pallas curve with blinding factor `rcm`. The Pedersen commitment uses independent generators for each field component:

```
cm = [rcm] * R + [d] * G_d + [pk_d] * G_pk + [v] * G_v + [rho] * G_rho + [psi] * G_psi
```

where `R, G_d, G_pk, G_v, G_rho, G_psi` are independently generated Pallas curve points (nothing-up-my-sleeve points derived via hash-to-curve from fixed domain separation strings).

The **homomorphic property** of Pedersen commitments is critical for value balance verification. A separate **value commitment** is computed for each action:

```
cv = ValueCommit(rcv, v) = [v] * G_v + [rcv] * R_v
```

where `rcv` is a random blinding factor and `G_v, R_v` are fixed generators. The transaction's value balance is verified by checking that the sum of all input value commitments minus all output value commitments equals the commitment to the transaction fee, without revealing any individual value.

### 5. Note Commitment Tree

All note commitments are appended to a global **incremental Merkle tree** with the following properties:

| Property | Value |
|----------|-------|
| Depth | 32 |
| Capacity | 2^32 = 4,294,967,296 notes |
| Hash function | Sinsemilla (over Pallas curve) |
| Leaf format | Note commitment `cm` (32 bytes) |
| Empty leaf | A fixed "uncommitted" value (the Pallas curve identity point) |
| Append-only | Notes are only appended; never removed or modified |

**Sinsemilla** is an algebraic hash function based on incomplete addition on the Pallas curve. It is efficient to evaluate inside a Halo 2 circuit (requiring far fewer constraints than traditional hash functions like SHA-256), which is essential for proving Merkle path authentication within the zero-knowledge proof.

The tree is **incremental**: nodes maintain only the frontier of the tree (the rightmost path), not the full tree. This allows efficient appending of new commitments without storing the entire tree in memory.

Each transaction references an **anchor**, the root of the note commitment tree at some recent block. The anchor must correspond to the tree state at a block within the most recent 100 blocks. This window:

- Ensures that spent notes actually exist in the tree.
- Prevents deep reorganization attacks on the shielded state.
- Allows transactions in the mempool to remain valid across minor chain tip changes.

### 6. Nullifier Set

Each note has a unique **nullifier** that is revealed when the note is spent. The nullifier is computed as:

```
nf = PoseidonHash(nk, rho, psi, cm)
```

where:
- `nk` is the **nullifier deriving key**, a component of the full viewing key.
- `rho`, `psi` are fields from the note being spent.
- `cm` is the note commitment.

**Consensus enforcement**: Full nodes maintain a set of all revealed nullifiers. A transaction is **invalid** if any of its nullifiers already exist in the set. This prevents double-spending. Nullifiers are added to the set atomically when the containing block is accepted into the chain.

**Privacy property**: The nullifier reveals nothing about which note commitment in the tree it corresponds to. An observer who sees a nullifier cannot determine:
- Which note commitment was spent.
- The value of the spent note.
- The identity of the spender.
- When the note was originally created.

This unlinkability between nullifiers and commitments is the core privacy mechanism of the shielded pool.

### 7. Action Model

Following Zcash's Orchard design, this proposal uses an **action model** where each action combines exactly one spend and one output:

```
Action = (Spend, Output)
```

A transaction contains one or more actions. Each action:

| Component | Spend half | Output half |
|-----------|-----------|-------------|
| Published | Nullifier `nf`, randomized verification key `rk`, spend authorization signature `spendAuthSig` | Note commitment `cm`, ephemeral key `epk`, encrypted note ciphertext `C_enc`, outgoing ciphertext `C_out` |
| Shared | Value commitment `cv` (net value of this action) |
| Proven inside ZK circuit | Note exists in tree at anchor, nullifier correctly derived, spend authorized, value commitment consistent | Note commitment correctly constructed, value commitment consistent |

**Arity hiding**: Because every action is exactly one spend and one output, an observer cannot distinguish between:
- A 1-input, 2-output payment (1 real spend + 1 real output + 1 dummy spend + 1 real change output = 2 actions)
- A 2-input, 1-output consolidation (2 real spends + 1 real output + 1 dummy output = 2 actions)
- A 2-input, 2-output transfer (2 real spends + 2 real outputs = 2 actions)

**Dummy actions**: When a transaction needs more spends than outputs (or vice versa), dummy actions are used. A dummy spend uses a zero-value note with a random nullifier that is proven to be correctly formed within the circuit. A dummy output creates a zero-value note commitment. Dummy actions are indistinguishable from real actions to any observer.

### 8. Transaction Format (TXv6)

A new transaction version is introduced for shielded transactions:

```
TXv6 {
    // Header
    version:              uint32    // = 6
    lock_time:            uint32    // Standard lock time semantics

    // Transparent components (migration only, empty after Phase 4)
    tx_in_count:          varint
    tx_in:                TxIn[]    // Standard BCH transparent inputs
    tx_out_count:         varint    // MUST be 0 after Phase 3 activation
    tx_out:               TxOut[]   // Standard BCH transparent outputs (migration only)

    // Shielded bundle
    shielded_bundle:      ShieldedBundle

    // Binding signature
    binding_sig:          64 bytes  // RedPallas signature proving value balance
}

ShieldedBundle {
    // Actions
    n_actions:            varint
    actions:              ActionDescription[]

    // Flags
    flags:                uint8     // Bit 0: spends_enabled, Bit 1: outputs_enabled

    // Value balance (net transparent <-> shielded flow)
    value_balance:        int64     // Positive = value flows from shielded to transparent
                                    // Negative = value flows from transparent to shielded

    // Anchor (note commitment tree root)
    anchor:               32 bytes  // Must match tree root at a block within last 100 blocks

    // Aggregated proof
    proof_size:           varint
    proof:                bytes     // Halo 2 proof (~1440 bytes for typical transactions)

    // Spend authorization signatures
    spend_auth_sigs:      Signature[]  // One per action where spends_enabled is set
}

ActionDescription {
    // Spend half
    nullifier:            32 bytes  // Nullifier of the spent note
    rk:                   32 bytes  // Randomized spend verification key

    // Output half
    note_commitment:      32 bytes  // Commitment to the new note
    ephemeral_key:        32 bytes  // For Diffie-Hellman key agreement with recipient
    encrypted_note:       580 bytes // Note plaintext encrypted to recipient's pk_d
    out_ciphertext:       80 bytes  // Encrypted data for sender recovery via ovk

    // Shared
    cv:                   32 bytes  // Value commitment for this action
}
```

**Transaction validity rules for TXv6**:

1. If `n_actions > 0`, the shielded bundle MUST contain a valid Halo 2 proof.
2. The `anchor` MUST correspond to a note commitment tree root at a block height within `[current_height - 100, current_height]`.
3. No nullifier in the transaction may already exist in the nullifier set.
4. No two nullifiers within the same transaction may be equal.
5. The binding signature MUST verify against the balance commitment derived from all value commitments and the `value_balance` field.
6. After Phase 3: `tx_out_count` MUST be 0.
7. After Phase 4: `tx_in_count` MUST be 0, unless spending a UTXO created before Phase 1 activation.
8. `value_balance` MUST be consistent: `sum(transparent_inputs) - sum(transparent_outputs) + value_balance = transaction_fee`, where the fee is committed via the binding signature.

### 9. Key Hierarchy

The key hierarchy enables fine-grained access control, separating spending authority from viewing authority and proof generation:

```
Spending Key (sk)                          [256 bits, secret]
│
├─► Ask = SpendingKeyToAsk(sk)             [Spend authorizing key, secret]
│   └─► ak = AskToAk(Ask)                 [Spend validating key, public]
│
├─► nk = SpendingKeyToNk(sk)              [Nullifier deriving key, can detect spends]
│
├─► rivk = SpendingKeyToRivk(sk)          [Commit ivk randomness, secret]
│
└─► Full Viewing Key (fvk) = (ak, nk, rivk)
    │
    ├─► Incoming Viewing Key (ivk)         [Detect and decrypt incoming payments]
    │   └─► For each diversifier d:
    │       └─► pk_d = DiversifyHash(d) * ivk   [Diversified transmission key]
    │       └─► Payment Address = (d, pk_d)      [Give to payers]
    │
    ├─► Outgoing Viewing Key (ovk)         [Decrypt outgoing payment details]
    │
    └─► Proof Authorizing Key              [Generate ZK proofs without spending authority]
```

**Key capabilities**:

| Key | Can spend | Can see incoming | Can see outgoing | Can generate proofs | Can derive addresses |
|-----|-----------|-----------------|------------------|--------------------|--------------------|
| Spending Key | Yes | Yes | Yes | Yes | Yes |
| Full Viewing Key | No | Yes | Yes | Yes | Yes |
| Incoming Viewing Key | No | Yes | No | No | Yes |
| Outgoing Viewing Key | No | No | Yes | No | No |
| Proof Authorizing Key | No | No | No | Yes | No |

**Use cases**:

- **Full node wallet**: Uses the spending key for complete control.
- **View-only wallet**: Uses the incoming viewing key to monitor balances for accounting or compliance without spending authority.
- **Hardware wallet**: The host device holds the proof authorizing key and generates proofs; the hardware wallet holds the spending key and only signs the spend authorization (~50ms operation).
- **Auditor access**: A user can share their incoming viewing key with an auditor to prove their transaction history without granting spending authority.

### 10. Value Balance and Fee Handling

Transaction balance is verified through the **binding signature** mechanism:

For each action `i` with value `v_i` and blinding factor `rcv_i`:

```
cv_i = [v_i] * G_v + [rcv_i] * R_v
```

The transaction's **balance commitment** is:

```
bvk = sum(cv_input) - sum(cv_output) - [fee] * G_v
```

The binding signature is a RedPallas signature under the key `bvk`. If the signature verifies, the verifier is assured that the values balance (inputs = outputs + fee) without learning any individual value.

**Fee handling**: The transaction fee is the only value that is publicly visible. It is encoded as a plaintext `int64` field and committed to via the balance equation. This is consistent with Zcash's approach and is necessary for miners to prioritize transactions. The fee amount does not reveal information about the total transaction value.

**Turnstile accounting** (during migration):

```
shielded_pool_balance += sum(transparent_inputs) - sum(transparent_outputs) - fee
```

At every block: `total_mined_supply = transparent_utxo_total + shielded_pool_balance`. This invariant is publicly verifiable and detects any inflation bug in the shielded pool implementation.

### 11. Consensus Rule Changes

The following new consensus rules are introduced:

**11.1 Note Commitment Tree**

- Every full node MUST maintain the incremental Merkle tree of note commitments.
- New commitments are appended to the tree in the order they appear in each block's transactions, in transaction order.
- The tree root after processing all transactions in a block MUST equal the `note_commitment_tree_root` in the block header.
- The tree MUST NOT exceed depth 32 (capacity 2^32 leaves).

**11.2 Nullifier Set**

- Every full node MUST maintain the complete set of revealed nullifiers.
- A block is invalid if any transaction contains a nullifier that already exists in the set or duplicates a nullifier within the same block.
- Nullifiers are added to the set atomically when a block is accepted.
- During a chain reorganization, nullifiers from orphaned blocks are removed from the set.

**11.3 Proof Verification**

- Every TXv6 transaction with a non-empty shielded bundle MUST contain a valid Halo 2 proof.
- The proof MUST verify against the **circuit verification key** (a fixed parameter of the protocol, embedded in node software).
- The circuit enforces all properties listed in [Section 1 (Proof System)](#1-proof-system-halo-2).
- Invalid proofs render the containing transaction, and therefore the containing block, invalid.

**11.4 Anchor Validity**

- The anchor in a shielded bundle MUST equal the note commitment tree root at some block height `h` where `current_height - 100 <= h <= current_height`.
- This 100-block anchor validity window:
  - Allows mempool transactions to survive minor reorgs.
  - Prevents transactions from referencing arbitrarily old tree states.
  - Balances security against usability (at ~10 min/block, this is approximately 16.7 hours).

**11.5 Phase-Specific Rules**

| Rule | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|------|---------|---------|---------|---------|
| TXv6 valid | Yes | Yes | Yes | Yes |
| Legacy TX (v1/v2) valid | Yes | Yes | No | No |
| Transparent outputs in TXv6 | Yes | Yes (with fee multiplier) | No | No |
| Transparent inputs in TXv6 | Yes | Yes | Shielding only | Legacy UTXOs only |
| Coinbase transparent | No | No | No | No |
| Coinbase shielded | Yes (required) | Yes (required) | Yes (required) | Yes (required) |

**11.6 Coinbase Shielding**

- From Phase 1 activation, coinbase transactions MUST produce shielded outputs.
- The coinbase transaction uses a special action where the "spent" note is a dummy (no real input) and the value flows from the coinbase reward into the shielded pool.
- The miner's block reward (subsidy + fees) is encoded in the value balance of the coinbase transaction's shielded bundle.

### 12. Block Header Changes

The block header is extended with three new fields:

```
BlockHeader {
    // Existing BCH fields (unchanged)
    version:                        int32
    prev_block_hash:                32 bytes
    merkle_root:                    32 bytes
    timestamp:                      uint32
    bits:                           uint32
    nonce:                          uint32

    // New fields (appended after nonce)
    note_commitment_tree_root:      32 bytes    // Root of the Sinsemilla Merkle tree
                                                // after processing all transactions in this block

    nullifier_tree_root:            32 bytes    // Root of an append-only accumulator of all
                                                // nullifiers (for compact sync proofs)

    shielded_value_pool_balance:    int64       // Cumulative net satoshis transferred into the
                                                // shielded pool since activation (turnstile)
}
```

The `shielded_value_pool_balance` serves as the turnstile: at every block, nodes verify that this value equals the sum of all `value_balance` fields (with appropriate sign) from all TXv6 transactions since Phase 1 activation. This provides a public, verifiable accounting of the shielded pool's total value without revealing individual transaction amounts.

### 13. Migration Mechanism

The migration from transparent UTXOs to the shielded pool proceeds as follows:

**13.1 Shielding Transactions**

A shielding transaction is a TXv6 transaction that:
- Spends one or more transparent inputs (standard BCH UTXOs with scriptSig).
- Produces one or more shielded outputs (actions with dummy spends).
- Has a negative `value_balance` (value flows from transparent to shielded).
- Has `tx_out_count = 0` (no transparent change, all change is shielded).

Example: Alice has a 1.5 BCH transparent UTXO and wants to shield it.

```
TXv6 {
    tx_in: [{ prev_out: Alice's UTXO, scriptSig: Alice's signature }]
    tx_out: []  // No transparent outputs
    shielded_bundle: {
        actions: [
            {
                nullifier: <dummy>,
                note_commitment: <commitment to 1.49999 BCH note for Alice>,
                cv: <value commitment>,
                ...
            }
        ],
        value_balance: -149999000,  // -1.49999 BCH (negative = transparent → shielded)
        ...
    }
    // Fee: 0.00001 BCH (from transparent input surplus)
}
```

**13.2 Wallet Auto-Migration**

Wallet software SHOULD implement automatic migration with the following properties:

- **Batched migration**: Rather than shielding all UTXOs in a single transaction, wallets SHOULD spread migration across multiple transactions over days or weeks. This reduces timing correlation that could link the transparent source to the shielded destination.
- **Random delays**: Each migration transaction SHOULD be submitted after a random delay (uniformly distributed over a configurable window, e.g., 1-48 hours).
- **UTXO consolidation**: Wallets SHOULD consolidate multiple small UTXOs into a single shielded note to reduce future transaction overhead.
- **Progress indication**: Wallets MUST clearly display migration progress to users, showing the percentage of funds migrated and estimated completion time.

**13.3 Turnstile Integrity**

At every block height `h` after activation:

```
shielded_pool_balance(h) = sum over all blocks [activation, h] of:
    sum over all TXv6 transactions of: -value_balance
```

And the invariant holds:

```
transparent_utxo_total(h) + shielded_pool_balance(h) = total_mined_supply(h)
```

If this invariant is violated, the block is invalid. This provides a continuous public audit of the total supply, detecting any inflation exploit in the shielded pool (e.g., due to a bug in the ZK circuit or the Halo 2 implementation) with at most one block of delay.

**13.4 Handling Lost Coins**

Some transparent UTXOs will never be migrated because:
- The private keys are permanently lost.
- The owner is deceased without key succession.
- The UTXOs are provably unspendable (e.g., OP_RETURN outputs, or outputs with invalid scripts).

These UTXOs are handled as follows:
- They remain in the transparent UTXO set indefinitely.
- They can be spent (shielded) at any time in the future, even after Phase 4.
- They contribute to a permanent, auditable reduction in circulating supply.
- No protocol-level "sweep" or forced migration is performed, as this would require knowledge of the spending keys.

### 14. Cryptographic Primitives

| Primitive | Construction | Purpose | Domain |
|-----------|-------------|---------|--------|
| Pedersen Commitment | Incomplete addition on Pallas | Note commitments, value commitments | Core privacy mechanism |
| Sinsemilla | Incomplete addition on Pallas with domain separation | Merkle tree internal hash | Note commitment tree |
| Poseidon | Algebraic hash, width-5 | Nullifier derivation, key derivation functions | In-circuit hashing |
| Inner Product Argument | Over Pallas pedersen commitments | Core proof system (no trusted setup) | Halo 2 proof generation/verification |
| UltraPLONK | Custom gates + lookup tables | Circuit arithmetization | ZK circuit representation |
| RedPallas | Schnorr-like signature on Pallas | Spend authorization signatures, binding signatures | Transaction signing |
| Diffie-Hellman | ECDH on Pallas curve | Note encryption key agreement | Encrypted note transmission |
| ChaCha20-Poly1305 | Symmetric AEAD cipher | Note plaintext encryption/decryption | In-band note transmission |
| BLAKE2b | 512-bit hash | Key derivation, PRFs, domain separation | Various |
| BLAKE2s | 256-bit hash | Personalized hashing for compact outputs | Various |

### 15. Address Format

Shielded addresses encode the payment address `(d, pk_d)` using a new address format that is visually distinguishable from existing BCH CashAddr addresses:

- **Prefix**: `shielded:` (e.g., `shielded:qz3p8f7k...`)
- **Encoding**: Bech32m (BIP-350) for error detection
- **Payload**: `diversifier (11 bytes) || pk_d (32 bytes)` = 43 bytes
- **Human-readable part**: `shielded` for mainnet, `shieldedtest` for chipnet/testnet

Example: `shielded:1qpzry9x8gf2tvdw0s3jn54khce6mua7l7rmhg3d...`

This address format:
- Is immediately distinguishable from transparent CashAddr addresses.
- Uses Bech32m for error detection (up to 4 character errors detected).
- Supports the full diversifier + transmission key pair required for shielded payments.

## Rationale

### Why Halo 2 Over Groth16

Groth16 (used in Zcash Sapling) produces smaller proofs (~192 bytes vs ~1,440 bytes) and has faster single-threaded verification. However, it requires a **trusted setup ceremony**, a multi-party computation where participants generate a Common Reference String (CRS) and must destroy their toxic waste. If any participant retains their contribution, they can forge proofs and counterfeit coins.

For Bitcoin Cash, trusted setup is unacceptable:

1. **Trust minimization is a core value**. BCH exists because the community rejected centralized decision-making. Requiring trust in a ceremony contradicts this ethos.
2. **Ongoing operational burden**. Every circuit change (e.g., a future upgrade to the ZK circuit) requires a new trusted setup ceremony.
3. **Undetectable counterfeiting**. A compromised setup enables minting arbitrary coins within the shielded pool with no way to detect it (unlike the transparent pool where supply is trivially auditable).

Halo 2 eliminates all of these concerns. The moderate increase in proof size (~7.5x) and verification time (comparable with 3-4 threads) is a worthwhile tradeoff for trustlessness.

### Why Halo 2 Over zk-STARKs

zk-STARKs offer two advantages: no trusted setup and plausible post-quantum security. However:

1. **Proof sizes are 10-100x larger** (tens of kilobytes vs ~1.4 KB). For a payment chain targeting low fees and high throughput, this is prohibitive since every byte of proof data occupies block space.
2. **Verification is slower** for the proof sizes that would be needed.
3. **Post-quantum security is not yet urgent**. Quantum computers capable of breaking elliptic curve cryptography are not imminent, and the protocol can be upgraded to a post-quantum proof system in a future hard fork if needed.
4. **Ecosystem maturity**. Halo 2 has been audited, deployed in production (Zcash NU5), and has mature Rust implementations. STARK-based privacy systems are less battle-tested in production cryptocurrency deployments.

### Why Mandatory Over Optional

The case for mandatory shielding rests on three pillars:

1. **Anonymity set size**. Privacy is a network effect: each additional user in the shielded pool increases the privacy of all other users. Optional shielding, as demonstrated by Zcash (~29% adoption after ~10 years), results in a fragmented anonymity set that provides weaker privacy guarantees.

2. **Elimination of the "privacy is suspicious" problem**. When shielding is optional, users who opt in are flagged as having something to hide. Exchanges, regulators, and chain analysis firms can apply enhanced scrutiny to shielded transactions precisely because they are uncommon. When ALL transactions are shielded, using privacy features carries no additional suspicion.

3. **True fungibility requires universality**. As long as some coins exist in the transparent pool, chain analysis can taint coins based on their history. Only when all coins are in the shielded pool is the concept of "coin history" eliminated entirely.

### Why Orchard-Style Actions

Earlier shielded protocols (Zcash Sprout, Zcash Sapling) used separate lists of spend descriptions and output descriptions. This reveals the **arity** of a transaction: an observer can count the number of inputs and outputs, which leaks information about the transaction type (payment vs. consolidation vs. split).

The Orchard action model pairs every spend with an output, using dummy operations to pad. This means all transactions with the same number of actions look identical, regardless of their actual input/output structure. The privacy improvement is significant for a payment chain where most transactions have predictable arity patterns.

### Why Flag-Day Activation

Bitcoin Cash has used Median Time Past (MTP) flag-day activation for every network upgrade since its inception. This mechanism:

- Is simple, predictable, and well-tested.
- Does not require miner signaling (avoiding political complications).
- Gives the entire ecosystem a clear deadline to prepare.
- Has been successfully used for consensus changes including CashTokens, VM Limits, and BigInt.

BIP9-style miner signaling is not used on BCH and would introduce unnecessary complexity and governance risks for this upgrade.

### Why a 6-Month Migration Window

The 6-month migration window is justified by Bitcoin Cash's current network activity:

- **Low transaction volume**: BCH processes approximately 12,000 transactions per day (~0.14 TPS). At this volume, the entire active UTXO set can be migrated in a matter of weeks, not years. A 6-month window provides ample time relative to actual on-chain activity.
- **Ecosystem readiness**: The chipnet testing period (6 months before mainnet activation) plus the 6-month migration window provides a full year of total preparation time for exchanges, wallets, and services. Major exchanges typically require 3-6 months of notice, which is satisfied by the chipnet announcement alone.
- **Urgency of privacy benefits**: Every additional month of optional-only shielding is a month where the anonymity set remains fragmented and BCH users lack protocol-level privacy. A shorter window accelerates the transition to mandatory privacy.
- **Economic incentive alignment**: The compressed escalating fee multiplier (2x → 5x → 20x over 2-month intervals) creates rapid, clear pressure to migrate. Given BCH's low transaction volume, the economic cost of the multiplier is manageable for any user who acts within the window.
- **Timing correlation**: While a shorter window provides less time for wallets to spread migration transactions, BCH's low daily volume means the absolute number of shielding transactions is small enough that even a 6-month spread provides adequate decorrelation.

A 6-month window is appropriate for a network processing ~12,000 transactions per day. Longer windows (e.g., 18 months) would unnecessarily delay privacy benefits without meaningfully improving migration completeness, given that the technical migration is simple and the UTXO set is modest in size.

## Performance and Scalability

### Transaction Size Impact

| Transaction type | Typical size | Notes |
|-----------------|-------------|-------|
| Current BCH (1-in, 2-out) | ~225 bytes | P2PKH, no OP_RETURN |
| Shielded (2 actions, typical payment) | ~2,820 bytes | 2 actions + proof + sigs |
| Shielded (1 action, simple transfer) | ~1,860 bytes | 1 action + proof + sig |
| Shielded (4 actions, complex) | ~4,740 bytes | 4 actions + proof + sigs |

Shielded transactions are approximately **8-13x larger** than their transparent equivalents. The dominant contributor is the encrypted note ciphertext (580 bytes per action) and the Halo 2 proof (~1,440 bytes per bundle, amortized across actions).

### Throughput Analysis

With Bitcoin Cash's current 32 MB block size limit:

| Metric | Transparent | Shielded (2-action) |
|--------|------------|-------------------|
| Max transactions per block | ~142,000 | ~11,300 |
| Effective TPS (10 min blocks) | ~237 | ~18.9 |

At ~19 TPS shielded, Bitcoin Cash would still process significantly more transactions per second than Zcash (~6 TPS) or Bitcoin (~7 TPS). If throughput becomes a bottleneck, the block size limit can be increased, consistent with BCH's existing scaling philosophy.

### Proof Generation Performance

| Platform | Time per action | 2-action transaction |
|----------|----------------|---------------------|
| Desktop (modern x86-64, single core) | < 25 ms | < 50 ms |
| Desktop (4 threads) | < 10 ms | < 20 ms |
| Mobile (modern ARM, single core) | ~200-500 ms | ~400 ms - 1 s |
| Hardware wallet (signing only) | ~50 ms | ~50 ms |

Proof generation is fast enough for interactive use on desktop and acceptable on mobile. For mobile users, the proof can be generated in the background while the user confirms the transaction details.

### Block Validation

| Block fullness | Transactions | Verification time (8 threads) |
|---------------|-------------|------------------------------|
| 25% full | ~2,825 | ~8-10 seconds |
| 50% full | ~5,650 | ~16-20 seconds |
| 100% full | ~11,300 | ~30-40 seconds |

Block validation is slower than for transparent transactions. However, proof verification is trivially parallelizable since each proof can be verified independently. On modern multi-core CPUs and with batch verification optimizations, full block validation remains practical.

### State Growth

| Component | Growth rate (at 50% block utilization) | Annual growth |
|-----------|---------------------------------------|---------------|
| Note commitment tree | ~32 bytes per note | ~5.7 GB/year |
| Nullifier set | ~32 bytes per spent note | ~5.7 GB/year |
| Block data (chain history) | Same as transparent (stored in blocks) | Proportional to block size |

The nullifier set is append-only and must be kept in fast-access storage (RAM or SSD) for validation. At ~12 GB/year total state growth (commitment tree + nullifiers), this is manageable with current hardware but warrants monitoring. Future optimizations include:

- **Nullifier set accumulator**: Replace the explicit set with a cryptographic accumulator (e.g., a Merkle Mountain Range) that supports compact membership proofs.
- **Pruning old tree paths**: Portions of the note commitment tree that are no longer needed for anchor validation (older than 100 blocks) can be pruned from working memory, retaining only the incremental frontier.

### Scalability Roadmap

Halo 2's recursive proof composition enables a clear long-term scalability path:

1. **Proof aggregation within blocks**: A block producer aggregates all transaction proofs within a block into a single recursive proof. Full nodes verify one proof per block instead of one per transaction, greatly reducing validation cost.
2. **Light client proofs**: A chain of recursive block proofs enables lightweight SPV-style clients that verify a single proof to validate the entire chain history, including shielded state transitions.
3. **Sharded verification**: Transaction proof verification can be distributed across multiple nodes, with each node verifying a subset of proofs and producing an aggregate attestation.

These optimizations are not required for initial deployment but become available for future upgrades without changes to the fundamental protocol design.

## Backwards Compatibility

**This proposal is a hard fork. There is no soft-fork alternative.**

The consensus rule changes introduced by this proposal (new transaction format, note commitment tree, nullifier set, proof verification, block header extensions) are incompatible with existing consensus rules. Non-upgraded nodes will reject post-activation blocks as invalid. This is consistent with all prior Bitcoin Cash network upgrades, which have used hard forks exclusively.

### Node Software

All full node implementations MUST upgrade before the Phase 1 activation timestamp. Nodes that do not upgrade will:

- Reject blocks containing TXv6 transactions as invalid (unknown transaction version).
- Reject blocks with the extended header fields as malformed.
- Fork onto a separate, incompatible chain with no shielded transaction support.

There is no grace period or backwards-compatible fallback. The activation is a clean break enforced by the MTP flag-day mechanism.

### Wallets

Wallet software compatibility changes across each phase:

| Phase | Upgraded wallets | Non-upgraded wallets |
|-------|-----------------|---------------------|
| Phase 1 (Activation) | Full functionality: can send/receive both transparent and shielded | Can still send/receive transparent transactions normally. Cannot create or detect shielded transactions. |
| Phase 2 (Migration) | Full functionality. SHOULD auto-migrate transparent UTXOs to shielded. | Can still send transparent transactions (with escalating fee penalties). Cannot receive shielded change or detect shielded payments. Increasingly degraded experience. |
| Phase 3 (Mandatory) | Full functionality. All transactions are shielded. | **Read-only for existing balances.** Can only spend transparent UTXOs into the shielded pool (which requires shielded output support they lack). Effectively unable to transact. |
| Phase 4 (Closure) | Full functionality. | **Fully non-functional.** Cannot construct any valid transaction. Balances are frozen until the wallet is upgraded. |

Non-upgraded wallets do not lose funds; the user's spending keys remain valid. Upgrading the wallet software at any point restores full access, including the ability to shield legacy transparent UTXOs.

### Why No Soft-Fork Alternative Exists

A soft-fork constrains the valid transaction set (new rules are a subset of old rules), allowing non-upgraded nodes to accept new blocks without understanding the new features. This is incompatible with shielded transactions because:

1. **New block header fields** (note commitment tree root, nullifier tree root, shielded pool balance) change the block serialization format. Old nodes cannot parse these headers.
2. **Proof verification is mandatory for consensus.** Old nodes that cannot verify Halo 2 proofs would accept blocks with invalid proofs, breaking consensus safety.
3. **The nullifier set is a new consensus-critical data structure.** Old nodes that do not maintain the nullifier set cannot detect double-spends within the shielded pool.
4. **Mandatory shielding (Phase 3+) explicitly invalidates transaction types that old nodes consider valid.** This is the opposite of a soft-fork, which only restricts (never expands) the set of invalid transactions from the perspective of old nodes.

A hard fork is the only viable activation mechanism for this class of consensus change.

### Existing On-Chain Data

All historical transparent transaction data remains valid and accessible on the blockchain. The hard fork does not alter, rewrite, or prune any existing blocks. Block explorers and archival nodes can continue to serve historical transparent data indefinitely. The privacy guarantees apply only to transactions created after the activation. Prior transaction history remains public.

## Risk Assessment

### Unprecedented Migration

No existing cryptocurrency has performed a mandatory migration from a transparent UTXO set to a fully shielded pool on a live network with significant economic value. The closest precedents are:

- **Monero's RingCT mandate** (September 2017): Made ring confidential transactions mandatory after an 8-month optional period. However, Monero was already designed for privacy; only the amount-hiding component changed.
- **Zcash's Sprout deprecation** (Canopy, 2020): Disabled new value entering the Sprout pool. However, transparent transactions remained fully available.

**Mitigation**: The 6-month migration window, escalating fee multiplier, and wallet auto-migration tooling are designed to maximize migration completeness before the mandatory cutoff. Given BCH's low transaction volume (~12,000 TX/day), 6 months provides ample time to migrate the entire active UTXO set. Extensive testing on chipnet (6 months) will validate the migration process.

### Regulatory Risk

Privacy coins face regulatory headwinds in multiple jurisdictions. Several exchanges have delisted Zcash, Monero, and other privacy-focused cryptocurrencies in response to regulatory pressure.

**Mitigation**: This is a risk that the BCH community must consciously accept. The proposal does not include backdoors, view key escrow, or "transparent mode" overrides, as these would undermine the privacy guarantees and create a false sense of compliance. The **incoming viewing key** architecture does allow users to voluntarily prove their transaction history to regulators or auditors on a per-key basis, providing a compliance path without compromising the protocol.

### Ecosystem Coordination

All software that interacts with Bitcoin Cash must be updated:

- **Node implementations**: Bitcoin Cash Node (BCHN), Bitcoin Unlimited, Knuth, Bitcoin Verde, and others must implement the full specification including Halo 2 verification, the note commitment tree, and the nullifier set.
- **Wallets**: All wallets must support shielded transaction construction, proof generation, trial decryption, and the new address format.
- **Exchanges**: Must support shielded deposits and withdrawals, and update their compliance tooling.
- **Block explorers**: Must adapt to a model where transaction details are hidden by default.
- **Payment processors**: Must support shielded payment workflows.

**Mitigation**: The timeline (CHIP publication → chipnet → mainnet → migration → mandatory) provides over a year of total preparation time from CHIP acceptance through mandatory shielding. Reference implementations and libraries should be published well in advance of chipnet activation.

### State Growth

The nullifier set grows monotonically (nullifiers are never removed) at approximately 32 bytes per spent note. Under sustained heavy usage, this could become a storage concern.

**Mitigation**: The nullifier set can be stored in a compact data structure (e.g., a Merkle Mountain Range or a sparse Merkle tree) that supports efficient membership queries. Additionally, the note commitment tree's incremental structure means only the frontier needs to be kept in memory.

### Cryptographic Assumptions

The security of the shielded pool relies on:

1. **Discrete Logarithm Problem** on the Pallas/Vesta curves: If broken (e.g., by quantum computers), proofs can be forged and coins counterfeited.
2. **Collision resistance** of Poseidon and Sinsemilla: If broken, nullifier uniqueness and tree integrity are compromised.
3. **Soundness of Halo 2**: If the proof system has a flaw, invalid proofs could be accepted.

**Mitigation**: These are standard, well-studied cryptographic assumptions. Halo 2 has undergone multiple independent audits (including by Kudelski Security). The Pallas/Vesta curves are designed specifically for this use case and have been analyzed by the Electric Coin Company's cryptography team.

### Quantum Resistance

Halo 2's PLONK arithmetization and IPA commitment scheme rely on the hardness of the discrete logarithm problem on elliptic curves (Pallas/Vesta). A sufficiently powerful quantum computer running Shor's algorithm could break these assumptions, enabling proof forgery and undetectable counterfeiting within the shielded pool.

The threat is not immediate (current estimates place cryptographically relevant quantum computers at least 10-20 years away), but it is a long-term concern that must be acknowledged:

- **Elliptic curve operations** (Pedersen commitments, Diffie-Hellman key agreement, RedPallas signatures) are all vulnerable to quantum attack.
- **Poseidon and Sinsemilla** hash functions are believed to retain security against quantum adversaries (quantum computers provide at most a quadratic speedup against hash preimage, which is manageable with sufficient output length).
- **The proof system itself** (IPA-based PLONK) would need to be replaced entirely with a post-quantum alternative.

**Mitigation**: This is a known, industry-wide challenge that affects every elliptic-curve-based cryptocurrency (including Bitcoin, Ethereum, and Zcash). The mitigation path is:

1. **Monitor quantum computing progress** and initiate a transition plan when cryptographically relevant quantum computers are within a 5-year horizon.
2. **Future hard fork** to migrate from Halo 2 to a post-quantum proof system. Candidates include:
   - **Hash-based zk-STARKs** (e.g., based on FRI commitments), which rely only on collision-resistant hashes and are plausibly post-quantum. The tradeoff is larger proof sizes (tens of KB vs ~1.4 KB).
   - **Lattice-based SNARKs**, an active area of research that may offer compact post-quantum proofs.
3. **Halo 2's recursive composition** makes the transition smoother: a new post-quantum proof system can verify old Halo 2 proofs recursively, enabling a gradual migration without invalidating the existing shielded pool.

The decision to use Halo 2 today is pragmatic: it is the most mature, audited, and performant trustless proof system available. Waiting for post-quantum alternatives that do not yet exist in production-ready form would delay privacy benefits indefinitely.

### ZK Circuit Implementation Bugs

Zero-knowledge proof circuits are complex software artifacts. A bug in the circuit, even a subtle one, could allow an attacker to forge valid proofs for invalid statements, enabling:

- **Undetectable inflation**: Creating shielded notes with value exceeding the spent inputs, minting BCH out of thin air within the shielded pool.
- **Double spending**: Producing valid proofs for nullifiers that do not correspond to real note commitments.
- **Theft**: Spending notes without possessing the correct spending key.

This is not a theoretical concern. Zcash has experienced circuit-level vulnerabilities:

- **CVE-2019-7167 (Zcash Sprout)**: A flaw in the Sprout circuit's constraint system would have allowed counterfeiting. It was discovered internally and patched in the Sapling upgrade before exploitation. The bug existed undetected from Zcash's launch in October 2016 until its discovery in early 2018, over a year of potential exposure.
- **The "InternalH" vulnerability (2018)**: A subtle flaw in the Sprout proving system's constraint generation that could have allowed proof forgery.

**Mitigation**:

1. **Multiple independent audits**. The Halo 2 circuit MUST be audited by at least three independent security firms with cryptographic expertise before chipnet deployment. Zcash's Orchard circuit was audited by multiple firms; this proposal should meet or exceed that standard.
2. **Formal verification**. Where feasible, critical circuit components (nullifier derivation, value commitment balance, Merkle path verification) should be formally verified using tools like Lean, Coq, or purpose-built circuit verification frameworks.
3. **Turnstile detection**. The turnstile mechanism (public tracking of `total_transparent + total_shielded = total_supply`) provides an independent check: if a circuit bug enables inflation within the shielded pool, the turnstile invariant will be violated as soon as inflated value attempts to exit the pool. This does not prevent the bug, but it enables detection.
4. **Bug bounty program**. A well-funded bug bounty program should be established before chipnet activation, with rewards commensurate with the severity of potential exploits.
5. **Conservative deployment timeline**. The 6-month chipnet testing period is a minimum. If audits reveal concerns, deployment should be delayed rather than rushed.

### Performance and Block Propagation

Shielded transactions are significantly larger and more expensive to validate than transparent transactions, with direct impact on network performance:

| Metric | Transparent | Shielded (2-action) | Impact |
|--------|------------|---------------------|--------|
| Transaction size | ~225 bytes | ~2,820 bytes | ~12.5x larger |
| Proof size | 0 bytes | ~1,440 bytes | New overhead per TX bundle |
| Verification (single-threaded) | < 1 ms | ~30 ms | ~30x slower per TX |
| Verification (8 threads, batch) | < 1 ms | ~5-8 ms | ~5-8x slower per TX |
| Full block validation (32 MB) | < 1 second | ~30-40 seconds | Significant increase |

**Block propagation**: Larger transactions mean larger blocks at equivalent transaction counts. A 32 MB block of shielded transactions contains ~11,300 TXs (vs ~142,000 transparent TXs). At equivalent economic throughput, blocks are not larger in bytes, but validation latency increases substantially. This impacts:

- **Compact block relay**: Miners who cannot validate blocks quickly face increased orphan rates, creating centralization pressure toward faster hardware.
- **Initial block download (IBD)**: Syncing a new node from genesis requires verifying every historical proof. A year of 50%-full blocks would require verifying ~2.9 million proofs during IBD.
- **Mempool validation**: Each incoming transaction requires proof verification before relay, increasing mempool processing latency.

**Mitigation**:

1. **Parallel verification**. Halo 2 proof verification is trivially parallelizable. Modern full nodes should use all available CPU cores for block validation. With 8+ cores, per-transaction verification time approaches transparent levels.
2. **Block validation caching**. Transactions already validated in the mempool do not need re-verification when they appear in a block. This eliminates the majority of validation overhead for miners who have been receiving transactions in real-time.
3. **Hardware requirements update**. Full node minimum hardware recommendations should be updated to reflect the increased CPU demands: a modern multi-core CPU (8+ cores recommended) and NVMe SSD storage for the nullifier set.
4. **Proof aggregation (future)**. The scalability roadmap (see [Scalability Roadmap](#scalability-roadmap)) includes recursive proof aggregation within blocks, which would reduce per-block verification to a single proof regardless of transaction count.

### Tooling and Compliance Breakage

Mandatory privacy changes what information is available to ecosystem tools:

- **Block explorers** can no longer display sender, receiver, or amount for any transaction. They can show: block height, transaction count, proof validity, nullifier count, fee amounts, and aggregate shielded pool balance. This is a significant reduction in functionality that affects user experience and debugging.
- **Tax reporting software** that relies on public blockchain data to reconstruct transaction histories will no longer function. Users must export transaction data from their own wallets (using viewing keys) to meet tax obligations.
- **AML/KYC compliance tools** used by exchanges (Chainalysis, Elliptic, etc.) will lose the ability to trace on-chain flows. Exchanges must rely on off-chain compliance measures (identity verification at deposit/withdrawal endpoints) rather than on-chain surveillance.
- **Payment verification**: Merchants and services that verify payments by watching the public blockchain must instead use the payment protocol or direct wallet-to-wallet confirmation via viewing keys.

**Mitigation**:

1. **Viewing key disclosure**. The key hierarchy provides a compliance path: users can share their **incoming viewing key** with exchanges, auditors, or tax authorities to prove specific transaction histories. This is voluntary and per-key, preserving the protocol's privacy guarantees while enabling individual compliance.
2. **Payment proofs**. Shielded transactions can include cryptographic payment proofs that the sender provides to the recipient, proving that a specific payment was made without revealing it to third parties.
3. **New explorer capabilities**. While individual transaction details are hidden, block explorers can still provide valuable aggregate data: network throughput, fee statistics, shielded pool growth, nullifier set size, and proof verification status.
4. **Transition support**. During the 6-month migration window (Phase 2), transparent transaction data remains available for historical transactions. This provides time for tooling providers to adapt their products.
5. **Exchange compliance**. Major exchanges already handle privacy coins (Monero remains listed on several major exchanges). The incoming viewing key model provides a stronger compliance story than Monero's view keys, because Zcash/Orchard-style viewing keys provide cryptographic certainty of completeness (all incoming transactions are visible to the key holder).

## Prior Art and Alternatives

### Zcash (Sapling and Orchard)

Zcash pioneered shielded transactions in cryptocurrency, introducing the first production deployment of zk-SNARKs (Sprout, 2016), followed by the more efficient Sapling protocol (2018) and the Orchard protocol with Halo 2 (2022).

**Key differences from this proposal**:
- Zcash shielding is **optional**; this proposal makes it **mandatory**.
- Zcash maintains full transparent transaction support; this proposal eliminates it after the migration period.
- Zcash's Sapling uses Groth16 (trusted setup); this proposal uses Halo 2 (no trusted setup).
- Zcash's anonymity set is fragmented (~30% shielded); this proposal targets 100%.

This proposal heavily borrows from Zcash's Orchard specification (note structure, action model, key hierarchy, proof system) while making the critical policy decision to mandate privacy rather than offering it optionally.

**Important**: This proposal is not intended to compete with or replace Zcash. Zcash remains a pioneering privacy cryptocurrency with an active research program and a distinct community. BCH and Zcash serve complementary roles: BCH as high-throughput, low-fee peer-to-peer cash with large-block scaling, and Zcash as a research-driven privacy platform. Both chains benefit from a larger ecosystem of privacy-preserving cryptocurrencies. See [Complementary to Zcash, Not a Replacement](#complementary-to-zcash-not-a-replacement) in the Motivation section for a full discussion.

### Monero (RingCT)

Monero uses Ring Confidential Transactions (RingCT) to hide transaction amounts and ring signatures to obfuscate the sender among a set of decoys. Since September 2017, RingCT has been mandatory.

**Key differences**:
- Monero uses ring signatures (not zero-knowledge proofs) for sender privacy, providing plausible deniability rather than cryptographic certainty.
- Monero's ring size (currently 16) limits the per-transaction anonymity set.
- Monero does not use a shielded pool model; it uses a UTXO model with decoy sets.
- RingCT does not require a note commitment tree or nullifier set.
- Monero's approach is simpler but provides weaker privacy guarantees (ring signature decoy analysis is an active area of chain surveillance research).

### Iron Fish

Iron Fish is a Layer 1 cryptocurrency launched with mandatory privacy by default, using zk-SNARKs (Groth16, Sapling-based circuit) for all transactions.

**Key differences**:
- Iron Fish was shielded from genesis (no migration needed).
- Iron Fish uses Groth16 (trusted setup); this proposal uses Halo 2.
- Iron Fish is a separate chain; this proposal upgrades an existing chain with significant economic activity.

Iron Fish demonstrates that a fully shielded UTXO chain is viable in production.

### Penumbra

Penumbra is a fully shielded Cosmos SDK chain supporting privacy-preserving DeFi operations (shielded DEX, staking, governance).

**Key differences**:
- Penumbra is built on Cosmos SDK (proof-of-stake, IBC interoperability); BCH is proof-of-work.
- Penumbra was shielded from genesis.
- Penumbra demonstrates that shielded pools can support DeFi primitives, suggesting a future path for BCH if DeFi features are desired.

### CashFusion (BCH)

CashFusion is the current state-of-the-art privacy tool for Bitcoin Cash. It creates large CoinJoin transactions where multiple users combine their inputs and outputs, making it difficult to trace individual funds.

**Limitations addressed by this proposal**:
- CashFusion requires opt-in participation (small anonymity sets).
- CashFusion transactions are identifiable on-chain.
- CashFusion does not hide amounts.
- CashFusion provides probabilistic privacy (can be degraded with enough analysis) vs. cryptographic privacy (guaranteed by the proof system).
- CashFusion requires coordination with other online users.

## Stakeholder Responses and Statements

*This section will be populated as the CHIP is reviewed by stakeholders. The following stakeholder categories are invited to provide responses:*

**Node implementations**:
- Bitcoin Cash Node (BCHN)
- Bitcoin Unlimited
- Knuth
- Bitcoin Verde

**Wallets**:
- Electron Cash
- Bitcoin.com Wallet
- Cashonize
- Paytaca

**Exchanges and services**:
- CoinFlex
- Prompt.cash
- SideShift.ai

**Independent reviewers**:
- *Invited*

## Test Vectors

Test vectors will be published in a companion repository upon completion of the reference implementation. The test suite will cover:

1. **Note commitment computation**: Given note fields, verify the correct commitment output.
2. **Nullifier derivation**: Given a note and nullifier deriving key, verify the correct nullifier.
3. **Value commitment**: Given a value and blinding factor, verify the correct commitment.
4. **Merkle tree operations**: Given a sequence of note commitments, verify tree roots after each insertion.
5. **Transaction validation**: Complete TXv6 transactions with valid and invalid proofs, nullifiers, anchors, and balances.
6. **Shielding transactions**: Transparent-to-shielded migration transactions with correct value balance accounting.
7. **Key derivation**: Given a spending key, verify the correct derivation of all subordinate keys and addresses.
8. **Encrypted note round-trip**: Encrypt a note, verify the recipient can decrypt it, verify the sender can recover it via OVK.

## Implementations

*Reference implementations will be listed here as they are developed.*

**Planned**:
- Halo 2 circuit (Rust, based on the `halo2` crate)
- Transaction builder library (Rust)
- Wallet integration library (Rust with C FFI for integration into C++ node software)
- BCHN integration (C++ node, linking to Rust libraries)

## Feedback and Reviews

Community discussion and review of this proposal is welcomed at:

- **Bitcoin Cash Research**: [To be posted]
- **Reddit r/btc**: [To be posted]
- **Telegram**: Bitcoin Cash Development group
- **GitHub Issues**: [https://github.com/sneakycapital/chip-bch-shielded-pool/issues](https://github.com/sneakycapital/chip-bch-shielded-pool/issues)

## Acknowledgements

This proposal builds extensively on the work of the Zcash protocol engineering team, particularly:

- The **Orchard protocol** specification (Daira-Emma Hopwood, Sean Bowe, Jack Grigg, and others at the Electric Coin Company).
- The **Halo 2** proving system (Sean Bowe, Jack Grigg, Daira-Emma Hopwood, Ying Tong Lai).
- The **Pallas and Vesta** elliptic curves (Daira-Emma Hopwood, Sean Bowe).
- The **Zcash Protocol Specification** (Daira-Emma Hopwood, Sean Bowe, Taylor Hornby, Nathan Wilcox).

The Bitcoin Cash community's prior work on privacy, including CashShuffle and CashFusion (Jonald Fyookball and others), demonstrated the demand for privacy features and informed the motivation for this proposal.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0-draft | 2026-02-18 | Initial publication. |
| 0.1.1-draft | 2026-02-19 | Compressed activation timeline from 30 months to 12 months. Updated Zcash shielded pool statistics to Feb 2026 figures. |

## Copyright

This document is placed in the public domain.
