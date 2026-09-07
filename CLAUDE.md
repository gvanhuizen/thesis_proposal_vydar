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
  framing revised 2026-09-07 to match the proposal-defense deck). Owns the goal
  (real-time close-range detection on the GateMate A1 + passive SWIR stereo head
  — decide whether an object is within a configurable distance band; not a
  distance measurement), the output specification and the **open granularity
  question** (dense map / sparse / bare boolean-or-zoned flag — decided by the
  search, not fixed), the fixed-hardware envelope and the moving-platform /
  ~600 m/s / purely-passive application constraints, the open engineering-approach
  search and the concrete method for choosing a winner, and the 30-week
  milestone / gate / kill-finding timeline (M1–M4). Everything else in this repo
  serves it.
- **`thesis-proposal.md`** — the **academic artefact**: the research framing
  wrapped around the product, cross-linked to `product-plan.md` and finalised
  once the approach search picks a winner. Holds the (currently provisional)
  research question and contribution-type discussion (§1, §1.1, Appendix B), the
  decided hardware/sensor envelope (§2), the bandwidth-arithmetic derivation
  with the `VERIFIED` / `VENDOR-CONFIRMED` / `ESTIMATED` legend (§3), the
  research-method rationale for the track structure (§4), the fallback thesis
  (§5), the open-items list (§7), and the chronological project decision log
  (§8). It used to be two files (`thesis-proposal.md` + `thesis-proposal2.md`);
  they were merged 2026-09-03. Camera-rig detail is not duplicated here — it
  lives in `../stereo_camera/`.
- **`implementation-plan.md`** — the **RTL menu the approach search evaluates**:
  a consolidated spec sheet, the constraint → viable-technique chain, and the
  candidate configurations with the reasoning for picks and rejections. As of
  2026-09-07 the menu is grouped into **three peer routes** (`product-plan.md`
  §4.1): **Route A** dense disparity map — Config 1 Census + fixed window,
  Config 2 SAD, Config 3 AD-Census + cross-based aggregation, Config 4 semi-dense
  seed-and-grow, Config 6 (optional) dual-path SGM; **Route B** — Config 5
  bounded sparse key-point; **Route C** — Config 7 band-limited plane-sweep /
  disparity-threshold trigger (folded in from
  `../stereo_camera_fpga/design/CLAUDE.md`). Draft, council-reviewed 2026-09-01,
  re-scoped 2026-09-03, reorganised into routes 2026-09-07. It does not pick the
  winner — `product-plan.md` §4–§5 does.

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
  Authoritative for: the goal, the output specification and the open granularity
  question, the moving-platform / ~600 m/s / purely-passive application
  constraints, the approach-search candidate menu (three peer routes) and the
  selection method, the milestone / gate timeline (M1–M4), and the kill
  findings. **Start here.**
- `thesis-proposal.md` in this folder is the **academic artefact**, driven by
  and cross-linked to `product-plan.md`. Authoritative for: the provisional
  research question and contribution type, hardware/sensor decisions, the
  bandwidth arithmetic *with the verification legend*, the research-method
  rationale for the tracks, the fallback thesis, and the chronological decision
  log (§8). Update research-framing detail here; update product / approach /
  schedule detail in `product-plan.md`.
- `implementation-plan.md` in this folder is authoritative for the **RTL config
  menu** the `product-plan.md` approach search evaluates — which techniques get
  built and why.
- `../stereo_camera_fpga/research/synthesis/hardware-and-technique-comparison.md`
  is the neutral GateMate-A1-vs-surveyed-literature reference matrix that both
  docs above draw on. `../stereo_camera_fpga/research/` holds the paper surveys
  (`summaries/`) and the other synthesis docs; `../stereo_camera_fpga/council/`
  holds the `/llm-council` deliberation snapshots referenced in the decision log.
- `../stereo_camera/README.md` and `../stereo_camera/literature/NOTES.md`
  remain authoritative for the camera-rig work and baseline/accuracy
  literature — that half's detail is not copied into `thesis-proposal.md`.

## Current framing (keep in sync with `product-plan.md` + `thesis-proposal.md`)

**Goal (as of 2026-09-07, matching the proposal-defense deck):** real-time
**close-range detection** on the GateMate A1 + passive SWIR stereo head — decide,
at the ~600 fps sensor rate, whether an object has come within a configurable
distance band. Missing a close object is the failure that matters most. The
*minimum* output is a boolean or zoned flag; the output **granularity** — dense
disparity map, sparse/semi-dense, or a bare flag — is **not fixed**, it is an
outcome of the approach search (`product-plan.md` §5.3). Authoritative in
`product-plan.md` §1–§2.

The matching **style** — and with it the granularity — is an open search over
**three peer routes** (no ranking): **Route A** a dense disparity map, **Route B**
a bounded sparse key-point matcher, **Route C** a band-limited plane-sweep /
disparity-threshold trigger. The candidate menu and how a winner is picked are
`product-plan.md` §4–§5; the RTL detail (Configs 1–7) is `implementation-plan.md`
§6.

The sensor rides a **moving platform** (no static background model), closing
speeds up to **~600 m/s**, **purely passive**, anything-near-is-a-valid-trigger,
with a textureless-object **UNKNOWN state** — folded into `product-plan.md` §3.1 /
§2.7 on 2026-09-07 (from `../stereo_camera_fpga/design/CLAUDE.md`).

`thesis-proposal.md` §1 carries a **provisional** research question framed
around that goal, to be finalised once the approach search (M3) selects an
approach; the "genuine research contribution" rests on the novel platform +
sensor, not on characterising an abstract cost × aggregation × resolution
surface (that framing was dropped 2026-09-03 — see the proposal's decision log).

FPGA stereo processing (`stereo_camera_fpga/`) is the expected primary
research contribution; the stereo-vision camera rig (`stereo_camera/`) is
supporting/practical work. This is the founder's current best guess, not
settled scope — when it changes (e.g. after M2/M3 results land, or after advisor
feedback), update this section, `product-plan.md`, and the header + §0 of
`thesis-proposal.md` together so they don't drift apart.
