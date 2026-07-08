# Requirements Specification — OsmAnd

**Project:** OsmAnd

**Repository:** https://github.com/osmandapp/OsmAnd


---

## Scope

OsmAnd is an open-source map-viewing and turn-by-turn navigation application for Android, built on OpenStreetMap data. Its defining property is full offline operation: routing, vector map rendering, address and POI search, GPX track recording, and voice guidance all work without a network connection using pre-downloaded regional map files.

This document covers the Android edition. Out of scope: OsmAnd MapCreator, OsmAnd Cloud server-side infrastructure, and web map services.

---

## 1. Glossary

| Term | Definition |
|---|---|
| OBF | OsmAnd Binary Format — compact indexed binary map data compiled from OpenStreetMap. |
| OsmAnd Live | Subscription service delivering hourly incremental map updates as binary diffs. |
| GPX | GPS Exchange Format — XML standard for recording tracks and waypoints. |
| OSF | OsmAnd Settings File — archive format for backup and restore of settings, tracks, and favourites. |
| POI | Point of Interest — a named geographic feature such as a shop, restaurant, or landmark. |
| A\* | A-star bidirectional pathfinding algorithm used for short and medium-distance routes. |
| HH routing | Hierarchical Highway routing — contraction-hierarchy algorithm used for long-distance routes. |
| TTS | Text-to-Speech — OS service that synthesises spoken instructions from text. |
| AIDL | Android Interface Definition Language — the IPC mechanism used for OsmAnd's third-party API. |
| Profile | A named configuration set for a specific travel mode (car, bicycle, pedestrian, etc.). |
| Plugin | An optional feature module that extends the app through defined extension hooks. |
| GNSS | Global Navigation Satellite System — encompasses GPS, GLONASS, Galileo, and BeiDou. |
| OAuth 2.0 | Open standard for delegated authorisation; used for OSM account linking. |
| PKCE | Proof Key for Code Exchange — OAuth 2.0 extension that prevents authorisation-code interception. |
| Foreground service | An Android service running with a persistent notification, exempt from aggressive background-process killing. |


## 2. World

The World is the environment OsmAnd operates in - static facts that hold whether or not the app exists.

1. Consumer GNSS receivers produce position fixes outdoors; accuracy degrades in tunnels, indoors, and urban canyons.
2. A moving device's position and heading can be extrapolated from its last known fix, speed, and bearing when GNSS signal is temporarily lost.
3. The OpenStreetMap database represents real-world geography — roads, buildings, and points of interest.
4. Road speed limits and turn restrictions are encoded in OSM tags.
5. OSM is a crowd-sourced database; any downloaded snapshot gradually diverges from reality over time.
6. OSM road coverage quality varies by region; rural and developing-world areas may have incomplete road data, missing speed limits, or absent turn restrictions.
7. Mobile devices have a screen, audio output, a GPS chip, a compass, and persistent storage.
8. Regional map files range from a few megabytes for small islands to several gigabytes for large countries; available on-device storage constrains which regions a user can hold simultaneously.
9. Mobile data networks are intermittent or absent in many real-world locations.
10. In areas with high mobile data costs, poor network coverage, or during international travel, offline map use is the primary expected mode of operation, not a fallback.
11. Android exposes location, Text-to-Speech, and audio focus services to applications.
12. An Android foreground service with a persistent notification is not killed by the OS during normal use.
13. Without a foreground service, the OS may kill an app's process at any time when the screen is off.
14. OSM contributions require a registered OSM account with OAuth 2.0 credentials.
15. OsmAnd servers host pre-compiled regional map files and incremental update packages.
16. Continuous GPS use and screen-on navigation can drain a phone battery in under 6 hours.
17. A user operating a vehicle cannot safely divert attention to a screen; spoken output and glanceable information are the only safe interaction channels during active driving.
18. Users speak many languages and read many scripts.

---

## 3. Requirements

Requirements state what users need from the system - no algorithms, file formats, or technical details appear here.

### Navigation and routing

1. A user with a downloaded map and no internet connection shall be able to get a route between any two reachable points.
2. The user shall be able to navigate by car, by bicycle, or on foot.
3. The user shall be able to add intermediate waypoints to a planned route.
4. When the user leaves the planned route, a new route to the destination shall be provided automatically.
5. The user shall be able to avoid certain road types or features when calculating a route.
6. The user shall be able to cancel an active navigation session at any time.

### Voice guidance

1. The user shall be informed of each upcoming manoeuvre by a spoken announcement, without needing to look at the screen.
2. Each announcement shall arrive early enough for the user to act on it safely at their current speed.
3. Announcements shall be delivered in the user's chosen language.
4. While navigating, the user shall be able to see the next manoeuvre, distance to it, road name, and estimated arrival time at a glance.
5. The user shall be warned when travelling faster than the legal speed limit for the current road.
6. Where lane data is available, the user shall see lane guidance before complex junctions.

### Map viewing

1. The user shall be able to view a downloaded region's map with no internet connection.
2. The user's current position and direction of travel shall be shown on the map.
3. The user shall be able to choose whether the map rotates with direction of travel or stays north-up.
4. The map shall automatically switch between day and night styles.
5. The user shall be able to choose the language and script for place name labels.
6. The user shall be able to display recorded or imported tracks as overlays on the map.
7. The user shall be able to change the map rendering style without restarting the app.

### Search

1. The user shall be able to find places by address using offline data.
2. The user shall be able to find places by category or name.
3. The user shall be able to locate a position by entering geographic coordinates.
4. The user shall be able to search for points of interest along an active route.

### Map data management

1. The user shall be able to download regional map data from within the app.
2. The user shall be able to receive map updates more frequently than the monthly base cycle.
3. A download interrupted by network loss or app termination shall be resumable without starting from the beginning.
4. The app shall detect corrupted map files and offer a re-download rather than crashing.

### Track recording

1. The user shall be able to record a trip as a GPX track while the screen is off or the app is in the background.
2. While recording, the user shall be able to see live speed, distance, and elapsed time.
3. A recording shall survive unexpected app termination without losing points already saved.
4. The user shall be able to import and export GPX tracks.
5. The user shall be able to upload a recorded track to OpenStreetMap.

### Profiles, favourites, and settings

1. The user shall be able to maintain multiple named profiles, each with independent navigation, routing, voice, and rendering settings.
2. The user shall be able to save locations as Favourites with custom names, icons, and groups.
3. The user shall be able to back up and restore all their data.
4. The user shall, when signed in, be able to sync their data across multiple devices.

### OSM contribution

1. The user shall be able to report a map error as an OSM note from within the app.
2. The user shall be able to add or edit a point of interest and submit the change to OSM, either immediately or when back online.

### Plugins and third-party integration

1. The user shall be able to enable or disable optional features at runtime without reinstalling the app.
2. A third-party app on the same device shall be able to start navigation, add markers, and query location from OsmAnd, subject to user permission.

### Accounts

1. All core features — map viewing, routing, search, and recording — shall be usable without signing in.
2. The user shall be able to sign in to an OsmAnd Cloud account to enable data sync.
3. The user shall be able to link an OSM account to enable community contributions.
4. Signed-in state shall survive app restarts and device reboots while the session token remains valid.
5. The user shall not be asked to re-authenticate for routine operations while their session remains valid.
6. The user shall be able to permanently delete their OsmAnd Cloud account and all data synced to it.

---

## 4. Specifications

Specifications define the technical details of how the Machine satisfies the requirements.

> **Note:** All numeric values below were checked against source (`OsmAnd-resources/routing/routing.xml` for routing constants; `BaseMapLayer.java` and `TileSourceManager.java` for zoom range and tile size), except the 5 s reroute target (Navigation and routing, #5) and the 1 s marker-redraw target (Map rendering, #4), which read as design targets rather than named constants and could not be pinned to a specific source line.

### Navigation and routing

1. Route calculation uses bidirectional A\* search over the road-network index of the OBF map file; for long-distance routes a Hierarchical Highway (HH) contraction-hierarchy algorithm is used instead.
2. All routing constants — speeds, penalties, avoidances per profile — are defined in an external `routing.xml` file; none are hard-coded in the application.
3. Route calculation makes no network call; it reads only on-device OBF files.
4. An off-route condition is raised when the current GNSS fix lies beyond a distance tolerance from the route corridor, evaluated on each location update.
5. A new route is produced within 5 s of an off-route condition on a mid-range device.
6. A route supports multiple ordered intermediate waypoints; no fixed upper count is enforced by the routing engine.
7. Default shortest-path speeds: car 45 km/h, bicycle 10 km/h.

### Voice guidance

1. Spoken instructions are delivered via the OS Text-to-Speech service or pre-recorded audio packs, as the user selects.
2. Announcement trigger distances are computed per profile from the current speed and a target lead time for each announcement tier (prepare, turn, immediate), not fixed distances; faster travel triggers earlier announcements.
3. Per-language voice grammars are JavaScript files executed by an embedded scripting engine, so language packs can be updated independently of the app binary.
4. The app requests transient audio focus before each prompt so that other media ducks or pauses; focus is yielded immediately on an incoming call.
5. The overspeed warning fires when current speed exceeds the road's `maxspeed` value; it repeats at most every 120 s.

### Map rendering

1. Vector rendering uses a hardware-accelerated OpenGL ES 2.0 pipeline via a native C++ library linked over JNI; a Java 2D Canvas fallback is provided for devices without OpenGL ES 2.0.
2. Rendering rules are defined in XML style files; a style change takes effect on the next frame without restarting the app or reloading map data.
3. Day/night switching is driven by civil twilight time computed from the current location and date.
4. The position marker is redrawn within 1 s of each GNSS fix; bearing is passed through a low-pass filter to suppress heading jitter at low speeds.
5. Map zoom range: 1–21 levels; standard tile size 256 × 256 px.

### Map data

1. Map data uses the OsmAnd Binary Format (OBF): a compact spatially-indexed binary compiled from OSM data by external tooling.
2. Each OBF file contains six indexes: road network (routing), map objects, address, POI, public transport, and a contraction-hierarchy (HH) routing index used for long-distance route pre-processing; each is spatially indexed for tile-based access.
3. OBF files are read via memory-mapped I/O so resident RAM scales with the visible viewport, not with the total file size on disk.
4. Live updates are binary diff files applied to the base OBF; available at minimum hourly cadence for subscribers.
5. Downloads use HTTP Range requests to resume from a partial-file offset after interruption.
6. Each downloaded file is verified for header validity and expected size before use; corrupted files are moved out of the active maps directory and re-download is offered.
7. The free build is limited to 7 map-file downloads; the paid build has no limit.

### Track recording

1. Track recording runs inside an Android Foreground Service with a persistent notification, keeping the process alive when the screen is off.
2. Recorded tracks conform to GPX 1.1; each point carries latitude, longitude, elevation, and timestamp; OsmAnd-specific extensions are written under the `osmand:` XML namespace.
3. Each GNSS fix is written to the in-progress file before the next fix arrives, so the track survives a process kill.
4. Distance is computed using the Haversine formula; speed via forward-difference derivatives over consecutive fixes.

### Profiles, favourites, and settings

1. Each profile is a named configuration instance; per-profile settings are namespaced by mode key in the on-device preferences store.
2. Settings backup and restore uses the `.osf` archive format, containing settings JSON, GPX tracks, favourites XML, and rendering style files; restore is atomic.
3. Cloud sync sends settings, favourites, and GPX tracks to OsmAnd Cloud over TLS; conflicts are resolved by last-modified timestamp.

### OSM contribution

1. Map-bug notes are submitted to `https://api.openstreetmap.org/api/0.6/notes`.
2. GPX tracks are uploaded to `https://api.openstreetmap.org/api/0.6/gpx/create`.
3. POI edits are staged locally and uploaded when network is available.
4. OSM account linking uses OAuth 2.0 authorization-code grant (client secret, no PKCE) against `https://www.openstreetmap.org/oauth2/authorize` and `https://www.openstreetmap.org/oauth2/token`; requested scopes are supplied as an authorization parameter and cover account, notes, GPX, and POI editing permissions.

### Plugins and third-party integration

1. Every plugin extends a common abstract base class; the app does not reference plugin classes directly outside defined extension hooks.
2. Enabling or disabling a plugin requires no process restart.
3. The third-party IPC surface exposes method groups over Android AIDL for navigation, location, search, favourites, markers, GPX, and map layers; callbacks are delivered via a registered AIDL callback interface.
4. All AIDL methods that change app state are gated behind a user-grantable permission requested at first bind.

### Accounts and credentials

1. OsmAnd Cloud sign-in uses an email plus verification-code flow over TLS; the account is created automatically on first successful verification.
2. The verification code is single use and valid for 10 minutes; the client enforces no separate resend cooldown or failed-attempt lockout.
3. A single access token is issued per device on successful code verification; there is no separate refresh token, and the client has no fixed expiry for it.
4. An invalid or expired access token is reported to the user as a sign-in error; the user re-authenticates via a new email verification code rather than a silent refresh.
5. The access token is stored in a local app preference; it is not written to logs, exports, or crash reports.
6. On sign-out, the device ID and access token are deleted; the account email and non-account-bound user data are not touched.
7. On account deletion, the same verification-code flow re-confirms identity; on confirmation, server-side synced data and the local device ID and access token are deleted; non-account-bound local data is not touched.

### Privacy and network

1. All traffic to OsmAnd-controlled endpoints uses TLS 1.2 or higher; plain HTTP is not used for any data transfer.
2. When the app is offline, no routing, rendering, search, or location code path opens an outbound network connection.
3. Log output does not include GPS coordinates, OAuth tokens, or account credentials.
4. Crash reports are written locally; upload to any remote service requires explicit user opt-in.
