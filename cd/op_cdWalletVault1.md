# OP_CD Wallet Vault

This is a basic wallet vault script using [OP_CD](bip-constraindestination.md) (without OP_POS, OP_BBV, or OP_LFC). It has a lot of limitations, but it is simple to understand and it is has fewer limitations than wallet vaults created with op_checktemplateverify. 

## Properties

* Can send to arbitrarily many destinations in a transaction: true
* Can always limit the transaction to one change output: true.
* Can omit the change address: true
* Can spend change immediately: true
* Can consolidate inputs: **false**. Each output must be spent to a different intermediate address, after which the outputs can be combined in a single output.
* Canceling each destination / change send either individually or together are both options: true.
* Finalization time: **5 days + 6 confirmations**. Sending to an address that expects to be able to spend the transaction earlier than this won't work. This is a fundamental limitation of a wallet vault.
* An attacker that has gained access to keyA can steal funds that are sent to `intermediateAddress_` once sent from the main wallet vault address.

## Addresses

The address scripts are written in [javascript-like pseudo-code](../notation.md).

### Main Wallet Vault Address

The first address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  spendImmediately(keyASig, keyBSig)
Script spend-path: 
  spendNormally(keyASig, outputValuesMap)

function spendImediately(keyASig, keyBSig):
  verify(checkSigs([[keyASig, publicKeyA1_], [keyBSig, publicKeyB1_]) >= 2)

function spendNormally(keyASig, outputValuesMap):
  constrainDestination([intermediateAddress_, changeAddress_], outputValuesMap)
  verify(checkSigs([[keyASig, publicKeyA1_]]) >= 1)
```

### Wallet Vault Change Address

The `changeAddress_` referenced above is identical to a main wallet vault address, including its own `changeAddress_`. There needs to be some base change address that does *not* contain a `changeAddress_`. This can be a chain of, for example, 1000 addresses or 1 million so it becomes unlikely or at least very rare that the user will run into the base address and be forced to spend the entire output to the `intermediateAddress_`.

### Intermediate Address

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
Key spend-path:
  spendImmediately(keyASig, keyBSig)
Script spend-path:
  spendToDestination(keyASig)

function spendToDestination(keyASig):
  verify(checkSigs([[keyASig, publicKeyA1_]]) >= 1)
```

### Creation

1. Create an *intermediate address*,
2. Create a base *Main Wallet Vault Address* that omits the `changeAddress_` and uses the above *intermediate address* as the `intermediateAddress_`.
3. Repeat steps 1 and 2 to create N more *Main Wallet Vault Addresses*, each of which use a new *intermediate address* and the previously created *Main Wallet Vault Address* as the `changeAddress_`. 

## Spending

The way funds in a *base wallet vault address* would be used is the following:

1. If the owner wants to spend the funds immediately in arbitrary ways, they create a transaction and sign it with both keys.
2. If the owner wants to spend normally:
   1. They create a transaction with two outputs: one to the `intermediateAddress_` and optionally one to the `changeAddress_`, and then sign with key A.
   3. After 5 days, the user can spend the transaction with key A to the desired destination.
3. If an attacker uses the "spendNormally" path described above, or if a mistake was made by the owner, the owner can cancel the transaction before the transaction has had 5 days of confirmations by creating a transaction and signing it with both keys.

## Script Implementation

These scripts are written with [Script pseudocode notation](../notation.md).

The main wallet vault address in a wallet vault would have the following spend paths:

```
Key spend-path:    
  Aggregated multi-signature for: publicKeyA1, publicKeyB1

Script spend-path:  
  # Ensure that the input is spent only to intermediateDestination or changeAddress.
  <2, intermediateAddress_, changeAddress_, _, _, _, ...> CONSTRAINDESTINATION
  
  # Require a signature for key A.
  <publicKeyA1, _> CHECKSIGVERIFY
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path witness:  `keyA1Sig`

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
Key spend-path: 
  Aggregated multi-signature for: publicKeyA1, publicKeyB1
  
Script spend-path: 
  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  
  # Verify the signature.
  <_, publicKeyA1_> CHECKSIGVERIFY
```

To spend this script you would use one of the following patterns:

1. Key spend-path witness: `keyA1B1Sig`
2. Script spend-path witness: `keyA1Sig`
