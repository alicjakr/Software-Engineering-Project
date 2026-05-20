# Requirements Specification — OsmAnd
## World/Machine Model (WMM)

**Project:** OsmAnd (OSM Automated Navigation Directions)  
**Repository:** https://github.com/osmandapp/OsmAnd  
**Version:** 1.0  
**Date:** April 2026  

---

## 1. System Overview

OsmAnd is an open-source, cross-platform (Android & iOS) map viewing and navigation application. It provides offline-capable routing, GPS tracking, map rendering, POI search, and trip recording, all built on top of OpenStreetMap (OSM) data. The system is developed in Java/Kotlin (Android) and available on Google Play, F-Droid, Huawei AppGallery, Amazon Appstore, and the Apple App Store.

The central design goal is **complete offline functionality**: all core features (routing, rendering, search, POI lookup) must work without any internet connection, using pre-downloaded binary map files (`.obf` format).

---

## 2. World / Machine Boundary

The **World/Machine Model** (Michael Jackson, 1995) separates the software system (the Machine) from the real-world environment it interacts with (the World). The boundary between them is defined by **shared phenomena** — events and data that both sides can observe or act upon.

```
┌─────────────────────────────────────────────────────────────────────┐
│                             THE WORLD                               │
│                                                                     │
│  [User]  [GPS Satellites]  [Road Network]  [OSM Servers]            │
│  [Device Filesystem]  [Text-to-Speech Engine]  [OsmAnd Cloud]       │
│                                                                     │
│           ↕ shared phenomena (interface)                            │
│ ─────────────────────────────────────────────────────────────────── │
│                            THE MACHINE                              │
│                                                                     │
│  [Map Renderer]  [Routing Engine]  [GPS/Location Manager]           │
│  [Search Engine]  [Map Data Manager]  [GPX Tracker]                 │
│  [Plugin System]  [Settings & Profiles]  [UI Layer]                 │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Actors and External Entities

| Actor / Entity | Type | Description |
|---|---|---|
| **User** | Human actor | Interacts with the app via touch gestures, buttons, and voice commands |
| **GPS Satellite System** | Hardware/external | Provides real-time geographic coordinates via the device's GPS chip |
| **OpenStreetMap (OSM)** | External data source | The crowd-sourced map database; provides raw map data for `.obf` compilation |
| **OsmAnd Map Servers** | External system | Hosts pre-compiled `.obf` map files and OsmAnd Live update diffs for download |
| **OsmAnd Cloud** | External system | Cloud storage for syncing user data (favourites, tracks, settings) across devices |
| **Device Filesystem** | Hardware/OS | Stores offline map files, GPX tracks, cached tiles, and user data |
| **Text-to-Speech (TTS) Engine** | OS service | Synthesises voice navigation instructions |
| **Bing / Online Tile Servers** | External system | Optional sources for online satellite and raster tile overlays |
| **OSM API** | External system | Accepts user contributions (POI additions, bug reports, GPX uploads) |
| **Telegram** | External system | OsmAnd Telegram plugin: shares live location with contacts |

---

## 4. Domain Phenomena (Shared Events at the Boundary)

These are the observable events and data flows that cross the World/Machine boundary:

| # | Phenomenon | Direction | Description |
|---|---|---|---|
| P1 | User taps "Navigate" | World → Machine | Initiates route calculation |
| P2 | GPS fix received | World → Machine | Device location updated with lat/lon/accuracy |
| P3 | Route instructions displayed | Machine → World | Turn-by-turn directions shown on screen |
| P4 | Voice instruction emitted | Machine → World | TTS announces manoeuvre ("In 200m, turn right") |
| P5 | Map file download requested | Machine → World | App fetches `.obf` from OsmAnd servers |
| P6 | Map file stored on device | World → Machine | Filesystem delivers binary map data for indexing |
| P7 | User deviates from route | World → Machine | GPS position falls outside calculated route corridor |
| P8 | Re-routing triggered | Machine → World | New route is calculated and displayed |
| P9 | User searches for POI | World → Machine | Search query submitted to local search engine |
| P10 | Search results displayed | Machine → World | Matching places shown on map and in list |
| P11 | GPX track recorded | Machine → World | Track written to device filesystem |
| P12 | User uploads GPX to OSM | Machine → World | Track submitted via OSM API |
| P13 | User edits POI | World → Machine | User submits OSM note or POI edit |
| P14 | Speed limit exceeded | World → Machine | GPS speed surpasses road limit stored in map data |
| P15 | Speed warning displayed | Machine → World | Visual/audio alert shown to user |

---

## 5. World

This section describes the real-world environment in which OsmAnd operates — the people, physical systems, infrastructure, and data sources that exist independently of the software and that the software must interact with or respond to.

### 5.1 Users and Their Context

OsmAnd's primary user is a **mobile device user navigating in the physical world** — on foot, by bicycle, by car, or using public transport. Users operate the app in diverse conditions: outdoors with variable GPS signal quality, while driving (limiting screen interaction), in areas without mobile internet coverage, and across a wide range of hardware capabilities from low-end Android devices to high-end smartphones. This context makes offline capability, battery efficiency, and minimal-interaction voice guidance first-class real-world concerns, not just technical preferences.

Users range from casual map browsers to professional field workers (surveyors, emergency services, outdoor enthusiasts). A subset of users actively contributes back to the world through OSM: adding POIs, uploading GPX tracks, and reporting map errors.

### 5.2 The Physical Road Network and Geographic World

OsmAnd operates over a representation of the **real physical world**: roads, paths, buildings, waterways, elevation, and points of interest. This world is inherently imperfect — roads change, businesses open and close, construction alters routes. The map data the app uses is a snapshot of this world, compiled from OpenStreetMap contributions, and is therefore always partially out of date. The app must handle discrepancies gracefully (e.g., re-routing when a road in the data does not match reality).

The physical world also imposes **GPS constraints**: signal accuracy degrades indoors, in urban canyons, and under dense tree cover. The app must tolerate noisy or intermittent location data.

### 5.3 GPS Satellite Infrastructure

The device's location is determined by the **Global Navigation Satellite System (GNSS)** — primarily GPS, with optional GLONASS, Galileo, and BeiDou support depending on device hardware. The satellite system is entirely outside OsmAnd's control. It delivers a stream of position fixes to the device's OS location API, which the app consumes. Fix accuracy typically ranges from 3–15 metres in open sky conditions but may degrade significantly in urban or indoor environments.

### 5.4 OpenStreetMap Ecosystem

OsmAnd's map content is entirely derived from **OpenStreetMap (OSM)** — a global, crowd-sourced geographic database maintained by millions of volunteers. OSM data is exported, processed, and compiled into OsmAnd's binary `.obf` format by the **OsmAnd MapCreator** tool (a separate project). The OSM ecosystem also provides the **OSM API**, through which OsmAnd allows users to contribute edits (notes, POIs, GPX uploads) back to the shared map.

Map data quality varies by region — densely populated areas in Europe and North America have near-complete coverage, while some regions of Central Asia and Central Africa are still being mapped. OsmAnd has no control over data quality; it can only render what OSM provides.

### 5.5 OsmAnd Server Infrastructure

The **OsmAnd map servers** host pre-compiled `.obf` regional map files, which are updated at least monthly and made available for in-app download. The **OsmAnd Live** service provides hourly incremental map diffs for subscribers. **OsmAnd Cloud** provides cross-device synchronisation for user data (favourites, tracks, settings, profiles). These services are external to the application itself; the app must function fully without them when offline.

### 5.6 Device Hardware and OS

OsmAnd runs on **Android** (primary) and **iOS** mobile devices. The relevant hardware environment includes the GPS chip, device filesystem (internal storage and optional external SD card), display (varying resolutions and sizes), compass sensor, and the device speaker/headphone output for voice guidance. The operating system provides location APIs, the TTS engine, filesystem access, and background process management. Battery life is a significant real-world constraint: navigation sessions can last several hours, and the app must minimise power consumption, particularly when the screen is off.

### 5.7 Voice and Audio Environment

Turn-by-turn instructions are delivered to the user through the device's **Text-to-Speech (TTS) engine** (OS-provided) or via pre-recorded audio files bundled with the app. The audio must be comprehensible while the user is driving, cycling, or walking — meaning timing (announced sufficiently in advance) and volume/clarity are real-world concerns that constrain how voice guidance is designed.

### 5.8 Online Tile and Satellite Providers

For users who optionally choose online map overlays, **Bing Maps** (satellite imagery) and various community tile servers (OpenTopoMap, Thunderforest, etc.) serve raster tiles over the internet. These are entirely external and optional; their availability is not guaranteed, and OsmAnd must not depend on them for core functionality.

### 5.9 Telegram (Live Location Sharing)

The **OsmAnd Tracker** plugin integrates with **Telegram** to broadcast the user's real-time GPS location to selected contacts via the Telegram messaging network. This is an optional social/safety feature dependent on Telegram's external infrastructure and the user's Telegram account.

---

## 6. Requirements

### 6.1 Functional Requirements

#### 6.1.1 Navigation & Routing

| ID | Requirement |
|---|---|
| FR-NAV-01 | The system **shall** calculate routes between a start and destination point for car, bicycle, and pedestrian profiles. |
| FR-NAV-11 | The system **shall** perform all core navigation functions (routing, voice guidance, re-routing) without any internet connection. |
| FR-NAV-02 | The system **shall** calculate routes entirely offline using pre-downloaded `.obf` map data. |
| FR-NAV-03 | The routing engine **shall** use a bidirectional A* algorithm based on fastest time (distance / (speed × priority) + penalties), configurable via `routing.xml`. |
| FR-NAV-04 | The system **shall** support intermediate waypoints on a route. |
| FR-NAV-05 | The system **shall** provide turn-by-turn voice guidance using either recorded audio files or the device TTS engine. |
| FR-NAV-06 | The system **shall** provide optional lane guidance, street name display, and estimated time of arrival (ETA). |
| FR-NAV-07 | The system **shall** automatically re-route when the user deviates from the current route. |
| FR-NAV-08 | The system **shall** allow users to avoid specific road types (motorways, toll roads, ferries, unpaved roads). |
| FR-NAV-09 | The system **shall** support public transport routing (bus, tram, metro, train). |
| FR-NAV-10 | The system **shall** display speed limit information and warn the user when the limit is exceeded. |

#### 6.1.2 Map Rendering & Viewing

| ID | Requirement |
|---|---|
| FR-MAP-01 | The system **shall** render vector maps from `.obf` binary files entirely on-device. |
| FR-MAP-10 | The system **shall** render maps and display all POI data without any internet connection. |
| FR-MAP-02 | The system **shall** display the user's current position and orientation on the map. |
| FR-MAP-03 | The system **shall** support map rotation by compass direction or direction of motion. |
| FR-MAP-04 | The system **shall** support customisable rendering styles (default, touring, nautical, ski, etc.) via XML rendering rules. |
| FR-MAP-05 | The system **shall** support display of online raster tile overlays (e.g., Bing satellite, OpenTopoMap). |
| FR-MAP-06 | The system **shall** support GPX track overlays on the map with customisable transparency. |
| FR-MAP-07 | The system **shall** support automatic day/night map style switching. |
| FR-MAP-08 | The system **shall** display place names in the user's choice of local, English, or phonetic spelling. |
| FR-MAP-09 | The system **shall** optionally display contour lines and hillshading (paid plugin). |

#### 6.1.3 Search

| ID | Requirement |
|---|---|
| FR-SRC-01 | The system **shall** support address search (city, street, house number) using offline data. |
| FR-SRC-02 | The system **shall** support POI (Point of Interest) search by category and name. |
| FR-SRC-03 | The system **shall** support search by geographic coordinates (decimal and DMS). |
| FR-SRC-04 | The system **shall** support search along a calculated route. |
| FR-SRC-05 | The system **shall** support Wikipedia POI lookup (premium feature). |

#### 6.1.4 Map Data Management

| ID | Requirement |
|---|---|
| FR-DATA-01 | The system **shall** allow users to download regional map files (`.obf`) directly from within the app. |
| FR-DATA-02 | The system **shall** support map updates at least once a month via full re-download. |
| FR-DATA-03 | The system **shall** support OsmAnd Live hourly map diff updates (subscription feature). |
| FR-DATA-04 | The system **shall** allow map files to be stored on external SD card. |
| FR-DATA-05 | The free version **shall** limit downloads to 16 map files; the paid version has no limit. |

#### 6.1.5 Track Recording & GPX

| ID | Requirement |
|---|---|
| FR-GPX-01 | The system **shall** record the user's movement as a GPX track, including background recording while the screen is off. |
| FR-GPX-02 | The system **shall** display speed, altitude, and distance statistics during track recording. |
| FR-GPX-03 | The system **shall** allow import and export of GPX files. |
| FR-GPX-04 | The system **shall** allow recorded GPX tracks to be uploaded directly to OSM. |

#### 6.1.6 User Profiles & Settings

| ID | Requirement |
|---|---|
| FR-PRF-01 | The system **shall** support multiple named user profiles (e.g., Driving, Cycling, Hiking), each with independent navigation settings, map styles, and routing parameters. |
| FR-PRF-02 | The system **shall** allow saving locations as Favourites with custom names and icons. |
| FR-PRF-03 | The system **shall** allow import/export of all user data (favourites, tracks, settings) for backup and migration. |

#### 6.1.7 OSM Contribution

| ID | Requirement |
|---|---|
| FR-OSM-01 | The system **shall** allow users to report map errors as OSM notes. |
| FR-OSM-02 | The system **shall** allow users to add new POIs and submit them to OSM directly or defer submission for offline use. |

#### 6.1.8 Plugin System

| ID | Requirement |
|---|---|
| FR-PLG-01 | The system **shall** provide an extensible plugin architecture for optional features. |
| FR-PLG-02 | Core plugins **shall** include: Contour Lines, Wikipedia, Nautical Map, Ski Map, Trip Recording, Parking Position, OsmAnd Tracker (Telegram). |
| FR-PLG-03 | Third-party developers **shall** be able to integrate with OsmAnd via the public AIDL API (`OsmAnd-api` module). |

---

### 6.2 Non-Functional Requirements

Non-functional requirements are constraints that do not directly define what the system does, but define conditions under which it must operate. They are grouped into three categories: **Product** (constraints on the software/hardware itself), **Organisational** (company policies and standards), and **External** (laws, regulations, and third-party obligations).

#### 6.2.1 Product Requirements

Constraints on the product's runtime behaviour, performance, and technical qualities.

| ID | Requirement |
|---|---|
| NFR-P01 | Map frame rendering shall achieve smooth scrolling (≥30 fps) on mid-range Android devices (e.g. 2GB RAM, quad-core CPU). |
| NFR-P02 | Route calculation for distances up to 200 km shall complete within 10 seconds on a mid-range device. |
| NFR-P03 | The app shall support Android 5.0 (API 21) and above, and iOS 12.0 and above. |
| NFR-P04 | The app shall consume no more storage than necessary; regional `.obf` map files shall be compact (e.g. all of Germany ≤ 900 MB). |
| NFR-P05 | Background navigation shall continue when the screen is off, consuming minimal CPU and battery. |
| NFR-P06 | The app shall support 70+ UI languages and display map labels in local script, English, or phonetic transliteration. |
| NFR-P07 | New navigation profiles and rendering styles shall be addable without recompiling the application. |
| NFR-P08 | The app shall remain stable (no crash) when GPS signal is lost or intermittent during an active navigation session. |

#### 6.2.2 Organisational Requirements

Constraints imposed by OsmAnd's development organisation, policies, and standards.

| ID | Requirement |
|---|---|
| NFR-O01 | The application source code shall be licensed under the **GNU General Public License v3 (GPLv3)**. |
| NFR-O02 | All external contributor pull requests shall be accepted under the **MIT License** to allow relicensing. |
| NFR-O03 | Map data shall be sourced exclusively from **OpenStreetMap** and shall not include proprietary map content. |
| NFR-O04 | The project shall maintain public issue tracking and accept community contributions via GitHub. |
| NFR-O05 | Map data updates shall be made available to users at least once per calendar month. |

#### 6.2.3 External Requirements

Constraints arising from external laws, regulations, or third-party platform rules.

| ID | Requirement |
|---|---|
| NFR-E01 | No user location data shall be transmitted to OsmAnd servers during normal offline operation, in compliance with applicable data protection regulations (e.g. GDPR). |
| NFR-E02 | The app shall comply with **Google Play** and **Apple App Store** distribution policies, including privacy disclosure requirements. |
| NFR-E03 | Any integration with the **OSM API** for user contributions shall comply with the OpenStreetMap Contributor Terms. |
| NFR-E04 | The **OsmAnd Tracker** plugin shall comply with **Telegram's Bot API Terms of Service** when transmitting location data. |
| NFR-E05 | Voice guidance audio files shall not infringe third-party copyright; all bundled audio shall be recorded under a compatible open licence. |

---

## 7. Specifications

Specifications are technical definitions that link the requirements (§6) to the system. Each specification below elaborates a requirement into a concrete technical constraint — defining algorithms, data formats, protocols, interfaces, or performance thresholds.

### 7.1 Navigation & Routing

| ID | Links to | Specification |
|---|---|---|
| SP-NAV-01 | FR-NAV-01 | Route calculation **shall** use a **bidirectional A\* algorithm** operating on a directed weighted graph loaded from the `.obf` road network index. Edge weights **shall** be computed as `distance / (speed × priority) + turn_penalty`, where all parameters are defined in `routing.xml`. |
| SP-NAV-02 | FR-NAV-02, FR-NAV-11 | The routing engine **shall** operate entirely on locally stored `.obf` files using `BinaryMapIndexReader`. No network call shall be made during route calculation. |
| SP-NAV-03 | FR-NAV-05 | Voice guidance **shall** be delivered via the Android `TextToSpeech` API or pre-recorded `.ogg` audio files. Manoeuvre announcements **shall** be triggered at distances configurable per profile (default: 2000 m, 500 m, and 50 m before the manoeuvre). |
| SP-NAV-04 | FR-NAV-07 | Off-route detection **shall** occur when the GPS position deviates more than a configurable threshold (default: 50 m for car, 20 m for pedestrian) from the nearest route segment. Re-routing **shall** complete within 5 seconds of deviation detection on a mid-range device. |
| SP-NAV-05 | FR-NAV-03 | All routing profiles, road type priorities, speed limits, and avoidance penalties **shall** be defined exclusively in `routing.xml`. No routing constant shall be hardcoded in Java source. |

### 7.2 Map Rendering

| ID | Links to | Specification |
|---|---|---|
| SP-MAP-01 | FR-MAP-01, FR-MAP-10 | Vector map rendering **shall** use the native **OsmAnd-core** C++ library via JNI for hardware-accelerated OpenGL ES 2.0 rendering. A Java 2D canvas fallback **shall** be available for devices without OpenGL ES support. |
| SP-MAP-02 | FR-MAP-04 | Rendering rules **shall** be defined in XML style files parsed by `RenderingRulesStorage`. Style files **shall** specify display conditions per zoom level, object type, and tag combination. Styles **shall** be swappable at runtime without restarting the application. |
| SP-MAP-03 | FR-MAP-07 | Day/night mode switching **shall** be implemented as a rendering style parameter. Automatic switching **shall** use the device's civil twilight time computed from the current GPS coordinates and date. |
| SP-MAP-04 | FR-MAP-02 | The user's position marker **shall** be updated on the map within 1 second of each GPS fix. Bearing **shall** be smoothed using a low-pass filter to reduce jitter from noisy GPS headings. |

### 7.3 Map Data Format

| ID | Links to | Specification |
|---|---|---|
| SP-DATA-01 | FR-DATA-01 | Map files **shall** use the **OsmAnd Binary Format (`.obf`)**, a compact indexed binary format compiled from OSM data by OsmAnd MapCreator. The application **shall not** parse raw OSM XML or PBF at runtime. |
| SP-DATA-02 | FR-DATA-01 | An `.obf` file **shall** contain the following indexed sections: road network graph, rendered map objects, address index (city/street/house), POI index, and public transport index. Each section **shall** be spatially indexed for tile-based access. |
| SP-DATA-03 | FR-DATA-03 | OsmAnd Live differential updates **shall** be delivered as binary diff files applied to the base `.obf` index. Diffs **shall** be available at minimum hourly intervals for active subscribers. |
| SP-DATA-04 | NFR-P04 | Map data **shall** be loaded using memory-mapped I/O (`FileChannel.map`) to bound RAM usage. Only tiles intersecting the current viewport or routing bounding box **shall** be loaded into memory at any time. |

### 7.4 Search

| ID | Links to | Specification |
|---|---|---|
| SP-SRC-01 | FR-SRC-01, FR-SRC-02 | All search queries **shall** be resolved against locally stored `.obf` address and POI indexes via `SearchUICore`. Search **shall** not require a network connection. |
| SP-SRC-02 | FR-SRC-01 | Address search **shall** follow a hierarchical resolution order: country → region → city → street → house number. Each level **shall** be matched against the address index using prefix and fuzzy matching. |
| SP-SRC-03 | FR-SRC-04 | Route-along search **shall** restrict POI results to amenities within a configurable corridor width (default: 1000 m) of the active route polyline, evaluated using spatial intersection. |

### 7.5 GPS and Location

| ID | Links to | Specification |
|---|---|---|
| SP-LOC-01 | FR-NAV-02, FR-GPX-01 | The app **shall** subscribe to location updates via the **Android `LocationManager` API** (or `FusedLocationProviderClient` where available), requesting updates at a minimum interval of 1 second with a minimum displacement of 1 metre during active navigation or recording. |
| SP-LOC-02 | FR-GPX-01 | Background track recording **shall** run as an **Android Foreground Service** with a persistent notification, ensuring the process is not killed by the OS during navigation or recording sessions. |
| SP-LOC-03 | FR-NAV-10 | Speed limit data **shall** be read from the `maxspeed` OSM tag stored in the `.obf` road network index. A speed warning **shall** be triggered when `current_speed > speed_limit × tolerance_factor` (default tolerance: 1.1). |

### 7.6 GPX

| ID | Links to | Specification |
|---|---|---|
| SP-GPX-01 | FR-GPX-01, FR-GPX-03 | GPX files **shall** conform to the **GPX 1.1 schema** (topografix.com/GPX/1/1). Track points **shall** include `lat`, `lon`, `ele` (if available), and `time` attributes. OsmAnd extensions **shall** be written under the `osmand:` namespace for speed and heading. |
| SP-GPX-02 | FR-GPX-02 | Track statistics (total distance, moving time, elevation gain/loss, average and max speed) **shall** be computed from the recorded `WptPt` sequence using the Haversine formula for distances and forward-difference derivatives for speed. |

### 7.7 User Data and Profiles

| ID | Links to | Specification |
|---|---|---|
| SP-PRF-01 | FR-PRF-01 | Each user profile **shall** be represented as an `ApplicationMode` instance. All profile-specific settings **shall** be stored under a namespaced key in `OsmandSettings`, backed by Android `SharedPreferences`. |
| SP-PRF-02 | FR-PRF-03 | Profile and user data export **shall** produce a `.osf` (OsmAnd Settings File) archive containing settings JSON, GPX tracks, favourites XML, and rendering style files. Import **shall** restore all contained data atomically. |

### 7.8 Third-Party Integration

| ID | Links to | Specification |
|---|---|---|
| SP-API-01 | FR-PLG-03 | The `OsmAnd-api` module **shall** expose navigation and map control functions to third-party Android apps via **Android Interface Definition Language (AIDL)**. The API **shall** include: set destination, add waypoint, show POI on map, get current location, and start/stop navigation. |
| SP-API-02 | NFR-E04 | The OsmAnd Tracker plugin **shall** transmit location data to Telegram contacts exclusively via the **Telegram Bot API** over HTTPS. No location data **shall** be routed through OsmAnd servers. |

### 7.9 Security and Privacy

| ID | Links to | Specification |
|---|---|---|
| SP-SEC-01 | NFR-E01 | During offline operation, the app **shall** make zero outbound network calls related to user location. Any optional telemetry or crash reporting **shall** be opt-in and disabled by default. |
| SP-SEC-02 | NFR-E02 | All communication with OsmAnd servers (map downloads, OsmAnd Live updates, OsmAnd Cloud sync) **shall** use **HTTPS (TLS 1.2 or higher)**. Plain HTTP connections **shall not** be used for any data transfer. |

---

## 8. Glossary

| Term | Definition |
|---|---|
| `.obf` | OsmAnd Binary Format — compact binary map data file compiled from OSM data |
| OsmAnd Live | Subscription service providing hourly map diff updates |
| GPX | GPS Exchange Format — XML-based standard for recording tracks and waypoints |
| POI | Point of Interest — named geographic feature (restaurant, hotel, hospital, etc.) |
| WMM | World/Machine Model — requirements technique distinguishing the software from its environment |
| A* | A-star pathfinding algorithm — used for route calculation in OsmAnd's routing engine |
| TTS | Text-to-Speech — OS service for synthesising voice navigation instructions |
| AIDL | Android Interface Definition Language — used for OsmAnd's third-party API |
| Rendering style | XML configuration file defining how map vector objects are visually displayed |
| Profile | Named configuration set (navigation type, map style, routing params) for a travel mode |
