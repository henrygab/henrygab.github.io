---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 5 - OOPS! Solution
author: Henry Gabryjelski
date: 2026-08-04 01:09:31 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 5 - OOPS! Solution for Stage 4

Turns out I never posted this, back in mid-April.  Better late than never?

Cross-platform, python-based programmable serial console is now available at [https://github.com/henrygab/sos_helper](https://github.com/henrygab/sos_helper).

Hints and review are provided in the [prior post](https://henrygab.github.io/sword%20of%20secrets/2026/04/01/Sword-Of-Secrets-Stage4a.html).

<hr/>

This walkthrough presumes you have already solved
the sword of secrets.  This page should be considered
entirely full of spoilers, and it is strongly 
recommended to read further only after you have
finished your own explorations.

No, really ... one you follow this, there's nothing
left.

<details markdown="1"><summary>Wrapper of spoilers</summary><P/>


## Expectations for Stage 5

<details markdown="1"><summary>Expectations for Stage 5</summary><P/>

You've already solved stages 1-4, so you:
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
* create a Brainf*** interpreter script
* Compile and review output of `objdump`
* Detect and fix an intentional stack corrupting error
* Modify the function loaded into global
  variable `code[]`
* Create chosen cleartext using the Padding Oracle
  as an encryption oracle
* Apply all the above to execute your own code

</details>

## Intended stage 4 solution is time consuming

<details markdown="1"><summary>Time-consuming aspects of intended stage 4 solution</summary><P/>

One of the most time-consuming aspects of the
intended solution to stage 4 is having to
brute-force the encrypted data via the padding
oracle ... a process that takes, on average,
about 2048 cycles (erase, write next attempt,
run solve and interpret the oracle's response).

That's really time consuming, but necessary because 
we don't know the key used to encrypt / decrypt the
data ... thus the need to use the padding oracle.

</details>

## Stage 4 - Alternate solution #1

### BF Script Dumpable Region

<details markdown="1"><summary>BF Script Dumpable Region</summary><P/>

In solving stage 4 (the intended way), the solution involved
using a bug in the BF interpreter to modify parts of the
global variable `code`, by writing outside the array bounds
of the `tape` variable.

As you recall, `tape` is the variable manipulated by the BF
script interpreter:
* The BF data pointer starts at `tape[0]`
* Each byte of the BF script can do one of:
  * increment the BF data pointer
  * decrement the BF data pointer
  * dump the byte at the BF data pointer
* A BF script can be up to 512 bytes in length

This means a BF script could dump all bytes in the range:
`tape - 0x1FF` through `tape + 0x1FF`.

What's in that range?  Let's first look at the
`.map` / `.lst` files...

</details>

### firmware.map / firmware.lst

<details markdown="1"><summary>firmware.map / firmware.lst</summary><P/>

From the depot root, same place you run `make` to compile the
firmware, you can also run `make firmware.bin`.  This generates
some useful files.  While they do not perfectly match the original
firmware, they can give significant insight into likely layout.

From a review of the `firmware.lst` on my system:

* `tape` @ `0x200000f4 .. 0x200001f4` (`0x100` bytes) 

If accurate, this would allow me to dump the following range
of firmware bytes, using only the BF script capabilities:

`[ 0x1FFF'FEF5 .. 0x2000'01f3 ]` (both ends inclusive)

</details>

### What is in range after the `tape` variable?

<details markdown="1"><summary>firmware.map / firmware.lst</summary><P/>

Let's take a look, maybe there will be something interesting.

First, load up the `firmware.map` file, which you generated
by the command `make firmware.bin`.

Sort the lines, then filter to those with "O" in the
appropriate column, and get a sense of what's stored
where (at least when you compile).   Here's what shows
up in my `firmware.map` file for the relevant range:

```
20000000 l     O .bootloader.data   00000010 iv
20000010 l     O .bootloader.data   000000c0 ctx
200000d0 l     O .data              00000004 cs
200000d8 l     O .data              00000008 xor_key
200000e0 l     O .bss               00000001 busy
200000e1 l     O .bss               00000001 flags
200000e4 l     O .bss               00000003 flash_ext_cmds
200000e8 l     O .bss               00000004 initial_jiffies
200000ec l     O .bss               00000008 status
200000f4 l     O .bss               00000100 tape
200001f4 l     O .bss               00000010 aes_key
20000204 l     O .bss               00000080 code
```

You can also load up `firmware.lst`, where sometimes
the variable address is listed after the assembly
instructions where it is used, surrounded by angle
brackets.  Search for any variable name that you're
particularly interested in, such as `<k>` and `<iv>`.

```
0c92:	ddc58593    addi  a1,a1,-548 # 00000ddc <k>
102e: fd658593    addi  a1,a1,-42  # 20000000 <iv>
```

What I got from the above:

* Immediately after `tape`, and before `code` lies
    the variable `aes_key`.
    * This is the key used to encrypt
      all the challenges(!)
    * It's also used to encrypt `theSwordOfSecrets` code
      before its written to 0x40000, and decrypt it
      when it's copied to `code`
    * At least for this compilation, it's within
      the address range dumpable by BF script
* Way back at address `0x20000000` is the variable
    `iv`.
    * `iv` is the initialization vector used in
      combination with the above `aes_key`
    * `iv` is also within the address range that is
      dumpable by BF script
* The variable `k` is at address `0x00000ddc`.
    * That's far outside the addressable range of
      addresses that can be accessed using a BF
      script....


</details>


### Dumping the values

<details markdown="1"><summary>xxx</summary><P/>

I do not believe the placement of the `aes_key`
variable so close to `tape` variable was intentional.
This provides an opportunity too good to pass up.

First, let's presume that all the above addresses
are 100% accurate.  In that case, one could dump
`aes_key` with the BF script:

```python
BF_SCRIPT_TO_DUMP_AES_KEY : bytes = bytes(
  b'>' * 256 + # advance from 0x2000'00F4 to 0x2000'01F4
  b'.>' * 16   # Dump the 16-byte `aes_key` variable
)
```

Similarly, one could dump `iv`, the initialization vector for the
firmware encryption, within range for the BF script.
One can dump the `iv` with the BF script:

```python
BF_SCRIPT_TO_DUMP_AES_KEY : bytes = bytes(
  b'<'  * 0xf4 + # Move data pointer from 0x2000'00F4 to 0x2000'0000
  b'.>' * 16     # Dump the 16-byte `iv` variable
)
```

</details>

### Reality of dumping the values

<details markdown="1"><summary>xxx</summary><P/>

The `firmware.map` included  the following addresses:
```
20000000 l     O .bootloader.data   00000010 iv
200000f4 l     O .bss               00000100 tape
200001f4 l     O .bss               00000010 aes_key
```

However, even the slightest changes in compilation
settings, compiler versions, etc. could change where
a variable ends up.  In fact, it's almost guaranteed
that the variables are in different locations, unless
you know the exact method used to create the firmware.

Therefore, it's probably easier to write a single
python extension for the SoS Helper python library,
noted at the head of this file.  That extension can
dump ***all*** the bytes that are accessible from
BF script, and do so in a fully automated fashion.

Then, with the dump of that data, one can simply
try all 16-byte values, to see if one of them
is the target AES key we're looking for.   Worst
case, we cannot extract it, and have to use the
padding oracle to encrypt each bit of code we run.

Hint:  The `aes_key` variable _is_ accessible.

</details>

### Solving Stage 4 ... the easier way

<details markdown="1"><summary>Solving Stage 4 ... the easier way</summary><P/>

With `aes_key` in hand, there's no longer any
need to keep the first AES block worth of instructions
from the original `theSwordOfSecrets()` function.

In fact, the plaintext can now be as simple as:

| Offset | Address  | Bytes         | Opcode     | Assembly           | Notes                                    |
|--------|----------|---------------|------------|--------------------|------------------------------------------|
| `0x00` | `0x16b2` | `aa 87      ` | `87aa    ` | `mv   a5, a0`      | fn* arg0 -> a5                           |
| `0x02` | `0x16b4` | `2e 85      ` | `852e    ` | `mv   a0, a1`      | arg1 -> arg0                             |
| `0x04` | `0x16b6` | `06 c0      ` | `c006    ` | `sw   ra, 0(sp)`   | PROLOG: store $ra to stack               |
| `0x06` | `0x16b8` | `71 11      ` | `1171    ` | `addi sp, sp, -4`  | PROLOG: use 4 bytes of stack             |
| `0x08` | `0x16ba` | `82 97      ` | `9782    ` | `jalr a5`          | call fn* arg0                            |
| `0x0A` | `0x16bc` | `01 45      ` | `4501    ` | `li   a0, 0`       | set success result                       |
| `0x0C` | `0x16be` | `11 01      ` | `0111    ` | `addi sp, sp,  4`  | EPILOG: corrected sequence               |
| `0x0E` | `0x16c0` | `82 40      ` | `4082    ` | `lw   ra, 0(sp)`   | EPILOG: corrected sequence               |
| `0x10` | `0x16c2` | `82 80      ` | `8082    ` | `ret`              | EPILOG: return                           |

Simply PKCS7 pad, AES-CBC encrypt using `aes_key`, and 
write the ciphertext to flash.  Write-protect, reboot,
and `SOLVE` won't even need a functioning BF script ...
the code will "just work".  At this point, you should
be able to do this trivially via python.

NOTE: The above code is untested / reconstructed from
memory, and thus may need slight tweaks before use.

</details>

## Fully Pwn your Sword of Secrets

<details markdown="1"><summary>Fully Pwn your Sword of Secrets</summary><P/>

You have 0x80 bytes of raw bytecode that you can
write, using the existing firmware.  If all the
instructions are two-byte (compressed) RISC-V,
that gives you 64 instructions that can literally
do anything you wish.

Example #1:
Dump the full, unencrypted firmware.  This fits
easily within the 0x80 byte restriction.

Example #2:
Serial loader, using the existing firmware's
functions for serial interaction, allowing
you to load any replacement firmware, without
changing what's on the device.

</details>


## FIN?

Quite likely, yes.  The firmware is fully dumped.
The source for (most of) the firmware is
available.

While I could provide the full scripts to dump
the firmware, that's not the point of this blog
post.   I hope you have enjoyed your Sword of
Secrets as much as I have.

Until next time!

