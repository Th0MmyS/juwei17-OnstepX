# Future plans

Ideas for improving the Juwei 17, roughly in order of value for money. Tick items off as they are done.

**Suggested order:** 1 → 2 (+ FRAM, if bought before flashing) → 3 → 4.

## Free: settings and setup

- [ ] **1. Flash OnStepX 10.27m and Smart Web Server 2.09g, then run the driver-status test.**
  Already prepared on this branch. Unlocks the per-axis settings page in the web UI and hardware-fault protection. Procedure: [flashing.md](flashing.md). If the mount reports a hardware fault right after flashing, set `AXIS1_DRIVER_STATUS` / `AXIS2_DRIVER_STATUS` to `OFF` ([configuration.md](configuration.md#driver-status)).

- [x] **2. Raise the motor currents.** *Done in `Config.h`; check temperatures after the first slews.*
  Stock motors: `IRUN 1000 / IGOTO 1300` (was 700 / 1000). With the MOONS' motors now `IRUN 1100 / IGOTO 1600` — ~770 / 1150 mA RMS, about half and three quarters of their 1.5 A rating. Details: [configuration.md](configuration.md#motor-current).

- [ ] **3. Calibrate steps per degree with plate solving** (after the motor swap).
  The gear ratio is disputed — store 400:1, factory firmware 404:1 ([hardware.md](hardware.md#mechanics)). With the current values, sync near DEC 0° low in the east, goto 60° west without crossing the meridian, then plate-solve without syncing:

  | RA result | Meaning | Set `AXIS1_STEPS_PER_DEGREE` to |
  |---|---|---|
  | overshoots by ~26′ | 400:1 | 14222.2 |
  | lands ~9′ short | 404:1 | 14364.4 |
  | within ~2′ | current value is right | keep 14326.8 |

  For DEC (sync at DEC 0°, goto DEC +60° on the same pier side): overshoot of ~36′ means 400:1 (14222.2); on target means 404:1 (keep). General formula: new = old × commanded distance ÷ actual distance. Repeat a 30° and a 60° goto — a real steps/degree error grows with distance.

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

- [ ] **MOONS' 0.9° motors** (MS17HA2P4150-M / MS17HA2P4150). *Motors arrived and `Config.h` is updated; fitting and testing still to do.* Checklist and test procedure: [motor-upgrade-plan.md](motor-upgrade-plan.md).

- [ ] **OnStepX 10.29+** once it has a few patch releases. Brings tracking/pulse-guide rate fixes and ESP32 core 3.x support; costs another settings reset and removing `LIMIT_STRICT` from `Config.h` ([upgrading-to-10.27.md](upgrading-to-10.27.md)).

## Sources

Owner reports from the Cloudy Nights [Juwei-17 thread](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/) (currents p.42, harmonic ratio p.67, MOONS' motor swap p.43–44, altitude wobble fix p.66) and the OnStepX 10.27m source in this repository.
