# BLHeli_S ESC Rework — Checklist & Notes

Scope: rework the inherited AT32F421 / NSG2065Q ESC into a **BLHeli_S** design using
EFM8BB51 + AGM310MAP complementary FETs. Whoop-class AIO, integrated ELRS, HD-video
connector. Target ~2S (leaning), possibly up to 4S.

**Status: library import done. No schematic/PCB work performed (by request).**

---

## Parts imported (LCSC → project-local `lib`)

| Symbol (in `libs/lib.kicad_sym`) | Footprint (`libs/lib.pretty/`) | 3D | LCSC | Role |
|---|---|---|---|---|
| `AGM310MAP` | `POWERDI3333-8_L3.3-W3.3-P0.65-LS3.4-BL-EP_UXC` | wrl+step | C5184858 | Complementary P+N half-bridge (1 per phase ×3) |
| `EFM8BB51F16G-C-QFN20R` | `QFN-20_L3.0-W3.0-P0.50-TL-EP1.7_SILICON_C8051F85X` | wrl+step | C6547511 | BLHeli_S MCU (8051, 50 MHz, 16 KB) |
| `BY25Q64ESCIG` | `DFN-8_L3.0-W2.0-P0.50-BL-EP-3` | wrl+step | C50176394 | 64 Mbit SPI NOR flash (blackbox) |

- 3D-model paths normalized to `${KIPRJMOD}/libs/lib.3dshapes/...` (was absolute on import).
- Symbol count `libs/lib.kicad_sym`: 99 → 106. Lib-tables unchanged (all under `lib` nickname).
- EFM8 imported is the **G** (commercial, −40…+85 °C) grade per request. I-grade
  `EFM8BB51F16I-C-QFN20R` (C3232998) is the wider-temp pin-identical alternate if G runs short.

## AGM310MAP — confirmed topology (AGMSEMI datasheet specs)

- **Complementary pair: 1× P-channel (high-side) + 1× N-channel (low-side)**, common drain.
- 30 V Vds. N-ch: 20 A, Rds(on) 11 mΩ@10V, Qg 16 nC. P-ch: 18 A, Rds(on) 19 mΩ@10V, Qg 45 nC.
- Vgs(th): N 1.6 V, P 1.5 V.
- Pinout: FET1 = G2/D8/S1, FET2 = G4/D5/S3. Tie the two drains (D5,D8) = phase output.
  P-ch source → VBAT, N-ch source → GND.
- Massively overspec for 2S (used at ~25% Vds); fine, just oversized. Low Rds = cool running.

## Gate drive — research result (the key open question)

Verified: small BLHeli_S whoop ESCs (e.g. **tinyPEPPER**, open hardware) **drive the FET
gates directly from the EFM8 — no gate-driver IC.** That is correct **for 1S only**:

- At 1S (≤4.2 V) the 3.3 V GPIO can turn the high-side P-ch both ON (Vgs≈−4 V) and OFF
  (gate pulled to ~pack voltage is reachable). MCU is powered straight off the 1S pack.
- **At 2S (8.4 V) direct drive breaks down:** the P-ch source sits at 8.4 V, and a 3.3 V GPIO
  cannot pull the gate up to 8.4 V to switch the high-side **off**. tinyPEPPER author:
  *"For 2S one needs FET drivers and a proper voltage regulator for the CPUs."*

### Implication for this design (target 2S, maybe 4S)
- [ ] **EFM8 needs a 3.3 V regulator** — it cannot run off 2S/4S directly (abs max ~4.2 V).
- [ ] **High-side P-ch needs drive help** above 1S. Options, lightest → heaviest:
  - Per-phase discrete level-shift (small NPN/N-FET pulling P-gate to VBAT, GPIO controls it) — keeps the "no driver IC" spirit, ~2 parts/phase.
  - A proper 3-phase gate driver IC (what the inherited NSG2065Q did) — most robust, biggest BOM.
- [ ] Low-side N-ch can still be GPIO-driven through a gate resistor at 2S (source at GND, 3.3 V > Vgs(th) 1.6 V — verify enhancement is adequate; consider a buffer if marginal).
- [ ] Pick gate resistors from Qg + desired edge rate (tinyPEPPER ref: ~100 Ω for ~50 nC / 500 ns).
- [ ] **If the design is locked to 1S**, direct drive with zero driver parts is valid — then most of the above collapses. Decide voltage target first; it gates everything.

## Other decisions captured (from discussion — not yet implemented)

- [ ] **Current sense:** drop the inherited 4× INA186 per-channel scheme. BLHeli_S commutates
      sensorless (BEMF), so per-phase sensing isn't needed. If telemetry wanted, use **one**
      total-current shunt on the pack return → single amp → FC ADC.
- [ ] **Blackbox:** replace the TF-021B microSD slot with the **BY25Q64 SPI flash** (8 MB).
      Wire to a free RP2350 SPI bus on the FC. Remove microSD slot + associated parts.
- [ ] **HD video connector:** add a standard HD-VTX connector (define pinout/standard — DJI/Walksnail-style 6-pin). Not yet specced.
- [ ] **Firmware:** EFM8BB51 is a BLHeli_S/Bluejay target (BB51). Confirm chosen target supports BB51 (classic BLHeli_S predates BB51; Bluejay fork adds it).

## Per-spin verification checklist

- [ ] Confirm AGM310MAP channel assignment in schematic matches P=high-side / N=low-side.
- [ ] EFM8 powered from regulated 3.3 V, decoupling per datasheet, C2 debug header exposed.
- [ ] Bulk/phase caps rated for chosen pack voltage (≥16 V if 2S, ≥25 V if up to 4S).
- [ ] Reverse-polarity protection on pack input.
- [ ] BLHeli_S signal pins (one per channel) routed to FC.

## Sources
- tinyPEPPER (fishpepper): https://fishpepper.de/projects/tinypepper/
- ckflight BLHeli_S ESC hardware: https://ckflight.github.io/ESC_BLDC_HARDWARE/
- BLHeli_S (bitdump): https://github.com/bitdump/BLHeli
- AGM310MAP: LCSC C5184858 (AGMSEMI), https://www.lcsc.com/product-detail/C5184858.html

---

# Update 2026-06-12 — sourcing + market reality check

## Market bar for a 2026 whoop AIO (from competitive research, see docs/MARKET-RESEARCH-2026-06.md)

- **12 A continuous / ~18 A burst is the class floor** (BetaFPV Air65 II Matrix 12A/18A, NBD BLV5 18A). 5 A designs are dead.
- Bluejay pre-flashed, 48 kHz (2S) / 96 kHz (1S), bidirectional DShot. Never stock BLHeli_S.
- 5V/3A BEC that holds full output down to ~2.8 V input is THE headline 1S spec (O4 Lite needs ≥10 W, >3.7 V at port → boost topology trade study needed for 1S).
- SH1.0 6-pin DJI O4-pinout VTX socket (VCC/GND/RX/TX/GND/SBUS — 6-pin, not 4).
- 25.5×25.5 mounting (BetaFPV's 26×26 is proprietary; everyone else is 25.5). Weight target ≤3.5 g (3.2 g is the competitive 1S digital bar, not 4.5 g).
- Reference competitor: BetaFPV Matrix 1S 3IN1 HD ($50, G473, 12A/18A, serial ELRS, 5V/3A, 3.2 g).

## ⚠ AGM310MAP thermal problem at the 12 A class floor

P-channel Rds(on) = 19 mΩ. At 12 A phase current with ~50% high-side conduction:
P ≈ 12² × 0.019 × 0.5 ≈ **1.4 W per phase high-side** (4.1 W total high-side at full load) in 3.3×3.3 mm packages on a ~25 mm board. The complementary P+N topology is comfortable in the **5–8 A class**, marginal at 10 A, and not credible at 12 A cont / 18 A burst.

Topology options, in increasing BOM weight:
1. **AGM310MAP as-is** → honest 6–8 A cont rating. Fine for a 65 mm 1S "Lite" tier, below the 2026 class floor for the main SKU.
2. **N+N FETs + 3-phase half-bridge driver per channel** (what 12 A+ commercial boards do; NBD "double-NMOS"). Driver candidates: FD6288Q-family clones (6288Q-MNS etc., proven in 4in1-mini lineage) — need ≥10 V gate supply, so 2S-only or a small boost.
3. Hybrid: low-side N direct GPIO, high-side N with bootstrap driver.

Decision still gates on pack voltage (1S vs 2S) — see checklist above.

## Sourcing status (LCSC, 2026-06-12)

| Part | LCSC | JLC asm stock | LCSC retail | Verdict |
|---|---|---|---|---|
| EFM8BB51F16G-C-QFN20R | C6547511 | **not in JLC lib** | 17,124 | LCSC plentiful; verify JLCPCB assembly availability or consign |
| EFM8BB51F16I (alt) | C3232998 | 119 | 0 | G-grade is the buy |
| AGM310MAP | C5184858 | 166 | 515 | **Production blocker** — 12/board → ~42 boards total. Second-source or change topology |
| BY25Q64ESCIG | C50176394 | 305 | **1** | Swap to W25Q64JV / ZD25Q64 class 2×3 mm 64 Mbit NOR |
| 2N7002 (level shift) | C8545 | **basic part** | 1.8M | OK |
| 1 mΩ 1206 shunt | C7467246 | — | 96k | OK (0.625 W @ 25 A = 63% of 1 W rating; 2010 WSLP for margin) |
| Motor plug 1.25 mm 3P | C293630 (Molex PicoBlade) | — | 29k | canonical whoop motor connector; C3029360 clone at 1/3 price |

Whole-family supply flag (affects FC sheets inherited from OpenFC-Lite-Mini): **NCV8187AMT180TAG (1.8 V gyro LDO, C893189) is effectively unobtainable (62 JLC / 19 LCSC)** — pick a replacement before any spin.

---

# Update 2026-06-12 (later) — topology locked: NO gate driver, FET research final

**Stan's decision: the Bluejay ESC has no gate-driver IC — EFM8 GPIO direct drive only, no board space.** This locks the power stage to complementary P+N per phase (high-side N is impossible without a driver/bootstrap). Consequences:

- **1S: clean direct drive**, zero extra parts (CrazyBee/tinyPEPPER pattern — confirmed by teardowns below).
- **2S: P-gate must reach VBAT (8.4 V) to turn OFF** → per-phase discrete level shift (2N7002 C8545 basic part + pull-up, ~2 parts/phase, 24 parts/board) is the only no-driver-IC option. If even that is too much board space, the design is 1S-only.
- The 12 A dual-N market tier (BetaFPV SiZ322DT boards) is **off the table** — dual-N needs high-side drive. Honest rating for this topology: **~6–8 A cont/phase**; that covers 65–75 mm 1S whoops (real currents ≤5 A/motor), not the 12 A spec-sheet war.

## FET selection — FINAL (2026-06-12): 2×2 mm package required (Stan)

AGM314MAP (3.3×3.3) is too big for the whoop board — it stays earmarked for the larger AM32 toothpick AIO. A full ≤2.2×2.2 sweep of LCSC found **exactly one power-class complementary N+P family at 2×2: Diodes DMCxxxxUFDB (U-DFN2020-6)**. Everything else complementary at SOT-563/SOT-363 is small-signal (≥350 mΩ). AGMSEMI has nothing below 3.3×3.3.

### Revision 4 — BetaFPV teardown observations (Stan) close the architecture question

- **2022 BetaFPV F4 1S 12A** (SiZ322DT dual-N): Stan IDs ~3-pin 0402-size parts + 3 Schottkys + 3 bootstrap caps per motor → the "undocumented high-side drive" is a **discrete bootstrap level shift per phase** (tiny transistor + bootstrap diode/cap). So dual-N without a driver IC costs ~3 extra parts per phase — 9/motor, 36/board. Not for us.
- **2025 Matrix 1S 5IN1 II (Air65 II, "12A/18A")** uses **AGMSEMI AGM210MAP — complementary P+N, GPIO direct drive. Same architecture we chose.** Datasheet verified (local): PDFN3.3×3.3 (NOT 2×2; note LCSC/BetaFPV listings say 25 V — datasheet says **20 V**), N 10 typ/14 max mΩ @4.5 V (14/20 @2.5 V), P 13 mΩ typ, Vgs(th) max 1.0 V, 25 A/100 A pulsed, RθJC 3.5. **C7431169, 10,623 stock** — 20× the stock of AGM310/314.
- Implications: (a) the flagship "12A/18A" 1S board is P+N direct drive — our topology is the current market architecture, not the legacy one; their 12 A claim on N10/P13 dies ≈ 1.7 W/phase — spec-sheet optimistic, consistent with our honest-rating stance. (b) AGM210MAP supersedes AGM310/314 as the AGMSEMI pick for the 3.3×3.3 tier (better dies, field-proven on Air65 II, real stock) — earmark for the mid-tier 2–4S Bluejay boards; 20 V means 2–3S ceiling, not 4S. (c) Why "25 V/20 V for a 1S part?" — voltage class is ~free in trench processes; it buys avalanche margin against phase-node inductive spikes and lets one part cover 1–2S+ product lines. XR8G02M (20 V) follows the same logic.
- Also noted from the same board (Stan): G473CEU6 (we use RP2354A), BB51F16G QFN-20 (= our EFM8BB51F16G), AT7456E analog OSD + RTC6705-class VTX (n/a — we're digital-only), **PUYA PY25Q128 16 MB blackbox flash** (we adopt 128 Mbit — see below), and a **2-Schottky diode-OR for 5 V/USB selection** instead of a power-mux IC (space/cost saver — candidate to replace TPS2116 on the whoop; costs ~0.3 V drop, Stan's circuit call).

**Blackbox flash decision: W25Q128JVPIQ (C190862, WSON-8 5×6, 16 MB, 6.4k stock + 110k in SOIC variant)** replaces the dead BY25Q64 (8 MB). 16 MB is the market bar; smallest stocked 3.3 V 128 Mbit package (2×3 DFN class tops out at 64 Mbit, all dead/1.8 V stock). Same capacity BetaFPV ships, smaller package than their SOP-8. Imported into `whoop` lib.

**RX MCU density option: ESP8685H4 (C4944062, QFN-28 4×4, 872 JLC stock)** — ESP32-C3 die with in-package 4 MB flash in a 4×4 (vs 5×5 QFN-32 ESP32-C3FH4); the "C3's ESP8285" Stan remembered. Runs the same ELRS ESP32-C3 targets; OpenRX already had its datasheet on file. New footprint + fewer GPIO — RX section relayout. Imported into `whoop` lib.

### Revision 3 (Stan's find): XR8G02M — primary for THIS board (2×2)

**XR8G02M (XNRUSEMI, C42457203, DFN2020-8L 2×2)** — found by Stan; missed by both parametric sweeps because LCSC files it under "Single FETs". Datasheet verified against the listing:
- N: 12 typ/15 max mΩ @4.5 V, 16/23 @2.5 V, Vgs(th) 0.5–1.5 V
- P: 17 typ/25 max @4.5 V, 24/30 @2.5 V
- **20 V** Vds (vs SiA527DJ's 12 V — proper spike margin at 1S), ±12 V Vgs, 8 A cont / 32(N)/28(P) A pulsed, avalanche-rated, PD 15/20 W
- Qg 15/19 nC @10 V (≈6–9 nC at 3.3 V) — slightly heavier gates than the alternatives; fine at 24–48 kHz PWM with direct GPIO drive, don't run 96 kHz
- Pinout: D1/G1/D2/G2 bottom row, S1 ×2 / S2 ×2 top, dual exposed pads; die 1 = N, die 2 = P — verify assignment in schematic
- Price $0.036–0.05 — cheapest candidate by 3–6×

Beats the SiA527DJ (N 29/P 41 max) and DMC1229UFDB (N 29/P 61 max) on every electrical axis. **Supply is the only weakness: LCSC live stock 80 pcs, XNRUSEMI is a small single-source brand, standard reel 4000 — order/quote a reel up front (consignment workflow accepted).** Keep SiA527DJ (dual-sourced LCSC+DigiKey) as the qualified second source on its own footprint, or the AON2406+YJQ1216A pair as the BOM-resilient fallback.

### What spec are we aiming for? (honest numbers vs market)

Market tier marketing: 12 A cont / 18 A burst (BetaFPV Matrix dual-N SiZ322DT ≈ 5.3+6.35 mΩ → ~0.84 W/phase @12 A — only reachable with dual-N + high-side drive, excluded by the no-driver constraint).
XR8G02M direct-drive P+N at ~3.3 V gate (est. ~14/20 mΩ typ): ~17 mΩ avg conduction → 1.1 W/phase @8 A, 2.4 W @12 A.
**Honest spec target: ~8 A continuous / 15–20 A burst per motor** (IDM 32/28 A). Real 1S whoop flight currents are 2–5 A with brief 8–10 A punch-outs — in practice this matches what "12 A" boards sustain on 65–75 mm 1S; we should publish the measured rating, not the marketing one. Package allocation across the family: **2×2 XR8G02M = this whoop board; the AM32 toothpick AIO keeps its own 3.3×3.3 N-FET + driver stage (DOY180N03T + NSG2065Q from 4in1-mini); AGM314MAP (3.3×3.3 complementary) is reserved for the mid-tier 2–4S Bluejay concepts (OpenAIO-Lite-HD/Analog in Notion).**

### Revision 2: deeper sweep incl. DigiKey — SiA527DJ is the dual-package ceiling

Second sweep covering Vishay PowerPAK SC-70-6, onsemi WDFN 2×2, Diodes 8-lead, AOS/Toshiba/Nexperia/Chinese-brand 2×2-8L (DigiKey acceptable, consignment OK):

- **No ≤2.3 mm complementary dual reaches N ≤20 / P ≤35 mΩ. The class ceiling is the Vishay SiA527DJ** (PowerPAK SC-70-6, 2.0×2.0×0.8): **N 29 / P 41 mΩ max @4.5 V**, spec'd down to 1.5 V gate, Vgs(th) ≤1.0 V, ±4.5 A cont / 20/−15 A pulsed, 12 V. LCSC C222486 (4.9k, ~$0.24) AND DigiKey 11k — dual-sourced. P-side 33% better than DMC1229UFDB; N-side tie. NOT pin-compatible with U-DFN2020-6 — different footprint, verify phase-leg orientation.
- Nexperia LFPAK22 is N-only (no P exists); AOS/Toshiba have no complementary ≤2.3 mm; LCSC has zero Chinese-brand complementary duals in any 2×2-8L package (verified via API).
- **If Rds is the priority, the discrete pair wins decisively: AON2406 (N, 12.5 mΩ max @4.5 V) + YJQ1216A (P, 19 max / 11 typ)** — 2.3× (N) and 3.2× (P) lower than the best dual, both spec'd at 2.5 V, ~$0.25/phase. Cost: 24 placements vs 12. YJQ1216A is LCSC-only; DigiKey P-alts at 2×2 are thin (DMP1022UFDF C155358 as second source).

All three imported into `whoop` lib (SIA527DJ-T1-GE3, AON2406, YJQ1216A); SiA527DJ datasheet local. **Open choice (Stan): SiA527DJ dual (12 pkgs, simpler) vs AON2406+YJQ1216A pair (24 pkgs, ~half the conduction loss).** DMC1229UFDB-7 below remains the fallback dual.

**Previous primary: DMC1229UFDB-7 (C443653, U-DFN2020-6, 2.0×2.0×0.6 mm)** — imported into `whoop` lib, datasheet local.
- ±12 V, N 29 mΩ / P 61 mΩ max @4.5 V, spec'd down to 1.8 V gate (true logic-level, Vgs(th) max 1.0 V) → @3.3 V worst-case ≈ N 31 / P 70 mΩ
- Id 5.6/−3.8 A cont, 20/−15 A pulsed; realistic on a dense whoop board: **~2.5 A cont / ~5 A seconds / 10 A+ pulses per phase** — matches 0802/1002 motor reality (2–5 A peaks)
- Smaller and thermally better (leadless, RθJC 30) than the SOT-23-6-class "EGX87C" parts on commercial 5 A whoop boards — not a downgrade
- Caveat: 12 V Vds at 1S — fine with low-inductance layout + per-phase ceramic bypass; watch switching spikes on the bench
- Stock: LCSC 2,280 + DigiKey backup; family stock swings hard → dual-source reels

Same-footprint variants (drop-in, U-DFN2020-6 Type B pinout): **DMC2041UFDB-7** (20 V, P 90 mΩ@4.5 V — derate to ~2 A; LCSC 165/DK 5,840) if spike margin worries; **DMC1030UFDB-7** (best silicon: N 17 typ/P 59 max, IDM 35 A, RθJC 18; LCSC 0/DK 2,784 — prototype via DK).

**Discrete-pair fallback (2× headroom): AON2406 (N, 12.5 mΩ@4.5 V, C3277752, 6.5k) + YJQ1216A (P, 11 mΩ@4.5 V, C919545, 1.9k)**, both DFN2020-6, $0.21/phase. True 5 A continuous per phase; 24 placements instead of 12 but total area (96 mm²) is still less than 12× AGM314MAP (131 mm²). BOM-resilient: N and P substitutes at 2×2 are plentiful and independent. Verify AON2406's low-Vgs behaviour (sister AON2408 C2931064 has the documented 2.5 V spec).

## FET selection (superseded) — AGM314MAP replaces AGM310MAP

**AGM314MAP (C17701056)** — pinout verified IDENTICAL to AGM310MAP from both datasheets (Pin1=S1, G1/S2/G2 bottom row, D1/D2 pads; FET1=N 30V/10mΩ@10V/30A, FET2=P −30V/18.5mΩ@10V/−20A). Same PDFN3.3×3.3 package. Strictly better dies, ~3k LCSC stock (vs 166–515), **$0.074 @500 (35% cheaper)**. Imported into `whoop` lib + datasheet at `hardware/datasheets/ESC/AGM314MAP.pdf`.

Caveat for direct 3.3 V drive: datasheet specs Rds at 10 V and 4.5 V only — N 16 mΩ typ / P 27 mΩ typ @ 4.5 V; Vgs(th) max 2.2 V means 3.3 V drive has only ~1.1 V overdrive worst-case. **Bench-verify Rds and switching at 3.3 V gate, 1S pack** before committing. (This is the same regime every CrazyBee-class board operates in.)

Second sources / hedges:
- Diodes **DMC3025LDV-13** — same PowerDI3333-8 footprint class, Western brand, DigiKey 5.7k @ $0.25/1k. BUT Vgs(th) 2.4 V / Rds spec'd @4.5 V → **not credible at 3.3 V direct drive**; it's a 2S-with-level-shift second source only.
- Discrete pair fallback on a dual-pad footprint: AGM30P08AP (P, 11.5 mΩ@4.5V, 6k stock) + AGM310AS (N, DFN2×2, $0.048) if AGMSEMI duals dry up.
- AGM318MAP / AGM312MAP exist in the same package but are worse and lower-stock — ignore.

## What commercial whoop ESCs actually use (teardown research, verified)

- **CrazyBee F4 V2 class (5–10 A 1S, the architecture we're copying)**: 12× N+P complementary pairs marked "EGX87C" (full MPN never decoded; sold as repair parts under the marking), **driven directly by EFM8 GPIO, no driver** — brushlesswhoop.com/anatomy-of-an-all-in-one-fc/
- NewBeeDrone BLV1–4 / Hummingbird V2/V3: official replacement part is "N+P MOS" ×12 — same topology, MPN undisclosed.
- **BetaFPV F4 1S 12A**: 12× Vishay SiZ322DT dual-N 25 V PowerPAK1212 (3.3×3.3) — part is **EOL at DigiKey** (successor SiZ350DT ~$2); high-side drive scheme undocumented in any teardown. Don't copy.
- NBD BLV5 / RaceSpec V2 "18A": "double N-channel", MPN undisclosed.
- 20 A+ toothpick tier: Toshiba TPN2R203NC-class discrete N + FD6288 drivers — i.e. the 4in1-mini architecture, irrelevant here.
- Bluejay layout naming note: the middle letter (Z_**H**_30, A_**X**_5) is the MCU family (H=BB21, X=BB51), NOT FET topology; only the deadtime digits hint at the stage.

## 1.8 V gyro LDO replacement (family-wide — DECIDED: TPS7A2018)

**TPS7A2018PDQNR** (X2SON-4 1×1 mm, LCSC C2878130 — only 42 pcs retail, DigiKey 25k @ $0.136 → consign; SOT-23-5 DBV C963430 with 25k LCSC stock as large-package alternate). Chosen over LP5912-1.8 on actual PSRR curves: 75 dB @ 10 kHz AND 100 kHz spec'd (LP5912 dips to ~58 dB right at the 20–30 kHz MEMS gyro drive band; the original NCV8187 collapsed to ~35 dB @ 300 kHz). Noise 7–10 µVrms. Trade-offs: PG pin lost (D8 LED net), new footprint — accepted, re-route was needed anyway. Keep a ferrite/RC pole ahead of the LDO for the ~1 MHz buck band where every candidate is ~45–50 dB. Imported into `whoop` lib here and `imports` lib in OpenAIO; full comparison table in OpenAIO `docs/SOURCING-2026-06.md`.
