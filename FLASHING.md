# Flashing the Juwei 17: OnStepX and the web server

This is the complete procedure for putting this repository's firmware on a Juwei 17 controller board — step by step, with the exact commands that were used on 2026-10-28. It covers both chips on the board:

| Chip | Runs | Source in this repository |
|---|---|---|
| **ESP32** | **OnStepX 10.27m** — the mount controller | the repository root (`OnStepX.ino`, `Config.h`, `Extended.config.h`, `src/`) |
| **ESP8266** | **Smart Web Server 2.09g** — the WiFi web interface | the [`SmartWebServer/`](SmartWebServer) folder |

Plan about an hour. Most of it is waiting for backups.

> ⚠️ **At your own risk.** Flashing custom firmware may damage hardware or void your warranty (see the disclaimer in the [README](README.md)). Every chip is backed up first, and a failed flash can always be repeated or restored — but read each step before doing it.

> **Upgrading from 10.22 or the factory firmware?** The first boot of OnStepX 10.27 **erases all settings stored on the mount**. Write them down first: site (latitude, longitude, elevation, UTC offset), park position, horizon/overhead/meridian limits, backlash, preferred rates and pier side. Details: [docs/upgrading-to-10.27.md](docs/upgrading-to-10.27.md).

## Before you start

### What you need

- The mount's **12 V power supply** and a **USB cable** (the board has a USB port with a CH340 USB-serial chip).
- A screwdriver to take the controller board out of the mount.
- **Arduino IDE 2** ([download](https://www.arduino.cc/en/software)). It includes `arduino-cli` and is all the software you need.
- This repository on your computer (download the ZIP from GitHub or `git clone`).

### Install the two board packages

Open PowerShell and set two shortcuts (paste them again whenever you open a new PowerShell window):

```powershell
$cli     = "$env:LOCALAPPDATA\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe"
$esptool = "$env:LOCALAPPDATA\Arduino15\packages\esp32\tools\esptool_py\4.5.1\esptool.exe"
```

Install the **ESP32** package version **2.0.11** (not 3.x — OnStepX 10.27 needs 2.0.x on this board) and the **ESP8266** package for the web server:

```powershell
& $cli core install esp32:esp32@2.0.11 --additional-urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
& $cli core install esp8266:esp8266@3.1.2 --additional-urls https://arduino.esp8266.com/stable/package_esp8266com_index.json
```

The ESP32 package also installs `esptool`, which this guide uses for backups and flashing. (In the Arduino IDE you can install the same packages under *Tools → Board → Boards Manager* after adding both URLs in *File → Preferences → Additional boards manager URLs*.)

### Find the COM port

Plug in the USB cable, then:

```powershell
& $cli board list
```

The board shows up as **`Serial Port (USB)`**, e.g. `COM3`. Set it as a shortcut and replace `COM3` with yours:

```powershell
$port = "COM3"
```

Also make a folder for the backups, outside the repository:

```powershell
$backup = "$env:USERPROFILE\Documents\juwei17-backups"
New-Item -ItemType Directory -Force $backup | Out-Null
```

## DIP switch positions

The DIP switch block on the board selects which chip the USB port talks to, and whether that chip starts in flash mode. Switches 1, 2, 3 belong to the ESP8266 (RX, TX, IO0); switches 4, 5, 6 to the ESP32 (IO0, RX, TX).

| Mode | 1 | 2 | 3 | 4 | 5 | 6 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| **Normal operation** | OFF | OFF | OFF | OFF | **ON** | **ON** |
| **Flash ESP32** (OnStepX) | OFF | OFF | OFF | **ON** | **ON** | **ON** |
| **Flash ESP8266** (web server) | **ON** | **ON** | **ON** | OFF | OFF | OFF |

All three positions were confirmed on this board. If the ESP32 does not enter flash mode with switch 4, also try switch 3 on.

Two rules that save a lot of confusion:

1. **Flash mode is only read at power-up.** After changing switches, always power-cycle: 12 V off, wait two seconds, on.
2. **One esptool connection per power-up, unless you keep the speed the same.** The board cannot reset its chips over USB. After an esptool command ends, the chip keeps running esptool's helper at the speed that command used, and the next command can only reach it at that same speed. This guide therefore runs everything at `-b 115200` and only uses a faster speed for the last command before you change switches. If you ever see *"No serial data received"*, just power-cycle and repeat the command.

## Step 1 — Take the board out

1. Unplug the mount's power.
2. Take the controller board out of the mount.
3. **Disconnect both motor cables** from the board (label them RA and DEC first).
4. Leave the 12 V power plug and the USB cable available — you need both.

## Step 2 — Back up the ESP32 (OnStepX)

1. Set the switches to **Flash ESP32** (4, 5, 6 ON). Connect USB and 12 V, power on.
2. Check that esptool can reach the chip:

   ```powershell
   & $esptool --chip esp32 -p $port -b 115200 --after no_reset flash_id
   ```

   Expected: `Chip is ESP32-D0WD-V3`, `Detected flash size: 4MB`.
3. Read the whole flash into a file (about 6 minutes):

   ```powershell
   & $esptool --chip esp32 -p $port -b 115200 --after no_reset read_flash 0 0x400000 "$backup\esp32_backup.bin"
   ```
4. Check the file against the chip, then keep a second copy:

   ```powershell
   & $esptool --chip esp32 -p $port -b 115200 --after no_reset verify_flash 0 "$backup\esp32_backup.bin"
   Copy-Item "$backup\esp32_backup.bin" "$backup\esp32_backup_copy2.bin"
   ```

   Expected: `-- verify OK (digest matched)`. Do not continue until a backup verifies.

## Step 3 — Build OnStepX

Arduino needs the sketch folder to be named like the `.ino` file, and Windows path length limits break the build in deep folders. So build from a copy on a short path. Run this from the repository folder:

```powershell
New-Item -ItemType Directory -Force C:\OnStepX | Out-Null
Copy-Item -Recurse -Force OnStepX.ino, Config.h, Extended.config.h, src C:\OnStepX\
& $cli compile --fqbn esp32:esp32:esp32 --output-dir C:\OnStepX\out C:\OnStepX
```

Expected: `Sketch uses ... (92%) of program storage space`, no errors. `C:\OnStepX\out` now holds `OnStepX.ino.bootloader.bin`, `OnStepX.ino.partitions.bin` and `OnStepX.ino.bin`.

Don't open `OnStepX.ino` directly from the repository in the Arduino IDE: the IDE offers to "move" it into a new `OnStepX` subfolder, which breaks the repository layout.

## Step 4 — Flash OnStepX

The board is still in **Flash ESP32** mode from step 2 (if you power-cycled since, that's fine). This writes the four parts of the firmware and checks each one:

```powershell
$boot_app0 = "$env:LOCALAPPDATA\Arduino15\packages\esp32\hardware\esp32\2.0.11\tools\partitions\boot_app0.bin"
& $esptool --chip esp32 -p $port -b 921600 --after no_reset write_flash -z --flash_mode dio --flash_freq 80m --flash_size 4MB `
  0x1000 C:\OnStepX\out\OnStepX.ino.bootloader.bin `
  0x8000 C:\OnStepX\out\OnStepX.ino.partitions.bin `
  0xe000 $boot_app0 `
  0x10000 C:\OnStepX\out\OnStepX.ino.bin
```

Expected: four times `Hash of data verified.` (about 20 seconds).

*Arduino IDE alternative:* open `C:\OnStepX\OnStepX.ino`, select *Tools → Board → esp32 → ESP32 Dev Module* (the "DOIT ESP32 DEVKIT V1" profile works the same), choose the port, and click *Upload*.

## Step 5 — Check OnStepX

1. Power off. Set the switches to **Normal** (only 5 and 6 ON). Power on and wait a few seconds.
2. Ask the mount for its version and status:

   ```powershell
   $s = New-Object System.IO.Ports.SerialPort $port, 9600
   $s.Open(); Start-Sleep 1; $null = $s.ReadExisting()
   foreach ($c in ':GVN#', ':GU#') { $s.Write($c); Start-Sleep -Milliseconds 800; "$c -> $($s.ReadExisting())" }
   $s.Close()
   ```

   Expected: `:GVN# -> 10.27m#`, and the `:GU#` reply ending in **`0#`** (no error). A reply ending in `1#` means "motor fault" — see [Troubleshooting](#troubleshooting).

## Step 6 — Back up the ESP8266 (web server)

1. Power off. Set the switches to **Flash ESP8266** (1, 2, 3 ON; 4, 5, 6 OFF). Power on.
2. Check the chip and back it up (about 6 minutes), then verify and copy:

   ```powershell
   & $esptool --chip esp8266 -p $port -b 115200 --after no_reset flash_id
   & $esptool --chip esp8266 -p $port -b 115200 --after no_reset read_flash 0 0x400000 "$backup\esp8266_backup.bin"
   & $esptool --chip esp8266 -p $port -b 115200 --after no_reset verify_flash 0 "$backup\esp8266_backup.bin"
   Copy-Item "$backup\esp8266_backup.bin" "$backup\esp8266_backup_copy2.bin"
   ```

   Expected: `Chip is ESP8266EX`, `Detected flash size: 4MB`, and `-- verify OK (digest matched)`. If `flash_id` reports 1 MB instead, use `0x100000` for the backup and `eesz=1M` / `--flash_size 1MB` in steps 7 and 8.

## Step 7 — Build the web server

The [`SmartWebServer/`](SmartWebServer) folder is Smart Web Server 2.09g with the Juwei 17 settings already in `Config.h` (`HOST_NAME "Juwei17"`, which is also the WiFi network name). Optionally set your own passwords in `SmartWebServer\Extended.config.h` (`AP_PASSWORD`, 8+ characters, and `PASSWORD_DEFAULT`) — or change them later on the web page. Then, from the repository folder:

```powershell
Copy-Item -Recurse -Force SmartWebServer C:\
& $cli compile --fqbn "esp8266:esp8266:generic:eesz=4M,FlashMode=dout,xtal=80" --output-dir C:\SmartWebServer\out C:\SmartWebServer
```

Expected: no errors; `C:\SmartWebServer\out\SmartWebServer.ino.bin` is about 470 KB.

## Step 8 — Flash the web server

The board is still in **Flash ESP8266** mode. This erases the whole ESP8266 (which also resets its WiFi settings, so the new network name takes effect), writes the web server and checks it:

```powershell
& $esptool --chip esp8266 -p $port -b 460800 --after no_reset write_flash --erase-all --flash_mode dout --flash_size 4MB 0x0 C:\SmartWebServer\out\SmartWebServer.ino.bin
```

Expected: `Chip erase completed successfully` and `Hash of data verified.`

*Arduino IDE alternative:* open `C:\SmartWebServer\SmartWebServer.ino`, select *Tools → Board → esp8266 → Generic ESP8266 Module*, *Flash Size: 4MB (FS:none OTA:~1019KB)*, *Flash Mode: DOUT*, *Erase Flash: All Flash Contents*, then *Upload*.

> Don't use OnStepX's built-in ESP8266 flasher (`SERIAL_B_ESP_FLASHING` / `:ESPFLASH#`) on this board: the pins it needs are already used. Use the DIP switches.

## Step 9 — Check the web server

1. Power off. Set the switches to **Normal** (only 5 and 6 ON). Power on.
2. Join the WiFi network **`Juwei17`** (password `password` unless you changed it).
3. Open **http://192.168.0.1** (the ESP8266 web server has no `.local` name). The header should read **Smart Web Server 2.09g (OnStep10.27m)**.
4. Click **Enable Advanced Configuration**, then reboot OnStep (*Reboot OnStep!* or power-cycle). The axis tiles now show editable settings: steps/degree, limits, reverse, microsteps, mA hold/run/goto, decay modes, 256x interpolation.
5. On the **Network** page, change the WiFi password and the settings password.

Once Advanced Configuration is enabled, the mount uses the values stored on it and **ignores the axis section of `Config.h`, even after reflashing**. To return to `Config.h`, use *Revert to Defaults* (per axis) or disable Advanced Configuration, and reboot. More: [docs/web-server.md](docs/web-server.md).

## Step 10 — Put everything back

1. Power off. Check the switches are on **Normal**.
2. Reconnect the motor cables (RA and DEC as labelled) and reinstall the board.
3. Power on, connect with your app (Bluetooth name **`OnStepX`**, or WiFi `Juwei17`).
4. Re-enter site, date/time, limits and backlash; set and save the park position.
5. Test: move each axis slowly, then a goto at 4 °/s; park and unpark (listen for the brake click on park); power-cycle once while parked to confirm it boots normally. After a few slews check that drivers and motors are only warm.

## Restoring a backup

Put the chip into its flash mode (switch table above), power-cycle, then:

```powershell
& $esptool --chip esp32   -p $port -b 115200 write_flash 0 "$backup\esp32_backup.bin"
& $esptool --chip esp8266 -p $port -b 115200 write_flash 0 "$backup\esp8266_backup.bin"
```

A bad flash cannot brick either chip: their bootloaders are in read-only memory, so you can always enter flash mode and write again. While the ESP8266 has no working web server, the mount still works over Bluetooth and USB.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Failed to connect ... No serial data received` on the first command | The chip is not in flash mode or not powered: check 12 V is on, check the switches, **power-cycle**. For the ESP32, also try switch 3 on. |
| `No serial data received` on a later command | The previous command left the chip at a different speed. Power-cycle and repeat, or keep all commands at `-b 115200`. |
| Upload starts, then fails | Use `-b 115200`; try a shorter USB cable. |
| Build fails with "file name too long" or missing files | Build from a short path such as `C:\OnStepX` (step 3). |
| Build fails on `Plugins.config.h` | `src/plugins/Plugins.config.h` is part of this repository — copy the whole `src` folder. |
| The Arduino IDE moved `OnStepX.ino` into an `OnStepX` subfolder | Move it back to the repository root and delete the empty folder (`git status` shows it). Build from `C:\OnStepX` instead. |
| `:GU#` ends in `1#` (motor fault), gotos refused | Driver status is reading garbage. On this board it must be `OFF` (`AXIS1_DRIVER_STATUS` / `AXIS2_DRIVER_STATUS`) — the board has no readback line. See [docs/configuration.md](docs/configuration.md#driver-status). |
| Mount does nothing after the upgrade | Stored settings were reset: re-enter site, date/time and limits, then unpark. |
| Web page shows "Editing of this axis has been disabled" | The web server is older than 2.09 (it uses the old `:GXA1#` command). Flash 2.09g (steps 6–8), enable Advanced Configuration, reboot OnStep. |
| Web page loads but shows no mount data | OnStepX and the web server must both start at 9600 baud (`SERIAL_B_BAUD_DEFAULT` in OnStepX, `SERIAL_BAUD_DEFAULT` in the web server) with `SERIAL_SWAP AUTO` — the defaults in this repository. |
| Can't find the WiFi | Range is short (no antenna); stand next to the mount. After an erase the network is called `Juwei17`. |
