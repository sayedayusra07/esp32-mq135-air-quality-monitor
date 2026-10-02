# Air Quality Monitoring System (ESP32 + MQ-135)

A low-cost, Wi-Fi-connected air quality monitor built around an ESP32 and an MQ-135 gas
sensor. The ESP32 reads the sensor's analog output, converts the raw reading into a
sensor resistance, normalizes it against a clean-air calibration value, and estimates
the relative concentration of gases such as carbon dioxide (CO2) and ammonia (NH3).
The results are pushed to ThingSpeak so they can be viewed and logged remotely.

---

## Overview

Cheap metal-oxide gas sensors like the MQ-135 change their internal resistance when
exposed to different gases. That resistance change shows up as a change in the sensor's
output voltage, which a microcontroller can measure.

This project turns that behavior into a small IoT device:

1. The MQ-135 senses the surrounding air and outputs an analog voltage.
2. The ESP32 samples that voltage on its ADC.
3. Firmware converts the sample into a voltage, then into a sensor resistance (`Rs`).
4. `Rs` is divided by `Ro`, the resistance measured in clean air during calibration,
   giving a ratio that is largely independent of the specific sensor unit.
5. The ratio is fed into an empirical curve to produce an estimated gas concentration.
6. The ESP32 publishes the values to ThingSpeak over Wi-Fi.

The system is intended as an educational / prototype build. It reports *relative*
changes in air quality, not laboratory-grade gas concentrations.

---

## Features

- Low-cost hardware: a single ESP32 plus one MQ-135 module.
- Analog gas sensing with on-board ADC sampling.
- Sensor resistance calculation from the raw reading.
- Clean-air calibration stored as `Ro`.
- Estimates for CO2 and NH3 from the `Rs/Ro` ratio.
- Wi-Fi upload to ThingSpeak for remote viewing.
- Historical data logging and graphing in the ThingSpeak channel.
- Fully scriptable in MicroPython — easy to modify or extend.

---

## Hardware Requirements

| Item | Notes |
|------|-------|
| ESP32 development board | Any common DevKit variant works |
| MQ-135 gas sensor module | Must expose an analog (AOUT) pin |
| Resistors for a voltage divider | See the Wiring section |
| Breadboard | For prototyping |
| Jumper wires | For connections |
| 5 V supply | The MQ-135 heater needs 5 V |
| USB cable | For programming and power |

Notes on power:
- The MQ-135 module should be powered from 5 V, which most ESP32 boards supply on the
  `VIN` / `5V` pin when powered over USB.
- The ESP32's ADC input must not exceed its allowed range, so the sensor output is
  scaled down with a divider before it reaches the ADC pin.

---

## Software Requirements

| Item | Notes |
|------|-------|
| MicroPython firmware | Flashed onto the ESP32 |
| Thonny IDE | Used to write files and run scripts on the board |
| ThingSpeak account | Free tier is sufficient |
| Wi-Fi network | 2.4 GHz (the ESP32 does not support 5 GHz-only networks) |

---

## Wiring

Connect the MQ-135 module to the ESP32 as follows.

| MQ-135 Pin | Connects To |
|------------|-------------|
| VCC | 5 V / VIN on the ESP32 |
| GND | GND on the ESP32 |
| AOUT | Voltage divider, whose midpoint goes to GPIO36 |
| DOUT | Not used |

The analog output is sampled on **GPIO36** (an ADC1 pin, which is safe to use while
Wi-Fi is active — ADC2 pins are not).

### Voltage divider

The sensor output can swing higher than the ESP32's ADC comfortably accepts, so it is
scaled down first. Two resistors form the divider:

```
   MQ-135 AOUT
        |
        R1
        |
        +---------> GPIO36 (ADC input)
        |
        R2
        |
       GND
```

The output seen by the ADC is:

```
Vadc = Vout * (R2 / (R1 + R2))
```

Pick R1 and R2 so that the highest expected sensor output still maps below the ADC's
maximum input. A ratio around 2:1 (for example R1 = 10 kOhm, R2 = 10 kOhm, or
R1 = 20 kOhm, R2 = 10 kOhm) is a reasonable starting point.

> The exact resistor values used in the original physical build are not recorded, so
> the values above are a recommended starting point rather than a verified copy of the
> original wiring. Confirm your divider ratio with a multimeter before relying on the
> absolute voltage values.

---

## Project Structure

```
air-quality-monitor/
|
|-- README.md
|-- src/
|   |-- main.py          # main loop: read, compute, upload
|   |-- calibrate.py     # measures Ro in clean air
|   |-- config.py        # Wi-Fi and ThingSpeak credentials
|-- docs/
|   |-- calibration.md   # notes on the calibration procedure
```
---

## How It Works

### 1. ADC reading

The ESP32 ADC is 12-bit, so a sample is reported on a scale of:

```
0 .. 4095
```

### 2. ADC value to voltage

Convert the raw sample to a voltage at the ADC pin:

```
Vpin = (adc_value / 4095.0) * Vref
```

`Vref` should reflect your attenuation setting (roughly 1.1 V, 1.5 V, 2.2 V, or
3.3 V for the standard ESP32 attenuation options). Undo the divider to recover the
sensor output:

```
Vout = Vpin * (R1 + R2) / R2
```

Note: the ESP32 ADC is not perfectly linear. For any use where absolute accuracy
matters, apply a per-board calibration curve or a lookup correction.

### 3. Sensor resistance

With a load resistor `RL` in the sensor's internal circuit, the sensing resistance is:

```
Rs = RL * (Vc - Vout) / Vout
```

where:

- `Rs`  = sensor resistance
- `RL`  = load resistance (the project documentation used 20 kOhm)
- `Vc`  = sensor supply voltage (5 V)
- `Vout` = sensor output voltage

### 4. Calibration

Every MQ-135 unit differs, and its absolute resistance drifts with temperature,
humidity, and age. To make readings comparable, measure `Rs` in clean air after the
sensor has warmed up, and store that value as `Ro`. Then work with the ratio:

```
ratio = Rs / Ro
```

Calibration procedure:

- Power the sensor and let it warm up (see Troubleshooting — this takes time).
- Place it in clean, still air, away from exhaust, solvents, or cooking fumes.
- Take many readings over several minutes and average the resulting `Rs` values.
- Save that average as `Ro` in your configuration.

### 5. Gas estimation

The relationship between `Rs/Ro` and concentration is nonlinear and follows an
empirical curve for each gas:

```
concentration = a * (Rs / Ro) ** b
```

The constants `a` and `b` come from the sensor's datasheet curves and differ per gas.
Because the MQ-135 responds to many gases at once, treat the CO2 and NH3 outputs as
relative estimates, not exact concentrations.

### 6. Upload to ThingSpeak

The ESP32 joins Wi-Fi and sends the computed values to a ThingSpeak channel using the
Write API key. A suggested field layout:

| Field | Value |
|-------|-------|
| Field 1 | Raw ADC value |
| Field 2 | Sensor voltage |
| Field 3 | Sensor resistance (`Rs`) |
| Field 4 | (spare) |
| Field 5 | Estimated CO2 |
| Field 6 | Estimated NH3 |

ThingSpeak then graphs each field and keeps a history you can review later.

---

## Installation and Setup

### Step 1 — Flash MicroPython

Install MicroPython firmware on the ESP32 using esptool or the Thonny firmware
installer, then reconnect the board.

### Step 2 — Prepare the IDE

Open Thonny, choose the ESP32 MicroPython interpreter, and confirm you can see the
board's filesystem.

### Step 3 — Wire the sensor

Follow the Wiring section. Double-check the 5 V and GND connections and confirm the
divider output before connecting it to GPIO36.

### Step 4 — Add your configuration

Create `src/config.py` with your own values:

```python
WIFI_SSID = "YOUR_WIFI_NAME"
WIFI_PASSWORD = "YOUR_WIFI_PASSWORD"
THINGSPEAK_API_KEY = "YOUR_WRITE_API_KEY"
THINGSPEAK_URL = "http://api.thingspeak.com/update"

ADC_PIN = 36
ADC_MAX = 4095
VREF = 3.3
R1 = 10000
R2 = 10000
RL = 20000
VC = 5.0
RO = 0.0   # filled in after calibration
```

Never commit real credentials. For a public repository, ship a `config.example.py`
with placeholders instead.

### Step 5 — Calibrate

Run `src/calibrate.py` with the sensor in clean air, let it warm up, and read the
reported `Ro`. Copy that value into `RO` in your config.

### Step 6 — Run the main program

Run `src/main.py`. It loops: read the sensor, compute voltage and resistance, compute
the ratio, estimate gases, upload, and wait before the next cycle (ThingSpeak's free
tier limits you to roughly one update every 15 seconds).

---

## Usage

Run from Thonny or copy the files to the board so it starts on boot:

- In Thonny: open `src/main.py` and click Run.
- For standalone operation: save the scripts to the board as `main.py` (MicroPython
  runs `main.py` automatically at power-up) plus any helper modules.
- Watch the ThingSpeak channel to see live and historical values.
- Adjust the update interval, thresholds, or gas constants directly in the source.

---

## Testing

1. **Open air** — place the sensor outdoors or in a well-ventilated room and record a
   baseline. Readings should be relatively low and stable.
2. **Enclosed space** — put the sensor in a closed container and watch the readings
   rise as gases accumulate.
3. **Near a source** — bring the sensor near (but not into) a source of CO2 or other
   test gas and confirm the readings respond, then return to baseline.
4. **Cloud check** — confirm values appear in ThingSpeak and that the graphs match
   what you observe locally.

---

## Troubleshooting

**Readings are high and drifting, never settling**
The MQ-135 heater needs a substantial warm-up — often several minutes, and the very
first use of a new sensor can take much longer to stabilize. Let it run before judging
the values, and re-calibrate `Ro` once it has settled.

**Values look random or the ADC output jumps around**
The ESP32 ADC is noisy. Average several samples per reading (for example 10–30) and
use `ADC.ATTN_11DB` for the widest input range. Keep sensor wires short and away from
the Wi-Fi antenna / power traces.

**ADC readings pinned at 0 or at 4095**
The input is out of range. Verify the divider ratio and that the sensor is actually on
5 V. If it saturates high, increase the divider ratio; if it sits near zero, check for
a bad connection or reversed sensor polarity.

**No data appears in ThingSpeak**
Check four things in order: the channel's Write API key is correct, the ESP32 actually
joined Wi-Fi (print the assigned IP), the field numbers written match your channel's
enabled fields, and you are not updating faster than the free-tier rate limit.

**Wi-Fi will not connect**
The ESP32 only supports 2.4 GHz networks. Make sure the SSID and password are exact,
the network is not captive-portal protected, and the signal is strong enough at the
device's location.

**Values differ a lot between two identical sensors**
This is expected. The MQ-135 has significant unit-to-unit variation, which is exactly
why each unit must be calibrated individually to get its own `Ro`.

**Sensor behaves differently in humid or hot conditions**
Temperature and humidity shift metal-oxide sensor response. For better accuracy, add a
temperature/humidity sensor and compensate, or at minimum re-calibrate in the
conditions where the device will normally operate.

---

## Limitations

- The MQ-135 responds to many gases at once and cannot identify a single gas reliably.
- Reported CO2 and NH3 values are estimates, not measurements from a calibrated
  analyzer.
- Accuracy depends on calibration quality, warm-up time, and ambient conditions.
- Absolute readings should not be used for safety-critical or regulatory purposes.

---

## Possible Improvements

- Add temperature and humidity sensing and compensate the readings.
- Add a local OLED or LCD display for standalone use.
- Add threshold-based alerts (buzzer, LED, or push notification).
- Log data locally as a backup when Wi-Fi is unavailable.
- Use dedicated sensors for individual gases where accuracy matters.
- Compare readings against a reference instrument to tune the `a` and `b` constants.
