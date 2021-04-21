```
BIP: BIP-BEFOREBLOCKVERIFY
Layer: Consensus (soft fork)
Title: BEFOREBLOCKVERIFY
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
- [Specification](#specification)
  * [Option A - Tapscript Opcode](#option-a---tapscript-opcode)
  * [Option B - Traditional Script Opcode](#option-b---traditional-script-opcode)
- [Design](#design)
  * [Transaction Evaluation](#transaction-evaluation)
- [Rationale](#rationale)
- [Considerations](#considerations)
- [Design Tradeoffs and Risks](#design-tradeoffs-and-risks)
    + [Transaction Expiry](#transaction-expiry)
      - [Mempool Handling](#mempool-handling)
      - [Reorgs](#reorgs)
      - [Spam Risk](#spam-risk)
- [Copyright](#copyright)

## Introduction

### Abstract

This BIP proposes a new script opcode, OP_BEFOREBLOCKVERIFY. The extension has applications for efficient bitcoin vaults, among other things. This could either be activated using a tapscript OP_SUCCESSx opcode or less efficiently as a traditional OP_NOPx opcode.

### Motivation

#### Efficient Wallet Vaults

The primary motivation for this opcode is to create efficient wallet vaults. See the *Motivation* section in the [parent BIP](Script Parameter, op_scriptparam,  OP_CHECKSCRIPTVERIFY, and  OP_BEFORESEQUENCEVERIFY.md). In short, without some ability for spend-paths to expire, there is fundamentally no way to structure wallet vaults such that spending can be done in a single step.

#### Time-limited Escrow

An escrow can be set up such that the payer pays a 2-of-3 multisig address where the payer, recipient, and escrow service all hold one key. Once the item is received, the money can be sent to the recipient by either the payer and recipient collaborating or the escrow service and one of the other parties collaborating. 

OP_BBV could be used to implement more efficient time-limited escrows, where the sender would create a transaction output that is spendable if signed before a certain block height by both the sender and escrow service, and after that block height the output could only be spent by the end recipient. This has the benefits of only requiring one transaction for a recipient to receive exclusive ownership over the funds.

Admittedly, this application of OP_BBV has not been studied to any depth and is just a concept.

## Specification

### Option A - Tapscript Opcode

OP_BEFOREBLOCKVERIFY (*OP_BBV for short*) redefines opcode OP_SUCCESS_80 (0x50). It does the following:

* Pops the top of the stack and marks the transaction invalid if the block that the transaction is mined in has a height of greater than or equal to that value. This can allow a spend path to expire.
* Reversion modes. For all the following situations, the opcode reverts to its OP_SUCCESS semantics:
  1. The top item on the stack is less than 0.
* Failure modes. For all the following additional situations, the transaction is marked invalid:
  1. The stack is empty.

### Option B - Traditional Script Opcode

OP_BEFOREBLOCKVERIFY (*OP_BBV for short*) redefines opcode OP_NOP5 (0xb4). It does the following:

* The same as OP_BEFOREBLOCKVERIFY above but does not pop the stack. 

## Design

TBD

### Transaction Evaluation

There are two distinct types of invalidity for this opcode:

1. In the case of evaluating a transaction within the context of a particular block (eg in block validation), the transaction can simply be marked invalid or continue execution, exactly as OP_VERIFY does. 
2. In the case of evaluating a transaction for inclusion into the mempool, the opcode should instead be treated as specifying a conditional validity. Record should be kept of the transaction and its boundary condition as long as the node might need to use the transaction, which are in the following cases:
   1. The specified block height has not been reached
   2. The specified block height has been reached, but a reorg is still possible with a significant likelihood.

## Rationale

This opcode allows a normal transaction from a bitcoin vault to take place in a single transaction, rather than in two transactions (like would be required by a bitcoin vault using [OP_CHECKTEMPLATEVERIFY](https://github.com/bitcoin/bips/blob/master/bip-0119.mediawiki#Deployment)). This cuts the space-cost (and therefore necessary fees) of these transactions nearly in half. 

This is structured as a check-and-verify operation to prevent any need to run the script more than once. Were the opcode to push a value on the stack, a completely different code path could be executed across the block height boundary passed to the operation. If more operations that had boundary conditions were implemented, it could end up multiplying the number of times the script must be run, which both uses more computer resources and adds complexity to handling the transaction validation and mempool. 

OP_BBV evaluates a height from the stack directly instead of using a method similar to OP_CHECKSEQUENCEVERIFY that requires adding transaction data (like nSequence). Peter Wuille [mentioned](https://bitcoin.stackexchange.com/a/45808/5254) that OP_CHECKSEQUENCEVERIFY constrains the nSequence which in turn actually constrains the relative locktime, rather than having OP_CSV constrain the relative locktime directly, and that this is done to provide a way to distinguish between an "outright invalid" transaction vs a temporarily invalid transaction. However, it should have been possible to define OP_CSV such that the script would always fully run, but OP_CSV would mark it as temporarily invalid if its check fails, similarly to the above. This way the script would never need to be rerun, as long as the state about when the transaction becomes valid is retained. 

This could be done like the following, using two types of invalidity:

1. In the case of evaluating a transaction within the context of a particular block (eg in block validation), the transaction can simply be marked invalid or continue execution, exactly as OP_VERIFY does. 
2. In the case of evaluating a transaction for inclusion into the mempool, the opcode should instead be treated as specifying a conditional validity. Record should be kept of the transaction and its boundary condition as long as the node might need to use the transaction, which are in the following cases:
   1. The specified block height has not been reached.
   2. The specified block height has been reached, but a reorg is still possible with a significant likelihood.

## Considerations

TBD

## Design Tradeoffs and Risks

#### Transaction Expiry

Since OP_BBV can cause a valid signed transaction to become invalid at some point in the future, consideration must be taken to ensure this doesn't cause problems for nodes. This can have an effect in two cases:

1. in normal mempool handling, and
2. in the case of reorgs. 

In the past, objections have been given for any possibility for transactions to expire on these grounds. I asked a question about this [on the Bitcoin stack exchange](https://bitcoin.stackexchange.com/questions/96366/what-are-the-reasons-to-avoid-spend-paths-that-become-invalid-over-time-without) but haven't gotten any answers. I also had [a conversation](https://www.reddit.com/r/Bitcoin/comments/gwjr8p/i_am_jeremy_rubin_the_author_of_bip119_ctv_ama/ft2die9?utm_source=share&utm_medium=web2x&context=3) with Jeremy Rubin about this. I am very curious to know what other arguments against doing this there are. I will address those concerns here to the best of my knowledge.

##### Mempool Handling

Mempool handling must consider a few possibilities:

* A transaction may expire and should be removed from the mempool at that time, 
* The transaction may expire before it is profitable to mine,
* Other mutually exclusive transactions may already be in the mempool that are valid until a different block height.

An efficient way to handle evicting transactions from the mempool upon expiry would be to keep a data structure that maps block height to transactions that will expire at that block height. Upon admission to the mempool, a miner would check the transaction for an OP_BBV requirement during validation of the transaction and would write a record of the transaction into the data structure (the map). On each newly mined block, each miner would check the map for transactions that expire upon that block and remove them from the mempool. This should be a fairly light data structure, requiring only an integer mapping to a list of pointers. The operations on this should all be quite fast: to insert into the map upon validation of each transaction, to evaluate on each new block, and to remove from the map when possible. This could also be done using a list of OP_BBV transactions ordered by expiry block, however inserting and removing into that list would be more expensive than using a map. 

Evaluating OP_BBV transactions for mempool inclusion has additional considerations in comparison to a normal transaction. Because it might expire, it might expire before it is profitable to mine. In order to evaluate this, a node should have a way to estimate the likelihood the transaction will be profitable to mine within the span of blocks the transaction is valid for. This likelihood, once calculated, can then be multiplied by the transaction's fee-rate to get an "expected fee-rate" of the transaction that can then be compared against the fee-rates of other normal transactions in order to determine whether inclusion and propagation are appropriate. 

This further complicates mempool handling because, while a normal transaction's expected fee-rate doesn't change over time, an OP_BBV transaction will change its expected fee-rate as it gets closer to expiring. A node could re-evaluate the expected fee-rate for each OP_BBV transaction periodically in the same way it was originally calculated. If it is determined this is too expensive to do, a simpler model of linearly declining expected value could be used instead, with less frequent periodic corrections using the more accurate estimate calculation. 

Consideration is also needed for transactions that are already in the mempool with a different expiration block height. In the case a transaction has both an earlier or equal expiration and a lower or equal fee-rate, that transaction can be either evicted from the mempool or rejected from inclusion. In the case a transaction has a later expiration date and a lower (or equal) fee, neither transaction is the obvious winner. In such a case, the expected fee-rate calculation described in the above two paragraphs can be used to compare those transactions and choose the higher value one. Like RBF transactions, it would be prudent to only accept new transactions to the mempool if the marginal value in comparison to the existing transaction is greater than a threshold, to prevent replacement-transaction spam. It may be acceptable for a node to accept multiple mutually-exclusive OP_BBV transactions into the mempool if the number is limited to only a handful (eg 3) and if the higher-value transaction has a lower enough probability of being mined. 

Note that because OP_CHECKSEQUENCEVERIFY transactions are not added to mempools or propagated through the network until they become valid, there is no additional complexity that arises from interactions between OP_BBV and OP_CSV.

##### Reorgs

Satoshi said the following [here](https://bitcointalk.org/index.php?topic=1786.msg22119#msg22119):

> In the event of a block chain reorg after a segmentation, transactions need to be able to get into the chain in a later block. The transaction and all its dependents would become invalid. This wouldn't be fair to later owners of the coins who weren't involved in the time limited transaction.

However, if a person waited for the standard 6 blocks before accepting a transaction as confirmed, there should be no significantly likely scenario where any finalized transaction needs to be reverted. If 6 blocks is indeed a safe threshold for finalization, then any transaction that has 5 or fewer confirmations should be considered fair game for reversal. It seems unreasonable to call this "unfair", in fact it is rather the standard assumption.  It seems Peter Todd [had similar thinking](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2013-July/002939.html).

> Satoshi was worried that in the event of a re-org long chains of
> transactions could become invalid and thus impossible to include in the
> blockchain again, however that's equally possibly through tx mutability
> or double-spends;(1) I don't think it's a valid concern in general.

I've asked [a question about  how reorgs are handled](https://bitcoin.stackexchange.com/questions/105525/how-do-nodes-handle-long-reorgs) on the Bitcoin stack exchange. I would expect that nodes need to revert their UTXO set to the point at which the chains diverged, re-admit transactions to the mempool from blocks evaluated after that divergence point, then re-evaluate the blocks. This perhaps might end with evicting conflicting transactions from the mempool after evaluation has completed. None of this process sounds like OP_BBV transactions would make a reorg substantially more difficult for nodes or for users sending transactions. 

##### Spam Risk

Using OP_BBV, someone could spam the mempool with valid transactions that expire in the next block. If the spammer waits for that block to be mined before broadcasting, they could spam the network with transactions that are no longer valid among nodes who have node received the new block yet.

An mitigation to this is to simply reject transactions that expire in the next block. However, the attacker may then create transactions with a fee too low to likely get into 2 blocks from now. Some evaluation of the likelihood of how quickly the transaction can get into a block is needed, and this is discussed in the *Mempool Handling* section above. 

## Copyright

This document is licensed under the 3-clause BSD license.