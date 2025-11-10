# PickOMatic Motion Controller

Custom motion-control PCB for the PickOMatic pick-and-place toolchain. The board centers on an ATmega2560 (16 MHz) and exposes everything needed to drive the mechanics of a multi-head component placer: coordinated stepper motion, pneumatic handling, lighting, endstop monitoring, and USB connectivity for streaming G-code.

## Repository Layout

- `hardware/` – KiCad 8 project: schematic, PCB, fabrication outputs, and backups.
- `schematic.png` / `pcb.png` / `3D.png` – quick previews of the current design.

Open the KiCad project (`hardware/hardware.kicad_pro`) to inspect or modify the schematic/PCB.

## Hardware Highlights

| Function | Implementation |
| --- | --- |
| MCU | ATmega2560 @ 16 MHz with a full 5 V I/O bank exposed for motion, sensors, and auxiliary loads. |
| Programming | Standard 6-pin AVR-ISP header for bootloader bring-up, plus an onboard CH340C USB-UART for firmware uploads and runtime G-code/telemetry. |
| Axis control | Five primary stepper channels (2×X, 2×Y, 1×Z) plus two auxiliary stepper channels dedicated to the dual picker heads. |
| Vertical compliance | Three hobby-servo outputs cooperate with the Z axis so each picker head can adjust nozzle height independently (fine pitch vs bulk parts). |
| Pneumatics & lighting | High-side driver for the vacuum/air pump and relay output for the inspection light that assists the vision camera. |
| Endstops | Five limit-switch inputs routed to PCINT lines so firmware can react immediately to any axis stop without polling latency. |

## Connectors & Loads

- **Power** – The digital domain remains at 5 V; provide the motor/lighting voltages required by your external drivers (consult the schematic to confirm connector pinouts and allowable current).
- **Steppers** – Eight pairs of STEP/DIR outputs let you interface with typical plug-in drivers (A4988, DRV8825, TMC series). Wire the two X and two Y channels to the corresponding gantry motors so firmware can keep them phase-aligned.
- **Servos** – Three 3-pin headers (signal/5 V/GND) drive standard RC servos for nozzle tilt or compliance adjustments.
- **Vacuum & light** – The pump driver is sized for continuous duty vacuum pumps; the relay contact can switch the illumination supply for the fiducial/alignment camera.
- **Endstops** – Use normally-closed switches whenever possible; the PCINT implementation catches interruptions even while the MCU is servicing other tasks.

Refer to `hardware.kicad_sch` for exact connector references, fuse ratings, and any net labels that tie into your firmware pin map.

## Getting Started

1. **Inspect the PCB** – Open the KiCad design and verify clearances, mounting, and connector orientation against your mechanical assembly.
2. **Assemble & power** – Populate the MCU, CH340C, power regulators, and the motion-control connectors. Bring up the board on a bench supply; check that 5 V and any motor rails stay within tolerance under load.
3. **Flash the bootloader** – Use an AVR ISP (e.g., USBasp, Atmel-ICE) on the 6-pin header:

   ```bash
   avrdude -c usbasp -p m2560 -U flash:w:optiboot_atmega2560.hex
   ```

4. **Deploy firmware** – Upload your motion firmware (Marlin, grbl-Mega, custom G-code interpreter) over the CH340C virtual COM port, typically at 115200 baud.
5. **Verify I/O** – Exercise each stepper, servo, and pneumatic output with low-current loads before connecting the actual actuators. Confirm that every endstop triggers in firmware by manually toggling the switches.
6. **Integrate with the PickOMatic toolchain** – Point your host software (Python scripts, OpenPnP, etc.) at the CH340C serial port and stream G-code or custom motion commands.

## Firmware Notes

- **Stepper pairing** – Configure your firmware so the twin X and Y motors share motion planners but can be trammed independently (use dual endstops or sensorless homing if supported).
- **PCINT handling** – Enable pin-change interrupts for the five limit inputs to guarantee hard-stop reactions regardless of motion planner load.
- **Auxiliary steppers** – Treat the dual picker-head steppers as independent “U” and “V” axes or bind them to tool-change macros to orient the nozzles before pick/place moves.
- **Servo timing** – The ATmega2560 has enough timers to dedicate 50 Hz PWM to the three servos; ensure your firmware does not double-book those timers for other peripherals.
- **Pneumatics & lighting** – Map the pump MOSFET and light relay pins to M-code handlers (e.g., `M8/M9` for vacuum, `M42` for light) so pick/place scripts can toggle them inline.

## ATmega2560 Pin Map

Pad numbers reference the TQFP-100 package used on this board. All mappings come directly from `hardware/hardware.kicad_pcb`.

### Stepper channels

| Channel | STEP pin / pad / net | DIR pin / pad / net | Suggested use |
| --- | --- | --- | --- |
| 1 | PC6 · pin 59 · `STEP_1` | PC7 · pin 60 · `DIR_1` | X gantry, left motor |
| 2 | PC4 · pin 57 · `STEP_2` | PC5 · pin 58 · `DIR_2` | X gantry, right motor |
| 3 | PC2 · pin 55 · `STEP_3` | PC3 · pin 56 · `DIR_3` | Y gantry, front motor |
| 4 | PC0 · pin 53 · `STEP_4` | PC1 · pin 54 · `DIR_4` | Y gantry, rear motor |
| 5 | PL5 · pin 40 · `STEP_5` | PL6 · pin 41 · `DIR_5` | Z axis lift |
| 6 | PL3 · pin 38 · `STEP_6` | PL4 · pin 39 · `DIR_6` | Picker A rotation |
| 7 | PL1 · pin 36 · `STEP_7` | PL2 · pin 37 · `DIR_7` | Picker B rotation |

> Firmware can reassign channel roles, but keeping 1–4 for the paired XY screws and 5–7 for the vertical/picker actuators matches the board routing and driver placement.

### Pneumatics, lighting & servos

| Function | Port / pad | Net | Notes |
| --- | --- | --- | --- |
| Vacuum / air pump relay | PA7 · pin 71 | `RELAY_1` | Drives MOSFET + relay combo for the pump manifold. |
| Vision light relay | PA6 · pin 72 | `RELAY_2` | Toggles the inspection/camera light. |
| Servo 1 (picker A tilt) | PE5 · pin 7 | `SERVO_1` | Standard 5 V RC servo header. |
| Servo 2 (picker B tilt) | PE4 · pin 6 | `SERVO_2` | Standard 5 V RC servo header. |
| Servo 3 (shared Z assist) | PE3 · pin 5 | `SERVO_3` | Use for nozzle height trim or feeder kicker. |
| Status LED | PA1 · pin 77 | `LED1` | Tie to firmware heartbeat. |
| Power LED | PA0 · pin 78 | `LED2` | Indicates logic rail alive. |

### End-stop inputs (PCINT capable)

| Input | Port / pad | Net |
| --- | --- | --- |
| Limit 1 | PJ0 · pin 63 | `END_STOP_1` |
| Limit 2 | PJ1 · pin 64 | `END_STOP_2` |
| Limit 3 | PJ2 · pin 65 | `END_STOP_3` |
| Limit 4 | PJ3 · pin 66 | `END_STOP_4` |
| Limit 5 | PJ4 · pin 67 | `END_STOP_5` |

All five pins sit on the `PCINT` bank, so configure them as pin-change interrupts for immediate homing stops.

### Communications & programming

| Interface | Port(s) / pad(s) | Net(s) | Purpose |
| --- | --- | --- | --- |
| USB-UART (CH340C) | PE0 pin 2 / PE1 pin 3 | `RX`, `TX` | Primary G-code link and bootloader uploads. |
| Aux UART1 | PD2 pin 45 / PD3 pin 46 | `RX1`, `TX1` | Spare serial for vision controller or toolchanger. |
| I²C / TWI | PD0 pin 43 / PD1 pin 44 | `SCL`, `SDA` | Plug sensors (pressure, temperature, fiducial LEDs). |
| AVR-ISP / SPI | PB1 pin 20 / PB2 pin 21 / PB3 pin 22 + RESET pin 30 | `SCK`, `MOSI`, `MISO`, `RST` | 6-pin ISP header for bootloader + in-system debugging. |

### Power, clock & reference

- VCC pins (10, 31, 61, 80, 100) tie to `+5V`; GND pins (11, 32, 62, 81, 99) tie to the main ground plane.
- `XTAL1`/`XTAL2` (pads 34/33) connect to the 16 MHz crystal network.
- `AVCC` (pad 100) shares the 5 V rail; add filtering if you extend the analog front-end.
- `AREF` (pad 98) is left uncommitted—strap it to 5 V through 100 nF if you enable analog conversions.

### Spare I/O for future features

PE2, PE6, PE7, PB0, PA2–PA5, PJ5–PJ7, PG0–PG4, PH0–PH7, and the full PF/PK analog banks are presently unassigned. They are broken out on the MCU but not routed elsewhere, so you can bodge-wire or respin the PCB for feeders, cameras, or sensor arrays without touching the core motion stack.

## Bring-Up Checklist

- Continuity-tested all power rails, especially between USB 5 V and main supply.
- Verified correct orientation of CH340C, crystal, and mini-USB connector.
- Checked that every driver enable pin defaults low (disabled) on reset.
- Confirmed pull-ups on endstop inputs so open switches read as “safe”.
- Logged baseline current draw of the pump and lighting relay to size fuses properly.

## License

This hardware is released under the **CERN Open Hardware Licence v2 (Strongly Reciprocal)**. The complete license text is provided in `LICENSE`; keep it with any derivatives or redistributions and review <https://ohwr.org/project/cernohl/wikis/Documents/CERN-OHL-version-2> for updates.

---

Questions, improvements, or bring-up notes are welcome—open an issue or document your findings so the community can build better pick-and-place rigs together.
