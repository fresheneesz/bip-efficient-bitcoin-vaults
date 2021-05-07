# Succinct Atomic Swaps With OP_BBV

Using two slightly asymmetric transactions each using timelocks and [OP_BEFOREBLOCKVERIFY](bip-beforeblockverify.md), the atomic swap can happen with a single transaction on each chain, and recovery normally does not require storing any secrets.

The transactions could look like the following:

```
// Notation
// Output-Name: [source spending requirements]
//   -> [output spend-path-1 requirements]
//   -> [output spend-path-2 requirements]
// ... 
// Note that the listed destinations are exhaustive, implying that no other destinations may be
// sent to.

BTC to Bob: AliceSig 
  -> Bob Success:  BobSig & timelock(1 day) 
  -> Alice Revoke: AliceSig & aliceSecret & bbv(1 day)

ALTC to Alice: BobSig 
  -> Alice Success: AliceSig & timelock(2 days)
  -> Bob Revoke:    BobSig & aliceSecret & bbv(2 days) 
```

Note that the `bbv(2 days)` in the `Bob Revoke` transaction isn't strictly necessary, since aliceSecret should never be exposed after 1 day, however it is nice to close out the possibility that Alice might leak aliceSecret sometime in the future which could allow Bob to steal her funds. 

Normal Case:

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. Wait 1 day.

Bob doesn't send the ALTC:

1. Alice sends "BTC to Bob" transaction
3. After say 6 hours, Alice sends the Revoke transaction.

"Alice Revoke" after Bob sends "ALTC to Alice":

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. After say 6 hours, Alice sends the Revoke transaction. This reveals aliceSecret.
4. Bob then sends "Bob Revoke".

## Properties

* In normal cases, only two transactions needed in total - one on each chain.
* In failure cases, four transactions in total will be needed.
* In normal cases, the transaction can be considered complete by both parties after 1 day (or whatever the smaller timeout is). No watching is needed after that time. 
* Recovery normally does not require any secrets - recovery should be possible from just the seed.
* Alice must keep track of a secret for a period of time until either Bob sends the "ALTC to Alice" transaction (and it is confirmed) or until Alice sends the "Alice Revoke" transaction. In practice, this could take well under an hour (or some small fraction of whatever smaller timeout is chosen). 

## Comparison to SAS without OP_BBV

[Ruben Somsen's SAS](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f) protocol... TBD