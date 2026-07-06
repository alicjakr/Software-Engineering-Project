# Requirements Specification — OsmAnd
## World / Machine Model

**Project:** OsmAnd
**Repository:** https://github.com/osmandapp/OsmAnd
**Version:** 3.0
**Date:** June 2026

Structure: §1 scope · §2 method · §3 actors · §4 external domains · §5 world context · §6 the machine · §7 shared phenomena · §8 domain assumptions · §9 requirements · §10 quality requirements · §11 constraints · §12 specifications · §13 D, S ⊢ R justification · §14 glossary · §15 references.

---

## 1. Scope

OsmAnd is an open-source map-viewing and turn-by-turn navigation application for Android and iOS, built on OpenStreetMap data. Its defining property is **full offline operation**: all core features — routing, vector map rendering, address and POI search, GPX recording, voice guidance — must work without any network connection, using pre-downloaded binary map files.

This document covers the Android edition. The iOS edition shares the cross-platform modules and inherits the same requirements.

Out of scope: OsmAnd MapCreator; OsmAnd Cloud server-side implementation; web map services.

---

## 2. Method

This document follows Michael Jackson's World/Machine Model (WMM):

- **R — Requirement.** What the user needs, in the user's vocabulary. No algorithms, file formats, version numbers, or API names.
- **S — Specification.** What the Machine must do at its boundary with the world. May name protocols, algorithms, and measurable thresholds.
- **W — Domain assumption.** How the World behaves, independent of the Machine. Indicative mood.
- **D, S ⊢ R** — Domain assumptions together with specifications must entail the requirements. §13 argues this for headline requirements.

Sections §3–§5 and §8 are **indicative** (describe the World as it is). Sections §9–§12 are **optative** (state what we want once the Machine is in place).

---

## 3. Actors

| Actor | Role |
|---|---|
| End user (navigator) | Operates the device while travelling; consumes map, route, and voice output |
| End user (recorder) | Records a GPX track for later use or upload |
| OSM contributor | Edits OSM through OsmAnd — adding POIs, reporting notes, uploading tracks |
| Third-party developer | Binds to OsmAnd's AIDL service to start navigation or query location |
| OsmAnd team | Operates the build and servers; accepts contributions |

---

## 4. External domains

| Domain | Class | Role |
|---|---|---|
| Global Navigation Satellite System | causal | Delivers position fixes via the device's GNSS chip |
| OpenStreetMap database | lexical | Source of all map content, compiled into `.obf` files |
| OSM API | biddable | Accepts POI edits, notes, and GPX uploads |
| OsmAnd map servers | biddable | Hosts `.obf` regional files and OsmAnd Live diff feeds |
| OsmAnd Cloud | biddable | Cross-device sync for user data |
| Online raster tile providers | biddable | Optional overlay sources (Bing satellite, etc.) |
| Android OS | causal | Provides Location API, TTS, foreground services, audio, notifications |
| Device filesystem | lexical | Persistent storage for `.obf`, `.gpx`, settings, voice packs |
| OS Text-to-Speech engine | causal | Synthesises spoken instructions |
| Internet | causal | Used for downloads, sync, online tiles, OSM uploads; not required for offline use |

---

## 5. World context

### 5.1 User context

Users operate OsmAnd while moving — on foot, by bicycle, in a vehicle. Attention and hand availability are limited: a driver cannot read fine text; a cyclist cannot tap precisely. Internet is often unavailable or expensive. Users speak many languages and read many scripts.

### 5.2 GNSS behaviour

Fix accuracy is typically 3–15 m in open sky but degrades in urban canyons, tunnels, and indoors. Fixes may be intermittent or briefly lost.

### 5.3 Map data and network

OsmAnd's map content derives entirely from OpenStreetMap. Coverage and quality vary by region. Mobile data is intermittent or absent in many real-world locations; the app must function fully offline.

### 5.4 Device constraints

Battery life is finite; continuous GPS and screen-on navigation can drain a phone in under 6 hours. The OS may kill background processes aggressively when the screen is off.

---

## 6. The Machine

### 6.1 External run-time shape

The Machine is the **OsmAnd Android application** installed on a single user's device. It runs as an Activity-based app with a foreground service for background sessions and a persistent settings store. It has no server-side counterpart at runtime.

### 6.2 Internal modules *(informative)*

| Module | Languages | Responsibility |
|---|---|---|
| `OsmAnd` | Java + Kotlin | Android UI, activity lifecycle, foreground service, plugin host, AIDL host |
| `OsmAnd-java` | Java | Platform-neutral core: `.obf` reading, A\* routing, search, rendering rules — no Android dependencies |
| `OsmAnd-shared` | Kotlin Multiplatform | Cross-platform domain models, GPX, settings, coroutines, SQL access |
| `OsmAnd-api` | AIDL + Java | Public IPC surface for third-party Android apps |

---

## 7. Shared phenomena

Only phenomena listed here may appear in specification clauses.

| Phenomenon | Type | Controlled by | Observed by |
|---|---|---|---|
| User taps "Navigate" with a destination | event | User | Machine |
| User submits a search query | event | User | Machine |
| User selects a profile | event | User | Machine |
| User performs a touch gesture on the map | event | User | Machine |
| User long-presses a point on the map | event | User | Machine |
| GNSS reports a position fix | event | GNSS | Machine |
| Location updates cease for ≥ N seconds | state | GNSS | Machine |
| Internet reachability of a given endpoint | state | Internet | Machine |
| `.obf` file for a region exists on the filesystem | state | Filesystem + Machine | both |
| A route polyline and instructions appear on screen | state | Machine | User |
| Spoken instruction emitted from speaker | event | Machine | User |
| Warning tone emitted | event | Machine | User |
| A rendered map frame is displayed | event | Machine | User |
| Bytes written to a `.gpx` file | event | Machine | Filesystem |
| HTTP request/response for a `.obf` region file | event | Machine | OsmAnd servers |
| HTTP request to OSM API with a contribution payload | event | Machine | OSM API |
| IPC method call into the OsmAnd AIDL service | event | Third-party app | Machine |
| Callback delivered to a bound third-party app | event | Machine | Third-party app |
| Log line written to the on-device log file | event | Machine | Filesystem |
| Crash diagnostic emitted | event | Machine | Filesystem / Internet |
| Identity provider issues a session token | event | OsmAnd Cloud / OSM API | Machine |
| Token invalidated by user or provider | event | User / provider | Machine |
| Machine holds a valid token for a given identity | state | Machine | User-facing surfaces |
| Region download advances or completes | event | Machine | User |

---

## 8. Domain assumptions

| ID | Assumption |
|---|---|
| W-1 | Outdoors with clear sky, a consumer GNSS receiver produces fixes at ≥ 1 Hz with horizontal accuracy ≤ 15 m. |
| W-2 | Indoors, in tunnels, or in dense urban canyons, fixes may be absent or accuracy may exceed 50 m. |
| W-3 | A single position fix may be absent for up to several minutes in covered areas; service resumes when sky is visible. |
| W-4 | Roads in OSM are tagged with `highway=*`; speed limits in `maxspeed`; turn restrictions in relations. |
| W-5 | A `.obf` snapshot is out of date with the live OSM database from the moment it is compiled. |
| W-6 | The Android OS exposes location through its Location API; updates are delivered at a requested rate. |
| W-7 | Android exposes a Text-to-Speech service with at least one language pre-installed. |
| W-8 | An Android foreground service with a persistent notification is not killed by the OS during normal use. |
| W-9 | With the screen off and no foreground service, the OS throttles background CPU and network. |
| W-10 | In-vehicle background noise is 60–75 dBA; spoken instructions must be audible above it. |
| W-11 | The user while driving cannot reliably read fine on-screen text; instructions must be delivered in advance and audibly. |
| W-12 | Users speak many natural languages and read many scripts. |
| W-13 | Continuous GPS and screen-on navigation can drain a phone in under 6 hours. |
| W-14 | Mobile data is intermittent or absent in many real-world locations. |
| W-15 | Files written to internal or SD-card storage persist across reboots and app updates. |
| W-16 | The OS may kill the app process at any time when the screen is off and no foreground service is running. |
| W-17 | OSM contributions require a registered OSM account; the API expects OAuth 2.0 credentials. |
| W-18 | OsmAnd Cloud accounts are created via the OsmAnd Cloud HTTP service; account creation requires network. |
| W-19 | OAuth access tokens have finite lifetime; refresh tokens may be revoked without notice. |
| W-20 | The Android OS exposes whether the current network is metered. |

---

## 9. Requirements

### 9.1 Navigation and routing

| ID | Requirement | Acceptance |
|---|---|---|
| R-NAV-1 | A user with no internet and a downloaded map shall obtain a route between any two reachable points in that region. | Disable network; pick two addressable points in a downloaded region; observe a route is produced. |
| R-NAV-2 | The user shall be able to compute routes appropriate to walking, cycling, driving, and public transport. | For each profile, compute a route; observe a plausible result. |
| R-NAV-3 | The user shall be able to add intermediate waypoints to a planned route. | Add 1–3 waypoints; observe the route passes through them in order. |
| R-NAV-4 | When the user's position falls away from the current route, the Machine shall produce an updated route without further user input. | Drive off-route; observe a new route appears without prompting. |
| R-NAV-5 | The user shall be able to avoid specific road types or features. | Toggle each avoidance; observe the route obeys the toggle. |
| R-NAV-6 | The user shall be able to cancel an active navigation session at any time and return to map browsing. | Tap stop-navigation; observe map view restored and guidance silenced. |

### 9.2 Voice guidance

| ID | Requirement | Acceptance |
|---|---|---|
| R-VOI-1 | The user shall be informed of each upcoming manoeuvre by spoken announcement, without needing to look at the screen. | Drive a multi-turn route eyes-off-screen; observe every turn announced audibly. |
| R-VOI-2 | Each announcement shall reach the user in time to act safely at the profile's typical speed. | Time the gap between announcement and turn; gap allows safe execution. |
| R-VOI-3 | Announcements shall be in the user's chosen language when available on the device. | Switch device TTS language; observe announcements follow. |
| R-VOI-4 | While navigating, the user shall see at a glance: next manoeuvre, distance to it, road name, and ETA. | One-second glance test; all four data items legible. |
| R-VOI-5 | The user shall be warned when observed speed exceeds the limit for the current road. | Travel above the limit on a tagged road; observe a warning. |
| R-VOI-6 | The user shall, where lane data is available, see lane-guidance hints before complex junctions. | Approach a junction with turn:lanes data; observe a lane-hint overlay. |

### 9.3 Map viewing

| ID | Requirement | Acceptance |
|---|---|---|
| R-MAP-1 | The user shall be able to view a downloaded region's map with no internet connection. | Disable network; pan over a downloaded region; observe map renders. |
| R-MAP-2 | The user's current position and direction of travel shall be shown on the map. | Move outdoors; observe position marker and bearing updating. |
| R-MAP-3 | The user shall be able to choose whether the map rotates with compass or direction of motion, or stays north-up. | Toggle between modes; observe rotation behaviour matches the selected mode. |
| R-MAP-4 | The map shall automatically darken at night and lighten during the day by default. | Cross civil twilight; observe the style switches. |
| R-MAP-5 | The user shall be able to choose whether place names appear in the local script, in English, or in phonetic transliteration. | Switch the setting; observe labels in the selected form. |
| R-MAP-6 | The user shall be able to display recorded or imported GPX tracks as overlays on the map. | Import a GPX; toggle visibility; observe overlay. |
| R-MAP-7 | The user shall be able to switch the rendering style without restarting the application. | Switch style while viewing a region; observe new style applied in-place. |

### 9.4 Search

| ID | Requirement | Acceptance |
|---|---|---|
| R-SRH-1 | The user shall be able to find places by address using offline data. | Search a known address in a downloaded region with no network; observe correct match. |
| R-SRH-2 | The user shall be able to find places by category and by name. | Search "pharmacy" in a downloaded region; observe results. |
| R-SRH-3 | The user shall be able to enter geographic coordinates and have the location shown on the map. | Enter each supported format; observe a pin at the correct location. |
| R-SRH-4 | The user shall be able to search for POIs along an active route. | While navigating, search "fuel along route"; observe results limited to the route corridor. |

### 9.5 Map data management

| ID | Requirement | Acceptance |
|---|---|---|
| R-DAT-1 | The user shall be able to download regional map data from within the app. | Pick a region; tap download; observe completion and offline usability. |
| R-DAT-2 | The user shall be able to opt in to receive map updates more frequently than the monthly base cycle. | Subscribe to live updates; observe a fresher data signature after the next cycle. |
| R-DAT-3 | A download interrupted by network loss or app death shall be resumable without restarting from byte zero. | Start a large download; kill mid-flight; relaunch; observe download resumes. |
| R-DAT-4 | The Machine shall detect corrupted map files and offer a re-download rather than crashing. | Corrupt a map file byte; relaunch; observe a clear error and re-download option. |

### 9.6 Track recording

| ID | Requirement | Acceptance |
|---|---|---|
| R-TRK-1 | The user shall be able to record a GPX track while the screen is off or the app is in the background. | Start recording; lock screen; travel; verify recording continues. |
| R-TRK-2 | While recording, the user shall be able to see live speed, distance, and elapsed time. | Open the recording panel mid-trip; observe live numbers. |
| R-TRK-3 | A track in progress shall survive unexpected app termination without losing points up to the last persisted moment. | Force-stop mid-trip; reopen; observe track recoverable with most points intact. |
| R-TRK-4 | The user shall be able to import and export GPX tracks. | Import a known GPX; observe correct rendering; export a recorded track; verify in an external tool. |
| R-TRK-5 | The user shall be able to upload a recorded track to the OSM database. | Configure OSM account; upload; verify on osm.org. |

### 9.7 Profiles, favourites, settings

| ID | Requirement | Acceptance |
|---|---|---|
| R-PRF-1 | The user shall be able to maintain multiple named profiles with independent navigation, routing, voice, and rendering settings. | Create two profiles; switch; observe distinct settings. |
| R-PRF-2 | The user shall be able to save a location as a Favourite with a custom name, icon, and group. | Save a Favourite; reopen later; observe pin and name. |
| R-PRF-3 | The user shall be able to back up and restore all user data. | Back up to a file; wipe; restore; observe data returned. |
| R-PRF-4 | The user shall, when signed in, be able to sync user data across multiple devices. | Modify a favourite on device A; observe it on device B. |

### 9.8 OSM contribution

| ID | Requirement | Acceptance |
|---|---|---|
| R-OSM-1 | The user shall be able to report a map error as an OSM note from within the app. | Submit a note; verify it appears on osm.org. |
| R-OSM-2 | The user shall be able to add or edit a POI and submit the edit to OSM, optionally deferred for offline submission later. | Add a POI offline; reconnect; observe upload; verify on osm.org. |

### 9.9 Third-party integration and plugins

| ID | Requirement | Acceptance |
|---|---|---|
| R-PLG-1 | The user shall be able to enable or disable any optional feature at runtime without reinstalling the app. | Toggle a plugin; observe its UI elements appear or disappear without restart. |
| R-PLG-2 | A third-party app on the same device shall be able to start navigation, add markers, query location, and receive turn updates from OsmAnd, subject to user permission. | Bind a sample external app; trigger each operation; observe the corresponding effect in OsmAnd. |

### 9.10 Accounts and authentication

Accounts are optional. Core features — map viewing, routing, search, recording — work with no sign-in.

| ID | Requirement | Acceptance |
|---|---|---|
| R-ACC-1 | The core feature set shall be usable with no signed-in account. | Fresh install with no sign-in; exercise each core feature; observe no auth wall. |
| R-ACC-2 | The user shall be able to sign in to an OsmAnd Cloud account from inside the app. | Complete the sign-in flow; observe signed-in state. |
| R-ACC-3 | The signed-in user shall be able to sign out, ending the session and removing stored tokens. | Tap Sign out; observe session cleared. |
| R-ACC-4 | The user shall be able to link an OSM account to enable OSM contributions. | Complete OAuth in browser; return to app; observe a valid link. |
| R-ACC-5 | Signed-in state shall survive app restart and device reboot while stored tokens remain valid. | Sign in; reboot; relaunch; observe still signed in. |
| R-ACC-6 | While a refresh token is valid, the user shall not be asked to re-authenticate for routine operations. | Wait beyond access-token lifetime; trigger a sync; observe no prompt. |

### 9.11 Observability

| ID | Requirement | Acceptance |
|---|---|---|
| R-OBS-1 | The user shall be able to retrieve a local log file recording significant events of recent sessions. | Trigger a known event; open the log; observe a timestamped entry. |
| R-OBS-2 | Any transmission of diagnostic or crash data shall be opt-in and disabled by default. | Fresh install; verify no telemetry sent; opt in; verify reports begin. |
| R-OBS-3 | Logs and crash reports shall not include personally identifying information unless the user explicitly opts in. | Inspect a default log; observe absence of GPS coordinates and credentials. |

### 9.12 Map interaction

| ID | Requirement | Acceptance |
|---|---|---|
| R-INT-1 | The user shall be able to pan, pinch-zoom, rotate, double-tap zoom, and tilt the map. | Perform each gesture; observe the expected viewport change. |
| R-INT-2 | A long-press on the map shall offer at minimum: navigate to here, save as favourite, share location. | Long-press anywhere; observe the menu with the three options. |

### 9.13 Platform requirements

| ID | Requirement |
|---|---|
| R-PLT-1 | Active navigation and recording sessions shall show a persistent foreground notification with live status and controls. |
| R-PLT-2 | Phone calls shall preempt voice guidance entirely; guidance shall resume after the call ends. |
| R-PLT-3 | Denying a permission shall disable only the affected feature cleanly, not crash the app. |
| R-PLT-4 | The user shall be able to restrict OsmAnd Live automatic updates to Wi-Fi only, per region. |
| R-PLT-5 | An app upgrade shall preserve all user data — favourites, tracks, settings, downloaded maps, profiles, and account state. |
| R-PLT-6 | When connected to an Android Auto head unit, the user shall be able to navigate and search from the head-unit display. |

---

## 10. Quality requirements

| ID | Requirement | Acceptance |
|---|---|---|
| Q-PERF-1 | Map panning and zooming shall feel smooth on a mid-range Android device. | Median frame-time ≤ 33 ms; measured per §12.13. |
| Q-PERF-2 | Route calculation shall complete within an interactive timeframe. | ≤ 10 s for routes up to 200 km; ≤ 5 s for routes ≤ 50 km; measured per §12.13. |
| Q-PERF-3 | Re-routing after a deviation shall complete within 5 s. | Measured per §12.13. |
| Q-PERF-4 | Cold start to interactive map shall complete within 5 s on a mid-range device. | Measured per §12.13. |
| Q-PERF-5 | Navigation with the screen off shall consume ≤ 40 % of the power of screen-on navigation. | Measured per §12.13. |
| Q-PERF-6 | Memory use shall remain bounded regardless of the size of installed map data. | Total resident memory ≤ 512 MB with any regional map loaded; measured per §12.13. |
| Q-REL-1 | The app shall remain operational when GPS is lost or intermittent during navigation. | Cut GPS for 5 min and restore; observe no crash; guidance resumes. |
| Q-REL-2 | Field crash rate shall not exceed 0.1 % of sessions for the headline build flavour. | Measured per §12.13. |
| Q-USA-1 | The UI shall be available in ≥ 70 natural-language locales. | Measured per §12.13. |
| Q-SEC-1 | Data exchanged with OsmAnd-controlled endpoints shall be protected from interception or tampering in transit. | Verified per §12.12. |
| Q-SEC-2 | Stored account credentials shall be protected from exposure if on-device storage is accessed directly. | Measured per §12.13. |
| Q-PRIV-1 | While offline, the app shall make no outbound network call containing user location. | Capture network while offline-routing; observe zero location-bearing packets. |

---

## 11. Constraints

| ID | Constraint |
|---|---|
| C-ORG-1 | The application source code shall be licensed under GPLv3. |
| C-ORG-2 | Map data shall be derived exclusively from OpenStreetMap; no proprietary map content shall be bundled. |
| C-LEG-1 | While offline, no user location data shall be transmitted to any OsmAnd-controlled server. |
| C-LEG-2 | Published builds shall comply with Google Play and Apple App Store distribution policies. |
| C-LEG-3 | All uploads to the OSM API shall comply with the OpenStreetMap Contributor Terms. |
| C-PLT-1 | The Android edition shall support Android 7.0 and above. |
| C-PLT-2 | The iOS edition shall support iOS 12.0 and above. |
| C-PLT-3 | The free Google Play flavour shall limit users to 7 map-file downloads; the paid flavour shall have no quota. |

---

## 12. Specifications

### 12.1 Routing *(→ R-NAV-1..6)*

| Attribute | Value |
|---|---|
| Primary algorithm | Bidirectional A\* over a directed weighted graph from the `.obf` road-network index |
| Long-distance algorithm | Hierarchical Highway (HH) contraction-hierarchy for routes above a configurable distance threshold |
| Offline-only | Route calculation makes no network call; reads only on-device `.obf` files |
| Profile constants | Defined in external `routing.xml`; none hard-coded |
| Deviation rule | Off-route condition raised after ≥ 3 consecutive off-route fixes |
| Reroute time | New route produced within 5 s on a mid-range device |
| Max intermediate waypoints | 32 |

### 12.2 Map rendering *(→ R-MAP-1..7)*

| Attribute | Value |
|---|---|
| Primary renderer | Hardware-accelerated OpenGL ES 2.0 via native C++ library over JNI |
| Fallback renderer | Java 2D Canvas for devices without OpenGL ES 2.0 |
| Rendering rules | XML style files; style change takes effect on next frame without restart |
| Day/night switching | Civil twilight time computed from current location and date |
| Position marker redraw | Within 1 s of each GNSS fix; bearing smoothed with a low-pass filter |

### 12.3 Map data — format and storage *(→ R-DAT-1..4, R-MAP-1)*

| Attribute | Value |
|---|---|
| Map format | OsmAnd Binary Format (OBF) — compact spatially-indexed binary |
| OBF sections | Road-network index, map-object index, address index, POI index, public-transport index |
| Live updates | Binary diffs applied to base OBF; available at minimum hourly cadence for subscribers |
| File reads | Memory-mapped I/O so resident RAM scales with the active viewport |
| Download granularity | Per-country and per-region; road-network-only available as smaller alternative |
| Free download cap | 7 files (free flavour); no cap (paid flavour) |
| Download retry | 15 attempts; HTTP Range requests resume from partial-file offset |

### 12.4 Search *(→ R-SRH-1..4)*

| Attribute | Value |
|---|---|
| Offline-only | All address, POI, coordinate, and along-route queries resolved against on-device indexes |
| Address hierarchy | Country → region → city → street → house number; prefix and fuzzy matching |
| Along-route search | Results restricted to amenities within a configurable corridor of the active route |

### 12.5 Location / GPS *(→ R-MAP-2, R-NAV-4, R-TRK-1..3)*

| Attribute | Value |
|---|---|
| Location subscription | OS Location API at ≥ 1 Hz and ≥ 1 m displacement while navigating or recording |
| Background operation | Android Foreground Service with persistent notification and wake-locks |
| GPS loss tolerance | Loss for ≤ 5 min does not terminate navigation or recording; guidance resumes on next fix |
| Speed-limit warning | Fires when current speed exceeds `maxspeed` for the current road by a configurable tolerance |

### 12.6 Voice / TTS *(→ R-VOI-1..6, R-PLT-2)*

| Attribute | Value |
|---|---|
| Delivery | OS Text-to-Speech service or pre-recorded audio packs, as the user selects |
| Announcement distances | Configurable per profile; defaults: 2 000 m, 500 m, 50 m before each manoeuvre |
| Voice grammars | Per-language JavaScript files executed by an embedded scripting engine |
| Audio focus | Transient audio focus requested before each prompt; yielded on incoming call |

### 12.7 GPX *(→ R-TRK-1..5)*

| Attribute | Value |
|---|---|
| Schema | GPX 1.1; track points carry lat, lon, ele, time; OsmAnd extensions under `osmand:` namespace |
| Statistics | Haversine formula for distance; forward-difference derivatives for speed |
| Persistence | Each fix persisted to the in-progress file before the next fix arrives |

### 12.8 Settings and profiles *(→ R-PRF-1..4)*

| Attribute | Value |
|---|---|
| Profile model | Each profile is a named configuration instance; settings namespaced by mode key |
| Backup format | `.osf` archive containing settings JSON, GPX tracks, favourites XML, style files; restore is atomic |
| Cloud sync | Settings, favourites, and GPX tracks synced with OsmAnd Cloud over TLS; conflict resolution by timestamp |

### 12.9 Third-party API and plugins *(→ R-PLG-1..2)*

| Attribute | Value |
|---|---|
| AIDL surface | Method groups for navigation, route, location, search, favourites, markers, GPX, layers, widgets, settings |
| Permission gating | All state-changing AIDL methods gated behind a user-grantable permission at first bind |
| Plugin model | Every plugin extends a common abstract base class; host does not reference plugins outside defined hooks |
| Runtime enable/disable | Enabling or disabling a plugin requires no process restart |

### 12.10 Identity and credentials *(→ R-ACC-1..6)*

| Attribute | Value |
|---|---|
| OsmAnd Cloud sign-in | Email + verification-code flow over TLS; account created automatically on first successful verification |
| Verification code | 6-digit numeric code, single use, valid for 10 minutes |
| Code resend | 60 s cooldown between sends; max 5 codes issued per email per hour |
| Failed attempts | Sign-in blocked for 15 minutes after 5 consecutive incorrect codes for the same email |
| Access token lifetime | 1 hour |
| Refresh token lifetime | 30 days, sliding, revoked immediately on sign-out |
| OSM account linking | OAuth 2.0 with PKCE; scopes limited to notes, POI edits, and GPX uploads; state parameter validated |
| Token storage | Access and refresh tokens in the platform's encrypted credential store; not written to logs or exports |
| Silent refresh | A 401 or expired-token response triggers silent refresh; user prompted only when refresh itself fails |
| Sign-out | Deletes access token, refresh token, and cached account profile; non-account-bound data untouched |

### 12.11 Logging and observability *(→ R-OBS-1..3)*

| Attribute | Value |
|---|---|
| Log routing | Android Log and rolling on-device log file; levels: error, warn, info, debug |
| Retention | At least 7 days or 5 MB, whichever is larger |
| PII policy | Production logger calls do not emit GPS coordinates, tokens, or credentials |
| Crash reports | Written locally; uploaded to remote channel only on user opt-in |

### 12.12 Network and TLS *(→ Q-SEC-1, Q-PRIV-1, R-PLT-4)*

| Attribute | Value |
|---|---|
| TLS minimum | TLS 1.2 on all traffic to OsmAnd-controlled endpoints |
| OSM endpoint | `https://api.openstreetmap.org/api/0.6/…` |
| Offline egress | No outbound socket opened by routing, rendering, voice, or location when offline |
| Live update Wi-Fi gate | Per-map Wi-Fi-only flag honoured by both the scheduler and executor |

### 12.13 Performance — measurement methods

| Quality requirement | Measurement |
|---|---|
| Q-PERF-1 (frame-time) | Frame callbacks during a scripted 60 s pan/zoom over a downloaded region on a reference device |
| Q-PERF-2 (route time) | From "Navigate" tap to route displayed, over 50 routes per profile per region |
| Q-PERF-3 (reroute time) | From deviation trigger to new route displayed |
| Q-PERF-4 (startup) | From process launch to first interactive frame, after force-stop and cache clear |
| Q-PERF-5 (battery) | Battery statistics over a 1 h scripted drive, screen on vs screen off, on a reference device |
| Q-PERF-6 (RAM) | Resident memory via OS memory-info API after 5 min of pan/zoom over the largest enabled region |
| Q-REL-2 (crash rate) | `1 − crashes_with_consent / sessions_with_consent`, weekly per build flavour, from opt-in telemetry |
| Q-USA-1 (locales) | Count of locale string bundles with ≥ 90 % key coverage of the default bundle |
| Q-SEC-2 (credentials) | Credentials stored in Android EncryptedSharedPreferences |

---

## 13. D, S ⊢ R — justification

For each headline requirement, the domain assumptions and specifications whose conjunction entails it.

| Requirement | Domain assumptions | Specifications | Argument |
|---|---|---|---|
| R-NAV-1 (offline route) | W-4, W-15 | §12.1 (offline-only, A\*), §12.3 (OBF format and sections) | OSM road graph is encoded in the OBF road-network index; the file persists on disk; A\* runs over it without network. |
| R-NAV-4 (recompute on deviation) | W-1 | §12.5 (location at ≥ 1 Hz), §12.1 (deviation rule, reroute ≤ 5 s) | Fixes arrive at ≥ 1 Hz; deviation rule fires after 3 consecutive off-route fixes; a new route is produced within 5 s. |
| R-VOI-1 (voice announcement) | W-7, W-10 | §12.6 (TTS delivery, advance distances, audio focus) | TTS is available; the Machine emits utterances in advance and claims audio focus so they are audible above road noise. |
| R-VOI-2 (announce in advance) | W-11 | §12.6 (announcement distances: 2 000 m, 500 m, 50 m) | The default distances provide head-room for any profile's typical speed. |
| R-MAP-1 (offline render) | W-15 | §12.2 (renderer), §12.3 (OBF format and sections, memory-mapped I/O) | All data and rendering rules needed to paint a tile are local; the renderer runs without network. |
| R-SRH-1 (address search) | W-4 | §12.4 (offline-only, address hierarchy) | The address index is on disk; the search engine queries it without network. |
| R-TRK-3 (survive kill) | W-15, W-16 | §12.7 (persist each fix), §12.5 (foreground service) | Each fix is persisted before the next; the foreground service reduces kill probability; the on-disk file is recoverable. |
| Q-PRIV-1 (no offline egress) | W-14 | §12.12 (no offline egress), §12.1 (offline-only routing), §12.4 (offline-only search) | No offline routing, rendering, or search code path opens a socket. |

---

## 14. Glossary

| Term | Definition |
|---|---|
| World | Everything outside the software that OsmAnd cares about — users, environment, external systems. |
| Machine | The OsmAnd Android application: its external run-time shape (§6.1) and internal module structure (§6.2). |
| Shared phenomenon | An event or state both the World and the Machine can observe or cause; the only vocabulary specifications may use. |
| D, S ⊢ R | Domain assumptions plus specifications entail the requirement. |
| OBF | OsmAnd Binary Format — compact indexed binary map data compiled from OSM. |
| OsmAnd Live | Subscription service providing hourly incremental map updates. |
| GPX | GPS Exchange Format — XML standard for tracks and waypoints. |
| OSF | OsmAnd Settings File — archive of settings, tracks, favourites, and style overrides. |
| POI | Point of Interest — named geographic feature. |
| A\* | A-star bidirectional pathfinding algorithm used for short and medium routes. |
| HH routing | Hierarchical Highway routing — contraction-hierarchy algorithm for long-distance routes. |
| TTS | Text-to-Speech — OS service synthesising spoken instructions. |
| AIDL | Android Interface Definition Language — OsmAnd's third-party IPC interface. |
| Profile | A named configuration set for a travel mode. |
| Plugin | An optional feature module extending the Machine through defined hooks. |
| GNSS | Global Navigation Satellite System — covers GPS, GLONASS, Galileo, BeiDou. |
| OAuth 2.0 | Open standard for delegated authorisation; used with PKCE for OSM account linking. |
| Foreground service | An Android service with a persistent notification, exempt from aggressive background-process killing. |
| Metered network | A network the OS reports as charging per byte — typically cellular. |

---

## 15. References

1. **Michael Jackson**, *Software Requirements & Specifications*, ACM Press / Addison-Wesley, 1995.
2. **Michael Jackson**, *Problem Frames*, Addison-Wesley, 2001.
3. **Pamela Zave and Michael Jackson**, "Four Dark Corners of Requirements Engineering," *ACM TOSEM*, 6:1–30, 1997.
4. **Ian Sommerville**, *Software Engineering*, 10th ed., Pearson, 2015.
5. **OpenStreetMap Wiki**, "Map Features" and "API v0.6" — https://wiki.openstreetmap.org/
6. **GPX 1.1 schema**, Topografix — https://www.topografix.com/GPX/1/1/
7. **OsmAnd repository** — https://github.com/osmandapp/OsmAnd
