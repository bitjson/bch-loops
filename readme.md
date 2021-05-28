# CHIP-2021-05-loops: Bounded Looping Operations

        Title: Bounded Looping Operations
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2021-05-28
        Latest Revision Date: 2021-05-28

## Summary

This proposal includes two new BCH virtual machine (VM) operations, `OP_BEGIN` and `OP_UNTIL`, which enable a variety of loop constructions in BCH contracts without increasing the processing or memory requirements of the VM.

## Deployment

Deployment of this specification is proposed for the May 2022 upgrade.

This proposal requires [`CHIP: Targeted Virtual Machine Limits`](https://github.com/bitjson/bch-vm-limits), or equivalent limits on the processing and memory usage of the VM.

## Motivation

The Bitcoin Cash VM is strictly limited to prevent maliciously-designed transactions from requiring excessive resources during transaction validation.

Loops were originally excluded from this design on the mistaken assumption that they risk consuming excessive validation resources. In reality, a small number of VM operations make up [the majority of resource usage during transaction validation](https://github.com/bitjson/bch-vm-limits#benchmarks), and any implementation of bounded loops can be many orders of magnitude faster than the most pathological VM bytecode constructions.

Introducing this basic control flow structure would make BCH contracts significantly more powerful and efficient.

## Benefits

Looping operations enable many use cases [where specifying a precise procedure using `OP_IF` is impractical or impossible](#aggregation), and they also significantly [reduce the bytecode length of many contracts with repeated procedures](#simplifying-repeated-procedures).

## Costs & Risk Mitigation

The following costs and risks have been assessed.

### Modification to Transaction Validation

Without adequate limits, looping operations have the potential to increase worst-case transaction validation costs and expose VM implementations to Denial of Service (DOS) attacks.

**Mitigations**: this proposal includes strict limits which ensure loops do not increase the maximum executable bytecode length (`MAX_SCRIPT_SIZE`, 10,000 bytes). This altogether avoids increasing the available surface area for Denial of Service (DOS) attacks.

### Node Upgrade Costs

This proposal affects consensus – all fully-validating node software must implement these VM changes to remain in consensus.

These VM changes are backwards-compatible: all past and currently-possible transactions remain valid under these new rules.

### Ecosystem Upgrade Costs

Because this proposal only affects internals of the VM, standard wallets, block explorers, and other services will not require software modifications for these changes. Only software which offers emulation of VM evaluation (e.g. [Bitauth IDE](https://github.com/bitauth/bitauth-ide)) will be required to upgrade.

Wallets and other services may also upgrade to add support for new contracts which will become possible after deployment of this proposal.

## Technical Specification

Two new VM operations are added: `OP_BEGIN` (`0x65`/`101`) and `OP_UNTIL` (`0x66`/`102`).

> Note: these opcodes were called `OP_VERIF` (`0x65`) and `OP_VERNOTIF` (`0x66`) in early versions of Bitcoin, but were disabled when it was realized that they created a fork-risk between different versions of the Bitcoin software. Both operations currently render the transaction invalid (even when found in an unexecuted OP_IF branch), so they are effectively equivalent to any codepoints in the undefined ranges. They are particularly well-suited for this application because they are the only two available, adjacent codepoints within the control-flow range of operations.

To support these operations, the control stack is modified to support integer values, and a new unsigned integer counter `Repeated Bytes` is added to the VM state.

### Control Stack

This proposal modifies the control stack (A.K.A. `ExecStack` or `ConditionStack`) to support integer values. When testing for operation execution, integer values must be ignored.

> Prior to this proposal, the control stack (A.K.A. `ExecStack` or `ConditionStack`) is conceptually represented as an array of boolean values: when a flow control operation (like `OP_IF`) is encountered, a `true` is pushed to the stack if the branch is to be executed, and `false` if not. (`OP_ELSE` toggles the top value on this stack.) Non-flow control operations are only evaluated if the top of the stack is `true`.

### Repeated Bytes

A new unsigned integer counter, `Repeated Bytes`, is added to the VM state, initialized to `0` at the beginning of evaluation.

> `Repeated Bytes` is only used by `OP_UNTIL` to prevent excessive use of loops.

### `OP_BEGIN` (`0x65`/`101`)

`OP_BEGIN` pushes the current instruction pointer index to the control stack (A.K.A. `ConditionStack`).

> This operation sets a pointer to which the evaluation will return when the matching `OP_UNTIL` is encountered. If a matching `OP_UNTIL` is never encountered, the evaluation returns an error.

### `OP_UNTIL` (`0x66`/`102`)

`OP_UNTIL` pops the top item from the control stack:

- if this control value is not an integer, error.
- The difference between the current instruction pointer and the control value is added to `Repeated Bytes`; if `Repeated Bytes` + `script.size()` is greater than `MAX_SCRIPT_SIZE`, error.
- The top item is popped from the stack, if the value is truthy (by the same test as `OP_IF`) the instruction pointer is set to the `OP_BEGIN` instruction (otherwise, evaluation continues past the `OP_UNTIL`).

> `OP_UNTIL` has two possible error conditions: 1) a mismatched `OP_BEGIN` (either `OP_BEGIN` is missing or the loop contains an `OP_IF` without a matching `OP_ENDIF`), or 2) executing the loop would cause the evaluation to exceed `MAX_SCRIPT_SIZE`. If neither of these occur, evaluation returns to the `OP_BEGIN`, and that instruction pointer index is again pushed to the control stack.

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
OP_CAT OP_HASH160

OP_FROMALTSTACK // check if tier 2 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH160

OP_FROMALTSTACK // check if tier 3 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH160

OP_FROMALTSTACK // check if tier 4 requires swap
OP_IF OP_SWAP OP_ENDIF
OP_CAT OP_HASH160

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
  OP_CAT OP_HASH160
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
OP_DUP <4> OP_LESSTHANOREQUAL OP_VERIFY // prevent malleability
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
<1> <0>
OP_BEGIN                                  // loop (re)starts with index, value
  OP_OVER OP_UTXOVALUE OP_ADD             // Add UTXO value to total
  OP_OVER <4> OP_NUMNOTEQUAL              // loop until index equals input count
OP_UNTIL
OP_NIP                                    // drop index after loop is complete
```

## Implementations

Please see the following reference implementations for additional examples and test vectors:

[TODO: after initial public feedback]

## Stakeholders & Statements

[TODO]

## Feedback & Reviews

[TODO]

## Copyright

This document is placed in the public domain.
