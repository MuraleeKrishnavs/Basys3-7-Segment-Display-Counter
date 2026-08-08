# Basys3 7-Segment Display Counter

VHDL design for the Digilent **Basys3** (Xilinx Artix-7) FPGA board that drives the on-board 4-digit 7-segment display. After reset it steps through the display's digit positions, then cycles through the letters **F-P-G-A**. A push-button reset restarts the sequence.

## Features

- Drives the Basys3's 4-digit, 7-segment display (`an` / `seg`)
- On-chip clock divider that slows the 100 MHz board clock to a human-visible rate
- Digit-by-digit counting sequence, ending in a repeating **F → P → G → A** display
- Asynchronous reset via the center push-button (`btnC`)

## Hardware & Tools

| | |
|---|---|
| Board | Digilent Basys3 |
| FPGA | Xilinx Artix-7, `xc7a35tcpg236-1` |
| HDL | VHDL |
| Toolchain | Xilinx Vivado (Vivado Simulator, mixed-language) |
| Top module | `counter_seg` |

## Repository Structure

```
.
├── counter_seg.vhd    # Top-level design: clock divider, counting FSM, display driver
├── constraints.xdc     # Pin constraints for the Basys3 (add — see Pin Constraints below)
└── README.md
```

## Ports

| Port | Dir | Width | Description |
|---|---|---|---|
| `clk` | in | 1 | 100 MHz board clock |
| `btnC` | in | 1 | Center push-button — asynchronous reset |
| `an` | out | 4 | Active-low anode select (chooses which digit is lit) |
| `seg` | out | 7 | Active-low segment select, segments a–g |

## How It Works

**1. Clock divider.** `clk` is divided by counting up to 9,999,999 and toggling `clk_slow` each time it wraps. At the Basys3's 100 MHz clock that's an update roughly every 0.1 s, so the digits change at a watchable, human-visible pace (about 5 times a second).

**2. Counting sequence.** Rather than a standard 0000→9999 odometer count, the design fills in one digit at a time, tracked by `digit_active`:
1. `unit` (ones) counts 0→9 while the other three digits hold at 0
2. `tens` then counts 0→9 while `unit` holds at 0
3. `hun` (hundreds) counts 0→9
4. `thou` (thousands) counts 0→9
5. When `thou` reaches 9, the design sets `word_active <= '1'` and stops counting

The whole sequence takes roughly 8 seconds from reset to reaching the letter display.

**3. Display output.** `an` and `seg` are driven from `digit_active`, which is the *same* signal used above to track counting progress. Because it only changes once per `clk_slow` tick (~every 0.1–0.2 s), only **one digit is lit at a time** rather than all four being refreshed fast enough for persistence-of-vision:
- **Counting phase** — the active digit visibly counts 0→9 on its own before the next digit to its left lights up and starts counting.
- **Letter phase** (`word_active = '1'`) — `digit_active` free-runs 0→3 and the `seg_char` function shows **F**, **P**, **G**, **A** one at a time, each for ~0.2 s, cycling continuously. That's too slow for the eye to blend into a steady "FPGA" — expect to see the letters flash in sequence rather than all four glowing at once.

**4. Reset.** Driving `btnC` high asynchronously clears every counter, `digit_active`, and `word_active`, restarting from `0000`.

## Pin Constraints

Add an `.xdc` file mapping `clk`, `btnC`, `an`, and `seg` to the board. This is the standard Basys3 pinout for those signals:

```tcl
## Clock signal
set_property -dict { PACKAGE_PIN W5   IOSTANDARD LVCMOS33 } [get_ports clk]
create_clock -add -name sys_clk_pin -period 10.00 -waveform {0 5} [get_ports clk]

##7 Segment Display
set_property -dict { PACKAGE_PIN W7   IOSTANDARD LVCMOS33 } [get_ports {seg[0]}]
set_property -dict { PACKAGE_PIN W6   IOSTANDARD LVCMOS33 } [get_ports {seg[1]}]
set_property -dict { PACKAGE_PIN U8   IOSTANDARD LVCMOS33 } [get_ports {seg[2]}]
set_property -dict { PACKAGE_PIN V8   IOSTANDARD LVCMOS33 } [get_ports {seg[3]}]
set_property -dict { PACKAGE_PIN U5   IOSTANDARD LVCMOS33 } [get_ports {seg[4]}]
set_property -dict { PACKAGE_PIN V5   IOSTANDARD LVCMOS33 } [get_ports {seg[5]}]
set_property -dict { PACKAGE_PIN U7   IOSTANDARD LVCMOS33 } [get_ports {seg[6]}]

set_property -dict { PACKAGE_PIN V7   IOSTANDARD LVCMOS33 } [get_ports dp]

set_property -dict { PACKAGE_PIN U2   IOSTANDARD LVCMOS33 } [get_ports {an[0]}]
set_property -dict { PACKAGE_PIN U4   IOSTANDARD LVCMOS33 } [get_ports {an[1]}]
set_property -dict { PACKAGE_PIN V4   IOSTANDARD LVCMOS33 } [get_ports {an[2]}]
set_property -dict { PACKAGE_PIN W4   IOSTANDARD LVCMOS33 } [get_ports {an[3]}]


##Buttons
set_property -dict { PACKAGE_PIN U18   IOSTANDARD LVCMOS33 } [get_ports btnC]

## Configuration options, can be used for all designs
set_property CONFIG_VOLTAGE 3.3 [current_design]
set_property CFGBVS VCCO [current_design]

## SPI configuration mode options for QSPI boot, can be used for all designs
set_property BITSTREAM.GENERAL.COMPRESS TRUE [current_design]
set_property BITSTREAM.CONFIG.CONFIGRATE 33 [current_design]
set_property CONFIG_MODE SPIx4 [current_design]

```

## Getting Started

1. Clone this repository
2. In Vivado, create an RTL project targeting part `xc7a35tcpg236-1`
3. Add `counter_seg.vhd` as a design source and set `counter_seg` as the top module
4. Add an `.xdc` constraints file (see [Pin Constraints](#pin-constraints)) as a constraints source
5. Run Synthesis → Implementation → Generate Bitstream
6. Program the Basys3 board; press `btnC` to reset and restart the sequence

## Possible Extensions

- Decouple the display refresh rate from the counting rate (e.g. multiplex at ~1 kHz) so all 4 digits appear lit simultaneously via persistence of vision
- Replace the digit-by-digit sequence with a true carrying 0000–9999 counter
- Add a switch or button to pause/freeze the count
- Use the decimal point for a clock-style `MM:SS` display
- Add a testbench for the Vivado Simulator (the project is already set up for mixed-language simulation)

## License

No license chosen yet — consider adding one (e.g. MIT) if you plan to make this repository public.
