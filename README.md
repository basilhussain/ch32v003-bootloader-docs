# WCH CH32V003 Factory Bootloader - The Missing Manual

This document aims to describe everything about the factory bootloader present on WCH's popular [CH32V003](https://www.wch-ic.com/products/CH32V003.html) RISC-V microcontroller, including what it does, how to use it, the details of its serial communications protocol, and its quirks and bugs.

WCH appear to have almost zero information publicly published about the factory bootloader, and what little available information about ISP bootloader capabilities for the CH32V003 there is, can be quite confusing.

## What is It?

The typical method for uploading software onto the CH32V003 microcontroller is via the single-wire debug interface - variably referred to as SWIO, SWD, SWDIO, or SDI - using a programming adapter such as the [WCH-LinkE](https://www.wch-ic.com/products/WCH-Link.html).

However, an alternate means of flash programming is available, using the factory-installed bootloader to perform what is known as In-System Programming (ISP), or alternatively, In-Application Programming (IAP). This method uses the UART serial interface of the microcontroller to communicate with a programming device (not necessarily a computer) using a custom protocol. As ISP does not rely on any proprietary hardware (i.e. WCH-Link adapter) and uses an interface (UART) that may already be exposed, it is very convenient and flexible for the purposes of, for example, firmware upgrades in the field.

The factory CH32V003 bootloader comes pre-programmed into the so-called 'System' or 'BOOT' area of flash memory. This area is a dedicated 1,920 byte section of flash memory starting at an address of `0x1FFFF000`, which is reserved specifically to hold a bootloader. It is completely separate from regular user application flash (a.k.a. 'Code' flash), and does not overlap with or 'steal' any of that flash space.

The bootloader is fairly limited in scope and not intended as a substitute to the comprehensive capabilities of SWIO. It is designed only to erase, write, and verify the user flash area, and also read and write the configuration option bytes.

There are some drawbacks for the bootloader:

* Cannot be used to program a blank, factory-fresh microcontroller chip (due to points below).
* Activation of the bootloader is delegated to the user application; there is no external hardware signal state that can be set to cause the bootloader to be activated on reset (unlike, for example, BOOT0/1 pins on other microcontrollers).
* Does not support reading back flash content.
* Slower programming than SWIO (communication speed is fixed at 115,200 bps).

## What is it Not?

The *factory* bootloader, that comes pre-programmed on brand new chips, is *not* the same as the 'CH32V003_IAP' example found in [WCH's GitHub repository](https://github.com/openwch/ch32v003/tree/main/EVT/EXAM/USART_IAP/CH32V003_IAP). One would think that is the openly-published source code for the factory bootloader, especially given its similar feature set, and the fact it is accompanied by a dedicated Windows application ('WCHMcuIAP'), but it is not.

The factory bootloader has more features, is more complex, and in fact shares some commonality with the USB bootloaders found in other CH32V chips (e.g. CH32V20x, CH32V30x), and a lineage that can be traced back to WCH's older range of 8051-based microcontrollers. It is intended to be used with the standalone WCHISPTool software, or MounRiver Studio IDE.

## How to Use the Bootloader

If one reads the manufacturer documentation, it may seem very straightforward. There exists an Option Byte (OB) flag for `START_MODE` which when set to `1` will cause the microcontroller to start from the bootloader area instead of the user area. Just configure that once, and away you go... right?

Not really. While that configuration flag *does* indeed cause the MCU to execute the bootloader at power-on or reset, the bootloader only remains running under one very specific condition; otherwise it immediately instructs the chip to reset back into user application code. Because this happens imperceptibly quickly, this has the confusing effect of the `START_MODE` OB setting appearing to have no effect.

> **Note:**
> In some early revisions of CH32V003 chip, the `START_MODE` OB flag, no matter what value it's set to, has *no effect*. The chip always acts as though it is set to `1`. This means the chip will always be starting up via the bootloader, and you cannot turn that off. Chips with this behaviour can be identified by the presence of a zero in the 5th-from-last digit of the lot number (last line of marking on the chip). For example, lot number "4123**0**9C34" would be affected.

What doesn't appear to be well documented (or, really, documented *at all*) is under what condition the factory bootloader will continue running. The only hint is in the description of the `MODE` bit of the `FLASH_STATR` register: "After *software* reset, you can switch to the BOOT area" (emphasis added). Accordingly, the CH32V003 bootloader does indeed check for that specific condition: the RCC peripheral's `RSTSCKR` register having the `SFTRSTF` (software reset) flag set to `1`. But, **only** that flag - all others **must** also be `0`.

**This means that the user app has to be in charge of activating the bootloader, and when desired must do so by initiating a specific software reset to cycle back into the bootloader.**

So, after all that is said, what *is* necessary to use the bootloader?

1. **The `START_MODE` flag in the Option Bytes must have been programmed to `1`.** This is a one-time task that must first be done via the SWIO interface, using WCH-LinkE adapter and WCH-LinkUtility (select "Starting from the Boot area" option). Other equivalent open-source software (e.g. wlink, minichlink, OpenOCD) can also perform this task.

2. **At start-up, your user app code should do whatever checks are necessary to see whether the bootloader should be entered.** For example, check the state of a GPIO pin, or check for a certain voltage present on an ADC input, etc. You could also have your user app switch to the bootloader at any other arbitrary point of time, not just start-up.

3. **Have your user app code enter the bootloader by performing the procedure described below.** Only if the previous step deemed it necessary, of course.

What follows is some example C code for the procedure for a user application to enter the bootloader:

```c
// Unlock the ability to write to the MODE flag of STATR register.
FLASH->BOOT_MODEKEYR = 0x45670123; // KEY1
FLASH->BOOT_MODEKEYR = 0xCDEF89AB; // KEY2

// Set to run BOOT flash at next reset (MODE = 1).
FLASH->STATR = 0x4000;

// Clear all reset status flags (RMVF = 1).
RCC->RSTSCKR |= 0x1000000;

// Perform software reset (KEYCODE = 0xBEEF, RESETSYS = 1).
PFIC->CFGR = 0xBEEF0080;
```

Once these pre-requisites are satisfied, and you are able to toggle your device into the bootloader on-demand, you are then in a position to use the relevant software to perform user application firmware re-flashing using the bootloader. Currently available software capable of doing so is:

* WCH's proprietary standalone [WCHISPTool](https://www.wch-ic.com/downloads/WCHISPTool_Setup_exe.html), a.k.a. WCH ISP Studio (Windows)
* WCH's proprietary [MounRiver Studio](http://www.mounriver.com) IDE (Linux, Windows)
* Open-source [wchisp](https://github.com/ch32-rs/wchisp) utility (version 0.3.0+) (Linux, Windows, macOS)

And, of course, now given the details in this document, you could even write your own if you so wish!

## Serial Protocol

The CH32V003 factory bootloader communicates using the UART's default TX and RX pins, GPIOs PD5 and PD6. See the respective chip datasheet pin-outs for which physical pins those GPIOs correspond to for your specific device package.

The bootloader configures the TX and RX pins as follows:

| Direction | GPIO | Configuration                       |
|:--------- | ---- |:----------------------------------- |
| TX        | PD5  | Open-drain output                   |
| RX        | PD6  | Input with internal pull-up enabled |

Note that because TX is configured as open-drain, it can only pull that line to ground, therefore it will be the responsibility of the connected programming device's RX to provide a pull-up to VDD.

The UART is configured for a fixed baud rate of 115,200 bps, 8 data bits, no parity, and 1 stop bit (also termed '8N1'). No other baud rates are supported (unlike some other CH32V bootloaders, which support a 'set baud' command).

The protocol is based upon a sequential command and response system. The programming device will transmit a command (received on chip's RX pin) to instruct the bootloader to perform an action, and when the requested action is complete (or an error has occurred) a response will be transmitted back (from the chip's TX pin).

Once engaged, the bootloader will wait a maximum of 60 seconds to receive a command. If this timeout period expires without having received a command, the bootloader will reset back into the user application. The timer is restarted every time the bootloader has finished receiving a complete and valid command packet.

### Packet Structure

Both commands and responses share a similar packet structure, consisting of a pair of header bytes, followed by a payload, and ended with a simple checksum byte.

| Byte Offset | Description                                                           |
| -----------:|:--------------------------------------------------------------------- |
| 0-1         | Header bytes: `0x57`, `0xAB` for command; `0x55`, `0xAA` for response |
| 2-n         | Payload                                                               |
| n+1         | Checksum                                                              |

If a command packet is received that starts with non-matching header bytes, the two incorrect bytes are ignored, and subsequent bytes will be considered as the start of a new packet. If non-matching bytes continue to be received, this repeats until no more are received (i.e. UART RX buffer is empty).

The checksum is calculated by taking the sum total of all payload bytes modulo 256. That is, add up all the values of each byte of payload, then truncate the total to an 8-bit value. Header bytes are not included in the checksum.

If a command packet is received that has a checksum that does not match the payload, then the command is completely and silently ignored - no response is sent.

### Command Payload Structure

| Byte Offset | Description                      |
| -----------:|:-------------------------------- |
| 0           | Command code (`0xA1` to `0xA8`)  |
| 1           | Length of data                   |
| 2           | Unused, must be `0x00`           |
| 3-n         | Command data (of 'length' bytes) |

(Byte offset shown is relative to the start of the 'payload' section of the containing packet.)

Note that although it may seem that the length potentially occupies 2 bytes and could be a 16-bit little-endian integer, the bootloader in fact only uses the first byte and ignores the second, meaning the length parameter is de facto an 8-bit value.

Also note that although the length byte can represent a value range of 0-255, and there is no inherent limit on the length of serial UART transactions (unlike for USB used in other chips), in practice the maximum value is limited by the command being executed. This is because certain commands either will only process up to a certain amount, or would cause errant behaviour if given too much data; see documentation of individual commands for applicable maximum length.

It's also worth mentioning in passing one quirk of the CH32V003 bootloader code is that when verifying the checksum of a received command packet, it actually sums *only* the command code, length, and data bytes. For some reason, it skips the unused 3rd byte of payload, effectively assuming it contains a value of zero.

### Response Payload Structure

| Byte Offset | Description                                                         |
| -----------:|:------------------------------------------------------------------- |
| 0           | Response code (typically repeat of command code being responded to) |
| 1           | Unknown (appears to have random value)                              |
| 2           | Length of data (maximum 60)                                         |
| 3           | Unused, always `0x00`                                               |
| 4-n         | Response data (of 'length' bytes)                                   |

(Byte offset shown is relative to the start of the 'payload' section of the containing packet.)

Note that although it may seem that the length potentially occupies 2 bytes and could be a 16-bit little-endian integer, the bootloader in fact only ever sets the first byte and defaults the second to zero, meaning the length parameter is de facto an 8-bit value.

It is unknown what the purpose of the 2nd byte of response payload is. It appears to have random value, probably because the CH32V003 bootloader code never pre-initialises or clears the area of RAM that response payloads are constructed in.

### Commands

Eight commands are implemented by the CH32V003 bootloader:

| Code   | Description                                                              |
| ------ |:------------------------------------------------------------------------ |
| `0xA1` | Authenticate and identify                                                |
| `0xA2` | End and reset                                                            |
| `0xA3` | XOR-key calculation                                                      |
| `0xA4` | Erase user flash                                                         |
| `0xA5` | Write user flash                                                         |
| `0xA6` | Verify user flash                                                        |
| `0xA7` | Read configuration (option bytes, plus bootloader version and unique ID) |
| `0xA8` | Erase and write configuration (option bytes)                             |

Other bootloaders on other chips in the WCH RISC-V range have a fuller command set with extra capabilities (e.g. change baud rate), but due to the limited space available on the CH32V003 in the bootloader area (just under 2K), compromises obviously had to be made and only this subset is implemented. There are also other compromises and limitations for some commands, details of which are noted where applicable for each command.

The necessary sequence of commands for programming the flash of a microcontroller is as follows:

1. Authenticate and identify (`0xA1`)
2. Read configuration (`0xA7`)
3. Erase and write new configuration (`0xA8`) - if required
4. XOR-key calculation (`0xA3`)
5. Erase user flash (`0xA4`)
6. Write user flash (`0xA5`) - multiple, in order of lowest to highest address
7. XOR-key calculation (`0xA3`)
8. Verify user flash (`0xA6`) - multiple, in order of lowest to highest address
9. End and reset (`0xA2`)

> **Important:**
> Some commands depend on state that is set by other commands, so it is recommended that you do not deviate from the above sequence. Any such dependencies are noted in the documentation of each command.

Where only a particular operation is desired (e.g. just verifying current contents of flash), a subset of the above sequence can be used. To take the example of flash verification, one may use the following sub-sequence:

1. Authenticate and identify (`0xA1`)
2. Read configuration (`0xA7`)
3. XOR-key calculation (`0xA3`)
4. Verify user flash (`0xA6`) - multiple
5. End and reset (`0xA2`)

As a rule of thumb, you should always start with commands `0xA1` and `0xA7`, then perform the desired operations, then finish with an `0xA2` command. Do not forget to bear in mind the important note above regarding dependencies of some commands upon others.

#### Authenticate & Identify (`0xA1`)

This command serves two purposes: to 'authenticate' the programmer with the bootloader by providing a 'passphrase', and to identify which device type (i.e. chip model) and variant (i.e. package) is being used.

Command payload consists of two bytes containing the expected device type and variant, plus the 'passphrase'.

| Byte Offset | Value            | Notes                                      |
| -----------:| ---------------- |:------------------------------------------ |
| 0           | `0xA1`           | Command                                    |
| 1           | `0x12`           | Length of 18                               |
| 2           | `0x00`           |                                            |
| 3           | (device variant) | Depends on chip package (see below)        |
| 4           | (device type)    | Value of `0x21` for CH32V003               |
| 5-20        | (passphrase)     | 16-byte ASCII string of `MCU ISP & WCH.CN` |

Possible values for CH32V003 device variants (i.e. models/packages) are:

| Model        | Package | Variant Code |
|:------------ | ------- | ------------ |
| CH32V003F4P6 | TSSOP20 | `0x30`       |
| CH32V003F4U6 | QFN20   | `0x31`       |
| CH32V003A4M6 | SOP16   | `0x32`       |
| CH32V003J4M6 | SOP8    | `0x33`       |

The device variant value is part of the `CHIPID` data in the (undocumented) Vendor Bytes area, at an address of `0x1FFFF7C4`; the variant code is the 3rd of the ID's 4 bytes. The value returned in response (see below) is read from this ID.

Note that the CH32V003 bootloader does not actually check that the given expected device type and variant values *match* those of the chip being used. In fact, the given values are ignored completely. The responsibility is therefore on the programming software to verify that the reported device type and variant matches those it is expecting.

The passphrase is comprised of just the specified 16-byte character string in the ASCII character set, without any additions such as a null termination byte.

> **Important:**
> All other commands must be performed *after* this command. One side-effect of this command is to unlock the device's flash controller for writing, so if prior execution of any commands that modify flash contents is attempted, they will fail (possibly by hanging the device).

A successful response payload consists of:

| Byte Offset | Value            | Notes                               |
| -----------:| ---------------- |:----------------------------------- |
| 0           | `0xA1`           |                                     |
| 1           | (random)         |                                     |
| 2           | `0x02`           | Length of 2                         |
| 3           | `0x00`           |                                     |
| 4           | (device variant) | Depends on chip package (see above) |
| 5           | `0x21`           | Indicates CH32V003                  |

An unsuccessful response payload consists of:

| Byte Offset | Value    | Notes                            |
| -----------:| -------- |:-------------------------------- |
| 0           | `0xA1`   |                                  |
| 1           | (random) |                                  |
| 2           | `0x02`   | Length of 2                      |
| 3           | `0x00`   |                                  |
| 4           | `0xF1`   | Indicates incorrect 'passphrase' |
| 5           | (random) |                                  |

#### End & Reset (`0xA2`)

This command signals the end of activity by the programmer, and that the microcontroller should (optionally) be reset.

Command payload consists of just one byte, a boolean flag indicating whether the device should be reset:

| Byte Offset | Value   | Notes                                                          |
| -----------:| ------- |:-------------------------------------------------------------- |
| 0           | `0xA2`  | Command                                                        |
| 1           | `0x01`  | Length of 1                                                    |
| 2           | `0x00`  |                                                                |
| 3           | (reset) | Whether to perform reset (`0x00` = do nothing, `0x01` = reset) |

If a reset is commanded, be aware that the target of the reset - user application or bootloader - can be determined by a previously executed command; some commands will change an internal flag specifying the target. Such target-changing behaviour is noted for the relevant commands. If unchanged by other commands, the default target is user application code.

When a reset is not commanded, the bootloader does nothing and just continues as normal to wait for the next command (essentially making the command a 'no-op').

A successful response payload consists of:

| Byte Offset | Value    | Notes             |
| -----------:| -------- |:----------------- |
| 0           | `0xA2`   |                   |
| 1           | (random) |                   |
| 2           | `0x02`   | Length of 2       |
| 3           | `0x00`   |                   |
| 4           | `0x00`   | Indicates success |
| 5           | `0x00`   |                   |

This command never responds with anything other than a success status.

#### XOR-Key Calculation (`0xA3`)

When data to be written to flash is sent to the bootloader with the flash write command `0xA5`, it is not sent in clear-text, but instead XOR-encoded - or, 'encrypted' if you will - with a key. This command provides a seed to the bootloader's key calculation algorithm, which it executes to form a decryption key with which later received data is decoded.

The rationale for this encryption scheme is debatable, but one thing it does accomplish is prevent a sequence of programming commands captured from the programming of one chip from being 'replayed' to program more chips. This is because the key is partly based upon the chip's embedded unique ID.

Command payload consists only of a number of bytes of seed data:

| Byte Offset | Value    | Notes                                   |
| -----------:| -------- |:--------------------------------------- |
| 0           | `0xA3`   | Command                                 |
| 1           | (length) | Length of seed (minimum 30, maximum 60) |
| 2           | `0x00`   |                                         |
| 3-n         | (seed)   | Random seed data (of 'length' bytes)    |

Seed data can be of variable length, from 30 to 60 bytes. However, the official WCHISPTool always appears to use the maximum 60 byte seed length.

The key calculation algorithm is parameterised by how many bytes of seed data are provided. Two parameters are set according to the length of the seed data; we shall refer to them here as `a` and `b`.

The key calculation algorithm is then, expressed as C-like code, as follows:

```c
a = sizeof(seed) / 5;
b = sizeof(seed) / 7;
xor_key[0] = seed[b * 4] ^ unique_id_checksum;
xor_key[1] = seed[a] ^ unique_id_checksum;
xor_key[2] = seed[b] ^ unique_id_checksum;
xor_key[3] = seed[b * 6] ^ unique_id_checksum;
xor_key[4] = seed[b * 3] ^ unique_id_checksum;
xor_key[5] = seed[a * 3] ^ unique_id_checksum;
xor_key[6] = seed[b * 5] ^ unique_id_checksum;
xor_key[7] = (xor_key[0] + device_variant) & 0xFF;
```

The calculated key is 8 bytes in length. The `unique_id_checksum` is a simple checksum formed from a sum, modulo 256, of all the bytes of the `UNIID1` and `UNIID2` registers of the Electronic Signature (ESIG) peripheral. These unique ID values are returned by the configuration read command `0xA7`. The `device_variant` is the device variant code as described in command `0xA1`.

If one is not concerned with such security measures, then to save effort when calculating the key for ourselves, we can cheat: if the seed sent is entirely `0x00` bytes, then the key that's generated depends only on the unique ID checksum and chip ID variant. This is due to the fact that the result of XOR-ing any value by zero is the value unchanged. Thus, the key calculation then simply becomes:

```c
xor_key[0] = unique_id_checksum
xor_key[1] = unique_id_checksum
xor_key[2] = unique_id_checksum
xor_key[3] = unique_id_checksum
xor_key[4] = unique_id_checksum
xor_key[5] = unique_id_checksum
xor_key[6] = unique_id_checksum
xor_key[7] = (xor_key[0] + device_variant) & 0xFF;
```

> **Important:**
> This command must only be performed at some point *after* a read configuration `0xA7` command, because the bootloader only calculates the necessary unique ID checksum during execution of that command. Also, this command must be executed *prior* to any flash writing commands (`0xA5`).

A successful response payload consists of:

| Byte Offset | Value      | Notes                      |
| -----------:| ---------- |:-------------------------- |
| 0           | `0xA3`     |                            |
| 1           | (random)   |                            |
| 2           | `0x02`     | Length of 2                |
| 3           | `0x00`     |                            |
| 4           | (checksum) | Checksum of calculated key |
| 5           | `0x00`     |                            |

The returned checksum is a simple checksum summing all the bytes of the calculated key, modulo 256. One may use this as a rudimentary way of verifying that the key generated programmer-side matches that calculated by the bootloader.

An unsuccessful response payload consists of:

| Byte Offset | Value    | Notes                    |
| -----------:| -------- |:------------------------ |
| 0           | `0xA3`   |                          |
| 1           | (random) |                          |
| 2           | `0x02`   | Length of 2              |
| 3           | `0x00`   |                          |
| 4           | `0xFE`   | Indicates seed too short |
| 5           | (random) |                          |

> **Caution:**
> This command's error handling is flawed: it is impossible to tell the difference between a successful response that happens to give a key checksum value of `0xFE` and an actual unsuccessful response! Therefore, it is recommended that care is taken to always ensure the seed is of the minimum length, and then instead consider any non-zero checksum value an indicator of success.

No error response is given where the seed is too long (i.e. greater than 60 bytes). In fact, the CH32V003 bootloader's logic will treat it as if it were a length between 56-59 bytes. However, it is still critical that seed length does not exceed 60 bytes, as otherwise the bootloader may suffer erroneous behaviour.

#### Erase User Flash (`0xA4`)

This command erases the *entire* user application flash area. Note that this behaviour differs from the bootloaders of other CH32V microcontrollers (see important note below).

Command is sent with a payload containing the number of 1 kilobyte-size flash sectors to erase:

| Byte Offset | Value     | Notes                                                   |
| -----------:| --------- |:------------------------------------------------------- |
| 0           | `0xA4`    | Command                                                 |
| 1           | `0x04`    | Length of 4                                             |
| 2           | `0x00`    |                                                         |
| 3-6         | (sectors) | Count of 1K-size sectors (32-bit little-endian integer) |

The sector count represents how many bytes are being requested to be erased, counted from the start of user application flash up to an offset equal to the given count multiplied by 1,024. For example, a given count of 12 requests that from offset 0 to 12,287 be erased.

> **Important:**
> The CH32V003 bootloader code for this command completely *ignores* the count parameter and always erases the *entire* user application flash. It is not possible to selectively erase only a portion of user flash with the factory bootloader on the CH32V003. (At a guess, this was presumably done as a compromise to fit the bootloader in to the limited space available on the CH32V003, because the code to properly implement would be too large.)

The official WCHISPTool software only requests erasure of the smallest number of 1K-size flash sectors that will accommodate the size of firmware binary, but always a minimum of at least 8 sectors. The reasoning for this minimum is unclear.

A successful response payload consists of:

| Byte Offset | Value    | Notes             |
| -----------:| -------- |:----------------- |
| 0           | `0xA4`   |                   |
| 1           | (random) |                   |
| 2           | `0x02`   | Length of 2       |
| 3           | `0x00`   |                   |
| 4           | `0x00`   | Indicates success |
| 5           | `0x00`   |                   |

This command never responds with anything other than a success status.

#### Write User Flash (`0xA5`)

This command writes an amount of data to a given location in user application flash.

Command payload consists of a zero-based offset into user application flash and a number of bytes of data to be written at that location:

| Byte Offset | Value    | Notes                                                        |
| -----------:| -------- |:------------------------------------------------------------ |
| 0           | `0xA5`   | Command                                                      |
| 1           | (length) | Length of bytes 3-n                                          |
| 2           | `0x00`   |                                                              |
| 3-6         | (offset) | Offset into flash (32-bit little-endian integer)             |
| 7           | `0x00`   | Unused (value unimportant)                                   |
| 8-n         | (data)   | Data to write, encoded with XOR key (length min. 0, max. 64) |

Note that the offset into flash is not an absolute address but a zero-based relative offset. For example if the data should ultimately reside at `0x08000040`, then the offset value would be `0x00000040`.

The data supplied must be encrypted with the pre-calculated key (as discussed for command `0xA3`) and will then be decrypted by the bootloader before writing to flash. To encrypt each byte, take the command's N-th byte of 'data' and XOR it with key byte from position N modulo 8. Expressed in C-like code, this process would be as follows:

```c
for(int n = 0; n < data_len; n++) {
    data[n] = xor_key[n % 8] ^ data[n];
}
```

> **Important:**
> All flash writing commands must occur only *after* an XOR key command (`0xA3`) has been executed. Otherwise, data will almost certainly be decrypted incorrectly, and garbage written to flash.

The official WCHISPTool software only ever sends up to 56 bytes of data per write command, but the bootloader can handle up to 64 bytes of data without problem. In fact, using the full 64-byte capability where possible will result in more expeditious flash programming, due to the minimisation of write buffering (discussed below).

We presume this use of 56 bytes is due to WCHISPTool's general need to work within the limitations of USB bootloaders on other chips (USB packet sizes are a maximum of 64 bytes; subtracting the payload overhead of 'length', 'offset', etc. gives 56 remaining from 64). But it's not clear why it maintains this restriction for serial UART transport on the CH32V003, where this transport has no such inherent limitation.

Due to the above behaviour not correlating with the CH32V003 bootloader's need to only want to write full flash pages (64 bytes), it employs a buffering mechanism to try and ensure it is only writing full pages wherever possible. Thus beware that the data of any given write command providing less than 64 bytes of data may not necessarily be written to flash straight away!

Received data will be buffered under the following circumstances:

* Less than 64 bytes of data are received. The data is buffered until more is received.
* The given 'offset' parameter and size of data results in data spilling over a page boundary. The overflow that would be on the next page is buffered.

The buffer is maintained and no writing to flash occurs until either of two conditions is met:

* The amount of buffered data becomes at least 64 bytes.
* A write command with data of zero size is received.

The latter is used to handle the situation where there is no more firmware binary data to be sent, but there is still some buffered from the last write command. A further single finalisation write command with incremented offset and zero-length of data can be issued to get that buffered data written.

> **Important:**
> It is recommended that one such zero-length-data final write command is *always* issued regardless of circumstance.

> **Important:**
> Due to the convoluted logic of the buffering mechanism of the write command, it is also recommended that firmware binary writing commands are always issued in contiguous order of lowest to highest 'offset'.

> **Important:**
> This command changes the internal reset target flag to be user application code. Unless subsequently changed by another command that modifies this flag, issuing an `0xA2` command after this one will reset back into user application code.

A successful response payload consists of:

| Byte Offset | Value    | Notes             |
| -----------:| -------- |:----------------- |
| 0           | `0xA5`   |                   |
| 1           | (random) |                   |
| 2           | `0x02`   | Length of 2       |
| 3           | `0x00`   |                   |
| 4           | `0x00`   | Indicates success |
| 5           | `0x00`   |                   |

This command never responds with anything other than a success status.

#### Verify User Flash (`0xA6`)

This command verifies that the data at a specified location in user application flash matches the given data.

Command payload consists of a zero-based offset into user application flash and a number of bytes of data to be verified against:

| Byte Offset | Value    | Notes                                                     |
| -----------:| -------- |:--------------------------------------------------------- |
| 0           | `0xA5`   | Command                                                   |
| 1           | (length) | Length of bytes 3-n                                       |
| 2           | `0x00`   |                                                           |
| 3-6         | (offset) | Offset into flash (32-bit little-endian integer)          |
| 7           | `0x00`   | Unknown (value unimportant)                               |
| 8-n         | (data)   | Data to be matched, encoded with XOR key (length max. 64) |

Note that the offset into flash is not an absolute address but a zero-based relative offset. For example if the data should ultimately reside at `0x08000040`, then the offset value would be `0x00000040`. Also note that the offset must be a multiple of 8.

The data supplied must be encrypted with the pre-calculated XOR key (see command `0xA3`) and will be decrypted by the bootloader before comparison. For encryption method, see the documentation for the flash write command `0xA5`. Note also that the size of data must be a multiple of 8.

> **Important:**
> This command maintains a temporary status flag recording whether verification has failed due to data mismatch. If one prior verify command has failed in this way, then all subsequent verification commands will immediately skip any data comparison and return with a failure result. Once set, this flag is cleared only by issuing an erasure command `0xA4` (or by resetting the microcontroller).

A successful response payload consists of:

| Byte Offset | Value    | Notes             |
| -----------:| -------- |:----------------- |
| 0           | `0xA6`   |                   |
| 1           | (random) |                   |
| 2           | `0x02`   | Length of 2       |
| 3           | `0x00`   |                   |
| 4           | `0x00`   | Indicates success |
| 5           | `0x00`   |                   |

An unsuccessful response payload due to verification mismatch consists of:

| Byte Offset | Value    | Notes                       |
| -----------:| -------- |:--------------------------- |
| 0           | `0xA6`   |                             |
| 1           | (random) |                             |
| 2           | `0x02`   | Length of 2                 |
| 3           | `0x00`   |                             |
| 4           | `0xF5`   | Indicates non-matching data |
| 5           | `0x00`   |                             |

An unsuccessful response payload due to other failure consists of:

| Byte Offset | Value    | Notes       |
| -----------:| -------- |:----------- |
| 0           | `0xA6`   |             |
| 1           | (random) |             |
| 2           | `0x02`   | Length of 2 |
| 3           | `0x00`   |             |
| 4           | `0xFE`   | See below   |
| 5           | `0x00`   |             |

Failure result of `0xFE` can be issued for a number of different failures:

* Offset not a multiple of 8.
* Offset was ≥ `0x1FFFF000` (i.e. actually an absolute address in bootloader area). Not sure why it checks this.
* Length of data being verified against not a multiple of 8.
* A prior verification command failed due to mismatch.

#### Read Configuration (`0xA7`)

This command will read and return the device's option bytes, plus other information such as bootloader version and the chip's unique ID.

Command payload consists of a bitmask specifying which configuration data elements to read:

| Byte Offset | Value     | Notes                                     |
| -----------:| --------- |:----------------------------------------- |
| 0           | `0xA7`    | Command                                   |
| 1           | `0x02`    | Length of 2                               |
| 2           | `0x00`    |                                           |
| 3           | (bitmask) | Bit flags indicating which data to return |
| 4           | `0x00`    |                                           |

The individual bits in the mask value have the following representation:

| Bit Position | Meaning                    |
| ------------:| -------------------------- |
| 0            | USER & RDPR                |
| 1            | DATA0 & DATA1              |
| 2            | WRPR                       |
| 3            | BTVER (bootloader version) |
| 4            | UID (chip unique ID)       |

For example, USER & RDPR + DATA0 & DATA1 + WRPR = `0x07`; all = `0x1F`.

> **Important:**
> The CH32V003 bootloader ignores the bitmask value and always returns *all* information, regardless of what has or has not been requested. (Again, this behaviour is probably a compromise made due to limited space for bootloader code.)

A successful response payload consists of:

| Byte Offset | Value           | Notes                                         |
| -----------:| --------------- |:--------------------------------------------- |
| 0           | `0xA7`          |                                               |
| 1           | (random)        |                                               |
| 2           | `0x1A`          | Length of 26                                  |
| 3           | `0x00`          |                                               |
| 4           | (bitmask)       | Bitmask from request, masked with `0x1F`      |
| 5           | `0x00`          |                                               |
| 6           | (rdpr)          | Option byte `RDPR`                            |
| 7           | (nrdpr)         | Option byte `nRDPR`                           |
| 8           | (user)          | Option byte `USER`                            |
| 9           | (nuser)         | Option byte `nUSER`                           |
| 10          | (data0)         | Option byte `DATA0`                           |
| 11          | (ndata0)        | Option byte `nDATA0`                          |
| 12          | (data1)         | Option byte `DATA1`                           |
| 13          | (ndata1)        | Option byte `nDATA1`                          |
| 14          | (wrpr0)         | Option byte `WRPR0`                           |
| 15          | (wrpr1)         | Option byte `WRPR1`                           |
| 16          | (wrpr2)         | Option byte `WRPR2`                           |
| 17          | (wrpr3)         | Option byte `WRPR3`                           |
| 18-19       | (major version) | Bootloader major version (BCD format)         |
| 20-21       | (minor version) | Bootloader minor version (BCD format)         |
| 22-25       | (uniid1)        | First part of chip unique ID (ESIG `UNIID1`)  |
| 26-29       | (uniid2)        | Second part of chip unique ID (ESIG `UNIID2`) |

Note that the option byte values do *not* include any of the inverse `nWRPRx` bytes. Also note that the CH32V003 does not use `WRPR2` or `WRPR3`, so the value of these bytes has no significance.

The bootloader version is comprised of two values: a major and minor version number. Each byte represents a numeric digit of the numbers in unpacked [BCD](https://en.wikipedia.org/wiki/Binary-coded_decimal) format. For example, bootloader version "02.30" is encoded as `0x00`, `0x02` for major version ("02") and `0x03`, `0x00` for minor version ("30").

The electronic signature (ESIG) `UNIID3` register value is not present because it is unused by the CH32V003 (its unique ID is only 64-bit, as opposed to the 96-bit ID of other chips) and so does not contain any useful data.

This command does not respond with any kind of failure status.

#### Erase and Write Configuration (`0xA8`)

This command will erase the option bytes section of flash and then write the provided new values.

Command payload consists of a bitmask specifying which option byte elements are to be written, plus the data to be written:

| Byte Offset | Value     | Notes                                    |
| -----------:| --------- |:---------------------------------------- |
| 0           | `0xA7`    | Command                                  |
| 1           | `0x0E`    | Length of 14                             |
| 2           | `0x00`    |                                          |
| 3           | (bitmask) | Bit flags indicating which data to write |
| 4           | `0x00`    |                                          |
| 5           | (rdpr)    | Value for option byte `RDPR`             |
| 6           | (nrdpr)   | Value for option byte `nRDPR`            |
| 7           | (user)    | Value for option byte `USER`             |
| 8           | (nuser)   | Value for option byte `nUSER`            |
| 9           | (data0)   | Value for option byte `DATA0`            |
| 10          | (ndata0)  | Value for option byte `nDATA0`           |
| 11          | (data1)   | Value for option byte `DATA1`            |
| 12          | (ndata1)  | Value for option byte `nDATA1`           |
| 13          | (wrpr0)   | Value for option byte `WRPR0`            |
| 14          | (wrpr1)   | Value for option byte `WRPR1`            |
| 15          | (wrpr2)   | Value for option byte `WRPR2`            |
| 16          | (wrpr3)   | Value for option byte `WRPR3`            |

For bitmask values, see the read configuration `0xA7` command.

> **Important:**
> Writing of option bytes will *only* take place if bit flags for all of USER & RDPR, DATA0 & DATA1, and WRPR are set (i.e. bitmask ≥ `0x07`). Any other flags are ignored (e.g. BTVER, UID), because it is neither possible to supply data for or write that config information.

When the option bytes are being written, the supplied inverse bytes (i.e. 'n' values) are ignored and instead calculated by the bootloader from the non-inverse values. For example, the given `nRDPR` value is ignored, and instead the value of `RDPR` is inverted and used for `nRDPR`.

Also, similarly, because no inverted values for `nWRPRx` are supplied, they are also calculated in the same way from `WRPRx` values. Note also that the CH32V003 does not use `WRPR2` or `WRPR3`, so although the writing of those values will take effect, the values written are irrelevant.

> **Warning:**
> Be aware that when the `RDPR` byte is changed from any other value to `0xA5` (i.e. read-protection is changed to disabled from a formerly enabled state), this will cause the device to automatically perform an erasure of the *entire* user application flash!

> **Important:**
> This command changes the internal reset target flag to be the bootloader. Unless subsequently changed by another command that modifies this flag, issuing an `0xA2` command after this one will reset back into the bootloader rather than the default of user application code.

A successful response payload consists of:

| Byte Offset | Value    | Notes             |
| -----------:| -------- |:----------------- |
| 0           | `0xA8`   |                   |
| 1           | (random) |                   |
| 2           | `0x02`   | Length of 2       |
| 3           | `0x00`   |                   |
| 4           | `0x00`   | Indicates success |
| 5           | `0x00`   |                   |

An unsuccessful response payload consists of:

| Byte Offset | Value    | Notes                                                    |
| -----------:| -------- |:-------------------------------------------------------- |
| 0           | `0xA8`   |                                                          |
| 1           | (random) |                                                          |
| 2           | `0x02`   | Length of 2                                              |
| 3           | `0x00`   |                                                          |
| 4           | `0xFE`   | Indicates flags for option bytes weren't part of bitmask |
| 5           | `0x00`   |                                                          |

#### Unknown Command

When an unknown command code is received - one outside the range `0xA1`-`0xA8` - the response payload will be as follows:

| Byte Offset | Value      | Notes                                      |
| -----------:| ---------- |:------------------------------------------ |
| 0           | (last cmd) | Whatever last *known* received command was |
| 1           | (random)   |                                            |
| 2           | `0x02`     | Length of 2                                |
| 3           | `0x00`     |                                            |
| 4           | `0xFE`     | Indicates unknown command                  |
| 5           | (random)   |                                            |

## Additional Resources

These resources you may find useful to provide additional information or utility:

* [WCH CH32V003 Datasheet](https://www.wch-ic.com/downloads/CH32V003DS0_PDF.html)
* [WCH CH32V003 Reference Manual](https://www.wch-ic.com/downloads/CH32V003RM_PDF.html)
* [WCH CH32V003 GitHub repository](https://github.com/openwch/ch32v003)
* [Ghidra patch for WCH 'XW' RISC-V instruction extension](https://github.com/NationalSecurityAgency/ghidra/pull/6390)
* [WCHISPTool](https://www.wch-ic.com/downloads/WCHISPTool_Setup_exe.html)
* [wchisp](https://github.com/ch32-rs/wchisp)

## Acknowledgements

Thanks to [Georg Icking-Konert](https://github.com/gicking) for proofreading and feedback.

## Licence

Copyright © 2024 Basil Hussain.

This work is licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International](https://creativecommons.org/licenses/by-nc-sa/4.0/) licence.
