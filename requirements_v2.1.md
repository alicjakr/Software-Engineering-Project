# Requirements Specification — OsmAnd
## World / Machine Model

**Project:** OsmAnd
**Repository:** https://github.com/osmandapp/OsmAnd
**Version:** 2.1
**Date:** June 2026

Structure: §1–§2 scope and method · §3–§5 the World · §6 the Machine · §7 shared phenomena · §8 domain assumptions · §9–§11 requirements and constraints · §12 specifications · §13 D, S ⊢ R justification · §14 traceability (informative) · §15 glossary · §16 references.

---

## 1. Scope

OsmAnd is an open-source map-viewing and turn-by-turn navigation application for Android and iOS, built on OpenStreetMap data. Its defining property is **full offline operation**: all core features — routing, vector map rendering, address and POI search, GPX recording, voice guidance — must work without any network connection, using pre-downloaded binary map files.

This document covers the Android edition. The iOS edition shares the cross-platform modules and inherits the same requirements; platform-specific specifications identify when a clause is Android-only.

Out of scope: OsmAnd MapCreator; OsmAnd Cloud server-side implementation; web map services.

---

## 2. Method

This document follows Michael Jackson's World/Machine Model (WMM):

- **R — Requirement.** What the user needs, in the user's vocabulary. No algorithms, file formats, version numbers, or API names.
- **S — Specification.** What the Machine must do at its boundary. May name protocols, algorithms, and measurable thresholds. Must not name internal implementation classes (those belong in §14).
- **D — Domain assumption.** How the World behaves, independent of the Machine. Indicative mood.
- **`D, S ⊢ R`** — Domain assumptions together with specifications must entail the requirements. §13 argues this for the headline requirements.

§§3–5 and §8 are **indicative** (describe the World as it is). §§9–12 are **optative** (state what we want once the Machine is in place).

ID prefixes: `REQ` functional requirement · `QUAL` quality requirement · `CON` constraint · `DOM` domain assumption · `SPEC` specification · `PHEN` shared phenomenon.

---

## 3. Actors

| ID | Actor | Role |
|---|---|---|
| ACT.user.navigator | End user | Operates the device while travelling. Issues commands; consumes map, route, and voice output. |
| ACT.user.recorder | End user | Records a GPX track for later use, sharing, or upload. |
| ACT.user.osm-contributor | OSM contributor | Edits OSM through OsmAnd — adding POIs, reporting notes, uploading tracks. |
| ACT.dev.third-party-app | Third-party developer | Binds to OsmAnd's public AIDL service to start navigation, add markers, or query location. |
| ACT.opr.osmand-team | OsmAnd organisation | Operates the build, the servers, and accepts contributions. |

---

## 4. External domains

| ID | Domain | Class | Role |
|---|---|---|---|
| EXT.gnss | Global Navigation Satellite System | causal | Delivers position fixes via the device's GNSS chip. Outside OsmAnd's control. |
| EXT.osm-db | OpenStreetMap database | lexical | Crowd-sourced global geographic dataset. Source of all map content. |
| EXT.osm-api | OSM API | biddable | HTTP API accepting POI edits, OSM notes, and GPX uploads. |
| EXT.osmand-servers | OsmAnd map servers | biddable | Hosts pre-compiled `.obf` regional files and OsmAnd Live diff feeds. |
| EXT.osmand-cloud | OsmAnd Cloud | biddable | Cross-device sync service for user data. |
| EXT.tile-servers | Online raster tile providers | biddable | Optional raster overlay sources (Bing satellite, OpenTopoMap, etc.). |
| EXT.android-os | Android operating system | causal | Provides Location API, TextToSpeech, foreground services, filesystem, audio, notifications. |
| EXT.filesystem | Device filesystem | lexical | Persistent storage for `.obf`, `.gpx`, `.osf`, voice packs, logs, and settings. |
| EXT.tts-engine | OS Text-to-Speech engine | causal | Synthesises spoken instructions from text in a chosen language. |
| EXT.network | Internet | causal | Used for downloads, sync, online tiles, and OSM uploads. Not required for offline operation. |

---

## 5. World context

### 5.1 User context

Users operate OsmAnd while moving — on foot, by bicycle, in a vehicle. Attention and hand availability are limited: a driver cannot read fine text; a cyclist cannot tap precisely. Internet is often unavailable or expensive. Users speak many languages and read many scripts.

*Drives:* `REQ.guidance.voice-announcement`, `REQ.guidance.glanceable-screen`, `REQ.routing.offline-route`, `REQ.map.local-script-labels`.

### 5.2 GNSS behaviour

Fix accuracy is typically 3–15 m in open sky but degrades in urban canyons, tunnels, and indoors. Fixes may be intermittent or briefly lost.

*Drives:* `REQ.navigation.tolerate-gps-loss`, `QUAL.reliability.no-crash-on-gps-loss`.

### 5.3 Map data and network

OsmAnd's map content derives entirely from OpenStreetMap. Coverage and quality vary by region. Mobile data is intermittent or absent in many real-world locations; the app must function fully offline.

*Drives:* `REQ.routing.offline-route`, `REQ.map.offline-render`, `REQ.data.download-region`.

### 5.4 Device constraints

Battery life is finite; continuous GPS and screen-on navigation can drain a phone in under 6 h. The OS may kill background processes aggressively when the screen is off.

*Drives:* `QUAL.perf.battery-screen-off`, `REQ.track.survive-kill`.

---

## 6. The Machine

### 6.1 External run-time shape

The Machine is the **OsmAnd Android application** installed on a single user's device. It runs as an Activity-based app with a foreground service for background sessions and a persistent settings store. It has no server-side counterpart at runtime: each user's Machine is the locally installed app plus its on-device data.

### 6.2 Internal modules *(informative)*

| Module | Languages | Responsibility |
|---|---|---|
| `OsmAnd` | Java + Kotlin | Android UI, activity lifecycle, foreground service, plugin host, AIDL host. |
| `OsmAnd-java` | Java | Platform-neutral core: `.obf` reading, A\* routing, search, rendering rules. No Android dependencies. |
| `OsmAnd-shared` | Kotlin Multiplatform | Cross-platform domain models, GPX, settings, coroutines, SQL access. |
| `OsmAnd-api` | AIDL + Java | Public IPC surface for third-party Android apps. |

Architecture diagrams: `diagrams/logical_components.png`, `diagrams/logical_context.png`, `diagrams/physical_architecture.png`.

---

## 7. Shared phenomena

Only phenomena listed here may appear in `SPEC.*` clauses.

| ID | Phenomenon | Type | Controlled by | Observed by |
|---|---|---|---|---|
| PHEN.navigate-tap | User taps "Navigate" with a destination | event | User | Machine |
| PHEN.search-query | User submits a search query | event | User | Machine |
| PHEN.profile-select | User selects a profile | event | User | Machine |
| PHEN.gps-fix | GNSS reports a position fix | event | EXT.gnss | Machine |
| PHEN.gps-loss | Location updates cease for ≥ N seconds | state | EXT.gnss | Machine |
| PHEN.network-available | Internet reachability of a given endpoint | state | EXT.network | Machine |
| PHEN.obf-present | `.obf` file for a region exists on the filesystem | state | EXT.filesystem + Machine | both |
| PHEN.route-displayed | A route polyline and instructions appear on screen | state | Machine | User |
| PHEN.tts-utter | Spoken instruction emitted from speaker | event | Machine | User |
| PHEN.alert-tone | Warning tone emitted | event | Machine | User |
| PHEN.map-frame | A rendered map frame is displayed | event | Machine | User |
| PHEN.gpx-write | Bytes written to a `.gpx` file on the filesystem | event | Machine | EXT.filesystem |
| PHEN.obf-download | HTTP request/response cycle for a `.obf` region file | event | Machine | EXT.osmand-servers |
| PHEN.osm-upload | HTTP request to OSM API with a contribution payload | event | Machine | EXT.osm-api |
| PHEN.aidl-call | An IPC method call into the OsmAnd AIDL service | event | EXT.third-party-app | Machine |
| PHEN.aidl-callback | A callback delivered to a bound third-party app | event | Machine | EXT.third-party-app |
| PHEN.log-line | A log line written to the on-device log file | event | Machine | EXT.filesystem |
| PHEN.crash-report | A crash diagnostic emitted to the configured channel | event | Machine | EXT.filesystem / EXT.network |
| PHEN.auth-token-issued | An identity provider issues a session or refresh token | event | EXT.osmand-cloud / EXT.osm-api | Machine |
| PHEN.auth-token-revoked | A token is invalidated by the user or provider | event | User / provider | Machine |
| PHEN.auth-signed-in | The Machine holds a valid token for a given identity | state | Machine | User-facing surfaces |
| PHEN.map-gesture | User performs a touch gesture on the map | event | User | Machine |
| PHEN.long-press | User long-presses a point on the map | event | User | Machine |
| PHEN.download-progress | A region download advances or completes | event | Machine | User |

---

## 8. Domain assumptions

| ID | Assumption |
|---|---|
| DOM.gps.fix-outdoors | Outdoors with clear sky, a consumer GNSS receiver produces fixes at ≥ 1 Hz with horizontal accuracy ≤ 15 m. |
| DOM.gps.fix-degraded | Indoors, in tunnels, or in dense urban canyons, fixes may be absent or accuracy may exceed 50 m. |
| DOM.gps.intermittent | A single position fix may be absent for several minutes in covered areas; service resumes when sky is visible. |
| DOM.osm.tag-conventions | Roads in OSM are tagged with `highway=*`; speed limits in `maxspeed`; turn restrictions in relations. |
| DOM.osm.data-staleness | A `.obf` snapshot is out of date with the live OSM database from the moment it is compiled. |
| DOM.android.location-api | The Android OS exposes location through its Location API; updates are delivered at a requested rate and displacement. |
| DOM.android.tts | Android exposes a Text-to-Speech service with at least one language pre-installed. |
| DOM.android.fgs | An Android foreground service with a persistent notification is not killed by the OS during normal use, even screen-off. |
| DOM.android.doze | With the screen off and app not in foreground, the OS throttles background CPU and network unless a foreground service is active. |
| DOM.android.audio-focus | The OS arbitrates audio output between apps via an audio-focus model; an app must yield when focus is lost. |
| DOM.audio.driving-noise | In-vehicle background noise is 60–75 dBA; spoken instructions must be audible above it. |
| DOM.user.attention | The user while driving cannot reliably read fine text; instructions must be delivered audibly and in advance. |
| DOM.user.languages | Users speak many natural languages and read many scripts. |
| DOM.device.battery | Continuous GPS + screen-on navigation can drain a phone in under 6 h. |
| DOM.network.intermittent | Mobile data is intermittent or absent in many real-world locations. |
| DOM.network.metered | The OS exposes whether the current network is metered; cellular is generally treated as metered. |
| DOM.fs.persistence | Files written to internal or SD-card storage persist across reboots and app updates. |
| DOM.power.kill | The OS may kill the app process at any time when the screen is off and no foreground service is running. |
| DOM.osm.contributor-account | OSM contributions require a registered OSM account; the API expects OAuth 2.0 credentials. |
| DOM.account.osm-account-external | OSM accounts are created on `osm.org`; OsmAnd is only a relying party. |
| DOM.account.cloud-account-via-server | An OsmAnd Cloud account is created through the OsmAnd Cloud HTTP service; requires network at registration time. |
| DOM.account.tokens-expire | OAuth access tokens have finite lifetime; refresh tokens may be revoked without notice to the Machine. |

---

## 9. Requirements

Requirements are stated in the user's vocabulary. No algorithms, file formats, byte counts, or class names appear here.

### 9.1 Navigation and routing

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.routing.offline-route | A user with no internet and a downloaded map shall obtain a route between any two reachable points in that region. | Disable network; pick two addressable points in a downloaded region; observe a route is produced. |
| REQ.routing.travel-modes | The user shall be able to compute routes appropriate to walking, cycling, driving, and public transport. | For each profile, compute a route; observe a plausible result. |
| REQ.routing.intermediate-waypoints | The user shall be able to add intermediate waypoints to a planned route. | Add 1–3 waypoints; observe the route passes through them in order. |
| REQ.routing.recompute-on-deviation | When the user's position falls away from the current route, the Machine shall produce an updated route without further user input. | Drive off-route; observe a new route appears without prompting. |
| REQ.routing.avoid-classes | The user shall be able to avoid named classes of road or feature. | Toggle each avoidance; observe the route obeys the toggle. |
| REQ.routing.cancel | The user shall be able to cancel an active navigation session at any time and return to map browsing. | Tap stop-navigation; observe map view restored and guidance silenced. |

### 9.2 Voice guidance

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.guidance.voice-announcement | The user shall be informed of each upcoming manoeuvre by spoken announcement, without needing to look at the screen. | Drive a multi-turn route eyes-off-screen; observe every turn announced audibly. |
| REQ.guidance.announce-in-advance | Each announcement shall reach the user in time to act safely at the profile's typical speed. | Time the gap between announcement and turn; gap allows safe execution. |
| REQ.guidance.user-language | Announcements shall be in the user's chosen language when available on the device. | Switch device TTS language; observe announcements follow. |
| REQ.guidance.glanceable-screen | While navigating, the user shall see at a glance: next manoeuvre, distance to it, road name, and ETA. | One-second glance test; all four data items legible. |
| REQ.guidance.speed-limit-warn | The user shall be warned when observed speed exceeds the limit for the current road. | Travel above the limit on a tagged road; observe a warning. |
| REQ.guidance.lane-hints | The user shall, where lane data is available, see lane-guidance hints before complex junctions. | Approach a junction with turn:lanes data; observe a lane-hint overlay. |

### 9.3 Map viewing

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.map.offline-render | The user shall be able to view a downloaded region's map with no internet connection. | Disable network; pan over a downloaded region; observe map renders. |
| REQ.map.position-orientation | The user's current position and direction of travel shall be shown on the map. | Move outdoors; observe position marker and bearing updating. |
| REQ.map.rotate-with-direction | The user shall be able to choose whether the map rotates with compass or direction of motion, or stays north-up. | Toggle between modes; observe rotation behaviour matches the selected mode. |
| REQ.map.day-night | The map shall automatically darken at night and lighten during the day by default. | Cross civil twilight; observe the style switches. |
| REQ.map.local-script-labels | The user shall be able to choose whether place names appear in the local script, in English, or in phonetic transliteration. | Switch the setting; observe labels in the selected form. |
| REQ.map.gpx-overlay | The user shall be able to display recorded or imported GPX tracks as overlays on the map. | Import a GPX; toggle visibility; observe overlay. |
| REQ.map.style-switch-runtime | The user shall be able to switch the rendering style without restarting the application. | Switch style while viewing a region; observe new style applied in-place. |

### 9.4 Search

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.search.address | The user shall be able to find places by address using offline data. | Search a known address in a downloaded region with no network; observe correct match. |
| REQ.search.poi | The user shall be able to find places by category and by name. | Search "pharmacy" in a downloaded region; observe results. |
| REQ.search.coordinates | The user shall be able to enter geographic coordinates and have the location shown on the map. | Enter each supported format; observe a pin at the correct location. |
| REQ.search.along-route | The user shall be able to search for POIs along an active route. | While navigating, search "fuel along route"; observe results limited to the route corridor. |

### 9.5 Map data management

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.data.download-region | The user shall be able to download regional map data from within the app. | Pick a region; tap download; observe completion and offline usability. |
| REQ.data.fresher-updates | The user shall be able to opt in to receive map updates more frequently than the monthly base cycle. | Subscribe to live updates; observe a fresher data signature after the next cycle. |
| REQ.storage.resume | A download interrupted by network loss or app death shall be resumable without restarting from byte zero. | Start a large download; kill mid-flight; relaunch; observe download resumes. |
| REQ.storage.validate | The Machine shall detect corrupted map files and offer a re-download rather than crashing. | Corrupt a map file byte; relaunch; observe a clear error and re-download option. |

### 9.6 Track recording

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.track.record-background | The user shall be able to record a GPX track while the screen is off or the app is in the background. | Start recording; lock screen; travel; verify recording continues. |
| REQ.track.stats | While recording, the user shall be able to see live speed, distance, and elapsed time. | Open the recording panel mid-trip; observe live numbers. |
| REQ.track.survive-kill | A track in progress shall survive unexpected app termination without losing points up to the last persisted moment. | Force-stop mid-trip; reopen; observe track recoverable with most points intact. |
| REQ.track.import-export | The user shall be able to import and export GPX tracks. | Import a known GPX; observe correct rendering. Export a recorded track; verify in an external tool. |
| REQ.track.upload-osm | The user shall be able to upload a recorded track to the OSM database. | Configure OSM account; upload; verify on osm.org. |

### 9.7 Profiles, favourites, settings

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.profile.multi | The user shall be able to maintain multiple named profiles with independent navigation, routing, voice, and rendering settings. | Create two profiles; switch; observe distinct settings. |
| REQ.favourites.save | The user shall be able to save a location as a Favourite with a custom name, icon, and group. | Save a Favourite; reopen later; observe pin and name. |
| REQ.settings.backup | The user shall be able to back up and restore all user data. | Back up to a file; wipe; restore; observe data returned. |
| REQ.settings.cloud-sync | The user shall, when signed in, be able to sync user data across multiple devices. | Modify a favourite on device A; observe it on device B. |

### 9.8 OSM contribution

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.osm.report-error | The user shall be able to report a map error as an OSM note from within the app. | Submit a note; verify it appears on osm.org. |
| REQ.osm.contribute-edit | The user shall be able to add or edit a POI and submit the edit to OSM, optionally deferred for offline submission later. | Add a POI offline; reconnect; observe upload; verify on osm.org. |

### 9.9 Third-party integration and plugins

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.plugin.enable-disable | The user shall be able to enable or disable any optional feature at runtime without reinstalling the app. | Toggle a plugin; observe its UI elements appear or disappear without restart. |
| REQ.thirdparty.control | A third-party app on the same device shall be able to start navigation, add markers, query location, and receive turn updates from OsmAnd, subject to user permission. | Bind a sample external app; trigger each operation; observe the corresponding effect in OsmAnd. |

### 9.10 Accounts and authentication

Accounts are optional. Core features — map viewing, routing, search, recording — work with no sign-in. Two account surfaces exist: **OsmAnd Cloud** for sync, and a **linked OSM account** for contributions.

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.account.optional | The core feature set shall be usable with no signed-in account. | Fresh install with no sign-in; exercise each core feature; observe no auth wall. |
| REQ.account.cloud-signin | The user shall be able to create or sign in to an OsmAnd Cloud account from inside the app. | Complete the sign-in flow; observe signed-in state. |
| REQ.account.cloud-signout | The signed-in user shall be able to sign out, ending the session and removing stored tokens. | Tap Sign out; observe session cleared. |
| REQ.account.osm-link | The user shall be able to link an OSM account to enable OSM contributions. | Complete OAuth in browser; return to app; observe a valid link. |
| REQ.account.session-survives-restart | Signed-in state shall survive app restart and device reboot while stored tokens remain valid. | Sign in; reboot; relaunch; observe still signed in. |
| REQ.account.silent-refresh | While a refresh token is valid, the user shall not be asked to re-authenticate for routine operations. | Wait beyond access-token lifetime; trigger a sync; observe no prompt. |

### 9.11 Observability and support

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.obs.local-log | The user shall be able to retrieve a local log file recording significant events of recent sessions. | Trigger a known event; open the log; observe a timestamped entry. |
| REQ.obs.opt-in-telemetry | Any transmission of diagnostic or crash data shall be opt-in and disabled by default. | Fresh install; verify no telemetry sent; opt in; verify reports begin. |
| REQ.obs.no-pii-by-default | Logs and crash reports shall not include personally identifying information unless the user explicitly opts in. | Inspect a default log; observe absence of GPS coordinates and credentials. |

### 9.12 Map interaction

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.interact.gestures | The user shall be able to pan, pinch-zoom, rotate, double-tap zoom, and tilt the map. | Perform each gesture; observe the expected viewport change. |
| REQ.interact.long-press-pin | A long-press on the map shall offer at minimum: navigate to here, save as favourite, share location. | Long-press anywhere; observe the menu with the three options. |

### 9.13 Platform and operational requirements *(summary)*

The following requirements apply but are not decomposed further, as they reflect standard platform and store obligations:

| ID | Requirement |
|---|---|
| REQ.notif.foreground | Active navigation and recording sessions shall show a persistent foreground notification with live status and controls. |
| REQ.audio.call-respects | Phone calls shall preempt voice guidance entirely; guidance shall resume after the call ends. |
| REQ.perm.graceful-denial | Denying a permission shall disable only the affected feature cleanly, not crash the app. |
| REQ.net.live-wifi-only | The user shall be able to restrict OsmAnd Live automatic updates to Wi-Fi only, per region. |
| REQ.upgrade.preserve-user-data | An app upgrade shall preserve all user data — favourites, tracks, settings, downloaded maps, profiles, and account state. |
| REQ.auto.android-auto | When connected to an Android Auto head unit, the user shall be able to navigate and search from the head-unit display. |

---

## 10. Quality requirements

| ID | Requirement | Acceptance |
|---|---|---|
| QUAL.perf.smooth-map | Map panning and zooming shall feel smooth on a mid-range Android device. | Median frame-time ≤ 33 ms per `SPEC.perf.measure-fps`. |
| QUAL.perf.route-under-10s | Route calculation shall complete within an interactive timeframe. | ≤ 10 s for routes up to 200 km; ≤ 5 s for routes ≤ 50 km; per `SPEC.perf.measure-route-time`. |
| QUAL.perf.reroute-under-5s | Re-routing after a deviation shall complete within 5 s. | Per `SPEC.perf.measure-reroute-time`. |
| QUAL.perf.startup-under-5s | Cold start to interactive map shall complete within 5 s on a mid-range device. | Per `SPEC.perf.measure-startup`. |
| QUAL.perf.battery-screen-off | Navigation with the screen off shall consume ≤ 40% of the power of screen-on navigation. | Per `SPEC.perf.measure-battery`. |
| QUAL.perf.obf-bounded-ram | Memory use shall scale with the visible viewport, not the total map file size. | Total resident memory ≤ 512 MB with any single regional map loaded; per `SPEC.storage.measure-ram`. |
| QUAL.reliability.no-crash-on-gps-loss | The app shall remain operational when GPS is lost or intermittent during navigation. | Cut GPS for 5 min and restore; observe no crash; guidance resumes. |
| QUAL.reliability.crash-budget | Field crash rate shall not exceed 0.1% of sessions for the headline build flavour. | Per `SPEC.obs.crash-rate-measurement`. |
| QUAL.usability.locales | The UI shall be available in ≥ 70 natural-language locales. | Per `SPEC.i18n.locale-count`. |
| QUAL.security.tls | All network traffic to OsmAnd-controlled endpoints shall use TLS 1.2 or higher. | Per `SPEC.net.tls-min`. |
| QUAL.security.no-plaintext-creds | OSM and OsmAnd-Cloud credentials shall not be stored in plaintext on disk. | Per `SPEC.sec.cred-storage`. |
| QUAL.privacy.offline-no-location-egress | While offline, the app shall make no outbound network call containing user location. | Capture device network while offline-routing; observe zero location-bearing packets. |

---

## 11. Constraints

| ID | Constraint |
|---|---|
| CON.org.gplv3 | The application source code shall be licensed under GPLv3. |
| CON.org.osm-only-data | Map data shall be derived exclusively from OpenStreetMap; no proprietary map content shall be bundled. |
| CON.legal.gdpr-no-location-upload | While offline, no user location data shall be transmitted to any OsmAnd-controlled server. |
| CON.legal.store-policies | Published builds shall comply with Google Play and Apple App Store distribution policies. |
| CON.legal.osm-contributor-terms | All uploads to the OSM API shall comply with the OpenStreetMap Contributor Terms. |
| CON.platform.android-min-sdk | The Android edition shall support Android 7.0 and above. |
| CON.platform.ios-min-version | The iOS edition shall support iOS 12.0 and above. |
| CON.platform.freemium-quota | The free Google Play flavour shall limit users to 7 map-file downloads; the paid flavour shall have no quota. |

---

## 12. Specifications

This is the only section that may name concrete algorithms, file formats, version numbers, byte sizes, protocols, and HTTP endpoints. Internal class names are confined to §14.

### 12.1 Routing

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.routing.bidirectional-astar | Route calculation shall be performed by bidirectional A\* search over a directed weighted graph extracted from the map file road-network index. Edge weight: `distance / speed + turn_penalty`. All profile constants shall be defined in external `routing.xml` configuration; none shall be hard-coded. | PHEN.navigate-tap |
| SPEC.routing.hh-long-distance | For routes above a configurable distance threshold, a Hierarchical Highway (HH) contraction-hierarchy algorithm shall be used in place of A\*, operating over a pre-computed hierarchy stored in the map file. | PHEN.navigate-tap |
| SPEC.routing.offline-only | The route-calculation path shall not perform any network call; it reads only on-device map files. The optional online router is available only via a profile-level user opt-in. | PHEN.navigate-tap, PHEN.network-available |
| SPEC.routing.deviation-threshold | An off-route condition shall be raised when the device position deviates from the nearest route segment continuously over ≥ 3 consecutive fixes. | PHEN.gps-fix, PHEN.route-displayed |
| SPEC.routing.reroute-5s | Upon an off-route condition, a new route to the same destination shall be produced within 5 s on a mid-range device. | PHEN.route-displayed |

### 12.2 Map rendering

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.render.opengl-primary | Vector rendering shall use a hardware-accelerated OpenGL ES 2.0 pipeline via a native C++ library linked over JNI. A Java 2D Canvas fallback shall be available for devices without OpenGL ES 2.0. | PHEN.map-frame |
| SPEC.render.rules-xml | Rendering rules shall be defined in XML style files specifying display conditions per zoom level, object type, and OSM tag combination. A style change shall take effect on the next frame without restarting the app or reloading map data. | PHEN.map-frame |
| SPEC.render.day-night | Day/night switching shall be implemented as a rendering-style parameter. Automatic switching shall use civil twilight time computed from the current location and date. | PHEN.map-frame |
| SPEC.render.position-marker | The position marker shall be redrawn within 1 s of each `PHEN.gps-fix`. Bearing shall be smoothed with a low-pass filter to suppress GNSS heading jitter at low speeds. | PHEN.gps-fix, PHEN.map-frame |

### 12.3 Map data — format and storage

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.data.obf-format | Map data shall use the OsmAnd Binary Format (OBF), a compact spatially-indexed binary compiled from OSM data by external tooling. The app shall not parse raw OSM XML or PBF at runtime. | PHEN.obf-present |
| SPEC.data.obf-sections | An `.obf` file shall contain a road-network index, a rendered-map-object index, an address index, a POI index, and a public-transport index. Each shall be spatially indexed for tile-based access. | PHEN.obf-present |
| SPEC.data.live-diffs | Live updates shall be binary diff files applied to the base `.obf`, available at minimum hourly cadence for subscribed users. | PHEN.obf-download |
| SPEC.storage.memory-mapped-obf | Map file reads shall use memory-mapped I/O so resident RAM scales with the active viewport and is bounded by `QUAL.perf.obf-bounded-ram`. | PHEN.obf-present |
| SPEC.data.download-quota | The free Google Play flavour shall enforce a 7-file download cap; the paid flavour shall enforce no cap. | PHEN.obf-download |
| SPEC.storage.resume | Region downloads shall use HTTP `Range` requests to resume from a partial-file offset on retry, avoiding re-download of already-received bytes. | PHEN.obf-download, PHEN.download-progress |
| SPEC.storage.validate | Each downloaded map file shall be verified for header validity and expected size before being declared complete; corrupted files shall be moved out of the active maps directory. | PHEN.obf-download |

### 12.4 Search

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.search.offline-only | All address, POI, coordinate, and along-route queries shall be resolved against on-device map indexes without network access. Online Wikipedia lookup is the sole exception and shall be clearly marked. | PHEN.search-query |
| SPEC.search.address-hierarchy | Address search shall follow the hierarchy country → region → city → street → house-number, each level matched with prefix and fuzzy matching against the map address index. | PHEN.search-query |
| SPEC.search.along-route-corridor | Along-route search shall restrict POI results to amenities within a configurable corridor of the active route polyline. | PHEN.search-query |

### 12.5 Location / GPS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.location.android-api | The Android edition shall subscribe to location through the OS Location API, requesting updates at ≥ 1 Hz and ≥ 1 m displacement while navigating or recording. | PHEN.gps-fix |
| SPEC.location.foreground-service | Background navigation and track recording shall run inside an Android Foreground Service with a persistent notification, holding any required wake-locks. | PHEN.gps-fix, PHEN.gps-loss |
| SPEC.location.tolerate-loss | Loss of location for ≤ 5 min shall not terminate an in-progress navigation or recording session; guidance shall resume when fixes return. | PHEN.gps-loss, PHEN.gps-fix |
| SPEC.location.speed-limit | Speed-limit warning shall fire when the current speed exceeds the `maxspeed` value for the current road by a configurable tolerance factor. | PHEN.gps-fix, PHEN.alert-tone |

### 12.6 Voice / TTS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.voice.delivery | Spoken instructions shall be delivered through the OS Text-to-Speech service or pre-recorded audio packs, as the user selects. | PHEN.tts-utter |
| SPEC.voice.advance-distances | Manoeuvre announcements shall be triggered at configurable distances; defaults: 2000 m, 500 m, and 50 m before the manoeuvre. | PHEN.tts-utter |
| SPEC.voice.command-script | Per-language voice grammars shall be defined as JavaScript files executed by an embedded scripting engine, enabling language-pack updates independent of the app binary. | PHEN.tts-utter |
| SPEC.voice.audio-focus | Before each TTS prompt, the Machine shall request transient audio focus from the OS so that other media ducks or pauses for the duration of the prompt. The Machine shall yield audio focus immediately on an incoming call. | PHEN.tts-utter |

### 12.7 GPX

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.gpx.schema | Recorded and exported tracks shall conform to GPX 1.1. Track points shall carry `lat`, `lon`, `ele`, and `time`. OsmAnd-specific extensions shall be written under the `osmand:` XML namespace. | PHEN.gpx-write |
| SPEC.gpx.statistics | Live track statistics shall be computed using the Haversine formula for distance and forward-difference derivatives for speed. | PHEN.gps-fix |
| SPEC.gpx.persist-each-fix | Each `PHEN.gps-fix` while recording shall be persisted to the in-progress track file before the next fix arrives. | PHEN.gps-fix, PHEN.gpx-write |

### 12.8 Settings and profiles

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.profile.applicationmode | Each profile shall be a named configuration instance; per-profile settings shall be namespaced by mode key in the app's preference store. | PHEN.profile-select |
| SPEC.settings.osf | Settings export/import shall use the `.osf` archive format containing settings JSON, GPX tracks, favourites XML, and per-profile rendering style files. Restore shall be atomic. | — |
| SPEC.settings.cloud-sync | The cloud sync subsystem shall synchronise the user's settings, favourites, and GPX tracks with the OsmAnd Cloud service over TLS, with conflict resolution by timestamp. | — |

### 12.9 Third-party API and plugins

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.api.aidl-surface | The public IPC surface shall expose method groups over Android AIDL for: navigation, route, location, search, favourites, map markers, GPX, map layers, widgets, settings, and context menus. Callbacks shall be delivered via a registered AIDL callback interface. | PHEN.aidl-call, PHEN.aidl-callback |
| SPEC.api.permission-gated | All AIDL methods that change Machine state shall be gated behind a user-grantable permission requested at first bind. | PHEN.aidl-call |
| SPEC.plugin.base-contract | Every in-tree plugin shall extend a common abstract plugin base class. The host shall not reference plugin classes directly outside the defined extension-point hooks. | — |
| SPEC.plugin.runtime-enable | Enabling or disabling a plugin shall not require a process restart; layer, widget, and preference hooks shall be re-invoked on the next relevant lifecycle event. | — |

### 12.10 Identity and credential lifecycle

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.account.cloud-email-flow | OsmAnd Cloud registration and sign-in shall use an email plus verification-code flow served over TLS. | PHEN.auth-token-issued |
| SPEC.account.osm-oauth2 | OSM account linking shall use OAuth 2.0 with PKCE against `https://www.openstreetmap.org/oauth2/authorize`, with scopes limited to notes, POI edits, and GPX uploads, and a registered redirect URI validated against the `state` parameter. | PHEN.auth-token-issued |
| SPEC.account.token-storage | Access and refresh tokens shall be stored in the platform's encrypted credential store. Tokens shall not appear in logs, GPX files, exported archives, or crash reports. | — |
| SPEC.account.token-refresh | A 401 or token-expired response shall trigger a silent refresh using the stored refresh token. On refresh failure, the session is marked invalid and the user is prompted to sign in only on their next authenticated action. | PHEN.auth-token-revoked |
| SPEC.account.signout-clears-local | On sign-out or unlink, the Machine shall delete the access token, refresh token, and cached account profile. Non-account-bound user data shall not be touched. | PHEN.auth-token-revoked |

### 12.11 Logging and observability

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.obs.logger | Logging shall route to Android Log and to a rolling on-device log file. Supported levels: `error`, `warn`, `info`, `debug`. | PHEN.log-line |
| SPEC.obs.local-log-retention | The local log file shall retain at least the last 7 days or 5 MB, whichever is larger, rotating older content. | PHEN.log-line |
| SPEC.obs.log-no-pii | Logger calls in production shall not emit GPS coordinates, OAuth tokens, or account credentials. A redaction test shall enforce this rule. | PHEN.log-line |
| SPEC.obs.crash-channel | Crash reports shall be written locally. Upload to a remote channel shall occur only when the user has opted in. | PHEN.crash-report |
| SPEC.obs.opt-in-telemetry | All telemetry and crash uploads shall be controlled by a user preference defaulting to off. | PHEN.crash-report |

### 12.12 Network and TLS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.net.tls-min | All HTTP traffic to OsmAnd-controlled endpoints shall use TLS 1.2 or higher. Plain HTTP shall not be used for any data transfer. | PHEN.obf-download, PHEN.osm-upload |
| SPEC.net.osm-endpoints | OSM API traffic shall target `https://api.openstreetmap.org/api/0.6/…`. | PHEN.osm-upload |
| SPEC.net.no-offline-egress | When offline mode is active, no outbound socket shall be opened by routing, rendering, voice, or location code paths. | PHEN.network-available |
| SPEC.net.live-wifi-only | The OsmAnd Live per-map Wi-Fi-only flag shall be honoured by both the update scheduler and executor; updates shall be deferred on metered networks when the flag is set. | PHEN.obf-download, PHEN.network-available |

### 12.13 Performance — measurement methods

| ID | Specification |
|---|---|
| SPEC.perf.measure-fps | Frame-time measured via the display's frame callback during a scripted 60 s pan/zoom over a downloaded region on a reference device. |
| SPEC.perf.measure-route-time | Route time measured from `PHEN.navigate-tap` to `PHEN.route-displayed` over a curated corpus of 50 routes per profile per region. |
| SPEC.perf.measure-reroute-time | Reroute time measured from `SPEC.routing.deviation-threshold` firing to `PHEN.route-displayed`. |
| SPEC.perf.measure-startup | Cold start measured from `am start` to first interactive frame, after force-stop and cache clear. |
| SPEC.perf.measure-battery | Battery drain measured via the OS battery statistics subsystem over a 1 h scripted drive, screen on vs screen off. |
| SPEC.storage.measure-ram | Resident memory measured via the OS memory-info API after 5 min of pan/zoom over the largest enabled region. |
| SPEC.i18n.locale-count | Count of locale-specific string resource bundles with ≥ 90% key coverage of the default bundle. |
| SPEC.sec.cred-storage | Credentials stored in Android `EncryptedSharedPreferences` (Android) / iOS Keychain (iOS). |
| SPEC.obs.crash-rate-measurement | Crash-free-session rate: `1 − crashes_with_consent / sessions_with_consent`, aggregated weekly per build flavour. |

---

## 13. Justification — `D, S ⊢ R`

| REQ | Entailing DOM | Entailing SPEC | Argument |
|---|---|---|---|
| REQ.routing.offline-route | DOM.osm.tag-conventions, DOM.fs.persistence | SPEC.routing.offline-only, SPEC.routing.bidirectional-astar, SPEC.data.obf-format, SPEC.data.obf-sections | OSM road graph is encoded in the `.obf` road-network index; the file persists on disk; A\* runs over it without network. |
| REQ.routing.recompute-on-deviation | DOM.gps.fix-outdoors | SPEC.location.android-api, SPEC.routing.deviation-threshold, SPEC.routing.reroute-5s | Fixes arrive at ≥ 1 Hz; deviation rule fires after 3 consecutive off-route fixes; reroute delivers a new route within 5 s. |
| REQ.guidance.voice-announcement | DOM.android.tts, DOM.audio.driving-noise | SPEC.voice.delivery, SPEC.voice.advance-distances, SPEC.voice.audio-focus | TTS is available; the Machine emits utterances in advance and claims audio focus so they are audible above road noise. |
| REQ.guidance.announce-in-advance | DOM.user.attention | SPEC.voice.advance-distances | The 2000/500/50 m defaults provide head-room for any profile's typical speed. |
| REQ.map.offline-render | DOM.fs.persistence | SPEC.data.obf-format, SPEC.data.obf-sections, SPEC.render.opengl-primary, SPEC.render.rules-xml, SPEC.storage.memory-mapped-obf | All data and rules to paint a tile are local; the renderer runs without network. |
| REQ.search.address | DOM.osm.tag-conventions | SPEC.search.offline-only, SPEC.search.address-hierarchy, SPEC.data.obf-sections | Address index is on-disk; the search engine queries it without network. |
| REQ.track.survive-kill | DOM.power.kill, DOM.fs.persistence | SPEC.gpx.persist-each-fix, SPEC.location.foreground-service | Each fix is persisted before the next; the foreground service reduces kill probability; the on-disk file is recoverable. |
| REQ.navigation.tolerate-gps-loss | DOM.gps.intermittent | SPEC.location.tolerate-loss, SPEC.location.foreground-service | Loss for ≤ 5 min is explicitly tolerated; the service keeps the process alive; guidance resumes on next fix. |
| REQ.obs.no-pii-by-default | — | SPEC.obs.log-no-pii, SPEC.obs.opt-in-telemetry | Production logger calls do not emit PII; remote upload is gated behind opt-in. |
| QUAL.privacy.offline-no-location-egress | DOM.network.intermittent | SPEC.net.no-offline-egress, SPEC.routing.offline-only, SPEC.search.offline-only | No offline routing/rendering/search code path opens a socket. |
| REQ.thirdparty.control | DOM.android.location-api | SPEC.api.aidl-surface, SPEC.api.permission-gated | The AIDL surface exposes the required method groups; permission gating preserves user control. |
| REQ.account.cloud-signin | DOM.account.cloud-account-via-server | SPEC.account.cloud-email-flow, SPEC.net.tls-min, SPEC.account.token-storage | The email flow drives the OsmAnd Cloud HTTP service over TLS; the service issues a token; the Machine stores it encrypted. |
| REQ.account.osm-link | DOM.account.osm-account-external | SPEC.account.osm-oauth2, SPEC.account.token-storage | The user authenticates against osm.org via PKCE; the Machine validates the `state` parameter and stores the token encrypted. |
| REQ.account.session-survives-restart | DOM.fs.persistence, DOM.account.tokens-expire | SPEC.account.token-storage | Tokens are persisted in encrypted storage and remain valid across process lifetimes. |
| REQ.account.silent-refresh | DOM.account.tokens-expire | SPEC.account.token-refresh | The refresh token exchanges a new access token without user interaction; the user is prompted only when the refresh itself fails. |
| REQ.storage.resume | DOM.network.intermittent | SPEC.storage.resume | HTTP `Range` requests resume from the recorded partial-file offset. |
| REQ.net.live-wifi-only | DOM.network.metered | SPEC.net.live-wifi-only | Both the scheduler and executor check the per-map flag before any cellular transfer. |

---

## 14. Specification → Module / Class *(informative)*

This section maps each `SPEC.*` family to the code that realises it. It is non-normative and does not participate in the `D, S ⊢ R` argument.

| SPEC | Realised in |
|---|---|
| SPEC.routing.* | `RoutingHelper`, `BinaryRoutePlanner`, `HHRoutePlanner`, `routing.xml` |
| SPEC.render.* | `OsmandRenderer`, `MapRenderRepositories`, `RenderingRulesStorage`; OsmAnd-core (C++ JNI) |
| SPEC.data.* / SPEC.storage.* | `BinaryMapIndexReader`, `IndexConstants`, `DownloadFileHelper`, `DownloadValidationManager` |
| SPEC.search.* | `SearchUICore`, `SearchPhrase`, `SearchResultMatcher` |
| SPEC.location.* | `OsmAndLocationProvider`, `NavigationService`, `GpsStatusListener` |
| SPEC.voice.* | `VoiceRouter`, `CommandPlayer`, `JsCommandBuilder`, `JsTtsCommandPlayer` |
| SPEC.gpx.* | `GpxFile`, `Track`, `WptPt`, `SaveGpxHelper`, `TrackDisplayHelper` |
| SPEC.profile.* / SPEC.settings.* | `OsmandSettings`, `ApplicationMode`, `BackupHelper`, `NetworkSettingsHelper` |
| SPEC.api.* / SPEC.plugin.* | `OsmandAidlService`, `IOsmAndAidlInterface`, `OsmandPlugin` base + per-plugin subclasses |
| SPEC.account.* | `BackupHelper`, `OsmOAuthHelper`, `AuthorizeFragment`, `DeleteAccountFragment`; Android `EncryptedSharedPreferences` |
| SPEC.obs.* | `PlatformUtil` logger, `osmand_log.txt` |
| SPEC.net.* | `AndroidNetworkUtils`, HTTP clients in `OsmAnd-shared` |

---

## 15. Glossary

| Term | Definition |
|---|---|
| World | Everything outside the software-to-be that the software cares about — users, environment, external systems. |
| Machine | The OsmAnd Android application: its external run-time shape (§6.1) and internal module structure (§6.2). |
| Shared phenomenon | An event or state both the World and the Machine can observe or cause; the only vocabulary §12 may use. |
| `D, S ⊢ R` | Domain assumptions plus specifications entail the requirement. |
| `.obf` | OsmAnd Binary Format — compact indexed binary map data compiled from OSM. |
| OsmAnd Live | Subscription service providing hourly incremental map updates. |
| GPX | GPS Exchange Format — XML standard for tracks and waypoints. |
| `.osf` | OsmAnd Settings File — archive of settings, tracks, favourites, and style overrides. |
| POI | Point of Interest — named geographic feature. |
| A\* | A-star bidirectional pathfinding algorithm used for short and medium routes. |
| HH routing | Hierarchical Highway routing — contraction-hierarchy algorithm used for long-distance routes. |
| TTS | Text-to-Speech — OS service synthesising spoken instructions. |
| AIDL | Android Interface Definition Language — OsmAnd's third-party IPC interface. |
| Profile | A named configuration set for a travel mode (`ApplicationMode`). |
| Plugin | An optional feature module extending the Machine through defined hooks. |
| GNSS | Global Navigation Satellite System — covers GPS, GLONASS, Galileo, BeiDou. |
| OAuth 2.0 | Open standard for delegated authorisation; OsmAnd uses it with PKCE for OSM account linking. |
| Foreground service | An Android service with a persistent notification, exempt from aggressive background-process killing. |
| Metered network | A network the OS reports as charging per byte — typically cellular. |

---

## 16. References

1. **Michael Jackson**, *Software Requirements & Specifications*, ACM Press / Addison-Wesley, 1995.
2. **Michael Jackson**, *Problem Frames*, Addison-Wesley, 2001.
3. **Pamela Zave and Michael Jackson**, "Four Dark Corners of Requirements Engineering," *ACM TOSEM*, 6:1–30, 1997.
4. **Ian Sommerville**, *Software Engineering*, 10th ed., Pearson, 2015.
5. **OpenStreetMap Wiki**, "Map Features" and "API v0.6" — https://wiki.openstreetmap.org/
6. **GPX 1.1 schema**, Topografix — https://www.topografix.com/GPX/1/1/
7. **OsmAnd repository** — https://github.com/osmandapp/OsmAnd
