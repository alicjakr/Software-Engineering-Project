# Architecture Diagrams — Reading Guide

This folder holds the architecture views for OsmAnd. The **logical** views describe
*what the software is made of and how the parts depend on each other*, independent of
deployment. The **physical** view is separate (see the bottom of this file).

All logical diagrams were reverse-engineered from the OsmAnd source (the `OsmAnd/`
checkout: modules `OsmAnd-java`, `OsmAnd-shared`, `OsmAnd`, `OsmAnd-api`,
`OsmAnd-telegram`). Every box maps to a real package or class, so a name in a diagram
can be found in the code.

## Read them in this order

The four logical diagrams zoom in progressively — start wide, then drill down.

| # | File | What it answers | Scope |
|---|------|-----------------|-------|
| 1 | `logical_context_detailed.png` | *What does the system talk to?* | OsmAnd as one box + every external system across the World/Machine boundary |
| 2 | `logical_components_detailed.png` | *How is the system structured?* | The internal layers and their aggregate dependencies |
| 3 | `logical_view_navigation.png` | *How does routing & guidance work?* | Drill-down slice: GPS fix → route → voice |
| 4 | `logical_view_mapdata.png` | *How does map data & rendering work?* | Drill-down slice: download → `.obf` → render → search |

### 1. Context — `logical_context_detailed.png`
The outermost view. OsmAnd is a single blue box; everything else is an external
system it exchanges data with, grouped into **Device & OS** (GNSS, TTS, sensors,
BLE/OBD/AIS, Android Auto), **OsmAnd Backend** (download servers, OsmAnd Live,
OsmAnd Cloud, weather service), **OSM Ecosystem** (OSM API, tile servers, online
routing, Mapillary), and **Companion & 3rd-party apps** (OsmAnd Telegram, AIDL
clients). Arrows are the shared phenomena that cross the boundary.

### 2. Component overview — `logical_components_detailed.png`
The inside of the box, as layers. Read **top to bottom** — each layer depends on the
one below it:

- **Presentation** (`net.osmand.plus` UI: MapActivity, widgets, search, settings, Android Auto)
- **Application services** (orchestration: `RoutingHelper`, `ResourceManager`,
  `OsmAndLocationProvider`, download, backup, voice, settings, AIDL)
- **Plugin system** (`OsmandPlugin` + 18 built-in plugins) — extends the layers above
  through defined hooks (layers, widgets, preferences, menu rows)
- **Domain core** (`OsmAnd-java`, pure Java: routing, search, `.obf` reader, render rules)
- **OsmAnd-shared** (Kotlin Multiplatform shared with iOS: GPX engine, device protocols)
- **Native C++ core** (`OsmAndCore`) — tile rendering and heavy routing at runtime; the
  Java engines are the fallback
- **Device storage** — the files and databases on disk

Arrows here are **aggregate** (layer-to-layer). For the real call chains, use the two
slices below. Solid = call/data flow; dashed = plugin extension point.

### 3. Navigation slice — `logical_view_navigation.png`
The routing and turn-by-turn guidance pipeline: `GNSS → OsmAndLocationProvider →
RoutingHelper → RouteProvider`, which picks a route service (offline OsmAnd engine /
online provider / follow a GPX track), then the voice chain (`VoiceRouter →
CommandPlayer → OS TTS` or recorded voice packs).

### 4. Map-data slice — `logical_view_mapdata.png`
The map-data lifecycle: downloading and Live-updating `.obf` files, opening them via
`ResourceManager` / `BinaryMapIndexReader`, the native rendering path with `.render.xml`
style evaluation, search, raster tile overlays, and OsmAnd Cloud sync.

## Conventions (shared across the logical diagrams)

- **Colour = layer.** The same colour means the same layer in every diagram
  (see each diagram's legend for the exact mapping).
- **Solid arrow** = call / data flow. **Dashed arrow** = plugin extension or delegation.
- **`( )` circles** in the slice diagrams are external systems — their full detail is
  in `logical_context_detailed.png`.
- Box labels use real source names (`RoutingHelper`, `BinaryMapIndexReader`,
  `SearchUICore`, …) so they're traceable to the code.

## Superseded / other files

- `logical_old/logical_context_v1.png`, `logical_old/logical_components_v1.png` —
  the earlier, coarser first-draft logical views. Kept for history; **use the
  `*_detailed` and `logical_view_*` files instead.**
- `physical_architecture.drawio.png` — the **physical** architecture (deployment /
  runtime nodes), a separate concern from the logical views above.
