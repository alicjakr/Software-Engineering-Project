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
│                            THE WORLD                                │
│                                                                     │
│  [User]  [GPS Satellites]  [Road Network]  [OSM Servers]            │
│  [Device Filesystem]  [Text-to-Speech Engine]  [OsmAnd Cloud]       │
│                                                                     │
│           ↕ shared phenomena (interface)                            │
│ ──────────────────────────────────────────────────────────────────  │
│                          THE MACHINE                                │
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

## 6. Specifications

### 6.1 Functional Requirements

#### 6.1.1 Navigation & Routing

| ID | Requirement |
|---|---|
| FR-NAV-01 | The system **shall** calculate routes between a start and destination point for car, bicycle, and pedestrian profiles. |
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

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | **Offline capability** | All core features (routing, rendering, search, POI) shall function without any network connection. |
| NFR-02 | **Performance – map rendering** | Map frame rendering shall achieve smooth scrolling (≥30 fps) on mid-range Android devices. |
| NFR-03 | **Performance – routing** | Route calculation for distances up to 200 km shall complete within 10 seconds on a mid-range device. |
| NFR-04 | **Storage efficiency** | Compact `.obf` binary format shall be used; e.g., all of Japan ≤ 700 MB, road-network-only ≤ 200 MB. |
| NFR-05 | **Battery efficiency** | The system shall provide a screen-off navigation mode where routing continues via voice guidance only. |
| NFR-06 | **Platform support** | The system shall support Android 5.0+ and iOS 12+. |
| NFR-07 | **Internationalisation** | The system shall support 70+ languages for the UI and map label display. |
| NFR-08 | **Accessibility** | Voice guidance shall be available for all navigation manoeuvres. |
| NFR-09 | **Openness** | The system shall be licensed under GPLv3; contributor pull requests shall be accepted under MIT license. |
| NFR-10 | **Data freshness** | Map data shall be updated at least monthly via full re-download; hourly updates available via OsmAnd Live. |
| NFR-11 | **Privacy** | No user location data shall be transmitted to OsmAnd servers during normal offline operation. |
| NFR-12 | **Extensibility** | New navigation profiles and rendering styles shall be addable without modifying core application code. |

---

## 7. Glossary

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
