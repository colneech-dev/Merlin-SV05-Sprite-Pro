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
| Hotend PID | Kp 17.62 / Ki 1.42 / Kd 54.63 | Tuned via M303 E0 S210 C15 on Sprite Pro |
| Bed heater | Bang-bang + BED_LIMIT_SWITCHING | Appropriate for large 235×235 bed |
| E-steps | 439.45 | Starting point — calibrate after flashing |
| Probe X offset | -37.5 mm | Measured by centering probe over bed |
| Probe Y offset | +6.2 mm | Measured by centering probe over bed |
| Probe Z offset | -4.07 mm | Calibrated with live baby-step tuning |
| Bed size | 220 × 220 × 300 mm | |
| Homing | X/Y home to MAX (rear-right) | Z homes to MIN |
| Probing margin | L:10 F:10 R:40 B:10 mm | Asymmetric — right constrained by -37.5mm X probe offset |
| Probe low point | -5 mm | Extra travel margin before probing fails |
| Bed levelling | Bilinear 5×5, double-probing | Each point probed twice and averaged |
| Linear Advance | K = 0.09 | Tune after e-step calibration |
| Input shaping | 40 Hz / zeta 0.15 | Tune with ringing tower after flashing |

---

## Features Enabled

- Auto Bed Levelling (bilinear 5×5 mesh, double-probing for accuracy)
- Probe-assisted bed tramming (CR Touch guides corner adjustment)
- Levelling always enabled after G28
- Custom menu: "Level Bed & Save" (homes, probes 5×5 mesh, saves to EEPROM in one step)
- Babystepping (always available, 0.05mm per click)
- Linear Advance
- Input Shaping (X & Y)
- S-Curve Acceleration
- Adaptive Step Smoothing
- Advanced Pause / Filament Change
- Filament Runout Sensor (Sprite Pro built-in, pin PA4)
- Nozzle Park
- Cancel Objects
- Print Counter
- EEPROM settings (auto-init on error)
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

Do these steps **in order** after flashing for the first time, or after any firmware rebuild. Each step builds on the previous one — don't skip ahead.

You will need a USB terminal to send G-code commands. Options:
- **OctoPrint** — Terminal tab
- **Pronterface** — free, cross-platform
- **VS Code** — Serial Monitor extension
- **Arduino IDE** — Serial Monitor (set baud to 115200)

---

### Step 1 — Flash the Firmware

1. Copy `firmware.bin` to the **root** of a FAT32-formatted SD card (≤32 GB)
2. Rename it to `firmware.bin` — do not put it in a folder
3. Power the printer **off**, insert the SD card, power **on**
4. The screen will go blank for ~30 seconds while flashing
5. When the boot screen appears, flashing is complete
6. Remove the SD card — if you leave it in, some versions will re-flash on every boot

> If the printer doesn't flash, try reformatting the SD card as FAT32 with 4096-byte allocation unit size.

---

### Step 2 — Reset EEPROM

**This must be the first thing you do.** Old EEPROM data from a previous firmware version will override the new defaults and cause erratic behaviour — wrong steps/mm, bad PID values, incorrect probe offsets.

Connect via USB terminal and send:

```gcode
M502   ; restore all settings to firmware defaults
M500   ; write defaults to EEPROM
M501   ; reload from EEPROM (confirms the save worked)
```

You should see the terminal output a list of settings. If you see `echo:SD init fail` that's fine — it just means the SD card was removed.

---

### Step 3 — PID Autotune (Hotend)

The Sprite Pro has a different heater cartridge and heater block to the stock SV05 hotend. The default PID values will not match and can cause thermal runaway protection to trip mid-print.

Heat the hotend and run 15 cycles of autotune at your most common print temperature:

```gcode
M303 E0 S210 C15   ; autotune at 210°C for 15 cycles (~5 minutes)
```

Watch the terminal — when it finishes you will see a line like:

```
Kp: 28.72  Ki: 2.54  Kd: 81.20
```

Copy those values and save them:

```gcode
M301 P28.72 I2.54 D81.20
M500
```

> If you print at significantly different temperatures (e.g. ABS at 250°C), run autotune again at that temperature too. Use whichever PID values give you the most stable temperature during actual printing.

**Bed PID** (optional but recommended):

```gcode
M303 E-1 S60 C8   ; autotune bed at 60°C for 8 cycles
```

Save with:

```gcode
M304 P<Kp> I<Ki> D<Kd>
M500
```

---

### Step 4 — Bed Tramming (Physical Levelling)

The mesh levelling (Step 5) compensates for small variations, but it works best when the bed is already physically level. Tramming gets the four corners as close as possible before running the mesh.

**This firmware uses probe-assisted tramming** — the CR Touch measures each corner and tells you which way to turn the knob, rather than relying on paper.

1. Home the printer:
   ```gcode
   G28
   ```
2. On the LCD navigate to: **Motion → Bed Tramming**
3. The nozzle will move to the first corner (front-left) and the CR Touch will probe it
4. The display will show how far out of level that corner is
5. Turn the bed adjustment knob until the display reads close to zero, then press the knob to move to the next corner
6. Work through all four corners: front-left → front-right → back-right → back-left
7. **Repeat the full cycle 2–3 times** — adjusting one corner slightly affects the others

**Target:** all four corners within ±0.1 mm of each other.

> If a corner needs more than 1–2 full turns of the knob, check that nothing is caught under the bed (wiring, clips) before continuing.

---

### Step 5 — Bed Mesh (Auto Bed Levelling)

Once the bed is physically level, run a 5×5 probe mesh to map any remaining warp or high/low spots. Marlin will use this mesh during printing to compensate automatically.

```gcode
G28    ; home all axes first (required before G29)
G29    ; probe all 25 points — takes ~3 minutes
M500   ; save the mesh to EEPROM
```

The terminal will print the mesh as a grid of numbers when it finishes. Ideally all values are within ±0.2 mm. If you see values larger than ±0.5 mm, re-check the physical tramming (Step 4) and run G29 again.

> **Re-run G29 any time you:**
> - Remove or reinstall the bed
> - Change the Z offset significantly
> - Notice first-layer inconsistency across the bed
> - Let the printer sit unused for a long time (beds can warp with temperature cycles)

---

### Step 6 — Z Offset Calibration

The Z offset tells the printer exactly how far below the probe trigger point the nozzle tip sits. This controls how close the nozzle is to the bed on the first layer. It is the **most important calibration** for print quality.

**Using the LCD wizard (recommended):**

Navigate to **Motion → Probe Z Offset**. The printer will home, then move the nozzle to the centre of the bed at Z=0. Use the knob to move the nozzle up or down until a sheet of paper slides under with slight resistance (you can feel drag but the paper isn't clamped). Confirm and save.

**Using G-code manually:**

```gcode
G28              ; home
G1 Z0 F300       ; move to theoretical Z=0
```

Slide paper under the nozzle. If there is a gap, the offset needs to be more negative. If the nozzle is dragging, it needs to be less negative. Adjust in 0.05 mm steps:

```gcode
M851 Z-2.10      ; example — adjust until paper drag feels right
M500
G28              ; re-home to apply new offset
G1 Z0 F300       ; check again
```

> **Re-calibrate Z offset any time you:**
> - Change the nozzle
> - Remove and reinstall the hotend
> - Adjust or remove the CR Touch mount
> - Notice that first layers are consistently too high or too low

**Fine-tuning during a print:** Double-click the knob while printing to access live baby-stepping. This adjusts Z in 0.05 mm increments without stopping the print. Once happy, run `M851 Z<new value>` + `M500` to make it permanent.

---

### Step 7 — E-Step Calibration

E-steps control how much filament the extruder pushes per mm of commanded movement. The firmware ships with 439.45 steps/mm as a starting point for the Sprite Pro — your actual value may differ slightly.

**How to calibrate:**

1. Heat the hotend to 200°C (filament must be loaded and hot enough to move freely)
2. Cut the filament flush with the top of the extruder, or use a Sharpie to mark exactly 120 mm above the extruder inlet
3. Send this command to extrude 100 mm slowly:
   ```gcode
   G1 E100 F100
   ```
4. Measure from the extruder inlet to the mark (or measure how much filament came out if cutting flush)
5. Calculate your actual extruded length: `actual = 120 - remaining_distance`
6. Calculate the corrected steps:
   ```
   new_steps = 439.45 × (100 ÷ actual_mm)
   ```
   Example: if only 96 mm extruded → `439.45 × (100 ÷ 96) = 457.76`
7. Set and save:
   ```gcode
   M92 E457.76
   M500
   ```
8. Repeat the test to confirm — you should now get very close to 100 mm

> E-steps are a **mechanical** calibration. Do not use them to fix under- or over-extrusion caused by temperature, print speed, or flow rate — those need to be tuned in your slicer.

---

### Step 8 — Linear Advance Tuning

Linear Advance compensates for pressure in the melt zone — it pushes a little extra filament before corners and retracts slightly on straight runs. This eliminates bulging corners and gaps at direction changes.

K = 0.09 is a starting point. The ideal value depends on your filament, temperature, and print speed.

**To find your K value:**

1. Go to the [Marlin K-factor calibration tool](https://marlinfw.org/tools/lin_advance/k-factor.html)
2. Set your printer dimensions, speeds, and a K range of 0.00–0.20
3. Print the generated pattern — it prints a series of lines with increasing K values
4. Look for the K value where corner bulging disappears without gaps appearing
5. Set and save:
   ```gcode
   M900 K0.07     ; example value — use what your print shows
   M500
   ```

> Tune Linear Advance **after** e-steps are correct and **after** you have a good Z offset. Errors in those will mask the LA results.
> Re-tune when changing filament type, brand, or significantly changing print temperature or speed.

---

### Step 9 — Input Shaping Tuning

Input shaping reduces ringing (ghosting/echoing) in prints caused by vibration at the printer's resonant frequency. It is enabled at 40 Hz as a starting point — your SV05 may resonate at a slightly different frequency.

**To find your resonant frequency:**

1. Download or slice a "ringing tower" or "input shaping calibration tower" for Marlin (search Printables or Thingiverse)
2. Print it without input shaping temporarily disabled: `M593 X F0` / `M593 Y F0`
3. Look at the tower — ringing appears as ripples on the surface after direction changes
4. Count the ripples and measure their spacing to estimate frequency, or use the tower model's built-in frequency markings
5. Re-enable shaping with the tuned frequency:
   ```gcode
   M593 X F37.5   ; example — tune X axis
   M593 Y F38.0   ; example — tune Y axis (may differ from X)
   M500
   ```

Alternatively, print the tower with shaping enabled and look for the layer where ringing disappears — this indicates the firmware's correction is kicking in at the right frequency.

> The default 40 Hz / zeta 0.15 works reasonably well as-is. Tune this last, after print quality is otherwise good.

---

### Step 10 — First Print Checklist

Before starting your first real print:

- [ ] EEPROM reset done (`M502` + `M500`)
- [ ] PID autotune done and saved
- [ ] Bed trammed (all corners within 0.1 mm)
- [ ] Bed mesh run (`G29` + `M500`)
- [ ] Z offset calibrated
- [ ] E-steps verified
- [ ] Filament loaded and primed
- [ ] Slicer start G-code includes `G28` and `G29` (or `M420 S1` to restore saved mesh)

**Recommended slicer start G-code:**

```gcode
M140 S[bed_temp]       ; start bed heating
M104 S150              ; preheat hotend (no ooze)
G28                    ; home all axes
M420 S1                ; restore saved bed mesh from EEPROM
M109 S[hotend_temp]    ; wait for hotend temp
M190 S[bed_temp]       ; wait for bed temp
G1 Z5 F3000            ; lift nozzle
G1 X-1 Y0 F3000        ; move to left edge (outside print area)
G1 Z0.3 F300           ; drop to purge height
G92 E0                 ; reset extruder
G1 Y220 E20 F500       ; purge line — front to back (full bed length)
G1 Y0 E40 F500         ; purge line — back to front
G1 Y55 E45 F500        ; purge line — 1/4 bed
G92 E0                 ; reset extruder
G1 Z2 F3000            ; lift before print
```

**During the first layer:**

Double-click the encoder knob to open baby-stepping. Adjust Z until the filament squishes into the bed slightly with no gap and no curling/dragging. A good first layer looks like smooth, slightly flattened lines with no visible gap between them.

Once the first layer looks good, save the Z offset:

```gcode
M851 Z<current_value>
M500
```

> Run `G29` + `M500` again after the printer has been printing for 20+ minutes and is fully at temperature — the mesh will be more accurate than one taken cold.

---

## Slicer Profiles

Ready-made profiles are included for three slicers:

### PrusaSlicer — [`prusaslicer/`](prusaslicer/)

Import via **File → Import → Import Config**. Import in this order:
1. `Sovol SV05 Sprite Pro.ini` — printer
2. `filament_PLA.ini`, `filament_PETG.ini`, `filament_TPU.ini` — filaments
3. Print profiles — select what you need

| Print profile | Use |
|---|---|
| SV05 PLA 0.1mm | Fine detail |
| SV05 PLA 0.2mm | Everyday |
| SV05 PLA 0.3mm | Fast/functional |
| SV05 PETG 0.1mm | Fine detail |
| SV05 PETG 0.2mm | Everyday |
| SV05 PETG 0.3mm | Fast/functional |
| SV05 TPU 0.2mm | Flexible |

### OrcaSlicer — [`orcaslicer/`](orcaslicer/)

Import via **File → Import Configs** and point it at the `orcaslicer/` folder. OrcaSlicer will pick up machine, process, and filament profiles automatically.

Same print profiles as PrusaSlicer. Includes `pressure_advance: 0.09` matching the firmware Linear Advance K value.

### Cura — [`cura/`](cura/)

Copy `Sovol SV05 Sprite Pro.def.json` to your Cura `definitions/` folder:
- Windows: `%AppData%\cura\<version>\definitions\`
- Then restart Cura and add the printer via **Add Printer → Custom FFF**

All profiles include:
- Start G-code: home → restore bed mesh (`M420 S1`) → heat → purge line → wipe
- End G-code: retract → lift → park at front → heaters/fan/motors off
- Temperatures pulled from the selected filament profile automatically

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
