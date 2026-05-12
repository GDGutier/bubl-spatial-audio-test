# Bubl Spatial Audio Test — Build Spec

**Read this entire document before writing code.** It contains the full context, the exact deliverable, and the acceptance criteria. Build the deliverable in one HTML file. No build step, no npm, no React. Plain HTML + CSS + JS + MapLibre via CDN.

---

## 1. Context (what this is and why it exists)

The author is an IYA grad student at USC building a project called **Bubl**, a spatial sound-sculpture system. Two surfaces eventually: a Spectacles app (the composer, already shipping, runs a 6-sector kit wheel) and a web app (the renderer, to be built). The web app lets anyone with a phone walk through a real-world location and hear sounds that have been GPS-anchored there, mixed spatially through the Web Audio API as they move.

Before building the real product, the author wants a **disposable prototype** that validates one specific question:

> Does GPS + compass heading + Web Audio `PannerNode` actually produce a convincing spatial audio experience when you walk through a real outdoor location?

The test location is San Pedro Fish Market: `33.7377, -118.2792`. The author will deploy this page, open it on their phone, place 3 audio bubbles on a map of the location, drag them around, set per-bubble radius, hit "Listen," and walk around the parking lot to evaluate whether the audio feels spatially anchored.

**This is a test tool, not the product.** Don't over-engineer it. No persistence, no accounts, no database, no slugs, no future-proofing. State lives in JS memory and dies on refresh. The author will throw it away.

---

## 2. Files in this folder

In the same folder as this spec, there are three audio files. Reference them via relative paths from the HTML file:

- `drumming-jungle-music.wav` — drum loop, ~1MB WAV
- `jungle_birds_canopy_X.mp3` — bird ambience, ~240KB MP3
- `jungle_monkeys_distant_X.mp3` — monkey calls, ~240KB MP3

These three files = three placeable bubbles. The author can pick which sound goes in which bubble via the UI; default assignment is drums → bubble 1, birds → bubble 2, monkeys → bubble 3.

Don't base64-encode the files into the HTML. The author will deploy this folder to Vercel/Netlify/GitHub Pages and the audio files will be served alongside `index.html`.

---

## 3. The deliverable

One file: `index.html`. Self-contained except for two CDN imports (MapLibre GL JS + Google Fonts) and the three audio files in the same folder.

Two distinct modes the user can toggle between via clear UI:

### Mode A: EDIT (default on load)
- MapLibre map centered on `33.7377, -118.2792`, zoom ~19 (close enough to see parking lot detail)
- Three bubbles rendered as draggable map markers, each with a distinct color
- Each bubble has a visible **radius ring** drawn on the map showing its audible falloff distance (rendered as a translucent circle via a MapLibre `circle` layer with `circle-radius` in meters, OR as a GeoJSON polygon if `circle-radius` doesn't support meter units in this MapLibre version — use whichever works)
- Tapping a bubble selects it (visual highlight + larger marker)
- Selected bubble shows a controls panel with:
  - **Radius slider** — 3m to 50m, default 15m, live-updates the ring on the map
  - **Volume slider** — 0.0 to 1.0, default 1.0
  - **Audio file picker** — dropdown of the 3 file names, lets the user swap which sound plays
  - **Mute button** — toggles whether this bubble is active in listen mode
- A "LISTEN" button to switch to Mode B
- Status overlay top-right showing current bubble count and selected bubble id

### Mode B: LISTEN
- Same map, but:
  - Bubbles are no longer draggable (they're locked in place)
  - The user's position appears as a distinct "listener" marker (white dot with green outline + a triangle indicating heading)
  - The map auto-pans to follow the user's GPS position
  - A direction indicator shows the device's compass heading
- Web Audio is live:
  - All three bubbles start looping audio buffers through `PannerNode`s positioned at their map coordinates
  - The `AudioListener` is positioned at the user's smoothed GPS location and oriented by `DeviceOrientationEvent.alpha`
  - As the user walks, the spatial mix updates in real time
- Status overlay shows:
  - GPS accuracy in meters, color-coded: green <8m, amber 8-15m, red >15m
  - Compass heading in degrees
  - Number of bubbles currently audible (distance < radius)
- A "STOP" button to return to Mode A

### Start screen (first thing the user sees)
Before either mode is active, show a full-screen overlay with:
- The word "Bubl" in a large serif font with an accent dot
- Subtitle: "Spatial audio test · San Pedro"
- One big primary button: **"Begin"**
- Tapping Begin must (in this order, inside the same gesture handler):
  1. Call `audioCtx.resume()` to unlock Web Audio
  2. Call `DeviceOrientationEvent.requestPermission()` if it exists (iOS 13+)
  3. Request `navigator.geolocation.getCurrentPosition()` once to trigger the permission prompt
  4. Hide the start screen and reveal Mode A
- If geolocation permission is denied or unavailable, show a clear error message and let the user use Mode A anyway (they can still edit bubbles, they just can't switch to Listen mode)

---

## 4. Technical requirements

### 4.1 Map
Use **MapLibre GL JS v4.7.1** via CDN:
```html
<link href="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.css" rel="stylesheet">
<script src="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.js"></script>
```

Use a free tile source. Recommended: the Carto Voyager or Positron style — they have no API key requirement for moderate use:
```
https://basemaps.cartocdn.com/gl/voyager-gl-style/style.json
```

If that 404s or rate-limits, fall back to OpenFreeMap:
```
https://tiles.openfreemap.org/styles/positron
```

Both are free. Pick one.

### 4.2 Audio
Web Audio API only. No Tone.js, no Howler.

For each bubble, on Mode B entry:
1. `fetch()` the audio file → `arrayBuffer()` → `audioCtx.decodeAudioData()` → `AudioBuffer`
2. Create an `AudioBufferSourceNode`, set `buffer`, set `loop = true`
3. Create a `GainNode`, set `gain.value = bubble.volume`
4. Create a `PannerNode`:
   - `panningModel = 'HRTF'`
   - `distanceModel = 'inverse'`
   - `refDistance = 1.0`
   - `maxDistance = bubble.radius` (the slider value, in meters)
   - `rolloffFactor = 1.5`
   - Position set via `positionX.value`, `positionY.value = 0`, `positionZ.value`
5. Chain: `source → gain → panner → audioCtx.destination`
6. Call `source.start()`

### 4.3 Geolocation → audio coordinates

The map is in lat/lng. The audio engine needs meters. Pick a fixed origin (use the initial map center: `33.7377, -118.2792`) and convert all positions (bubble positions and listener position) to local meters relative to that origin:

```javascript
const ORIGIN_LAT = 33.7377;
const ORIGIN_LNG = -118.2792;
const EARTH_R = 6371000;

function latLngToMeters(lat, lng) {
  const dLat = (lat - ORIGIN_LAT) * Math.PI / 180;
  const dLng = (lng - ORIGIN_LNG) * Math.PI / 180;
  const x = dLng * Math.cos(ORIGIN_LAT * Math.PI / 180) * EARTH_R; // east is +x
  const z = -dLat * EARTH_R;                                        // north is -z (Y-up, right-handed)
  return { x, z };
}
```

For each bubble: convert its lat/lng to (x, z) once when its position changes, set the panner's `positionX` and `positionZ` to those values.

For the listener: convert each GPS update's lat/lng to (x, z), set `audioCtx.listener.positionX.value = x` and `audioCtx.listener.positionZ.value = z`. Set `listener.positionY.value = 1.6` once (ear height; doesn't matter much but conventionally non-zero).

### 4.4 Compass heading → listener orientation

Listen for `deviceorientation` events:
```javascript
window.addEventListener('deviceorientation', (e) => {
  // Prefer webkitCompassHeading on iOS (it's relative to magnetic north)
  // Fall back to alpha (may be relative to device boot orientation on Android)
  const heading = e.webkitCompassHeading != null ? e.webkitCompassHeading : (e.alpha != null ? (360 - e.alpha) : 0);
  const rad = heading * Math.PI / 180;
  const fwdX = Math.sin(rad);
  const fwdZ = -Math.cos(rad);
  audioCtx.listener.forwardX.value = fwdX;
  audioCtx.listener.forwardY.value = 0;
  audioCtx.listener.forwardZ.value = fwdZ;
  audioCtx.listener.upX.value = 0;
  audioCtx.listener.upY.value = 1;
  audioCtx.listener.upZ.value = 0;
});
```

### 4.5 GPS smoothing

Skip Kalman for this prototype. Just use the raw GPS updates with `enableHighAccuracy: true, maximumAge: 0, timeout: 10000` and trust them. If the audio feels jittery during testing, the author can request a filter in v2 of the prototype.

### 4.6 No fallback simulation

If the user is testing on a desktop browser with no real GPS, the page just doesn't work in Listen mode. Don't build a "click on the map to simulate the listener position" debug feature unless the author explicitly asks for it. Keep the scope small.

---

## 5. Visual design

Aim for **utilitarian-warm, dark theme, cartographic-feeling**. This is a test tool, but it shouldn't look like a Bootstrap admin panel.

### Type
- **Display**: Fraunces (Google Fonts) for the brand mark, large headers, and start screen
- **Mono**: JetBrains Mono (Google Fonts) for numeric readouts, status text, button labels (uppercased, letter-spaced)
- No Inter, no Roboto, no Arial

### Color
- Background: deep near-black `#0e0e10`
- Panels: `rgba(20, 20, 23, 0.92)` with `backdrop-filter: blur(20px)` and a 1px white-alpha border `rgba(255,255,255,0.08)`
- Ink: warm off-white `#f5f4ef`
- Accent: lime `#d4ff3a` (used sparingly — start button, listener marker, focus highlights)
- Per-bubble distinct colors: lime `#d4ff3a`, warm orange `#ff8b4a`, cool blue `#5ec3ff`
- Each bubble's marker and its radius ring on the map share its color

### Layout
- Map fills the viewport
- Header pinned top: brand mark left, compact status box right
- Controls pinned bottom: subdued panel with bubble list and selected-bubble controls
- Start screen is a full-screen overlay with a soft radial gradient

### Bubble markers on the map
- Custom HTML markers via MapLibre's `Marker` API with a custom element
- Each marker is a small circle (~28px) in the bubble's color with a subtle outer glow (`box-shadow: 0 0 12px currentColor`)
- A short label above each marker shows its name in mono ("DRUMS", "BIRDS", "MONKEYS")
- Selected bubble grows slightly and gains a brighter halo

### Radius rings on the map
- Render as a translucent circle in the bubble's color, ~10% opacity fill, ~30% opacity stroke
- Implementation: use a MapLibre GeoJSON source per bubble (or one source with three features), and a `fill` + `line` layer
- For meter-accurate circles, generate the polygon points in JS using a 64-vertex circle at the bubble's GPS center with radius converted to lat/lng deltas (the math is straightforward: `dLat = radius_m / 111111; dLng = radius_m / (111111 * cos(lat))`)

### Animations
- Start screen → Mode A: fade + soft scale (300ms)
- Mode A ↔ Mode B: bottom panel slides down/up
- Marker selection: 150ms ease on scale + glow
- No bouncing, no parallax, no scroll-triggered anything. Keep motion functional.

### Typography example
Use `font-variation-settings: "opsz" 144` on the large "Bubl" mark to access Fraunces' display optical size. Use `opsz 24` for the italic subtitle.

---

## 6. State model

In-memory only. One global `state` object is fine:

```javascript
const state = {
  mode: 'start',  // 'start' | 'edit' | 'listen'
  selectedBubbleId: null,
  bubbles: [
    { id: 'b1', name: 'DRUMS',   color: '#d4ff3a', lat: 33.7378, lng: -118.27915, radius: 15, volume: 1.0, muted: false, audioFile: 'drumming-jungle-music.wav' },
    { id: 'b2', name: 'BIRDS',   color: '#ff8b4a', lat: 33.7377, lng: -118.27925, radius: 15, volume: 1.0, muted: false, audioFile: 'jungle_birds_canopy_X.mp3' },
    { id: 'b3', name: 'MONKEYS', color: '#5ec3ff', lat: 33.7376, lng: -118.27915, radius: 15, volume: 1.0, muted: false, audioFile: 'jungle_monkeys_distant_X.mp3' },
  ],
  listener: { lat: null, lng: null, accuracy: null, heading: 0 },
  audio: {
    ctx: null,
    buffers: {},     // keyed by filename
    nodes: {},       // keyed by bubble id: { source, gain, panner }
  },
  watchId: null,     // navigator.geolocation.watchPosition handle
};
```

Mutations are direct. After mutating, call a `render()` helper that updates the affected UI/map elements. No framework needed.

---

## 7. iOS-specific quirks to handle

1. **AudioContext starts suspended.** Must call `audioCtx.resume()` inside the Begin button's click handler. Don't do it on page load.

2. **DeviceOrientationEvent.requestPermission().** On iOS 13+, this exists and must be called from a user gesture. Check `typeof DeviceOrientationEvent.requestPermission === 'function'` before calling. On Android and older iOS, it doesn't exist; just add the event listener directly.

3. **Geolocation permission prompt.** Trigger it from the Begin gesture by calling `getCurrentPosition` once. Then start `watchPosition` once the user grants permission. Don't start the watcher until after the prompt resolves.

4. **HTTPS required.** Geolocation and DeviceOrientation both refuse to work on `http://`. The author needs to deploy this to an HTTPS host (Vercel/Netlify/GitHub Pages all work). Include a note about this somewhere in the code comments.

5. **Wake Lock.** Optional but nice. After the user enters Listen mode, call `navigator.wakeLock.request('screen')` inside a try/catch and ignore failures. This stops the phone from sleeping mid-walk.

6. **Compass heading on iOS uses `webkitCompassHeading`**, which is in degrees clockwise from north (the useful convention). Android uses `alpha`, which is degrees counterclockwise from boot orientation (less useful — `360 - alpha` is a rough approximation but not magnetic). Prefer `webkitCompassHeading` when present.

---

## 8. Acceptance criteria

When the author tests this on their phone at San Pedro Fish Market, the following must all be true:

- [ ] Page loads on phone over HTTPS, start screen visible
- [ ] Tapping Begin transitions smoothly to Edit mode and shows the map centered on the Fish Market with 3 colored bubbles visible
- [ ] Each bubble has a visible radius ring in its color on the map
- [ ] Dragging a bubble updates its position and ring smoothly
- [ ] Tapping a bubble selects it; controls panel updates with that bubble's name and current values
- [ ] Radius slider changes the ring size in real time
- [ ] Audio file picker changes which file the bubble plays in listen mode
- [ ] Tapping LISTEN transitions to Listen mode; map locks; listener marker appears at user's GPS position
- [ ] As the author walks toward a bubble, that bubble's sound gets louder
- [ ] As the author turns their head/body, the stereo image of audible bubbles rotates accordingly (with headphones)
- [ ] As the author walks past a bubble at close range, the sound clearly pans from one side to the other
- [ ] When the author leaves all radii, audio goes silent
- [ ] GPS accuracy indicator updates and color-codes correctly
- [ ] Tapping STOP cleanly returns to Edit mode with no audio leaking
- [ ] Refreshing the page resets state (expected — no persistence)

---

## 9. What to leave out

Explicitly do not build any of these:

- ❌ Saving compositions / persistence of any kind
- ❌ Multiple compositions or a composition switcher
- ❌ User accounts / auth
- ❌ Onboarding tutorial / help text beyond the start screen subtitle
- ❌ Mobile gesture polish beyond what MapLibre + native browser handles
- ❌ Desktop simulation mode (clicking the map to fake a listener position)
- ❌ Kalman filtering on GPS
- ❌ Any reference to Supabase, kits, sectors, the Bubl product, or future plans
- ❌ Service worker / PWA / offline support
- ❌ Analytics
- ❌ Multiple base map styles or theme switcher
- ❌ Bubble add/delete UI (the 3 bubbles are hardcoded; user can only edit them)

If something below is ambiguous and you're tempted to add a feature, default to NOT adding it. The author wants the smallest thing that proves spatial audio works outdoors.

---

## 10. Deliverable summary

One file: `index.html` in the same folder as the three audio files and this spec. Open it locally or deploy the folder to any static host with HTTPS. The author handles deployment.

When you're done, tell the author:
1. That the file is ready
2. The three deployment options (Vercel CLI: `npx vercel`, Netlify drop-zone, GitHub Pages)
3. To test on a phone, outdoors, with headphones, at the actual coordinates
4. To send back any bugs or "this feels wrong" feedback for a v2 pass

Build clean, build small, ship it.
