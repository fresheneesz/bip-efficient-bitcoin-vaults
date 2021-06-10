## OP_CONSTRAIN DESTINATION Examples

This page shows a number of examples of transactions that might use [OP_CD](bip-constraindestination.md) and how its evaluated in the context of each example.

### Example Transactions

For the following examples, I'll write this information using [javascript-like pseudocode](notation.md) as `OP_CD(sampleWindowFactor, feeFactor, [Address1, Address2,...], {id1: amount1, id2: amount2, ...})`. For example, the above example script would be written as `OP_CD(300 blocks, 7, [D, C], {0: 24345200, ...})`. I'll also use 10 sats/byte as the 300-block median fee rate.

Example Transaction A:

* Inputs:
  * One `input` of 1000 satoshi with `OP_CD(300 blocks, 5, [D, C], {0: 100, 1: 850})`
* Outputs:
  * 100 sat to destination address `D`
  * 850 sat to change address `C`

The `input` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. Since there is only one input, that means the total transaction fee can be no more than 320 sats. The input specifies that it can only send money to addresses `D` and `C`. The operation specifies that 100 sats is sent to output 0 (at address D) and 850 sats to output 1 (at address C) and since `1000 - 850+100 = 50 sats` is less than 320 sats, and the values specified in the OP_CD call don't exceed the values of the outputs, the transaction is valid.

Example Transaction B:

* Inputs:
  * `inputA` with a value of 2000 satoshi.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 5, [D2, C2], {2:100, 3:550})`.
* Outputs:
  * 200 sats to destination address `D1`
  * 1400 sats to change address `C1`
  * 100 sats to destination address `D2`
  * 550 sats to change address `C2`

`inputB` can contribute a maximum of `10 sats/byte * 2^5 = 320 sats` to the fee. The transaction has a total fee of 700 sats. `inputB` specifies that it can only send money to addresses `D2` and `C2`, and that it sends 100 sats to output 2 and 550 sats to output 3. Since those two outputs receive a total of 650 sats, the fee contributed by `inputB` is 350 sats, which is more than 320, the transaction is invalid.

Example Transaction C:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_CD(300 blocks, 5, [D1, C1], {0: 880})`.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 4, [D1], {0: 20})`.
* Outputs:
  * 900 sats to destination address `D1`.

`inputA` can contribute a maximum of `10* 2^5 = 320 sats` to the fee. It specifies sending 880 to output 0, which is less than that output's value, `inputA`'s minimum contribution to the fee is `inputA - D1 = 1200 - 900 = 300 sats`, which is less than 320 sats, and so the immediate evaluation of the operation succeeds for `inputA`.

`inputB` can contribute a maximum of `10 * 2^4 = 160 sats` to the fee. `inputB`'s OP_CD specifies sending only 20 sats to output 0, which means its contributing 980 sats of fee, which is way over its limit, so the transaction is invalid. Note that given the input values and output value, there is no combination of OP_CD output value specification from `inputA` and `inputB` that would allow the transaction to succeed (the output value is too small). 

Example Transaction D:

* Inputs:
  * `inputA` of 1200 satoshi and an `OP_CD(300 blocks, 5, [D1, C1], {0: 880})`.
  * `inputB` of 1000 satoshi and an `OP_CD(300 blocks, 4, [D1, D2], {0: 20, 1:950})`.
  * `inputC` of 7000 satoshi and an `OP_CD(300 blocks, 10, [D2], {1: 5600})`.
* Outputs:
  * 900 sats to destination address `D1`
  * 7000 sats to destination address `D2`

Maximum fee contributions:

* `inputA`:  `10 * 2^5 = 320 sats` 
* `inputB`: `10 * 2^4 = 160 sats`
* `inputC`: `10 * 2^10 = 10,240 sats`

Checks done:

* `inputA`: ` 0 <= 1000 - 880 <= max fee contribution of 320 sats` **OK**
* `inputB`: `0 <= 1000 - (950 + 20) <= max fee contribution of 160 sats` **OK**
* `inputC`: `0 <= 7000 - 5600 <= max fee contribution of 10,240 sats` **OK**
* `D1`: `880 + 20 <= 900` **OK**
* `D2`: `950 + 5600 <= 7000` **OK**

All those checks also succeed, so the transaction is valid.