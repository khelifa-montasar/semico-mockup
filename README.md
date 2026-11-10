# IRRI-DIAG

A browser-only digital twin for exploring drip-irrigation faults before connecting real hardware.

![Status: mockup v0.1](https://img.shields.io/badge/status-mockup%20v0.1-orange)
![Runtime: browser](https://img.shields.io/badge/runtime-browser-blue)
![Leaflet: 1.9.4](https://img.shields.io/badge/Leaflet-1.9.4-199900)
![License: unspecified](https://img.shields.io/badge/license-unspecified-lightgrey)

## Description

IRRI-DIAG simulates a drip-irrigation network and diagnoses problems from flow, pressure and soil readings. Its dashboard combines a farm map, live measurements, alerts with recommended actions, water-use/loss estimates and 24-hour charts.

The application is contained in `IRRI-DIAG-mockup.html:1`. The default demo represents a 240 × 180 m farm near Barika, Batna, with 12 irrigation zones, one hydraulic node and one soil node per zone, plus a pump station. The coordinates are marked for verification in the source.

**This is a simulated prototype, not an ESP32 firmware project or a connected irrigation controller.** No backend, device transport or hardware actuation is implemented. Setting `farm.realNode` only adds an ESP32 label to a simulated node.

## Features

- Interactive Leaflet map with configurable farm geometry, pipes, valves and sensor positions.
- Pump/filter/pipe/dripper simulation, soil moisture and temperature, and seeded measurement noise.
- Automatic watering turns, manual valve commands, pause/resume and 1×, 10×, 60× or 300× playback.
- Fault injection for bursts, clogging, stuck valves, silent hydraulic sensors, main-pipe leaks and dirty filters.
- Diagnosis using learned hydraulic signatures, persistence thresholds, flow balance, pressure loss and soil thresholds; alert acknowledgement and demo repair controls.
- French, English and Arabic interfaces, including RTL layout; JSON configuration import/export and browser-local settings.

## Tech Stack

| Area | Implementation |
| --- | --- |
| UI | HTML, inline CSS, vanilla JavaScript and DOM updates |
| Mapping | Leaflet 1.9.4, with JavaScript and CSS embedded in the HTML |
| Imagery | Esri World Imagery tiles requested over HTTPS |
| Charts | Custom SVG renderer; no charting dependency |
| State | In-memory application state; configuration in `localStorage` |
| Execution | Browser JavaScript, `requestAnimationFrame`, `Intl` and File/Blob APIs |
| Tooling | No package manager, build step, framework or server runtime required |

Simulation, controls and charts run locally. Satellite imagery requires internet access; there is no offline tile cache.

## Architecture

```text
Saved/default configuration -> farm geometry -> simulator
                                                  |
                                     telemetry + valve commands
                                                  |
                                                  v
                                  rule engine + rolling history
                                                  |
                                                  v
                                     map, KPIs, alerts, charts

UI controls -> IrriApp.actions -> simulator / engine / settings
```

The application modules are ordered inline script blocks, not separate files:

| Module | Responsibility | Source |
| --- | --- | --- |
| `IrriConfig` | Defaults, merging, storage and derived hydraulic values | `IRRI-DIAG-mockup.html:979` |
| `IrriI18n` | Translations, time formatting and page direction | `IRRI-DIAG-mockup.html:1101` |
| `IrriFarm` | Convert dimensions, rotation and zone layout to map coordinates | `IRRI-DIAG-mockup.html:1257` |
| `IrriSim` | Network/soil model, schedule, telemetry and injected faults | `IRRI-DIAG-mockup.html:1329` |
| `IrriRules` | Baselines, diagnosis, alerts and water accounting | `IRRI-DIAG-mockup.html:1547` |
| `IrriChart` | SVG time-series charts and hover values | `IRRI-DIAG-mockup.html:1738` |
| `IrriUI` | Map, KPIs, controls, alerts, drawers and configuration panel | `IRRI-DIAG-mockup.html:1829` |
| `IrriApp` | Startup, actions, history and simulation/render loop | `IRRI-DIAG-mockup.html:2415` |

Startup loads settings, constructs the farm, simulator and rule engine, then runs 1,440 one-minute warm-up steps to populate history and learn baselines. Playback starts ten minutes before the configured watering start on the browser's current local date: 05:50 with defaults. Rendering is throttled to roughly four updates per second.

The diagnostic signature is `K = Q / P^x`, compared with a learned per-zone baseline. Injected faults alter the simulator's physics; the rule engine receives readings and valve commands, not the injected-fault list.

History retains 1,440 samples per node, using `null` for missing readings. The engine retains up to 300 alert-log entries. Reloading or applying configuration creates fresh simulator/engine state and regenerates history; only configuration persists.

## Install and Run

Use a modern browser with JavaScript enabled. There are no packages to install. Run the shell commands from the directory containing the HTML files.

Open the application directly through your browser's file picker, or on Linux:

```sh
xdg-open ./IRRI-DIAG-mockup.html
```

For a consistent HTTP origin and browser-storage behavior, optionally serve it with Python 3:

```sh
python3 -m http.server 8000 --bind 127.0.0.1
```

Visit `http://127.0.0.1:8000/IRRI-DIAG-mockup.html`. Stop the server with `Ctrl+C`. Python only serves the static file; it is not an application backend.

No environment variables, credentials, database setup or API keys are configured by this application.

## Configuration

Open settings with the footer gear, `Ctrl+Shift+K`, or by loading `http://127.0.0.1:8000/IRRI-DIAG-mockup.html#config`. The configuration panel is English-only and is not protected by authentication.

- **Save & apply** persists settings and restarts the simulation.
- **Export JSON** downloads the form's configuration as `irridiag-config.json`.
- **Import JSON** fills the form; use **Save & apply** to activate it.
- **Reset defaults** removes saved settings and fills the form with defaults; applying restarts the simulation.
- **Pick center on map** and **Use current map view** update the geographic settings.

Defaults are defined at `IRRI-DIAG-mockup.html:987`:

| Setting | Default | Meaning |
| --- | --- | --- |
| `farm.id` | `"batna01"` | Farm identifier; absent from the settings form |
| `farm.name` | `"Ferme pilote — Barika (Batna)"` | Display name |
| `farm.lat` | `35.4333` | Center latitude, degrees |
| `farm.lng` | `5.2865` | Center longitude, degrees |
| `farm.zoom` | `19` | Maximum zoom when fitting the farm |
| `farm.width` | `240` | Width along drip lines, m |
| `farm.height` | `180` | Height, m |
| `farm.rotation` | `27` | Clockwise rotation, degrees |
| `farm.rows` | `3` | Zone rows |
| `farm.cols` | `4` | Zone columns |
| `farm.drawnLines` | `6` | Drawn drip lines per zone; visual only |
| `farm.soilPerZone` | `1` | Soil nodes per zone |
| `farm.showChem` | `false` | Include simulated EC, pH and N/P/K readings |
| `farm.realNode` | `""` | Node ID to label ESP32; does not connect hardware |
| `hyd.emitterLph` | `2` | Design flow per dripper, L/h |
| `hyd.emittersPerZone` | `4800` | Drippers per zone |
| `hyd.emitterExp` | `0.5` | Exponent in `Q = C·P^x` |
| `hyd.pumpShutoffBar` | `3.0` | Pump pressure with no flow, bar |
| `hyd.pumpDesignBar` | `2.5` | Pump pressure at design flow, bar |
| `hyd.filterCleanBar` | `0.15` | Clean-filter loss at design flow, bar |
| `hyd.mainLossBar` | `0.15` | Main-pipe loss at design flow, bar |
| `hyd.zoneLossBar` | `0.3` | Zone-pipe and valve loss at zone design flow, bar |
| `rules.highPct` | `20` | Signature increase above baseline for burst detection, % |
| `rules.lowPct` | `15` | Signature decrease below baseline for clog detection, % |
| `rules.lowPressPct` | `20` | Pressure decrease below baseline for low-supply detection, % |
| `rules.persist` | `3` | Qualifying evaluations to raise/clear non-offline alerts |
| `rules.ignoreMin` | `3` | Zone-rule settling window after valve-command changes, min |
| `rules.learnN` | `20` | Eligible readings required to learn each zone baseline |
| `rules.closedFlowLpm` | `3` | Allowed flow with a commanded-closed valve, L/min |
| `rules.noFlowPct` | `10` | Low-flow threshold as % of reference flow; absent from form |
| `rules.balancePct` | `8` | Station/zone flow-balance tolerance, % |
| `rules.balanceMinLpm` | `8` | Minimum flow-balance tolerance, L/min |
| `rules.filterDpBar` | `0.5` | Filter-loss threshold normalized to design flow, bar |
| `rules.dryPct` | `15` | Soil moisture below this raises a dry alert, % |
| `rules.wetPct` | `40` | Soil moisture above this raises a wet alert, % |
| `rules.offlineMin` | `3` | Sensor-silence timeout, min; no additional persistence delay |
| `sim.speed` | `60` | Initial simulated minutes per real minute |
| `sim.noisePct` | `1.5` | Relative flow-measurement noise scale, % |
| `sim.startTime` | `"06:00"` | Daily watering start in browser-local time, HH:MM |
| `sim.zonesPerTurn` | `3` | Zones scheduled together |
| `sim.turnMin` | `90` | Duration of each watering turn, min |
| `sim.etMmDay` | `4` | Simulated daily evaporation, mm/day |
| `sim.seed` | `7` | Pseudo-random seed; zero falls back to one |
| `ui.lang` | `"fr"` | Interface language: `fr`, `en` or `ar` |

The defaults derive 160 L/min per zone, 480 L/min station design flow, 1.90 bar zone design pressure and 3,600 m² per zone. These are model design values, not measured operating guarantees.

The form clamps rows/columns to 1–8, soil nodes to 0–3, drawn lines to 1–20 and zoom to 10–20. It does not comprehensively validate hydraulic values or time ranges. Use positive dimensions, flows and durations, physically consistent pump/loss values, and a valid 24-hour start time. Direct API calls bypass form clamping.

Settings use `localStorage` key `irridiag.config.v1`, scoped to the browser origin. Changing host or port creates a separate storage context; direct-file storage behavior is browser-dependent. Storage failures are silently ignored, so JSON export is useful for retaining a configuration.

Import applies visible form fields, not every property in the JSON object. Use the browser API for `farm.id` or `rules.noFlowPct`; change language with the language buttons or `setLang()`. For example, after startup:

```js
const nextConfig = IrriConfig.clone(IrriApp.cfg);
nextConfig.rules.noFlowPct = 12;
IrriApp.actions.applyConfig(nextConfig);
```

To recover from invalid saved settings and fully reload defaults:

```js
IrriConfig.reset();
location.reload();
```

## Usage

The default schedule waters three zones at a time for 90 simulated minutes per turn, starting at 06:00. At 60×, one simulated minute takes approximately one real second while the page is active.

- Click a zone or station to inspect measurements, diagnostic evidence and charts.
- Click a valve to change its command; this switches the simulator to manual mode. Select auto mode to resume the schedule.
- Select a target zone before injecting a zone fault. Some faults need flowing water and learned baselines before they can be diagnosed.
- Acknowledge marks an alert as seen; it does not repair the fault. Repair controls reset simulated faults and clear associated alerts.
- Repair faults before using **Re-learn normal values** to avoid learning a faulty operating condition as the baseline.

To demonstrate a leak, load defaults, wait for startup to finish and run these commands in the browser developer console:

```js
IrriApp.actions.setLang('en');
IrriApp.actions.setSpeed(60);
if (IrriApp.paused) IrriApp.actions.togglePause();
IrriApp.actions.setValve('Z1', true);
IrriApp.actions.inject('burst', 'Z1');
```

Allow several simulated minutes for settling and persistence, then inspect:

```js
IrriApp.eng.last('Z1-H');
console.table(IrriApp.eng.active());
IrriApp.eng.water();
```

Remove the demo faults and return to scheduled watering:

```js
IrriApp.actions.repair('all');
IrriApp.actions.setMode('auto');
```

## API

There is **no HTTP, REST, MQTT or WebSocket API**. The following browser-console interface is exposed through `window.IrriApp`; it is an internal debugging interface, not a versioned SDK. Actions are defined at `IRRI-DIAG-mockup.html:2481`, and engine accessors at `IRRI-DIAG-mockup.html:1721`.

| Call | Behavior |
| --- | --- |
| `IrriApp.actions.setSpeed(speed)` | Set session playback speed; does not persist the setting |
| `IrriApp.actions.togglePause()` | Pause/resume simulated time |
| `IrriApp.actions.setLang(lang)` | Apply and persist `fr`, `en` or `ar` |
| `IrriApp.actions.setMode(mode)` | Select `auto` or `manual` watering |
| `IrriApp.actions.setValve(zoneId, open)` | Set a boolean valve command and enter manual mode |
| `IrriApp.actions.inject(type, target)` | Inject a supported simulated fault |
| `IrriApp.actions.repair(target)` | Repair a zone ID or `all`; use `all` for station faults |
| `IrriApp.actions.learn()` | Reset all zone baselines for relearning |
| `IrriApp.actions.ack(alertId)` | Mark a logged numeric alert ID as acknowledged |
| `IrriApp.actions.applyConfig(config)` | Persist a complete configuration and asynchronously restart the simulation |
| `IrriApp.eng.last(nodeId)` | Return the latest telemetry message, or `undefined` |
| `IrriApp.eng.active()` | Return currently active alerts |
| `IrriApp.eng.log()` | Return retained alerts, newest first, including resolved entries |
| `IrriApp.eng.water()` | Return `{ day, usedL, lostL }` for the simulated local day |
| `IrriApp.eng.status(target)` | Return a zone/station status: `ok`, `learning`, `warning`, `critical` or `offline` |
| `IrriApp.eng.zoneInfo(zoneId)` | Return the hydraulic signature, baseline, ratio and command state |

Supported injection types are `burst`, `clog`, `stuckOpen`, `stuckClosed` and `offline` for zone IDs such as `Z1`; use `mainLeak` or `filter` with target `ST` for station faults. The `offline` injector suppresses that zone's hydraulic messages, not its soil messages.

### Telemetry Contract

Messages are ordinary in-memory objects emitted by `IrriSim.step()`, not network payloads. Every message includes `node`, `type` and `ts` (Unix epoch milliseconds). Definitions start at `IRRI-DIAG-mockup.html:1452`.

| Type | Node ID | Additional fields |
| --- | --- | --- |
| `station` | `ST` | `flow_lpm`, `vol_l`, `pump_bar`, `filter_dp_bar` |
| `hydraulic` | `Z1-H`, etc. | `zone`, `flow_lpm`, `vol_l`, `pressure_bar`, `valve` (`open` or `closed`) |
| `soil` | `Z1-S1`, etc. | `zone`, `moisture_pct`, `soil_temp_c`, `simulated: true`; optional `ec`, `ph`, `n`, `p`, `k` |

`valve` reports the command, not a measured physical valve position. The controller also supplies commands separately through `eng.setCommands(commands, ts)` before ingestion/evaluation. `vol_l` is a cumulative simulator counter; daily water KPIs are calculated separately. Telemetry export and durable telemetry storage are not implemented.

## Tests

No automated tests, test runner, CI configuration, linter or formatter are included. Comments mention Node tests, but the necessary standalone modules and test files are absent. There is no `npm test` or build command.

With the local server running, this optional delivery check requires `curl`:

```sh
curl --fail --head http://127.0.0.1:8000/IRRI-DIAG-mockup.html
```

A successful response checks static-file delivery only; it does not execute JavaScript. Manual browser checks:

- Load defaults and confirm the loading screen clears, 12 zones appear, and the simulated clock advances.
- Switch languages and verify Arabic RTL layout.
- Open a valve, inject a burst, wait for an alert, acknowledge it and repair the zone.
- Inject an offline fault and check the hydraulic node status and chart gaps after the silence timeout.
- Export/import settings, apply them and reload to check configuration persistence and simulation reset.
- Disable network access and reload; simulation and charts should remain usable without satellite imagery.

These are suggested manual checks, not a claim of an executed browser test suite.

## Deployment

Deploy as a **static demo**. No bundling, database migration, environment file or server-side process is needed. Upload the application HTML unchanged, or package it as the site's entry page:

```sh
mkdir -p public
cp IRRI-DIAG-mockup.html public/index.html
python3 -m http.server 8080 --bind 127.0.0.1 --directory public
```

Preview at `http://127.0.0.1:8080/`, then upload the generated `public/index.html` to your static host. The Python server is for local preview, not production hosting. The wiring-reference HTML is optional and not required by the dashboard.

Use HTTPS for hosted access. A strict Content Security Policy must accommodate the inline scripts/styles and imagery requests to `server.arcgisonline.com`. Retain third-party attribution and review imagery-provider terms before publishing. Hosted instances still store settings only in each browser; deployment does not add accounts, shared data or device connectivity.

## Project Tree

```text
.
├── IRRI-DIAG-mockup.html
├── ESP32 soil node wiring (wired + battery) V4.html
└── README.md
```

`ESP32 soil node wiring (wired + battery) V4.html:40` is a separate interactive USB/battery wiring reference, not firmware or a runtime dependency. The `public/` directory in the deployment example is generated manually, not supplied source.

## Limitations and Development Notes

- Backend WebSocket input and real ESP32 integration are future phases mentioned at `IRRI-DIAG-mockup.html:2416`. The UI still calls simulator methods directly, so integration requires more than replacing a network URL.
- Configuration and telemetry lack comprehensive schema, range and timestamp-order validation. Invalid values can produce invalid geometry or non-finite hydraulic calculations.
- The physics and water-loss calculations are demonstration models, not a validated agronomic or safety-critical control system. There is no authentication or physical command acknowledgement.
- Startup, configuration changes and page reloads discard runtime history, alerts, acknowledgements and learned baselines. Browser background throttling can affect playback timing.
- The source references missing `PLAN.md`, `config.js` and `leaflet.js.map` files. CommonJS branches in inline modules do not constitute a runnable Node package; all supplied application code remains in the HTML.

## License

**No project-wide license is specified, and no license file is supplied.** Do not assume permission to redistribute or reuse the application under MIT, Apache or another open-source license; obtain clarification from the project owner.

The embedded Leaflet header credits Vladimir Agafonkin and CloudMade (`IRRI-DIAG-mockup.html:972`). The page credits Esri, Maxar and Earthstar Geographics for imagery (`IRRI-DIAG-mockup.html:963`). Third-party notices and applicable upstream terms are separate from permission to use this application's own code.
