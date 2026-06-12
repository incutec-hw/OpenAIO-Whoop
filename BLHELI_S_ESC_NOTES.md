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
