# CH32V00x Bootloader Differences

These are some brief notes about differences in behaviour that have been so far noted for the CH32V00x series of chips - e.g. CH32V002, CH32V006, etc.

> [!CAUTION]
> **This is currently a work-in-progress, and some information may be incomplete or incorrect.**

## 'BOOT' Area Size

They all feature a larger bootloader flash area of 3,328 bytes.

It appears that the entire range shares identical bootloader code (MD5 hash of `dce6e3922b225073b53826ab2217c32c`).

## UART Connection

Only for CH32V005D6U6 (QFN12, variant `0x53`) and CH32V002D4U6 (QFN12, variant `0x23`) chips, uses UART alternate pin remapping of PD0 for TX and PD1 for RX. Otherwise, same default pins of PD5 for TX and PD6 for RX.

## Authenticate & Identify Command (`0xA1`)

Device variant values that may be given and returned are:

| Model        | Variant Code |
| ------------ | ------------ |
| CH32V002A4M6 | `0x22`       |
| CH32V002D4U6 | `0x23`       |
| CH32V002F4P6 | `0x20`       |
| CH32V002F4U6 | `0x21`       |
| CH32V002J4M6 | `0x24`       |
| CH32V004F6P1 | `0x40`       |
| CH32V004F6U1 | `0x41`       |
| CH32V005D6U6 | `0x53`       |
| CH32V005E6R6 | `0x50`       |
| CH32V005F6P6 | `0x52`       |
| CH32V005F6U6 | `0x51`       |
| CH32V006E8R6 | `0x61`       |
| CH32V006F4U6 | `0x64`       |
| CH32V006F8P6 | `0x63`       |
| CH32V006F8U6 | `0x62`       |
| CH32V006K8U6 | `0x60`       |
| CH32V007E8R6 | `0x71`       |
| CH32V007K8U6 | `0x72`       |
| CH32M007E8R6 | `0x73`       |
| CH32M007E8U6 | `0x74`       |
| CH32M007G8R6 | `0x70`       |

Device type for all models is still `0x21`, same as for CH32V003.

Still ignores the given device variant and type and does not verify them against the device in use, despite extra bootloader flash space available to implement verification.

## XOR-Key Calculation Command (`0xA3`)

The code uses GCC's libgcc integer division routine (`__udivsi3`) within XOR key calculation rather than previous custom branching logic, but this wastes 80 bytes of bootloader flash, because accompanying code for modulus calculations (`__umodsi3`, `__modsi3`) is also linked in but never used.

## Erase User Flash Command (`0xA4`)

Still ignores the sector count parameter to the command (so always erases entire flash), even though plenty of extra bootloader flash space available to implement it fully.

## Read Configuration Command (`0xA7`)

Reported bootloader version is "02.40", versus "02.30" for CH32V003.

Still ignores bitmask specifying values to return and always returns all values, even though extra flash space exists to implement full behaviour.

## Erase & Write Configuration Command (`0xA8`)

Behaves slightly differently. When the given new `RDPR` value is equal to `0xA5` - i.e. read-out protection is being turned off - regardless of any other values, then:

* Reset target is changed to the bootloader.
* If the currently-stored option byte `RDPR` value is *not* `0xA5`, erases the entire option bytes and writes the default `RDPR` and `nRDPR` (`0xA5`, `0x5A`) values, and no other values. Then immediately returns a successful response to the command.

The reason for this change is unclear.

Internally, instead of a dedicated function to write to option bytes, copies and munges (i.e. creating inverse values, etc.) the new values to the flash write buffer in RAM, then uses the general flash-writing function to write the option bytes area. This may mean it's unsafe to issue an `0xA8` command in-between a sequence of `0xA5` flash write commands.

Command still also only works if bitmask for all values is given, even though flash space exists to implement better behaviour.
