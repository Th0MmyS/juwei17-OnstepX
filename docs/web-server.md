# Smart Web Server (WiFi)

The WiFi web interface does not run on the ESP32 with OnStepX. It runs on the board's separate **ESP8266**, which talks to OnStepX over a serial link (OnStepX `SERIAL_B` = ESP32 `Serial2`, GPIO16/17). The firmware is the OnStep **Smart Web Server (SWS)**. Step-by-step flashing is in [flashing.md](flashing.md#step-3--flash-the-web-server-on-the-esp8266).

## Which version

The Juwei 17 ships with an old SWS of unknown version. We use **upstream SWS 2.09g** ([hjd1964/SmartWebServer](https://github.com/hjd1964/SmartWebServer) commit `0d3aac0`, January 2026), the release from the same period as OnStepX 10.27m.

| OnStepX | SWS | Per-axis settings page |
|---|---|---|
| 10.22 | any | not available (old protocol) |
| 10.27m | 2.08 or older | not available |
| **10.27m** | **2.09g** | **available** |

Why exactly these:

- The per-axis *Advanced Configuration* (steps/degree, limits, reverse, microsteps, goto microsteps, mA hold/run/goto, decay modes) was redesigned in SWS 2.09a and only appears when OnStepX reports **10.26 or newer** (`src/pages/index/AxisTile.cpp` in the SWS). OnStepX 10.22 used an older command format, which is the main reason for [the 10.27 upgrade](upgrading-to-10.27.md).
- SWS 2.10 and later (released alongside OnStepX 10.28+) switched to a new settings storage system, which resets the SWS WiFi settings, and added OTA updates, which need about twice the sketch size in flash (tight on a 1 MB ESP8266). 2.09g is the conservative match for 10.27m.
- The MLAstro SAL-33 web server is SWS 2.09c with only its name changed (`HOST_NAME`) and two display options turned off. Its `Extended.config.h` also has a typo in `STA_GW_ADDR` (`{192,168,1,}`), so don't copy it — use upstream.

## Configuration

`Config.h`:

| Setting | Value | Note |
|---|---|---|
| `HOST_NAME` | `"Juwei17"` | Up to 16 characters. Also the WiFi access-point name (`AP_SSID` defaults to it). |
| `OPERATIONAL_MODE` | `WIFI` | |
| `SERIAL_BAUD_DEFAULT` | `9600` | Must match OnStepX `SERIAL_B_BAUD_DEFAULT` (9600). |
| `SERIAL_BAUD` | `115200` | Both sides switch to this after connecting. |
| `SERIAL_SWAP` | `AUTO` | |
| `DISPLAY_WEATHER` | `OFF` | No weather sensor fitted. |

`Extended.config.h`:

| Setting | Value | Note |
|---|---|---|
| `AP_PASSWORD` | your own, 8+ characters | WiFi password; default is `"password"`. |
| `PASSWORD_DEFAULT` | your own | Password for the settings pages. |
| `AP_IP_ADDR` | `{192,168,0,1}` (default) | Address of the web UI. |
| `NV_WIPE`, `DEBUG` | `OFF` | Debug output would go down the link to OnStepX. |

WiFi settings in `Config.h`/`Extended.config.h` are only applied on the first boot (or after the SWS memory is wiped); afterwards change them from the *Network* page.

## ESP8266 limitations

- **No mDNS** on ESP8266 builds — browse to **http://192.168.0.1**, not a `.local` name.
- 1 KB of settings storage, no Bluetooth gamepad, no ESP32-only encoder modes.
- WiFi range is short on the stock board (no external antenna).

## Using Advanced Configuration safely

Advanced Configuration lets you change axis settings from the browser without reflashing. Once enabled:

- OnStepX stores the axis settings in its own memory and **ignores the axis section of `Config.h`**, even after you reflash. To go back to `Config.h`, use *Revert to Defaults* per axis or *Disable Advanced Configuration*, then reboot.
- Most changes need an **OnStepX reboot** to take effect.
- An invalid value is rejected silently — re-check the page after *Upload*.
- Changing the ratio of RA steps per worm rotation to steps per degree can corrupt the stored object library — back up any catalogs first.

Remember that the mA fields are **peak** current (see [configuration.md](configuration.md#motor-current)).
