# Configuration reference

This page explains every setting in [`Config.h`](../Config.h) that differs from the OnStepX 10.27m template, why it is set that way, and the settings we looked at but deliberately left alone. Line references point at the 10.27m source in this repository.

> **Runtime overrides.** If *Advanced Configuration* is enabled from the Smart Web Server, the mount stores its axis settings (steps/degree, limits, reverse, microsteps, currents, decay) in its own memory and **ignores the axis section of `Config.h`** until you revert or disable it and reboot. See [web-server.md](web-server.md).

## Settings at a glance

| Setting | Value | Template default | Why |
|---|---|---|---|
| `HOST_NAME` | `"OnStepX"` | `"OnStep"` | 10.27 names the Bluetooth device after this; `"OnStepX"` keeps the pairing from 10.22 |
| `PINMAP` | `MaxESP3` | `OFF` | The Juwei 17 controller board is pin-compatible with MaxESP3 |
| `SERIAL_RADIO` | `BLUETOOTH` | `OFF` | ESP32 Bluetooth serial command channel |
| `STEP_WAVE_FORM` | `PULSE` | `SQUARE` | More step-rate headroom, half the interrupts — see [Step signal](#step-signal-and-slew-speed) |
| `TIME_LOCATION_SOURCE` | `DS3231` | `OFF` | On-board real-time clock with coin-cell backup; the factory firmware used it too |
| `AXIS1/2_DRIVER_MODEL` | `TMC5160` | `OFF` | Stock TMC2208/2209 drivers replaced, see [hardware.md](hardware.md) |
| `AXIS1_STEPS_PER_DEGREE` | `7163.4` | `12800` | Calibrated for this mount |
| `AXIS2_STEPS_PER_DEGREE` | `7182.4` | `12800` | ≈ 200 steps × 32 µsteps × 404 ÷ 360 |
| `AXIS1_REVERSE` | `ON` | `OFF` | Matches the motor wiring |
| `AXIS1_LIMIT_MIN / MAX` | `-190 / 190` | `-180 / 180` | Extra tracking past the meridian; RA has no mechanical hard stop |
| `AXIS2_LIMIT_MIN` | `-10` | `-90` | Keeps the telescope from swinging into the tripod/pier |
| `AXIS1/2_DRIVER_MICROSTEPS` | `32` | `OFF` | Tracking resolution |
| `AXIS1/2_DRIVER_MICROSTEPS_GOTO` | `8` | `OFF` | Smoother, quieter slews than 2 — see [Step signal](#step-signal-and-slew-speed) |
| `AXIS1/2_DRIVER_IRUN` | `1000` | `OFF` | Tracking current — see [Motor current](#motor-current) |
| `AXIS1/2_DRIVER_IGOTO` | `1300` | `OFF` | Slew current |
| `AXIS1/2_DRIVER_STATUS` | `OFF` | `OFF` | Tested `ON`: the board has no status readback line, so every read looked like a fault — see [Driver status](#driver-status) |
| `MOUNT_ENABLE_IN_STANDBY` | `ON` | `OFF` | Motors hold the axes while idle |
| `MOUNT_DISABLE_WHEN_PARKED` | `ON` | `OFF` (added in this repo) | Motors off while parked, like the factory firmware — see [Mount behaviour](#mount-behaviour) |
| `STATUS_MOUNT_LED` | `ON` | `OFF` | Mount activity LED |
| `ST4_INTERFACE` | `ON` | `OFF` | ST4 guide port / hand controller |
| `ST4_HAND_CONTROL_FOCUSER` | `OFF` | `ON` | No focuser fitted |
| `GUIDE_TIME_LIMIT` | `0` | `10` | No time limit on guide commands |
| `SLEW_RATE_BASE_DESIRED` | `4.0` | `1.0` | Goto speed in °/s |
| `SLEW_ACCELERATION_DIST` | `6.0` | `5.0` | Gentler acceleration ramp |
| `MFLIP_AUTOMATIC_DEFAULT` / `_MEMORY` | `ON` / `ON` | `OFF` / `OFF` | Automatic meridian flips, remembered across power cycles |
| `MFLIP_PAUSE_HOME_MEMORY` | `ON` | `OFF` | Remember the "pause at home during flip" setting |
| `PIER_SIDE_PREFERRED_DEFAULT` | `EAST` | `BEST` | Predictable orientation when a target is reachable from either side |

Everything else is at the 10.27m default, including the settings discussed in [Left at default on purpose](#left-at-default-on-purpose).

## Step signal and slew speed

`STEP_WAVE_FORM` sets the shape of the STEP signal sent to the drivers:

- **`SQUARE`** — the step timer fires twice per step (pin high, then low), giving wide 50 % duty pulses. Most robust signal, but twice the interrupts.
- **`PULSE`** — the timer fires once per step and the pin drops for only ~300 ns between steps. The firmware allows a step rate about 1.6× faster ([Goto.cpp:711](../src/telescope/mount/goto/Goto.cpp)). TMC drivers need ≥ 103 ns, so this is safe; the drivers sit on the same board with short traces.

On ESP32 the fastest step period is 40 µs in `SQUARE` and 25 µs in `PULSE` ([ESP32Libraries2.h](../src/HAL/esp/ESP32Libraries2.h)). That budget is shared across the microsteps skipped during a slew (32 tracking ÷ goto microsteps), so the speed cap depends on `MICROSTEPS_GOTO`:

| Goto microsteps | Max slew, `SQUARE` | Max slew, `PULSE` |
|---|---|---|
| 2 | ~56 °/s | ~89 °/s |
| 4 | ~28 °/s | ~45 °/s |
| **8 (ours)** | ~14 °/s | **~22 °/s** |
| 16 | ~7 °/s | ~11 °/s |
| 32 (no switching) | ~3.5 °/s | ~5.6 °/s |

At 4 °/s we use well under a fifth of the available step rate. If a desired rate exceeds the cap, the firmware silently drops the default rate to half the cap ([Goto.cpp](../src/telescope/mount/goto/Goto.cpp), `usPerStepLowerLimit`).

### Acceleration

OnStepX accelerates at **a = v² ÷ (2 × distance)** ([Goto.cpp:693](../src/telescope/mount/goto/Goto.cpp)):

| | Value |
|---|---|
| Normal acceleration (4 °/s, 6°) | 1.33 °/s², full speed after ~3 s |
| Emergency stop (abort / limit hit, `SLEW_RAPID_STOP_DIST` 2°) | 4 °/s² |
| Gotos that never reach full speed | shorter than ~12° |

### What else follows the slew rate

- Manual move rates 8 and 9 (app / hand controller) are half and full slew rate: **2 and 4 °/s**.
- Homing, parking and meridian flips run at the slew rate.
- The rate can be changed at runtime from 50 % to 200 % (**2–8 °/s**). `SLEW_RATE_MEMORY` is `OFF`, so it resets to 4 °/s at every boot.

We tested the idea of 6 °/s and went back to 4 °/s: at 6 °/s emergency braking becomes 9 °/s² and the stock motors have less torque in reserve.

## Motor current

The stock motors are Sumtor **42HS4013A4** (RA) and **42HS4013B4** (DEC): NEMA 17, 1.8°, rated **1.3 A**.

**OnStepX treats `IRUN` / `IHOLD` / `IGOTO` as *peak* current**, not RMS (the TMCStepper path even multiplies by 0.7071, e.g. [Tmc2160.h:49](../src/lib/axis/motor/stepDir/tmc/tmcStepper/tmc2160/Tmc2160.h)). With the default 0.075 Ω sense resistors the TMC5160 sets current in 32 coarse steps of ~135 mA peak, and GLOBALSCALER is never used ([TmcSPI.cpp:37](../src/lib/axis/motor/stepDir/tmc/legacy/TmcSPI.cpp)). What the motors actually get:

| Setting | Actual peak | Actual RMS | % of 1.3 A |
|---|---|---|---|
| `IHOLD OFF` (= IRUN ÷ 2 = 500) | 542 mA | 383 mA | 29 % |
| **`IRUN 1000`** | 948 mA | 670 mA | 52 % |
| **`IGOTO 1300`** | 1354 mA | 958 mA | 74 % |
| *previous `IRUN 700`* | 677 mA | 479 mA | 37 % |
| *previous `IGOTO 1000`* | 948 mA | 670 mA | 52 % |

What other Juwei 17 owners use (Cloudy Nights thread, see [hardware.md](hardware.md#sources)):

- Stock TMC2209 drivers are set with a trimmer: early units shipped at ~350–400 mA, newer ones at ~750–850 mA; most owners raise them to **~900 mA**.
- TMC5160 swap on the stock board: `IRUN 700 / IGOTO 1000` (where our earlier values came from).
- TMC2209 on a FYSETC E4 board: `IRUN 1100`, `IGOTO` = `IRUN`.

The current `IRUN 1000 / IGOTO 1300` is in line with the ~900 mA owners settled on and stays well below the motors' 1.3 A rating: about half of it while tracking and three quarters during gotos. The TMC5160 modules (~3 A RMS) are far from their limit. After flashing, check driver and motor temperature after a few slews — the board sits in a closed housing. Warm is normal; too hot to keep a finger on is not.

These numbers assume 0.075 Ω sense resistors (the OnStepX default, used on BTT TMC5160 Pro modules). You can check: the sense resistors on the driver module are marked **R075**. With a smaller value (e.g. R050) the real current would be 1.5× higher — in that case set `AXISn_DRIVER_RSENSE` to the real value.

## Driver status

With `AXISn_DRIVER_STATUS ON`, OnStepX reads each TMC5160's status register over SPI every 200 ms ([StepDirDriver.cpp:267](../src/lib/axis/motor/stepDir/StepDirDriver.cpp)). It flags a **fault** on:

- a coil shorted to ground (either phase)
- the driver's over-temperature pre-warning or over-temperature shutdown

Open-load is read but ignored, because it is unreliable at standstill. On a fault the mount aborts any goto or guide, refuses new gotos/park/guide with a hardware-fault error, and stops the axis ([Mount.cpp:297](../src/telescope/mount/Mount.cpp)). Without monitoring, an overheating driver or a shorted cable just skips steps while OnStepX keeps believing its position.

Both drivers return data on one shared pin, GPIO4 (AUX2). That pin is free in this configuration because `SERIAL_B_ESP_FLASHING` is `OFF` and there is no focuser.

> **Result on this board (tested 2026-10-28): driver status does not work, so it is `OFF`.** With it `ON`, OnStepX 10.27m reported every flag on both drivers at once — `:GXU1#` and `:GXU2#` both returned `ST,OA,OB,GA,GB,OT,PW,GF` (standstill, open load, short to ground, over-temperature, pre-warning, fault) with the motors unplugged and the board cold — and `:GU#` ended in error `1` (motor fault), which blocks gotos and parking. That is the all-ones pattern: the TMC5160 data-out (SDO) pins are not connected to GPIO4 on the Juwei board, or the line is pulled high (the MaxESP3 pinmap also labels GPIO4 as the ESP8266 reset). OnStepX only rejects all-zero reads ([TmcSPI.cpp:215](../src/lib/axis/motor/stepDir/tmc/legacy/TmcSPI.cpp)), so all ones look like a real fault.
>
> Writing settings to the drivers is unaffected: currents, microsteps and decay modes still go out over SPI (the drivers would not run at all otherwise). Only the readback is missing. Fault protection would need a wire from the drivers' SDO pins to GPIO4.

## Decay modes

`AXISn_DRIVER_DECAY` and `_DECAY_GOTO` are left at `OFF`, which means **StealthChop while tracking and SpreadCycle during gotos** ([TmcStepDirDriver.cpp:175](../src/lib/axis/motor/stepDir/tmc/TmcStepDirDriver.cpp)). OnStepX switches mode (and current, IRUN ↔ IGOTO) at the start and end of every slew.

| | StealthChop | SpreadCycle |
|---|---|---|
| Noise | nearly silent | audible hiss at low speed |
| Low-speed smoothness | excellent | good |
| Torque at speed / under load | weaker, can stall | full torque |
| Best for | tracking | gotos |

The MLAstro SAL-33 forces StealthChop for gotos too; that works for them at 2 °/s. At our speeds SpreadCycle's torque is the safer choice. On a TMC5160 only `STEALTHCHOP` and `SPREADCYCLE` are meaningful; `MIXED`, `FAST` and `SLOW` are for non-TMC drivers.

## Mount behaviour

- **`MOUNT_ENABLE_IN_STANDBY ON`** — motor drivers stay energised at hold current after boot, after parking and after a home reset ([Mount.cpp:68](../src/telescope/mount/Mount.cpp), [Park.cpp:201](../src/telescope/mount/park/Park.cpp)). With `TRACK_AUTOSTART OFF` the mount otherwise sits unpowered until the first goto, held only by the gear train — and the RA brake releases as soon as the board has power. While parked, `MOUNT_DISABLE_WHEN_PARKED` takes over (next item).
- **`MOUNT_DISABLE_WHEN_PARKED ON`** (added in this repo, not upstream OnStepX) — the motors are powered down while parked, and the shared enable pin GPIO12 follows motor power (HIGH = off), as the factory firmware did. Park completion, unpark, `:Te#`, gotos, guiding, home reset and motor faults all switch power through `Mount::enable()`, which refuses to power up while parked and then drives GPIO12 ([Mount.cpp](../src/telescope/mount/Mount.cpp)). At boot GPIO12 stays LOW through the TMC5160 calibration, then goes HIGH if the mount is parked. A failed park leaves the motors holding. Set `OFF` to get stock OnStepX behaviour back. Whether this also engages the RA brake depends on its wiring — see [hardware.md](hardware.md#ra-brake).
- **`GOTO_OFFSET 0.25`** (default) — every goto first aims 0.25° short in both axes, waits, then makes a short final move, so the last approach is always from the same direction and any slack (the belt stage in front of the harmonic drive, reported DEC backlash) is taken up consistently ([Goto.cpp:129](../src/telescope/mount/goto/Goto.cpp)).
- **`MFLIP_SKIP_HOME OFF`** (default) — a goto that changes pier side first slews to home (counterweight down), with an extra avoidance move if the scope is low, then to the target ([Goto.cpp:143](../src/telescope/mount/goto/Goto.cpp)). Going direct (`ON`) is faster but the scope can sweep lower than either end point mid-move. `MFLIP_PAUSE_HOME` only works with the home waypoint.
- **`PIER_SIDE_PREFERRED_DEFAULT EAST`** — 10.27 also offers `AUTOMATIC` (prefers west near the under-pole meridian, east otherwise).

## Left at default on purpose

| Setting | Why it stays |
|---|---|
| `MOUNT_COORDS_MEMORY OFF` | Saves the position every second; the firmware refuses to build without FRAM memory ([Mount.cpp:24](../src/telescope/mount/Mount.cpp)). The board uses ESP32 flash (`NV_DEFAULT`), which would wear out. Needs an MB85RC64 FRAM on I2C plus `NV_DRIVER NV_MB85RC64` (which resets stored settings). |
| `GOTO_OFFSET 0.25` | See above; the SAL-33 uses `0`. |
| `AXISn_DRIVER_DECAY*` | See [Decay modes](#decay-modes). |
| `TRACK_AUTOSTART OFF` | Tracking after an unexpected power cycle can drive the scope into the tripod. |
| `SLEW_RAPID_STOP_DIST 2.0` | Gives 4 °/s² braking at 4 °/s. Raise it if you ever go faster. |

## Compared with the MLAstro SAL-33

The MLAstro SAL-33 is a commercial harmonic mount running OnStepX 10.27 with a TMC2209 board. We compared configurations and took what fits this hardware:

| Idea from SAL-33 | Here |
|---|---|
| `STEP_WAVE_FORM PULSE` | Adopted |
| Finer goto microsteps (16) | Adopted as 8 |
| Driver status `ON` | Adopted |
| `MOUNT_ENABLE_IN_STANDBY ON` | Adopted |
| `MFLIP_SKIP_HOME OFF` | Adopted |
| Acceleration distance 6° | Adopted |
| StealthChop for gotos | Not adopted (torque) |
| `GOTO_OFFSET 0` | Not adopted (belt slack) |
| `MOUNT_COORDS_MEMORY ON` | Not possible without FRAM |
| `TRACK_AUTOSTART ON` | Not adopted (safety) |
| OnStepX 10.27 | Adopted as upstream 10.27m — see [upgrading-to-10.27.md](upgrading-to-10.27.md) |
