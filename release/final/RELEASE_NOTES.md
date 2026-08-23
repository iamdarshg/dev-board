# dev-board Fabrication Release Notes

Revision: main @ (see SHA256SUMS.csv for exact source hashes)
Generated: 2026-08-24 (Asia/Calcutta)
Generator: KiCad 9.0.7 CLI

## Contents

- dev-real-gerbers/ — Gerbers (F.Cu, In1, In2, B.Cu, masks, pastes, silkscreens, Edge.Cuts) + Excellon drills (PTH + 3 blind/buried lamination pairs) + drill maps (PDF) for dev-real.kicad_pcb
- dev-real-production-gerbers/ — Same set for dev-real-production.kicad_pcb
- dev-real-bom.csv / dev-real-production-bom.csv — Bill of Materials
- dev-real-positions.csv / dev-real-production-positions.csv — Pick-and-place positions
- dev-real-schematic.pdf — Full hierarchical schematic (9 sheets)
- erc-report.rpt — Full-hierarchy ERC report
- drc-dev-real.rpt / drc-dev-real-production.rpt — DRC reports with schematic parity

## Verification results

| Check | dev-real | dev-real-production |
|---|---|---|
| DRC errors | 0 | 0 |
| DRC unconnected items | 0 | 0 |
| ERC errors | 88 | (same schematic) |
| ERC warnings | 840 | (same schematic) |
| Board size | 105 x 105 mm | 105 x 105 mm |
| Layers | 4 (F.Cu/In1-GND/In2-PWR/B.Cu), 1.6 mm | same |

ERC error classification: all remaining ERC errors are deliberate design conditions or
library-symbol consistency warnings — no actual electrical faults found:
- power_pin_not_driven (32): power-input pins driven through hierarchical sheet boundaries;
  KiCad''s ERC cannot see the driving source across sheets.
- label_dangling (25): root-sheet hierarchical labels for optional/debug interfaces left unwired by design.
- pin_not_connected (7): deliberately unused RESV/NC pins on IC1, J6 pin 7, U5 DIO_7.
- hier_label_mismatch (9): hierarchical labels present in child sheets but intentionally not broken out to parent.
- pin_to_pin (1): VDDA_10RF output-output tie on U13 (IWRL6432) per TI reference design.

## Known accepted issue: /AD9609/SCOPE_BIAS

The net /AD9609/SCOPE_BIAS connects only R36 pin 2 and C73 pin 2 (C73''s other side is GND).
R36''s other end ties to the MCP6D11 differential driver''s IN- summing node. The net has no DC source —
it is electrically floating on the assembled board. Determining the intended bias voltage (GND vs VCM vs VREF,
given the PGA113 output operating point and VOCM setting) would require inventing analogue design intent that
is not documented anywhere in the project. Per explicit user decision this issue is accepted as-is and left
unresolved. It affects only the AD9609 scope-driver subcircuit (U22 MCP6D11 input bias reference).

No other nets are unresolved. Both PCB variants have identical SCOPE_BIAS connectivity (2 pads each).

## Manufacturing notes

- HDI build: microvias and blind/buried via pairs (front-In1, front-In2, In1-In2 laminations) require
  sequential-lamination capability; not a standard low-cost through-via process.
- Design rules: min track 0.127 mm, min via 0.254 mm/0.127 drill, microvia 0.2 mm/0.1 drill, clearance 0.08 mm.
- Drill files use absolute origin, millimetre units, Excellon format. Layer-specific drill files are provided
  for the blind/buried pairs plus a combined PTH file.
- Board outline: closed rectangle, Edge.Cuts layer, 105 x 105 mm.
- Zones refilled at time of export; filled polygons present on both boards.

