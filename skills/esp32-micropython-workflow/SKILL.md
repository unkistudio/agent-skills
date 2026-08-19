---
name: esp32-micropython-workflow
description: >
  How to set up and use an ESP32 microcontroller from scratch using esptool and MicroPython.
  Use this skill whenever the user wants to work with an ESP32, ESP32-S3, or any ESP32-series chip —
  probing unknown devices, flashing firmware, writing custom MicroPython programs, and managing
  serial communication. Also use this when the user needs to read device specs (flash size, PSRAM,
  MAC, revision) from a connected ESP32 using esptool.
---

# ESP32 MicroPython Workflow

## Prerequisites — Serial Access

### Finding the port

Run one of these to discover which port the ESP32 is connected to:

```bash
ls /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
dmesg | grep tty | tail                   # last few kernel messages about serial devices
python3 -c "import serial.tools.list_ports; print([p.device for p in serial.tools.list_ports.comports()])"
```

Common ports: `/dev/ttyACM0` (USB Serial/JTAG on ESP32-S3/C3), `/dev/ttyUSB0` (USB-UART bridge like CP2102/CH340).

### Granting access

Two approaches, pick one:

**A) Add user to `dialout` group (permanent):**
```bash
sudo usermod -aG dialout "$USER"
# log out and back in after this
```

**B) Allow the current session only (quick & dirty, reverts on reboot):**
```bash
sudo chmod 666 /dev/ttyACM0
# or more generally:
sudo chmod 666 /dev/ttyACM* /dev/ttyUSB* 2>/dev/null
```

If the agent does not have sudo access, it should **ask the user to run the appropriate command** rather than attempting to run sudo itself. Tell the user exactly which command to paste.

## Setting Up the Environment

Create a Python venv with esptool and mpremote:

```bash
python3 -m venv venv
venv/bin/pip install esptool mpremote
```

Always use the venv's esptool, never a system install (to avoid version conflicts and permission issues):

```bash
venv/bin/esptool <args>
```

## Probing an Unknown ESP32

If no `ESP-SPECS.md` exists, probe the device to build one.

**Get chip info, MAC, flash & PSRAM size, security state:**

```bash
venv/bin/esptool --port /dev/ttyACM0 chip_id
venv/bin/esptool --port /dev/ttyACM0 flash_id
```

- `chip_id` → chip family, revision, MAC, feature flags (PSRAM, BT, etc.)
- `flash_id` → manufacturer, device ID, flash size (infer from hex IDs via esptool's lookup)

**Detect flash size programmatically:**

```bash
venv/bin/esptool --port /dev/ttyACM0 flash_id | grep -oP 'Detected flash size: \K\w+'
```

**Check security efuses:**

```bash
venv/bin/esptool --port /dev/ttyACM0 efuse_summary
```

**Common baud rates:** 921600 is stable for most ESP32-S3 boards; start high and fall back to 115200 if connection fails.

## Erasing Flash

Wipe the entire flash before flashing a new firmware to avoid corruption from leftover data:

```bash
venv/bin/esptool --port /dev/ttyACM0 erase-flash
```

For ESP32-S3/C3, also erase the NVS partition (non-volatile storage) if re-flashing without a full erase:

```bash
venv/bin/esptool --port /dev/ttyACM0 erase-region 0x9000 0x5000
```

**Important:** `erase-flash` takes several seconds and shows a progress bar. Do not disconnect or reset the board while erasing — this can brick the flash controller and require a low-level recovery.

## Flashing MicroPython

### Choose the right firmware

- Go to https://micropython.org/download/ and search for the exact chip (e.g., `ESP32_GENERIC_S3`)
- If the chip has **PSRAM**, use the `SPIRAM` or `SPIRAM_OCT` variant (e.g., `ESP32_GENERIC_S3-SPIRAM_OCT`); check the number of PSRAM pins (8-bit = OCT, 4-bit = regular SPIRAM)
- If flash is smaller than default (e.g., 4MB), use the `-FLASH_4M` variant

### Flash

```bash
venv/bin/esptool --chip <chip> --port /dev/ttyACM0 --baud 921600 \
  write-flash -z --flash-size <size> 0x0 firmware.bin
```

- `--chip`: auto-detected from connected chip, but be explicit (e.g., `esp32s3`)
- `--flash-size`: e.g., `16MB` — detected from `flash_id` output
- Address `0x0` is correct for MicroPython unified binaries (they contain the bootloader)

## Writing Custom Programs

MicroPython runs scripts from the filesystem. The boot script is `main.py`.

### Write a script

MicroPython modules differ from CPython. Common ones:
- `network` — WiFi (scan, connect, access point)
- `machine` — GPIO, I2C, SPI, ADC, PWM, deep sleep
- `bluetooth` — BLE (BLEWithAdvertisements on some versions)
- `time` — `sleep()`, `ticks_ms()`, `localtime()`
- `os` — filesystem operations

Important MicroPython gotchas:
- No nested f-strings (use `%` formatting or `str.format()`)
- F-strings with literal strings inside braces (`f"{'foo':5s}"`) may fail — use `%` formatting instead
- `socket` module is available but minimal
- Garbage collection is manual (`gc.collect()`) — PSRAM is not automatically managed by the GC
- Always deactivate WiFi after use to save power: `wlan.active(False)`

### Upload and run

```bash
# Upload a file
venv/bin/mpremote connect /dev/ttyACM0 fs cp local_file.py :remote_file.py

# Run once (prints to console)
venv/bin/mpremote connect /dev/ttyACM0 run script.py

# Open REPL for interactive use
venv/bin/mpremote connect /dev/ttyACM0 repl

# List files on device
venv/bin/mpremote connect /dev/ttyACM0 fs ls

# Remove a file
venv/bin/mpremote connect /dev/ttyACM0 fs rm main.py
```

If `mpremote` fails to connect, try resetting the board (press the RST button) and try again.

### Structure for a script that can be a program or a module

```python
def do_thing():
    ...
    # side-effect-free logic

def main():
    # CLI-like entry point with side effects

if __name__ == "__main__":
    main()
```

This lets the script run standalone (`mpremote run`) or be imported from REPL for repeated use.

## REPL Debugging Loop

1. Write script locally
2. `mpremote connect /dev/ttyACM0 run script.py` — test it
3. Edit and repeat

For interactive debugging, use `mpremote repl`, paste code directly, or import modules:

```
>>> import my_script
>>> my_script.do_thing()
```

## Full Workflow Summary

```
# 1. Create venv (once)
python3 -m venv venv
venv/bin/pip install esptool mpremote

# 2. Probe device (if unknown)
venv/bin/esptool --port /dev/ttyACM0 chip_id
venv/bin/esptool --port /dev/ttyACM0 flash_id

# 3. Download firmware from micropython.org/download/
curl -LO <firmware-url>

# 4. Flash
venv/bin/esptool --chip <chip> --port /dev/ttyACM0 --baud 921600 \
  write-flash -z --flash-size <size> 0x0 firmware.bin

# 5. Write program and upload
# (edit script.py)
venv/bin/mpremote connect /dev/ttyACM0 fs cp script.py :main.py
venv/bin/mpremote connect /dev/ttyACM0 run script.py
```
