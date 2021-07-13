# Wallet Vault 1

This is a basic wallet vault script using the operations OP_POS, OP_CD, and OP_BBV. It has a lot of limitations, but it is simple to understand and it is has fewer limitations than wallet vaults created with op_checktemplateverify.

## Properties

* Can send to arbitrarily many destinations in a transaction: **false**
* Can always limit the transaction to one change output: true.
* Can omit the change address: **false**
* Can spend change immediately: **false**
* Can consolidate inputs: true. Arbitrary number of wallet vault inputs can be spent in the same transaction and send to the same destination/change address. 
* Canceling each destination / change send either individually or together are both options: **true**.
* Finalization time: **5 days**. Sending to an address that expects to be able to spend the transaction earlier than this won't work. This is a fundamental limitation of a wallet vault.

## Addresses

The address scripts are written in [javascript-like pseudo-code](notation.md).

### Base Wallet Vault Address

The first address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  spendImmediately(keyASig, keyBSig)
Script spend-path: 
  spendNormally(signature, destinationOutputId, destinationAddress, changeOutputId, changeAddress)

function spendImediately(keyASig, keyBSig):
  verify(checkSigs([[keyASig, publicKeyA1_], [keyBSig, publicKeyB1_]) >= 2)

function spendNormally(signature, destinationOutputId, destinationAddress, changeOutputId, changeAddress):
  pushOutputStack(destinationIntermediateAddress_, {destinationOutputId: [destinationAddress]})
  pushOutputStack(changeIntermediateAddress_, {changeOutputId: [changeAddress]})
  constrainDestination(
    [destinationIntermediateAddress_, changeIntermediateAddress_], 300 blocks, 512x median fee-rate
  )
  verify(checkSigs([[signature, publicKeyA1_], [signature, publicKeyB1_]]) >= 1)
```

### Intermediate Address

The `destinationIntermediateAddress_` and `changeIntermediateAddress_` referenced above are addresses with the following spend paths:

```
Key spend-path:
  <none>
Script spend-path 1:
  destinationSpend(destinationSignature, destinationPublicKey)
Script spend-path 2:
  cancelTransaction(keyANSig, keyBNSig)

function destinationSpend(destinationSignature, destinationPublicKey):
  // Transaction creating this output must have been confirmed at LEAST 5 days ago.
  checkSequenceVerify(5 days)
  let destinationAddress = getOutputStackItem(0)
  verifyPublicKey(destinationPublicKey, destinationAddress)
  verify(checkSigs([destinationSignature, destinationPublicKey]) >= 1)
  
function cancelTransaction(keyANSig, keyBNSig):
  // Transaction creating this output must have been confirmed at MOST 5 days ago.
  beforeBlockVerify(5 days)
  verify(checkSigs(2, [[keyANSig, publicKeyAN_], [keyBNSig, publicKeyBN_]]) >= 2)
```

### Creation

To create receiving addresses for the wallet, first two *intermediate address*es are created, each with different values for both `publicKeyAN_` and `publicKeyBN_ `. One will be used for `destinationIntermediateAddress_`, and the other will be used for `changeIntermediateAddress_`. Then, a *base wallet vault address* will be created that references those addresses.

## Spending

The way funds in a *base wallet vault address* would be used is the following:

1. If the owner wants to spend the funds immediately in arbitrary ways, they create a transaction and sign it with both keys.
2. If the owner wants to spend normally:
   1. They create a new *base wallet vault address* to use as the `changeAddress`. 
   2. then they create a transaction with two outputs: one to the `destinationIntermediateAddress_` and one to the `changeIntermediateAddress_`, specify the outputIds and addresses as inputs to the script, and then sign with just one of the two keys.
   3. After 5 days, the transaction is final and the destination address can spend it arbitrarily.
3. If an attacker uses the "spendNormally" path described above, or if a mistake was made by the owner, the owner can cancel the transaction before the transaction has had 5 days of confirmations by creating a transaction and signing it with both keys. Note that the owner can choose to cancel the spends to the `destinationIntermediateAddress_` and `changeIntermediateAddress_` individually or together in one transaction and can choose to cancel one and not the other.

## Script Implementation

These scripts are written with [Script pseudocode notation](notation.md).

The base wallet vault address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  Aggregated multi-signature for: publicKeyA1, publicKeyB1

Script spend-path: 
  # Push the destination address onto the output stack for the
  # destinationIntermediateAddress output.
  <destinationIntermediateAddress, 1, _, _> PUSHOUTPUTSTACK

  # Push the change address onto the output stack for the changeIntermediateAddress_ output.
  <changeAddress1, 1, _, _> PUSHOUTPUTSTACK
  
  # Ensure that the input is spent only to intermediateDestination or changeAddress1 with 
  # a maximum fee of 512 times the 300-block median fee-per-byte.
  <2, destinationIntermediateAddress_, changeAddress1, 300 blocks, 512x> CONSTRAINDESTINATION
  
  # Require a signature for 1 of the 2 keys.
  <publicKeyA1, _> CHECKSIG
  <publicKeyB1, _, _> CHECKSIGADD
  <1, _> LESSTHANOREQUAL
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path witness:  `keySig DUP changeAddress changeOutputId destinationAddress destinationOutputId`, where
   * `destinationAddress` is the address of the destination, and
   * `keySig` is the signature for either `publicKeyA1` or `publicKeyB1` (`keyA1Sig` or `keyB1Sig`).

In the script spend-path, each output's scriptPubKey would have an output stack appended to it:

* `destinationIntermediateAddress` script spend-path: `destinationAddress`.
* `changeIntermediateAddress` script spend-path: `changeAddress `.

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
Key spend-path: 
  <none>
  
Script spend-path 1: 
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  
  # Verify the `destinationPublicKey` against the `destinationAddress` put on the
  # stack by OP_CD in the transaction that created this output.
  verifyPublicKey(_, _)
  
  # Uses the passed in destinationPublicKey.
  <_, _> CHECKSIG
  
Script spend-path 2:
  # Drop the `destinationAddress` added from the output stack.
  DROP

  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> BEFOREBLOCKVERIFY
  
  # Require a 2-of-2 signature for two return keys. 
  <publicKeyAN_, _> CHECKSIG
  <publicKeyBN_, _, _> CHECKSIGADD
  <2, _> NUMEQUAL
  
  # Note that the destinationPublicKey is left intentionally unused in stack position 2.
  
# First param is the address, the second is the public key. Hashes are passed to the script
# (via OP_POSS from the previous transaction) rather than raw public keys to retain
# quantum resistance. 
pseudofunction verifyPublicKey(_, _):
  OVER
  <_> SHA256
  <_> RIPEMD160
  <_, _> NUMEQUALVERIFY
```

To spend this script you would use one of the following patterns:

1. Script spend-path 1 witness: `destinationKeySig destinationPublicKey`
2. Script spend-path 2 witness: `publicKeyBNSig publicKeyANSig`