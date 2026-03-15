---
status: published
published: true
layout: post
title: Interacting with the W25Q128 flash memory chip
author: Henry Gabryjelski
date: 2026-03-15 07:03:04 UTC
categories: [Sword of Secrets]
tags: [CTF, sword_of_secrets]
comments: []
---

Good news!  You don't need a physical board to follow
along ... simply load the [virtual challenge webite](https://swordofsecrets.com/mini-rv32ima.html).

## Get the W25Q128 flash memory chip's datasheet

<details><summary>Get the W25Q128 flash memory chip's datasheet</summary><P/>

Download the [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
Take a moment to look over the datasheet you downloaded,
especially the table listing the supported commands, and
what bytes you have to send after each command byte.

This post will walk through the `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands, so if you've never dealt
with SPI devices, this walkthrough will give you most of
what you need to play through these challenges.

</details>

## Discover the commands available

<details><summary>Discover the commands available</summary></P>

First step is figuring out how you can interact with
the Sword through its serial port.

Luckily, the source code is available.  One option
is to start at the function that [reads from the
serial port](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/main.c#L337-L366)
and trace into the function(s) it calls.

Can you find the strings that you can send via the
serial port to cause the Sword to do something?
(e.g., the list of commands)

</details>

## SPI Basics

<details><summary>SPI Basics</summary><P/>

Now that you've discovered all the commands, next is
to understand how to put them together to do something
useful.

Once again, I recommend looking at how the firmware
interacts with the SPI flash device.  A simple example
is how the firmware [gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

As you might notice, the SPI chip only "listens" to the
data line when the `/CS` line is `ASSERT`'d.  Thus, `DATA`
commands are only responded to by the flash chip when
they are between an `ASSERT` and `RELEASE`.

After the `/CS` line is asserted, the first byte
transferred defines the command, and thus also
defines how many additional bytes need to be sent,
and how many bytes of data are available from the
device.  Read the datasheet for details for each
command of interest.

If you thought it was odd to transfer zero bytes
when wanting to receive data, you'd be right --
it is odd. Although technically SPI allows data
to be sent and received simultaneously, the SPI
flash chips were designed so data only flows one 
direction at a time.  Because SPI only allows
the device to respond with data when the host is
sending data, the firmware sends bytes of `0`
when it gets data from the flash device.

</details>

## Using SPI via the Sword of Secrets

<details><summary>Using SPI via the Sword of Secrets</summary><P/>

The above command sequence will map substantially
one-to-one with the serial commands you discovered
can be sent via the serial port, e.g.,

* `BEGIN` to setup the SPI peripheral
* `ASSERT` to drive the `/CS` line low
* `DATA 9F` to send the command `0x9F`
* `DATA 0 0 0` to read three bytes
* `RELEASE` to drive the `/CS` line high
* `END` to release the SPI peripheral

A note about the `DATA` command.  It is ***not*** necessary
to send all data bytes in a single `DATA` line.  In fact,
the device may lock up or silently ignore some of the data,
if you send too much in a single line.

Each hex byte that follows `DATA` may be one of three types:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

I've found it helps keep things orderly if the first `DATA`
command I send includes the command and any parameter bytes
required (and padding/dummy bytes, for those command that
need them).   If expecting data in response, I send separate
`DATA` commands, reading at most `0x10` bytes per line.

</details>

## Example 1: JEDEC ID (9Fh)

<details><summary>Example 1: JEDEC ID (9Fh)</summary><P/>

This is a simple one-byte command, which expects three bytes of data.
The expected three-byte response (per datasheet) is:

* `EFh` - Manufacturer ID for Winbond Serial Flash
* `40h 18h` - Device ID for W25Q128JV-IQ and W25Q128JV-JQ
* `70h 18h` - Device ID for W25Q128JV-IM and W25Q128JV-JM

Here's a console log of retrieving this data from the
website version of the SwordOfSecrets:

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

## Example 2a: READ DATA (03h) - 16 bytes from `0x123456`

<details><summary>Example 2a: READ DATA (03h) - 24 bytes from `0x123456`</summary><P/>

This command requires a 24-bit address indicating
where the read should begin:

* `A23-A16` -- The most significant byte of the address
* `A15-A8` -- ...
* `A7-A0` -- The least significant byte of the address

This is essentially big-endian byte ordering.  Note
that erased (never-written) flash will have the value
`FFh` for each byte.

Here's a log showing one way to read the first 16 bytes
from address `0x123456`.  I'll use my preferred format
where the first `DATA` is only the command, followed by
a separate `DATA` commands for (up to) 16 bytes each.

```
>> BEGIN

>> ASSERT

>> DATA 03 12 34 56
00 00 00 00 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 

>> DATA 0 0 0 0 0 0 0 0
ff ff ff ff ff ff ff ff

>> RELEASE

>> END
```

Yes, the data is all `0xFF`, which just means it's
blank (not written) area of the flash.


</details>


## Example 2b: READ DATA (03h) - 16 bytes from `0x010000`

<details><summary>Example 2b: READ DATA (03h) - 16 bytes from `0x010000`</summary><P/>

Substantially the same as the Example 2a, only
now you'll read 16 bytes from an area that has
been written to.

```
>> BEGIN

>> ASSERT

>> DATA 03 01 00 00
00 00 00 00 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
00 00 00 00 0e 05 13 07 36 0f 37 69 22 27 3f 65 

>> RELEASE

>> END
```

</details>

## Example 2c: READ DATA (03h) - 16 bytes from `0x010000`

<details><summary>Example 2c: READ DATA (03h) - 16 bytes from `0x010000`</summary><P/>

Identical to Example 2b, except that it requests data
from the device in four-byte increments, instead of 16.
This just shows that `DATA` line can be split, so long
as it's still between the `ASSERT` and `RELEASE`.

```
>> BEGIN

>> ASSERT

>> DATA 03 01 00 00
00 00 00 00 

>> DATA 0 0 0 0
00 00 00 00 

>> DATA 0 0 0 0
0e 05 13 07 

>> DATA 0 0 0 0
36 0f 37 69 

>> DATA 0 0 0 0
22 27 3f 65 

>> RELEASE

>> END
```

</details>

## Example 2d: READ DATA (03h) - 16 bytes from `0x010000`

<details><summary>Example 2d: READ DATA (03h) - 16 bytes from `0x010000`</summary><P/>

While there's no requirement to split the `DATA` line
(other than input and output line length limits),
I feel it's harder to read the results.

Let's use a single `DATA` line.  Note that, if choosing
to use this form, the response data includes four bytes
of `00h` that correspond to the four command bytes.
Thus, the device is still reporting the same 16 bytes of data.

This example also highlights that input data bytes do
NOT require a leading zero.  Although I prefer to always
send two hex digits for any bytes I send to the device,
that's just a personal preference.

```
>> BEGIN

>> ASSERT

>> DATA 3 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
00 00 00 00 00 00 00 00 0e 05 13 07 36 0f 37 69 22 27 3f 65 

>> RELEASE

>> END
```

</details>

## Example 3: WRITE ENABLE (06h)

<details><summary>Example 3: WRITE ENABLE (06h)</summary><P/>

This is a single-byte command, with no response.
It must immediately precede commands that modify
the device, such as erase, write, etc.  However,
the command does nothing in and of itself.

See next two commands (erase 4k and page program)
for examples.

</details>


## Example 4: SECTOR ERASE 4k (20h) for address 010000h

<details><summary>Example 4: SECTOR ERASE 4k (20h) for address 010000h</summary><P/>

This command requires a 24-bit address indicating
where the erase should begin:

* `A23-A16` -- The most significant byte of the address
* `A15-A8` -- ...
* `A7-A0` -- The least significant byte of the address

Since `4k` == `1000h`, the address bits must reflect
an address that is a multiple of `1000h`.  Said another
way, `assert((address & 0x0FFFu) == 0)`.  If any
of those low 12 bits are set to one, the datasheet does
not clearly define the device behavior.  It may reject
the command, or it may internally ignore those least
significant 12 bits.  It is not recommended to rely
upon such undocumented behavior.

Also recall the requirement to have the `WRITE ENABLE 06h`
command immediately preceding, but within the same `BEGIN`/`END`.

NOTE: Neither the 32k BLOCK ERASE nor 64k BLOCK ERASE
      seem to be supported on the virtual challenge website.

```
>> BEGIN

>> ASSERT

>> DATA 06
00 

>> RELEASE

>> ASSERT

>> DATA 20 01 00 00
00 00 00 00 

>> RELEASE

>> END
```

To verify this worked, read the data from 0x10000 again
(any of Examples 2b through 2d).  The data should have
changed into all-`0xFF` bytes.

</details>

## Example 5: Page program (02h) for address 010000h

<details><summary>Example 5: Page program (02h) for address 010000h</summary><P/>

Now that you've erased the sector at `0x010000`, it's
ready to be written.  Programming new data also
requires the immediately preceding command to be
WRITE ENABLE (06h).  Here, just write a distinguished
pattern to `0x010000`.

```
>> BEGIN

>> ASSERT

>> DATA 03 01 00 00
00 00 00 00 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ff ee dd cc bb aa 99 88 77 66 55 44 33 22 11 18 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
18 11 22 33 44 55 66 77 88 99 aa bb cc dd ee ff 

>> DATA 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff ff 

>> RELEASE

>> END
```

To verify this worked, read the data from 0x10000 again
(any of Examples 2b through 2d).  The data should have
changed into all-`0xFF` bytes.

Finally, to reset the data on the flash chip to the
original challenge data, send the `RESET` command
to the Sword of Secrets.

</details>

## FIN

That should give you some options for exploring
the flash memory.   See if you can figure out
the first challenge, before reading about it
in my the next post.  And remember, the first
stage can be done entirely via the [virtual challenge webite](https://swordofsecrets.com/mini-rv32ima.html).

