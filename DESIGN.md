# Home Assistant Wall Dashboard — Design Specification

Display target: **LG 25UM64-S** — 25" UltraWide IPS, **2560 × 1080**, 21:9, 250 cd/m², HDMI 1.4.
Purpose: always-on family wallboard: security visibility first, then quick control of shades/lights/climate/access/media.

---

## 1. Design principles

1. **Pixel-native, zero scrolling.** Every region is sized to exact pixels of a 2560×1080 canvas. Nothing scales, nothing overflows, nothing requires interaction to be useful.
2. **Cameras are the hero.** The top ⅔ of the canvas is an 8-tile video wall. Tiles are exactly **640×360** — native 16:9, and the natural resolution of camera *sub-streams*, so streams render 1:1 without GPU scaling.
3. **Direct Play philosophy applies to UI too:** sub-streams + WebRTC everywhere on the wall; main streams only on demand in the detail view.
4. **Glanceable states, color-coded:** closed/green, open/red, moving/blue, unknown/gray. Readable from across the room.
5. **One default view** the display returns to automatically. Everything else is one tap away.
6. **Dark theme only** — this panel lives on a wall; dimmed backlight at night.
7. **Visual language: LCARS.** The UI skin mimics the *Star Trek: TNG* Library Computer Access/Retrieval System (see `mockup/dashboard-mockup-lcars.html`). Rules:
   - Typeface: **Antonio** (Google Fonts; standard LCARS substitute), uppercase labels, generous letter-spacing
   - Palette: black background; blocks/pills in orange `#FF9C00`, gold `#FFCC66`, peach `#FFCC99`, salmon `#CC6666`, lilac `#CC99CC`, periwinkle `#9999CC`, sky `#99CCFF`; alerts in `#DD4444`
   - Panels framed by colored end-caps with asymmetric radii ("elbows"); buttons are pills with black text
   - All sliders/gauges render as **segmented block gauges**, not continuous bars
   - Devices get registry-style names (`CAM 01 · DRIVEWAY`, `ENV 04 Environmental Systems`, `AUDIO NODE 02`); Ring units are designated `COMMS`
   - Footer ticker styled as scrolling `LOG ENTRY` feed incl. a computed stardate
   - Functional notes: keep contrast accessible (peach/sky text on black); alert blink uses steps() not fades so it reads at distance

---

## 2. Device inventory → placement map

| Device family | Integration | Entities used | Where it appears |
|---|---|---|---|
| Reolink NVR (6 cams) | `reolink` (core) + go2rtc | `camera.*_sub` per channel, binary_sensor motion/person | Camera wall tiles 1–6; detail view (main streams) |
| Ring doorbells ×2 | `ring` (cloud) | `camera.front_door`, `binary_sensor.ding/motion`, battery | Camera wall tiles 7–8 as **snapshot stills**, tap → live |
| IKEA roller shades | Z2M/ZHA (`cover`) | `cover.bedroom_blinds` … position, battery | Bottom-center column, one row per room |
| IKEA Trådfri lights | Z2M/ZHA (`light`) | Grouped per room via Light Group helpers | Bottom-center, under each room's shade row |
| Midea dehumidifier | `midea_lan` (HACS) → `humidifier` | target/current humidity, fan mode | Bottom-left, top card |
| Govee air purifier | Govee integration → `fan`/sensor | switch, PM2.5, filter life | Bottom-left, second card |
| Aqara sensors (temp/RH, door/window, motion, leak, vibration) | Z2M | `sensor.*`, `binary_sensor.*` grouped by area | Left column chips; leak/battery feed footer ticker |
| Aqara cameras (G2H/G3) | via Aqara/Z2M or Matter bridge | `camera.*` | Swap-in slot on wall (tap-to-swap) or detail view |
| Apple HomePod minis | `apple_tv` (AirPlay) | `media_player.homepod_*`, volume, play/pause | Bottom-right media row + TTS announce |
| RATGDO garage | `ratgdo` (core) / ESPHome | `cover.garage`, obstruction/motion/button sensors, `light.ratgdo` | Bottom-right, large state tile |

---

## 3. Canvas blueprint (pixel math)

```
2560 ───────────────────────────────────────────────────────────────────
┌────────────────────────────────────────────────── HEADER · 56px ─────┐
│ ⌂ WALL │ CAMERAS │ CLIMATE │ LIGHTS            68°F · Rain · 14:32  │
├──────────────────────── CAMERA WALL · 720px (4×2 @ 640×360) ─────────┤
│ Driveway       │ Front Door(Ring)│ Backyard      │ Garage Int     │
│ (Reolink sub)  │ snapshot 30s    │ (Reolink sub) │ (Reolink sub)  │
│ Patio          │ Side Gate       │ Living (Aqara)│ Porch (Ring)   │
│ (Reolink sub)  │ (Reolink sub)   │ swap-in slot  │ snapshot 30s   │
├─────────────── BOTTOM BAND · 248px ──────────────────────────────────┤
│ CLIMATE · 640px  │ SHADES + LIGHTS · 960px     │ ACCESS + MEDIA·960px │
│ Dehumidifier     │ Bedroom  ▥72% ◉ 4 lights ▂▄ │ Garage CLOSED  [OPEN]│
│ 48%→45% RH       │ Living   ▥100% ◉ 6 lights   │ Obstruction: clear   │
│ Purifier · PM2.5 │ Office   ▥85%  ◉ 3 lights   │ Doorbell events ▢▢▢  │
│ Temp/RH chips    │ Kitchen  ▥—    ◉ 5 lights   │ HomePods ▶ 🔊 TTS ✎ │
├──────────────────────── FOOTER · 56px ───────────────────────────────┤
│ ⚠ Basement leak 2d ago · Batteries ≥ 34% · HA up 12d · 14:32 · Mon   │
└──────────────────────────────────────────────────────────────────────┘
   56 + 360 + 360 + 248 + 56 = 1080 ✔      640×4 = 960+960+640 = 2560 ✔
```

Grid definition (layout-card):

```yaml
layout:
  grid-template-columns: 640px 640px 640px 640px
  grid-template-rows: 56px 360px 360px 248px 56px
  grid-template-areas:
    "header header header header"
    "cam1   cam2   cam3   cam4"
    "cam5   cam6   cam7   cam8"
    "climate shades shades access"
    "footer footer footer footer"
```

(`shades` spans two 640px tracks = 960… i.e. columns named `c1 c2 c3 c4`; areas `"climate c2 c3 access"` with climate=640, middle=1280 split internally, access right. Implementation detail in §8.)

---

## 4. View strategy

| # | View | Purpose | Notes |
|---|---|---|---|
| 0 | **WALL** (default) | The blueprint above; no scroll | Auto-restored after 5 min idle |
| 1 | CAMERAS | 3×3 grid of **main** streams, tap = fullscreen, PTZ pad where supported | On demand only |
| 2 | CLIMATE | Full humidity/temp history graphs, purifier/dehumidifier detail, all Aqara env sensors by room | Graphs via Mini Graph Card |
| 3 | LIGHTS | All rooms expanded: individual bulbs, color temps, scenes ("Movie", "Goodnight") | Scenes row at top |
| 4 | SYSTEM | HA health, UGOS/Docker hosts, battery report, Zigbee network map | Maintenance |

Kiosk behavior: hide sidebar/header chrome, disable context menus, auto-return timer, night dimming (§10).

---

## 5. Streaming architecture

```
Reolink NVR ──RTSP sub (640×360 H.264)──▶ go2rtc (built into HA) ──WebRTC──▶ wall tiles
             └─RTSP main (4K)───────────▶ (pulled lazily, detail view only)
Ring cloud ──snapshot polling──────────▶ /local/snapshots/*.jpg ──▶ wall stills
Ring cloud ──live view (on tap)────────▶ ring integration player
```

go2rtc additions (`configuration.yaml`):

```yaml
go2rtc:
  streams:
    cam_driveway:  rtsp://user:pass@NVR_IP:554/Preview_01_sub
    cam_backyard:  rtsp://user:pass@NVR_IP:554/Preview_02_sub
    cam_side_gate: rtsp://user:pass@NVR_IP:554/Preview_03_sub
    cam_garage_in: rtsp://user:pass@NVR_IP:554/Preview_04_sub
    cam_patio:     rtsp://user:pass@NVR_IP:554/Preview_05_sub
    cam_living:    rtsp://user:pass@AQARA_G3_IP:554/stream1   # Aqara G3
```

Rules of thumb:
- **Sub-streams only** on the wall (H.264 mandatory — WebRTC won't carry Reolink's HEVC sub-streams; set sub-stream codec to H.264, 640×360 @ 15 fps in the NVR UI).
- WebRTC ≈ 0.5 s glass-to-glass; never use HLS on the wall (5–20 s lag, high CPU).
- Ring has no local RTSP: poll snapshots via automation (§10), keep live view behind a tap. Ring's cloud rate limits make continuous streaming both slow and fragile — don't fight it.
- Budget: 8 × 640×360 WebRTC decodes is trivial for any modern kiosk box.

---

## 6. Custom components required

| Component | Why |
|---|---|
| **layout-card** | Pixel-exact grid areas on the fixed canvas |
| **WebRTC Camera Card** (AlexxIT) | Sub-second streams, lazy-load via `intersection` |
| **Mushroom** + **card-mod** | Compact uniform rows for lights/shades/chips |
| **mini-media-player** | HomePod row |
| **button-card** | Garage tile, scene buttons, shade arrows |
| **auto-entities** | Footer ticker (low batteries, leaks, open doors) |
| **Mini Graph Card** | Climate detail view |
| **Browser Mod** or **Fully Kiosk** | Chrome hiding, idle return, night dimming |

---

## 7. Entity hygiene (do this before building cards)

- Areas assigned for every device; friendly names like `Living Room Blinds`, `Basement Leak`.
- Create **Light Group helpers** per room (`light.group_living_room`) so one card drives many bulbs.
- Template sensor `sensor.lowest_battery` (min of all battery entities) → footer.
- Template binary sensor `alert_active`: OR of water leaks, garage left open >10 min, any door unlocked overnight → drives footer highlight + red pulse on header dot.
- Input booleans: `kiosk_night_mode` (automation-set), `wall_camera_live` (future pause-all toggle).

---

## 8. Dashboard scaffold (key YAML)

### 8.1 View skeleton

```yaml
title: Wall
path: wall
type: custom:grid-layout
layout:
  grid-template-columns: 640px 640px 640px 640px
  grid-template-rows: 56px 360px 360px 248px 56px
  grid-template-areas: |
    "header header header header"
    "cam1   cam2   cam3   cam4"
    "cam5   cam6   cam7   cam8"
    "climate shades shades access"
    "footer footer footer footer"
  mediaquery: {}
cards: []   # populated below
```

### 8.2 Camera tile (repeats ×6)

```yaml
- type: custom:webrtc-camera-card
  view_config_name: wall
  entity: camera.driveway_sub          # or url: cam_driveway (go2rtc name)
  intersection: 0.5                    # pause decode when hidden
  muted: true
  card_mod:
    style: |
      ha-card { border-radius: 0; border: none; }
      .video { height: 360px; object-fit: cover; }
  double_tap_action:
    action: navigate
    navigation_path: cameras
```

### 8.3 Ring doorbell tile (snapshot, tap-for-live)

```yaml
- type: picture-entity
  entity: camera.front_door_ring
  camera_view: still                   # static snapshot, cheap
  show_name: false
  show_state: false
  tap_action:
    action: more-info                  # opens dialog with live view
  card_mod:
    style: |
      ha-card { height: 360px; }
      img { object-fit: cover; height: 100%; }
```

Snapshot freshness automation:

```yaml
automation:
  - alias: Ring snapshot refresh (wall)
    triggers:
      - trigger: time_pattern
        minutes: "/1"
    actions:
      - action: camera.snapshot
        data:
          entity_id: camera.front_door_ring
          filename: /config/www/snapshots/front_door.jpg
      - action: camera.snapshot
        data:
          entity_id: camera.side_gate_ring
          filename: /config/www/snapshots/side_gate.jpg
```

(Use `picture-elements`/`picture` pointing at `/local/snapshots/front_door.jpg?v={{ now().minute }}` if you prefer cache-busted stills over the camera entity.)

### 8.4 Climate column

```yaml
- type: vertical-stack
  grid_area: climate
  cards:
    - type: custom:mushroom-humidifier-card
      entity: humidifier.basement_dehumidifier
      name: Dehumidifier
      collapsible_controls: true
    - type: custom:mushroom-fan-card
      entity: fan.air_purifier
      name: Purifier
      icon_animation: true
    - type: horizontal-stack
      cards:
        - type: custom:mushroom-entity-card
          entity: sensor.living_room_temperature
          use_light_color: false
        - type: custom:mushroom-entity-card
          entity: sensor.living_room_humidity
        - type: custom:mushroom-entity-card
          entity: sensor.nursery_temperature
        - type: custom:mushroom-entity-card
          entity: sensor.basement_humidity
```

### 8.5 Shades + lights column (one row per room)

```yaml
- type: vertical-stack
  grid_area: shades
  cards:
    - type: horizontal-stack
      cards:
        - type: custom:mushroom-cover-card
          entity: cover.living_room_blinds
          name: Living
          show_position_control: true
          icon_type: horizontal
        - type: custom:mushroom-light-card
          entity: light.group_living_room
          name: Lights
          show_brightness_control: true
          collapsible_controls: true
    # ... repeat rows: Bedroom, Office, Kitchen, Nursery
```

### 8.6 Access + media column

```yaml
- type: vertical-stack
  grid_area: access
  cards:
    - type: horizontal-stack
      cards:
        - type: custom:button-card
          entity: cover.garage_door
          name: Garage
          show_state: true
          state:
            - operator: "=="
              value: "open"
              color: red
            - operator: "=="
              value: "closed"
              color: green
            - operator: default
              color: orange
          tap_action:
            action: toggle            # ratgdo cover toggle w/ confirm
          hold_action:
            action: more-info
        - type: vertical-stack
          cards:
            - type: custom:mushroom-entity-card
              entity: binary_sensor.garage_vehicle_present
              name: Car
            - type: custom:mushroom-entity-card
              entity: binary_sensor.garage_obstruction
              name: Obstruction
    - type: horizontal-stack
      cards:
        - type: custom:mini-media-player
          entity: media_player.kitchen_homepod_mini
          name: Kitchen Pod
          artwork: none
          shortcuts: []
        - type: custom:mini-media-player
          entity: media_player.office_homepod_mini
          name: Office Pod
          artwork: none
    - type: custom:mushroom-template-card
      name: "Announce:"
      label: "{{ states('input_text.tts_message') }}"
      multiline_secondary: true
      tap_action:
        action: perform-action
        perform_action: script.announce_all_homepods
```

`script.announce_all_homepods` loops `tts.speak`/`media_player.play_media` (TTS) across the HomePod media_players at a defined volume, restoring volume afterwards.

### 8.7 Header + footer

Header: navigation chips (navigate actions to views 1–4) + weather condition/temp + time. Footer: `auto-entities` card filtered to:

```yaml
- type: custom:auto-entities
  card:
    type: glance
    columns: 8
  filter:
    include:
      - entity_id: binary_sensor/*leak*
        state: "on"
      - entity_id: sensor.*_battery
        below: 20
    exclude: []
```

plus a clock card and uptime sensor. Red background pulse when `alert_active` is on (card-mod animation).

---

## 9. Detail views (build after Wall works)

- **CAMERAS**: grid of 6 Reolink **main** streams + 2 Ring live cards; WebRTC card per tile with `muted: false`, fullscreen double-tap; PTZ feature for the PTZ cam.
- **CLIMATE**: Mini Graph Cards (24 h humidity/temp per room), dehumidifier + purifier controls expanded.
- **LIGHTS**: per-room stacks with individual bulbs, color-temp controls, and scene buttons (button-card): Movie, Dinner, Goodnight (also closes shades, kills media).
- **SYSTEM**: HA disk/CPU, Zigbee mesh map link, battery report table, recent logbook errors.

---

## 10. Display & kiosk management

If Android kiosk box + Fully Kiosk (recommended):

- Start URL = `/dashboard-wall/wall?kiosk`, keep screen on, hide system bars
- Motion detection (front camera) → wake; else schedule: 100% brightness 07:00–22:00, 15% 22:00–23:59, screen off 00:00–06:30
- Remote admin pinned so you can push config changes
- Screensaver disabled (the dashboard *is* the screensaver)

Alternative (any device): **Browser Mod** + **kiosk-mode** HACS plugins; replicate dimming with a companion-app command or CEC.

Automation pair:

```yaml
- alias: Wall night dimming
  triggers:
    - trigger: time
      at: "22:00:00"
    - trigger: time
      at: "07:00:00"
  actions/conditions:
    # set input_boolean.kiosk_night_mode; Fully Kiosk command or browser_mod
    # brightness service accordingly
```

Idle-return: Fully Kiosk "Return to default screen" after 300 s, or Browser Mod's idle hook navigating back to `/wall`.

---

## 11. Performance budget (sanity check)

| Load | Cost | Verdict |
|---|---|---|
| 6× WebRTC sub 640×360 decode | ~1 core total on any N100-class box | fine |
| 2× JPEG still refresh/min | negligible | fine |
| SQLite recorder incl. all these entities | moderate | exclude camera entities from recorder; 10-day purge |
| Zigbee traffic (shades/lights/sensors) | trivial | ensure good mesh: ≥2 routers (Trådfri bulbs are routers) |

HA settings: recorder `exclude: domains: [camera]`; logbook likewise. Stream component left default; go2rtc does the heavy lifting.

---

## 12. Build checklist

1. Integrations online, entities named/areaed (§2, §7)
2. NVR sub-streams reconfigured to H.264 640×360; verified in go2rtc UI (:1984)
3. Custom components installed (§6)
4. Scaffold grid renders at exactly 2560×1080 with no scrollbar (§8.1)
5. Camera tiles live <1 s latency (§8.2), Ring stills refreshing (§8.3)
6. Bottom band interactive: shade moves, light toggles, garage cycles, TTS announces
7. Footer shows a test low-battery entity
8. Kiosk: boot-to-dashboard, chrome hidden, idle-return, night dimming verified
9. Recorder excludes configured; 48 h soak test; check RAM/CPU trend

---

## 13. References

- Visual mockups: `mockup/dashboard-mockup-lcars.html` (approved LCARS skin; auto-scales to any window), plus earlier variants `dashboard-mockup.html` (neutral dark) and `dashboard-mockup-whimsical.html` — open in any browser
- LG 25UM64-S spec sheet: <https://www.lg.com/us/support/products/documents/25UM64-S%20Spec%20Sheet.pdf>
- go2rtc integration: <https://www.home-assistant.io/integrations/go2rtc/>
- WebRTC Camera Card: <https://github.com/AlexxIT/WebRTC>
- Reolink integration: <https://www.home-assistant.io/integrations/reolink/>
- layout-card: <https://github.com/thomasloven/lovelace-layout-card> · Mushroom: <https://github.com/piitaya/lovelace-mushroom> · mini-media-player: <https://github.com/kalkih/mini-media-player> · button-card: <https://github.com/custom-cards/button-card> · auto-entities: <https://github.com/thomasloven/lovelace-auto-entities>
- Fully Kiosk: <https://www.fully-kiosk.com/> · kiosk-mode: <https://github.com/maykar/kiosk-mode>
