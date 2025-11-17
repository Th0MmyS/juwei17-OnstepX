# OnStepX Telescope Controller for Juwei 17

Advanced computerized telescope controller firmware for precision astronomical mount control.

**This configuration is specifically set up for the [Juwei 17 Harmonic Drive Mount](https://www.aliexpress.com/item/1005009419602903.html)** - a compact equatorial mount with harmonic drive technology providing high precision and minimal backlash. The Juwei 17 controller board is 1:1 pin-compatible with the MaxESP3 pinmap.

## Overview

OnStepX is a powerful open-source telescope control system designed by Howard Dutton. It provides comprehensive control for telescope mounts with support for multiple motor driver types including Step/Dir, ODrive, and Servo configurations.

## Configuration

Configuration is primarily handled through the `Config.h` file. The most critical settings involve the two main axes that control the telescope mount.

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

### Juwei 17 Mount Configuration

This setup is configured as an **Equatorial Mount (GEM)**:
- AXIS1: RA tracking axis with TMC5160 driver (7163.4 steps/degree)
- AXIS2: DEC positioning axis with TMC5160 driver (7182.4 steps/degree)
- Mount type: `GEM`
- Hardware: MaxESP3 controller board
- Harmonic drive system provides exceptional precision and minimal backlash

**Hardware Upgrades:**
- **TMC5160 Drivers**: Upgraded from TMC2209 to TMC5160 for faster slewing speeds and improved overall performance. The TMC5160 provides higher current capability and better thermal management during rapid GoTo operations.

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

5. Flash the firmware to your controller

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
