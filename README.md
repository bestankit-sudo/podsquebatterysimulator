# Podsque Simulator

Interactive simulator for Podsque hardware modules — e-ink image generation and battery life estimation. Single-file HTML app — no build step, no backend.

**Live:** [podsquebatterysimulator.vercel.app](https://podsquebatterysimulator.vercel.app/)

## Modules

| Tab | Status | Description |
|---|---|---|
| **E-Ink Image** | Live | PNG generator for e-ink displays — coffee card layouts at target resolutions |
| **Counting + Hub** | Live | Battery estimator for nRF5340 + nRF7002 pod-counting sensor module |
| **E-Ink** | Live | Battery estimator for nRF52832 + 3.9" e-paper display module |
| **Weighing Scale** | Coming Soon | Weighing scale module |

## E-Ink Image Generator

Generates PNG images for Podsque e-ink displays to test readability at target resolutions before hardware production.

### Features

- Select up to 4 coffees from a catalogue of 25 (mix of regular and decaf)
- Choose up to 6 display fields: Brand Name, Flavour Name, Flavour, Remarks, Decaf, Cup Size, Intensity, Bitterness
- Predefined landscape resolutions from candidate display panels:
  - 3.68" — 792×528
  - 3.97" — 800×480 (default, best mechanical fit per Akash)
  - 3.98" — 768×552
  - Custom resolution input
- Numeric values (Intensity, Bitterness) rendered as dot visualization or plain numbers
- Layout adapts to coffee count: full canvas (1), left/right split (2), 2×2 grid (3-4)
- Position labels (FRONT, LEFT/RIGHT, FRONT L/R, BACK L/R) match physical pod holder slots
- Color mode (B/W, 3-Color, 4-Color) — coming soon
- Download as PNG for testing on physical e-ink panels

## Counting + Hub

Estimates battery life across 4 battery options: 2x AAA, 4x AAA, Li-ion 2000 mAh, Li-ion 4000 mAh.

### Modes

- **Without Learning** — fixed sync interval, configurable Wi-Fi session, drink count, MCU wake overhead
- **With Learning** — aggressive Phase 1 sync during learning window, then optimized Phase 2 with active/idle cadence. Batteries recharged/swapped after learning.

### Power model components

- Wi-Fi sync (nRF7002, 60 mA per cycle)
- MCU wake overhead (nRF5340, 4 mA)
- Sensor measurement (4x VL53L4CD, 23 mA)
- BLE TX to e-ink (4 mA per drink)
- BLE advertising (10 µA constant — always on for e-ink + phone connectivity)
- Li-ion protection IC leakage (6 µA constant)
- Board deep sleep (30 µA — nRF5340 + nRF7002 shutdown + PCB leakage)
- Sensor ULP mode (optional, +220 µA constant)

### Hardware

| Component | Part | Role |
|---|---|---|
| MCU + BLE | nRF5340 | Host processor, BLE to e-ink |
| Wi-Fi | nRF7002 | Wi-Fi 6 companion, cloud sync |
| Sensors | 4x VL53L4CD | IR ToF pod-counting |
| PMIC | nPM1100 | Battery charger + regulator |

## E-Ink Display

Estimates battery life across 4 options: CR2032, CR2450, CR2477, 2x AAA Alkaline.

### Controls

- Content changes per week (layout/pod type changes)
- Live inventory count toggle (refresh on every coffee)
- Payload type (Scene JSON min/max, full-screen image)
- Freshness mode (Max Battery 60min / Balanced 30min / Real-time 5min)
- BLE advertising interval (1s or 2s)
- E-paper refresh duration and board sleep current

### Power model components

- E-paper refresh (100 mA peak, 3.5s — per Akash, Hardware & BLE Strategy Sync)
- MCU wait during refresh (1.9 mA idle on BUSY pin, on top of e-paper draw)
- BLE connection setup + data receive (7 mA avg)
- MCU render — JSON parse + layout + framebuffer (4.5 mA, 0.3s)
- SPI framebuffer transfer to panel (4.5 mA, 0.05s)
- BLE version checks (6 mA per check, per freshness mode)
- BLE advertising (7-12 µA constant — always on for Sync Now)
- Board sleep (5 µA — nRF52832 1.9 µA + regulator + leakage)
- E-paper controller standby (0.5 µA)

### Coin cell limitation

Per Akash (Hardware & BLE Strategy Sync, 2026-03-29): e-paper refresh draws ~100 mA peak, which sags coin cell voltage to 2.4-2.5V causing MCU brownout. Coin cells are modeled at 30-45% effective capacity. 2x AAA has no peak current limitation and is the recommended path.

### Hardware

| Component | Part | Role |
|---|---|---|
| MCU + BLE | nRF52832 | BLE SoC, receives payload, renders scene, drives panel |
| Display | 3.9" 800x480 e-paper | Monochrome electrophoretic, zero power to hold image |
| Battery | Coin cell or 2x AAA | User-replaceable, ≤10 mm module thickness |

## Power Model Sources

- [nRF5340 PS v2.0](https://docs.nordicsemi.com/bundle/ps_nrf5340/page/keyfeatures_html5.html) — MCU + BLE for counting hub
- [nRF7002 PS v1.2](https://docs.nordicsemi.com/bundle/ps_nrf7002/page/keyfeatures_html5.html) — Wi-Fi 6 companion
- [nRF52832 PS v1.8](https://docs.nordicsemi.com/bundle/ps_nrf52832/page/keyfeatures_html5.html) — BLE SoC for e-ink module
- [nPM1100 PS v1.4](https://docs.nordicsemi.com/bundle/ps_npm1100/page/keyfeatures_html5.html) — PMIC
- [VL53L4CD DS rev.8](https://www.st.com/resource/en/datasheet/vl53l4cd.pdf) — ToF sensor
- [Nordic DevZone #112208](https://devzone.nordicsemi.com/f/nordic-q-a/112208) — power measurement reference

## Deploy

Single file `index.html` — deploy via Vercel, GitHub Pages, or any static host. Works with `file://` too.
