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
| `AXIS1_STEPS_PER_DEGREE` | `14326.8` | `12800` | MOONS' 0.9° motors (400 steps) at 32 µsteps; the earlier calibrated 7163.4 × 2 — to be confirmed by plate solving |
| `AXIS2_STEPS_PER_DEGREE` | `14364.8` | `12800` | ≈ 400 steps × 32 µsteps × 404 ÷ 360 |
| `AXIS1_LIMIT_MIN / MAX` | `-190 / 190` | `-180 / 180` | Extra tracking past the meridian; RA has no mechanical hard stop |
| `AXIS2_LIMIT_MIN` | `-10` | `-90` | Keeps the telescope from swinging into the tripod/pier |
| `AXIS1/2_DRIVER_MICROSTEPS` | `32` | `OFF` | Tracking resolution (0.25″ per step with the 0.9° motors) |
| `AXIS1/2_DRIVER_MICROSTEPS_GOTO` | `4` | `OFF` | Fewer step pulses during slews; raises the speed ceiling to ~22 °/s — see [Step signal](#step-signal-and-slew-speed) |
| `AXIS1/2_DRIVER_IRUN` | `1100` | `OFF` | Tracking current — see [Motor current](#motor-current) |
| `AXIS1/2_DRIVER_IGOTO` | `1600` | `OFF` | Slew current |
| `TMC_SPI_CHOPCONF_TOFF / HSTRT / HEND / TBL` | `3 / 4 / 1 / 2` | `4 / 0 / 0 / 1` (added in this repo) | Trinamic's recommended SpreadCycle settings; upstream's made the MOONS' motors vibrate and stall — see [Chopper tuning](#chopper-tuning-for-the-09-motors) |
| `TMC_SPI_PWMCONF_AUTOGRAD` | `1` | `0` (added in this repo) | StealthChop adapts to the motor's back-EMF (TMC5160 power-on default) |
| `AXIS1/2_DRIVER_STATUS` | `OFF` | `OFF` | Tested `ON`: the board has no status readback line, so every read looked like a fault — see [Driver status](#driver-status) |
| `MOUNT_ENABLE_IN_STANDBY` | `ON` | `OFF` | Motors hold the axes while idle |
| `MOUNT_DISABLE_WHEN_PARKED` | `ON` | `OFF` (added in this repo) | Motors off while parked, like the factory firmware — see [Mount behaviour](#mount-behaviour) |
| `STATUS_MOUNT_LED` | `ON` | `OFF` | Mount activity LED |
| `ST4_INTERFACE` | `ON` | `OFF` | ST4 guide port / hand controller |
| `ST4_HAND_CONTROL_FOCUSER` | `OFF` | `ON` | No focuser fitted |
| `GUIDE_TIME_LIMIT` | `0` | `10` | No time limit on guide commands |
| `SLEW_RATE_BASE_DESIRED` | `6.0` | `1.0` | Goto speed in °/s; 10 °/s worked only ~90 % of the time |
| `SLEW_ACCELERATION_DIST` | `9.0` | `5.0` | 3 s to full speed at 6 °/s (2 °/s²) |
| `SLEW_RAPID_STOP_DIST` | `5.0` | `2.0` | Emergency braking 3.6 °/s² at 6 °/s instead of ~9 °/s² |
| `MFLIP_AUTOMATIC_DEFAULT` / `_MEMORY` | `ON` / `ON` | `OFF` / `OFF` | Automatic meridian flips, remembered across power cycles |
| `MFLIP_PAUSE_HOME_MEMORY` | `ON` | `OFF` | Remember the "pause at home during flip" setting |
| `PIER_SIDE_PREFERRED_DEFAULT` | `EAST` | `BEST` | Predictable orientation when a target is reachable from either side |

Everything else is at the 10.27m default, including the settings discussed in [Left at default on purpose](#left-at-default-on-purpose).

## Step signal and slew speed

`STEP_WAVE_FORM` sets the shape of the STEP signal sent to the drivers:

- **`SQUARE`** — the step timer fires twice per step (pin high, then low), giving wide 50 % duty pulses. Most robust signal, but twice the interrupts.
- **`PULSE`** — the timer fires once per step and the pin drops for only ~300 ns between steps. The firmware allows a step rate about 1.6× faster ([Goto.cpp:711](../src/telescope/mount/goto/Goto.cpp)). TMC drivers need ≥ 103 ns, so this is safe; the drivers sit on the same board with short traces.

On ESP32 the fastest step period is 40 µs in `SQUARE` and 25 µs in `PULSE` ([ESP32Libraries2.h](../src/HAL/esp/ESP32Libraries2.h)). That budget is shared across the microsteps skipped during a slew (32 tracking ÷ goto microsteps), so the speed cap depends on `MICROSTEPS_GOTO` (values for the 0.9° motors, 14326.8 steps/degree):

| Goto microsteps | Max slew, `SQUARE` | Max slew, `PULSE` |
|---|---|---|
| 2 | ~28 °/s | ~45 °/s |
| **4 (ours)** | ~14 °/s | **~22 °/s** |
| 8 | ~7 °/s | ~11 °/s |
| 16 | ~3.5 °/s | ~5.6 °/s |
| 32 (no switching) | ~1.7 °/s | ~2.8 °/s |

At 6 °/s we use about a quarter of the available step rate (`:GX99#` reports the limit: 3.125 µs per tracking step). Above that, the motors are the limit, not the ESP32. If a desired rate exceeds the cap, the firmware silently drops the default rate to half the cap ([Goto.cpp](../src/telescope/mount/goto/Goto.cpp), `usPerStepLowerLimit`).

### Acceleration

OnStepX accelerates at **a = v² ÷ (2 × distance)** ([Goto.cpp:693](../src/telescope/mount/goto/Goto.cpp)):

| | Value |
|---|---|
| Normal acceleration (6 °/s, 9°) | 2 °/s², full speed after 3 s |
| Emergency stop (abort / limit hit, `SLEW_RAPID_STOP_DIST` 5°) | 3.6 °/s², about 1.7 s |
| Gotos that never reach full speed | shorter than ~18° |

Time to full speed is 2 × distance ÷ speed, so for a different ramp time: distance = time × speed ÷ 2.

### What else follows the slew rate

- Manual move rates 8 and 9 (app / hand controller, *VF* / *VVF* in the web server's guide tile) are half and full slew rate: **3 and 6 °/s**.
- Homing, parking and meridian flips run at the slew rate.
- The web server's goto rate buttons change it at runtime from 50 % to 200 %: **3 / 4.5 / 6 / 9 / 12 °/s**. 12 °/s stalls; 9 °/s is close to the limit. `SLEW_RATE_MEMORY` is `OFF`, so it resets to 6 °/s at every boot.

Speed tests with the MOONS' motors, no telescope (2026-11-04): 5 and 7.5 °/s clean, 10 °/s about 90 % successful — hence 6 °/s as the default.

## Motor current

The motors are MOONS' **MS17HA2P4150-M** (RA) and **MS17HA2P4150** (DEC): NEMA 17, 0.9°, rated **1.5 A**. They replaced the stock Sumtor 42HS4013A4/B4 (1.8°, 1.3 A) in November 2026 — see [motor-upgrade-plan.md](motor-upgrade-plan.md).

**OnStepX treats `IRUN` / `IHOLD` / `IGOTO` as *peak* current**, not RMS (the TMCStepper path even multiplies by 0.7071, e.g. [Tmc2160.h:49](../src/lib/axis/motor/stepDir/tmc/tmcStepper/tmc2160/Tmc2160.h)). With the default 0.075 Ω sense resistors the TMC5160 sets current in 32 coarse steps of ~135 mA peak, and GLOBALSCALER is never used ([TmcSPI.cpp:37](../src/lib/axis/motor/stepDir/tmc/legacy/TmcSPI.cpp)). What the motors actually get:

| Setting | Actual peak | Actual RMS | % of 1.5 A |
|---|---|---|---|
| `IHOLD OFF` (= IRUN ÷ 2 = 550) | 542 mA | 383 mA | 26 % |
| **`IRUN 1100`** | 1083 mA | 766 mA | 51 % |
| **`IGOTO 1600`** | 1625 mA | 1149 mA | 77 % |
| *stock motors: `IRUN 1000`* | 948 mA | 670 mA | 52 % of 1.3 A |
| *stock motors: `IGOTO 1300`* | 1354 mA | 958 mA | 74 % of 1.3 A |

What other Juwei 17 owners use (Cloudy Nights thread, see [hardware.md](hardware.md#sources)):

- Stock TMC2209 drivers are set with a trimmer: early units shipped at ~350–400 mA, newer ones at ~750–850 mA; most owners raise them to **~900 mA**.
- TMC5160 swap on the stock board: `IRUN 700 / IGOTO 1000` (where our earlier values came from).
- TMC2209 on a FYSETC E4 board: `IRUN 1100`, `IGOTO` = `IRUN`.

`IRUN 1100 / IGOTO 1600` stays well below the MOONS' motors' 1.5 A rating: about half of it while tracking and three quarters during gotos. With the stock motors, `1000 / 1300` matched the ~900 mA other owners settled on. The TMC5160 modules (~3 A RMS) are far from their limit. After flashing, check driver and motor temperature after a few slews — the board sits in a closed housing. Warm is normal; too hot to keep a finger on is not.

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

The MLAstro SAL-33 forces StealthChop for gotos too; that works for them at 2 °/s. With the MOONS' motors we tested both: StealthChop for gotos stalled above about 1.5 °/s, SpreadCycle with the corrected chopper settings below runs cleanly to 7.5 °/s. On a TMC5160 only `STEALTHCHOP` and `SPREADCYCLE` are meaningful; `MIXED`, `FAST` and `SLOW` are for non-TMC drivers.

### Chopper tuning for the 0.9° motors

With upstream OnStepX's TMC5160 settings the MOONS' motors vibrated strongly and stalled with a loud noise in SpreadCycle — even without a telescope at 2 °/s. StealthChop was smoother but lost torque and stalled at 3 °/s. Wiring and currents were fine. Two driver settings in the legacy TMC SPI driver ([TmcSPI.h](../src/lib/axis/motor/stepDir/tmc/legacy/TmcSPI.h)) caused it:

| Setting | Upstream | Now | Effect |
|---|---|---|---|
| SpreadCycle `TOFF / HSTRT / HEND / TBL` | 4 / 0 / 0 / 1 | **3 / 4 / 1 / 2** | Upstream has almost no hysteresis, so current regulation is poor; these are the values from Trinamic's TMC5160 datasheet example (`CHOPCONF 0x000100C3`) |
| StealthChop `pwm_autograd` | 0 (off) | **1 (on)** | Lets StealthChop compensate the motor's back-EMF as speed rises; it is the TMC5160's own power-on default |

This repository makes them overridable (`TMC_SPI_CHOPCONF_*`, `TMC_SPI_PWMCONF_AUTOGRAD` in `Config.h`); without those lines the driver behaves exactly like upstream. After the change, SpreadCycle gotos ran cleanly up to 7.5 °/s, and archcam on Cloudy Nights reached 10 °/s with the same motors on a MaxPCB4 board.

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

## Compared with the MLAstro SAL-33

The MLAstro SAL-33 is a commercial harmonic mount running OnStepX 10.27 with a TMC2209 board. We compared configurations and took what fits this hardware:

| Idea from SAL-33 | Here |
|---|---|
| `STEP_WAVE_FORM PULSE` | Adopted |
| Finer goto microsteps (16) | Tried 8; now 4 with the 0.9° motors, for speed |
| Driver status `ON` | Tried; doesn't work on this board (no readback line) |
| `MOUNT_ENABLE_IN_STANDBY ON` | Adopted |
| `MFLIP_SKIP_HOME OFF` | Adopted |
| Acceleration distance 6° | Adopted, later 9° at 6 °/s |
| StealthChop for gotos | Tested with the MOONS' motors: stalls above ~1.5 °/s; SpreadCycle with tuned chopper is better |
| `GOTO_OFFSET 0` | Not adopted (belt slack) |
| `MOUNT_COORDS_MEMORY ON` | Not possible without FRAM |
| `TRACK_AUTOSTART ON` | Not adopted (safety) |
| OnStepX 10.27 | Adopted as upstream 10.27m — see [upgrading-to-10.27.md](upgrading-to-10.27.md) |
