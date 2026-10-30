# Brookstone Rover ESP32-S3 Conversion — Project Brief

## Project Summary
Converting a Brookstone Rover spy tank (tracked chassis with 2 DC motors)
into a BLE-gamepad-controlled robot using an ESP32-S3-DevKitC-1 N16R8,
replacing the original WiFi control board entirely.

## Hardware

| Component | Part | Notes |
|---|---|---|
| MCU | ESP32-S3-DevKitC-1 N16R8 | 16 MB flash, 8 MB OPI PSRAM |
| Motor driver | DRV8833 dual H-bridge module | WWZMDiB breakout board |
| Power regulation | MP1584EN adjustable buck converter | Set to 5V output |
| Battery | 6x NiMH AA (7.2V nominal) | Rechargeable |
| Power switch | Toggle switch | On battery + line, before split |
| Chassis | Brookstone Rover tank treads | 2 independent tracks, left + right |

## Wiring

**Power:**
- Battery + → switch → splits to: MP1584 IN+ AND DRV8833 VCC (motor supply)
- Battery − → common GND bus
- MP1584 OUT+ (5V) → ESP32 VIN
- MP1584 OUT− → common GND bus
- ESP32 3V3 → DRV8833 EEP/nSLEEP (enable)
- Common GND bus ties together: battery −, MP1584 GND, ESP32 GND, DRV8833 GND

**Motor control signals (move the wires to these ESP32-S3 GPIOs):**
- GPIO 4 → DRV8833 IN1 (left motor, channel A)
- GPIO 5 → DRV8833 IN2 (left motor, channel B)
- GPIO 6 → DRV8833 IN3 (right motor, channel A)
- GPIO 7 → DRV8833 IN4 (right motor, channel B)

**DRV8833 motor outputs:**
- OUT1/OUT2 → left motor terminals
- OUT3/OUT4 → right motor terminals
- (If a motor spins the wrong direction, swap its two OUT wires — no code change needed)

Avoid GPIOs reserved for the N16R8 module's flash and PSRAM.

## Motor Control Logic (DRV8833)

| IN_a | IN_b | Result |
|---|---|---|
| HIGH | PWM | Forward, slow-decay PWM |
| PWM | HIGH | Reverse, slow-decay PWM |
| LOW | LOW | Coast (failsafe stop) |
| HIGH | HIGH | Brake |

## Software Requirements

1. **Framework**: PlatformIO (preferred environment), Arduino framework for ESP32
2. **BLE gamepad support**: Use the **Bluepad32** library — supports PS4, PS5, Xbox, and generic BLE gamepads with robust pairing/reconnection handling
3. **Control scheme**:
   - Left analog stick Y-axis → throttle
   - Right analog stick X-axis → steering
   - Mix throttle and steering into left/right track commands (arcade drive)
4. **PWM**: Use ESP32 `ledcWrite()` / LEDC peripheral for smooth speed control on all 4 GPIO pins (avoid `analogWrite` — not native on ESP32 in all cores)
5. **Safety features to include**:
   - Deadzone handling on analog stick input to prevent motor jitter at rest
   - Failsafe: if BLE controller disconnects, stop both motors immediately
   - Optional: gentle ramp-up/ramp-down rather than instant full power, to protect the gearbox


## What to ask Claude Code to do

Use the prompt below as your starting message to Claude Code.

---

### PROMPT FOR CLAUDE CODE

```
I'm building a tracked robot using an ESP32-S3-DevKitC-1 N16R8 and a DRV8833 motor
driver, controlled by a BLE gamepad. Please set up a PlatformIO project for this.

Hardware:
- ESP32-S3-DevKitC-1 N16R8 (16 MB flash, 8 MB OPI PSRAM)
- DRV8833 dual H-bridge motor driver
- Two DC motors (left track, right track)
- Powered by 6x NiMH AA batteries (7.2V) through a buck converter to 5V for the ESP32

Pin wiring:
- GPIO 4 -> DRV8833 IN1 (left motor, channel A)
- GPIO 5 -> DRV8833 IN2 (left motor, channel B)
- GPIO 6 -> DRV8833 IN3 (right motor, channel A)
- GPIO 7 -> DRV8833 IN4 (right motor, channel B)

Requirements:
1. Use the Bluepad32 library for BLE gamepad support (PS4/PS5/Xbox/generic controllers)
2. Use ESP32 LEDC PWM (ledcWrite) for motor speed control, not analogWrite
3. Arcade drive: left stick Y-axis controls throttle; right stick X-axis
   controls steering, mixed across both tracks
4. Add a deadzone (~10%) on stick input to prevent motor jitter when centered
5. If the gamepad disconnects, immediately stop both motors (failsafe)
6. Add a basic ramp limiter so motor speed changes smoothly rather than
   snapping instantly to full power/reverse, to protect the gearbox
7. Set up platformio.ini with the `esp32-s3-devkitc-1` board target, 16 MB
   flash, and OPI PSRAM. Use the Bluepad32 framework package override already
   used by this project; do not add a `lib_deps` entry.

Please create the full project structure, write the main.cpp with clear
comments, and explain how to flash it to the board over USB.
```

---

## Notes for You

- Before flashing, **double-check the MP1584 is set to 5V** with a multimeter — this is a one-time setup step, not part of the firmware
- The motor signal wires must connect to the updated GPIO mapping above
- The wiring image still shows the original ESP32 DevKit V1; the GPIO mapping
  above is authoritative for the S3 board.
