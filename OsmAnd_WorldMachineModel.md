# World Machine Model — OsmAnd Android
**Version 5.4.0 · Min SDK 24 · Target SDK 35**

---

## 0. Model Conventions

This document follows Jackson's *World and Machine* framework. It separates three kinds of statement, and the test used to classify each one:

- **Domain Assumptions (W)** — *indicative*. Facts about the real world that hold **whether or not the app exists**. The machine relies on them but does not make them true.
- **Requirements (R)** — *optative*. What a stakeholder wants to be true **in the world**. Expressed in the user's vocabulary, qualitatively. **A requirement contains no numbers, parameters, or algorithm names** — if it does, it is really a specification.
- **Specifications (S)** — *optative*. What the **machine** must do at its interface with the world, stated in terms of phenomena shared by both. **Specifications are where numbers and algorithms live.**

The intended relationship is the satisfaction obligation **S ∧ W ⊨ R**: each specification group, *given* the relevant domain assumptions, is what makes the referenced requirement true. Traceability is given in §9.

A fourth, non-WMM category is kept for completeness:

- **Implementation Parameters (P)** — internal constants and tunings that are **not observable at the world↔machine boundary** (memory budgets, byte overheads, bit-field layouts). They carry numbers like specifications but are design choices, not statements about the world, so they are listed separately in §8 and are **not** specifications.

---

## 1. System Purpose

OsmAnd (OSM Automated Navigation Directions) is an open-source map and navigation application for mobile devices. It transforms raw OpenStreetMap (OSM) binary vector data into interactive, offline-capable maps with turn-by-turn navigation, POI search, GPX recording, and OSM contribution workflows.

---

## 2. Actors

| Actor | Role |
|---|---|
| End User | Views maps, navigates, searches, records trips |
| OSM Contributor | Reports bugs, uploads GPX tracks, edits/adds POIs |
| OsmAnd Pro Subscriber | Unlocks live map updates and advanced features |
| External App | Communicates via AIDL / OsmAnd API (`:OsmAnd-api`) |
| OSM Server | Provides note/POI/GPX upload endpoints (`api/0.6/…`) |
| OsmAnd Server | Hosts tile servers, index lists, live-update packages |
| Telegram App | Shares location via `:OsmAnd-telegram` integration |

---

## 3. Domain Assumptions (W) — *indicative*

These are properties of the world the machine depends on. They are the half of the model that lets a specification entail a requirement.

- **W-1 (Positioning):** The device's GNSS/network hardware yields a position fix (latitude, longitude, speed, bearing, altitude); accuracy degrades or is lost in tunnels, indoors, and urban canyons.
- **W-2 (Map fidelity):** OSM-derived OBF data corresponds to real-world geography, roads, and points of interest.
- **W-3 (Network fidelity):** The real road network's topology, turn restrictions, and one-way rules match those encoded in the routing data.
- **W-4 (Speed limits):** Where a road's legal speed limit is tagged in the data, it matches the real posted limit.
- **W-5 (Co-location):** The user physically carries the device along the journey, so the device's position is the user's position.
- **W-6 (Connectivity):** Network connectivity is intermittent or absent across many areas where the app is used.
- **W-7 (Device capability):** The device has a screen, audio output, persistent storage, and orientation/compass sensors.
- **W-8 (OSM servers):** OSM servers accept note, POI, and GPX uploads at the OSM API (`api/0.6`).
- **W-9 (OsmAnd servers):** OsmAnd servers host downloadable map indexes and live-update packages.
- **W-10 (Drift):** The real world and the OSM database both change over time, so any downloaded map gradually diverges from reality.
- **W-11 (Predictable motion):** A moving vehicle's near-future position can be extrapolated from its current speed and heading.

---

## 4. Requirements (R) — *optative, world goals, no numbers*

### 4.1 Map Awareness
- **R-MAP-1:** The user can tell where they are and which way they are facing in the real world.
- **R-MAP-2:** The user can read the map comfortably in both daylight and darkness.
- **R-MAP-3:** The user can see relevant real-world features — points of interest, their own saved places, and recorded tracks — in geographic context.
- **R-MAP-4:** The user can read place names in a script they understand.
- **R-MAP-5:** The user can supplement the vector map with photographic/aerial imagery of the real terrain.

### 4.2 Offline Capability
- **R-OFF-1:** The user can view maps and navigate in areas with no network connectivity.
- **R-OFF-2:** The user can obtain coverage for only the regions they care about, within their device's storage.
- **R-OFF-3:** The user can rely on map data that reflects the current state of the real world.

### 4.3 Navigation
- **R-NAV-1:** The user can reach a chosen destination without needing to read the map while travelling.
- **R-NAV-2:** The user can navigate by car, by bicycle, or on foot.
- **R-NAV-3:** The user can plan a journey through one or more chosen stopovers.
- **R-NAV-4:** The user still receives a usable route to their destination after deviating from the planned one.
- **R-NAV-5:** The user is made aware when they are travelling faster than the legal limit for their location.

### 4.4 Search
- **R-SRH-1:** The user can find a real-world place from what they already know about it — its name, address, type, or coordinates.
- **R-SRH-2:** The user can discover real-world points of interest near them.

### 4.5 Trip Recording
- **R-TRP-1:** The user can keep a record of a journey they have taken.
- **R-TRP-2:** The user can record a journey without keeping the screen on or the app in the foreground.

### 4.6 OSM Contribution
- **R-OSM-1:** The contributor can report real-world map errors.
- **R-OSM-2:** The contributor can share recorded journeys with the OSM community.
- **R-OSM-3:** The contributor can improve real-world POI data, even while offline.

### 4.7 Public Transport
- **R-PUB-1:** The user can find public transport stops and the lines that serve them.
- **R-PUB-2:** The user can plan a journey by public transport.

### 4.8 Extensibility
- **R-PLG-1:** The user can tailor the app to only the capabilities they need.
- **R-PLG-2:** Third-party applications can build on OsmAnd's navigation and map data.

### 4.9 Cross-Cutting (Non-Functional)
- **R-NFR-1 (Privacy):** The user's location and usage history remain private and are not disclosed to third parties without consent.
- **R-NFR-2 (Resource economy):** Continuous navigation and background recording do not exhaust the device's battery or storage unreasonably.
- **R-NFR-3 (Portability):** The user can rely on identical core behaviour regardless of the platform the core logic runs on.
- **R-NFR-4 (Eyes-free use):** A user who cannot look at the screen can still be guided to their destination by sound alone.

---

## 5. Machine Interface — Shared Phenomena

The boundary at which specifications are legitimate. Internal module structure is in §11; this is the world↔machine edge only.

**Sensed by the machine (world → machine)**
- GNSS/network location fixes (lat, lon, accuracy, speed, bearing, altitude)
- Device orientation / compass heading
- Touch and gesture input on the screen
- System clock and time of day (day/night, live-update windows)
- Stored OBF/GPX files on device storage
- Network responses from tile servers, OSM API, and OsmAnd servers

**Actuated by the machine (machine → world)**
- Rendered map and overlays on the screen
- Spoken voice guidance and audible alarms via audio output
- GPX files written to storage
- Notes, POIs, and GPX uploaded to OSM servers
- Map and live-update files downloaded to storage
- Data and callbacks delivered to external apps via the AIDL API

---

## 6. Specifications (S) — *optative, machine behaviour, numeric/algorithmic*

Each group names the requirement(s) it serves. The intended argument is **S ∧ W ⊨ R**.

### 6.1 Map Rendering — *(→ R-MAP-1..5)*
| Attribute | Value |
|---|---|
| Standard tile size | 256 × 256 px |
| Map zoom range (UI) | 1 – 17 |
| Map zoom range (layers) | 1 – 21 |
| Coordinate grid max zoom | 22 |
| Detailed map minimum zoom (OBF) | 9 |
| Place-name rendering | English / local / phonetic script *(R-MAP-4)* |
| Day/night switching | automatic, driven by clock + sun position *(R-MAP-2)* |
| Map orientation | follow compass **or** direction of motion *(R-MAP-1)* |
| Raster online maps | available as alternative or overlay *(R-MAP-5, R-OFF-1)* |
| Satellite imagery (Bing) | optional layer *(R-MAP-5)* |

### 6.2 Offline Maps, Downloads & Updates — *(→ R-OFF-1..3)*
| Attribute | Value |
|---|---|
| Map storage format | OBF compact binary vector format |
| Download granularity | per-country and per-region |
| Road-network-only download | available as smaller alternative |
| Update cadence | at least once per month *(R-OFF-3)* |
| Live update morning window start | 08:00 |
| Live update night window start | 21:00 |
| Free build download limit | 7 map files |
| Download retry attempts | 15 |
| Timeout between retries | 8,000 ms |
| Max tile download errors per timeout | 50 |

### 6.3 Routing Algorithms — *(→ R-NAV-1, R-NAV-2, R-NAV-3)*
| Attribute | Value |
|---|---|
| Primary algorithm | Bidirectional A\* (BinaryRoutePlanner) |
| Long-distance algorithm | Hierarchical Highway routing (HHRoutePlanner) |
| Calculation modes | online (server) and fully offline *(R-OFF-1)* |
| Supported profiles | car, bicycle, pedestrian (+ others) *(R-NAV-2)* |
| Intermediate waypoints | supported *(R-NAV-3)* |
| Car shortest-path default speed | 55 km/h (≈ 15.28 m/s) |
| Bicycle shortest-path default speed | 15 km/h (≈ 4.17 m/s) |
| Min straight-line routing distance | 50,000 m |
| GPX point approximation default | 50 m |
| GPX min intermediate distance | 10 m |
| GPX nearest-point extra search distance | 300 m |
| Route segment sampling points | 11 |

### 6.4 Route Recalculation — *(→ R-NAV-4)*
| Attribute | Value |
|---|---|
| Deviation radius (intermediate reroute) | 3,000 m |
| Max bearing deviation for route snapping | 45° |
| Recalculation count triggering full recalculate | 3 |
| Recalculation interval window | 2 min (120,000 ms) |
| Route cache radius | 100,000 m |

### 6.5 Voice & Lane Guidance — *(→ R-NAV-1, R-NFR-4)*
| Attribute | Value |
|---|---|
| Announce states | TURN\_NOW(0), TURN\_IN(1), PREPARE\_TURN(2), LONG\_PREPARE\_TURN(3), SHORT\_ALARM(4), LONG\_ALARM(5), SHORT\_PNT\_APPROACH(6), LONG\_PNT\_APPROACH(7) |
| Lane guidance cut-off (unspoken turn) | 800 m |
| Lane guidance cut-off (spoken turn) | 1,200 m |
| Positioning tolerance | 12 m |

### 6.6 Speed & Hazard Warning — *(→ R-NAV-5)*
| Attribute | Value |
|---|---|
| Speed limit indicator | optional, with overspeed warning |
| Speeding announcement interval | 120 s (120,000 ms) |
| Speeding announcement cancel window | 30 s (30,000 ms) |
| Speeding announcement waiting period | 5 s (5,000 ms) |
| Upcoming-tunnel detection distance | 250 m |

### 6.7 Location Processing — *(→ R-MAP-1, supports R-NAV-1)*
| Attribute | Value |
|---|---|
| Kalman filter | enabled |
| Accuracy threshold (GPX & routing) | 50 m |
| Lost-location check delay | 18,000 ms |
| GPS → network fallback inhibit period | 12,000 ms |
| Location simulation start delay | 2,000 ms |

### 6.8 Auto-Zoom — *(→ R-NAV-1)*
| Attribute | Value |
|---|---|
| Auto-zoom activation speed | 7 km/h |
| Lookahead horizon | 45 s at current speed *(W-11)* |
| Zoom change rate | 0.1 zoom / s |
| Min zoom animation duration | 1,500 ms |
| Focus pixel ratio X | 0.5 (horizontal center) |
| Focus pixel ratio Y | 1/3 (upper third) |

### 6.9 POI / Nearby Search — *(→ R-SRH-1, R-SRH-2)*
| Attribute | Value |
|---|---|
| Search keys | address, POI type, coordinates, name |
| Max nearby POI results | 10 |
| Initial search radius | 250 m |
| Maximum search radius | 1,000 m |
| Radius expansion factor | ×2 per iteration |

### 6.10 Trip Recording — *(→ R-TRP-1, R-TRP-2)*
| Attribute | Value |
|---|---|
| Record target | local GPX files |
| Background recording | via foreground service, screen off |
| Live readout | current speed and altitude |
| Recording accuracy threshold | 50 m |

### 6.11 Route Statistics & Slope Analysis — *(→ R-MAP-3, R-TRP-1)*
| Attribute | Value |
|---|---|
| Elevation step (H\_STEP) | 5 m |
| Slope approximation window | 100 m |
| Min incline bucket | −101% |
| Max incline bucket | +100% |
| Min divided incline | −20% |
| Max divided incline | +20% |
| Incline bucket step | 4% |

### 6.12 OSM Contribution — *(→ R-OSM-1, R-OSM-2, R-OSM-3)*
| Attribute | Value |
|---|---|
| Map-bug reporting | OSM Notes from within the app |
| GPX upload | to openstreetmap.org |
| POI editing | add/edit, offline-queued or immediate online upload |
| Mapper promo threshold | 30 changes within 60 days |

### 6.13 Public Transport — *(→ R-PUB-1, R-PUB-2)*
| Attribute | Value |
|---|---|
| Stop display | bus, tram, train with line names |
| Transit routing | multi-leg, multi-mode |

### 6.14 Plugins & API — *(→ R-PLG-1, R-PLG-2)*
| Attribute | Value |
|---|---|
| Pluggable features | Nautical, Ski, Weather, Contour lines, Mapillary, Wikipedia, OSM Editing |
| Plugin control | enable/disable at runtime |
| External API | public AIDL API: control navigation, query map data, receive location updates |

---

## 7. Environment & Platform Constraints

Indicative facts about the build/runtime environment. Not behavioural specifications.

| Attribute | Value |
|---|---|
| App version | 5.4.0 (versionCode 5399) |
| Minimum Android SDK | 24 (Android 7.0) |
| Target / Compile SDK | 35 (Android 15) |
| JVM target | Java 17 |
| Kotlin version | 2.1.10 |
| Android Gradle Plugin | 8.7.3 |
| Supported ABIs | armeabi-v7a, arm64-v8a, x86, x86\_64 |
| Render core | OpenGL ES ≥ 3.0 (via OsmAndCore C++ + JNI) |
| Fallback render | legacy software renderer (no OpenGL) |
| Build flavours | nightlyFree, androidFull, gplayFree, gplayFull, huawei |

Project policy constraints:
- **No Gradle builds** may be triggered by automated agents.
- New UI code must follow **Google Material Design** and use `BaseOsmAndFragment`.
- All user-visible strings must be registered in `OsmAnd/res/values/strings.xml`.
- Core logic must remain in `:OsmAnd-java` or `:OsmAnd-shared` to stay Android-independent *(→ R-NFR-3)*.
- Optional features must use the **plugin architecture** (`OsmandPlugin` subclasses) *(→ R-PLG-1)*.
- Logging must use `PlatformUtil.getLog()`, not `android.util.Log`, for portability *(→ R-NFR-3)*.

---

## 8. Implementation Parameters (P) — *internal constants, NOT specifications*

These carry numbers but are invisible at the world↔machine boundary; they are design/tuning choices, not statements about the world or observable behaviour. Listed for completeness only.

| Parameter | Value | Locus |
|---|---|---|
| Default routing memory limit | 30 MB | routing engine |
| Default native memory limit | 256 MB | OsmAndCore |
| HH max cluster-routing points | 150,000 | HHRoutePlanner |
| HH max incremental cost correction | 10.0 | HHRoutePlanner |
| Road queue memory overhead | 220 bytes/road | routing engine |
| Road visited memory overhead | 150 bytes/road | routing engine |
| Requests before location check | 100 | location provider |
| Kalman coefficient | 0.04 | location filter |
| Download buffer size | 32,256 bytes | download service |
| Raster tile default size estimate | 0.012 MB/tile | tile cache |
| OpenGL ES minimum major version | 3 | render core |
| OSM Edits DB version | 6 | persistence |
| OSM Bugs DB version | 1 | persistence |
| OBF ID bit shift | 6 | OBF codec |
| OBF multipolygon ID shift | 43 bits | OBF codec |
| OBF non-split existing ID shift | 41 bits | OBF codec |
| OBF propagated node ID shift | 50 bits | OBF codec |
| OBF propagated node bits | 11 bits | OBF codec |
| OBF duplicate split constant | 5 | OBF codec |

---

## 9. Traceability (R ← S, given W)

| Requirement | Satisfying specifications | Relies on |
|---|---|---|
| R-MAP-1 | 6.1, 6.7 | W-1, W-5, W-7 |
| R-MAP-2 | 6.1 | W-7 |
| R-MAP-3 | 6.1, 6.11 | W-2 |
| R-MAP-4 | 6.1 | W-2 |
| R-MAP-5 | 6.1 | W-2, W-6 |
| R-OFF-1 | 6.1, 6.2, 6.3 | W-6, W-9 |
| R-OFF-2 | 6.2 | W-7, W-9 |
| R-OFF-3 | 6.2 | W-9, W-10 |
| R-NAV-1 | 6.3, 6.5, 6.7, 6.8 | W-1, W-3, W-5, W-11 |
| R-NAV-2 | 6.3 | W-3 |
| R-NAV-3 | 6.3 | W-3 |
| R-NAV-4 | 6.4 | W-1, W-3 |
| R-NAV-5 | 6.6 | W-1, W-4 |
| R-SRH-1 | 6.9 | W-2 |
| R-SRH-2 | 6.9 | W-1, W-2 |
| R-TRP-1 | 6.10, 6.11 | W-1, W-5 |
| R-TRP-2 | 6.10 | W-7 |
| R-OSM-1 | 6.12 | W-8 |
| R-OSM-2 | 6.12 | W-8 |
| R-OSM-3 | 6.12 | W-8 |
| R-PUB-1 | 6.13 | W-2 |
| R-PUB-2 | 6.13 | W-2, W-3 |
| R-PLG-1 | 6.14, §7 policy | — |
| R-PLG-2 | 6.14 | — |
| R-NFR-1 | (offline-first design; no spec mandates upload) | W-6 |
| R-NFR-2 | 6.10 (foreground service), 6.7 | W-7 |
| R-NFR-3 | §7 policy constraints | — |
| R-NFR-4 | 6.5, 6.6 | W-7 |

> **Open gaps:** R-NFR-1 (privacy) and R-NFR-2 (resource economy) have no quantified specifications yet — there is no measured battery/storage budget or data-handling spec to point to. These are candidate specifications to add.

---

## 10. Data Model (Key Entities)

```
OsmandApplication
├── OsmandSettings          (all user preferences, serialised to SharedPreferences)
├── OsmAndLocationProvider  (GPS/network position, Kalman filtering)
├── RoutingHelper           (active navigation state, voice router)
│   ├── VoiceRouter         (AnnounceTimeDistances, VoiceCommandPending)
│   └── RouteRecalculationHelper
├── ResourceManager         (OBF index lifecycle, tile cache)
│   └── BinaryMapIndexReader (reads .obf files: address, POI, routing, transport)
├── DownloadService         (map & resource downloads)
├── NavigationService       (background foreground service)
├── OsmandAidlService       (AIDL API for external apps)
└── PluginManager
    └── OsmandPlugin[]      (OsmEditing, Nautical, Ski, Weather, Wikipedia, …)
```

**Map Objects**
- `BinaryMapDataObject` — vector map feature (ways, nodes, relations)
- `RouteDataObject` — road segment with speed/restriction attributes
- `Amenity` — POI with tags, coordinates, opening hours
- `FavouritePoint` — user-saved location
- `WptPt` — GPX waypoint (lat, lon, ele, time, speed)
- `TransportStop` / `TransportRoute` — public transit primitives

---

## 11. Sub-System Map

```
┌─────────────────────────────────────────────────────┐
│                    :OsmAnd (Android App)              │
│  MapActivity ──► OsmandMapTileView ──► Map Layers    │
│  RoutingHelper  VoiceRouter  NavigationService       │
│  DownloadService  PluginManager  OsmandSettings      │
└───────────┬──────────────────────┬──────────────────┘
            │ (pure Java, no Android)   │ (KMP shared)
┌───────────▼──────────┐  ┌────────────▼─────────────┐
│   :OsmAnd-java        │  │   :OsmAnd-shared (KMP)   │
│  BinaryRoutePlanner   │  │  commonMain: serialise,  │
│  HHRoutePlanner       │  │  coroutines, datetime    │
│  RoutingConfiguration │  │  androidMain / iosMain / │
│  BinaryMapIndexReader │  │  jvmMain: SQLite, I/O    │
│  SearchUICore         │  └──────────────────────────┘
│  RouteStatisticsHelper│
└───────────────────────┘
            │
┌───────────▼──────────┐
│  OsmAndCore (C++ JNI) │
│  OpenGL ES 3 render   │
│  libOsmAndCoreWithJNI │
└───────────────────────┘
```

---

## 12. External Dependencies (Selected)

| Library | Purpose |
|---|---|
| Rhino (JavaScript) | Voice guidance scripts |
| JTS Topology Suite | Geometric operations (polygon, intersection) |
| SQLDelight | Type-safe SQLite (shared module) |
| Picasso | Image loading |
| MPAndroidChart (custom) | Elevation / route charts |
| kotlinx-serialization | JSON (KMP shared) |
| okio | Cross-platform I/O (KMP shared) |
| stately | Concurrent collections (KMP shared) |
| desugar\_jdk\_libs 2.1.3 | Java API back-porting to SDK 24 |
| Huawei HMS IAP 6.4.0.301 | Huawei in-app purchases |
