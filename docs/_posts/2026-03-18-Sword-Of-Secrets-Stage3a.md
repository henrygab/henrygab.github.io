---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 3 Preparation
author: Henry Gabryjelski
date: 2026-03-18 07:50:23 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 3 - Part A

<hr/>

## Review

### Things you've learned

<details markdown="1"><summary>Things you've learned</summary><P/>

You've already solved stages 1 and 2, so you:
* can erase parts of the flash chip
* can write data to parts of the flash chip
* are somewhat familiar with the source code
* are somewhat familiar with block-based
  AES cryptography used in `ECB` mode
* can check if you've progressed via `SOLVE` command

Definitions:
* cleartext === original data to encrypt
* plaintext === data provided to the encryption routine, typically padded
* ciphertext === the encrypted data corresponding to the plaintext

</details>

### Things you will learn

<details markdown="1"><summary>Things you will learn</summary><P/>

For this stage, a second method of applying
AES cryptography is used insecurely.

* Learn about `Cipher Block Chaining` (`CBC`) mode.
* Learn about PCKS7 padding
* Learn about weaknesses with AES in `CBC` mode (... specifics redacted until later ...)
* Applying those learned weaknesses to get the next flag.

</details>

### ECB and CBC Block Ciphers

<details markdown="1"><summary>ECB and CBC Block Ciphers</summary><P/>

The second challenge used AES in `ECB` mode, and
this third challenge uses AES in `CBC` mode.  Let's
take a moment to brush up on how `CBC` mode works.

#### `ECB` Refresher

The prior stage used `ECB` mode, which encrypts and decrypts
each block independently.

Thus, for `ECB` mode, encryption and decryption were
given by:
```
Cᵢ = Eₖ( Pᵢ )
Pᵢ = Dₖ( Cᵢ )
```

Where:
* `Eₖ(·)` denotes the block-cipher encryption under key `k`
* `Dₖ(·)` denotes the block-cipher decryption under key `k`
* `Cᵢ` denotes the ciphertext for the `ᵢ`th block
* `Pᵢ` denotes the plaintext for the `ᵢ`th block

As can be seen, with `ECB` mode, both encryption and
decryption rely ***ONLY*** on the key and the data
of the current block being encrypted or decrypted.

#### `CBC` Encryption and Decryption

Another mode called `Cipher Block Chaining` addresses
some of the issues with `ECB`, by ensuring the prior
block's ciphertext will impact the encryption of the
subsequent block (hence the term "chaining").

In `CBC` mode encryption, the ***ciphertext*** of
the prior block is `XOR`'d with the ***cleartext***
of the current block, ***prior to*** performing the
encryption of the current block.  This prevents the
simple modification attacks (described in Stage 2)
that are possible when `ECB` mode was used, because
any modification to the plaintext of one block
would cause changes to the ciphertext of all future
blocks (in a non-predictable way).

For `CBC` decryption, the ciphertext of the current
block is decrypted, and then this intermediate data
is `XOR`'d with the ***ciphertext*** of the prior
block.

Thus, for `CBC` mode, encryption and decryption are
given by:

```
Cᵢ = Eₖ( Pᵢ ⊕ Cᵢ₋₁ )
Pᵢ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
```

Where:
* `Eₖ(·)` denotes the block-cipher encryption under key `k`
* `Dₖ(·)` denotes the block-cipher decryption under key `k`
* `Cᵢ` denotes the ciphertext for the `ᵢ`th block
* `Pᵢ` denotes the plaintext for the `ᵢ`th block
* `⊕` represents the XOR operation

</details>

### Programmatic access

<details markdown="1"><summary>Programmatic Access</summary><P/>

This is the first stage that, practically
speaking, ***requires*** programmatic interaction
with the CTF board communications.

You ***could*** do all the steps by hand.
However, it's will quickly become quite
tedious, as many steps guess or brute-force
some change in the data on flash.  Each
iteration of those steps requires erasing a
sector of flash, writing the updated data,
performing additional commands, and then
acting based on the response from the Sword.

On average, one might have to perform
the above ... ~2048 cycles / iteractions.

Thus, it's ***strongly*** recommended to
write some scripts that programmatically
send data via the serial port, receive
the response data, and parse it to a more
useful form so you can easily add your
own logic / scripts and automate some of
the repetitive parts required for this stage.

Here are some python libraries I used, which you
might find useful if you also use python to
script parts of the interaction.

```
    "pyserial>=3.5",
    "pyserial-asyncio>=0.6",
    "prompt_toolkit>=3.0",
    "pycryptodome>=3.19",
```

</details>

### Recommended helper functions

<details markdown="1"><summary>Recommended helper functions</summary><P/>

The cycle of sending a single command via the Sword's
serial port is repetitive:
* `BEGIN` to setup the SPI peripheral on the MCU
* `ASSERT` to make the flash chip start listening
* `DATA` to send the command
* `DATA` when getting the flash chip's response
* `RELEASE` to signal to the flash chip the end of that command
* Sometimes, additional `ASSERT`, `DATA`*, `RELEASE` cycles (e.g., commands needing `WRITE ENABLE`)
* `END` to release the SPI peripheral on the MCU

Here's some thoughts on useful base functions which
have served me well, and would be the minimum set
that I'd recommend having as a starting point.

* `read_flash <address> <count>` -- single `DATA` to send the command, followed by an additional `DATA` for every 16 bytes to be read.   Collect the response from the Sword and return as a `bytearray` or similar.
* `erase_flash_4k <address>` -- restrict address to be 4k aligned, starts with `WRITE ENABLE`, followed by the `ERASE 4k` command.
* `write_flash <address> <hex_data>` -- User should have already erased.  `WRITE_ENABLE`, `PROGRAM PAGE`, and then reading it back to ensure it was written correctly.

</details>
<hr/>


## Hints

### Setup function in the Source

<details markdown="1"><summary>Setup function in the Source</summary><P/>

We're focusing on the third challenge, so the first
functions relevant are in `posternSetup()` and
`postern()`.  Let's start with a deeper look at what
`posternSetup()` does.

`posternSetup` defines the cleartext as:

```c
char message[128] = FLAG_BANNER "{Passwd: " FINAL_PASSWORD "}";
char message[128] = "MAGICLIB{Passwd: " FINAL_PASSWORD "}";
char message[128] = "MAGICLIB{Passwd: " "XXXXXX" S(PLUNDER_ADDR_DATA) "}";
// I'm presuming five hex digits... which will give exactly
// one byte of padding and total ciphertext lenght of 0x20.
// If S(PLUNDER_ADDR_DATA) creates a longer string, then
// the ciphertext length will be 0x30 (or greater).  Since
// we know the ciphertext length is 0x20 bytes, we know
// that S(PLUNDER_ADDR_DATA) is five bytes or fewer in length.
char message[128] = "MAGICLIB{Passwd: XXXXXX0x_____}";
//                   ....-....1....-....2....-....3..
//                   \__ block 1 ___/\__ block 3 ___/
```

The cleartext is PKCS7 padded into plaintext via
`PKCS7Pad(message, strlen(message))`,  which AES-CBC
encryption is applied to the plaintext blocks to generate
the ciphertext.

Finally, before writing the ciphertext to the flash media,
`posternSetup()` intentionally corrupts the last byte of the
penultimate (second-to-last) block of the ciphertext.
In this case, since there's only two blocks of AES data,
this corrupts the last byte of the first AES block of ciphertext:

> `message[len - AES_BLOCK_LEN - 1] = '\0'`

Similar to prior stages, `posternSetup()` writes the length
of the ciphertext to the first four bytes of flash at `0x30000`,
followed by the actual ciphertext bytes.

You should confirm this by looking at the data stored on
the flash at `0x30000`, and specifically the `0x00` byte
corruption at `0x30013`:

```
00030000: 20 00 00 00 f7 60 4a 1f 5e 96 39 7e 96 f5 9e 31   ....`J.^.9~...1
00030010: 72 0b d9 00 d7 6b ed c8 d1 d1 47 34 81 46 9a 24  r....k....G4.F.$
00030020: bf aa 90 22 ff ff ff ff ff ff ff ff ff ff ff ff  ..."............
```

</details>

### Validation function in the source

<details markdown="1"><summary>Validation function in the source</summary><P/>

The validation function called when using `SOLVE`
is `postern()`.  That function  will read three
things from ~`0x30000`:
* `len` field at offset zero (limited to 128)
* `len` bytes into `message` (ciphertext) at offset `4`
* `len` bytes into `response` at offset `4+len`

The `message` is `AES-CBC` decrypted, and the PKCS7
padding is validated.  If the padding is incorrect,
`postern()` returns error `1`.

The PKCS7 padding is removed (null at first byte),
ensuring it's a null-terminated string (if correct).

The `response` data on the flash is then `memcmp()`
with `FINAL_PASSWORD`.  If the response data does
not match, `postern()` returns error `-1`.

Otherwise, all is verified, the message is printed,
and `postern()` returns `0`.

</details>

### What weaknesses in AES-CBC?

<details markdown=1><summary>What weaknesses in AES-CBC?</summary>

Search for and learn about "Padding Oracles".

</details>

### Anything special about the last block of plaintext?

<details markdown=1><summary>Anything special about the last block of plaintext?</summary>

As you know, AES is a block cipher, which means every encryption
operation operates on a fixed block of bytes.  For AES, the block
size is 16 bytes.  This means that the last bytes of cleartext
need to have some padding added.

The specific padding technique here is PKCS7.  In PKCS7, the
last plaintext byte of the last block is always a padding byte.
Each padding byte's value indicates how many padding bytes
were added to the data.  Thus, there are only sixteen valid
values for the last byte of plaintext: `0x01` through `0x10`.

Moreover, we can enumerate all the valid padding blocks.
As before, I use underscores for values that are unknown,
or in this case, don't affect the validity of the padding block:

```
__ __ __ __ __ __ __ __ __ __ __ __ __ __ __ 01
__ __ __ __ __ __ __ __ __ __ __ __ __ __ 02 02
__ __ __ __ __ __ __ __ __ __ __ __ __ 03 03 03
__ __ __ __ __ __ __ __ __ __ __ __ 04 04 04 04
__ __ __ __ __ __ __ __ __ __ __ 05 05 05 05 05
__ __ __ __ __ __ __ __ __ __ 06 06 06 06 06 06
__ __ __ __ __ __ __ __ __ 07 07 07 07 07 07 07
__ __ __ __ __ __ __ __ 08 08 08 08 08 08 08 08
__ __ __ __ __ __ __ 09 09 09 09 09 09 09 09 09
__ __ __ __ __ __ 0A 0A 0A 0A 0A 0A 0A 0A 0A 0A
__ __ __ __ __ 0B 0B 0B 0B 0B 0B 0B 0B 0B 0B 0B
__ __ __ __ 0C 0C 0C 0C 0C 0C 0C 0C 0C 0C 0C 0C
__ __ __ 0D 0D 0D 0D 0D 0D 0D 0D 0D 0D 0D 0D 0D
__ __ 0E 0E 0E 0E 0E 0E 0E 0E 0E 0E 0E 0E 0E 0E
__ 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F 0F
10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10
```

</details>

### Deep hints...

<details markdown=1><summary>Deep hints...</summary>

Definitions:
* cleartext === original data to encrypt
* plaintext === data provided to the encryption routine, typically padded
* ciphertext === the encrypted data corresponding to the plaintext

Recall the definition of how AES-CBC decrypts data:

```
Pᵢ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
```

Questions:

1. Ignoring negative side effects to anything other than the
   last AES block, can you devise a way to modify a single
   byte of the ***plaintext*** of that last AES block of data?
2. Can you control which byte of that last AES block's
   plaintext get changed?
3. If you know the plaintext value of a byte (e.g., `0xYY`)
   in that last AES block of data, can you extend the
   technique to change the plaintext value to a value
   of your choosing (e.g., from `0xYY` to `0xZZ`)?

</details>

### Example data with all-zero AES key

There's no better way to understand some of these things,
than to pad, encrypt, and decrypt some data yourself.
In support of that, here's some example data that I'll
show the encryption and decryption of, using an all-zero
AES key.

<details markdown="1"><summary>Example Data</summary><P/>

#### Example with all-zero AES key

Let's pretend that `FINAL_PASSWORD` is `OMEGA`,
instead of `"XXXXXX" S(PLUNDER_ADDR_DATA)` ... 
primarily because I wrote this before knowing
the likely format of the secret, but also 
because it provides a useful tool to show other
optimizations.

Here, that would make the cleartext:

```c
char cleartext[128] = "MAGICLIB{Passwd: OMEGA}";
//                     ....-....1....-....2....-....3..
//                     \__ block 1 ___/\__ block 3 ___/
```

That gives 0x17 bytes of cleartext, which requires
0x09 bytes of PKCS7 padding to be added to create
the block-aligned plaintext that AES can process:

```c
// "MAGICLIB{Passwd: OMEGA}" ("\x09" * 9)
unsigned char plaintext[0x20] = {
    // AES Block 0
    0x4D, 0x41, 0x47, 0x49,  0x43, 0x4C, 0x49, 0x42,
    0x7B, 0x50, 0x61, 0x73,  0x73, 0x77, 0x64, 0x3A,
    // AES Block 1
    0x20, 0x4F, 0x4D, 0x45,  0x47, 0x41, 0x7D, 0x09,
    0x09, 0x09, 0x09, 0x09,  0x09, 0x09, 0x09, 0x09,
};
```

#### Generate the ciphertext

Using the all-zero AES key and AES-CBC,
this gives the corresponding ciphertext:

```
00000000  d1 3d b3 73 f7 80 d1 7e  |Ñ=³s÷.Ñ~|
00000008  0f 12 f4 1d a4 c2 c2 12  |..ô.¤ÂÂ.|

00000010  bb bb 93 70 65 88 b8 a0  |»».pe.¸ |
00000018  44 73 63 eb 1c 23 1a 31  |Dscë.#.1|
```

#### Corrupt it (just like the challenge)

Byte 0x0F is set to zero to corrupt the data.
This is equivalent to `XOR`'ing that byte with
its prior value of `0x12`:

```
00000000  d1 3d b3 73 f7 80 d1 7e  |Ñ=³s÷.Ñ~|
00000008  0f 12 f4 1d a4 c2 c2 00  |..ô.¤ÂÂ.|

00000010  bb bb 93 70 65 88 b8 a0  |»».pe.¸ |
00000018  44 73 63 eb 1c 23 1a 31  |Dscë.#.1|
```

#### What did that corruption do?

We know that modification of a single byte of the ciphertext
will scramble the contents of the entire corresponding
block of plaintext ... after all, that's a key feature
of any good cipher.

But what actually happens to the decrypted ciphertext
based on the above corruption?

The first block decrypts to an unpredictable set of `0x10`
bytes.   But what does the second AES block decrypt to?

Recall how the decryption works:

```
Pᵢ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
```

Thus, if you modify just the last byte of the previous
block's ***ciphertext***, what happens to the last byte
of the current block's ***plaintext***?

Here, I'll use `Pᵢ` to refer to the original plaintext,
and `Pᵢᵩ` to refer to the plaintext when decrypting
with the prior block having that single-byte corruption.

Note: apply the listed XOR operation only the last byte,
because that's the only byte corrupted.

```
REM Definition of the decryption operation
Pᵢ  = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
REM Corrupt that last byte by XOR with 0x12
Pᵢᵩ = Dₖ( Cᵢ ) ⊕ ( Cᵢ₋₁ ⊕ 0x12)
REM Remove parenthesis since XOR is commutative
Pᵢᵩ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁ ⊕ 0x12
REM XOR both sides with 0x12
Pᵢᵩ ⊕ 0x12 = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁ ⊕ 0x12 ⊕ 0x12
REM Simplify right side, because X ⊕ X === 0
Pᵢᵩ ⊕ 0x12 = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
REM Right side is the definition of Pᵢ
Pᵢᵩ ⊕ 0x12 = Pᵢ
```

The first part (`Dₖ( Cᵢ )`, the AES block decryption of `Cᵢ`)
is ***NOT*** affected by the previous block's corrupt byte.

*(Yes, the **prior** block gets fully corrupted...
but the **final** block has byte-level controlled changes)*

Instead, because `Dₖ( Cᵢ )` is simply XOR'd with `Cᵢ₋₁`,
ONLY THE CORRESPONDING BYTE OF THE PLAINTEXT IN
THE FINAL BLOCK gets modified ... no other byte of
the final block of plaintext is affected.

Read that last sentence again ... it's the core concept.

Moreover, that corresponding byte gets corrupted
in an interesting and useful manner.

</details>

## FIN

There's a LOT to do at this point, just to explore
the Sword more easily, start getting programmatic
access, playing around with AES-CBC, and testing
stuff out with a test key (e.g., all-zero).

Take some time, explore, enjoy, and see what you
can come up with.

Next posts might be the deep walkthrough, or
I might clean up and post my python serial port
helpers.  So, it might take a bit longer for
the next post.....
