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
  * [Option A - Tapscript Opcode](#option-a---tapscript-opcode)
    + [Opcode Example Call](#opcode-example-call)
    + [Pseudocode Examples](#pseudocode-examples)
  * [Option B - Traditional Script Opcode](#option-b---traditional-script-opcode)
- [Rationale](#rationale)
- [Detailed Specification](#detailed-specification)
- [Design Tradeoffs and Risks](#design-tradeoffs-and-risks)
    + [Fungibility Risks with OP_CD](#fungibility-risks-with-op_cd)
    + [Combination of OP_CD UTXOs DOS Vector](#combination-of-op_cd-utxos-dos-vector)
    + [Feature Redundancy](#feature-redundancy)

##  Introduction

### Abstract

This BIP proposes a new tapscript opcode, OP_CONSTRAINDESTINATION. The extension has applications for efficient bitcoin vaults, among other things. This could either be activated using a tapscript OP_SUCCESSx opcode or less efficiently as a traditional OP_NOPx opcode.

### Motivation

#### Better Wallet Vaults

The primary motivation for this opcode is for wallet vaults. See the *Motivation* section in the [parent BIP](Script Parameter, op_scriptparam,  OP_CHECKSCRIPTVERIFY, and  OP_BEFORESEQUENCEVERIFY.md).

#### Congestion Controlled Transactions

In times of mempool congestion, certain payers might be in a situation where they need to commit to transactions now, but their receivers might be able to wait to have spending access to the funds until after the congestion passes. In such a situation, it could be advantageous for the payers to create a smaller transaction that commits to sending funds to various addresses at a later date. 

For example, if the payer wants to create a batch transaction with 3 inputs and 20 outputs, the payer could instead create a transaction with the same 3 inputs but only a single output that uses a covenant (eg using OP_CD) to ensure the funds in that output can only be spent to the 20 addresses. The payer would then broadcast the smaller transaction during congestion with a higher fee-rate (but lower fee), then once the congestion has passed, would spend the created output to send coins to the 20 originally intended addresses. If done correctly, this can generally result in a lower total fee among the two transactions. 

This could save payers money, but would end up being larger on the blockchain, so there is a downside cost felt from a set of transactions like this. Its also unclear how often this would be useful. However, there is more information about this (including simulations) [in the material about OP_CTV](https://utxos.org/uses/scaling/) and the material about [OP_SECURETHEBAG](http://diyhpl.us/wiki/transcripts/scalingbitcoin/tel-aviv-2019/bip-securethebag/).

#### Increased Payment Channel Routes

Since the number of HTLCs is limited to 483 in order to keep the transaction from becoming too large to be valid, covenant transactions can be used to expand the number of HTLCs. (More details here](https://utxos.org/uses/htlcs/)

## Design

TBD

## Specification

### Option A - Tapscript Opcode

OP_CONSTRAINDESTINATION (*OP_CD for short*) redefines opcode OP_SUCCESS_82 (0x52). The opcode can be used to limit the destinations that and fee that a input can contribute to. It does the following: 

1. The item on the top of the stack is popped and interpreted as a `numberOfAddresses`.
2. The next few items on the stack, numbering `numberOfAddresses`, are popped and interpreted as the list `addressList`.
3. The next item popped from the stack is interpreted as the positive integer `sampleWindowFactor`. This determines the sample window used to calculate the recent median fee. The window is described as a number of blocks:
   * If `sampleWindowFactor` is 0, the `windowLength` is 6 blocks.
   * If `sampleWindowFactor` is 1, the `windowLength` is 30 blocks.
   * If `sampleWindowFactor` is 2, the `windowLength` is 300 blocks.
   * If `sampleWindowFactor` is 3, the `windowLength` is 3000 blocks.
4. The next item popped from the stack is interpreted as an integer `feeFactor` (which can be positive of negative). If the number is -1, it is interpreted as -infinity (meaning the transaction can contribute nothing to the fee). 
5. The `medianFeeRate` is defined as the median fee rate per vbyte for the most recent `windowLength` blocks.
6. The input can contribute no more than `medianFeeRate * 2^feeFactor` of the fee, or the transaction will be marked invalid. Note that this is a limitation on the fee, not on the fee-rate. If `feeFactor` is -1, the input cannot contribute to the fee at all. For each relevant set of inputs constrained by OP_CD, ensure that `sum(inputs) - sum(constrainedOutputs) < sum(maxFeesForEachInput)`, otherwise mark the transaction as invalid. This check does not need to be performed on (irrelevant) sets of inputs where some group of those inputs shares no output address constraints (for outputs that exist in the transaction) with another non-overlapping group within that set. There is likely an optimal way to minimize the number of calculations that must be done for this, but that's left as future work.
7. Reversion modes. For all the following situations, the opcode reverts to its OP_SUCCESS semantics:
   1. `numberOfAddresses` is less than 0.
   2. `sampleWindowFactor` is anything but a number 0 through 3.
   3. `feeFactor` is less than -1 or more than 16.
8. Failure modes. For all the following additional situations, the transaction is marked invalid:
   1. If the number of values on the stack after `numberOfAddresses` is popped is less than `numberOfAddresses`.
   2. If any address in `addressList` are invalid.

#### Opcode Example Call

Given a median 300-block fee-rate of 10 sats/byte, the following is a script that limits the input to being spent to only address `D` and address `C` with a maximum fee of `10 * 2^5= 320 sats`.

```
5
2
C
D
2
OP_CONSTRAINDESTINATION 
```

#### Pseudocode Examples

For the following examples, I'll write this information as `OP_CD(sampleWindowFactor, feeFactor, [Address1, Address2,...])`. For example, the above example script would be written as `OP_CD(300 blocks, 7, [D, C])`. I'll also use 10 sats/byte as the 300-block median fee rate.

Example Transaction A:

* Inputs:
  * One `input` of 1000 satoshi with `OP_CD(300 blocks, 5, [D, C])`
* Outputs:
  * 100 sat to destination address `D`
  * 850 sat to change address `C`

The `input` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. Since there is only one input, that means the total transaction fee can be no more than 320 sats. The input specifies that it can only send money to addresses `D` and `C`. Since the transaction is sending a total of 950 sat to those addresses, the fee is 50 sats, which is less than 320, so the transaction is valid. 

Example Transaction B:

* Inputs:
  * `inputA` with a value of 2000 satoshi.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 5, [D2, C2])`.
* Outputs:
  * 200 sats to destination address `D1`
  * 1400 sats to change address `C1`
  * 100 sats to destination address `D2`
  * 550 sats to change address `C2`

`inputB` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. The transaction has a total fee of 700 sats. `inputB` specifies that it can only send money to addresses `D2` and `C2`. Since those two addresses would receive a total of 650 sats, the fee contributed by `inputB` is 350 sats, which is more than 320, so the transaction is invalid.

Example Transaction C:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_CD(300 blocks, 5, [D1, C1])`.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 4, [D1])`.
* Outputs:
  * 900 sats to destination address `D1`.

`inputA` can contribute a maximum of `10* 2^5 = 320 sats` to the fee. `inputA`'s minimum contribution to the fee is `inputA - D1 = 1200 - 900 = 300 sats`, which is less than 320 sats, and so the immediate evaluation of the operation succeeds.

`inputB` can contribute a maximum of `10 * 2^4 = 160 sats` to the fee. `inputB`'s minimum contribution to the fee is `inputB - D1 = 1000 - 900 = 100 sats`, which is less than 160 sats and so the immediate evaluation of the operation succeeds.

Once both scripts run, the last step is to check the sum of the inputs, outputs, and fee limits for all inputs that use OP_CONSTRAINDESTINATION. `inputA + inputB - D1 = 1200 sats + 1000 sats - 900 sats = 1300 sats`, which is larger than the sum of the maximum fees of `320 + 160 =  480 sats`, so the transaction is invalid.

Example Transaction D:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_CD(300 blocks, 5, [D1, C1])`.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 4, [D1, D2])`.
  * `inputC` of 7000 satoshi and an `OP_CD(300 blocks, 10, [D2])`.
* Outputs:
  * 900 sats to destination address `D1`
  * 7000 sats to destination address `D2`

Maximum fee contributions:

* `inputA`:  `10 * 2^5 = 320 sats` 
* `inputB`: `10 * 2^4 = 160 sats`
* `inputC`: `10 * 2^10 = 10,240 sats`

Sets to ensure the contributed fee is within constraints: 

* `inputA`: `inputA - D1 = 1200 - 900 = 300 sat`. **OK**
* `inputB`: `inputB - D1 - D2 = 1000 - 900 - 7000 = -6900 sats`. **OK**. Note that this value is negative because another input (`inputC`) contributed funds to D2, but it's input isn't considered in this step.
* `inputC`: `inputC - D2 = 7000 - 7000 = 0`. **OK**
* `inputA` & `inputB`: `inputA + inputB - D1 - D2 = 1200 + 1000 - 900 - 7000 = -5700`, which is less than their combined maximum fee of `320 + 160 = 480 sat`. **OK**
* `inputB` & `inputC`: `inputB + inputC - D1 - D2 = 1000 + 7000 - 900 - 7000 = 100 sat`, which is less than their combined maximum fee of `160 + 10,240 = 10,400 sat`. **OK**
* `inputA`, `inputB`, & `inputC`. `inputA + inputB + inputC - D1 - D2 = 1200 + 1000 + 7000 - 900 - 7000 = 1300 sats`, which is less than their combined maximum fee of `320 + 160 + 7000 = 7480 sats`.  **OK**

The combination of `inputA` & `inputC` is not relevant because those inputs don't share possible destinations. 

All those checks also succeed, so the transaction is valid.

### Option B - Traditional Script Opcode

The traditional script version of OP_CONSTRAINDESTINATION redefines opcode OP_NOP7 (0xb6). This does the same as OP_CONSTRAINDESTINATION above, except that it does not pop any stack items.

## Rationale

These opcodes provide a way to ensure that individual inputs transfer their value to specified addresses, allowing them to send to P2SH and related addresses that have spending constraints. This allows for covenants and prescribed sequences of transactions. 

The ability to limit the fee that an input can contribute sets limits to the amount of funds an attacker can drain from a victim's bitcoin vault. The fee is proportional to a calculation of recent median fee-rates so that a transaction output's maximum fee can scale as the network changes. If median fee rates rise, the owner still wants the ability to place a reasonable fee on the transaction so its spendable in a reasonable amount of time. If the median fee rates go down, the limit also scales down to limit the damage a griefer could do. The assumption is that median fees will scale primarily with change in purchasing power of bitcoin and not with changing network congestion (which might offer an attacker the opportunity to perform a more damaging grief attack).

The ability to set the `sampleWindowFactor` to options from 6 blocks to 3000 blocks is to give the owner the ability to choose, basically, how much of a rush they think they might be in. If they are generally going to want transactions to go through quickly during network congestion spikes, they'll want a shorter sample-window so they can set their fee higher. However, this also allows a griefer to set a higher fee during network congestion spikes. A longer sample window would be a more accurate measure of general fee rates, but the limit might not be high enough to allow fast confirmation times during congested network conditions. 

Using a `feeFactor` to specify multiples of powers of two for the fee contribution limit assumes that finer-grained fee limit specification is unnecessary. The feeFactor is a two's compliment number so it can be negative in order to specify fractional multiples if they want to for some reason. 

Note also that this design solves the half-spend problem without restricting things like the number of inputs (as op_ctv does). The half-spend problem is a potential problem with covenants where certain types of constraints have loopholes if multiple outputs with the same constraints are used. For example, if you had a constraint saying "the transaction spending this output must send at least 1 bitcoin to address A", if you spent the outputs individually address A would receive at least 1 bitcoin for each transaction. However, if you create a transaction using multiple outputs with that same constraint, the constraint would be met by sending 1 bitcoin to address A, even if dozens of outputs are used. This would mean an attacker could potentially either steal funds, or maybe just grief the victim by spending those bitcoins as fees. Either way, not ideal. OP_CD solves the half-spend problem by ensuring that the constraints of all outputs are summed up and validated. There is no combination of outputs where the intuitive meaning of the OP_CD constrains would be violated.

OP_CD is defined as a check-and-verify opcode rather than pushing 1 or 0 on the stack because that would conceptually allow the execution of an output's script to be affected by the scripts of other outputs. That would add complexity that is almost certainly unacceptable. As currently defined, the interactions between outputs that OP_CD allows can only result in the transaction being declared invalid, rather than changing any execution behavior of the scripts. 

The `medianFeeRate` is used to calculate the maximum fee contribution of an output rather than the median fee so that a change in the average bytes-per-transaction over time doesn't skew the calculations of fee limits.

The opcode has several cases where it reverts to OP_SUCCESSx semantics (or OP_NOPx semantics in the case of the traditional script version). This is done rather than defining it as failure so that future soft-forks can redefine these semantics.

## Detailed Specification

TBD

## Design Tradeoffs and Risks

####  Fungibility Risks with OP_CD

With OP_CD, there is the possibility of creating covenants with unbounded chains of constraints. This could be used to put permanent restrictions on particular coins. However, its already possible to permanently restrict coins. A multisig setup could be created where the coins can only be spent if a particular key approves. I don't see much of a reason to prevent people from doing this kind of thing with their own money. Restricting its use can generally only reduce the value of the coins, so at worst its similar to provably burning coins. 

#### Combination of OP_CD UTXOs DOS Vector

An attacker might construct transactions with many inputs that share potential OP_CD destinations in different combinations. If there are enough combinations that must be checked, this might substantially slow down validation and open a DOS vector for attackers. 

One mitigation for this would be to put a hard limit on the number of inputs that can exist in a transaction that require this combination multiplication (which is if they share some but not all constrained destinations). Another mitigation that could be done is to weight transactions more heavily the more combinations that need to be checked, so that more complex OP_CD transactions need to pay for that complexity. 

#### Feature Redundancy