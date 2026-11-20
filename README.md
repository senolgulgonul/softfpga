# softfpga — iCE40 and Gowin FPGA Emulator

> A browser-based emulator for iCE40 and Gowin FPGAs. Write Verilog, synthesize with Yosys,
> assign pins, and watch it run on a virtual board — no install, no server, no toolchain.

**Live demo:** <https://senolgulgonul.github.io/softfpga/>

---

## What it does

softfpga compiles real Verilog to an iCE40 logic netlist and simulates it
cycle-by-cycle in your browser. Synthesis is done by Yosys (compiled to
WebAssembly); simulation is a small LUT/flip-flop engine. You connect the design
to a virtual board with a constraint file, the same way you would on real hardware.

The flow is the real FPGA flow:

1. **Synthesize** — Verilog becomes LUTs and flip-flops.
2. **Place & Route** — every port of the design gets a board pin, from your constraints.
3. **Run** — the design is on the board; the clock ticks, LEDs light, switches work.

---

## The virtual board

Every peripheral sits on a numbered pin.

| Pins   | Connected to                                | Direction |
| ------ | ------------------------------------------- | --------- |
| 0 – 7  | switches `sw0` … `sw7`                      | input     |
| 8 – 15 | LEDs `led0` … `led7` **and** the 7-segment display | output |
| 16     | clock, 1 kHz                                | input     |

Pins 8–15 each drive one LED and one segment of the single 7-segment digit:

| Pin     | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| ------- | - | - | -- | -- | -- | -- | -- | -- |
| LED     | led0 | led1 | led2 | led3 | led4 | led5 | led6 | led7 |
| Segment | a | b | c | d | e | f | g | dp |

The clock is fixed at 1 kHz, like the oscillator soldered on a real board; there is
no speed selector. That is far too fast to see, so a design that should be visible
divides it with a counter, exactly as on a real board, only with smaller numbers:
1000 clocks are one second. The examples do this with one constant, `CLK_HZ`.

While the design runs, an LED whose pin switches faster than the eye can follow is
drawn dimmer, according to the share of the time the pin is 1.

A pin at 1 lights its LED and its segment. The display is not decoded for you:
to show a digit, write a hex-to-7-segment decoder (see the `sevenseg` example).

---

## A real board: iCEBreaker

The board menu above the board switches between softfpga's own teaching board and
the [iCEBreaker](https://1bitsquared.com/products/icebreaker) from 1BitSquared, with
its real pin numbers. The iCE40UP5K on it is the chip softfpga emulates anyway.

The virtual iCEBreaker is the board as used in the beginners' workshops:

| Where | What | Pins |
| ----- | ---- | ---- |
| main board | clock `CLK` | 35 |
| main board | user button `BTN_N` (0 while pressed) | 10 |
| main board | red LED `LEDR_N`, green LED `LEDG_N` (lit at 0) | 11, 37 |
| PMOD 2, the snap-off section | `LED1` … `LED5` (lit at 1) | 26 27 25 23 21 |
| PMOD 2, the snap-off section | `BTN1` … `BTN3` (1 while pressed) | 20 19 18 |
| PMOD 1A, 7-segment Pmod | segments a … g (lit at 0) | 4 2 47 45 3 48 46 |
| PMOD 1A, 7-segment Pmod | digit select (1 = right digit) | 44 |
| PMOD 1B, DIP switch Pmod | switches 1 … 8 | 43 38 34 31 42 36 32 28 |

- **Pin file.** The board comes with its pin file, `icebreaker.pcf`, already in the
  second tab. It names every pin (`CLK`, `LED1`, `BTN1`, `P1A1` …); a design that
  uses those names for its ports needs nothing else. `-nowarn` on a line means
  "skip this line if the design has no such port", which is why one file serves
  every design. For a name of your own, add a line: `set_io myport 26`.
- **Buttons.** A click holds a button down and the next click releases it, so that
  several buttons can be down together.
- **Two digits.** The two digits of the 7-segment Pmod share the segment pins; the
  design shows them in turn and selects the digit with pin 44. While the design runs
  both digits are visible, as on the real display.
- **Clock.** softfpga runs `CLK` at 1 kHz; the real board has 12 MHz. The clocked
  examples use one constant, `CLK_HZ`, so that only that number changes.
- **Same files on the real board.** The Verilog and the pin file are what the real
  board's tools read. To build a bitstream and load it, use the open-source iCE40
  flow (Yosys, nextpnr, IceStorm) or Lattice Radiant.

Not drawn, but named in the pin file and visible in the logic analyzer: the UART
pins and the RGB LED. Pin numbers and polarities are taken from the board's
`icebreaker.pcf` and from the iCEBreaker and WTFpga example designs; softfpga was
not checked against a physical board.

---

## A real board: Tang Nano 9K

The [Tang Nano 9K](https://wiki.sipeed.com/hardware/en/tang/Tang-Nano-9K/Nano-9K.html)
from Sipeed carries a Gowin GW1NR-9C (8640 LUTs, 6480 flip-flops). With it selected,
softfpga synthesizes for that chip: Yosys' `synth_gowin` maps the design onto Gowin's
own cells, the same LUTs, ALU carry chains and flip-flops that Gowin's synthesis
uses, and the emulator runs that netlist.

| What | Pins |
| ---- | ---- |
| clock (27 MHz crystal on the board, 1 kHz here) | 52 |
| `LED0` … `LED5` (lit at 0) | 10 11 13 14 15 16 |
| button `S1` (0 while pressed) | 4 |
| button `S2` (0 while pressed) | 3 |
| USB serial TX, RX (not drawn) | 17, 18 |
| the two 24-pin headers (not drawn) | 25–42, 48, 49, 51, 53–57, 63, 68–86 |

- **Pin file as in Gowin IDE.** Constraints are written the Gowin way,
  `IO_LOC "led[0]" 10;`, and `IO_PORT` lines are accepted. A new design starts
  from `tangnano9k.cst`, which lists every pin commented out; the examples bring
  their own `.cst`. The same `.v` and `.cst` open in Gowin IDE.
- **Active low.** Both the LEDs and the buttons are active low, as on the board:
  an LED lights at 0, a button reads 0 while pressed.
- **Clock.** softfpga runs the clock at 1 kHz; the board has 27 MHz. The clocked
  examples use one constant, `CLK_HZ`, so that only that number changes.
- **All the user pins.** Every IO pin of the two headers, of the USB serial and of
  the SPI flash can be given to a port. They are not drawn on the board; the logic
  analyzer shows them. An input on such a pin reads 1, as its pull-up would hold it.
- **Uninitialized registers.** Gowin flip-flops with a set or preset start at 1
  unless the Verilog gives an initial value; softfpga follows the cell library.

Pin numbers come from Sipeed's schematic and from the constraint files of Sipeed's
`TangNano-9K-example` repository. S1 is pin 4 and S2 is pin 3: these are the switches
S1 and S2 in Sipeed's schematic since revision 6202 ("Fixed the silkscreen of S1 and
S2"). The schematic's net names, `PIN3_IOT2A_BUTTON_S1` and `PIN4_IOL5A_JTAG_SEL_S2`,
still carry the old labels, and boards made before that revision may print S1 and S2
the other way round; the pin numbers are what count in the `.cst`.

---

## Two files per design

A design is two files, shown as tabs above the editor:

| File          | Contents                                              |
| ------------- | ----------------------------------------------------- |
| `switches.v`   | the Verilog design; the top module can have any name  |
| `switches.cst` | the constraints: which port goes to which board pin   |

Click a tab to edit that file; double-click it to rename the file.

After **Synthesize** a third, read-only tab shows the **netlist**: the synthesized
circuit as a Verilog file of the chip's cells, one instance per LUT, ALU or
flip-flop, with their settings as `defparam` lines. On the Tang Nano 9K it is
`name.vg`, in the form Gowin EDA writes its own netlist, so the two can be put side
by side; on the iCE40 boards it is `name_netlist.v`. **Save** downloads it. The tab
goes away when the design changes, until the next Synthesize.

## Constraints (.cst)

Port names are free. Nothing is connected to the board until the `.cst` file
says so, one line per port bit:

```
// clock
IO_LOC "clk" 16;

// switches
IO_LOC "sw[0]" 0;
IO_LOC "sw[1]" 1;

// LEDs
IO_LOC "led[0]" 8;
IO_LOC "led[1]" 9;
```

This is the syntax of Gowin EDA's physical constraints file, so the habit
carries over to a real board. The iCE40 PCF form (`set_io led[0] 8`) is read too.
Comments start with `//` or `#`.

**Place & Route** reads the `.cst` file and reports, with the line number:

- a port with no pin, or a pin with two ports
- a port name that is not in the design
- a bus constrained without a bit index (`led` instead of `led[0]`)
- a pin that does not exist on the board
- an output on a switch pin, or an input on an LED pin

If the clock port is not on pin 16, the design is still placed, with a warning.
A clock on a switch pin works: each flip of the switch is a clock edge.

---

## How to use it

The toolbar follows the order of the work:

**Examples · New · Open · Save │ Synthesize · Place & Route · Run ··· Help**

1. Pick a design from **Examples**, or press **New** and write your own. **Open**
   takes a `.v` file, a `.cst` file, or both at once.
2. Press **⚡ Synthesize**. The first synthesis of a session unpacks Yosys from
   `yosys-0.66.1164.js` and starts it, which takes a few seconds; after that
   synthesis is immediate. The console then lists the ports that need a pin.
3. Open the `.cst` tab and give every port a pin. (Examples come with theirs.)
4. Press **📍 Place & Route**. The console prints the pin report or the errors.
5. Press **▶ Run** to start the 1 kHz clock; press it again to pause.
   Place & Route again to start from power-up.
6. Click the switches on the board. Combinational logic reacts at once, without a clock.

The blue button is always the next step: Synthesize, then Place & Route, then
Run. Editing a file moves it back to the step that has to be redone.

**Save** writes the file of the open tab. The bottom panel has three tabs:
**Console**, **Logic analyzer** and **Schematic**.

---

## Logic analyzer

The analyzer
records every placed pin once per clock cycle while the design runs and draws the
waveforms against time.

- **Pins.** The Pins row lists every port that has a pin in the `.cst` file. All
  start switched off, like probes that are not connected yet: click the pins you
  want to watch, click again to hide one. Every pin is recorded whether shown or
  not, so a pin picked later shows its whole history. To watch an internal signal,
  make it an output and give it a pin.
- **Time.** The axis is in seconds since power-up: one cycle of the 1 kHz clock is
  1 ms. The view shows one second by default. Zoomed in, the clock is drawn as a
  square wave, high for the first half of each cycle; zoomed out, as a band.
- **Marker.** Click in the plot to place the marker, a vertical line. Its time is
  shown in the bar (`Marker: 12.5 ms`) and the value of every pin at that moment is
  printed beside its name, as in a waveform viewer. Time is read in 0.1 ms steps
  (a tenth of a cycle); near a clock edge the marker snaps onto the edge. Without a
  marker the values are those of now. Multi-bit ports are also shown as one hex
  number, for example `step=0x3`.
- **Mouse.** Scroll zooms (2 ms to a minute shown; the **−** and **+** buttons do
  the same), drag pans back through the recording, and double-click fits the whole
  recording into the view. While the view shows the newest cycle it follows the
  clock; scrolled back, it stays put.
- **Size.** Drag the bar between the editor and the bottom panel up or down to
  give the waveforms more room; each tab keeps its own height, and a double-click
  on the bar restores the default. With more pins than fit, the plot scrolls.
- The recording is bounded: it keeps the last minute (60,000 cycles, about 0.5 MB)
  and drops older samples. Place & Route starts a new recording. If an error is reported while the analyzer
  is open, the panel switches back to the Console.

---

## Schematic

The **Schematic** tab of the bottom panel draws the circuit that synthesis made of
your design, the same way VeriSim does:

- **Gates** — AND, OR and NOT gates and flip-flops.
- **RTL** — word-level blocks (adders, comparators, multiplexers, registers), with
  the module hierarchy kept.
- **Chip** — the chip's own cells: the netlist Synthesize made and the board runs.
  On the iCE40 boards `SB_LUT4`, `SB_CARRY` and `SB_DFF…`; on the Tang Nano 9K
  `LUT1`–`LUT4`, `ALU` carry chains, `MUX2_LUT5`… and `DFF…`, the cells Gowin EDA
  uses too. Cells get short names (`ALU_3`, `LUT4_7`) so the drawing stays readable.

Scroll zooms, drag pans, double-click fits the drawing into the panel. For a larger
workspace, **⤢ Enlarge** (top right of the bottom panel) makes the panel fill the
window below the toolbar, for the Console and the Logic analyzer as well; Esc or
**⤡ Restore** brings the normal layout back. **↗ New window** opens the drawing in
its own browser tab, with the same zoom and pan, from where it can be saved or printed. The drawing
is made after each successful Synthesize; the bar shows how many cells of each
kind the circuit has. softfpga and VeriSim use the same Yosys release, the same
scripts and the same drawing libraries, so the same design gives the same
schematic in both. Gates and RTL are readable views of the same logic; Chip is the
netlist the emulator runs.

---

## Built-in examples

The Examples menu is grouped, and each example is two files (`name.v`, `name.cst`).

**Gates and wires** (no clock)

| Example   | Demonstrates                                                     |
| --------- | ---------------------------------------------------------------- |
| switches  | Switch passthrough: the switches drive the LEDs                  |
| gates     | Primitives — a half adder from `xor` / `and` gates               |
| fulladder | Hierarchical modeling — a full adder from two half adders        |
| dataflow  | Dataflow modeling — continuous assignment, bitwise operators     |
| adder     | A 4-bit adder, 5-bit result on the LEDs                          |

**Clock and counters**

| Example   | Demonstrates                                                     |
| --------- | ---------------------------------------------------------------- |
| blink     | Clock divider: the LED is on for half a second, off for half     |
| counter   | An 8-bit counter on the LEDs, 10 counts per second               |
| shift     | An 8-bit shift register with a serial input, 4 shifts per second |
| pwm       | PWM with a 4-cycle period: 25 % or 50 % duty, set by a switch    |
| fsm       | Up/down counter with enable and reset                            |

**7-segment display**

| Example   | Demonstrates                                                     |
| --------- | ---------------------------------------------------------------- |
| sevenseg  | Hex to 7-segment decoder: switches select the digit              |
| digits    | Clock divider + counter: one digit per second                    |

With the iCEBreaker selected, the menu lists its examples instead:

| Example         | Demonstrates                                                |
| --------------- | ----------------------------------------------------------- |
| buttons         | Buttons to LEDs; active-high and active-low pins            |
| full_adder_1bit | 1-bit full adder on three buttons (original iCEBreaker example) |
| cmp_1bit        | 1-bit comparator on two buttons (original iCEBreaker example)   |
| led_counter     | Clock divider, a 5-bit counter on the LEDs, a button counter |
| two_digits      | Seconds 00–99 on the two multiplexed digits                 |
| dip_hex         | The DIP switch as a two-digit hex number                    |

With the Tang Nano 9K selected:

| Example      | Demonstrates                                                     |
| ------------ | ---------------------------------------------------------------- |
| two_buttons  | S1 and S2 on the LEDs; active-low inputs and outputs, gates      |
| blink_1s     | Clock divider with `CLK_HZ`: LED0 on for half a second, off for half |
| running_led  | A lit LED runs along the six; S2 is an asynchronous reset        |
| binary_count | A 6-bit counter, four steps a second; S1 stops it, S2 clears it  |

---

## How it works

```
Verilog source
  │
  ▼  Yosys WASM  (@yowasp/yosys 0.66.1164, runs in the browser; same call as VeriSim)
  │  read_verilog design.v; synth_ice40 -nocarry -nobram -flatten; write_json
  │
  │  (Tang Nano 9K: synth_gowin -nobram -nolutram -nodsp -noiopads)
  │
  ▼  yosysJsonToNetlist()
  │  maps SB_LUT4, SB_CARRY and the SB_DFF family; lists the top-level ports
  │  Gowin: LUT1-LUT4, MUX2_LUT5-8 and ALU become truth tables; the DFF family
  │  becomes flip-flops
  │
  ▼  placeAndRoute()
  │  reads the .cst file, checks it, connects each port to its board pin
  │
  ▼  ICE40Emu.tick()  — one clock cycle:
     clock rises → settle logic → flip-flops whose clock had an edge sample → settle
     clock falls → the same again (for falling-edge flip-flops)
```

A LUT4 is a 16-bit truth table indexed by its four inputs:

```js
evalLUT(lut) {
  let addr = 0;
  for (let i = 0; i < 4; i++)
    addr |= this.rd(lut.inputs[i]) << i;   // I0 = LSB, I3 = MSB
  return (lut.truth_table >> addr) & 1;
}
```

Flip-flops are modeled as the real iCE40 cells: clock enable (`SB_DFFE`),
synchronous set/reset (`SB_DFFSR`, `SB_DFFSS`), asynchronous set/reset
(`SB_DFFR`, `SB_DFFS`), their enable variants, and falling-edge versions
(`SB_DFFN*`). Each flip-flop samples on an edge of its own clock net, so clocks
made by other flip-flops (dividers, ripple counters) work.

Gowin cells follow Yosys' Gowin library (`techlibs/gowin/cells_sim.v`). An `ALU` is
two 4-input tables, `SUM` and `COUT`, built from its `ALU_MODE`; a `MUX2_LUT5` …
`MUX2_LUT8` is a 3-input table. They are counted as ALUs and muxes, not as LUTs.
Two details differ from iCE40: in `DFFRE` / `DFFSE` the synchronous reset or set acts
whether or not the clock enable is on, and set or preset flip-flops start at 1.

---

## Verification

The engine was checked against CXXRTL, Yosys' own C++ simulator, running the
original Verilog: 24 designs, pseudo-random inputs, every output compared after
every clock cycle. The set covers counters, shift registers, state machines, UART
transmit and receive, memories, arithmetic, asynchronous resets and a small CPU.

The Gowin path was checked against the iCE40 path: each design synthesized both with
`synth_gowin` and with `synth_ice40`, both netlists run with the same pseudo-random
inputs, every output compared every cycle. 17 designs: synchronous and asynchronous
set and reset with and without enable, falling-edge clocks, initial values, adders,
subtractors and comparators (ALU modes), wide multiplexers, a multiplier, a state
machine, and the Tang Nano 9K examples. No difference.

---

## Repo layout

```
softfpga/
├── index.html          ← the app: HTML, CSS, JS in one file, no build step
├── yosys-0.66.1164.js   ← Yosys (YoWASP build), packed; loaded when Synthesize is pressed
├── elk.bundled.js      ← ELK graph layout, used by the Schematic tab
├── netlistsvg.bundle.js← netlistsvg, draws the Schematic tab
├── README.md
├── _headers            ← Cross-origin headers (Cloudflare Pages)
├── netlify.toml        ← Cross-origin headers (Netlify)
└── vercel.json         ← Cross-origin headers (Vercel)
```

Nothing is loaded from the internet. Yosys sits next to `index.html` as one
16 MB file holding the YoWASP build of Yosys 0.66.1164, gzip-compressed. Because
it is a plain script, softfpga works offline and also when `index.html` is opened
straight from disk. The release is fixed on purpose: the latest YoWASP build has
grown to 79 MB and changes without notice, and 0.66.1164 is the one the engine was
verified with.

Keep the four files together (`index.html`, `yosys-0.66.1164.js`, `elk.bundled.js`,
`netlistsvg.bundle.js`). If `yosys-0.66.1164.js` is missing, softfpga says so in the
console and downloads the same release from the jsDelivr CDN instead. Without the
other two, everything works except the Schematic tab.

---

## Limitations

softfpga models the **chip's logic fabric** (real LUT4s, real flip-flops, real
Yosys synthesis). Its own teaching board is not a commercial one, and its pin
numbers are not the pins of an iCE40 package; the iCEBreaker and Tang Nano 9K
boards use the real ones.

Not modeled: signal timing and propagation delay, physical placement and routing,
block RAM, DSP blocks, PLLs, tri-state and bidirectional pins, latches, and
bitstream generation. Memories are built from flip-flops, so keep them small.
The board clock is 1 kHz; designs that divide a MHz clock need a smaller divider here.
On the Tang Nano 9K, multipliers are built from LUTs and ALUs instead of DSP blocks.

---

## Roadmap

- [x] Yosys WASM Verilog synthesis
- [x] LUT4 + full SB_DFF family (up to 5280 LUTs)
- [x] Constraint file and Place & Route step
- [x] Virtual board: 8 switches, 8 LEDs, 7-segment digit, clock
- [x] Schematic of the synthesized circuit, as gates or RTL
- [x] Two-file designs (`.v` + `.cst`): new, open, save
- [x] Logic analyzer on the board pins
- [ ] UART terminal on two board pins
- [ ] Waveform (VCD) export
- [ ] Real-time clock (MHz range)
- [ ] Block RAM emulation
- [x] A real board: iCEBreaker, with its pin file and examples
- [x] A second real board: Tang Nano 9K (Gowin GW1NR-9C), with Gowin synthesis
- [ ] Open a netlist from Gowin IDE (`.vg`) and run it on the Tang Nano 9K

**Out of scope: bitstream generation.** softfpga is for people who do not have a
board. If you have one, use the vendor's free tools to build and load the
bitstream (Gowin EDA for Tang Nano boards, Lattice Radiant or the open-source
Yosys / nextpnr flow for iCE40). What you practised here carries over: the Verilog,
the pin constraints, and the habit of checking the design with a logic analyzer.

---

## How to cite

> Gülgönül, S. (2026). *softfpga: A browser-based iCE40 FPGA emulator.* <https://github.com/senolgulgonul/softfpga>

```
@software{softfpga,
  author  = {G\"ulg\"on\"ul, \c{S}enol},
  title   = {softfpga: A browser-based iCE40 FPGA emulator},
  year    = {2026},
  url     = {https://github.com/senolgulgonul/softfpga},
  version = {2.29}
}
```

---

## Acknowledgements

softfpga uses [Yosys](https://yosyshq.net/yosys/) for synthesis, via the
[YoWASP](https://yowasp.org/) WebAssembly build. Yosys is developed by
Claire Xenia Wolf and the YosysHQ team (ISC license).

The iCEBreaker is an open-hardware board by [1BitSquared](https://1bitsquared.com/).
The `full_adder_1bit` and `cmp_1bit` examples are from
[icebreaker-verilog-examples](https://codeberg.org/icebreaker-fpga/icebreaker-verilog-examples)
by Albin Söderqvist (ISC license), included unchanged with their license text.

The Schematic tab is drawn by [netlistsvg](https://github.com/nturley/netlistsvg)
by Neil Turley (MIT license) with the [ELK](https://github.com/kieler/elkjs) layout
engine (EPL-2.0).

---

## License

MIT
