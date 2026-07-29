# SpecVue

A 4-layer PCB for smart glasses featuring an ESP32-S3 with camera live streaming and on-device object classification over Wi-Fi.

## Overview

This board is a compact (90×17 mm) smart glasses controller designed around the ESP32-S3. It captures video from a parallel-interface camera module, runs a lightweight object classifier (e.g., TensorFlow Lite Micro / ESP-DL) on the ESP32-S3, and streams results (bounding boxes, labels) over Wi-Fi — optionally with live video. Li-Po battery powered with built-in charging.

### Block Diagram

```
USB-C ─┬─► MCP73831 (Charger) ──► Li-Po Battery
        │                              │
        └──► VSYS ◄────────────────────┘
               │
        ┌──────┼──────────┐
        │      │          │
   AMS1117  AP2121AK  LP2985
    3.3V      1.3V      2.8V
        │      │          │
   VDD_CPU  Core Rail  Camera IO
     & I/O    (1.3v)    (2.8v)

┌──────────────── Data Flow ────────────────┐

 Camera ──► ESP32-S3 ──┬─► TFLite Micro ──► Wi-Fi ──► Detected objects
 (DVP)      (capture)   │   (classifier)    (MQTT/UDP)
                        └─► Wi-Fi ──► Live video stream (optional)
```

### Specifications

| Parameter | Value |
|-----------|-------|
| Board dimensions | 90 mm × 17 mm |
| Layer count | 4 (F.Cu, In1.Cu, In2.Cu, B.Cu) |
| Board thickness | 1.6 mm |
| Copper weight | 1 oz (0.035 mm) all layers |
| Dielectric | FR4 (εr = 4.5) |
| Input power | USB VBUS (5 V) or Li-Po battery (3.7–4.2 V) |
| Charger current | 500 mA (set by R1 = 2 kΩ) |

## Power Tree

| Rail | Voltage | Regulator | Max Current | Supplies |
|------|---------|-----------|-------------|----------|
| VSYS | Battery / 5V | (direct from USB-Diode-BAT path) | — | Input to all regulators |
| 3.3 V | 3.3 V | AMS1117-3.3 (U4) | 800 mA | ESP32-S3 core + I/O, flash, mic |
| 1.3 V | 1.3 V | AP2121AK-1.3 (U6) | 400 mA | Camera core (1.3v rail) |
| 2.8 V | 2.8 V | LP2985-2.8 (U5) | 150 mA | Camera analog/digital IO (2.8v rail) |

### Power Path

1. **USB VBUS** feeds the MCP73831-2-OT (U3) Li-Po charger IC.
2. Battery connects through AO3401A (Q1) P-FET for reverse-polarity protection.
3. The FET's output (VSYS) is shared between USB and battery — no external power mux needed (Q1's body diode provides OR-ing).
4. VSYS powers all three LDOs in parallel.
5. SS14 (D4) Schottky diode provides additional input protection.

## Microcontroller

**ESP32-S3 (U1)**, QFN-56 7×7 mm, 0.4 mm pitch.

- Dual-core Xtensa LX7 @ up to 240 MHz
- Built-in Wi-Fi 802.11 b/g/n + BLE 5.0
- 512 kB SRAM, supports external PSRAM
- 40 MHz main crystal (Y2, 2520 package, 10 ppm) with 15 pF load caps
- CHIP_PU (pin 4) pulled high with a resistor, with reset button (Reset)

### GPIO Allocation

| GPIO(s) | Function | Connected To |
|---------|----------|-------------|
| GPIO4 | ADC / Battery monitor | Voltage divider on VBAT |
| GPIO10–16, GPIO38–40, GPIO47–48 | Camera parallel data bus | J4 FFC connector |
| GPIO17 | I2C SCL | J4 pin 5 (camera control) |
| GPIO18 | I2C SDA | J4 pin 4 (camera control) |
| GPIO0 | Boot mode selection | Flash button |
| GPIO46 | Strapping pin | Pull-up/pull-down resistors |
| U0TXD, U0RXD | Serial console | Debug UART |

**Note**: Several GPIOs (21, 33, 37, 41–46) are available for future expansion or current sensing.

## Memory

**W25Q128JVS (U2)** — 128 Mbit (16 MB) SPI flash, SOIC-8 package.

| ESP32-S3 Pin | Flash Pin | Net |
|-------------|-----------|-----|
| GPIO26 (SPICS0) | CS# | SPICS0 |
| GPIO27 (SPID) | DO/IO1 | SPID |
| GPIO28 (SPIQ) | DI/IO0 | SPIQ |
| GPIO29 (SPIWP) | WP#/IO2 | SPIWP |
| GPIO30 (SPIHD) | HOLD#/IO3 | SPIHD |
| GPIO31 (SPICLK) | CLK | SPISCK |

The ESP32-S3 uses its dedicated SPI2 controller for flash, with four I/O lines for Quad-SPI operation. The flash operates at 3.3 V from the VDD_CPU rail.

## Camera Interface

**J4** — Molex 5034802400, 24-pin 0.5 mm pitch FFC connector.

| Pin | Net | Typical Camera Function |
|-----|-----|------------------------|
| 1 | NC | — |
| 2, 22 | 2.8 V | AVDD (analog supply) |
| 3, 11 | GND | Ground |
| 4 | IO18/SDA | SCCB / I2C data |
| 5 | IO17/SCL | SCCB / I2C clock |
| 6 | IO16 | D0 (data bit 0) |
| 7 | IO15 | D1 |
| 8 | IO14 | D2 |
| 9 | IO13 | D3 |
| 10 | IO12 | D4 |
| 12 | IO11 | D5 |
| 13 | IO10 | D6 |
| 14 | IO48 | XCLK (camera master clock) |
| 15 | 2.8 V | DVDD (digital supply) |
| 16 | 1.3 V | DOVDD (core supply) |
| 17 | IO47 | D7 (data bit 7) |
| 18 | NC/Pull config | Camera ID / detection |
| 19 | IO38 | VSYNC (vertical sync) |
| 20 | Pull-down (10 kΩ) | PWDN (power-down, pulled low for normal operation) |
| 21 | IO39 | HREF / HSYNC (horizontal sync) |
| 23 | IO40 | PCLK (pixel clock) |
| 24 | (shared pull) | Reset / detection |

The pinout is compatible with OV2640/OV5640/GC2035 8-bit parallel (DVP) camera modules. The 1.3 V rail (core) and 2.8 V rails (analog + digital IO) match typical CMOS sensor requirements.

## Audio

**MP34DT05TR-A (U7)** — MEMS digital microphone, PDM output.

| Mic Pin | Net |
|---------|-----|
| VDD | 3.3 V (from AMS1117) |
| GND | GND |
| CLK | PDM clock from ESP32-S3 |
| DAT | PDM data to ESP32-S3 |
| LDO | Output from internal LDO (decoupled with 0.1 µF + 10 µF) |

The MP34DT05TR-A uses a 4-pin custom footprint. The PDM clock and data connect to ESP32-S3 GPIOs for audio capture alongside video streaming.

## USB-C

**J1** — USB-C receptacle, 14-pin (USB 2.0 only).

| Feature | Implementation |
|---------|---------------|
| CC1, CC2 | 5.1 kΩ pulldown (R17, R14) — sink-only, no PD |
| D+, D- | Series 33 Ω resistors for HS impedance matching |
| VBUS | Direct to charger input |
| SHIELD | Connected to GND via filter |

No SBU lines or e-marker chip — this is a straightforward USB 2.0 sink configuration.

## Battery Charger

**MCP73831-2-OT (U3)** — Single-cell Li-Po linear charger.

| Component | Value | Purpose |
|-----------|-------|---------|
| R1 | 2 kΩ | PROG resistor — sets charge current to 500 mA |
| STAT | LED (D1) | Charge status indicator — on during charging, off when done, blink for fault |

## Status LEDs

| LED | Color | Driven By | Function |
|-----|-------|-----------|----------|
| D1 | Red (0603) | MCP73831 STAT | Charging status — on during charging, off when done |
| D2 | (0603) | VDD_CPU via 470 Ω | Power indicator (always on when 3.3 V present) |
| D3 | (0603) | GPIO1 — ESP32-S3 | User indicator |

All LEDs have ~470 Ω series current-limiting resistors.

## Antenna

**AE1** — 2.4 GHz PCB antenna (TI SWRA117D reference design), left-edge placement.

This is a PCB trace antenna based on Texas Instruments application note SWRA117D. It uses an inverted-F or meandered PCB trace pattern on the board edge with a ground clearance keep-out area beneath it. The RF feed uses a pi-network (C20, C21, L6) for impedance matching. No separate antenna component is needed — the antenna is part of the PCB itself.

## PCB Stackup

| Layer | Material | Thickness | Type |
|-------|----------|-----------|------|
| Top solder mask | — | 0.01 mm | — |
| F.Cu | Copper | 0.035 mm | Signal + traces |
| Prepreg | FR4 (εr = 4.5) | 0.1 mm | — |
| In1.Cu | Copper | 0.035 mm | Signal routing |
| Core | FR4 | 1.24 mm | — |
| In2.Cu | Copper | 0.035 mm | Signal routing |
| Prepreg | FR4 (εr = 4.5) | 0.1 mm | — |
| B.Cu | Copper | 0.035 mm | Signal + traces |
| Bottom solder mask | — | 0.01 mm | — |

## Component Count

| Category | Count |
|----------|-------|
| Resistors (0402) | 17 |
| Capacitors (0402) | 36 |
| Inductors | 2 |
| ICs | 7 |
| Connectors | 3 (USB-C, FFC, pin header) |
| LEDs (0603) | 3 |
| Push buttons | 2 |
| Crystal | 1 |
| Antenna | 1 |
| Transistor (P-FET) | 1 |
| Test points | 2 |
| Schottky diode | 1 |
| **Total footprints** | **~76** |

## Getting Started

### Prerequisites

- **KiCad 9.x or later** (files saved with generator v9.0)
- A 24-pin 0.5 mm pitch FFC camera module (OV2640/OV5640 compatible)

### Opening the Project

```bash
git clone <repo-url>
cd SpecVue
# Open SpecVue.kicad_pro in KiCad
```

### First-Time Setup — Footprint Libraries

Two custom footprints live in the project root as `.kicad_mod` files. The first time you open the project, KiCad may show warnings like *"Footprint library 'FFC24' not found"*. This is expected — the footprint geometry is already embedded in the PCB (the board displays correctly), but the schematic-to-footprint link needs a library entry.

**To resolve (one-time setup):**

1. In KiCad, open **Preferences → Manage Footprint Libraries** (or **PCB Editor → Preferences → Footprint Libraries**)
2. Switch to the **Project** tab
3. Click **Add empty row** (folder+star icon) and configure each:

| Nickname | Library Path | Type |
|----------|-------------|------|
| `FFC24` | `./MOLEX_5034802400.kicad_mod` | `KiCad` |
| `footprints` | `./MP34DT05TR-A_STM.kicad_mod` (or use folder containing it) | `KiCad` |

Alternatively, use a single entry pointing to the project root:

| Nickname | Library Path | Type |
|----------|-------------|------|
| `Custom` | `./` (project directory) | `KiCad` |

Then update the footprints via **Tools → Update Footprints from Library** and reassign `FFC24:` and `footprints:` references to the new library.

(If you skip this step, the schematic will show footprint warnings, but the PCB opens and edits fine — the geometry is stored in the `.kicad_pcb` file.)

### Generating Gerbers

**File → Fabrication → Gerbers** in KiCad (no pre-built gerbers are committed).

### Bill of Materials

See [`SpecVue.csv`](./SpecVue.csv) — all components listed with DigiKey part numbers. Key ICs:

| Ref | Part | Package | DigiKey (or equivalent) |
|-----|------|---------|------------------------|
| U1 | ESP32-S3 | QFN-56 7×7 mm | Espressif |
| U2 | W25Q128JVS | SOIC-8 | W25Q128JVS |
| U3 | MCP73831-2-OT | SOT-23-5 | MCP73831T-2ATI/OT |
| U4 | AMS1117-3.3 | SOT-223 | AMS1117-3.3 |
| U5 | LP2985-2.8 | SOT-23-5 | LP2985-2.8DBVR |
| U6 | AP2121AK-1.3 | SOT-23-5 | AP2121AK-1.3TRG1 |
| U7 | MP34DT05TR-A | Custom (MEMS mic) | MP34DT05TR-A |
| J1 | USB-C (14-pin) | HC-TYPE-C-16P-01A | HCTL |
| J2 | Programming header | 1×02 pin header 2.54 mm | — |
| J4 | Molex 5034802400 | 24-pin FFC 0.5 mm | 5034802400 |
| Y2 | 40 MHz crystal | 2520 SMD 10 ppm | — |
| Q1 | AO3401A | SOT-23 | P-Channel MOSFET |
| D4 | SS14 | SMA | Schottky diode |

## Firmware Development

Use **ESP-IDF** or **Arduino-ESP32** with the ESP32-S3 target. Key framework integration points:

- Camera: `esp32-camera` library (8-bit DVP mode)
- Classification: TensorFlow Lite Micro or ESP-DL for on-device inference; output is label + confidence per frame
- Communication: Stream detected objects via MQTT or UDP to a companion app/server; optionally embed bounding boxes in a live video stream
- Audio: I2S peripheral in PDM mode for the MEMS mic
- Wi-Fi: ESP32 native Wi-Fi stack for video streaming and detection telemetry

## Board Dimensions

```
  ┌──────────────────────────────────┐
  │  90.00 mm                        │
  │                                  │
  │                                  │ 17.00 mm
  │                                  │
  └──────────────────────────────────┘
```

The 90×17 mm aspect fits within a standard glasses temple arm. No mounting holes — the board is intended to be secured by enclosure pressure or adhesive within the frame.

## License

MIT — see [LICENSE](LICENSE).
