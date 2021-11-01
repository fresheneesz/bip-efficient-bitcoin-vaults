# Notation

This file describes the notation used in these BIPs for a couple different kinds of things.

## Transaction Sequence Notation

This notation is used to describe sequences of transactions and what spend paths they have. Spending requirements are written as a logical operation with these being some conventions:

* `someoneAddr` - Means fulfilling the witness requirements of that address. 
* `someoneSig` - Means a signature is required.
* `someoneSecret` - Means that revealing a secret is required.
* `timelock(x days/hours/etc)` - Means that much time must have been waited since the transaction was confirmed. 
* `absTimelock(x days/hours/etc)` - Like `timelock` except is an absolute timelock, where the time is probably dependent on the protocol (sequence of transactions) being decided on by the parties involved.
* `&` - And operator.
* `||` - Or operator. 
* `&||` - And/or operator. Useful if you want to specify that a user can send a transaction that contains multiple outputs. `a &| b` is equivalent to `a || b || a & b`

An example of spending requirements: `AliceSig & bobSecret || CarolAddr`

The following describes a simple situation where one output can be spent to another output with particular spending requirements. These can be chained into many steps. Any output/address that doesn't list more transactions from that point indicates that it can be spend to anywhere if the spending requirements are fulfilled. 

```
Address/Output 1: spending requirements 1 -> Address/Output 2: spending requirements 2
```

The following describes spending one UTXO to one of two outputs. These can be chained to add as many possible outputs as needed.

```
Address/Output 1: spending requirements
-> Address/Output 1.1: spending requirements
   -> Address/Output 1.1.1: spending requirements
   -> Address/Output 1.1.2: spending requirements
-> Address/Output 1.2: spending requirements
   -> Address/Output 1.2.1: spending requirements
```

Note that the listed destinations are exhaustive, implying that no other destinations may be sent to. This may be accomplished by either pre-signed transactions, covenants, etc. 

In cases where an address/output has multiple spend-paths, the following notation can be used:

```
Address/Output Name 1:
* Spend-path Name: [spending requirements 1]
-> Output Name 2: [spending requirements]
-> Output Name 3: [spending requirements]
* Spend-path Name: [spending requirements 2]
-> Output Name 4: [spending requirements]
... 
```

Note that the following two scripts are equivalent:

```
Address X: AliceKey
-> BobKey || CarolKey
```

```
Address X: AliceSig
* BobKey
* CarolKey
```

In cases where a transaction may or must contain multiple outputs, already-defined addresses can be combined using logical operators on the already-defined addresses. For example:

```

Multisig Address: AliceSig & BobSig
-> AliceKey & BobSecret
   -> AliceKey &|| BobKey &|| MultisigAddress
```

The three forms above can be mixed and matched depending on what is most succinct. For example:

```
AliceSig
-> BobKey -> CarolKey
-> CarolKey
   -> DarrelKey
-> Multisig Address:
   * AliceKey & BobKey
   -> CarolKey -> BobKey
   * AliceKey & CarolKey & aliceSecret
```

## JS-like Pseudocode Notation

This notation uses javascript-like pseudocode to describe a script. Function calls who's names match an opcode usually have this meaning:

```
opcode(topOfStack, secondToTop, etc)
```

And functions are written like this (for succinctness):

```
function functionName(param1, param2, etc):
  // Lines
  // of
  // code.
```

Sometimes values like `5 days` or something like that will be written that are closer to their semantic meaning, but have a different representation in actual scripts. Everything else can be considered more-or-less standard javascript.

Variables that end in an underscore (eg `intermediateAddress_`) represent hard coded values in the script.

These are utility functions that are used in some of these scripts:

```
// Returns the number of keys with matching signatures in the witness.
function checkSigs(sigPubKeyPairs);

// Verifies that the public key matches the address.
function verifypublicKey(publicKey, address):
  verify(ripeMd160(sha256(publicKey)) == address)
  
// Pushes the values onto the scriptSig for the output corresponding with outputId.
// outputValueMap is a map that maps outputId to a list of values to push onto the output stack.
function pushOutputStack(address, outputValueMap);

// Requires that the output only send money to the listed addresses, with a maximum fee determined
// by window-length and maxFeeContribution.
function constrainDestination(addresses, windowLength, maxFeeContribution);

// Verifies that the block this transaction is mined in has a height less than blockheight.
function beforeBlockVerify(blockheight);
```

## Script Pseudocode Notation

This notation is very close to actual Bitcoin Script. The notations for opcode calls looks like:

```
<topOfStack, secondToTop, etc> OPCODE
```

This represents the following script:

```
etc
secondToTop
topOfStack
OP_OPCODE
```

An underscore (`_`_ will be put in place of a parameter when that value is expected to already be on the stack (eg from the witness, or from previously run script lines). The `_` should be used for pseudofunctions just like for opcodes.  This could look like the following:

```
<1, 2> ADD
<3, _> ADD
```

which represents the following script:

```
2
1
ADD
3
ADD
```

Sometimes a function will be used, which looks like:

```
# Definition
pseudofunction functionName(topOfStack, secondToTop, etc):
  # Script Pseudocode Line
  # Another Script Pseudocode Line
  # Etc
  
# Call
functionName(topOfStack, secondToTop, etc)
```

