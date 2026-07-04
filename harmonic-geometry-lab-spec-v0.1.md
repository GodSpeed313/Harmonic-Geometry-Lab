# Harmonic Geometry Lab — Data & Interaction Spec v0.1

**Status:** Draft for review
**Scope:** Fixes and additions to `index.html` (NEXUS / Harmonic Geometry Lab). Does not yet cover the audio-reactive bridge to Hemispheric Sync or the Voxluminara naming layer — those are Phase 2 and Phase 3, noted at the bottom.

---

## 1. Problem Statement

The app currently *displays* geometry well but doesn't *capture* it. Angles are computed transiently, shown once in the footer, pushed into an 8-entry in-memory log, and lost on reload. There's no way to pull numeric data out of the app for use elsewhere (Voxluminara naming, the ebook, video content, further math). CUSTOM mode and SKETCH mode also don't yet do what their names imply. This spec closes those gaps.

---

## 2. Angle & Measurement Data Model

### 2.1 The core decision

There are three different "angles" a connection between two vertices could mean, and the app currently only computes one of them — possibly not the useful one:

| Type | What it measures | Current status |
|---|---|---|
| **Origin-relative angle** | Angle between vectors from scene center (0,0,0) to each vertex | ✅ Currently computed (`p1.angleTo(p2)`) |
| **Edge angle (direction)** | The connecting line's orientation in 3D space (e.g. as spherical coordinates, or angle from vertical/horizontal) | ❌ Not computed |
| **Interior/vertex angle** | The angle formed *at* a vertex between two edges meeting there (needs 3 points, not 2) | ❌ Not computed |

**Decision (locked):** Log all three types in v0.1, including interior/vertex angle. Origin-relative angle is cheap and already works — keep it, but relabel it clearly in the UI/export so it isn't confused with a structural angle. Edge angle (direction of the connecting line in space) is the most broadly useful for your stated goal ("numbers I can reuse for other things") since it's independent of where the object sits in the scene. Interior angle is the most geometrically "sacred geometry"-flavored one (this is what people usually mean by "the angles of a dodecahedron") — it requires detecting when a vertex has two or more connections and computing the angle between each pair of edges meeting there (standard vector dot-product-based angle-between-two-edges math, using the shared vertex as the origin for that specific calculation — not the scene origin). More work than the other two, but confirmed in-scope for v0.1.

### 2.2 What gets recorded per measurement

Every time two nodes are connected (CONNECT, CUSTOM, or SKETCH mode), record a structured entry — not just a display string:

```json
{
  "id": "m-0007",
  "timestamp": "2026-07-02T14:32:10Z",
  "solid": "dodeca",
  "nodeA": { "id": "vtx-3", "position": [0.618, 1.0, 0.0] },
  "nodeB": { "id": "vtx-11", "position": [-0.618, 1.0, 0.382] },
  "distance": 1.236,
  "originRelativeAngleDeg": 41.8,
  "edgeDirectionDeg": { "azimuth": 132.4, "elevation": 8.1 },
  "mode": "CONNECT",
  "frequencyAtCapture": 432.0
}
```

Key changes from current behavior:
- Raw vertex coordinates are stored, not just the derived scalar — this is what lets you reuse the numbers for anything else later, even math the app itself never anticipated.
- `frequencyAtCapture` future-proofs this for Phase 2, so a specimen logged during a session carries the tone that shaped it.
- No cap at 8 entries. History persists for the life of the current solid session; export clears or archives it (your call, see 2.3).

### 2.3 Persistence

Currently: nothing survives a reload. Fix using the same `window.storage` pattern already working correctly in Hemispheric Sync's practice log.

- Key structure: `specimens:<solid>:<sessionId>` for grouped storage, or a single `specimens:log` key holding the full array if volume stays low (simplest to start).
- Personal data only (`shared: false`) — no reason for this to be shared/public storage.
- A "Clear Log" action already exists for connections visually (`clearConnectionsBtn`) — extend it to also clear the persisted measurement log, or better: separate "clear the 3D scene" from "clear my saved data" so you don't accidentally nuke logged numbers while just resetting the view.

### 2.4 Export

Add an "Export Specimen" action to the Measurement Lab panel:
- Downloads current measurement history as a `.json` file (structured, per 2.2) and optionally a flattened `.csv` for spreadsheet use (distance, angle columns, one row per connection).
- Filename pattern: `specimen_<solid>_<timestamp>.json`

---

## 3. CUSTOM Mode — Redefinition

**Current state:** functionally identical to default CONNECT mode. No differentiation.

**Proposed behavior:** CUSTOM mode starts from an *empty* point field rather than a preset Platonic solid:
- Entering CUSTOM mode hides (or fades) the active solid's faces/wireframe, leaving just a way to place new points in 3D space (simplest v0.1 approach: place points on a fixed reference plane or sphere shell via click, rather than full free 3D placement, which needs a more involved gizmo).
- Points placed in CUSTOM mode become vertex dots identical in behavior to a solid's native vertices — same hover/select/connect logic already built.
- Connecting points in CUSTOM mode logs measurements exactly like CONNECT mode (reuses 2.2 data model).
- This directly serves your original vision: build a shape that isn't one of the six presets, from scratch, and still get real angle/distance data out of it.

**Decision (locked):** Build in two stages rather than choosing one approach outright.

- **Stage 1 — Plane placement.** Click-to-place on a construction plane, reusing the existing raycasting/vertex-dot pipeline. This is the real, functional v0.1 of CUSTOM mode — not a placeholder, a working custom-shape builder.
- **Stage 2 — Free-space lift.** Add `THREE.TransformControls` (a draggable X/Y/Z gizmo) so a placed point can be dragged off the plane into true free 3D space. Requires: importing the addon, gating it against `OrbitControls` so camera-orbit drags and point drags don't conflict, attach/detach logic per selected point, and re-triggering the §2.2 measurement recompute whenever a point moves (since raw coordinates are stored, not just a scalar angle at creation time — a moved point needs its dependent measurements refreshed, not left stale).

Stage 2 is meaningfully more build time than Stage 1 — expect it to be the single largest item in this spec, more debugging-the-interaction-feel than math. Sequencing it as a direct extension of Stage 1 (not a rebuild) means you get a working CUSTOM mode fast without foreclosing on full free placement later.

---

## 4. SKETCH Mode — Integration

**Current state:** draws lines in an isolated `sketchGroup`, invisible to the stencil canvas, not logged anywhere.

**Fix:** SKETCH edges should feed the same measurement pipeline as CONNECT/CUSTOM (2.2), and `drawStencilView()` should render `sketchLines` alongside `connectionLines` (different color already exists — purple vs. gold — so the visual distinction is free). Once wired in, SKETCH stops being a dead end and becomes a genuine "provisional/exploratory connections" mode that still produces real data.

---

## 5. Bug Fixes (independent of the above)

1. `setStencilProjection()` — FACES/EDGES/VERTICES readout hardcodes `12/30/20` regardless of active solid. Should pull from the same `labels`/`readouts` map already defined for the catalog click handler.
2. Duplicate `drawStencilViews(); drawStencilViews();` call in the default connect branch — remove one.
3. "DIHEDRAL ANGLE" field in the Face Detail canvas is a static string (`"Variable"`) — either compute it for real (ties into §2.1's interior-angle decision) or remove the field until it's real, so nothing on screen implies data that isn't there.

---

## 6. Out of Scope for v0.1 (Phase 2 / Phase 3, for later specs)

- **Phase 2 — Audio-reactive geometry:** binaural carrier/beat values from Hemispheric Sync driving live vertex deformation on the active solid, with the deterministic tone→shape mapping discussed separately.
- **Phase 3 — Voxluminara naming layer:** auto-naming exported specimens using existing root vocabulary based on frequency range or shape properties.

Both phases depend on the data model in §2 existing first — this is why it's the right thing to spec and build before either of those.

---

## 7. Suggested Build Order

1. Bug fixes (§5) — fast, no design decisions needed.
2. Data model + persistence + export, including interior/vertex angle (§2) — the highest-leverage piece; unlocks "write the numbers down."
3. SKETCH integration (§4) — small, reuses §2's pipeline directly.
4. CUSTOM mode Stage 1 — plane placement (§3) — ships a working custom-shape builder.
5. CUSTOM mode Stage 2 — free-space gizmo lift (§3) — largest single item; sequence last since it depends on §2's data model being solid first (moved points must re-trigger measurement recompute).
