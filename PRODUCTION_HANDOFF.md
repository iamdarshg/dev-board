# Production handoff

## Release artifacts

- PCB: `dev-real-production.kicad_pcb`
- KiCad project rules: `dev-real-production.kicad_pro`
- Fabrication archive: `dev-real-production-fab.zip`
- DRC report: `dev-real-production-drc.rpt`
- Detailed pair audit: `dev-real-production-length-audit.txt`

The current KiCad 9.0.7 DRC result is **26 warnings, 0 errors, 0 unconnected
items, and 0 footprint errors** after the 2026-09-09 dangling-copper cleanup
and full eight-zone refill. The remaining warnings are explicitly classified
in the current-routing section below.

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

## Current routing status (2026-09-09; cleaned manual-routing candidate)

The checked-in board contains the latest manual KiCad routing plus a
conservative basic-DRC cleanup. The cleanup removed **59 zero-pad signal track
segments**, **20 zero-pad signal vias**, **5 byte-identical duplicate vias**,
and **12 exact F.SilkS primitives** named by DRC. The undersized `TR1` and `U6`
reference text was resized to 0.8 mm and moved clear. This eliminated every
`silk_over_copper`, `silk_overlap`, and `text_height` violation. A deliberately
broader cleanup was tested and rejected because it created a genuine
unconnected item; sensitive and ambiguous one-pad routes therefore remain for
visual review instead of being deleted automatically.

The 2026-09-09 follow-up then iteratively removed every newly exposed dead-end
chain that passed a fresh connectivity check: **90 additional dangling track
blocks and 27 additional dangling-via blocks** were removed. Nine more
redundant members of co-located via pairs were removed directly; combined with
the dangling-via removal, this cleared **18 of the 20** co-location markers.
All eight zones were refilled. The refill removed the two former physical
via-to-zone conflicts (four error records) without creating shorts or
unconnected items.

KiCad 9.0.7 DRC on the resulting board reports **26 warnings, 0 errors, 0
unconnected items, and 0 footprint errors**. The remaining warnings are:

- **One `track_dangling` marker:** `/STM32H7/JTDI` at approximately
  (21.5420, 40.3517) mm. A test deletion created a real missing connection, so
  that deletion was reverted and this marker is intentionally accepted.
- **Two `holes_co_located` markers:** the `/Power/SW` microvia-to-buried-via
  transitions at (71.5000, 66.0000) mm and (71.5000, 66.2000) mm. Removing a
  member created a missing connection. A tested stagger-and-bridge alternative
  cleared co-location but created four `/Power/SW` to `+3V3` shorts after zone
  refill, so it too was reverted. These are required stacked transitions and
  need fabricator approval rather than deletion.
- **Five `copper_sliver` markers:** two Top Layer, two Bottom Layer, and one
  Power Layer 2. KiCad's machine-readable report supplies no coordinates or
  attached objects for these markers. A complete eight-zone refill was clean
  electrically but did not remove the warnings, so they remain accepted for
  later interactive marker inspection rather than hiding them by lowering DRC
  severity.
- **Ten `hole_to_hole` warnings:** remaining HDI drill-spacing/stack geometry
  that must be accepted by the selected fabricator or manually restacked.
- **Eight `lib_footprint_mismatch` warnings:** `J7`, `U3`, `RANT1`, and
  `CANT1` reflect intentional local silkscreen cleanup; `NT1`, `U21`, `U14`,
  and `U15` were pre-existing. There are no footprint errors.

This remains a review candidate, not a fabrication release. The previously
released board at commit `3a95473` remains the clean fabrication baseline, and
the fabrication outputs listed above have not been regenerated.

The following are the remaining measured end-to-end pair mismatches on the
checked-in board. They are documented for interactive KiCad follow-up; they
must not be treated as solved merely because the board passes ordinary DRC:

| Interface | Measured mismatch | Status |
|---|---:|---|
| IWRL USB DM/DP to J10 | 0.097 mm | manually tuned; verify after DRC fix |
| RP2350 USB D-/D+ to J4 | 0.967 mm | manually tuned; verify after DRC fix |
| STM32 USB FS D-/D+ to J20 | 0.414 mm | manually tuned; verify after DRC fixes |
| AD9609 sample clock CLK-/CLK+ | 0.304 mm | manually tuned; verify after DRC fixes |
| Ethernet RX RX-/RX+ to J21 | 0.021 mm | manually tuned; verify after DRC fix |
| Ethernet TX TX-/TX+ | 0.000 mm | corrected and DRC-verified |
| MM8108 USB D-/D+ to J17 | 0.675 mm | manually tuned; verify after DRC fix |
| AD9609 ADC input | 0.581 mm | open; analog path review |

The IWRL connector is J10 (`USB-C USB2 DEBUG`); the RP2350 connector is J4
(`USB-C Receptacle USB2`). The STM32 USB connector is J20, and the MM8108
USB connector is J17. The current edited-board values above supersede the
older clean-board values in the historical audit below. The basic cleanup was
limited to zero-pad islands, exact duplicates, and silkscreen; it did not
intentionally tune these endpoint paths. They are not release values until DRC
is clean again.

The first-order field-based impedance estimate used the checked-in four-layer
stackup (1.6 mm nominal, 175 um prepreg to the adjacent reference plane,
FR-4 dielectric approximately 4.4) and the actual routed widths. It estimates
roughly 65--70 ohm single-ended and 89--96 ohm differential for the common
0.127--0.300 mm geometries. This is an engineering estimate, not a fab
acceptance measurement: the fabricator must field-solve the final stackup,
copper thickness, soldermask, and trace spacing and provide a coupon or
controlled-impedance confirmation. The short CC1352 RF balanced launches are
length-matched, but their 50 ohm launch impedance has not been certified by a
3-D field solver.

## Historical clean-board pair-length audit

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

The historical Ethernet TX value in the table above predates the committed
correction. Use the 0.000 mm value in the current-routing section for the
checked-in production board.

## Fabricator instructions

- Four copper layers, nominal finished thickness 1.6 mm.
- The board contains F.Cu-to-In1.Cu laser microvias and In1.Cu-to-In2.Cu buried vias. Use an HDI process capable of the separated drill sets supplied in the archive.
- Confirm the smallest finished drill, annular ring, stacked-via policy, via filling/capping, soldermask expansion, and copper-to-edge limits against the KiCad board before accepting the order.
- Differential impedance must be field-solved by the selected fabricator using its actual laminate and plating data. Do not treat the nominal KiCad geometry as an impedance coupon.
- The project says only `Lead-Free` for copper finish; select the required finish explicitly on the fabrication purchase order.

