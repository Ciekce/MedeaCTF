# MedeaCTF

MedeaCTF is a heavily modified, stripped down, 16-bit version of the private Medea [instruction set architecture](https://en.wikipedia.org/wiki/Instruction_set_architecture) designed for reverse-engineering challenges.

Specification version: 1.11.0

## Architecture

- Little-endian
- 2's complement signed arithmetic
- 3 general purpose single-word registers `RX`, `RY` and `RZ`
- Single-word target register `RTRGT`
  - *`RTRGT` is write-only. Attempts to read from it, except through instructions that modify a register in-place, will read `0x0000`.*
- Single-word flags register `RSTAT`
  - *`RSTAT` is read-only. Attempts to write to it, except through `[CALL]`, `[RTRN]`, `[RTRV]` and instructions that modify flags, have no effect.*
- Single-word call register `RCALL`
  - *`RCALL` is read-only. Attempts to write to it, except through `[CALL]`, have no effect.*
- Single-word stack pointer register `RSK`
  - *`RSK` cannot be written to or read from, except through `[PUSH]` and `[POP]`.*
- Single-word stack return register `RSR`
  - *`RSR` cannot be written to or read from, except through `[CALL]`.*
- 65536-word stack space `SSK`
  - *`SSK` cannot be written to or read from, except through `[PUSH]` and `[POP]`.*
- 65536-word input memory space `SIN`
  - *`SIN` is read-only. Attempts to write to it have no effect.*
- 65536-word general purpose memory space `SMAIN`
- 65536-word code memory space `SCODE`
  - *`SCODE` is hidden from the program and cannot be written to or read from.*
- Single-word instruction pointer

`SST`, `SIN` and `SMAIN` and all registers except `RSTAT` and `RSK` are initially filled with `0x0000`. `RSTAT` initially has `FSE` set. `RSK` is initially `0xFFFF`, as the stack grows downwards.

The instruction pointer is an index into `SCODE` and indicates the next instruction to be executed. Its value is initially `0x0001` and is incremented by the length of each instruction after its execution, unless the instruction modified it itself (i.e., a `[CALL]` or any jump).

Writing to `0x0000` in any space has no effect, and reading from `0x0000` in any readable space reads `0x0000`.

## Flags

`RSTAT` contains flags dependent on the previous operation:

|**Index:**|`0xF` - `0x8`|`0x7`|`0x6`|`0x5`|`0x4`|`0x3`|`0x2`|`0x1`|`0x0`|
|---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|**Name:**|*reserved*|`FSF`|`FSE`|`FINF`|`FCRRY`|`FGT`|`FLT`|`FEQUL`|`FZERO`|

- `FZERO` is set when an instruction produces a value of zero, or if one of the values compared by `[CMP]` was zero.
- `FEQUL`, `FLT` and `FGT` are set by `[CMP]` if a value was equal to another, if a value was less than another, and if a value was greater than another, respectively.
- `FCRRY` is set when an addition or subtraction operation produces a carry, and cleared when one does not.
- `FINF` is set when the result of an operation is infinite. `0x0000` is returned as the result of such operations.
- `FSE` is set when the stack is empty. `FSF` is set when the stack is full.

If an instruction sets a flag if some condition occurs, it will clear it if the condition does not occur unless explicitly stated.

## Stack

Pushing a value onto the stack involves decrementing `RSK` and writing the value to the byte in `SSK` at `RSK`. Conversely, popping a value off the stack reads the value in `SSK` at `RSK`, and increments `RSK`. Pushing while the stack is full (i.e., `RSK` = `0x0000`) has no effect, and popping while the stack is empty (i.e., `RSK` = `0xFFFF`) has no effect on `RSK` and reads `0x0000`.

## Calling

Every function has 3 16-bit parameters, which may be ignored. To call a function, the address of the function in `SCODE` is moved into `RTRGT`, and the `[CALL]` instruction is executed with the parameters to the function as arguments (by convention, ignored parameters are passed as `0x0000`). `[CALL]` is implemented as follows:

1. Push `RCALL`, `RSTAT`, `RX`, `RY` and `RZ` onto the stack, in order.
2. Push the length of this `[CALL]` instruction onto the stack.
3. Move `RSK` into `RSR`.
4. Move the function's parameters into `RX`, `RY` and `RZ` respectively.
5. Move the address this `[CALL]` instruction was executed from into `RCALL`.
6. Set the instruction pointer to the value of `RTRGT`.
7. Clear `RTRGT` and `RSTAT`.

Executing `[CALL]` with `RTRGT` set to `0x0000` has no effect.

Every function returns one single-word value, which may be ignored. To return from a function, either the return value is moved into `RTRGT` and `[RTRN]` is executed, or `[RTRV]` is executed, which is an alias for moving `0x0000` into `RTRGT` and calling `[RTRN]`. `[RTRN]` is implemented as follows:

1. Move `RSR` into `RSK`.
2. Pop `RX` off the stack.
3. Set the instruction pointer to the value of `RCALL` plus the value of `RX`.
4. Pop `RCALL`, `RSTAT`, `RX`, `RY` and `RZ` off the stack, in reverse order.
5. Push `RTRGT` onto the stack.
6. Clear `RTRGT`.

## Instruction format

Instructions may have up to 3 arguments, passed in either a register or a memory location.

#### Definition word

|**Name:**|`AFLG0`|`AFLG1`|`AFLG2`|`SIGN`|`OPCODE`|
|---:|:---:|:---:|:---:|:---:|:---:|
|**Size:**|*2 bits*|*2 bits*|*2 bits*|*1 bit*|*9 bits*|

If the instruction requires arguments, `AFLG0` through `2` determine with their first bit whether the argument is in a memory location (`1`) (as opposed to a register (`0`)), and if so, with their second bit whether the argument is in `SIN` (`0`)  or `SMAIN` (`1`). If a particular argument *n* is passed in a register, the second bit of `AFLGn` is ignored, and if the instruction requires fewer than 3 arguments, any later unused `AFLG`s are ignored. Immediate arguments, if the instruction requires any, must have the first `AFLG` bit set. `SIGN` determines whether the instruction, if it performs some form of arithmetic, operates on signed (`1`) or unsigned (`0`) values.

#### Argument words

Arguments are passed in words following the definition word. If one or more of the arguments are registers, they are packed into the first word, in order:

|**Name:**|register index 0|register index 1|register index 2|*reserved*|
|---:|:---:|:---:|:---:|:---:|
|**Size:**|*4 bits*|*4 bits*|*4 bits*|*4 bits*|

If there are fewer than 3 register arguments, register indices are shifted right such that the least significant 4 bits of the register index word before the reserved slot are the last register argument.

Register indices map to registers as follows:

|0x1|0x2|0x3|0x4|0x5|0x6|
|---|---|---|---|---|---|
|`RX`|`RY`|`RZ`|`RTRGT`|`RSTAT`|`RCALL`|

Any other register index maps to a null register.

Non-register arguments follow any register arguments in words as they are, in order:

|**Name:**|value|
|---:|:---:|
|**Size:**|*16 bits*|

## Instructions

|Opcode|Mnemonic|Argument count|Uses `SIGN`?|Description|Notes|
|:---:|:---:|:---:|:---:|---|---|
|`0x000`|`[HALT]`|0|No|Causes execution to halt.||
|`0x001`|`[NOOP]`|0|No|Consumes one cycle and does nothing.||
|`0x002`|`[INC]`|1|Yes|Increments the value of the argument.|Sets `FZERO` if the result is 0.|
|`0x003`|`[DEC]`|1|Yes|Decrements the value of the argument.|Sets `FZERO` if the result is 0.|
|`0x004`|`[ADD]`|2|Yes|Adds the second argument to the first.|Sets `FZERO` if the result is 0. Sets `FCRRY` if the operation produces a carry.|
|`0x005`|`[SUB]`|2|Yes|Subtracts the second argument from the first.|Sets `FZERO` if the result is 0. Sets `FCRRY` if the operation produces a carry.|
|`0x006`|`[MUL]`|2|Yes|Multiplies the first argument in place by the second.|Sets `FZERO` if the result is 0.|
|`0x007`|`[DIV]`|2|Yes|Divides the first argument by the second.|Sets `FZERO` if the result is 0. Sets `FINF` if the result is infinite.|
|`0x008`|`[ADDC]`|2|Yes|Adds the two arguments and `FCRRY` together.|Sets `FZERO` if the result is 0. Sets `FCRRY` if the operation produces a carry.|
|`0x009`|`[SUBC]`|2|Yes|Subtracts the second argument, plus `FCRRY`, from the first argument.|Sets `FZERO` if the result is 0. Sets `FCRRY` if the operation produces a carry.|
|`0x00A`|`[READ]`|1|No|Reads an [ISO-8859-1](https://en.wikipedia.org/wiki/ISO/IEC_8859-1) character from user input (stdin) and places it in the argument location.|Sets `FZERO` if the character is null.|
|`0x00B`|`[WRIT]`|1|No|Writes the low 8 bits of the argument to output (stdout) as an ISO-8859-1 character.||
|`0x00C`|`[CPY]`|2|No|Moves the first argument into the second argument location.|Sets `FZERO` if the value is 0.|
|`0x00D`|`[MCPY]`|3|No|Moves the first argument bitwise AND the third argument into the second argument location.|Sets `FZERO` if the result is 0.|
|`0x00E`|`[ICPY]`|2|No|Moves the first argument as an immediate value into the second argument location.|Sets `FZERO` if the value is 0. Ignores `AFLG0`.|
|`0x00F`|`[CMP]`|2|No|Compares the first argument to the second.|Sets `FZERO` if either argument is 0. Sets `FEQUL` if the two arguments are equal. Sets `FLT` or `FGT` if the first argument is less or greater respectively than the second.|
|`0x010`|`[AND]`|2|No|Bitwise ANDs the first argument in place with the second.|Sets `FZERO` if the result is 0.|
|`0x011`|`[OR]`|2|No|Bitwise ORs the first argument in place with the second.|Sets `FZERO` if the result is 0.|
|`0x012`|`[CMPL]`|1|No|Replaces the argument with its bitwise complement.|Sets `FZERO` if the result is 0.|
|`0x013`|`[LSHF]`|2|No|Shifts the first argument left by the second argument's number of bits.|Sets `FZERO` if the result is 0. Fills bits on the right with 0.|
|`0x014`|`[RSHF]`|2|Yes|Shifts the first argument right by the second argument's number of bits.|Sets `FZERO` if the result is 0. Fills bits on the left with 0 if `SIGN` is `0`, and the leftmost bit of the original value if `SIGN` is `1`.|
|`0x015`|`[PUSH]`|1|No|Pushes the first argument onto the stack.|Sets `FZERO` if the value is 0. Sets `FSF` if the stack is full.|
|`0x016`|`[POP]`|1|No|Pops a value off the stack into the first argument location.|Sets `FZERO` if the value is 0. Sets `FSE` if the stack is empty.|
|`0x017`|`[CFLG]`|0|No|Clears `RSTAT`.||
|`0x018`|`[CALL]`|3|No|Calls a function at `RTRGT`.||
|`0x019`|`[RTRN]`|0|No|Returns a value from a function.|Sets `FZERO` if the return value is 0.|
|`0x01A`|`[RTRV]`|0|No|Returns 0 from a function.|Sets `FZERO`.|
|`0x01B`|`[RTL]`|2|No|Rotates the first argument left by the second argument's number of bits.|Sets `FZERO` if the result is 0.|
|`0x01C`|`[RTR]`|2|No|Rotates the first argument right by the second argument's number of bits.|Sets `FZERO` if the result is 0.|
|`0x01D`|`[CIP]`|1|No|Copies the instruction pointer into the argument's location.||
|`0x01E`|`[BSWP]`|1|No|Swaps the argument's high byte and low byte.|Sets `FZERO` if the result is 0.|
|`0x01F`|`[JUMP]`|0|No|Jumps to `RTRGT`.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x020`|`[JZRO]`|0|No|Jumps to `RTRGT` if `FZERO` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x021`|`[JEQU]`|0|No|Jumps to `RTRGT` if `FEQUL` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x022`|`[JLT]`|0|No|Jumps to `RTRGT` if `FLT` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x023`|`[JGT]`|0|No|Jumps to `RTRGT` if `FGT` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x024`|`[JCRY]`|0|No|Jumps to `RTRGT` if `FCRRY` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x025`|`[JINF]`|0|No|Jumps to `RTRGT` if `FINF` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x026`|`[JSE]`|0|No|Jumps to `RTRGT` if `FSE` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x027`|`[JSF]`|0|No|Jumps to `RTRGT` if `FSF` is set.|Jumping to `0x0000` is equivalent to `[HALT]`.|
|`0x030`|`[CZRO]`|0|No|Clears `FZERO`.||
|`0x034`|`[CCRY]`|0|No|Clears `FCRRY`.||
|`0x040`|`[XOR]`|2|No|Bitwise XORs the first argument in place with the second.|Sets `FZERO` if the result is 0.|
|`0x041`|`[SWAP]`|2|No|Swaps the first and second arguments.|Sets `FZERO` if either argument is 0.|
|`0x042`|`[RCPT]`|2|Yes|Copies the first argument to the second argument offset forwards (if `SIGN` is set) or backwards (if not) by `RTRGT`.|The second argument must be a memory space address. Sets `FZERO` if the value is 0. Overflow is resolved by wrapping.|
|`0x043`|`[RCPF]`|2|Yes|Copies the second argument offset forwards (if `SIGN` is set) or backwards (if not) by `RTRGT` to the first argument.|The first argument must be a memory space address. Sets `FZERO` if the value is 0. Overflow is resolved by wrapping.|

Any unrecognized opcode or violation of a requirement for arguments specified in an opcode's notes causes execution to halt.

## Images

A MedeaCTF image begins with the sequence of bytes [`0x6D`, `0x43`, `0x54`, `0x46`] ("mCTF"), followed by a series of memory space chunks, as follows:

|**Name:**|space index|length *n*|space data|
|---:|:---:|:---:|:---:|
|**Size:**|*1 byte*|*2 bytes*|*n words*|

The length field is an unsigned little-endian integer. A length field greater than 65534 (i.e., 65535) is an error, as address `0x0000` is reserved. Space indices map to memory spaces as follows:

|0x01|0x02|0x03|
|---|---|---|
|`SIN`|`SCODE`|`SMAIN`|

Any other space index value is an error, as is more than one chunk with the same space index.

Data in memory space chunks is used as the initial data for the specified memory space offset by 1 word, and right-padded with `0x0000` if shorter than 65534 words.

Images may also begin with the sequence [`0x6D`, `0x43`, `0x54`, `0x5A`] ("mCTZ"), which indicates that the remainder of the image is compressed with [Zstandard](https://github.com/facebook/zstd).

## Glossary

|Term|Definition|
|---|---|
|*word*|16-bit value|
|*memory space*|contiguous array of words|
|*memory location*|single-word slot in a memory space|
|*register*|distinct single-word slot|
|*null register*|register that does nothing when written to and returns `0x0000` when read from|
|*clear*|set a bit to `0` or a word to `0x0000`|
|*jump*|set the instruction pointer to an address in `SCODE`|
|*wrap*|reduce a value requiring more than 16 bits to represent by discarding bits higher than the 16th, effectively producing a result modulo 65535|
