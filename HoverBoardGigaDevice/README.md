# Hoverboard Gen2 firmware — PlatformIO on Windows

This folder contains the GD32F130C8 firmware and a [PlatformIO](https://platformio.org/) project. Follow these steps on a **new Windows PC** to build and upload firmware with PlatformIO.

## What you need

### Software

1. **Git** — [https://git-scm.com/download/win](https://git-scm.com/download/win)
2. **Visual Studio Code** — [https://code.visualstudio.com/](https://code.visualstudio.com/)
3. **PlatformIO IDE extension** for VS Code  
   - Open VS Code → Extensions (`Ctrl+Shift+X`) → search **PlatformIO IDE** → Install  
   - The first build will download the ARM toolchain and STM32 platform automatically.

### Hardware

- Hoverboard mainboard (GD32F130C8T6)
- **ST-Link V2** programmer (genuine or common Chinese clone)
- 3 wires for SWD: **GND**, **SWDIO**, **SWCLK**
- USB cable for the ST-Link

The repo already includes a working Windows flasher at:

`tools/stlink-1.7.0/stlink-1.7.0-x86_64-w64-mingw32/bin/st-flash.exe`

You do **not** need to install st-flash separately on Windows after cloning this repo.

---

## 1. Clone the repository

```powershell
git clone https://github.com/GilMe/Hoverboard-Firmware-Hack-Gen2.git
cd Hoverboard-Firmware-Hack-Gen2\HoverBoardGigaDevice
```

> **Important:** Open the `HoverBoardGigaDevice` folder in VS Code, not the repository root. PlatformIO expects `platformio.ini` in the workspace root.

In VS Code: **File → Open Folder…** → select `HoverBoardGigaDevice`.

---

## 2. Install the ST-Link USB driver (Windows)

Chinese ST-Link V2 clones often need a driver before Windows can see the programmer.

**Option A — ST official driver (recommended first try)**  
Download and install [STSW-LINK009](https://www.st.com/en/development-tools/stsw-link009.html) from STMicroelectronics.

**Option B — Zadig (if Option A does not work)**  
1. Download [Zadig](https://zadig.akeo.ie/)  
2. Connect the ST-Link  
3. Select the ST-Link device → install **WinUSB** driver

After installation, plug in the ST-Link and check that it appears in Device Manager without a warning icon.

---

## 3. Wire the mainboard

Connect the ST-Link to the hoverboard SWD header:

| ST-Link pin | Mainboard pin |
|-------------|---------------|
| GND         | GND           |
| SWDIO       | SWDIO (sometimes labelled DIO) |
| SWCLK       | SWCLK (sometimes labelled CLK) |

**Power during flashing**

- The GD32 must be powered while you flash. Use the hoverboard battery **or** the 3.3 V output from the ST-Link if your board supports it.
- **Hold the hoverboard power button** while uploading. The board can switch itself off during programming if you do not.

There are **two** mainboards in a hoverboard. Flash them **one at a time** with the same firmware.

---

## 4. Unlock the chip (first flash only)

If this mainboard has never been flashed with custom firmware, the GD32 flash may be **read-protected**. Unlock it once using one of these tools:

- [STM32CubeProgrammer](https://www.st.com/en/development-tools/stm32cubeprog.html) — connect → Option Bytes → disable read protection  
- Legacy ST-LINK Utility  
- OpenOCD (if installed via PlatformIO packages):

```powershell
& "$env:USERPROFILE\.platformio\packages\tool-openocd\bin\openocd.exe" `
  -f interface/stlink.cfg -f target/stm32f1x.cfg `
  -c "init; reset halt; stm32f1x unlock 0; shutdown"
```

You only need to do this once per board (unless protection is turned on again).

---

## 5. Build with PlatformIO

### Using the VS Code UI

1. Open the `HoverBoardGigaDevice` folder in VS Code  
2. Wait for PlatformIO to finish loading (status bar shows a checkmark)  
3. Click the **PlatformIO icon** in the left sidebar (alien head)  
4. Under **PROJECT TASKS → GD32F130C8T6 → General**, click **Build**

### Using the terminal

From the `HoverBoardGigaDevice` folder:

```powershell
pio run
```

The first build downloads packages and may take several minutes.

A successful build ends with something like:

```
RAM:   [          ]   3.0% (used 244 bytes from 8196 bytes)
Flash: [==        ]  21.3% (used 13944 bytes from 65536 bytes)
========================= [SUCCESS] =========================
```

Build output is written to `.pio/build/GD32F130C8T6/firmware.bin`.

---

## 6. Upload with PlatformIO

This project uses a **custom upload command** in `platformio.ini` because the default OpenOCD/ST-Link upload fails on GD32F130 chips (HardFault during flash programming). Upload is handled by **st-flash v1.7.0**, which is bundled in `tools/`.

### Using the VS Code UI

1. Connect ST-Link and mainboard (see wiring above)  
2. **Hold the hoverboard power button**  
3. Under **PROJECT TASKS → GD32F130C8T6 → General**, click **Upload**

If you only see a “remote upload” task, open the full task list: **Terminal → Run Task…** and choose the local **PlatformIO: Upload** task.

### Using the terminal

```powershell
pio run --target upload
```

A successful upload ends with:

```
Flash written and verified! jolly good!
========================= [SUCCESS] =========================
```

---

## 7. How upload is configured

In `platformio.ini`:

```ini
upload_protocol = custom
upload_command = "$PROJECT_DIR/tools/stlink-1.7.0/stlink-1.7.0-x86_64-w64-mingw32/bin/st-flash.exe" write "$BUILD_DIR/firmware.bin" 0x08000000
```

PlatformIO builds `firmware.bin`, then runs the bundled `st-flash.exe` to write it to address `0x08000000`.

Do **not** switch back to `upload_protocol = stlink` unless you have verified OpenOCD upload works on your hardware (it usually does not on GD32F130).

---

## Troubleshooting

### “Cannot find a linker script” warning

This warning can appear during build but the firmware still links and uploads correctly. It is safe to ignore for now.

### Upload fails immediately / ST-Link not found

- Re-seat USB and SWD wires  
- Install or reinstall the ST-Link driver (step 2)  
- Try a different USB port (directly on the PC, not through a hub)  
- Unplug and replug the ST-Link

### OpenOCD HardFault / “error waiting for target flash write algorithm”

You are using the default ST-Link upload instead of the custom `st-flash` command. Confirm `platformio.ini` still has `upload_protocol = custom` and that `tools/stlink-1.7.0/.../st-flash.exe` exists.

### Upload stops partway through (e.g. 5/14 pages)

- Use the bundled **st-flash v1.7.0** (not the old flasher inside PlatformIO’s packages)  
- Hold the power button during the entire upload  
- Run mass erase once, then upload again:

```powershell
& ".\tools\stlink-1.7.0\stlink-1.7.0-x86_64-w64-mingw32\bin\st-flash.exe" erase
pio run --target upload
```

### Chinese ST-Link clone still unreliable

The clone is often good enough for SWD if the driver is correct. If problems persist after following this guide, try a genuine ST-Link V2 or a J-Link.

---

## Project layout

| Path | Purpose |
|------|---------|
| `platformio.ini` | PlatformIO environment and upload settings |
| `Src/` | Application source code |
| `Inc/` | Headers |
| `boards/` | GD32 board definitions |
| `tools/stlink-1.7.0/` | Windows st-flash used for upload |
| `.pio/` | Build output (generated, not in git) |

---

## Alternative: Keil

The original author also provided a Keil µVision project (`Hoverboard.uvprojx`). PlatformIO is the recommended route for new setups on Windows; Keil is optional.
