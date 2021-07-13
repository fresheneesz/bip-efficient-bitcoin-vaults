```
BIP: BIP-PUSHOUTPUTSTACK
Layer: Consensus (soft fork)
Title: PUSHOUTPUTSTACK
Author: Billy Tetrud <bitetrudpublic@gmail.com>
Looking for co-authors.
Comments-Summary: TBD
Comments-URI: TBD
Status: Draft
Type: Standards Track
Created: 2020-06-06
License: BSD-3-Clause: OSI-approved BSD 3-clause license
```

##  Introduction

### Abstract

This BIP proposes a new tapscript opcode, OP_PUSHOUTPUTSTACK, which allows an input's script to pass data onto the script for particular outputs. The extension has applications for efficient bitcoin vaults, among other things. 

This could either be activated using a tapscript OP_SUCCESSx opcode or less efficiently as a traditional OP_NOPx opcode.

### Motivation

#### Better Wallet Vaults

* No pre-signed transactions necessary.
* Fixes the security hole in OP_CTV vaults that allows an attacker to steal some funds that the vault owner is sending out of the vault. 

The primary motivation for this opcode is to create efficient wallet vaults. This allows a parent output constrained with a covenant to pass information from its witness to a child output. This makes it possible to both constrain a parent output with a covenant and at the same time put requirements on the child output that weren't known on creation of the parent output. This allows, for example, the destination to be passed to the covenant parent output in its witness and then require the child output to be spendable by that destination. See the *Motivation* section in the [parent BIP](README.md) for details and larger context. 

## Specification

### Option A

OP_PUSHOUTPUTSTACK (*OP_POS for short*) redefines opcode OP_SUCCESS_81 (0x51). It pushes items onto the "output stack" for outputs to a particular address. The "output stack" is a stack of data that will be pushed onto the stack after the witness script runs, but before the primary script runs. This allows a script writer to constrain behavior of a chained transaction output with witness data that was used to unlock a covenant input script.

It does the following:

1. Pops two values from the stack and interprets them as:
   * Top: `address`
   * Top - 1: `numberOfValues`
2. If there are no outputs sent to the `address`, the operation is complete. Otherwise, go to the next step.
3. It then pops `numberOfValues+1` values from the stack and interprets them as:
   * Top: `outputIndex`
   * Top - 1 - n: `valueN` where `n` is the zero-based index of a list of values.
4. For the output identified by `outputIndex`, the set of values that follow the `outputIndex` are pushed on a new conceptual stack called the "output stack" for that output/input pair. 
5. If there is more than one output to the `address`, loop back to step 3 for each additional output.
6. When the transaction is validated, the output's output-stacks for each input are compared to ensure they are identical. If not, the transaction fails.
7. Upon evaluation of a UTXO as an input to a subsequent transaction, after the witness is evaluated, the output stack associated with one of those inputs (it is arbitrary since they have all been validated to be identical) is pushed onto the execution stack such that the last item on the output stack becomes the first item on the stack (in other words, the output stack values pushed onto the execution stack will be in the same order as the output stack). 

Reversion modes. For all the following situations, the opcode reverts to its OP_SUCCESS semantics:

* The `numberOfValues` is not a positive number.

Failure modes. For all the following situations, the transaction is marked invalid:

1. If being executed in anything other than the scriptPubKey (eg in the scriptSig or witness script).
2. The number of values on the stack is less than 2.
3. If `numberOfValues` is >= 0 and the number of values on the stack after `numberOfValues` is popped is less than `numberOfOutputsForAddress * (numberOfValues+1)`.
4. Any `outputIndex` does not correspond to an output in the transaction.
5. The output corresponding to a given `outputIndex` is not being sent to the given `address`.
6. Any output sent to the given `address` isn't listed as an `outputIndex` given values in this operation. All outputs to the given `address` must be given values.
7. If two different inputs push values on an output stack for the same output and after both input scripts are evaluated, the output stacks for that output for each input script are not identical.

### Option B

OP_PUSHOUTPUTSTACK (*OP_POS for short*) redefines opcode OP_NOP6 (0xb5). It does the same as the tapscript version above except that it does not pop stack items and instead of adding the output stack onto the child output's execution stack, it instead requires the `scriptSig` spending the relevant output of the transaction to have particular values on the top of its stack after evaluation. When that UTXO is spent, after evaluation of the UTXO's `scriptSig`, the top three values on the stack are verified to match the "output stack" stored with the UTXO. If they don't match, the transaction is marked invalid.

## Rationale

These opcodes provide a way to pass data from the witness used to spend an input to the created output. In the context of wallet vaults, this allows the wallet vault to be created without knowing the destination while also allowing a transaction spending a wallet vault output to commit to a destination with a single transactions. Using just OP_CTV by contrast, the destination is not committed to when sending coins from the vault, and a second transaction must be made that chooses the destination. 

The opcode requires an address rather than just outputIds so that a script can ensure that particular data ends up being added for particular addresses that have expected scripts. OP_POS requires that data is pushed onto the output stack for all outputs with a given address, because there should be no possibility that the user (or an attacker) can manipulate what data ends up in those locations on the stack. If the transaction creator could choose which outputs to add the data for, the user could choose to omit certain outputs and add their own data in those stack locations which could be used to steal funds from the output.

## Design Tradeoffs and Risks

#### Edge-case foot gun

Because OP_POS is designed to put data on the execution stack before the scriptPubKey executes, in cases where the script usually expects two different addresses and pushes separate data using OP_POS, if two identical addresses are used instead, this can add extra things onto the stack and potentially cause the output to become unspendable or spendable by an unintended party. If the witness can be set to cause this scenario, the script should verify that the addresses are different. 

#### Abuse of Output Stack Data Generation

Once a transaction using OP_POS has been evaluated, it needs to be stored in some form in the UTXO set so nodes can correctly initialize the output stack of the relevant UTXO.

One could imagine storing the output stack directly, however its possible for someone to abuse this opcode to multiply the number of items pushed to output stacks. A rough calculation of how much data could be pushed onto the output stack, given a 9-byte minimum output size, if say 100kb were used for outputs and the rest was used to write a 100kb of data to the output stacks, this would over 5,500 outputs times 49kb, which would be 1.1 GB of data. Clearly, this isn't something we want nodes storing for a single transaction. By using multiple transactions in a row that use OP_POS, even higher amounts of data could be generated. This could be optimized in cases where the output stacks for all outputs are identical, but there is a risk that malicious users could abuse whatever optimizations are made to similar effect. 

One way to mitigate this issue is to instead re-evaluate the parent output script to regenerate the output stack before use in the subsequent transaction. That way only the parent output needs to be stored, rather than the data generated. The transaction need not be fully evaluated either the second time around. Any opcodes that verify things and don't add anything onto the stack could simply avoid most of their evaluation steps, for example. 

Another way to mitigate this is to limit how much data can be pushed to output stacks in total. The limit could be set to something like 10kb. A third way to mitigate this is to add this output stack data into the calculation of a transaction's vbytes so that extra effort used in building the output stacks from various inputs would be paid for. 

#### Polluting an output's stack

In cases where an output's script is not designed to use additional output stack values or the particular set of output stack values given to it, this could render the output unspendable or spendable by someone other than the agreed upon recipient. This could be a problem in cases where software isn't built to recognize this error (and instead tells the user they have been paid) or in cases where a sender is actively trying to defraud the recipient. In the second case, the attack might go as follows:

1. The attacker agrees to a transaction with the victim.
2. The victim sends the attacker an output.
3. The attacker recognizes the output as exploitable via this attack and sends a transaction that results in adding values on the output stack that result in the output being spendable by the attack, or unspendable if the attacker is simply trying to grief the victim. 
4. The victim may be alerted by their software that the transaction has an unrecognizable or unexpected output stack, and ask the attacker about it.
5. The attacker may claim they're using some new technology or that this error is normal and expected. They might have even notified the victim up front that this warning might be shown to them. 
6. In some cases, the victim might be tricked and deliver the goods despite the warning only to find later they can't spend the output, or that it has already been spent (presumably by the attacker).

This could be mitigated slightly by requiring a companion opcode that might hypothetically be named OP_LOADOUTPUTSTACK which can be run to put the output stack values on the execution stack. That way, any script that doesn't expect output stack values at all can cleanly avoid this risk/complication. However, this wouldn't work for addresses whose scripts do expect an output stack, in which case the problem still would exist for those kinds of addresses.

Individual scripts could mitigate this by having some kind of validation of the output stack values, and if those values don't validate, separate spend paths could be used. However, this would bloat each script and might not be possible in many cases or may only cover a fraction of possible output stack pollution.

This proposal recommends that software clearly warn users of this problem in relevant cases so an attacker has a low likelihood of tricking them in a case where this can happen. It seems likely to be rare that an attacker could exploit this to steal money, however the griefing possibility is a lot more likely to be possible, if users ignore their software's warnings or if the software omits warnings in this case.

#### Links inputs and outputs together

Because this opcode requires specifying specific output IDs, that ties those outputs to the relevant input. This might be undesirable in coinjoins or similar constructions that seek to obfuscate the link between sources and destinations.

#### Individually specifying all output IDs

Theoretically, it could be useful to allow a mechanism to push output stack data for all outputs to a given address (without explicitly specifying them). However, if you allow a mechanism to push values on all outputs for a given address, its possible that two vault inputs will conflict by pushing different values on the output stack for the same output. Such a mechanism would increase the likelihood that unrelated outputs could not be used together in the same transaction. That's why this ability was not included in this opcode.

#### Output stack data is dependent on the destination

There is no mechanism to ensure that output data written to a transaction will result in an output that can be spent. This does mean that improperly written scripts could accidentally burn coins.

## Copyright

This document is licensed under the 3-clause BSD license.