```
BIP: BIP-LIMITFEECONTRIBUTION
Layer: Consensus (soft fork)
Title: LIMITFEECONTRIBUTION
Author: Billy Tetrud <bitetrudpublic@gmail.com>
Looking for co-authors and/or reviewers.
Comments-Summary: TBD
Comments-URI: 
 * https://bitcointalk.org/index.php?topic=5349117.msg57499720
 * https://www.mail-archive.com/bitcoin-dev@lists.linuxfoundation.org/msg10292.html
Status: Draft
Type: Standards Track
Created: 2021-10-29
License: BSD-3-Clause: OSI-approved BSD 3-clause license
```

- [Introduction](#introduction)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
- [Design](#design)
- [Specification](#specification)
  + [Opcode Example Call](#opcode-example-call)
- [Rationale](#rationale)
- [Detailed Specification](#detailed-specification)
- [Design Tradeoffs and Risks](#design-tradeoffs-and-risks)
    + [Theft via fees](#theft-via-fees)
    + [Manipulation of the average fee rate](#manipulation-of-the-average-fee-rate)

##  Introduction

### Abstract

This BIP proposes a new tapscript opcode, OP_LIMITFEECONTRIBUTION (*OP_LFC for short*), which can be used to constrain the amount of the fee that output is allowed to contribute to. The opcode has applications for limiting fee griefing and theft-via-fees. 

This could be activated using a tapscript OP_SUCCESSx opcode.

### Motivation

* Preventing fee griefing in transactions using [OP_CONSTRAINDESTINATION](cd/bip-constraindestination.md) or other covenant opcodes.

## Design

TBD

## Specification

`OP_LFC(sampleWindowFactor, feeFactor)`

OP_LIMITFEECONTRIBUTION (*OP_LFC) redefines opcode OP_SUCCESS_83 (0x53). The opcode can be used to limit how of much the fee that an input can contribute to. It does the following: 

1. The top item on the stack is popped and expected to be an 8 bit long value. 
   
   1. The first two bits are interpreted as the positive integer `sampleWindowFactor`. This determines the sample window used to calculate the recent median fee. The window is described as a number of blocks:
   
     * If `sampleWindowFactor` is 0, the `windowLength` is 6 blocks.
     * If `sampleWindowFactor` is 1, the `windowLength` is 30 blocks.
     * If `sampleWindowFactor` is 2, the `windowLength` is 300 blocks.
     * If `sampleWindowFactor` is 3, the `windowLength` is 3000 blocks.
   
   2. The other 6 bits are interpreted as a signed integer `feeFactor` (which can be positive, `0`, or `-1`). If the number is `-1`, it is interpreted as `-infinity` (meaning the transaction can contribute nothing to the fee). 
   
2. A `blockMedianFeeRate` is defined as the median fee-rate per vbyte for a given block. The `averageFeeRate` is defined as the average of `blockMedianFeeRate` for each block in the most recent `windowLength` blocks. The `maxFeeContribution` is defined as `averageFeeRate * 2^feeFactor` of the fee. Note that this is a limitation on the fee, not on the fee-rate. If `feeFactor` is -1, `maxFeeContribution` is 0.

3. Once all input scripts have been evaluated, add up all output values minus their fee limits, and ensure the result is larger than the transaction fee. 

**Reversion modes**. For all the following situations, the opcode reverts to its OP_SUCCESS semantics:

3. The first item on the stack is anything other than an 8 bit value.
4. `sampleWindowFactor` is anything but a number 0 through 3.
5. `feeFactor` is less than -1.

**Failure modes**. For all the following additional situations, the transaction is marked invalid:
6. If `sum(allOutputValues) - sum(allFeeLimits) < transactionFee`.

Note: Any opcode that limits how many satoshi an input can contribute to a given output should consider the fee limit.

### Opcode Example Call

Given a 300-block `averageFeeRate` of 10 sats/byte, the following is a script that sets a maximum fee contribution of `10 * 2^5= 320 sats`.

```
2
5
OP_LIMITFEECONTRIBUTION 
```

## Rationale

The ability to limit the fee that an input can contribute sets limits to the amount of damage an attacker can do to a victim by way of setting excessive fees on a transactions. The fee is proportional to a calculation of recent median fee-rates so that a transaction output's maximum fee can scale as the network changes. If median fee rates rise, the owner still wants the ability to place a reasonable fee on the transaction so its spendable in a reasonable amount of time. If the median fee rates go down, the limit also scales down to limit the damage a griefer could do. The assumption is that median fees will scale primarily with change in purchasing power of bitcoin and not with changing network congestion (which might offer an attacker the opportunity to perform a more damaging grief attack).

The ability to set the `sampleWindowFactor` to options from 6 blocks to 3000 blocks is to give the owner the ability to choose, basically, how much of a rush they think they might be in. If they are generally going to want transactions to go through quickly during network congestion spikes, they'll want a shorter sample-window so they can set their fee higher. However, this also allows a griefer to set a higher fee during network congestion spikes. A longer sample window would be a more accurate measure of general fee rates, but the limit might not be high enough to allow fast confirmation times during congested network conditions. 

Using a `feeFactor` to specify multiples of powers of two for the fee contribution limit assumes that finer-grained fee limit specification is unnecessary. 

The `blockMedianFeeRate` is used to calculate the maximum fee contribution of an output rather than the raw median fee so that a change in the average bytes-per-transaction over time doesn't skew the calculations of fee limits.

The opcode has several cases where it reverts to OP_SUCCESSx semantics (or OP_NOPx semantics in the case of the traditional script version). This is done rather than defining it as failure so that future soft-forks can redefine these semantics.

## Detailed Specification

TBD

## Design Tradeoffs and Risks

#### Theft via fees

The purpose of the fee limit is to limit the risk of griefing attacks and theft via fees. However, this doesn't fully prevent fee abuse. For example, a miner could purchase some goods with a transaction using this opcode, and then choose a time during high fees to send the transaction with as high a fee as possible. OP_LFC carries the risk that some actors will play dirty like this and extract higher than expected fees in multi-party contracts like this. However, because the fee market is naturally limited by people's willingness to pay, its unlikely that such behavior could extract significantly more value out of these transactions. 

Single-party use cases (eg wallet vaults) don't carry as much risk of this, since the user is sending to themselves. However, if a wallet vault's key is stolen by someone connected with a miner, an attacker may use this approach to extract some value from their victim via fees or simply grief their victim. 

#### Manipulation of the average fee rate

Miners could potentially manipulate the fee rate, for example by sending themselves payments with high fees. This could allow those miners (or those complicit with them) to extract more value from a victim via fees. Doing this would be costly for miners tho, and recouping the cost would require a reasonably large attack (at least hundreds of transactions from victims' wallets for a 6 block sample window, and on the order of hundreds of thousands of transactions for a 3000 block sample window).

Miners who manipulate the average fee rate this way could be vulnerable to fee-sniping. This would either be a deterrent  for miners to do this, or would be pressure for these manipulative miners to cartelize. 