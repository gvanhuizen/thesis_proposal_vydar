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

- **`product-plan.md`** — the **primary driver document** (created 2026-09-03
  when the product was made the organising goal). Owns the product goal
  (a high-speed near/far distance-threshold alarm on the GateMate A1 + passive
  SWIR stereo head — not a distance measurement, not a dense map), the
  threshold-alarm output spec, the open engineering-approach search and the
  concrete method for choosing a winner, and the 30-week milestone / gate /
  kill-finding timeline (M1–M4). Everything else in this repo serves it.
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
  2026-09-03 the menu spans the **density axis**: Config 1 Census + fixed
  window, Config 2 SAD, Config 3 AD-Census + cross-based aggregation, Config 4
  semi-dense seed-and-grow, Config 5 bounded sparse key-point, Config 6
  (optional) dual-path SGM. Draft, council-reviewed 2026-09-01, re-scoped
  2026-09-03. It does not pick the winner — `product-plan.md` §4–§5 does.

## Directory-layout dependency

The relative links in `product-plan.md`, `thesis-proposal.md` and
`implementation-plan.md` (`../stereo_camera/...`, `../stereo_camera_fpga/...`)
assume the repos stay siblings under the same parent directory (currently
`thesis/`). If this repo is ever cloned or moved on its own, those links break —
they are not git submodules and there's no fetch/resolve mechanism, just
relative filesystem paths.

## What to read, and where things are authoritative

- `product-plan.md` in this folder is **the entry point and the driver**.
  Authoritative for: the product goal, the near/far threshold-alarm output spec,
  the approach-search candidate menu and the selection method, the product
  milestone / gate timeline (M1–M4), and the kill findings. **Start here.**
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

**Product goal (the organising goal, as of 2026-09-03):** a high-speed near/far
**distance-threshold alarm** on the GateMate A1 + passive SWIR stereo head — a
boolean or zoned signal when an object enters a configurable distance band,
**not** a distance measurement and **not** a dense depth map. Authoritative in
`product-plan.md` §1–§2. The matching *style* (dense / semi-dense / sparse /
key-point / flow-assisted) is an open search — the candidate menu and how a
winner is picked are `product-plan.md` §4–§5; the RTL detail is
`implementation-plan.md` §6.

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
