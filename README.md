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
| Probing margin | L:10 F:10 R:50 B:10 mm | Asymmetric — right constrained by -49.3mm X probe offset |
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

Do these steps **in order** after flashing for the first time, or after any firmware rebuild.

### Step 1 — Reset EEPROM

Send via terminal (OctoPrint, Pronterface, or the printer's USB):

```
M502   ; reset all settings to firmware defaults
M500   ; save to EEPROM
```

> Do this before anything else — old EEPROM values from a previous firmware will cause unpredictable behaviour.

---

### Step 2 — PID Autotune (hotend)

The Sprite Pro heater block differs from stock. Run autotune so thermal runaway doesn't trip during prints:

```
M303 E0 S210 C15   ; heat to 210°C, 15 cycles
```

When it finishes it will print `Kp`, `Ki`, `Kd` values. Save them:

```
M301 P<Kp> I<Ki> D<Kd>
M500
```

---

### Step 3 — Bed Tramming (manual levelling)

Before running the mesh, get the bed roughly level by hand:

1. Home the printer: `G28`
2. On the LCD: **Motion → Bed Tramming**
3. Work through each corner — the CR Touch will measure the height and prompt you to turn the knob
4. Repeat until all corners read within ~0.1 mm of each other

---

### Step 4 — Bed Mesh (auto bed levelling)

```
G28    ; home all axes
G29    ; probe 5×5 mesh
M500   ; save mesh to EEPROM
```

---

### Step 5 — Z Offset Calibration

Use the wizard on the LCD: **Motion → Probe Z Offset**, or send:

```
G28
M851 Z0         ; reset offset
G1 Z0 F300      ; move to Z=0
```

Slide a sheet of paper under the nozzle and adjust until you feel slight drag. Note the Z reading and set it:

```
M851 Z-<value>  ; e.g. M851 Z-2.06
M500
```

> Re-run this any time you adjust the probe mount or change nozzles.

---

### Step 6 — E-Step Calibration

The firmware ships with a starting value of 439.45 steps/mm. Verify it:

1. Heat hotend to 200°C
2. Mark the filament 100mm above the extruder entrance
3. Send: `G1 E100 F100`
4. Measure how much filament actually moved
5. Calculate: `new_steps = 439.45 × (100 / actual_mm)`
6. Set and save:

```
M92 E<new_steps>
M500
```

---

### Step 7 — Linear Advance Tuning

Linear Advance K is set to 0.09 as a starting point for the Sprite Pro. Print a [LA calibration pattern](https://marlinfw.org/tools/lin_advance/k-factor.html) and adjust:

```
M900 K<value>   ; e.g. M900 K0.05
M500
```

---

### Step 8 — Input Shaping Tuning

Input shaping is enabled at 40 Hz / zeta 0.15 as a starting point. To tune:

1. Print a ringing tower (search: "Marlin input shaping tower")
2. Identify the layer where ringing stops — that corresponds to a frequency
3. Adjust per axis:

```
M593 X F<hz>    ; e.g. M593 X F37
M593 Y F<hz>
M500
```

---

### Step 9 — First Print

- Use **baby stepping** during the first layer to fine-tune Z height live (double-click the knob)
- Run another `G29` + `M500` after the printer is fully warmed up for the most accurate mesh

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
