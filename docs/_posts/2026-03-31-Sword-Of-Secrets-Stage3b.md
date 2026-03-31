---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 3 - Part B
author: Henry Gabryjelski
date: 2026-03-31 03:16:33 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Stage 3 - Part B

Cross-platform, python-based programmable serial console is now available at [https://github.com/henrygab/sos_helper](https://github.com/henrygab/sos_helper).

Hints and review are provided in the [prior post](https://henrygab.github.io/sword%20of%20secrets/2026/03/18/Sword-Of-Secrets-Stage3a.html), with the deep hints providing questions that guide you in the right direction.

There's also example data to walk through, showing the mechanism in
action.   It's well worth working through before reading this post.

<hr/>

## Walkthrough

<details markdown="1"><summary>Wrapper of spoilers</summary><P/>

### Things you will apply

<details markdown="1"><summary>Things you will apply</summary><P/>

For this challenge, you will learn how to apply the
"padding oracle" attack in four stages:
1. using the padding oracle to correct the one-byte corruption
   the setup code for this challenge intentionally introduced.
2. Detecting how many padding bytes result from decrypting
   that corrected ciphertext.
3. using the padding oracle to expand the padding in that
   final AES block until it takes up the entire 16 bytes.
4. once the final AES block is fully expanded to 16 bytes,
   decrypting the plaintext of that final AES block.
5. Writing the solution to the flash

Once you've decrypted the original plaintext, writing the
full solution data back to the device, thus solving stage 3.

</details>

## Part 1 -- fixing the intentional one-byte corruption

<details markdown="1"><summary>Part 1 -- fixing the intentional one-byte corruption</summary><P/>

If you recall, for AES in `CBC` mode, decryption of the
ciphertext into plaintext using AES in `CBC` mode is
given by:

```
Pᵢ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
```

Or, in English, you first decrypt the AES block, and
then XOR that result with the ciphertext of the prior
AES block.

Therefore, when the challenge setup corrupted the last
byte of the first AES block, it also corrupted the
last byte of the second AES block. (there are only
two AES blocks in this challenge.)   This corrupted
the padding.

Luckily, the program will print a different message
when the padding is correct, vs. when the padding is
wrong.  This is the "Padding Oracle".

So, simply re-write the sector (up to `0x100` times,
with values from `0x00` .. `0xFF`), and send the
`SOLVE` command after each modification. Eventually,
you'll get an indication that the padding is correct.

Go ahead and script that in Python, using
the `sos_helper` program listed above.

Huzzah!  That's the first part.

</details>

## Part 2 -- Detecting how many padding bytes...

<details markdown="1"><summary>Part 2 -- Detecting how many padding bytes...</summary><P/>

As you may recall, PKCS#7 padding can use between 1 and 16 bytes
(inclusive) because the AES block size is 16 bytes.  Therefore,
there are exactly sixteen possible padding sizes that might be
in use, and so you need to first determine which one was in use.

Using underscores to indicate "don't care" bytes, here's the
list of all possible paddings:

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

If the Oracle says that the padding is correct,
and you modify the first block's first byte of
ciphertext, that will change the first byte of
the plaintext for the last block.  (If this
sentence isn't making sense, walk through the
example data in the [prior post](https://henrygab.github.io/sword%20of%20secrets/2026/03/18/Sword-Of-Secrets-Stage3a.html),
found under the "Deep Hints" section.)

So, if you modify the first byte of plaintext
and the Oracle now says the padding is wrong,
then the only PKCS#7 padding that would occur
with is where all sixteen bytes are padding.
If this happens, you're quite lucky!

Most likely, the Oracle is still happy.  Now
modify the second byte of plaintext ... which
if the Oracle says makes the padding wrong,
means it's fifteen bytes of padding.

Lather, rinse, repeat ... until you know how
many bytes of padding are in the corrected
blob of ciphertext.

</details>


## Part 3 -- Expanding the padding bytes...

<details markdown="1"><summary>Part 3 -- Expanding the padding bytes...</summary><P/>

Now that you discovered that there was only a single byte
of PKCS#7 padding, it's time to begin the iterative process
of expanding that padding.

The crux of solving this goes back to the definition of
how AES in `CBC` mode decrypts data, mixed in with some
algebraic manipulation made possible by the use of XOR.

AES decryption (for `CBC` mode) is given, then both sides
are XOR'd by `Cᵢ₋₁`, simplifying based on `A ^ A === 0`,
and then swap sides:

```
Pᵢ         = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
Pᵢ ⊕ Cᵢ₋₁ = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁ ⊕ Cᵢ₋₁
Pᵢ ⊕ Cᵢ₋₁ = Dₖ( Cᵢ )
Dₖ( Cᵢ )   = Pᵢ ⊕ Cᵢ₋₁
```

This means that you can change any byte of the plaintext
for a given block, by changing the corresponding byte
of the prior block.  If you know the value of the
decrypted plaintext byte, and you know the value of the
corresponding byte of the prior ciphertext block, then
you can reliably change the value of the decrypted
plaintext byte ... by corrupting the prior block's data.

Yes, this will result in an unpredictable block of
decrypted data in the prior block. That's OK, since the
secret you're looking for is in the second / last AES
block.

Right now, you know that the last byte of the last AES
block, when decrypted, gives a plaintext byte with the
value `0x01` (because it's PKCS#7 padding).

You also know the ciphertext of the prior block for
that same byte.  Plugging them into the above equation:

```
Pᵢ           = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
0x01         = Dₖ( Cᵢ ) ⊕ 0x32
// And you want:
0x02         = Dₖ( Cᵢ ) ⊕ 0x32
```

How to get there?
If you XOR the current plaintext by its value, you
will get zero.  Then, XOR it by the desired value,
and you get the desired value.

Just do that same operation on both sides of the
equals sign:

```
Pᵢ           = Dₖ( Cᵢ ) ⊕ Cᵢ₋₁
0x01         = Dₖ( Cᵢ ) ⊕ 0x32
0x01 ⊕ 0x01 = Dₖ( Cᵢ ) ⊕ 0x32 ⊕ 0x01
0x00         = Dₖ( Cᵢ ) ⊕ 0x33
0x00 ⊕ 0x02 = Dₖ( Cᵢ ) ⊕ 0x33 ⊕ 0x02
0x02         = Dₖ( Cᵢ ) ⊕ 0x31
```

Thus, you modify the last byte of the prior block's
ciphertext from `0x32` to `0x31`, and this will change
the plaintext of the final byte from `0x01` to `0x02`,
giving you plaintext:

```
__ __ __ __ __ __ __ __ __ __ __ __ __ __ ?? 02
```

The question marks represent the unknown byte, where
you need to brute-force the value until the Oracle
says the padding is valid.  This is the same process
you did for that last byte, only now you're modifying
the second-to-last byte of the penultimate block's
ciphertext (instead of the last byte).

When completed, your ***penultimate ciphertext*** block
will now be:

```
__ __ __ __ __ __ __ __ __ __ __ __ __ __ a6 31
```

Resulting in the ***final plaintext*** block:
```
__ __ __ __ __ __ __ __ __ __ __ __ __ __ 02 02
```

Increment both values from `0x02` to `0x03`,
by XORing the penultimate block's padding bytes
by `0x02 ⊕ 0x03 == 0x01`, to get penultimate ciphertext:

```
__ __ __ __ __ __ __ __ __ __ __ __ __ ?? a7 30
```

Lather, rinse, repeat ... until you've done all
sixteen bytes.

</details>

## Part 4 -- Decrypting the final block's plaintext

<details markdown="1"><summary>Part 4 -- Decrypting the final block's plaintext</summary><P/>

From the completion of part 3, for the penultimate AES
block, you now have:
* The original ciphertext (with corrupt byte fixed)
* ciphertext which causes the final block's plaintext
  to be the 16-byte PKCS#7 padding block:

```
10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 
```

And you want to find the plaintext of the original
data stored on flash (with the corrupt byte fixed).

```
b`\x10` * 16 = Dₖ( Cᵢ ) ⊕ c7 ... 2b b4 23
????         = Dₖ( Cᵢ ) ⊕ f7 ... 0b d9 32
// XOR the two equations so Dₖ( Cᵢ ) cancels out
???? ⊕ (b`\x10`*16) =    30 ... 20 6d 11
// XOR both sides by the padding
???? = 20 ... 30 7d 01
```

And ... TADA!  You have decrypted the last block,
without ever knowing what key was used to encrypt
it.

</details>

## Part 5 -- Writing the solution to the flash

<details markdown="1"><summary>Part 5 -- Writing the solution to the flash</summary><P/>


In the prior step, you determined the decrypted last
AES block was:
```
20 35 33 52 33 37 20 30 78 35 30 30 30 30 7d 01    53R37 0x50000}.
```

What does `postern()` need, to consider the challenge solved?

https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/armory.c#L127-L153

The flash must have a four-byte length of the encrypted data,
e.g., `20 00 00 00`, followed by the corrected ciphertext of
that length, followed by the ***plaintext*** `FINAL_PASSWORD`.

what part of your decrypted data corresponds to `FINAL_PASSWORD`?

https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/armory.c#L411

Filling that in, step by step, with the additional knowledge
that there is exactly one padding byte required (thus
strlen(cleartext) is 0x1E aka 31 bytes):

```
FLAG_BANNER "{Passwd: " FINAL_PASSWORD "}";
MAGICLIB{Passwd: _____________}
....-....1....-..**2****-****3.
________________ 53R37 0x50000}
```

Thus FINAL_PASSWORD is `53R37 0x50000`.

Putting it all together, you then must write the following
data to the flash chip at `0x30000`:

```
    # length of encrypted data field
    b'\x20\x00\x00\x00' +
    # first AES-CBC block (w/corrected byte)
    b'\xf7\x60\x4a\x1f\x5e\x96\x39\x7e' +
    b'\x96\xf5\x9e\x31\x72\x0b\xd9\x32' +
    # second AES-CBC block
    b'\xd7\x6b\xed\xc8\xd1\xd1\x47\x34' +
    b'\x81\x46\x9a\x24\xbf\xaa\x90\x22' +
    # Solution cleartext... no PKCS#7
    b'\x35\x33\x52\x33\x37\x20\x30\x78' +
    b'\x35\x30\x30\x30\x30'
```

</details>

## Verifying it worked

<details markdown="1"><summary>Verifying it worked</summary><P/>

As with every other stage, the next step is to see if the
solution is accepted.  Run `SOLVE`, and if it worked, you
will see the following printout:

```
MAGICLIB{No one can break this! 0x20000}
MAGICLIB{53Cr37 5745H: 0x30000}
MAGICLIB{Passwd: 53R37 0x50000}
```

That's great!  ... but, as you quickly notice, not
all is well.  Press the hardware reset button to
get back to a usable prompt ... the hardware will
have locked up when it tries to validate stage 4.

But ... that's a whole new challenge for you,
and a story for another day.

</details>

</details>

## FIN

Stage 4 is significant, and while it builds on
your learnings from Stage 3, it also takes a turn
into new areas.

Stage 4 will be multiple posts, and will once
again have summary of things to learn about,
hints, and separate walkthrough.

Until then, enjoy poking at stage 4!
