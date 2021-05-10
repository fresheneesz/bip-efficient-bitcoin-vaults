# Succinct Atomic Swaps With Emulated OP_BBV Without Altcoin Timelock

The [SAS with OP_BBV without Altcoin Timelock](SAS-with-op-bbv-and-witness-secret.md) can be emulated using [this emulation technique](bip-beforeblockverify.md#emulation-with-absolute-and-relative-timelocks). This has the downside that one party (Bob) must watch the blockchain until he spends the coins he received in the swap. 

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
* In failure cases, up to five transactions in total may be needed.
* Alice can consider the transaction complete after 1 day.
* Bob can only consider the transaction fully complete when he spends the `Bob Success` spend-path. Until then, Bob must watch the chain for Alice attempting to cheat. 
* Recovery for Bob and Alice both require `bobSecret`, which complicates backup procedures and likely reduces their resilience. 
* Alice must keep track of a `aliceSecret` for a period of time until either Bob sends the "ALTC to Alice" transaction (and it is confirmed) or until Alice sends the "Alice Revoke" transaction. In practice, this could take well under an hour (or some small fraction of whatever smaller timeout is chosen). 

## Comparison to SAS without OP_BBV

[Ruben Somsen's SAS](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f) protocol... TBD