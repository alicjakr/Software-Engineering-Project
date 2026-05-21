# Requirements Specification - OsmAnd
## World / Machine Model

**Project:** OsmAnd
**Repository:** https://github.com/osmandapp/OsmAnd
**Version:** 2.0
**Date:** May 2026

This document is structured around Michael Jackson's World/Machine framing. Where statement *modes* matter the section header marks them. The document layers as follows:

- §1 Scope. §2 Method.
- §3–§5 describe **the World** - actors, external domains, environment.
- §6 describes **the Machine** as we are building it: its external run-time shape and its internal module structure.
- §7 lists **shared phenomena** - the only vocabulary §10 specifications may use.
- §8 lists **domain assumptions**.
- §9 lists **requirements** on the user's situation.
- §10 lists **quality requirements**.
- §11 lists **organisational and external constraints**.
- §12 lists **specifications** - what the Machine must do at the boundary, with concrete versions, measures, and algorithms.
- §13 sketches the **`D, S ⊢ R` justification** for headline requirements.
- §14 traceability matrix.
- §15 glossary, §16 references.

---

## 1. Scope

OsmAnd is an open-source map-viewing and turn-by-turn navigation application for Android and iOS, built on OpenStreetMap data. Its defining property is **full offline operation**: all core features - routing, vector map rendering, address and POI search, GPX recording, voice guidance - must work without any network connection, using pre-downloaded binary map files.

The **system-to-be** described in this document is the Android edition of OsmAnd, treated at two levels of detail: its external run-time shape - what the rest of the World observes of it - and its internal module structure. The iOS edition shares the cross-platform modules and inherits the same requirements; platform-specific specifications below identify when a clause is Android-only.

Note on Jackson's framing: in a strict reading of the World/Machine Model, the Machine is treated only by its external behaviour, and its internal decomposition belongs to design rather than requirements. This document keeps §6.1 (external shape) inside the WMM proper, and treats §6.2-§6.4 (modules, classes, plugin hooks) and §14.2 (specification-to-code map) as *informative* sections that ground the class and dynamics diagrams listed in the project README, without being load-bearing for the `D, S ⊢ R` argument in §13.

Out of scope: the OsmAnd MapCreator tool; the OsmAnd Cloud server-side implementation; web map services.

---

## 2. Method: World/Machine Model discipline

### 2.1 Layering

- **R - Requirement.** A statement about the World that we want to be made true once the Machine is in place. Optative. Phrased in the user's vocabulary.
- **S - Specification.** A statement about the **shared phenomena** that the Machine must satisfy. Optative. Phrased in the boundary's vocabulary.
- **D - Domain knowledge.** A statement about how the World already behaves, independent of the Machine. Indicative.
- **P - Program.** Source code. Lives in the repo, not this document.
- **M - Machine.** The Program running on its target platform.

The central entailment that this document is structured to support is

> **D, S ⊢ R**

- the domain assumptions together with the specifications must logically entail the requirements. §13 carries out this argument for the headline requirements; a requirement that cannot be entailed by any combination of `DOM.*` + `SPEC.*` is either incomplete or wishful.

### 2.2 Indicative vs optative mood

Every sentence in §§3–5 and §8 is **indicative** - it describes what is, has been, or will be true of the World regardless of OsmAnd. Every sentence in §§9–12 is **optative** - it states what we want to be true once OsmAnd is in place. Each table caption restates the mood.

### 2.3 ID conventions

Requirements and friends are identified by **semantic slugs** rather than opaque numeric codes. This document adopts:

| Prefix | Layer | Example |
|---|---|---|
| `REQ.<domain>.<slug>` | functional requirement | `REQ.routing.offline-route` |
| `QUAL.<attribute>.<slug>` | quality requirement | `QUAL.perf.route-under-5s` |
| `CON.<scope>.<slug>` | constraint | `CON.legal.gdpr-no-location-upload` |
| `DOM.<topic>.<slug>` | domain assumption | `DOM.gps.fix-rate-outdoors` |
| `SPEC.<area>.<slug>` | specification | `SPEC.routing.bidirectional-astar` |
| `PHEN.<slug>` | shared phenomenon | `PHEN.gps-fix` |

Rationale: slugs survive reordering, are greppable, and tell the reader what the line is about. Stability of IDs is preserved by *not renaming the slug after publication*; if the meaning changes, deprecate and add a new ID.

---

## 3. Actors

An actor is a World entity that *acts* - it chooses to cause phenomena. Passive external domains are listed in §4.

| ID | Actor | Role |
|---|---|---|
| ACT.user.navigator | **End user** | Operates the device while travelling on foot, by bicycle, by car, by boat, or using public transport. Issues commands; consumes map, route, and voice output. |
| ACT.user.recorder | **End user** | Same person, different task: records a GPX track for later use, sharing, or upload. |
| ACT.user.osm-contributor | **OSM contributor** | A subset of users who edit OSM through OsmAnd - adding POIs, reporting notes, uploading GPX tracks. |
| ACT.dev.third-party-app | **Third-party Android application developer** | An external developer whose app binds to OsmAnd's public AIDL service to start navigation, add markers, or query location. |
| ACT.opr.osm-community | **OSM community** | Volunteer mappers who maintain the underlying OSM database. Not direct users of OsmAnd, but the source of map content. |
| ACT.opr.osmand-team | **OsmAnd project organisation** | The OsmAnd development team and maintainers. Operates the build, the servers, and accepts contributions. |

---

## 4. External domains

External domains are passive or service-providing entities the Machine must interact with. Classified per Jackson: **causal** domains have predictable physical behaviour; **lexical** domains are pure data stores; **biddable** domains accept commands but cannot be assumed to respond.

| ID | Domain | Class | Role |
|---|---|---|---|
| EXT.gnss | **Global Navigation Satellite System** | causal | Delivers position fixes via the device's GNSS chip; covers GPS, GLONASS, Galileo, BeiDou, depending on hardware. Outside OsmAnd's control. |
| EXT.osm-db | **OpenStreetMap database** | lexical | Crowd-sourced global geographic dataset. Source of all map content; ingested into `.obf` by external tooling. |
| EXT.osm-api | **OSM API** | biddable | HTTP API accepting POI edits, OSM notes, and GPX uploads from registered contributors. |
| EXT.osmand-servers | **OsmAnd map servers** | biddable | Hosts pre-compiled `.obf` regional files and OsmAnd Live diff feeds. |
| EXT.osmand-cloud | **OsmAnd Cloud** | biddable | Cross-device sync service for user data. |
| EXT.tile-servers | **Online raster tile providers** | biddable | Bing satellite, OpenTopoMap, Thunderforest, etc. Optional overlay sources. |
| EXT.telegram | **Telegram messaging platform** | biddable | External chat platform; OsmAnd Tracker plugin posts location updates via the Telegram Bot API. |
| EXT.android-os | **Android operating system** | causal | Provides Location API, TextToSpeech, foreground services, filesystem, sensors, audio, notifications. |
| EXT.filesystem | **Device filesystem** | lexical | Persistent storage for `.obf`, `.gpx`, `.osf`, voice packs, logs, and settings. |
| EXT.tts-engine | **OS Text-to-Speech engine** | causal | Synthesises spoken instructions from text in a chosen language. |
| EXT.audio-out | **Device audio output** | causal | Renders spoken instructions and tones. |
| EXT.network | **Internet** | causal | Used for downloads, sync, online tiles, and OSM contribution upload. Not required for offline operation. |
| EXT.battery | **Device battery and power management** | causal | Finite energy budget; aggressive doze and background limits on Android. |

---

## 5. Physical environment and user context 

The World OsmAnd operates in has properties that shape what the Machine must do. Each subsection below is purely descriptive; the requirements it *motivates* are cited with their `REQ.*` IDs so the reader can trace the chain.

### 5.1 User context

Users use OsmAnd while moving - on foot, by bicycle, in a vehicle, or on water. The user's *attention budget* is small and the user's *hands are busy*: a driver cannot read the screen; a cyclist cannot tap precisely; a hiker may be wearing gloves; an emergency-services worker may be operating one-handed. Users routinely operate the app in places where mobile internet is unavailable, unreliable, or expensive. Users speak many languages and read many scripts; a non-trivial subset cannot read Latin script at all.

*Drives:* `REQ.guidance.voice-announcement`, `REQ.guidance.glanceable-screen`, `REQ.routing.offline-route`, `REQ.map.local-script-labels`, `QUAL.usability.locales`, `QUAL.access.large-touch-targets`.

### 5.2 The physical road network and geographic World

OsmAnd operates over a representation of the real physical world: roads, paths, buildings, waterways, elevation, points of interest. This World is inherently imperfect - roads change, businesses open and close, construction alters routes. The map data the app uses is a snapshot, always partially out of date.

*Drives:* `REQ.routing.recompute-on-deviation`, `REQ.osm.report-error`, `REQ.osm.contribute-edit`.

### 5.3 GNSS behaviour

The device's location is determined by the **Global Navigation Satellite System**. Fix accuracy typically ranges from 3–15 m in open sky, but degrades in urban canyons, indoors, in tunnels, and under dense tree cover. Fixes may be intermittent or briefly lost.

*Drives:* `REQ.navigation.tolerate-gps-loss`, `QUAL.reliability.no-crash-on-gps-loss`, `SPEC.location.fusion-low-pass`.

### 5.4 OpenStreetMap ecosystem

OsmAnd's map content is derived entirely from OpenStreetMap. Coverage and quality vary by region - densely populated Europe and North America have near-complete coverage; parts of Central Asia and Central Africa are sparser. OsmAnd has no control over data quality; it can only render and route over what OSM provides.

*Drives:* `CON.org.osm-only-data`, `REQ.osm.contribute-edit`.

### 5.5 OsmAnd server and cloud infrastructure

The OsmAnd servers host pre-compiled `.obf` regional files; the OsmAnd Live service provides hourly incremental diffs for subscribers; OsmAnd Cloud syncs user data. These services are external to the Machine; the app must function fully without them when offline.

*Drives:* `REQ.data.download-region`, `REQ.data.fresher-updates`, `REQ.sync.cross-device`.

### 5.6 Device hardware and OS - scope statement

OsmAnd's target devices and OSes are part of the Machine's scope, not part of the indicative World description. They are recorded here as scope and recur in `CON.platform.*` and `SPEC.platform.*`. The relevant device/OS surface includes the GPS chip, filesystem, the compass sensor, the display, the speaker / headphone / Bluetooth audio outputs, the OS Location API, the OS TextToSpeech service, and the OS foreground-service / background-execution policies. Battery is a finite resource; navigation sessions can run for hours, and the Machine must minimise power draw, particularly when the screen is off.

*Scope:* `CON.platform.android-min-sdk`, `CON.platform.ios-min-version`, `QUAL.perf.battery-screen-off`, `SPEC.location.foreground-service`, `SPEC.storage.memory-mapped-obf`.

### 5.7 Voice and audio environment - user requirements

Turn-by-turn instructions are delivered to the user through the device's audio output. The audio must be comprehensible while the user is driving, cycling, or walking - meaning announcement *timing*, *clarity*, and *language* are real user-side requirements, not just environmental observations.

*Drives:* `REQ.guidance.voice-announcement`, `REQ.guidance.announce-in-advance`, `REQ.guidance.user-language`, `QUAL.usability.locales`.

### 5.8 Optional online overlays

For users who optionally choose online map overlays, Bing Maps and various community tile servers serve raster tiles over the internet. These are entirely external and optional; OsmAnd must not depend on them for core functionality.

### 5.9 Telegram

The OsmAnd Tracker plugin integrates with Telegram to broadcast the user's real-time location to selected contacts via Telegram's messaging network. Optional safety/social feature dependent on Telegram's external infrastructure.

---

## 6. The Machine - internal structure

### 6.1 External run-time shape

This is the view the rest of the World has of the Machine at run time. The Machine is the **OsmAnd Android application** installed and running on a single user's device. It runs as an Android Activity-based app with a single visible `MapActivity`, a `NavigationService` foreground service for background sessions, and a persistent settings store. It has no central server-side counterpart in the runtime: each user's Machine is the locally installed app plus its on-device data. Everything else the Machine does at the boundary is captured by the shared phenomena in §7; everything else it is, internally, lives in §6.2-§6.4 and is informative for the WMM.

### 6.2 Internal modules

These are the real top-level modules in the OsmAnd repository - referenced here so that requirements, specifications, and the class diagrams in `diagrams/` can talk about them by name rather than by guess.

| Module | Languages | Responsibility |
|---|---|---|
| `OsmAnd` | Java + Kotlin | Android UI, activity lifecycle, foreground service, plugin host, AIDL service host. |
| `OsmAnd-java` | Java | Platform-neutral core logic: `.obf` reading, A\* routing, search, rendering rules. No Android dependencies. Shared with iOS / JVM. |
| `OsmAnd-shared` | Kotlin Multiplatform | Cross-platform domain models, GPX, settings, coroutines, HTTP client, SQL access. |
| `OsmAnd-api` | AIDL + Java | Public IPC surface for third-party Android apps: `IOsmAndAidlInterface`, `IOsmAndAidlCallback`, parcelable data classes. |
| `OsmAnd-telegram` | Kotlin + Java | Standalone companion app implementing the Telegram-based live location sharing. |
| `plugins/*` | Java + Kotlin | Built-in optional plugins - each a subclass of `OsmandPlugin`. |

### 6.3 Load-bearing classes

The conceptual subsystems in this document map to specific code surfaces in the modules above. When a `SPEC.*` cites a class, it cites one of these.

| Conceptual subsystem | Headline classes | Module |
|---|---|---|
| App lifecycle | `OsmandApplication`, `AppInitializer`, `MapActivity` | OsmAnd |
| Routing | `RoutingHelper`, `RoutingContext`, `RoutingEnvironment`, `VoiceRouter`, `GpxRouteHelper`, `OnlineRoutingHelper` | OsmAnd + OsmAnd-java |
| Map data | `BinaryMapIndexReader`, `IndexConstants`, `OsmandRegions` | OsmAnd-java |
| Map rendering | `OsmandRenderer`, `MapRenderRepositories`, `RendererRegistry`, `OsmAndMapLayersView`, `OsmAndMapSurfaceView`; `RenderingRulesStorage`, `RenderingRule` | OsmAnd + OsmAnd-java |
| Search | `SearchUICore`, `SearchPhrase`, `SearchResultMatcher` | OsmAnd-java |
| Location | `OsmAndLocationProvider`, `NavigationService`, `GpsStatusListener` | OsmAnd |
| Voice / TTS | `VoiceRouter`, `CommandPlayer`, `JsCommandBuilder`, `JsTtsCommandPlayer`, `JsMediaCommandPlayer` | OsmAnd |
| GPX | `GpxFile`, `Track`, `WptPt`; `SaveGpxHelper`, `TrackDisplayHelper`, `GpxSelectionHelper` | OsmAnd-shared + OsmAnd |
| Settings / profiles | `OsmandSettings`, `ApplicationMode`, `OsmandPreference<T>` | OsmAnd |
| Plugin host | `OsmandPlugin` base + per-plugin subclasses | OsmAnd + plugins |
| Cloud sync / backup | `BackupHelper`, `NetworkSettingsHelper` | OsmAnd |
| Third-party IPC | `OsmandAidlService`; AIDL interfaces in `OsmAnd-api` | OsmAnd + OsmAnd-api |
| Downloads | `DownloadIndexesThread`, `DownloadService` | OsmAnd |
| Logging | `PlatformUtil` logger factory routing to Android Log + on-device file | OsmAnd-java + OsmAnd |

A logical view of these modules is in `diagrams/logical_components.png`; physical deployment in `diagrams/physical_architecture.png`; the system context view in `diagrams/logical_context.png`.

### 6.4 Plugin extension points

`OsmandPlugin` is the abstract base for all plugins. Subclasses extend the Machine by hooking the following extension points:

- `registerLayers` - add custom `OsmandMapLayer` instances drawn over the base map.
- `createWidgets` - add map-corner widgets.
- `registerMapContextMenuActions` - add long-press menu items.
- `getPreferences` - add a settings fragment.
- `prepareLocationPoint` / `prepareExternalPoiCategory` - extend GPX point processing and POI taxonomy.
- A plugin may also bundle its own `routing.xml` profile fragment and its own rendering-rules XML.

---

## 7. Shared phenomena

A shared phenomenon is something both the World and the Machine can observe or cause. Each row records its **type**, its **controller**, and which side **observes** it. Only phenomena listed here may appear in `SPEC.*` clauses; anything else is internal to one side.

| ID | Phenomenon | Type | Controlled by | Observed by |
|---|---|---|---|---|
| PHEN.navigate-tap | User taps "Navigate" with destination | event | User | Machine |
| PHEN.search-query | User submits a search query | event | User | Machine |
| PHEN.poi-tap | User taps a POI / map object | event | User | Machine |
| PHEN.profile-select | User selects a profile | event | User | Machine |
| PHEN.gps-fix | GNSS reports a position fix | event | EXT.gnss | Machine |
| PHEN.gps-loss | Location updates cease for ≥ N seconds | state | EXT.gnss | Machine |
| PHEN.network-available | Internet reachability of a given endpoint | state | EXT.network | Machine |
| PHEN.obf-present | `.obf` file for a region exists on the filesystem | state | EXT.filesystem + Machine on download | both |
| PHEN.route-displayed | A route polyline + instructions appear on the screen | state | Machine | User |
| PHEN.tts-utter | Spoken instruction emitted from speaker | event | Machine | User |
| PHEN.alert-tone | Warning tone emitted | event | Machine | User |
| PHEN.map-frame | A rendered map frame is displayed | event | Machine | User |
| PHEN.gpx-write | Bytes written to a `.gpx` file on the filesystem | event | Machine | EXT.filesystem |
| PHEN.obf-download | HTTP request/response cycle for a `.obf` region file | event | Machine | EXT.osmand-servers |
| PHEN.osm-upload | HTTP request to OSM API with a contribution payload | event | Machine | EXT.osm-api |
| PHEN.aidl-call | An IPC method call into `OsmandAidlService` | event | EXT.third-party-app | Machine |
| PHEN.aidl-callback | A callback delivered to a bound third-party app | event | Machine | EXT.third-party-app |
| PHEN.log-line | A log line written to `osmand_log.txt` and / or Android Log | event | Machine | EXT.filesystem + developer |
| PHEN.crash-report | A crash diagnostic emitted to the configured channel | event | Machine | EXT.filesystem and EXT.network |
| PHEN.auth-signin-tap | User taps Sign in, Register, or Link OSM account | event | User | Machine |
| PHEN.auth-credentials-entered | User submits credentials in an in-app form or in the in-browser OAuth page | event | User | Machine |
| PHEN.auth-token-issued | An identity provider issues a session or refresh token to the Machine | event | EXT.osmand-cloud or EXT.osm-api | Machine |
| PHEN.auth-token-revoked | A session or refresh token is invalidated by the user, the provider, or expiry | event | User or provider | Machine |
| PHEN.auth-signed-in | The Machine currently holds a valid token for a given identity | state | Machine | User-facing surfaces |
| PHEN.permission-prompt | OS-rendered permission dialog initiated by the Machine | event | Machine | User |
| PHEN.permission-decision | User grants or denies an OS permission | event | User | Machine |
| PHEN.map-gesture | User performs a touch gesture on the map - pinch, pan, rotate, tilt, double-tap | event | User | Machine |
| PHEN.long-press | User long-presses a point on the map | event | User | Machine |
| PHEN.share-out | Machine offers content to other apps via an `ACTION_SEND` intent | event | Machine | EXT.android-os |
| PHEN.share-in | Another app delivers content - URL, GPX, OBF, KML - to the Machine via `ACTION_SEND` or `ACTION_VIEW` | event | EXT.android-os | Machine |
| PHEN.deep-link | An `osmand.net/go` or `osmand.net/map` URL is delivered to the Machine via the OS link handler | event | EXT.android-os | Machine |
| PHEN.notification-shown | A persistent or transient notification is posted to the system tray | state | Machine | User |
| PHEN.notification-action | User taps an action button or body of a Machine notification | event | User | Machine |
| PHEN.audio-focus-gained | The OS grants exclusive, transient, or transient-may-duck audio focus to the Machine | event | EXT.android-os | Machine |
| PHEN.audio-focus-lost | The OS revokes audio focus, transiently or permanently | event | EXT.android-os | Machine |
| PHEN.purchase-event | A user purchase or subscription state transition is reported by the billing service | event | EXT.osmand-servers | Machine |
| PHEN.app-upgrade | The app process starts on a version-code higher than the last-known one | event | EXT.android-os | Machine |
| PHEN.download-progress | A region download advances or completes | event | Machine | User |
| PHEN.car-session | An Android Auto head unit connects or disconnects | state | EXT.android-os | Machine |
| PHEN.sensor-reading | A paired BLE or ANT+ sensor delivers a reading - heart rate, cadence, power | event | EXT.android-os | Machine |
| PHEN.a11y-event | The OS accessibility service requests or signals an event - TalkBack focus, hover, action | event | EXT.android-os | Machine |

Phenomena explicitly **not** shared, and therefore not in this table: routing-queue state, render cache hit ratio, thread on which a computation runs, the user's emotional state, satellite orbital ephemerides. These are internal to one side.

---

## 8. Domain assumptions

Each `DOM.*` is a statement about the World that the requirements depend on. If the assumption fails, the requirement it underwrites may fail without that being a Machine bug.

| ID | Assumption |
|---|---|
| DOM.gps.fix-outdoors | Outdoors, with clear sky and ≥ 4 satellites in view, a consumer GNSS receiver produces position fixes at ≥ 1 Hz with horizontal accuracy ≤ 15 m typically. |
| DOM.gps.fix-degraded | Indoors, in tunnels, in dense urban canyons, or under dense canopy, fixes may be absent or accuracy may exceed 50 m. |
| DOM.gps.intermittent | A single position fix may be wholly absent for up to several minutes in tunnels or covered areas; service usually resumes when sky is again visible. |
| DOM.osm.data-density | OSM coverage is near-complete in Europe and North America and partial in some regions of Central Asia and Central Africa, as of the v1.0 baseline. |
| DOM.osm.data-staleness | OSM is updated continuously by volunteers; a `.obf` snapshot is therefore out of date with respect to the live OSM database from the moment it is compiled. |
| DOM.osm.tag-conventions | Roads in OSM are tagged with `highway=*`; speed limits in `maxspeed`; access restrictions in `access=*`; turn restrictions in relations. |
| DOM.android.location-api | The Android OS exposes location through `LocationManager` and, on Google-services devices, `FusedLocationProviderClient`. Updates are delivered as `Location` objects on a chosen Looper. |
| DOM.android.tts | Android exposes a `TextToSpeech` system service with at least one language pre-installed; additional languages may need installation by the user. |
| DOM.android.fgs | An Android foreground service with a persistent notification is not killed by the OS during normal use, even when the screen is off. |
| DOM.android.doze | When the screen is off and the app is not foreground, the OS aggressively throttles background CPU and network unless a wake-lock or foreground service is in effect. |
| DOM.audio.driving-noise | In-vehicle background noise is in the 60–75 dBA range; spoken instructions need adequate volume and intelligibility to be heard. |
| DOM.user.attention | The user, while driving or cycling, cannot reliably read fine on-screen text; turn instructions must be delivered in advance and audibly. |
| DOM.user.languages | Users speak many natural languages and read many scripts; some users cannot read Latin script at all. |
| DOM.device.storage | A consumer Android device typically has 16–256 GB of internal storage; users may attach an SD card with comparable or larger capacity. |
| DOM.device.battery | A typical phone battery delivers 8–24 h of mixed use; continuous GPS + screen-on navigation can drain it in under 6 h. |
| DOM.network.intermittent | Mobile data is intermittent or absent in many real-world locations. |
| DOM.fs.persistence | Files written to internal or SD-card storage persist across reboots and app updates unless the user uninstalls or wipes the device. |
| DOM.power.kill | The OS may kill the app process at any time when the screen is off and the app is not running a foreground service. |
| DOM.osm.contributor-account | OSM contributions require a registered OSM account; the OSM API expects OAuth-1.0a or OAuth-2.0 credentials. |
| DOM.account.osm-account-external | OSM user accounts are created and recovered on `osm.org`, not in OsmAnd. The identity provider for OSM editing is the OSM web service; OsmAnd is only a relying party. |
| DOM.account.cloud-account-via-server | An OsmAnd Cloud account is created and verified through the OsmAnd Cloud HTTP service; account creation requires the device to reach that service over the network. |
| DOM.account.tokens-expire | OAuth access tokens have finite lifetime; refresh tokens may be revoked by the provider or the user without notice to the Machine. |
| DOM.account.oauth-redirect | The OAuth flow relies on the OS to deliver a redirect URI back to OsmAnd via a registered custom scheme or HTTPS app link. |
| DOM.android.runtime-permissions | Since Android 6.0 / API 23, dangerous permissions - location, microphone, camera, storage, notifications, background-location - are granted at runtime by the user, not at install time. |
| DOM.android.intent-system | Apps inter-communicate through Intents - `ACTION_SEND` for share, `ACTION_VIEW` for open, custom schemes for deep links. The OS dispatches intents to apps with matching `<intent-filter>` declarations. |
| DOM.android.notification-channels | Since Android 8.0 / API 26 every notification belongs to a user-configurable channel; importance, sound, and DND override behaviour are per-channel. |
| DOM.android.audio-focus | The OS arbitrates audio output between apps via an audio-focus model: requests are *exclusive*, *transient*, or *transient-may-duck*; an app must lower volume or pause when focus is lost. |
| DOM.android.play-billing | On Google Play distributions, in-app purchases and subscriptions are mediated by the Google Play Billing service; OsmAnd's process never sees the user's payment instrument. |
| DOM.android.app-upgrade | The OS sets the app's installed version code; on each cold start the Machine can compare it against the last persisted code to detect an upgrade. |
| DOM.android.auto | Android Auto exposes apps to in-car head units only through a constrained template UI - `CarAppService` plus `Screen` templates - and only after the user pairs the phone with the head unit. |
| DOM.android.bluetooth | Pairing of a BLE or ANT+ sensor with the device is performed once via the OS pairing UI; after pairing, the Machine connects directly using `BluetoothAdapter` and a recorded MAC. |
| DOM.android.accessibility | Android exposes UI semantics to TalkBack and other accessibility services through `contentDescription`, `accessibilityHeading`, and live regions; correctly tagged views are read aloud automatically. |
| DOM.android.deep-link | Web URLs under `osmand.net/go` and `osmand.net/map` are registered as Android App Links and are dispatched to OsmAnd when present on the device. |
| DOM.network.metered | The Android `ConnectivityManager` exposes whether the current network is metered; the user generally regards cellular as metered and Wi-Fi as not, but exceptions exist. |

---

## 9. Requirements

Each requirement states a goal in the user's vocabulary. Algorithms, file formats, version numbers, byte sizes, and API names are deliberately absent here - they are confined to §12. Each requirement has a measurable **acceptance** criterion phrased as observable World behaviour.

### 9.1 Navigation and routing

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.routing.offline-route | A user with no internet connection and a downloaded regional map shall obtain a routable path between any two reachable points in that region. | Disable network; pick any two addressable points within a downloaded region; observe a route is produced. |
| REQ.routing.travel-modes | The user shall be able to compute routes appropriate to walking, cycling, driving, public transport, and boat travel. | For each profile, compute a route between fixed test points; observe the produced route is plausible. |
| REQ.routing.intermediate-waypoints | The user shall be able to add intermediate waypoints to a planned route. | Add 1–3 waypoints; observe the route passes through them in order. |
| REQ.routing.recompute-on-deviation | When the user's observed position falls away from the current route, the Machine shall produce an updated route to the same destination without further user input. | Drive a divergence from the calculated route; observe a new route is shown without prompting. |
| REQ.routing.avoid-classes | The user shall be able to avoid named classes of road or feature. | Toggle each avoidance; observe the produced route obeys the toggle. |
| REQ.routing.cancel | The user shall be able to cancel an active navigation session at any time and return to map browsing. | Tap stop-navigation; observe map view restored and guidance silenced. |

### 9.2 Voice guidance

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.guidance.voice-announcement | The user shall be informed of each upcoming manoeuvre by spoken announcement, without needing to look at the screen. | Drive a multi-turn route eyes-off-screen; observe every turn was announced audibly. |
| REQ.guidance.announce-in-advance | Each manoeuvre announcement shall reach the user in time to act on it safely, considering the profile's typical speed. | Time the gap between announcement and turn; gap allows safe execution at the profile's reference speed. |
| REQ.guidance.user-language | Announcements shall be delivered in the user's chosen language, when that language is available on the device. | Switch device TTS to language L; observe announcements in L. |
| REQ.guidance.glanceable-screen | While navigation is active, the user shall be able to see, at a glance, the next manoeuvre, distance to it, the road name, and the estimated time of arrival. | One-second glance test on a passenger-driven trip: all four data items legible. |
| REQ.guidance.speed-limit-warn | The user shall be warned when the observed speed exceeds the speed limit recorded for the current road. | Travel at speed > limit on a tagged road; observe visible and / or audible warning. |
| REQ.guidance.lane-hints | The user shall, where the data is available, see lane-guidance hints before complex junctions. | Approach a junction with `turn:lanes` data; observe a lane-hint overlay. |

### 9.3 Map viewing

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.map.offline-render | The user shall be able to view the map of a downloaded region with no internet connection. | Disable network; pan over a downloaded region; observe map renders without empty tiles. |
| REQ.map.position-orientation | The user's current position and direction of travel shall be shown on the map. | Move outdoors; observe position marker and bearing indicator updating. |
| REQ.map.rotate-with-direction | The user shall be able to choose whether the map rotates with compass / motion direction or stays north-up. | Toggle between modes; observe the rotation behaviour matches. |
| REQ.map.day-night | The map shall automatically darken at night and lighten during the day, by default. | Cross the device-local civil twilight; observe the style switches. |
| REQ.map.local-script-labels | The user shall be able to choose whether place names appear in the local script, in English, or in phonetic transliteration. | Switch settings; observe labels in selected form. |
| REQ.map.gpx-overlay | The user shall be able to display recorded or imported GPX tracks as overlays on the map. | Import a GPX; toggle visibility; observe overlay. |
| REQ.map.style-switch-runtime | The user shall be able to switch the rendering style at runtime without restarting the application. | Switch from `default` to `topo` while viewing a region; observe new style applied in-place. |
| REQ.map.online-raster-overlay | The user shall be able to optionally enable online raster overlays when network is available. | Enable overlay; pan over a populated region with network; observe tiles appear. |

### 9.4 Search

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.search.address | The user shall be able to find places by typing an address - country, region, city, street, house number - using offline data. | Search a known address in a downloaded region with no network; observe correct match. |
| REQ.search.poi | The user shall be able to find places by category and by name. | Search "pharmacy" while in a downloaded region; observe results. |
| REQ.search.coordinates | The user shall be able to enter geographic coordinates in common notations and have the location displayed. | Paste each format; observe pin on the map at the correct location. |
| REQ.search.along-route | The user shall be able to search for POIs along an active route. | While navigating, search "fuel along route"; observe matches restricted to the route corridor. |
| REQ.search.recent | The user shall be able to recall recent search queries and previously selected destinations. | After several searches, open history; observe ordered list. |

### 9.5 Map data management

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.data.download-region | The user shall be able to download regional map data directly from within the app. | Pick a region; tap download; observe completion and that the region is then usable offline. |
| REQ.data.external-storage | The user shall be able to store map data on the device's external SD card when available. | Move maps location to SD; observe maps still usable. |
| REQ.data.fresher-updates | The user shall be able to opt in to receive map updates more frequently than the monthly base cycle. | Subscribe to live updates; observe a fresher data signature after the next cycle. |
| REQ.data.update-monthly | The user shall be able to receive new versions of regional maps at least monthly. | Compare the publication date of two consecutive `.obf` versions for the same region; gap ≤ 1 month. |

### 9.6 Track recording

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.track.record-background | The user shall be able to record a GPX track during a journey, including while the screen is off and / or the app is in the background. | Start recording; lock screen; travel; verify recording continues. |
| REQ.track.stats | While recording, the user shall be able to see live statistics. | Open the recording panel mid-trip; observe live numbers. |
| REQ.track.survive-kill | A track in progress shall survive an unexpected termination of the app without losing recorded points up to the last persisted moment. | Force-stop mid-trip; reopen; observe track is recoverable with most points intact. |
| REQ.track.import-export | The user shall be able to import and export GPX tracks. | Import a known GPX; observe correct rendering. Export a recorded track; verify the file in an external tool. |
| REQ.track.upload-osm | The user shall be able to upload a recorded track to the OSM database for community use. | Configure OSM account; upload; verify on osm.org. |

### 9.7 Profiles, favourites, settings

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.profile.multi | The user shall be able to maintain multiple named profiles, each with independent navigation, routing, voice, and rendering settings. | Create two profiles; switch; observe distinct settings. |
| REQ.favourites.save | The user shall be able to save a location as a Favourite with a custom name, icon, and group. | Save a Favourite; reopen later; observe pin and name. |
| REQ.settings.backup | The user shall be able to back up and restore all user data. | Back up to a file; wipe; restore; observe data returned. |
| REQ.settings.cloud-sync | The user shall, when signed in, be able to sync user data across multiple devices. | Sign in on device A and device B; modify a favourite on A; observe it on B. |

### 9.8 OSM contribution

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.osm.report-error | The user shall be able to report a map error as an OSM note from within the app. | Submit a note; verify it appears on osm.org. |
| REQ.osm.contribute-edit | The user shall be able to add or edit a POI and submit the edit to OSM, optionally deferred for offline submission later. | Add a POI offline; reconnect; observe upload; verify on osm.org. |

### 9.9 Plugins and third-party integration

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.plugin.enable-disable | The user shall be able to enable or disable any optional feature at runtime, without reinstalling the app. | Toggle a plugin; observe its widgets / menus / layers appear or disappear without restart. |
| REQ.thirdparty.control | A third-party application installed on the same device shall be able to start navigation, add map markers, query the current location, and receive turn-by-turn updates from OsmAnd, subject to user permission. | Bind a sample external app; trigger each operation; observe the corresponding effect in OsmAnd. |

### 9.10 Observability and support

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.obs.local-log | The user shall be able to retrieve a local log file that records significant events of the current and recent sessions, for use in support requests. | Trigger a known event; open the log; observe a corresponding line with a timestamp. |
| REQ.obs.opt-in-telemetry | Any transmission of diagnostic, telemetry, or crash data to OsmAnd or any third party shall be opt-in and disabled by default. | Fresh install; verify no telemetry is sent; opt in; verify reports begin. |
| REQ.obs.no-pii-by-default | A log retrieved by the user, or a crash report sent under opt-in telemetry, shall not include personally identifying information unless the user has explicitly chosen to include them for support. | Inspect a default log and crash report; observe absence of PII; verify opt-in toggle adds explicit GPS coordinates. |
| REQ.obs.user-visible-errors | Errors that prevent the user from completing an action shall be communicated to the user in plain language with a recommended next step. | Trigger each error; observe message and a suggested action. |

### 9.11 Accounts and authentication

Accounts are *optional* in OsmAnd. The core navigation, map-viewing, search, and recording features work with no identity. Two distinct account surfaces exist, each with its own identity provider: **OsmAnd Cloud** for cross-device sync, owned by the OsmAnd project; and a **linked OSM account** for community contributions, owned by `osm.org`. Both sign-in flows are entered from inside the app. For the OsmAnd Cloud flow the credential entry is also in-app; for the OSM flow the credential entry happens on the OSM web page opened in a browser tab, with OsmAnd handling the OAuth dance.

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.account.optional | The core feature set - map viewing, routing, voice guidance, search, GPX recording, profile management - shall be usable with no signed-in account. | Fresh install with no sign-in; exercise each core feature; observe no auth wall. |
| REQ.account.cloud-register | The user shall be able to create an OsmAnd Cloud account from inside the app, using an email address. | Tap Register; supply email; receive verification mail; complete verification; observe signed-in state. |
| REQ.account.cloud-signin | The user shall be able to sign in to an existing OsmAnd Cloud account from inside the app. | Tap Sign in; enter email and verification code; observe signed-in state. |
| REQ.account.cloud-signout | The signed-in user shall be able to sign out from inside the app, ending the session and removing local copies of the access and refresh tokens. | Tap Sign out; observe signed-in state cleared and a subsequent sync attempt requires re-auth. |
| REQ.account.cloud-delete | The signed-in user shall be able to delete their OsmAnd Cloud account from inside the app, with explicit confirmation. | Open account settings; tap Delete account; confirm; observe local state cleared and the account no longer usable. |
| REQ.account.osm-link | The user shall be able to link an existing OSM account to OsmAnd to enable OSM contributions. | Tap Link OSM account; complete OAuth in browser; return to app; observe a valid link. |
| REQ.account.osm-unlink | The user shall be able to unlink the OSM account, revoking the locally stored token. | Tap Unlink; observe contribution features disabled until re-link. |
| REQ.account.session-survives-restart | Signed-in or linked state shall survive app restart and device reboot, without the user re-entering credentials, while the stored tokens remain valid. | Sign in; force-stop; relaunch; observe still signed in. Reboot; relaunch; observe still signed in. |
| REQ.account.silent-refresh | While a refresh token is valid, the user shall not be asked to re-authenticate for routine operations. | Wait beyond access-token lifetime while signed in; trigger a sync; observe no prompt. |
| REQ.account.clear-auth-errors | When an authentication operation fails, the user shall see a plain-language message identifying the failure mode and a recommended next step. | Submit a wrong code; observe a specific message. Use an expired link; observe a specific message. |

### 9.12 Map interaction

The map is the central surface of the app; touch interaction with it is its own family of requirements distinct from rendering or routing.

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.interact.zoom-pinch | The user shall be able to zoom in and out by pinching with two fingers. | Pinch in; observe zoom in. Pinch out; observe zoom out. |
| REQ.interact.zoom-double-tap | The user shall be able to zoom in by double-tapping a point on the map. | Double-tap a point; observe zoom centred on that point. |
| REQ.interact.pan | The user shall be able to pan the visible map by dragging with one finger. | Drag; observe map centre moves in the drag direction. |
| REQ.interact.rotate | The user shall be able to rotate the map by a two-finger twist gesture when rotation is unlocked. | Twist; observe map rotates with the gesture. |
| REQ.interact.tilt | The user shall be able to tilt the perspective by a two-finger vertical drag gesture when tilt is enabled. | Two-finger vertical drag; observe perspective change. |
| REQ.interact.long-press-pin | A long-press on the map shall present a context menu for the pointed location with options at minimum: navigate to here, save as favourite, share location. | Long-press anywhere; observe menu with the three options. |
| REQ.interact.add-marker | The user shall be able to place, name, edit, and remove a map marker from the long-press menu or the markers panel. | Place; rename; remove. Each operation persists across restart. |

### 9.13 Sharing and deep links

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.share.location-out | The user shall be able to share the currently selected location or marker with another app via the OS share sheet. | Long-press a point; tap Share; observe Android share sheet listing target apps; pick one and observe a usable representation arrives. |
| REQ.share.gpx-out | The user shall be able to share or export a GPX file to another app. | Open a recorded track; tap Share; observe the GPX file delivered. |
| REQ.share.file-in | The user shall be able to open a GPX, KML, KMZ, OBF, SQLite, or `.osf` file received from another app or a browser download in OsmAnd. | Tap a GPX file from a file manager; observe OsmAnd offered as a handler; opening it imports / displays the content. |
| REQ.share.deep-link-in | An `osmand.net/go` or `osmand.net/map` URL opened on the device shall be routed to OsmAnd, restoring the encoded map view or destination. | Open such a URL from a browser; observe OsmAnd opens at the encoded location. |

### 9.14 OS permissions

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.perm.rationale | Before requesting a dangerous OS permission for the first time, the Machine shall explain why it is needed in plain language. | Trigger first-time location permission; observe a rationale before the OS dialog. |
| REQ.perm.graceful-denial | When the user denies a permission, the Machine shall continue to function with the affected features clearly disabled, not crash and not pretend they still work. | Deny location at install; observe core map, search, route planning still usable; live navigation is offered with a clear "needs location" prompt. |
| REQ.perm.deferred-request | The Machine shall request only those permissions required for the feature the user is invoking, at the moment the user invokes it, not all permissions at first launch. | Fresh install; observe no permission prompts until the user invokes a feature that needs one. |

### 9.15 Notifications

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.notif.nav-foreground | During an active navigation session, the Machine shall display a persistent foreground notification with the next manoeuvre, ETA, and a stop-navigation action. | Start navigation; lock screen; observe notification with live content; tap stop; observe session ends. |
| REQ.notif.recording | While a GPX track is being recorded, the Machine shall display a foreground notification with elapsed time and distance, plus pause and stop actions. | Start recording; observe notification with live stats and controls. |
| REQ.notif.download-progress | Region downloads, OsmAnd Live updates, and cloud sync shall surface their progress in the notification tray, including success / failure outcomes. | Start a region download; observe progress notification; on completion, observe terminal notification. |
| REQ.notif.user-controllable | The user shall be able to mute, customise the importance of, or disable each notification category independently from the OS or in-app settings. | Mute the navigation channel from OS settings; observe no nav notifications subsequently posted while the navigation continues silently. |

### 9.16 Audio focus and call handling

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.audio.focus-request | Before each voice prompt, the Machine shall request transient audio focus from the OS so that other media ducks or pauses. | Play music; trigger a prompt; observe music ducks during the prompt and resumes after. |
| REQ.audio.call-respects | A phone call shall preempt voice guidance entirely; no prompts shall play during the call. Guidance shall resume after the call ends. | Take a call while navigating; observe silence during the call; observe guidance returns after hang-up. |
| REQ.audio.route-bluetooth | Voice prompts shall play through the user's selected audio output - phone speaker, wired headphones, or Bluetooth audio device. | Pair a Bluetooth headset; navigate; observe prompts come through the headset. |

### 9.17 First-run, units, and locale

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.firstrun.wizard | On first launch the Machine shall present an onboarding flow that establishes language, primary travel profile, requests core permissions, and offers to download at least one regional map. | Fresh install; observe a multi-step wizard covering all four items. |
| REQ.firstrun.skippable | The onboarding flow shall be skippable; the app shall remain functional without completing it, with later prompts at the relevant feature use. | Skip the wizard; observe the app opens to the map at a sensible default; downloading a map is still accessible from the main menu. |
| REQ.units.system | The user shall be able to select metric, imperial, or nautical units; distance, speed, altitude, and depth displays shall update accordingly across the entire app. | Switch to imperial; observe miles, mph, feet across map widgets, search results, and route summaries. |
| REQ.units.coord-format | The user shall be able to select the coordinate display format from at least: decimal degrees, DMS, UTM, MGRS, and Plus Code / OLC. | Switch format; observe all coordinate displays change. |
| REQ.units.time-format | The user shall be able to select 12-hour or 24-hour time display for ETA and timestamps. | Switch format; observe ETA updates. |

### 9.18 Storage and download lifecycle

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.storage.list | The user shall be able to see, for every installed map and resource, its region, file size, version, and download date. | Open downloads / Local; observe the list with all four columns per entry. |
| REQ.storage.delete | The user shall be able to delete a downloaded map or resource at any time and immediately reclaim the disk space. | Delete a region; observe free-space increase and the region no longer usable offline. |
| REQ.storage.resume | A region download interrupted by app death or network loss shall be resumable from the point of interruption rather than restarting from byte zero. | Start a 200 MB download; kill mid-flight; relaunch; resume; observe download completes without re-downloading the previously received bytes. |
| REQ.storage.validate | The Machine shall detect corrupted or partial `.obf` files and refuse to use them rather than crashing; the user shall be offered a re-download. | Corrupt a `.obf` byte; relaunch; observe a clear error and an option to re-download. |
| REQ.storage.location-choice | The user shall be able to keep the OsmAnd data directory on internal storage or on a removable SD card and switch between them. | Move data to SD; observe all maps and tracks remain available. |

### 9.19 Network awareness for downloads

OsmAnd's download surfaces treat metered networks asymmetrically: scheduled OsmAnd Live updates have a hard Wi-Fi-only gate per-map; regional map downloads have an advisory cellular warning; cloud sync currently has no restriction.

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.net.live-wifi-only | The user shall be able, per region, to restrict OsmAnd Live automatic incremental updates to Wi-Fi networks; while restricted, the scheduler shall defer cellular updates rather than perform them. | Enable Wi-Fi-only for a region; force a scheduled update on cellular; observe the update is deferred. Reconnect Wi-Fi; observe it runs. |
| REQ.net.large-download-warn | When the user initiates a regional map download while the device is on a metered network, the Machine shall warn them before consuming cellular data. | Disconnect Wi-Fi; start a 200 MB region download; observe a clear warning dialog with a continue / cancel choice. |
| REQ.net.size-before-download | The user shall see the byte size of a region or resource before committing to its download. | Open the download list; observe size beside each entry. |

### 9.20 App upgrade and data migration

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.upgrade.preserve-user-data | An app upgrade shall preserve favourites, tracks, settings, downloaded maps, profiles, and account state. | Note state pre-upgrade; install a newer version over it; observe all state intact post-upgrade. |
| REQ.upgrade.migrate-legacy | The Machine shall migrate settings and data formats written by earlier versions into the current schema on first launch after upgrade, without user intervention. | Install an old version; populate data; upgrade; observe data still usable, format silently migrated. |
| REQ.upgrade.skip-corrupt | If a migration step encounters corrupt or unreadable user data, the affected item shall be quarantined with a user-visible report rather than aborting the upgrade. | Corrupt a settings file; upgrade; observe the rest of the data still loads and the user is told which item failed. |

### 9.21 In-app purchase and subscription

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.iap.buy | The user shall be able to purchase a one-time premium upgrade or a recurring subscription from inside the app via the platform billing service. | Tap upgrade; complete the OS billing flow; observe premium features unlock. |
| REQ.iap.restore | The user shall be able to restore previously made purchases on a new device or after a re-install, without paying again. | On a clean install signed in to the same store account, tap restore purchases; observe premium reactivated. |
| REQ.iap.state-clear | The user shall be able to see, at any time, which premium entitlements they hold and when a subscription next renews or expires. | Open Premium settings; observe entitlement list and renewal date. |
| REQ.iap.no-feature-loss-offline | A confirmed premium entitlement shall continue to unlock the relevant features when the device is offline. | Disconnect network; observe premium features still available. |

### 9.22 Automotive integration

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.auto.android-auto | When the device is connected to an Android Auto head unit, the user shall be able to navigate, search, and pick favourites from the head-unit display using the Android Auto-constrained UI. | Connect to a head unit; observe an OsmAnd tile; start a route from the head-unit UI; observe guidance on the head unit. |
| REQ.auto.car-permissions | An Android Auto session shall request only the additional permissions required for in-car operation, with a clear rationale specific to the automotive context. | Open OsmAnd on a head unit for the first time; observe automotive-specific permission prompts and rationale. |
| REQ.auto.car-safe-ui | The head-unit UI shall use only the Android Auto template surfaces and shall not present typing or freeform-touch interactions that would distract from driving. | Inspect head-unit screens; observe each uses an Android Auto template such as `NavigationTemplate` or `SearchTemplate`. |

### 9.23 Optional plugin integrations

These are real features delivered by built-in plugins that may be enabled or disabled by the user (`REQ.plugin.enable-disable`).

| ID | Requirement | Acceptance |
|---|---|---|
| REQ.plugin.sensors-pair | The user shall be able to pair external Bluetooth Low Energy or ANT+ sensors - heart rate, cadence, speed, power - with OsmAnd and see live readings during recording. | Enable External Sensors plugin; pair a BLE HR strap; record a ride; observe live HR on a widget and in the resulting GPX. |
| REQ.plugin.sensors-record | Readings from paired sensors shall be embedded in the recorded GPX track for later review. | Open the recorded GPX; observe per-point HR / cadence values. |
| REQ.plugin.a11y-audio-feedback | The Accessibility plugin shall provide spoken feedback for non-navigation events - direction changes, map context-menu items, settings - using the device TTS with a user-configurable speech rate. | Enable Accessibility plugin; navigate the map context menu with TalkBack on; observe spoken descriptions. |
| REQ.plugin.a11y-magnification | The Accessibility plugin shall support map magnification by pinch-zoom with predictable, configurable steps. | Enable plugin; pinch; observe stepped magnification. |

---

## 10. Quality requirements

Each quality requirement is **measurable** and is tied to a scenario. Implementation budgets and measurement methods are in the matching `SPEC.*` in §12.

| ID | Requirement | Acceptance |
|---|---|---|
| QUAL.perf.smooth-map | Map panning and zooming shall feel smooth on a mid-range Android device. | Median frame-time ≤ 33 ms measured per `SPEC.perf.measure-fps`. |
| QUAL.perf.route-under-5s | Route calculation for typical journeys shall complete within an interactive timeframe. | ≤ 10 s for end-to-end routes up to 200 km on a mid-range device; ≤ 5 s for routes ≤ 50 km; per `SPEC.perf.measure-route-time`. |
| QUAL.perf.reroute-under-5s | Re-routing after a deviation shall complete within 5 s. | Measured per `SPEC.perf.measure-reroute-time`. |
| QUAL.perf.startup-under-5s | Cold start to interactive map shall complete within 5 s on a mid-range device. | Measured per `SPEC.perf.measure-startup`. |
| QUAL.perf.battery-screen-off | An active navigation session with the screen off shall consume markedly less power than with the screen on. | Battery drain rate with screen off ≤ 40 % of with screen on; measured per `SPEC.perf.measure-battery`. |
| QUAL.perf.obf-bounded-ram | Memory use shall scale with the visible viewport, not with the total `.obf` size on disk. | Total resident memory ≤ 512 MB on a mid-range device with any single regional `.obf` loaded; measured per `SPEC.storage.measure-ram`. |
| QUAL.reliability.no-crash-on-gps-loss | The app shall remain operational when GPS is lost or intermittent during navigation. | Cut GPS for 5 min and restore; observe no crash, no frozen UI; guidance resumes. |
| QUAL.reliability.no-crash-on-doze | The app shall survive Android Doze mode without losing an in-progress track or active navigation. | Trigger Doze while navigating; observe state intact when device wakes. |
| QUAL.reliability.crash-budget | Field crash rate per `SPEC.obs.crash-rate-measurement` shall not exceed 0.1 % of sessions for the headline build flavour. | Tracked via opt-in telemetry; surfaced in release reviews. |
| QUAL.usability.locales | The UI shall be available in ≥ 70 natural-language locales. | Counted in resource bundles; per `SPEC.i18n.locale-count`. |
| QUAL.access.large-touch-targets | All primary controls shall have a touch target ≥ 48 × 48 dp. | Audited via UI test per `SPEC.access.audit`. |
| QUAL.access.contrast | Text contrast ratios in default light and night styles shall meet WCAG 2.1 AA. | Measured per `SPEC.access.contrast-measurement`. |
| QUAL.security.tls | All non-trivial network traffic with OsmAnd-controlled endpoints shall use TLS 1.2 or higher. | Verified by `SPEC.net.tls-min`. |
| QUAL.security.no-plaintext-creds | OSM and OsmAnd-Cloud credentials shall not be stored in plaintext on disk. | Inspect on-disk storage; verify via `SPEC.sec.cred-storage`. |
| QUAL.privacy.offline-no-location-egress | While operating offline, the app shall make no outbound network call that contains user location. | Capture network at the device boundary while offline-routing; observe zero matching packets. |

---

## 11. Constraints

Constraints are imposed externally rather than being derived from user goals.

### 11.1 Organisational

| ID | Constraint |
|---|---|
| CON.org.gplv3 | The application source code shall be licensed under GPLv3. |
| CON.org.contrib-mit | Contributions made via external pull request shall be accepted under MIT, enabling relicensing into the GPLv3 codebase. |
| CON.org.osm-only-data | Map data shipped with or downloaded by the application shall be derived exclusively from OpenStreetMap; no proprietary map content shall be bundled. |
| CON.org.public-issues | The project shall maintain public issue tracking and accept community contributions through GitHub. |

### 11.2 External

| ID | Constraint |
|---|---|
| CON.legal.gdpr-no-location-upload | While operating offline, no user location data shall be transmitted to any server controlled by the OsmAnd project. |
| CON.legal.store-policies | The published builds shall comply with Google Play and Apple App Store distribution policies, including privacy disclosure. |
| CON.legal.osm-contributor-terms | All uploads to the OSM API shall comply with the OpenStreetMap Contributor Terms. |
| CON.legal.telegram-bot-tos | The OsmAnd Tracker plugin shall comply with Telegram's Bot API Terms of Service. |
| CON.legal.audio-licensing | Bundled voice prompt audio shall not infringe third-party copyright. |

### 11.3 Platform scope

| ID | Constraint |
|---|---|
| CON.platform.android-min-sdk | The Android edition shall support Android 7.0 and above. |
| CON.platform.ios-min-version | The iOS edition shall support iOS 12.0 and above. |
| CON.platform.freemium-quota | The free Google Play flavour shall limit users to 16 map-file downloads; the paid flavour shall have no quota. |

---

## 12. Specifications

This is the only section that may name concrete algorithms, file formats, versions, byte sizes, APIs, classes, and HTTP endpoints. Each `SPEC.*` is bound to one or more shared phenomena from §7 and traces back to one or more `REQ.*` / `QUAL.*` / `CON.*` in §14.

### 12.1 Routing

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.routing.bidirectional-astar | Route calculation shall be performed by a bidirectional A\* search over a directed weighted graph extracted from the `.obf` road-network index. Edge weight shall be `distance / + turn_penalty`. All profile constants - vehicle, speed, priorities, penalties, avoidance - shall be defined in `routing.xml`; no routing constant shall be hard-coded in source. | PHEN.navigate-tap, PHEN.gps-fix |
| SPEC.routing.offline-only | The route-calculation path triggered by `PHEN.navigate-tap` shall not perform any network call; the routing engine reads only `.obf` files via `BinaryMapIndexReader`. The optional online router is reachable only via a profile-level user opt-in. | PHEN.navigate-tap, PHEN.network-available |
| SPEC.routing.deviation-threshold | An off-route condition shall be raised when the device position deviates from the nearest route segment by more than a configurable threshold, continuously over ≥ 3 fixes. | PHEN.gps-fix, PHEN.route-displayed |
| SPEC.routing.reroute-5s | Upon raised off-route condition, a new route to the same destination shall be produced within 5 s on a mid-range device. | PHEN.route-displayed |
| SPEC.routing.intermediate-points | The active route shall support up to 32 ordered intermediate waypoints. | PHEN.navigate-tap |

### 12.2 Map rendering

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.render.opengl-primary | Vector rendering shall use the native OsmAnd-core C++ library over JNI for hardware-accelerated OpenGL ES 2.0 output, when available. A Java 2D Canvas fallback shall be available for devices without OpenGL ES 2.0. | PHEN.map-frame |
| SPEC.render.rules-xml | Rendering rules shall be defined in XML style files parsed and compiled by `RenderingRulesStorage`. Style files shall specify display conditions per zoom level, object type, and tag combination. | PHEN.map-frame |
| SPEC.render.style-runtime-switch | A rendering style change shall take effect on subsequent frames without restarting the application or reloading any `.obf` data. | PHEN.map-frame |
| SPEC.render.day-night | Day / night switching shall be implemented as a rendering-style parameter. Automatic switching shall use the civil twilight time computed from the current location and date. | PHEN.map-frame |
| SPEC.render.position-marker | The position marker shall be redrawn within 1 s of each `PHEN.gps-fix`. Bearing shall be passed through a low-pass filter to suppress jitter from noisy GNSS headings. | PHEN.gps-fix, PHEN.map-frame |

### 12.3 Map data - format and storage

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.data.obf-format | Map data shall use the OsmAnd Binary Format, a compact indexed binary compiled by the external OsmAnd MapCreator from OSM data. The application shall not parse raw OSM XML or PBF at runtime. | PHEN.obf-present |
| SPEC.data.obf-sections | An `.obf` file shall contain a road-network index, a rendered-map-object index, an address index, a POI index, and a public-transport index. Each shall be spatially indexed for tile-based access. | PHEN.obf-present |
| SPEC.data.live-diffs | Live updates shall be delivered as binary diff files applied to the base `.obf`, available at minimum hourly cadence for subscribed users. | PHEN.obf-download |
| SPEC.storage.memory-mapped-obf | `.obf` reads shall use memory-mapped I/O so resident RAM scales with the active viewport / routing bounding box and is bounded by `QUAL.perf.obf-bounded-ram`. | PHEN.obf-present |
| SPEC.data.region-size-budget | At the v1.0 baseline, a regional `.obf` file shall be ≤ 1 GB per country-sized region; growth ≤ 25 % year-over-year is acceptable. Specific case: Germany ≤ 1 GB. | PHEN.obf-download |
| SPEC.data.download-quota | The free Google Play flavour shall enforce a 16 file download cap as configured by `OsmandSettings`; the paid flavour shall enforce no cap. | PHEN.obf-download |

### 12.4 Search

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.search.offline-only | All address, POI, coordinate, and route-along queries shall be resolved by `SearchUICore` against on-device `.obf` indexes without network access. Online Wikipedia lookup is the sole exception and shall be clearly marked. | PHEN.search-query |
| SPEC.search.address-hierarchy | Address search shall follow the hierarchy country → region → city → street → house-number, each level matched with prefix and fuzzy matching against the `.obf` address index. | PHEN.search-query |
| SPEC.search.along-route-corridor | Along-route search shall restrict POI results to amenities within a configurable corridor of the active route polyline. | PHEN.search-query |

### 12.5 Location / GPS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.location.android-api | The Android edition shall subscribe to location through `LocationManager` and `FusedLocationProviderClient`, requesting updates at ≥ 1 Hz and ≥ 1 m displacement while navigating or recording. | PHEN.gps-fix |
| SPEC.location.foreground-service | Background navigation and track recording shall run inside an Android Foreground Service with a persistent notification, holding any necessary wake-locks. | PHEN.gps-fix, PHEN.gps-loss |
| SPEC.location.fusion-low-pass | Reported bearing shall be smoothed with a low-pass filter to suppress GNSS heading jitter at low speeds. | PHEN.gps-fix |
| SPEC.location.tolerate-loss | Loss of location for ≤ 5 min shall not terminate an in-progress navigation or recording session; the last known position shall remain displayed and guidance resumes when fixes return. | PHEN.gps-loss, PHEN.gps-fix |
| SPEC.location.speed-limit | Speed-limit warning shall fire when current speed exceeds the `maxspeed` value for the current road by a tolerance factor. | PHEN.gps-fix, PHEN.alert-tone |

### 12.6 Voice / TTS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.voice.delivery | Spoken instructions shall be delivered through either the Android `TextToSpeech` system service or pre-recorded `.ogg` voice packs, as the user selects. The `voice/` resource directory holds the language packs. | PHEN.tts-utter |
| SPEC.voice.advance-distances | Manoeuvre announcements shall be triggered at distances configurable per profile; defaults: 2000 m, 500 m, and 50 m before the manoeuvre. | PHEN.tts-utter |
| SPEC.voice.command-script | Per-language voice grammars shall be defined as JavaScript files executed by the Rhino engine. | PHEN.tts-utter |
| SPEC.voice.audio-focus | TTS playback shall request transient audio focus and shall respect call audio. | PHEN.tts-utter |

### 12.7 GPX

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.gpx.schema | Recorded and exported tracks shall conform to GPX 1.1. Track points shall carry `lat`, `lon`, `ele`, and `time`. OsmAnd extensions shall be written under the `osmand:` XML namespace. | PHEN.gpx-write |
| SPEC.gpx.statistics | Live track statistics shall be computed using the Haversine formula for distance and forward-difference derivatives for speed. | none direct - internal aggregate over PHEN.gps-fix |
| SPEC.gpx.persist-each-fix | Each `PHEN.gps-fix` while recording is active shall be persisted to the in-progress track file before the next fix arrives, so `REQ.track.survive-kill` holds. | PHEN.gps-fix, PHEN.gpx-write |

### 12.8 Settings and profiles

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.profile.applicationmode | Each profile shall be represented at runtime by an `ApplicationMode` instance; per-profile settings shall be namespaced by mode key in `OsmandSettings`, backed by Android `SharedPreferences`. | PHEN.profile-select |
| SPEC.settings.osf | Settings export / import shall use the `.osf` archive format containing settings JSON, GPX tracks, favourites XML, and per-profile rendering style files. Restore shall be atomic. | none direct |
| SPEC.settings.cloud-sync | `BackupHelper` + `NetworkSettingsHelper` shall sync the user's settings, favourites, and GPX tracks with the OsmAnd Cloud service over TLS, with conflict resolution by timestamp. | PHEN.obf-download |

### 12.9 Third-party API

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.api.aidl-surface | The `OsmAnd-api` module shall expose `IOsmAndAidlInterface` over Android IPC, with method groups for navigation, route, location, search, favourites, map markers, GPX, map layers, widgets, settings, and context menus. Callback delivery shall use `IOsmAndAidlCallback`. | PHEN.aidl-call, PHEN.aidl-callback |
| SPEC.api.permission-gated | All AIDL methods that change Machine state shall be gated behind a user-grantable permission requested at first bind. | PHEN.aidl-call |

### 12.10 Plugin system

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.plugin.osmandplugin-base | Every in-tree plugin shall extend `OsmandPlugin` and live under `plugins/` or `OsmAnd/src/net/osmand/plus/plugins/`. The host shall not refer to plugin classes directly outside of the extension-point hooks. | none direct |
| SPEC.plugin.runtime-enable | A plugin's enable / disable shall not require a process restart; its `registerLayers`, `createWidgets`, and `getPreferences` hooks shall be re-invoked. | none direct |

### 12.11 Logging and observability

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.obs.logger | Logging shall go through `PlatformUtil.getLog` which routes to Android `Log` and to a rolling file `osmand_log.txt` in the OsmAnd data directory. Levels: `error`, `warn`, `info`, `debug`. | PHEN.log-line |
| SPEC.obs.local-log-retention | The local log file shall retain at least the last 7 days of session lines or 5 MB, whichever is larger, rotating older content. | PHEN.log-line |
| SPEC.obs.log-no-pii | Logger calls in production shall not emit precise GPS coordinates, OAuth tokens, OSM passwords, or account email addresses. A redaction unit test shall enforce this rule. | PHEN.log-line |
| SPEC.obs.crash-channel | Crash reports shall be written locally as a `.crash` text file. Upload to a remote channel shall occur only when the user has opted in. | PHEN.crash-report |
| SPEC.obs.opt-in-telemetry | All telemetry, analytics, and crash uploads shall be controlled by an `OsmandSettings` flag defaulting to off. Toggling the flag shall take effect on the next session start. | PHEN.crash-report |
| SPEC.obs.user-error-strings | User-facing error strings shall be defined in `strings.xml`, referenced by a small enum of error categories, and accompanied by a recommended-action hint. | none direct |

### 12.12 Network and TLS

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.net.tls-min | All HTTP traffic to OsmAnd-controlled endpoints shall use TLS 1.2 or higher. Plain HTTP shall not be used for any data transfer. | PHEN.obf-download, PHEN.osm-upload |
| SPEC.net.osm-endpoints | OSM API traffic shall target `https://api.openstreetmap.org/api/0.6/…`. The Tracker plugin shall reach Telegram through `https://api.telegram.org/bot…`. | PHEN.osm-upload |
| SPEC.net.no-offline-egress | When `OsmandSettings.NETWORK_OFFLINE_MODE = true`, no outbound socket shall be opened by routing, rendering, voice, or location code paths. | PHEN.network-available |

### 12.13 Identity, sessions, and credential lifecycle

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.account.cloud-email-flow | OsmAnd Cloud registration and sign-in shall use an email plus verification-code flow served by `BackupHelper` against the OsmAnd Cloud HTTP endpoint over TLS. UI is realised by `backup/ui/AuthorizeFragment` and `BackupAuthorizationFragment`; dialog state by `LoginDialogType`. | PHEN.auth-signin-tap, PHEN.auth-credentials-entered, PHEN.auth-token-issued |
| SPEC.account.osm-oauth2 | OSM account linking shall use OAuth 2.0 with PKCE against `https://www.openstreetmap.org/oauth2/authorize`, with scopes limited to those required for notes, POI edits, and GPX uploads, and a registered redirect URI. Realised by `OsmOAuthHelper` and `OsmOAuthAuthorizationAdapter` under `plugins/osmedit/oauth/`. | PHEN.auth-signin-tap, PHEN.auth-credentials-entered, PHEN.auth-token-issued |
| SPEC.account.osm-redirect-uri | The OSM OAuth redirect URI shall be a custom scheme registered in the Android manifest; the Machine shall validate the returned `state` parameter to prevent cross-session capture. | PHEN.auth-credentials-entered |
| SPEC.account.token-storage | Access and refresh tokens for both OsmAnd Cloud and OSM shall be stored at rest using the storage primitive defined in `SPEC.sec.cred-storage` - Android `EncryptedSharedPreferences` or its iOS equivalent. Tokens shall not be written to logs, GPX files, exported `.osf` archives, or crash reports. | none direct - related to SPEC.obs.log-no-pii |
| SPEC.account.token-refresh | A `401` or token-expired response shall trigger a silent refresh using the stored refresh token. On refresh failure, the session shall be marked invalid, and the user shall be prompted to sign in again only on their next authenticated operation, not pre-emptively. | PHEN.auth-token-revoked |
| SPEC.account.signout-clears-local | On sign-out or unlink, the Machine shall delete the access token, the refresh token, the cached account profile, and any pending changes flagged as identity-bound such as un-uploaded OSM contributions. Non-account-bound user data shall not be touched. | PHEN.auth-token-revoked |
| SPEC.account.delete-confirms | Account deletion shall require an explicit confirmation step and shall additionally inform the user that data already synced to the OsmAnd Cloud server will be removed server-side. Realised by `DeleteAccountFragment`. | PHEN.auth-token-revoked |
| SPEC.account.session-state | Signed-in or linked state shall be exposed to the rest of the Machine through `BackupHelper.isRegistered` and `OsmOAuthHelper.isAuthorised`. Surfaces that require an identity shall query these and offer an in-app sign-in entry point when false. | PHEN.auth-signed-in |

### 12.14 Map interaction

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.interact.multitouch | Multi-touch gestures shall be detected by `MultiTouchSupport` and `DoubleTapScaleDetector` in `plus/views/`. The viewport state - centre, zoom, rotation, tilt - shall be expressed as a `RotatedTileBox`. | PHEN.map-gesture |
| SPEC.interact.context-menu | A long-press on the map shall be routed through `MapActivity` to `MapContextMenu.show()` with the projected geographic point; menu items are owned by `MapContextMenuFragment`. | PHEN.long-press |
| SPEC.interact.markers | Map markers shall be modelled by `MapMarker` and the surrounding `plus/mapmarkers/` subsystem. Persistence shall be in the same store as favourites, exportable through `SPEC.settings.osf`. | PHEN.long-press |

### 12.15 Sharing and deep links

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.share.intent-filters | The manifest shall declare `<intent-filter>` entries for outgoing `ACTION_SEND` and incoming `ACTION_VIEW` with MIME and `pathPattern` matchers for `.gpx`, `.kml`, `.kmz`, `.obf`, `.sqlitedb`, `.osf`. Outgoing shares shall be assembled by `ShareMenu` / `ShareMenuFragment`. | PHEN.share-out, PHEN.share-in |
| SPEC.share.deep-link | The manifest shall register App Links for `osmand.net/go` and `osmand.net/map`; parsing of the encoded view shall be implemented in `IntentHelper`. | PHEN.deep-link |
| SPEC.share.receiver | Incoming shared files shall be received by `ShareSheetReceiver`, validated, and routed to the appropriate importer - GPX into `GpxFile`, settings into `SettingsHelper`, maps into `DownloadActivity`. | PHEN.share-in |

### 12.16 OS permissions

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.perm.runtime-request | Runtime permission requests shall be issued through `AndroidUtils.requestPermissions` from the fragment / activity that actually needs the permission, not from `OsmandApplication`. A rationale UI shall be shown when `shouldShowRequestPermissionRationale` returns true. | PHEN.permission-prompt, PHEN.permission-decision |
| SPEC.perm.declared | The manifest shall declare the permissions actually used: `ACCESS_FINE_LOCATION`, `ACCESS_BACKGROUND_LOCATION`, `ACCESS_COARSE_LOCATION`, `POST_NOTIFICATIONS`, `RECORD_AUDIO`, `CAMERA`, `BLUETOOTH`, `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT`, `FOREGROUND_SERVICE`, `FOREGROUND_SERVICE_LOCATION`, `WAKE_LOCK`, `MODIFY_AUDIO_SETTINGS`. | PHEN.permission-prompt |
| SPEC.perm.feature-gates | Features that require a denied permission shall be gated by an in-app screen that explains the dependency and offers a one-tap re-request, rather than failing silently. | PHEN.permission-decision |

### 12.17 Notifications

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.notif.helper | Notification posting shall go through `NotificationHelper`; concrete channels are realised by `NavigationNotification`, `DownloadNotification`, `GpxNotification`, and `CarAppNotification`. | PHEN.notification-shown |
| SPEC.notif.channels | Each notification category shall live in a distinct OS notification channel created at app startup; the channel IDs shall be stable across versions to preserve user-set importance. | PHEN.notification-shown |
| SPEC.notif.actions | The navigation, recording, and download notifications shall expose their respective stop / pause / cancel actions as notification action buttons, dispatched to the same handlers as the in-app UI. | PHEN.notification-action |

### 12.18 Audio focus

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.audio.request | `AudioFocusHelperImpl` shall request transient-may-duck focus from `AudioManager` before each prompt and release it on completion; the request stream shall be `STREAM_MUSIC` by default, configurable to `STREAM_VOICE_CALL` / `STREAM_NOTIFICATION`. | PHEN.audio-focus-gained, PHEN.tts-utter |
| SPEC.audio.focus-listener | An `OnAudioFocusChangeListener` shall pause TTS on `AUDIOFOCUS_LOSS_TRANSIENT` and abandon the current prompt on `AUDIOFOCUS_LOSS`. | PHEN.audio-focus-lost |
| SPEC.audio.output-route | Voice prompts shall follow the OS-selected audio output; explicit per-profile overrides shall write to `OsmandSettings.AUDIO_STREAM_GUIDANCE`. | PHEN.tts-utter |

### 12.19 First-run, units, and locale

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.firstrun.wizard | The first-run wizard shall be implemented by `FirstUsageWizardFragment` + sibling fragments (`FirstUsageWelcomeFragment`, `FirstUsageLocationBottomSheet`, `FirstUsageActionsBottomSheet`), driven by `WizardType`. It shall be triggered exactly once and recorded as completed in `OsmandSettings`. | none direct |
| SPEC.units.constants | Unit selection shall be modelled by `MetricsConstants`, `SpeedConstants`, and `AngularConstants`; the selected enum value shall be read by `OsmAndFormatter` for every display. | none direct |
| SPEC.units.coordinate-formats | Coordinate display shall be implemented by `LocationConvert` / `LocationFormatter`, supporting at minimum the formats enumerated in `CoordinateInputFormats`: decimal degrees, DMS, UTM, MGRS, and Plus Code / OLC. | none direct |

### 12.20 Storage and download lifecycle

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.storage.local-index | The local-map list shall be assembled by `LocalIndexInfo` / `LocalIndexesFragment` over the data directory defined by `IndexConstants`; sizes shall be read from the filesystem rather than cached. | none direct |
| SPEC.storage.resume | Region downloads shall be performed by `DownloadFileHelper` using HTTP `Range` requests to resume from a partial-file offset on retry. | PHEN.obf-download, PHEN.download-progress |
| SPEC.storage.validate | `DownloadValidationManager` shall verify each downloaded `.obf` for header validity and expected size before declaring the download complete; corrupted files shall be moved out of the active maps directory. | PHEN.obf-download |
| SPEC.storage.path-options | The user-selected data root shall be persisted in `OsmandSettings.EXTERNAL_STORAGE_DIR`; switching paths shall trigger a one-time atomic move of existing content. | none direct |

### 12.21 Network awareness

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.net.live-wifi-only | OsmAnd Live's per-map Wi-Fi-only restriction shall be a `BooleanPreference` created by `LiveUpdatesHelper.preferenceDownloadViaWiFi(name)`; the alarm receiver (`LiveUpdatesAlarmReceiver`) and the executor (`PerformLiveUpdateAsyncTask`) shall both honour the flag, deferring on cellular. | PHEN.obf-download, PHEN.network-available |
| SPEC.net.large-download-warn | `DownloadValidationManager` shall, on detecting a metered network for a regional download, show the `download_using_mobile_internet` confirmation; the dialog shall display the byte size of the requested download. | PHEN.obf-download, PHEN.network-available |
| SPEC.net.size-display | Region-list rows shall show the byte size from the server's manifest as soon as the manifest has been fetched. | none direct |

### 12.22 App upgrade and migration

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.upgrade.startup-migrate | On every cold start `AppInitializer` shall invoke `AppVersionUpgradeOnInit` with the last-known version code and the current version code; the upgrade path runs the per-version migration steps in order. | PHEN.app-upgrade |
| SPEC.upgrade.quarantine | A migration step that throws shall be caught at step boundaries; the affected file or preference shall be renamed with a `.quarantine` suffix and a log line written via `SPEC.obs.logger`. | PHEN.app-upgrade, PHEN.log-line |
| SPEC.upgrade.version-store | The persisted last-known version code shall be written to `OsmandSettings.PREVIOUS_INSTALLED_VERSION` only after a migration completes successfully. | PHEN.app-upgrade |

### 12.23 In-app purchase

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.iap.billing-client | Purchases shall use the Google Play Billing Library through `InAppPurchaseHelper` / `InAppPurchases`; SKU sets and feature gates shall be defined in `InAppPurchases`. | PHEN.purchase-event |
| SPEC.iap.restore | Restore shall query the user's owned purchases via `BillingClient.queryPurchasesAsync` and rebuild the local entitlement state without prompting for payment. | PHEN.purchase-event |
| SPEC.iap.entitlement-cache | A successful purchase or restore shall be cached locally so premium features stay available offline; expiry of a subscription shall be checked online when the device is next connected and shall not lock the user out until the configured grace period has passed. | PHEN.purchase-event, PHEN.network-available |
| SPEC.iap.auto-purchase-screen | Premium prompts on Android Auto head units shall be presented through `RequestPurchaseScreen` using the Android Auto template UI; the actual purchase shall fall back to the phone. | PHEN.car-session, PHEN.purchase-event |

### 12.24 Automotive

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.auto.car-service | Android Auto integration shall be hosted by `NavigationCarAppService` extending `CarAppService`; the manifest shall declare the service with the Auto-required intent-filter and `androidx.car.app.minCarApiLevel` metadata. | PHEN.car-session |
| SPEC.auto.templates | Each in-car screen shall extend `androidx.car.app.Screen` and emit only Android Auto-permitted templates (`NavigationTemplate`, `MapTemplate`, `SearchTemplate`, `PlaceListNavigationTemplate`, `MessageTemplate`). Realised under `plus/auto/screens/`. | PHEN.car-session |
| SPEC.auto.permission-screen | First-time use on a head unit shall route through `RequestPermissionScreen` for the automotive-specific permission grants. | PHEN.permission-prompt |

### 12.25 External sensors

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.sensor.plugin | External-sensor support shall be packaged as the `ExternalSensorsPlugin` under `plus/plugins/externalsensors/`. Discovery, pairing, and live streaming shall use the Android `BluetoothAdapter` for BLE devices and the dsi.ant.plugins.antplus package for ANT+. | PHEN.sensor-reading |
| SPEC.sensor.device-types | Supported device types - heart rate, bike speed, bike cadence, running cadence, power, blood pressure - shall be enumerated by `DeviceType` with one concrete implementation class per type under `plus/plugins/externalsensors/devices/`. | PHEN.sensor-reading |
| SPEC.sensor.gpx-embed | Sensor readings shall be embedded in the recording GPX under the `osmand:` namespace defined by `SPEC.gpx.schema`, sample-aligned with the position fix that bears the same timestamp. | PHEN.sensor-reading, PHEN.gpx-write |

### 12.26 Accessibility

| ID | Specification | Phenomena |
|---|---|---|
| SPEC.access.plugin | Accessibility shall be packaged as `AccessibilityPlugin` with helpers `AccessibilityAssistant`, `AccessibilityActionsProvider`, `MapAccessibilityActions`. TTS audio for assistive output shall use `AudioAttributes.USAGE_ASSISTANCE_ACCESSIBILITY`. | PHEN.a11y-event, PHEN.tts-utter |
| SPEC.access.content-descriptions | Every primary UI control shall set a non-default `contentDescription`; map-context-menu items and search results shall be readable by TalkBack without explicit user interaction. | PHEN.a11y-event |
| SPEC.access.speech-rate | TTS speech rate, pitch, and verbosity shall be configurable settings (`SPEECH_RATE`, `DIRECTION_AUDIO_FEEDBACK` in `OsmandSettings`) and shall apply to both turn-by-turn guidance and accessibility prompts. | PHEN.tts-utter |

### 12.27 Performance - measurement methods

These specifications define **how** the `QUAL.perf.*` numbers in §10 are measured. They are part of S because they are statements about boundary-observable signals.

| ID | Specification |
|---|---|
| SPEC.perf.measure-fps | Frame-time is measured via `Choreographer` callbacks during a scripted 60 s pan / zoom over a downloaded region on a reference device. |
| SPEC.perf.measure-route-time | Route time is measured from `PHEN.navigate-tap` to `PHEN.route-displayed` over a curated test corpus of 50 routes per profile per region, on the reference device. |
| SPEC.perf.measure-reroute-time | Reroute time is measured from the moment `SPEC.routing.deviation-threshold` fires to `PHEN.route-displayed`, on the reference device. |
| SPEC.perf.measure-startup | Cold start is measured from `am start` to the first frame in which `MapActivity` is interactive, after force-stop and cache clear. |
| SPEC.perf.measure-battery | Battery drain is measured with `dumpsys batterystats` over a 1 h scripted drive, screen on vs screen off, on the reference device. |
| SPEC.storage.measure-ram | Resident memory is measured by `Debug.getMemoryInfo` after 5 min of pan / zoom over the largest enabled region. |
| SPEC.i18n.locale-count | The locale count is the number of `values-xx*/strings.xml` bundles with ≥ 90 % string-key coverage of `values/strings.xml`. |
| SPEC.access.audit | Touch-target sizes are audited by automated UI test over all primary control screens. |
| SPEC.access.contrast-measurement | Contrast ratios are computed from the rendered colour samples of label and surface in the default day and night styles, against WCAG 2.1 AA thresholds. |
| SPEC.sec.cred-storage | OSM and OsmAnd-Cloud credentials are stored in Android `EncryptedSharedPreferences`. |
| SPEC.obs.crash-rate-measurement | Crash-free-session rate is computed from opt-in telemetry as `1 − crashes_with_user_consent / sessions_with_user_consent`, aggregated weekly per flavour. |

---

## 13. Justification - `D, S ⊢ R` for headline requirements

For each headline requirement, the table below names the domain assumptions and specifications whose combination entails it. The argument is informal but should be readable as: *given these facts and these promises, the requirement is true.*

| REQ | Entailing `DOM` | Entailing `SPEC` | Argument sketch |
|---|---|---|---|
| REQ.routing.offline-route | DOM.osm.tag-conventions, DOM.fs.persistence | SPEC.routing.offline-only, SPEC.routing.bidirectional-astar, SPEC.data.obf-format, SPEC.data.obf-sections, SPEC.storage.memory-mapped-obf | OSM road graph is encoded in the `.obf` road-network index, the file persists on disk, A\* runs over that graph without network, so an offline route is producible. |
| REQ.routing.recompute-on-deviation | DOM.gps.fix-outdoors | SPEC.location.android-api, SPEC.routing.deviation-threshold, SPEC.routing.reroute-5s | Fixes arrive at ≥ 1 Hz outdoors; the deviation rule raises an off-route event; the reroute SPEC delivers an updated route within 5 s. |
| REQ.guidance.voice-announcement | DOM.android.tts, DOM.audio.driving-noise | SPEC.voice.delivery, SPEC.voice.advance-distances, SPEC.voice.audio-focus | TTS is available; the Machine emits utterances in advance and claims audio focus so they are audible above road noise. |
| REQ.guidance.announce-in-advance | DOM.user.attention | SPEC.voice.advance-distances | The 2000 / 500 / 50 m defaults provide head-room for any profile's typical speed. |
| REQ.map.offline-render | DOM.fs.persistence | SPEC.data.obf-format, SPEC.data.obf-sections, SPEC.render.opengl-primary, SPEC.render.rules-xml, SPEC.storage.memory-mapped-obf | All data and rules needed to paint a tile are local; the renderer runs offline. |
| REQ.search.address | DOM.osm.tag-conventions | SPEC.search.offline-only, SPEC.search.address-hierarchy, SPEC.data.obf-sections | Address index is on-disk; the search engine queries it without network. |
| REQ.track.survive-kill | DOM.power.kill, DOM.fs.persistence | SPEC.gpx.persist-each-fix, SPEC.location.foreground-service | Each fix is persisted before the next; the foreground service reduces kill probability; survivors are recoverable from the on-disk file. |
| REQ.navigation.tolerate-gps-loss | DOM.gps.intermittent, DOM.gps.fix-degraded | SPEC.location.tolerate-loss, SPEC.location.foreground-service | Loss for ≤ 5 min is explicitly tolerated; the service keeps the process alive; guidance resumes on next fix. |
| REQ.obs.local-log | DOM.fs.persistence | SPEC.obs.logger, SPEC.obs.local-log-retention | Lines are written to a rotating file under the OsmAnd data directory. |
| REQ.obs.no-pii-by-default | - | SPEC.obs.log-no-pii, SPEC.obs.opt-in-telemetry | Production logger calls do not emit PII; remote upload is gated behind opt-in. |
| REQ.privacy.offline-no-location-egress | DOM.network.intermittent | SPEC.net.no-offline-egress, SPEC.routing.offline-only, SPEC.search.offline-only | No code path on the offline routing / rendering / search trace opens a socket. |
| REQ.thirdparty.control | DOM.android.location-api | SPEC.api.aidl-surface, SPEC.api.permission-gated | The AIDL surface exposes the listed method groups; permission gating preserves user control. |
| REQ.account.cloud-register | DOM.account.cloud-account-via-server | SPEC.account.cloud-email-flow, SPEC.net.tls-min, SPEC.account.token-storage | The email flow drives the OsmAnd Cloud HTTP service over TLS; the service issues a session token; the Machine stores it encrypted. |
| REQ.account.osm-link | DOM.account.osm-account-external, DOM.account.oauth-redirect | SPEC.account.osm-oauth2, SPEC.account.osm-redirect-uri, SPEC.account.token-storage | The user authenticates against osm.org; the redirect URI delivers the authorisation code back; the Machine exchanges it for a token, validates `state`, and stores it. |
| REQ.account.session-survives-restart | DOM.fs.persistence, DOM.account.tokens-expire | SPEC.account.token-storage, SPEC.account.session-state | Tokens are persisted in encrypted storage and remain valid across process lifetimes; the session-state predicate is read on each launch. |
| REQ.account.silent-refresh | DOM.account.tokens-expire | SPEC.account.token-refresh | The refresh token exchanges a fresh access token without user interaction; the user is only prompted when the refresh itself fails. |
| REQ.account.cloud-signout | - | SPEC.account.signout-clears-local | Local token deletion is sufficient to end the session for in-app surfaces. |
| REQ.account.cloud-delete | DOM.account.cloud-account-via-server | SPEC.account.delete-confirms, SPEC.account.signout-clears-local, SPEC.net.tls-min | A server-side delete is requested over TLS; on confirmation, the local credential set is cleared. |
| REQ.account.clear-auth-errors | - | SPEC.obs.user-error-strings | Auth error strings are translatable and carry a recommended action. |
| REQ.interact.long-press-pin | - | SPEC.interact.context-menu, SPEC.interact.markers | A long-press is routed to `MapContextMenu` with the projected geographic point; the menu offers the standard set of actions. |
| REQ.share.deep-link-in | DOM.android.deep-link, DOM.android.intent-system | SPEC.share.deep-link, SPEC.share.intent-filters | App Links are registered; the OS dispatches matching URLs; `IntentHelper` parses them. |
| REQ.perm.rationale | DOM.android.runtime-permissions | SPEC.perm.runtime-request, SPEC.perm.feature-gates | The fragment needing a permission shows a rationale before invoking `requestPermissions`, and shows a feature-gate screen on denial. |
| REQ.perm.graceful-denial | DOM.android.runtime-permissions | SPEC.perm.feature-gates | Denied permissions disable the affected features behind explicit gates rather than crashing. |
| REQ.notif.nav-foreground | DOM.android.fgs, DOM.android.notification-channels | SPEC.notif.helper, SPEC.notif.channels, SPEC.notif.actions, SPEC.location.foreground-service | The navigation session's foreground service holds a channel-tagged notification with live content and a stop action. |
| REQ.audio.call-respects | DOM.android.audio-focus | SPEC.audio.request, SPEC.audio.focus-listener | The focus listener suspends TTS on transient loss and abandons on permanent loss, which is what a call triggers. |
| REQ.firstrun.wizard | - | SPEC.firstrun.wizard | The wizard runs once and records completion. |
| REQ.units.system | - | SPEC.units.constants | The selected `MetricsConstants` value drives all formatter output. |
| REQ.storage.resume | DOM.network.intermittent | SPEC.storage.resume | HTTP `Range` requests resume from the recorded partial-file offset. |
| REQ.storage.validate | - | SPEC.storage.validate | The validator catches corruption before the file enters the active maps directory. |
| REQ.net.live-wifi-only | DOM.network.metered, DOM.android.fgs | SPEC.net.live-wifi-only | The per-map preference is checked by both the alarm receiver and the executor before any cellular transfer. |
| REQ.upgrade.preserve-user-data | DOM.android.app-upgrade, DOM.fs.persistence | SPEC.upgrade.startup-migrate, SPEC.upgrade.quarantine, SPEC.upgrade.version-store | Migration runs at startup, isolates failures, and only commits the new version code on success. |
| REQ.iap.restore | DOM.android.play-billing | SPEC.iap.billing-client, SPEC.iap.restore | The billing client rebuilds entitlements from the user's owned-purchase list. |
| REQ.iap.no-feature-loss-offline | DOM.network.intermittent | SPEC.iap.entitlement-cache | A cached entitlement keeps premium features unlocked while offline. |
| REQ.auto.android-auto | DOM.android.auto | SPEC.auto.car-service, SPEC.auto.templates, SPEC.auto.permission-screen | The CarAppService exposes templates the head unit can render; permission screen handles automotive grants. |
| REQ.plugin.sensors-pair | DOM.android.bluetooth | SPEC.sensor.plugin, SPEC.sensor.device-types | Pairing uses the OS Bluetooth stack; per-type device classes handle the GATT / ANT+ protocol. |
| REQ.plugin.a11y-audio-feedback | DOM.android.accessibility | SPEC.access.plugin, SPEC.access.speech-rate | The Accessibility plugin emits TTS with accessibility usage and uses the same speech-rate setting as guidance. |

---

## 14. Traceability matrix

The matrix below is read as *which `SPEC.*` and `DOM.*` does each `REQ.*` / `QUAL.*` rely on?* and *which `REQ.*` does each `SPEC.*` serve?* It is generated from §13 and from the per-row `Phenomena` columns of §12 by inspection.

### 14.1 Requirement → Specification → Domain

| REQ / QUAL | SPEC | DOM |
|---|---|---|
| REQ.routing.offline-route | SPEC.routing.offline-only, SPEC.routing.bidirectional-astar, SPEC.data.obf-format, SPEC.data.obf-sections, SPEC.storage.memory-mapped-obf | DOM.osm.tag-conventions, DOM.fs.persistence |
| REQ.routing.travel-modes | SPEC.routing.bidirectional-astar, SPEC.profile.applicationmode | DOM.osm.tag-conventions |
| REQ.routing.intermediate-waypoints | SPEC.routing.intermediate-points | - |
| REQ.routing.recompute-on-deviation | SPEC.routing.deviation-threshold, SPEC.routing.reroute-5s, SPEC.location.android-api | DOM.gps.fix-outdoors |
| REQ.routing.avoid-classes | SPEC.routing.bidirectional-astar | DOM.osm.tag-conventions |
| REQ.routing.cancel | SPEC.api.aidl-surface, local UI | - |
| REQ.guidance.voice-announcement | SPEC.voice.delivery, SPEC.voice.command-script, SPEC.voice.audio-focus | DOM.android.tts, DOM.audio.driving-noise |
| REQ.guidance.announce-in-advance | SPEC.voice.advance-distances | DOM.user.attention |
| REQ.guidance.user-language | SPEC.voice.delivery, SPEC.voice.command-script | DOM.user.languages |
| REQ.guidance.glanceable-screen | SPEC.render.position-marker, SPEC.render.rules-xml | DOM.user.attention |
| REQ.guidance.speed-limit-warn | SPEC.location.speed-limit | DOM.osm.tag-conventions |
| REQ.guidance.lane-hints | SPEC.render.rules-xml | DOM.osm.tag-conventions |
| REQ.map.offline-render | SPEC.render.opengl-primary, SPEC.render.rules-xml, SPEC.data.obf-format, SPEC.storage.memory-mapped-obf | DOM.fs.persistence |
| REQ.map.position-orientation | SPEC.render.position-marker, SPEC.location.android-api | DOM.gps.fix-outdoors |
| REQ.map.rotate-with-direction | SPEC.render.position-marker | - |
| REQ.map.day-night | SPEC.render.day-night | - |
| REQ.map.local-script-labels | SPEC.render.rules-xml | DOM.user.languages |
| REQ.map.gpx-overlay | SPEC.gpx.schema | - |
| REQ.map.style-switch-runtime | SPEC.render.style-runtime-switch, SPEC.render.rules-xml | - |
| REQ.map.online-raster-overlay | SPEC.net.tls-min | DOM.network.intermittent |
| REQ.search.address | SPEC.search.offline-only, SPEC.search.address-hierarchy, SPEC.data.obf-sections | DOM.osm.tag-conventions |
| REQ.search.poi | SPEC.search.offline-only, SPEC.data.obf-sections | DOM.osm.tag-conventions |
| REQ.search.coordinates | SPEC.search.offline-only | - |
| REQ.search.along-route | SPEC.search.along-route-corridor | - |
| REQ.search.recent | SPEC.profile.applicationmode | - |
| REQ.data.download-region | SPEC.data.obf-format, SPEC.net.tls-min | DOM.network.intermittent |
| REQ.data.external-storage | SPEC.storage.memory-mapped-obf, SPEC.data.obf-format | DOM.device.storage |
| REQ.data.fresher-updates | SPEC.data.live-diffs, SPEC.net.tls-min | - |
| REQ.data.update-monthly | SPEC.data.obf-format | - |
| REQ.track.record-background | SPEC.location.foreground-service, SPEC.gpx.schema | DOM.android.fgs, DOM.android.doze |
| REQ.track.stats | SPEC.gpx.statistics | - |
| REQ.track.survive-kill | SPEC.gpx.persist-each-fix, SPEC.location.foreground-service | DOM.power.kill, DOM.fs.persistence |
| REQ.track.import-export | SPEC.gpx.schema | - |
| REQ.track.upload-osm | SPEC.net.osm-endpoints, SPEC.net.tls-min | DOM.osm.contributor-account |
| REQ.profile.multi | SPEC.profile.applicationmode | - |
| REQ.favourites.save | SPEC.profile.applicationmode | - |
| REQ.settings.backup | SPEC.settings.osf | DOM.fs.persistence |
| REQ.settings.cloud-sync | SPEC.settings.cloud-sync, SPEC.net.tls-min | DOM.network.intermittent |
| REQ.osm.report-error | SPEC.net.osm-endpoints | DOM.osm.contributor-account |
| REQ.osm.contribute-edit | SPEC.net.osm-endpoints | DOM.osm.contributor-account |
| REQ.plugin.enable-disable | SPEC.plugin.osmandplugin-base, SPEC.plugin.runtime-enable | - |
| REQ.thirdparty.control | SPEC.api.aidl-surface, SPEC.api.permission-gated | DOM.android.location-api |
| REQ.obs.local-log | SPEC.obs.logger, SPEC.obs.local-log-retention | DOM.fs.persistence |
| REQ.obs.opt-in-telemetry | SPEC.obs.opt-in-telemetry, SPEC.obs.crash-channel | - |
| REQ.obs.no-pii-by-default | SPEC.obs.log-no-pii, SPEC.obs.opt-in-telemetry | - |
| REQ.obs.user-visible-errors | SPEC.obs.user-error-strings | - |
| REQ.account.optional | - | - |
| REQ.account.cloud-register | SPEC.account.cloud-email-flow, SPEC.account.token-storage, SPEC.net.tls-min | DOM.account.cloud-account-via-server |
| REQ.account.cloud-signin | SPEC.account.cloud-email-flow, SPEC.account.token-storage, SPEC.net.tls-min | DOM.account.cloud-account-via-server |
| REQ.account.cloud-signout | SPEC.account.signout-clears-local | - |
| REQ.account.cloud-delete | SPEC.account.delete-confirms, SPEC.account.signout-clears-local, SPEC.net.tls-min | DOM.account.cloud-account-via-server |
| REQ.account.osm-link | SPEC.account.osm-oauth2, SPEC.account.osm-redirect-uri, SPEC.account.token-storage, SPEC.net.tls-min | DOM.account.osm-account-external, DOM.account.oauth-redirect |
| REQ.account.osm-unlink | SPEC.account.signout-clears-local | - |
| REQ.account.session-survives-restart | SPEC.account.token-storage, SPEC.account.session-state | DOM.fs.persistence, DOM.account.tokens-expire |
| REQ.account.silent-refresh | SPEC.account.token-refresh | DOM.account.tokens-expire |
| REQ.account.clear-auth-errors | SPEC.obs.user-error-strings | - |
| REQ.interact.zoom-pinch | SPEC.interact.multitouch | - |
| REQ.interact.zoom-double-tap | SPEC.interact.multitouch | - |
| REQ.interact.pan | SPEC.interact.multitouch | - |
| REQ.interact.rotate | SPEC.interact.multitouch | - |
| REQ.interact.tilt | SPEC.interact.multitouch | - |
| REQ.interact.long-press-pin | SPEC.interact.context-menu | - |
| REQ.interact.add-marker | SPEC.interact.context-menu, SPEC.interact.markers | DOM.fs.persistence |
| REQ.share.location-out | SPEC.share.intent-filters | DOM.android.intent-system |
| REQ.share.gpx-out | SPEC.share.intent-filters | DOM.android.intent-system |
| REQ.share.file-in | SPEC.share.receiver, SPEC.share.intent-filters | DOM.android.intent-system |
| REQ.share.deep-link-in | SPEC.share.deep-link | DOM.android.deep-link |
| REQ.perm.rationale | SPEC.perm.runtime-request | DOM.android.runtime-permissions |
| REQ.perm.graceful-denial | SPEC.perm.feature-gates | DOM.android.runtime-permissions |
| REQ.perm.deferred-request | SPEC.perm.runtime-request | DOM.android.runtime-permissions |
| REQ.notif.nav-foreground | SPEC.notif.helper, SPEC.notif.channels, SPEC.notif.actions, SPEC.location.foreground-service | DOM.android.notification-channels, DOM.android.fgs |
| REQ.notif.recording | SPEC.notif.helper, SPEC.notif.channels, SPEC.notif.actions | DOM.android.notification-channels |
| REQ.notif.download-progress | SPEC.notif.helper, SPEC.notif.channels | DOM.android.notification-channels |
| REQ.notif.user-controllable | SPEC.notif.channels | DOM.android.notification-channels |
| REQ.audio.focus-request | SPEC.audio.request | DOM.android.audio-focus |
| REQ.audio.call-respects | SPEC.audio.focus-listener | DOM.android.audio-focus |
| REQ.audio.route-bluetooth | SPEC.audio.output-route | DOM.android.bluetooth |
| REQ.firstrun.wizard | SPEC.firstrun.wizard | - |
| REQ.firstrun.skippable | SPEC.firstrun.wizard | - |
| REQ.units.system | SPEC.units.constants | - |
| REQ.units.coord-format | SPEC.units.coordinate-formats | - |
| REQ.units.time-format | SPEC.units.constants | - |
| REQ.storage.list | SPEC.storage.local-index | DOM.fs.persistence |
| REQ.storage.delete | SPEC.storage.local-index | DOM.fs.persistence |
| REQ.storage.resume | SPEC.storage.resume | DOM.network.intermittent |
| REQ.storage.validate | SPEC.storage.validate | - |
| REQ.storage.location-choice | SPEC.storage.path-options | DOM.device.storage |
| REQ.net.live-wifi-only | SPEC.net.live-wifi-only | DOM.network.metered |
| REQ.net.large-download-warn | SPEC.net.large-download-warn | DOM.network.metered |
| REQ.net.size-before-download | SPEC.net.size-display | - |
| REQ.upgrade.preserve-user-data | SPEC.upgrade.startup-migrate, SPEC.upgrade.version-store | DOM.android.app-upgrade, DOM.fs.persistence |
| REQ.upgrade.migrate-legacy | SPEC.upgrade.startup-migrate | DOM.android.app-upgrade |
| REQ.upgrade.skip-corrupt | SPEC.upgrade.quarantine | - |
| REQ.iap.buy | SPEC.iap.billing-client | DOM.android.play-billing |
| REQ.iap.restore | SPEC.iap.restore | DOM.android.play-billing |
| REQ.iap.state-clear | SPEC.iap.entitlement-cache | DOM.android.play-billing |
| REQ.iap.no-feature-loss-offline | SPEC.iap.entitlement-cache | DOM.network.intermittent |
| REQ.auto.android-auto | SPEC.auto.car-service, SPEC.auto.templates | DOM.android.auto |
| REQ.auto.car-permissions | SPEC.auto.permission-screen | DOM.android.auto, DOM.android.runtime-permissions |
| REQ.auto.car-safe-ui | SPEC.auto.templates | DOM.android.auto |
| REQ.plugin.sensors-pair | SPEC.sensor.plugin, SPEC.sensor.device-types | DOM.android.bluetooth |
| REQ.plugin.sensors-record | SPEC.sensor.gpx-embed | - |
| REQ.plugin.a11y-audio-feedback | SPEC.access.plugin, SPEC.access.speech-rate | DOM.android.accessibility |
| REQ.plugin.a11y-magnification | SPEC.access.plugin | DOM.android.accessibility |
| QUAL.perf.smooth-map | SPEC.perf.measure-fps, SPEC.render.opengl-primary | - |
| QUAL.perf.route-under-5s | SPEC.perf.measure-route-time, SPEC.routing.bidirectional-astar | - |
| QUAL.perf.reroute-under-5s | SPEC.perf.measure-reroute-time, SPEC.routing.reroute-5s | - |
| QUAL.perf.startup-under-5s | SPEC.perf.measure-startup | - |
| QUAL.perf.battery-screen-off | SPEC.perf.measure-battery, SPEC.location.foreground-service | DOM.device.battery, DOM.android.doze |
| QUAL.perf.obf-bounded-ram | SPEC.storage.measure-ram, SPEC.storage.memory-mapped-obf | - |
| QUAL.reliability.no-crash-on-gps-loss | SPEC.location.tolerate-loss, SPEC.location.foreground-service | DOM.gps.intermittent |
| QUAL.reliability.no-crash-on-doze | SPEC.location.foreground-service | DOM.android.doze, DOM.android.fgs |
| QUAL.reliability.crash-budget | SPEC.obs.crash-rate-measurement, SPEC.obs.opt-in-telemetry | - |
| QUAL.usability.locales | SPEC.i18n.locale-count | DOM.user.languages |
| QUAL.access.large-touch-targets | SPEC.access.audit | - |
| QUAL.access.contrast | SPEC.access.contrast-measurement | - |
| QUAL.security.tls | SPEC.net.tls-min | - |
| QUAL.security.no-plaintext-creds | SPEC.sec.cred-storage | - |
| QUAL.privacy.offline-no-location-egress | SPEC.net.no-offline-egress, SPEC.routing.offline-only, SPEC.search.offline-only | DOM.network.intermittent |

### 14.2 Specification → Module / Class *(informative, non-normative)*

This subsection is **informative**: it points from each `SPEC.*` family to the code that realises it (Jackson's **P**, the Program). Strictly, P is out of scope for the World/Machine Model - the WMM ends at the boundary defined in §7. The mapping below is provided as a convenience for development, code review, and for grounding the class and dynamics diagrams in `diagrams/`. It is not load-bearing for the `D, S ⊢ R` argument in §13 and may be omitted by a reader who only wants the WMM proper.

| SPEC | Realised in |
|---|---|
| SPEC.routing.* | `RoutingHelper`, `RoutingContext`, `routing.xml` resources |
| SPEC.render.* | `OsmandRenderer`, `MapRenderRepositories`, `RendererRegistry`; `RenderingRulesStorage`; OsmAnd-core |
| SPEC.data.* / SPEC.storage.* | `BinaryMapIndexReader`; `IndexConstants`; `DownloadIndexesThread` |
| SPEC.search.* | `SearchUICore`, `SearchPhrase`, `SearchResultMatcher` |
| SPEC.location.* | `OsmAndLocationProvider`, `NavigationService`, `GpsStatusListener` |
| SPEC.voice.* | `VoiceRouter`, `CommandPlayer`, `JsCommandBuilder`, `JsTtsCommandPlayer`, `JsMediaCommandPlayer` |
| SPEC.gpx.* | `GpxFile`, `Track`, `WptPt`; `SaveGpxHelper`, `TrackDisplayHelper` |
| SPEC.profile.* / SPEC.settings.* | `OsmandSettings`, `ApplicationMode`, `OsmandPreference<T>`, `BackupHelper`, `NetworkSettingsHelper` |
| SPEC.api.* | `OsmandAidlService` + `IOsmAndAidlInterface`, `IOsmAndAidlCallback` |
| SPEC.plugin.* | `OsmandPlugin` base + per-plugin subclasses |
| SPEC.obs.* | `PlatformUtil` logger; `osmand_log.txt`; crash sink |
| SPEC.net.* | `AndroidNetworkUtils`; HTTP clients in OsmAnd-shared |
| SPEC.account.* | `BackupHelper`, `BackupListeners` and `backup/ui/AuthorizeFragment`, `BackupAuthorizationFragment`, `LoginDialogType`, `LogoutBottomSheet`, `DeleteAccountFragment`; `OsmOAuthHelper`, `OsmOAuthAuthorizationAdapter` under `plugins/osmedit/oauth/`; credential storage via Android `EncryptedSharedPreferences` |
| SPEC.interact.* | `MultiTouchSupport`, `DoubleTapScaleDetector`, `RotatedTileBox` under `plus/views/`; `MapContextMenu`, `MapContextMenuFragment`; `plus/mapmarkers/` subsystem |
| SPEC.share.* | manifest intent-filters; `ShareMenu`, `ShareMenuFragment`, `ShareSheetReceiver`, `IntentHelper` (`plus/helpers/`) |
| SPEC.perm.* | `AndroidUtils.requestPermissions`; per-fragment / activity rationale UIs; manifest `<uses-permission>` entries |
| SPEC.notif.* | `NotificationHelper`, `NavigationNotification`, `DownloadNotification`, `GpxNotification`, `CarAppNotification` under `plus/notifications/` |
| SPEC.audio.* | `AudioFocusHelperImpl`, `JsMediaCommandPlayer`, `JsTtsCommandPlayer` under `plus/voice/`; `CommandPlayer` |
| SPEC.firstrun.* | `plus/firstusage/FirstUsageWizardFragment`, `FirstUsageWelcomeFragment`, `FirstUsageLocationBottomSheet`, `FirstUsageActionsBottomSheet`, `WizardType` |
| SPEC.units.* | `MetricsConstants`, `SpeedConstants`, `AngularConstants`, `OsmAndFormatter`, `LocationConvert`, `LocationFormatter`, `CoordinateInputFormats`, `ChooseCoordsFormatDialogFragment` |
| SPEC.storage.* (downloads) | `plus/download/` - `DownloadActivity`, `DownloadResources`, `DownloadIndexesThread`, `DownloadFileHelper`, `DownloadValidationManager`, `LocalIndexInfo`, `LocalIndexesFragment` |
| SPEC.net.live-wifi-only / large-download-warn / size-display | `LiveUpdatesHelper`, `LiveUpdatesAlarmReceiver`, `PerformLiveUpdateAsyncTask`, `LiveUpdatesSettingsBottomSheet`; `DownloadValidationManager` |
| SPEC.upgrade.* | `AppInitializer`, `AppVersionUpgradeOnInit`; version constants in `OsmandSettings` |
| SPEC.iap.* | `plus/inapp/` - `InAppPurchaseHelper`, `InAppPurchases`; `auto/screens/RequestPurchaseScreen` |
| SPEC.auto.* | `plus/auto/` - `NavigationCarAppService`, `NavigationScreen`, `SearchScreen`, `HistoryScreen.kt`, `RequestPermissionScreen`, `CarAppNotification` |
| SPEC.sensor.* | `plus/plugins/externalsensors/` - `ExternalSensorsPlugin`, `DeviceType`, BLE / ANT+ device implementations |
| SPEC.access.* (plugin) | `plus/plugins/accessibility/` - `AccessibilityPlugin`, `AccessibilityAssistant`, `AccessibilityActionsProvider`, `MapAccessibilityActions` |

---

## 15. Glossary

| Term | Definition |
|---|---|
| World | Everything outside the software-to-be that the software cares about or depends on - users, environment, external systems. |
| Machine | The software-to-be: the OsmAnd Android application, described at two levels - its external run-time shape (§6.1) and its internal module structure (§6.2-§6.4). |
| Shared phenomenon | An event, state, or value that both the World and the Machine can observe or cause; the only vocabulary specifications may use. |
| Indicative | Describes what is, has been, or will be true of the World regardless of the Machine. |
| Optative | Describes what we want to be true once the Machine is in place. |
| R, S, D, P, M | Jackson's reference-model symbols for Requirement, Specification, Domain knowledge, Program, Machine. |
| `D, S ⊢ R` | The central entailment: domain assumptions plus specifications must entail the requirement. |
| `.obf` | OsmAnd Binary Format - compact indexed binary map data compiled from OSM data by external tooling. |
| OsmAnd Live | Subscription service providing hourly map diff updates. |
| GPX | GPS Exchange Format - XML-based standard for tracks and waypoints. |
| `.osf` | OsmAnd Settings File - archive of settings + tracks + favourites + style overrides for backup/migration. |
| POI | Point of Interest - named geographic feature. |
| A\* | A-star pathfinding algorithm - used by the routing engine. |
| TTS | Text-to-Speech - OS service synthesising spoken instructions. |
| AIDL | Android Interface Definition Language - OsmAnd's third-party IPC interface. |
| Rendering style | XML configuration defining how map vector objects are visually displayed. |
| Profile | An `ApplicationMode` instance: named configuration set for a travel mode. |
| Plugin | A subclass of `OsmandPlugin` extending the Machine through defined hooks. |
| AGNSS / GNSS | Global Navigation Satellite System; covers GPS, GLONASS, Galileo, BeiDou. |
| Doze | Android OS power-saving mode that throttles background work when the device is stationary and the screen is off. |
| OAuth 2.0 | Open standard for delegated authorisation; OsmAnd uses it with PKCE to obtain access and refresh tokens from `osm.org` without ever handling the user's OSM password. |
| OsmAnd Cloud | OsmAnd-operated HTTP service for cross-device sync of settings, favourites, and GPX tracks. Has its own email-based account system, separate from OSM. |
| Access token | Short-lived credential proving the Machine is authorised to act as a given identity. |
| Refresh token | Longer-lived credential exchanged with the identity provider to obtain a new access token without user interaction. |
| Android Auto | Google's in-vehicle UI framework; apps expose constrained navigation, search, and list templates via `CarAppService`. |
| Audio focus | Android arbitration mechanism between apps competing for audio output: transient, transient-may-duck, or exclusive. |
| App Link | Verified HTTPS URL pattern an Android app may claim as a deep link, dispatched by the OS without an app chooser. |
| Foreground service | An Android service running with a persistent notification, exempt from many background-execution restrictions. |
| Notification channel | A user-configurable grouping for notifications introduced in Android 8.0; importance, sound, and DND override are per-channel. |
| Metered network | A network the OS reports as charging the user per byte - typically cellular or a hotspot. |
| Map marker | A user-placed pin on the map, persisting across sessions; distinct from a Favourite in that markers form an active to-visit list. |
| Wizard / Onboarding | The first-run multi-step flow shown to a new user to configure language, profile, permissions, and an initial map download. |
| In-app purchase | A one-time purchase of an item or feature unlock inside the app, mediated by the platform billing service. |
| Subscription | A recurring purchase that grants entitlements while active; renewed by the platform billing service. |
| BLE | Bluetooth Low Energy - the standard for connecting fitness sensors such as heart-rate straps, cadence sensors, and power meters. |
| ANT+ | A low-power wireless protocol used by some fitness sensors; supported on Android via the dsi.ant.plugins.antplus package. |

---

## 16. References

1. **Pamela Zave and Michael Jackson**, "Four Dark Corners of Requirements Engineering," *ACM Transactions on Software Engineering and Methodology*, 6:1–30, January 1997.
2. **Michael Jackson**, *Software Requirements & Specifications: A Lexicon of Practice, Principles and Prejudices*, ACM Press / Addison-Wesley, 1995.
3. **Michael Jackson**, *Problem Frames: Analysing and Structuring Software Development Problems*, Addison-Wesley, 2001.
4. **Axel van Lamsweerde**, *Requirements Engineering: From System Goals to UML Models to Software Specifications*, Wiley, 2009.
5. **Ian Sommerville**, *Software Engineering*, 10th edition, Pearson, 2015.
6. **OpenStreetMap Wiki**, "Map Features" and "API v0.6" - https://wiki.openstreetmap.org/
7. **GPX 1.1 schema**, Topografix - https://www.topografix.com/GPX/1/1/
8. **OsmAnd repository**, https://github.com/osmandapp/OsmAnd.
