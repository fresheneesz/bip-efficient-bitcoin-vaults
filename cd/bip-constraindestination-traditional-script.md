# OP_CD with traditional script

[OP_CD](bip-constraindestination.md) could be activated less efficiently as a traditional OP_NOPx opcode. Since Taproot is all but locked in, this option is probably no longer necessary to discuss.

## Specification

The traditional script version of OP_CONSTRAINDESTINATION redefines opcode OP_NOP7 (0xb6). This does the same as OP_CONSTRAINDESTINATION above, except that it does not pop any stack items.
