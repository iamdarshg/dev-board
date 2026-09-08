# Production handoff

## Release artifacts

- PCB: `dev-real-production.kicad_pcb`
- KiCad project rules: `dev-real-production.kicad_pro`
- Fabrication archive: `dev-real-production-fab.zip`
- DRC report: `dev-real-production-drc.rpt`
- Detailed pair audit: `dev-real-production-length-audit.txt`

The current KiCad 9.0.7 DRC result is **107 violations: 4 errors, 103
warnings, and 0 unconnected items**. The four error records represent two
physical via-to-zone conflicts (each reported once for copper clearance and
once for hole clearance); see the current-routing section for coordinates and
the remaining manual work.

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

## Current routing status (2026-09-08; cleaned manual-routing candidate)

The checked-in board contains the latest manual KiCad routing plus a
conservative basic-DRC cleanup. The cleanup removed **59 zero-pad signal track
segments**, **20 zero-pad signal vias**, **5 byte-identical duplicate vias**,
and **12 exact F.SilkS primitives** named by DRC. The undersized `TR1` and `U6`
reference text was resized to 0.8 mm and moved clear. This eliminated every
`silk_over_copper`, `silk_overlap`, and `text_height` violation. A deliberately
broader cleanup was tested and rejected because it created a genuine
unconnected item; sensitive and ambiguous one-pad routes therefore remain for
visual review instead of being deleted automatically.

KiCad 9.0.7 DRC on the resulting board reports **107 violations: 4 errors and
103 warnings, with 0 unconnected items and 0 footprint errors**. The repeatable
pre-cleanup control result was 176 violations (4 errors and 172 warnings), so
this pass removed 69 warnings without increasing the error or unconnected
counts. This remains a review candidate, not a fabrication release. The
previously released board at commit `3a95473` remains the clean fabrication
baseline, and the fabrication outputs listed above have not been regenerated.

### Manual actions still required

1. **Fix the two real via-to-zone conflicts first.** Each creates both a
   clearance and a hole-clearance error:
   - `/MM8108/SDIO_D0_SPI_MISO` through via at **(24.3991, 83.3043) mm** versus
     the Ground Layer 1 GND zone: copper clearance is 0.0061 mm versus 0.1000
     mm required; hole clearance is 0.0561 mm versus 0.1200 mm required.
   - GND through via at **(22.5030, 80.6170) mm** versus the Power Layer 2
     `PWR_3V3_PLANE`: both reported actual clearances are 0.0000 mm. Move the
     via or reshape the zone while preserving the via's intended connection.
2. **Inspect the 39 remaining dangling-track markers in KiCad before deleting
   anything.** Highest-risk examples are `/IWRL6432BD/XTALM`,
   `/IWRL6432BD/RF_TX1`, `/IWRL6432BD/RF_TX2`, `/IWRL6432BD/GPIO_2`,
   `/STM32H7/VREFP`, `/STM32H7/ETH_RXD0`, `/STM32H7/ETH_RXD1`,
   `/MM8108/ANT`, `/AD9609/DCO`, `/AD9609/D9`, and the IWRL USB VBUS/ID
   routes. These include one-pad stubs, sensitive launches, and branches that
   cannot be classified as unused from connectivity alone. The remaining
   markers also include GND/+3V3/power stubs that should be judged against the
   intended plane connection.
3. **Inspect the 20 remaining dangling-via markers.** They include one-pad
   supply/decoupling transitions and signal transitions on `/Power/VDD`,
   `/Power/SW`, `/Power/FB`, `/Power/VIN`, `/Power/AGND`, `/Power/EN`,
   `/IWRL6432BD/RADAR_SRAM_1V2`, `/IWRL6432BD/FT_TXD`,
   `/IWRL6432BD/QSPI_D2`, `/AD9609/3V3_AFE`, `/THX8136/AR`, `/THX8136/AG`,
   `/THX8136/AB`, and `/STM32H7/PM3`. Delete only after confirming the
   transition is not intentional.
4. **Resolve or obtain fabricator approval for 20 co-located-hole and 11
   hole-to-hole warnings.** Most are intentional-looking stacked microvia /
   through-via structures on GND and power nets, but they need confirmation
   against the selected HDI process. The same-position `/AD9609/DCO` via pair
   at approximately (90.0, 54.25) mm also needs visual inspection.
5. **Open and reshape the five copper-sliver DRC markers** (two Top Layer, two
   Bottom Layer, one Power Layer 2). The text report identifies their layers
   but does not provide coordinates, so use the interactive DRC marker list.
6. **Review the eight library-footprint mismatch warnings.** `J7`, `U3`,
   `RANT1`, and `CANT1` are expected after this local silkscreen cleanup;
   `NT1`, `U21`, `U14`, and `U15` were pre-existing. These are not footprint
   errors, but verify their land patterns before fabrication.

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

