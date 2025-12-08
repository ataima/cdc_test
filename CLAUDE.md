# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an **STM32F103C8T6** (Blue Pill) embedded firmware project implementing a **USB CDC (Communication Device Class)** interface for virtual COM port functionality. The project is built using STM32CubeIDE and uses the STM32 HAL (Hardware Abstraction Layer) libraries.

**Target MCU:** STM32F103C8Tx
- ARM Cortex-M3 core
- 64KB Flash memory
- 20KB RAM
- 48 MHz system clock (from 8 MHz external HSE with PLL×6)
- USB Full-Speed Device peripheral

## Build System

This project uses **STM32CubeIDE** (Eclipse-based) as the primary build environment. It is configured as an Eclipse managed build project with two build configurations:

### Build Commands

Build the project within STM32CubeIDE or using the command line:

```bash
# Clean build
Project → Clean... (in STM32CubeIDE GUI)

# Build Debug configuration
Project → Build Project (in STM32CubeIDE GUI)

# Build Release configuration
Switch to Release configuration first, then Build Project
```

**Note:** This project does NOT have a standalone Makefile. Building must be done through STM32CubeIDE's Eclipse-based build system which auto-generates build files.

### Build Configurations

- **Debug:** Full debug symbols (-g3), no optimization, includes DEBUG symbol
- **Release:** No debug info (-g0), size optimization (-Os)

### Build Artifacts

- Binary output: `Debug/CDC_test.elf` or `Release/CDC_test.elf`
- Also generates: `.hex`, `.bin`, `.list`, `.srec` files

## Architecture Overview

### Code Generation and STM32CubeMX

This project uses **STM32CubeMX** for peripheral initialization code generation:

- **CDC_test.ioc:** STM32CubeMX project file defining all peripheral configurations
- Regenerating code from this file will overwrite auto-generated sections
- **USER CODE blocks** are preserved during regeneration - all custom code must go in these sections

### Directory Structure

```
Core/
├── Inc/           # Application headers and HAL configuration
│   ├── main.h
│   ├── stm32f1xx_hal_conf.h  # HAL library configuration
│   └── stm32f1xx_it.h        # Interrupt handlers
├── Src/           # Application source code
│   ├── main.c                # Main application entry point
│   ├── stm32f1xx_it.c        # Interrupt service routines
│   ├── stm32f1xx_hal_msp.c   # HAL MSP (peripheral init/deinit)
│   ├── system_stm32f1xx.c    # System initialization
│   ├── syscalls.c            # Newlib syscall stubs
│   └── sysmem.c              # Memory allocation
└── Startup/
    └── startup_stm32f103c8tx.s  # Startup assembly code

USB_DEVICE/
├── App/           # USB device application layer
│   ├── usb_device.c/h        # USB device initialization
│   ├── usbd_desc.c/h         # USB descriptors (VID/PID, strings)
│   └── usbd_cdc_if.c/h       # CDC interface implementation
└── Target/
    └── usbd_conf.c/h         # USB device HAL configuration

Drivers/
├── STM32F1xx_HAL_Driver/     # STM32 HAL peripheral drivers
├── CMSIS/                     # ARM CMSIS headers
└── (Do not modify - ST provided libraries)

Middlewares/
└── ST/STM32_USB_Device_Library/  # USB device middleware
    ├── Core/                      # USB core stack
    └── Class/CDC/                 # CDC class driver
```

### Hardware Abstraction Layers

The codebase is structured in distinct layers:

1. **Application Layer** (`Core/Src/main.c`, `USB_DEVICE/App/`)
   - Main loop and application logic
   - USB CDC interface callbacks and data handling

2. **HAL Layer** (`Drivers/STM32F1xx_HAL_Driver/`)
   - Hardware abstraction for all peripherals
   - Consistent API across STM32 families

3. **USB Middleware** (`Middlewares/ST/STM32_USB_Device_Library/`)
   - USB device stack (core protocol handling)
   - CDC class implementation

4. **CMSIS Layer** (`Drivers/CMSIS/`)
   - ARM Cortex-M core definitions
   - STM32F1 device-specific headers and register definitions

### USB CDC Architecture

The USB CDC implementation creates a virtual COM port:

**Data Flow:**
- PC → USB → `CDC_Receive_FS()` in `usbd_cdc_if.c` → Application
- Application → `CDC_Transmit_FS()` in `usbd_cdc_if.c` → USB → PC

**Key Files:**
- `usbd_cdc_if.c`: Interface between USB CDC class and application
  - `CDC_Receive_FS()`: Called when data received from host
  - `CDC_Transmit_FS()`: Send data to host (check TxState to avoid USBD_BUSY)
  - `UserRxBufferFS[APP_RX_DATA_SIZE]`: RX buffer (1024 bytes)
  - `UserTxBufferFS[APP_TX_DATA_SIZE]`: TX buffer (1024 bytes)

- `usbd_desc.c`: USB descriptors defining device identity
- `usbd_conf.c`: Low-level USB peripheral configuration (interrupts, callbacks)

### Clock Configuration

System clock is configured in `SystemClock_Config()` in `main.c`:
- HSE (High-Speed External): 8 MHz crystal oscillator
- PLL multiplier: ×6
- System clock (SYSCLK): 48 MHz
- AHB: 48 MHz (no divider)
- APB1: 24 MHz (÷2)
- APB2: 48 MHz (no divider)
- USB clock: 48 MHz (required for USB Full-Speed)
- MCO output: SYSCLK on PA8

### Memory Layout

Defined in `STM32F103C8TX_FLASH.ld`:
- Flash: 0x08000000 - 64KB
- RAM: 0x20000000 - 20KB
- Stack size: 1KB (0x400)
- Heap size: 512 bytes (0x200)

## Development Guidelines

### Modifying Auto-Generated Code

STM32CubeMX regenerates initialization code. To preserve custom code:

1. Place all custom code inside `/* USER CODE BEGIN */` and `/* USER CODE END */` blocks
2. These exist in strategic locations in all auto-generated files
3. Code outside these blocks will be overwritten when regenerating from the .ioc file

Example from `main.c`:
```c
/* USER CODE BEGIN 0 */
// Custom functions and variables go here
/* USER CODE END 0 */
```

### Adding USB CDC Functionality

To send data over USB CDC:
```c
#include "usbd_cdc_if.h"

uint8_t data[] = "Hello World\r\n";
CDC_Transmit_FS(data, strlen(data));
```

To receive data, implement handling in `CDC_Receive_FS()` callback in `usbd_cdc_if.c`:
```c
static int8_t CDC_Receive_FS(uint8_t* Buf, uint32_t *Len)
{
  /* USER CODE BEGIN 6 */
  // Process received data in Buf with length *Len

  USBD_CDC_SetRxBuffer(&hUsbDeviceFS, &Buf[0]);
  USBD_CDC_ReceivePacket(&hUsbDeviceFS);
  return (USBD_OK);
  /* USER CODE END 6 */
}
```

### Peripheral Configuration

To add or modify peripheral configurations:
1. Open `CDC_test.ioc` in STM32CubeMX
2. Configure peripherals using the graphical interface
3. Generate code (will update initialization code while preserving USER CODE sections)
4. Rebuild project in STM32CubeIDE

### Debugging

The project is configured for hardware debugging:
- **SWD (Serial Wire Debug) interface** - JTAG is disabled
- Compatible with ST-Link debuggers
- Debug configuration: `CDC_test.launch`

**Important:** In STM32CubeIDE debug configuration, ensure the interface is set to **SWD** (not JTAG):
1. Right-click project → **Debug As → Debug Configurations...**
2. Select `CDC_test Debug` configuration
3. Go to **Debugger** tab
4. Under **Debug probe**, verify:
   - Debug probe: ST-LINK (ST-LINK GDB server)
   - Interface: **SWD** ← Must be SWD, not JTAG!
5. Apply and Debug

Since the firmware uses `__HAL_AFIO_REMAP_SWJ_NOJTAG()`, JTAG pins are released as GPIOs and only SWD remains active. Using JTAG interface in debugger will fail to connect.

### Critical Constraints

- **USB requires 48 MHz clock:** Do not change clock configuration without ensuring USB clock remains at 48 MHz
- **Limited RAM (20KB):** Be mindful of stack/heap usage and buffer sizes
- **Limited Flash (64KB):** Includes all code, USB stack, HAL drivers
- **USB TX state checking:** Always check TxState before calling `CDC_Transmit_FS()` to avoid USBD_BUSY errors

## Critical Warning: Debug Interface Configuration

### ⚠️ NEVER Disable SWD Debug Interface!

In STM32CubeMX configuration (`CDC_test.ioc`):
- **System Core → SYS → Debug** must be set to **"Serial Wire"** or **"Trace Asynchronous Sw"**
- **NEVER** set to **"No Debug"**

**Why:** Setting "No Debug" generates code that disables the SWD interface (`__HAL_AFIO_REMAP_SWJ_DISABLE()`), making the STM32 impossible to reprogram via ST-LINK. This appears as "Unknown MCU" error when connecting.

### Recovery Procedure (If SWD is Disabled)

If you accidentally programmed firmware with SWD disabled:

1. **Set Boot Mode:**
   - Disconnect power from Blue Pill
   - Set jumper **BOOT0 = 1** (right position on 3-pin header)
   - Keep **BOOT1 = 0** (left position)
   - Reconnect power

2. **Fix Configuration:**
   - Open `CDC_test.ioc` in STM32CubeMX
   - Set System Core → SYS → Debug to **"Serial Wire"**
   - Generate code (Project → Generate Code)
   - Verify `Core/Src/stm32f1xx_hal_msp.c` contains `__HAL_AFIO_REMAP_SWJ_NOJTAG()` instead of `__HAL_AFIO_REMAP_SWJ_DISABLE()`

3. **Rebuild and Program:**
   - Clean and rebuild project
   - Generate binary: `arm-none-eabi-objcopy -O binary ./Debug/CDC_test.elf ./Debug/CDC_test.bin`
   - Program: `st-flash write ./Debug/CDC_test.bin 0x08000000`

4. **Restore Normal Boot:**
   - Disconnect power
   - Set **BOOT0 = 0** (left position)
   - Reconnect power

The STM32 will now boot with SWD enabled and ST-LINK can connect normally.

### Generating Binary Files

By default, STM32CubeIDE only generates `.elf` files. To enable `.bin` and `.hex` generation:

1. Right-click project → **Properties**
2. **C/C++ Build → Settings → MCU Post build outputs**
3. Enable:
   - ✅ Convert to binary file (.bin)
   - ✅ Convert to Intel Hex file (.hex)
4. Apply and rebuild

Alternative: Manual conversion from ELF:
```bash
arm-none-eabi-objcopy -O binary ./Debug/CDC_test.elf ./Debug/CDC_test.bin
```
