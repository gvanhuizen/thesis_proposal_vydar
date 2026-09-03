# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a small, standalone git repo whose only job is to hold
`thesis-proposal.md`, the **single planning document** for a master's thesis
built from two separate implementations that live in sibling directories on
disk, each their own git repo:

- `../stereo_camera/` (github.com/gvanhuizen/stereo_camera)
- `../stereo_camera_fpga/` (github.com/gvanhuizen/stereo_camera_fpga)

There is no source code here, no build/lint/test tooling, and none is
expected. As of 2026-09-03 `thesis-proposal.md` holds **both** the
combined-scope framing (header + §0: which half is primary, how they relate)
**and** the full detailed FPGA-track plan (§1–§8). It used to be two files
(`thesis-proposal.md` + `thesis-proposal2.md`); they were merged. Detail for
the supporting camera-rig half is still not duplicated here — it lives in
`../stereo_camera/`.

(`implementation-plan.md`, also in this folder, is a separate untracked
working doc — not part of the proposal.)

## Directory-layout dependency

`thesis-proposal.md`'s links (`../stereo_camera/...`,
`../stereo_camera_fpga/...`) are relative paths that assume the three repos
stay siblings under the same parent directory (currently `thesis/`). If this
repo is ever cloned or moved on its own, those links break — they are not
git submodules and there's no fetch/resolve mechanism, just relative
filesystem paths.

## What to read, and where things are authoritative

- `thesis-proposal.md` in this folder is **the** authoritative planning
  document for the thesis — both the combined framing (header + §0) and the
  detailed FPGA-track plan (§1–§8: research question, hardware/sensor
  decisions, bandwidth arithmetic, Track A–D structure, gates/kill-criteria,
  fallback thesis, decision log). Update FPGA-track detail here.
- `../stereo_camera/README.md` and `../stereo_camera/literature/NOTES.md`
  remain authoritative for the camera-rig work and baseline/accuracy
  literature — that half's detail is not copied into `thesis-proposal.md`.

## Current framing (keep in sync with `thesis-proposal.md`)

FPGA stereo processing (`stereo_camera_fpga/`) is the expected primary
research contribution; the stereo-vision camera rig (`stereo_camera/`) is
supporting/practical work. This is the founder's current best guess, not
settled scope — when it changes (e.g. after FPGA Track B/C results land, or
after advisor feedback), update both this section and the header + §0 of
`thesis-proposal.md` together so they don't drift apart.
