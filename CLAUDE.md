# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automatic chicken coop door controller. ATmega328P (Arduino Pro Mini 16MHz). Controls servo-driven gate via rope wound on ~30×30mm drum. Open/close decisions based on light intensity (BH1750FVI) + time windows (DS1307 RTC). Sub-$20 BOM.

## Build & Flash Commands

```bash
pio run                          # build
pio run --target upload          # build + flash (requires FTDI adapter)
pio device monitor               # serial monitor at 9600 baud
pio run --target clean           # clean build artifacts
```

Board: `pro16MHzatmega328` (Arduino Pro Mini 16MHz). Upload protocol configured in `platformio.ini`.

**Flashing gotcha:** No USB on Pro Mini. Program via TTL-USB converter. Must manually short RST pin to GND at the right moment during upload — timing is finicky.

## Compile-Time Switches (`include/switches.hpp`)

Control all debug/logging features via `#define`:

| Switch | Effect |
|--------|--------|
| `DEBUG` | Serial debug prints |
| `PRINT` | Print date + light + gate state each loop |
| `ADJUST_TIME` | Sync RTC to compile time on next boot (disable after use) |
| `LOG_MOVES` | Log open/close events to EEPROM |
| `LOG_LIGHT` | Log light readings to EEPROM every 5 min |
| `PRINT_LOG_LIGHT` | Dump EEPROM light log on boot |
| `PRINT_LOG_MOVES` | Dump EEPROM move log on boot |
| `RESET_EEPROM_COUNTER` | Reset EEPROM log counter on next boot |

**Critical deployment checklist** (comment in main.cpp):
1. Reset EEPROM counter (`RESET_EEPROM_COUNTER` once, then disable)
2. Set RTC date (`ADJUST_TIME` once, then disable)
3. Manual reset required before upload (board limitation)

## Architecture

### Decision Logic (`include/Gate.hpp`)

Gate open/close decisions use two independent mechanisms:

- **Light-based** (`shouldOpen`/`shouldClose`): Requires ALL 5 light samples above/below threshold AND time within window (6:00–18:00). Prevents false triggers from brief shadows.
- **Absolute time override** (`shouldAbsoluteOpen`/`shouldAbsoluteClose`): Forces open after 08:00, forces close after 22:00, regardless of light. Overrides manual button clicks.

Thresholds in `sGate`: `lightOpen=8 lx`, `lightClose=3 lx`, `hourOpen=6`, `hourClose=18`, `absoluteHourOpen=8`, `absoluteHourClose=22`.

### Gate State Machine (`src/Gate.cpp`, `include/Gate.hpp`)

`sGate` struct tracks: `isOpening`, `isClosing`, `isOpened`, `isClosed`, `isSafetyStop`, `isOpenButtonClicked`, `isCloseButtonClicked`.

Safety: if movement exceeds `maxMovingTime` (21s), `safetyStop()` detaches servo and sets `isSafetyStop`. Safety stop clears only on manual button press.

End-of-travel detected via hardware limit switches on pins 2/3 (interrupt-driven ISRs).

### LED Signals

| LED | Pattern | Meaning |
|-----|---------|---------|
| Green solid | — | Gate fully open |
| Green blinking | — | Gate moving |
| Red solid | — | Gate fully closed |
| Red blinking | — | Safety stop triggered |
| Yellow blinking | — | Battery below 2.05V |

### Key Files

- `include/switches.hpp` — all feature flags (edit here for debug/logging)
- `include/Gate.hpp` — decision logic + thresholds (inline methods)
- `include/constants.hpp` — timing/voltage constants
- `include/pins.hpp` — all pin assignments
- `src/main.cpp` — setup/loop, manual button handling
- `src/functions.cpp` — setup helpers, ISRs, EEPROM read/write, RTC drift compensation

### RTC Drift Compensation

Runs at midnight: delays `secondsDriftOffset` seconds (4s), then subtracts that offset from RTC. Compensates known DS1307 drift. Adjust `secondsDriftOffset` in `constants.hpp` if drift changes.

### EEPROM Layout

Counter byte at address 0. Log entries start at address 2. Circular buffer — wraps when full. Two log formats (mutually exclusive via switches): `sLog` (move events) or `LogLight` (light readings, stored as lux/2 capped at 62 lx).

### Hardware Notes

**Servo:** MG995 continuous rotation (360°). Door is light, rope winds on ~30×30mm drum. MG995 chosen because it stalls under load and holds position — acts as mechanical brake keeping door up.

**MCU:** Arduino Pro Mini 328 (5V/16MHz). No USB, poor programmability (manual RST trick). Upside: 5V logic avoids level shifters and boost converters for sensors/servo.

**Power:** microUSB-to-VCC/GND adapter board + bulk capacitor (~100µF, TBD — verify on board) for supply stabilization.

**RTC:** DS1307 — known poor choice; significant drift. That's why `compensateRtcDrift()` exists (subtracts `secondsDriftOffset`=4s at midnight). Verify actual drift and tune `secondsDriftOffset` in `constants.hpp` if needed.

**Light sensor:** GY-302 (BH1750FVI). Works well for dusk/dawn detection in this application. Time windows (`hourOpen`/`hourClose`) intentionally prevent clouds, storms, and other anomalies from closing the door prematurely.

**Limit switches:** Microswitches as end-of-travel stops (pins 2/3, interrupt-driven). Plus 2 manual push-buttons for open/close override.

**LEDs:** Standard 3mm (red/yellow/green, 20mA rated). Large series resistors chosen to limit current to <1mA for power saving.

**Enclosure:** Electronics box placed in dark location — relevant because ambient light on the sensor must come only from outside, not from the box interior.

### Power Notes

CPU clock prescaler code exists but is commented out (`CLKPR` in `setup()`). 1MHz mode can be re-enabled by uncommenting. Watchdog timer (`wdt_enable`) also commented out.
