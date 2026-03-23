# China Rail Map

An interactive web map of China's mainland rail network, covering urban rail, national rail, and intercity/metro-area rail. Conventional passenger rail service is not currently included.

Built as a single self-contained HTML file. All spatial data layers are hosted on my personal ArcGIS Online account; no backend is required.

---

## Features

- Urban rail lines and stations across all mainland cities
- National rail lines (high-speed and conventional passenger) with interurban service overlay
- Intercity and metro-area rail lines and stations
- Bilingual interface (Simplified Chinese / English), auto-detected from browser language
- Status filtering: Operating, Under Construction, Planned, Abandoned
- National rail service type filter
- Layer toggles for each of the six data layers
- Basemap switching: Light, Dark, Satellite (Esri World Imagery at reduced opacity)
- Station search with same-station consolidation across lines
- Popups with detailed attributes for lines and stations
- Train consist visualiser in city line popups
- Platform layout diagram in national station popups
- Zoom badge and coordinate display
- Mobile-compatible layout

---

## Data Sources

All feature classes (ArcGIS layer terms) are fetched at runtime from my personal hosted ArcGIS Online feature service:

```
https://services7.arcgis.com/m6uLpqj7MgjPU371/arcgis/rest/services/Mainland/FeatureServer
```


| Layer ID | Content |
|---|---|
| 0 | National rail stations |
| 1 | Urban rail stations |
| 3 | Urban rail lines |
| 4 | National rail lines |
| 36 | Intercity / metro-area rail lines |
| 37 | Intercity / metro-area rail stations |

Data is fetched in full on page load and held in memory. No further network requests are made during map interaction, except for basemap tiles.

This map was orginally developed as an ArcGIS Online web map. Symbology and tooltips contents were orginally inspired by ArcGIS Online webmap. Since laungh of this site, the web map hosted on ArcGIS Online is no longer updated.
```
https://hztranspo.maps.arcgis.com/home/item.html?id=4b97d964f44840a086bd58237351f964
```

---

## Architecture

The entire application is a single HTML file with no build step and no dependencies to install. All libraries are loaded from CDN at runtime.

**Libraries used:**

- Leaflet 1.9 — map rendering and interaction
- Esri Leaflet — basemap tile layer helpers
- Google Fonts (Noto Sans SC, JetBrains Mono) — typography

**Design principles:**

- HTML owns all symbology — no Esri renderers are used
- ArcGIS REST API provides geometry and attributes only
- `effectiveStatus(p)` is the single source of truth for display status
- All data is pre-fetched into memory on load
- Viewport culling is applied only to label DOM creation, not to data fetching
- `zoomend` and `moveend` fire all six draw functions

---

## Status Categories

| Category | Raw values | Default visibility |
|---|---|---|
| Operating | `运营中` | On |
| Under Construction | `建设中`, `未使用` | On |
| Planned | `规划中` | Off |
| Abandoned | `已停工` | Off |
| Hidden (never shown) | `远期规划`, `远景规划` | Never |

`effectiveStatus()` overrides: if a feature has status `运营中` but its `OpenDate` is in the future, it is treated as `建设中` for display purposes.

---

## Line Symbology

**City lines** — weight varies by service type:

| Service Type | Weight |
|---|---|
| Metro Express Heavy Rail | 4 |
| Metro Heavy Rail | 3 |
| Metro Light Rail, Tourist Rail | 2 |
| LRT, APM | 1.5 |
| Streetcar, BRT | 1 |

**National rail lines** — base weight 2, modified by `TrackNumber` / `TotalTrackNumber` via `trackStyle()`.

**Intercity lines** — base weight 2.5, also modified by `trackStyle()`.

**Dash patterns** (white:colored gap ratio 4:6):

| Status | dashArray |
|---|---|
| Under Construction | `6 4` |
| Planned | `3 2` |
| Operating | solid |
| Abandoned | `4 3` |

All dashed lines use `lineCap: butt` and `lineJoin: bevel`. A solid white underline is drawn beneath dashed lines so gaps read white rather than transparent against the basemap.

**White centerline (national and intercity lines only)**

National rail lines with active or pending interurban service display a white centerline stripe. The centerline dash is driven by interurban service status independently of the corridor status:

- Interurban service active: solid centerline
- Interurban service pending (future `OpenInterurbandate`): dashed centerline
- No interurban date (dedicated intercity lines): falls back to corridor status

---

## Station Symbology

All station styles are controlled by `stationStyleFor(status, { weight, radius })`:

| Status | Fill | Outline | Opacity | Notes |
|---|---|---|---|---|
| Operating | white | black | 1.0 | Transfer stations detected by name proximity |
| Under Construction | white | #aaaaaa | 1.0 | |
| Abandoned | #888888 | #555555 | 0.5 | |
| Planned | #cccccc | transparent | 0.8 | |

National stations use a heavier weight (1 operating, 0.75 other) and zoom-dependent radius (2.5 at z>=10, 2 below). City and intercity stations use radius 2.5.

Transfer detection for city stations: if the same station name appears more than once within 300 metres among visible features, all instances are flagged as transfer stations and receive a black outline.

---

## Station Labels

Permanent labels appear at zoom thresholds:

| Layer | Symbol threshold | Label threshold | Dedup radius |
|---|---|---|---|
| National (standard) | z >= 6 | z >= 10 | 1000 m |
| National (interurban service) | z >= 10 | z >= 12 | 500 m |
| Intercity | z >= 10 | z >= 12 | 500 m |
| City | z >= 12 | z >= 14 | 300 m |

Labels are viewport-culled — only stations within the current map bounds receive permanent tooltip DOM nodes, preventing crashes on large datasets.

Label appearance by status:

- Operating: strong background halo using `var(--bg)`
- Under Construction / other: light grey halo (`#aaaaaa`), muted text color

---

## Search

The search box queries all three station layers simultaneously. Results are consolidated using `consolidateStations()`: features with the same name within a proximity threshold are merged into a single result, showing the merged line list and a count badge. Same-name stations in different cities remain as separate results.

Proximity thresholds match label dedup: 300 m city, 500 m intercity, 1000 m national.

---

## Coded Domains

The following ArcGIS coded domains are resolved client-side via lookup tables:

- `车型` (Fleet Type) — `FLEET_TYPE` — 24 entries
- Intercity operator — `INTERCITY_OPERATOR` — 5 entries
- National rail operator — `NAT_OPERATOR` — 22 entries

Unknown codes fall back to displaying the raw code value.

---

## Network and Loading

Data is fetched in six sequential requests, each paginated at 2000 features per page. Each page fetch has a 30-second timeout via `AbortController`, with up to 2 retries on timeout before surfacing an error message in the loading status bar.

The map is not suitable for use from a `file://` URL on some mobile browsers due to CORS behaviour differences. Hosting on any static web server resolves this.

---

## Deployment

The map is a single static HTML file requiring no server-side processing. It can be hosted on any static hosting platform.

**GitHub Pages**

1. Create a public repository
2. Upload the HTML file renamed to `index.html`
3. Enable Pages under Settings, deploying from the main branch root
4. The map is available at `https://zhenghan1994.github.io/China-Rail-Map`

To update, overwrite `index.html` with a new version and commit.

**Alternatives:** Netlify, Cloudflare Pages, or any static file host. A drag-and-drop deploy to Netlify requires no account setup for an initial deployment.

---

## Known Limitations

- Conventional passenger rail service is not included
- The HTML preview panel in Claude.ai does not render the map, as the sandbox blocks outbound fetch requests
- On very slow mobile connections, the six sequential data fetches may time out; the retry logic will surface a clear error message rather than hanging indefinitely
- A debug `console.log('zoom:', map.getZoom())` remains in the `zoomend` handler and should be removed once zoom thresholds are finalised

---

## File Versioning

The file follows a `v{major}.{minor}` naming scheme. The current release is v3.27.
