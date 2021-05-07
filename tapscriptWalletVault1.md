# Wallet Vault 1

This is a basic wallet vault script using the operations OP_POSS, OP_CD, and OP_BBV. It has a lot of limitations, but it is simple to understand and it is has fewer limitations than wallet vaults created with op_checktemplateverify. 

The scripts are written in javascript-like pseudo-code. Variables that end in an underscore (eg `intermediateAddress_`) represent hard coded values in the script.

## Properties

* Can send to arbitrarily many destinations in a transaction: **false**
* Can always limit the transaction to one change output: true.
* Can omit the change address: **false**
* Can spend change immediately: **false**
* Can consolidate inputs: true. Arbitrary number of wallet vault inputs can be spent in the same transaction and send to the same destination/change address. 
* Canceling each destination / change send either individually or together are both options: **true**.
* Finalization time: **5 days**. Sending to an address that expects to be able to spend the transaction earlier than this won't work. This is a fundamental limitation of a wallet vault.

## Utility functions

These are utility functions that will be used in the pseudo-scripts below:

```
// Returns the number of keys with matching signatures in the witness.
function checkSigs(sigPubKeyPairs);
  
// Pushes the values onto the scriptSig for the output corresponding with outputId.
// outputValueMap is a map that maps outputId to a list of values to push onto the output stack.
function pushOutputStack(address, outputValueMap);

// Requires that the output only send money to the listed addresses, with a maximum fee determined
// by window-length and maxFeeContribution.
function constrainDestination(addresses, windowLength, maxFeeContribution);

// Verifies that the block this transaction is mined in has a height less than blockheight.
function beforeBlockVerify(blockheight);

// Verifies that the public key matches the address.
function verifypublicKey(publicKey, address):
  verify(ripeMd160(sha256(publicKey)) == address)
```

## Base Wallet Vault Address

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

## Intermediate Address

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
Key spend-path:
  <none>
Script spend-path 1:
  destinationSpend(destinationPublicKey)
Script spend-path 2:
  cancelTransaction()

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

## Creation

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

These scripts are written with notation where `<a, b, c>` indicates values that exist on the stack in that order (ie c is pushed, then b is pushed, then a is pushed). `...` indicates a value that was already on the stack before that line, and `.....` indicates more values matching an example pattern.

The base wallet vault address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  Aggregated multi-signature for: publicKeyA1_, publicKeyB1_

Script spend-path: 
  # Push the destination address onto the output stack for the
  # destinationIntermediateAddress_ output. 
  <destinationIntermediateAddress_, 1, ..., ...> PUSHOUTPUTSTACK

  # Push the change address onto the output stack for the changeIntermediateAddress_ output. 
  <changeAddress1, 1, ..., ...> PUSHOUTPUTSTACK
  
  # Ensure that the input is spent only to intermediateDestination or changeAddress1 with 
  # a maximum fee of 512 times the 300-block median fee-per-byte.
  <2, destinationIntermediateAddress_, changeAddress1, 300 blocks, 512x> CONSTRAINDESTINATION
  
  # Require a signature for 1 of the 2 keys.
  <publicKeyA1_, ...> CHECKSIG
  <publicKeyB1_, ..., ...> CHECKSIGADD
  <1, ...> LESSTHANOREQUAL
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path witness:  `keySig DUP changeAddress changeOutputId destinationAddress destinationOutputId`, where
   * `destinationAddress` is the address of the destination, and
   * `keySig` is the signature for either `publicKeyA1_` or `publicKeyB1_` (`keyA1Sig` or `keyB1Sig`).

In the script spend-path, each output's scriptPubKey would have an output stack appended to it:

* `destinationIntermediateAddress_` script spend-path: `destinationAddress`.
* `changeIntermediateAddress_` script spend-path: `changeAddress `.

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
  verifyPublicKey(..., ...);
  
  # Uses the passed in destinationPublicKey.
  <..., ...> CHECKSIG
  
Script spend-path 2:
  # Drop the `destinationAddress` added from the output stack.
  DROP

  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> BEFOREBLOCKVERIFY
  
  # Require a 2-of-2 signature for two return keys. 
  <publicKeyAN_, ...> CHECKSIG
  <publicKeyBN_, ..., ...> CHECKSIGADD
  <2, ...> NUMEQUAL
  
  # Note that the destinationPublicKey is left intentionally unused in stack position 2.
  
# First param is the address, the second is the public key. Hashes are passed to the script
# (via OP_POSS from the previous transaction) rather than raw public keys to retain
# quantum resistance. 
pseudofunction verifyPublicKey(..., ...):
  OVER
  <...> SHA256
  <...> RIPEMD160
  <..., ...> NUMEQUALVERIFY
```

To spend this script you would use one of the following patterns:

1. Script spend-path 1 witness: `destinationKeySig destinationPublicKey`
2. Script spend-path 2 witness: `publicKeyBNSig publicKeyANSig`