# ESP32 Game & Watch Handheld (Single Screen D-Pad)

A handheld that plays classic Nintendo Game & Watch games on an ESP32-S3, with a 2.4" colour screen, a D-pad, speaker and rechargeable battery.

This is a fork of [slowlane112/Esp32-Game-and-Watch](https://github.com/slowlane112/Esp32-Game-and-Watch). All credit for the original firmware, emulator integration and hardware designs goes to that project. This fork adds a new hardware model, `single_screen_dpad`, plus a few extra features.

Full build guide on Instructables: *(add link)*

## Need help building it?

Upload `CLAUDE_GUIDE.md` to Claude and tell it where you are up to. It will walk you through each step, from setting up the build tools to flashing the firmware.

## What this fork adds

- A new `single_screen_dpad` model: one screen, a 4-way D-pad, a Jump button, Game A, Game B and Time
- 13 games on one device
- Pause
- Continue after a game over (rewinds roughly 10 to 15 seconds)
- Colour mode: an optional colour tint for each game, set from the menu and remembered after power off
- A two-button layout for the games that originally used only a left and a right button

Only the `single_screen_dpad` model has been built and tested with these changes.

## Games

| # | Game | Year | File name |
|---|---|---|---|
| 1 | Donkey Kong Jr. | 1982 | `Game & Watch Donkey Kong Jr. (New Wide Screen).gw` |
| 2 | Balloon Fight | 1986 | `Game & Watch Balloon Fight (Crystal Screen).gw` |
| 3 | Climber | 1986 | `Game & Watch Climber (Crystal Screen).gw` |
| 4 | Super Mario Bros. | 1986 | `Game & Watch Super Mario Bros. (Crystal Screen).gw` |
| 5 | Ball | 1980 | `Game & Watch Ball.gw` |
| 6 | Helmet | 1981 | `Game & Watch Helmet (CN-17 version).gw` |
| 7 | Parachute | 1981 | `Game & Watch Parachute.gw` |
| 8 | Octopus | 1981 | `Game & Watch Octopus.gw` |
| 9 | Popeye | 1981 | `Game & Watch Popeye (Wide Screen).gw` |
| 10 | Fire | 1981 | `Game & Watch Fire (Wide Screen).gw` |
| 11 | Turtle Bridge | 1982 | `Game & Watch Turtle Bridge.gw` |
| 12 | Tropical Fish | 1985 | `Game & Watch Tropical Fish.gw` |
| 13 | Mario the Juggler | 1991 | `Game & Watch Mario The Juggler.gw` |

Game files are not included in this repository. See [Game files](#game-files) below.

## Hardware

| Part | Notes |
|---|---|
| Waveshare ESP32-S3-Zero | Main controller |
| 2.4" 240x320 SPI TFT, ST7789V | Tested with the GMT024-08-SPI8P board (no backlight pin) |
| MAX98357A I2S amplifier module | |
| Small speaker | 4 to 8 ohm |
| USB-C charger / 5V boost module | 18 x 23.6 mm |
| Single cell 3.7V lithium battery | |
| SPDT slide switch | Power |
| 8 x 3x6x2.5mm SMD tactile switches | Left, Right, Up, Down, Jump, Game A, Game B, Time |
| 2 x 100uF capacitors, 1 x 100nF capacitor | Power filtering |
| Male and female 2.54mm headers | Screen and front panel connections |

The build uses three PCBs: a main board, a front panel that holds the screen and buttons, and a back panel. KiCad files are in this repository.

## Wiring

### Buttons

All buttons connect between the GPIO and GND. Internal pull-ups are used, so no resistors are needed.

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

GPIO14 and GPIO15 are the small surface pads near the edge of the ESP32-S3-Zero, not header pins, so they need a short wire from the pad to the main board.

### Display (SPI)

| Display pin | Connection |
|---|---|
| GND | GND |
| VCC | 3.3V |
| SCL | GPIO12 |
| SDA | GPIO11 |
| RST | GPIO13 |
| DC | GPIO9 |
| CS | GND |

The display is the only device on the SPI bus, so CS is tied to GND.

### Audio (MAX98357A)

| Amp pin | Connection |
|---|---|
| LRC | GPIO4 |
| BCLK | GPIO10 |
| DIN | GPIO6 |
| GAIN | GND |
| GND | GND |
| VIN | 3.3V |

Tying GAIN to GND gave clean sound with the speaker I used. Leaving it floating caused distortion.

## Controls

### Menu

| Control | Action |
|---|---|
| Left / Right | Choose a game |
| Jump | Load the game |
| Time + Left / Right | Volume down / up |
| Hold Time + Up for 2 seconds | Turn colour mode on or off for the selected game |

### In a game

| Control | Action |
|---|---|
| Game A / Game B | Start game A or game B |
| Tap Time | Pause / resume |
| Time + Left / Right | Volume down / up |
| Hold Time + Down for 2 seconds | Back to the menu |
| Hold Time + Up for 2 seconds | Continue after a game over |

Donkey Kong Jr., Balloon Fight, Climber and Super Mario Bros. use the D-pad and Jump as normal.

The other nine games were originally played with a left and a right button. On this handheld, **D-pad Right** moves left and **Jump** moves right, so there is one button under each thumb.

Continue works by keeping a short history of the game in memory. Use it within about 10 seconds of the game ending. Your score goes back to what it was at that point too.

## Game files

The game files (`.gw`) are made from original Game & Watch ROMs and MAME artwork. They are not included here, and you need to supply your own.

1. For each game, you need the MAME ROM zip and the matching MAME artwork zip, both with the same short name (for example `gnw_dkjr.zip`).
2. Convert them with [LCD-Game-Shrinker](https://github.com/bzhxx/LCD-Game-Shrinker). Put the ROM zip in `input/rom/` and the artwork zip in `input/artwork/`, then run:

   ```
   python3 shrink_it.py input/rom/gnw_dkjr.zip
   ```

   If you have ESP-IDF active in the same terminal, use `/usr/bin/python3` instead. The ESP-IDF Python does not have the libraries the shrinker needs.

3. Copy the finished `.gw` files from LCD-Game-Shrinker's `output/` folder into `gandw_handheld/main/`. The names must match the table above exactly.

MAME short names for the games in this build:

| Game | Short name |
|---|---|
| Donkey Kong Jr. | `gnw_dkjr` |
| Balloon Fight | `gnw_bfight` |
| Climber | `gnw_climber` |
| Super Mario Bros. | `gnw_smb` |
| Ball | `gnw_ball` |
| Helmet | `gnw_helmet` |
| Parachute | `gnw_pchute` |
| Octopus | `gnw_octopus` |
| Popeye | `gnw_popeye` |
| Fire (Wide Screen) | `gnw_fire` |
| Turtle Bridge | `gnw_tbridge` |
| Tropical Fish | `gnw_tfish` |
| Mario the Juggler | `gnw_mariotj` |

## Building the firmware

This was built with ESP-IDF v6.0 on Ubuntu.

```
source ~/esp/esp-idf/export.sh
cd gandw_handheld
idf.py -DMODEL=single_screen_dpad -DDISPLAY=st7789 build
```

Both flags are needed. Without `-DDISPLAY=st7789` the build defaults to the ILI9341 driver, and the screen will not work with an ST7789V panel.

## Flashing

With the board connected over USB:

```
idf.py -p PORT flash
```

Or with esptool, from the `gandw_handheld/build` folder:

```
python -m esptool --chip esp32s3 -b 460800 --port PORT --before default-reset --after hard-reset write-flash --flash-mode dio --flash-size 4MB --flash-freq 80m 0x0 bootloader/bootloader.bin 0x8000 partition_table/partition-table.bin 0xe000 ota_data_initial.bin 0x10000 gandw_handheld.bin
```

Replace `PORT` with your serial port, for example `COM4` on Windows or `/dev/ttyACM0` on Linux.

If it gets stuck on "Connecting...", hold BOOT, tap RESET, release BOOT and try again.

## Adding more games

Each game uses roughly 130 to 160 KB of flash (the game file plus its menu icon). With 13 games the firmware is about 2 MB of the 3 MB app partition.

To add a game:

1. Convert it with LCD-Game-Shrinker and copy the `.gw` file into `gandw_handheld/main/`.
2. Add the file name to the `single_screen_dpad` list in `gandw_handheld/main/CMakeLists.txt`.
3. Add an entry to the `games[]` list and a matching ROM entry in `game_load()` in `gandw_handheld/main/game.c`.
4. Add its menu icon to `menu_single_screen_dpad.raw` in the same position as its entry in `games[]`.
5. Add a tint colour for it in `game_tints[]` if you want colour mode.

Tropical Fish swaps Game A and Game B in the button code, and that is tied to its position in the list. If you insert a game before it, update that index in `button.c`.

## Credits

- [slowlane112/Esp32-Game-and-Watch](https://github.com/slowlane112/Esp32-Game-and-Watch): the original project this is based on
- [bzhxx](https://github.com/bzhxx): LCD Game Emulator and LCD-Game-Shrinker
- MAME and the MAME artwork community: the ROM and artwork sets the game files are built from

## License

GPL-3.0, the same as the original project. See [LICENSE](LICENSE).

Game & Watch is a trademark of Nintendo. This project is not affiliated with or endorsed by Nintendo.
