# MOONS' 0.9° motor upgrade

> **Status: fitted and running (2026-11-03/04).** MOONS' **MS17HA2P4150-M** on RA and **MS17HA2P4150** on DEC, with the settings in [Final settings](#final-settings). Still to do: plate-solve calibration of steps/degree and tests with the telescope mounted.

## What we found on the mount

1. **Directions:** with the new motors both axes need `REVERSE OFF` (RA was `ON` with the stock motors). Judge directions at a slow manual rate, with correct site and time, and remember that DEC turns the other way mechanically on the west side of the pier.
2. **Stalls with upstream driver settings:** in SpreadCycle the motors vibrated strongly and stalled with a loud noise, even without a telescope at 2 °/s. Wiring (checked against the MOONS' drawing) and currents were fine. StealthChop for gotos was smoother but stalled at 3 °/s.
3. **Fix:** Trinamic's recommended SpreadCycle chopper settings plus StealthChop autograd ([configuration.md](configuration.md#chopper-tuning-for-the-09-motors)). After that, SpreadCycle gotos ran cleanly at 5 and 7.5 °/s; 10 °/s worked about 90 % of the time (no telescope).
4. **Result:** 6 °/s default with 4 goto microsteps and a 3 s acceleration ramp — faster than the stock motors' 4 °/s, with twice the tracking resolution.

The torque estimate below (from the MOONS' curve) predicted 4 °/s would be comfortable; in practice the driver tuning, not the torque, was the limit.

## The motors

| | Stock (Sumtor 42HS4013A4 / B4) | MOONS' MS17HA2P4150 / -M |
|---|---|---|
| Step angle | 1.8° (200 steps/rev) | **0.9° (400 steps/rev)** |
| Rated current | 1.3 A | **1.5 A** |
| Holding torque | not published | 0.40 N·m |
| Resistance / inductance | not published | 1.95 Ω / 5.4 mH |
| Rotor inertia | not published | 57 g·cm² |
| Body | NEMA 17, ~40 mm | NEMA 17, 39.8 mm |
| Shaft | 5 mm | 5 mm × 24 mm front; **-M adds a 5 mm × 13 mm rear shaft** |
| Connection | 4-wire | 4-wire, plug |
| Price | — | about USD 41 each (MOONS' shop) |

The two MOONS' parts are electrically identical; `-M` only adds the rear shaft. One Juwei 17 owner (archcam) fitted exactly this pair with TMC5160 drivers: on Cloudy Nights (p.43–45) he reported 64/4 microsteps, 6 °/s gotos and steps lost above about 9 °/s; on the OnStep group he later reported 32/4 microsteps and 10 °/s on a MaxPCB4 board.

**Why `-M` for RA:** the RA motor carries the power-off brake. If the stock brake mounts on the motor's rear shaft, only the double-shaft `-M` can take it. Check this on the stock RA motor before ordering.

## What gets better, what gets worse

| Better | Worse |
|---|---|
| Twice the steps per revolution: 0.25″ per tracking microstep at 32 microsteps (now 0.50″) | Each motor revolution needs twice the steps, so torque falls off with speed sooner |
| Smoother, quieter motion; owner reports no vibration or resonance at high speed | Mechanical work: pulley, brake, wiring |
| 15 % higher rated current, similar holding torque | Voids any remaining warranty |

## Speed and torque at 12 V

At the Juwei's ~404:1 reduction, the motors spin at **~270 rpm at 4 °/s** and ~400 rpm at 6 °/s. From MOONS' torque-speed curve (measured at 1.5 A RMS):

| Slew speed | Motor rpm | Torque at 12 V | Torque at 24 V | Approx. axis torque at 12 V ¹ |
|---|---|---|---|---|
| 2.6 °/s | ~175 | ~0.16 N·m | ~0.24 N·m | ~48 N·m |
| **4 °/s** | **~270** | **~0.11 N·m** | ~0.19 N·m | **~33 N·m** |
| 6 °/s | ~400 | ~0.07 N·m | ~0.15 N·m | ~21 N·m |

¹ × 404 reduction × an assumed 75 % efficiency for the belt and harmonic drive — a rough estimate.

A balanced scope needs far less than this, so **4 °/s at 12 V is comfortable and 6 °/s is plausible** — 6 °/s is now confirmed on the mount without a telescope. Our goto current (below) is about 77 % of the 1.5 A used for the curve, so low-speed torque is somewhat lower than shown. Do **not** switch the mount to 24 V unless the Juwei board's regulators, drivers and the 12 V RA brake are confirmed to handle it.

## Final settings

| Setting | Stock motors | MOONS' (in use) | Why |
|---|---|---|---|
| `AXIS1_STEPS_PER_DEGREE` | 7163.4 | **14326.8**, to be recalibrated | 400 instead of 200 steps/rev |
| `AXIS2_STEPS_PER_DEGREE` | 7182.4 | **14364.8** | same |
| `AXISn_DRIVER_MICROSTEPS` / `_GOTO` | 32 / 8 | **32 / 4** | 4 goto microsteps raise the ESP32 speed ceiling to ~22 °/s |
| `AXISn_DRIVER_IRUN` / `IGOTO` | 1000 / 1300 | **1100 / 1600** | ~51 % / 77 % of 1.5 A |
| `AXISn_DRIVER_IHOLD` | OFF | OFF | = IRUN/2 = 550 → 383 mA RMS |
| `AXIS1_REVERSE` / `AXIS2_REVERSE` | ON / OFF | **OFF / OFF** | tested on the mount |
| Decay tracking / goto | StealthChop / SpreadCycle | StealthChop / SpreadCycle | with the tuned chopper |
| `TMC_SPI_CHOPCONF_*`, `TMC_SPI_PWMCONF_AUTOGRAD` | upstream | **3 / 4 / 1 / 2, autograd on** | see [Chopper tuning](configuration.md#chopper-tuning-for-the-09-motors) |
| `SLEW_RATE_BASE_DESIRED` | 4.0 | **6.0** | 10 °/s only ~90 % reliable |
| `SLEW_ACCELERATION_DIST` / `SLEW_RAPID_STOP_DIST` | 6 / 2 | **9 / 5** | 3 s ramp; gentler emergency stop |

Current values are **peak** mA as OnStepX uses them, assuming 0.075 Ω sense resistors (see [configuration.md](configuration.md#motor-current)). With Advanced Configuration enabled in the web server, new `Config.h` axis values only take effect after *Revert to Defaults* (or `:SXA1,R#` / `:SXA2,R#` and a reboot) — see [FLASHING.md](../FLASHING.md#axis-settings-when-advanced-configuration-is-on).

Microstep alternatives:

| Tracking / goto microsteps | Steps/degree (RA) | Speed cap (`PULSE`) | Tracking microstep |
|---|---|---|---|
| 16 / 4 | 7163.4 | ~22 °/s | 0.50″ |
| 32 / 8 | 14326.8 | ~11 °/s | 0.25″ |
| **32 / 4 (in use)** | **14326.8** | **~22 °/s** | **0.25″** |
| 64 / 8 | 28653.6 | ~11 °/s | 0.13″ |
| 64 / 4 | 28653.6 | ~22 °/s | 0.13″ |

64 tracking microsteps would need steps/degree doubled and gains little: the motor's own microstep accuracy is coarser than 0.25″ on the sky.

## Hardware checklist for the swap

For anyone repeating it:

- Check that the stock RA brake mounts on the motor's rear shaft and fits a 5 mm × 13 mm shaft (`-M`).
- Check clearance for the 39.8 mm body.
- Check the pulley bore (5 mm) and how its set screw grips the MOONS' shaft; keep the same pulley position so the belt runs straight.
- Wiring: MOONS' motors use a plug; get a 4-wire harness (MOONS' 00723 is 300 mm, AWG 26) and the board-side connector. Identify coil pairs with an ohmmeter (~1.95 Ω across a pair; pinout in [hardware.md](hardware.md#motors)).
- Re-tension the belts after fitting.
- Label RA/DEC cables before unplugging anything.

## Procedure after fitting

| # | Step | Status |
|---|---|---|
| 1 | **Back up first**: both chips ([FLASHING.md](../FLASHING.md#step-2--back-up-the-esp32-onstepx)) and any object catalogs — changing steps/degree can corrupt the stored library. | done |
| 2 | Update `Config.h` as above, build and flash ([FLASHING.md](../FLASHING.md)). If Advanced Configuration is on, load the new axis values ([FLASHING.md](../FLASHING.md#axis-settings-when-advanced-configuration-is-on)). | done |
| 3 | Power on with the motors connected and still (the drivers calibrate StealthChop at boot). | done |
| 4 | Set site, date, time and UTC offset, then move each axis slowly by hand control; fix direction with `AXISn_REVERSE`. A wrong site or clock puts the mount on the other pier side and makes both axes look reversed. | done |
| 5 | Test gotos at 2 °/s, then raise the speed step by step. Watch for stalls and noise. | done without telescope; repeat with the scope mounted and balanced |
| 6 | Track for 30 minutes and do several gotos; check motor and driver temperature. | to do |
| 7 | Calibrate steps/degree: plate-solve before and after a long goto (e.g. 90°) and correct the value. This also settles the 100:1 vs 101:1 harmonic ratio question ([hardware.md](hardware.md#mechanics)). | to do |
| 8 | Redo home and park positions, alignment, and PHD2 calibration. | to do |

## Sources

- MOONS' product pages: [MS17HA2P4150](https://www.moonsindustries.com/p/nema-17-high-precision-hybrid-stepper-motors/ms17ha2p4150-000004611110015918), [MS17HA2P4150-M](https://www.moonsindustries.com/p/nema-17-high-precision-hybrid-stepper-motors/ms17ha2p4150-m-000004611110027530) (specifications and torque-speed curves)
- Cloudy Nights, Juwei-17 thread: archcam's motor swap ([p.43](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/43/), [p.44](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/44/), [p.45](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/45/)), stock motor models ([p.21](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/21/))
