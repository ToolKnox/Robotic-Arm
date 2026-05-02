# Robotic-Arm

Firmware for a 6-axis, mostly-3D-printed desktop robot arm. Runs on an Arduino Mega 2560 + RAMPS 1.4 with six TB6560 stepper drivers, plus a hobby servo for the gripper.

The codebase is a fork of [Marlin](https://github.com/MarlinFirmware/Marlin) by way of the [BCN3D Moveo](https://github.com/BCN3D/BCN3D-Moveo) firmware, reworked for this arm's mechanics.

> **Status:** working firmware, but parts of this README are still TODO — flagged inline. PRs welcome.

## Hardware this firmware is written for

- **MCU:** Arduino Mega 2560
- **Driver shield:** RAMPS 1.4
- **Steppers:** 6× TB6560 external drivers, driving 1× Nema 14, 2× Nema 17, 1× Nema 17 + 5:1 planetary, 2× Nema 23 (3 Nm)
- **Gripper:** 13 kg·cm hobby servo
- **PSU:** 24 V / 320 W, with a 24→12 V buck for fans / logic-side rails

The full mechanical BOM, STLs and assembly guide live with the project page on Printables — see the project description.

## Repository layout

All firmware sources currently sit at the repo root. Notable files:

| File | What it is |
|---|---|
| `Marlin.ino` / `Marlin_BCN3D_Moveo.ino` | Sketch entry point. **TODO:** confirm which one is canonical and remove the other — Arduino IDE will not happily compile a folder containing two `.ino` files with different names. |
| `Configuration.h` | The file you'll edit most: board selection, axis steps/mm, max feedrates, endstop polarity, baud rate. |
| `Configuration_adv.h` | Advanced tunables (acceleration shaping, watchdog, thermal protection — most of which is irrelevant for an arm with no hotend). |
| `pins.h` | Pin map for every Marlin-supported board. The active map is selected by the `MOTHERBOARD` define in `Configuration.h`. |
| `planner.cpp` / `stepper.cpp` / `motion_control.cpp` | Motion core, inherited from Marlin. The 6-axis plumbing for this arm lives here — see source. |
| `ultralcd*.{h,cpp}`, `dogm_*.h`, `cardreader.{h,cpp}` | Optional LCD + SD-card UI, inherited from Marlin. Disable in `Configuration.h` if you aren't using an LCD. |
| `language_*.h` | Locale strings for the LCD, inherited from upstream Marlin. Only `language_en.h` is needed for this build. |
| `qr_solve.{h,cpp}`, `vector_3.{h,cpp}` | Linear-algebra helpers — likely used for inverse-kinematics / coordinate transforms. See source. |

> **TODO:** the STL files, BOM, and CAD source are not currently in this repo. Decide whether to add a `hardware/` folder or to point at the Printables page.

## Building and flashing

### With the Arduino IDE

1. Install the **Arduino IDE** (1.8.19 or newer, or 2.x).
2. Clone this repo. **Rename the cloned folder** so its name matches the `.ino` file you're keeping — the IDE requires sketch folder name == sketch file name. So either:
   - keep `Marlin.ino` and rename the folder to `Marlin/`, or
   - keep `Marlin_BCN3D_Moveo.ino` and rename the folder to `Marlin_BCN3D_Moveo/`.
3. Open the `.ino` in the IDE. All `.h`/`.cpp` files in the folder will load as tabs.
4. **Board:** *Arduino Mega or Mega 2560*. **Processor:** *ATmega2560*.
5. Pick the right serial port, hit **Upload**.

No external libraries are needed — `Servo`, `LiquidCrystalRus`, `SdFat`, and the U8glib display drivers are all vendored in the repo.

### With PlatformIO (recommended for reproducible builds)

> **TODO:** there is no `platformio.ini` in the repo today. Adding one targeting `board = megaatmega2560` and `framework = arduino` would let people build from the command line without touching the IDE. Contributions welcome.

## Configuration — what you'll need to edit

Before the first flash, open `Configuration.h` and confirm:

- `MOTHERBOARD` — must be set to the RAMPS 1.4 variant. **TODO: confirm exact value** (e.g. `BOARD_RAMPS_14_EFB`).
- `BAUDRATE` — pick a value your host tool supports (Marlin default is `250000`; many BCN3D Moveo forks use `115200`). **TODO: confirm.**
- `DEFAULT_AXIS_STEPS_PER_UNIT` — must be calibrated for this arm's gear ratios per joint. **TODO: publish the calibrated values.**
- `INVERT_*_DIR` — flip per joint after the first power-on direction check.
- Endstop enables and `*_HOME_DIR` — depend on whether you've wired physical endstops to RAMPS.
- Thermistor / heater settings — disable; this arm has no hotend or heated bed.

`Configuration_adv.h` rarely needs touching for a first run.

## Pin assignments

The active pin map is selected by `MOTHERBOARD` in `Configuration.h` and resolved in `pins.h`. RAMPS 1.4 only provides 5 stepper sockets natively (X, Y, Z, E0, E1); the 6th axis is wired to spare pins.

> **TODO:** lift the per-axis STEP / DIR / EN / endstop pins for *this* build into a table here, plus whichever RAMPS header the 6th driver is hooked to and the gripper-servo pin. Until then, see `pins.h` (the section guarded by your `MOTHERBOARD` define) and the wiring section of the assembly guide.

## Driving the arm

> **TODO:** confirm the control method.
>
> - If it's host-side **G-code over USB**, document baud and the joint-to-axis-letter mapping (e.g. `J1→X`, `J2→Y`, …, `J6→E1`), and give an example: `G1 X10 F500`.
> - If it's the **LCD + SD card** path, document which display the firmware is built for (the manifest contains both a Hitachi HD44780 implementation and a DOGM/u8glib one) and how to load a `.gcode` file.
> - If there's a host-side GUI, link it.

## Troubleshooting

- **"Compile error: redefinition of `loop`"** or the IDE refusing to open the sketch — caused by having two `.ino` files in the folder. Delete one (see Building above).
- **All axes home in the wrong direction** — flip the relevant `INVERT_*_DIR` in `Configuration.h`. Don't move them by hand under power.
- **One joint never moves** — check the TB6560 enable logic (active-low on RAMPS) and the per-driver microstepping DIP switches before suspecting firmware.
- **MCU resets under load** — almost always a power problem (24 V rail sagging, or USB sharing logic ground with the motor return). The watchdog (`watchdog.cpp`) will reboot cleanly; the symptom is real.

## Credits

- [Marlin](https://github.com/MarlinFirmware/Marlin) — the firmware core (planner, stepper, serial, LCD/SD).
- [BCN3D Moveo](https://github.com/BCN3D/BCN3D-Moveo) — the original arm-shaped Marlin fork this build is derived from.

## License

The firmware in this repo is derived from Marlin and is therefore distributed under **GPL-2.0-or-later**. The upstream `LICENSE` file should be committed alongside this README — **TODO**.

Mechanical / CAD assets (STLs, BOM, drawings) on the project page are released under **TODO: confirm CC-BY / CC-BY-NC / CC-BY-SA**.