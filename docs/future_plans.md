# Future plans

Ideas for improving the Juwei 17, roughly in order of value for money. Tick items off as they are done.

**Suggested order:** 1 → 2 (+ FRAM, if bought before flashing) → 3 → 4.

## Free: settings and setup

- [ ] **1. Flash OnStepX 10.27m and Smart Web Server 2.09g.** *OnStepX 10.27m flashed 2026-10-28 (backup of the old firmware in `Documents\juwei17-backups`); web server still to do.*
  Unlocks the per-axis settings page in the web UI. The driver-status test failed on this board (no readback line), so `AXIS1/2_DRIVER_STATUS` is `OFF` ([configuration.md](configuration.md#driver-status)). Procedure: [flashing.md](flashing.md).

- [x] **2. Raise the motor currents.** *Done in `Config.h`; check temperatures after the first slews.*
  `AXISn_DRIVER_IRUN 1000` and `AXISn_DRIVER_IGOTO 1300` (was 700 / 1000, only ~480 / 670 mA RMS; now ~670 / 960 mA RMS). Owners settled around 900 mA; this gives more torque in reserve for gotos while staying below the motors' 1.3 A rating. Details: [configuration.md](configuration.md#motor-current).

- [ ] **3. Calibrate steps per degree with plate solving.**
  The gear ratio is disputed — store 400:1, factory firmware 404:1 ([hardware.md](hardware.md#mechanics)). With the current values, sync near DEC 0° low in the east, goto 60° west without crossing the meridian, then plate-solve without syncing:

  | RA result | Meaning | `AXIS1_STEPS_PER_DEGREE` (stock motors) | With MOONS' motors |
  |---|---|---|---|
  | overshoots by ~26′ | 400:1 | 7111.1 | 14222.2 |
  | lands ~9′ short | 404:1 | 7182.2 | 14364.4 |
  | within ~2′ | current value is right | keep 7163.4 | 14326.8 |

  For DEC (sync at DEC 0°, goto DEC +60° on the same pier side): overshoot of ~36′ means 400:1; on target means 404:1 (keep). General formula: new = old × commanded distance ÷ actual distance. Repeat a 30° and a 60° goto — a real steps/degree error grows with distance. If the motor swap is close, calibrate after it.

- [ ] **4. Check whether DEC backlash is real.**
  PHD2's Guiding Assistant reports 3–4 s of DEC backlash on these mounts even without mechanical play — a known quirk on strain-wave mounts. If DEC guiding looks fine, ignore it and don't enter compensation (it would cause overshoot). Only if the DEC graph shows corrections that lag after a direction change and then jump: check belt tension, then try a small PHD2 DEC backlash compensation.

- [x] **5. Enable PHD2's "Predictive PEC" guide algorithm for RA.** *Done.*
  Harmonic drives have a repeating periodic error in RA. Predictive PEC learns and cancels it while guiding — no hardware needed.

- [x] **6. Set limits and positions in the app.** *Done.*
  Horizon, overhead and meridian limits; then save the park and home positions. After the 10.27 flash these need re-entering, because the first boot resets stored settings.

## Cheap hardware (under ~$20)

- [ ] **FRAM memory chip** (MB85RC64 breakout, ~$5) on the I2C bus next to the DS3231 (GPIO21/22).
  Enables `MOUNT_COORDS_MEMORY ON` (the mount remembers where it points after a power cut) with `NV_DRIVER NV_MB85RC64`. Changing the memory device wipes stored settings, so **do it together with the 10.27 flash**, which wipes them anyway. Background: [configuration.md](configuration.md#left-at-default-on-purpose).

- [x] **Heatsinks on the TMC5160 drivers.** *Fitted.* They touch the DEC motor for lack of space: put Kapton tape between heatsink and motor (stops rubbing and heat flowing from motor to driver), or use low-profile (3–5 mm) heatsinks.

- [ ] **Mechanical check.** Belt tension; owners fixed altitude wobble by partially releasing the base plate and tightening the altitude bolts.

## Bigger projects

- [ ] **MOONS' 0.9° motors** (MS17HA2P4150-M / MS17HA2P4150). *Motors arrived; fitting waits until the park/brake logic is tested with the stock motors.* The configuration is ready in commit `c6454d8` (reverted for now). Plan: [motor-upgrade-plan.md](motor-upgrade-plan.md).

- [ ] **OnStepX 10.29+** once it has a few patch releases. Brings tracking/pulse-guide rate fixes and ESP32 core 3.x support; costs another settings reset and removing `LIMIT_STRICT` from `Config.h` ([upgrading-to-10.27.md](upgrading-to-10.27.md)).

## Sources

Owner reports from the Cloudy Nights [Juwei-17 thread](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/) (currents p.42, harmonic ratio p.67, MOONS' motor swap p.43–44, altitude wobble fix p.66) and the OnStepX 10.27m source in this repository.
