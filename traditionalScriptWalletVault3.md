# Wallet Vault 3

This is a traditional script variant of [Tapscript Wallet Vault 3](tapscriptWalletVault3.md). The js-like pseudocode, properties, etc are all the same, so just the bitcoin scripts are shown.

## Script Implementation

The base wallet vault address in a wallet vault would have the following spend paths:

```
IF 
  <2, publicKeyA1_, publicKeyB1_, 2, ..., ...> CHECKMULTISIGVERIFY
ELSE
  # Push each return addresses onto the output stack for outputs to intermediateAddress_.
  <intermediateAddress_, 3, -1, changeAddress_, publicKeyA2Hash_, publicKeyB2Hash_>
  PUSHOUTPUTSTACK
  2DROP
  2DROP
  2DROP
  
  # Ensure that the input is spent only to intermediateDestination or changeAddress1 with 
  # a maximum fee of 512 times the 300-block median fee-per-byte.
  <1, intermediateAddress_, 300 blocks, 512x> CONSTRAINDESTINATION
  2DROP
  2DROP
  
  # Require a signature for 1 of the 2 keys.
  <2, publicKeyA1_, publicKeyB1_, 1, ...> CHECKMULTISIGVERIFY
  
  # Push each destination addresses onto the output stack for the associated output. This
  # must be last to allow for a variable number of inputs
  <intermediateAddress_, 1, ..., ..., .....> PUSHOUTPUTSTACK
  
  # All arguments to OP_POS are intentionally left on the stack. Are non-zero to signal success.
ENDIF
```

To spend this script you would use one of the following scriptSig patterns:

1. `ignored keyB1Sig keyA1Sig 1`
2. `... destinationAddressN outputNId keySig 0`, where
   * the number of address-outputId pairs equals the number of outputs.
   * `keySig` is a signature for either `publicKeyA1_` or `publicKeyB1_`.

In the script spend-path, each output's output stack would be:

* `intermediateAddress_` script spend-path: 
  * `changeAddress_ publicKeyA2Hash_ publicKeyB2Hash_ destinationAddressN`.

The `intermediateAddress_` referenced above is an address with the following spend paths:

```
# Grab the first if condition.
5 OP_ROLL
IF
  # Move up the destinationPublicKey and destinationKeySig, and position the
  # destinationKeySig destinationPublicKey on top.
  2ROT
  ROT 

  # Transaction creating this output must have been confirmed at LEAST 5 days ago.
  <5 days> CHECKSEQUENCEVERIFY
  DROP
  
  # Verify the `destinationPublicKey` against the `destinationAddress` that came from the
  # output stack.
  verifyPublicKey(..., ...);
  
  # Uses the passed in destinationPublicKey.
  <..., ...> CHECKSIG  
  
  # changeAddress_, publicKeyA2Hash_, and publicKeyB2Hash_ are all intentionally left on the stack.
  
ELSE
  # Grab the second if condition.
  5 OP_ROLL 
  IF
    # Drop destinationAddressN that was put on the stack from the output stack.
    DROP
    # Drop publicKeyA2Hash_ and publicKeyB2Hash_ that were put on the stack from the output stack.
    2DROP

    # Verify the `changePublicKey` against the `changeAddress` that came from the output stack.
    verifyPublicKey(..., ...);

    # Uses the passed in changePublicKey.
    <..., ...> CHECKSIG  
  
  ELSE
    # Transaction creating this output must have been confirmed at MOST 5 days ago.
    <5 days> BEFOREBLOCKVERIFY
    DROP

    # Drop destinationAddressN that was put on the stack from the output stack.
    DROP
    # Move three values so they're in the order:
    # [publicKeyB2Hash_, publicKeyB, publicKeyA, publicKeyA2Hash_].
    2ROT
    ROT

    # Verify publicKeyB and save it to the alt stack.
    verifyPublicKey(..., ...)
    <...> TOALTSTACK

    # Move the publicKeyA2Hash_ to before publicKeyA.
    SWAP

    # Verify publicKeyA.
    verifyPublicKey(..., ...)

    # Remove changeAddress_ that came from the output stack. 
    NIP
    
    # Add back publicKeyB.
    OP_FROMALTSTACK
    
    # Move the stack items into the order required b CHECKMULTISIGVERIFY
    2
    OP_ROT
    OP_ROT
    OP_SWAP

    # Require a 2-of-2 signature for two return keys. 
    <2, ..., ..., ..., ..., ...> CHECKMULTISIGVERIFY
  ENDIF
ENDIF
  
# Top-of-stack param is the addess, the second param is the public key. Hashes are passed to
# the script (via OP_POSS from the previous transaction) rather than raw public keys to
# retain quantum resistance. 
pseudofunction verifyPublicKey(..., ...):
  OVER
  <...> SHA256
  <...> RIPEMD160
  <..., ...> NUMEQUALVERIFY  
  
```

To spend this script you would use one of the following scriptSig patterns:

1. `destinationKeySig destinationPublicKey 1`
2. `changeKeySig changePublicKey 1 0`
3. `ignored publicKeyBNSig publicKeyA publicKeyB publicKeyANSig 0 0`