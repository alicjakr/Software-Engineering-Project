# OsmAnd

**OsmAnd (OSM Automated Navigation Directions)** is an offline-first map viewing and
turn-by-turn navigation application for Android, built entirely on OpenStreetMap (OSM)
data. Its defining property is that routing, vector map rendering, address/POI search,
GPX track recording, and voice guidance all work fully offline, using regional map
files downloaded ahead of time — no network connection is required at any point during
navigation.

> This README documents the Android edition, based on the requirements and architecture
> analysis in this repository (see [Project Documentation](#project-documentation)
> below). Out of scope: OsmAnd MapCreator, OsmAnd Cloud server-side infrastructure, and
> web map services.

## Features

**Navigation and routing**
- Turn-by-turn routing for car, bicycle, and pedestrian profiles, fully offline
- Intermediate waypoints and automatic re-routing when the user leaves the planned route
- Road-type and feature avoidance (tolls, highways, ferries, etc.)

**Voice guidance**
- Spoken manoeuvre announcements timed to arrive early enough to act on safely
- Recorded or synthesized (TTS) voices in the user's chosen language
- Glanceable next-manoeuvre, distance, road name, ETA, lane guidance, and speed-limit warnings

**Map viewing**
- Fully offline vector map rendering with automatic day/night styles
- North-up or direction-of-travel map rotation
- Track overlays, switchable rendering styles, and multilingual place-name labels

**Search**
- Offline address, category/name, and coordinate search
- POI search along an active route

**Map data management**
- In-app download of regional map data, resumable after interruption
- More-frequent-than-monthly incremental updates
- Corruption detection with re-download offered instead of a crash

**Track recording**
- Background GPX recording via a persistent foreground service, resilient to app kills
- Live speed/distance/time display, GPX import/export, and direct upload to OSM

**Profiles, favourites, and accounts**
- Multiple named profiles with independent navigation, voice, and rendering settings
- Favourites, backup/restore, and optional cross-device sync via OsmAnd Cloud
- All core features (viewing, routing, search, recording) work fully signed-out

**OSM contribution and third-party integration**
- Report map bugs, add/edit POIs, and upload GPX tracks to OpenStreetMap from within the app
- A documented AIDL API lets other Android apps trigger navigation and query location, subject to user-granted permission

## Architecture

OsmAnd is a multi-module Android project:

| Module | Responsibility |
|---|---|
| `OsmAnd` | Main Android application: UI, navigation, map viewing, plugins |
| `OsmAnd-java` | Java/Kotlin routing, search, and map-data logic shared across the app |
| `OsmAnd-shared` | Cross-platform (Kotlin Multiplatform) core logic |
| `OsmAnd-api` | Public AIDL interface for third-party app integration |
| `OsmAnd-telegram` | Companion app for live location sharing via Telegram |
| `plugins/*` | Optional feature plugins (nautical charts, SRTM/contour lines, ski maps) |

Performance-critical work — vector map rendering and route calculation — runs in a
native C++ core (OsmAnd-core) linked into the app over JNI, rather than in the JVM
layer. Map data is distributed as OBF (OsmAnd Binary Format): a compact, spatially
indexed binary compiled from OSM data, read via memory-mapped I/O so resident memory
scales with the visible viewport rather than the file size on disk.

Key subsystems, as modeled in this repository's class diagrams:
- **Routing** — `RoutingHelper` → `RouteProvider` → `RoutePlannerFrontEnd`, dispatching
  to `BinaryRoutePlanner` (bidirectional A\*) for short/medium routes or `HHRoutePlanner`
  (contraction hierarchies) for long-distance routes
- **Map data & resource management** — OBF index loading and caching
- **App core / profiles / user data** — `ApplicationMode` and `PluginsHelper`
- **Auth & accounts** — OsmAnd Cloud sign-in (email + verification code) and OSM account
  linking (OAuth 2.0)

## Building from Source

The project builds with Gradle (`./gradlew`) and Android Studio. Because the map
rendering and routing engines are native code linked over JNI, a full build also
requires the native `OsmAnd-core` toolchain (CMake/NDK). Authoritative, up-to-date build
instructions are maintained at
[osmand.net/docs/technical/build-osmand](https://www.osmand.net/docs/technical/build-osmand/).

## Alternative Design Approaches

A few of OsmAnd's core design decisions could reasonably have gone the other way:

- **On-device routing (A\*/HH over local OBF files) vs. server-side routing** — a
  cloud routing API (e.g. OSRM/GraphHopper-style) would shrink the app and simplify
  updates, at the cost of the offline guarantee that drives most of the requirements.
- **Native C++ rendering/routing core via JNI vs. a pure-JVM implementation** — the
  existing Java 2D Canvas fallback shows the trade-off directly: simpler to maintain,
  but slower on the render/search-heavy paths the native core targets.
- **Local-first data with opt-in cloud sync vs. cloud-first accounts** — OsmAnd treats
  the device as the source of truth and OsmAnd Cloud as an optional sync target; an
  account-first design would simplify multi-device consistency but would break the
  "everything works fully signed-out" requirement.

## Project Documentation

This repository contains a Requirements Engineering / architecture analysis of OsmAnd,
produced using the World-Machine Model (Jackson problem-frames) method:

- [`requirements_specification.md`](requirements_specification.md) — glossary, World
  assumptions, Requirements, and Specifications
- [`diagrams/logical/`](diagrams/logical) — logical architecture: context, components,
  mapdata/navigation drill-down views
- [`diagrams/physical/`](diagrams/physical) — physical (deployment) architecture
- [`diagrams/class/`](diagrams/class) — class diagrams: routing subsystem, map data &
  resource management, auth & accounts, app core/profiles/user data
- [`diagrams/system_dynamics/`](diagrams/system_dynamics) — sequence (route
  calculation) and state machine (cloud sign-in) diagrams
- [`diagrams/activity/`](diagrams/activity) — activity diagrams: the search → route
  calculation → guidance flow, split into three phases
  
System dynamics diagrams are in diagrams/system_dynamics and (separated) diagrams/activity.

## License

OsmAnd is open source, released under the terms described in
[LICENSE](https://github.com/osmandapp/OsmAnd/blob/master/LICENSE) in the upstream
repository.
