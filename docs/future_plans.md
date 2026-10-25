# Future plans

Ideas for improving the Juwei 17, roughly in order of value for money. Tick items off as they are done.

**Suggested order:** 1 → 2 (+ FRAM, if bought before flashing) → 3 → 4 → 5.

## Free: settings and setup

- [ ] **1. Flash OnStepX 10.27m and Smart Web Server 2.09g, then run the driver-status test.**
  Already prepared on this branch. Unlocks the per-axis settings page in the web UI and hardware-fault protection. Procedure: [flashing.md](flashing.md). If the mount reports a hardware fault right after flashing, set `AXIS1_DRIVER_STATUS` / `AXIS2_DRIVER_STATUS` to `OFF` ([configuration.md](configuration.md#driver-status)).

- [ ] **2. Raise the motor currents.**
  `AXISn_DRIVER_IRUN 1000` and `AXISn_DRIVER_IGOTO 1300` (now 700 / 1000, which is only ~480 / 670 mA RMS). Owners settled around 900 mA; this gives more torque in reserve for gotos while staying below the motors' 1.3 A rating. Check driver and motor temperature after a few slews. Details: [configuration.md](configuration.md#motor-current).

- [ ] **3. Calibrate steps per degree with plate solving.**
  Solve, make a long goto (~90°), solve again, and correct `AXIS1/2_STEPS_PER_DEGREE`. One owner found the harmonic drive is 100:1 rather than the 101:1 the factory firmware assumes, which fixed gotos needing small final corrections ([hardware.md](hardware.md#mechanics)).

- [ ] **4. Measure and compensate DEC backlash.**
  Owners see DEC backlash in PHD2's Guiding Assistant. Set OnStepX's backlash value (app / web UI) or use PHD2's DEC backlash compensation. `GOTO_OFFSET 0.25` already makes gotos approach from the same direction.

- [ ] **5. Enable PHD2's "Predictive PEC" guide algorithm for RA.**
  Harmonic drives have a repeating periodic error in RA. Predictive PEC learns and cancels it while guiding — no hardware needed.

- [ ] **6. Set limits and positions in the app.**
  Horizon, overhead and meridian limits; then save the park and home positions. Prevents collisions, which matters more now that meridian flips are automatic.

- [ ] **7. Refraction-compensated tracking.**
  `TRACK_COMPENSATION_DEFAULT REFRACTION`. Better tracking near the horizon; mostly useful for unguided and planetary imaging.

- [ ] **8. Remember the pointing model** — only if the mount stays on a fixed pier.
  `ALIGN_MODEL_MEMORY ON` keeps the alignment model across power cycles.

## Cheap hardware (under ~$20)

- [ ] **FRAM memory chip** (MB85RC64 breakout, ~$5) on the I2C bus next to the DS3231 (GPIO21/22).
  Enables `MOUNT_COORDS_MEMORY ON` (the mount remembers where it points after a power cut) with `NV_DRIVER NV_MB85RC64`. Changing the memory device wipes stored settings, so **do it together with the 10.27 flash**, which wipes them anyway. Background: [configuration.md](configuration.md#left-at-default-on-purpose).

- [ ] **Heatsinks on the TMC5160 drivers.** The board sits in a closed housing and has none from the factory.

- [ ] **WiFi antenna mod.** The stock board has no antenna; range is a few metres.

- [ ] **Mechanical check.** Belt tension; owners fixed altitude wobble by partially releasing the base plate and tightening the altitude bolts.

- [ ] **Good 12 V supply, ≥ 3 A.** Brown-outs cause resets and lost position.

## Bigger projects

- [ ] **RA brake engages when parked.** `PARK_STATUS` output on GPIO25 driving a logic-level MOSFET; design in [hardware.md](hardware.md#idea-engage-the-brake-when-parked).

- [ ] **Home sensors** for automatic homing after a power loss. On the stock board the home-switch pins are shared with the DS3231 clock's I2C bus.

- [ ] **PEC index sensor** to use OnStepX's own periodic-error correction (needs `PEC_STEPS_PER_WORM_ROTATION` and `PEC_SENSE`).

- [ ] **MOONS' 0.9° motors** (MS17HA2P4150-M / MS17HA2P4150). Full plan: [motor-upgrade-plan.md](motor-upgrade-plan.md).

- [ ] **FYSETC E4 controller board** — only worth it for the extras: reliable WiFi with an external antenna, built-in MOSFET outputs (brake on park, dew heaters), inputs for home/limit/PEC sensors, 12–24 V supply. Costs: weaker TMC2209 drivers than the current TMC5160s, no DS3231 clock, WiFi *or* Bluetooth (not both), and mounting/wiring work.

- [ ] **OnStepX 10.29+** once it has a few patch releases. Brings tracking/pulse-guide rate fixes and ESP32 core 3.x support; costs another settings reset and removing `LIMIT_STRICT` from `Config.h` ([upgrading-to-10.27.md](upgrading-to-10.27.md)).

## Sources

Owner reports from the Cloudy Nights [Juwei-17 thread](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/) (currents p.42, harmonic ratio and E4 p.67, PEC/home sensors p.43–44, altitude wobble fix p.66) and the OnStepX 10.27m source in this repository.
