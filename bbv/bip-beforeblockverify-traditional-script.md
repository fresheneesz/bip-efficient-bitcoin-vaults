# OP_BBV with traditional script

[OP_BBV](bip-beforeblockverify.md) could be less efficiently implemented as a traditional OP_NOPx opcode. This seems no longer necessary to discuss, since Taproot has locked in.

## Specification

OP_BEFOREBLOCKVERIFY (*OP_BBV for short*) redefines opcode OP_NOP5 (0xb4). It does the following:

* The same as OP_BEFOREBLOCKVERIFY above but does not pop the stack. 
