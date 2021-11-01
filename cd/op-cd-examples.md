## OP_CONSTRAIN DESTINATION Examples

This page shows a number of examples of transactions that might use [OP_CD](bip-constraindestination.md) and how its evaluated in the context of each example.

### Example Transactions

For the following examples, I'll write this information using [javascript-like pseudocode](../notation.md) as `OP_CD([Address1, Address2,...], {id1: amount1, id2: amount2, ...})`. For example, the above example script would be written as `OP_CD({0: 24345200, ...}, [D, C])`.

**Example Transaction A:**

* Inputs:
  * One `input` of 1000 satoshi with `OP_CD({0: 100, 1: 850}, [D, C])`
* Outputs:
  * 100 sat to destination address `D`
  * 850 sat to change address `C`

The input specifies that it can only send money to addresses `D` and `C`. The operation specifies that 100 sats is sent to output 0 (at address D) and 850 sats to output 1 (at address C) and since `1000 > 850+100` , and the values specified in the OP_CD call don't exceed the values of the outputs, the transaction is valid.

**Example Transaction B:**

* Inputs:
  * `inputA` with a value of 2000 satoshi.
  * `inputB` of 600 satoshi and an `OP_CD({2:100, 3:550}, [D2, C2])`.
* Outputs:
  * 200 sats to destination address `D1`
  * 1400 sats to change address `C1`
  * 100 sats to destination address `D2`
  * 550 sats to change address `C2`

`inputB` specifies that it can only send money to addresses `D2` and `C2`, and that it sends 100 sats to output 2 and 550 sats to output 3. This totals to 650 sats of claims. Since the UTXO is only 600 sats, the transaction is invalid.

**Example Transaction C:**
In this example, we're going to add fee limitations.

* Inputs:
  * `inputA` of 1200 satoshi, with `OP_CD({0: 880}, [D1, C1])` and a 320 sat fee limit.
  * `inputB` of 1000 satoshi with `OP_CD([D1], {0: 20})` and a 160 sat fee limit.
* Outputs:
  * 900 sats to destination address `D1`.

It specifies sending 880 to output 0, which is less than that output's value, `inputA`'s minimum contribution to the fee is `inputA - D1 = 1200 - 900 = 300 sats`, which is less than the 320 sat fee limit, and so the immediate evaluation of the operation succeeds for `inputA`.

`inputB`'s OP_CD specifies sending only 20 sats to output 0, which means its contributing 980 sats of fee, which is way over its 160 sat fee limit, so the transaction is invalid. Note that given the input values and output value, there is also no combination of OP_CD output value specification from `inputA` and `inputB` that would allow the transaction to succeed (the output value is too small). 

**Example Transaction D:**

* Inputs:
  * `inputA` of 1200 satoshi with `OP_CD([D1, C1], {0: 880})` and a 320 sat fee limit.
  * `inputB` of 1000 satoshi with `OP_CD([D1, D2], {0: 20, 1:950})` and a 160 sat fee limit.
  * `inputC` of 7000 satoshi with `OP_CD([D2], {1: 5600})` and a 10,240 sat fee limit.
* Outputs:
  * 900 sats to destination address `D1`
  * 7000 sats to destination address `D2`

Checks done:

* `inputA`: ` 0 <= 1000 - 880 <= max fee contribution of 320 sats` **OK**
* `inputB`: `0 <= 1000 - (950 + 20) <= max fee contribution of 160 sats` **OK**
* `inputC`: `0 <= 7000 - 5600 <= max fee contribution of 10,240 sats` **OK**
* `D1`: `880 + 20 <= 900` **OK**
* `D2`: `950 + 5600 <= 7000` **OK**

All those checks also succeed, so the transaction is valid.