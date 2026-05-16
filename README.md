# Awair Element Awtrix Companion

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Ftioan%2Fawair-element-awtrix-companion%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Ftioan%2Fawair-companion.yaml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Created with Claude](https://img.shields.io/badge/Created%20with-Claude%20Sonnet%204.6-blueviolet.svg?logo=anthropic)](https://claude.ai)

Awair-inspired air-quality visualisation for the **Awtrix 3** 32×8 LED matrix display, in the same visual language as [`air-dots-card`](https://github.com/tioan/air-dots-card) for Home Assistant.

A Home Assistant **automation blueprint** that registers two custom apps on the Awtrix and pushes them via MQTT:

**App 1 — `awair`** — pure dot bars + score (static, 8 s)

```
┌──────────────────────────────────────┐  32×8 LED matrix
│                                      │
│  8 5    P  P  P  P  P                │  ← top dots = worst (purple)
│  8 5    R  R  R  R  R                │
│  8 5    O  O  O  O  O                │
│  8 5    Y  Y  Y  Y  Y                │
│  8 5    G  G  G  G  G                │  ← bottom dots = best (green)
│                                      │
│  ████████████████░░░░░░░░░░░░░░░░░░  │  ← score progress bar
└──────────────────────────────────────┘
```

**App 2 — `awair_text`** — scrolling line of all values, each colored by severity (~12 s)

```
…  Score 85   Temp 22C   Hum 45%   CO2 750   VOC 250   PM 8  …
```

Each sensor's dot bar uses the Awair Element style: colors are fixed by position (🟢 → 🟡 → 🟠 → 🔴 → 🟣 from bottom to top), more lit dots = worse. The score number and the progress bar share the score's severity color. In the scrolling text, labels stay gray and only the values are tinted, so the eye reads "what changed" first.

## Requirements

- An **Awtrix 3** display (Ulanzi TC001 or DIY) with MQTT enabled
- **Home Assistant** with the MQTT integration set up and reachable from the Awtrix
- 5 numeric sensors — Awair Element entities work out of the box; SCD40, SEN55, BME680, IKEA VINDSTYRKA, etc. work too if you adjust thresholds

## Install — Blueprint (recommended)

### One-click import

Click the **Open in Home Assistant** button at the top of this README. It opens HA's blueprint-import dialog pre-filled with the raw URL. Confirm, and the blueprint is added.

### Or manually

1. Copy `blueprints/automation/tioan/awair-companion.yaml` into your HA config at the same path: `/config/blueprints/automation/tioan/awair-companion.yaml`
2. Reload automations or restart HA.
3. **Settings → Automations & Scenes → Blueprints** — "Awair Element Awtrix Companion" appears.
4. Click **Use blueprint** and fill in the form:
   - **MQTT prefix** — Awtrix Web UI → MQTT → "Prefix", e.g. `awtrix_a1b2c3`
   - **Score sensor** + **5 sensor entities** (Temp / Humidity / CO₂ / TVOC / PM2.5)
   - (Optional) thresholds, colors, score position, progress bar position
5. Save. The display updates on every sensor change + once per minute.

## Configuration options

All exposed via the blueprint UI, grouped into sections:

| Section | Inputs |
|---|---|
| **Awtrix device** | MQTT prefix, custom app name (default `awair`) |
| **Air-quality score** | Score sensor entity (0–100, higher = better) |
| **Sensors** | Temperature, Humidity, CO₂, TVOC, PM2.5 entities |
| **Bar colors** (advanced) | 5 color pickers, one per severity level. LED-friendly defaults; tweak if the green looks too bluish on your panel. |
| **Thresholds** (advanced) | Comma-separated quad per sensor, Awair defaults pre-filled |
| **Display behavior** (advanced) | Score number position (`left` / `right` / `off`), progress bar position (`top` / `bottom` / `off`), text app on/off, app durations, scroll speed |

### Default Awair thresholds

| Sensor | Unit | Thresholds | Scale |
|---|---|---|---|
| Temperature | °C | `18, 20, 25, 27` | symmetric — optimum in the middle |
| Humidity | % | `30, 40, 60, 65` | symmetric |
| CO₂ | ppm | `600, 1000, 2000, 4500` | linear — higher = worse |
| TVOCs | ppb | `300, 500, 3000, 25000` | linear |
| PM2.5 | µg/m³ | `12, 35, 55, 150` | linear |

## Local preview

Open `preview/index.html` in a browser — same renderer the HA template runs, with sliders for every sensor and a tab to switch between the dots overview, the scrolling text, and auto-rotate.

## Advanced — package YAML instead of the blueprint

For users who want to hack on per-sensor labels, symmetric flags, or want a debug template sensor exposing the rendered JSON: drop `packages/awair-companion.yaml` into your `/config/packages/` folder. Functionally identical to the blueprint, exposed in plain YAML.

## Testing without HA

```bash
# Push a payload directly via HTTP
curl -X POST "http://<AWTRIX_IP>/api/custom?name=awair" \
  -H "Content-Type: application/json" \
  -d @payload.json

# Clear
curl -X POST "http://<AWTRIX_IP>/api/custom?name=awair" -d ''
```

Copy the JSON from the preview's "MQTT payloads" details panel as `payload.json`.

## Troubleshooting

**Apps don't appear**
MQTT prefix wrong. Awtrix Web UI → MQTT → "Prefix" must match the blueprint input.

**Apps appear but don't cycle**
Check `curl http://<AWTRIX_IP>/api/settings` for `ATIME` (should be 5–10) and `MATP` (should be `false`). Or the device is in hold-mode — long-press the middle touch button on the Ulanzi to release.

**Score shows "99" when it should be "100"**
Score is clamped to 99 in the dots app so 3 digits don't overlap the first bar. The text app shows the real value.

**All dots dim, text shows "--"**
Sensor states are `unknown` / `unavailable`. Check entities in Developer Tools.

**Text app shows no colors**
Pre-0.96 Awtrix firmware doesn't support segmented text colors. Upgrade firmware, or switch to the package version and replace `text` with a single string.

**Green looks bluish**
The default L1 color is `#4ADE80` (Tailwind green-400). On many LED matrices this leans cyan. Try `[0, 255, 102]` or `[34, 197, 94]` in the blueprint's Bar colors section.

## Credits

This blueprint was developed interactively with **[Claude Sonnet 4.6](https://claude.ai)** by Anthropic. The visual style is inspired by the [Awair Element](https://www.getawair.com/) air-quality monitor and the [`air-dots-card`](https://github.com/tioan/air-dots-card) Home Assistant card.

## License

[MIT](LICENSE) — same as `air-dots-card`.
