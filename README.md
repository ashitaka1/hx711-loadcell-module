# HX711 Load Cell Sensor Module

A Viam module for HX711 load cell amplifiers, providing calibrated force and weight measurements.

## Features

- ✅ Raw ADC readings from HX711
- ✅ Calibrated weight measurements (kg)
- ✅ Calibrated force measurements (N)
- ✅ Configurable sampling rates
- ✅ Multi-sensor support
- ✅ Auto-taring for uncalibrated sensors

## Configuration

### Basic Configuration
```json
{
  "data_pin": 5,
  "clock_pin": 6,
  "samples": 10
}
```

### With Calibration
```json
{
  "data_pin": 5,
  "clock_pin": 6,
  "samples": 10,
  "calibration_slope": 0.000123,
  "calibration_offset": -0.5
}
```

**Note:** Pin numbers use BCM/GPIO logical numbering. For example, GPIO 5 (BCM) is physical pin 29 on a Raspberry Pi.

## Returns

- `raw_value`: Raw ADC reading
- `weight_kg`: Calibrated weight in kilograms (if calibrated)
- `force_N`: Calibrated force in Newtons (if calibrated)

## Example
```python
from viam.components.sensor import Sensor

loadcell = Sensor.from_robot(robot, "loadcell1")
readings = await loadcell.get_readings()
force = readings["force_N"]
```
