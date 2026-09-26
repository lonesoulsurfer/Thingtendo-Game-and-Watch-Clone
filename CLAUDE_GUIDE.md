# Build Assistant Guide: ESP32 Game & Watch Handheld

## For people using this file

If you are new to ESP32 projects, Linux or building firmware, you can get step by step help from Claude (or another AI assistant):

1. Start a new chat with Claude.
2. Upload this file (`CLAUDE_GUIDE.md`).
3. Tell it where you are up to, for example: "I have all the parts and want to build the firmware. I use Windows."

It will walk you through one step at a time and help you fix problems along the way.

Everything below this line is written for the assistant.

---

## Instructions for the assistant

You are helping someone build the ESP32 Game & Watch handheld from this repository. They may be a beginner. Follow these rules:

- **Go one step at a time.** Give one step, ask the user to run it and paste the output, then check the output before moving on. Do not dump every step at once.
- **Verify, do not assume.** After each command, check the output for errors, wrong file sizes, wrong names or old timestamps. Many problems in this build come from a file that did not copy, or a command run in the wrong folder or the wrong terminal.
- **Ask about their setup first:** Windows, Mac or Linux; whether they use a Linux virtual machine; whether they have soldered the hardware yet; and what they want to do (flash, build, add games, fix a problem).
- **Explain briefly and plainly.** Say what a step does in one sentence, without jargon.
- **Do not help find or download ROMs or artwork.** The user must supply their own. You can explain what files are needed and how to convert them.
- **Never ask for secrets.** If the user shares a GitHub token or password, tell them to revoke it and make a new one.
- **Say when you are unsure,** and suggest a way to check rather than guessing.

## Project overview

A handheld that plays Nintendo Game & Watch games on an ESP32-S3. It is a fork of [slowlane112/Esp32-Game-and-Watch](https://github.com/slowlane112/Esp32-Game-and-Watch) and adds a new firmware model called `single_screen_dpad`.

Features added in this fork:
- 13 games on one device
- Pause, continue after a game over, and a per-game colour mode
- A two-button layout for the games that originally had only a left and a right button
- Support for the ST7789V display

## Repository layout

| Path | Contents |
|---|---|
| `gandw_handheld/` | Firmware project (ESP-IDF) |
| `gandw_handheld/main/` | Firmware source. The `.gw` game files also go here (not included) |
| `components/lcd_game_emulator/` | The Game & Watch emulator |
| `assets/menu/` | Menu icon images and scripts from the original project |
| `Kicad_GW/`, `Kicad_Panel_Design/` | KiCad files for the main board, front panel and back panel |
| `Gerber_Files/` | Gerbers ready to send to a PCB maker |
| `Schematic.pdf` | Schematic |

## Hardware

- Waveshare ESP32-S3-Zero
- 2.4" 240x320 SPI TFT with ST7789V driver (tested with GMT024-08-SPI8P, which has no backlight pin)
- MAX98357A I2S amplifier module and a 4 to 8 ohm speaker
- USB-C charger / 5V boost module (18 x 23.6 mm) and a single cell lithium battery
- SPDT power switch
- 8 tactile switches: Left, Right, Up, Down, Jump, Game A, Game B, Time
- 2 x 100uF and 1 x 100nF capacitors
- 2.54mm male and female headers

## Wiring

### Buttons (each button between the GPIO and GND, internal pull-ups, no resistors)

| Button | GPIO |
|---|---|
| Left | 7 |
| Right | 8 |
| Up | 15 |
| Down | 2 |
| Jump | 1 |
| Game A | 14 |
| Game B | 5 |
| Time | 3 |

GPIO14 and GPIO15 are small surface pads on the ESP32-S3-Zero, not header pins. They need a short wire from the pad to the board.

### Display

| Display pin | Connection |
|---|---|
| GND | GND |
| VCC | 3.3V |
| SCL | GPIO12 |
| SDA | GPIO11 |
| RST | GPIO13 |
| DC | GPIO9 |
| CS | GND |

### Audio (MAX98357A)

| Amp pin | Connection |
|---|---|
| LRC | GPIO4 |
| BCLK | GPIO10 |
| DIN | GPIO6 |
| GAIN | GND |
| GND | GND |
| VIN | 3.3V |

## The overall process

1. Set up a Linux build environment with ESP-IDF.
2. Get the game files: the user supplies MAME ROM and artwork zips, then converts them with LCD-Game-Shrinker.
3. Build the firmware.
4. Flash it to the ESP32-S3-Zero.
5. Test the controls.

Ready made firmware is not provided, because the finished firmware contains the game data. Everyone builds their own.

## Step 1: Build environment

The firmware was built on Ubuntu with ESP-IDF v6.0. Windows users can use an Ubuntu virtual machine (for example VirtualBox) or WSL.

- Install ESP-IDF v6.0 for the ESP32-S3, following Espressif's official ESP-IDF Getting Started guide for Linux. Check the exact version tag and install commands with the user against that guide rather than from memory.
- Clone this repository.
- In **every new terminal**, run `source ~/esp/esp-idf/export.sh` (adjust the path if ESP-IDF was installed elsewhere) before using `idf.py`. If `idf.py` gives "command not found", this is almost always why.

If the user works in a VirtualBox VM with a shared folder:
- Files copied through the shared folder sometimes do not update. Always check sizes and timestamps with `ls -la` (Linux) or `dir` (Windows) after copying.
- Clipboard sharing is under Devices > Shared Clipboard > Bidirectional. In the Linux terminal, paste with Ctrl+Shift+V, not Ctrl+V.
- Linux paths are case sensitive (`ROM` is not the same as `rom`).

## Step 2: Game files

Each game needs two zips with the same MAME short name: a ROM zip (small, tens of KB) and an artwork zip (usually several MB, containing PNG files and `default.lay`). If a "ROM" zip is several MB and contains PNGs, it is actually artwork.

Convert them with [LCD-Game-Shrinker](https://github.com/bzhxx/LCD-Game-Shrinker):
1. Install it and its requirements as described in its README.
2. Put the ROM zip in `input/rom/` and the artwork zip in `input/artwork/`.
3. Run `python3 shrink_it.py input/rom/<shortname>.zip`.
4. The finished file appears in `output/`, for example `Game & Watch Ball.gw`.

Important: if ESP-IDF's `export.sh` has been run in the same terminal, `python3` points to ESP-IDF's Python, which does not have `lxml`. Use `/usr/bin/python3 shrink_it.py ...` or a fresh terminal. The error looks like `No module named 'lxml'`.

A NumPy `DeprecationWarning` during conversion is harmless.

Copy the `.gw` files into `gandw_handheld/main/`. The names must match exactly:

| # | Game | Short name | File name |
|---|---|---|---|
| 0 | Donkey Kong Jr. | `gnw_dkjr` | `Game & Watch Donkey Kong Jr. (New Wide Screen).gw` |
| 1 | Balloon Fight | `gnw_bfight` | `Game & Watch Balloon Fight (Crystal Screen).gw` |
| 2 | Climber | `gnw_climber` | `Game & Watch Climber (Crystal Screen).gw` |
| 3 | Super Mario Bros. | `gnw_smb` | `Game & Watch Super Mario Bros. (Crystal Screen).gw` |
| 4 | Ball | `gnw_ball` | `Game & Watch Ball.gw` |
| 5 | Helmet | `gnw_helmet` | `Game & Watch Helmet (CN-17 version).gw` |
| 6 | Parachute | `gnw_pchute` | `Game & Watch Parachute.gw` |
| 7 | Octopus | `gnw_octopus` | `Game & Watch Octopus.gw` |
| 8 | Popeye | `gnw_popeye` | `Game & Watch Popeye (Wide Screen).gw` |
| 9 | Fire | `gnw_fire` | `Game & Watch Fire (Wide Screen).gw` |
| 10 | Turtle Bridge | `gnw_tbridge` | `Game & Watch Turtle Bridge.gw` |
| 11 | Tropical Fish | `gnw_tfish` | `Game & Watch Tropical Fish.gw` |
| 12 | Mario the Juggler | `gnw_mariotj` | `Game & Watch Mario The Juggler.gw` |

If any game file is missing, the build fails. Check with `ls gandw_handheld/main/*.gw | wc -l`, which should print 13.

## Step 3: Build

```
source ~/esp/esp-idf/export.sh
cd gandw_handheld
rm -rf build
idf.py -DMODEL=single_screen_dpad -DDISPLAY=st7789 build
```

- Both `-D` flags are required. Without `-DDISPLAY=st7789`, the build silently uses the ILI9341 driver and the ST7789V screen will not work.
- Success ends with "Project build complete". The size line should show the app at about 2 MB of a 3 MB (`0x300000`) partition.
- When searching the log for errors, lines like `error.c.obj` are just file names, not errors. Look for `error:`.
- If the build fails after files were renamed or added, clear the build folder with `rm -rf build` and try again.

## Step 4: Flash

The files needed are in `gandw_handheld/build/`: `bootloader/bootloader.bin`, `partition_table/partition-table.bin`, `ota_data_initial.bin` and `gandw_handheld.bin`.

From Linux with the board connected:
```
idf.py -p PORT flash
```

From Windows (easiest if the build was done in a VM): install Python and esptool (`pip install esptool`), find the COM port in Device Manager under Ports (COM & LPT), copy the four files into one folder, open a terminal in that folder, then run:
```
python -m esptool --chip esp32s3 -b 460800 --port COMx --before default-reset --after hard-reset write-flash --flash-mode dio --flash-size 4MB --flash-freq 80m 0x0 bootloader.bin 0x8000 partition-table.bin 0xe000 ota_data_initial.bin 0x10000 gandw_handheld.bin
```

- "No such file or directory" for a `.bin` file means the terminal is in the wrong folder. Use `cd` to the folder first, with quotes around paths that contain spaces.
- If it hangs at "Connecting...", hold BOOT, tap RESET, release BOOT and try again.
- Success ends with "Hash of data verified" for each file.
- After every rebuild, copy all four files again and check their timestamps before flashing.

## Step 5: Controls

### Menu

| Control | Action |
|---|---|
| Left / Right | Choose a game |
| Jump | Load the game |
| Time + Left / Right | Volume |
| Hold Time + Up for 2 seconds | Toggle colour mode for the selected game (beeps, remembered after power off) |

### In a game

| Control | Action |
|---|---|
| Game A / Game B | Start game A or game B |
| Tap Time | Pause / resume |
| Time + Left / Right | Volume |
| Hold Time + Down for 2 seconds | Back to the menu |
| Hold Time + Up for 2 seconds | Continue after a game over (use within about 10 seconds) |

Games 0 to 3 use the D-pad and Jump normally. For Donkey Kong Jr., the key is grabbed with Jump + Left at the edge of the top branch.

Games 4 to 12 use a two-button layout: D-pad Right moves left and Jump moves right.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Screen blank or garbled | Built without `-DDISPLAY=st7789`, or SPI wiring wrong |
| Colours look like a negative | Colour inversion setting in `display.c` (`esp_lcd_panel_invert_color` in `display_setup_lcd_1`) does not suit this panel batch |
| Picture upside down | Mirror settings in `display_setup_lcd_1` in `display.c` |
| Sound distorted | MAX98357A GAIN pin floating. Tie it to GND |
| Left and Right reversed | Swap the GPIO numbers for `BUTTON_LEFT` and `BUTTON_RIGHT` in the `MODEL_SINGLE_SCREEN_DPAD` block of `button.c` |
| A game always starts in Game B, or a button seems stuck | Check for a faulty or shorted switch with a multimeter (GPIO to GND should only beep when pressed) |
| Wrong game or icon for a menu entry | The order of `games[]`, `game_load()` and the icons in `menu_single_screen_dpad.raw` does not match |
| `idf.py: command not found` | ESP-IDF not sourced in this terminal |
| Build error `undefined reference to _binary_...` | A `.gw` file is missing or named differently from the list |

### A useful debugging technique

To see exactly which button signals the firmware sends to a game, temporarily add a log line in `button_get_dpad_buttons()` in `button.c` that prints `hw_buttons` only when it changes, add `#include "esp_log.h"`, then watch the output with `python -m serial.tools.miniterm COMx 115200`. Values: `0x01` Left, `0x02` Up, `0x04` Right, `0x08` Down, `0x10` A (Jump), `0x40` Time (Game B), `0x80` Game (Game A). A value that stays set with nothing pressed points to a hardware fault. Remove the log line afterwards.

## Code notes for changes

- The model's button pins, combos (pause, continue, menu return) and the two-button remap are in `gandw_handheld/main/button.c`.
- Game list, ROM loading and colour tints are in `gandw_handheld/main/game.c`. `game_count` is worked out automatically from `games[]`.
- Colour mode on/off settings are stored in NVS as two 8-bit values: `color_mask` (games 0 to 7) and `color_mask2` (games 8 to 15).
- Continue uses the emulator's `gw_state_save` / `gw_state_load` in `main.c`, called between frames, never from inside the button code.
- Tropical Fish swaps Game A and Game B, tied to its position (index 11) in `button.c`. If games are inserted before it, update that index.
- The two-button remap applies to index 4 and above. Adding a game that needs the normal D-pad means changing that rule.
- Each game costs about 130 to 160 KB of flash (game file plus uncompressed menu icon). About 1 MB is free with 13 games.
- Menu icons are stored in `menu_single_screen_dpad.raw` as raw RGB565, little endian, one icon after another in the same order as `games[]`. Each entry's `img_unit_width` and `img_unit_height` must match its icon size exactly.
