# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a small, standalone git repo whose only job is to hold
`thesis-proposal.md`, the **umbrella scope document** for a master's thesis
built from two separate implementations that live in sibling directories on
disk, each their own git repo:

- `../stereo_camera/` (github.com/gvanhuizen/stereo_camera)
- `../fpga_stereo_camera/` (github.com/gvanhuizen/stereo_camera_fpga)

There is no source code here, no build/lint/test tooling, and none is
expected — this folder exists purely to state that those two projects are
one thesis with one scope, and to say where the real detail for each half
lives.

## Directory-layout dependency

`thesis-proposal.md`'s links (`../stereo_camera/...`,
`../fpga_stereo_camera/...`) are relative paths that assume the three repos
stay siblings under the same parent directory (currently `Vydar/`). If this
repo is ever cloned or moved on its own, those links break — they are not
git submodules and there's no fetch/resolve mechanism, just relative
filesystem paths.

## What to read, and where things are authoritative

- `thesis-proposal.md` in this folder is the one living document to read
  and update *here* — it covers only the combined framing (which half is
  primary, how they relate, cross-cutting open items) and links out for
  everything else.
- `../fpga_stereo_camera/thesis_proposal/thesis-proposal.md` remains the
  **authoritative detailed plan for the FPGA track** — research question,
  hardware/sensor decisions, bandwidth arithmetic, Track A–D structure,
  gates/kill-criteria, fallback thesis, decision log. Update FPGA-track
  detail there, not here; don't let a duplicate copy accumulate in this
  repo.
- `../stereo_camera/README.md` and `../stereo_camera/literature/NOTES.md`
  remain authoritative for the camera-rig work and baseline/accuracy
  literature.

## Current framing (keep in sync with `thesis-proposal.md`)

FPGA stereo processing (`fpga_stereo_camera/`) is the expected primary
research contribution; the stereo-vision camera rig (`stereo_camera/`) is
supporting/practical work. This is the founder's current best guess, not
settled scope — when it changes (e.g. after FPGA Track B/C results land, or
after advisor feedback), update both this section and the "Framing" section
of `thesis-proposal.md` together so they don't drift apart.
