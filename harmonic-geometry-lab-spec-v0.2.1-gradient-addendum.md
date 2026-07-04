# Harmonic Geometry Lab — Gradient-Driven Deformation Addendum v0.2.1

**Status:** Built and verified
**Scope:** Revises the RESONANCE deformation formula from v0.2 (Phase 2, §4) to be gradient-derived from a scalar potential field, rather than a cached normal scaled by an independently-computed wave. Touches only `cacheRestState()` and `updateResonanceDeformation()` in `index.html`. Does not change the audio pipeline (§2/§3 of v0.2), mode lifecycle (§5), or the specimen log. Not a rewrite — a targeted swap of how displacement direction and magnitude are derived.

---

## 0. Bug Fix (found during verification) — Audio Analysis Flicker

While verifying this addendum against a real recorded session (`voxluminara-session-2026-07-04T00-22-09.webm`), playback showed a real vertex/UI flicker unrelated to the gradient math above. Root cause: `readDominantFrequency()` (v0.2, §2.2) picks the single loudest FFT bin every frame with no smoothing on *which* bin wins. In the real file, a stretch around the 15s mark has two adjacent bins (~516 Hz and ~539 Hz) at nearly equal energy, so the pick alternated frame to frame — `carrierHz` snapping between them, which fed `driverValue` and made the pre-existing dial-driven bob/glow/emissive effects visibly stutter. (`beatHz` happened to read a stable `0` in this segment since L/R flipped in lockstep, so the RESONANCE wobble's own speed wasn't the visible symptom here — the older dial-driven effects were.)

**Fix:** exponential smoothing on the derived `carrierHz`/`beatHz`/`audioAmplitude` in `updateAudioAnalysis()`, reusing the existing `let` variables as the "previous value" (no new state):

```js
const AUDIO_SMOOTHING = 0.15;
carrierHz += (newCarrier - carrierHz) * AUDIO_SMOOTHING;
beatHz += (newBeat - beatHz) * AUDIO_SMOOTHING;
audioAmplitude += (newAmp - audioAmplitude) * AUDIO_SMOOTHING;
```

Verified against the same real file/timestamp: before the fix, `carrierHz` alternated 515.63 ↔ 539.06 every couple of frames; after, it converges smoothly (264 → 302 → 334 → ... → 490) with no oscillation. Pre-existing v0.2 bug, unrelated to the gradient work in this doc — confirmed separately that the gradient formula is deterministic given stable input.

---

## 1. Problem Statement

v0.2 computes displacement as two independently-chosen pieces multiplied together:

```js
position = originalPosition + normal * amplitude * sin(t * beatFreqScalar + phaseOffset)
```

`normal` (displacement direction) is cached once from `(restPosition − referenceCenter).normalize()` — a full 3D, spherical direction. `phaseOffset` (the wave's spatial term) is cached separately from `√(x² + z²) * RESONANCE_RIPPLE_CONSTANT` — a **cylindrical** distance from the Y-axis, ignoring height entirely.

These are two different radii describing the same vertex. That was harmless as independent multiplied terms, but it means the deformation isn't actually derived from one consistent spatial model — it's direction-from-one-r and phase-from-another-r, hand-assembled. This addendum unifies both under a single scalar field φ and takes its gradient, so direction and magnitude come from the same differentiated source.

---

## 2. The Field

**Decision (locked):**

```
φ(vertex, t) = amplitude · sin(k·r − ω·t)
```

where:
- `r = |vertex − referenceCenter|` — full 3D (spherical) distance from center, **not** cylindrical distance from an axis.
- `k = RESONANCE_RIPPLE_CONSTANT` (0.6, unchanged from v0.2)
- `ω = freqScalar`, i.e. the existing `currentBeatFreqScalar()` (unchanged from v0.2)
- `amplitude` = the existing `currentWobbleAmplitude()` (unchanged from v0.2)

Taking the gradient (chain rule, since φ depends on position only through `r`):

```
∇φ = amplitude · k · cos(k·r − ω·t) · r̂
```

where `r̂ = (vertex − referenceCenter) / r` — the unit radial direction, which is exactly what `normal` already is today. Displacement direction is therefore **unchanged** from v0.2; only the magnitude/phase profile changes (`cos` instead of `sin`, driven by the same `r` used for direction instead of a separately-cached cylindrical value).

### 2.1 Why spherical `r`, not cylindrical

Cylindrical `r̂ = (x, 0, z)/√(x²+z²)` is undefined/unstable for any vertex near the Y-axis (top/bottom of a solid) — division by a value approaching zero. Spherical `r` has no such singularity for any real vertex position relative to `referenceCenter`, and its `r̂` is identical to the existing cached `normal`. This isn't a style preference — cylindrical `r` would break at the poles.

### 2.2 The constant fold

Naively, `∇φ`'s `amplitude · k` factor would shrink peak displacement by ~40% relative to v0.2 (since `k = 0.6` and v0.2 had no such multiplier). Both `amplitude` and `k` are art-directed constants, not measured physical quantities — there is no real invariant preserved by keeping them un-folded. Substituting `amplitude' = amplitude / k`:

```
∇φ = amplitude · cos(k·r − ω·t) · r̂
```

The `k` cancels exactly. **Net result: peak displacement magnitude is identical to v0.2's tuning** (`currentWobbleAmplitude()` unchanged) — only `sin(... + cylindricalPhase)` becomes `cos(k·sphericalR − ...)`. No new constants, no re-tuning pass.

---

## 3. Code-Level Change

**`cacheRestState(dot, referenceCenter)`** — replace the cylindrical `phaseOffset` calc:

```js
// v0.2 (removed):
dot.userData.phaseOffset = Math.sqrt(rest.x * rest.x + rest.z * rest.z) * RESONANCE_RIPPLE_CONSTANT;

// v0.2.1:
dot.userData.restRadius = rest.clone().sub(referenceCenter).length();
```

`normal` caching is unchanged — it already equals the `r̂` this addendum needs.

**`updateResonanceDeformation(t)`** — replace the wave term:

```js
// v0.2 (removed):
const wobble = amp * Math.sin(t * freqScalar + dot.userData.phaseOffset);

// v0.2.1:
const wobble = amp * Math.cos(RESONANCE_RIPPLE_CONSTANT * dot.userData.restRadius - t * freqScalar);
```

Everything downstream (`op.x + normal.x * wobble`, etc.) is untouched.

`RESONANCE_AMP_SCALE`, `RESONANCE_BASE_AMPLITUDE`, `RESONANCE_BASE_BEAT_HZ`, and `RESONANCE_RIPPLE_CONSTANT` all keep their current values — none need re-tuning per §2.2.

---

## 4. Edge Cases (carried over, unaffected)

- **Custom points:** `referenceCenter` is already passed per-caller (`NATIVE_VERTEX_CENTER` for native vertices, `constructionPlaneFill.position` for custom points) — `restRadius` computes correctly for both with no new logic.
- **Drag-end re-cache:** `refreshMeasurementsForNode()` already calls `cacheRestState()` on drag-end, which will now cache `restRadius` fresh from the new position — same lifecycle as v0.2's `phaseOffset` re-cache, just a different field name.
- **Solid switching:** re-caching on solid switch (v0.2 §6.2) is unaffected — same call sites, same trigger.

---

## 5. Explicitly Out of Scope

- **Angular/rotational term (`m·θ` in φ, "Option C"):** would make `∇φ` gain a tangential component (ripples spiral instead of breathe), diverging from `normal` direction for the first time. Bigger change, not needed for this addendum. Candidate for Phase 3 — possibly tying `m` to each solid's natural symmetry order (tetra vs. octa vs. icosa having different mode counts).
- **True rotational/vortex motion via curl (`∇×A`):** mathematically distinct from anything gradient-based — `∇×(∇φ) ≡ 0` always, so no scalar field (however θ-dependent) can produce genuine circulation. If ever wanted, requires a separately-defined vector potential `A`, added on top of `∇φ` (Helmholtz decomposition: irrotational + solenoidal parts). Parked alongside the Option C note above, not needed now.
- **Multi-source interference/superposition:** current single `carrierHz`/`beatHz` driver stays single-source. Layering multiple simultaneous frequency drivers onto one mesh is a separate, larger change to the audio pipeline (v0.2 §2), not touched here.

---

## 6. Build Order

1. Add `restRadius` caching in `cacheRestState()`, remove `phaseOffset`.
2. Swap the wave term in `updateResonanceDeformation()` to the `cos(k·r − ωt)` form.
3. Manual test: confirm RESONANCE mode wobble looks the same or better with no visible amplitude regression (per §2.2, peak magnitude should match v0.2 exactly).
4. Confirm custom points + drag-end re-cache still behave correctly (§4).
