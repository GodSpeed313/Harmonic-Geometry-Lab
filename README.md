# Harmonic Geometry Lab

A Three.js sacred geometry visualizer with a dark holographic UI. Renders the six Platonic solids (Tetrahedron, Hexahedron, Octahedron, Icosahedron, Dodecahedron, Merkaba), lets you click vertices to connect and measure them, and includes a real audio-reactive cymatics engine — play a tone and watch the solid deform into a genuine standing-wave pattern, not a decorative wobble.

Also included: **Voxluminara** (`voxluminara.html`) — a companion conlang app whose phonemes are tied to real Solfeggio frequencies. Click a phoneme there and the Lab reacts live, in real time, in a separate window.

Single-file apps — everything lives in `index.html` / `voxluminara.html`, no build step, no backend.

## Running it

Because of browser `file://` restrictions on ES modules, serve the directory rather than opening the files directly:

```
python -m http.server 8000
```

Then open `http://localhost:8000/index.html`.

## How to Use

### The basics

- Pick a solid from the **SPECIMEN CATALOG** on the right (Tetra, Hexa, Octa, Icosa, Dodeca, Merkaba).
- **VIEW** — just look at it, orbit the camera.
- **CONNECT** — click two vertex dots to draw a line between them and measure the distance, angle, and edge direction. **DISCONNECT** removes a link.
- **CUT** — slice the solid with a plane to inspect its interior.
- **SKETCH** — draw freeform lines independent of the solid's native edges.
- **CUSTOM** — place your own points on a construction plane and drag them with a gizmo; they measure and connect just like native vertices.
- Every connection you make logs to the **MEASUREMENT LAB** panel and persists across reloads — **EXPORT** downloads the full log as JSON/CSV, **CLEAR DATA** wipes it.

### RESONANCE mode — making it breathe

Switch to **RESONANCE** (or press `7`). This is where the solid stops being static:

1. **Pick an input**: **UPLOAD** an audio file, or **MIC** for live input. With nothing connected, the manual frequency dial drives things instead.
2. **Pick a deformation mode** with the **MODE** button — it cycles through three:
   - **RADIAL** — a simple pulse, driven by the audio's beat frequency. The original, baseline effect.
   - **HARMONIC** — real spherical-harmonic math. Each Solfeggio frequency (396–963 Hz) maps to a genuine vibration mode, so different tones produce visibly different deformation *patterns*, not just different speeds. Node/antinode coloring shows you exactly where the solid is staying still versus swinging wide.
   - **EDGES** — classic Chladni-style standing waves *along the edges themselves*. Higher frequencies show visibly more nodes per edge — 396 Hz bows each edge once, 963 Hz shows a dramatically spikier multi-node pattern.
3. Play a real tone and watch — the CARRIER/BEAT/AMP readout tells you what's actually being detected.

### The Voxluminara Bridge — the full instrument

This is the part worth trying end to end:

1. In RESONANCE mode, click **BRIDGE**. A new window opens with Voxluminara.
2. Scroll to **THE PHONEME SYSTEM** — 18 glyphs, each tied to one of the Solfeggio frequencies and a specific deformation mode.
3. Click any phoneme card. It plays its tone *and* the Lab window reacts immediately — same frequency, same mode, live, no file uploads, no manual steps.
4. Click **SILENCE** in Voxluminara (or another phoneme) to change or stop it.

Try `Α` (963 Hz, "Source") for the busiest edge pattern, or `Ζ` (852 Hz, "Crown") for a distinct harmonic shape — then compare against `Σ` (396 Hz, "Void") to see how different the same solid can look.

## Features

- **Six Platonic solids** with net/ortho/face-detail stencil projections
- **CONNECT / DISCONNECT** — click vertex dots to measure distances, angles, and edge directions between them
- **CUT / SKETCH** — freeform edge sketching independent of the solid's native geometry
- **CUSTOM mode** — place your own points on a construction plane and drag them with a TransformControls gizmo
- **Specimen log** — every measurement persists to `localStorage` and exports to JSON/CSV
- **RESONANCE mode** — three real deformation engines, not one:
  - **Radial** — scalar potential field gradient (`∇φ`), not a hand-tuned wobble
  - **Harmonic** — real spherical harmonics (`Y_l^m`), closed-form Cartesian polynomials, each Solfeggio frequency mapped to a genuine vibration mode, node/antinode coloring
  - **Edges** — true interior standing waves along each edge, node count scales with frequency, derived via Gram-Schmidt projection for a well-defined bow direction
  - Input via file upload or microphone (stereo-aware, extracts carrier/beat frequency for binaural/Hemispheric Sync audio), or a manual dial fallback
- **Voxluminara bridge** — a companion conlang app's phonemes drive the Lab live via `postMessage`, no shared backend required

## Project status

| Version | What it covers | Status |
|---|---|---|
| v0.1 | Data model, persistence, export, CUSTOM mode, SKETCH integration | ✅ Built & confirmed |
| v0.2 (Phase 2) | Audio input pipeline, RESONANCE mode, vertex deformation | ✅ Built & confirmed |
| v0.2.1 | Gradient-driven deformation addendum + audio-analysis flicker fix | ✅ Built & verified |
| v0.3 | Cymatic Resonance — real spherical harmonics, wireframe/mesh follow-through, node/antinode coloring | ✅ Built, verified w/ real audio, user-confirmed |
| v0.4 | Edge Standing Waves — true interior nodes along each edge, Chladni-style | ✅ Built, verified w/ real audio, user-confirmed |
| Voxluminara Bridge v1.0 | Live cross-app instrument — phoneme clicks drive the Lab in real time | ✅ Built, verified, user-confirmed |

Full spec documents for each stage are tracked alongside this repo (see `harmonic-geometry-lab-spec-*.md` and `voxluminara-lab-bridge-spec-v1.0.md`).

## Tech

- [Three.js](https://threejs.org/) (ES module, no bundler)
- Web Audio API (`AnalyserNode`, `ChannelSplitterNode`) for real-time frequency/beat extraction
- Real spherical harmonics and Gram-Schmidt-derived edge geometry — no physics engine, just the math
- Cross-window `postMessage` bridge — no server, no shared origin required
- `localStorage` for measurement persistence — no backend
