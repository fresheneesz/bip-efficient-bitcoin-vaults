# Succinct Atomic Swaps With  Emulated OP_BBV

The [SAS with OP_BBV](SAS-with-op-bbv.md) can be emulated using [this emulation technique](bip-beforeblockverify.md#emulation-with-absolute-and-relative-timelocks). This has the downside that one party (Bob) must watch the blockchain until he spends the coins he received in the swap. 

The transactions could look like the following using [spend-path notation](notation.md):

```
AliceSig
-> BTC to Bob:
   * Bob Success: BobSig & relTimelock(1 day)
   * Alice Revoke: AliceSig & aliceSecret & BobSig
     -> Revoke Address:
        * Alice Revoke Success: AliceSig & aliceSecret & relTimelock(1 day)
        * Alice Revoke Fail: BobSig & absTimelock(2 days)
   
BobSig
-> ALTC to Alice: 
   * Alice Success: AliceSig & relTimelock(2 day)
   * Bob Revoke: BobSig & aliceSecret
```

## Cases

### Normal Case

1. Alice creates the transaction creating the outputs `BTC to Bob` and `revokeAddress`. The absolute time-locks are set relative to this moment. 
2. Bob signs a transaction spending the `Alice Revoke` spend-path to `Revoke Address`.
3. Alice sends "BTC to Bob" transaction
4. Bob sends "ALTC to Alice" transaction
5. Wait 1 day.
6. Bob must watch the chain until he spends using the `Bob Success` spend-path, because Alice might spend `Alice Revoke` at any time before that. 
7. Eventually Bob spends the output using the `Bob Success` spend path.

### Bob doesn't send the ALTC after step 3

4. After say 6 hours, Alice sends the transaction spending `Alice Revoke`.
5. After another day, Alice spends using the `Alice Revoke Success` to retrieve her BTC. 

### `Alice Revoke` after Bob sends `ALTC to Alice` in step 4:

5. After say 6 hours, Alice sends the `Alice Revoke` transaction. This reveals `aliceSecret`.
6. Bob then spends `Bob Revoke` to retrieve his ALTC.
7. A. Alice would then probably spend the `Alice Revoke Success` path to retrieve the BTC. 
   B. If Alice does not do step 7, after another day, Bob can then spend `Alice Revoke Fail` to take the BTC as well as the ALTC.

So at the end of this, Bob gets back the ALTC and Alice gets back the BTC (minus fees). 

### `Alice Revoke` after 1 day (step 5)

6. Alice sends the `Alice Revoke` transaction. This reveals `aliceSecret`.
7. Bob then spends `Bob Revoke` to retrieve his ALTC.
8. After another day, Bob spends `Alice Revoke Fail`.

At the end of this, Bob gets both the ALTC and the BTC. Its in Alice's best interests to never do this. 

## Properties

* In normal cases, only two transactions needed in total - one on each chain.
* In failure cases, up to five transactions in total may be needed.
* Alice can consider the transaction complete after 1 day.
* Bob can only consider the transaction fully complete when he spends the `Bob Success` spend-path. Until then, Bob must watch the chain for Alice attempting to cheat. 
* Recovery normally does not require any secrets - recovery should be possible from just the seed.
* Alice must keep track of a secret for a period of time until either Bob sends the "ALTC to Alice" transaction (and it is confirmed) or until Alice sends the "Alice Revoke Success" transaction. In practice, this could normally take well under an hour (or some small fraction of whatever smaller timeout is chosen). 

## Comparison to SAS without OP_BBV

[Ruben Somsen's SAS](https://gist.github.com/RubenSomsen/8853a66a64825716f51b409be528355f) protocol... TBD