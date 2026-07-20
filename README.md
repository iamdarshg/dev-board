# dev-real — what's left to do

Self-contained project: board + schematics + libraries. Unzip anywhere and open
`dev-real.kicad_pro` in KiCad 9.

Board is at **0 DRC errors and 0 unconnected items — fully routed.**

---

## 1. Routing complete (historical)

All 14 unconnected items from the previous pass (was 499 originally) have
been resolved, including the last 6 hold-outs around the U13 radar BGA
corner (GND ball M12, an In2/GND stub near (56.9, 16.0), C34 pad 1, U21 pad
26, a split GND plane island on In1, and an NRESET In1↔F gap) — these needed
hand-routing (via-in-pad style connections plus small shoves of a couple of
neighboring nets — FT_RXD, GPIO_4 — that were routed directly under the
affected pads) since the escape-track congestion in that corner defeats the
interactive shove router's default settings. `Inspect → Design Rules
Checker → Run DRC` now reports **0 errors, 0 unconnected** with zones
refilled.

If future edits reintroduce unconnected items, the same workflow applies:

1. Open `dev-real.kicad_pro` → PCB Editor.
2. `Route → Interactive Router Settings…` → set **Mode: Shove** (Walk Around
   struggles near the U13 BGA corner, where several nets route directly
   under neighboring balls).
3. `Inspect → Design Rules Checker → Run DRC`, open the **Unconnected
   Items** tab, double-click an item to zoom to it.
4. Hover the pad → press `X` → route → double-click to finish. Press `V`
   mid-route to drop a via and switch layers.
5. Press `B` to refill all zones before the final DRC check — stale zone
   fill after a track/via edit can itself show up as a false clearance
   violation until refilled.

## 2. Fix SCOPE_BIAS in the schematic

`/AD9609/SCOPE_BIAS` has **no DC reference** anywhere in the schematic — the net
floats on the real board. Open `ad9609bcpzrl7_80_breakout.kicad_sch` and define
the intended bias source, then `Tools → Update PCB from Schematic` (F8).

## 3. Sanity-check before you save

After any edit: press `B` to refill zones, then `Inspect → Design Rules
Checker → Run DRC` and confirm it still reads **0 errors, 0 unconnected**
before `Ctrl+S`.

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
