# Podsque Battery Life Calculator — Project Spec

## Overview

A single-page interactive battery life estimator for the **Podsque top module** (sensor + hub). The module counts coffee pods in a storage unit using IR sensors and syncs the count to the cloud and to an e-ink display over BLE.

The tool is intended to be embedded in a Notion page via Netlify Drop (public, no-auth URL). It must work as a **single self-contained HTML file** with no build step, no external API calls, and no backend.

---

## Tech Stack

- **Single file**: `index.html` — all HTML, CSS, and JS inline
- **One external dependency**: Chart.js 4.4.1 via CDN (`https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js`)
- **One Google Font**: IBM Plex Mono + IBM Plex Sans via Google Fonts CDN
- No React, no build tooling, no npm

---

## Hardware Context

| Component | Part | Role |
|---|---|---|
| MCU + BLE | nRF5340 | Host processor, runs BLE to e-ink |
| Wi-Fi companion | nRF7002 | Wi-Fi 6 companion IC, syncs to cloud |
| Sensors | 4× VL53L4CD | IR ToF pod-counting sensors |
| PMIC | nPM1100 | Battery charger + regulator |

---

## Battery Options (3 total, no 2×AAA)

| ID | Name | Nominal Wh | Efficiency | Usable Wh | Notes |
|---|---|---|---|---|---|
| `aaa4` | 4× AAA Alkaline | 7.50 Wh | **70%** | 5.25 Wh | Physically swapped when depleted |
| `li2k` | 1S 2000 mAh Li-ion | 7.40 Wh | **80%** | 5.92 Wh | Recharged via USB |
| `li4k` | 1S 4000 mAh Li-ion | 14.80 Wh | **80%** | 11.84 Wh | Recharged via USB |

**Efficiency factors must be visible in the UI** (see Efficiency Banner section).

**AAA efficiency breakdown (70%):** pulse-load derating ~78% × buck-boost regulator ~87% × temp/aging/cutoff ~94%

**Li-ion efficiency breakdown (80%):** regulator ~90% × aging/cycle degradation ~92% × temperature margin ~95% × cutoff reserve ~94%

**Usable mAh equivalent** = `(usableWh × 1000) / 3.3V` rail voltage

---

## Power Model — Fixed Constants (datasheet-sourced)

| Parameter | Value | Source |
|---|---|---|
| nRF7002 Wi-Fi active average | **60 mA** | nRF7002 PS v1.2 (~55 mA scan, ~65 mA TX) |
| nRF7002 shutdown (BUCKEN=low) | 15 µA | nRF7002 PS v1.2 |
| nRF5340 BLE TX @ 0 dBm | 3.4 mA | nRF5340 PS v2.0 |
| nRF5340 BLE avg modelled | **4 mA** | includes RX + stack overhead |
| nRF5340 deep-sleep floor | 1.1 µA | silicon; practical board is higher |
| Board deep-sleep (practical) | **30 µA** | nRF5340 ~5 µA + nRF7002 shutdown ~15 µA + leakage |
| nPM1100 quiescent | 700 nA | nPM1100 PS v1.4 — negligible |
| VL53L4CD active average | **23 mA** | DS rev.8, across 4 sensors during 1 s combined |
| VL53L4CD ULP per sensor | 55 µA | DS rev.8 |
| BLE session duration | 3.0 s | per e-ink update |
| Sensor on-time per cycle | 1.0 s | 4 sensors sequential |

---

## Daily Energy Formula

```
dailyMah = wifiMah + wakeMah + sensorMah + bleMah + sleepMah [+ ulpMah]

where:
  cycles    = syncsPerDay + appRefreshes
  wifiMah   = (wifiSec / 3600) × 60mA  × cycles
  wakeMah   = (wakeS   / 3600) × 4mA   × cycles
  sensorMah = (1.0     / 3600) × 23mA  × cycles
  bleMah    = drinks × (3.0 / 3600) × 4mA
  sleepMah  = (30µA / 1000) × 24h
  ulpMah    = [if enabled] (4 × 55µA / 1000) × 24h

batteryLifeMonths = (usableMahEq / dailyMah) / 30.44
```

---

## UI Layout — Page Structure

### Sticky header (always visible, does not scroll away)

Contains in order:
1. Title bar: "Podsque Top Module" + "Battery Estimator" tag + "nRF5340 + nRF7002" chip tag
2. Tab bar: **Without Learning** | **With Learning**
3. Control row (see below) — all panels horizontal in one row, scrollable on small screens
4. Sync summary info bar — one line showing computed syncs/day and mAh/day, updates live

### Scrollable results zone (everything below the sticky header)

Contains in order:
1. Efficiency banner (3 cards)
2. Tab-specific results (cards, bar chart, line graph, breakdown table)

---

## Tab 1: Without Learning

### Controls (one horizontal row, 3 panels)

**Panel 1 — Sync Cadence**
- Number input: "Sync every [___] min" (range: 1–1440, default: 60)
  - Derived text below: "= N syncs / day"
- Slider: App-triggered refreshes / day (0–20, default: 0)

**Panel 2 — Usage**
- Slider: Wi-Fi session duration (4–20 s, default: 8 s)
  - Sub-label: "nRF7002 boot + send + shutdown"
- Slider: Coffee drinks / day (0–20, default: 3)
  - Sub-label: "each triggers BLE sync to e-ink"

**Panel 3 — Advanced**
- Slider: MCU wake overhead / cycle (0.5–5.0 s, step 0.5, default: 1.5 s)
  - Sub-label: "nRF5340 app core @ 4 mA pre-Wi-Fi"
- Toggle: Sensors in ULP between cycles
  - Sub-label: "4× VL53L4CD @ 55 µA ea. = +220 µA constant"

### Results

**Efficiency banner** — 3 cards (one per battery):
- Battery name
- Nominal Wh
- Efficiency % (labelled "conservative")
- Usable Wh + mAh eq.
- Small note listing the efficiency factors

**Result cards** — 3 cards (one per battery):
- Verdict badge (see Verdict Logic below)
- Battery name
- Large number: estimated life (e.g. "14.2 mo" or "2.3 yr")
- Sub-line: mAh/day · usable mAh eq. · efficiency %

**Bar chart** — horizontal bars:
- One bar per battery, proportional to life in months
- Each bar labelled with battery short name
- A vertical marker line at the 9-month position on each bar (green, dashed)
- Value displayed at end of bar

**Line graph** — battery % remaining over time:
- X-axis: 0–36 months
- Y-axis: 0–100%
- One line per battery (colors: AAA=amber, Li-ion 2000=blue, Li-ion 4000=green)
- Dashed vertical green line at 9 months labelled "9 mo"
- Legend below title
- Tooltip on hover showing month and % remaining for each battery

**Daily energy breakdown table:**
- Columns: Component | mAh/day | %
- Rows: Wi-Fi sync, MCU wake, Sensor measurement, BLE → e-ink, Board sleep, [Sensor ULP if enabled]
- Each row has a small badge: "sourced" (green) or "plan" (amber)
- Total row at bottom, mAh/day in accent colour

**Formula box** (below breakdown):
- Shows the arithmetic: each component value, sum, and resulting Li-ion 4000 life

---

## Tab 2: With Learning

### Controls (one horizontal row, 4 panels)

**Panel 1 — Phase 1: Learning** (amber border highlight)
- Number input: "Sync every [___] min" (range: 1–30, default: 5)
  - Derived text: "= N syncs / day"
- Slider: Learning phase duration (1–60 days, default: 30 days)
- Slider: Wi-Fi session during learning (4–20 s, default: 8 s)

**Panel 2 — Phase 2: Active Window** (blue border highlight)
- Slider: Coffee drinks / day (0–20, default: 3)
- Slider: Active window per drink (5–90 min, step 5, default: 30 min)
- Number input: Active window — sync every [___] min (range: 1–60, default: 5)

**Panel 3 — Phase 2: Idle**
- Number input: Idle — sync every [___] min (range: 1–1440, default: 60)
- Slider: Wi-Fi session post-learning (4–20 s, default: 8 s)
- Slider: App-triggered refreshes / day (0–20, default: 0)

**Panel 4 — Advanced**
- Slider: MCU wake overhead / cycle (0.5–5.0 s, step 0.5, default: 1.5 s)
- Toggle: Sensors in ULP between cycles

### Post-Learning Sync Calculation

```
activeMinsPerDay = min(1440, drinks × windowMinutes)
idleMinsPerDay   = 1440 - activeMinsPerDay
activeSyncs      = activeMinsPerDay / activeInterval
idleSyncs        = idleMinsPerDay / idleInterval
postSyncsPerDay  = activeSyncs + idleSyncs + appRefreshes
```

### Sync Summary Bar (inside sticky header, learning tab)

Shows two phases inline:
- Phase 1: "every X min → N syncs/day · D mAh/day"
- Phase 2: "drinks × window = active mins @ interval = N syncs + idle mins @ interval = N syncs → total syncs/day · mAh/day"

### Battery Swap / Recharge Logic

After the learning phase:
- **Li-ion batteries (li2k, li4k)**: recharged to 100%
- **4×AAA**: physically swapped for fresh cells

In both cases: **post-learning life is calculated from 100% capacity using post-learning daily rate.**

Post-learning life = `usableMahEq / postDailyMah / 30.44` months

### Results

**Efficiency banner** — same 3 cards as Tab 1.

**Recharge/Swap callout** (green-tinted box):
- Title: "After learning — batteries recharged / swapped · Post-learning life from 100%"
- 3 mini cards (one per battery):
  - Battery name + action label ("recharged" for Li-ion, "swapped" for AAA)
  - Large life number (post-learning, from fresh)
  - Sub-line: post-learning mAh/day · usable mAh eq. · efficiency %
  - Life number colour follows verdict (green ≥9 mo, blue 6–9, amber <6, red <1)

**Phase depletion note** (one line, below callout):
- Shows % depleted per battery during Phase 1
- Example: "Phase 1 used: 14.6 mAh/day × 30d = 4×AAA: 83.5% depleted → swapped · Li-ion 2000: 73.9% → recharged · Li-ion 4000: 37.0% → recharged → Phase 2 starts from 100%"

**Result cards** — 3 cards showing post-learning life (same structure as Tab 1)

**Bar chart** — same structure as Tab 1, showing post-learning life

**Line graph** — full depletion journey (0–36 months):
- Lines show Phase 1 depletion (steeper), then a **discontinuity / gap** at the swap/recharge day, then lines **reset to 100%** and deplete at Phase 2 rate
- Dashed amber vertical line at learning phase end labelled "swap ↑" or "recharge ↑"
- Dashed green vertical line at 9 months labelled "9 mo"
- Legend includes: 3 battery lines + "9-mo target" + "recharge/swap" marker

**Daily energy breakdown table** — Post-learning cadence only (same structure as Tab 1)

**Formula box** — shows post-learning arithmetic

---

## Verdict Logic

Applied to both tabs. Based on estimated life in months:

| Condition | Card border | Badge text | Badge colour |
|---|---|---|---|
| ≥ 9 months | Green | "≥ 9 months ✓" | Green |
| 6–8.9 months | Blue | "6–9 months" | Blue |
| 1–5.9 months | Amber | "< 6 months" | Amber |
| < 1 month | Red (dimmed) | "< 1 month" | Red |

The 9-month threshold is the **acceptable recharge/swap cycle** for Podsque v1. Anything ≥ 9 months is green.

---

## Life Formatting Rules

| Value | Display |
|---|---|
| < 0.5 months | "X.X d" (days) |
| 0.5–23.9 months | "X.X mo" |
| ≥ 24 months | "X.X yr" |

---

## Visual Design

**Colour scheme**: Dark background (`#0f1117`), surface cards (`#181c27`), amber accent (`#f59e0b`), green (`#10b981`), blue (`#3b82f6`), red (`#ef4444`).

**Fonts**: IBM Plex Mono for all numbers, labels, formulas, badges. IBM Plex Sans for body text.

**Battery line colours on graph**:
- 4×AAA: `rgba(245,158,11,.85)` (amber)
- Li-ion 2000: `rgba(59,130,246,.85)` (blue)
- Li-ion 4000: `rgba(16,185,129,.9)` (green)

**Panel highlights**:
- Phase 1 panel: amber border `rgba(245,158,11,.4)`
- Phase 2 active panel: blue border `rgba(59,130,246,.3)`

**All sliders** use the accent amber colour for the thumb.

**Number inputs** use a slightly elevated surface background with accent border on focus.

**Sourced badge** (green): datasheet-verified values
**Plan badge** (amber): engineering planning assumptions

---

## Behaviour Requirements

- All controls update results **live on every input/change event** — no submit button
- Number inputs clamp to their min/max on blur
- Slider range bounds shown in small text below each slider
- The sync summary bar in the sticky header updates with every control change
- Tab switching shows/hides the correct control panels and result zones; charts are created once and updated (not destroyed/recreated) on subsequent renders to avoid flicker
- Chart.js `spanGaps: false` so the recharge discontinuity renders as a gap, not a connected line
- The custom annotation plugin (vertical dashed lines drawn via `afterDraw`) is registered once at Chart.js level

---

## Footer

Small text at the bottom of the scrollable zone:

> nRF5340 PS v2.0 · nRF7002 PS v1.2 · Nordic DevZone #112208 · nPM1100 PS v1.4 · VL53L4CD DS rev.8 · Board sleep = 30 µA practical (nRF5340 ~5 µA + nRF7002 shutdown ~15 µA + leakage) · AAA efficiency 70%: pulse-load ~78% × buck-boost ~87% × temp/aging/cutoff ~94% · Li-ion efficiency 80%: regulator ~90% × aging ~92% × temp ~95% × cutoff ~94% · 9-month threshold = acceptable charge/swap cycle for Podsque v1

---

## File Deliverable

Single file: `index.html`
- No separate CSS or JS files
- No build step required
- Deployable by dragging onto Netlify Drop
- Works correctly when served from any static host or opened locally via `file://`
