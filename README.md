# OpenAIO-Whoop

Open-source 25.5×25.5 mounting-pattern whoop AIO: flight controller + 4× Bluejay brushless ESC + ExpressLRS 2.4 GHz receiver, **digital video only** (SH1.0 6-pin HD-VTX port, no onboard analog VTX/OSD). Target: 1S/2S whoop class (65–85 mm).

| Stage | Basis | Status |
|---|---|---|
| FC | OpenFC-Lite-Mini rev 2 sheets (RP2354A) | imported |
| ESC | **new design** — EFM8BB51 (Bluejay BB51) + power stage TBD | parts imported, schematic not drawn |
| RX | OpenRX-Lite (ESP32-C3 + SX1281, serial ELRS) | imported |

## Status — early design, no hardware

This repo was forked from OpenAIO (toothpick AIO) and is being adapted for whoop class. Done so far: repo restructure, current FC/RX sheets imported, Bluejay ESC parts imported (`whoop` library: EFM8BB51F16G, AGM310MAP, BY25Q64ESCIG), datasheets pulled, market + sourcing research in `docs/`.

Open design decisions (see `docs/ESC_DESIGN.md`):
- **Pack voltage: 1S vs 2S** — gates the entire ESC gate-drive topology and the BEC topology (1S needs a boost to feed an O4 Lite properly)
- **ESC power stage**: AGM310MAP complementary P+N is a 5–8 A-class topology; the 2026 market floor is 12 A cont — N+N + driver alternative documented
- Blackbox: BY25Q64 SPI NOR replaces the microSD slot (sheet still shows microSD until redrawn); BY25Q64ESCIG itself needs a substitute (stock = 1 pc)
- ESC channel sheets (`esc_channel.kicad_sch` ×4) still contain the inherited AT32/AM32 toothpick channel — to be gutted and redrawn as EFM8BB51 + power stage

## Repository layout

```
OpenAIO-Whoop/
├── README.md  LICENSE
├── docs/
│   ├── ESC_DESIGN.md               ← Bluejay ESC topology research + sourcing
│   └── MARKET-RESEARCH-2026-06.md  ← competitive landscape, both AIO classes
└── hardware/
    ├── OpenAIO-Whoop.kicad_pro / .kicad_sch / .kicad_pcb
    ├── schematics/
    │   ├── fc/      rp2350a, power, imu, blackbox, pads  (OpenFC-Lite-Mini rev2; no osd — digital only)
    │   ├── esc/     esc_channel (×4 from root — inherited placeholder, to be redrawn)
    │   └── elrs/    elrs                                  (OpenRX-Lite)
    ├── lib.kicad_sym / lib.pretty / lib.3dshapes           ← FC library
    ├── whoop.kicad_sym / whoop.pretty / whoop.3dshapes     ← Bluejay ESC parts
    ├── components.kicad_sym / 4in1ESC.pretty / .3dshapes   ← ESC donor parts (INA186 etc.)
    ├── OpenRX-Shared.kicad_sym / .pretty / .3dshapes       ← RX library
    ├── sym-lib-table / fp-lib-table
    └── datasheets/    FC/  ESC/  ELRS/
```

All libraries project-local; KiCad standard libs are the only external references. KiCad 10.

## Key parts

### Bluejay ESC (`whoop` library)
| Function | Part | LCSC | Note |
|---|---|---|---|
| ESC MCU ×4 | EFM8BB51F16G-C-QFN20R (8051, 50 MHz) | C6547511 | Bluejay BB51 target; not in JLC assembly lib — verify/consign |
| Power stage | DMC1229UFDB-7 complementary N+P, U-DFN2020-6 (2×2 mm), ×3/channel, GPIO direct drive | C443653 | 12 V, N 29 mΩ/P 61 mΩ @4.5 V; ~2.5 A cont/5 A burst per phase. Fallback: AON2406+YJQ1216A discrete pair |
| 1.8 V gyro LDO | TPS7A2018PDQNR, X2SON-4 (1×1 mm) | C2878130 | 75 dB PSRR @10k–100 kHz; LCSC thin, consign from DigiKey |
| Blackbox flash | BY25Q64ESCIG 64 Mbit SPI NOR | C50176394 | stock ≈ 0 — substitute W25Q64JV-class |

### FC / RX (inherited from sibling designs)
RP2354A, LSM6DSV16XTR, 2× LMR51430, TPS2116, LP5912, NCV8187 (**unobtainable — replace**), TF-021B microSD (to be replaced by SPI flash), ESP32-C3FH4 + SX1281 + TLV75533. Full tables in the OpenAIO repo README.

## Firmware targets

| Stage | Target |
|---|---|
| FC | Betaflight, derived from `OPENFC_LITE_MINI_RP2350A` (custom target) |
| ESC ×4 | **Bluejay** BB51, 48 kHz (2S) / 96 kHz (1S), bidirectional DShot |
| RX | ExpressLRS `Unified_ESP32C3_2400_RX` (survives in ELRS 4.0 targets) |

## Competitive bar (2026)

Reference: BetaFPV Matrix 1S 3IN1 HD ($50, 12A/18A Bluejay, serial ELRS, 5V/3A BEC, SH1.0 6-pin O4 port, 3.2 g). To match the class: 12 A cont ESC, 5V/3A holding to ~2.8 Vin, 25.5×25.5 mount, ≤3.5 g. Differentiators: open hardware, real blackbox flash, real current shunt + published scale, maintained BF target. Details: `docs/MARKET-RESEARCH-2026-06.md`.

## License

Hardware: [CERN-OHL-S-2.0](https://ohwr.org/cern_ohl_s_v2.txt). Firmware references upstream projects (Betaflight, Bluejay, ExpressLRS) under their respective licenses.
