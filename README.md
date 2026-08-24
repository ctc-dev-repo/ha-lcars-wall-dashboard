# ha-lcars-wall-dashboard

Design spec + interactive mockups for a Home Assistant wall dashboard rendered
on an **LG 25UM64-S** (25" UltraWide, 2560×1080), skinned in the classic
**Star Trek LCARS** interface style.

- Full design document: [`DESIGN.md`](DESIGN.md)
- **Approved mockup (open in any browser — auto-scales to your window):**
  [`mockup/dashboard-mockup-lcars.html`](mockup/dashboard-mockup-lcars.html)

## What it shows

| Region | Content |
|---|---|
| Header | LCARS elbow bar, view nav, live clock |
| Camera wall (4×2 @ 640×360) | 6× Reolink NVR sub-streams via go2rtc/WebRTC, 2× Ring doorbell snapshot stills, Aqara G3 swap-in slot, motion alerts blink red |
| ENV 04 · Environmental | Midea dehumidifier, Govee air purifier, Aqara temp/RH chips |
| SYS 12 · Portal & Illumination | IKEA roller shades (segmented position gauges) + Trådfri light groups per room |
| OPS 07 · Access Bay & Audio | RATGDO garage door state/open control, doorbell event log, HomePod mini audio nodes + TTS announce |
| Footer | Scrolling `LOG ENTRY` ticker: leaks, low batteries, lock status, uptime, stardate |

## Repo contents

```
DESIGN.md                     full spec: layout math, integrations, YAML, automations,
                              performance budget, build checklist
mockup/dashboard-mockup-lcars.html      approved LCARS skin (primary)
mockup/dashboard-mockup.html            earlier neutral-dark variant
mockup/dashboard-mockup-whimsical.html  earlier playful variant
```

## Quick start for the mockups

```bash
open mockup/dashboard-mockup-lcars.html     # macOS; or double-click the file
```

No dependencies; the Antonio font loads from Google Fonts when online and falls
back to Arial Narrow offline.
