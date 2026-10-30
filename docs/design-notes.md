# HX-6 design notes (rev B)

Technical reference for the board. The project overview is in the main README.

## Status

| Part | State |
|---|---|
| Schematic (4 sheets, 267 parts, 163 nets) | Complete. Netlist verified identical to the PCB. |
| PCB placement, 6 layers, 170 × 115 mm | Complete, no courtyard collisions, 0 pad conflicts |
| Power planes, switch-node pours, 315 fan-out vias | Done |
| Signal routing | **Complete**: 0 unconnected (Freerouting + scripted finishing, 6 layers). |
| DRC (KiCad 7) | 0 errors. 3 info notes: ESP32 / fuse / USB-C footprints differ from library only in off-board silkscreen moved to Fab. |
| Electrical check | 0 errors, every pin either on a net or deliberately no-connect. Schematic, PCB and circuit netlists identical (163 nets). |
| Fab outputs | `fab/` folder: JLCPCB Gerbers + drill (zip), BOM, CPL. |

## Architecture

- **Input:** 12 V via XT60, 15 A mini-blade fuse, SMBJ15CA TVS, 4 × 270 µF polymer.
- **Core rail (+VCORE):** TPS53647 4-phase controller + 4 × CSD95490Q5MC (75 A each) + 4 × 150 nH XAL7070. Runs in NVM mode (strap pins open), configured by firmware over PMBus at **0x71**.
- **ASICs:** 6 × BM1370 on one parallel core rail, 3 × 2 block under one CPU cooler, both rows on the same aligned grid (see Chain).
- **IO rails:** 1V2 and 0V8 from TLV75801 LDOs (enabled by firmware via LDO_EN).
- **Aux:** 12 V → 5 V and 5 V → 3V3 (TPS54302). USB VBUS feeds the 5 V rail through a Schottky diode, so you can flash the board without 12 V.
- **Controller:** ESP32-S3-WROOM-1-N16R8, native USB-C. The antenna overhangs the top edge.
- **Display:** Waveshare 2.8inch Capacitive Touch LCD (SKU 27579): ST7789, 240×320, SPI, CST328 capacitive touch on I2C. JST GH 1.25 mm 13-pin socket (SM13B-GHS-TB) in Waveshare's pin order, so the cable that ships with the module plugs straight in.
- **Fans:** EMC2302-1 (0x2E), 2 × 4-pin PWM headers.
- **Temperature:** 3 × TMP1075 (0x48, 0x49, 0x4A).

## Cooling (rev B)

- All 6 BM1370 sit in one 3 × 2 block (15 × 16 mm pitch, aligned grid) inside the 46 × 46 mm cooler-base area, so one CPU cooler base covers every chip.
- The top side under the cooler base carries only the six chips. All chip decoupling, the temperature sensors and the sense resistors are on the bottom, directly under the chips.
- Cooler mount: Intel LGA115x/1200 pattern, 75 × 75 mm, four 4.0 mm NPTH holes with 9 mm copper keep-outs. Most tower and low-profile coolers fit.
- The bottom parts under the array are at most 1.75 mm tall (SOIC-8). Use the cooler's insulated backplate on standoffs of 2 mm or more, or skip the backplate and use M3 screws, springs and nuts.
- Apply thermal paste or a pad on all six chips. Chip tops are coplanar on the same footprint.
- FAN1 is the cooler fan, FAN2 an optional case fan.

## Chain

Snake chain: 1 → 2 → 3 left to right along the top row, straight down to 4, then 4 → 5 → 6 right to left along the bottom row. The bottom row is turned 180° on the same grid so every hop is a short link between neighbours.

## Layer stack

F.Cu signals + switch-node / 12 V / core pours · In1 GND · In2 VCORE (VRM + ASIC area) / GND · In3 GND · In4 signals · B.Cu signals, 12 V plane under the VRM, GND.

## ESP32-S3 pin map

| GPIO | Function | GPIO | Function |
|---|---|---|---|
| 0 | BOOT button | 12 | LCD SCK |
| 1 | ASIC reset | 11 | LCD MOSI |
| 17 | ASIC TX → CI | 2 | LCD MISO |
| 18 | ASIC RX ← RO | 7 | LCD CS |
| 8 | I2C SDA | 6 | LCD DC |
| 9 | I2C SCL | 5 | LCD RST |
| 10 | VRM enable | 38 | LCD backlight |
| 13 | LDO enable (1V2/0V8) | 15 | Touch INT |
| 14 | VR_RDY | 16 | Touch RST |
| 47 | VR ALERT | 21 | User button |
| 4 | VIN sense (12 V ÷ 11) | 48 | Status LED |
| 19/20 | USB D−/D+ | TXD0/RXD0 | UART header |

**Firmware:** ESP-Miner-NerdQAxePlus already has a 6× BM1370 + TPS53647 4-phase profile (NerdHaxe-γ). This board needs a board class with the pin map above and an SPI display driver for the 2.8" panel.

## Where the data came from

- **BM1370 pin map and land pattern:** from the public open-source reverse-engineering (NerdQAxe++ files). Bitmain publishes no datasheet.
- **CSD95490 land pattern (TI DMC0012A):** pad geometry from the same public library, because TI's drawing is image-only. Pin functions checked against TI's pin table.
- Everything else (circuit, values, placement, board) is original to HX-6.

## Verify before ordering

1. Open in KiCad 8/9/10 and run DRC once more (newer KiCad versions add checks). R20 is do-not-fit: VR_HOT (TPS53647 pin 19, open drain, unused by firmware) is left unconnected on purpose.
2. VRM compensation (10 k + 2.2 nF, 47 pF), IMON (75 k) and LSET (110 k from TI's LSET equation for 150 nH) are starting values. Check them on the bench at low load.
3. 25 MHz oscillator must run at 1.2 V (Taitien OXLTDLJANF-25.000000 or equal). The footprint is the 3.2 × 2.5 mm 4-pad type.
4. Display: the socket follows the Waveshare 27579 pinout (VCC, GND, MISO, MOSI, SCLK, LCD_CS, LCD_DC, LCD_RST, LCD_BL, TP_SDA, TP_SCL, TP_INT, TP_RST). The touch controller is CST328, so firmware needs its driver.
5. Fab notes: vias in the ASIC/power-stage pads must be filled and capped (via-in-pad). 6 layers, 1 oz outer / 2 oz inner, ENIG.

## Ordering at JLCPCB

- Upload `HX6_revB_gerbers_JLCPCB.zip`. Layers: 6, thickness 1.6 mm, outer copper 1 oz, inner copper 2 oz recommended (≈ 100 A core current), surface finish ENIG.
- Via-in-pad: the ASIC and power-stage exposed pads have vias inside them. Choose **epoxy-filled and capped vias** (JLCPCB "POFV / via-in-pad") so solder is not wicked away.
- Min track/space 0.127 mm, min via 0.45 / 0.2 mm, within JLCPCB 6-layer standard capability.
- Assembly: upload `HX6_revB_BOM_JLCPCB.csv` + `HX6_revB_CPL_JLCPCB.csv` and match LCSC parts in their tool. BM1370 (and possibly ESP32 module, CSD95490, TPS53647) are not JLCPCB stock: supply them or hand-solder. Check CPL rotations in the JLCPCB preview, especially bottom-side parts.
