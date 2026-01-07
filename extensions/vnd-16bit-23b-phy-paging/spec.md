# Vendor 23-bit physical address extension for 16-bit paging
**Extension State: Stable, as long as CP1.13 doesn't change**

**Requires: CP1.2, [CP1.13: WIP 16-bit paging](https://github.com/ETC-A/etca-spec/tree/initial-paging-proposals/extensions/16-bit-paging)**

**Incompatible with: CP2.2: Priviliged Mode**

**CPUID Bit: CP2.12**

Note that this extension itself is "stable", but the underlying 16-bit paging extension is currently under development!

NOT COMPATIBLE WITH CP2.2!

## Overview
(Ab)uses the `U` bit in page table entries, designed to mark a page as user-mode page, as 13th (counting from 0) bit of the physical frame number,
extending physical addresses to 23-bits, instead of 22-bits.

## Specification
The `U` bit (the 2nd bit, counting from 0) in page table entries is used as 13th bit (counting from 0) of the physical frame number,
which extends the physical address space to 23-bits.
