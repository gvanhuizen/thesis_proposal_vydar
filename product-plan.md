# Product Plan: High-Speed Near/Far Distance-Threshold Detection on the GateMate A1

**Status:** primary driver document — active. Created 2026-09-03 when the product
was re-prioritised as the organising goal (see `thesis-proposal.md` §8 decision
log).
**Last updated:** 2026-09-03
**Clock:** 30 weeks, shared with the master's thesis.

This is the driver document for the whole effort. The product is the point: a
working near/far distance-threshold detector on the fixed hardware below. The
research framing (`thesis-proposal.md`) is a lightweight wrapper applied to
whichever engineering approach the search in §4–§5 selects — it does not drive
the work, and it is deliberately not being settled now.

---

## 0. What this document is, and how the docs relate

- **`product-plan.md`** (this file) — the driver and entry point. Owns: the
  product goal (§1), the near/far threshold-alarm output spec (§2), the
  fixed-hardware envelope in brief (§3), the open engineering-approach search
  (§4), the concrete method for choosing a winner (§5), the milestone / gate /
  kill-finding timeline (§6), and the product-level decisions log (§8).
- **`thesis-proposal.md`** (this folder) — the academic artefact, wrapped around
  the chosen approach and cross-linked here. Authoritative for: the decided
  hardware / sensor spec (§2), the bandwidth-arithmetic derivation with the
  `VERIFIED` / `VENDOR-CONFIRMED` / `ESTIMATED` verification legend (§3), the
  research-method description of the tracks (§4), the fallback thesis (§5), the
  open-items list with owners and dates (§7), and the chronological project
  decision log (§8). The research question (§1) is provisional until the
  approach search lands.
- **`implementation-plan.md`** (this folder) — the RTL menu the approach search
  evaluates: consolidated spec sheet (§2), constraint → viable-technique chain
  (§5), the candidate configurations with picks and rejections (§6–§7). It does
  not decide which approach wins; §4–§5 here do.
- **`../stereo_camera_fpga/`** — the FPGA build workspace (no RTL yet).
  **`../stereo_camera/`** — the supporting Raspberry Pi stereo rig.
  **`../stereo_camera_fpga/research/synthesis/`** — the neutral technique /
  device reference matrices.

**Directory-layout dependency:** the relative links in this file assume the four
repos stay siblings under one parent (`thesis/`). They are not git submodules;
if this repo is moved on its own, the `../stereo_camera_fpga/...` links break.

---

## 1. Product goal

A working product for **high-speed near/far distance detection** on the Cologne
Chip GateMate A1 FPGA fed by a **passive SWIR (InGaAs) stereo head**: at the
~600 fps sensor rate, decide whether an object has entered a configurable
distance band and raise an alarm.

**"Working v1" means:** a reliable alarm edge at the configured band, produced at
sensor frame rate, demonstrated on real hardware — or, if hardware bring-up
slips past its kill gate, in synthesis + simulation against stored frames (the
M2 + M3 fallback, §6).

**Explicitly not:**

- a distance *measurement* (no metric range output);
- a dense per-pixel depth map;
- active illumination or a projected pattern;
- monocular / single-camera depth.

Passive SWIR stereo is fixed. The matching **style** — dense, semi-dense,
sparse, key-point, flow-assisted — is an open search (§4).

**Why this hardware is worth the constraints:**

- Fully open toolchain (yosys + nextpnr-himbaechel + openFPGALoader) — ~$100 of
  hardware, reproducible end-to-end, and a citable "first" in the FOSS-EDA
  space.
- SWIR (~1–1.7 µm) sees through fog / haze and works in low light and behind
  some materials where visible-light stereo is blind — the scenes the product
  exists for.
- No prior real-time FPGA stereo work runs on a device this small / this shape,
  and none uses a SWIR sensor — so the research wrapper is easy to attach later
  whatever the approach turns out to be (§7).

Decided hardware / sensor spec: `thesis-proposal.md` §2.

---

## 2. Output specification — the near/far threshold alarm

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

The scenes the product must hold up in — fog / haze, glass, low light — and the
minimum valid-match density / confidence needed for the near/far decision to be
trustworthy in them. A near-free per-pixel confidence gate (reject matches whose
local gradient energy is below a threshold rather than emitting a garbage
disparity) is assumed available to every candidate approach.

### 2.7 What alarm-only output buys

This is the lever the whole product-first reframe rests on:

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

### 2.8 Open items

- [ ] **Electrical output form** — GPIO / PMOD logic line, UART message, VGA
      overlay, or several. Drives the control-plane design and whether egress
      needs anything beyond VGA.
- [ ] **Boolean vs zoned for v1** — is a single near/far boolean enough, or is a
      small number of zones a hard requirement.
- [ ] **Band values and the real head's baseline** — `Z_near` / `Z_far` in
      metres, and `B` for the *real* product head (the Pi rig's geometry is
      moot — throwaway scaffolding).
- [ ] **Latency budget number** (§2.4).
- [ ] **Quantitative false-negative target** (§2.5), agreed before M3.
- [ ] **Money-scene weighting** — fog dominant, or fog / glass / low-light
      equally weighted; sets how heavily SWIR-texture robustness is weighted in
      selection (§5).

---

## 3. Fixed hardware envelope (brief — detail is elsewhere)

- **FPGA:** Cologne Chip GateMate A1 (CCGM1A1) on an Olimex GateMateA1-EVB —
  20,480 CPEs, no hardened DSP/MAC, VGA-only egress, no hardened CPU.
- **Sensor:** QDI Systems SWIR (InGaAs) camera, 640×512, 14-bit, ~600–700 fps,
  Camera-Link-style LVDS.
- **Toolchain:** yosys + nextpnr-himbaechel + openFPGALoader, fully open-source.

Hard constraints that bound *every* candidate approach:

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

## 4. The approach search — options to evaluate

Passive SWIR stereo is fixed; the matching **style** is open, and **none of the
options below is privileged**. The job of M2 / M3 (§6) is to find the most
promising engineering approach for the §2 output on the §3 hardware, scored by
§5. The research wrapper is attached to the winner afterward (§7).

### 4.1 Candidate menu

Full RTL detail for each is in `implementation-plan.md` §6.

| Approach | What it is | A1 fit | Main risk |
|---|---|---|---|
| **Dense Census + fixed window** | Census transform, Hamming cost, W×W window, winner-take-all | Hamming = XOR + popcount, LUT-native; lowest timing-closure risk of any real matcher | Edge-fattening at depth discontinuities |
| **Dense SAD + box-filter moving sum** | SAD over a fixed window, O(1) window-sum update, WTA | Simplest arithmetic (subtract / abs / add on the carry chains); no serial dependency | Radiometrically fragile between two SWIR cameras with any gain/offset mismatch |
| **AD-Census + cross-based aggregation** | Capped AD + Census cost, adaptive "+"-shaped support from 4 arm registers | Small extra logic on Census; cheap in registers, no big buffer | Adds a data-dependent stage; needs a synthesis run to confirm it stays cheap |
| **Semi-dense seed-and-grow / ELAS-style** | Confident Census/Hamming seeds → guided fill along gradients, fan-out bounded on fabric | Emits far less than a dense map; reintroduces a smoothness prior cheaply | Bounding the growth on fabric without the data-dependent tail; every surveyed system used an ARM core to grow |
| **Bounded sparse key-point** | Single-scale FAST/Harris + BRIEF or reused Census word + 1-D along-row Hamming; hard cap + spatial buckets + top-K; deterministic LRC + ordering + fixed epipolar-offset + parabola sub-pixel; **no RANSAC / triangulation / scatter-gather** | Lowest output bandwidth; reuses the Census datapath; the band decision needs a bounded set of near matches, not a point cloud | SWIR texture starvation → too few / too-weak key-points in exactly the fog / glass / low-light scenes the product targets |
| **Dual-path (H+V) SGM via dependency-relaxation** | H + V path aggregation only, recursion reading n pixels back, datapath replicated across n PUs | Avoids the width-scaling diagonal buffers and the wide comparator tree | Quantified accuracy cost +0.12 disparity error / +1.96 % bad-pixel per PU; only if resource headroom remains |
| **Optical flow as an assist** | Frame-to-frame key-point tracking to amortise detection | Bounded if track count + iterations are capped | Orthogonal — lowers per-frame detect cost, does not itself produce disparity |

For a threshold alarm the dense entries above can run as a **single / few-plane
occupancy test** — cost evaluated only at the band's disparity plane(s), not an
`argmin` over `0..D` — which collapses the disparity search and the on-chip
state. The head-to-head of that dense form against the bounded key-point form,
with techniques and FPGA literature, is
`../stereo_camera_fpga/research/synthesis/dense-vs-keypoint-for-threshold-alarm.md`
(its recommendation: dense few-plane primary, bounded key-point as the
M2-characterised challenger, decide in M3 on measured false-negative rate +
fmax + BRAM).

### 4.2 Structurally excluded

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

### 4.3 What the threshold-alarm output changes per family

Every family benefits from §2.7: the single-plane band test collapses the
disparity search, there is no dense-map write, and density is free. The
semi-dense and bounded-sparse rows shrink the BRAM row budget most (leaving more
headroom for the rectification `k`, §3). The counter-pressure from SWIR texture
starvation (§2.7) is heaviest on the pure-local dense and bounded-sparse rows
and lightest on the semi-dense middle.

---

## 5. Selection method — how the winner is picked

### 5.1 Evaluation criteria

All driven by the §2 alarm output:

1. **Detection reliability at the band edge** — correct near/far call vs true
   stand-off on known-distance targets.
2. **False-negative rate on close objects** — the safety-critical failure
   (§2.5); weighted highest.
3. **False-positive (nuisance) rate.**
4. **Sustained fps at the sensor rate.**
5. **Resource fit** — CPE utilisation, BRAM against the ~160 KB line
   (after the rectification `k` budget), residual PSRAM traffic.
6. **Timing-closure risk / confidence on `nextpnr-himbaechel`** — the real
   feasibility gate (§3).
7. **SWIR-texture robustness** in fog / glass / low-light, reported spatially.
8. **End-to-end latency** vs the §2.4 budget.

### 5.2 Scoring

Criterion 2 (false-negative rate on close objects) is a **hard gate** — a
candidate that cannot clear the agreed target is out regardless of its other
numbers. The remaining criteria are a weighted comparison, with the §2.8
money-scene weighting setting how heavily criterion 7 counts.

**Write the kill line before M3:** the minimum fps + minimum detection
reliability below which the product story collapses, agreed with the founder /
advisor. Without it up front, the pressure to flatter a weak result is
irresistible. A defensible negative outcome — "no candidate approach clears the
bar on this part + this sensor" — is only honest if the bar predates the
result.

### 5.3 How the milestones feed the pick

- **M2** (synthesis sweep) → the feasible set: which candidates close timing at
  pixel rate and fit BRAM (Gate C, §6).
- **M3** (implement + benchmark the feasible set against stored frames) →
  the §5.1 numbers → **the approach is selected here.**
- **M4** (live integration) → validation of the selected approach on real
  hardware, not a re-open of the choice.

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
      `../TODO.md` item 4).
- [ ] **Dataset-plumbing path for M3** — how stored frames reach the device
      (UART / flash / streamed to mimic the camera) (`implementation-plan.md`
      §8).

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
  descoped to a stretch goal; the product / thesis run on M2 + M3.
- Before burning solo weeks: check GateMate / nextpnr-himbaechel community prior
  art (forums, Cologne Chip reference designs, example repos).

### M2 (= Track B) — Synthesis-only feasibility sweep
*Needs no working silicon — starts day one, in parallel with M1.*

- Push the §4 candidate menu through yosys / nextpnr as **parameterised RTL**
  (resolution, disparity range, window size as synthesis-time generics),
  recording CPE utilisation, achievable clock, BRAM usage, and — critically —
  whether timing closes at pixel rate on the young flow, for each.
- **Gate C** (~wk 10): enough sweeps done to name the 2–3 feasible
  configurations to carry into M3.

### M3 (= Track C) — Implement & benchmark the feasible set
*Depends on M2's Gate C. Does not need M1 to have succeeded — runs against
stored test-vector frames even if bring-up has stalled.*

- Weeks ~10–20: build and benchmark the feasible configurations against stored
  frames + known-distance SWIR targets, scored by the §5.1 criteria.
- **The approach is selected here** (§5.3).

### M4 (= Track D) — Live integration & real-hardware validation
*Depends on M1 Gate B and M3's selected approach. Stretch goal, not
load-bearing — per the fallback below.*

- Weeks ~20–26: integrate the selected approach with the live SWIR head
  end-to-end (capture, rectification, matching, alarm output), honouring the
  streaming constraint, and validate real fps + detection reliability on
  hardware.

### Write-up / product hardening
- Weeks ~26–30: harden the selected approach; write up around the research
  question, finalised at that point around the selected approach
  (`thesis-proposal.md` §1, §7).

### Kill findings

Each is a legitimate stop, *provided the §5.2 kill line was written first*:

- **Rectification `k`** cannot be bounded small enough to leave BRAM room for
  the matching window `W` and disparity buffers `D` for *any* candidate
  approach (including the sparse / semi-dense ones, which need fewer rows).
- **No candidate** clears the fps + false-negative gate in M2 / M3.
- **Two-camera ingest** cannot be made to stream — SerDes lane count and/or the
  two sensors cannot be line-locked without a frame buffer.

### Academic fallback

If M1 stalls past its kill gates or M4 does not land: the thesis stands on
**M2 + M3 alone** — a synthesis / simulation-only feasibility study and
approach selection, benchmarked against stored vectors. Detail and the
required advisor pre-approval: `thesis-proposal.md` §5.

---

## 7. Research framing (deferred)

The research wrapper is written around whichever approach M3 selects. The
platform — an open-toolchain, DSP-less FPGA with ~160 KB of BRAM, plus a SWIR
sensor, with no prior art in any surveyed paper — makes a defensible research
contribution easy to frame for *any* of the §4 candidates, so this is
deliberately not being solved now.

`thesis-proposal.md` holds: the provisional research question (§1), the deferred
contribution-type discussion (§1.1 + Appendix B), the hardware / bandwidth
derivation (§2–§3), the research-method description of the tracks (§4), the
fallback thesis (§5), the open-items list (§7), and the chronological project
decision log (§8).

**When M3 lands:** finalise `thesis-proposal.md` §1 / §1.1 around the selected
approach, and record it in `thesis-proposal.md` §8 and in §8 below.

---

## 8. Decisions & status

Product-level decisions log. The full chronological *project* log (research
framing, hardware verification, council sessions) lives in `thesis-proposal.md`
§8 — this section does not duplicate it, only records product-level calls and
rolls up the open items.

- **2026-09-03:** Product re-prioritised as the organising goal (founder
  direction). This document created as the primary driver; `thesis-proposal.md`
  demoted to the academic artefact wrapped around it. Product output fixed as a
  **near/far distance-threshold alarm** (boolean or zoned), not a distance
  measurement and not a dense depth map. Search scope fixed as **passive SWIR
  stereo, any matching style**. Hardware confirmed fully fixed (GateMate A1 +
  Olimex EVB + QDI SWIR). The engineering approach is chosen by the M2 / M3
  search (§4–§5); the research framing is deferred to the winner (§7).

### Open items (rolled up)

From §2.8:

- [ ] Electrical output form.
- [ ] Boolean vs zoned for v1.
- [ ] `Z_near` / `Z_far` band values and the real head's baseline `B`.
- [ ] Latency budget number.
- [ ] Quantitative false-negative target (before M3).
- [ ] Money-scene weighting.

From §5.5:

- [ ] Threshold-alarm quality bar quantified (`thesis-proposal.md` §7).
- [ ] SWIR ground-truth / benchmark plan chosen (`thesis-proposal.md` §7).
- [ ] Rectification `k` spec in hand, by ~week 4 (`thesis-proposal.md` §7,
      `../TODO.md` item 4).
- [ ] Dataset-plumbing path for M3 (`implementation-plan.md` §8).

Advisor-facing:

- [ ] Heads-up to the advisor that `thesis-proposal.md` §1 is now marked
      "provisional pending approach selection" — so it reads as sequencing, not
      backsliding.
