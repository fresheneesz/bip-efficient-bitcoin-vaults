# Emulated OP_BBV

The following [OP_BBV](bip-beforeblockverify.md) pattern can be emulated (note that this uses [transaction-sequence notation](https://github.com/fresheneesz/bip-efficient-bitcoin-vaults/blob/main/notation.md):

```
Address:
* BBV Spend-path:     <requirements A> & bbv(x days)
* Non-BBV Spend-path: <requirements B> & absTimelock(x days)
```

The emulation of the above pattern would look like:

```
Address:
* BBV Spend-path: <requirements A>
  -> BBV Test:
     * Success: <requirements A> & relTimelock(1 days)
     * Revoke: <requirements B> & absTimelock(x+1 days)
// Either a relative or absolute timelock here will be equivalent, since the address must be sent to immediately for this emulation to work. 
* Non-BBV Spend-path: <requirements B> & timelock(x days) 
```

* This will only work when the `Address` will have coins sent to it at the same time as the sequence of transactions is created, because time ticks for the absolute time lock regardless of when Address A is sent to. Semantically in this case, the absolute time lock of the Revoke spend-path can be thought of as a timelock relative to when the parent-output was confirmed (tho its really when the top level ancestor of the transaction tree was created).

If `BBV Spend-path` is spent/confirmed before 1 day is up, the `Success` spend-path will be able to be spent first. If `BBV Spend-path` is spent/confirmed after 1 day is up, the `Revoke` spend-path will be able to spend first. In such a situation, its expected that cheaters will simply not spend using the `BBV Spend-path` if they won't be able to send the `Success` transaction subsequently. 

### Use cases

This can be used to create a form of [Succinct Atomic Swap](SAS-with-emulated-bbv-and-witness-secret.md), however its materially inferior to Rubin Somsen's original version.