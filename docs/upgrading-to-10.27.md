# Upgrading from OnStepX 10.22 to 10.27

This repository moved from OnStepX **10.22v** to **10.27m** in October 2026. This page explains why, what changed, and what to do on the mount. The step-by-step flashing procedure is in [FLASHING.md](../FLASHING.md).

## Why 10.27m, and why from upstream

- **The web server's per-axis settings page needs OnStepX 10.26 or newer.** 10.22 uses an older command format that current Smart Web Server versions no longer speak. See [web-server.md](web-server.md).
- **Upstream 10.27m** ([hjd1964/OnStepX](https://github.com/hjd1964/OnStepX) commit `6f928aafe36dbb70bd0a01af9c79f01862f423e7`, 2026-02-25) is the final 10.27 patch.
- The MLAstro SAL-33 firmware is unmodified upstream **10.27j** plus its own `Config.h`. 10.27m is newer: it fixes a build error with `SERIAL_B_ESP_FLASHING ON` and reverts an ESP32 step-timer change that affects ESP32 core 2.x boards like this one.
- **Not 10.28+ yet:** 10.28 introduced a new settings storage system (another settings reset) and ESP32 core 3.x support. It can be a later step.

Before the upgrade, `src/` was identical to upstream 10.22v, so nothing custom was lost by replacing it.

## What changed in this repository

| File | Change |
|---|---|
| `src/`, `OnStepX.ino` | Replaced with upstream 10.27m (415 source files; driver, HAL and storage code reorganised) |
| `Config.h` | All Juwei 17 settings merged onto the 10.27m template. New: `HOST_NAME "OnStepX"`, `NV_DRIVER NV_DEFAULT` (same as `NV_DEF`), `MOUNT_ALTERNATE_ORIENTATION OFF`, `AUTOMATIC` pier-side option |
| `Extended.config.h` | 10.27m template (adds `DEBUG_CAN OFF`) |
| `src/plugins/Plugins.config.h` | 10.27m template (adds `PLUGINn_COMMAND_PROCESSING`) |

New in 10.27 that is worth knowing (all default `OFF`, set in `Config.h` if wanted):

- `LIMIT_RECOVERY` — allows short gotos back out of a horizon, meridian or RA limit.
- `TRACK_WITHOUT_LIMITS` — by default 10.27 stops tracking if limits are disabled.
- `AXISn_DRIVER_RSENSE` — sense-resistor value for non-standard driver modules.
- Axis settings (steps/degree, limits, microsteps, currents, decay) are runtime parameters editable from the web server.

The motor current maths for the TMC5160 is unchanged, so `IRUN`/`IGOTO` values carry over as-is.

## Before you flash

1. **Write down everything stored on the mount.** 10.27 uses a new storage key (`INIT_NV_KEY` 583928939 → 583928945), so **the first boot wipes all stored settings**:
   - site latitude, longitude, elevation, UTC offset
   - park position
   - horizon/overhead and meridian limits
   - backlash (RA and DEC)
   - preferred rates, pier side, meridian-flip options
   - alignment model, PEC data
2. **Back up both chips** ([FLASHING.md, step 2](../FLASHING.md#step-2--back-up-the-esp32-onstepx)).
3. **Use ESP32 board package 2.0.x** (tested with 2.0.11).

The Bluetooth name stays **`OnStepX`** (`HOST_NAME`), so paired devices keep working.

## After flashing

1. Connect (Bluetooth or USB) and check the version reads **10.27m**.
2. Re-enter site, date/time, limits and backlash.
3. Set and save the park position again, then test park/unpark and a meridian flip.
4. Upgrade the web server to SWS 2.09g ([FLASHING.md, step 7](../FLASHING.md#step-7--build-the-web-server)).

## Rolling back

- The commit before the upgrade (`e7ce021`, "Tune slewing and driver settings…") is 10.22v with the same tuning. Build and flash it as usual, or restore the backup `.bin` with esptool.
- Going back to 10.22 also resets stored settings, because its storage key differs.
