# dev-real — what's left to do

Self-contained project: board + schematics + libraries. Unzip anywhere and open
`dev-real.kicad_pro` in KiCad 9.

Board is at **0 DRC errors** with **14 unconnected items** left (was 499).
Everything below is the complete remaining work. Est. 15–30 min.

---

## 1. Finish the 14 connections (interactive routing)

**Setup**

1. Open `dev-real.kicad_pro` → PCB Editor.
2. `Route → Interactive Router Settings…` → set **Mode: Shove** (it defaults to
   Walk Around, which is why these 14 can't be auto-routed — they need
   existing tracks pushed aside).

**Finding each one**

`Inspect → Design Rules Checker → Run DRC`, open the **Unconnected Items** tab.
**Double-click any item to zoom straight to it.** Work down the list.

**Routing each one**

Hover the pad → press `X` → move to the target → double-click to finish.
Press `V` mid-route to drop a via and switch layers. `Esc` cancels cleanly.

### The list

| # | Net | From | To | Note |
|---|-----|------|----|------|
| 1 | GND | U24 pad 25 @ (45.00, 78.00) | via @ (49.47, 79.31) | pad boxed in by unnetted pads |
| 2 | GND | U13 ball M12 @ (55.25, 21.25) | In2 track @ (54.80, 19.45) | BGA ball → plane |
| 3 | GND | U13 ball K9 @ (56.75, 22.25) | In2 track @ (57.45, 21.90) | BGA ball → plane |
| 4 | GND | C34 pad 1 @ (59.97, 29.91) | B track @ (59.62, 28.75) | short hop, needs a via |
| 5 | GND | U21 pad 26 @ (69.64, 24.04) | B track @ (68.98, 22.50) | ADC ground |
| 6 | GND | In1 plane fragment | In1 plane | plane split — bridge the two In1 islands, or add a GND via where they nearly touch |
| 7 | +3V3 | F track @ (52.82, 18.34) | B track @ (54.66, 21.30) | needs a via |
| 8 | +3V3 | U13 ball G12 @ (55.25, 23.75) | via @ (55.86, 21.31) | BGA ball |
| 9 | +3V3 | C36 pad 2 @ (56.03, 29.91) | C45 pad 2 @ (56.01, 31.61) | straight 1.7 mm hop |
| 10 | /Power/PG | U3 pad 32 @ (60.73, 62.26) | F track @ (67.71, 55.78) | regulator power-good |
| 11 | JTAG_TMS | U13 ball E12 @ (55.25, 24.75) | In2 track @ (62.21, 22.91) | BGA ball |
| 12 | JTAG_TDI | U13 ball G11 @ (55.75, 23.75) | J11 pad 8 @ (89.27, 21.81) | long haul |
| 13 | GPIO_3 | U13 ball J11 @ (55.75, 22.75) | U4 pad 19 @ (54.75, 89.40) | long haul, U13 → U4 |
| 14 | RDIF_D2 | U13 ball L12 @ (55.25, 21.75) | U4 pad 25 @ (57.80, 91.25) | long haul, U13 → U4 |

Items 2, 3, 8, 11, 12, 13, 14 are all **U13 (radar BGA) balls** — zoom in once
around (55, 22) and do them together. The blocker is the same everywhere:
inner-layer escape tracks pass directly under those balls, so a via-in-pad
misses clearance by ~0.02 mm. Shove mode moves them out of the way.

## 2. Fix SCOPE_BIAS in the schematic

`/AD9609/SCOPE_BIAS` has **no DC reference** anywhere in the schematic — the net
floats on the real board. Open `ad9609bcpzrl7_80_breakout.kicad_sch` and define
the intended bias source, then `Tools → Update PCB from Schematic` (F8).

## 3. Finish up

1. Press `B` to refill all zones.
2. `Inspect → Design Rules Checker → Run DRC` → confirm **0 errors,
   0 unconnected**.
3. Save (`Ctrl+S`).

---

### Design rules already in effect (don't change)

Signal 0.127 mm / vias 0.254 mm with 0.127 mm drill / clearance 0.08 mm /
min through-drill 0.127 mm. Power rails run 0.2–0.5 mm.
The 0.2/0.1 vias on this board are **microvias** — place a new *through*-via
that small and DRC will (correctly) reject it.

### What's in here

- `dev-real.kicad_pcb` — the board
- `dev-real.kicad_pro` — project + design rules (**keep beside the board**;
  without it KiCad invents default rules and DRC becomes meaningless)
- `dev-real.kicad_sch` + 8 hierarchical sheets — full schematic
- `fp-lib-table`, `sym-lib-table` + the custom symbol/footprint libraries they
  reference (AD9609, IWRL6432, MM8108, THS8136, XCC1352, ICM-42670, ENS161,
  MMC5633, plus the project's `.pretty` footprint dirs)

Verified standalone: netlist exports all 318 components across the 9-sheet
hierarchy, and DRC output is identical to the working copy.

Full history, method, DRC report, renders and the routing scripts stay in
`D:\dev-real-updated\dev-real-updated\dev-real-routing-finish\`.
