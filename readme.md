# CHIP-2021-05 Loops: Bounded Looping Operations

        Title: Bounded Looping Operations
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Initial Publication Date: 2021-05-28
        Latest Revision Date: 2025-09-05
        Version: 1.2.3 (bd3ebc76)
        Status: Frozen for Lock-In

## Summary

This proposal adds two new BCH virtual machine (VM) operations, `OP_BEGIN` and `OP_UNTIL`, enabling a variety of loop constructions in BCH contracts without increasing the processing or memory requirements of the VM.

## Deployment

Deployment of this specification is proposed for the May 2026 upgrade.

- Activation is proposed for `1763208000` MTP, (`2025-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1778846400` MTP, (`2026-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Motivation

The Bitcoin Cash VM is [strictly limited](https://github.com/bitjson/bch-vm-limits) to prevent maliciously-designed transactions from requiring excessive resources during transaction validation.

Loops were originally excluded from the VM design as part of an initial, anti-Denial-of-Service approach. However, despite the "no loops" approach being quietly abandoned for explicit limits ([`reverted makefile.unix wx-config -- version 0.3.6` – July 29, 2010](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/commit/757f0769d8360ea043f469f3a35f6ec204740446)), the Bitcoin Cash VM is still missing this basic control flow structure, creating significant waste in many contracts by requiring duplication of contract bytecode.

**This proposal adds the well-established, `OP_BEGIN`/`OP_UNTIL` loop construction used by most Forth-like languages**, making BCH contracts significantly more powerful and efficient.

## Benefits

Looping operations enable many use cases [where specifying a precise procedure using `OP_IF` is impractical or impossible](#aggregation), and they also significantly [reduce the bytecode length of many contracts with repeated procedures](#simplifying-repeated-procedures).

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Modification to Transaction Validation

Without adequate limits, looping operations have the potential to increase worst-case transaction validation costs and expose VM implementations to Denial of Service (DOS) attacks.

Following [`CHIP 2021-05 VM Limits`](https://github.com/bitjson/bch-vm-limits/), the Bitcoin Cash VM consistently prevents abuse of all VM operations via density-based limits. Accordingly, new flow control structures cannot magnify worst-case validation performance – all constructions made more concise by this proposal are equivalently limited to a corresponding degree across each of the VM's existing abuse prevention metrics. (Note also that this proposal was specifically reviewed as part of the [VM Limits CHIP: Risk Assessment](https://github.com/bitjson/bch-vm-limits/blob/master/risk-assessment.md#consideration-of-possible-future-changes).)

Additionally, this proposal includes new test vectors covering both functional behavior and worst-case validation performance (across both standard and nonstandard validation).

### Node Upgrade Costs

This proposal affects consensus – all fully-validating node software must implement these VM changes to remain in consensus.

These VM changes are backwards-compatible: all past and currently-possible transactions remain valid under these new rules.

### Ecosystem Upgrade Costs

Because this proposal only affects internals of the VM, standard wallets, block explorers, and other services will not require software modifications for these changes. Only software which offers VM evaluation (e.g. [Bitauth IDE](https://github.com/bitauth/bitauth-ide)) will be required to upgrade.

Wallets and other services may optionally upgrade to add support for new contracts which will become possible after deployment of this proposal.

## Technical Specification

Two new VM operations are added: `OP_BEGIN` (`0x65`/`101`) and `OP_UNTIL` (`0x66`/`102`). To support these operations, the control stack is modified to accept instruction pointer positions.

<details>

<summary>Codepoint Selection</summary>

These codepoints were previously used by `OP_VERIF` (`0x65`) and `OP_VERNOTIF` (`0x66`) in early versions of Bitcoin, but those operations were disabled when it was realized that they created a fork-risk between different versions of the Bitcoin software. They are particularly well-suited for this application because they are the only two available, adjacent codepoints within the control-flow range of operations.

</details>

### Control Stack

This proposal modifies the control stack (A.K.A. `vfExec` or `ConditionStack`) to support instruction pointer positions. When testing for conditional execution, implementations must ignore items representing these positions.

<details>

<summary>Condition Stack vs. Control Stack</summary>

Prior to this proposal, the control stack is conceptually an array of boolean values: when a flow control operation (`OP_IF` or `OP_NOTIF`) is encountered, a `true` is pushed to the stack if the branch is to be executed, and `false` if not. (`OP_ELSE` toggles the top value on this stack.) Non-flow control operations are only evaluated if the control stack contains no `false` values.

Following implementation of this proposal, the control stack is conceptually an an array of either boolean **_or integer values_**: when `OP_BEGIN` is encountered, the current instruction pointer position is pushed to the control stack. Later, if a matching `OP_UNTIL` isn't satisfied, the program returns to that position.

</details>

### `OP_BEGIN` (`0x65`/`101`)

`OP_BEGIN` pushes the next instruction pointer index to the control stack (A.K.A. `vfExec` or `ConditionStack`).

<details>

<summary>Explanation</summary>

This operation sets a pointer to which the evaluation will return when the matching `OP_UNTIL` is encountered. IF an `OP_ENDIF` is encountered before the matching `OP_UNTIL` – or a matching `OP_UNTIL` is never encountered – the evaluation returns an error.

</details>

### `OP_UNTIL` (`0x66`/`102`)

`OP_UNTIL` inspects the top item from the control stack:

- If this control item is not an instruction pointer position (set by `OP_BEGIN`), error.
- The top item is popped from the stack.
- If the value is `0` (the same test as `OP_NOTIF`) the evaluation's instruction pointer is set to the position indicated by the control item (immediately after the matching `OP_BEGIN` instruction); otherwise, evaluation continues past the `OP_UNTIL`.

## Rationale

This section documents design decisions made in this specification.

### Naming and Semantics

This proposal is influenced by the `BEGIN ... UNTIL` indefinite loop construction in the Forth programming language (upon which the rest of the Bitcoin Cash VM is based). This particular construction offers maximal flexibility: definite loops can be easily emulated with indefinite loops, but indefinite loops cannot be emulated with definite loops.

### Suitability in Covenant Applications

One alternative to the proposed `OP_BEGIN ... OP_UNTIL` construction could be a single `OP_GOTO` operation which (with certain limits) jumps to the instruction pointer at the top of the stack.

While such an operation could help to further increase the efficiency and expressiveness of the BCH instruction set, a `GOTO` instruction also makes modeling execution more difficult. This is particularly limiting for covenant applications – it is notably more difficult for covenants to keep track of expected instruction pointer indexes (to reconstruct contracts with hypothetical `OP_GOTO` operations) than it is for them to use this proposal's construction.

The higher-level `OP_BEGIN ... OP_UNTIL` construction also better matches the semantics of the existing VM instruction set (e.g. `OP_IF ... OP_ENDIF`).

## Usage Examples

The following examples demonstrate usage of `OP_BEGIN` and `OP_UNTIL`.

### Simplifying Repeated Procedures

Bounded loops reduce wasted bytes in repeated procedures like merkle proof validation.

Merkle trees offer a versatile data structure for storing state in BCH contracts. Below is one possible strategy for validating a merkle proof (shown validating the left-most element in a tree of 16 elements):

```
<sibling_tier_4_hash>
<sibling_tier_3_hash>
<sibling_tier_2_hash>
<sibling_leaf_hash>
<value>
<1> OP_TOALTSTACK
<1> OP_TOALTSTACK
<1> OP_TOALTSTACK
<1> OP_TOALTSTACK

OP_FROMALTSTACK // check if leaf requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH256

OP_FROMALTSTACK // check if tier 2 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH256

OP_FROMALTSTACK // check if tier 3 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH256

OP_FROMALTSTACK // check if tier 4 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH256

<root> OP_EQUALVERIFY
```

With bounded loops, this procedure can be simplified to:

```
<sibling_tier_4_hash>
<sibling_tier_3_hash>
<sibling_tier_2_hash>
<sibling_leaf_hash>
<value>
<0> OP_TOALTSTACK
<1> OP_TOALTSTACK
<1> OP_TOALTSTACK
<1> OP_TOALTSTACK
<1>

OP_BEGIN
  OP_IF OP_SWAP OP_ENDIF // check if leaf requires swap
  OP_CAT OP_HASH256
  OP_FROMALTSTACK
OP_UNTIL

<root> OP_EQUALVERIFY
```

### Aggregation

Without bounded loops, contracts are forced to emulate aggregation by re-specifying a particular aggregation procedure for each possible set size.

For example, to aggregate the UTXO value of each transaction input without bounded loops (for up to 4 inputs using [transaction introspection operations](https://gitlab.com/GeneralProtocols/research/chips/-/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md)):

```
<0> // aggregated value
<4> // the count of values to aggregate

// locking bytecode
OP_DUP OP_TXINPUTCOUNT OP_LESSTHANOREQUAL OP_VERIFY // prevent malleability
OP_1SUB OP_DUP <0> OP_GREATERTHANOREQUAL OP_IF
    OP_SWAP
    <0> OP_UTXOVALUE
    OP_ADD
    OP_SWAP
OP_ENDIF
OP_1SUB OP_DUP <0> OP_GREATERTHANOREQUAL OP_IF
    OP_SWAP
    <1> OP_UTXOVALUE
    OP_ADD
    OP_SWAP
OP_ENDIF
OP_1SUB OP_DUP <0> OP_GREATERTHANOREQUAL OP_IF
    OP_SWAP
    <2> OP_UTXOVALUE
    OP_ADD
    OP_SWAP
OP_ENDIF
OP_1SUB OP_DUP <0> OP_GREATERTHANOREQUAL OP_IF
    OP_SWAP
    <3> OP_UTXOVALUE
    OP_ADD
    OP_SWAP
OP_ENDIF
OP_DROP // drop remaining index, leaving aggregated value on top

```

With bounded loops, this aggregation can be simplified to:

```
<0> <0>
OP_BEGIN                            // Loop (re)starts with index, value
  OP_OVER OP_UTXOVALUE OP_ADD       // Add UTXO value at index to total
  OP_SWAP OP_1ADD OP_SWAP           // Increment index
  OP_OVER OP_TXINPUTCOUNT OP_EQUAL
OP_UNTIL                            // Loop until index equals the TX input count
OP_NIP                              // Drop index, leaving sum of UTXO values
```

## Test Vectors

This proposal includes [a suite of functional tests and benchmarks](./vmb_tests/) to verify the performance of all operations within virtual machine implementations.

## Implementations

Please see the following implementations for additional examples and test vectors:

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1937](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1937).
- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next).

## Feedback & Reviews

- [Loops CHIP Issues](https://github.com/bitjson/bch-loops/issues)
- [`CHIP 2021-05 Bounded Looping Operations` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2021-05-bounded-looping-operations/463)

## Changelog

This section summarizes the evolution of this proposal.

- **v1.2.3 – 2025-09-05** ([`bd3ebc76`](https://github.com/bitjson/bch-loops/commit/bd3ebc76bfa2190255fd3b5fcd84ec8910652f4e) – [diff vs. `master`](https://github.com/bitjson/bch-loops/compare/master...bd3ebc76bfa2190255fd3b5fcd84ec8910652f4e))
  - Revert "Reduce Base Instruction Cost" ([#8](https://github.com/bitjson/bch-loops/issues/8))
  - Update VMB tests and benchmarks
- **v1.2.2 – 2025-06-30** ([`9c1f55d9`](https://github.com/bitjson/bch-loops/commit/9c1f55d9625e210c9c6eff113c3fa34049b586a5))
  - Reduce Base Instruction Cost
- **v1.2.1 – 2025-05-02** ([`e0af77e9`](https://github.com/bitjson/bch-loops/commit/e0af77e951ad27cb5158cb86ad0ad60a76f336e7))
  - Clarify descriptions
  - Commit latest test vectors
  - Link to BCHN implementation
- **v1.2.0 – 2024-12-18** ([`b358b5a3`](https://github.com/bitjson/bch-loops/commit/b358b5a3bccd2ae2eac4d941f387e8cb969ec018))
  - Re-propose for 2026
  - Eliminate `Repeated Bytes` counter ([#5](https://github.com/bitjson/bch-loops/issues/5))
- **v1.1.0 – 2021-06-10** ([`d764bd43`](https://github.com/bitjson/bch-loops/commit/d764bd43eafc59fe1f7602adb518b980fbe619d4))
  - Correct `OP_UNTIL` to complete on truthy values
- **v1.0.0 – 2021-05-27** ([`056aec90`](https://github.com/bitjson/bch-loops/commit/056aec90ea5afdb8115f8ad4dc692392e1f0aa87))
  - Initial publication

## Copyright

This document is placed in the public domain.
