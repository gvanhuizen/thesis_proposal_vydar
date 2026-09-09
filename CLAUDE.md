# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a small, standalone git repo holding the thesis planning documents for a
master's thesis built from two separate implementations that live in sibling
directories on disk, each their own git repo:

- `../stereo_camera/` (github.com/gvanhuizen/stereo_camera)
- `../stereo_camera_fpga/` (github.com/gvanhuizen/stereo_camera_fpga)

There is no source code here, no build/lint/test tooling, and none is expected.
Three tracked files:

- **`product-plan.md`** — the **primary driver document** (created 2026-09-03;
  framing revised 2026-09-07 to match the proposal-defense deck; engineering
  approach **decided 2026-09-09**). Owns the goal (real-time close-range detection
  on the GateMate A1 + passive SWIR stereo head — decide whether an object is
  within a configurable distance band; not a distance measurement), the output
  specification (**decided 2026-09-09**: a boolean-or-zoned range-threshold flag,
  no dense map, no sparse key-point set — boolean-vs-zoned for v1 still open), the
  fixed-hardware envelope and the moving-platform / ~600 m/s / purely-passive
  application constraints, the **decided engineering approach** (a band-limited
  disparity-range sweep = Route C / `implementation-plan.md` Config 7) with the
  Route A / B alternatives recorded as set aside, what M2/M3 still characterise
  (`K`, boolean-vs-zoned, key-point pre-gate), and the 30-week milestone / gate /
  kill-finding timeline (M1–M4). Everything else in this repo serves it.
- **`thesis-proposal.md`** — the **academic artefact**: the research framing
  wrapped around the product, cross-linked to `product-plan.md`. The engineering
  approach it wraps is decided (2026-09-09, the range sweep); the research
  question (§1) is **kept provisional pending the advisor**. Holds the (currently
  provisional)
  research question and contribution-type discussion (§1, §1.1, Appendix B), the
  decided hardware/sensor envelope (§2), the bandwidth-arithmetic derivation
  with the `VERIFIED` / `VENDOR-CONFIRMED` / `ESTIMATED` legend (§3), the
  research-method rationale for the track structure (§4), the fallback thesis
  (§5), the open-items list (§7), and the chronological project decision log
  (§8). It used to be two files (`thesis-proposal.md` + `thesis-proposal2.md`);
  they were merged 2026-09-03. Camera-rig detail is not duplicated here — it
  lives in `../stereo_camera/`.
- **`implementation-plan.md`** — the **RTL detail**: a consolidated spec sheet,
  the constraint → viable-technique chain, and the configurations with the
  reasoning and rejections. **As of 2026-09-09 the build target is Config 7**
  (Route C — band-limited plane-sweep / disparity-threshold trigger = the decided
  disparity-range sweep; per-stage design in
  `../stereo_camera_fpga/design/CLAUDE.md`). Configs 1–6 (Route A dense forms +
  Config 5 Route B bounded sparse key-point) are retained as the alternatives
  considered and set aside, and as M2 comparison points. Draft, council-reviewed
  2026-09-01, re-scoped 2026-09-03, reorganised into routes 2026-09-07, approach
  decided 2026-09-09. The build target is set by `product-plan.md` §4.

Also present, **not authoritative** — point-in-time prep artifacts (like the
`../stereo_camera_fpga/council/` snapshots), safe to supersede:

- **`advisor-sync-2026-09.md`** — debrief agenda for the proposal defense: the
  two before-week-1 sign-offs (fallback thesis, combined scope), the three deck
  questions, and `PROPOSED` starting values for the four pre-M3 quantitative bars
  (latency budget, min fps, detection-reliability floor + close-object FN target,
  money-scene weighting). Agreed items fold back into `product-plan.md` §2.4 /
  §2.5 / §2.9 / §5.2 and `thesis-proposal.md` §7 / §8.

## Directory-layout dependency

The relative links in `product-plan.md`, `thesis-proposal.md` and
`implementation-plan.md` (`../stereo_camera/...`, `../stereo_camera_fpga/...`)
assume the repos stay siblings under the same parent directory (currently
`thesis/`). If this repo is ever cloned or moved on its own, those links break —
they are not git submodules and there's no fetch/resolve mechanism, just
relative filesystem paths.

## What to read, and where things are authoritative

- `product-plan.md` in this folder is **the entry point and the driver**.
  Authoritative for: the goal, the output specification (decided 2026-09-09 —
  boolean/zoned flag), the moving-platform / ~600 m/s / purely-passive application
  constraints, the **decided engineering approach** (§4 — a band-limited
  disparity-range sweep = Route C / Config 7) plus the Route A / B alternatives
  set aside, what M2/M3 still characterise (§5), the milestone / gate timeline
  (M1–M4), and the kill findings. **Start here.**
- `thesis-proposal.md` in this folder is the **academic artefact**, driven by
  and cross-linked to `product-plan.md`. Authoritative for: the provisional
  research question and contribution type, hardware/sensor decisions, the
  bandwidth arithmetic *with the verification legend*, the research-method
  rationale for the tracks, the fallback thesis, and the chronological decision
  log (§8). Update research-framing detail here; update product / approach /
  schedule detail in `product-plan.md`.
- `implementation-plan.md` in this folder is authoritative for the **RTL detail**
  — the build target is Config 7 (Route C, the decided range sweep); Configs 1–6
  are the set-aside alternatives.
- `../stereo_camera_fpga/research/synthesis/hardware-and-technique-comparison.md`
  is the neutral GateMate-A1-vs-surveyed-literature reference matrix that both
  docs above draw on. `../stereo_camera_fpga/research/` holds the paper surveys
  (`summaries/`) and the other synthesis docs; `../stereo_camera_fpga/council/`
  holds the `/llm-council` deliberation snapshots referenced in the decision log.
- `../stereo_camera/README.md` and `../stereo_camera/literature/NOTES.md`
  remain authoritative for the camera-rig work and baseline/accuracy
  literature — that half's detail is not copied into `thesis-proposal.md`.
- `../design_process/` (sibling repo, created 2026-09-09) recasts the FPGA effort
  as a formal design process — one Markdown file per design-flow step (problem
  spec → algorithm development → architecture selection → system implementation →
  testing & debugging), after Bailey, *Design for Embedded Image Processing on
  FPGAs*, Ch. 3. It **digests** `product-plan.md` / `thesis-proposal.md` /
  `implementation-plan.md` under that structure and links back; it is authoritative
  for nothing. When a decision in these three docs moves, the matching step file
  there needs the same update.

## Current framing (keep in sync with `product-plan.md` + `thesis-proposal.md`)

**Goal (unchanged):** real-time **close-range detection** on the GateMate A1 +
passive SWIR stereo head — decide, at the ~600 fps sensor rate, whether an object
has come within a configurable distance band. Missing a close object is the
failure that matters most. Authoritative in `product-plan.md` §1–§2.

**Output (decided 2026-09-09):** a **boolean-or-zoned range-threshold flag**
(optionally carrying the surviving plane index) — **no dense disparity map, no
sparse key-point set**. Boolean-vs-zoned for v1 is the one open sub-choice.

**Engineering approach (decided 2026-09-09):** a **band-limited disparity-range
sweep** — Census transform → Hamming cost only at the `K` disparity planes of the
watch band → per-pixel confidence gates → spatial reduction → two-frame Δd
confirm → boolean/zoned readout; never `argmin` over `0..D`, never a map. This is
the route earlier called **Route C**; its RTL form is `implementation-plan.md`
**Config 7** and its per-stage design is `../stereo_camera_fpga/design/CLAUDE.md`.
The earlier open search over three peer routes (Route A dense disparity map,
Route B bounded sparse key-point, Route C plane-sweep trigger) is **closed** —
`product-plan.md` §4 keeps A and B as the recorded alternatives. What M2/M3 still
fix: `K`, boolean-vs-zoned, and whether a key-point pre-gate earns its place
(`product-plan.md` §5).

The sensor rides a **moving platform** (no static background model), closing
speeds up to **~600 m/s**, **purely passive**, anything-near-is-a-valid-trigger,
with a textureless-object **UNKNOWN state** — folded into `product-plan.md` §3.1 /
§2.7 on 2026-09-07 (from `../stereo_camera_fpga/design/CLAUDE.md`).

`thesis-proposal.md` §1 carries a **provisional** research question framed
around that goal, **kept provisional pending the advisor** even though the
engineering approach is now decided; it now narrows around the range sweep
specifically (feasibility + characterisation of a band-limited plane-sweep
trigger on this platform). The "genuine research contribution" rests on the novel
platform + sensor, not on characterising an abstract cost × aggregation ×
resolution
surface (that framing was dropped 2026-09-03 — see the proposal's decision log).

FPGA stereo processing (`stereo_camera_fpga/`) is the expected primary
research contribution; the stereo-vision camera rig (`stereo_camera/`) is
supporting/practical work. This is the founder's current best guess, not
settled scope — when it changes (e.g. after M2/M3 results land, or after advisor
feedback), update this section, `product-plan.md`, and the header + §0 of
`thesis-proposal.md` together so they don't drift apart.
