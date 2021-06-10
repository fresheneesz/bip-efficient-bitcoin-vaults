# Wallet Vault 3

This is a slight upgrade from [Wallet Vault 2](walletVault2.md). It allows spending change immediately without waiting for a timeout. Note that the notation used on this page [can be found here](notation.md)

```
Wallet Vault:
* Immediate: a1Key & b1Key
* Normal:    (a1Key || b1Key)
-> Intermediate Address:
   * Receiver Spend: destinationSig & timelock(5 days)
   * Sender Cancel:  a1Key & b1Key & bbv(5 days)
-> Change Address:
   * Immediate:    change1Sig & change2Sig
   * Normal:       change1Sig & change2Sig
                     & (feeContributed(0) || locktime(5 days)
     -> Intermediate Address &|| Change Address
-> Intermediate Address &|| Change Address
```

## Properties

* Can send to multiple destinations in a transaction: true
* Can omit the change address: true
* Can spend change immediately: true
* Can consolidate inputs: true. Arbitrary number of wallet vault inputs can be spent in the same transaction and send to the same destination/change address. 
* Canceling each destination / change send either individually or together are both options: **false**. However, in the scenario of an attacker, this ability isn't needed. 
* Finalization time: **5 days**. Sending to an address that expects to be able to spend the transaction earlier than this won't work. This is a fundamental limitation of a wallet vault.

## Addresses

The address scripts are written in [javascript-like pseudo-code](notation.md).

### Base Wallet Vault Address

The first address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  spendImmediately(keyA1Sig, keyB1Sig, publicKeyA1_, publicKeyB1_)
Script spend-path: 
  spendNormally(signature, outputAddressAssociations, 512x median fee-rate)

function spendImediately(keyA1Sig, keyB1Sig, publicKeyA, publicKeyB):
  verify(checkSigs([[keyA1Sig, publicKeyA], [keyB1Sig, publicKeyB]]) >= 2)

function spendNormally(signature, outputAddressAssociations, maxFee):
  let outputValueMap = {}
  for a in outputAddressAssociations:
    outputValueMap[a.outputId] = [a.address]
  pushOutputStack(intermediateAddress_, outputValueMap)
  pushOutputStack(
    intermediateAddress_, {-1: [publicKeyA2Hash_, publicKeyB2Hash_]}
  )
  constrainDestination([intermediateAddress_, changeAddress_], 300 blocks, maxFee)
  verify(checkSigs([[signature, publicKeyA1_], [signature, publicKeyB1_]]) >= 1)
```

### Intermediate Address

The `intermediateAddress_` referenced above is a static address who's scripts don't change depending on the wallet vault or its keys. In fact, all wallet vaults of this specific form would share this address. Its execution is differentiated only by the output stack values placed into it by the parent input's OP_POS execution. It would have the following spend paths:

```
Key spend-path:
  <none>
Script spend-path 1:
  destinationSpend(destinationSignature, destinationPublicKey)
Script spend-path 2:
  cancelTransaction(keyASig, publicKeyA, keyBSig, publicKeyB)

function destinationSpend(destinationSignature, destinationPublicKey):
  // Transaction creating this output must have been confirmed at LEAST 5 days ago.
  checkSequenceVerify(5 days)
  let destinationAddress = getOutputStackItem(0)
  verifyPublicKey(destinationPublicKey, destinationAddress)
  verify(checkSigs([[destinationSignature, destinationPublicKey]]) >= 1)
  
function cancelTransaction(keyASig, publicKeyA, keyBSig, publicKeyB):
  // Transaction creating this output must have been confirmed at MOST 5 days ago.
  beforeBlockVerify(5 days)
  let publicKeyAHash = getOutputStackItem(2)
  let publicKeyBHash = getOutputStackItem(3)
  verifyPublicKey(publicKeyA, publicKeyAHash)
  verifyPublicKey(publicKeyB, publicKeyBHash)
  verify(checkSigs(2, [[keyASig, publicKeyA], [keyBSig, publicKeyB]]) >= 2)
```

### Change Address

The `changeAddress_` is another static address that doesn't contain any data unique to the wallet (just like `intermediateAddress_` above. It has two very similar spend paths, one 

```
Key spend-path:
  spendChangeImmediately(changeKeyA1Sig, changeKeyB1Sig, publicKeyA, publicKeyB)
Script spend-path 1:
  spendChange(true, changeKeySig, changePublicKeyA, changePublicKeyB)
Script spend-path 2:
  spendChange(false, changeKeySig, changePublicKeyA, changePublicKeyB)
  
function spendChangeImmediately(changeKeyA1Sig, changeKeyB1Sig, publicKeyA, publicKeyB):
  verify(checkSigs([[keyA1Sig, changePubkeyA_], [keyB1Sig, changePubkeyB_]]) >= 2)

function spendChange(beforeCooldown, changeKeySig, outputAddressAssociations):
  let maxFee = 512x median fee-rate
  if(beforeCooldown):
    maxFee = 0
  else:
    // Transaction creating this output must have been confirmed at LEAST 5 days ago.
    checkSequenceVerify(5 days)
  spendNormally(signature, outputAddressAssociations, maxFee)
```

## Creation

To create receiving addresses for the wallet, create a *basic wallet vault address* where `changeAddress_` is unspendable. Then iteratively create *basic wallet vault address*es with the last created address is used as `changeAddress_`. The number of iterations can be thousands or millions to ensure that immediately spending change is always possible. 

## Spending

The way funds in a *base wallet vault address* would be used is the following:

1. If the owner wants to spend the funds immediately in arbitrary ways, they create a transaction and sign it with both keys.
2. If the owner wants to spend normally:
   1. Then they create a transaction with outputs to the desired destinations, and specify the outputIds and addresses as inputs to the script, and then sign with just one of the two keys. 
   2. After 5 days, the transaction is final and the destination address can spend it arbitrarily.
3. If an attacker uses the "spendNormally" path described above, or if a mistake was made by the owner, the owner can cancel the transaction before the transaction has had 5 days of confirmations by creating a transaction and signing it with both keys. Note that the owner should cancel all outputs in the same transaction in order to retain quantum resistance. 

## Script Implementation

The base wallet vault address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  Aggregated multi-signature for: publicKeyA1, publicKeyB1
  
Script spend-path: 
  spendNormally(512x, ....., ..., ..., ...)
  
pseudofunction spendNormally(maxFee, ....., ..., ..., ...):
  # Push each destination addresses onto the output stack for the associated output.
  <intermediateAddress, 1, ..., ..., .....> PUSHOUTPUTSTACK
  
  # Push each return addresses onto the output stack for outputs to intermediateAddress.
  <intermediateAddress, 2, -1, publicKeyA2Hash, publicKeyB2Hash>
  PUSHOUTPUTSTACK
  
  # Ensure that the input is spent only to intermediateAddress or changeAddress with 
  # a maximum fee of 512 times the 300-block median fee-per-byte.
  <2, intermediateAddress, changeAddress, 300 blocks, maxFee> CONSTRAINDESTINATION
  
  # Require a signature for 1 of the 2 keys.
  <publicKeyA1, ...> CHECKSIG
  <publicKeyB1, ...> CHECKSIGADD
  <1, ...> LESSTHANOREQUAL
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path witness:  `keySig DUP ... destinationAddressN outputNId`, where
   * the number of address-outputId pairs equals the number of outputs.
   * `keySig` is a signature for either `publicKeyA1` or `publicKeyB1`.

In the script spend-path, each output's output stack would have an output stack appended to it:

* `intermediateAddress` outputs: 
  * `destinationAddressN publicKeyA2Hash publicKeyB2Hash`.
* `intermediateChangeAddress` outputs: 
  * `changeAddress`

The `intermediateAddress` referenced above is an address with the following spend paths:

```
Key spend-path: 
  <none>
  
Script spend-path 1: 
  # Drop publicKeyA2Hash and publicKeyB2Hash that were put on the stack
  # from the output stack.
  2DROP

  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  
  # Verify the `destinationPublicKey` against the `destinationAddress` that came from the
  # output stack.
  verifyPublicKey(..., ...);
  
  # Uses the passed in destinationPublicKey.
  <..., ...> CHECKSIG  
  
Script spend-path:
  # Transaction creating this output must have been confirmed at MOST 5 days ago.
  <5 days> BEFOREBLOCKVERIFY
  
  # Move three values to be in the order:
  # [publicKeyA, publicKeyA2Hash, publicKeyB, publicKeyB2Hash].
  2ROT
  ROT
  
  # Verify publicKeyA and save it to the alt stack.
  verifyPublicKey(..., ...)
  <...> TOALTSTACK
  
  # Move the publicKeyB2Hash to before publicKeyB
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
3. Script spend-path 2 witness:  `publicKeyBNSig publicKeyA publicKeyB publicKeyANSig`

The `changeAddress` referenced above is an address with the following spend paths:

```
Key spend-path: 
  Aggregated multi-signature for: changePublicKeyA1_, changePublicKeyB1_
  
Script spend-path 2:
  spendNormally(-8, ....., ..., ..., ...)
  
Script spend-path 3:
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  spendNormally(512x, ....., ..., ..., ...)
  
```

To spend this script you would use one of the following patterns:

1. Script spend-path witness:  `changeKeySig changePublicKey`

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `changePublickeyA1B1Sig`
2. Script spend-path witness:  `changeKeySig DUP ... destinationAddressN outputNId`, where
   * the number of address-outputId pairs equals the number of outputs.
   * `keySig` is a signature for either `changePublicKeyA1` or `changePublicKeyB1`.