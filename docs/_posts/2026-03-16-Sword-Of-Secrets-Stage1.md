---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 1
author: Henry Gabryjelski
date: 2026-03-16 05:24:14 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 1

Welcome to my guide for the first stage of the
Sword of Secrets CTF / Hardware Escape Room.
This stage can be done with a physical
[Sword of Secrets](https://swordofsecrets.com),
or by loading the
[Virtual Challenge](https://swordofsecrets.com/mini-rv32ima.html)
in your preferred web browser.

I've tried to split the sections into background, hints,
and full-on spoilers.  However, it is still recommended
that you explore the challenge on your own before reading
anything here, as even the review section spoils some
of the basic things one must learn to interact with
the Sword.

<hr/>

## Review

<hr/>

### Review: Expectations for Stage 1

<details markdown="1"><summary>Review: Expectations for Stage 1</summary><P/>

* You interact with the Sword via a terminal (USB serial port; virtual challenge has a web-based CLI).
* It is expected that you will review the source code, and thus find the commands that can be sent via the terminal, and what they do.  See prior post for help with that.
* It is also expected that you will have the datasheet for the SPI flash chip, which shows what commands are supported by that chip.
* Term definitions:
  * `Cleartext` - original unencrypted data
  * `Plaintext` - the padded (or otherwise processed) original unencrypted data that is fed into the encryption algorithm.
  * `Ciphertext` - the encrypted plaintext data

</details>

### Review: Verify communication with SPI flash chip

<details markdown="1"><summary>Review: Verify communication with SPI flash chip</summary><P/>

First thing to do is very basic connectivity to the SPI flash chip.
Let's try to send the `9Fh` command, to get the chip ID from the chip.
This requires a sequence of commands:

1. `BEGIN` -- this enables the SPI controller, preparing it for sending and receiving data transfers to / from the SPI flash chip.
2. `ASSERT` -- tells the flash chip to pay attention to the data line ... essentially enabling the chip (`/CS`)
3. `DATA 9F` -- Sends the command bytes to the SPI flash chip.  Returned data should be zeros.
4. `DATA 0 0 0` -- Sends dummy bytes via SPI, allowing the flash chip to send back the data for the command.
5. `RELEASE` -- tells the flash chip to stop paying attention to the data line. (`/CS`)
6. `END` -- disable the SPI controller

At step 4, you should see the flash chip's ID.

Example console log from website version:
```
>> BEGIN

>> ASSERT

>> DATA 9F
00 

>> DATA 0 0 0
ef 40 18 

>> RELEASE

>> END

```

</details>

### Review: Source Code

<details markdown="1"><summary>Review: Source Code</summary><P/>

Start with parsing of your input.  You've found the commands,
so next is to figure out which code is run when you type `SOLVE`?
Which code is relevant to this first challenge?

</details>

### Review: Binary Algebra

<details markdown="1"><summary>Review: Binary Algebra</summary><P/>

Using a single bit value, XOR is defined simply as true when one or the
other inputs is true, but false if both inputs are true.  One interesting
facet is that XOR is reversible ... which allows the `xcryptXor()` function
to perform the same process for both encryption and decryption.

XOR provides a few nice features that are always true:
* `X ^ X === 0`
* `A ^ B ^ B === A`, for all values of `A` and `B`.
* If `A ^ B === C`, then `A === C ^ B` and `B === C ^ A`

That last one is actually a couple of steps.  Let me give
one in more detail:
* `A ^ B === C` -- starting point
* `A ^ B ^ B === C ^ B` -- XOR both sides by `B`
* `A ^ 0 === C ^ B` -- Replace `B ^ B` with zero
* `A === C ^ B`

We will use some of these characteristics of XOR for this challenge.

</details>

<hr/>

## Hints

<hr/>

### Hint: Source Code

<details markdown="1"><summary>Hint: Source Code</summary><P/>

We're focusing on the first challenge, so the relevant code is
in `palisadeSetup()` and `palisade()`  Take a moment to go
through those, and also review the function, `xcryptXor()`.

</details>

### Hint: Problem to be Solved

<details markdown="1"><summary>Hint: Problem to be solved</summary><P/>

Your job is to get `palisade()` to consider the data stored on
the flash valid.  If it considers the data valid, it will then
print the cleartext message, allowing you to continue to the
next challenge. 

In `palisadeSetup()`, the cleartext message is defined as:

```c
char message[] = FLAG_BANNER "{No one can break this! " S(PARAPET_FLASH_ADDR) "}";
// which becomes:
char message[] = "MAGICLIB{No one can break this! " S(PARAPET_FLASH_ADDR) "}";
//                ....-....1....-....2....-....3..  ?????????????????????  ..
char message[] = "MAGICLIB{No one can break this! 0x*****}";
//                ....-....1....-....2....-....3....-....4 == 40 bytes (0x38)
```

If the ciphertext is written to flash, then `palisade()` would
decrypt it, detect the `FLAG_BANNER` aka `"MAGICLIB"` at the start
of the plaintext, and as a result would print the plaintext and
consider the stage successfully completed.

However, the first four bytes of the ciphertext, in order to
make this a challenge, get intentionally corrupted just before
being written to flash:

```c
// Oh noes! Something happened... X_X
*((uint32_t *)(message)) = 0x00000000; // random(0xffffffff);
```

Therefore, to solve the challenge, one must fix the ciphertext
on the flash by figuring out what should have been in those
four byte, and writing the corrected data to that sector of
the flash.

</details>

<hr/>
## Walkthrough (spoilers)

<hr/>

<details markdown="1"><summary>Wrapper of spoilers</summary><P/>

### What is secret?  What is not?

<details markdown="1"><summary>What is secret?  What is not?</summary><P/>

For stage 1, the address where stage 1 (palisade) data is stored
is NOT a secret.  In fact, the opening screen flat out tells you
that you are at [spoiler]`0x10000`[/spoiler].

In addition, `FLAG_BANNER` is not a secret ... it's defined as the
eight-byte value `"MAGICLIB"` in the source code.

Reviewing `palisadeSetup()`, the message in is converted to ciphertext by
`xcryptXor()`, and the ciphertext is then written to the
stage 1 flash address `0x10000`.


</details>

### Read the stage 1 ciphertext

<details markdown="1"><summary>Read the stage 1 ciphertext</summary><P/>

Console log showing the `0x29` bytes of ciphertext generated
by `palisadeSetup()`.  There may be more bytes, as one cannot
differentiate between data of `0xFF` vs. unused flash.

That's Ok, we actually need quite a bit less data to solve
this challenge.

```
>> BEGIN

>> ASSERT

>> REM send read (03h) for 0x010000

>> DATA 03 01 00 00
00 00 00 00 

>> REM following lines read 16 bytes each

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
00 00 00 00 0e 05 13 07 36 0f 37 69 22 27 3f 65 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
2e 20 36 69 2f 3b 3f 24 26 61 2c 21 24 3a 7b 65 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
7d 39 6a 79 7d 79 6a 38 4d ff ff ff ff ff ff ff 

>> RELEASE

>> END
```

</details>

### Exploiting weakness to get `xor_key`

<details markdown="1"><summary>Exploiting weakness to get `xor_key`</summary><P/>

Let:
* `P` stands for plaintext
* `C` stands for ciphertext
* `K` stands for the key

The `xorcrypt()` function simply XOR's each byte with the next byte
of the key, looping back to the first byte of the key after using
the last byte of the key.

In other words, `P ^ K === C`.  Using the algebra trick for XOR
describe earlier, this means `P ^ C === K`.

The plaintext, `P` is nearly completely known.   Out of 0x29 values,
only five bytes near the end (as the flag) have unknown values.
I'll use underscores to indicate unknown values:

```js
00000000 4D 41 47 49 43 4C 49 42 7B 4E 6F 20 6F 6E 65 20  MAGICLIB{No one
00000010 63 61 6E 20 62 72 65 61 6B 20 74 68 69 73 21 20  can break this!
00000020 30 78 __ __ __ __ __ 7D 00                       0x?????}
```

Similarly, the ciphertext `C` is nearly completely known,
except for the first four bytes that were intentionally
corrupted.

```js
00000000 __ __ __ __ 0E 05 13 07 36 0F 37 69 22 27 3F 65
00000010 2E 20 36 69 2F 3B 3F 24 26 61 2C 21 24 3A 7B 65
00000020 7D 39 6A 79 7D 79 6A 38 4D
```

Using the knowledge that `P ^ C === K`, let's XOR the
plaintext and ciphertext, tracking the unknown / corrupted
spots with underscores, and see if we can discover the
repeated key that's in use:

```js
C 00000000 __ __ __ __ 0E 05 13 07 36 0F 37 69 22 27 3F 65
P 00000000 4D 41 47 49 43 4C 49 42 7B 4E 6F 20 6F 6E 65 20  MAGICLIB{No one
K 00000000 __ __ __ __ 4D 49 5A 45 4D 41 58 49 4D 49 5A 45  ????MIZEMAXIMIZE
                                   ^^^^^^^^^^^^^^^^^^^^^^^          \______/

C 00000010 2E 20 36 69 2F 3B 3F 24 26 61 2C 21 24 3A 7B 65
P 00000010 63 61 6E 20 62 72 65 61 6B 20 74 68 69 73 21 20  can break this!
K 00000010 4D 41 58 49 4D 49 5A 45 4D 41 58 49 4D 49 5A 45  MAXIMIZEMAXIMIZE
           ^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^  \______/\______/

C 00000020 7D 39 6A 79 7D 79 6A 38 4D
P 00000020 30 78 __ __ __ __ __ 7D 00                       0x?????}
K 00000020 4D 41 __ __ __ __ __ 45 4D                       MA?????EM
```

Because we had 30 consecutive bytes where we knew both the
plaintext and the ciphertext, it was easy to recover the
secret `xor_key`.

</details>

### Generating the corrected ciphertext

<details markdown="1"><summary>Generating the corrected ciphertext</summary><P/>

As you recall, `palisadeSetup()` overwrote the first four
bytes of ciphertext with zero bytes.  To solve this challenge,
one needs to fix the first four bytes on the flash.

Now that we know the key and the plaintext of those four bytes,
and we know that `P ^ K = C`, this is trivial:

```
Plaintext  4D 41 47 49 43 4C 49 42 7B 4E 6F 20 6F 6E 65 20  MAGICLIB{No one
xor_key    4D 41 58 49 4D 49 5A 45 4D 41 58 49 4D 49 5A 45  MAXIMIZEMAXIMIZE
RESULT     00 00 1F 00 0E 05 13 07 36 0F 37 69 22 27 3F 65
```

Great!  We replace the first four bytes of `00 00 00 00`
with the corrected value `00 00 1F 00`.

Thus, if we write the following data to the flash chip
at `0x10000`, `palisade()` should be happy!

```js
00000000  00 00 1F 00 0E 05 13 07 36 0F 37 69 22 27 3F 65
00000010  2E 20 36 69 2F 3B 3F 24 26 61 2C 21 24 3A 7B 65
00000020  7D 39 6A 79 7D 79 6A 38 4D FF FF FF FF FF FF FF
```

</details>

### Writing to SPI Flash

<details markdown="1"><summary>Writing to SPI Flash</summary><P/>

Read through the table listing all the commands the W25Q
supports.  Notice the write command (02h = PROGRAM PAGE).

That won't work (yet) because flash can only shift bits
from `1 -> 0`.   Therefore, to write the updated data,
we'll first have to erase that area of the flash, which
will reset it to `0xFF`.

If you've not played with flash before, note that while
any bit can be changed from `1 -> 0` at any time,
erase operations operate on larger sections at a time.

In this case, we will be erasing 4k (0x010000 bytes) at
a time.  Thus, erasing the first four bytes will also
erase all the other ciphertext for this challenge. In
other words, you'll want to keep a copy of the data
before erasing it.

</details>

### First try erasing the sector

<details markdown="1"><summary>First try erasing the sector</summary><P/>

Let's send the Sector Erase command, and then re-read
the data.  Here's a console log:

```
>> BEGIN

>> REM send SECTOR ERASE (20h)

>> ASSERT

>> DATA 20 01 00 00
00 00 00 00 

>> RELEASE

>> REM send READ DATA (03h)

>> ASSERT

>> DATA 03 01 00 00
00 00 00 00 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
00 00 00 00 0e 05 13 07 36 0f 37 69 22 27 3f 65 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
2e 20 36 69 2f 3b 3f 24 26 61 2c 21 24 3a 7b 65 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
7d 39 6a 79 7d 79 6a 38 4d ff ff ff ff ff ff ff 

>> RELEASE

>> END

```

Bummer, the erase didn't work,
since the data is still there.
What went wrong?

</details>

### Erasing the sector, take two

<details markdown="1"><summary>Erasing the sector, take two</summary><P/>

Looking at the datasheet, are there other commands in
that table that might need to be looked at?

Status registers allow various ways to lock the pages so
they cannot be written ... but SR1/SR2/SR3 read as
`00h`/`02h`/`00h`, which means the flash isn't write protected.

Read the descriptions `PAGE PROGRAM 02h` command.
Aha!  Check out the [spoiler]ENABLE WRITE (06h)[/spoiler]
command.  Let's try that.  Here's another
console log showing the commands:


```
>> BEGIN

>> REM send that command

>> ASSERT

>> DATA 06
00 

>> RELEASE

>> REM immediately send SECTOR ERASE (20h)

>> ASSERT

>> DATA 20 01 00 00
00 00 00 00 

>> RELEASE

>> REM and verify the data is gone

>> REM send READ DATA (03h)

>> ASSERT

>> DATA 03 01 00 00
00 00 00 00 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF FF

>> RELEASE
```

Success!  That sector is now erased, and
all bytes are `0xff`.

</details>

### Writing the updated data

<details markdown="1"><summary>Writing the updated data</summary><P/>

So the flash is successfully erased.  By now, you should
have this well in hand.   Note that the `ENABLE WRITE`
command has to be sent prior to ***EVERY*** write ... because
it auto-disables after each such write (at least on the
real physical product ... the virtual challenge may have
a bug in this area.)

Here's a console log writing the updated data:

```
>> ASSERT

>> DATA 02 01 00 00
00 00 00 00 

>> DATA 00 00 1F 00 0E 05 13 07
00 00 00 00 00 00 00 00 

>> DATA 36 0F 37 69 22 27 3F 65
00 00 00 00 00 00 00 00 

>> DATA 2E 20 36 69 2F 3B 3F 24
00 00 00 00 00 00 00 00 

>> DATA 26 61 2C 21 24 3A 7B 65
00 00 00 00 00 00 00 00 

>> DATA 7D 39 6A 79 7D 79 6A 38
00 00 00 00 00 00 00 00 

>> DATA 4D FF FF FF FF FF FF FF
00 00 00 00 00 00 00 00 

>> RELEASE

>> END
```

</details>

### Solve!

<details markdown="1"><summary>Solve!</summary><P/>

Use the `SOLVE` command to see if `palisade()`
considers your data a correct solution.

</details>

</details><hr/>

<hr/>
## FIN

If you enjoyed this first stage, and were using the
virtual challenge website, consider picking up the
physical product ... the remaining challenges are
more difficult, while each builds upon the prior
challenges.  Along the way, you'll learn about
historical mistakes in cryptography (and how to
exploit those mistakes), how to exploit bugs in
script interpreters, and more!



