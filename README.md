# Harmonic Geometry Lab

A Three.js sacred geometry visualizer with a dark holographic UI. Renders the six Platonic solids (Tetrahedron, Hexahedron, Octahedron, Icosahedron, Dodecahedron, Merkaba), lets you click vertices to connect and measure them, and includes an audio-reactive deformation mode driven by real-time frequency analysis.

Single-file app — everything lives in `index.html`, no build step required.

## Running it

Because of browser file:// restrictions on ES modules, serve the directory rather than opening the file directly:

```
python -m http.server 8000
```

Then open `http://localhost:8000/index.html`.

## Features

- **Six Platonic solids** with net/ortho/face-detail stencil projections
- **CONNECT / DISCONNECT** — click vertex dots to measure distances, angles, and edge directions between them
- **CUT / SKETCH** — freeform edge sketching independent of the solid's native geometry
- **CUSTOM mode** — place your own points on a construction plane and drag them with a TransformControls gizmo
- **Specimen log** — every measurement persists to `localStorage` and exports to JSON/CSV
- **RESONANCE mode** — audio-reactive vertex deformation:
  - Input via file upload or microphone (stereo-aware, extracts carrier/beat frequency for binaural/Hemispheric Sync audio)
  - Displacement is derived from a scalar potential field gradient (`∇φ`), not a hand-tuned wobble — see the spec docs below for the math
  - Falls back to a manual frequency dial when no audio source is connected

## Project status

| Version | What it covers | Status |
|---|---|---|
| v0.1 | Data model, persistence, export, CUSTOM mode, SKETCH integration | ✅ Built & confirmed |
| v0.2 (Phase 2) | Audio input pipeline, RESONANCE mode, vertex deformation | ✅ Built & confirmed |
| v0.2.1 | Gradient-driven deformation addendum + audio-analysis flicker fix | ✅ Built & verified |
| Phase 3 | Voxluminara naming layer (conlang-based specimen naming, `[PHYSICS]`/`[INVENTED]` tagging) | 📝 Needs spec |

Full spec documents for each stage are tracked alongside this repo (see `harmonic-geometry-lab-spec-*.md`).

## Tech

- [Three.js](https://threejs.org/) (ES module, no bundler)
- Web Audio API (`AnalyserNode`, `ChannelSplitterNode`) for real-time frequency/beat extraction
- `localStorage` for measurement persistence — no backend
