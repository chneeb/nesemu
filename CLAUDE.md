# CLAUDE.md — ESP32 NES Emulator (nesemu)

## Project Overview
Arduino IDE project that emulates a NES on an ESP32 with an ILI9341 320x240 TFT display.
Based on the nofrendo NES emulator, ported to ESP32 and wrapped in the Arduino framework.

## Build
- **IDE**: Arduino IDE (tested with v1.8.13, ESP32 lib v1.04)
- **Entry point**: `nesemu.ino`
- **Config**: `config.h` — all pin mappings, feature flags, and controller selection live here

## Key Architecture
- Emulation runs on core 1; video SPI transfer runs as a separate task on core 0 (`RUN_VIDEO_AS_TASK`)
- Frame pacing via FreeRTOS software timer at 60 Hz (`nofrendo_ticks`)
- FreeRTOS scheduler runs at 300 Hz (`CONFIG_FREERTOS_HZ 300`)
- LCD uses VSPI bus at 80 MHz (required for performance)
- ROM files stored on SPIFFS or SD card (FAT, 8.3 filenames), listed in `roms.txt`

## Key Feature Flags (config.h)

| Define | Default | Description |
|--------|---------|-------------|
| `BILINEAR` | defined | Bilinear interpolation for screen scaling (vs nearest-neighbor) |
| `FULL_SCREEN` | `1` | X-axis stretch to fill 320px width (0=off, 1=on at startup) |
| `NES_MAX_SPRITES` | `64` | Max sprites per scanline (default NES is 8; 64 removes flicker) |
| `ROTATE_LCD_180` | defined | Rotate display 180 degrees |
| `RUN_VIDEO_AS_TASK` | `0` | Run SPI transfer as separate task on given core (0 or 1) |
| `USE_SPI_DMA` | undefined | Use DMA for SPI (vs CPU; about equal speed, CPU has lower latency) |
| `USE_OS_SEMAPHORES` | undefined | Use FreeRTOS semaphores between tasks (seems slower) |
| `CONFIG_SOUND_ENABLED` | undefined | Enable I2S audio on GPIO 25/26 |
| `BOOT_ROM` | `0` | Index of ROM to boot directly (skips menu selection) |
| `SKIP_MENU` | undefined | Skip menu entirely, boot ROM at hardcoded path |
| `CHEAP_YELLOW_DISPLAY_CONF` | undefined | Alternate pin config for "Cheap Yellow Display" ESP32 board |
| `PAL` | undefined | PAL timing; NTSC used if not defined |

## Controllers
Only one controller type can be active at a time — set via `#define` in `config.h`.

| Define | Type | Notes |
|--------|------|-------|
| `CONFIG_HW_CONTROLLER_NES` | NES controller | Active-low, shift-register |
| `CONFIG_HW_CONTROLLER_SNES` | SNES controller | Active-low, shift-register |
| `CONFIG_HW_CONTROLLER_PSX` | PSX/PS2 controller | SPI protocol |
| `CONFIG_HW_CONTROLLER_GPIO` | Raw GPIO buttons | Direct pin reads |
| `CONFIG_HW_CONTROLLER_NESMINI` | NES Classic Mini (I2C) | Active-high; see wii_i2c.{h,c} |

### NESMINI I2C Notes (important)
- Uses Wii extension controller protocol over I2C at address `0x52`
- Full 3-step init sequence is required: `{0xf0,0x55}`, `{0xfb,0x00}`, `{0xfe,0x03}` — the `{0xfe,0x03}` step is critical for clone hardware
- Arduino's Wire library pre-initializes I2C port 0; must call `i2c_driver_delete()` + `periph_module_reset(PERIPH_I2C0_MODULE)` before `i2c_driver_install()` to avoid timeout errors (done in `wii_i2c_setup_i2c()`)
- `NESMINI_poll()` in `Controller.c` manually decodes raw 8-byte I2C data (does **not** use `wii_i2c_decode_nesmini()`); returns 1 in a bit position when that button is pressed
- `ReadControllerInput()` for NESMINI uses `bit` directly (no `!`) because the decode already inverts the active-low hardware signal
- Default pins: SDA=GPIO21, SCL=GPIO22
- `WII_I2C_ENABLE_MULTI_CORE` in `wii_i2c.h` enables a background FreeRTOS task for I2C reads (currently disabled, set to 0)

## Button/Input Convention
`ReadControllerInput()` returns a 16-bit value starting at `0xFFFF` (PSX active-low convention).
Pressed buttons are represented by their bit being **cleared** (subtracted from 0xFFFF).
- PSX and NES/SNES controllers: raw values are active-low → use `!bit` to detect press
- NESMINI: poll returns active-high → use `bit` directly (no `!`) to detect press

### Button Combos
- **Menu toggle**: SELECT+LEFT (NES/NESMINI), L button (SNES), L1 (PSX)
- **Power/sleep**: SELECT+START (NES/NESMINI), R button (SNES), R1 (PSX)

### In-Game Overlay Menu (when showMenu is true)
Toggled by the MENU_BUTTON combo. Controls while menu is open:
- Up/Down: volume (0–4)
- Left/Right: brightness (0–4)
- A: toggle progressive/interlaced update mode
- B: toggle X-axis stretch
- TurboA/TurboB: cycle turbo speed (0–5)

## Sound
- I2S audio on GPIO 25/26. Enable with `#define CONFIG_SOUND_ENABLED` in `config.h`
- Do not assign anything else to pins 25/26 when sound is enabled

## Important Files
| File | Purpose |
|------|---------|
| `config.h` | All hardware config, pin maps, feature flags |
| `nesemu.ino` | Arduino setup/loop, filesystem mount |
| `src/Controller.c` | Controller init, polling, `ReadControllerInput()`, in-game overlay menu logic |
| `src/wii_i2c.{c,h}` | Wii I2C extension protocol driver (nunchuk, classic, NES Mini) |
| `src/pretty_effect.c` | Menu renderer: splash screen, ROM list display, user input handling for menu |
| `src/video_audio.c` | LCD blit and audio output |
| `src/menu.c` | ROM list loading (`roms.txt`), `runMenu()` entry point |
| `src/spi_lcd.c` | Low-level SPI/LCD driver, backlight PWM |
| `src/nes.c` | Main emulation loop, `osd_getinput()` |
