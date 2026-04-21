---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 4b - Intended Solution
author: Henry Gabryjelski
date: 2026-04-17 03:01:16 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 4b - Intended Solution

Cross-platform, python-based programmable serial console is now available at [https://github.com/henrygab/sos_helper](https://github.com/henrygab/sos_helper).

Hints and review are provided in the [prior post](https://henrygab.github.io/sword%20of%20secrets/2026/04/01/Sword-Of-Secrets-Stage4a.html).

<hr/>

## Walkthrough

This walkthrough will guide you through steps that,
if followed, will lead to what I believe to be the
intended solution.  Its goal is to allow you to
follow along for a section, then explore on your
own, and repeat.

If you just want the solution
summary, it's at the end of this post.

<details markdown="1"><summary>Wrapper of spoilers</summary><P/>


### Expectations for Stage 4

<details markdown="1"><summary>Expectations for Stage 4</summary><P/>

You've already solved stages 1-3, so you:
* can read, erase, and write to the flash chip
* are familiar with the source code
* are familiar with block-based AES-128
  cryptography used in both `ECB` and `CBC` mode
* have some type of programmatic access to
  serial port of the CTF board
* can cause an `ECB` ciphertext block
  to be decrypted and printed from stage 2
* can decrypt `CBC` ciphertext block using
  the padding oracle from stage 3
* can check if you've progressed via `SOLVE` command

For this stage, ...

* Learn about Brainf*** interpreters
* Compile and review output of `objdump`
* Detect and fix an intentional stack corrupting error
* Find a way to modify the function loaded into global
  variable `code[]`
* Use the Padding Oracle -> Decryption Oracle, and
  take it one step further into an Encryption Oracle
* Apply all the above to get the final flag

</details>

### Review source code for Stage 4

<details markdown="1"><summary>Review source code for Stage 4</summary><P/>

`challenges()` will walk through and validate
stages 1-3 (`palisade()`, `parapet()`, and
`postern()`), as noted in prior steps.

After that, `challenges()` will:
* call `void digForTreasure(void)`
* call `int treasuryVisit(void)`

Success is indicated when `treasuryVisit()`
returns a non-negative value.

#### Helpful notes

<details markdown="1"><summary>Helpful notes</summary><P/>

There are two similarly-named `#define`'d symbols:
* `PLUNDER_ADDR_DATA` ... which contains script interpreted by
  a Brainf*** interpreter.  This is `0x50000` ... and you are
  pointed to this location when solving Stage 3.
* `PLUNDER_ADDR` ... the `swordOfSecrets()` instructions
  are encrypted using AES-CBC, written to this address, and
  then another function loads this encrypted code, decrypts
  it, and stores the result in `armory.c`'s global
  variable `data[]`.  The code in `data[]` is executed as
  part of `SOLVE` when `treasuryVisit()` is called.

</details>

#### `digForTreasure()` / `treasure()`

<details markdown="1"><summary>`digForTreasure()` / `treasure()`</summary><P/>

`digForTreasure()` does the following:
1. reads 4 bytes at `PLUNDER_ADDR_DATA` (???) as `len`.
2. reads `MIN(len,512)` bytes at `PLUNDER_ADDR_DATA+4`
   into a 512-byte stack buffer `data`.
3. calls `treasure(data,len)`

Unlike prior challenges, the data at `0x50000`  is
***NOT*** encrypted.   Instead, it is interpreted as
a script for what is commonly referred to as a
Brainf**** interpreter.

Here, the BF interpreter is beautifully and artistically
rendered in the file `secret.c` as function `treasure()`.
The function can be difficult to mentally parse in that
artistic form, so I recommend reformatting it using
automated code formatting tools before looking at it too
much.

</details>

#### Analysis of `treasure()`

<details markdown="1"><summary>Analysis of `treasure()`</summary><P/>

Here's my summary of the function.

* `uint8_t tape[256]` is a global, and is ostensibly
  used to storing data for use by a BF script.
* `int ip` is the instruction pointer
* `int dp` is the data pointer (an offset within `tape`)
* `int sp` is a stack pointer (tracks start of loops)
* `int stack[16]` stores the `ip` value from loop starts

Only eight characters are interpreted: `><+-.,[]`:
* `>` increments variable `dp` (offset into `tape[]`)
* `<` decrements variable `dp` (offset into `tape[]`)
* `+` increment the byte at `tape[dp]`
* `-` decrement the byte at `tape[dp]`
* `.` output the hex value of the byte at `tape[dp]`
* `,` not implemented (other BF interpreters can take input)
* `[` begin a loop
* `]` end a loop

NOTE: See much later section, which will describe
      how the BF interpreter is abused for our gain.

</details>

#### `treasuryVisit()`

<details markdown="1"><summary>`treasuryVisit()`</summary><P/>

This simply casts a global variable, `code`, into a function
pointer, and then calls that function pointer with two
parameters:
* `println_export`, a function pointer that essentially calls
  printf() and then outputs a `\n`.
* a pointer to the `SUBMIT_PASSWORD` data ... the ***ONLY***
  location this variable is used is in this function!

`SUBMIT_PASSWORD` is defined as a 55-character/55-byte string:
```c
// Obviously, this is a placeholder for the real submit password....
#define SUBMIT_PASSWORD "FLAG{XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX}"
//                       ....-....1....-....2....-....3....-....4....-....
```

</details>

#### Wait... so what's in global variable `code`?

<details markdown="1"><summary>Wait... so what's in global variable `code`?</summary><P/>

The global variable `code`'s contents are loaded
by function `plunderLoad()`.

This is done only ***ONCE*** per boot of the CTF board:

* `main()` calls `setupQuest()`
* `setupQuest()` calls `prizeSetup()`
* `prizeSetup()` copies `0x32` (`50`)bytes of the bytecode for the function
  `theSwordOfSecrets()` into a local buffer, AES-CBC encrypts it,
  and writes it (with 4-byte length prefix) to `PLUNDER_ADDR`.

Because this only occurs once during boot, it raises
a challenge ... how to modify the function if you can't
provide input (and thus cannot modify the state) until
after the above sequence happens?

</details>

#### Generating an assembly listing

<details markdown="1"><summary>Generating an assembly listing</summary><P/>

The steps are already in the makefile, but
are not executed by default.

```makefile
$(TARGET).bin : $(TARGET).elf
	$(PREFIX)-size $^
	$(PREFIX)-objdump -S $^ > $(TARGET).lst
	$(PREFIX)-objdump -t $^ > $(TARGET).map
	$(PREFIX)-objcopy -O binary $< $(TARGET).bin
	$(PREFIX)-objcopy -O ihex $< $(TARGET).hex
```

To create the `firmware.lst` file, simply type 
`make firmware.bin`.  Load the `firmware.lst`
file into your favorite text editor, and search
for `theSwordOfSecrets` to jump directly to the
definition of this function.

</details>

#### Assembly in `code`

<details markdown="1"><summary>Assembly in `code`</summary><P/>

The data stored in the flash is stored AES-ECB encrypted.
Review the generated file `firmware.lst`, and search for
`theSwordOfSecrets`, and you'll see the original code
(although secret values will obviously not be present).

Below, I've copied the results into a table format,
with some additional derived data such as the
relative offset within that function
(useful to split it into AES block size chunks).

| Offset | Address  | Bytes         | Opcode     | Assembly           | Notes                                    |
|--------|----------|---------------|------------|--------------------|------------------------------------------|
| `0x00` | `0x16b2` | `aa 87      ` | `87aa    ` | `mv   a5, a0`      | fn* arg0 -> a5                           |
| `0x02` | `0x16b4` | `2e 85      ` | `852e    ` | `mv   a0, a1`      | arg1 -> arg0                             |
| `0x04` | `0x16b6` | `06 c0      ` | `c006    ` | `sw   ra, 0(sp)`   | PROLOG: store $ra to stack               |
| `0x06` | `0x16b8` | `71 11      ` | `1171    ` | `addi sp, sp, -4`  | PROLOG: use 4 bytes of stack             |
| `0x08` | `0x16ba` | `37 07 00 08` | `08000737` | `lui  a4, 0x8000`  | Load 0x8000'0000 to a4                   |
| `0x0C` | `0x16be` | `18 43      ` | `4318    ` | `lw   a4, 0(a4)`   | load *(0x8000'0000) to a4                |
| `0x0E` | `0x16c0` | `11 e7      ` | `e711    ` | `bnez a4, ip+0x0C` | if non-zero, jump to fail (offset 0x1A)  |
| `0x10` | `0x16c2` | `82 97      ` | `9782    ` | `jalr a5`          | else call fn* arg0                       |
| `0x12` | `0x16c4` | `01 45      ` | `4501    ` | `li   a0, 0`       | set success result                       |
| `0x14` | `0x16c6` | `82 40      ` | `4082    ` | `lw   ra, 0(sp)`   | EPILOG: corrupts the stack!!             |
| `0x16` | `0x16c8` | `11 01      ` | `0111    ` | `addi sp, sp,  4`  | EPILOG: release 4 bytes of stack         |
| `0x18` | `0x16ca` | `82 80      ` | `8082    ` | `ret`              | EPILOG: return                           |
| `0x1A` | `0x16cc` | `7d 55      ` | `557d    ` | `li   a0, -1`      | FAIL: set failure result                 |
| `0x1C` | `0x16ce` | `e5 bf      ` | `bfe5    ` | `j    ip-0x08`     | jump to epilog (offset 0x14)             |
| `0x1E` | `0x16d0` | `aa 87      ` | `87aa    ` | `mv   a5, a0`      | (start of next function...)              |

##### Important -- intentional stack corruption

<details markdown="1"><summary>Important -- intentional stack corruption</summary><P/>

The function `theSwordOfSecrets()` has `__attribute__((naked))`
applied.  I had never seen this prior to this CTF, but looking
into the documentation, this simply prevents automatic generation
of the function's prolog and epilog.  Put another way, the
attribute indicates that the function will handle stack space
reservation and restoration itself.

Here, the manually generated prolog and epilog corrupt the stack.

Prolog:
```
    __asm__("sw ra, 0(sp)");
    __asm__("addi sp, sp, -4");
```

Epilog:
```
    __asm__("lw ra, 0(sp)");
    __asm__("addi sp, sp, 4");
    __asm__("ret");
```

</details>

##### Example showing the stack corruption

<details markdown="1"><summary>Example showing the stack corruption</summary><P/>

Here I'll use `_____` to indicate the value being modified
by the current instruction.  The stack pointer will start
out as `0x800`, and I'll use `qqqqq` for the value stored
in the uninitialized stack location `0x7FC`.

| `$sp`   | `$ra`   | `0x7FC` | `0x800` | Next Instruction  | Comments 
|---------|---------|---------|---------|-------------------|------------
| `0x800` | `0xAE8` | `qqqqq` | `_____` | `sw ra, 0(sp)`    | PROLOG: store `ra` 
| `_____` | `0xAE8` | `qqqqq` | `0xAE8` | `addi sp, sp, -4` | PROLOG: reserve stack
| `0x7FC` | `0xAE8` | `qqqqq` | `0xAE8` | ...               | ...
| `0x7FC` | `_____` | `qqqqq` | `0xAE8` | `lw ra, 0(sp)`    | EPILOG: load `ra` from `*sp` aka `0x7FC`
| `_____` | `qqqqq` | `qqqqq` | `0xAE8` | `addi sp, sp,  4` | EPILOG: release stack
| `0x800` | `qqqqq` | `qqqqq` | `0xAE8` | `ret`             | jumps to ra == `qqqqq`

The two instructions in the EPILOG should be swapped to
fix this bug.  I believe this was an intentional addition
to make the challenge slightly more difficult.

</details>

</details>

</details>


### Creating the desired data for `code[]`

#### Desired code

<details markdown="1"><summary>Desired code</summary><P/>

As you know, the first AES block of instructions cannot be
predictably changed using the padding oracle, so we will
rely on the BF interpreter to modify the branch instruction
to skip past the second AES block.
The second AES block will be manipulated using the padding
oracle, to create attacker-controlled plaintext in the
third AES block of instructions.

Here's what we ***WANT*** the decrypted data in `code[]` to be:

| Offset | Address  | Bytes         | Opcode     | Assembly           | Notes                                    |
|--------|----------|---------------|------------|--------------------|------------------------------------------|
| `0x00` | `0x16b2` | `aa 87      ` | `87aa    ` | `mv   a5, a0`      | fn* arg0 -> a5                           |
| `0x02` | `0x16b4` | `2e 85      ` | `852e    ` | `mv   a0, a1`      | arg1 -> arg0                             |
| `0x04` | `0x16b6` | `06 c0      ` | `c006    ` | `sw   ra, 0(sp)`   | PROLOG: store $ra to stack               |
| `0x06` | `0x16b8` | `71 11      ` | `1171    ` | `addi sp, sp, -4`  | PROLOG: use 4 bytes of stack             |
| `0x08` | `0x16ba` | `37 07 00 08` | `08000737` | `lui  a4, 0x8000`  | Load 0x8000'0000 to a4                   |
| `0x0C` | `0x16be` | `18 43      ` | `4318    ` | `lw   a4, 0(a4)`   | load *(0x8000'0000) to a4                |
| `0x0E` | `0x16c0` | `11 e7      ` | `e711    ` | `bnez a4, ip+0x0C` | ... to be changed by BF script ...       |
| `0x10` | `0x16c2` | `__ __ __ __` | `________` | unpredictable      | ... will never execute ...               |
| `0x14` | `0x16c6` | `__ __ __ __` | `________` | unpredictable      | ... will never execute ...               |
| `0x18` | `0x16ca` | `__ __ __ __` | `________` | unpredictable      | ... will never execute ...               |
| `0x1C` | `0x16ce` | `__ __ __ __` | `________` | unpredictable      | ... will never execute ...               |
| `0x20` | `0x16d2` | `82 97      ` | `9782    ` | `jalr a5`          | call fn* arg0                            |
| `0x22` | `0x16d4` | `01 45      ` | `4501    ` | `li   a0, 0`       | set success result                       |
| `0x24` | `0x16d6` | `11 01      ` | `0111    ` | `addi sp, sp,  4`  | EPILOG: corrected sequence               |
| `0x26` | `0x16d8` | `82 40      ` | `4082    ` | `lw   ra, 0(sp)`   | EPILOG: corrected sequence               |
| `0x28` | `0x16da` | `82 80      ` | `8082    ` | `ret`              | EPILOG: return                           |
| `0x2A` | `0x16dc` | `06 06      ` | `06060606` | PKCS7 padding      | PKCS7 padding (6 bytes)                  |
| `0x2C` | `0x16de` | `06 06 06 06` | `0606    ` | PKCS7 padding      | PKCS7 padding (6 bytes)                  |

</details>

#### Original Encrypted `code[]`

<details markdown="1"><summary>Original Encrypted `code[]`</summary><P/>

This is what's stored at `0x40000`, the temporary
location where the encrypted `theSwordOfSecrets()`
function is written to.  I do not know why the
function decided to encrypt 50 (0x32) bytes of
data, when the real function is only 0x1E bytes.

Luckily, `plunderLoad()` will use whatever length
is encoded into the first four bytes, allowing us
to adjust this (up to 0x80 bytes) as needed.

Here's the encrypted data written at boot to `0x40000`:

```
DATA 40 00 00 00
DATA 8e 3c 5a f5 56 82 d4 0c 5a 26 db 1d 71 f3 cf 11
DATA 5c 71 14 fa c8 30 db cd c9 88 58 42 ce 68 9c c6
DATA b5 78 94 3d 59 1f a9 d4 e2 bc 7b 33 45 c5 99 8e
DATA c8 4e b4 4e 38 a2 aa 12 a8 78 fb 43 8d 7f 8a cd
```

</details>

#### Converted to data to send to the Sword...

<details markdown="1"><summary>Converted to data to send to the Sword...</summary><P/>

We're going to be creating the third block from
scratch, by corrupting the second block and using
the padding oracle.   Therefore, the initial values
of the second and third AES block can be any value.
I chose to initially set the second block to all
zero, and vanity-fill the third block.

```
DATA 30 00 00 00
DATA 8e 3c 5a f5 56 82 d4 0c 5a 26 db 1d 71 f3 cf 11
DATA 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
DATA 68 65 6e 72 79 67 61 62 68 65 6e 72 79 67 61 62
```

In order to use the padding oracle, we will write
this data to `0x30000`.  The process will be very
similar to that done in Stage 3, where we use the
padding oracle to (byte by byte) ensure the final
AES block decrypts to sixteen bytes that are all
`0x10` (a full PKCS7 padding block):

</details>

#### Creating the full 0x10 byte PKCS7 padding

<details markdown="1"><summary>Creating the full 0x10 byte PKCS7 padding</summary><P/>

As in stage 3, here I will track only the contents
of the second AES block, as we brute-force from having
no known plaintext values, step-by-step to a full
0x10 bytes of PKCS7 padding.

```
START       00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 __   # Brute force last padding byte
GOLD(0x01)  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 b2
            00 00 00 00 00 00 00 00 00 00 00 00 00 00 __ b1   # XOR 0x03 (0x01 ^ 0x02)
GOLD(0x02)  00 00 00 00 00 00 00 00 00 00 00 00 00 00 21 b1
            00 00 00 00 00 00 00 00 00 00 00 00 00 __ 20 b2   # XOR 0x01 (0x02 ^ 0x03)
GOLD(0x03)  00 00 00 00 00 00 00 00 00 00 00 00 00 66 20 b2
            00 00 00 00 00 00 00 00 00 00 00 00 __ 61 27 b7   # XOR 0x07 (0x03 ^ 0x04)
GOLD(0x04)  00 00 00 00 00 00 00 00 00 00 00 00 d9 61 27 b7
            00 00 00 00 00 00 00 00 00 00 00 __ d8 60 26 b6   # XOR 0x01 (0x04 ^ 0x05)
GOLD(0x05)  00 00 00 00 00 00 00 00 00 00 00 48 d8 60 26 b6
            00 00 00 00 00 00 00 00 00 00 __ 4b db 63 25 b5   # XOR 0x03 (0x05 ^ 0x06)
GOLD(0x06)  00 00 00 00 00 00 00 00 00 00 42 4b db 63 25 b5
            00 00 00 00 00 00 00 00 00 __ 43 4a da 62 24 b4   # XOR 0x01 (0x06 ^ 0x07)
GOLD(0x07)  00 00 00 00 00 00 00 00 00 c4 43 4a da 62 24 b4
            00 00 00 00 00 00 00 00 __ cb 4c 45 d5 6d 2b bb   # XOR 0x0F (0x07 ^ 0x08)
GOLD(0x08)  00 00 00 00 00 00 00 00 5c cb 4c 45 d5 6d 2b bb
            00 00 00 00 00 00 00 __ 5d ca 4d 44 d4 6c 2a ba   # XOR 0x01 (0x08 ^ 0x09)
GOLD(0x09)  00 00 00 00 00 00 00 4e 5d ca 4d 44 d4 6c 2a ba
            00 00 00 00 00 00 __ 4d 5e c9 4e 47 d7 6f 29 b9   # XOR 0x03 (0x09 ^ 0x0a)
GOLD(0x0a)  00 00 00 00 00 00 8e 4d 5e c9 4e 47 d7 6f 29 b9
            00 00 00 00 00 __ 8f 4c 5f c8 4f 46 d6 6e 28 b8   # XOR 0x01 (0x0a ^ 0x0b)
GOLD(0x0b)  00 00 00 00 00 64 8f 4c 5f c8 4f 46 d6 6e 28 b8
            00 00 00 00 __ 63 88 4b 58 cf 48 41 d1 69 2f bf   # XOR 0x07 (0x0b ^ 0x0c)
GOLD(0x0c)  00 00 00 00 6c 63 88 4b 58 cf 48 41 d1 69 2f bf
            00 00 00 __ 6d 62 89 4a 59 ce 49 40 d0 68 2e be   # XOR 0x01 (0x0c ^ 0x0d)
GOLD(0x0d)  00 00 00 84 6d 62 89 4a 59 ce 49 40 d0 68 2e be

            00 00 __ 87 6e 61 8a 49 5a cd 4a 43 d3 6b 2d bd   # XOR 0x03 (0x0d ^ 0x0e)
GOLD(0x0e)  00 00 4c 87 6e 61 8a 49 5a cd 4a 43 d3 6b 2d bd
            00 __ 4d 86 6f 60 8b 48 5b cc 4b 42 d2 6a 2c bc   # XOR 0x01 (0x0e ^ 0x0f)
GOLD(0x0f)  00 5c 4d 86 6f 60 8b 48 5b cc 4b 42 d2 6a 2c bc
            __ 43 52 99 70 7f 94 57 44 d3 54 5d cd 75 33 a3   # XOR 0x1F (0x0f ^ 0x10)
GOLD(0x10)  e9 43 52 99 70 7f 94 57 44 d3 54 5d cd 75 33 a3
```

The result is the following encrypted data blob:

```
ENCRYPTED0:  8e 3c 5a f5 56 82 d4 0c 5a 26 db 1d 71 f3 cf 11
ENCRYPTED1:  e9 43 52 99 70 7f 94 57 44 d3 54 5d cd 75 33 a3
ENCRYPTED2:  68 65 6e 72 79 67 61 62 68 65 6e 72 79 67 61 62
```

Which decrypts into:

```
DECRYPTED0:  aa 87 2e 85 06 c0 71 11 37 07 00 08 18 43 11 e7
DECRYPTED1:  __ __ __ __ __ __ __ __ __ __ __ __ __ __ __ __
DECRYPTED2:  10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10
```

</details>

#### From decryption oracle to encryption oracle

<details markdown="1"><summary>From decryption oracle to encryption oracle</summary><P/>

Obviously, we now want to change the last sixteen bytes from
the PKCS7 padding of `b'\x10' * 16`, into the instructions
described earlier (plus six bytes of PKCS7 padding).
Specifically, the plaintext we are aiming for is:

```
DECRYPTED2': 82 97 01 45 11 01 82 40 82 80 06 06 06 06 06 06
```

Remember how we could XOR the prior block of AES ciphertext
to change the PKCS7 padding bytes when expanding it by one
byte?   We can do substantially the same thing here, by
`XOR`'ing each byte with both the current value and the
new (desired) value.  The only difference here is that the
new (desired) value will be different for each byte.

```
  ENCRYPTED1:  e9 43 52 99 70 7f 94 57 44 d3 54 5d cd 75 33 a3
^ DECRYPTED2': 82 97 01 45 11 01 82 40 82 80 06 06 06 06 06 06
==============================================================
intermediate:  6b d4 53 dc 61 7e 16 17 c6 53 52 5b cb 73 35 a5
^ DECRYPTED2:  10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10
==============================================================
      FINAL2:  7b c4 43 cc 71 6e 06 07 d6 43 42 4b db 63 25 b5
```

And that gives us the new ciphertext for the second AES block,
which will cause the third AES block to decrypt to our desired
instructions:

```
DATA 03 00 00 00
DATA 8e 3c 5a f5 56 82 d4 0c 5a 26 db 1d 71 f3 cf 11
DATA 7b c4 43 cc 71 6e 06 07 d6 43 42 4b db 63 25 b5
DATA 68 65 6e 72 79 67 61 62 68 65 6e 72 79 67 61 62
```

Which decrypts into:

```
DATA aa 87 2e 85 06 c0 71 11 37 07 00 08 18 43 11 e7 ......q.7....C..
DATA __ __ __ __ __ __ __ __ __ __ __ __ __ __ __ __ ????????????????
DATA 82 97 01 45 11 01 82 40 82 80 06 06 06 06 06 06 ...E...@........
```

Success!

</details>

#### Remember to restore stage 3's solution...

<details markdown="1"><summary>Remember to restore stage 3's solution...</summary><P/>

So you've written the final data to `0x40000`, and
you've written the BF script to `0x50000`.

However, the process of creating the encrypted data
required you to smash your stage 3 data.  This is
your friendly reminder to restore your stage 3
solution at `0x30000`.

</details>

### What BF script must change in first AES block

<details markdown="1"><summary>What BF script must change in first AES block</summary><P/>

The script nominally modifies data in a global variable,
`uint8_t tape[256]`.  `int dp` is used as the index when
accessing the data (a signed integer), and the BF interpreter
has ZERO logic to prevent the `dp` variable from being
incremented past 256 (or decremented below zero).  The only
limitation is the maximum script size of 512 bytes.

Therefore, if your BF script increments `dp` more than
`255` times, the `+`, `-`, and `.` commands will index
`tape[dp]`, except that will now point past the allocated
size of `tape` ... likely other global variables.

After you compiled the source, you can review the `firmware.lst`
file ... search for `<tape>` and `<code>`:
* `tape` @ `0x200000f4`
* `code` @ `0x20000204`

Thus, `code` exists `0x110` bytes after the first byte
of the `tape` buffer.

Luckily, it is fairly easy to use a BF script to change the final
instruction of the first AES block:

Bytes   | OpCode | Instruction        | Notes
--------|--------|--------------------|------------
`11 e7` | `e711` | `bnez a4, ip+0x0C` | Original Instruction
`09 eb` | `eb09` | `bnez a4, ip+0x12` | Desired Instruction

Modification via BF is required because the first AES block
cannot be modified to attacker-controlled data using the padding
oracle, and to generate attacker-controlled data later, you
first need a block that can be decrypt to unpredictable values.
By causing this instruction to jump to `+0x12`, the function
will never look at the second AES block ... so it won't matter
what those bytes decrypt to.   This allows the padding oracle
to be used to generate attacker-controlled data in the third
(and later, if desired) AES blocks.

```python
bf_script = bytes(
  '>' * 0x110 + # skip from data[0] to code[0]
  '>' *   0xE + # skip to the first byte of the target instruction (0x11)
  '-' *   0x8 + # change that first byte from 0x11 -> 0x09 (-8)
  '>' *   0x1 + # goto the second byte of the instruction (0xe7)
  '+' *   0x4   # change that second byte from 0xe7 -> 0xeb (+4)
) #     299 bytes ... well shy of 512 byte maximum script size
```

And that gives you the BF script to be written to `0x50000`.

> Side note:  This is the first time you'll be writing more than
> 256 bytes of data.  Carefully read the datasheet's description
> of the `PAGE PROGRAM 02h` command.   Yep ... if you just tried
> to send 299 bytes of data, you'd only write 256 bytes.  Worse,
> the first 43 (299 - 256) bytes would be written twice, and just
> leave you with corrupted data.
> Thus, the sequence is:
> * Enable Write, Erase the block
> * Enable Write, Write the first 256 bytes @ `0x50000`
> * Enable Write, write the remaining 43 bytes @ `0x50100`
> Of course, if you're using the sos_helper library ... this
> is handled automagically for you.

</details>

### Write-protecting the flash

<details markdown="1"><summary>Write-protecting the flash</summary><P/>

The data you've manipulated for stage 4 will be
overwritten every boot by `prizeSetup()`.  Thus,
after writing the replacement ciphertext, it's
critical to write-protect at least that sector
before rebooting.

See the [prior post](https://henrygab.github.io/sword%20of%20secrets/2026/04/01/Sword-Of-Secrets-Stage4a.html),
under the "Deep Hint" section, for easiest way to write-protect the flash chip.

</details>


### And ... SOLVE!

<details markdown="1"><summary>And ... SOLVE!</summary><P/>

With the chip write-protected:

* `REBOOT` ... will reboot the MCU, leaving the flash write-protected
  * `main()` will call `setupQuest()`
  * `setupQuest()` will call `prizeSetup()`
  * `prizeSetup()` will try to write the encrypted `theSwordOfSecrets()`
    function to the flash (and not notice that it was write-protected)
  * `main()` will then call `plunderLoad()`
  * `plunderLoad()` will read in your updated ciphertext, decrypt it,
    and store the decrypted result into `code[]`
* `SOLVE`
  * `challenges()` will validate stage 1 `palisade()`
  * `challenges()` will validate stage 2 `parapet()`
  * `challenges()` will validate stage 3 `postern()`
  * `challenges()` will call `digForTreasure()`
    * `digForTreasure()` will load the BF script and call `treasure()`
    * `treasure()` will modify the jump instruction stored in `code[]`
  * `challenges()` will call `treasuryVisit()`
    * ... which calls into `code[]`, executing the function that the
      BF script just fixed the jump instruction of ...
* And ... success!

</details>

</details>


## Solution Summary

<details markdown="1"><summary>Solution Summary</summary><P/>

This solution relies on combining multiple abilities:

1. The ability to create a BF script that will modify one of the
   instructions in the first AES block of `theSwordOfSecrets()`,
   so that the function will entirely skip the second AES block
   (allowing it to be corrupted/unpredictable) and jump directly
   to the third AES block. (new)
2. The ability to use the encryption oracle to create arbitrary
   attacker-controlled plaintext, by having one earlier AES block
   whose plaintext will be corrupted / unpredictable, as learned
   about during Stage 3.  Note that this method does not allow
   changing the first block to be attacker-chosen plaintext.
3. The knowledge of how to find / modify the bytes for one
   instruction to change it into a second instruction. (new)
4. The ability to write-protect the flash, and have that stay
   in effect while rebooting the Sword. (new)
</details>

## FIN

Helper script will be updated with the support functions for
this stage.  However, stage 3 has all the functions needed,
since most of the work is in understanding what's happening,
and the stage 3 padding oracle is re-used here.

Even so, believe it or not, this is ***not*** the
end of the story.

There are some additional solutions to share, which while
not the intended solution, are reasonable alternatives to
the above.

Enjoy!

Note: This post was edited to use vanity data in the third
AES block of ciphertext (instead of all-zero data).
