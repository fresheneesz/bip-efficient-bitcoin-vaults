# Succinct Atomic Swaps With OP_BBV Without Altcoin Timelock

This is very similar to [Succinct Atomic Swaps With OP_BBV](SAS-with-op-bbv.md), however it can be done if the altcoin does not support timelocks. Note the same notation is used as in that write up. 

This allows these simple Succinct Atomic Swaps with one chain not supporting timelocks at the expense of backup complexity for the party receiving the coin that doesn't support time locks, Alice, and the ability for Bob to prevent Alice from accessing those coins until Bob spends his coins. 

The transactions could look like the following:

```
BTC to Bob: AliceSig 
  -> Bob Success:  BobSig & bobSecret & timelock(1 day) 
  -> Alice Revoke: AliceSig & aliceSecret & bbv(1 day)

ALTC to Alice: BobSig 
  -> Alice Success: AliceSig & bobSecret
  -> Bob Revoke:    BobSig & aliceSecret
```

Normal Case:

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. Wait 1 day.
4. Bob sends Alice `bobSecret`

Bob doesn't send LTC:

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. After say 6 hours, Alice sends the Revoke transaction.

"Alice Revoke" after Bob sends "LTC to Alice":

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. After say 6 hours, Alice sends the Revoke transaction. This reveals aliceSecret.
4. Bob then sends "Bob Revoke".

Bob never reveals `bobSecret` in step 4:

1. Alice sends "BTC to Bob" transaction
2. Bob sends "ALTC to Alice" transaction
3. Wait 1 day.
4. Alice must watch the blockchain.
5. When Bob spends "Bob Success", that will reveal `bobSecret` allowing Alice to finally claim her coins. 

## Properties

* In normal cases, only two transactions needed in total - one on each chain.
* In failure cases, four transactions in total will be needed.
* In normal cases, the transaction can be considered complete by both parties after 1 day (or whatever the smaller timeout is). No watching is needed after that time. 
* Recovery for Alice does requires her seed *and* `bobSecret`, which complicates backup procedures and likely reduces their resilience. 
* Alice must keep track of a secret for a period of time until either Bob sends the "ALTC to Alice" transaction (and it is confirmed) or until Alice sends the "Alice Revoke" transaction. In practice, this could take well under an hour (or some small fraction of whatever smaller timeout is chosen). 

## Comparison to SAS without OP_BBV

[Ruben Somsen's SAS](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f) protocol... TBD