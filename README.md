# softfpga — iCE40 FPGA Emulator

> A browser-based emulator for iCE40 FPGAs. Write Verilog, synthesize with Yosys,
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
| 16     | clock, 10 Hz                                | input     |

Pins 8–15 each drive one LED and one segment of the single 7-segment digit:

| Pin     | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
| ------- | - | - | -- | -- | -- | -- | -- | -- |
| LED     | led0 | led1 | led2 | led3 | led4 | led5 | led6 | led7 |
| Segment | a | b | c | d | e | f | g | dp |

The clock is fixed at 10 Hz, like the oscillator soldered on a real board; there is
no speed selector. A design that needs a slower rate divides it (see `digits`).

A pin at 1 lights its LED and its segment. The display is not decoded for you:
to show a digit, write a hex-to-7-segment decoder (see the `sevenseg` example).

---

## Two files per design

A design is two files, shown as tabs above the editor:

| File          | Contents                                              |
| ------------- | ----------------------------------------------------- |
| `switches.v`   | the Verilog design; the top module can have any name  |
| `switches.cst` | the constraints: which port goes to which board pin   |

Click a tab to edit that file; double-click it to rename the file.

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

**Examples · New · Open · Save │ Synthesize · Place & Route · Run · Schematic ··· Help**

1. Pick a design from **Examples**, or press **New** and write your own. **Open**
   takes a `.v` file, a `.cst` file, or both at once.
2. Press **⚡ Synthesize**. The first synthesis of a session unpacks Yosys from
   `yosys-0.50.857.js` and starts it, which takes a few seconds; after that
   synthesis is immediate. The console then lists the ports that need a pin.
3. Open the `.cst` tab and give every port a pin. (Examples come with theirs.)
4. Press **📍 Place & Route**. The console prints the pin report or the errors.
5. Press **▶ Run** to start the 10 Hz clock; press it again to pause.
   Place & Route again to start from power-up.
6. Click the switches on the board. Combinational logic reacts at once, without a clock.

The blue button is always the next step: Synthesize, then Place & Route, then
Run. Editing a file moves it back to the step that has to be redone.

**Schematic** shows the synthesized circuit: LUTs, flip-flops and the ports
(with their pins once placed). **Save** writes the file of the open tab.

---

## Built-in examples

| Example   | Demonstrates                                                     |
| --------- | ---------------------------------------------------------------- |
| switches  | Switch passthrough: the switches drive the LEDs                  |
| gates     | Primitives — a half adder from `xor` / `and` gates               |
| fulladder | Hierarchical modeling — a full adder from two half adders        |
| dataflow  | Dataflow modeling — continuous assignment, bitwise operators     |
| adder     | A 4-bit adder, 5-bit result on the LEDs                          |
| sevenseg  | Hex to 7-segment decoder: switches select the digit              |
| blink     | A single flip-flop toggling every clock                          |
| counter   | An 8-bit synchronous counter on the LEDs                         |
| shift     | An 8-bit shift register with a serial input                      |
| pwm       | PWM with a 4-cycle period: 25 % or 50 % duty, set by a switch    |
| fsm       | Behavioral modeling — up/down counter with enable and reset      |
| digits    | Clock divider + counter: one digit per second on the 7-segment   |

---

## How it works

```
Verilog source
  │
  ▼  Yosys WASM  (@yowasp/yosys 0.50.857, runs in the browser; same call as VeriSim)
  │  read_verilog design.v; synth_ice40 -nocarry -nobram -flatten; write_json
  │
  ▼  yosysJsonToNetlist()
  │  maps SB_LUT4, SB_CARRY and the SB_DFF family; lists the top-level ports
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

---

## Changes in v2

- **Constraints and Place & Route.** The reserved port names (`clk`, `sw`, `led`,
  `hex`) are gone. Ports get their pins from a `.cst` file.
- **Two file tabs** (`name.v`, `name.cst`) with New, Open and Save. The Verilog /
  Netlist JSON mode buttons, Step, Reset and the clock speed selector are removed.
- **Fixed 10 Hz board clock** on pin 16.
- **One 7-segment digit** on pins 8–15, shared with the LEDs, driven segment by
  segment. The separate decoded `hex` port no longer exists.
- **Flip-flop enable, set and reset are simulated.** v1 read only `D` and `Q`, so
  a design such as `if (en) q <= d;` behaved as if `en` were always 1.
- **Any top module name**, and the synthesis call is the one VeriSim uses.
- **Switches act immediately**; combinational designs no longer need the clock.
- **Derived and falling-edge clocks** are simulated.

The hand-written JSON netlist mode of v1 is removed.

---

## Verification

The engine was checked against CXXRTL, Yosys' own C++ simulator, running the
original Verilog: 23 designs, pseudo-random inputs, every output compared after
every clock cycle. The set covers counters, shift registers, state machines, UART
transmit and receive, memories, arithmetic, asynchronous resets and a small CPU.

---

## Repo layout

```
softfpga/
├── index.html          ← the app: HTML, CSS, JS in one file, no build step
├── yosys-0.50.857.js   ← Yosys (YoWASP build), packed; loaded when Synthesize is pressed
├── README.md
├── _headers            ← Cross-origin headers (Cloudflare Pages)
├── netlify.toml        ← Cross-origin headers (Netlify)
└── vercel.json         ← Cross-origin headers (Vercel)
```

Nothing is loaded from the internet. Yosys sits next to `index.html` as one
11 MB file holding the YoWASP build of Yosys 0.50.857, gzip-compressed. Because
it is a plain script, softfpga works offline and also when `index.html` is opened
straight from disk. The release is fixed on purpose: the latest YoWASP build has
grown to 79 MB and changes without notice, and 0.50.857 is the one the engine was
verified with.

Keep the two files together. If `yosys-0.50.857.js` is missing, softfpga says so
in the console and downloads the same release from the jsDelivr CDN instead.

---

## Limitations

softfpga models the **chip's logic fabric** (real LUT4s, real flip-flops, real
Yosys synthesis). The board is a teaching board, not a commercial one, and its
pin numbers are not the pins of an iCE40 package.

Not modeled: signal timing and propagation delay, physical placement and routing,
block RAM, DSP blocks, PLLs, tri-state and bidirectional pins, latches, and
bitstream generation. Memories are built from flip-flops, so keep them small.
The board clock is 10 Hz; designs that divide a MHz clock need a smaller divider here.

---

## Roadmap

- [x] Yosys WASM Verilog synthesis
- [x] LUT4 + full SB_DFF family (up to 5280 LUTs)
- [x] Constraint file and Place & Route step
- [x] Virtual board: 8 switches, 8 LEDs, 7-segment digit, clock
- [x] Schematic of the synthesized circuit
- [x] Two-file designs (`.v` + `.cst`): new, open, save
- [ ] UART terminal on two board pins
- [ ] Logic analyzer on the board pins
- [ ] Real-time clock (MHz range)
- [ ] Block RAM emulation

---

## How to cite

> Gülgönül, S. (2026). *softfpga: A browser-based iCE40 FPGA emulator.* <https://github.com/senolgulgonul/softfpga>

```
@software{softfpga,
  author  = {G\"ulg\"on\"ul, \c{S}enol},
  title   = {softfpga: A browser-based iCE40 FPGA emulator},
  year    = {2026},
  url     = {https://github.com/senolgulgonul/softfpga},
  version = {2.12}
}
```

---

## Acknowledgements

softfpga uses [Yosys](https://yosyshq.net/yosys/) for synthesis, via the
[YoWASP](https://yowasp.org/) WebAssembly build. Yosys is developed by
Claire Xenia Wolf and the YosysHQ team.

---

## License

MIT
