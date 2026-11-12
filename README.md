# SEMICO — Soil and Land Monitoring

An Arabic-first, browser-only pilot-farm dashboard with editable land boundaries, simulated soil sensors, and independent diagnostics.

![Runtime: browser](https://img.shields.io/badge/runtime-browser-blue)
![Leaflet: 1.9.4](https://img.shields.io/badge/Leaflet-1.9.4-199900)
![License: unspecified](https://img.shields.io/badge/license-unspecified-lightgrey)

## Project Files

```text
.
├── index.html
├── SEMICO-mockup.html
├── USTHB.jpg
└── README.md
```

- `SEMICO-mockup.html` is the editable source of the application.
- `index.html` is the identical standalone entry page for static hosting.
- `USTHB.jpg` is the original university logo. Its bytes are embedded in both HTML files, so it is not a runtime dependency.

Both pages contain all application CSS, JavaScript, SVG charts, and Leaflet 1.9.4. No framework, build step, package installation, backend, or real device connection is required.

## Run Locally

Open either HTML file directly in a modern browser. JavaScript must be enabled.

For an optional local HTTP preview, run from this directory:

```sh
python3 -m http.server 8000 --bind 127.0.0.1
```

Visit `http://127.0.0.1:8000/`. Stop the server with `Ctrl+C`.

Esri World Imagery and Google Fonts are the only external services used by the application. Without internet, the land overlay, controls, simulation, and charts still work with system fonts and a schematic map background.

## Features

- Default USTHB pilot plot in Bab Ezzouar, Algiers: center 36.7160 N, 3.1828 E, dimensions 300 × 200 m, rotation 18°, and a 2 × 4 monitoring grid.
- Arabic/English switching, RTL/LTR layouts, and light/dark themes.
- Original USTHB branding in the header, loading screen, footer, and browser icon.
- Custom boundaries defined by map points or coordinate entry.
- Concave-safe zone clipping and sensor placement inside the selected land.
- Area-weighted soil moisture and temperature charts, with 1,440 rolling one-minute samples and gaps for missing packets.
- Seeded soil simulation, wet/dry scenarios, sensor silence, and gateway outage scenarios.
- Learned soil references, sustained moisture alarms, and correlated outage suppression.
- Validated browser-local configuration with JSON import/export.

All readings and conditions are simulated. They are not field measurements or agronomic operating guarantees.

## Define the Land

1. Choose **Define land / تحديد حدود الأرض**, or **Edit land** for an existing custom outline.
2. Click perimeter corners in order. Drag a vertex to move it; select one to insert a new point after it or remove it.
3. Alternatively, expand coordinate entry and enter one latitude, longitude pair per line, or a JSON array of pairs.
4. Choose **Apply boundary & restart study** to save the outline and rebuild the study. **Cancel editing** restores the previous study without applying the draft.

Example coordinate format:

```text
36.7150, 3.1810
36.7150, 3.1840
36.7170, 3.1840
36.7170, 3.1810
```

The boundary must be a single, non-self-intersecting outline with 3–64 points. Concave outlines are supported; holes and separate parcels are not. The local model accepts areas of 100–4,000,000 m² and spans up to 5 km in either direction.

Only occupied portions of the monitoring grid are used. Empty cells are omitted, and averages are weighted by the actual clipped zone areas. Geographic areas are approximate, not cadastral or survey-grade measurements.

Applying configuration or a boundary rebuilds the simulation, learned references, and synthetic 24-hour history. It does not preserve the previous runtime alarm feed.

## Configuration

Open settings using the gear, `Ctrl+Shift+K`, or the `#config` URL hash.

The five settings groups are farm geometry, soil properties, diagnostic thresholds, simulation/climate, and interface preferences. All 29 settings are available in the form, with units and numeric bounds.

| Setting | Default | Purpose |
| --- | --- | --- |
| `farm.boundary` | `[]` | Custom latitude/longitude perimeter; empty uses the rectangle |
| `farm.rows`, `farm.cols` | `2`, `4` | Monitoring-grid layout |
| `farm.width`, `farm.height` | `300`, `200` m | Rectangle dimensions; ignored for a custom footprint |
| `farm.rotation` | `18`° | Monitoring-grid orientation |
| `soil.rootDepthMm` | `120` mm | Depth used by the soil-moisture model |
| `soil.fieldCapacityPct` | `60`% | Soil field-capacity reference |
| `soil.drainRate` | `0.0005`/min | Simulated excess-moisture drainage |
| `rules.learnN` | `20` | Valid readings required to learn a reference |
| `rules.moistureLowPct`, `rules.moistureHighPct` | `18`, `70`% | Sustained moisture thresholds |
| `rules.persist`, `rules.hysteresisPct` | `3`, `3`% | Confirmation count and clearing margin |
| `rules.offlineMin` | `3` min | Zone-packet silence timeout |
| `rules.gatewayOfflineSec` | `120` s | Gateway-heartbeat silence timeout |
| `sim.speed` | `300`× | Approximately five simulated minutes per real second |
| `sim.startTime` | `06:00` | Playback start in browser-local time |
| `sim.etMmDay` | `6` mm/day | Simulated evaporation budget |
| `sim.noisePct`, `sim.seed` | `0.15`, `7` | Measurement noise and reproducibility |
| `ui.lang`, `ui.theme` | `ar`, `light` | Persisted language and theme |

Settings are stored under `semico.config.v1` in `localStorage`, scoped to the browser origin. Older configuration files are filtered to supported keys. Invalid saved settings fall back to defaults.

**Export JSON** downloads `semico-config.json`. **Import JSON** fills the form without applying it; use **Save & apply** to commit. Coordinates retain their numeric precision. An empty boundary field restores rectangular mode.

## Architecture

The following plain JavaScript namespaces are inline in each HTML file:

| Module | Responsibility |
| --- | --- |
| `SemConfig` | Defaults, validation, supported-key migration, and persistence |
| `SemI18n` | Complete Arabic/English dictionaries and direction switching |
| `SemFarm` | Projection, polygon validation, clipping, geometry, and map layers |
| `SemSim` | Minute-stepped soil conditions, temperature, faults, and telemetry bus |
| `SemRules` | Telemetry-only references, moisture alarms, and missing-data diagnosis |
| `SemChart` | Accessible SVG charts and pointer/keyboard tooltips |
| `SemApp` | Bootstrap, boundary editor, UI bindings, history, and console actions |

```text
Configuration → bounded monitoring zones → simulated soil packets
                                         ↓
                         independent rules + rolling history
                                         ↓
                         map, charts, readings, and alarm feed
```

Startup silently simulates one full day before exposing the dashboard. The display updates approximately four times per second. Diagnostic logic never receives the simulator's fault flags.

## Console API

After startup, use `window.SemApp.actions` in the browser developer console:

```js
SemApp.actions.pause();
SemApp.actions.injectFault('wet', 'Z1');
SemApp.actions.advance(100);
SemApp.actions.getState();
SemApp.actions.clearFault('wet', 'Z1');
SemApp.actions.setLang('en');
SemApp.actions.setTheme('dark');
SemApp.actions.resume();
```

| Action | Purpose |
| --- | --- |
| `injectFault(type, target)` | Apply `dry`, `wet`, or `sensorOffline` to a zone; `gatewayOffline` targets `GW` |
| `clearFault(type, target)` | Clear a specified condition; `all` clears conditions in the given scope |
| `repair(target)` | Clear all simulated conditions for a zone, `GW`, or `all` |
| `pause()`, `resume()` | Control playback; resume is blocked while editing a boundary |
| `setSpeed(value)` | Set and persist a playback speed from 1–3000 |
| `setLang(value)`, `setTheme(value)` | Apply and persist supported interface preferences |
| `setBoundary(points)`, `clearBoundary()` | Apply a perimeter or restore rectangular mode |
| `applyConfig(config)` | Validate and apply a complete configuration |
| `advance(minutes)` | Deterministic fast-forward of 1–10,080 minutes |
| `ack(id)` | Acknowledge a retained alarm |
| `getState()` | Return a copy of configuration, geometry, readings, references, alarms, and history metadata |

Clearing a wet/dry condition does not instantly restore the previous soil moisture. Moisture alarms clear only after subsequent readings satisfy the clearing threshold.

## Verification and Deployment

There is no committed test runner or build command. Useful manual checks are startup, all language/theme combinations, wet/dry diagnostics, sensor/gateway outages, boundary apply/cancel, settings reset while editing, and JSON round-trips.

Before publishing, ensure the two pages match:

```sh
cp SEMICO-mockup.html index.html
cmp -s SEMICO-mockup.html index.html && echo "Pages are synchronized"
```

Upload `index.html` to a static host. `SEMICO-mockup.html` may also be distributed independently. No server-side application process is needed, and hosted instances still retain settings only in each browser.

## License

No project-wide license file or explicit license for this application's code is supplied. Obtain permission from the project owner before redistribution or reuse. Retain the embedded Leaflet copyright notice, university branding, and applicable imagery attribution.
