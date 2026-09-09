# Project Plan: Real-Time Close-Range Detection from Passive SWIR Stereo on the GateMate A1

**Status:** primary driver document — active. Created 2026-09-03 as the driver
for the whole effort (see `thesis-proposal.md` §8 decision log); framing revised
2026-09-07 to match the proposal-defense deck
(`../presentations/proposal-defense/deck.md`); engineering approach **decided
2026-09-09** — see §4 and §8.
**Last updated:** 2026-09-09
**Clock:** 30 weeks, shared with the master's thesis.

This is the driver document for the whole effort: the engineering objective, the
fixed hardware envelope, the decided engineering approach, and the 30-week
schedule. The concrete goal is **real-time close-range detection** — decide, at
the sensor frame rate, whether an object has come within a configured distance
band, on the fixed hardware below. The research framing (`thesis-proposal.md`) is
developed alongside this document and finalised around the decided approach.

**Approach decided 2026-09-09 — a band-limited disparity-range sweep.** The
system tests the match cost only at the `K` disparity planes of the configured
watch band, gates / reduces / confirms, and emits a boolean-or-zoned
range-threshold flag. It does **not** compute a per-pixel disparity map, and it
does **not** build a sparse key-point set. This is the route §4–§5 evaluated as
**Route C** (`implementation-plan.md` Config 7 — "band-limited plane-sweep /
disparity-threshold trigger"). §4 now records it as the decision; the dense-map
(Route A) and bounded sparse key-point (Route B) alternatives are kept there as
the considered-and-set-aside options. The output *granularity* is therefore
settled: a boolean/zoned flag (optionally carrying the surviving plane index),
not a dense or semi-dense map. Boolean-vs-zoned for v1 remains an open sub-choice
(§2.9).

---

## 0. What this document is, and how the docs relate

- **`product-plan.md`** (this file) — the driver and entry point. Owns: the
  engineering goal (§1), the output specification (§2), the fixed-hardware
  envelope and the application constraints in brief (§3), the decided engineering
  approach and the alternatives considered (§4), what M2/M3 still characterise for
  the chosen approach (§5), the milestone / gate / kill-finding timeline (§6), and
  the project-level decisions log (§8).
- **`thesis-proposal.md`** (this folder) — the academic artefact, wrapped around
  the chosen approach and cross-linked here. Authoritative for: the decided
  hardware / sensor spec (§2), the bandwidth-arithmetic derivation with the
  `VERIFIED` / `VENDOR-CONFIRMED` / `ESTIMATED` verification legend (§3), the
  research-method description of the tracks (§4), the fallback thesis (§5), the
  open-items list with owners and dates (§7), and the chronological project
  decision log (§8). The research question (§1) is kept provisional pending the
  advisor, though the engineering approach it wraps is now decided (§4).
- **`implementation-plan.md`** (this folder) — the RTL detail: consolidated spec
  sheet (§2), constraint → viable-technique chain (§5), the configurations (§6–§7).
  As of 2026-09-09 the build target is **Config 7** (Route C, the decided
  disparity-range sweep); Configs 1–6 are retained there as the alternatives that
  were considered and set aside.
- **`../stereo_camera_fpga/`** — the FPGA build workspace (no RTL yet).
  **`../stereo_camera/`** — the supporting Raspberry Pi stereo rig.
  **`../stereo_camera_fpga/research/synthesis/`** — the neutral technique /
  device reference matrices.

**Directory-layout dependency:** the relative links in this file assume the four
repos stay siblings under one parent (`thesis/`). They are not git submodules;
if this repo is moved on its own, the `../stereo_camera_fpga/...` links break.

---

## 1. Goal

**Real-time close-range detection** on the Cologne Chip GateMate A1 FPGA fed by
a **passive SWIR (InGaAs) stereo head**: at the ~600 fps sensor rate, decide
reliably whether an object has come within a configurable distance band.
**Missing a close object is the failure that matters most** (§2.5).

**"Working v1" means:** a reliable in-band / out-of-band decision at the
configured band, produced at sensor frame rate, demonstrated on real hardware —
or, if hardware bring-up slips past its kill gate, in synthesis + simulation
against stored frames (the M2 + M3 fallback, §6).

**Explicitly not:**

- a distance *measurement* (no metric range output);
- active illumination or a projected pattern;
- monocular / single-camera depth.

**Decided 2026-09-09 — the output granularity.** The system emits a
**boolean-or-zoned range-threshold flag** (optionally carrying the surviving
plane index), *not* a dense disparity map and *not* a sparse key-point set. What
binds every part of the design is the decision the system must deliver and the
reliability bar it must clear (§2.5). Boolean-vs-zoned for v1 is still open
(§2.9).

Passive SWIR stereo is fixed. The matching **style** is also decided: a
**band-limited disparity-range sweep** — cost evaluated only at the `K` disparity
planes of the watch band, never an `argmin` over `0..D` (§4). The sensor rides a
**moving platform** (not a fixed-mount zone monitor), with closing speeds up to
~600 m/s, and the system is **purely passive** — no active illumination, no
sensor fusion. These narrow the design; see §3.

**Why this hardware, despite the constraints:**

- Fully open toolchain (yosys + nextpnr-himbaechel + openFPGALoader) —
  reproducible end-to-end on ~$100 of hardware, and a citable "first" for
  real-time stereo in the open-EDA space (§7).
- SWIR (~1–1.7 µm) sees through fog / haze and works in low light and behind
  some materials where visible-light stereo is blind — the scenes that motivate
  the work.
- No prior real-time FPGA stereo work runs on a device this small / this shape,
  and none uses a SWIR sensor — so the platform itself carries the research
  contribution regardless of the engineering approach (§7).

Decided hardware / sensor spec: `thesis-proposal.md` §2.

---

## 2. What the system must decide

The system must deliver a trustworthy **in-band decision** — is an object within
`[Z_near, Z_far]` — at frame rate. As of 2026-09-09 the *form* of that output is
decided (§4):

- a **boolean (or zoned N-bit) flag** (§2.1–§2.3), optionally carrying the
  surviving watch-band plane index for debug / display;
- **no dense disparity map and no sparse key-point list** — the band-limited
  range sweep never produces one;
- boolean-vs-zoned for v1 is the one open sub-choice (§2.9).

The rest of this section specifies that boolean/zoned output and the reliability
metric that binds it.

### 2.1 What fires

A boolean (or zoned N-bit) signal, asserted while an object occupies the
configured distance band and de-asserted otherwise. The consumer is an external
system reacting to "something is close," not a depth map viewer.

### 2.2 Configurable band

Near and far limits `Z_near`, `Z_far` (metres), converted once to a disparity
plane `d = f·B / Z` for the real product head's focal length `f` and baseline
`B`, and held in a runtime register so the band can be retuned without a
re-synthesis. Closer object ⇒ larger disparity, so the test is "is there a
sufficiently large, sufficiently confident region of disparity ≥ `d(Z_far)` (and
≤ `d(Z_near)` if the band is bounded on both sides)."

### 2.3 Zoned variant (optional)

N adjacent bands → an N-bit output (which band the nearest object is in). Only
carried if it is near-free on the fabric — it costs N disparity-plane tests
instead of one.

### 2.4 Latency budget

End-to-end photon-to-alarm-edge latency target — needs a figure from the use
case to become a selection criterion (§5). Placeholder working assumption until
then: a few frame-times at 600 fps plus pipeline depth, i.e. single-digit
milliseconds.

### 2.5 Safety-critical metric

**A false negative on a close object is the dominant failure** — missing
something that is actually inside the band. The alarm's target false-negative
rate at the band edge is stated separately from its false-positive (nuisance)
rate. Because SWIR matching failure is spatially structured by material
(specular metal, glass, very low-albedo surfaces), the metric must report
*where* detection is unreliable, not only an aggregate rate.

### 2.6 Robustness envelope

The scenes the system must hold up in — fog / haze, glass, low light — and the
minimum valid-match density / confidence needed for the near/far decision to be
trustworthy in them. A near-free per-pixel confidence gate (reject matches whose
local gradient energy is below a threshold rather than emitting a garbage
disparity) is assumed available to every candidate approach.

### 2.7 Blind / UNKNOWN state

A textureless close object with **no textured rim in the field of view** is an
**irreducible** residual for any passive method without heavy regularisation
(`../stereo_camera_fpga/design/CLAUDE.md` §1). It must not be reported silently
as "clear." The output therefore carries a companion **valid / blind (UNKNOWN)**
flag, and a large, sustained UNKNOWN region inside the watch zone triggers a
**conservative policy** — treat as a possible near object / degrade gracefully —
agreed with the alarm consumer. Boolean-or-zoned v1 currently specifies neither
the flag nor the policy; both are §2.9 open items.

### 2.8 What relaxing the metric-depth requirement buys

Because the system owes a *decision*, not a metric depth map, several of the
expensive parts of stereo relax — and the decided approach (§4, the band-limited
range sweep) is the one that takes all of them:

- **Density is a free variable** — the output is a decision, not a map, so
  dense / semi-dense / sparse are all admissible and the bandwidth budget
  favours emitting *less*.
- **The disparity search collapses** — a band test is effectively a comparison
  against ~one disparity plane, not an `argmin` over `0..D`, so the dominant fps
  lever (shrink the disparity range) is already most of the way pulled.
- **No dense-map write traffic** — sparse / semi-dense output is a short list or
  a coarse mask, not a full raster.
- **Sub-pixel refinement stops being load-bearing** — it matters for *metric*
  depth precision, not for a band decision.
- **Coarser resolution and holes are tolerable**, bounded only by the §2.5
  detection bar.
- **Ground-truth burden shrinks** — known-distance targets at measured stand-off
  are enough to score a band decision; no dense-disparity ground truth needed
  (`thesis-proposal.md` §7).

**Counter-pressure:** SWIR texture starvation. Every prior IR / thermal stereo
*system* pairs its local cost with semi-global aggregation plus edge-preserving
refinement because the raw local cost is texture-starved — pushing back toward
*some* aggregation, i.e. the semi-dense middle of the menu (§4).

Bandwidth consequences of all of the above: `thesis-proposal.md` §3,
`implementation-plan.md` §2.5. Not reproduced here.

### 2.9 Open items

- [ ] **Electrical output form** — GPIO / PMOD logic line, UART message, VGA
      overlay, or several. Drives the control-plane design and whether egress
      needs anything beyond VGA.
- [ ] **Boolean vs zoned for v1** — is a single near/far boolean enough, or is a
      small number of zones a hard requirement.
- [ ] **UNKNOWN-state flag + policy** (§2.7) — the companion valid/blind bit and
      the large-blind-region conservative default are unspecified.
- [ ] **Band values and the real head's baseline** — `Z_near` / `Z_far` in
      metres, and `B` for the *real* product head (the Pi rig's geometry is
      moot — throwaway scaffolding).
- [ ] **Detection-range requirement** — drives baseline / focal length / frame
      rate. A ~12 cm baseline at 600 fps gives only ~2–4 frames of warning and
      usable stereo disparity in the last few metres; a longer standoff needs a
      wider baseline, a longer focal length, a higher (windowed-ROI) frame rate,
      or stereo as a last-metres confirm behind a monocular looming detector
      (`../stereo_camera_fpga/design/CLAUDE.md` §4).
- [ ] **Latency budget number** (§2.4) — the alarm consumer's reaction time; if
      it is not single-digit ms the loop does not close at 600 m/s and detection
      must move farther out.
- [ ] **Quantitative false-negative target** (§2.5), agreed before M3.
- [ ] **Money-scene weighting** — fog dominant, or fog / glass / low-light
      equally weighted; sets how heavily SWIR-texture robustness is weighted in
      selection (§5).

---

## 3. Fixed hardware envelope and application constraints (brief — detail is elsewhere)

- **FPGA:** Cologne Chip GateMate A1 (CCGM1A1) on an Olimex GateMateA1-EVB —
  20,480 CPEs, no hardened DSP/MAC, VGA-only egress, no hardened CPU.
- **Sensor:** QDI Systems SWIR (InGaAs) camera, 640×512, 14-bit, ~600–700 fps,
  Camera-Link-style LVDS.
- **Toolchain:** yosys + nextpnr-himbaechel + openFPGALoader, fully open-source.

### 3.1 Application constraints

From the 2026-09-04 design exploration
(`../stereo_camera_fpga/design/CLAUDE.md` §1, §4). They narrow the design well
beyond a generic fixed-mount zone monitor:

- **Moving platform** — the pushbroom-stereo regime (Barry & Tedrake), not a
  fixed mount. No static background-disparity model can be calibrated, and
  online background learning on the FPGA is rejected (stateful, memory-hungry,
  can learn away a real intruder). This rules out the background-subtraction
  false-positive fixes that assume a static scene.
- **Closing speed up to ~600 m/s** — at 600 fps that is ~1.0 m of travel per
  frame; an object crossing a 6→3 m watch band is present for only ~3 frames,
  and 3 m → contact is ~5 ms. Long-window (32–64-frame) temporal filters are
  dead; approach confirmation collapses to a 2-of-3 / fast-path frame-to-frame
  Δd test. The alarm consumer must act in single-digit ms, or detection must
  move farther out regardless of sensor quality.
- **Anything that comes near is a valid trigger** — ground / terrain approach
  included. No ground-plane reject, no angular gating. This *simplifies* the
  pipeline.
- **Purely passive** — no active illumination, no projected texture, no sensor
  fusion. Founder-firm.
- **Irreducible residual** — a textureless close object with no textured rim in
  the field of view falls to the UNKNOWN state (§2.7), not to detection.

### 3.2 Hard constraints that bound *every* candidate approach

- **BRAM ~160 KB total** `VERIFIED` — the line-buffer ceiling; every line
  buffer, cost buffer, and disparity buffer comes out of this one pool.
- **PSRAM ~97.6 MB/s effective, single-port, SDR** `ESTIMATED` (part ID
  `VERIFIED`) — reads and writes share one bus; capacity ~7 stereo pairs. No
  frame staging → **streaming / line-buffer processing is forced, not a
  choice.**
- **No hardened DSP/MAC** — 2×2-bit multiplier per CPE is the largest hard
  arithmetic primitive; wide multiplies are chained CPEs.
- **Camera ingest 344 MB/s @ 600 fps** (401 MB/s at the 700 fps LVDS ceiling)
  `VENDOR-CONFIRMED`. One 5 Gb/s SerDes lane carries roughly one SWIR stream,
  not two — two-camera ingest needs a second path, and the two sensors must be
  **line-locked** or realigning them forces a frame buffer and the streaming
  premise collapses.
- **Rectification** must be computed on the fly from ~a dozen polynomial
  lens-distortion + rotation coefficients (fixed at calibration time) and fused
  into the matching line buffer — a full per-pixel remap LUT is ~16× the BRAM
  budget and ~15× the PSRAM bandwidth. The vertical-misalignment budget `k`
  must be a signed mechanical/optical spec for the real head (target ≤ ±16
  rows); realistic free on-chip headroom after the matching window and disparity
  buffers is ~16–32 rows.
- **Timing closure on the young `nextpnr-himbaechel` flow** — not CPE / BRAM
  count — is the real feasibility gate. There is zero published fmax for any
  real GateMate design.

Full detail and all bandwidth arithmetic: `thesis-proposal.md` §2–§3,
`implementation-plan.md` §2. Not duplicated here.

---

## 4. The decided engineering approach — a band-limited disparity-range sweep

**Decided 2026-09-09.** Passive SWIR stereo was already fixed; the matching
**style** is now decided too. The system performs a **band-limited
disparity-range sweep**: rectify on the fly (§3.2) → Census transform → evaluate
the Hamming cost only at the `K` disparity planes of the configured watch band →
per-pixel confidence gates → spatial reduction → a two-frame Δd confirm → a
boolean/zoned readout. It never runs an `argmin` over `0..D` and never emits a
disparity map. This is the route the earlier search called **Route C**; its RTL
form is `implementation-plan.md` **Config 7** and its per-stage design is
`../stereo_camera_fpga/design/CLAUDE.md`.

What M2 / M3 (§6) now do is **characterise and parameterise this one approach**
against the §2 decision on the §3 hardware, scored by §5 — not choose between
routes. The research framing is finalised around it afterward (§7).

### 4.1 The decision, and the alternatives set aside

The work began aimed at a dense disparity map (Route A, the textbook stereo
output). The research since (§2.8) showed the close-range-decision goal relaxes
the expensive parts of stereo — the disparity search collapses to ~one plane,
density becomes a free variable, sub-pixel stops being load-bearing — so the
disparity-range sweep delivers the decision at the smallest on-chip footprint of
anything considered. The two alternatives, kept here as the record of what was
weighed:

- **Route A — a dense disparity map (set aside).** Census/SAD cost, W×W aggregation,
  winner-take-all, one disparity per pixel; optional stronger smoothing
  (dual-path SGM) or a density-reduced semi-dense seed-and-grow variant. The
  route the work started from. For a band decision the dense forms can run as a
  **single / few-plane occupancy test** — cost evaluated only at the band's
  disparity plane(s), not an `argmin` over `0..D` — collapsing the disparity
  search and the on-chip state. *Strain on the A1:* the aggregation stage — the
  good (recursive) methods are inherently serial, hard for a young place-and-route
  tool to close timing on, and their per-path accumulators want BRAM the A1
  lacks; a full raster per frame is also what the streaming budget least wants.
  RTL: `implementation-plan.md` §6 Configs 1–4, 6.
- **Route B — bounded sparse key-point matching (set aside).** Single-scale
  FAST/Harris → BRIEF or the reused Census word → 1-D along-row Hamming search →
  bounded verify (LRC + ordering + fixed epipolar offset) → short list of matched
  near points. **No RANSAC, no triangulation, no variable-length scatter/gather.**
  *Why set aside:* the detector + non-max suppression + top-K ranking are extra
  fabric structure, single-scale only (no image pyramid in 160 KB), list /
  bucket / verify sequencing with no CPU — and SWIR texture starvation on top,
  giving too few / too-weak key-points in exactly the target scenes, where a
  missed detection is the safety-critical failure. Retained as an *additive*
  fast-path channel option inside the chosen approach, not as the front end
  (`../stereo_camera_fpga/design/CLAUDE.md`, keypoint-pre-gate variant). RTL:
  `implementation-plan.md` §6 Config 5.
- **Route C — the decided approach: a direct range-threshold plane-sweep.** Never
  builds a map: Census-transform both images, compute the Hamming cost at just
  the `K` planes of a watch band, gate each pixel with per-pixel confidence
  filters (texture gate + curve-shape reject + LRC), reduce the surviving near
  mask spatially (U/V-disparity histograms), confirm with a two-frame Δd test,
  emit a boolean/zoned readout. *Strain on the A1 to resolve in M2:* the `K`-plane
  cost bank must run combinational at pixel rate — **this is the timing-closure
  gate** — and false positives must be held down without a static background model
  (the platform moves, §3.1). RTL: `implementation-plan.md` §6 Config 7; full
  pipeline `../stereo_camera_fpga/design/CLAUDE.md`.

Background on how the dense few-plane form was weighed against bounded
key-point, with techniques and FPGA literature:
`../stereo_camera_fpga/research/synthesis/dense-vs-keypoint-for-threshold-alarm.md`
(a point-in-time analysis; its "decide in M3" recommendation is superseded by the
2026-09-09 decision).

### 4.2 The configuration built, and the ones considered

The build target is the **band-limited plane-sweep / disparity-threshold
trigger** (Route C, `implementation-plan.md` §6 Config 7). The other rows are the
alternatives that were weighed and set aside (§4.1); they are kept for the record
and as M2 comparison points if a synthesis result forces a rethink. Full RTL
detail for each is in `implementation-plan.md` §6.

| Approach | Route | What it is | A1 fit | Main risk |
|---|---|---|---|---|
| **Dense Census + fixed window** | A | Census transform, Hamming cost, W×W window, winner-take-all | Hamming = XOR + popcount, LUT-native; lowest timing-closure risk of any real matcher | Edge-fattening at depth discontinuities |
| **Dense SAD + box-filter moving sum** | A | SAD over a fixed window, O(1) window-sum update, WTA | Simplest arithmetic (subtract / abs / add on the carry chains); no serial dependency | Radiometrically fragile between two SWIR cameras with any gain/offset mismatch |
| **AD-Census + cross-based aggregation** | A | Capped AD + Census cost, adaptive "+"-shaped support from 4 arm registers | Small extra logic on Census; cheap in registers, no big buffer | Adds a data-dependent stage; needs a synthesis run to confirm it stays cheap |
| **Semi-dense seed-and-grow / ELAS-style** | A | Confident Census/Hamming seeds → guided fill along gradients, fan-out bounded on fabric | Emits far less than a dense map; reintroduces a smoothness prior cheaply | Bounding the growth on fabric without the data-dependent tail; every surveyed system used an ARM core to grow |
| **Bounded sparse key-point** | B | Single-scale FAST/Harris + BRIEF or reused Census word + 1-D along-row Hamming; hard cap + spatial buckets + top-K; deterministic LRC + ordering + fixed epipolar-offset + parabola sub-pixel; **no RANSAC / triangulation / scatter-gather** | Lowest output bandwidth; reuses the Census datapath; the band decision needs a bounded set of near matches, not a point cloud | SWIR texture starvation → too few / too-weak key-points in exactly the fog / glass / low-light scenes the work targets |
| **Band-limited plane-sweep / disparity-threshold trigger — DECIDED (2026-09-09)** | C | Census + Hamming cost at `K` watch-band planes only; per-pixel texture / curve-shape / LRC gates; per-frame far-prior; U/V-disparity spatial reduction; two-frame Δd confirm; boolean/zoned readout. **No `argmin` over `0..D`, no disparity map.** | Smallest on-chip footprint of the options considered; disparity search already collapsed to ~one plane; no cost volume, no dense-map writes | The `K`-plane Hamming bank must run combinational at pixel rate — the timing-closure gate; false positives without a static background model |
| **Dual-path (H+V) SGM via dependency-relaxation** | A | H + V path aggregation only, recursion reading n pixels back, datapath replicated across n PUs | Avoids the width-scaling diagonal buffers and the wide comparator tree | Quantified accuracy cost +0.12 disparity error / +1.96 % bad-pixel per PU; only if resource headroom remains |
| **Optical flow as an assist** | A/C | Frame-to-frame key-point tracking to amortise detection | Bounded if track count + iterations are capped | Orthogonal — lowers per-frame detect cost, does not itself produce disparity |

### 4.3 Structurally excluded

Named for completeness; the exclusion is architectural, not a close call.

- **Full 8-direction SGM** — the four diagonal paths need buffers that scale
  with image width, out of the 160 KB pool.
- **Belief propagation** — per-node memory scales with disparity range ×
  iteration count, exactly what the BRAM ceiling bites hardest.
- **CNN / learned matching** — needs a hardened MAC array the A1 does not have.
- **Full sparse-SLAM tail** (unbounded key-point list, NN over descriptor sets,
  RANSAC essential-matrix fit, Delaunay triangulation) — the irregular /
  iterative tail every surveyed FPGA design offloads to a CPU the A1 lacks.
- **Image pyramids / integral images** (DoG, SURF, CenSurE, BRISK scale space) —
  the octave stack / summed-area table does not fit in BRAM at 640×512.

### 4.4 What the decided approach takes from §2.8

The band-limited range sweep takes every relaxation §2.8 allows: the band test
collapses the disparity search to ~one plane, there is no dense-map write
traffic, density is a free variable, and sub-pixel is dropped. It shrinks the
on-chip footprint furthest of the options considered. The remaining
counter-pressure is SWIR texture starvation (§2.8); the approach absorbs it with
its per-frame far-prior + spatial-coherence + Δd stack rather than with
aggregation.

### 4.5 Deferred side option — monocular TTC looming channel (NOT part of the approach)

**Monocular time-to-contact (TTC) / optical-expansion looming.** Recorded
2026-09-08 as an optional future **side-experiment** — **not** part of the
decided range-sweep approach, and **not** on the M1–M4 critical path. Do not
integrate it into the main pipeline now.

Kept on the books because it is cheap to bolt on and cheap to falsify: it reuses
the front end every stereo route already builds (rectify → Census / gradient),
and a Horn–Fang–Masaki-2007-style *direct* estimator adds only ~10 image-wide
fixed-point accumulators plus a small solve — no multipliers, minimal BRAM, one
streaming pass, low timing-closure risk — and it can be characterised entirely
offline (NumPy + rendered / KITTI forward-motion clips) with no SWIR dataset and
no hardware contention. It would run concurrently on one already-ingested stream
and may only ever **OR-in** an alert, never gate a stereo route.

The one larger dependency is **IMU / ego-motion**: without known ego-velocity the
output is a *time*, not a `[Z_near, Z_far]` decision, and object-vs-background
separation is done on raw flow structure; with an IMU, `Z ≈ W·τ` converts the
band directly and the separation becomes residual detection. That step is out of
scope for a first pass.

llm-council (2026-09-08) verdict: **not viable as a primary trigger or a
mandatory pre-filter** — the focus-of-expansion degeneracy coincides with the
head-on threat, the ~3-frame / ~600 m/s regime defeats the temporal averaging
TTC accuracy needs, and SWIR low texture + motion blur along the expansion axis
attack the signal. Viable only as an **optional OR-in looming channel**.

**Observability check — done 2026-09-08** (`.../synthesis/time-to-contact-explained.md`
§7, `f_px = 800` placeholder): **provisional pass-with-caveats, not a kill.** For
an extended target (`S ≳ 0.1 m`) in a ≤ ~6–9 m band the per-frame image
expansion is 2–25+ px (not sub-pixel), and for `S > B` it exceeds the stereo
per-frame disparity-change signal — raw detectability is not the close-range
blocker. But motion blur ≈ the inter-frame step, so with ~3 frames the τ
*precision* caps at ~15–35 % (≈ ±0.5–1 m at a 3 m edge) — a coarse OR-in flag,
not a calibrated band; and `Δs ∝ f_px·S/Z²` sends small (≲ 3 cm) or far
(≳ 10–15 m) targets sub-pixel. Full notes:
`../stereo_camera_fpga/research/synthesis/time-to-contact-explained.md` §6–§7,
`../stereo_camera_fpga/research/summaries/2026-09-08-ttc-optical-flow-fpga.md`,
`../stereo_camera_fpga/research/summaries/2026-09-08-monocular-depth-perception.md`.

---

## 5. What M2/M3 characterise for the decided approach

The engineering approach is decided (§4). M2/M3 no longer choose between routes;
they **characterise and parameterise the band-limited range sweep** — set `K`
(watch-band width), boolean vs zoned, whether a key-point pre-gate earns its
place — and either **confirm it clears the bar or return a kill finding** (§6).

### 5.1 Evaluation criteria

All driven by the §2 in-band decision:

1. **Detection reliability at the band edge** — correct near/far call vs true
   stand-off on known-distance targets.
2. **False-negative rate on close objects** — the safety-critical failure
   (§2.5); weighted highest.
3. **False-positive (nuisance) rate.**
4. **Sustained fps at the sensor rate.**
5. **Resource fit** — CPE utilisation, BRAM against the ~160 KB line
   (after the rectification `k` budget), residual PSRAM traffic.
6. **Timing-closure risk / confidence on `nextpnr-himbaechel`** — the real
   feasibility gate (§3.2), and for this approach specifically the `K`-plane
   Hamming bank at pixel rate.
7. **SWIR-texture robustness** in fog / glass / low-light, reported spatially.
8. **End-to-end latency** vs the §2.4 budget — at ~600 m/s the loop only closes
   if this is single-digit ms (§3.1).

### 5.2 Scoring

Criterion 2 (false-negative rate on close objects) is a **hard gate** — if the
chosen approach cannot clear the agreed target it is a kill finding (§6),
regardless of its other numbers. The remaining criteria are characterisation
results and inputs to the `K` / boolean-vs-zoned / key-point-pre-gate parameter
choices, with the §2.9 money-scene weighting setting how heavily criterion 7
counts.

**Write the kill line before M3:** the minimum fps + minimum detection
reliability below which the goal is not met, agreed with the advisor. Without it
up front, the pressure to flatter a weak result is irresistible. A defensible
negative outcome — "no candidate approach clears the bar on this part + this
sensor" — is only honest if the bar predates the result.

### 5.3 How the milestones feed the parameterisation

- **M2** (synthesis sweep) → does the range-sweep RTL (Config 7) close timing at
  pixel rate and fit BRAM, and at what `K` / resolution / disparity range
  (Gate C, §6). A "no point clears the product minimum" result is a kill finding.
- **M3** (implement + benchmark against stored frames) → the §5.1 numbers →
  **`K`, boolean vs zoned, and whether a key-point pre-gate is used are fixed
  here**, and the approach is confirmed against the bar or killed.
- **M4** (live integration) → validation of the approach on real hardware, not a
  re-open of the choice.

### 5.4 Evidence standard

Synthesis-backed place-and-route numbers (CPE / BRAM / fmax) and benchmark
measurements — not spreadsheet arithmetic. Napkin math is not a basis for the
pick.

### 5.5 Open items feeding selection

- [ ] **Threshold-alarm quality bar quantified** (§2.4, §2.5, §5.2) — the
      authoritative open item is `thesis-proposal.md` §7.
- [ ] **SWIR ground-truth / benchmark plan chosen** — evaluate on visible-light
      and argue transfer, a small SWIR rig with known-distance targets, or
      synthetic SWIR (`thesis-proposal.md` §7).
- [ ] **Rectification `k` spec in hand** — signed mechanical/optical budget for
      the real head, target ≤ ±16 rows, by ~week 4 (`thesis-proposal.md` §7,
      `../TODO.md` item 3).
- [ ] **Dataset-plumbing path for M3** — how stored frames reach the device
      (UART / flash / streamed to mimic the camera) (`implementation-plan.md`
      §8).
- [ ] **Detection-range vs. sensing geometry** (§2.9) — whether the stated
      standoff is reachable at all with a realistic baseline / focal length at
      ~600 fps, or whether the geometry (not the matcher) is the blocker.
- [ ] **IMU / ego-motion available?** — decides whether Route C's confirm stage
      is a motion-compensated predictor (Barry & Tedrake) or a plain
      N-of-M + monotonic-Δd test (`../stereo_camera_fpga/design/CLAUDE.md` §7).

---

## 6. Milestones, gates and kill findings

The 30-week schedule, relocated here from `thesis-proposal.md` §4 (which now
keeps only the research-method rationale for the parallel-track structure and
points here). Two independently-gated parallel tracks, then two sequential ones.

```
Week:   1    2    3    4    5    6  ...  16       24       30
M1:   [--- bring-up ---][Gate A][-- camera bring-up --][Gate B]
M2:   [-- synthesis-only feasibility sweep --][Gate C]
M3:                          [-- implement & benchmark candidates --]
M4:                                      [-- live integration --][write-up / harden]
```

### M1 (= Track A) — Toolchain + hardware bring-up
*Needs real silicon. Highest personal-risk item (first time flashing a board,
unproven toolchain on this device).*

- **Gate A** (~wk 4–5): a *non-trivial* design (not blinky) synthesised via
  yosys / nextpnr-himbaechel, placed & routed, flashed, and producing
  observable output on real hardware; programmer flow confirmed repeatable.
- **Gate B** (~wk 8): the SWIR camera streams pixels into BRAM, readable back
  over UART / VGA, at a measured (not assumed) sustained rate.
- **Kill:** Gate A missed by wk 6, or Gate B by wk 9 → live integration (M4) is
  descoped to a stretch goal; the effort runs on M2 + M3.
- Before burning solo weeks: check GateMate / nextpnr-himbaechel community prior
  art (forums, Cologne Chip reference designs, example repos).

### M2 (= Track B) — Synthesis-only feasibility sweep
*Needs no working silicon — starts day one, in parallel with M1.*

- Push the **Route C RTL** (Config 7, plus its `K` / resolution / disparity-range
  / key-point-pre-gate variants) through yosys / nextpnr as **parameterised RTL**
  (`K`, resolution, disparity range, window size as synthesis-time generics),
  recording CPE utilisation, achievable clock, BRAM usage, and — critically —
  whether the `K`-plane Hamming bank closes timing at pixel rate on the young
  flow.
- **Gate C** (~wk 10): enough sweeps done to confirm the range sweep closes
  timing and to fix the feasible `K` / resolution envelope for M3 (or to return a
  kill finding).

### M3 (= Track C) — Implement & benchmark the feasible set
*Depends on M2's Gate C. Does not need M1 to have succeeded — runs against
stored test-vector frames even if bring-up has stalled.*

- Weeks ~10–20: build and benchmark the range-sweep configuration(s) against
  stored frames + known-distance SWIR targets, scored by the §5.1 criteria.
- **The approach's parameters (`K`, boolean vs zoned, key-point pre-gate y/n) are
  fixed here, and it is confirmed against the bar or killed** (§5.3).

### M4 (= Track D) — Live integration & real-hardware validation
*Depends on M1 Gate B and M3's selected approach. Stretch goal, not
load-bearing — per the fallback below.*

- Weeks ~20–26: integrate the selected approach with the live SWIR head
  end-to-end (capture, rectification, matching, alarm output), honouring the
  streaming constraint, and validate real fps + detection reliability on
  hardware.

### Write-up / hardening
- Weeks ~26–30: harden the selected approach; write up around the research
  question, finalised at that point around the selected approach
  (`thesis-proposal.md` §1, §7).

### Kill findings

Each is a legitimate stop, *provided the §5.2 kill line was written first*:

- **Rectification `k`** cannot be bounded small enough to leave BRAM room for
  the `K`-plane matcher and its line-buffer band, even at the minimum viable `K`
  and window size.
- **The range-sweep** does not clear the fps + false-negative gate in M2 / M3 —
  the `K`-plane Hamming bank will not close timing at pixel rate, or the
  false-negative rate at the band edge stays above the agreed target.
- **Two-camera ingest** cannot be made to stream — SerDes lane count and/or the
  two sensors cannot be line-locked without a frame buffer.
- **Sensing geometry, not the matcher, is the blocker** — a realistic baseline /
  focal length at ~600 fps cannot yield enough frames of warning (or usable
  disparity) for the stated detection range (§2.9, §3.1). This bounds what any
  matching approach could do and is diagnosable from geometry before M3.

### Academic fallback

If M1 stalls past its kill gates or M4 does not land: the thesis stands on
**M2 + M3 alone** — a synthesis / simulation-only feasibility study and
approach selection, benchmarked against stored vectors. Detail and the
required advisor pre-approval: `thesis-proposal.md` §5.

---

## 7. Research framing (deferred)

The engineering approach is decided (§4, 2026-09-09 — the band-limited
disparity-range sweep). The **research question in `thesis-proposal.md` §1 is
still kept provisional**, to be finalised with the advisor; it now narrows around
this approach specifically — the feasibility and characterisation of a
band-limited plane-sweep trigger on this platform (how small `K` can go, how far
the output reduces, what false-negative rate the part + sensor deliver) — rather
than "which of three routes wins." The platform — an open-toolchain, DSP-less
FPGA with ~160 KB of BRAM, plus a SWIR sensor, with no prior art in any surveyed
paper — makes the contribution defensible regardless, so hardening the exact
wording is deliberately deferred.

`thesis-proposal.md` holds: the provisional research question (§1), the deferred
contribution-type discussion (§1.1 + Appendix B), the hardware / bandwidth
derivation (§2–§3), the research-method description of the tracks (§4), the
fallback thesis (§5), the open-items list (§7), and the chronological project
decision log (§8).

**Before / with the advisor:** finalise `thesis-proposal.md` §1 / §1.1 around the
decided approach (the range sweep), and record it in `thesis-proposal.md` §8 and
in §8 below. M3 supplies the numbers that make the wording concrete, not the
choice of approach.

---

## 8. Decisions & status

Project-level decisions log. The full chronological *project* log (research
framing, hardware verification, council sessions) lives in `thesis-proposal.md`
§8 — this section does not duplicate it, only records project-level calls and
rolls up the open items.

- **2026-09-03:** Effort re-prioritised around the engineering goal (founder
  direction). This document created as the primary driver; `thesis-proposal.md`
  set as the academic artefact developed alongside it. Output taken as a
  **near/far distance-threshold alarm** (boolean or zoned), not a distance
  measurement and not a dense depth map. Search scope fixed as **passive SWIR
  stereo, any matching style**. Hardware confirmed fully fixed (GateMate A1 +
  Olimex EVB + QDI SWIR). The engineering approach is chosen by the M2 / M3
  search (§4–§5); the research framing is finalised around the winner (§7).
- **2026-09-07:** Framing revised to match the proposal-defense deck
  (`../presentations/proposal-defense/deck.md`).
  - **Output granularity re-opened.** The goal now leads as *close-range
    detection within a configurable band* (§1); the output *form* — dense
    disparity map, sparse/semi-dense, or a bare boolean/zoned flag — is an
    outcome of the §4 approach search scored by §5 (criterion 9), **not fixed
    here**. The
    earlier "explicitly not a dense depth map" exclusion is dropped — a dense
    map is Route A. The boolean/zoned spec (§2.1–§2.3) stays as the *minimum*
    output and the reliability bar (§2.5) binds every route regardless of
    granularity.
  - **Three peer routes.** The candidate space is organised as Route A (dense
    disparity map), Route B (bounded sparse key-point), Route C (band-limited
    plane-sweep / disparity-threshold trigger) — presented without a ranking
    (§4.1). `implementation-plan.md` §6 gains **Config 7** for Route C, folded
    in from `../stereo_camera_fpga/design/CLAUDE.md`.
  - **Application constraints folded in** (§3.1, §2.7): moving platform / no
    static background model, closing speed up to ~600 m/s, anything-near-is-valid,
    purely passive, and the textureless-object **UNKNOWN state**.
  - **Startup / commercial framing de-emphasised.** This document reads as the
    engineering plan for a research project with a concrete application goal;
    the 2026-09-03 decision history above is unchanged.
- **2026-09-08:** Monocular **TTC / optical-expansion looming** recorded as a
  **deferred side option** (§4.5) — not a route in the §4 search, not on the
  M1–M4 path. Rationale: it bolts onto the shared front end for near-zero
  incremental RTL and is testable entirely offline; the only larger extra step
  is IMU / ego-motion. Reviewed by `/llm-council` the same day: viable only as an
  optional OR-in channel; disqualified as a primary or mandatory-pre-filter
  trigger (focus-of-expansion degeneracy on the head-on case, ~3-frame regime,
  SWIR texture / blur). **Observability check done the same day**
  (`.../synthesis/time-to-contact-explained.md` §7): provisional
  pass-with-caveats — per-frame image expansion is supra-pixel for extended
  targets in the close band (not a kill), but blur ≈ the inter-frame step caps τ
  precision at ~15–35 %, so it stays a coarse OR-in flag only. Notes:
  `../stereo_camera_fpga/research/synthesis/time-to-contact-explained.md` §6–§7,
  `../stereo_camera_fpga/research/summaries/2026-09-08-monocular-depth-perception.md`,
  `../stereo_camera_fpga/research/summaries/2026-09-08-ttc-optical-flow-fpga.md`.
- **2026-09-09:** **Engineering approach decided (founder direction).** The
  open approach search over three peer routes is **closed**: the system will
  perform a **band-limited disparity-range sweep** — cost tested only at the `K`
  disparity planes of the watch band, gate / reduce / confirm, boolean-or-zoned
  readout; **no per-pixel disparity map, no sparse key-point set**. This is
  **Route C** (`implementation-plan.md` Config 7); the per-stage design is
  `../stereo_camera_fpga/design/CLAUDE.md`. Consequences applied here: §1–§2
  (output granularity settled — boolean/zoned, boolean-vs-zoned still open);
  §4 retitled and rewritten as the decision plus the Route A / Route B
  alternatives set aside (with the reasoning kept); §5 recast from "pick a
  winner" to "characterise and parameterise the chosen approach — `K`, boolean vs
  zoned, key-point pre-gate y/n — and confirm it against the bar or kill it";
  §5.1 criterion 9 (output-granularity-delivered) dropped as moot; §6 M2/M3 and
  the kill findings rewritten to bite the range sweep specifically; §7 notes the
  research question narrows around this approach but stays provisional pending the
  advisor. The proposal-defense deck (`../presentations/proposal-defense/`) is
  **frozen** — the defense was delivered 2026-09-08 and that folder is not edited
  further; the decision post-dates the meeting and is recorded in
  `../presentations/CLAUDE.md`. Point-in-time research/synthesis docs and council
  snapshots are left unedited.

### Open items (rolled up)

From §2.9:

- [ ] Electrical output form.
- [ ] Boolean vs zoned for v1.
- [ ] UNKNOWN-state flag + large-blind-region policy (§2.7).
- [ ] `Z_near` / `Z_far` band values and the real head's baseline `B`.
- [ ] Detection-range requirement → baseline / focal length / frame rate.
- [ ] Latency budget number.
- [ ] Quantitative false-negative target (before M3).
- [ ] Money-scene weighting.

From §5.5:

- [ ] Threshold-alarm quality bar quantified (`thesis-proposal.md` §7).
- [ ] SWIR ground-truth / benchmark plan chosen (`thesis-proposal.md` §7).
- [ ] Rectification `k` spec in hand, by ~week 4 (`thesis-proposal.md` §7,
      `../TODO.md` item 3).
- [ ] Dataset-plumbing path for M3 (`implementation-plan.md` §8).
- [ ] Detection-range vs. sensing geometry — is the standoff reachable at all.
- [ ] IMU / ego-motion available? — sets Route C's confirm stage.

Advisor-facing:

- [ ] Heads-up to the advisor that `thesis-proposal.md` §1 is now marked
      "provisional pending approach selection" — so it reads as sequencing, not
      backsliding.
