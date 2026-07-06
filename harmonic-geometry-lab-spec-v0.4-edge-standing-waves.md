# Harmonic Geometry Lab — Edge Standing Waves Spec v0.4

**Status:** Draft for review
**Scope:** Adds true interior-edge standing waves — multiple nodes bowing *within* a single straight edge, not just at vertices. This is the piece explicitly descoped from v0.3 §5 ("Explicitly out of scope: true interior-edge standing waves"), revisited now that v0.3's vertex-level harmonics are built, verified against real Voxluminara audio, and confirmed working. Depends entirely on v0.3's infrastructure (`harmonicValue`, `CARRIER_HARMONIC_MODES`, `nearestHarmonicMode`, `currentCarrierFreqScalar`, `currentWobbleAmplitude`, `wireframeVertexMap`) — nothing in v0.3 is modified.

---

## 1. Problem Statement

v0.3 gives each vertex a displacement value from a spherical harmonic mode, which produces a real spatial pattern across the *whole solid* (some vertices push out, some pull in) and colors nodes/antinodes accordingly. What it can't show is the specific signature of classic cymatics: a single edge or membrane segment developing **multiple nodes along its own length** as frequency increases — the "more bands appear as the tone rises" effect that's the most recognizable visual signature of a Chladni figure.

Three things made this hard when it was scoped out of v0.3 (§5):
1. `EdgesGeometry` bakes static two-point line segments — no interior points to bow.
2. A vertex has a natural displacement axis (radial from center); a straight edge in 3D has infinitely many directions perpendicular to it — no natural choice.
3. Multiple nodes along one edge need their own node-count-vs-frequency formula, separate from the vertex-level mode.

This spec resolves all three by reusing infrastructure v0.3 already built, rather than inventing new machinery.

---

## 2. Edge Topology — Reused, Not Rebuilt

**Decision (locked):** No new edge detection is needed. `wireframeVertexMap` (built in `loadSolid()`, v0.3) already maps every point in `currentWireframe`'s `EdgesGeometry` position buffer back to a `uniqueVerts` index, and `EdgesGeometry`'s convention pairs consecutive points as one segment each. So the full edge list falls out for free:

```js
const edgeList = []; // [vertexIndexA, vertexIndexB] per edge
for (let i = 0; i < wireframeVertexMap.length; i += 2) {
    edgeList.push([wireframeVertexMap[i], wireframeVertexMap[i + 1]]);
}
```

Built once per `loadSolid()` call, right after `wireframeVertexMap` itself.

---

## 3. Per-Edge Perpendicular Direction

**Decision (locked):** For each edge, derive the bow direction by projecting the "radially outward from center" direction onto the plane perpendicular to the edge's own tangent — a Gram-Schmidt step, not an arbitrary choice:

```js
function computeEdgeBowDirection(posA, posB, center) {
    const tangent = posB.clone().sub(posA).normalize();
    const midpoint = posA.clone().add(posB).multiplyScalar(0.5);
    const radialGuess = midpoint.clone().sub(center).normalize();
    const bowDir = radialGuess.clone().sub(tangent.clone().multiplyScalar(radialGuess.dot(tangent)));
    if (bowDir.lengthSq() < 1e-6) bowDir.set(0, 1, 0).sub(tangent.clone().multiplyScalar(tangent.y)); // degenerate fallback: edge passes through center
    return bowDir.normalize();
}
```

Computed once per edge at `loadSolid()` time (rest positions don't change), cached alongside the edge's endpoint indices. This gives every edge a well-defined, stable "which way does it bow" direction with no per-frame recomputation.

---

## 4. Standing Wave Formula

**Decision (locked):** Node count along an edge is tied directly to the same `mode.l` already selected per-frame from `nearestHarmonicMode(currentFrequencyHz())` (v0.3 §2/§4) — no new frequency mapping, no new `[INVENTED]` table. `l` already increases with mapped frequency (396→1, 528→2, 639→2, 741→3, 852→3, 963→4), so reusing it directly gives the expected "more nodes at higher frequency" behavior for free.

```js
function edgeBowOffset(l, s, t, amp, freqScalar) {
    return amp * EDGE_AMP_SCALE * Math.sin(l * Math.PI * s) * Math.cos(t * freqScalar);
}
```

- `s` — normalized position along the edge, `0` to `1`, at each of the `EDGE_SUBDIVISIONS` interior sample points.
- `sin(l·π·s)` is exactly `0` at `s=0` and `s=1` for any integer `l` — endpoints are always pinned to the real vertex position with no discontinuity, no special-casing needed.
- `l=1`: one hump, no interior node (matches a simple standing wave with only the two endpoint nodes).
- `l=2`: crosses zero at `s=0.5` — one interior node, two lobes.
- `l=3`: two interior nodes, three lobes. `l=4`: three interior nodes. Directly reproduces the "more nodal structure at higher order" cymatics signature that neither v0.3's vertex-only approach nor a single-edge bow alone could show.
- `amp`/`freqScalar` reuse `currentWobbleAmplitude()`/`currentCarrierFreqScalar()` from v0.3 exactly as-is.
- `EDGE_AMP_SCALE` — new tunable constant, separate from `RESONANCE_AMP_SCALE`, since perpendicular bow may read stronger/weaker than radial wobble at the same amplitude. Reasoned starting estimate: `0.5`. Tune by feel once observed, same as `RESONANCE_RIPPLE_CONSTANT` was in v0.2.

**Point position:**

```js
const s = i / (EDGE_SUBDIVISIONS - 1);
const base = posA.clone().lerp(posB, s); // posA/posB = LIVE endpoint dot positions, not rest — see §6
const bow = edgeBowOffset(mode.l, s, t, amp, freqScalar);
point.set(base.x + bowDir.x * bow, base.y + bowDir.y * bow, base.z + bowDir.z * bow);
```

`EDGE_SUBDIVISIONS = 20` — a reasoned estimate (19 segments per edge is plenty to read `l=4`'s 3 interior nodes as a smooth curve; icosa/dodeca have 30 edges, so 30×20 = 600 points updated per frame, trivially cheap).

---

## 5. A Third Deformation Mode — Isolated, Not Composited

**Decision (locked):** Add `'edges'` as a third value for `deformationMode` (alongside v0.3's `'radial'`/`'harmonic'`), not an always-on overlay on top of `'harmonic'`.

**Rationale:** Compositing edge-bow with simultaneous per-vertex Ylm displacement (v0.3 Tier 3) means the cage is breathing at the mode-shape level *and* each edge is independently rippling — two effects stacked makes it hard to tell which structure you're looking at, and defeats the purpose of adding edges (readability). Isolating it — vertices stay at rest, *only* edges bow — gives a clean, classic-Chladni read: a rigid solid whose edges visibly develop more bands as frequency rises. This mirrors the same reasoning that made v0.3's Tier 1/Tier 3 an if/else rather than a blend.

**UI:** extend the existing mode-toggle button (v0.3 §4) to a three-state cycle — `RADIAL → HARMONIC → EDGES → RADIAL` — rather than adding a second button. Keeps the RESONANCE panel's control count unchanged.

**Open call, not locked — decide during build:** whether vertex dots stay visible (dimmed, static, for spatial reference) or hide entirely while in `'edges'` mode. Recommend starting with dimmed-and-static (reuse existing dot objects, just skip the wobble/color update for them in this mode) since it's zero extra code, and revisit if it looks cluttered once observed.

---

## 6. Rendering — New Group, Existing Wireframe Untouched

**Decision (locked):** Build a new `cymaticEdgeGroup` (a `THREE.Group` of one `THREE.Line` per edge — not a modification of `currentWireframe`'s `EdgesGeometry`). This is lower-risk than restructuring the existing wireframe: v0.3's `currentWireframe`/`currentMesh` position-and-color sync (already verified against real audio) is untouched, and the new group is simply shown/hidden by mode.

- Built once per `loadSolid()`, immediately after `edgeList`/bow directions (§2/§3): one `THREE.Line` per edge, geometry pre-sized to `EDGE_SUBDIVISIONS` points, added to `cymaticEdgeGroup`, added to `coreGroup`.
- **Visibility:** `cymaticEdgeGroup.visible = (deformationMode === 'edges')`; `currentWireframe.visible`/`currentMesh.visible` invert (hidden while in `'edges'` mode, shown otherwise) — the two representations are mutually exclusive, never both on screen.
- **Per-frame update (only runs when `deformationMode === 'edges'`):** for each edge, read its two endpoint dots' **live** positions (§4's `posA`/`posB` — these are `vertexDots[edgeList[k][0]].position` / `vertexDots[edgeList[k][1]].position`, whatever v0.3's own logic last set them to; since vertices don't move in `'edges'` mode per §5, this is just their rest position in practice, but reading it live rather than assuming rest keeps the code correct if that constraint ever changes), compute all `EDGE_SUBDIVISIONS` points via §4's formula, write into that edge's `Line` geometry position attribute, `needsUpdate = true`.
- **Coloring:** reuse the same HSL-lightness-by-magnitude approach as v0.3 §6, keyed to `|sin(l·π·s)|` at each point instead of `|normalizedHarmonicValue|` — same gold hue/saturation, dark at nodes (`s` where the sine crosses zero), bright at antinodes. Endpoints' color still comes from v0.3's existing per-vertex `applyHarmonicColors` (dots keep their v0.3 coloring even while dimmed/static per §5).

---

## 7. Cleanup & Edge Cases

1. **Switching solids:** rebuild `edgeList`, bow directions, and `cymaticEdgeGroup`'s `Line` objects from scratch in `loadSolid()`, same as every other per-solid cache.
2. **Switching deformation mode:** call a new `resetEdgeBow()` (parallels `resetHarmonicColors()`) that snaps every edge back to a straight line between its endpoints' current positions and hides `cymaticEdgeGroup`, before switching modes — avoids a visible snap, same discipline as v0.3 §7.2.
3. **Leaving RESONANCE mode entirely:** `resetAllVerticesToRest()` (already called on exit, v0.2) should also call `resetEdgeBow()` and ensure `cymaticEdgeGroup.visible = false`.
4. **CUSTOM points:** excluded from edge bowing entirely — they're user-placed and not part of the base polyhedron's edge topology, same exclusion already established for v0.3's wireframe/mesh sync.

---

## 8. Out of Scope for v0.4

- Compositing edge-bow with simultaneous vertex Ylm displacement (§5) — possible future v0.5 toggle, not built here.
- True rotation/circulation via curl (`∇×A`) — already ruled out permanently in the v0.2.1 addendum; a scalar standing wave, however constructed, cannot produce real vortex motion.
- A dedicated node-count-vs-frequency table independent of vertex `l` — deliberately not built; reusing `mode.l` directly is simpler and keeps one `[INVENTED]` mapping instead of two.
- Per-face nodal surface overlays (translucent Chladni-style dark regions across whole faces, not just along edges) — a further future idea, not required to hit this spec's goal.

---

## 9. Suggested Build Order

1. `edgeList` construction from `wireframeVertexMap` (§2) — pure data derivation, verify against a known solid's edge count before touching rendering (e.g. dodeca should yield exactly 30 edges).
2. `computeEdgeBowDirection()` (§3) — pure function, testable standalone same as `harmonicValue` was in v0.3.
3. `edgeBowOffset()` (§4) — pure function; verify `l=1..4` produce the expected zero-crossing counts before wiring into the render loop.
4. `cymaticEdgeGroup` construction + visibility toggle (§5/§6) — get the three-state mode cycle and show/hide working with edges *flat* (bow forced to 0) before adding real motion, so any topology/direction bugs are caught before the standing-wave math is layered on.
5. Wire `edgeBowOffset` into the per-frame update (§6) — real motion.
6. Coloring (§6) and cleanup/reset (§7).
