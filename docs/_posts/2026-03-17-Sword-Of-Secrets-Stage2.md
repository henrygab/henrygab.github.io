---
status: published
published: true
layout: post
title: Sword of Secrets - Stage 2
author: Henry Gabryjelski
date: 2026-03-17 06:54:37 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---


# Stage 2

It's recommended to try to solve each stage on your own.
Here, I will try to separate background learning into a
"Review" section, help defining the problem into a "Hints"
section, and finally provide a full walkthrough section.

If you were stuck before reading one of those sections,
I encourage you to go back and look at the code again ...
each of the Review and Hints section provides learnings
that might trigger your subconcious to look at things
in a new way, and it's so much more fun to solve the
challenges yourself (even if you use hints).

Without further ado, the three sections...

## Review

### Expectations for Stage 2

<details markdown="1"><summary>Expectations for Stage 2</summary><P/>

You've already solved stage 1, so you can now erase parts of the flash chip,
write data to parts of the flash chip, you're somewhat familiar with
the source code, and can check if you've progressed via `SOLVE` command.

For this stage, basic AES cryptography is used insecurely.  You need to
learn about `Electronic Code Book` aka `ECB` mode, and how it generally
works to encrypt large blobs of data.

</details>

### ECB background, part 1

<details markdown="1"><summary>ECB background, part 1</summary><P/>

To keep the data size small, let's pretend that AES ECB works on 4-byte
chunks instead of 16-byte chunks.  Let's also pretend that we are using
an AES key that encrypt the following chunks as noted.  Obviously, this
data is simply to make it easier to know at a glance the transition from
plaintext to ciphertext and back.

plaintext     | ciphertext    | notes
--------------|---------------|-------------
`00 00 00 00` | `89 AB CD EF` | zero ... volunteers
`11 00 00 11` | `AA 87 54 AA` | typical employee
`87 65 56 78` | `BA AD F0 0D` | CEO ... big bucks
`04 04 04 04` | `FF FF FF FF` | PKCS padding block (not interpreted as data if it's the last block)

Now let's imagine you're encrypting a database that's a sorted
list (by employeeID) of how much the person was paid, and each
record of payment is exactly four bytes.

Although encrypted, you can see still see the ciphertext that's in the
database.  We know that there is only one CEO, exactly three volunteers,
and the rest of the folks are paid a standard salary.

The encrypted database column (sorted by Employee ID) looks like this:

```
BA AD F0 0D
89 AB CD EF
89 AB CD EF
AA 87 54 AA
AA 87 54 AA
AA 87 54 AA
AA 87 54 AA
89 AB CD EF
AA 87 54 AA
AA 87 54 AA
FF FF FF FF
```

**Can you tell from the above which 4-block ciphertext corresponds
to the CEO's payment?**  

</details>

### ECB background, part 2

<details markdown="1"><summary>ECB background, part 2</summary><P/>

Recall that there are three volunteers, a bunch of employees
paid a standard salary, and only one CEO.

The encrypted database (split into 4-byte chunks for this example)
has the ciphertext `89 AB CD EF` three times, the ciphertext
`AA 87 54 AA` six times, but the ciphertext `BA AD F0 0D` only
once. (The last four bytes are padding.)

This means the ciphertext for the CEO's payment is `BA AD F0 0D`,
even if we don't know the actual value of the payment.

</details>

### ECB background, part 3

<details markdown="1"><summary>ECB background, part 3</summary><P/>

Now let's imagine an attacker somehow got the ability to write
to the database, but does not have access to the encryption key.

What sort of mayhem could be done?


They then changed the values to have the following:

```
89 AB CD EF  (was BA AD F0 0D aka CEO payment)
89 AB CD EF
89 AB CD EF
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
89 AB CD EF
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
BA AD F0 0D  (was AA 87 54 AA aka employee payment)
FF FF FF FF
```

Because ECB mode ***independently*** encrypts and decrypts each
block, the above modified database decrypts successfully.
However, the CEO's record now shows a payment equal to what the
volunteers were paid (e.g., likely nothing), and the standard
employees were all modified to get the CEO's payment amount.
Chaos ensues.

Put another way, an attacker is free to re-order individual
blocks of ciphertext when encoded in ECB mode, and each block
will decrypt to the same plaintext as when the ciphertext was
in its original location ... so long as the entire
block is moved at once.

Although not used above, an attacker could also extend (or
shorten) the plaintext by multiples of BLOCK_SIZE (4 here),
giving even more ways to alter the data while having some
control over the decrypted plaintext ... so long as the last
block is still the last block (the padding block).

In the real world, AES-ECB uses a block size of 16 bytes.
Otherwise, the above weaknesses still apply.

</details><hr/>

## Hints

### Review the Source Code

<details markdown="1"><summary>Review the Source Code</summary><P/>

We're focusing on the second challenge, so the first functions relevant
are in `parapetSetup()` and `parapet()`  Take a moment
to go though those, and brush up on how `ECB` mode encryption works.

#### `parapetSetup()`

`parapetSetup` encrypts the following plaintext using AES in ECB mode:

```c
char message[128] = "Important message to transmit - " FLAG_BANNER "{53Cr37 5745H: " S(POSTERN_FLASH_ADDR) "}";
char message[128] = "Important message to transmit - MAGICLIB{53Cr37 5745H: 0x?????}";
//                   ....-....1....-....2....-....3....-....4....-....5....-....6...
//                   \__ block 1 ___/\__ block 3 ___/\__ block 3 ___/\__ block 4 ___/
```

First it pads using PKCS7, so the 63-byte string is now a multiple
of the AES block size (`0x10`), appending a single `\x01` byte to
the cleartext.

That padded cleartext is then encrypted using AES in ECB mode
to get the ciphertext.  The ciphertext is written to `0x20000`
(the flash address for this stage).

#### `parapet()`

`parapet()` will read the ciphertext from `0x20000`, decrypt it using
AES in ECB mode, and then check if the decrypted message starts
with the string `MAGICLIB`.  If so, it will print out up to 32 bytes
of the string (e.g., via `printf(message)`).

</details>

### What is secret?  What is not?

<details markdown="1"><summary>What is secret?  What is not?</summary><P/>

For stage 2, you know the address of stage 1 data and the address
of stage 2 data.  You also know all but the last few bytes of the
63-byte plaintext message being encrypted, that the encryption is
using AES in ECB mode, and you have the 64-byte ciphertext (by
reading it from the flash address).  Finally, you know that the
`parapet()` function will consider it a success if the plaintext
message it decrypts starts with `"MAGICLIB"`.

The AES key used in the encryption is secret.

</details>

### Problem to be solved

<details markdown="1"><summary>Problem to be solved</summary><P/>

Your job is to modify the ciphertext so that the `parapet()`
function will decrypt it to have a message that starts with
`"MAGICLIB"`, and which contains the flag (the unknown bytes
at the end of the message) within the first 32 bytes.

</details>

## Stage 2 Walkthrough

<details markdown="1"><summary>Wrapper of spoilers</summary><P/>

### Read the stage 2 ciphertext

<details markdown="1"><summary>Read the stage 2 ciphertext</summary><P/>

The ciphertext, when read from flash at `0x20000`, has the
following content in four 16-byte chunks:

```python
b'\x4b\x40\x6f\xe6\xa3\xd4\x32\x1b\x26\x28\xb7\xc6\xff\xf5\xfc\x9f'
b'\x6e\x61\x71\x38\x48\x3e\xf9\x86\x9c\xb8\x4c\x9c\xc0\xd2\x72\xa3'
b'\xde\x90\xe7\xd0\xae\x83\x38\xb0\x7a\xac\x38\x94\x75\x74\x69\x00'
b'\x41\x5d\x39\x41\xde\xd0\xe4\xe3\xad\xc5\x45\x98\x42\xdc\xa5\x8d'
```

Which corresponds to the four plaintext 16-byte chunks:

```python
"Important messag"
"e to transmit - "
"MAGICLIB{53Cr37 "
"5745H: 0x_____}\x01"
```

</details>

### Generate ciphertext that beings with "MAGICLIB"

<details markdown="1"><summary>Generate ciphertext that beings with MAGICLIB</summary><P/>

Since you can re-order the block of AES-ECB encrypted ciphertext,
simply copy the third and fourth ciphertext blocks over the first
and second ciphertext blocks.  Doing so would then decrypt to:

```python
"MAGICLIB{53Cr37 "
"5745H: 0x?????}\x01"
"MAGICLIB{53Cr37 "
"5745H: 0x?????}\x01"
```

The corresponding ciphertext would be:

```python
b'\xde\x90\xe7\xd0\xae\x83\x38\xb0\x7a\xac\x38\x94\x75\x74\x69\x00'
b'\x41\x5d\x39\x41\xde\xd0\xe4\xe3\xad\xc5\x45\x98\x42\xdc\xa5\x8d'
b'\xde\x90\xe7\xd0\xae\x83\x38\xb0\x7a\xac\x38\x94\x75\x74\x69\x00'
b'\x41\x5d\x39\x41\xde\xd0\xe4\xe3\xad\xc5\x45\x98\x42\xdc\xa5\x8d'
```

</details>

### Write the new ciphertext to 0x020000

See prior challenges... this is something you should now
be able to do.

### Solve!

<details markdown="1"><summary>Solve!</summary><P/>

Use the `SOLVE` command to see if `parapet()` was happy
with your changes, and get a pointer to the next stage.

</details><hr/>


</details><hr/>

## FIN



