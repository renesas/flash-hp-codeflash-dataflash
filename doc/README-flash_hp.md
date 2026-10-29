# Flash HP Software Building Block

## Overview

This Software Building Block (SBB) demonstrates basic use of the Renesas Flash HP (`r_flash_hp`) driver for code-flash and data-flash operations. The reference application initializes Flash HP and provides a serial-terminal menu for executing predefined erase, blank-check, write, read-back, and verification operations.

This document covers only:

- FPB-RA4E2
- FPB-RA6E2
- FPB-RA8E1

All three boards are explicitly listed as supported in the supplied `readme.txt`.

## What use case does this SBB demonstrate?

The SBB demonstrates common embedded nonvolatile-memory operations through the FSP Flash HP driver.

The reference application provides three terminal options:

1. Code flash operations.
2. Data flash operations.
3. Exit.

For code flash, the application erases code-flash block 8, blank-checks the configured code-flash block size, writes a predefined data pattern, reads the programmed bytes back, and verifies that the read data matches the write buffer.

For data flash, it performs the corresponding erase, blank check, write, read-back, and verification sequence on data-flash block 1.

When the target reports support through `BSP_FEATURE_FLASH_SUPPORTS_ACCESS_WINDOW`, the code-flash demonstration additionally exercises an access-window sequence.

## Hardware requirements

- 1 x FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1
- 1 x appropriate USB cable for programming and debugging

Connect the board USB debug port to the host PC.

### FPB-RA4E2

No additional FPB-RA4E2-specific hardware connection is stated in the supplied `readme.txt`.

### FPB-RA6E2

No additional FPB-RA6E2-specific hardware connection is stated in the supplied `readme.txt`.

### FPB-RA8E1

Open jumpers **J6** and **J9** to use the on-board debug functionality.

## What does the SBB core code do?

The reusable SBB layer is implemented by:

- `flash_hp_ep.c`
- `flash_hp_ep.h`

It wraps the FSP Flash HP driver and provides:

- Flash HP initialization and cleanup.
- Startup-area selection.
- Code/data flash erase.
- Code/data flash blank check.
- Code/data flash write.
- BGO event handling for supported data-flash operations.
- Conditional code-flash access-window operations.
- Board-dependent code-flash and data-flash address definitions.

The implementation translates Flash HP callback events into internal flags for blank-check, erase-complete, and write-complete handling.

## Demo/reference application versus reusable SBB code

### Reusable SBB code

`flash_hp_ep.c` and `flash_hp_ep.h` form the reusable Flash HP layer. They contain the FSP wrapper calls, BGO completion handling, flash-memory definitions, and optional access-window operation.

### Demo/reference application code

`flash_hp_example_app.c` and `flash_hp_example_app.h` provide the interactive reference application. This layer:

- Initializes the UART console.
- Displays SBB and FSP version information.
- Presents code-flash, data-flash, and exit choices.
- Creates sequential test data.
- Selects the predefined flash blocks used by the demo.
- Sequences erase, blank check, write, read-back, and comparison operations.
- Prints the read data.
- Disables interrupts around the demonstrated code-flash erase/blank-check/write sequence.
- Handles errors and enters a breakpoint trap.

The application entry point is:

```c
void flash_hp_example_app_run(void);
```

The source states that this is the single function called from `hal_entry.c`.

## Which public functions should I look at first?

### `flash_hp_init()`

```c
fsp_err_t flash_hp_init(void);
```

Opens the generated Flash HP instance using `g_flash_ctrl` and `g_flash_cfg`.

### `flash_hp_startup_block()`

```c
fsp_err_t flash_hp_startup_block(void);
```

Calls `R_FLASH_HP_StartUpAreaSelect()` to select `FLASH_STARTUP_AREA_BLOCK0` as the default startup setup block.

### `flash_hp_erase()`

```c
fsp_err_t flash_hp_erase(uint32_t block_addr);
```

Erases `BLOCK_NUM` blocks beginning at the requested flash address. `BLOCK_NUM` is defined as `2` in the supplied SBB. When a data-flash address is used and `g_flash_cfg.data_flash_bgo` is enabled, the wrapper waits for the erase-complete callback flag.

### `flash_hp_blank_check()`

```c
fsp_err_t flash_hp_blank_check(uint32_t block_addr,
                               uint32_t num_bytes);
```

Performs a blank check for the requested address and length. It supports immediate blank/not-blank results and the BGO-active result path.

### `flash_hp_write()`

```c
fsp_err_t flash_hp_write(uint8_t * src_address,
                         uint32_t flash_address,
                         uint32_t num_bytes);
```

Writes the requested number of bytes to code or data flash. For data flash with BGO enabled, the function waits for the write-complete callback.

### `flash_hp_access_window_operations()`

```c
fsp_err_t flash_hp_access_window_operations(uint8_t * write_buffer);
```

This API is compiled only when `BSP_FEATURE_FLASH_SUPPORTS_ACCESS_WINDOW` is enabled. It sets an access window from code-flash block 3 to block 7, erases block 5, writes `BLOCK_SIZE` bytes to block 5, and clears the access window.

### `flash_hp_deinit()`

```c
void flash_hp_deinit(void);
```

Closes Flash HP when the generated control instance is open.

### `bgo_callback()`

```c
void bgo_callback(flash_callback_args_t * p_args);
```

Records Flash HP blank, not-blank, erase-complete, and write-complete events for the wrapper functions.

## What configuration assumptions were made for the reference boards?

### Common operation constants

For all three requested boards, the supplied header selects the non-RA4L1/RA4C1 path and therefore defines:

```c
#define FLASH_HP_CF_BLOCK_SIZE  (8 * 1024)
#define FLASH_HP_DF_BLOCK_SIZE  (64)
#define BLOCK_SIZE              (128)
#define BLOCK_NUM               (2)
```

The demonstration therefore creates 128-byte read/write buffers, while the erase wrapper requests two blocks.

### FPB-RA4E2

The supplied source selects the standard code-flash map for FPB-RA4E2. Relevant demonstration addresses include:

- Code-flash block 5: `0x0000A000`
- Code-flash block 8: `0x00010000`
- Data-flash block 0: `0x08000000`
- Data-flash block 1: `0x08000040`

The data-flash address definitions use 64-byte block increments.

### FPB-RA6E2

FPB-RA6E2 uses the same code-flash and data-flash address definitions used by FPB-RA4E2 in the supplied header:

- Code-flash block 5: `0x0000A000`
- Code-flash block 8: `0x00010000`
- Data-flash block 0: `0x08000000`
- Data-flash block 1: `0x08000040`

### FPB-RA8E1

FPB-RA8E1 has separate code/data flash address definitions in the supplied header:

- Code-flash block 5: `0x0200A000`
- Code-flash block 8: `0x02010000`
- Data-flash block 0: `0x27000000`
- Data-flash block 1: `0x27000040`

The header documents code-flash blocks 0 through 7 as 8 KB regions and block 8 onward with the address ranges specified in the source.

### Predefined demo blocks

The demonstration does not ask the user to enter an arbitrary flash block. It uses:

- `FLASH_HP_CF_BLOCK_8` for the normal code-flash demonstration.
- `FLASH_HP_DF_BLOCK_1` for the data-flash demonstration.
- `FLASH_HP_CF_BLOCK_3` through `FLASH_HP_CF_BLOCK_7` for the optional access window, with block 5 used for erase/write.

### Data-flash BGO

Erase and write operations check `g_flash_cfg.data_flash_bgo`. Blank check also handles `FLASH_RESULT_BGO_ACTIVE`. BGO therefore depends on the generated Flash HP configuration and `bgo_callback()`.

## Important flash-operation precautions

The code-flash flow disables interrupts before erase, blank-check, and write operations and re-enables them afterward. The source states that this prevents vector-table access while code flash is in program/erase mode.

The optional access-window implementation contains a stronger warning: setting the access window incorrectly can lock code flash, and that lock persists across a power cycle. The supplied implementation clears the access window after its block-5 operation.

The reference application performs destructive erase/write operations on predefined flash blocks. An integration must ensure that equivalent target addresses do not contain firmware or other information that must be retained.

## What must I change for my board or application?

### Flash addresses and block geometry

The addresses in `flash_hp_ep.h` are MCU/board dependent. If the SBB is moved to another target or different memory region, validate the code-flash and data-flash addresses and block sizes against that target before running erase or write operations.

### Generated Flash HP configuration

The reusable source expects:

```text
g_flash_ctrl
g_flash_cfg
```

The generated instance configuration determines behavior such as data-flash BGO and the callback used for background events.

### Code-flash target

The reference application programs `FLASH_HP_CF_BLOCK_8`. A product integration must choose a code-flash location that is valid for its memory layout and does not overwrite executing code or required data.

### Data-flash target

The reference application uses `FLASH_HP_DF_BLOCK_1`. Change the target only after validating the data-flash layout and erase/write requirements for the target MCU.

### Access-window configuration

Access-window functionality is conditional on `BSP_FEATURE_FLASH_SUPPORTS_ACCESS_WINDOW`. If retained, the block-3-to-block-7 window and block-5 operation must be validated for the product memory map and security/configuration requirements.

### Startup-area selection

The reference application calls `flash_hp_startup_block()` immediately after initialization and selects `FLASH_STARTUP_AREA_BLOCK0`. Review that behavior if the product has its own startup-area or boot configuration requirements.

### Clocks and timing

The supplied README specifically mentions a HOCO requirement for EK-RA4L1 and EK-RA4C1, not for FPB-RA4E2, FPB-RA6E2, or FPB-RA8E1. The supplied files do not otherwise provide board-specific flash clock settings or measured erase/write timing for the three boards covered here.

Any clock changes should therefore be validated against the target's generated Flash HP configuration and Flash HP requirements.

### Memory

The code-flash and data-flash demo functions each allocate a `BLOCK_SIZE` write buffer and read buffer. For the three requested boards, `BLOCK_SIZE` is 128 bytes. The supplied files do not provide the complete RAM/ROM footprint of the SBB.

### Task priorities and execution model

The supplied reference application does not define an RTOS or task priorities. Data-flash BGO completion is communicated through callback flags, but the wrapper waits in polling loops until those flags are set. An RTOS-based integration must define its own synchronization and scheduling behavior.

### Console middleware

The terminal menu uses `console/uart_console.h` and standard I/O. This is demonstration code. The reusable Flash HP wrappers do not require the demo menu itself.

## Dependencies

### Renesas FSP Flash HP driver

The reusable SBB uses:

- `R_FLASH_HP_Open()`
- `R_FLASH_HP_StartUpAreaSelect()`
- `R_FLASH_HP_Erase()`
- `R_FLASH_HP_BlankCheck()`
- `R_FLASH_HP_Write()`
- `R_FLASH_HP_AccessWindowSet()` when access-window support is compiled
- `R_FLASH_HP_AccessWindowClear()` when access-window support is compiled
- `R_FLASH_HP_Close()`

### Generated FSP configuration

The implementation includes `hal_data.h` and relies on `g_flash_ctrl`, `g_flash_cfg`, BSP feature macros, and the generated Flash HP callback configuration.

### Callback events

The BGO callback handles:

- `FLASH_EVENT_NOT_BLANK`
- `FLASH_EVENT_BLANK`
- `FLASH_EVENT_ERASE_COMPLETE`
- `FLASH_EVENT_WRITE_COMPLETE`

### Console/reference application

The demo uses:

- `console/uart_console.h`
- `uart_console_init()`
- `uart_console_has_char()`
- `uart_console_flush()`
- `printf()`
- `getchar()`
- `memcpy()`
- `memcmp()`
- `R_FSP_VersionGet()`

It also uses `__disable_irq()`, `__enable_irq()`, and `__BKPT()` in the flash and error-management flows.

## Debug-session setup from the supplied README

The supplied README states that after a successful build the user should initialize the device with **Renesas Device Partition Manager** before launching the debug session. It specifies **SWD** as the connection type and points users to the referenced Renesas documentation for device initialization.

This setup requirement is part of the provided Flash HP example documentation and should be retained when reproducing that reference workflow.

## Serial terminal configuration

The supplied README specifies:

| Setting | Value |
|---|---|
| Port | Port provided by J-Link On-Board |
| Speed | 115200 |
| Data | 8 bit |
| Parity | None |
| Stop bits | 1 bit |
| Flow control | None |

For Tera Term:

- Receive newline: **CR**
- Transmit newline: **CR**
- **Local echo** enabled

The README states that user input should not exceed 15 bytes and that operation is not guaranteed for non-integer terminal input. It also warns that entering input after selecting the exit option can result in an error trap and is not recommended.

## What behavior and performance were demonstrated or measured?

### Demonstrated code-flash behavior

The reference application:

1. Creates a sequential 128-byte write pattern.
2. Disables interrupts.
3. Erases starting at code-flash block 8.
4. Blank-checks the configured code-flash block size.
5. Writes 128 bytes to block 8.
6. Re-enables interrupts.
7. Reads 128 bytes back using `memcpy()`.
8. Compares the read and write buffers.
9. Prints the read data if verification succeeds.
10. Runs the optional access-window operation when supported.

### Demonstrated data-flash behavior

The application:

1. Erases starting at data-flash block 1.
2. Blank-checks the configured data-flash block size.
3. Creates the sequential 128-byte pattern.
4. Writes 128 bytes to data-flash block 1.
5. Reads the data back.
6. Compares the read and write buffers.
7. Prints the data when verification succeeds.

### Performance measurements

The supplied files do **not** report measured erase time, write time, blank-check time, CPU utilization, RAM/ROM footprint, flash endurance, BGO latency, or throughput.

The 128-byte demonstration data size and defined block sizes are configuration/implementation properties rather than measured performance results. This SBB should therefore be treated as a functional Flash HP reference rather than a performance benchmark.

## What files are safe to modify and what files are generated, preset, or demo-only?

| File | Classification | Guidance |
|---|---|---|
| `flash_hp_ep.c` | Reusable SBB implementation | Modify when intentionally changing Flash HP wrappers, BGO behavior, startup-area selection, or optional access-window behavior. |
| `flash_hp_ep.h` | Reusable SBB interface and board/device presets | Modify carefully when changing public APIs, flash addresses, block sizes, command constants, or BSP-dependent configuration. |
| `flash_hp_example_app.c` | Demo/reference application | Appropriate location for changing menu behavior, test patterns, predefined demo targets, output, verification, and error handling. |
| `flash_hp_example_app.h` | Demo/reference application interface | Contains the application entry point, version, banner, menu, and descriptive text. |
| `readme.txt` | Existing example documentation | Source for the validated board list, FPB-RA8E1 jumper setup, debug-session notes, input restrictions, and terminal settings. |

The SBB includes `hal_data.h` and depends on generated FSP configuration, but the supplied files do not enumerate the generated files individually. Those generated files should be treated separately from the hand-maintained SBB and demo sources.

## What happens if I modify the source and later adopt a newer SBB release?

Local changes to `flash_hp_ep.c` or `flash_hp_ep.h` become modifications to the reusable SBB layer and may overlap changes in a future release. This is particularly important because `flash_hp_ep.h` contains board/device-specific flash geometry and addresses.

To reduce upgrade conflicts:

- Keep menu behavior, display text, sample data, and demo sequencing in `flash_hp_example_app.*`.
- Prefer generated FSP configuration for settings owned by the Flash HP configurator.
- Keep product memory-map changes explicit and separately reviewable.
- Keep reusable-source changes focused and document why they are required.
- Compare a newer `flash_hp_ep.c` and `flash_hp_ep.h` against the locally modified versions before carrying local changes forward.
- Revalidate code/data flash addresses, block geometry, startup-area selection, BGO configuration, interrupt handling, access-window behavior, and read-back verification after integration.

The supplied files do not define an automated migration or compatibility mechanism for future SBB releases. Compatibility must therefore be validated by the integrating project.

## Recommended reading order

1. **`flash_hp_ep.h`** to understand flash geometry, board-specific addresses, public functions, and feature-controlled behavior.
2. **`flash_hp_ep.c`** to understand Flash HP initialization, erase/blank-check/write wrappers, BGO processing, startup-area selection, and access-window behavior.
3. **`flash_hp_example_app.c`** to see the complete code-flash and data-flash workflows and verification sequence.
4. **`flash_hp_example_app.h`** to understand the application entry point and terminal menu.
5. **`readme.txt`** for the original supported-board, debug-session, FPB-RA8E1 jumper, input, and terminal notes.
