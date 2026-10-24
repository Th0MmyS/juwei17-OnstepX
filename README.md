# OnStepX Telescope Controller for Juwei 17

Advanced computerized telescope controller firmware for precision astronomical mount control.

**This configuration is specifically set up for the [Juwei 17 Harmonic Drive Mount](https://www.aliexpress.com/item/1005009419602903.html)** - a compact equatorial mount with harmonic drive technology providing high precision and minimal backlash. The Juwei 17 controller board is 1:1 pin-compatible with the MaxESP3 pinmap.

> ⚠️ **Disclaimer:** Any hardware or software modification described in this repository is performed **entirely at your own risk**. Flashing custom firmware, changing DIP switch settings, swapping stepper drivers (e.g. TMC2209 → TMC5160), adjusting axis travel limits, motor currents, or any other parameter in `Config.h` may damage your mount, controller board, motors, telescope, or void your warranty. Neither the author of this fork nor the upstream OnStepX authors accept responsibility for damage, injury, or loss resulting from its use. Verify every setting against your own hardware before powering on.

## At a glance

| | |
|---|---|
| Firmware | **OnStepX 10.27m** (upstream [hjd1964/OnStepX](https://github.com/hjd1964/OnStepX) commit `6f928aaf`) |
| Controller | Juwei 17 board — ESP32 (OnStepX) + ESP8266 (WiFi web server), `PINMAP MaxESP3` |
| Drivers | **TMC5160** over SPI on both axes (stock TMC2208/2209 replaced) |
| Mount | German equatorial (`GEM`), preferred pier side `EAST` |
| Connectivity | Bluetooth (`OnStepX`), USB, WiFi via Smart Web Server 2.09g |
| Time | DS3231 real-time clock |

Key settings (full explanation in [docs/configuration.md](docs/configuration.md)):

| Setting | RA (Axis 1) | DEC (Axis 2) |
|---|---|---|
| Steps per degree | 7163.4 | 7182.4 |
| Microsteps tracking / goto | 32 / 8 | 32 / 8 |
| Current hold / run / goto (mA, peak) | OFF (=350) / 700 / 1000 | OFF (=350) / 700 / 1000 |
| Limits | −190° … +190° | −10° … +90° |
| Driver status monitoring | ON | ON |

| Mount behaviour | |
|---|---|
| Slew speed / acceleration distance | 4 °/s / 6° |
| Step signal | `PULSE` |
| Meridian flips | automatic, routed through home |
| Motors in standby | powered (hold position) |
| Tracking at power-up | off |

## Documentation

| Guide | What's in it |
|---|---|
| [Flashing both chips](docs/flashing.md) | Backup, build and flash OnStepX (ESP32) and the web server (ESP8266), DIP switches, troubleshooting |
| [Configuration reference](docs/configuration.md) | Every customised `Config.h` setting and why: step signal, speed limits, acceleration, motor current, driver status, decay modes |
| [Upgrading to 10.27](docs/upgrading-to-10.27.md) | Why 10.27m, what changed, what to write down before flashing (settings are reset) |
| [Web server](docs/web-server.md) | Smart Web Server version, configuration, Advanced Configuration caveats |
| [Hardware notes](docs/hardware.md) | Motors, gearing, drivers, pin usage, RA brake, sources |

## Building

1. Install [Arduino IDE 2](https://www.arduino.cc/en/software) and the **ESP32 board package by Espressif, version 2.0.x** (tested with 2.0.11). Do not use 3.x with OnStepX 10.27.
2. Copy `OnStepX.ino`, `Config.h`, `Extended.config.h` and `src/` into a folder named **`OnStepX`** on a short path (e.g. `C:\OnStepX`). Arduino requires the folder name to match the `.ino`, and long Windows paths break the build.
3. Open `OnStepX.ino`, select **Tools → Board → ESP32 Dev Module**, and verify. It should build without warnings at about 92 % of program storage.

Or with `arduino-cli`:

```powershell
arduino-cli compile --fqbn esp32:esp32:esp32 C:\OnStepX
```

`src/plugins/Plugins.config.h` is part of this repository (all plugins off); the firmware does not build without it.

## Flashing the Firmware

Before flashing, you must set the board's DIP switches into bootloader mode to allow programming over USB. The full procedure for both chips, including backups and the web server, is in [docs/flashing.md](docs/flashing.md).

### DIP Switch Mapping
- **Switches 1, 2, 3**: Control the ESP8266 (`1 = RX`, `2 = TX`, `3 = IO0`)
- **Switches 4, 5, 6**: Control the ESP32 (`4 = IO0`, `5 = RX`, `6 = TX`)

### Standard vs. Flashing State

| Mode | Switch 1 | Switch 2 | Switch 3 | Switch 4 | Switch 5 | Switch 6 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **Normal Operation** | OFF | OFF | OFF | OFF | **ON** | **ON** |
| **ESP32 Flashing** | OFF | OFF | OFF ¹ | **ON** | **ON** | **ON** |

¹ If the ESP32 does not enter flash mode with switch 4 alone, also turn switch 3 on.

### Step-by-Step Flashing Procedure (ESP32 / OnStepX)

1. **Remove Board**: Take the controller board out of the mount enclosure.
2. **Disconnect Motors**: Ensure the motors are **not connected** to the board during flashing.
3. **Set Flashing Mode**: Flip DIP switch **4 to ON** (switches 5 and 6 remain ON; switches 1, 2, 3 remain OFF).
4. **Connect Power & USB**: Plug in your power source and connect the USB cable to your computer.
5. **Turn On**: Power on the board.
6. **Flash Firmware**: Upload the compiled OnStepX firmware using **Arduino IDE** or **esptool**.
7. **Restore Switches**: Once flashing is complete, turn off power, flip DIP switch **4 back to OFF**, reinstall the board, and reconnect your motors.

> **Upgrading from 10.22?** The first boot of 10.27 resets all settings stored on the mount (site, park, limits, backlash, rates). Write them down first — see [docs/upgrading-to-10.27.md](docs/upgrading-to-10.27.md). The web server on the ESP8266 should be updated to Smart Web Server 2.09g to get the per-axis settings page.

## Version

Current firmware: **v10.27m** (upstream [hjd1964/OnStepX](https://github.com/hjd1964/OnStepX) commit `6f928aaf`, 2026-02-25).

## Support

- **Discussion Group**: [OnStep Groups.io](https://onstep.groups.io/g/main)
- **OnStepX Wiki**: [onstep.groups.io wiki](https://onstep.groups.io/g/main/wiki)
- **OnStepX source**: [hjd1964/OnStepX](https://github.com/hjd1964/OnStepX)
- **Juwei 17 owners**: [Cloudy Nights thread](https://www.cloudynights.com/forums/topic/918425-juwei-17-mounts-on-aliexpress/)

## License

OnStepX is open source software licensed under the GNU General Public License v3.0.

See the `LICENSE` file for full license text.

## Author

**Howard Dutton** — author of OnStepX and the Smart Web Server
- Website: [Stellar Journey](http://www.stellarjourney.com)
- Email: hjd1964@gmail.com

---

*For detailed configuration of additional features including PEC, weather sensors, and other advanced options, refer to the heavily commented `Config.h` file and the online documentation.*

## Source

Juwei 17 flashing procedure and DIP switch mapping originally documented on the OnStep group: <https://onstep.groups.io/g/main/message/58916>
