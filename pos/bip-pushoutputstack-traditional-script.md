# OP_POS with traditional script

[OP_POS](bip-pushoutputstack.md) could be activated less efficiently as a traditional OP_NOPx opcode. Since Taproot has locked in, this option is probably no longer necessary to discuss.

## Specification

The traditional script version of OP_PUSHOUTPUTSTACK redefines opcode OP_NOP6 (0xb5). It does the same as the tapscript version above except that it does not pop stack items and instead of adding the output stack onto the child output's execution stack, it instead requires the `scriptSig` spending the relevant output of the transaction to have particular values on the top of its stack after evaluation. When that UTXO is spent, after evaluation of the UTXO's `scriptSig`, the top three values on the stack are verified to match the "output stack" stored with the UTXO. If they don't match, the transaction is marked invalid.
