# Claude Code: EspControl Port to Guition JC8048W550C

## What we're doing

Porting `jtenniswood/espcontrol` (ESPHome-based Home Assistant touch panel firmware)
to the **Guition JC8048W550C**, a 5" 800×480 ESP32-S3 capacitive touch display
that the upstream project does not yet support.

The repo is a fork of `jtenniswood/espcontrol`. Work happens on a `port/jc8048w550c`
branch. The upstream `4848S040` device is the closest reference — same SoC family
(ESP32-S3), same touch controller (GT911), but a different display driver and
different resolution.

## Hardware facts (verified before this port started)

- **SoC**: ESP32-S3-WROOM-1 N16R8 (16MB flash, 8MB **octal** PSRAM)
- **Display**: 5" IPS, 800×480, parallel RGB (no SPI command bus on the panel)
- **Touch**: GT911 capacitive at I²C address `0x5D`
- **Backlight**: GPIO2, PWM @ 1220Hz (per agillis reference)
- **I²C**: SDA=GPIO19, SCL=GPIO20
- **RGB control pins**: DE=GPIO40, HSYNC=GPIO39, VSYNC=GPIO41, PCLK=GPIO42
- **RGB data pins** (verified working from agillis/esphome-modular-lvgl-buttons):
  - Red:   GPIO45, GPIO48, GPIO47, GPIO21, GPIO14
  - Green: GPIO5,  GPIO6,  GPIO7,  GPIO15, GPIO16, GPIO4
  - Blue:  GPIO8,  GPIO3,  GPIO46, GPIO9,  GPIO1
- **`invert_colors: true`** is required (verified across multiple working configs)
- **No on-board relays** on this variant (the 4848S040C has 3; the W550C has none).

## Critical porting decisions already made

1. **Display platform: `rpi_dpi_rgb`, NOT `mipi_rgb`.** The 4848S040 reference uses
   `mipi_rgb` because that panel has an SPI command bus for the ST7701S init
   sequence. The JC8048W550C panel is pure parallel RGB with no init commands —
   `rpi_dpi_rgb` is the correct platform. The SPI block from `device.yaml` is
   removed.
2. **GPIO45/47/48 are RGB data lines on this board.** The 4848S040 uses GPIO45/47/48
   for SPI to the panel; here those are pixel data. The I²C SCL was GPIO45 on the
   4848S040 — moved to GPIO20 here.
3. **Grid: 4×3 (12 slots) for 800×480 landscape.** A 3×3 grid like the 4848S040
   would waste a lot of space; 4×3 cells are roughly 190×140px which matches the
   font sizing of the existing `_sm` fonts.
4. **Reuse `_sm` fonts** from the 4848S040 reference. Cell pixel area is similar
   enough that a font rebuild isn't needed for the first boot.

## What I created (read these first)

```
devices/guition-esp32-s3-jc8048w550c/
├── dev.yaml                    # local build entry point (compile + flash from here)
├── esphome.yaml                # release/end-user entry (pulls remote package)
├── packages.yaml               # include manifest, substitutions, 12-slot generated block
└── device/
    ├── device.yaml             # hardware config — RGB display, touch, backlight, PSRAM
    ├── lvgl.yaml               # 4×3 main page grid + top-layer clock/temp HUD
    ├── fonts.yaml              # _sm font family for this panel
    └── sensors.yaml            # COPIED FROM 4848S040 — needs regeneration (see below)

builds/
├── guition-esp32-s3-jc8048w550c.yaml         # CI build profile
└── guition-esp32-s3-jc8048w550c.factory.yaml # web-installer factory build
```

`sensors.yaml` is currently a copy of the 4848S040 version with 9 slots. It MUST
be regenerated for 12 slots before first compile.

`device_slots.snippet.json` in the repo root is the entry to add to
`scripts/device_slots.json`. After adding it, run the generator.

## Tasks, in order

### 1. Wire the new device into the build system

```bash
# Add the device entry to scripts/device_slots.json
# (merge the object from device_slots.snippet.json into the "devices" array)

# Regenerate sensors.yaml + the BEGIN/END GENERATED BUTTON PACKAGES blocks:
python scripts/generate_device_slots.py
```

Verify after running:
- `devices/guition-esp32-s3-jc8048w550c/device/sensors.yaml` now has 12 slots
- `devices/guition-esp32-s3-jc8048w550c/packages.yaml` has 12 `btn_N` entries

### 2. First compile (no flash yet)

Use the same Docker pattern Scott uses in his other ESPHome projects (and that
the espframe project documents):

```bash
docker run --rm -v "${PWD}:/config" \
  ghcr.io/esphome/esphome:2026.4.0 \
  compile /config/builds/guition-esp32-s3-jc8048w550c.factory.yaml
```

If it doesn't compile cleanly:
- **Read the full error**, don't skim. ESPHome error messages are usually precise.
- If the error references a substitution or include path, it's almost certainly
  in `packages.yaml` or the relative paths in `lvgl.yaml`.
- If the error is in the C++ phase (post-codegen), it's likely a slot-count mismatch
  in `sensors.yaml` — go back to step 1.
- Pin/GPIO conflicts surface at codegen with a clear "GPIO X is already used"
  message. Cross-reference the agillis pin map.

Iterate compile-only until clean. Do NOT flash a board until compile is clean.

### 3. First flash over USB

```bash
# Connect the JC8048W550C via USB-C (data-capable cable, not charge-only)
docker run --rm -v "${PWD}:/config" --device=/dev/ttyACM0 \
  ghcr.io/esphome/esphome:2026.4.0 \
  run /config/devices/guition-esp32-s3-jc8048w550c/dev.yaml
```

Adjust `--device=` to the actual port (`/dev/ttyACM0` or `/dev/ttyUSB0` typically).

If the flash hangs or fails, hold BOOT, tap RESET, release BOOT, retry. The board
ships with factory demo firmware that sometimes blocks the first flash.

### 4. Diagnose what we see on the panel

Boot outcomes and what they mean:

| Symptom | Likely cause | Fix |
|---|---|---|
| Blank black screen, backlight on | RGB timing wrong, or panel not initialized | Verify `pclk_frequency: 16MHz`. Try 14MHz or 12MHz to rule out signal integrity. |
| Backlight off | LEDC not driving GPIO2 | Confirm GPIO2 is the BL pin on this revision; some have GPIO38 |
| Garbled / striped / wrong-color image | data-pin order wrong, or `invert_colors` wrong | Check pin map against agillis exactly. Try toggling `invert_colors`. |
| Image OK but inverted colors | `invert_colors: true` should be `false`, or vice versa | Toggle. |
| Image OK but mirrored / upside down | rotation or transform issue | Add `transform: { mirror_x: true }` or `mirror_y: true` to `touchscreen:` and/or set `rotation:` on `lvgl:` |
| Image OK, touch dead | I²C scan didn't find GT911 | Check `i2c.scan: true` log output; try address `0x14` if `0x5D` fails |
| Image OK, touch coords wrong/swapped | rotation transform mismatch | Adjust `transform:` on the `touchscreen` block to match `rotation:` on `lvgl` |
| Bootloop with `Brownout detector triggered` | USB power insufficient | Use a 5V/2A wall PSU, not laptop USB |
| Bootloop with `assert failed: heap_caps_alloc` or PSRAM errors | Octal PSRAM init failed | Confirm `psram.mode: octal`; the N16R8 module is octal — quad will brick boot |

Once the panel boots and shows the EspControl loading screen, OTA works and we
no longer need USB:

```bash
# OTA from now on
esphome run devices/guition-esp32-s3-jc8048w550c/dev.yaml --device <device-ip>
```

### 5. Connect to Home Assistant and verify the EspControl UI

- Connect the panel's setup hotspot, give it WiFi creds
- HA should auto-discover via ESPHome integration
- Open `http://<panel-ip>` and add a few buttons
- Verify: button taps reach HA, sensor cards update, screensaver works,
  backlight schedule works, OTA works

### 6. Layout polish (only after the panel is functional)

- If labels/icons look off, tune `fonts.yaml` sizes
- If the 4×3 grid feels cramped or sparse, consider 5×3 (15 slots) or 4×2 (8 slots)
  — would require updating `device_slots.json` and re-running the generator
- The `clock_cx`/`clock_cy` substitutions in `packages.yaml` center the
  full-screen clock; tweak if it looks off-center
- Consider a 5-column variant matching the JC1060P470 (which is also 1024×600
  landscape) if 4 columns feels too sparse

## What NOT to do

- **Do not edit `sensors.yaml` directly.** It's generated. Fix `device_slots.json`
  and re-run the generator.
- **Do not change pin assignments without checking the agillis reference first.**
  The pin map was verified by multiple independent sources; speculative changes
  are how you brick a working setup.
- **Do not flash a build that hasn't compiled cleanly.** ESPHome compile errors
  are almost always real and won't fix themselves on-device.
- **Do not commit `secrets.yaml` or any file containing WiFi passwords.** Use
  `secrets.yaml.example` as a template; the real one is gitignored.
- **Do not modify the `common/` directory** unless absolutely necessary — those
  files are shared with all other devices and changes there can break upstream.
  All board-specific work lives under `devices/guition-esp32-s3-jc8048w550c/`.

## When you're stuck

Reference configs to consult, in order of relevance:

1. `devices/guition-esp32-s3-4848s040/` — same SoC family, closest analogue
2. `https://github.com/agillis/esphome-modular-lvgl-buttons/blob/main/hardware/guition-esp32-jc8048w550.yaml`
   — gold-standard pin map for this exact panel
3. `https://www.openhasp.com/0.7.0/hardware/guition/jc8048w550/` — alternate
   working config, useful when something disagrees with agillis
4. `https://github.com/rzeldent/esp32-smartdisplay/discussions/185` — community
   notes specifically on the JC8048W550C variant

If a problem isn't in the table above and isn't obvious from logs, paste the
**full** ESPHome log to the user (Scott) before guessing — he can usually spot
the issue from his hardware experience.

## Done definition

- Compiles cleanly via `builds/guition-esp32-s3-jc8048w550c.factory.yaml`
- Boots to the EspControl loading screen
- HA discovers the device via ESPHome integration
- At least 4 buttons configured and working from the panel's web UI
- OTA update works
- README in the device folder documents the port for future maintainers
- (Stretch) PR opened against jtenniswood/espcontrol upstream
