---
name: logisim-circ-to-hdl
description: Analyze Logisim Evolution `.circ` schematics (emu-russia/breaks PPU_Evo.circ) and translate them to Verilog HDL — the XML format, component/wire geometry for tracing connectivity, the subcircuit default pin appearance, and the recurring translator bug patterns (undriven z outputs, LSB-first bus reversal, uninitialized RAM x).
whenToUse: Use when reading or analyzing a `.circ` file, tracing Logisim circuit connectivity, writing or fixing a `.circ`→Verilog translator, or debugging floating (z/x) signals in translator-generated HDL against the schematic.
---

# Logisim `.circ` → Verilog analysis

Hard-won facts from completing the NES PPU Verilog model (emu-russia/breaks, issue #1376). The schematic is the ground truth; the Verilog must match its wiring, not the translator's assumptions.

## Where things live

- Reference schematic: `/mnt/c/Work/breaks/Logisim/Evolution/PPU_Evo.circ` (created by Logisim-evolution **v3.8.0**; `<project source="3.8.0" version="1.0">`).
- Translated Verilog (committed, hand-verifiable): `HDL/PPU/*.v`, primitives in `HDL/Common/*.v`.
- Translator (a means-to-end, lives in `/tmp` — ephemeral, do not rely on it persisting): `/tmp/circ2v/circ2v.py` + `finalize_all.py` / `integrate2.py`.

## `.circ` XML format

- `<circuit name="...">` blocks do NOT nest — scan balanced `<circuit>`/`</circuit>`.
- `<comp lib="0" loc="(x,y)" name="Pin">` carries `<a name="label" val="..."/>`, `<a name="width" val="N"/>`, `<a name="output" val="true"/>` (else input), `<a name="facing" val="west"/>`.
- `<wire from="(x1,y1)" to="(x2,y2)"/>` — horizontal or vertical segments only; two endpoints at the same `(x,y)` are the same net; a vertical wire passing through a point also connects it.
- Subcircuit instances are self-closing: `<comp loc="(x,y)" name="SubName"/>`. Nested custom subcircuits like `DLatch_x4`, `MUX_Control`, `Tile_TH_Counter` may be self-closing or carry `<a>` children (label etc.).
- `lib` codes: `0` = Wiring (Pin, Constant, Tunnel, Splitter, NoConnect), `1` = Gates (NOT Gate, NOR/AND/OR/NAND…), `2` = Plexers (Multiplexer, Decoder, Demultiplexer), `4` = Memory (D Flip-Flop, RAM). Subcircuits have no `lib`.
- A circuit with `<a name="circuitnamedboxfixedsize" val="true"/>` renders with a fixed-size box (affects default pin offset width).

## Component geometry (pin locations)

- Gate `loc` = **output** pin (east side). NOT/Buffer input at `(x-size, y)`; NOT `size` default 30, so input at `(x-30, y)`.
- Multi-input gates: `size` default 50; NAND/NOR add 10 to the axis length for the negated output. The `negate{i}` attribute (0-based) refers to the *input* at index `i` and adds 10 to that input's x-offset.
- **Multiplexer** (`lib=2`): `loc` = output `O`. For a 2-input mux facing east (wide): `D0=(x-30,y-10)`, `D1=(x-30,y+10)`, `S=(x-20, y±20)` — `selloc="tr"` → `-20`, `selloc="bl"` (default) → `+20`. `width="4"` makes a 4-bit bus mux. Narrow (`size="narrow"`) shrinks w/s to 20/10.
- **Splitter** (`lib=0`): combined BUS at `loc`; individual fanout bits at computed offsets. Facing east: `dxEnd0=+20`, `ddy=+10` per bit; facing west: `dxEnd0=-20`. `appear="right"` (vs default `left`) changes the vertical justification (`dyEnd0`). **F{i} maps to bus bit `i`** (bit0 = LSB) unless explicit `bit{i}` attributes say otherwise — the v3.8.0 default is `bit{i}=i`.
- Splitter with `fanout == incoming` (a "1:1" splitter) is a plain bus-split/join; `fanout < incoming` merges.

## Subcircuit default pin appearance — VERIFY, don't trust

`DefaultEvolutionAppearance` (v3.8.0) places a subcircuit's pins at: outputs on the EAST edge at `(0, 20k)` from the anchor, inputs on the WEST edge at `(-width, 20k)`; `width = (textWidth/10)*10 + 20`, fixed-size `textWidth = 25*8 = 200` → `width=220`; `dy=20`; the anchor sits on the first output pin.

**Critical gotcha:** in `PPU_Evo.circ` the actual subcircuit pin positions did NOT match that formula — empirically `MUX_Control` and `DLatch_x4` pins were at **10px spacing** with a **-110** x-offset for inputs (not 20px / -220). The wires are the ground truth, so always determine pin positions empirically from the wires rather than trusting a computed offset. Method: find the wire endpoints that form a vertical column adjacent to each instance; those are the real pins.

## Connectivity tracing (the reliable method)

1. Parse all `<comp>`/`<wire>` for the circuit.
2. Build a union-find over `(x,y)` points: union every wire's two endpoints, and union each component port's `(x,y)` with itself.
3. Group ports by their union-find root → each group is one net.
4. Read off which component pins share a net. This gives the true connectivity regardless of naming/geometry bugs.

This is how floating (z) signals were diagnosed: a module output port whose net group contains only itself is undriven; a bus whose fanout bits are wired but whose combined end is a dead-end lost the connection.

## Translator bug patterns (why signals end up z/x)

1. **Undriven output** (z): an output declared but never assigned. Cause: the bus fans out through a splitter to other named nets, so the canonical naming resolves the bus to those nets (a concatenation) and the output pin itself is left floating. Examples in PPU: `THO`/`TVO` (TileCnt), `n_FVO`/`n_THO`/`n_TVO` (counter inverted outputs), `PAddr_out[12]` (PAR), `CGA[3:0]` (PictureMUX).
2. **Splitter combined end not connected**: a splitter's BUS goes to a dead-end wire because the subcircuit pin it should land on was computed at the wrong offset (see appearance gotcha). Example: the PictureMUX BG/OBJ color mux pipeline.
3. **LSB-first concatenation reversal**: the translator builds an *aliased* bus as `{bit0, bit1, …}` (LSB first), but Verilog `{…}` is MSB-first, so the bit order is reversed. Symptom: address/counter bits permuted, not z. Examples: `THO`/`TVO` (fixed to `{AT_adr[2], AT_adr[1], …}`).
4. **Uninitialized RAM → x**: a behavioral RAM written with z data when the write-enable isn't also gated on the data-bus-enable. Fix: gate on the direction AND the bus enable (e.g. CRAM write `if (n_DB_CB==1'b0 && n_DBE==1'b0)`, matching the OAM write).

## Fixing pattern

When a signal is z: dump the circuit's pins + wires, run the union-find, find the net that *should* drive the floating output, then add a direct `assign <out> = <net>` (or the small concatenation) in the Verilog. When a bus is permuted, reverse the concatenation so `{MSB, …, LSB}` matches the schematic's bit-to-net wiring. Rebuild with `iverilog -g2012 -D ICARUS -D RP2C02` (and `-D RP2C07`) and re-check the VCD for z/x.

## Concrete PPU fixes (committed)

- `tilecnt.v`: `THO`/`TVO` (and their `assign`s) + `n_FVO`/`n_TVO`/`n_THO`; `AT_adr[7:9]=0`, `NT_adr[2:4]=AT_adr[0:2]`, `NT_adr[7:13]=AT_adr[3]/[10:13]`.
- `mux.v` PictureMUX: BG/OBJ color-mux pipeline, `MUX_Control` inputs, OCOL/EXT select swap.
- `par.v`: `PAddr_out[12] = w11` (ParControl PAD12).
- `cram.v`: palette write gated on `n_DBE`.
- `fsm.v`: `H0_D = ~h0_latch1_nq`.
