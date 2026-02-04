# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Viam module for HX711 load cell amplifiers. It wraps the HX711 hardware sensor as a Viam sensor component, providing raw ADC readings and optionally calibrated weight (kg) and force (N) measurements.

**Module ID:** `chris:hx711-loadcell`
**Model:** `chris:sensor:hx711`
**API:** `rdk:component:sensor`

## Architecture

### Core Components

**src/main.py** - Main module implementation
- `HX711Sensor` class: Viam sensor component wrapper
- Handles configuration, reconfiguration, and reading lifecycle
- Applies calibration formula: `weight_kg = (slope × raw_ADC) + offset`
- Converts weight to force: `force_N = weight_kg × 9.81`

**src/hx711.py** - Low-level HX711 driver (bundled third-party library)
- Handles GPIO communication with HX711 chip
- Provides raw ADC reading, taring, and averaging functions
- Thread-safe with readLock for concurrent access

### Key Architectural Patterns

**Two Operating Modes:**

1. **Uncalibrated Mode** (no calibration params):
   - Auto-tares on initialization (zeroes the sensor)
   - Returns relative ADC counts only
   - Uses `get_value()` which subtracts tare offset

2. **Calibrated Mode** (with slope + offset):
   - Does NOT tare (needs raw ADC for calibration formula)
   - Returns raw_value, weight_kg, and force_N
   - Must use raw ADC values for accurate calibration

**Critical Design Decision:**
The module conditionally tares based on calibration presence (main.py:84-85):
```python
if self.calibration_slope is None or self.calibration_offset is None:
    self.hx.tare()
```

This prevents taring from interfering with calibration formulas that expect raw ADC values.

## Configuration

Required parameters:
- `data_pin`: GPIO pin for HX711 DOUT
- `clock_pin`: GPIO pin for HX711 PD_SCK

Optional parameters:
- `samples`: Number of samples to average (default: 1)
- `calibration_slope`: Linear calibration slope (kg per ADC count)
- `calibration_offset`: Linear calibration offset (kg)
- `output_unit`: Preferred output unit ("raw", "kg", "N")

## Development Commands

### Testing on Hardware
```bash
# Install dependencies
./setup.sh

# Run module directly (requires actual HX711 hardware on Raspberry Pi)
python3 src/main.py
```

### Module Lifecycle
- **First run:** `setup.sh` installs viam-sdk>=0.27.0
- **Entry point:** `src/main.py` registers and starts the module
- **Reconfiguration:** Safe to reconfigure pins/calibration at runtime

## Known Issues & Code Review

### CRITICAL BUGS TO FIX

1. **Calibration Reading Bug (main.py:91)**
   - Currently uses `self.hx.get_value()` which subtracts offset
   - Should use `self.hx.read_median()` for true raw ADC when calibrated
   - Impact: All calibrated readings are off by `1 × calibration_slope`

2. **Default OFFSET Bug (hx711.py:27)**
   - `self.OFFSET = 1` should be `self.OFFSET = 0`
   - Causes 1-unit offset in all readings before taring

3. **Syntax Errors (hx711.py:362, 419)**
   - `get_reference_unit()`: missing `self.` prefix
   - `hx711_add_event_detect()`: uses `self` instead of `hx711_instance` parameter
   - Both will crash with NameError if called

4. **Thread Safety (main.py:74-91)**
   - Race condition: `reconfigure()` can invalidate `self.hx` while `get_readings()` is using it
   - Need lock or atomic reference swap

5. **Deadlock Risk (hx711.py:108-127)**
   - `readLock.acquire()` without try/finally means lock never released on exception
   - `while not self.is_ready()` has no timeout, can hang forever if hardware fails

6. **GPIO Resource Leak (main.py:109-111)**
   - `close()` doesn't call `GPIO.cleanup()`
   - Causes warnings on restart, potential conflicts

### MEDIUM PRIORITY

7. **No config validation** - Missing data_pin/clock_pin causes cryptic KeyError
8. **Bare except clause (main.py:75-78)** - Silently swallows all errors including KeyboardInterrupt
9. **Wrong error message (hx711.py:355)** - Says "_A()" in set_reference_unit_B()
10. **No calibration validation** - calibration_slope could be 0

### MINOR ISSUES

- Unreachable `return` after `raise` (hx711.py:347, 356)
- Float division where int expected (hx711.py:219)
- Error returned as dict instead of exception (main.py:106-107)
- Invalid gain values silently ignored (hx711.py:50-56)

## Testing Calibration

To calibrate a load cell:
1. Read raw ADC with no load → `raw_empty`
2. Read raw ADC with known weight (e.g., 10kg) → `raw_loaded`
3. Calculate:
   - `slope = (weight_kg - 0) / (raw_loaded - raw_empty)`
   - `offset = 0 - (slope × raw_empty)`

Example:
```python
# raw_empty = 8432100
# raw_loaded = 8556230 (with 10kg)
slope = 10 / (8556230 - 8432100) = 0.0000805
offset = 0 - (0.0000805 × 8432100) = -678.78
```

## Important Implementation Notes

### When Modifying Reading Logic
- Uncalibrated mode needs tared values: use `get_value()` AFTER calling `tare()`
- Calibrated mode needs raw ADC: use `read_median()` directly, NO taring
- The distinction is critical for accuracy

### GPIO Pin Management
- Module uses RPi.GPIO with BCM numbering
- Pins must be valid GPIO numbers for Raspberry Pi
- Cleanup is required to prevent resource conflicts

### HX711 Hardware Timing
- HX711 DOUT goes low when data is ready (~10-80 Hz depending on rate setting)
- Gain is set by number of clock pulses after 24 data bits (128=1 pulse, 64=3 pulses, 32=2 pulses)
- Power down requires PD_SCK high for >60µs

## Repository Configuration

- **Origin:** https://github.com/ashitaka1/hx711-loadcell-module (fork)
- **Upstream:** https://github.com/c-j-payne/hx711-loadcell-module (original)
- **License:** MIT
