# ESP32-S3 Smart Specs

A 4-layer PCB for smart glasses featuring an ESP32-S3 with camera live streaming over Wi-Fi.

## Features

- **ESP32-S3 Microcontroller** with camera interface
- **Camera Module** for live video capture
- **Wi-Fi Streaming** — real-time video over Wi-Fi
- **USB-C Port** — power and firmware flashing
- **Li-Ion Battery Charger** — integrated charger IC with status indicators
- **4-Layer PCB Design** — optimized for signal integrity and compact form factor

## Design

Designed in KiCad (v9/v10). Includes full schematics, PCB layout, and board design files.

### Custom Footprints

Two custom footprints (not in default KiCad libraries) are included in the repo:

| Footprint | Part | File |
|-----------|------|------|
| `MOLEX_5034802400` | J4 — 24-pin 0.5mm FFC connector (camera) | `MOLEX_5034802400.kicad_mod` |
| `MP34DT05TR-A_STM` | U7 — MEMS microphone | `MP34DT05TR-A_STM.kicad_mod` |

These were extracted from the PCB and stored in the project root. If KiCad reports missing footprints on first open, associate these `.kicad_mod` files via the footprint library table or reassign to equivalent library footprints.

### Generating Gerbers

Run **File → Fabrication → Gerbers** in KiCad to regenerate production files — no pre-built gerbers are committed.

## License

MIT — see [LICENSE](LICENSE).
