# Thesis Proposal: Real-Time Stereo Depth — Combined Scope

**Status:** living planning doc, initial draft — not yet reviewed with advisor
**Last updated:** 2026-08-12

This document is the **umbrella scope document** for one master's thesis
built from two separate implementations tracked as separate git repos on
disk, siblings of this folder:

- `../fpga_stereo_camera/` — real-time stereo matching on a Cologne Chip
  GateMate A1 FPGA (DSP-less, open-toolchain). Expected to be the thesis's
  **primary research contribution**.
- `../stereo_camera/` — a working Rust + Python stereo depth rig on two
  Raspberry Pi cameras, plus a literature review on baseline selection and
  depth-accuracy error models. Currently framed as **supporting/practical
  work**, not the thing being defended.

That primary/supporting split is a **current best guess, not settled
scope** — revisit it as Track B/C results land in the FPGA plan (see
below), and update this section when it firms up or changes.

This doc does not duplicate either project's detail — it points to where
that detail actually lives and stays authoritative there, so there's one
place to update each thing, not two copies drifting apart.

## Where the detail lives

**FPGA implementation (primary):**

- `../fpga_stereo_camera/thesis_proposal/thesis-proposal.md` — the
  authoritative, detailed plan: research question, target hardware/sensor
  decisions, PSRAM bandwidth arithmetic, the Track A–D structure (parallel
  toolchain-bring-up and synthesis-only resource-characterization tracks,
  gates/kill-criteria, fallback thesis), open items, and a dated decision
  log. **Read that file, not this one, before touching FPGA-track scope or
  scheduling** — this doc only links to it.
- `../fpga_stereo_camera/CLAUDE.md` — working notes on the FPGA project's
  state and target hardware.
- `../fpga_stereo_camera/research/` — literature surveys (FPGA stereo
  pipelines, SAD/Census matching, GateMate A1 hardware) and
  `../fpga_stereo_camera/council/` — saved `/llm-council` deliberations that
  shaped the current plan.

**Stereo-vision HW rig (supporting):**

- `../stereo_camera/README.md` — the working pipeline (capture → rectify →
  match → depth estimate) and current baseline/accuracy results.
- `../stereo_camera/literature/NOTES.md` — literature review on how
  baseline length affects depth accuracy, range, and disparity resolution.

## How the two halves relate

Not yet fully defined — this is the main open question this document
exists to eventually answer, not to prematurely settle:

- The FPGA thesis's stereo-matching algorithm choices (SAD/Census family,
  per the FPGA research surveys) are sensor- and rig-agnostic in principle,
  so `stereo_camera/`'s baseline/error-model literature review is
  potentially directly reusable background for the FPGA thesis's own
  accuracy/disparity-quality bar (an open item in the FPGA plan, §7).
- `stereo_camera/`'s working calibration/rectification pipeline
  (Kalibr-based) is a proof of the same rectify-then-match architecture the
  FPGA implementation will also need, on different hardware — useful prior
  art, not code that transfers directly (different language, different
  camera, different resource constraints).
- Whether `stereo_camera/` results are cited as prior/parallel work for
  context, or actively reused as source material (e.g., its baseline
  literature review folded into the FPGA thesis's own accuracy-bar
  decision) is **not yet decided** — see open items.

## Open items

- [ ] Define precisely how/whether `stereo_camera/` results (baseline
      literature, calibration pipeline, error-model findings) feed into the
      FPGA thesis, versus being prior work mentioned for context only.
- [ ] Advisor sign-off on this combined scope (FPGA-primary,
      camera-rig-supporting) — separate from, and in addition to, the FPGA
      fallback-thesis sign-off already tracked as its own open item in
      `../fpga_stereo_camera/thesis_proposal/thesis-proposal.md` §7.
- [ ] University's required formal thesis-proposal format/template is not
      yet known. This document stays in living-planning-doc form (like its
      FPGA counterpart) until that's confirmed, then gets reformatted into
      whatever's actually required for submission.

## Decision log

- **2026-08-12:** Created this document and the `thesis_proposal/` repo to
  give the two-implementation thesis one place stating combined scope.
  Decisions made setting it up:
  - Lives at `Vydar/thesis_proposal/`, sibling to both projects, as its own
    new git repo — `stereo_camera/` and `fpga_stereo_camera/` (and their
    GitHub remotes) are untouched.
  - `../fpga_stereo_camera/thesis_proposal/thesis-proposal.md` stays where
    it is as the authoritative detailed FPGA-track plan; this document
    links to it rather than moving or copying it.
  - No files were copied or moved from either project into this folder —
    this document links out instead, to avoid duplicate copies going stale.
  - This document is a living planning doc, not yet in a university
    submission format.
