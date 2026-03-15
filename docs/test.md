---
status: published
published: true
layout: default
title: Testing Jekyll Formatting of Markdown Wit
---

Good news!  You don't need a physical board to follow
along ... simply load the [virtual challenge webite](https://swordofsecrets.com/mini-rv32ima.html).

## GHPages / Kramdown Fix: `markdown="1"`

<details markdown="1"><summary>GHPages Fix: markdown="1" attribute</summary>

Hyperlink: [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
More text.

Embedded code: `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands.  More text.

A simple example of bulleted list
[gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

And a numbered list:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

And of course a multi-line code block:

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

</details><hr/>


## Default Section Format

<details><summary>Default Section Format</summary><P/>

Hyperlink: [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
More text.

Embedded code: `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands.  More text.

A simple example of bulleted list
[gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

And a numbered list:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

And of course a multi-line code block:

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

</details><hr/>


## Everything in a P tag

<details><summary>Everything in a P tag</summary><P>

Hyperlink: [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
More text.

Embedded code: `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands.  More text.

A simple example of bulleted list
[gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

And a numbered list:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

And of course a multi-line code block:

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

</P></details><hr/>

## Everything in a DIV tag

<details><summary>Everything in a DIV tag</summary><div>

Hyperlink: [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
More text.

Embedded code: `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands.  More text.

A simple example of bulleted list
[gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

And a numbered list:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

And of course a multi-line code block:

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

</div></details><hr/>

## Nothing wrapping the contents

<details><summary>Nothing wrapping the contents</summary>

Hyperlink: [datasheet](https://www.mouser.co.il/datasheet/2/949/w25q128jv_revf_03272018_plus-1489608.pdf).
More text.

Embedded code: `JEDEC ID (9Fh)`,
`READ DATA 03h`, `WRITE ENABLE 06h`, `SECTOR ERASE 4K (20h)`,
and `PAGE PROGRAM 02h` commands.  More text.

A simple example of bulleted list
[gets the flash's ID](https://github.com/gili-yankovitch/SwordOfSecrets/blob/15df21724a45f923e5b52ed7315c40b80923d41b/src/spiflash.c#L70-L85):

* `SPI_Begin_8()` to setup the SPI peripheral
* `CSASSERT()` to drive the `/CS` line low
* `SPI_transfer_8(0x9F)` to send the command `0x9F`
* `uint8_t x = SPI_transfer_8(0)` three times to get the device's response
* `CSRELEASE()` to drive the `/CS` line high
* `SPI_End()` to release the SPI peripheral

And a numbered list:

1. Data sent to the flash chip, such as commands and command parameters.
2. So-called "dummy" bytes (ignored by the flash chip, but have to be sent).
3. Data sent from the flash chip, such as response to the command.

And of course a multi-line code block:

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

</details><hr/>





## FIN

That should give you some options for exploring
the flash memory.   See if you can figure out
the first challenge, before reading about it
in my the next post.  And remember, the first
stage can be done entirely via the [virtual challenge webite](https://swordofsecrets.com/mini-rv32ima.html).

