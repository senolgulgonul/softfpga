# softfpga — iCE40 FPGA Emulator

> A browser-based emulator for iCE40 FPGAs. Write Verilog, synthesize with Yosys,
> and watch it run on a virtual board — no install, no server, no toolchain.

**Live demo:** https://senolgulgonul.github.io/softfpga/

---

## What it does

softfpga compiles real Verilog to an iCE40 logic netlist and simulates it
cycle-by-cycle in your browser. Synthesis is done by Yosys (compiled to
WebAssembly); simulation is a small LUT/flip-flop engine. The result is wired
to a fixed virtual board with LEDs, a hex display, and switches.

- **Verilog mode** — real Verilog, synthesized in-browser by Yosys WASM
- **JSON mode** — hand-written LUT/FF netlists for instant simulation, zero dependencies
- **iCE40UP5K model** — LUT4 + D-flip-flop fabric, up to 5,280 LUTs
- **Fixed virtual board** — 8 LEDs, a 2-digit hex display, 8 switches
- **LUT inspector** — view any LUT's 16-entry truth table with the active row highlighted
- **Netlist dump** — export the raw Yosys JSON and the converted netlist for debugging

---

## The virtual board: reserved IO names

A real FPGA needs a *constraint file* (a `.pcf`) that maps each top-level
signal to a physical pin. softfpga removes that step entirely. Instead, the
board is **fixed**, and you connect to its peripherals simply by **naming your
Verilog ports with reserved names**. The port name *is* the pin assignment.

| Verilog port              | Connects to            | Notes                                  |
|---------------------------|------------------------|----------------------------------------|
| `input clk`               | the clock              | advances one tick per simulated cycle  |
| `input [7:0] sw`          | 8 toggle switches      | click a switch in the UI to set its bit |
| `output [7:0] led`        | 8 LEDs                 | bit 0 = `led[0]`, shown rightmost      |
| `output [7:0] hex`        | 2-digit hex display    | shows the 8-bit value as two hex digits |

Rules:

- Names are matched by **prefix**: any output beginning with `led` drives LEDs,
  any beginning with `hex` drives the hex display, any input beginning with
  `sw` becomes switches.
- The **hex display is driven only by a `hex` port.** A design with no `hex`
  output leaves the display blank — there is no fallback to the LED value.
- LEDs are shown **MSB-left, LSB-right**, so the row reads like a binary number.
- An output port that isn't recognized as `hex` defaults to driving LEDs.

This is why **no place & route is needed.** A normal FPGA flow is
*synthesis → place & route → bitstream*, where place & route assigns physical
locations and pins. softfpga does real synthesis but replaces place & route
with the naming convention above. You trade physical realism (signal timing,
pin locations, routing) for a zero-configuration experience — the right
trade-off for learning and experimenting with digital logic.

---

## How to use it

1. Open the app and make sure the **⚡ Verilog** tab is selected.
2. Pick an example (counter, blink, shift, pwm, adder) or write your own.
3. Press **⚡ Compile**. On first use this downloads Yosys WASM (~8 MB from a
   CDN, cached afterwards), then synthesizes your design. The console reports
   the LUT / flip-flop / IO counts.
4. Press **▶ Run** to start the clock, or **⏭ Step** to advance one cycle at a
   time. **↺ Reset** clears all flip-flops back to zero.
5. Use the **clock speed** dropdown to slow fast designs down enough to watch.
6. For designs with switches, click the switch toggles to set input bits, then
   Run or Step to see the effect.

The typical loop is just **Compile → Run**. Edit, Compile again, Run again.

### Opening and saving files

A small toolbar above the editor lets you work with files on your own disk:

- **📂 Open** — load a Verilog (`.v`, `.sv`, `.vh`) or netlist (`.json`) file
  from disk into the editor. The mode switches automatically to match the
  file type.
- **💾 Save** — write the editor contents back to a file. Edit the filename in
  the toolbar field first to choose the name. On Chromium browsers (Chrome,
  Edge) served over `https`/`localhost`, this opens a native *Save As* dialog;
  elsewhere it downloads the file to your browser's download folder.

Everything is fully client-side — no upload, no server. The Save extension is
corrected automatically to `.v` or `.json` to match the current mode.

### Writing your own design

```verilog
// An 8-bit counter shown on the LEDs and the hex display
module top (
  input  clk,
  output [7:0] led,
  output [7:0] hex
);
  reg [7:0] count = 0;
  always @(posedge clk)
    count <= count + 1;
  assign led = count;
  assign hex = count;
endmodule
```

The top module must be named `top`. Declare only the reserved ports you need.
Keep designs synchronous (`always @(posedge clk)`) — that is what the
flip-flop-based simulation models.

---

## Built-in examples

| Example  | Demonstrates                                              |
|----------|-----------------------------------------------------------|
| counter  | An 8-bit synchronous counter on LEDs and hex              |
| blink    | A single flip-flop toggling every clock                   |
| shift    | An 8-bit shift register; `sw[0]` is the serial input      |
| pwm      | A PWM generator; `sw[0]` selects the duty cycle           |
| adder    | A 4-bit adder: `sw[3:0] + sw[7:4]`, result on LEDs and hex |

---

## How it works

```
Verilog source
  │
  ▼  Yosys WASM  (@yowasp/yosys, runs in the browser)
  │  synth_ice40 -nocarry -flatten -json   →   Yosys netlist JSON
  │
  ▼  yosysJsonToNetlist()
  │  maps SB_LUT4 and SB_DFF cells; resolves nets; tags IO by reserved name
  │
  ▼  ICE40Emu.tick()  — once per clock cycle:
     1. settle all combinational logic (LUT truth tables)
     2. sample every flip-flop's D input, then apply the clock edge
     3. read the IO nets and update LEDs / hex display
```

The whole simulation core is a LUT4 evaluator and a flip-flop update. A LUT4
is a 16-bit truth table indexed by its four inputs:

```js
evalLUT(lut) {
  let addr = 0;
  for (let i = 0; i < 4; i++)
    addr |= this.rd(lut.inputs[i]) << i;   // I0 = LSB, I3 = MSB
  return (lut.truth_table >> addr) & 1;
}
```

`synth_ice40` is run with `-nocarry` so adders map onto plain LUTs, which keeps
the simulation engine simple.

---

## Repo layout

```
softfpga/
├── index.html      ← the entire app: HTML, CSS, JS in one file, no build step
├── README.md
├── _headers        ← Cross-origin headers (Cloudflare Pages)
├── netlify.toml    ← Cross-origin headers (Netlify)
└── vercel.json     ← Cross-origin headers (Vercel)
```

`index.html` is fully self-contained. The only external dependency is the
Yosys WASM module, loaded from a CDN on first compile in Verilog mode.
JSON mode is fully offline.

---

## Deploy

### GitHub Pages

```bash
git clone https://github.com/senolgulgonul/softfpga
cd softfpga
# edit files, then:
git add . && git commit -m "update" && git push
# Settings → Pages → Source: main / root → Save
```

### Cloudflare Pages / Netlify / Vercel

The repo includes `_headers`, `netlify.toml`, and `vercel.json` which set the
`Cross-Origin-Opener-Policy` and `Cross-Origin-Embedder-Policy` headers used
for optimal Yosys WASM performance. Connect the repo, set the output directory
to `.` with no build command, and deploy.

---

## Limitations

softfpga models the **chip's logic fabric** faithfully (real LUT4s, real
flip-flops, real Yosys synthesis), but the **board peripherals are a fixed
simplification**. A real iCE40UP5K has 39 general-purpose IO pins and no
built-in LEDs or displays — what those pins connect to depends on the physical
board. softfpga instead provides one fixed board (8 LEDs, hex display, 8
switches) addressed by reserved names.

Not modeled: signal timing and propagation delay, place & route, block RAM,
DSP/multiplier blocks, the hardened RGB LED driver, PLLs, and bitstream
generation. Pure wire-through designs (an output assigned directly from an
input with no logic) are not currently supported.

---

## Roadmap

- [x] JSON netlist emulation
- [x] Yosys WASM Verilog synthesis
- [x] LUT4 + D-flip-flop fabric (up to 5280 LUTs)
- [x] Fixed virtual board: 8 LEDs, hex display, 8 switches
- [x] Reserved-name IO model (no constraint file needed)
- [x] LUT inspector and utilisation meter
- [x] Netlist dump for debugging
- [x] Open / save .v files from local disk
- [ ] Clock prescaler for visually slowing fast designs
- [ ] Block RAM emulation
- [ ] UART terminal peripheral
- [ ] Waveform (VCD) export
- [ ] Pure wire-through (combinational pass-through) support

---

## How to cite

If you use softfpga in academic work, teaching material, or a publication,
please cite it. Update the year and URL as appropriate.

**Plain text:**

> Gülgönül, S. (2026). *softfpga: A browser-based iCE40 FPGA emulator.*
> https://github.com/senolgulgonul/softfpga

**BibTeX:**

```bibtex
@software{softfpga,
  author  = {G\"ulg\"on\"ul, \c{S}enol},
  title   = {softfpga: A browser-based iCE40 FPGA emulator},
  year    = {2026},
  url     = {https://github.com/senolgulgonul/softfpga},
  version = {1.21}
}
```

For a citable, versioned snapshot with a DOI, you can connect the GitHub
repository to [Zenodo](https://zenodo.org) and create a release; Zenodo will
mint a DOI for each tagged version.

---

## Acknowledgements

softfpga uses [Yosys](https://yosyshq.net/yosys/) for synthesis, via the
[YoWASP](https://yowasp.org/) WebAssembly build. Yosys is developed by
Claire Xenia Wolf and the YosysHQ team.

---

## License

MIT
