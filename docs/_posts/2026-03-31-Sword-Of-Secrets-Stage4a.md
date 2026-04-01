---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 4a - Review and Early Hints
author: Henry Gabryjelski
date: 2026-04-01 00:32:07 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 4 - Part A - Review and Early Hints

Cross-platform, python-based programmable serial console is now available at [https://github.com/henrygab/sos_helper](https://github.com/henrygab/sos_helper).  Designed as a building block to your own scripting exploration of the Sword of Secrets.

The excitement from solving Stage 3 is somewhat short-lived,
as you realize that when the sword attempts to verify Stage 4,
it locks up the device.   Stage 4 will build upon what you've
learned in Stage 3, but also dive into two entirely different
areas.

<hr/>

## Review

### Things you've learned

<details markdown="1"><summary>Things you've learned</summary><P/>

You've already solved stages 1, 2, and 3, so you:
* can erase parts of the flash chip
* can write data to parts of the flash chip
* are somewhat familiar with the source code
* are somewhat familiar with block-based
  AES cryptography used in `ECB` mode
* are somewhat familiar with block-based
  AES cryptography used in `CBC` mode
* can check if you've progressed via `SOLVE` command

Moreover, for ciphertext encrypted using AES in `CBC`
mode, if you have a Padding Oracle, you can also:
* Cause the final block to decrypt into valid PKCS#7
  padding (by corrupting the prior block).
* Detect the size of the cleartext data by determining
  the number of bytes of padding used.
* Force the final block to be a full 16-byte padding
* and even decrypt that final block of ciphertext
  (`sos_helper` takes ~35 minutes per block)

Definitions:
* cleartext === original data to encrypt
* plaintext === data provided to the encryption routine, typically padded
* ciphertext === the encrypted data corresponding to the plaintext

</details>

### Things you might learn

<details markdown="1"><summary>Things you might learn</summary><P/>

There are multiple solutions to this stage.  The first set of
posts will focus on the "intended" solution.  As part of that,
you will:
* Figure out why the MCU hangs when checking the stage 4 solution.
* Learn about byte code that corresponds to the assembly code.
* Learn about so-called "Brainfuck" (hereinafter, BF) interpreters.
* Combine the above with your abuse of the AES CBC padding oracle
  to solve the challenge.

</details>

## Hints

### What happens during Solve

<details markdown="1"><summary>What happens during Solve</summary><P/>

`challenges()` will walk through and validate
stages 1-3 (`palisade()`, `parapet()`, and
`postern()`), as noted in prior steps.

After that, `challenges()` will:
* call `void digForTreasure(void)`
* call `int treasuryVisit(void)`

Success is indicated when `treasuryVisit()` returns a non-negative value.

* `treasuryVisit()` jumps to the code in the global variable `data[]`,
  as found in `armory.c`, but...

**Where does `data[]` get its data from?**

Pause here ... look through the code ... because the next
section spells out the answer.

</details>

### Helpful notes

<details markdown="1"><summary>Helpful notes</summary><P/>

There are two similarly-named `#define`'d symbols:
* `PLUNDER_ADDR_DATA` ... which contains script interpreted by
  a Brainf*** interpreter.  I'll cover the details later ...
  but for now this symbol can be ignored.
* `PLUNDER_ADDR` 
  * This is the symbol to look for.  As you'll see,
    the setup function:
    * reads the raw bytes of the function `swordOfSecrets()`
    * encrypts them using AES in `CBC` mode
    * writes that encrypted data to this other address
  * Later in boot, but ***only once per boot***, another function:
    * loads the encrypted data from that location in flash
    * decrypts the function
    * stores the decrypted function in the global variable
      `data[]` found in file `armory.c`.

As a result, before you even get a terminal to appear,
all the above steps have occurred, and the data in
that global `data[]` has already been written.

</details>

### Problem(s) to be solved

<details markdown="1"><summary>Problem(s) to be solved</summary><P/>

1. Figure out what code is crashing
2. Figure out a way to prevent that code from crashing
3. ... and do so in a way that returns a non-negative value

It's a real brain teaser.

</details>

</details>

## Deep Hint

This is separately protected, as it may be considered too
strong of a hint for some.

<details markdown="1"><summary>Deep Hint</summary><P/>

### Write-protecting the flash chip

My solutions for stage 4 all require learning
how to write-protect the SPI flash chip.
The chip's datasheet lists a number of ways
in which to write-protect the chip ...
all of them by setting various bits in the
Status Registers.

The simplest method to write-protect the flash is
to set SR3 bit 2 (`WP`):  `SR3 |= 0x04`.  Similarly, to
allow writing again, clear SR3 bit 2: `SR3 &= 0xFB`.
As with many writes, attempting to change `SR3` requires
first sending the `WRITE ENABLE 06h` command.


In BusPirate5 script form:

```
   [ 0x06 ] [ 0x11 0x04 ] D:30
```

Using the serial port of the Sword, it's slightly more verbose:

```
BEGIN
ASSERT
DATA 06
RELEASE
ASSERT
DATA 11 04
RELEASE
```

However, if you're using the `sos_helper` module, you can
add two commands similar to:

```python
from . import sword_of_secrets as sos

async def cmd_enable_write_protect(_: str, ctx: CommandContext):
   sos.set_write_protect_state(sos.WriteProtectionType.PROTECT_ALL, ctx)
async def cmd_disable_write_protect(_: str, ctx: CommandContext):
   sos.set_write_protect_state(sos.WriteProtectionType.NONE, ctx)
```


Note that the write protection setting ***WILL***
survive a power cycle, so when you later wonder why you
can no longer write, hopefully this note will remind
you so you don't spend hours wondering and debugging
why your scripts that previously worked fine suddenly
stopped working.  (ask me how I know....)

</details>


## FIN

While the intended solution for Stage 4 builds upon
and uses your your learnings from Stage 3, it also
takes a turn into some new areas.  There's multiple
pieces that have be combined, and I found the
combination a nice challenge.

Until the next post, enjoy poking at stage 4,
and don't forget to clone the `sos_helper` repo ...
it makes scripting your interactions with the
sword much easier.
