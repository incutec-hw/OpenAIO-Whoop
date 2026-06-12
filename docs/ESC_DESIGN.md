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

## FET selection — AGM314MAP replaces AGM310MAP

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

## 1.8 V gyro LDO replacement (family-wide, decided by research)

**LP5912-1.8DRVR (C2876234)** — same WSON-6 2×2 land pattern as the LP5912-3.3DRVR already in the design, PSRR 75 dB@1 kHz (= NCV8187), noise 12 µVrms (better), keeps the PG pin (D8 LED net survives). Pin order differs from NCV8187 (IN/OUT corners swap, SNS gone) → minor re-route, same pads. **Stock: LCSC only 550 pcs (1.8 V is the rare variant), DigiKey 5.8k — buy/consign the reel early.** Runner-up: TPS7A2018 (95 dB/7 µVrms, X2SON-4 1×1 or SOT-23-5, DK 25k) if dropping PG and changing footprint is acceptable.
