---
status: published
published: true
layout: post
title: Sword of Secrets - now with Python scripting
author: Henry Gabryjelski
date: 2026-03-30 01:31:43 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

# Simplifying via Python

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

### Programmatic access

<details markdown="1"><summary>Programmatic Access</summary><P/>

As I mentioned in the last post, stage 3
essentially ***requires*** programmatic interaction
with the CTF board communications.

Although not complete, I'd now like to share
the helper library I used for this CTF.

Check out the [sos_helper repository](https://github.com/henrygab/sos_helper/).

Each stage will have (at least) one module,
and there's a base module that greatly simplifies
reading, writing, and erasing the flash.

Each stage's module has registered commands you
can run for that stage ... everything from a
simple XOR test (for stage 1), ones that walk
you through a complex solution (e.g., stage 3),
all the way through "autosolve" commands which
just bypass that challenge altogether.

The base library and stages 1-2 are already
published.  I'll publish the remaining stages
as I publish the final steps of the walkthrough.

</details>
<hr/>

## FIN

With the release of this library, you should
have a solid basis for writing your own extensions
to investigate and solve stage 3.

My next post will include the deep walkthrough
of stage 3 ... and it's a doozy!
