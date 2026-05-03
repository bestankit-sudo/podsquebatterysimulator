# Podsque E-Ink Rendering Specification

Reference document for replicating the Podsque e-ink coffee card rendering on a firmware/embedded target. All values are **fixed pixels** at the target resolution — no scaling formulas.

> **Note:** The web simulator at `/index.html` uses these defaults but allows overriding per-layout values via `localStorage`. This document captures the canonical defaults committed to the codebase.

---

## Per-Layout Profiles

The rendering uses **3 layout profiles** based on coffee count, plus a 4th for 4-color mode.

| Property | Type | 1 Coffee | 2 Coffees | 3–4 Coffees |
|---|---|---|---|---|
| Brand | Font size | **27 px** | **18 px** | **12 px** |
| Flavour Name | Font size | **47 px** | **36 px** | **22 px** |
| Field Label | Font size | **20 px** | **14 px** | **10 px** |
| Field Value | Font size | **30 px** | **18 px** | **12 px** |
| Dot Radius | Circle | **10 px** | **4 px** | **3 px** |
| Dot Spacing | Circle (center-to-center) | **25 px** | **11 px** | **8 px** |
| Dot-to-Number Gap | Circle | **50 px** | **22 px** | **16 px** |
| Label-to-Value (LV) | Spacing (center-to-center) | **30 px** | **22 px** | **16 px** |
| Section Break (SEC) | Spacing (center-to-center) | **50 px** | **32 px** | **22 px** |
| Padding | Layout (fixed) | **15 px** | **15 px** | **15 px** |

---

## Typography (Fixed)

- Font family: **IBM Plex Sans** (Google Fonts CDN)
- Brand: weight **500**, uppercase
- Flavour Name: weight **700** (bold)
- Field Labels (NOTES, BEST SERVED AS, INTENSITY, BITTERNESS, YOUR REVIEW): weight **600**, uppercase
- Field Values: weight **700** (bold)
- Position labels (FRONT, LEFT, RIGHT, FRONT L, FRONT R, BACK L, BACK R): weight **600**, **75% of label size**, color `#666666`
- Text baseline offset from element vertical center: **`fontSize × 0.35`**

---

## Layout (Fixed)

| Setting | Value |
|---|---|
| Background | `#FFFFFF` (white) |
| Text color | `#000000` (black) |
| Position label color | `#666666` |
| Padding (all sides) | **15 px** |
| Card divider lines | **2 px**, `#000000` |
| Content overflow | Clipped to card bounds |

### Layout per coffee count
- **1 coffee** — full canvas, position label `FRONT`
- **2 coffees** — left/right split, vertical divider, labels `LEFT` / `RIGHT`
- **3–4 coffees** — 2×2 grid, both dividers, labels `FRONT L` / `FRONT R` / `BACK L` / `BACK R`

---

## Dot Rendering (Intensity & Bitterness)

- Always **10 circles** representing the 1–10 scale
- Filled (●): black fill `#000000` + **1.5 px** stroke
- Empty (○): **1.5 px** stroke only, no fill
- "(N)" number after dots: uses **field label font size** (600 weight), vertically centered to dot row
- Dot row top-aligned at cursor; center = cursor + dotR
- Horizontal layout: 10 dots + dot-to-number gap + "(N)" text
- Intensity & Bitterness rendered **side-by-side** on one row when both selected (left half / right half of contentW)

---

## DECAF Badge

- Rendered **right-aligned** on the BEST SERVED AS label row (not its own block)
- Only shown when `coffee.decaf === true` AND `decaf` field is selected
- Border: **1.5 px** black, **square corners** (no radius)
- Padding inside badge: **4 px** horizontal, **2 px** vertical
- Text: same font as field labels (600 weight, label size)
- Vertically centered with the BEST SERVED AS label

---

## Spacing Rules (Center-to-Center Pitch Model)

All vertical spacing is measured **center-to-center** between elements. The cursor variable always tracks the vertical center of the current element.

| Transition | Pitch |
|---|---|
| Brand → Flavour name | **`brandSize/2 + 1 + nameSize/2`** (tight, treated as one unit) |
| Flavour → first field label | **`PITCH_SEC + (nameSize − valueSize) / 2`** (compensates for larger flavour font) |
| Field label → its value | **`PITCH_LV`** (Label-to-Value pitch) |
| Field value → next field label | **`PITCH_SEC`** (Section Break pitch) |
| Last value → bottom of card | natural (clipped if overflow) |

---

## Field Ordering

Draw order in each card (top to bottom):

1. **Position label** (top-right corner, small gray text)
2. **Brand** (uppercase, 500 weight)
3. **Flavour Name** (large bold, wraps if needed)
4. **NOTES** (tasting notes)
5. **BEST SERVED AS** (cup size) — with optional **DECAF** badge right-aligned
6. **INTENSITY + BITTERNESS** (side-by-side row with dots)
7. **YOUR REVIEW** (remarks) — always last, max 75 chars, wraps to 2 lines

---

## Resolutions (Landscape)

| Display | Width × Height | Notes |
|---|---|---|
| 3.68" | **792 × 528** | Vendor: HanLin |
| 3.97" | **800 × 480** | **Default** — best mechanical fit per Akash |
| 3.98" | **768 × 552** | Vendor: WaveShare |

- Web preview canvas renders at **2× DPR** for sharp onscreen display
- PNG/Bitmap exports downscale to target resolution

---

## Bitmap Array Format (Firmware)

For embedded firmware on the e-ink controller:

| Property | Value |
|---|---|
| Bits per pixel | **1 bpp** |
| Bit order | **MSB first** |
| Pixel order | Row-major |
| Color encoding | White = **1**, Black = **0** |
| Luminance threshold | **0.299·R + 0.587·G + 0.114·B ≥ 128** |
| Total bytes | **`ceil(width × height / 8)`** |
| Orientation | **180° rotated** to match e-ink controller (pixels read from bottom-right to top-left) |

The web app exports a C header file with:

```c
#define EINK_IMAGE_WIDTH  800
#define EINK_IMAGE_HEIGHT 480
#define EINK_IMAGE_BYTES  48000

static const uint8_t eink_image_800x480[48000] = {
  // ... 1bpp packed pixel data, 16 bytes per line
};
```

---

## Coffee Data Schema

Each coffee object passed to the renderer has these fields:

| Field | Type | Source label | Notes |
|---|---|---|---|
| `brand` | string | (top, no label) | e.g. "Nespresso" |
| `flavour` | string | (top, no label) | e.g. "Arpeggio Decaffeinato" |
| `tasting` | string | NOTES | tasting notes |
| `cupSize` | string | BEST SERVED AS | e.g. "Espresso, Lungo" |
| `intensity` | number 1–10 | INTENSITY | rendered as dots |
| `bitterness` | number 1–10 | BITTERNESS | rendered as dots |
| `decaf` | boolean | DECAF (badge) | inline on cup size row |
| `remarks` | string | YOUR REVIEW | max 75 chars |

---

## References

- Live spec preview: simulator tab in [index.html](index.html), section "Fixed Specs & Developer Notes"
- Profile values can be tuned in the simulator and persisted to `localStorage` under key `einkProfileOverrides`
- Default values match those in `index.html` `DEFAULT_PROFILES` constant
