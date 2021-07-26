```
BIP: BIP-CONSTRAINDESTINATION
Layer: Consensus (soft fork)
Title: CONSTRAINDESTINATION
Author: Billy Tetrud <bitetrudpublic@gmail.com>
Looking for co-authors.
Comments-Summary: TBD
Comments-URI: TBD
Status: Draft
Type: Standards Track
Created: 2020-06-06
License: BSD-3-Clause: OSI-approved BSD 3-clause license
```

- [Introduction](#introduction)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
    + [Better Wallet Vaults](#better-wallet-vaults)
    + [Congestion Controlled Transactions](#congestion-controlled-transactions)
    + [Increased Payment Channel Routes](#increased-payment-channel-routes)
- [Design](#design)
- [Specification](#specification)
  + [Opcode Example Call](#opcode-example-call)
- [Rationale](#rationale)
- [Detailed Specification](#detailed-specification)
- [Design Tradeoffs and Risks](#design-tradeoffs-and-risks)
    + [Fungibility Risks with OP_CD](#fungibility-risks-with-op_cd)
    + [Combination of OP_CD UTXOs DOS Vector](#combination-of-op_cd-utxos-dos-vector)
    + [Feature Redundancy](#feature-redundancy)

##  Introduction

### Abstract

This BIP proposes a new tapscript opcode, OP_CONSTRAINDESTINATION, which can be used to constrain the destination the output may be spent to. This involves both specifying particular addresses the output is allowed to send coins to, as well as constraining the amount of the fee that output is allowed to contribute to. The extension has applications for efficient bitcoin vaults, among other things. 

This could be activated using a tapscript OP_SUCCESSx opcode.

### Motivation

* Covenants that are more flexible than OP_CTV. OP_CD covenant outputs can be used as inputs in any combination and spent to any combination of outputs (vs OP_CTV vaults where each output must be spent in its own separate transaction with no other inputs and only a single list of specific outputs).
* Wallet vaults. 
* Congestion controlled transactions.
* Non-interactive payment channels.

#### Better Wallet Vaults

* Far more flexible than OP_CTV vaults. Outputs can be spent in a transaction with any other outputs.

The primary motivation for this opcode is for wallet vaults. See [this example wallet vault](op_cdWalletVault1.md) that uses this opcode. See also the *Motivation* section in the [parent BIP](../README.md) for broader context (involving other proposed opcodes).

#### Congestion Controlled Transactions

In times of mempool congestion, certain payers might be in a situation where they need to commit to transactions now, but their receivers might be able to wait to have spending access to the funds until after the congestion passes. In such a situation, it could be advantageous for the payers to create a smaller transaction that commits to sending funds to various addresses at a later date. 

For example, if the payer wants to create a batch transaction with 3 inputs and 20 outputs, the payer could instead create a transaction with the same 3 inputs but only a single output that uses a covenant (eg using OP_CD) to ensure the funds in that output can only be spent to the 20 addresses. The payer would then broadcast the smaller transaction during congestion with a higher fee-rate (but lower fee), then once the congestion has passed, would spend the created output to send coins to the 20 originally intended addresses. If done correctly, this can generally result in a lower total fee among the two transactions. 

This could save payers money, but would end up being larger on the blockchain, so there is a downside cost felt from a set of transactions like this. Its also unclear how often this would be useful. However, there is more information about this (including simulations) [in the material about OP_CTV](https://utxos.org/uses/scaling/) and the material about [OP_SECURETHEBAG](http://diyhpl.us/wiki/transcripts/scalingbitcoin/tel-aviv-2019/bip-securethebag/).

#### Increased Payment Channel Routes

Since the number of HTLCs is limited to 483 in order to keep the transaction from becoming too large to be valid, covenant transactions can be used to expand the number of HTLCs. [More details here](https://utxos.org/uses/htlcs/)

## Design

TBD

## Specification

OP_CONSTRAINDESTINATION (*OP_CD for short*) redefines opcode OP_SUCCESS_82 (0x52). The opcode can be used to limit the destinations that and fee that a input can contribute to. It does the following: 

1. The top item on the stack is popped and interpreted as a `numberOfOutputs`.
2. The next few items on the stack, numbering `2 * numberOfOutputs`, are popped and interpreted as the map `outputValues` which maps output IDs to an amount of bitcoin sent to that address from the UTXO.  For each pair, the output ID appears first, followed by the amount of bitcoin. 
3. The next item on the stack is popped and interpreted as a `numberOfAddresses`.
4. The next few items on the stack, numbering `numberOfAddresses`, are popped and interpreted as the list `addressList`.
5. The next item popped from the stack is interpreted as the positive integer `sampleWindowFactor`. This determines the sample window used to calculate the recent median fee. The window is described as a number of blocks:
   * If `sampleWindowFactor` is 0, the `windowLength` is 6 blocks.
   * If `sampleWindowFactor` is 1, the `windowLength` is 30 blocks.
   * If `sampleWindowFactor` is 2, the `windowLength` is 300 blocks.
   * If `sampleWindowFactor` is 3, the `windowLength` is 3000 blocks.
6. The next item popped from the stack is interpreted as an integer `feeFactor` (which can be positive of negative). If the number is -1, it is interpreted as -infinity (meaning the transaction can contribute nothing to the fee). 
7. A `blockMedianFeeRate` is defined as the median fee-rate per vbyte for a given block. The `averageFeeRate` is defined as the average of `blockMedianFeeRate` for each block in the most recent `windowLength` blocks. The `maxFeeContribution` is defined as `averageFeeRate * 2^feeFactor` of the fee. Note that this is a limitation on the fee, not on the fee-rate. If `feeFactor` is -1, `maxFeeContribution` is 0.
8. Once all input scripts have been evaluated, for each output, verify that the amount of bitcoin claimed to have been contributed to that output in all OP_CD calls (from all input) sums to an amount smaller than the that output's value. 

Reversion modes. For all the following situations, the opcode reverts to its OP_SUCCESS semantics:

1. `numberOfOutputs` is less than 0.
2. `numberOfAddresses` is less than 0.
3. `sampleWindowFactor` is anything but a number 0 through 3.
4. `feeFactor` is less than -1 or more than 16.

Failure modes. For all the following additional situations, the transaction is marked invalid:
1. If the number of values on the stack after `numberOfAddresses` is popped is less than `numberOfAddresses`.
2. If any address in `addressList` are invalid.
3. If any output ID in the `outputValues` maps to an output whose destination isn't in `addressList`.
4. If any output ID in the `outputValues` does not correspond to an output in the transaction.
5. If the sum of bitcoin amount values in `outputValues` is greater than the UTXO's value.
6. If the value of the output minus the sum of all bitcoin amounts in `outputValues` is greater than `maxFeeContribution`.
7. If the sum of all given OP_CD contributions (from all inputs) for an output is greater than the value of the output.

### Opcode Example Call

Given a 300-block `averageFeeRate` of 10 sats/byte, the following is a script that limits the input to being spent to only address `D` and address `C` in outputs 0 (for 24345200 satoshi), 1 (for 329435400 satoshi), and 2 (for 12345 satoshi) with a maximum fee of `10 * 2^5= 320 sats`.

```
5
2
C
D
2
12345
2
329435400
1
24345200
0
3
OP_CONSTRAINDESTINATION 
```

For some walk-throughs of example transactions, take a look [here](op-cd-examples.md).

## Rationale

These opcodes provide a way to ensure that individual inputs transfer their value to specified addresses, allowing them to send to P2SH and related addresses that have spending constraints. This allows for covenants and prescribed sequences of transactions. 

The ability to limit the fee that an input can contribute sets limits to the amount of funds an attacker can drain from a victim's bitcoin vault. The fee is proportional to a calculation of recent median fee-rates so that a transaction output's maximum fee can scale as the network changes. If median fee rates rise, the owner still wants the ability to place a reasonable fee on the transaction so its spendable in a reasonable amount of time. If the median fee rates go down, the limit also scales down to limit the damage a griefer could do. The assumption is that median fees will scale primarily with change in purchasing power of bitcoin and not with changing network congestion (which might offer an attacker the opportunity to perform a more damaging grief attack).

The ability to set the `sampleWindowFactor` to options from 6 blocks to 3000 blocks is to give the owner the ability to choose, basically, how much of a rush they think they might be in. If they are generally going to want transactions to go through quickly during network congestion spikes, they'll want a shorter sample-window so they can set their fee higher. However, this also allows a griefer to set a higher fee during network congestion spikes. A longer sample window would be a more accurate measure of general fee rates, but the limit might not be high enough to allow fast confirmation times during congested network conditions. 

Using a `feeFactor` to specify multiples of powers of two for the fee contribution limit assumes that finer-grained fee limit specification is unnecessary. The feeFactor is a two's compliment number so it can be negative in order to specify fractional multiples if they want to for some reason. 

Note also that this design solves the half-spend problem without restricting things like the number of inputs (as op_ctv does). The half-spend problem is a potential problem with covenants where certain types of constraints have loopholes if multiple outputs with the same constraints are used. For example, if you had a constraint saying "the transaction spending this output must send at least 1 bitcoin to address A", if you spent the outputs individually address A would receive at least 1 bitcoin for each transaction. However, if you create a transaction using multiple outputs with that same constraint, the constraint would be met by sending 1 bitcoin to address A, even if dozens of outputs are used. This would mean an attacker could potentially either steal funds, or maybe just grief the victim by spending those bitcoins as fees. Either way, not ideal. OP_CD solves the half-spend problem by ensuring that the constraints of all outputs are summed up and validated. There is no combination of outputs where the intuitive meaning of the OP_CD constrains would be violated.

OP_CD is defined as a check-and-verify opcode rather than pushing 1 or 0 on the stack because that would conceptually allow the execution of an output's script to be affected by the scripts of other outputs. That would add complexity that is almost certainly unacceptable. As currently defined, the interactions between outputs that OP_CD allows can only result in the transaction being declared invalid, rather than changing any execution behavior of the scripts. 

The `averageFeeRate` is used to calculate the maximum fee contribution of an output rather than the median fee so that a change in the average bytes-per-transaction over time doesn't skew the calculations of fee limits.

The opcode has several cases where it reverts to OP_SUCCESSx semantics (or OP_NOPx semantics in the case of the traditional script version). This is done rather than defining it as failure so that future soft-forks can redefine these semantics.

## Detailed Specification

TBD

## Design Tradeoffs and Risks

#### Specifying values sent to each output

OP_CD requires specifying each output and the amount sent to the output in order to make it simple and efficient to calculate validity of OP_CD across multiple inputs that might share OP_CD destinations. Validity could be calculated without this specification, but it would require a potentially very large number of combinations being checked. Without specifying the amounts the UTXO will contribute for each output, a DOS attack vector is opened up where the attacker may create an intentionally difficult-to-verify transaction using many outputs each using OP_CD. An attacker might construct transactions with many inputs that share potential OP_CD destinations in different combinations. If there are enough combinations that must be checked, this might substantially slow down validation and open a DOS vector for attackers. 

There are other mitigations to this (limit the number of combinations that need to be checked, or increase the vbyte weight of transactions with many combinations that need to be checked), however these mitigations either limit the use of the opcode or are complex and imperfect mitigations. Requiring the user to specify the solution the validation equation makes validation simpler and safer, at the expense of slightly larger transactions. 

####  Fungibility Risks with OP_CD

With OP_CD, there is the possibility of creating covenants with unbounded chains of constraints. This could be used to put permanent restrictions on particular coins. However, its already possible to permanently restrict coins. A multisig setup could be created where the coins can only be spent if a particular key approves. It seems there is a [reasonable amount of community consensus](https://www.mail-archive.com/bitcoin-dev@lists.linuxfoundation.org/msg10222.html) that these fungibility issues aren't much of a concern. Restricting its use can generally only reduce the value of the coins, so at worst its similar to provably burning coins. 

#### Flexible output value claims

As specified, the output-value claims in an OP_CD call do not have to add up to the actual value sent to that output. For example, a transaction with a single input that has OP_CD claim to send 100 sats to an output, while the output actually receives 150 sats. This can still be valid as the claim doesn't (or sum of claims for an output don't) *exceed* the output value. If this flexibility is seen as a problem, it could be fixed. But it seems like a relatively safe thing to allow.

#### Theft via fees

The purpose of the fee limit is to limit the risk of griefing attacks and theft via fees. However, this doesn't fully prevent fee abuse. For example, a miner could purchase some goods with a congestion controlled transaction, and then choose a time during high fees to send the transaction with as high a fee as possible. OP_CD carries the risk that some actors will play dirty like this and extract higher than expected fees in multi-party contracts like this.

Single-party use cases (eg wallet vaults) don't carry much risk of this, since the user is sending to themselves. However, if a wallet vault's key is stolen, an attack may use this approach to extract some value from their victim via fees. 

#### Manipulation of the average fee rate

Miners could potentially manipulate the fee rate, for example by sending themselves payments with high fees. This could allow those miners (or those complicit with them) to extract more value from a victim via fees. Doing this would be costly for miners tho, and recouping the cost would require a reasonably large attack (at least hundreds of transactions from victims' wallets for a 6 block sample window, and on the order of hundreds of thousands of transactions for a 3000 block sample window).