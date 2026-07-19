# Production handoff

## Release artifacts

- PCB: `dev-real-production.kicad_pcb`
- KiCad project rules: `dev-real-production.kicad_pro`
- Fabrication archive: `dev-real-production-fab.zip`
- DRC report: `dev-real-production-drc.rpt`
- Detailed pair audit: `dev-real-production-length-audit.txt`

The final KiCad 9 DRC result is **0 unconnected pads** and **1 remaining error** (see
"L1 inductor" below) at error severity.

**Fab outputs are stale as of the L1 change below.** `dev-real-production-fab/`
(gerbers, drill files, `-positions.csv`, `-netlist.xml`, `-drc.rpt`, `.zip`) and
`dev-real-production-length-audit.txt` were generated before L1 was replaced and
rerouted, and were intentionally **not** regenerated here (kicad-cli's default
export job doesn't reproduce this project's curated gerber/drill file set —
e.g. the PTH/NPTH split — without its saved plot-job settings). Before sending
to a fabricator: open `dev-real-production.kicad_pro` in KiCad 9 and re-run
**File → Fabrication Outputs** (or the project's existing plot job) to
regenerate the whole `dev-real-production-fab/` package and `.zip` from the
current board.

## L1 inductor (1 µH buck output, `/Power/SW` → `+3V3`)

L1 was a placeholder (`L_TODO_SELECT_1uH_15Aplus`, physically a 0603 footprint)
and is now **KYOCERA AVX LMLP07C7M1R0DTAS** (1 µH ±20 %, DCR 6.1/6.5 mΩ typ/max,
IDC 15 A typ, Isat 20 A typ) on footprint `L_Kyocera_LMLP07C7D_7.3x6.6mm`
(land pattern built from the LMLP07 series dimensional table in
`datasheets.kyocera-avx.com/LMLPD.pdf` — verify against that drawing before
committing to fab). COUT2 (the buck output cap, itself still a
`C_TODO_SELECT_100uF` placeholder) was nudged ~1.85 mm to make room, and the
~12 local nets crossing the new footprint's footprint were rerouted and
verified against real `kicad-cli pcb drc`.

**Residual, not fixed:** `courtyards_overlap` between C5 and COUT2. COUT2 is
now sandwiched in a channel only ~1.45 mm tall between C5 and L1's new body —
too narrow for COUT2's ~1.5–3.0 mm footprint in either orientation. This is a
manufacturing/assembly-keepout warning, not a short or connectivity fault.
Resolving it cleanly needs either relocating COUT2 further away (with a full
reroute of its two pads) or finalizing COUT2's own real part (it's still a
placeholder) with a smaller footprint — left for interactive placement in
KiCad rather than forced here.

## ADC/DAC test breakout

The exposed 1.0 mm test pads are grouped on the bottom edge at 2.0 mm pitch:

| Reference | Net | Position (mm) |
|---|---|---:|
| TP8 | ADC VIN+ | 79, 11 |
| TP9 | ADC VIN- | 81, 11 |
| TP1 | DAC red / AR | 83, 11 |
| TP2 | DAC green / AG | 85, 11 |
| TP3 | DAC blue / AB | 87, 11 |

ADC test-branch mismatch is 0.509 mm. These are measurement branches; keep probes and flying leads short.

## Pair-length audit

Key routed mismatches:

| Pair | End-to-end mismatch |
|---|---:|
| ADC analog input | 0.072 mm |
| ADC sample clock | 0.572 mm |
| Ethernet RX | 0.260 mm |
| Ethernet TX | 3.723 mm |
| STM32 USB FS | 5.049 mm |
| MM8108 USB | 7.819 mm |
| RP2350 USB | 9.702 mm |
| FTDI USB | 16.235 mm |

The dense USB routes were not given forced serpentine sections because no collision-free tuning corridor was available. The STM32, RP2350, and FTDI paths are identified as full-speed USB in this design and are retained as routed. Confirm the MM8108 interface speed before fabrication: if it operates at USB high speed, its 7.819 mm residual mismatch should be rerouted against the module's timing requirement.

## Fabricator instructions

- Four copper layers, nominal finished thickness 1.6 mm.
- The board contains F.Cu-to-In1.Cu laser microvias and In1.Cu-to-In2.Cu buried vias. Use an HDI process capable of the separated drill sets supplied in the archive.
- Confirm the smallest finished drill, annular ring, stacked-via policy, via filling/capping, soldermask expansion, and copper-to-edge limits against the KiCad board before accepting the order.
- Differential impedance must be field-solved by the selected fabricator using its actual laminate and plating data. Do not treat the nominal KiCad geometry as an impedance coupon.
- The project says only `Lead-Free` for copper finish; select the required finish explicitly on the fabrication purchase order.

