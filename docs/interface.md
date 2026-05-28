# ASIC Interface Guide

This document describes how to interface with the ASIC using the FPGA controller (`toplevel`). It covers the hardware connections, UART programming protocol, configurable parameters, and test utilities available on the Arty S7 board.

---

## Overview

The FPGA acts as a controller bridge between a host PC and the ASIC under test. Communication flows over UART from the host, which is decoded into register writes and control signals that drive the ASIC's RX/TX interfaces, clock, PRBS stimulus, and I2C-controlled PGA watchdog.

```
Host PC  ──UART──►  FPGA (toplevel)  ──────►  ASIC
                         │
                         ├── PRBS generators (cross, enable)
                         ├── Watchdog PGA controller (I2C)
                         ├── RX/TX cycle controller
                         └── Test pins (CLK, PRBS)
```

---

## Hardware Connections

### FPGA Pin Map

| Signal | Direction | Description |
|---|---|---|
| `clk` | Input | 100 MHz system clock |
| `rstb` | Input | Active-low global reset |
| `uart_rx_i` | Input | UART RX from host PC |
| `testen_toggle_sw` | Input | Hardware toggle for `testen` override |
| `afeen_toggle_sw` | Input | Hardware toggle for `afeen` override |
| `test_clock_toggle_sw` | Input | Enables 100 MHz test clock output |
| `test_prbs_toggle_sw` | Input | Enables 50 MHz PRBS test output |
| `prbs_cross_out` | Output | PRBS cross signal to ASIC |
| `prbs_enable_out` | Output | PRBS enable signal to ASIC |
| `i2c_cse_n` | Output | I2C chip select (active-low) |
| `i2c_sda` | Output | I2C data |
| `i2c_scl` | Output | I2C clock |
| `chip_rstb` | Output | ASIC reset (active-low) |
| `rxtxb` | Output | RX/TX direction select |
| `rxstate` | Output | RX state indicator |
| `testen` | Output | Test enable to ASIC |
| `afeen` | Output | AFE enable to ASIC |
| `cseb` | Output | Chip select enable (active-low) |
| `clkafe` | Output | AFE clock |
| `experiment_ongoing` | Output | High while experiment is running |
| `experiment_done` | Output | Pulses high when experiment completes |
| `test_clk100mhz` | Output | JC1 — gated 100 MHz debug clock |
| `test_prbs50mhz` | Output | JC2 — 50 MHz PRBS debug output |

### JC PMOD Test Pins (Arty S7)

The two debug outputs are routed to the JC PMOD header. Add the following constraints to your XDC file:

```tcl
# test_clk100mhz -> JC1
set_property -dict { PACKAGE_PIN U15  IOSTANDARD LVCMOS33 } [get_ports { test_clk100mhz }];
# test_prbs50mhz -> JC2
set_property -dict { PACKAGE_PIN V16  IOSTANDARD LVCMOS33 } [get_ports { test_prbs50mhz }];

# Relax timing on debug outputs
set_false_path -to [get_ports { test_clk100mhz }]
set_false_path -to [get_ports { test_prbs50mhz }]
```

> **Note:** `test_clk100mhz` is a gated clock routed through fabric. It is suitable for oscilloscope probing but is not a clean clock source for downstream logic.

---

## UART Protocol

### Physical Layer

Configure your serial terminal or script with the following settings (baud rate is set by the `uart_rx` submodule — confirm with that module's parameters):

| Parameter | Value |
|---|---|
| Data bits | 8 |
| Stop bits | 1 |
| Parity | None |
| Flow control | None |

### Packet Format

Every transaction is exactly **two bytes**:

```
┌────────────┬────────────┐
│  BYTE 0    │  BYTE 1    │
│  Address   │  Data      │
└────────────┴────────────┘
```

The FPGA FSM waits for `BYTE 0` (address), then `BYTE 1` (data), then returns to waiting. There is no acknowledgement byte — the host must pace transmissions appropriately.

### Address Map

| Address | Name | Description |
|---|---|---|
| `0x01` | `RUN_EXPERIMENT` | Start or stop the experiment |
| `0x02` | `PRBS_CROSS` | Load seed into cross PRBS LFSR |
| `0x03` | `PRBS_ENABLE` | Load seed into enable PRBS LFSR |
| `0x04` | `WATCHDOG_PROGRAM` | Set watchdog PGA maximum count |
| `0x05` | `EXPERIMENT_PROG` | Set experiment duration (µs) |
| `0x06` | `TOGGLE_PROG` | Set RX/TX toggle period (µs) |

---

## Register Reference

### `0x01` — RUN_EXPERIMENT

Controls the global experiment enable. This is an **active-low** enable internally; all submodules (PRBS generators, watchdog, RX/TX controller) gate on this signal.

| Data byte | Effect |
|---|---|
| `0x00` | Start experiment (`experiment_enb` deasserted) |
| Any non-zero | Stop experiment (`experiment_enb` asserted) |

**Example — start experiment:**
```
TX: 0x01 0x00
```

**Example — stop experiment:**
```
TX: 0x01 0xFF
```

---

### `0x02` — PRBS_CROSS

Programs the seed of the **cross** PRBS LFSR. The write is captured on the cycle the second byte is received, while the experiment is stopped.

| Data byte | Effect |
|---|---|
| `0x01`–`0xFF` | Load as LFSR seed |
| `0x00` | Avoid — all-zero seed locks up the LFSR |

**Example — load seed 0xA5:**
```
TX: 0x02 0xA5
```

---

### `0x03` — PRBS_ENABLE

Programs the seed of the **enable** PRBS LFSR. Same behavior as `0x02`.

**Example — load seed 0x3C:**
```
TX: 0x03 0x3C
```

---

### `0x04` — WATCHDOG_PROGRAM

Sets the maximum count threshold for the watchdog PGA controller. Refer to the `watchdog_pga_controller` module documentation for the count-to-time conversion based on clock frequency.

**Example:**
```
TX: 0x04 0x10
```

---

### `0x05` — EXPERIMENT_PROG

Programs the total experiment duration in microseconds (interpreted by `rx_tx_cycle_controller`). Resolution and maximum value depend on the internal counter width — refer to the `rx_tx_cycle_controller` module.

**Example — 100 µs duration:**
```
TX: 0x05 0x64
```

---

### `0x06` — TOGGLE_PROG

Programs the RX/TX toggle period in microseconds. Controls how frequently the controller alternates between RX and TX modes during the experiment.

**Example — 50 µs toggle period:**
```
TX: 0x06 0x32
```

---

## PRBS Generators

Two independent PRBS generators drive the ASIC:

| Instance | Output pin | Seed address | Role |
|---|---|---|---|
| `prbs_cross` | `prbs_cross_out` | `0x02` | Cross-channel stimulus |
| `prbs_enable` | `prbs_enable_out` | `0x03` | Enable-channel stimulus |

Both use an **8-bit Galois LFSR** with polynomial `x⁸ + x⁶ + x⁵ + x⁴ + 1` (maximal-length, 255-state sequence), clocked at an effective **50 MHz** rate via a clock-enable divider.

Both generators run only while the experiment is active (`experiment_enb` low). They hold their last state when the experiment is stopped.

> **Seed warning:** Never program a seed of `0x00`. The all-zero state has no feedback path and the LFSR will output a permanent zero.

---

## RX/TX Cycle Controller

The `rx_tx_cycle_controller` manages the timed alternation between RX and TX modes and drives the following ASIC control signals:

| Signal | Description |
|---|---|
| `chip_rstb` | ASIC reset, released at experiment start |
| `rxtxb` | RX=0 / TX=1 mode select |
| `rxstate` | Current RX state output |
| `testen` | Test enable (also overridable via `testen_toggle_sw`) |
| `afeen` | AFE enable (also overridable via `afeen_toggle_sw`) |
| `cseb` | Chip select (active-low) |
| `clkafe` | AFE clock |

The `testen_toggle_sw` and `afeen_toggle_sw` board switches provide a **hardware override** for `testen` and `afeen` independent of the UART-programmed experiment state. Refer to the `rx_tx_cycle_controller` module for override priority.

### Experiment Status Outputs

| Signal | Behavior |
|---|---|
| `experiment_ongoing` | High for the duration of an active experiment |
| `experiment_done` | Goes high when the experiment completes |

---

## Watchdog PGA Controller

The watchdog monitors experiment progress and communicates with an external PGA over I2C.

| Signal | Description |
|---|---|
| `i2c_cse_n` | I2C chip select (active-low) |
| `i2c_sda` | I2C data line |
| `i2c_scl` | I2C clock line |

The watchdog threshold is programmed via address `0x04`. It is active only while the experiment is running.

---

## Test Utilities

### Test Clock (`test_clk100mhz`)

Enabled by `test_clock_toggle_sw`. Outputs a gated copy of the 100 MHz system clock on JC1.

```
test_clk100mhz = test_clock_toggle_sw & clk
```

Use this to verify the FPGA is alive and clocking correctly with a scope or logic analyzer.

### Test PRBS (`test_prbs50mhz`)

Enabled by `test_prbs_toggle_sw`. Outputs a 50 MHz PRBS sequence on JC2, using a **hardcoded seed of 42** (`0x2A`).

The same LFSR polynomial is used as the experiment PRBS generators (`x⁸ + x⁶ + x⁵ + x⁴ + 1`).

**Behavior by switch state:**

| `test_prbs_toggle_sw` | Behavior |
|---|---|
| Rising edge (0→1) | Seed `0x2A` is loaded into the LFSR (one-shot) |
| High | LFSR runs freely, output active on JC2 |
| Low | LFSR paused, output held |

This output is independent of the experiment state and can be used at any time for signal integrity checks or scope triggering.

---

## Typical Programming Sequence

```
1.  Assert reset (rstb low), then release.
2.  Program PRBS seeds:
        TX: 0x02 <cross_seed>
        TX: 0x03 <enable_seed>
3.  Program experiment duration:
        TX: 0x05 <duration_us>
4.  Program toggle period:
        TX: 0x06 <toggle_period_us>
5.  Program watchdog threshold:
        TX: 0x04 <watchdog_max>
6.  Start experiment:
        TX: 0x01 0x00
7.  Poll or monitor experiment_ongoing / experiment_done.
8.  Stop experiment (optional early stop):
        TX: 0x01 0xFF
```

---

## Submodule Index

| Module | Instance | Description |
|---|---|---|
| `uart_rx` | `uart0` | UART byte receiver |
| `address_decoder` | `decoder0` | Maps received address to WE strobes |
| `prbs_lfsr` | `prbs_cross` | Cross PRBS generator |
| `prbs_lfsr` | `prbs_enable` | Enable PRBS generator |
| `prbs_lfsr` | `test_prbs` | Standalone test PRBS (seed = 42) |
| `watchdog_pga_controller` | `watchdog0` | I2C PGA watchdog |
| `rx_tx_cycle_controller` | `rx_tx0` | Timed RX/TX sequencer |