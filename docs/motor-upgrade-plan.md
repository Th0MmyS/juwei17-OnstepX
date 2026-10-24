# Plan: upgrading to MOONS' 0.9° motors

> **Status: plan only — nothing here is applied yet.** This lists what would change if the stock motors are replaced with MOONS' **MS17HA2P4150-M** (RA) and **MS17HA2P4150** (DEC).

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

The two MOONS' parts are electrically identical; `-M` only adds the rear shaft. One Juwei 17 owner (archcam, Cloudy Nights p.43–45) fitted exactly this pair, running OnStepX with TMC5160 drivers at 64/4 microsteps and 6 °/s gotos; he reached about 9 °/s before losing steps.

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

A balanced scope needs far less than this, so **4 °/s at 12 V is comfortable and 6 °/s is plausible**. Our goto current (below) is about 77 % of the 1.5 A used for the curve, so low-speed torque is somewhat lower than shown. Do **not** switch the mount to 24 V unless the Juwei board's regulators, drivers and the 12 V RA brake are confirmed to handle it.

## Configuration changes

| Setting | Now | With MOONS' | Why |
|---|---|---|---|
| `AXIS1_STEPS_PER_DEGREE` | 7163.4 | **14326.8**, then recalibrate | 400 instead of 200 steps/rev |
| `AXIS2_STEPS_PER_DEGREE` | 7182.4 | **14364.8**, then recalibrate | same |
| `AXISn_DRIVER_MICROSTEPS` / `_GOTO` | 32 / 8 | 32 / 8 (keep) | Speed cap drops from ~22 to ~11 °/s — still well above 4 °/s |
| `AXISn_DRIVER_IRUN` | 700 | **1100** | 766 mA RMS, 51 % of 1.5 A |
| `AXISn_DRIVER_IGOTO` | 1000 | **1600** | 1149 mA RMS, 77 % of 1.5 A |
| `AXISn_DRIVER_IHOLD` | OFF | OFF | = IRUN/2 = 550 → 383 mA RMS |
| `AXIS1_REVERSE` / `AXIS2_REVERSE` | ON / OFF | check after the first move | New motors may turn the other way |

Everything else stays: `STEP_WAVE_FORM PULSE`, interpolation on, default decay modes, driver status, slew speed 4 °/s. Current values are **peak** mA as OnStepX uses them, assuming 0.075 Ω sense resistors (see [configuration.md](configuration.md#motor-current)).

Microstep alternatives:

| Tracking / goto microsteps | Steps/degree (RA) | Speed cap (`PULSE`) | Tracking microstep |
|---|---|---|---|
| 16 / 4 | 7163.4 (same as now) | ~22 °/s | 0.50″ |
| **32 / 8 (recommended)** | **14326.8** | **~11 °/s** | **0.25″** |
| 32 / 4 | 14326.8 | ~22 °/s | 0.25″ |
| 64 / 8 | 28653.6 | ~11 °/s | 0.13″ |
| 64 / 4 (archcam) | 28653.6 | ~22 °/s | 0.13″ |

If *Advanced Configuration* is enabled in the web server, make these changes there (or revert to `Config.h` first) — otherwise the mount ignores the `Config.h` axis values.

## Hardware checklist

- [ ] Confirm the stock RA brake mounts on the motor's rear shaft and fits a 5 mm × 13 mm shaft (`-M`).
- [ ] Measure the stock motor body length and check clearance for 39.8 mm.
- [ ] Check the pulley bore (5 mm) and how its set screw grips the MOONS' shaft; keep the same pulley position so the belt runs straight.
- [ ] Wiring: MOONS' motors use a plug; get a 4-wire harness (MOONS' 00723 is 300 mm, AWG 26) and the board-side connector. Identify coil pairs with an ohmmeter (~1.95 Ω across a pair) and wire A/B pairs the same way as the stock motors.
- [ ] Re-tension the belts after fitting.
- [ ] Label RA/DEC cables before unplugging anything.

## Procedure after fitting

1. **Back up first**: both chips ([flashing.md](flashing.md#step-1--back-up-both-chips)) and any object catalogs — changing steps/degree can corrupt the stored library.
2. Update `Config.h` as above, build and flash ([flashing.md](flashing.md)).
3. Power on with the motors connected and still (the drivers calibrate StealthChop at boot).
4. Move each axis slowly by hand control; fix direction with `AXISn_REVERSE`.
5. Test gotos at 2 °/s, then 4 °/s, with the scope mounted and balanced. Watch for stalls and driver-status faults.
6. Track for 30 minutes and do several gotos; check motor and driver temperature.
7. Calibrate steps/degree: plate-solve before and after a long goto (e.g. 90°) and correct the value. This is also the moment to settle the 100:1 vs 101:1 harmonic ratio question ([hardware.md](hardware.md#mechanics)).
8. Redo home and park positions, alignment, and PHD2 calibration.

## Sources

- MOONS' product pages: [MS17HA2P4150](https://www.moonsindustries.com/p/nema-17-high-precision-hybrid-stepper-motors/ms17ha2p4150-000004611110015918), [MS17HA2P4150-M](https://www.moonsindustries.com/p/nema-17-high-precision-hybrid-stepper-motors/ms17ha2p4150-m-000004611110027530) (specifications and torque-speed curves)
- Cloudy Nights, Juwei-17 thread: archcam's motor swap ([p.43](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/43/), [p.44](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/44/), [p.45](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/45/)), stock motor models ([p.21](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/21/))
