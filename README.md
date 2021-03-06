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

- [Introduction](#introduction)
  * [Abstract](#abstract)
  * [Motivation](#motivation)
    + [Better Wallet Vaults](#better-wallet-vaults)
    + [Bitcoin Vault with OP_CHECKTEMPLATEVERIFY](#bitcoin-vault-with-op_checktemplateverify)
    + [An Improved Wallet Vault](#an-improved-wallet-vault)
- [Design](#design)
- [Specification](#specification)
  * [Opcodes](#opcodes)
  * [Wallet Vaults](#wallet-vaults)
- [Considerations](#considerations)
  * [Single-transaction receiving from Wallet Vaults](#single-transaction-receiving-from-wallet-vaults)
  * [Wallet Vault Transactions with multiple vault inputs](#wallet-vault-transactions-with-multiple-vault-inputs)
  * [Wallet Vault Transactions sending to multiple addresses](#wallet-vault-transactions-sending-to-multiple-addresses)
  * [Recovery with static backups for Wallet Vaults](#recovery-with-static-backups-for-wallet-vaults)
- [Design Tradeoffs and Risks](#design-tradeoffs-and-risks)
  * [Mitigations of potential attacks on a wallet vault](#mitigations-of-potential-attacks-on-a-wallet-vault)
  * [OP_SUCCESSx vs OP_NOPx](#op_successx-vs-op_nopx)
- [Open questions](#open-questions)
- [Similar work](#similar-work)
- [Copyright](#copyright)

## Introduction

### Abstract

This BIP proposes four new script opcodes: [OP_BEFOREBLOCKVERIFY](bbv/bip-beforeblockverify.md), [OP_PUSHOUTPUTSTACK](pos/bip-pushoutputstack.md),  [OP_CONSTRAINDESTINATION](cd/bip-constraindestination.md), and [OP_LIMITFEECONTRIBUTION](lfc/bip-limit-fee-contribution.md). These extensions have applications for efficient bitcoin vaults, among other things, which are described in the *Motivation* sections of this BIP and the opcode BIPs.

### Motivation

* Efficient wallet vaults (in comparison to OP_CTV wallet vaults).
  * Half as expensive, because spending only requires one transaction (instead of two).
  * Funds are not vulnerable during spending (vs OP_CTV vaults where an attacker who only has one of many keys can steal funds in transit).
  * Very flexible - vault outputs can be used as inputs in any combination and spent to any combination of outputs (vs OP_CTV vaults where each output must be spent in its own separate transaction with no other inputs and only a single list of specific outputs). 
* Other motivations specific to each of the four opcodes.

The primary motivation for this BIP is to allow the creation of better wallet vaults with more ideal characteristics that have as few limitations as possible in comparison to normal bitcoin transactions. Additional motivations specific to each opcode are discussed in the BIPs for each opcode. 

#### Better Wallet Vaults

A "wallet vault" is a wallet construct that uses time-delayed transactions to allows a window of time for the sender to reverse a transaction made maliciously, accidentally, or incorrectly. The opcodes described in this BIP can be used to create a efficient and flexible wallet vaults that are more efficient that ones that could be created using OP_CTV and solves security vulnerabilities of such wallet vaults. The vaults that can be created with the opcodes in this BIP have the following benefits:

* Half as expensive as wallet vaults using OP_CHECKTEMPLATEVERIFY. If one day most on-chain transactions are from wallet vaults to lightning nodes (or vice versa), this could potentially reduce on-chain traffic by nearly 25%.
* Far more flexible than OP_CTV vaults. Outputs can be spent in a transaction with any other outputs.
* Fixes the security hole in OP_CTV vaults that allows an attacker to steal some funds that the vault owner is sending out of the vault. 
* No presigned transactions necessary.

#### Bitcoin Vault with OP_CHECKTEMPLATEVERIFY

OP_CHECKTEMPLATEVERIFY specified in [BIP 119](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki) allows for creating [wallet vaults](https://utxos.org/uses/vaults/) of form described below.

Let's say we construct two addresses, each with their own script.

`Address A` has 2 spend paths:

1. Can be spent by `key1`, only to `Address B` or `ChangeAddress`.
2. Can be spent by `key1` + `key2`, to any address.

`Address B` has 2 spend paths:

1. Can be spent by `key1`, after a relative time-lock of 5 days, to any address.
2. Can be spent by `key1` + `key2` without waiting for any time-lock, to any address.

In [spend-path notation](notation.md), the above would be:

```
Address A:
* Normal: key1Sig
-> Address B: key1Sig & timelock(5 days) || key1Sig & key2Sig
-> Change Address A: ... // With similar spending requirements to Address A
* Immediate: key1Sig & key2Sig
```

This set up allows a normal transaction to proceed as follows:

1. Use `Address A`'s `Normal` spent-path to send to `Address B`.
2. If the transaction was in fact intended by the owner, after the relative time-lock, the owner can spend the output from Address B's to the final destination using a `key1` signature.

If the transaction needs to be canceled after step 1 (eg because key1 was compromised and the transaction wasn't actually intended by the owner), `Address B`'s spend path 2 can be used to send the funds to a new wallet.

There are a couple issues with the above bitcoin vault construct:

* Sending to a transaction normally requires 2 transactions (assuming Address A's spend-path 2 is usually avoided for ease of use and security reasons) for any trustless transaction that is sent to another person. In the cases that a person is sending from their wallet vault to another address they own, this downside doesn't exist. Similarly in cases where the recipient already trusts the sender to not claw back funds, this may not be a concern. 
* An attacker who has compromised `key1` can still steal or grief funds if they patiently wait for a transaction to come through, as mentioned in ["Custody Protocols Using Bitcoin Vaults"](https://arxiv.org/pdf/2005.11776.pdf) by Swambo et al. 
  * The attack could proceed as follows: An attacker who has gained a copy of `key1` would simply wait until the victim sends a particularly large transaction into `Address B`. As soon as the timeout expires on the transaction from `Address A` -> `Address B`, the attacker could broadcast a transaction using `key2` from `Address B` to wherever they want. Even if the victim always broadcasts a transaction immediately upon timeout expiry, its likely the attacker will simply use an exorbitant fee in anticipation of that possibility and beat out the victim. Its unlikely for the victim to be able to react in time manually to this, and even if automated reactions were prepared, this would generally also end in an arms race of fees where the victim still loses their funds. So the best case scenario for this is a grief-attack vector.
* Outputs in the vault can only be spent one at a time - they cannot be combined other outputs, neither from the same vault nor any other output.
* [Footguns related to address reuse and incorrect receipt amounts.](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki#forwarding-addresses)

#### An Improved Wallet Vault

By using OP_CD, OP_BBV, OP_POS, and OP_LFC together, these issues can be eliminated in a more effective setup.

Let's say we construct two addresses, each with their own script. `Address A` has the following spend paths:

1. Can be spent by `key1`, to `IntermediateAddress` (or a similar change address). The fee must be less than some multiple of the median fee-rate of the last 30 blocks.
2. Can be spent by `key1` + `key2`, without waiting for any time-lock, to any address.

`IntermediateAddress` has the following spend paths:

1. Can be spent by the destination address after a relative time-lock of 5 days.
2. Can be spent by `key1` + `key2` *before* the relative time-lock of 5 days unlocks.

The couple issues above are eliminated. This can be used to create highly redundant, highly secure wallets like [this one](https://github.com/fresheneesz/TordlWalletProtocols/blob/master/experimental/singleWalletProtocols/Time-locked-3-Seed-Cold-Wallet.md).

* Spending requires only a single transaction.
* Funds cannot be stolen if only 1 of [`key1`, `key2`] are compromised. All keys must be compromised to steal funds. Griefing is strictly limited.
* Vault outputs can be used in any combination, with each other, and with arbitrary other outputs. 
* Arbitrary amounts can be sent to and from the vaults, so there are no footguns related to address reuse or incorrect receipt amounts.

A more detailed description of a simple wallet of this type is described in [tapscriptWalletVault1.md](tapscriptWalletVault1.md), note that this has some limitations solved by [tapscriptWalletVault3.md](tapscriptWalletVault3.md).

## Design

TBD

## Specification

### Opcodes

Four opcodes are needed for the wallet vaults described in this BIP:

* [OP_BEFOREBLOCKVERIFY](bbv/bip-beforeblockverify.md) - Verifies that the block the transaction is within has a block height below a particular number. This allows a spend-path to expire. 
* [OP_PUSHOUTPUTSTACK](pos/bip-pushoutputstack.md) - Pushes data onto the "output stack" for outputs to a particular address. The "output stack" is a stack of data that will be pushed onto the stack after the witness script runs, but before the primary script runs. This allows a script writer to constrain behavior of a chained transaction output with witness data that was used a covenant input script.
* [OP_CONSTRAINDESTINATION](cd/bip-constraindestination.md) - Limits the destinations that an input can send to. This allows for the creation of covenant transactions. 
* [OP_LIMITFEECONTRIBUTION](lfc/bip-limit-fee-contribution.md) - Limits the fee that an input can contribute to.

There are two options given: one option where the opcodes redefine tapscript OP_SUCCESSx opcodes and one option where the new opcodes redefine OP_NOPx opcodes. 

### Wallet Vaults

The scripts proposed in this BIP are specified in [tapscriptWalletVault3.md](tapscriptWalletVault3.md), with [traditionalScriptWalletVault3.md](traditionalScriptWalletVault3.md) containing the non-tapscript version.

## Considerations

### Single-transaction receiving from Wallet Vaults

In order for the receiver to be convinced they have actually received the coins from the transaction, they must be able to verify that their address is the only one that has valid spend paths for the output, now and in the future. To do this, the recipient must be able to see the entire script that `Destination` has. If it isn't on the blockchain then they need to be sent the full script by the transaction sender.

In the proposed wallet vault design, as long as the script templates (minus keys specific to the owner) are available as standards in the software, no additional information is needed on the blockchain. Coins can be sent to an arbitrary external address in a single transaction. After waiting the time-out period, a receiver can treat the output as their own indefinitely.

### Wallet Vault Transactions with multiple vault inputs

The proposed wallet vault design allows consolidating inputs: multiple wallet vault UTXOs can be used to send to a transaction with a single output (or single output and change). This is done by using a static intermediate address that has no values that are unique to the particular wallet vault address. In this way, all wallet vault addresses can be constrained to send to that static intermediate address, which allows multiple UTXOs to be consolidated into a single input. Another requirement for this is that OP_PUSHOUTPUTSTACK does not fail if multiple inputs write to the output stack for the same output as long as the resulting out stack for that output is the same for every input that writes to the output stack for that output. 

### Wallet Vault Transactions sending to multiple addresses

The proposed wallet vault design allows using a single UTXO to send to many addresses. Similarly to the considerations for transactions with many vault inputs, the static intermediate address is important, since all wallet vault inputs can send to that intermediate address. This can be made into effectively sending to many output addresses by using OP_PUSHOUTPUTSTACK's ability to push different values on the output stack for each output. In this way, the destination address can be passed via the output stack while still technically using the same static intermediate address. 

### Recovery with static backups for Wallet Vaults

A normal transaction that sends someone money can be recovered just using a seed and scanning the blockchain for transactions that can be spent by the keys that seed can generate. The proposed wallet vault design can also do this, but require a slightly different mechanism.

As long as the bitcoin vault scripts used are standardized and built into the software the user is using, all the information needed to recover transactions sent to a receiver can be generated from the receiver's seed or found on the blockchain. Software that is syncing the blockchain need only check for outputs who's output stack contains one of the addresses that can be generated from the receiver's seed. When the software finds an output like this, the script can be generated using the output stack data. If the resulting script matches, then the user's software can count it amongst the user's funds (as long as the wait-time has passed).

#### Potential Attack Vectors on Wallet Vaults

There are a couple attacks that could be executed on wallet vaults in general. This assumes one with a single-key path and a dual-key path:

A. An attacker who has found `key1` could attempt to send funds to themselves. 

B. An attacker who has found `key1` could attempt to grief the victim by maximizing the fee used in transactions that spend the wallet's outputs. This could theoretically be done by or with a miner to actual steal funds.

C. An attacker who has found both `key1` and `key2` could steal the funds.

D. An attacker who has found both `key1` and `key2` could grief the victim by burning the funds in transactions as fee.

Item **A** is mitigated in a wallet vault by reversing the transaction using `key1` and `key2`. 

Item **B** can be mitigated by limiting how much fee an output can contribute to. OP_LFC does exactly this. One could imagine an opcode that limits how much fee the transaction as a whole can leave as fee, however this can have negative effects like preventing the ability to do replace-by-fee and is inflexible with regards to transaction size. You could also imagine limiting the fee **rate** of a transactions. This allows RBF and arbitrary transaction sizes, however it allows an attacker to create artificially large transactions and waste the output's funds in fee anyway. OP_LFC has none of these problems because it constrains the fee that a particular output can contribute to, and leaves other outputs unconstrained. 

Item **C** can be somewhat mitigated by adding more keys with additional levels of lock-times. But it can actually be entirely prevented by having a spend-path that allows someone with all of the wallet vault's keys to reverse even transactions that used all the keys, as long as the reverse transaction is done within a time limit. If an attacker attempts to steal funds, the transaction can simply be reverted. 

However, this still leaves item **D**. If the mitigation to item **C** is done, the attacker can still continue to try to steal the funds, effectively taking them hostage. The victim might be forced to capitulate with demands in return for their money back. However, it still makes the attack substantially more difficult. 

## Design Tradeoffs and Risks

### OP_SUCCESSx vs OP_NOPx

The versions using OP_SUCCESSx opcodes are more efficient than their OP_NOPx counterparts. Using OP_NOPx for these opcodes has the following downsides:

* the extra OP_DROPs that are needed,
* the extra transaction size necessary to spend a transaction that has a non-empty output stack (from OP_PUSHOUTPUTSTACK ), because of the extra values necessary for the spender to add into the scriptSig in order to spend. 

Since Taproot has locked in, there is little reason to consider doing this using OP_NOPx. The soft forks for OP_CHECKSEQUENCEVERIFY and OP_CHECKLOCKTIMEVERIFY (see [BIP-0065](https://github.com/bitcoin/bips/blob/master/bip-0065.mediawiki) and [BIP-0112](https://github.com/bitcoin/bips/blob/master/bip-0112.mediawiki)) have similarly done without introducing compatibility issues.

## Open questions

1. What concrete objections are there to OP_BEFOREBLOCKVERIFY?
2. Does OP_CD make it possible to make [channel factories](https://utxos.org/uses/batch-channels/) with the same benefits as op_ctv?
3. Does OP_CD make it possible to make [non-interactive channels](https://utxos.org/uses/non-interactive-channels/) like op_ctv can?
5. Terminology around the definition of OP_CD - it defines it as constraining to an "address", but maybe saying it constrains the script would be more accurate? 

## Similar work

* [OP_CHECKTEMPLATEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki)
* A number of alternatives listed here: https://utxos.org/alternatives/
  * OP_CHECKOUTPUTVERIFY
  * OP_PUSHTXDATA
  * OP_CAT + OPCHECKSIGFROMSTACKVERIFY
  * OP_CHECKTXOUTSCRIPTHASHVERIFY
  * SIGHASH_NOINPUT / ANYPREVOUT

## Copyright

This document is licensed under the 3-clause BSD license.