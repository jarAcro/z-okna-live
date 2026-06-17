# Z Okna — Live: Complete Technical Reference

## What It Is

Z Okna is a single-file, browser-based interactive light installation. A live camera feed drives real-time motion detection; detected motion casts soft shadows across a canvas filled with shimmering light "coins." The light quality shifts automatically with the real sun position at a selected geographic location, pulling from live weather data to adjust intensity and colour temperature. Everything runs in a single `<canvas>` element at 680×420px, composited from four offscreen canvases at ~60fps.

There are no external libraries, no build step, no server. The entire application is one `.html` file.

---

## File Structure

```
z-okna-v1-stable.html
├── <style>          — All CSS (dark UI, preset bar, transport, controls panel, scene modal)
├── <body>
│   ├── #start-screen   — Initial camera permission prompt
│   ├── #app            — Main UI (hidden until camera granted)
│   │   ├── #preset-bar     — Preset buttons + save/delete
│   │   ├── #transport      — Camera select, mirror, invert, reset, record, grab, kiosk, edit scenes
│   │   ├── #scene-indicator — Active scene name + elevation progress bar
│   │   └── #ui             — Surface, grading, slat, weather, audio controls
│   ├── #stage          — The main canvas + grab-flash overlay
│   ├── #kiosk-hint     — "Shift+K to exit" overlay (kiosk only)
│   ├── #scene-modal    — Scene editor modal (conditionally shown)
│   └── Hidden elements — <video id="cam">, <video id="bg-vid">, 4 offscreen canvases
└── <script>         — All application logic (~1100 lines, strict mode)
```

---

## Canvas Architecture

Five canvases are used. Four are offscreen (hidden in the DOM); one is the visible output.

| Canvas ID | Variable | Size    | Purpose |
|-----------|----------|---------|---------|
| `#main-cv` | `cx`    | 680×420 | Final composited output — drawn to the page |
| `#c-cur`   | `xCur`  | 40×25   | Motion detection — tiny for speed, frame-diff only |
| `#lfc`     | `lfx`   | 680×420 | Light field — 80 radial-gradient "coins" drawn here |
| `#texc`    | `tcx`   | 680×420 | Surface texture — background colour + imported image/video + slat mask |
| `#shc`     | `shx`   | 680×420 | Shadow field — motion shadow pixels drawn here |
| `#gradc`   | `gcx`   | 680×420 | Grading buffer — contrast/brightness/saturation + duotone applied here before blending into texc |

### Compositing Order (per frame)

```
cx ← texc              (source-over, α=1)  — background surface
cx ← lfc               (screen, α=1)       — light coins blended additively
cx ← shc               (multiply, α=1)     — shadow darkens what's beneath
cx ← vignette radial   (source-over)       — optional edge darkening
```

The `screen` blend mode for light means coins brighten the background without clipping. The `multiply` blend for shadows darkens proportionally — a zero-value shadow pixel (α=0) is a no-op.

---

## Startup Flow

1. Page loads → `#start-screen` shown, `#app` hidden.
2. User clicks **Enable camera** → `startCamera(null)` called.
3. On success: `getCameras()` populates the camera select dropdown; `#start-screen` hidden, `#app` shown.
4. `buildSlatMap()` pre-computes the slat occlusion map; `texDirty=true` queues a texture rebuild.
5. `renderPresetBar()` renders preset buttons; `applyPreset('Default')` loads default values into `P`.
6. `requestAnimationFrame(frame)` starts the render loop.

---

## Render Loop (`frame()`)

Called every animation frame (~16ms at 60fps). Each tick:

```js
t += 0.016;              // global time accumulator (seconds)
processMotion();         // camera frame diff → motion/smooth/shadow fields
updateLivingWorld();     // breathing, colour drift, celestial, coin drift
applyBgVideo() or buildTexture();  // rebuild surface if dirty
buildLightField();       // draw 80 coins onto lfc
// composite: texc → lfc (screen) → shc (multiply) → vignette
requestAnimationFrame(frame);
```

`t` is a monotonically increasing float used as the time argument for all oscillators. It does not represent real clock seconds.

---

## Motion Detection

Camera frames are downscaled to **40×25** (1000 pixels total) for performance. Each pixel is compared to the previous frame using luminance difference.

### `processMotion()`

1. Draw camera frame (mirrored if enabled) into the 40×25 `c-cur` canvas.
2. Read pixels; compute per-pixel luma diff vs. `prevFrameData`.
3. **`velocityField`** — instantaneous motion speed: `max(0, (diff - sensitivity) / 150)`.
4. **`motionField`** — accumulated presence:
   - If diff > threshold → increase (motion detected).
   - If diff ≤ threshold → decay at rate `L.FADE` (scene parameter).
   - Inverted mode reverses the logic: stillness builds the field, motion drains it.
5. **`shadowMemory`** — very slow accumulation: only grows if `motionField > 0.3`; drains slowly enough that 5 minutes of stillness fully clears it.
6. **Cohesion blur** — a Gaussian blur over `motionField` using a precomputed kernel of radius `L.COHESION`. The kernel is rebuilt only when the radius changes. This spreads motion into physically larger shadow blobs.

The result is `smoothField` — a spatially blurred, temporally smoothed representation of presence in the frame.

### Motion Field Layout

Fields are `Float32Array` of size `SW*SH` (40×25 = 1000 values). Spatial coordinates: `index = y * SW + x`.

---

## Light Coin System

80 coins (configurable via `NUM_COINS`) are scattered randomly at startup and never move significantly. Each is a radial gradient ellipse (rotated, squeezed) drawn onto `lfc`.

### Coin Properties (randomized at creation)

| Property | Range | Purpose |
|----------|-------|---------|
| `x`, `y` | 0–W, 0–H | Fixed position on canvas |
| `uBase`, `plBase` | 0.6–1.4, 0.3–2.1 | Two radius blend targets |
| `sizeF` | 0.5–1.5 | Per-coin size multiplier |
| `sqzVar` | 0.85–1.15 | Per-coin vertical squeeze variation |
| `angVar` | ±0.13 rad | Per-coin angle offset |
| `f1/p1`, `f2/p2`, `f3/p3` | frequency/phase pairs | Three oscillators for shimmer |
| `driftX`, `driftY` | ±0.4 | Slow random drift velocity |
| `offX`, `offY` | (live) | Accumulated drift offset |

### Per-Frame Coin Rendering

For each coin:
1. Compute shimmer value from three sine oscillators at different frequencies × `L.SHIMSPD`.
2. Sample `smoothField` at the coin's position to get local motion intensity.
3. Compute radius `R` from `L.CSZ` (scene size), coin size, height scale, night boost.
4. Build a 5-stop radial gradient using `r/g/b` from `getLightColor()` and brightness values derived from `L.LINT` and shimmer.
5. Draw gradient ellipse (scaled by `L.SQZ`, rotated by `L.LANG + sunAngleOffset + coin.angVar`).
6. If slats are on, apply slat mask to the light field pixel-by-pixel, with a subtle "leak" effect at slat edges driven by global shimmer.

### Blend mode

Without slats: `screen` (light adds to whatever is below).
With slats: `source-over` (so the slat masking step can use the alpha channel directly).

---

## Shadow Field

`buildShadowField()` converts `smoothField` + `shadowMemory` into an RGBA image written to `shc`.

### Algorithm

For each pixel at full resolution (680×420):
1. Sample `smoothField` at the pixel's normalized coordinates via bilinear interpolation (`sampleSmooth()`).
2. Also sample at 4 offset positions (±`SSFT/SW` in x and y) to give the shadow physical softness/spread.
3. Sample `shadowMemory` bilinearly for a gentle "ghost" trace of past presence.
4. Combine: `raw = (s0*2.5 + s1 + s2 + s3 + s4) / 6.5 + mem * 0.08`.
5. Apply smoothstep to `raw` → `sv` (0=no shadow, 1=full shadow).
6. Shadow alpha = `sv * sdepth * 255`, where `sdepth = L.SDEPTH * (1 - nightMode * 0.45)`.
7. RGB set to the scene's `SH_TINT` colour.

The shadow ImageData (`shadowID`) is allocated once and reused every frame — no GC pressure.

---

## Surface / Texture (`buildTexture()` / `applyBgVideo()`)

The `texc` canvas holds the static background. It is rebuilt only when `texDirty = true` (parameter change or preset switch). For video backgrounds, `applyBgVideo()` is called every frame instead.

### Build steps

1. Fill with HSL from `P.BGH / P.BGS / P.BGL`, shifted by `nightMode` (hue +185°, saturation +4, lightness -15 at full night).
2. If `MATERIAL = 'import'` and an image is loaded:
   a. Draw the image into `gradc` scaled to fill (cover-fit × `P.ZOOM`).
   b. Apply CSS filter: `contrast(BG_CONTRAST) brightness(BG_BRIGHT) saturate(BG_SAT)`.
   c. If `DUO_MIX > 0.01`, run a duotone pixel pass: remap each pixel's luminance to a gradient between `DUO_A` and `DUO_B`.
   d. Composite `gradc` onto `texc` at `TEX_DEPTH` alpha using `BG_BLEND` composite operation.
3. If `SLAT_ON`, apply the precomputed `slatMap` pixel-by-pixel (multiply RGB, keep alpha).

---

## Slat Overlay (`buildSlatMap()`)

Precomputed at startup and whenever any `SLAT_*` parameter changes. Stored as `Float32Array` of size `W*H` — one float per pixel, 0=fully occluded, 1=fully open.

### Algorithm

For each pixel `(x, y)`:
1. Compute projection onto the slat angle: `proj = (x - W/2) * cos(ang) + (y - H/2) * sin(ang) + SLAT_OFF`.
2. Map into repeating slat pitch (`SLAT_W + GAP_W`).
3. Determine if pixel is in a slat or gap.
4. Compute distance to nearest slat edge.
5. Apply smoothstep over `SLAT_BLUR` pixels → soft edge.
6. Return `1 - softEdge * SLAT_DEPTH` inside the slat, `1` in the gap.

---

## Scene System

### Six Time-of-Day Scenes

`Night`, `Sunrise`, `Morning`, `Midday`, `Late Afternoon`, `Sunset`

Each scene defines 17 parameters split into two groups:

**Light field parameters:**

| Key | Description |
|-----|-------------|
| `LINT` | Base light intensity (0.05–1.0) |
| `CSZ` | Coin radius in pixels (20–200) |
| `SQZ` | Vertical squeeze of ellipses (0.2–1.0) |
| `LANG` | Light angle in degrees (0–180) |
| `ESFT` | Edge softness / falloff (0.05–0.55) |
| `HEIGHT` | Directional height — affects gradient shape (0–1) |
| `COIN_VAR` | Size variety blend between uniform/varied (0–1) |
| `SHIMMER` | Shimmer amplitude (0–1) |
| `SHIMSPD` | Shimmer oscillator speed multiplier (0.05–3.0) |
| `SUN_DRIFT` | Autonomous sun angle drift rate (0–0.5) |
| `PAL` | Light colour palette key (`moon`, `sunrise`, `warm`, `noon`, `late`, `sunset`) |
| `SH_TINT` | Shadow colour hex string |

**Motion shadow parameters:**

| Key | Description |
|-----|-------------|
| `SENSITIVITY` | Luma diff threshold before motion registers (1–60) |
| `SDEPTH` | Shadow opacity multiplier (0–1) |
| `COHESION` | Gaussian blur radius on motion field (1–6) |
| `FADE` | Per-frame motion field decay rate (0.001–0.06) |
| `SSFT` | Shadow spread — samples taken at ±this distance in motion-pixel units (0.1–4.0) |

### Scene Blending

The engine blends continuously between adjacent scenes based on `celestialElevation` (sun elevation in degrees) and whether the sun is ascending or descending.

**Ascending path:** Night → Sunrise → Morning → Midday
**Descending path:** Midday → Late Afternoon → Sunset → Night

Transition ranges (ascending example):
- `elev < -10` → Night
- `-10 to 0` → Night → Sunrise
- `0 to 8` → Sunrise → Morning
- `8 to 32` → Morning → Midday
- `> 32` → Midday

All numeric parameters are linearly interpolated; string/select parameters (`PAL`, `SH_TINT`) snap at t=0.5. The interpolation uses a `smoothstep` curve for natural easing.

### Scene Storage

Custom edits are saved to `localStorage` under key `zokna-scenes` as a JSON object. On load, saved values are merged over defaults (per-key override — missing keys fall back to defaults). Scenes can be exported as a `.json` file and re-imported.

---

## Celestial Engine

### Sun Position (`sunPos()`)

Implements a low-precision solar position algorithm (accurate to ~1°):
1. Compute Julian Date from `Date.now()`.
2. Mean longitude and mean anomaly.
3. Ecliptic longitude (including equation of center correction).
4. Obliquity of the ecliptic.
5. Declination and right ascension.
6. Greenwich Mean Sidereal Time → Hour Angle.
7. Altitude (elevation) and Azimuth.

### Moon Position (`moonPos()`)

Similar approach for the moon, also computing phase (0–1) and a brightness factor based on the phase (full moon ≈ 0.9–1.0, new moon ≈ 0.1).

### `updateCelestial()`

Throttled to run at most once every **5 minutes** (`lastCelestialCalc` timestamp guard). The 5-minute interval matches the pace at which the sun actually moves noticeably.

**With a location selected:**
- Compute `sun.el` (elevation) and `sun.az` (azimuth).
- Compare to 10 minutes ago to determine ascending/descending.
- `el > 2°`: daytime — `targetNightMode = 0`, celestial intensity from `sin(el)`.
- `-12° < el ≤ 2°`: twilight — blend `nightMode` 0→1, intensity fades.
- `el ≤ -12°`: night — `targetNightMode = 1`, switch to moon position for angle and brightness.

**Without a location:** approximate from clock hour using a sine curve peaking at noon.

All targets are chased slowly each frame (lerp factor 0.001 or 0.0005) to prevent sudden jumps from flashing the render.

---

## Weather Integration

**Service:** [open-meteo.com](https://open-meteo.com) — free, no API key.

**Endpoint:**
```
GET https://api.open-meteo.com/v1/forecast
  ?latitude={lat}&longitude={lon}
  &current=temperature_2m,cloudcover,is_day
  &hourly=uv_index
  &forecast_days=1&timezone=auto
```

**Fields used:**
- `cloudcover` (0–100%) → `clouds` fraction
- `is_day` (0 or 1) → day/night flag
- `temperature_2m` (°C) → `temp`
- `uv_index[currentHour]` → `uv`

**Derived modifiers applied to light:**
- `intensityMod` = daytime: `max(0.3, 1 - clouds*0.6) * min(1.4, uv/5)` | night: `max(0.15, 0.25 - clouds*0.1)`
- `tempMod` = `((temp - 10) / 40) * 0.6 - clouds * 0.35 + (isDay ? 0.1 : -0.4)`

These modifiers feed into `getLightColor()`:
- `intensityMod` scales the final coin brightness.
- `tempMod` shifts red channel up / blue channel down (warmer) or the opposite (cooler).

**Fetch schedule:** on location select, on manual "↻ Fetch" click, then auto every 30 minutes via `setInterval`.

**Live clock:** once weather data is loaded, a 1-second interval displays `{city} · HH:MM:SS · lat, lon · condition` in the weather status span, using the location's IANA timezone.

### Hardcoded Locations

| Key | City | Lat | Lon | Timezone |
|-----|------|-----|-----|----------|
| `zamosc` | Zamość, Poland | 50.7272 | 23.2449 | Europe/Warsaw |
| `mesa` | Mesa, Arizona | 33.4451 | -111.7052 | America/Phoenix |

---

## Light Colour Palettes

Six named palettes, selected per scene via the `PAL` key:

| Key | Description | RGB |
|-----|-------------|-----|
| `moon` | Cool silver-blue moonlight | 175, 205, 255 |
| `sunrise` | Soft violet-blush | 255, 200, 175 |
| `warm` | Golden amber morning | 255, 218, 120 |
| `noon` | Near-white midday | 255, 252, 225 |
| `late` | Deep amber-orange | 255, 190, 85 |
| `sunset` | Coral-magenta | 255, 145, 100 |

`getLightColor()` takes the base palette colour and applies:
- `tempDrift` (organic colour oscillator, ±0.55 range) shifts red up, blue down for warmth.
- `weatherData.tempMod` adds weather-driven warmth/cool shift.
- `breathVal` × `intensityMod` × `nightDim` × `celestialIntensity` scales the final brightness.

Night floors: `nightDim` floors at 0.35, `celestialIntensity` floors at 0.15 — so coins never fully vanish even at midnight.

---

## Organic Autonomous Animation

Three slow oscillators run every frame independently of camera input:

### Breathing (`breathVal`)
- Phase advances at `0.016 * 0.0028` per frame (~2.8ms/frame × time factor).
- `breathVal = 1.0 + sin(phase)*0.18 + sin(phase*1.618)*0.07` — two incommensurate frequencies for non-repeating rhythm.
- Clamped 0.72–1.28. Dampened at night by `(1 - nightMode * 0.5)`.
- Multiplied into final coin intensity.

### Temperature Drift (`tempDrift`)
- Phase advances at `0.016 * 0.0019` per frame (~1.9ms/frame).
- Three-frequency oscillator: `sin(p)*0.55 + sin(p*0.618)*0.28 + sin(p*2.414)*0.12`.
- Range approximately ±1.0.
- Feeds into `getLightColor()` as RGB warm/cool shift.

### Coin Drift
- Each coin has a `driftX / driftY` velocity (px/sec equivalent).
- Position offset `offX/offY` updated each frame.
- Reverses when offset exceeds 8% of canvas dimension.
- Small random noise added each frame (`±0.001`), clamped to ±0.6.

---

## Installation Presets

Presets control the **surface/grading/slat** parameters only (the `P` object). They do not affect scenes.

### Built-in Presets

| Name | Notable settings |
|------|-----------------|
| `Default` | Plain colour surface, no slats |
| `Morning window` | Slats on, warm amber surface |
| `Z Okna` | Import texture, soft-light blend, strong duotone, slats on |
| `Memory` | Import texture, soft-light blend, duotone, no slats |

### Preset Parameters (`P` object)

**Surface:**

| Key | Default | Description |
|-----|---------|-------------|
| `BGH` | 35 | Background hue (0–360) |
| `BGS` | 14 | Background saturation (0–60) |
| `BGL` | 25 | Background lightness (5–95) |
| `ZOOM` | 1.0 | Imported image zoom scale |
| `MATERIAL` | `plain` | `plain` or `import` |
| `TEX_DEPTH` | 0.10 | Imported texture opacity (0–1) |

**Image grading:**

| Key | Default | Description |
|-----|---------|-------------|
| `BG_BLEND` | `overlay` | Canvas composite op for texture blend |
| `BG_CONTRAST` | 2.5 | CSS filter contrast |
| `BG_BRIGHT` | 1.0 | CSS filter brightness |
| `BG_SAT` | 1.0 | CSS filter saturate |
| `DUO_A` | `#000000` | Duotone shadow colour |
| `DUO_B` | `#ffffff` | Duotone highlight colour |
| `DUO_MIX` | 1.0 | Duotone mix amount (0 = off) |
| `VIGNETTE` | 0.2 | Radial vignette opacity (0–1) |

**Slat overlay:**

| Key | Default | Description |
|-----|---------|-------------|
| `SLAT_ON` | false | Enable slat occlusion |
| `SLAT_ANG` | 85 | Slat angle in degrees |
| `SLAT_W` | 90 | Slat width in pixels |
| `GAP_W` | 18 | Gap width in pixels |
| `SLAT_BLUR` | 40 | Edge softness in pixels |
| `SLAT_DEPTH` | 1.0 | Slat occlusion depth (0–1) |
| `SLAT_OFF` | 45 | Slat pattern phase offset |

### Preset Storage

User presets are stored in `localStorage` under key `zokna-presets`. Built-in presets are never overwritten. The `Default` preset and built-in presets cannot be deleted.

---

## Recording and Capture

### Video Recording (`btn-rec`)
- Uses `HTMLCanvasElement.captureStream(30)` to get a 30fps `MediaStream` from the main canvas.
- `MediaRecorder` with VP9 (preferred), VP8, or generic WebM fallback.
- `videoBitsPerSecond: 8_000_000` (8 Mbps).
- On stop: creates a `Blob`, triggers a download of `zokna-{timestamp}.webm`.

### Frame Grab (`btn-grab`)
- `mainCv.toBlob(..., 'image/png')` → download `zokna-{timestamp}.png`.
- Brief white flash overlay for visual feedback.

---

## Audio

Handled via the Web Audio API.

### Import
- File input accepts `audio/*` and `video/mp4`.
- `arrayBuffer()` → `AudioContext.decodeAudioData()` → `audioBuf`.
- Plays immediately after decode.

### Playback
- `AudioBufferSourceNode` connected through a `GainNode` to `destination`.
- Volume controlled live via `audioGain.gain.value`.
- Loop mode toggled via `audioSource.loop`.
- Progress bar: 500ms interval updates `(elapsed / duration) * 100%`.
- Progress bar click: seek to clicked position (restarts source node at offset).

### AudioContext Resume
A click listener on `document` calls `audioCtx.resume()` if suspended — handles browser autoplay policy.

---

## Transport Controls

| Control | Effect |
|---------|--------|
| Camera select | Switch to a different video input device |
| ⇄ Mirror | Toggle horizontal flip of camera input (default: on) |
| ☯ Invert | Invert motion logic (stillness = presence) |
| ↺ Reset | Zero all motion fields; skip 4 camera frames while exposure settles |
| ⏺ Record | Start/stop WebM recording |
| 📷 Frame | Save PNG snapshot |
| ⬛ Kiosk | Fullscreen + hide all UI |
| ✦ Edit Scenes | Open scene editor modal |

---

## Kiosk Mode

`body.kiosk` CSS class hides all children of `#app` except `#stage`, making the canvas fill the viewport. Keyboard shortcuts:

| Shortcut | Effect |
|----------|--------|
| `Escape` | Exit kiosk |
| `Shift+K` | Toggle kiosk |
| `Shift+S` (in kiosk) | Temporarily show settings for 8 seconds |

`requestFullscreen()` is called on the canvas element when entering kiosk. Exiting fullscreen via browser UI also triggers `exitKiosk()`.

---

## Data Persistence (localStorage)

| Key | Content |
|-----|---------|
| `zokna-presets` | JSON object of user-saved presets (`{name: {P params}}`) |
| `zokna-scenes` | JSON object of all 6 scenes, only keys differing from defaults need to be present |

Nothing else is stored. No cookies, no server communication except the weather API call.

---

## Performance Notes

- **Motion canvas at 40×25** keeps frame-diff and Gaussian blur at ~1000 pixels instead of 285,600 — roughly 285× cheaper.
- **Typed arrays allocated once** (`Float32Array` for all fields, `ImageData` for shadow) — no GC pressure per frame.
- **`slatMap` is precomputed** and only rebuilt on parameter change, not every frame.
- **Cohesion kernel cached** — only recomputed when `L.COHESION` radius changes.
- **`texDirty` flag** prevents texture rebuilds on frames where nothing changed.
- **Celestial engine throttled** to 5-minute intervals — sun position barely changes faster.
- **Coin count = 80** — tuned for Raspberry Pi / iPad. Reduce `NUM_COINS` at the top of the script for slower devices.

---

## Quick Reference: Where to Change Things

| Task | Location |
|------|----------|
| Add a location for weather | `LOCATIONS` object (~line 415) |
| Change coin count | `NUM_COINS` constant (~line 390) |
| Add a built-in preset | `BUILTIN_PRESETS` object (~line 294) |
| Change default surface settings | `DEFAULT_PRESET` object (~line 288) |
| Change scene defaults | `DEFAULT_SCENES` object (~line 307) |
| Add a colour palette | `palColors` object (~line 429) |
| Change scene transition ranges | `getSceneBlend()` function (~line 453) |
| Change render resolution | `W`, `H` constants (~line 359) |
| Change motion detection resolution | `SW`, `SH` constants (~line 359) |
| Change weather refresh interval | `setInterval` in `weather-select` handler (~line 965) |
