# CLI Controller Guide

The FPGA UART Serial Controller is a terminal application for programming and running experiments through the FPGA interface. It provides a persistent REPL with tab-completion, a live connection status panel, and a full TX packet log.

---

## Installation

### Dependencies

```bash
pip install rich prompt_toolkit pyserial
```

| Package | Role |
|---|---|
| `rich` | Terminal rendering — panels, tables, colour |
| `prompt_toolkit` | Interactive REPL with tab-completion |
| `pyserial` | Serial port communication |

The CLI also requires `backend.py` (`FPGABackendLink`) to be present in the same directory.

---

## Launching

```bash
python cli.py [--port <port>] [--baud <baud>]
```

### Options

| Flag | Short | Default | Description |
|---|---|---|---|
| `--port` | `-p` | _(none)_ | Serial port to connect to on startup |
| `--baud` | `-b` | `115200` | Baud rate |

### Examples

```bash
# Launch and connect immediately
python cli.py --port /dev/ttyUSB0 --baud 115200

# Windows
python cli.py -p COM3

# Launch without connecting (connect manually inside the REPL)
python cli.py
```

If `--port` is omitted the application starts disconnected. Use the `connect` command inside the REPL to establish a connection.

---

## Interface

On launch the application displays an ASCII banner followed by the REPL prompt:

```
[FPGA] ›                                              disconnected
```

The right-hand side of the prompt shows the active port name when connected, or `disconnected` in red. The prompt itself stays open between commands — type the next command and press Enter.

**Tab-completion** is available on all commands and register names.

---

## Command Reference

### Connection

#### `connect [<port>] [<baud>]`

Opens a serial connection to the FPGA.

```
[FPGA] › connect /dev/ttyUSB0 115200
[FPGA] › connect COM3
[FPGA] › connect
```

If no port is specified the application selects the first available port automatically. If no baud rate is specified it defaults to `115200`.

---

#### `disconnect`

Closes the active serial connection.

```
[FPGA] › disconnect
```

---

#### `ports`

Lists all detected serial ports with device name, description, USB VID, and PID.

```
[FPGA] › ports
```

Useful for identifying the correct port before connecting.

---

### Register Programming

Registers are set locally first, then pushed to the FPGA explicitly. This two-step model lets you stage a full configuration before sending anything over the wire.

#### `set <register> <value>`

Updates a register's local value. Does **not** transmit anything. Values can be decimal or `0x`-prefixed hex.

```
[FPGA] › set prbs_cross 0xA5
[FPGA] › set prbs_enable A5
[FPGA] › set watchdog 16
[FPGA] › set duration 100
[FPGA] › set toggle 50
```

| Register | FPGA Address | Description | Range |
|---|---|---|---|
| `prbs_cross` | `0x02` | PRBS cross LFSR seed | `0x00`–`0xFF` |
| `prbs_enable` | `0x03` | PRBS enable LFSR seed | `0x00`–`0xFF` |
| `watchdog` | `0x04` | Watchdog PGA maximum count | `0`–`255` |
| `duration` | `0x05` | Experiment duration (µs) | `0`–`255` |
| `toggle` | `0x06` | RX/TX toggle period (µs) | `0`–`255` |

> **PRBS seed warning:** Avoid setting `prbs_cross` or `prbs_enable` to `0x00`. The all-zero state has no feedback path and the LFSR will lock up permanently.

---

#### `program [<register>]`

Transmits the current register value(s) to the FPGA over UART.

```bash
# Push all registers at once
[FPGA] › program

# Push a single register
[FPGA] › program prbs_cross
[FPGA] › program duration
```

Each `program` call sends a two-byte packet `[addr, data]` per register. The FPGA processes them in sequence. A connection must be open before calling `program`.

---

### Experiment Control

#### `run`

Starts the experiment. Sends `[0x01, 0x00]` to the FPGA.

```
[FPGA] › run
```

The FPGA deasserts `experiment_enb`, releasing all submodules (PRBS generators, watchdog, RX/TX controller).

---

#### `stop`

Stops the experiment. Sends `[0x01, 0xFF]` to the FPGA.

```
[FPGA] › stop
```

---

### Diagnostics

#### `status`

Displays a panel showing the current connection state and all staged register values.

```
[FPGA] › status
```

Output example:

```
╭─────────────── ◈  FPGA STATUS ────────────────╮
│  CONNECTED  /dev/ttyUSB0  @  115200 baud       │
│                                                │
│  Register                  Value    Addr       │
│  PRBS Cross Seed           0xA5     0x02       │
│  PRBS Enable Seed          0x3C     0x03       │
│  Watchdog Max              16       0x04       │
│  Experiment Duration (µs)  100      0x05       │
│  Toggle Period (µs)        50       0x06       │
╰────────────────────────────────────────────────╯
```

---

#### `log`

Prints a table of all UART packets transmitted in the current session, with sequence number, timestamp, address, register label, and data byte.

```
[FPGA] › log
```

#### `log clear`

Clears the TX packet log.

```
[FPGA] › log clear
```

---

#### `help`

Prints the full command reference inside the terminal.

```
[FPGA] › help
```

---

#### `exit` / `quit`

Closes the serial connection and exits the application. `Ctrl+C` and `Ctrl+D` are also handled gracefully.

```
[FPGA] › exit
```

---

## Typical Session

```bash
# 1. Launch and connect
python cli.py --port /dev/ttyUSB0

# 2. Stage register values
[FPGA] › set prbs_cross 0xA5
[FPGA] › set prbs_enable 0x3C
[FPGA] › set duration 100
[FPGA] › set toggle 50
[FPGA] › set watchdog 16

# 3. Push all registers to FPGA
[FPGA] › program

# 4. Verify staged values
[FPGA] › status

# 5. Start experiment
[FPGA] › run

# 6. Check what was sent
[FPGA] › log

# 7. Stop when done
[FPGA] › stop
```

---

## Value Format Reference

All numeric arguments accept either decimal or `0x`-prefixed hex. The bare hex form (without `0x`) is also accepted.

| Input | Interpreted as |
|---|---|
| `100` | decimal 100 |
| `0x64` | hex → decimal 100 |
| `64` | hex → decimal 100 (`int(..., 0)` base inference) |
| `0xFF` | 255 |

All values must be in the range `0`–`255` (8-bit). The CLI will reject out-of-range values before transmitting.

---

## Error Handling

The CLI distinguishes three output levels:

| Symbol | Colour | Meaning |
|---|---|---|
| `✔` | Green | Operation succeeded |
| `✘` | Red | Operation failed — message gives the reason |
| `!` | Amber | Warning — action may need attention |
| `→` | Amber | Informational — no action required |

If a `program` or `run`/`stop` call fails, the error reported by the backend (`backend.state.error`) is printed inline. The connection remains open — retry after resolving the issue.