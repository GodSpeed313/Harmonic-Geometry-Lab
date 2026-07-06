# Voxluminara ↔ Harmonic Geometry Lab — Live Bridge Spec v1.0

**Status:** Draft for review
**Scope:** Wires Voxluminara's phoneme cards (currently inert — hover-only CSS, no click handler, despite the UI text promising "click any phoneme to speak its tone") to actually play their tone, and bridges that click in real time to the Harmonic Geometry Lab so the solid visibly responds the moment a phoneme is clicked — no manual file loading, no separate steps. Touches both `C:\Users\User\Voxluminara\voxluminara.html` and `C:\Users\User\HTML\index.html`.

---

## 1. Problem Statement

Two gaps, confirmed against the running code:

1. **Voxluminara's phoneme cards do nothing.** Verified: `.phoneme-card` elements have hover CSS only (`voxluminara.html:258-282`) and no `onclick`/listener anywhere. The lexicon section's own copy says "Click any phoneme to speak its tone" (line 962) — a broken promise. A *different*, already-working section (the 8 `freq-btn` buttons, lines 905-912) does call a real `playFreq()`, so the synthesis code already exists; the phoneme cards just aren't wired to it.
2. **The Lab can't react to Voxluminara live.** Today, watching a tone affect the solid means: play/record a tone in Voxluminara, export or find the file, switch to the Lab, upload it there. This spec closes that gap so clicking a phoneme drives the Lab immediately, in the same moment.

---

## 2. Bridge Mechanism

**Decision (locked):** `window.open()` + `postMessage`, not `BroadcastChannel`/`localStorage` events. Both apps are standalone static files with no shared backend and are typically opened as `file://` pages — each `file://` page is its own isolated origin in Chromium, so same-origin-dependent APIs (`BroadcastChannel`, storage events) won't connect two independently-opened files. `postMessage` has no such restriction: as long as one page holds a window reference to the other, they can talk regardless of origin.

**Decision (locked):** The **Lab launches Voxluminara**, not the other way around. A new button in the Lab's RESONANCE panel opens Voxluminara via `window.open(url, 'voxluminara-bridge')`. Because Voxluminara is the *child* window in this relationship, it automatically receives a `window.opener` reference back to the Lab — no need for the Lab to track anything beyond having opened it.

```js
// Voxluminara side, wherever a phoneme is clicked or its tone stops:
if (window.opener && !window.opener.closed) {
    window.opener.postMessage({ type: 'voxluminara-tone', hz, active, mode }, '*');
}
```

`'*'` as the target origin is intentional, not sloppy — `file://` origins don't serialize to a predictable string across environments, and this is a bridge between two of the user's own local files, not a public-facing integration.

---

## 3. Message Contract

**Decision (locked):** the message carries `hz`, `active` (bool — tone started/stopped), and `mode` (one of `'radial'`/`'harmonic'`/`'edges'`) — nothing else. The Lab does not receive amplitude from Voxluminara (there's no meaningful "loudness" to read from a deliberate click the way there is from analyzing real playback); it uses a fixed strong amplitude on receipt, matching how a deliberate, clear trigger should read visually.

```js
{ type: 'voxluminara-tone', hz: 852, active: true, mode: 'harmonic' }
{ type: 'voxluminara-tone', hz: 852, active: false, mode: 'harmonic' }  // on stop
```

**Lab-side handling (locked):** on receipt, the Lab does **not** re-run its own `AnalyserNode`/FFT pipeline for this — it already knows the exact frequency, no detection needed. It directly sets:

```js
window.addEventListener('message', (e) => {
    if (e.data?.type !== 'voxluminara-tone') return;
    if (interactionMode !== 'RESONANCE') setInteractionMode('RESONANCE');
    audioActive = e.data.active;
    if (e.data.active) {
        carrierHz = e.data.hz;
        audioAmplitude = 1.0; // deliberate external trigger, not a measured level — read as fully present
        deformationMode = e.data.mode;
    }
});
```

Reuses `audioActive`/`carrierHz`/`audioAmplitude`/`deformationMode` exactly as they already work for mic/file input — the bridge is just a fourth way those variables get set, not a parallel system.

---

## 4. Phoneme → Frequency Assignment

**Decision (locked):** assign a frequency to all 18 phoneme cards, not just the 14 letters already in a documented class (`Κ,Τ,Π`=174, `Σ,Ξ,Ψ`=396, `Λ,Ω,Θ`=528, `Ν,Μ,Λ(soft)`=639, `Ζ,Φ,Χ`=852). Tagged `[INVENTED]` — same discipline as everything else in this app and the Lab's own `CARRIER_HARMONIC_MODES`.

- **`Λ`'s dual membership** (appears in both Æther/528 "hard" and Bridge/639 "soft" in the class table) resolves to **528 Hz** for click purposes — the single phoneme card can only carry one frequency, and 528 is the primary/first-documented form; the "soft" 639 variant is a pronunciation nuance in the class table, not a second card.
- **The 4 previously-uncovered letters** (`Α, Η, Ρ, Υ`) fill the 3 Lab frequencies Voxluminara's class table never assigned (`417, 741, 963`), matched against each letter's own already-written meaning *and* Voxluminara's own frequency-button labels (`"417 Hz — Change"`, `"741 Hz — Voice"`, `"963 Hz — Source"`) — not arbitrary:

| Glyph | Existing meaning | Assigned Hz | Rationale |
|---|---|---|---|
| `Α` | Origin · First · Source light | **963** | Voxluminara's own button literally calls 963 "Source" |
| `Υ` | Union · Womb · Emergence | **417** | Emergence/becoming is what "Change" means |
| `Ρ` | River · Motion · Continuity | **741** | Continuity/flow fits "Voice" (continuous expression) |
| `Η` | Seeking · Reaching · Query | **639** | Reaching-across fits "Bridge"; joins `Ν,Μ,Λ(soft)` |
| `174` (Root Stops) has no Lab mode/harmonic entry — that's fine, only `'radial'` is assigned to it (§5), which needs no `(l,m)` value. |

**Full table (`PHONEME_FREQUENCY`, keyed by glyph):**

```js
const PHONEME_FREQUENCY = {
    'Κ':174, 'Τ':174, 'Π':174,
    'Σ':396, 'Ξ':396, 'Ψ':396,
    'Λ':528, 'Ω':528, 'Θ':528,
    'Ν':639, 'Μ':639, 'Η':639,
    'Ζ':852, 'Φ':852, 'Χ':852,
    'Υ':417, 'Ρ':741, 'Α':963,
};
```

---

## 5. Frequency → Deformation Mode Assignment

**Decision (locked):** mode is assigned per **frequency** (8 values), not per individual phoneme letter (18) — consistent with `CARRIER_HARMONIC_MODES` already being keyed by frequency, and because phonemes sharing a frequency (e.g. `Κ,Τ,Π` all 174) should behave identically when clicked, differing only in which sound plays.

| Hz | Voxluminara name | Mode | Rationale |
|---|---|---|---|
| 174 | Root | `radial` | Simplest mode for the most basic/grounding tone — no harmonic entry needed or used |
| 396 | Void | `harmonic` | Reuses the Lab's own existing `(1,0)` dipole — the first differentiation out of stillness |
| 417 | Change | `edges` | "Change" reads best as visible structural motion — edges actively bowing |
| 528 | Æther | `harmonic` | Reuses the existing `(2,0)` — "wholeness" as a real spatial mode shape |
| 639 | Bridge | `edges` | Edges *are* literally the bridges between vertices — direct thematic match |
| 741 | Voice | `edges` | Continuous flowing motion along edges fits "voice/expression" |
| 852 | Crown | `harmonic` | Reuses the existing `(3,2)` "star" mode — higher activation as real structure |
| 963 | Source | `edges` | Reuses the existing `(4,4)` — the richest, most textured pattern we've built (confirmed spectacular in v0.4 testing) for the "source of everything" |

```js
const FREQUENCY_MODE = { 174:'radial', 396:'harmonic', 417:'edges', 528:'harmonic', 639:'edges', 741:'edges', 852:'harmonic', 963:'edges' };
```

---

## 6. Voxluminara-Side Changes

1. Add a click handler to every `.phoneme-card` (currently none exist): on click, look up `PHONEME_FREQUENCY[glyph]` and `FREQUENCY_MODE[hz]`, call the **existing** `playFreq(hz)` (no changes to that function — it already does the right synthesis), and `postMessage` per §2/§3.
2. On `stopFreq()` (existing function, already called by the "SILENCE" button and re-clicking a playing card), also `postMessage` an `active:false` message so the Lab resets.
3. Visual feedback on the clicked card (reuse the existing `.freq-btn.active` class pattern already used by the frequency buttons) so it's clear which phoneme is currently sounding.

---

## 7. Lab-Side Changes

1. New button in the RESONANCE panel: "OPEN VOXLUMINARA BRIDGE" — `window.open('file:///C:/Users/User/Voxluminara/voxluminara.html', 'voxluminara-bridge')`. (Exact path is a local build detail; a relative/configurable path is fine too.)
2. `window.addEventListener('message', ...)` per §3 — added once, at the top level, not inside `animate()`.
3. No changes to `updateResonanceDeformation`, `harmonicValue`, `updateCymaticEdges`, etc. — the bridge only ever sets the same state variables (`audioActive`, `carrierHz`, `audioAmplitude`, `deformationMode`) those functions already read every frame.

---

## 8. Cleanup & Edge Cases

1. **Popup blocked:** `window.open()` can return `null` if blocked. Check for it and show a message ("allow popups for this bridge") rather than throwing on the next `postMessage` call site (which lives in Voxluminara, not the Lab, so this only affects the Lab's own launch button failing silently otherwise).
2. **Voxluminara window closed while a tone is "active":** no automatic message fires when a window is closed by the user (no `beforeunload` message guaranteed to arrive). The Lab has no way to detect this from `message` events alone. Mitigation: don't try to detect it — leaving the last-received state visible until the next click/mode-switch is an acceptable, low-cost edge case, not worth adding a heartbeat/ping mechanism for v1.0.
3. **Rapid phoneme clicking:** each click's message fully overwrites `carrierHz`/`deformationMode` — no queueing needed, last click wins, matching how `updateResonanceDeformation` already reads these as live, not queued, values.
4. **User manually changes mode/frequency in the Lab after a bridge message:** allowed and expected — the bridge only *sets* these variables on message receipt, it doesn't lock them; the existing mode-toggle button and dial remain fully functional in between bridge events.
5. **Leaving RESONANCE mode in the Lab while Voxluminara is still playing a tone:** existing `resetAllVerticesToRest()` cleanup (already fires on `setInteractionMode` away from RESONANCE) runs as normal — the bridge doesn't need special-case cleanup beyond what already exists for audio-active state.

---

## 9. Out of Scope for v1.0

- Detecting/handling the Voxluminara window being closed (§8.2) — acceptable rough edge for v1.0.
- The Lab driving Voxluminara (reverse direction) — this spec is one-directional (Voxluminara → Lab) only.
- Amplitude/loudness data from Voxluminara — fixed strong amplitude on trigger, no volume slider integration.
- Any change to `playFreq()`'s actual synthesis (binaural offset, oscillator setup) — untouched, reused as-is.
- The original "naming layer" idea (auto-naming exported Lab specimens) — separate, unrelated, still unspecced.

---

## 10. Suggested Build Order

1. `PHONEME_FREQUENCY`/`FREQUENCY_MODE` tables (§4/§5) as plain data in Voxluminara — no behavior yet, verify the tables are complete (all 18 glyphs, all 8 frequencies) before wiring clicks.
2. Wire phoneme card clicks to call the existing `playFreq()` (§6.1) — verify tone-playing works standalone in Voxluminara first, independent of the Lab entirely.
3. Add the `postMessage` call alongside the existing `playFreq()`/`stopFreq()` calls (§6.1/§6.2).
4. Lab-side: "OPEN VOXLUMINARA BRIDGE" button + `window.open()` (§7.1).
5. Lab-side: `message` listener (§7.2) — verify with a manual `postMessage` call from the console before relying on the real Voxluminara click path, to isolate bugs in the receiver from bugs in the sender.
6. End-to-end: real click in a real Voxluminara window, real reaction in a real Lab window, both open side by side.
