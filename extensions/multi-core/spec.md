# Draft Multi-Core Extension
**Extension State: Under Development**

**Requires: Base**

**CPUID Bit: CP2.8**

## Overview
Specifies how a multi-core system works.

## Specification
The system consists of 1 or more cores, each of which have independent data, flag, and control registers.

Every core can support a different set of extensions, however,
all extensions and features below that are present on one core,
need to be present on all cores:
- 32 and 64 Bit Address Space: TODO: maybe not? 
- Von Neumann

For all ETC.A extensions,
the word "CPU" in the extension specification refers to a single core,
with the following exceptions: 
- Cache Instructions
- Interrupts: see below
- Von Neumann

Each core has a unique core-ID,
which is a 16 bit unsigned number starting with 0,
and ending with the total number of cores in that system -1.

The core with ID 0 starts at address `0x8000` at reset.
All other cores are halted at reset, but with the program counter initialized with `0x8000`.

See the `MCORE_EN` control register for information on how to start other cores.

All cores share the same address space,
except for the optional core-local memory, which is for this core only.
Every core can have different core-local memory specifications.
Information about the core-local memory can be accessed via the `CLS_SIZE` and `CLS_BEGIN` control registers.

If the core-local memory area contains the start address (`0x8000`), the system is a independent multicore system.
That means non-core-local memory is still shared, but a operating system can only execute on one of the cores, because the cores are (mostly) independent.
This feature might be used in a accelerator like a graphics processor (for the multiple compute cores). When this is a independent multicore system, there is no guarantee to which cores are running and which ones aren't.

When multiple cores write (or one write and multiple read) to overlapping addresses
(which are not in the core-local memory)
at the same time, the data might get corrupted, but the system will not crash.

Interrupts affect:
| Name                     | Affects             |
| ------------------------ | ------------------- |
| External Interrupt       | whole processor (1) |
| System Call              | origin core         |
| Illegal Instruction      | origin core         |
| Memory Alignment Error   | origin core         |
| General Protection Fault | origin core         |
| Divide Error             | origin core         |

1) If the system is independent multicore, this interrupt only affects the current core

Interrupts that affect the originating core get handled by that core only, and all other cores continue execution.

Interrupts that affect the whole processor get handled by any single core in the processor, and all other cores continue execution.
It is recommended for a CPU to distribute the interrupts across cores evenly.

## Added Control Registers 

|   CRN   |    Name     | Description     | priviliged-mode read | priviliged-mode write | user-mode read | user-mode write |
| ------- | ----------- | --------------- | -------------------- | --------------------- | -------------- | --------------- |
| `cr128` | `CORE_ID`   | current core ID | 16-bit value         | undefined             | 16-bit value   | gp fault        |
| `cr129` | `CORE_NUM`  | amount of cores | 16-bit value         | undefined             | 16-bit value   | gp fault        |
| `cr130` | `MCORE_EN`  | (1)             | undefined            | 1-bit value (1)       | undefined      | gp fault        |
| `cr131` | `CLS_BEGIN` | (2)             | data-width value     | undefined             | gp fault       | gp fault        |
| `cr132` | `CLS_SIZE`  | (3)             | data-width value     | undefined             | gp fault       | gp fault        |

1) When writing a `1` into it, all cores in the processor start executing. Writing a `0` causes a gp fault. If however the system is independent multicore, this does nothing.

3) containing the pointer to the beginning of the core-local memory of this core. If a system does not have core-local memory, reading from this is unspecified behaviour.

4) specifying how big the core-local memory (starting at `CLS_BEGIN`) of this core is. If a system does not have core-local memory, this is `0`.

## Code Example
```c 
void os_init(size_t num_cores);

__attribute__ ((noreturn))
void os_start_core(size_t core_id);

__attribute__ ((noreturn))
void _start(void) {
    if (has_ext(EXT_VND_MULTICORE)) {
        // check if the system is independent multicore
        if (getcr(CLS_SIZE) && getcr(CLS_BEGIN) <= 0x8000 && (getcr(CLS_BEGIN)+getcr(CLS_SIZE)) > 0x8000) {
            // independent multicore system
            os_init(1);
            os_start_core(0);
        } else {
            size_t core_id = getcr(CORE_ID);
            if (core_id == 0) {
                os_init(getcr(CORE_NUM));
                setcr(MCORE_EN, 1);
            }
            os_start_core(core_id);
        }
    }
    else {
        os_init(1);
        os_start_core(0);
    }
}
```
