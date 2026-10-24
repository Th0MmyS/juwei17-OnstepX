# Flashing both chips

The Juwei 17 controller board has **two microcontrollers** that are flashed separately:

| Chip | Runs | Source |
|---|---|---|
| **ESP32** | OnStepX — the mount controller | this repository |
| **ESP8266** | Smart Web Server (SWS) — the WiFi web UI | [hjd1964/SmartWebServer](https://github.com/hjd1964/SmartWebServer), see [web-server.md](web-server.md) |

Both share one USB port. The **DIP switch block** on the controller board decides which chip the USB port talks to and whether that chip boots into flash mode.

**Always flash with the board taken out of the mount and the motors disconnected.** Plug in power and USB only.

> ⚠️ Flashing is at your own risk (see the disclaimer in the [README](../README.md)). A failed flash is recoverable over USB as long as you can put the chip back into flash mode, and the backups in step 1 let you return to the factory firmware.

## DIP switch positions

Switches 1, 2, 3 belong to the ESP8266 (RX, TX, IO0); switches 4, 5, 6 belong to the ESP32 (IO0, RX, TX).

| Mode | 1 (8266 RX) | 2 (8266 TX) | 3 (8266 IO0) | 4 (ESP32 IO0) | 5 (ESP32 RX) | 6 (ESP32 TX) |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| **Normal operation** | OFF | OFF | OFF | OFF | **ON** | **ON** |
| **Flash ESP32** (OnStepX) | OFF | OFF | OFF ¹ | **ON** | **ON** | **ON** |
| **Flash ESP8266** (web server) ² | **ON** | **ON** | **ON** | OFF | OFF | OFF |

¹ Normally only switch 4 is needed. If the ESP32 will not enter flash mode, also switch 3 on.
² As reported by owners on Cloudy Nights ("4, 5, 6 on and 1, 2, 3 off for the ESP32, the opposite for the ESP8266"). Check it against the labels printed on your board.

The IO0 switches (3 and 4) only take effect at power-up: **always power-cycle after changing switches.**

## Recommended order

1. Write down the settings stored on the mount (only needed when the OnStepX upgrade resets them, e.g. 10.22 → 10.27 — see [upgrading-to-10.27.md](upgrading-to-10.27.md)).
2. Back up both chips.
3. Flash OnStepX on the ESP32 and check it works.
4. Flash the web server on the ESP8266 and check it works.
5. Return the DIP switches to normal.

Doing the ESP32 first means you can confirm the mount works over Bluetooth/USB before touching WiFi.

## What you need

- A USB cable and the mount's 12 V supply.
- [Arduino IDE 2](https://www.arduino.cc/en/software) (or `arduino-cli`).
- **ESP32 board package `esp32` by Espressif, version 2.0.x** (tested with 2.0.11). Do not use 3.x with OnStepX 10.27: MaxESP3 support for core 3.x only arrived in 10.28.
- **ESP8266 board package `esp8266` by ESP8266 Community, 3.1.x** — only for the web server. Add `https://arduino.esp8266.com/stable/package_esp8266com_index.json` under *File → Preferences → Additional boards manager URLs*, then install it from the Boards Manager.
- The COM port of the board: plug it in and look under *Device Manager → Ports (COM & LPT)*.

The commands below use `esptool.exe`, which the ESP32 board package already installs at:

```
%LOCALAPPDATA%\Arduino15\packages\esp32\tools\esptool_py\4.5.1\esptool.exe
```

Replace `COM5` with your port. In PowerShell, set a shortcut first:

```powershell
$esptool = "$env:LOCALAPPDATA\Arduino15\packages\esp32\tools\esptool_py\4.5.1\esptool.exe"
```

## Step 1 — Back up both chips

Do this once, before the first flash. Keep the `.bin` files somewhere safe.

**ESP32:** set the DIP switches to *Flash ESP32*, power-cycle the board, then:

```powershell
& $esptool --chip esp32 -p COM5 flash_id
& $esptool --chip esp32 -p COM5 read_flash 0 0x400000 juwei17_esp32_backup.bin
```

`flash_id` prints the flash size; use `0x400000` for 4 MB (the usual size) or adjust to match.

**ESP8266:** set the DIP switches to *Flash ESP8266*, power-cycle, then:

```powershell
& $esptool --chip esp8266 -p COM5 flash_id
& $esptool --chip esp8266 -p COM5 read_flash 0 0x100000 juwei17_esp8266_backup.bin
```

Use `0x100000` for a 1 MB module or `0x400000` for 4 MB, as reported by `flash_id`. **Note the flash size** — you need it in step 3.

## Step 2 — Flash OnStepX on the ESP32

1. **Prepare the sketch folder.** Arduino requires the folder name to match the `.ino` file, and this repository's folder is not called `OnStepX`. Copy `OnStepX.ino`, `Config.h`, `Extended.config.h` and the `src` folder into a new folder named `OnStepX` (keep the path short, e.g. `C:\OnStepX`, to stay under the Windows 260-character path limit). Do not let the IDE "move" files inside the repository.
2. **Open** `OnStepX\OnStepX.ino` in Arduino IDE.
3. **Select the board:** *Tools → Board → esp32 → ESP32 Dev Module*. Leave the defaults (4 MB flash, default partition scheme). Upload speed 921600 works; use 115200 if uploads fail.
4. **Verify** (✓ button). It should build without errors and use about 92 % of program storage.
5. **Take the board out of the mount and disconnect the motors.** Set the DIP switches to *Flash ESP32* (switch 4 on), connect power and USB, power on.
6. **Upload** (→ button). Wait for `Hard resetting via RTS pin...`.
7. **Power off, set the DIP switches back to *Normal*** (switch 4 off), reinstall the board, reconnect the motors, power on.

With `arduino-cli` instead of the IDE (steps 3–6):

```powershell
arduino-cli compile --fqbn esp32:esp32:esp32 C:\OnStepX
arduino-cli upload  --fqbn esp32:esp32:esp32 -p COM5 C:\OnStepX
```

**Check it:** connect over Bluetooth to **`OnStepX`** (or USB at 9600 baud) with your usual app/ASCOM driver. The firmware version should read **10.27m**. After an upgrade from 10.22, re-enter site, date/time, park position, limits and backlash.

## Step 3 — Flash the web server on the ESP8266

1. **Get the source** for SWS 2.09g (the release that matches OnStepX 10.27m):
   `https://github.com/hjd1964/SmartWebServer/archive/0d3aac0bc98458b67ce624a2575bc3763d03407c.zip`
   Extract it and rename the folder to **`SmartWebServer`** (again short path, e.g. `C:\SmartWebServer`).
2. **Configure it** — the values are explained in [web-server.md](web-server.md#configuration):
   - `Config.h`: `HOST_NAME "Juwei17"` (this is also the WiFi network name); leave `OPERATIONAL_MODE WIFI`, `SERIAL_BAUD_DEFAULT 9600`, `SERIAL_BAUD 115200`, `SERIAL_SWAP AUTO`.
   - `Extended.config.h`: change `AP_PASSWORD` (8+ characters) and `PASSWORD_DEFAULT`.
3. **Select the board:** *Tools → Board → esp8266 → Generic ESP8266 Module*. Set *Flash Size* to the size `flash_id` reported in step 1 (1 MB or 4 MB). For the first flash set *Erase Flash* to *All Flash Contents*. Upload speed 115200.
4. **Verify** (✓ button).
5. **With the board out of the mount and the motors disconnected,** set the DIP switches to *Flash ESP8266*, connect power and USB, power on.
6. **Upload** (→ button).
7. **Power off, set the DIP switches back to *Normal*,** reinstall the board, reconnect the motors, power on.

**Check it:** join the WiFi network **`Juwei17`** and open **http://192.168.0.1** (the ESP8266 build does not start mDNS, so use the IP). The page header should show **Smart Web Server 2.09g (OnStep10.27m)**.

> Do **not** use OnStepX's built-in ESP8266 flasher (`SERIAL_B_ESP_FLASHING` / `:ESPFLASH#`) on this board: the pins it needs are GPIO4 (used for TMC5160 driver status) and GPIO26 (the DEC direction signal). Use the DIP switches.

## Restoring a backup

Put the chip into flash mode as above, then write the backup back:

```powershell
& $esptool --chip esp32 -p COM5 write_flash 0 juwei17_esp32_backup.bin
& $esptool --chip esp8266 -p COM5 write_flash 0 juwei17_esp8266_backup.bin
```

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Failed to connect ... Wrong boot mode` | The chip is not in flash mode: check the DIP switches for that chip and **power-cycle**. For the ESP32, try switch 3 on as well as switch 4. |
| Upload starts then fails | Lower the upload speed to 115200; use a shorter/better USB cable. |
| Build fails with "file name too long" or missing files | Move the sketch folder to a short path such as `C:\OnStepX`. |
| Build fails on `Plugins.config.h` | The `src/plugins/Plugins.config.h` file is missing; it is part of this repository. |
| Mount does nothing after an OnStepX upgrade | Stored settings were reset; re-enter site, date/time and limits, then unpark. |
| Web page loads but shows no mount data | Check both sides use 9600 baud as the starting rate (`SERIAL_B_BAUD_DEFAULT` in OnStepX, `SERIAL_BAUD_DEFAULT` in the SWS) and `SERIAL_SWAP AUTO`. |
| Mount won't move, "hardware fault" | Driver status found a fault (over-temperature or shorted coil). Check wiring and cooling; see [configuration.md](configuration.md#driver-status). |
