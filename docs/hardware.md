# Juwei 17 hardware notes

Facts about the mount and its controller that the configuration depends on. Most of this was collected from the owners' thread on Cloudy Nights and confirmed against the OnStepX source where possible (see [Sources](#sources)).

## Mechanics

- **Drive train (each axis):** stepper → 1:4 belt/pulley → harmonic (strain-wave) drive.
- **Reduction — not settled:**
  - The store listing says **400:1** ("double 17 harmonic reducer", 100:1 × 4 belt).
  - The factory firmware uses 3591.111 steps/degree at 16 microsteps, which is exactly **404:1** (a 101:1 harmonic).
  - punkgeek checked the harmonic drive serial numbers against the maker's data, found 100:1, and reported that switching fixed long gotos needing small final corrections.
  - Psittacula pointed out that Harmonic Drive's catalog gives **100:1 or 101:1 for the same gear set, depending on which element is held fixed**; fmendes believes the Juwei uses the 101:1 arrangement and runs the factory ratio without problems.

  With the MOONS' 0.9° motors at 32 microsteps the candidates are **14222.2** (400:1) and **14364.4** (404:1). This repository uses **RA 14326.8** and **DEC 14364.8** (the previous 7163.4 / 7182.4 doubled). A plate-solved goto settles it: see [future_plans.md](future_plans.md).
- **RA has no mechanical hard stop**, which is why RA limits are extended to ±190°. **DEC** can drive the telescope into the tripod, hence the −10° lower limit.
- **Backlash:** owners find no mechanical play, yet PHD2's Guiding Assistant reports 3–4 s of DEC backlash while guiding stays at ~0.6″. Owners regard this as a known PHD2 quirk on strain-wave mounts ("as long as your mount reacts fine during guiding, just ignore it"). A likely contributor is the small elastic wind-up of the harmonic gear and belt when DEC reverses. `GOTO_OFFSET` stays at 0.25° so gotos always approach from the same direction.

## RA brake

- The RA motor has an electromagnetic brake that is **released while 12 V is present and engages when power is lost** (you can hear it click when you unplug the mount).
- Owner correspondence says it is wired straight to 12 V, in which case firmware cannot control it and **parking does not engage the brake**. Not yet measured on this mount.

### Factory firmware: motors off when parked

The factory firmware (OnStep 4.24s on the ESP32, with the stock TMC2209 drivers in standalone mode) powers the motors down when a park completes by driving the shared enable pin **GPIO12 HIGH**, and drives it LOW again on unpark. That is the click heard on park and unpark. It drives no other pin, so parking engages the brake only if the board feeds the brake from the enable line.

`MOUNT_DISABLE_WHEN_PARKED ON` reproduces this in OnStepX ([configuration.md](configuration.md#mount-behaviour)). To find out whether it also engages the brake, measure the voltage across the brake coil unparked and then parked:

- about 12 V both times: the brake is wired straight to 12 V. Parking only switches the motors off, and RA is then held by the gear train alone.
- about 0 V when parked: the brake follows GPIO12, and parking engages it.

### Idea: engage the brake when parked

OnStepX has a `PARK_STATUS` output that drives a pin according to the park state ([Status.cpp:92](../src/telescope/mount/status/Status.cpp)). With

```cpp
#define PARK_STATUS                   LOW   // pin LOW when parked, HIGH otherwise
#define PARK_STATUS_PIN                25   // GPIO25 / AUX8, the only free output in this config
```

and a small logic-level N-MOSFET (e.g. AO3400, IRLZ44N) switching the brake's 12 V — gate resistor, pull-down, and a flyback diode across the brake coil — the brake is powered (released) while unparked and engages when parked, during boot, or without power. This needs a hardware change; check that AUX8 is reachable on the Juwei board first. Not implemented.

## Motors

| Axis | Model | Type | Rated current |
|---|---|---|---|
| RA | **MOONS' MS17HA2P4150-M** (double shaft) | NEMA 17, 0.9° (400 steps), 0.40 N·m | 1.5 A |
| DEC | **MOONS' MS17HA2P4150** | NEMA 17, 0.9° (400 steps), 0.40 N·m | 1.5 A |
| *stock RA (replaced)* | Sumtor 42HS4013A4 | NEMA 17, 1.8° (200 steps) | 1.3 A |
| *stock DEC (replaced)* | Sumtor 42HS4013B4 | NEMA 17, 1.8° (200 steps) | 1.3 A |

Details of the swap: [motor-upgrade-plan.md](motor-upgrade-plan.md). See [configuration.md](configuration.md#motor-current) for how OnStepX current settings map to these motors.

## Controller board

- **ESP32** runs OnStepX; **ESP8266** runs the Smart Web Server (WiFi). Both are flashed over the same USB port selected with DIP switches — see [flashing.md](flashing.md).
- **DS3231** real-time clock on I2C.
- Pin-compatible with the OnStepX **MaxESP3** pinmap.
- WiFi range is short (no external antenna).

### Stepper drivers

| | Stock | This build |
|---|---|---|
| Driver | TMC2208 (early units) / TMC2209 (later), generic modules | **TMC5160** (SPI) |
| Current setting | trimmer (Vref) on the module; ~350–400 mA early, ~750–850 mA later | from `Config.h` / web UI |
| Chopper | StealthChop only, no UART/SPI control | StealthChop tracking, SpreadCycle gotos |
| Practical slew speed | ~2.6–3 °/s | 4 °/s (tested by owners up to 5–6 °/s) |

Notes from owners who swapped to TMC5160:

- Early boards have the drivers **soldered in**; newer boards reportedly use pin headers.
- **BTT TMC5160 Pro** style modules may need their SPI-mode jumper bridged and the CLK pin cut before they work in SPI mode.
- No heatsinks are fitted at the factory and the board sits in a closed housing — add heatsinks.
- The default sense resistor in OnStepX is 0.075 Ω, matching BTT TMC5160 Pro modules.

### Pin usage (MaxESP3 pinmap, this configuration)

| GPIO | Used for | Alternatives in the pinmap |
|---|---|---|
| 0 | RA direction | — |
| 4 (AUX2) | TMC5160 SPI data out (MISO), both drivers → **driver status** ⚠️ wiring on the Juwei board unconfirmed, see [configuration.md](configuration.md#driver-status) | ESP8266 reset, focuser enable |
| 16 / 17 | Serial2 to the ESP8266 (web server) | — |
| 21 / 22 (AUX3/4) | I2C → DS3231 clock | home switches |
| 25 (AUX8) | **free** | status LED, buzzer, 1-Wire, `PARK_STATUS` |
| 26 | DEC direction | ESP8266 GPIO0 for `:ESPFLASH#` |
| 39 (AUX7) | free, **input only** | limit switch, PPS |

Because GPIO4 and GPIO26 are in use, OnStepX's built-in ESP8266 flasher (`SERIAL_B_ESP_FLASHING`) must stay `OFF`; flash the ESP8266 with the DIP switches.

## Power

12 V DC through a 5.5 × 2.1 mm centre-positive plug; owners recommend a supply rated for at least 3 A.

## Sources

- Cloudy Nights, ["Juwei-17 mounts on Aliexpress"](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/) — driver swap and currents ([p.3](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/3/)), motor models and gearing ([p.21](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/21/)), trimmer currents ([p.42](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/42/), [p.47](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/47/)), slew speeds ([p.44](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/44/), [p.55](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/55/)), TMC5160 module jumper ([p.61](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/61/)), summary of common mods ([p.66](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/66/)), harmonic ratio ([p.67](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/page/67/))
- punkgeek's Juwei 17 OnStepX configuration: [geeksville/OnStepX, E4 branch](https://github.com/geeksville/OnStepX/blob/E4/Config.h)
- [satakagi/OnStep-JUWEI-17](https://github.com/satakagi/OnStep-JUWEI-17) — open controller design, gear ratio notes
- RA brake behaviour: owner correspondence (j21blackjack)
- Flashing procedure and DIP switch mapping: [OnStep group message 58916](https://onstep.groups.io/g/main/message/58916)
