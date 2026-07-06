# Activity Diagrams — Reading Guide

UML 2.x **activity diagrams** for OsmAnd. They model *behaviour over time* — the
sequence of actions, decisions, and concurrency in a user-facing flow — as opposed to
the structural `logical_*` diagrams one level up (which show *what the parts are*).

The flows were verified against the OsmAnd source (`OsmAnd/` checkout): the behaviour
shown here matches `SearchUICore`, `QuickSearchDialogFragment`, `RoutingHelper`,
`RouteRecalculationHelper`, `RouteProvider`, `OsmAndLocationProvider`, and `VoiceRouter`.

## The flow modelled here

**"Search a destination, calculate a route, and navigate to it, with asynchronous
calculation, missing-map handling, and automatic re-routing when the user deviates."**

Because the full flow is long, it is split into **three phases** that run in sequence.
Read them in order — each ends by handing off to the next via a `proceed to Phase N`
action.

| # | File | Covers | Starts from | Ends at |
|---|------|--------|-------------|---------|
| 1 | `activity_route_1_search.png` | Enter query → find the destination | User opens navigation | destination chosen → Phase 2 |
| 2 | `activity_route_2_calc.png` | Calculate route with the profile's engine | destination chosen | route ready, user taps Start → Phase 3 |
| 3 | `activity_route_3_guidance.png` | Turn-by-turn guidance + auto re-routing | navigation started | arrival / navigation stopped |

### Phase 1 — Search (`activity_route_1_search.png`)
Offline search **always runs first**, over whatever `.obf` indexes are installed —
there is no up-front "region downloaded?" gate; missing maps simply yield few or no
results. When results are empty, the app **offers** "search online address"; online
geocoding is **user-initiated**, and connectivity is checked only at that point. Ends
when a destination is selected (or at a "no results" / abandoned-search final node).

### Phase 2 — Route calculation (`activity_route_2_calc.png`)
The routing engine is **fixed by the active profile** (`RouteService`: OSMAND /
BROUTER / ONLINE / STRAIGHT / DIRECT_TO), chosen by the user beforehand — there is
**no automatic local ↔ online fallback**. Calculation is **asynchronous (fork)**: the
user keeps interacting and may cancel. **Missing maps are detected in the calculation
result**, after the attempt — only then does the app suggest downloading the missing
regions. The download is also asynchronous, with exhaustive outcomes (complete /
storage full / interrupted / cancelled); after a successful download the calculation
can be re-run (loop). Ends when the route is displayed and the user taps Start.

### Phase 3 — Guidance & re-routing (`activity_route_3_guidance.png`)
The guidance loop, evaluated on each GPS tick: **GPS lost** (wait for the next fix —
arrival is not evaluated without one), **on-route** (update map + announce maneuver),
and **off-route** (automatic re-route). Re-routing is **asynchronous** — guidance
continues on the old route while the recalculation runs — and reuses the **same
profile engine** as Phase 2. If recalculation fails (e.g. ONLINE engine without
internet), the old route is kept and a **retry is scheduled with backoff
(3 s → max 120 s)** — no dead end. On arrival the app announces it and finishes the
route automatically.

## How to read the notation

- **Swimlanes (vertical partitions)** show *who performs each action*. Every action sits in
  exactly one lane:
  - **User** — taps, choices, cancellations
  - **App UI** — the app's foreground logic and screens
  - **Device (engine + storage)** — the local routing/search engine reading the on-device
    `.obf` map files
  - **Network** — online services (geocoding, online routing, map download). Appears in
    Phases 1–2 only; in Phase 3 the re-route reuses the profile engine, so any ONLINE
    request is noted inside the recompute action rather than given its own lane.
- **● (filled circle)** = initial node (one per diagram). **◉ (encircled dot)** =
  activity final; each error/cancel path terminates in its own activity final.
- **Rounded boxes** = actions (verb phrases).
- **Diamonds** = decision/merge nodes; every outgoing edge is labelled with a guard in
  `[brackets]`, and the guards on each decision are mutually exclusive and exhaustive
  (multi-way outcomes such as `calculation result?` and `download result?` enumerate
  the error and cancel cases, not just the happy path).
- **Thick horizontal bars** = fork/join — the actions between them run **concurrently**
  (async route calculation, async map download, async re-route). The "keep using app
  (may cancel)" branches carry notes stating that cancelling **interrupts** the
  concurrent work — that is what produces the `[cancelled by user]` outcomes.
- **`proceed to Phase N`** actions are off-page connectors linking the three diagrams.
