# OnStepX Telescope Controller for Juwei 17

Advanced computerized telescope controller firmware for precision astronomical mount control.

**This configuration is specifically set up for the [Juwei 17 Harmonic Drive Mount](https://www.aliexpress.com/item/1005009419602903.html)** - a compact equatorial mount with harmonic drive technology providing high precision and minimal backlash. The Juwei 17 controller board is 1:1 pin-compatible with the MaxESP3 pinmap.

> ⚠️ **Disclaimer:** Any hardware or software modification described in this repository is performed **entirely at your own risk**. Flashing custom firmware, changing DIP switch settings, swapping stepper drivers (e.g. TMC2209 → TMC5160), adjusting axis travel limits, motor currents, or any other parameter in `Config.h` may damage your mount, controller board, motors, telescope, or void your warranty. Neither the author of this fork nor the upstream OnStepX authors accept responsibility for damage, injury, or loss resulting from its use. Verify every setting against your own hardware before powering on.

## Overview

OnStepX is a powerful open-source telescope control system designed by Howard Dutton. It provides comprehensive control for telescope mounts with support for multiple motor driver types including Step/Dir, ODrive, and Servo configurations.

## Configuration

Configuration is primarily handled through the `Config.h` file. The most critical settings involve the two main axes that control the telescope mount.

**Hardware Upgrades:**
- **TMC5160 Drivers**: Upgraded from TMC2209 to TMC5160 for faster slewing speeds and improved overall performance. The TMC5160 provides higher current capability and better thermal management during rapid GoTo operations.

### AXIS1 - Right Ascension (RA) / Azimuth

AXIS1 controls the primary rotation axis. For equatorial mounts, this is the Right Ascension (RA) axis that tracks celestial objects. For Alt/Az mounts, this is the Azimuth axis.

**Essential Configuration Parameters:**

```cpp
#define AXIS1_DRIVER_MODEL        TMC5160
```
Specifies the stepper motor driver model. Common options include TMC2209, TMC5160, TMC2130, DRV8825, A4988.

```cpp
#define AXIS1_STEPS_PER_DEGREE    7163.4
```
Critical calibration value calculated as:
`(stepper_steps × micro_steps × gear_reduction) / 360.0`

**Axis Travel Limits:**

```cpp
#define AXIS1_LIMIT_MIN           -200
#define AXIS1_LIMIT_MAX            200
```
Extended beyond the default ±180° to allow additional travel past the meridian on the harmonic drive. The Juwei 17 has no mechanical hard stops on RA, so the extra ±20° of range provides more flexibility for tracking targets across the meridian before a flip is required.

**Microstepping Configuration:**

```cpp
#define AXIS1_DRIVER_MICROSTEPS   32
```
Microstepping resolution during tracking. Higher values (16, 32, 64, 128, 256) provide smoother motion.

```cpp
#define AXIS1_DRIVER_MICROSTEPS_GOTO  2
```
Microstepping during slews. Lower values allow faster movement while maintaining torque.

**Current Settings (TMC Drivers):**

```cpp
#define AXIS1_DRIVER_IRUN         700
```
Running current in milliamps during tracking. Set based on motor specifications (typically 50-70% of motor rating).

```cpp
#define AXIS1_DRIVER_IGOTO        1000
```
Slew current in milliamps during GoTo operations. Can be higher for faster movements.

```cpp
#define AXIS1_DRIVER_IHOLD        OFF
```
Standstill current. OFF defaults to IRUN/2. Lower values reduce heat and power consumption.

### AXIS2 - Declination (DEC) / Altitude

AXIS2 controls the secondary rotation axis. For equatorial mounts, this is the Declination (DEC) axis. For Alt/Az mounts, this is the Altitude axis.

**Essential Configuration Parameters:**

```cpp
#define AXIS2_DRIVER_MODEL        TMC5160
```
Motor driver model for the DEC/ALT axis. Should match or be compatible with AXIS1 driver.

```cpp
#define AXIS2_STEPS_PER_DEGREE    7182.4
```
Steps per degree calibration. May differ from AXIS1 if gear ratios are different.

**Microstepping Configuration:**

```cpp
#define AXIS2_DRIVER_MICROSTEPS   32
#define AXIS2_DRIVER_MICROSTEPS_GOTO  2
```
Same principles as AXIS1. DEC axis often uses identical settings.

**Current Settings:**

```cpp
#define AXIS2_DRIVER_IRUN         700
#define AXIS2_DRIVER_IGOTO        1000
#define AXIS2_DRIVER_IHOLD        OFF
```
Current settings for DEC/ALT motor. DEC axis may require different current than RA depending on load balance.

**Axis Travel Limits:**

```cpp
#define AXIS2_LIMIT_MIN            -10
#define AXIS2_LIMIT_MAX             90
```
The lower DEC limit is restricted from the default `-90°` to `-10°` to prevent the OTA from swinging into the tripod/pier on the Juwei 17. The upper limit remains at `90°` (celestial pole).

### Mount Behavior

```cpp
#define PIER_SIDE_PREFERRED_DEFAULT   EAST
```
Overrides the default `BEST` behavior. The mount now prefers the EAST pier side when a target is reachable from either side, giving more consistent startup orientation and predictable meridian-flip behavior for the Juwei 17.

### Juwei 17 Mount Configuration

This setup is configured as an **Equatorial Mount (GEM)**:
- AXIS1: RA tracking axis with TMC5160 driver (7163.4 steps/degree), travel limits ±200°
- AXIS2: DEC positioning axis with TMC5160 driver (7182.4 steps/degree), travel limits -10° to +90°
- Mount type: `GEM`
- Preferred pier side: `EAST`
- Hardware: MaxESP3 controller board
- Harmonic drive system provides exceptional precision and minimal backlash

## Flashing the Firmware

Before flashing, you must set the board's DIP switches into bootloader mode to allow programming over USB.

### DIP Switch Mapping
- **Switches 1, 2, 3**: Control the ESP8266 (`1 = RX`, `2 = TX`, `3 = IO0`)
- **Switches 4, 5, 6**: Control the ESP32 (`4 = IO0`, `5 = RX`, `6 = TX`)

### Standard vs. Flashing State

| Mode | Switch 1 | Switch 2 | Switch 3 | Switch 4 | Switch 5 | Switch 6 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Normal Operation** | OFF | OFF | OFF | OFF | **ON** | **ON** |
| **ESP32 Flashing** | OFF | OFF | OFF | **ON** | **ON** | **ON** |

### Step-by-Step Flashing Procedure

1. **Remove Board**: Take the controller board out of the mount enclosure.
2. **Disconnect Motors**: Ensure the motors are **not connected** to the board during flashing.
3. **Set Flashing Mode**: Flip DIP switch **4 to ON** (switches 5 and 6 remain ON; switches 1, 2, 3 remain OFF).
4. **Connect Power & USB**: Plug in your power source and connect the USB cable to your computer.
5. **Turn On**: Power on the board.
6. **Flash Firmware**: Upload the compiled OnStepX firmware using **Arduino IDE** or **esptool**.
7. **Restore Switches**: Once flashing is complete, turn off power, flip DIP switch **4 back to OFF**, reinstall the board, and reconnect your motors.

## Getting Started

1. Select your hardware pinmap in `Config.h`:
```cpp
#define PINMAP    MaxESP3
```

2. Configure AXIS1 and AXIS2 driver models and steps per degree

3. Set mount type:
```cpp
#define MOUNT_TYPE    GEM
```

4. Configure time/location source if available:
```cpp
#define TIME_LOCATION_SOURCE    DS3231
```

5. Flash the firmware to your controller following the instructions above.

6. Perform initial alignment and calibration

## Documentation

Comprehensive documentation available at the [OnStep Wiki](https://groups.io/g/onstep/wiki/home):
- Hardware builds and PCB designs
- Detailed configuration guides
- Troubleshooting and support
- Wiring diagrams

## Support

- **Discussion Group**: [OnStep Groups.io](https://groups.io/g/onstep/)
- **GitHub**: [OnStep Repository](https://github.com/hjd1964/OnStep)

## Version

Current firmware: **v10.26g**

## License

OnStepX is open source software licensed under the GNU General Public License v3.0.

See the `LICENSE` file for full license text.

## Author

**Howard Dutton**
- Website: [Stellar Journey](http://www.stellarjourney.com)
- Email: hjd1964@gmail.com

---

*For detailed configuration of additional features including PEC, weather sensors, and other advanced options, refer to the heavily commented `Config.h` file and the online documentation.*