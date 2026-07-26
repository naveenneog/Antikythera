# ANTIKYTHERA

A ~29-second motion-graphics title sequence, in **one self-contained `index.html`**.
No build step, no server, no API keys, no video files, no image assets.
Everything you see and hear is generated in the browser at runtime.

**▶ Live:** https://naveenneog.github.io/Antikythera/

---

## The subject

A title sequence for a documentary about the **Antikythera mechanism** — the bronze
geared analog computer built around 150 BCE, lost in a shipwreck off the island of
Antikythera around 60 BCE, and dredged up by sponge divers in 1901. It predicted
eclipses, tracked the Metonic and Saros cycles, and modelled the motion of the sun and
moon. Nothing of comparable mechanical complexity appears again for over a thousand years.

It was chosen because it hands a motion designer **two opposing motion vocabularies to
choreograph against each other**:

| Register | Behaviour | Where |
|---|---|---|
| **Organic** | slow, sine-driven drift, long eases, no hard stops | the sea floor (Act I) |
| **Mechanical** | quantised, staccato, hard `expo` landings, overshoot | the machine (Acts II–III) |

The edit cuts *between* those two registers. That tension is the piece.

---

## The edit

Five acts, ~29s, then it holds on the final frame.

| t | Act | Beat |
|---|---|---|
| 0.0s | **The Dark** | Frame closes to 2.39:1. Marine snow, light shafts, coordinates type on. A raking light uncovers a corroded bronze fragment. |
| 5.55s | **The Find** | Hard cut to a cold technical register. Corner reticles snap in; a scan bar crosses the artifact and draws the gear line-art *exactly where it has already passed*. |
| 9.45s | **The Machine** | Hard cut. Eight bronze gears fly out of the dark and land. The train starts to turn. Four cycle cards cut over it. Everything accelerates, then slams to a dead stop. |
| 16.5s | *(silence)* | **Cut to black. 0.36 seconds of nothing.** |
| 16.85s | **The Sky** | The great gear's rim becomes the outer orbit. An orrery runs, then a total solar eclipse. Totality holds. |
| 23.85s | **Title** | The corona ring is squashed on one axis until it *is* the rule under the title. |

### Three match cuts carry the structure

1. The artifact's silhouette becomes its own **X-ray** — same path, same position, register change only.
2. The great gear's rim becomes the **outer orbit**. Both are on screen together for ~0.9s at the
   same pixel radius (~298px vs 300px) while the machine dissolves out from under the circle.
3. The corona ring is **scaled on one axis to zero height** — a circle with no height is a line —
   and that line becomes the rule under the title. One object, three meanings.

---

## Everything is generated

No hotlinked video, no stock footage, no image files, no audio files.

### Geometry
Gears are real `THREE.ExtrudeGeometry`, built from a `THREE.Shape` whose tooth profile is
generated from **authentic Antikythera tooth counts** — 223 (Saros), 127 (Metonic), 64, 53, 48,
38, 32, 27 — on a **shared module**, so pitch radii are `m·teeth/2` and the gears genuinely mesh.
Each gear's angular velocity is `ω·(223/teeth)`, so the train turns at true ratios. Spoke cutouts
are `THREE.Path` holes in the same shape.

### Surface
The corroded bronze is a **value-noise heightfield** evaluated in JavaScript:
fBm → screen-space normals → Lambert term → bronze/verdigris albedo mix → pit self-shadowing.
It's rasterised once into three canvases and reused as an **SVG pattern fill** (the artifact),
a **`map`**, a **`roughnessMap`** and a **`bumpMap`** (the gears).

### Environment
No HDRI to hotlink, so the environment map is *painted* — a canvas gradient with a warm key blob
and a cold kicker, run through `PMREMGenerator` for real metal reflections.

### Post-processing
A hand-written chain (no `EffectComposer`): render to a half-float target → bright pass →
two-scale separable gaussian bloom → one composite pass doing **barrel distortion, prismatic
radial smear, chromatic aberration, grade, ACES tonemap, sRGB encode and vignette**. The
expensive smear/aberration path sits behind a *uniform* branch, so it only costs anything on the
two or three beats that actually use it.

### Optical treatment
Film grain is eight pre-baked gaussian-noise frames cycled at **24fps** (not 60 — real grain is
24) with a randomised offset, composited in `mix-blend-mode: overlay`. Plus **film gate weave**:
sub-pixel drift on the whole frame, because projected film never sits perfectly still.

### Sound
Every cue is synthesised with the **Web Audio API** — no files. A detuned-saw sub drone under a
slow filter sweep; pitch-enveloped sine hits on the cuts; band-passed noise bursts for gear
ticks; a noise+saw riser into the slam; a five-note sine pad for the sky; a reversed swell into
the title. Reverb is a **synthesised impulse response** (exponentially decaying noise) through a
`ConvolverNode`. All cues are fired from the same GSAP timeline that drives the picture, so sound
and image cannot drift.

---

## Architecture

Three layers composite into one frame:

```
#gl     <canvas>   Three.js  — particles, gears, starfield, corona, post chain
#vec    <svg>      line art  — artifact, X-ray gears, orbits, the title rule
#type   <div>      DOM       — all typography (variable fonts, per-character masks)
        grain / scanlines / halation / vignette / flash / letterbox
```

Typography stays in the DOM on purpose: real font rendering, real variable-font axes, and it
stays razor sharp regardless of the WebGL render scale.

One `gsap.timeline()` is the single source of truth. Every beat is placed at an **absolute**
time, so cut points are authored rather than accumulated. GSAP writes to a plain state object
(`GLS`) and the renderer only ever *reads* it — picture logic and draw logic never touch.

**Ten authored `CustomEase` curves.** Nothing in the piece uses a default ease:

```js
CE.create('cinema' ,'M0,0 C0.62,0 0.05,1 1,1');        // long filmic settle
CE.create('slam'   ,'M0,0 C0.04,0.92 0.09,1 1,1');      // hard mechanical stop
CE.create('settle' ,'M0,0 C0.12,0.86 0.18,1.045 1,1');  // land + micro overshoot
CE.create('antic'  ,'M0,0 C0.34,-0.30 0.18,1 1,1');     // pull back, then go
CE.create('choke'  ,'M0,0 C0.86,0.02 0.98,0.62 1,1');   // slow start, late rush
```

---

## Performance

Adaptive resolution: if rolling frame time exceeds ~21.5ms the render scale steps down
(1.0 → 0.8 → 0.64 → 0.5). It only ever steps down, at most three times, and never changes
composition, timing or colour. On a GPU-less box running a software rasterizer this doubled
throughput (15 → 32fps); on real hardware it stays at full resolution.

---

## Running it

Open `index.html`. That's it — it works from `file://`.

It uses **Three.js r149**, deliberately: r149 is the last release that ships a real UMD
`build/three.min.js`. ES-module builds require a server (module CORS blocks `file://`), which
would break the "no build step, no server" constraint.

| Control | |
|---|---|
| **Replay** | full state reset and replay |
| **Space** | same |
| **Sound** | **on by default.** Browsers won't start an `AudioContext` without a user gesture, so if yours refuses, the first click *anywhere* on the page unlocks it — and the drone joins at the point in its envelope the picture has already reached, rather than restarting underneath it. A small "click anywhere for sound" hint appears only if the browser actually blocked it. |

A small transport API is exposed for scrubbing and inspection:

```js
SEQ.play()          // resume
SEQ.seek(12.4)      // jump to a time (renders onUpdate-driven writes faithfully)
SEQ.start()         // restart from zero
SEQ.duration        // 29.05
SEQ.audio           // { state, on, want, drone, gain }
SEQ.enableSound()   // turn sound on from an embedding page's own gesture
SEQ.muteSound()
```

---

## Dependencies

Three CDN references. No package manager, no bundler.

- [Three.js r149](https://threejs.org) (UMD build)
- [GSAP 3.12.5](https://gsap.com) + CustomEase
- Google Fonts: **Bodoni Moda** (variable `wght`/`opsz`), **Archivo**, **JetBrains Mono**

---

## Licence

MIT — see [LICENSE](LICENSE).

Built by [Naveen Gopalakrishna](https://naveenneog.github.io/AI4Good/about/) with
GitHub Copilot CLI. Part of the [#AI4Good](https://naveenneog.github.io/AI4Good) series.
