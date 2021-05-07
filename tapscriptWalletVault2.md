# Wallet Vault 2

This is a major upgrade from [Wallet Vault 1](walletVault1.md). Its still quite simple, but allows sending to an arbitrary number of destinations and allows omitting a change address. The only downside vs Wallet Vault 1 is that change and destinations can't be cancelled independently without exposing active public keys (and thus losing quantum resistance). 

These scripts use the same psuedo-code style and utility functions used by [Wallet Vault 1](walletVault1.md). 

## Properties

* Can send to arbitrarily many destinations in a transaction: true
* Can always limit the transaction to one change output: true.
* Can omit the change address: true
* Can spend change immediately: **false**
* Can consolidate inputs: true. Arbitrary number of wallet vault inputs can be spent in the same transaction and send to the same destination/change address. 
* Canceling each destination / change send either individually or together are both options: **false**. However, in the scenario of an attacker, this ability isn't needed. 
* Finalization time: **5 days**. Sending to an address that expects to be able to spend the transaction earlier than this won't work. This is a fundamental limitation of a wallet vault.

## Base Wallet Vault Address

The first address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  spendImmediately(keyA1Sig, keyB1Sig)
Script spend-path: 
  spendNormally(signature, outputAddressAssociations)

function spendImediately(keyA1Sig, keyB1Sig):
  verify(checkSigs([[keyA1Sig, publicKeyA1_], [keyB1Sig, publicKeyB1_]]) >= 2)

function spendNormally(signature, outputAddressAssociations):
  let outputValueMap = {}
  for a in outputAddressAssociations:
    outputValueMap[a.outputId] = [a.address]
  pushOutputStack(intermediateAddress_, outputValueMap)
  pushOutputStack(intermediateAddress_, {-1: publicKeyA2Hash_, publicKeyB2Hash_})
  constrainDestination([intermediateAddress_], 300 blocks, 512x median fee-rate)
  verify(checkSigs([[signature, publicKeyA1_], [signature, publicKeyB1_]]) >= 1)
```

## Intermediate Address

The `intermediateAddress_` referenced above is a static address who's scripts don't change depending on the wallet vault or its keys. In fact, all wallet vaults of this specific form would share this address. Its execution is differentiated only by the output stack values placed into it by the parent input's OP_POS execution. It would have the following spend paths:

```
Key spend-path:
  <none>
Script spend-path 1:
  destinationSpend(destinationKeySig, destinationPublicKey)
Script spend-path 2:
  cancelTransaction(keyASig, publicKeyA, keyBSig, publicKeyB)

function destinationSpend(destinationKeySig, destinationPublicKey):
  // Transaction creating this output must have been confirmed at LEAST 5 days ago.
  checkSequenceVerify(5 days)
  let destinationAddress = getOutputStackItem(0)
  verifyPublicKey(destinationPublicKey, destinationAddress)
  verify(checkSigs([[destinationKeySig, destinationPublicKey]]) >= 1)
  
function cancelTransaction(keyASig, publicKeyA, keyBSig, publicKeyB):
  // Transaction creating this output must have been confirmed at MOST 5 days ago.
  beforeBlockVerify(5 days)
  let publicKeyAHash = getOutputStackItem(1)
  let publicKeyBHash = getOutputStackItem(2)
  verifyPublicKey(publicKeyA, publicKeyAHash)
  verifyPublicKey(publicKeyB, publicKeyBHash)
  verify(checkSigs(2, [[keyASig, publicKeyA], [keyBSig, publicKeyB]]) >= 2)
```

## Creation

To create receiving addresses for the wallet, create a *basic wallet vault address*.

## Spending

The way funds in a *base wallet vault address* would be used is the following:

1. If the owner wants to spend the funds immediately in arbitrary ways, they create a transaction and sign it with both keys.
2. If the owner wants to spend normally:
   1. They create a new *base wallet vault address* to use as the change address (unless not needed). 
   2. Then they create a transaction with outputs to the desired destinations (including the change address created above if applicable), and specify the outputIds and addresses as inputs to the script, and then sign with just one of the two keys. 
   3. After 5 days, the transaction is final and the destination address can spend it arbitrarily.
3. If an attacker uses the "spendNormally" path described above, or if a mistake was made by the owner, the owner can cancel the transaction before the transaction has had 5 days of confirmations by creating a transaction and signing it with both keys. Note that the owner should cancel all outputs in the same transaction in order to retain quantum resistance. 

## Rationale

* `publicKeyA2Hash_` and `publicKeyB2Hash_` are passed instead of the bare publicKeys so they remain quantum resistant. 

## Notes

This could add additional cancelation public key hashes if the ability to cancel some destinations and not others, but it would increase the size of the script and would almost never be necessary. However, a limited number (3 or 4) public key hashes could be added there so transactions that spend the output to a limited number of destinations could take advantage of the ability to cancel some destinations and not others. Doesn't seem worth it tho.

## Script Implementation

The base wallet vault address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  Aggregated multi-signature for: publicKeyA1_, publicKeyB1_
  
Script spend-path: 
  # Push each destination addresses onto the output stack for the associated output.
  <intermediateAddress_, 1, ..., ..., .....> PUSHOUTPUTSTACK
  
  # Push each return addresses onto the output stack for outputs to intermediateAddress_.
  <intermediateAddress_, 2, -1, publicKeyA2Hash_, publicKeyB2Hash_> PUSHOUTPUTSTACK
  
  # Ensure that the input is spent only to intermediateDestination or changeAddress1 with 
  # a maximum fee of 512 times the 300-block median fee-per-byte.
  <1, intermediateAddress_, 300 blocks, 512x> CONSTRAINDESTINATION
  
  # Require a signature for 1 of the 2 keys.
  <publicKeyA1_, ...> CHECKSIG
  <publicKeyB1_, ...> CHECKSIGADD
  <1, ...> LESSTHANOREQUAL
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path scriptSig:  `keySig DUP ... destinationAddressN outputNId`, where
   * the number of address-outputId pairs equals the number of outputs.
   * `keySig` is a signature for either `publicKeyA1_` or `publicKeyB1_`.

In the script spend-path, each output's scriptPubKey would have an output stack appended to it:

* `intermediateAddress_` script spend-path: `destinationAddressN publicKeyB2Hash_ publicKeyA2Hash_`.

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
Key spend-path: 
  <none>
  
Script spend-path 1: 
  # Drop publicKeyA2Hash_ and publicKeyB2Hash_ that were put on the stack from the output stack.
  2DROP

  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  
  # Verify the `destinationPublicKey` against the `destinationAddress` put on the
  # stack by OP_CD in the transaction that created this output.
  verifyPublicKey(..., ...);
  
  # Uses the passed in destinationPublicKey.
  <..., ...> CHECKSIG  
  
Script spend-path 2:
  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> BEFOREBLOCKVERIFY
  
  # Move three values in order: [publicKeyA, publicKeyA2Hash_, publicKeyB, publicKeyB2Hash_].
  2ROT
  ROT
  
  # Verify publicKeyA and save it to the alt stack.
  verifyPublicKey(..., ...)
  <...> TOALTSTACK
  
  # Move the publicKeyB2Hash_ to before publicKeyB
  SWAP
  
  # Verify publicKeyB.
  verifyPublicKey(..., ...)
  
  # Remove destinationAddressN that came from the output stack. 
  NIP
  
  # Require a 2-of-2 signature for two return keys. 
  <..., ...> CHECKSIG
  FROMALTSTACK
  <..., ..., ...> CHECKSIGADD
  <2, ...> NUMEQUAL
  
# Top-of-stack param is the addess, the second param is the public key. Hashes are passed to
# the script (via OP_POSS from the previous transaction) rather than raw public keys to
# retain quantum resistance. 
pseudofunction verifyPublicKey(..., ...):
  OVER
  <...> SHA256
  <...> RIPEMD160
  <..., ...> NUMEQUALVERIFY
```

To spend this script you would use one of the following patterns:

1. Script spend-path 1 witness:  `destinationKeySig destinationPublicKey`
2. Script spend-path 2 witness:  `publicKeyANSig publicKeyB publicKeyA publicKeyBNSig`