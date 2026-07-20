# Production handoff

## Release artifacts

- PCB: `dev-real-production.kicad_pcb`
- KiCad project rules: `dev-real-production.kicad_pro`
- Fabrication archive: `dev-real-production-fab.zip`
- DRC report: `dev-real-production-drc.rpt`
- Detailed pair audit: `dev-real-production-length-audit.txt`

The final KiCad 9 DRC result is **0 unconnected pads** and **0 errors** at error
severity.

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

**Fixed:** the `courtyards_overlap` between C5 and COUT2 noted above has been
resolved. COUT2 could not be moved (the channel between C5 and L1's new body
is too narrow for COUT2's footprint in any orientation, and COUT2's own
routing was already verified/stable), so C5 (100 nF, `C_0603_1608Metric`) was
relocated instead, from (68.5, 59.5) mm to (67.7, 57.75) mm — clear of both
COUT2's and C2's courtyards, with a real `kicad-cli pcb drc` clean pass
(0 violations / 0 unconnected). Its two local stub nets were rerouted to the
new pad positions on the Top Layer: `/Power/RIPPLE_INJ` (pad at
(66.925, 57.75)) reconnects with a single straight segment to its existing
run at (65.674999, 57.449999); `/Power/FB` (pad at (68.475, 57.75)) is
routed around an `/Power/EXTVDD` track that crosses the direct path, via
waypoints (68.6, 58.0) → (70.4, 58.0) → (70.8, 58.4) → (71.0, 58.4) →
(71.6, 59.0) → (71.0, 59.6) → (69.2, 59.6), rejoining the existing FB run at
(69.3, 59.524999). Six duplicate/redundant `/Power/FB` stub segments left
over from earlier routing passes (all at the same old coordinates) were
deduped down to the single new route in the process.

## High-current trace widths

The `/Power/SW` and `+3V3` segments added/touched by the L1 reroute were
widened from their as-routed 0.1–0.2 mm to 0.2–0.4 mm (clearance-checked
per segment, matching this board's existing convention for local
high-current power hops — see the `+3V3`/`/Power/SW` segments elsewhere on
the board already at 0.4 mm). A board-wide sweep of the other plausible
current-carrying nets (`/Power/VIN`, `/Power/SVIN`, `/Power/PVDD`,
`/Power/EXTVDD`, the USB VBUS nets) found them already consistent with this
convention (0.18–0.35 mm), with one exception: `/IWRL6432BD/USB_VBUS` had
~7.8 mm routed at only 0.15 mm across a few inner-layer segments. Five of
those segments have been widened to 0.2–0.3 mm. Two short segments — a
4.64 mm run on Power Layer 2 near (52.7–57.2, 27.1–28.0) and a 3.19 mm run
on Ground Layer 1 near (57.5–59.5, 28.3–30.7) — remain at 0.15 mm because a
neighboring `/Power/1V8` via sits close enough that widening them trips
real DRC clearance; widening those two needs either moving that via or
re-routing around it, left for interactive follow-up.

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

The Ethernet TX pair's 3.723 mm mismatch is not called out above with a
verdict: at typical FR-4 propagation velocity that's roughly 25 ps of skew,
under common 1000BASE-T intra-pair skew budgets (~50 ps) but close enough to
the boundary to flag for review rather than treat as automatically fine.

## Fabricator instructions

- Four copper layers, nominal finished thickness 1.6 mm.
- The board contains F.Cu-to-In1.Cu laser microvias and In1.Cu-to-In2.Cu buried vias. Use an HDI process capable of the separated drill sets supplied in the archive.
- Confirm the smallest finished drill, annular ring, stacked-via policy, via filling/capping, soldermask expansion, and copper-to-edge limits against the KiCad board before accepting the order.
- Differential impedance must be field-solved by the selected fabricator using its actual laminate and plating data. Do not treat the nominal KiCad geometry as an impedance coupon.
- The project says only `Lead-Free` for copper finish; select the required finish explicitly on the fabrication purchase order.

