![Firmware Reverse Engineering](https://placehold.co/600x150/111827/E5E7EB?text=Firmware+Reverse+Engineering&font=raleway)

# Firmware Reverse Engineering and Static/Dynamic Analysis

## Table of Contents
1. [About The Project](#about-the-project)
2. [Tools](#tools)
3. [Getting Started](#getting-started)
   - [Implementation Details](#implementation-details)
   - [How to Run](#how-to-run)
4. [Results](#results)
5. [Related Links](#related-links)
6. [Authors](#authors)

## About The Project

This project focuses on reverse engineering and analyzing an unknown embedded firmware image without access to its original source code. The main goal is to reconstruct the internal structure and behavior of the firmware by combining static analysis with dynamic analysis and emulation.

The project addresses the challenge of treating a firmware binary as a black box: identifying its processor architecture, memory layout, program entry point, control flow, operating system, threads, interrupt handlers, and peripheral usage. The analysis combines initial binary inspection, disassembly/decompilation in Ghidra, and runtime validation through QEMU and GDB.

The main outcomes of the project are:

- Identification of the firmware as running on a **32-bit little-endian ARM Cortex-M3** based microcontroller from the **STM32** family.
- Identification of **RIOT-OS** components and startup behavior from strings and reverse-engineered control flow.
- Reconstruction of the **vector table**, reset handler, memory layout, and initialization sequence.
- Recovery of the main high-level execution path from `Reset_Handler` to `main`.
- Identification and analysis of **USART1, USART2, and USART3** interrupt handling.
- Dynamic validation of the static analysis using **QEMU + GDB**, including breakpoint-based tracing of the firmware startup path.

## Tools

The project uses the following software and technologies:

- **Ghidra**: Static analysis, disassembly, decompilation, function analysis, and control-flow reconstruction.
- **Binwalk**: Initial firmware/binary inspection and signature-based analysis.
- **strings**: Extraction of printable strings from the firmware image to obtain early clues about the operating system and runtime behavior.
- **QEMU**: Dynamic analysis and emulation of the target embedded environment.
- **GDB**: Remote debugging, breakpoints, register inspection, and runtime validation.
- **RIOT-OS**: Operating-system environment identified inside the firmware.
- **ARM Cortex-M3**: Target CPU architecture identified during the reverse-engineering process.
- **STM32 microcontroller family**: Target microcontroller family identified from the firmware analysis.

## Getting Started

### Implementation Details

The reverse-engineering process was divided into three main phases.

#### Phase 1 — Initial Binary Inspection and Architecture Identification

The first phase was used to obtain information without assuming the firmware format or target architecture.

1. Run `strings` on the firmware image and inspect the extracted text.
2. Use recognizable strings such as RIOT-OS startup messages and Cortex-M fault-handler text to identify the operating environment and CPU family.
3. Use `binwalk` as an initial structural/signature-based inspection tool.
4. Determine that the input is a raw firmware memory image rather than a conventional ELF/PE executable with standard file headers.
5. Narrow the processor down to **ARM Cortex-M3** and the **STM32** family.

#### Phase 2 — Static Analysis with Ghidra

The firmware was then loaded into Ghidra using the identified architecture and memory mapping.

The main static-analysis tasks were:

- Configure the binary as **32-bit little-endian ARM Cortex-M** code.
- Map the firmware into the Flash region beginning at `0x08000000`.
- Inspect the **vector table** at the beginning of Flash.
- Recover the reset entry point and follow the startup code.
- Reconstruct the memory layout, including Flash, `.data`, `.bss`, stack, and heap regions.
- Analyze important initialization functions such as:
  - `Reset_Handler`
  - `cpu_init`
  - `cortexm_init`
  - `clock_switch_to_pll_hse`
  - `uart_init`
  - `spi_init`
  - `constructors_run`
  - `kernel_init`
  - `thread_create`
  - `scheduler_start`
- Reconstruct the thread startup and context-switching sequence.
- Identify USART interrupt vectors and their shared interrupt handler.

A simplified startup flow reconstructed from the static analysis is:

```text
Reset_Handler
    |
    +--> board_init_pre
    +--> copy .data
    +--> zero .bss
    +--> board_init_post
    +--> cpu_init
    |      +--> cortexm_init
    |      +--> clock_switch_to_pll_hse
    |      +--> rcc_clock_enable
    |      +--> uart_init
    |      +--> spi_init
    |
    +--> constructors_run
    +--> kernel_init
           +--> create idle thread
           +--> create main thread
           +--> scheduler_start
                  +--> enable interrupts
                  +--> SVC #1
                  +--> PendSV
                  +--> context switch
                  +--> main
```

#### Phase 3 — Dynamic Analysis and Validation

The statically reconstructed execution path was validated by running the firmware under QEMU and attaching GDB to the emulated target.

Breakpoints were placed at important addresses, including the reset handler, data/BSS initialization, CPU initialization, UART/SPI initialization, kernel initialization, scheduler startup, and the `main` and `idle` thread entry points.

This allowed the team to compare the observed runtime execution against the control flow reconstructed from Ghidra.

### How to Run

The original project was analyzed using a firmware image together with Ghidra, QEMU, and GDB. The exact build environment for producing the original firmware is not part of the report, so this repository is focused on **analysis and validation of the provided firmware binary**, rather than rebuilding the original firmware from source.

A typical analysis workflow is:

1. **Inspect the firmware with `strings`:**

   ```bash
   strings bin.firmware
   ```

2. **Inspect the firmware with Binwalk:**

   ```bash
   binwalk bin.firmware
   ```

   A zero/empty Binwalk result is not necessarily an error for this firmware because the image is a raw memory dump without standard executable headers.

3. **Open the firmware in Ghidra:**

   - Import the binary as a raw binary.
   - Select the appropriate ARM Cortex-M architecture.
   - Map the Flash region starting at `0x08000000`.
   - Analyze the vector table and startup code.

4. **Start the QEMU emulation environment** for the target machine used in the project.

5. **Attach GDB:**

   ```gdb
   target remote localhost:1234
   set architecture armv7
   set arm fallback-mode thumb
   set arm force-mode thumb
   set disassemble-next-line on
   display/i $pc
   ```

6. **Set breakpoints at the important firmware addresses:**

   ```gdb
   break *0x0800041c
   break *0x08000432
   break *0x0800043c
   break *0x08000bf0
   break *0x08000d64
   break *0x08000b40
   break *0x0800062e
   break *0x08000dd0
   break *0x08001094
   break *0x0800084c
   break *0x0800086a
   break *0x08000884
   break *0x08000324
   break *0x08000328
   break *0x08000844
   break *0x0800082c
   ```

7. **Continue execution** and compare the runtime path with the static control-flow reconstruction.

> **Note:** During emulation, the clock initialization routine entered an infinite loop because the QEMU model did not update the expected RCC readiness flags. The analysis therefore required manually advancing execution past the affected clock-status checks in GDB before continuing with the remaining validation.

## Results

The analysis produced a detailed reconstruction of the firmware's startup and runtime structure.

### Architecture and Operating System

The firmware was identified as a **32-bit little-endian ARM Cortex-M3** image targeting an **STM32** microcontroller. The presence of strings such as RIOT startup and fault-handler messages provided strong early evidence that the firmware was built on **RIOT-OS**.

### Memory Layout

The recovered memory organization included:

| Region | Address Range | Size | Purpose |
| :----- | :------------ | ---: | :------ |
| `.text` / `.rodata` | `0x08000000 - 0x0800239F` | `0x23A0` | Executable code, vector table, read-only data |
| `.data` in Flash | `0x080023A0 - 0x08002417` | `0x78` | Initial values for writable data |
| System / Early Stack | `0x20000000 - 0x200001FF` | `0x200` | Early RAM/stack area |
| `.data` in RAM | `0x20000200 - 0x20000277` | `0x78` | Active initialized global/static data |
| `.bss` | `0x20000278 - 0x20000A97` | `0x820` | Zero-initialized global/static data |
| Heap / Main Stack | From `0x20000A98` onward | — | Dynamic allocation and runtime stack |

The startup routine was also shown to copy initialized `.data` values from Flash to RAM and clear the `.bss` region before entering the higher-level RIOT initialization sequence.

### Control Flow and Threading

The recovered execution sequence shows that the firmware creates two important threads:

- **Idle thread** — priority `15`, with an entry point containing an infinite `WFI` loop for processor idle/sleep behavior.
- **Main thread** — priority `7`, which executes the user `main` function and prints the RIOT startup message.

The scheduler is started using `SVC #1`, which leads to `PendSV` and the first context switch to the higher-priority main thread.

### USART Analysis

The vector table analysis identified three USART interrupt sources:

| Peripheral | IRQ | Vector Offset | Base Address |
| :--------- | --: | ------------: | -----------: |
| USART1 | 37 | `0xD4` | `0x40013800` |
| USART2 | 38 | `0xD8` | `0x40004400` |
| USART3 | 39 | `0xDC` | `0x40004800` |

The three interrupt entry points use software indices `0`, `1`, and `2` and branch to a shared interrupt handler. That handler uses the index to look up a software descriptor table, accesses the corresponding USART status/data registers, handles transmission-complete and line-idle conditions, and may trigger a context switch when a thread is waiting for received data.

### Dynamic Validation

The QEMU/GDB analysis successfully confirmed the major milestones reconstructed through static analysis, including:

- `Reset_Handler`
- `.data` initialization
- `.bss` clearing
- `cpu_init`
- `cortexm_init`
- UART initialization
- SPI initialization
- `constructors_run`
- `kernel_init`
- creation of Idle and Main threads
- `scheduler_start`
- `SVC #1`
- transition to the Main thread

The dynamic analysis also exposed an emulator limitation in RCC clock-status handling, which produced an infinite loop during clock initialization. This became an additional example of why static and dynamic analysis should be used together.

## Related Links

- [Arm Cortex-M3 Processor Datasheet](https://support.arm.com/documentation/102831/latest)
- [RIOT Documentation](https://guide.riot-os.org/)
- [STM32VLDISCOVERY Datasheet](https://www.alldatasheet.com/datasheet-pdf/pdf/472034/STMICROELECTRONICS/STM32VLDISCOVERY.html)
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra)
- [Binwalk](https://github.com/ReFirmLabs/binwalk)
- [QEMU](https://www.qemu.org/)
- [GDB](https://www.gnu.org/software/gdb/)
- [RIOT-OS](https://github.com/RIOT-OS/RIOT)

## Authors

The authors of this project are:
- **Mohammadamin Haghjou** -- Student ID: `403110585`
- **Amiryousef Abdi** -- Student ID: `403106284`
- **Parsa Adlparvar** -- Student ID: `403106302`
