# Podsque Battery Life Simulator

Interactive battery life estimator for Podsque hardware modules. Single-file HTML app — no build step, no backend.

## Modules

| Tab | Status | Description |
|---|---|---|
| **Counting + Hub** | Live | nRF5340 + nRF7002 pod-counting sensor module |
| **E-Ink** | Coming Soon | E-ink display module |
| **Weighing Scale** | Coming Soon | Weighing scale module |

## Counting + Hub

Estimates battery life across 4 battery options:

- 2x AAA Alkaline (3.75 Wh, 70% efficiency)
- 4x AAA Alkaline (7.50 Wh, 70% efficiency)
- Li-ion 2000 mAh (7.40 Wh, 80% efficiency)
- Li-ion 4000 mAh (14.80 Wh, 80% efficiency)

### Two modes

- **Without Learning** — fixed sync interval, configurable Wi-Fi session, drink count, and MCU wake overhead
- **With Learning** — aggressive Phase 1 sync during a learning window, then optimized Phase 2 with active/idle cadence. Batteries recharged/swapped after learning.

### Features

- Live-updating results on every input change
- Depletion line chart (0-18 months) with 9-month target marker
- Daily energy breakdown table with sourced/plan badges
- Dark / light theme toggle
- Verdict badges: cyan (>= 9 mo), violet (6-9 mo), orange (< 6 mo), red (< 1 mo)

## Hardware

| Component | Part | Role |
|---|---|---|
| MCU + BLE | nRF5340 | Host processor, BLE to e-ink |
| Wi-Fi | nRF7002 | Wi-Fi 6 companion, cloud sync |
| Sensors | 4x VL53L4CD | IR ToF pod-counting |
| PMIC | nPM1100 | Battery charger + regulator |

## Power Model Sources

- [nRF5340 PS v2.0](https://docs.nordicsemi.com/bundle/ps_nrf5340/page/keyfeatures_html5.html)
- [nRF7002 PS v1.2](https://docs.nordicsemi.com/bundle/ps_nrf7002/page/keyfeatures_html5.html)
- [nPM1100 PS v1.4](https://docs.nordicsemi.com/bundle/ps_npm1100/page/keyfeatures_html5.html)
- [VL53L4CD DS rev.8](https://www.st.com/resource/en/datasheet/vl53l4cd.pdf)
- [Nordic DevZone #112208](https://devzone.nordicsemi.com/f/nordic-q-a/112208)

## Deploy

Single file `index.html` — deploy via GitHub Pages, Netlify Drop, or any static host. Works with `file://` too.
