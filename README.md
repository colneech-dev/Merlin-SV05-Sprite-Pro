# Marlin 2.1.2.7 — Sovol SV05 + Creality Sprite Pro

Custom Marlin firmware build for the **Sovol SV05** 3D printer fitted with a **Creality Sprite Pro** direct drive extruder and **CR Touch** auto bed levelling probe on a slim/adapter mount.

---

## Hardware

| Component | Details |
|---|---|
| Printer | Sovol SV05 |
| Mainboard | Creality V4.2.2 (STM32F103RET6) |
| Stepper drivers | TMC2208 (standalone) |
| Extruder | Creality Sprite Pro (direct drive) |
| Probe | CR Touch on slim/adapter mount |
| Display | CR-10 Stock Display (rotary knob) |

---

## Key Configuration

| Setting | Value | Notes |
|---|---|---|
| Firmware | Marlin 2.1.2.7 | |
| Board | `BOARD_CREALITY_V4` | V4.2.2 STM32F103RE |
| Thermistor | Type 5 (ATC Semitec 104GT-2) | Sprite Pro 300°C version (thick red wiring) |
| Max hotend temp | 300°C | |
| E-steps | 439.45 | Starting point — calibrate after flashing |
| Probe X offset | -49.3 mm | Slim CR Touch mount |
| Probe Y offset | 0 mm | |
| Probe Z offset | -2.06 mm | Calibrate with M851 after flashing |
| Bed size | 220 × 220 × 300 mm | |
| Homing | X/Y home to MAX (rear-right) | Z homes to MIN |
| Probing margin | 55 mm | Required for -49.3mm X probe offset |
| Bed levelling | Bilinear 5×5 | |
| Linear Advance | K = 0.09 | Tune after e-step calibration |
| Input shaping | 40 Hz / zeta 0.15 | Tune with ringing tower after flashing |

---

## Features Enabled

- Auto Bed Levelling (bilinear 5×5 mesh)
- Probe-assisted bed tramming (CR Touch guides corner adjustment)
- Restore levelling after G28
- Babystepping (always available, 0.05mm per click)
- Linear Advance
- Input Shaping (X & Y)
- S-Curve Acceleration
- Advanced Pause / Filament Change
- Nozzle Park
- Cancel Objects
- Print Counter
- EEPROM settings
- Emergency Parser
- Host Action Commands + Prompt Support
- Thermal runaway protection (hotend + bed)

---

## Preheat Presets

| Preset | Hotend | Bed | Fan |
|---|---|---|---|
| PLA | 200°C | 60°C | 100% |
| PETG | 240°C | 70°C | 100% |
| ABS | 240°C | 100°C | Off |
| TPU | 230°C | 40°C | 100% |

---

## Building

Requires [PlatformIO](https://platformio.org/) (VS Code extension recommended).

```bash
pio run -e STM32F103RE_creality
```

Output: `.pio/build/STM32F103RE_creality/firmware.bin`

Flash by copying `firmware.bin` to the SD card root and powering on.

---

## First-Flash Calibration Sequence

1. `M502` + `M500` — reset EEPROM to firmware defaults
2. `M303 E0 S210 C15` — PID autotune hotend (thermistor changed from stock)
3. `M301 P<p> I<i> D<d>` + `M500` — save PID results
4. Motion > Bed Tramming — level corners using CR Touch feedback
5. `G28` then `G29` — home and run bed mesh
6. `M851` / Probe Z Offset wizard — calibrate Z offset
7. Calibrate e-steps (extrude 100mm, measure, adjust `M92 E<value>`)
8. Print a ringing tower to tune Input Shaping frequencies
   - Adjust with `M593 X F<hz>` and `M593 Y F<hz>` + `M500`

---

## Custom Screen Images

To replace the boot/status screen bitmaps:

1. Create a 128×64 black & white PNG
2. Convert at `marlinfw.org/tools/u8glib/converter.html`
3. Paste output into `Marlin/_Bootscreen.h` or `Marlin/_Statusscreen.h`
4. Rebuild

---

## Reference Configs

The `config_reference/` folder contains the source configuration files used as a base for this build (Ender-5 Pro CrealityV422, Marlin 2.1.2.7 release). Do not place `.h` files directly in a `config/` folder — PlatformIO will detect them as example configurations.
