# Succinct Atomic Swaps With Emulated OP_BBV Without Altcoin Timelock

This is very similar to [Succinct Atomic Swaps With OP_BBV](SAS-with-op-bbv.md), however it can be done if the altcoin does not support timelocks. Note the same notation is used as in that write up. 

This allows these simple Succinct Atomic Swaps with one chain not supporting timelocks at the expense of backup complexity for the party receiving the coin that doesn't support time locks, Alice, and the ability for Bob to prevent Alice from accessing those coins until Bob spends his coins. 

The transactions could look like the following using [spend-path notation](notation.md):

```
AliceSig
-> BTC to Bob:
   * Bob Success: BobSig & bobSecret & relTimelock(1 day)
   * Alice Revoke: AliceSig & aliceSecret & BobSig
     -> Revoke Address:
        * Alice Revoke Success: AliceSig & aliceSecret & relTimelock(1 day)
        * Alice Revoke Fail: BobSig & absTimelock(2 days)
   
BobSig
-> ALTC to Alice: 
   * Alice Success: AliceSig & bobSecret
   * Bob Revoke: BobSig & aliceSecret
```

## Cases:

### Normal Case:

1. Alice creates the transaction creating the outputs `BTC to Bob` and `revokeAddress`. The absolute time-locks are set relative to this moment. 
2. Bob signs a transaction spending the `Alice Revoke` spend-path to `Revoke Address`.
3. Alice sends "BTC to Bob" transaction
4. Bob sends "ALTC to Alice" transaction
5. Wait 1 day.
6. Bob sends Alice `bobSecret`
7. Bob must watch the chain until he spends `Bob Success`, because Alice might spend `Alice Revoke` at anytime before that.
8. Eventually Bob spends `Bob Success`.

### Bob doesn't send ALTC:

4. After say 6 hours, Alice sends the `Alice Revoke` transaction.
5. After 1 day, Alice spends the `Alice Revoke Success` spend-path to retrieve her bitcoin.

### "Alice Revoke" after Bob sends "ALTC to Alice":

5. After say 6 hours, Alice sends the `Alice Revoke` transaction. This reveals `aliceSecret`.
6. Bob then sends `Bob Revoke` to retrieve his ALTC.
7. After a day, Alice spends `Alice Revoke Success` to retrieve her BTC.

### Bob never reveals `bobSecret` in step 6:

6. Alice must watch the blockchain until Bob spends `Bob Success`.
7. When Bob spends `Bob Success`, that will reveal `bobSecret` allowing Alice to finally claim her coins. 

In this case, Alice must wait indefinitely at step 6 until Bob spends his part. 

## Properties

* In normal cases, only two transactions needed in total - one on each chain.
* In failure cases, four transactions in total will be needed.
* In normal cases, the transaction can be considered complete by both parties after 1 day (or whatever the smaller timeout is). No watching is needed after that time. 
* Recovery for Alice does requires her seed *and* `bobSecret`, which complicates backup procedures and likely reduces their resilience. 
* Alice must keep track of a secret for a period of time until either Bob sends the "ALTC to Alice" transaction (and it is confirmed) or until Alice sends the "Alice Revoke" transaction. In practice, this could take well under an hour (or some small fraction of whatever smaller timeout is chosen). 

## Comparison to SAS without OP_BBV

[Ruben Somsen's SAS](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f) protocol... TBD