# Harmonic Geometry Lab — Cymatic Resonance Spec v0.3

**Status:** Draft for review
**Scope:** Replaces Phase 2's uniform-pulse vertex wobble with a real spatially-varying deformation (spherical harmonics), makes the wireframe/mesh actually follow that deformation instead of standing rigid, and adds node/antinode color readability. Depends on the audio pipeline and RESONANCE mode shipped in v0.2/v0.2.1 (already built). Does **not** cover the Voxluminara naming layer — that remains a separate, unscheduled future item, not part of this spec.

---

## 1. Problem Statement

Two bugs, confirmed against the running code, block the actual goal (seeing *which* frequency is playing by looking at the solid):

1. **No spatial variation.** `updateResonanceDeformation()` (index.html:1146) computes `wobble = amp * cos(k * restRadius - t * freqScalar)`. Every vertex of a Platonic solid sits on the same circumscribed sphere, so `restRadius` is identical for all of them — every vertex gets the same value every frame. The solid pulses like a heartbeat; there's no pattern, no nodes, nothing that differs by *where* a vertex sits.
2. **The wrong input drives it.** The deformation's temporal rate comes from `beatHz` (index.html:1135), and Voxluminara hardcodes a fixed +4 Hz binaural offset for every Solfeggio tone (`voxluminara.html:1471-1474`). So `beatHz` barely changes across the whole 396–963 Hz set — the vertex wobble would pulse at nearly the same speed for every tone, regardless of which one is actually playing. `carrierHz` (the value that *does* vary meaningfully per tone) currently only drives ambient bob/glow (`driverValue`, index.html:1741+), never the vertex deformation itself.

A third, structural issue found while scoping this: only the small vertex-dot markers (`SphereGeometry(0.028,...)`, index.html:377) move at all. The wireframe cage (`currentWireframe`) and the solid mesh (`currentMesh`) never have their geometry touched by `updateResonanceDeformation` — they stay rigid in their rest shape. What's been "breathing" in Phase 2 is a handful of tiny dots, not the visually dominant cage.

This spec fixes all three, without discarding Phase 2 — the existing radial wobble becomes one of two selectable modes, not a replacement.

---

## 2. Frequency → Mode Mapping

**Decision (locked):** Deformation is driven by `carrierHz`, not `beatHz`. Carrier is what actually varies across the Solfeggio set; beat is a near-constant binaural offset and should stay confined to whatever it already does (ambient bob/glow), not be asked to differentiate tones it can't differentiate.

**Tagging discipline (locked, carried over from the Voxluminara project's own `[PHYSICS]`/`[INVENTED]` convention):**

| Element | Tag | Why |
|---|---|---|
| Spherical harmonics as the deformation basis | `[PHYSICS]` | Y_l^m are the real eigenfunctions of vibration on a sphere — legitimate math, correctly applied. |
| A specific Hz → (l, m) assignment | `[INVENTED]` | There is no physical law tying a 528 Hz acoustic tone to a quadrupole mode on a dodecahedron. This is an artistic mapping, not a derived one, and should never be presented as measured or discovered. |

**v1 mapping table** (invented, adjust freely — this is a starting point, not a constraint):

| Freq (Hz) | Mode (l, m) | Shape |
|---|---|---|
| 396 | (1, 0) | dipole — stretches along one axis |
| 417 | (1, 0) | dipole (shares 396's mode; differs in rate only) |
| 528 | (2, 0) | quadrupole — peanut / figure-8 |
| 639 | (2, 2) | four-lobe |
| 741 | (3, 0) | three-band |
| 852 | (3, 2) | star pattern |
| 963 | (4, 4) | highest-order mode in this table |

Modes may be reused across frequencies (as above for 396/417) — the table doesn't need to be injective, it needs to be legible.

---

## 3. Displacement Formula — Implementation Note (deviates from the naive version)

**Decision (locked):** Compute each mode as a **closed-form Cartesian polynomial** of the vertex's unit-sphere position `(x, y, z)`, not via `acos`/`atan2` into `(θ, φ)` and then a general Legendre-polynomial evaluator.

An earlier version of this idea proposed `theta = acos(op.y / op.length())`, `phi = atan2(op.z, op.x)`, then combined them as `cos(k*(theta+phi) - t*freqScalar)`. That has two problems this spec avoids: (a) `θ`/`φ` have a coordinate singularity at the poles (`φ` is undefined when a vertex sits on the axis), which risks jitter for pole-adjacent vertices; (b) adding `θ + φ` mixes two angles with different ranges with no basis in the actual Y_l^m functions — it doesn't reproduce real spherical-harmonic mode shapes, just an arbitrary diagonal ripple.

The modes actually needed for the v1 table (§2) have simple closed forms in normalized `(x, y, z)` — no trig calls, no singularities:

```js
// v = normalized vertex position (unit vector from solid's center)
function harmonicValue(l, m, v) {
    const { x, y, z } = v;
    if (l === 1 && m === 0) return z;
    if (l === 2 && m === 0) return 1.5 * z * z - 0.5;
    if (l === 2 && m === 2) return x * x - y * y;
    if (l === 3 && m === 0) return 2.5 * z * z * z - 1.5 * z;
    if (l === 3 && m === 2) return z * (x * x - y * y);
    if (l === 4 && m === 4) return x*x*x*x - 6*x*x*y*y + y*y*y*y;
    return 0; // unmapped (l,m) — extend table as needed
}
```

Each of these is a standard real solid harmonic (harmonic polynomial), evaluated directly from Cartesian coordinates — robust at every point on the sphere, including the poles.

**Full displacement (replaces §4.1 of v0.2):**

```js
const h = harmonicValue(mode.l, mode.m, normalizedVertexDirection);
const wobble = amp * h * Math.cos(t * freqScalar); // freqScalar now derived from carrierHz, not beatHz
position = originalPosition + normal * wobble;
```

`h` supplies the spatial pattern (fixed per vertex, cached at solid-load time same as `restRadius` is today); the `cos(t * freqScalar)` term supplies the temporal breathing. Vertices where `h ≈ 0` stay still (nodes); vertices where `|h|` is large swing widely (antinodes) — this is the actual nodal structure Phase 2 never had.

---

## 4. Mode Toggle — Tier 1 vs. Tier 3, if/else

**Decision (locked):** Keep Phase 2's existing radial-pulse formula intact as one mode; add the harmonic formula (§3) as a second mode; switch with a flag, not a rewrite.

```js
function updateResonanceDeformation(t) {
    const applyWobble = (dot) => {
        const wobble = (deformationMode === 'harmonic')
            ? harmonicWobble(dot, t)   // §3
            : radialWobble(dot, t);   // existing v0.2 formula, unchanged
        dot.position.set(/* op + normal * wobble, as today */);
    };
    // ...unchanged iteration over dots
}
```

Both modes share the existing cached `originalPosition`/`normal` per vertex; `harmonicValue(l, m, ...)` is the only new piece cached at load time (alongside, not instead of, `restRadius` — Tier 1 still needs it). No existing Phase 2 code is removed.

---

## 5. Wireframe/Mesh Must Follow the Displacement (new — this was silently missing in Phase 2)

**Decision (locked):** `currentWireframe` and `currentMesh` geometry must track the same displaced positions the dots already use, instead of remaining rigid.

- At `loadSolid()` time, build a one-time lookup from each `EdgesGeometry` position-array entry to its corresponding vertex/dot (same proximity-matching approach already used for `uniqueVerts` dedup, index.html:412).
- Each frame, after computing dot displacement, write the same displaced position into the wireframe's `position` BufferAttribute at the looked-up indices and set `.needsUpdate = true`.
- Apply the same treatment to `currentMesh`'s geometry so the (very low-opacity) solid faces deform in sync — cheap, since it reuses the identical per-vertex value already computed for the dots, no second computation pass.

This is the single highest-leverage visual change in this spec: the actual cage most people are looking at starts reflecting the deformation, instead of only the small corner markers.

**Explicitly out of scope:** true interior-edge standing waves (multiple nodes bowing *within* one straight edge). That requires subdividing each edge into N interior points, inventing a per-edge perpendicular offset direction (ambiguous in 3D, unlike a vertex's radial normal), and a separate node-count-vs-frequency formula for position-along-edge. None of that is needed to hit this spec's goal — §5's simpler "follow the displaced endpoints" plus §6's coloring already deliver a structurally responsive, per-frequency-distinct shape. Full interior-edge curvature is a candidate for a future v0.4, not required here.

---

## 6. Node/Antinode Color Readability

**Decision (locked):** Add a `color` BufferAttribute (same length as `position`) to the wireframe and dot materials (`vertexColors: true`), driven by `|h|` (the harmonic value from §3) at each point — muted/dark near `h ≈ 0` (nodes), bright/hot at high `|h|` (antinodes).

This delivers the actual "you know which frequency you're at by which vertices go quiet" signal directly from data already being computed in §3 — no new geometry, no separate nodal-surface mesh, just a color write alongside the position write in the same per-frame pass. Only applies meaningfully in `deformationMode === 'harmonic'`; in radial mode, `h` is undefined and color should fall back to the existing static gold.

---

## 7. Cleanup & Edge Cases

1. **Switching solids:** re-cache `harmonicValue` per vertex (alongside existing `restRadius`/`normal` caching) for the new solid's vertex set — same pattern as today's solid-switch re-cache.
2. **Switching deformation mode mid-RESONANCE:** call `resetAllVerticesToRest()` (already exists) before switching, so there's no visible snap from leftover displacement computed under the other mode's formula.
3. **CUSTOM points:** as noted in v0.2 §6.3 and still unresolved — custom points don't have a natural `(l,m)`-mappable position the way solid vertices do (their position is user-placed, not tied to the shape's symmetry). Flag as a build-time judgment call: either approximate via direction-from-centroid (already the existing approach for their `normal`), or exclude them from `harmonic` mode and let them only respond in `radial` mode.
4. **Mode/frequency mapping table (§2) is data, not code logic** — store it as a lookup object so extending it (new frequency, new (l,m) pair) doesn't require touching `updateResonanceDeformation`.

---

## 8. Out of Scope for v0.3

- True interior-edge standing waves / edge subdivision (§5) — future v0.4 candidate.
- Voxluminara naming layer — unrelated, separate, unscheduled.
- Any physical claim that a specific Hz literally produces a specific (l,m) mode in nature — the mapping is `[INVENTED]` per §2 and should stay labeled as such in any UI copy.
- Extending `harmonicValue` beyond the six modes needed for the v1 table — add more (l,m) closed forms only when a new frequency actually needs one.

---

## 9. Suggested Build Order

1. `harmonicValue(l, m, v)` (§3) — pure function, testable in isolation before touching the render loop.
2. Frequency → mode lookup table (§2) as data.
3. Mode toggle + `harmonicWobble` path (§4) — verify against existing `radial` path with a UI switch, confirm Tier 1 is unaffected.
4. Wireframe/mesh follow-the-displacement (§5) — the highest-visual-impact step; do this before color so the shape-change itself can be judged on its own first.
5. Node/antinode coloring (§6).
6. Edge cases (§7): solid-switch re-cache, mode-switch reset, CUSTOM point decision.
