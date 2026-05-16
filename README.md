# Epicyon User Guide

**Version 0.1.0**  
*inspire innovation*

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Architecture](#2-system-architecture)
3. [Prerequisites & Installation](#3-prerequisites--installation)
4. [First Launch](#4-first-launch)
5. [Creating a Project](#5-creating-a-project)
6. [Preparing Your Simulink Model](#6-preparing-your-simulink-model)
7. [Epicyon Simulink Blocks Reference](#7-epicyon-simulink-blocks-reference)
8. [Code Generation](#8-code-generation)
9. [Build](#9-build)
10. [Flashing to Hardware](#10-flashing-to-hardware)
11. [Live Dashboard](#11-live-dashboard)
12. [Live Parameter Tuning](#12-live-parameter-tuning)
13. [Standalone Monitoring from Simulink](#13-standalone-monitoring-from-simulink)
14. [Tool Discovery & Troubleshooting](#14-tool-discovery--troubleshooting)
15. [Appendix](#15-appendix)

---

## 1. Introduction

Epicyon is a rapid-deployment platform that bridges the gap between **MATLAB/Simulink** model-based design and **STM32 microcontroller** hardware. It automates the entire chain from algorithm design to running embedded firmware — without requiring manual project configuration in STM32CubeIDE, hand-written Makefiles, or linker scripts.

### What Epicyon does

| Stage | Without Epicyon | With Epicyon |
|-------|----------------|--------------|
| Code generation | Manually invoke Embedded Coder, copy files | One click |
| Compilation | Write & maintain Makefiles, manage include paths | One click |
| Flashing | Open STM32CubeIDE or STM32CubeProgrammer manually | One click |
| Live monitoring | UART terminal, oscilloscope, or External Mode setup | Browser dashboard with live charts |
| Parameter tuning | Re-flash firmware for every change | Drag a slider, change takes effect instantly |

### Intended users

Epicyon is designed for **control engineers** who are familiar with Simulink and want to validate their algorithms on real hardware quickly, without spending time on embedded toolchain management.

---

## 2. System Architecture

Understanding the overall architecture helps you use Epicyon more effectively.

```
┌─────────────────────────────────────────────────────────┐
│                      YOUR PC                            │
│                                                         │
│  ┌──────────┐    ┌─────────────┐    ┌───────────────┐  │
│  │ MATLAB / │───▶│   Epicyon   │───▶│   Browser UI  │  │
│  │ Simulink │    │    Agent    │    │ localhost:8845 │  │
│  └──────────┘    └──────┬──────┘    └───────────────┘  │
│                         │ USB / ST-Link                 │
└─────────────────────────┼───────────────────────────────┘
                          │
                  ┌───────▼────────┐
                  │  STM32 Board   │
                  │  (Nucleo etc.) │
                  └────────────────┘
```

> **Figure 1** — Epicyon system overview. The Agent runs locally on your PC and orchestrates code generation (via MATLAB), compilation (via GCC), flashing (via ST-Link), and live telemetry streaming (via UART).

### Components

**Epicyon Agent** (`epicyon.exe`)
The core background process. It exposes a REST API on `localhost:8845`, manages project files, invokes the compiler, calls STM32_Programmer_CLI for flashing, and maintains a WebSocket connection to stream live telemetry to the browser.

**Browser UI**
A web application served by the Agent. It provides the Projects page, Build page, and live Dashboard. No internet connection is required — everything runs locally.

**Epicyon Firmware Runtime** (embedded in generated firmware)
A small C library compiled into your firmware that handles:
- Reading ADC/GPIO inputs at each model step
- Writing PWM/GPIO outputs at each model step
- Sending telemetry packets over UART (signal values)
- Receiving parameter update packets over UART

**MATLAB S-Function Blocks**
Custom Simulink blocks (`epicyon_adc`, `epicyon_pwm`, `epicyon_din`, `epicyon_dout`, `epicyon_signal`, `epicyon_param`) that integrate hardware I/O and live monitoring into your Simulink model.

---

## 3. Prerequisites & Installation

### 3.1 Prerequisites

Before installing Epicyon, ensure the following software is installed on your PC:

#### STM32CubeIDE *(mandatory)*
STM32CubeIDE is a free IDE from STMicroelectronics. Epicyon does **not** use the IDE itself, but it relies on the bundled tools:
- `arm-none-eabi-gcc` — the ARM cross-compiler
- `OpenOCD` — the on-chip debugger
- `STM32_Programmer_CLI` — the command-line flashing utility

Download from: [st.com/stm32cubeide](https://www.st.com/en/development-tools/stm32cubeide.html)

> **Note:** After installing STM32CubeIDE, open it once and create any dummy project. This triggers the download of the STM32CubeF4 HAL library to `~/STM32Cube/Repository/`, which Epicyon needs for compilation.

#### MATLAB, Simulink, and Embedded Coder *(mandatory for code generation)*
Epicyon calls MATLAB's `slbuild()` function to generate C code from your Simulink model. A valid Embedded Coder license is required.

If you only want to **flash and monitor** a pre-built firmware, MATLAB is not required on that machine.

> **Figure 2** — Screenshot of the Tool Discovery panel in Epicyon showing all four tools found (green checkmarks).

### 3.2 Installation

1. Double-click `epicyon-setup-v0.1.0.exe`
2. Choose an install folder (default: `C:\Epicyon`)
3. Click **Install** — the process takes approximately 30 seconds
4. A desktop shortcut named **Epicyon** is created

No Python, Node.js, or other runtime is required. The installer is self-contained.

### 3.3 MATLAB Setup

After installing Epicyon, run the following once in MATLAB to register the Simulink blocks:

```matlab
run('C:\Epicyon\simulink\epicyon_setup.m')
```

This script automatically:
1. Adds the Epicyon block library to your MATLAB path (permanently saved)
2. Compiles the MEX S-functions using your installed C compiler (MinGW)
3. Registers the path in `startup.m` so it loads automatically in future MATLAB sessions

> **Figure 3** — MATLAB Command Window output after running `epicyon_setup.m` showing successful compilation of all blocks.

If MinGW is not installed, MATLAB will prompt you to install it from Add-Ons. It is free and takes about 2 minutes.

---

## 4. First Launch

1. Double-click the **Epicyon** desktop shortcut
2. A console window opens — this is the Epicyon Agent. **Keep it running** while using Epicyon
3. Your default browser opens automatically at `http://127.0.0.1:8845`
4. The **Projects** page appears

> **Figure 4** — The Epicyon Projects page on first launch. The Tool Discovery panel on the right confirms which tools were found automatically.

If the browser does not open automatically, navigate to `http://127.0.0.1:8845` manually.

**Closing Epicyon:** Close the console window. The browser tab can be closed separately. Any STM32 board already flashed continues running its firmware independently.

---

## 5. Creating a Project

A **project** in Epicyon holds all the files associated with one firmware: the generated C source code, compiled binary, build log, and signal/parameter configuration.

### Steps

1. On the Projects page, enter a **project name** in the Name field
   - Use only letters, digits, and underscores (e.g., `speed_controller`, `motor_test_1`)
2. Select your **target board** from the Board dropdown (e.g., `nucleo-f411re`)
3. Click **Create Project**
4. The **Build** page opens automatically for the new project

> **Figure 5** — The New Project form. Name field (left) and Board selector (right). The board selection determines pin assignments, clock speed, and HAL configuration.

Multiple projects can coexist. You can switch between them from the Projects page at any time.

---

## 6. Preparing Your Simulink Model

### 6.1 Solver Configuration

Epicyon requires your Simulink model to use a **Fixed-Step solver**. Variable-step solvers are not supported because embedded firmware must execute at a precise, deterministic sample rate.

In MATLAB: **Simulation → Model Configuration Parameters → Solver**
- Type: `Fixed-step`
- Solver: `ode3 (Bogacki-Shampine)` or `ode1 (Euler)` — both work well
- Fixed-step size: your desired sample period in seconds (e.g., `0.001` for 1 ms)

> **Figure 6** — Simulink Model Configuration Parameters window with Fixed-step solver selected and a 1 ms step size.

### 6.2 Data Types

All Epicyon blocks use **double-precision** signals (the default in Simulink). You do not need to add Data Type Conversion blocks — the hardware I/O abstraction handles casting internally.

### 6.3 Model Structure

A typical Epicyon model has the following structure:

```
[EpicyonADC] ──▶ [Your Control Algorithm] ──▶ [EpicyonPWM]
                         │
                  [EpicyonSignal]   ← streams value to dashboard
```

> **Figure 7** — Example Simulink block diagram showing a PI speed controller. ADC block reads the sensor, the PI block computes the control output, PWM block drives the actuator, and Signal blocks send selected variables to the Epicyon dashboard.

### 6.4 Model Initialization (recommended)

If you plan to use `epicyon_signal` blocks for Simulink-side monitoring (see Section 13), add the following to your model's **InitFcn** callback:

```matlab
% Model → Model Properties → Callbacks → InitFcn
epicyon_recv('open', 'COM3', 115200)
```

And to the **StopFcn** callback:
```matlab
epicyon_recv('close')
```

---

## 7. Epicyon Simulink Blocks Reference

After running `epicyon_setup.m`, the following blocks are available. Add them to your model using an **S-Function** block and entering the block name.

> **Figure 8** — Simulink Library Browser showing the Epicyon block group with all available blocks.

---

### 7.1 EpicyonADC — Analog Input

**S-Function name:** `epicyon_adc`  
**Dialog parameter:** `channel` (integer, 0-based)  
**Output:** scalar double — normalized voltage ratio [0.0 … 1.0]

Reads the voltage on the specified ADC channel. The output represents the ratio relative to the reference voltage (3.3 V):

```
0.0  →  0.0 V  (GND)
1.0  →  3.3 V  (Vref)
```

To convert to actual voltage in your model: multiply by 3.3.

**Example:** Reading a potentiometer connected to ADC channel 0:

```
[EpicyonADC (ch=0)] ──▶ [Gain: 3.3] ──▶ voltage_volts
```

> **Figure 9** — EpicyonADC block in a Simulink model with channel parameter dialog open.

| Channel Index | Physical Pin (Nucleo-F411RE) |
|--------------|------------------------------|
| 0 | PA0 (A0 on Arduino header) |
| 1 | PA1 (A1) |
| 2 | PA4 (A2) |
| 3 | PB0 (A3) |

---

### 7.2 EpicyonPWM — Analog Output (PWM)

**S-Function name:** `epicyon_pwm`  
**Dialog parameter:** `channel` (integer, 0-based)  
**Input:** scalar double — duty cycle [0.0 … 1.0]

Sets the PWM duty cycle on the specified output channel. The input is clamped to [0, 1]:

```
0.0  →  0 % on   (always LOW)
0.5  →  50 % duty cycle
1.0  →  100 % on (always HIGH)
```

The PWM frequency is configured per-board in the board definition file (default: 20 kHz for motor control applications).

> **Figure 10** — EpicyonPWM block connected to a Saturation block (limits to [0, 1]) and a PI controller output.

**Important:** Always add a **Saturation** block before `EpicyonPWM` to clamp the signal to [0.0, 1.0], even if your controller output is theoretically bounded. Sensor noise or startup transients can produce out-of-range values.

---

### 7.3 EpicyonDIN — Digital Input

**S-Function name:** `epicyon_din`  
**Dialog parameter:** `channel` (integer, 0-based)  
**Output:** scalar double — logic level (0.0 = LOW, 1.0 = HIGH)

Reads the logic level of a GPIO input pin. The output is 0.0 or 1.0 (double), compatible with Simulink Switch, Compare to Constant, and logical operator blocks without conversion.

**Typical uses:**
- Reading a push button (with pull-up resistor)
- Reading a limit switch or end-stop
- Reading an optocoupler output for 24 V industrial signals

> **Figure 11** — EpicyonDIN block reading a push button, connected to a Switch block that selects between two reference setpoints.

---

### 7.4 EpicyonDOUT — Digital Output

**S-Function name:** `epicyon_dout`  
**Dialog parameter:** `channel` (integer, 0-based)  
**Input:** scalar double — logic level (threshold: ≥ 0.5 → HIGH, < 0.5 → LOW)

Drives a GPIO output pin. The input threshold (0.5) means you can connect boolean signals or 0/1 doubles directly without a conversion block.

**Typical uses:**
- Driving an LED indicator
- Controlling a relay via a transistor driver
- Enabling/disabling a power stage

---

### 7.5 EpicyonSignal — Telemetry Output (Dashboard Monitor)

**S-Function name:** `epicyon_signal`  
**Dialog parameter:** `sig_id` (integer, 0-based)  
**Input:** *(none — reads from telemetry stream)*  
**Output:** scalar double — most recently received signal value

This block receives a signal value that was **sent by the STM32 firmware** over UART and displays it in the Epicyon browser dashboard. It is used to bring a live variable from the running firmware back into your Simulink model for monitoring or use in a Simulink scope.

> **Figure 12** — EpicyonSignal block (sig_id=0) connected to a Simulink Scope. The scope shows the real-time ADC value streaming from the STM32 into MATLAB during simulation.

**How it works:**
1. In your Embedded Coder model (the model flashed to STM32), you place `EpicyonMonitor` blocks that tag signals with a `sig_id`
2. The firmware streams those values over UART as telemetry packets
3. The `EpicyonSignal` block in a *separate Simulink monitoring model* reads those packets and outputs the value

The signal ID must match between the firmware model and the monitoring model.

---

### 7.6 EpicyonParam — Runtime Parameter Input

**S-Function name:** `epicyon_param`  
**Dialog parameter:** `param_id` (integer, 0-based)  
**Output:** scalar double — current parameter value

Reads the current value of a tunable runtime parameter. The value is set from the Epicyon browser dashboard (slider or numeric input) and is transmitted to the STM32 over UART. The firmware receives the new value and applies it within one sample period.

> **Figure 13** — EpicyonParam block (param_id=0) used as the proportional gain input to a Gain block. The gain value is controlled live from the Epicyon dashboard without re-flashing.

**Example use case — live PID tuning:**

```
[EpicyonParam (id=0)] ──▶ Kp (proportional gain)
[EpicyonParam (id=1)] ──▶ Ki (integral gain)
[EpicyonParam (id=2)] ──▶ Kd (derivative gain)
```

Configure min/max ranges for each parameter in the Epicyon Build page (Signals & Parameters section) before building.

---

## 8. Code Generation

Code generation converts your Simulink block diagram into C source code using MATLAB's Embedded Coder.

### Steps

1. Open the **Build** page for your project
2. Enter the **Model Name** — the filename of your `.slx` model without the extension
   - Example: if your model is `speed_controller.slx`, enter `speed_controller`
3. Enter the **Sample Time** in microseconds
   - Example: if your Fixed-Step size is `0.001` s → enter `1000` µs
4. Click **Generate Code**

Epicyon calls `slbuild('your_model')` in the background MATLAB instance. MATLAB must be open and the model must be on the path.

> **Figure 14** — Build page with Model Name and Sample Time filled in before clicking Generate Code.

### What happens internally

1. Epicyon calls MATLAB via a script: `slbuild('model_name')`
2. Embedded Coder generates C files into a `model_name_ert_rtw/` folder
3. Epicyon copies the generated `.c` and `.h` files into the project folder
4. A scan is performed to detect `sig_id` and `param_id` assignments in the generated code
5. The Signals and Parameters sections of the Build page update automatically

> **Figure 15** — Build page after successful code generation, showing detected signals and parameters listed in the Signals & Parameters section.

### Common issues

| Issue | Cause | Fix |
|-------|-------|-----|
| MATLAB not responding | Model not open | Open the `.slx` file in MATLAB first |
| "Embedded Coder not found" | License missing | Check license with `ver` in MATLAB |
| "Solver must be Fixed-Step" | Variable-step solver selected | Change solver in Model Configuration |
| Generated code missing signals | EpicyonMonitor blocks not in model | Add blocks and re-generate |

---

## 9. Build

The Build step compiles the generated C code together with the STM32 HAL library to produce a flashable binary.

### Steps

1. After code generation, click **Build** on the Build page
2. The compiler log streams in real time in the log panel
3. A green **success** badge indicates a successful build
4. A `firmware.bin` file is produced in the project folder

> **Figure 16** — Build page showing the real-time compiler log with a successful build result.

### What happens internally

1. Epicyon locates `arm-none-eabi-gcc` (found automatically from STM32CubeIDE)
2. Compiles all generated `.c` files + STM32 HAL sources with the correct flags for the target MCU
3. Links the ELF and converts to `firmware.bin` using `arm-none-eabi-objcopy`

Build time is typically **10–30 seconds** depending on model complexity and PC speed.

### Build flags (for reference)

```
-mcpu=cortex-m4 -mfpu=fpv4-sp-d16 -mfloat-abi=hard
-O2 -Wall -fdata-sections -ffunction-sections
```

The FPU (`-mfloat-abi=hard`) is enabled by default — floating-point operations in your control algorithm run on hardware, not emulated in software.

---

## 10. Flashing to Hardware

### Hardware connection

Connect your STM32 Nucleo board to your PC via the **USB Mini-B / USB-C** connector on the Nucleo board (not the target MCU's own USB connector, if present). This connection provides both ST-Link programming and the VCP UART for telemetry.

> **Figure 17** — Nucleo-F411RE board with USB cable connected to the ST-Link (CN1 connector, top of board). The green LED confirms USB enumeration.

### Steps

1. Ensure the board is connected and recognised (check Device Manager — a ST-Link COM port should appear)
2. Click **Flash** on the Build page
3. Epicyon calls `STM32_Programmer_CLI` to erase and program the flash memory
4. The MCU resets automatically — your firmware starts running immediately

> **Figure 18** — Flash progress output. STM32_Programmer_CLI connects to the ST-Link, erases the flash, writes the binary, and resets the target.

Flashing takes approximately **3–8 seconds** for typical firmware sizes.

### After flashing

The STM32 is now running your Simulink model as embedded firmware. It:
- Executes your control algorithm at the configured sample rate
- Reads ADC/GPIO inputs at each step
- Writes PWM/GPIO outputs at each step
- Streams telemetry packets over the VCP UART

The PC connection is no longer needed for the firmware to run. The board can be powered from any 5 V USB supply.

---

## 11. Live Dashboard

The Dashboard provides real-time signal monitoring and parameter tuning without re-flashing.

### Connecting

1. Click **Dashboard** in the left sidebar
2. In the COM port field, enter the port corresponding to the ST-Link VCP
   - On Windows: open Device Manager → Ports (COM & LPT) → look for "STMicroelectronics STLink Virtual COM Port"
   - Leave the field empty to let Epicyon auto-detect
3. Click **Connect**
4. The status indicator turns green and shows the connected port

> **Figure 19** — Dashboard page immediately after connecting. The step counter in the top-right corner increments, confirming the firmware is running and the telemetry link is active.

### Signals section

Each signal defined in your model appears as a **live chart**. Charts update in real time as packets arrive from the STM32.

> **Figure 20** — Dashboard Signals section showing two live charts: ADC channel 0 voltage (top-left) and PWM duty cycle (top-right). Both update at the model sample rate.

The time axis shows elapsed seconds since connection. Up to 400 samples are retained per signal (older samples scroll off the left edge).

The chart color is assigned automatically and consistently per signal ID across sessions.

### Heartbeat

The step counter in the top bar (`step XXXXXXX`) increments with every firmware execution cycle. If this counter is not incrementing, the UART link is not receiving data — check the COM port selection and baud rate.

---

## 12. Live Parameter Tuning

Parameters appear in the **Parameters section** of the Dashboard as interactive sliders.

> **Figure 21** — Dashboard Parameters section showing three PID gain sliders (Kp, Ki, Kd). Each has a numeric input field on the right for precise entry.

### Changing a parameter value

- **Slider:** drag to the desired value — the new value is sent to the MCU immediately
- **Numeric field:** type a value and press Enter

The MCU applies the new value within one sample period (typically < 1 ms). There is no need to pause, re-flash, or restart.

### Parameter range configuration

The min/max range of each slider is configured in the **Build page → Parameters section**, before building. Set appropriate physical limits — for example, a PID gain should not go negative, and a setpoint should not exceed sensor range.

> **Figure 22** — Build page Parameters configuration table. Each row allows setting the parameter name, default value, minimum, and maximum.

### Use case: PID tuning workflow

1. Flash the initial firmware with conservative gain values
2. Connect to the Dashboard
3. Observe the system response on the Signal charts
4. Adjust Kp, Ki, Kd sliders until the response is satisfactory
5. Note the final values — update your Simulink model's initial values accordingly

This workflow replaces the traditional cycle of: *edit → build → flash → observe → repeat*.

---

## 13. Standalone Monitoring from Simulink

In addition to the browser dashboard, you can receive live telemetry signals directly into a **running Simulink simulation**. This allows you to use Simulink Scope blocks, signal analysers, or model-based post-processing on live hardware data.

### Setup

1. Flash your firmware to the STM32 as normal
2. In MATLAB, open or create a **monitoring model** (separate from the firmware model)
3. Add `EpicyonSignal` blocks for each signal you want to observe
4. Run the following before starting the simulation:

```matlab
epicyon_recv('open', 'COM3', 115200)
```

5. Run the simulation — the `EpicyonSignal` blocks will output live values from the STM32

> **Figure 23** — Simulink monitoring model with three EpicyonSignal blocks feeding a Scope. The Scope shows the controller reference, output, and error signals streaming live from the STM32 firmware.

### Stopping

```matlab
epicyon_recv('close')
```

Or add these calls to the model's `InitFcn` / `StopFcn` callbacks for automatic management.

### Simultaneous use with browser dashboard

The browser dashboard and the Simulink monitoring approach **cannot be used simultaneously** — only one application can own the serial port at a time. Use whichever suits your current task.

---

## 14. Tool Discovery & Troubleshooting

### 14.1 Tool Discovery

The **Tool Discovery** panel on the Projects page shows whether Epicyon found all required tools. Epicyon searches the standard installation paths automatically — no manual PATH configuration is needed.

> **Figure 24** — Tool Discovery panel showing all four tools found (green) and one missing tool (amber warning with the searched path).

| Tool | Search locations |
|------|-----------------|
| `arm-none-eabi-gcc` | STM32CubeIDE plugin directory, `C:\Program Files\GNU Arm Embedded Toolchain\` |
| STM32Cube HAL | `~/STM32Cube/Repository/STM32Cube_FW_F4_*` |
| OpenOCD | STM32CubeIDE debug plugin, `C:\Program Files\OpenOCD\` |
| STM32_Programmer_CLI | `C:\Program Files\STMicroelectronics\STM32Cube\STM32CubeProgrammer\` |

If a tool shows as **Not Found**:
1. Ensure the corresponding software is installed
2. Restart the Epicyon Agent (close and reopen the console window)
3. The Tool Discovery panel refreshes automatically on the next page load

### 14.2 Troubleshooting

#### Build fails — compiler error in log

The build log shows the exact error line and file. Common causes:

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `arm-none-eabi-gcc: command not found` | GCC not found | Check Tool Discovery |
| `epicyon_io.h: No such file` | Include path wrong | Re-run `epicyon_setup.m` |
| `undefined reference to epicyon_adc_read` | HAL not found | Open STM32CubeIDE once to trigger HAL download |
| `multiple definition of main` | Old generated files | Delete project source files and re-generate |

#### Flash fails

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `No ST-LINK detected` | Board not connected | Check USB cable and Device Manager |
| `Connection error` | ST-Link driver missing | Install ST-Link driver from STM32CubeIDE |
| `Error erasing flash` | Board in protection mode | Use STM32CubeProgrammer to remove protection |

#### Dashboard shows no data

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Step counter not incrementing | Wrong COM port | Select the ST-Link VCP port (higher numbered COM) |
| Charts empty but counter runs | No `EpicyonMonitor` blocks in model | Add monitor blocks and rebuild |
| Connection refused | Wrong baud rate | Default is 115200 — check firmware configuration |

#### MATLAB code generation hangs or fails

1. Open MATLAB manually and run `slbuild('your_model')` to see detailed error output
2. Ensure the model is open and on the MATLAB path
3. Verify Embedded Coder is licensed: run `ver` in MATLAB and look for `Embedded Coder`
4. Set solver to Fixed-Step before generating

---

## 15. Appendix

### 15.1 Telemetry Protocol

Epicyon uses a compact binary protocol over UART (default: **115200 baud, 8N1**).

#### Telemetry packet (STM32 → PC)
```
Byte 0:   0xEA  (magic)
Byte 1:   0x01  (type: telemetry)
Byte 2:   sig_id (uint8, 0-based)
Bytes 3–6: value (float32, little-endian)
Byte 7:   CRC (XOR of bytes 0–6)
Total: 8 bytes
```

#### Heartbeat packet (STM32 → PC)
```
Byte 0:   0xEA  (magic)
Byte 1:   0x00  (type: heartbeat)
Bytes 2–5: step_count (uint32, little-endian)
Byte 6:   CRC (XOR of bytes 0–5)
Total: 7 bytes
```

#### Parameter write packet (PC → STM32)
```
Byte 0:   0xEA  (magic)
Byte 1:   0x02  (type: param write)
Byte 2:   param_id (uint8, 0-based)
Bytes 3–6: value (float32, little-endian)
Byte 7:   CRC (XOR of bytes 0–6)
Total: 8 bytes
```

### 15.2 Project File Structure

After creating a project and building, the project folder contains:

```
C:\Epicyon\app\projects\<project_name>\
├── project.json          ← project metadata (board, model name, signals, params)
├── src\                  ← generated C source files from Embedded Coder
│   ├── <model_name>.c
│   ├── <model_name>.h
│   ├── epicyon_hal.c     ← generated hardware abstraction (board-specific)
│   └── ...
├── firmware.elf          ← compiled ELF (debug info included)
├── firmware.bin          ← flashable binary
└── build.log             ← last build compiler output
```

### 15.3 Board Definition Format

Epicyon supports multiple boards through JSON board definition files located in `C:\Epicyon\app\boards\`.

```json
{
  "id":        "nucleo-f411re",
  "mcu":       "STM32F411RETx",
  "clock_mhz": 96,
  "board":     "NUCLEO-F411RE",
  "channels": {
    "ADC":  [{"id": "adc0", "pin": "PA0"}, ...],
    "PWM":  [{"id": "pwm0", "pin": "PA8"}, ...],
    "DIN":  [{"id": "din0", "pin": "PC13"}, ...],
    "DOUT": [{"id": "dout0", "pin": "PA5"}, ...]
  }
}
```

To add support for a new board, create a new folder under `boards/` with a `board.json` following this format and a corresponding linker script and startup file.

### 15.4 Supported Hardware

| Board | MCU | Clock | Flash | RAM | Status |
|-------|-----|-------|-------|-----|--------|
| NUCLEO-F411RE | STM32F411RETx | 96 MHz | 512 KB | 128 KB | ✓ Supported |
| NUCLEO-G474RE | STM32G474RETx | 170 MHz | 512 KB | 128 KB | *Planned* |
| Custom PCB | STM32G474 | 170 MHz | 512 KB | 128 KB | *Planned* |

### 15.5 Keyboard Shortcuts (Browser UI)

| Key | Action |
|-----|--------|
| `B` | Trigger Build (on Build page) |
| `F` | Trigger Flash (on Build page) |
| `L` | Toggle Light / Dark mode |

### 15.6 Uninstalling

Delete the `C:\Epicyon` folder and the desktop shortcut. No registry entries are created by the installer.

---

*Epicyon User Guide — Version 0.1.0*  
*© 2026 Epicyon. inspire innovation.*
