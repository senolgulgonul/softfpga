# Softfpga — iCE40 FPGA Emulator

> A browser emulator for iCE40 FPGAs. Write Verilog, simulate instantly. No install. No server.

**Live demo:** `(https://senolgulgonul.github.io/softfpga/)`

---

## What it does

- **Verilog mode** — write real Verilog, compiled by Yosys WASM in the browser
- **JSON mode** — direct LUT/FF netlist for instant simulation, zero dependencies
- **iCE40UP5K parity** — up to 5,280 LUTs, enforced at compile time
- **Virtual IO** — LEDs, 7-segment hex display, input switches
- **LUT inspector** — 16-entry truth table with live active-entry highlight
- **Utilisation bar** — see how much of the chip your design uses

---

## Repo layout

```
softice/
├── index.html      ← entire app, single file, no build step
├── README.md
├── _headers        ← Cloudflare Pages COOP/COEP headers
├── netlify.toml    ← Netlify headers
└── vercel.json     ← Vercel headers
```

## How it works

Verilog
  │
  ▼  Yosys WASM (@yowasp/yosys via jsDelivr, runs in browser)
  │  synth_ice40 -json → Yosys netlist JSON
  │
  ▼  yosysJsonToNetlist() — maps SB_LUT4 / SB_DFF cells
  │
  ▼  ICE40Emu.tick() — per clock:
     1. eval all LUT4 truth tables  (combinational)
     2. clock all D flip-flops      (sequential)
     3. read IO nets → LEDs / 7-seg
```

### The LUT4 core (the whole emulator in 4 lines)

```js
evalLUT(lut) {
  let addr = 0;
  for (let i = 0; i < 4; i++)
    addr |= (nets[lut.inputs[i]] & 1) << i;
  return (lut.truth_table >> addr) & 1;
}
```

---

## Verilog tips

```verilog
// Name outputs "led" or "led[N]" → auto-detected as LEDs
output [7:0] led

// Name inputs "sw_*" → detected as toggle switches
input sw_enable

// Keep designs synchronous — async paths work but are harder to map
always @(posedge clk) count <= count + 1;
```

---

## Roadmap

- [x] JSON netlist emulation
- [x] Yosys WASM Verilog compile
- [x] LUT4 + D-FF fabric (up to 5280 LUTs)
- [x] Virtual LEDs, 7-segment, switches
- [x] LUT inspector + utilisation meter
- [ ] UART terminal peripheral
- [ ] Clock prescaler (slow down fast designs visually)
- [ ] BRAM emulation
- [ ] VGA / framebuffer
- [ ] Waveform export (VCD)
- [ ] nextpnr place & route (real bitstream path)

---


## License

MIT
