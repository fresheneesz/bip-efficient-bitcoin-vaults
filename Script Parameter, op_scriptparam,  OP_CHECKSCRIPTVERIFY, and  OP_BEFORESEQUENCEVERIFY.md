```
BIP: BIP-Efficient-Bitcoin-Vaults
  Layer: Consensus (soft fork)
  Title: Efficient Bitcoin Vaults
  Author: Billy Tetrud <bitetrudpublic@gmail.com>
          Looking for co-authors.
  Comments-Summary: TBD
  Comments-URI: TBD
  Status: Draft
  Type: Standards Track
  Created: 2020-06-06
  License: BSD-3-Clause: OSI-approved BSD 3-clause license
```

## Abstract

This BIP proposes an extension to the transaction format and three new script opcodes. The extension has applications for efficient bitcoin vaults, which are described in the *Motivation* section of this BIP.

There are two options given: one option where the opcodes redefine tapscript OP_SUCCESSx opcodes and one option where the new opcodes redefine OP_NOPx opcodes.

It probably makes sense to eventually split this into one BIP per opcode. 

## Summary

### Option A

Option A creates 3 new opcodes by redefining taproot OP_SUCCESSx opcodes. These are more efficient than their option B counterparts.

#### Additional transaction output fields

* output stack - A stack (ordered list) of values that the spending script

#### OP_BEFORESEQUENCE

OP_BEFORESEQUENCE (*OP_BS for short*) redefines opcode OP_SUCCESS_80 (0x50). It does the following:

* Pops the top of the stack and pushes true if the relative lock time of the input is shorter than that value, otherwise pushes false. This can allow a spend path to expire.

#### OP_PUSHOUTPUTSCRIPTSIG

OP_PUSHOUTPUTSCRIPTSIG (*OP_POSS for short*) redefines opcode OP_SUCCESS_81 (0x51). It adds items to the `scriptSig` of the output (not input) of this transaction. It does the following:

1. Pops a number of values values from the stack and interprets them as:
   * Top: `outputId`
   * Top - 1: `numberOfValues`
   * Top - 1 - n: `values`
2. Each of the `values` is then pushed on a new conceptual stack called the "output stack". When the transaction is validated, the output stack is pushed onto the front of the `scriptSig` of the output given by `outputId`.

#### OP_DESTINATIONCONSTRAINED

OP_CONSTRAINDESTINATION (*OP_DC for short*) redefines opcode OP_SUCCESS_82 (0x52). The opcode can be used to limit the destinations that a transaction spending that input can send to and limits the fee that the input can contribute to. It does the following: 

1. The item on the top of the stack is popped and interpreted as a `numberOfAddresses`.
2. The next few items on the stack (numbering `numberOfAddresses`) are popped and interpreted as the list `addressList`.
3. The next item popped from the stack is interpreted as the positive integer `sampleWindowFactor`. This determines the sample window used to calculate the recent median fee. The window is described as a number of blocks:
   * If `sampleWindowFactor` is 0, the `windowLength` is 6 blocks
   * If `sampleWindowFactor` is 1, the `windowLength` is 30 blocks
   * If `sampleWindowFactor` is 2, the `windowLength` is 300 blocks
   * If `sampleWindowFactor` is 3, the `windowLength` is 3000 blocks
   * If `sampleWindowFactor` is any other number, mark the transaction as invalid.
4. The next item popped from the stack is interpreted as an integer `feeFactor` (which can be positive of negative).
5. The `medianFeeRate` is defined as the median fee rate per vbyte for the most recent `windowLength` blocks.
6. The opcode pushes either true or false. The input can contribute no more than `medianFeeRate * 2^feeFactor` of the fee, or false will be pushed. For each relevant set of inputs constrained by OP_DC, ensure that `sum(inputs) - sum(constrainedOutputs) < sum(maxFeesForEachInput)`, otherwise push false. This check does not need to be performed on (irrelevant) sets of inputs where some group of those inputs shares no output address constraints (for outputs that exist in the transaction) with another non-overlapping group within that set. There is likely an optimal way to minimize the number of calculations that must be done for this, but that's left as future work.

For the following examples, I'll write this information as `OP_DC(sampleWindowFactor, feeFactor, [Address1, Address2,...])`. For example, the above example script would be written as `OP_DC(300 blocks, 7, [D, C])`. I'll also use 10 sats/byte as the 300-block median fee rate.

Example opcode call: `OP_CONSTRAINDESTINATION with parameter [2, D, C, 3, 5]` is a script that limits the input to being spent to only address `D` and address `C` with a maximum fee of `10 * 2^5= 320 sats`.

Example Transaction A:

* Inputs:
  * One `input` of 1000 satoshi with `OP_DC(300 blocks, 5, [D, C])`
* Outputs:
  * 100 sat to destination address `D`
  * 850 sat to change address `C`

The `input` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. Since there is only one input, that means the total transaction fee can be no more than 320 sats. The input specifies that it can only send money to addresses `D` and `C`. Since the transaction is sending a total of 950 sat to those addresses, the fee is 50 sats, which is less than 320, so the transaction is valid. 

Example Transaction B:

* Inputs:
  * `inputA` with a value of 2000 satoshi.
  * `inputB` of 1000 satoshi and an `OP_DC(300 blocks, 5, [D2, C2])`.
* Outputs:
  * 200 sats to destination address `D1`
  * 1400 sats to change address `C1`
  * 100 sats to destination address `D2`
  * 550 sats to change address `C2`

`inputB` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. The transaction has a total fee of 700 sats. `inputB` specifies that it can only send money to addresses `D2` and `C2`. Since those two addresses would receive a total of 650 sats, the fee contributed by `inputB` is 350 sats, which is more than 320, so the transaction is invalid.

Example Transaction C:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_DC(300 blocks, 5, [D1, C1])`.
  * `inputB` of 1000 satoshi and an `OP_DC(300 blocks, 4, [D1])`.
* Outputs:
  * 900 sats to destination address `D1`.

`inputA` can contribute a maximum of `10* 2^5 = 320 sats` to the fee. `inputA`'s minimum contribution to the fee is `inputA - D1 = 1200 - 900 = 300 sats`, which is less than 320 sats, and so the immediate evaluation of the operation succeeds.

`inputB` can contribute a maximum of `10 * 2^4 = 160 sats` to the fee. `inputB`'s minimum contribution to the fee is `inputB - D1 = 1000 - 900 = 100 sats`, which is less than 160 sats and so the immediate evaluation of the operation succeeds.

Once both scripts run, the last step is to check the sum of the inputs, outputs, and fee limits for all inputs that use OP_CONSTRAINDESTINATION. `inputA + inputB - D1 = 1200 sats + 1000 sats - 900 sats = 1300 sats`, which is larger than the sum of the maximum fees of `320 + 160 =  480 sats`, so the transaction is invalid.

Example Transaction D:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_DC(300 blocks, 5, [D1, C1])`.
  * `inputB` of 1000 satoshi and an `OP_DC(300 blocks, 4, [D1, D2])`.
  * `inputC` of 7000 satoshi and an `OP_DC(300 blocks, 10, [D2])`.
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

### Option B

Option B creates 3 new opcodes by redefining NOP opcodes. The downsides are

* the extra OP_DROPs that are needed,
* the extra transaction size necessary to spend a transaction that has a non-empty output stack, which comes from the extra values necessary for the spender to add into the scriptSig in order to spend in comparison to Option A. 

#### Additional transaction output fields

* output stack - Same as above.

#### (Option B) OP_BEFORESEQUENCEVERIFY

OP_BEFORESEQUENCEVERIFY (*OP_BSV for short*) redefines opcode OP_NOP5 (0xb4). It does the following:

* The same as OP_BEFORESEQUENCE above but does not pop the stack and does not verify. 

#### (Option B) OP_PUSHOUTPUTDATA

OP_PUSHOUTPUTDATA (*OP_POD for short*) redefines opcode OP_NOP6 (0xb5). It does the same as OP_PUSHOUTPUTDATA above except it does not pop stack items and instead of adding the output stack onto the top of the `scriptPubKey`, it instead requires the `scriptSig` spending the relevant output of the transaction to have particular values on the top of its stack after evaluation. When that UTXO is spent, after evaluation of the UTXO's `scriptSig`, the top three values on the stack are verified to match the "output stack" stored with the UTXO. If they don't match, the transaction is marked invalid. 

#### (Option B) OP_CONSTRAINDESTINATION

OP_CONSTRAINDESTINATION (*OP_CD for short*) redefines opcode OP_NOP7 (0xb6). This does the same as OP_DESTINATIONCONSTRAINED above, except that it does not pop any stack items and does not verify.

## Motivation

### Better Wallet Vaults

A Bitcoin Vault is a wallet construct that uses time-delayed transactions to allows a window of time for the sender to reverse a transaction made maliciously, accidentally, or incorrectly. The opcodes described in this BIP can be used to create an efficient and flexible Bitcoin Vault that is more efficient and solves an attack that Bitcoin Vaults using op_ctv have. 

#### Bitcoin Vault with OP_CHECKTEMPLATEVERIFY

OP_CHECKTEMPLATEVERIFY specified in [BIP 119](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki) allows for creating [wallet vaults}(https://utxos.org/uses/vaults/) of the following form:

Let's say you construct two addresses, each with their own script.

* `Address A` has 2 spend paths:

1. Can be spent by `key1`, only to `Address B` or `ChangeAddress`.
2. Can be spent by `key1` + `key2`, to any address.

* `Address B` has 2 spend paths:

1. Can be spent by `key2`, after a relative time-lock of 5 days, to any address.
2. Can be spent by `key1` + `key2` without waiting for any time-lock, to any address.

This set up allows a normal transaction to proceed as follows:

1. Use `Address A`'s spent-path 1 to send to `Address B`.
2. If the transaction was in fact intended by the owner, after the relative time-lock, the owner can use Address B's spend-path 1 to send to the final destination. 

If the transaction needs to be canceled after step 1 (eg because key1 was compromised and the transaction wasn't actually intended by the owner), `Address B`'s spend path 2 can be used to send the funds to a new wallet.

There are a couple issues with the above bitcoin vault construct:

A. Sending to a transaction normally requires 2 transactions (assuming Address A's spend-path 2 is usually avoided for ease of use and security reasons).

B. An attacker who has compromised `key2` can still steal funds if they patiently wait for a transaction to come through, as mentioned in ["Custody Protocols Using Bitcoin Vaults"](https://arxiv.org/pdf/2005.11776.pdf) by Swambo et al.

C. Outputs can only be sent one at a time - they cannot be combined other outputs from the same vault, nor any other output.

D. [Footguns related to address reuse and incorrect receipt amounts.](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki#forwarding-addresses)

#### A more efficient Bitcoin Vault using the proposed new opcodes

By using OP_DC, OP_BS, and OP_POSS together, these issues can be eliminated in a more effective setup.

Let's say you construct `IntermediateAddress` which has the following spend paths:

1. Can be spent after a relative time-lock of 5 days, by the destination address (passed using OP_POSS or alternatively required to be passed in the scriptSig by OP_POD).
2. Can be spent by `key1` + `key2`, *before* the relative time-lock of 5 days unlocks, to any address.

Then construct an `Address A` with the following spend paths:

1. Can be spent by `key1`, to ``IntermediateAddress` (or a similar change address). The fee must be less than some multiple of the median fee of the last 30 blocks.
2. Can be spent by `key1` + `key2`, without waiting for any time-lock, to any address.

The couple issues above are eliminated. This can be used to create highly redundant, highly secure wallets like [this one](https://github.com/fresheneesz/TordlWalletProtocols/blob/master/experimental/singleWalletProtocols/Time-locked-3-Seed-Cold-Wallet.md).

* Spending requires only a single transaction.
* Funds cannot be stolen if only 1 of [`key1`, `key2`] are compromised. All keys must be compromised to steal funds.
* Vault outputs can be used in any combination, with each other, and with arbitrary other outputs. 
* Arbitrary amounts can be sent to and from the vaults, so there are no footguns related to address reuse or incorrect receipt amounts.

The scripts below define a bitcoin vault that nearly has usage parity with a normal bitcoin address. The only situations where this kind of wallet vault can't be used in a transaction are:

* Situations where the transaction must finalize before the wallet vault's timeout, for example situations where a subsequent transaction is expected to be possible after 6 confirmations. 
* Situations where the receiver doesn't recognize the wallet vault construct. 

#### Simple Script for Example Bitcoin Vault - Option A

`Address A` could have the following spend paths (using pseudo-code):

```
Key spend-path: 
  Aggregated multi-signature for: publicKeyA, publicKeyB
Script spend-path: 
  # Duplicate the top 2 stack items so they can be used for two OP_POSS operations
  OP_2DUP
  # Push the destination public key onto the output stack for IntermediateDestination.
  <IntermediateDestination, ..., ...> OP_PUSHOUTPUTSCRIPTSIG
  # Push the destination public key onto the output stack for ChangeAddress.
  <ChangeAddress, ..., ...> OP_PUSHOUTPUTSCRIPTSIG
  # Ensure that the input is spent only to IntermediateDestination or ChangeAddress1 with 
  # a maximum fee of 4 times the 300-block median fee.
  <2, IntermediateDestination, ChangeAddress1, 300 blocks, 2> OP_DESTINATIONCONSTRAINED
  # Require a signature for 1 of the 2 keys.
  <publicKeyA> OP_CHECKSIGADD
  <publicKeyB> OP_CHECKSIGADD
  # Note that this is a 2 here instead of 1 because publicKeyA was checked with CHECKSIGADD, 
  # which is done as an optimization to save adding an OP_VERIFY.
  <2, ...> OP_LESSTHANOREQUAL
```

To spend this script you would use one of the following patterns:

1. Key spend-path scriptSig: `<none>`, Witness: `keyASig keyBSig`
2. Script spend-path 1A scriptSig:  `destinationPublicKey 1`, Witness: `keyASig`
3. Script spend-path 1B scriptSig: `destinationPublicKey 1`, Witness: `keyBSig`

In pattern 2 or 3, each output's scriptPubKey would have the value `<destinationPublicKey>` on the top of its stack. 

`IntermediateDestination` would then have the following script:

```
Key spend-path: 
  None
Script spend-path 1: 
  # Uses the destinationPublicKey put on the stack by OP_CD in the transaction that created 
  # this output 
  <...> OP_CHECKSIGVERIFY
  # Transaction creating this output must have been confirmed at LEAST 5 days ago. Note, no
  # drop is needed here because the top of the stack will be non-zero.
  <5 days> OP_CHECKSEQUENCEVERIFY
Script spend-path 2: 
  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> OP_BEFORESEQUENCEVERIFY
  # Require a signature for both source keys
  <publicKeyA> OP_CHECKSIG
  <publicKeyB> OP_CHECKSIGADD
  <2> OP_NUMEQUAL
```

This script can be spent using one of the following patterns:

1. Script spend-path 1 scriptSig: `<none>`, Witness: `destinationSig` 
2. Script spend-path 2 scriptSig: `<none>`, Witness: `keyASig keyBSig`

`ChangeAddress1` would be an address that is somewhat of a combination of the above two scripts:

```
Key spend-path: 
  None
Script spend-path 1: 
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> OP_CHECKSEQUENCEVERIFY
  # Drop the <5 days> item and destinationPublicKey added to the stack by OP_POSS in
  # the parent transaction. The combination of this is just an optimization that saves
  # putting in two OP_DROPs.
  OP_2DROP
  # Duplicate the top 2 stack items so they can be used for two OP_POSS operations
  OP_2DUP
  # Push the destination public key onto the output stack for IntermediateDestination.
  <IntermediateDestination, ..., ...> OP_PUSHOUTPUTSCRIPTSIG
  # Push the destination public key onto the output stack for ChangeAddress2.
  <ChangeAddress2, ..., ...> OP_PUSHOUTPUTSCRIPTSIG
  # Ensure that the input is spent only to IntermediateDestination or ChangeAddress2 with 
  # a maximum fee of 4 times the 300-block median fee.
  <2, IntermediateDestination, ChangeAddress2, 300 blocks, 2> OP_DESTINATIONCONSTRAINED
  # Require a signature for 1 of the 2 change keys
  <changePublicKeyA> OP_CHECKSIGADD
  <changePublicKeyB> OP_CHECKSIGADD
  # Note that this is a 2 here instead of 1 because publicKeyA was checked with CHECKSIGADD, 
  # which is done as an optimization to save adding an OP_VERIFY.
  <2> OP_LESSTHANOREQUAL
Script spend-path 2: 
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> OP_CHECKSEQUENCEVERIFY
  # Drop the <5 days> item and the OP_POSS value as above.
  OP_2DROP
  # Require a signature for 2 of 2 change keys
  <changePublicKeyA> OP_CHECKSIG
  <changePublicKeyB> OP_CHECKSIGADD
  <2> OP_NUMEQUAL
Script spend-path 3: 
  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> OP_BEFORESEQUENCEVERIFY
  # Require a signature for both source keys
  <publicKeyA> OP_CHECKSIG
  <publicKeyB> OP_CHECKSIGADD
  <2> OP_NUMEQUAL
```

This script can be spent using one of the following patterns:

1. Script spend-path 1A scriptSig: `newDestinationPublicKey 1`, Witness: `changeASig`
2. Script spend-path 1B scriptSig: `newDestinationPublicKey 1`, Witness: `changeBSig`
3. Script spend-path 2 scriptSig: `<none>`, Witness: `changeKeyASig changeKeyBSig`
4. Script spend-path 3 scriptSig: `<none>`, Witness: `sourcekeyASig sourcekeyBSig`

This example shows a transactions that can be spent using either 1 or 2 keys, however this can be extended to a vault that has paths for any number of keys in any number of combinations. 

#### Simple Script for Example Bitcoin Vault - Option B

To create `Address A`, it could have the following script (using pseudo-code):

```
OP_IF 
  # Require a signature for one of the 2 keys
  <2, publicKeyA, publicKeyB, 1, ...> OP_CHECKMULTISIGVERIFY
  # Ensure that the input is spent only to IntermediateDestination or ChangeAddress1 with 
  # a maximum fee of 4 times the 300-block median fee.
  <2, IntermediateDestination, ChangeAddress1, 300 blocks, 2> OP_CONSTRAINDESTINATION
  OP_2DROP
  OP_2DROP
  OP_DROP
  # Push the destination public key onto the output stack.
  <IntermediateDestination, ..., ...> OP_PUSHOUTPUTDATA
  # Remove IntermediateDestination
  OP_DROP
  # Note the two stack values are reused from above (because OP_POD only peeks stack items).
  <ChangeAddress, ..., ...> OP_PUSHOUTPUTDATA  
  # This leaves one non-zero item on the stack to mark success.
  OP_2DROP
OP_ELSE
  <2, publicKeyA, publicKeyB, 2, ..., ...> OP_CHECKMULTISIGVERIFY
  1 # To mark success.
OP_ENDIF
```

To spend this script you would use one of the following `scriptSig` patterns:

1. `destinationPublicKey 1 ignored keyASig 1`
2. `destinationPublicKey 1 ignored keyBSig 1`
3. `ignored keyASig keyBSig 0`

`IntermediateDestination` would then have the following script:

```
# Swap is used here because the ouptut stack from Address A above requires the
# destinationPublicKey to be on top.
OP_SWAP
OP_IF
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> OP_CHECKSEQUENCEVERIFY
  OP_DROP
  <..., ...> OP_CHECKSIG
OP_ELSE
  <2, publicKeyA, publicKeyB, 2 ..., ...> OP_CHECKMULTISIGVERIFY
  # Transaction creating this output must have been confirmed at MOST 5 days ago. Note, no
  # drop is done here because the top of the stack will be non-zero to signal success.
  <5 days> OP_BEFORESEQUENCEVERIFY
OP_ENDIF
```

This script can be spent using one of the following `scriptSig` patterns:

1. `destinationPublicKey 1 destinationSig`
2. `ignored 2 keyBSig 0 keyASig`

`ChangeAddress` would be an address that is somewhat of a combination of the above two scripts:

```
# Swap is used here because the ouptut stack from Address A above requires the
# destinationPublicKey to be on top.
OP_SWAP
OP_IF
  # Require a signature for the change address (should probably be a P2SH that can do a 
  # 1 of 2 multisig check). This is possible, right?
  <..., ...> OP_CHECKSIGVERIFY
  # Ensure that the input is spent only to IntermediateDestination or ChangeAddress1 with 
  # a maximum fee of 4 times the 300-block median fee.
  <2, IntermediateDestination, ChangeAddress1, 300 blocks, 2> OP_CONSTRAINDESTINATION
  OP_2DROP
  OP_2DROP
  # Transaction creating this output must have been confirmed at LEAST 5 days ago. Note, no
  # drop is done here because the top of the stack will be non-zero to signal success.
  <5 days> OP_CHECKSEQUENCEVERIFY
  # Drops the <5 days> and one additional item used for OP_CD.
  OP_2DROP
  # Push the destination public key onto the output stack.
  <IntermediateDestination, ..., ...> OP_PUSHOUTPUTDATA
  # Remove IntermediateDestination
  OP_DROP
  # Note the two stack values are reused from above (because OP_POD only peeks stack items). Three
  # items are left on the stack, the topmost one being non-zero to mark success.
  <ChangeAddress, ..., ...> OP_PUSHOUTPUTDATA
OP_ELSE
  # Drop the changePublicKey.
  OP_DROP
  OP_IF
    <2, changePublicKeyA, changePublicKeyB, 2, ..., ..., ...> OP_CHECKMULTISIGVERIFY
    # Transaction creating this output must have been confirmed at LEAST 5 days ago. Note, no
    # drop is done here because the top of the stack will be non-zero to signal success.
    <5 days> OP_CHECKSEQUENCEVERIFY
  OP_ELSE
    <2, publicKeyA, publicKeyB, 2, ..., ..., ...> OP_CHECKMULTISIGVERIFY
    # Transaction creating this output must have been confirmed at MOST 5 days ago. Note, no
    # drop is done here because the top of the stack will be non-zero to signal success.
    <5 days> OP_BEFORESEQUENCEVERIFY
  OP_ENDIF
OP_ENDIF
```

This script can be spent using one of the following `scriptSig` patterns:

1. `newDestinationPublicKey 1 changeASig 1 changePublicKey`
2. `newDestinationPublicKey 1 changeBSig 1 changePublicKey`
3. `ignored changeKeyASig changeKeyBSig 1 0 changePublicKey`
4. `ignored sourcekeyASig sourcekeyBSig 0 0 changePublicKey`

Note that `OP_DROP` and `OP_2DROP` are used here to remove the item on the stack read by OP_CD, OP_BSV, and OP_POD because none of those remove a value from the stack (to be consistent with the NOP operations they redefine).

### Payment Channels

#### Increased Channel Routes

Since the number of HTLCs is limited to 483 in order to keep the transaction from becoming too large to be valid, covenant transactions can be used to expand the number of HTLCs. (More details here](https://utxos.org/uses/htlcs/)

### Congestion Controlled Transactions

In times of mempool congestion, certain payers might be in a situation where they need to commit to transactions now, but their receivers might be able to wait to have spending access to the funds until after the congestion passes. In such a situation, it could be advantageous for the payers to create a smaller transaction that commits to sending funds to various addresses at a later date. 

For example, if the payer wants to create a batch transaction with 3 inputs and to 20 addresses, they could instead create a transaction with the same 3 inputs, but a single output that uses a covenant (eg using OP_DC) to ensure the funds can only be spent to the 20 addresses. The payer would then broadcast the smaller transaction during congestion with a higher fee, then once the congestion has passed, would spend the created output to send coins to the 20 originally intended addresses.

This could save payers money, but would end up being larger on the blockchain, so there is a downside cost felt from a set of transactions like this. Its also unclear how often this would be useful. However, there is more information about this (including simulations) [in the material about OP_CTV](https://utxos.org/uses/scaling/).

OP_DC and OP_CD can both be used for the same purpose.

## Detailed Specification

TBD

## Rationale

#### OP_BEFORESEQUENCE and OP_BEFORESEQUENCEVERIFY

These opcodes allow a normal transaction from a bitcoin vault to take place in a single transaction, rather than in two transactions (like would be required by a bitcoin vault that only has the use of [OP_CHECKTEMPLATEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki#Deployment)). This cuts the space-cost (and therefore necessary fees) of these transactions nearly in half. 

#### OP_DESTINATIONCONSTRAINED and OP_CONSTRAINDESTINATION

These opcodes provide a way to ensure that individual inputs transfer their value to specified addresses, allowing them to send to P2SH and related addresses that have spending constraints. This allows for covenants and prescribed sequences of transactions. 

The ability to limit the fee that an input can contribute sets limits to the amount of funds an attacker can drain from a victim's bitcoin vault. The fee is proportional to a calculation of recent median fees so that a transaction output's maximum fee can scale as the network changes. If median fee rates rise, the owner still wants the ability to place a reasonable fee on the transaction so its spendable in a reasonable amount of time. If the median fee rates go down, the limit also scales down to limit the damage a griefer could do. The assumption is that median fees will scale primarily with change in purchasing power of bitcoin and not with changing network congestion (which might offer an attacker the opportunity to perform a more damaging grief attack).

The ability to set the `sampleWindowFactor` to options from 6 blocks to 3000 blocks is to give the owner the ability to choose, basically, how much of a rush they think they might be in. If they are generally going to want transactions to go through quickly during network congestion spikes, they'll want a shorter sample-window so they can set their fee higher. However, this also allows a griefer to set a higher fee during network congestion spikes. A longer sample window would be a more accurate measure of general fee rates, but the limit might not be high enough to allow fast confirmation times during congested network conditions. 

Using a `feeFactor` to specify multiples of powers of two for the fee contribution limit assumes that finer-grained fee limit specification is unnecessary. The feeFactor is a two's compliment number so it can be negative in order to specify fractional multiples if they want to for some reason. 

Note also that this design solves the half-spend problem without restricting things like the number of inputs (as op_ctv does). The half-spend problem is a potential problem with covenants where certain types of constraints have loopholes if multiple outputs with the same constraints are used. For example, if you had a constraint saying "the transaction spending this output must send at least 1 bitcoin to address A", if you spent the outputs individually address A would receive at least 1 bitcoin for each transaction. However, if you create a transaction using multiple outputs with that same constraint, the constraint would be met by sending 1 bitcoin to address A, even if dozens of outputs are used. This would mean an attacker could potentially either steal funds, or maybe just grief the victim by spending those bitcoins as fees. Either way, not idea.

OP_DC and OP_CD both solve the half-spend problem by ensuring that the constraints of all outputs are summed up and validated. There is no combination of outputs where the intuitive meaning of the OP_DC/CD constrains would be violated.

#### OP_PUSHOUTPUTSCRIPTSIG and OP_PUSHOUTPUTDATA

These opcodes provide a way to pass data from the scriptSig used to spend an input to the created output. In the context of wallet vaults, this allows the destination to both be unknown at the time the wallet vault is created and at the same time, be able to commit to that destination with a single transactions. With OP_CTV by contrast, the destination is not committed to when sending coins from the vault, and a second transaction must be made that chooses the destination. 

## Considerations

#### Single-transaction receiving

In order for the receiver to be convinced they have actually received the coins from the transaction, they must be able to verify that their address is the only one that has valid spend paths for the output, now and in the future. To do this, the recipient must be able to see the entire script that `Destination` has. If it isn't on the blockchain then they need to be sent the full script by the transaction sender.

#### Recovery with static backups

The bitcoin vaults that can be created with these opcodes are designed so that coins can be sent to an arbitrary external address in a single transaction. After waiting the time-out period, a receiver can treat the output as their own indefinitely. A normal transaction that sends someone money can be recovered just using a seed and scanning the blockchain for transactions that can be spent by the keys that seed can generate. The vaults created with the above scripts can also do this, but require a slightly different mechanism.

As long as the bitcoin vault scripts used are standardized and built into the software the user is using, all the information needed to recover transactions sent to a receiver can be generated from the receiver's seed or found on the blockchain. Software that is syncing the blockchain need only check for outputs who's output stack contains one of the seeds that can be generated from the receiver's seed. When the software finds an output like this, the script can be generated using the two sender public keys exposed when the sender spent the input that created the output. If the resulting script matches, then the user's software can count it amongst the user's funds (as long as the wait-time has passed). A similar process can be used by the sender to recover change outputs.

#### Mempool handling

Because OP_BS and OP_BSV cause transactions to potentially expire in a new way, miners will need to take this into account when updating their mempool. An efficient way to do this would be to keep a data structure that maps block height to transactions that will expire at that block height. Upon admission to the mempool, a miner would check the transaction for an OP_BEFORESEQUENCEVERIFY requirement during validation of the transaction and would write a record of the transaction into the data structure (the map). On each newly mined block, each miner would check the map for transactions that expire upon that block and remove them from the mempool. This should be a fairly light data structure, requiring only an integer mapping to a list of pointers. This should also be a fast operation to insert into the map upon validation of each transaction, and a fast operation to evaluate on each new block. 

### Design Tradeoffs and Risks

#### OP_BS and OP_BSV transaction expiry

Objections have been given for any possibility for transactions to expire. Satoshi said the following [here](https://bitcointalk.org/index.php?topic=1786.msg22119#msg22119):

> In the event of a block chain reorg after a segmentation, transactions need to be able to get into the chain in a later block. The transaction and all its dependents would become invalid. This wouldn't be fair to later owners of the coins who weren't involved in the time limited transaction.

However, if a person waited for the standard 6 blocks before accepting a transaction as confirmed, there should be no significantly likely scenario where any finalized transaction needs to be reverted. If 6 blocks is indeed a safe threshold for finalization, then any transaction that has 5 or fewer confirmations should be considered fair game for reversal. It seems unreasonable to call this "unfair", in fact it is rather the standard assumption. 

Another reason I heard of was mempool handling issues, which I addressed in the "Mempool handling" Considerations section above. 

I asked a question about this [on the Bitcoin stack exchange](https://bitcoin.stackexchange.com/questions/96366/what-are-the-reasons-to-avoid-spend-paths-that-become-invalid-over-time-without) but haven't gotten any answers. I would be very curious to know what other arguments against doing this there are. 

Without some ability for spend paths to expire, there is fundamentally no way to structure wallet vaults in a way where spending can be done in a single step. 

####  Fungibility Risks with OP_DC and OP_CD

With OP_DC/CD, there is the possibility of creating covenants with unbounded chains of constraints. This could be used to put permanent restrictions on particular coins. However, its already possible to permanently restrict coins. A multisig setup could be created where the coins can only be spent if a particular key approves. I don't see much of a reason to prevent people from doing this kind of thing with their own money. Restricting its use can generally only reduce the value of the coins, so its similar to provably burning coins. 

#### Mitigations of potential attacks on a wallet vault

There are a couple attacks that could be executed on a vault like the one described above. 

A. An attacker who has found `key1` could attempt to send funds to themselves. This would fail as long as the owner checks their wallet more frequently than the relative lock time (in the above example, every 5 days) minus the time it takes them to find and use both keys to recover.

B. An attacker who has found `key1` could attempt to grief the victim by sending potentially all the funds in the wallet to any address using most of the funds as fee. An attacker could theoretically spend 1 sat to an arbitrary address, and have the rest of the entire input as miner fees, effectively sweeping the funds into the hands of the miner. This could be mitigated by limiting how much fee the transaction (as a whole) can leave to the miner as a multiple of the median transaction fee rate (in a rolling set of recent blocks), though point C below has a better general solution. 

C. Similar to B, an attacker who has found `key1` and knows the attacker has many smaller UTXOs in the bitcoin vault wallet could send one transaction per UTXO with the maximum fee, maximizing the amount the victim loses. This could be mitigated by limiting how much each UTXO can contribute to the fee, which is what OP_DC or OP_CD does.

D. An attacker who has found both `key1` and `key2` could of course freely steal the funds. This can be mitigated by adding more keys with additional levels of lock-times. 

E. An attacker might construct transactions with many inputs that share potential OP_DC/OP_CD destinations in different combinations. If there are enough combinations that must be checked, this might substantially slow down validation. One mitigation for this would be to put a hard limit on the number of inputs that can exist in a transaction that share some but not all constrained destinations. 

## Backwards Compatibility

This BIP proposes two alternatives that address backwards compatibility in different ways. Either option is able to be introduced in a soft fork.

The taproot script option uses OP_SUCCESSx opcodes which provide a very clear and simple way to ensure backwards compatibility with a soft fork. This is the option I (the author) prefer. 

The other option uses the more traditional OP_NOPx opcodes, which the soft forks for OP_CHECKSEQUENCEVERIFY and OP_CHECKLOCKTIMEVERIFY (see [BIP-0065](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki) and [BIP-0112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)) have similarly done without introducing compatibility issues. There are downsides of this approach that were listed above in the description of Option B.

## Open questions

1. What concrete objections are there to OP_BEFORESEQUENCE?
2. Does OP_DC make it possible to make [channel factories](https://utxos.org/uses/batch-channels/) with the same benefits as op_ctv?
3. Does OP_DC make it possible to make [non-interactive channels](https://utxos.org/uses/non-interactive-channels/) like op_ctv can?

## Similar work

* [OP_CHECKTEMPLATEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki)
* A number of alternatives listed here: https://utxos.org/alternatives/
  * OP_CHECKOUTPUTVERIFY
  * OP_PUSHTXDATA
  * OP_CAT + OPCHECKSIGFROMSTACKVERIFY
  * OP_CHECKTXOUTSCRIPTHASHVERIFY
  * SIGHASH_NOINPUT / ANYPREVOUT