# Harmonic Geometry Lab — Audio-Reactive Geometry Spec v0.2 (Phase 2)

**Status:** Draft for review
**Scope:** Adds real-time audio-reactive vertex deformation to `index.html`, bridging the app to an external Hemispheric Sync (binaural beat) source via file upload or microphone. Depends on the data model shipped in v0.1 (§2 of that spec) and the CUSTOM mode gizmo (§3 Stage 2), both already built. Does not cover Phase 3 (Voxluminara naming), noted at the bottom.

---

## 1. Problem Statement

The app currently has one driver of "resonance" visuals: a manual dial (`frequency`, 0–1) that the user drags by hand. It already drives group-level bob, wireframe opacity pulse, and emissive glow (see the `animate()` loop). What it can't do is react to *actual* binaural audio — a Hemi-Sync-style file or live mic input — and it has no per-vertex deformation at all; every existing effect moves the whole solid as a rigid body, not its individual vertices.

This spec adds: (1) a real audio input pipeline that extracts carrier and beat frequency from binaural audio, and (2) per-vertex organic wobble driven by that data, layered on top of the existing dial-driven effects rather than replacing them.

---

## 2. Audio Input Pipeline

### 2.1 Input sources

**Decision (locked):** Support both file upload and microphone input in v1 — they share the same downstream `AnalyserNode` pipeline, only the source node differs:

- **File upload:** `<input type="file">` → `AudioContext.decodeAudioData()` → `AudioBufferSourceNode`, looped, with basic play/pause.
- **Microphone:** `navigator.mediaDevices.getUserMedia({ audio: { channelCount: 2 } })` → `MediaStreamAudioSourceNode`. Request stereo explicitly — see §2.2 for why.

Both feed the same downstream graph, so switching between them is just swapping the source node.

### 2.2 Extracting carrier vs. beat (the core decision)

A single `AnalyserNode` on a mixed/summed signal cannot separate carrier from beat — a binaural beat isn't literally present as amplitude modulation in either channel; it's the *difference* between two nearly-identical tones, one per ear, perceived by the brain. Extracting it requires the stereo channels kept apart:

```
source → ChannelSplitterNode(2) → [AnalyserNode(L), AnalyserNode(R)]
```

Per animation frame:
1. Pull `getByteFrequencyData()` from each analyser.
2. Find each channel's dominant bin → convert to Hz (`bin * sampleRate / fftSize`).
3. `carrierHz = (leftDominantHz + rightDominantHz) / 2`
4. `beatHz = |leftDominantHz - rightDominantHz|`
5. `amplitude` = average magnitude across bins (either channel — used for overall intensity, not carrier/beat).

**fftSize:** 2048 (reasonable frequency resolution without excessive per-frame cost).

**Mono fallback (locked):** If the input resolves to a single channel (some mic hardware/browsers won't honor the stereo constraint), skip the split — run one `AnalyserNode` on the mono signal, use its dominant frequency for `carrierHz`, and hold `beatHz` at a fixed default (e.g. 10 Hz, mid-alpha range) rather than pretending to extract something that isn't there.

---

## 3. Driver Value Architecture — Layering Over the Manual Dial

**Decision (locked):** Audio input is a second data source feeding the *same* parameters the manual dial already drives — not a separate mode that turns the dial off.

```js
const driverValue = audioActive ? normalize(carrierHz) : frequency; // frequency = existing manual dial value
```

Where `normalize(carrierHz)` inverts the existing dial's own mapping (`110 + frequency * 660`) back to a 0–1 scalar: `(carrierHz - 110) / 660`, clamped `[0, 1]`.

This means every existing `frequency`-driven effect (group bob, wireframe opacity, emissive intensity, ground glow) keeps working completely unchanged — it just reads `driverValue` instead of `frequency` directly, and doesn't care whether that value came from the dial or from audio analysis. No parallel effect system, no separate code path for "manual mode" vs. "audio mode."

Silence handling falls out of this for free: if the source is paused or the mic is picking up near-silence, `amplitude`/`carrierHz` naturally sit near zero, and the existing effects already handle a near-zero `frequency` gracefully (that's the resting state today) — no special-casing needed.

Leaving RESONANCE mode (§5) always returns `audioActive` to `false`, handing control back to the manual dial immediately.

---

## 4. Vertex Deformation Model (new)

### 4.1 Displacement formula

Every vertex of the active solid gets displaced along its own normal, each frame, from a cached rest state:

```js
position = originalPosition + normal * amplitude * sin(t * beatFreqScalar + phaseOffset)
```

- `originalPositions` — a plain array of rest-state vertex positions, cloned once when a solid loads or CUSTOM points are placed. Required so displacement never compounds frame-over-frame.
- `amplitude` — scaled from §2's `amplitude` value (overall audio level) when audio is active, or a small constant when driven by the manual dial alone (so RESONANCE mode isn't inert without an audio source connected — dial-only wobble is subtle but present).
- `beatFreqScalar` — derived from `beatHz` (§2.2), scaled to a usable animation-speed range (beat frequencies are low, typically 1–40 Hz for binaural work, so this needs a multiplier tuned by feel, not a 1:1 Hz mapping).

### 4.2 Phase offset — radial distance, local space

**Decision (locked):** Phase offset per vertex is derived from that vertex's distance from the solid's own primary axis (whichever axis its existing rotation logic already treats as primary — likely Y), computed **in local/object space** at mesh-load time and cached alongside `originalPositions`.

```js
phaseOffset = distanceFromAxis(vertex.originalPosition) * rippleConstant
```

Rationale: the solid already auto-rotates continuously in the render loop (`coreGroup.rotation.y = t * 0.12`). Computing distance in world space would mean recomputing it every frame as the shape turns — expensive and pointless. Local-space, cached once, means the ripple pattern is anchored to the geometry itself: vertices equidistant from the shape's own center move in phase together, producing a traveling wave that reads as "the form resonating from its own center" rather than independent per-vertex jitter (which is what index-based phase offset would produce — visible "rows" tied to arbitrary vertex construction order, not the shape's actual symmetry). This also reads closer to real cymatics patterns (concentric standing waves) than a noise-based alternative would, without adding a dependency.

**Explicitly out of scope for v0.2:** a toggle for world-space "wash across the rotating solid" ripples, and noise-based phase offset (simplex-noise dependency) — both are viable future refinements, not needed for v1.

---

## 5. RESONANCE Mode

**Decision (locked):** New top-level mode alongside VIEW / CONNECT / DISCONNECT / CUT / SKETCH / CUSTOM, not an always-on background layer.

- Entering RESONANCE shows file-upload and mic-toggle controls in place of (or alongside) the existing frequency dial.
- Entering does **not** by itself start audio analysis — the user picks a file or grants mic permission explicitly (mic access must never be requested silently on mode entry; that's a hard requirement, not a nice-to-have).
- While active, vertex deformation (§4) runs on the current solid's vertices in addition to the existing dial-driven group effects (§3).
- Leaving RESONANCE mode:
  - Sets `audioActive = false`.
  - Stops and releases the mic stream's tracks (`stream.getTracks().forEach(t => t.stop())`) if mic was in use — required cleanup, mirrors the detach-on-mode-exit pattern already established for CUSTOM mode's TransformControls.
  - Pauses/releases the `AudioBufferSourceNode` if a file was playing.
  - Resets all vertices to `originalPositions` (no lingering deformation from the last frame before exit).

---

## 6. Cleanup & Edge Cases

1. **Mic permission denial:** if `getUserMedia` rejects, fall back silently to file-upload-only controls in the same mode — don't block the mode or throw an unhandled rejection.
2. **Switching solids while RESONANCE is active:** re-cache `originalPositions` and per-vertex `phaseOffset` for the new solid's vertex set, same as today's solid-switch reset for connections/custom points.
3. **CUSTOM points + RESONANCE:** custom points (`cust-N`) are also vertices with normals-of-a-sort (approximate via direction from the solid's centroid, since they don't belong to a face) — deform them the same way for consistency, or explicitly exclude them if that approximation looks wrong once built. Flagging as a build-time judgment call, not locking it now.
4. Audio graph teardown (disconnecting nodes, closing `AudioContext` if reused across sessions) matters for avoiding zombie audio contexts on repeated mode entry/exit — needs an explicit teardown function, not just letting garbage collection handle it.

---

## 7. Out of Scope for v0.2 (future refinement)

- World-space "wash across" ripple axis mode (§4.2).
- Noise-based phase offset.
- Direct same-tab routing from a generator app (no separate file/mic step) — only worth it if the generator ever actually lives in the same page.
- Snapshotting live deformation frames into the specimen log (§2.2 of v0.1) — RESONANCE mode is a visual/exploratory layer in v0.2, not a new measurement-logging event source.
- **Phase 3 — Voxluminara naming layer:** unchanged from v0.1's note — auto-naming exported specimens using root vocabulary based on frequency range or shape properties.

---

## 8. Suggested Build Order

1. Audio input pipeline (§2) — file upload first (no permission prompt, faster to test), then mic.
2. Driver value architecture (§3) — wire `driverValue` into the existing `frequency`-consuming effects; verify dial fallback still works with zero audio connected.
3. Vertex deformation model (§4) — `originalPositions` cache, displacement formula, radial phase offset.
4. RESONANCE mode UI + lifecycle (§5) — mode button, controls, enter/exit cleanup.
5. Edge cases (§6) — solid-switch re-caching, permission-denial fallback, audio graph teardown.
