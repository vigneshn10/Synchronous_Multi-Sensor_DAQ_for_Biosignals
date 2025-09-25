# Synchronous Multi-Sensor DAQ for Biosignals — Teensy 4.1

A real-time, multi-threaded synchronous data acquisition system built on the **Teensy 4.1** microcontroller. The system simultaneously acquires four physiological signals — **ECG** (electrocardiogram), **PCG** (phonocardiogram / heart sounds), **SCG** (seismocardiogram via IMU), and **body temperature** — time-stamped to a shared 1 kHz hardware clock and streamed over USB serial for offline analysis or real-time processing.

This repository also includes standalone test and utility sketches for each individual sensor, useful for hardware validation and signal characterization.

---

## Table of Contents

- [Hardware](#hardware)
- [Projects](#projects)
- [Firmware Architecture](#firmware-architecture)
- [Dependencies](#dependencies)
- [Setup & Upload](#setup--upload)
- [Usage](#usage)
- [Data Format](#data-format)
- [Fault Handling & Watchdog](#fault-handling--watchdog)
- [Signal Details](#signal-details)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Hardware

### Components

| Component | Part Number | Interface | Purpose |
|-----------|-------------|-----------|---------|
| Microcontroller | **Teensy 4.1** (MIMXRT1062) | — | 600 MHz Cortex-M7, 1 MB RAM |
| ECG frontend | **MAX30003** | SPI | Biopotential AFE with 18-bit ADC |
| IMU | **LSM9DS1** | I2C | 3-axis accel/gyro/mag (SCG via Z-accel) |
| Temperature | **MAX30205** | I2C | ±0.1°C resolution body temperature |
| PCG mic | Analog electret microphone | ADC (A2) | Heart sounds / phonocardiogram |

### Board Schematic

![Teensy 4.1 Schematic](images/teensy41_schematic.png)

### Pin Assignments

The table below covers the main multi-sensor project (`ecg_pcg_imu_temp_queued`). All I2C sensors share the same bus (pins 18/19); SPI is dedicated to the ECG frontend.

| Signal | Teensy Pin | Notes |
|--------|-----------|-------|
| **I2C SDA** | 18 | LSM9DS1 + MAX30205 shared bus |
| **I2C SCL** | 19 | 400 kHz fast mode |
| **PCG analog in** | A2 (pin 16) | DC bias ~1.55 V, gain ×10 in software |
| **ECG CS** | 10 | Active low chip select |
| **ECG MOSI** | 11 | Default SPI |
| **ECG MISO** | 12 | Default SPI |
| **ECG SCK** | 13 | Default SPI |
| **ECG INT** | 9 | FIFO interrupt, INPUT_PULLUP |
| **Status LED** | LED_BUILTIN | Heartbeat blink / error flash |

> **I2C Addresses:** LSM9DS1 Accel/Gyro = `0x6B`, LSM9DS1 Mag = `0x1E`, MAX30205 = `0x48`

---

## Projects

### `ecg_pcg_imu_temp_queued` — Main Project

The primary DAQ firmware. Runs five concurrent threads on the Teensy 4.1, each responsible for a single sensor or output task. All sensor data is queued, timestamped, and flushed to USB serial in a structured CSV format.

Two firmware versions are included:

| Version | PCG Filter | ECG API | Notes |
|---------|-----------|---------|-------|
| `v1/` | External [Filters](https://github.com/JonHub/Filters) library (1-pole lowpass) | `getECGSamples()` / `getECGSample()` | Original implementation |
| `v2/` | Self-contained `SimpleLowPassFilter` class | `readEcgSample()` | Removes the Filters dependency; cleaner ECG driver usage |

Use **v2** for new deployments. v1 is retained for reference.

---

### `pcg_recorder`

Standalone PCG recording sketch. Captures analog heart sounds from pin A2, applies a **bandpass filter at 80 Hz** using the Teensy Audio library's biquad filter, and writes a standard **44.1 kHz, 16-bit mono WAV file** directly to the onboard SD card. Recording automatically stops after 10 seconds and the WAV header is finalized with the correct file size. Files are named `RECORD001.WAV`, `RECORD002.WAV`, etc. — incrementing automatically to avoid overwrites.

Useful for offline PCG signal analysis and building phonocardiogram datasets.

---

### `temp_max30205`

Minimal MAX30205 body temperature test sketch. Scans the I2C bus for the sensor at startup, initializes it in continuous conversion mode, and prints the temperature in °C to the serial monitor at **10 Hz**. Useful for validating I2C wiring and sensor function before integrating into the full DAQ system.

---

### `accelerometer_test`

Reads the **Z-axis accelerometer** from the LSM9DS1 at **100 Hz** using the Adafruit LSM9DS1 library and prints values to serial. The Z-axis is oriented along the body's sternum-spine axis, making it the primary axis for **SCG (seismocardiography)** — detecting micro-vibrations caused by cardiac mechanical activity (valve closures, ventricular ejection) through the chest wall.

---

## Firmware Architecture

The main DAQ system (`ecg_pcg_imu_temp_queued`) uses a **hardware-timer-synchronized, multi-threaded architecture** built on [TeensyThreads](https://github.com/ftrias/TeensyThreads):

```
                    IntervalTimer ISR (1 kHz)
                           │
                    masterTimestamp++
                           │
          ┌────────────────┼──────────────────┐
          │                │                  │
     pcgThread        accelThread         tempThread
     (1000 Hz)         (1000 Hz)           (1 Hz)
     ADC read         I2C read            I2C read
     LPF filter       LSM9DS1             MAX30205
          │                │                  │
     pcgQueue         accelQueue          tempQueue
    (Mutex)           (Mutex)             (Mutex)
          │                │                  │
          └────────────────┼──────────────────┘
                           │
                      ecgThread              outputThread
                      (125 Hz)               (continuous)
                      SPI read            ┌──drain queues──┐
                      MAX30003            │  Serial print  │
                           │              │  Watchdog check│
                      ecgQueue            └────────────────┘
                      (Mutex)
```

### Key Design Decisions

- **Hardware timer as master clock:** A 1 kHz `IntervalTimer` ISR increments a shared `masterTimestamp`. All threads sample relative to this timestamp, ensuring consistent inter-sensor alignment regardless of execution timing.
- **Mutex-protected queues:** Each sensor has its own `std::queue` and `Threads::Mutex`. Producers push with a lock; the output thread pops with a lock. Queues are capped at 100 entries (10 at 1 Hz for temp) to prevent unbounded growth.
- **Sensor validation at boot:** All four sensors are validated before threads start. The system refuses to run if any sensor fails initialization, preventing silent data loss.
- **Watchdog in output thread:** Every second, the output thread checks the last update timestamp of each sensor. If any sensor goes silent for >5 seconds, the system halts and flashes the LED at 5 Hz (2 Hz for initialization failure).

---

## Dependencies

### Required: All Projects

| Library | Install via | Link |
|---------|------------|------|
| **Teensyduino** | Manual installer | https://www.pjrc.com/teensy/td_download.html |

### Required: `ecg_pcg_imu_temp_queued`

| Library | Install via | Link |
|---------|------------|------|
| **TeensyThreads** | Arduino Library Manager or GitHub | https://github.com/ftrias/TeensyThreads |
| **SparkFun LSM9DS1** | Arduino Library Manager → *SparkFun LSM9DS1* | https://github.com/sparkfun/SparkFun_LSM9DS1_Arduino_Library |
| **Protocentral MAX30003** | GitHub (manual) | https://github.com/Protocentral/protocentral_max30003 |
| **Filters** *(v1 only)* | GitHub (manual) | https://github.com/JonHub/Filters |

> `Protocentral_MAX30205.h` is bundled directly in the sketch folder — no separate install needed.

### Required: `pcg_recorder`

| Library | Install via |
|---------|------------|
| **Teensy Audio** | Bundled with Teensyduino |
| **SD** | Bundled with Teensyduino |

### Required: `accelerometer_test`

| Library | Install via |
|---------|------------|
| **Adafruit LSM9DS1** | Arduino Library Manager → *Adafruit LSM9DS1* |
| **Adafruit Unified Sensor** | Arduino Library Manager → *Adafruit Unified Sensor* |

---

## Setup & Upload

1. Install [Arduino IDE](https://www.arduino.cc/en/software) (1.8.x or 2.x)
2. Install [Teensyduino](https://www.pjrc.com/teensy/td_download.html) add-on
3. Install the required libraries listed above
4. Open the desired `.ino` file in Arduino IDE
5. Configure board settings:
   - **Tools → Board → Teensy 4.1**
   - **Tools → CPU Speed → 600 MHz**
   - **Tools → USB Type → Serial**
   - **Tools → Optimize → Faster**
6. Connect the Teensy 4.1 via USB
7. Click **Upload**

---

## Usage

### `ecg_pcg_imu_temp_queued`

Open **Tools → Serial Monitor** at **115200 baud** immediately after uploading. The firmware runs a sensor validation sequence at boot:

```
========================================
Multi-Sensor Data Acquisition System
Teensy 4.1 - Medical Sensor Array
========================================

I2C initialized at 400kHz
SPI initialized for ECG

Initializing MAX30205 Temperature Sensor...
✓ Temperature sensor initialized

Initializing LSM9DS1 Accelerometer...
✓ Accelerometer initialized

Initializing MAX30003 ECG Sensor...
✓ ECG sensor initialized

✓ Master timer started at 1kHz

========== SENSOR VALIDATION ==========
Validating PCG... OK
Validating LSM9DS1 Accelerometer... OK
Validating MAX30205 Temperature... OK (36.78°C)
Validating MAX30003 ECG... OK
=======================================

✓ All sensors validated successfully!

Starting acquisition threads...

✓✓✓ ALL SYSTEMS OPERATIONAL ✓✓✓
Data format: SENSOR,TIMESTAMP,VALUE(S)
Watchdog monitoring active

ECG,8,15234
PCG,9,18
ACCEL,10,0.0014,0.0022,9.8041
ECG,16,-3201
PCG,17,24
...
TEMP,1000,36.78
```

---

## Data Format

All data is streamed to USB serial as **plain-text CSV lines**, one sample per line:

| Sensor | Line Format | Sample Rate | Value Units |
|--------|-------------|-------------|-------------|
| ECG | `ECG,<ms>,<raw_int32>` | 125 Hz | Raw 18-bit ADC counts (MAX30003) |
| PCG | `PCG,<ms>,<filtered_int16>` | 1000 Hz | Filtered, amplified ADC value |
| Accelerometer | `ACCEL,<ms>,<x>,<y>,<z>` | 1000 Hz | m/s² (4 decimal places) |
| Temperature | `TEMP,<ms>,<celsius>` | 1 Hz | °C (2 decimal places) |

`<ms>` is milliseconds elapsed since system startup, sourced from the shared 1 kHz hardware timer — all four streams share the same timebase, so inter-sensor alignment is straightforward during post-processing.

### Example Python Parser

```python
import pandas as pd

rows = []
with open("recording.txt") as f:
    for line in f:
        parts = line.strip().split(",")
        rows.append(parts)

ecg   = [(int(r[1]), int(r[2]))   for r in rows if r[0] == "ECG"]
pcg   = [(int(r[1]), int(r[2]))   for r in rows if r[0] == "PCG"]
accel = [(int(r[1]), float(r[2]), float(r[3]), float(r[4])) for r in rows if r[0] == "ACCEL"]
temp  = [(int(r[1]), float(r[2])) for r in rows if r[0] == "TEMP"]
```

---

## Fault Handling & Watchdog

The `outputThread` runs a watchdog check every second:

| Condition | Timeout | Response |
|-----------|---------|----------|
| PCG silent | 5 s | System HALT + LED flashes at 5 Hz |
| Accelerometer silent | 5 s | System HALT + LED flashes at 5 Hz |
| ECG silent | 5 s | System HALT + LED flashes at 5 Hz |
| Temperature silent | 10 s | System HALT + LED flashes at 5 Hz |
| Sensor init failure at boot | — | System refuses to start, LED blinks at 2 Hz |

On halt, the firmware prints which sensors failed to serial before entering the flash loop. **Restart the device to recover.**

---

## Signal Details

### ECG — MAX30003

The MAX30003 is a dedicated biopotential analog front-end with a built-in 18-bit sigma-delta ADC, lead-off detection, and programmable gain. It communicates over SPI and outputs ECG samples at **125 Hz** via FIFO interrupt on pin 9. Raw output is 18-bit signed integer counts — apply the datasheet conversion factor for millivolts.

### PCG — Analog Microphone

Heart sounds are acquired via an electret microphone on analog pin A2. The firmware applies a DC offset correction (measured ~1550 counts at 3.3 V ref), a software gain of ×10, and a first-order **150 Hz low-pass filter** to suppress high-frequency noise. The `pcg_recorder` project uses the Teensy Audio library's hardware biquad filter at **80 Hz bandpass** for cleaner WAV recordings.

### SCG — LSM9DS1 Z-Axis Accelerometer

The LSM9DS1 accelerometer is configured at **±2G range, ~1 kHz ODR**. The Z-axis (sternum-spine direction when the sensor is placed on the chest) captures seismocardiographic vibrations — the mechanical impulses from valve closures and ventricular contraction/ejection. These sub-g vibrations are superimposed on the ~9.8 m/s² gravitational component.

### Temperature — MAX30205

The MAX30205 is a clinical-grade I2C temperature sensor with **±0.1°C accuracy** and 0.00390625°C resolution (16-bit). It is sampled at **1 Hz** — sufficient for skin/body temperature monitoring. The sensor address is hardcoded at `0x48` (A0/A1/A2 pins tied low).

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `Accelerometer initialization failed` | Wrong I2C address or wiring | Check SDA/SCL (pins 18/19); verify `agAddress = 0x6B` |
| `Temperature sensor not found` | I2C address mismatch or missing pull-ups | Scan I2C bus; confirm A0–A2 pins are GND; add 4.7 kΩ pull-ups |
| `ECG sensor initialization failed` | SPI wiring or CS pin issue | Verify pins 10–13 and INT on pin 9; check 3.3 V supply to MAX30003 |
| PCG always reads near zero | Mic not powered or DC bias wrong | Confirm mic VCC connected; adjust `PCG_DC_OFFSET` to match your mic's bias |
| Data stops streaming mid-run | Watchdog triggered | Check serial output for which sensor failed; inspect power and connections |
| WAV file not created | SD card not detected | Ensure SD card is formatted FAT32; check it seats properly in Teensy's slot |

---

## License

MIT License — see [LICENSE](LICENSE)

Copyright (c) 2025 Vignesh N
