# Thesis Proposal: Real-Time Stereo Proximity Detection on a DSP-less, Open-Toolchain FPGA

**Status:** draft — not yet submitted to advisor. Combined-scope framing and
the detailed FPGA-track plan merged into this one file on 2026-09-03 (see
decision log); FPGA-track structure was restructured per council review on
2026-08-05; research question re-centred on the close-range-detection goal on
2026-09-03, then [`product-plan.md`](./product-plan.md) was set as the driver
and this document as the academic artefact behind it later the same day;
framing revised 2026-09-07 to match the proposal-defense deck
(`../presentations/proposal-defense/deck.md`) — see decision log. **The research
question (§1) and the title above are provisional** pending the approach
selection driven by `product-plan.md`.
**Last updated:** 2026-09-07
**Clock:** 30 weeks, shared with development of the same work as a product

**Goal (drives this proposal):** **real-time close-range detection** on the
GateMate A1 + passive SWIR stereo head — decide, at the ~600 fps sensor rate,
whether an object has come within a configurable distance band. The *minimum*
output is a boolean or zoned flag; the output **granularity** — dense disparity
map, sparse/semi-dense, or a bare flag — is an outcome of the approach search,
not fixed. The output spec, the granularity question, the latency budget, the
safety-critical false-negative metric and the moving-platform / ~600 m/s /
purely-passive application constraints live in
[`product-plan.md`](./product-plan.md) §2–§3. The research framing below is
finalised around whichever engineering approach `product-plan.md`'s search (its
§4–§5) selects.

This is the **academic artefact** of the project. The primary driver — the
goal, the output specification, the open engineering-approach search and how its
winner is chosen, and the 30-week milestone / gate / kill schedule — is
[`product-plan.md`](./product-plan.md) in this folder. This document wraps that
work in the research framing the advisor and committee see, cross-linked to
`product-plan.md` and finalised once the approach search (Track C / M3) lands.
It stays authoritative for the decided hardware / sensor spec (§2), the
bandwidth arithmetic with the verification legend (§3), the research-method
rationale for the track structure (§4), the fallback thesis (§5), the
open-items list (§7), and the chronological decision log (§8).

The thesis is built from two implementations tracked as separate git repos on
disk, siblings of this folder:

- **`../stereo_camera_fpga/`** — real-time stereo matching on a Cologne Chip
  GateMate A1 FPGA (DSP-less, open-toolchain). Expected **primary research
  contribution**. Its full plan is §1–§8 below.
- **`../stereo_camera/`** — a working Rust + Python stereo depth rig on two
  Raspberry Pi cameras, plus a literature review on baseline selection and
  depth-accuracy error models. **Supporting / practical work**, not the thing
  being defended. Detail is not duplicated here — it lives in
  `../stereo_camera/README.md` (working pipeline, current baseline/accuracy
  results) and `../stereo_camera/literature/NOTES.md` (how baseline length
  affects depth accuracy, range, and disparity resolution).

The primary/supporting split is a **current best guess, not settled scope** —
revisit it as Track B/C results land (§4), and update this section and §0
when it firms up or changes.

The FPGA-track plan below supersedes the original 5-phase draft (preserved in
[Appendix A](#appendix-a-original-draft-proposal-pre-review)) by applying the
restructuring recommended in
`../stereo_camera_fpga/council/council-transcript-2026-08-05_2141.md`. Update
phase status and open items here as work happens — don't let this drift out
of sync the way the original draft did.

**Verification legend:** any numeric claim that gates a Track's existence or
kill-gate carries one of three tags — `VERIFIED` (primary-source citation +
date), `VENDOR-CONFIRMED` (direct vendor contact + date), or `ESTIMATED`
(proxy/derived, unconfirmed, ± error). Added 2026-08-06 after a council
session (`../stereo_camera_fpga/council/council-transcript-2026-08-06_1043.md`) caught an
unverified PSRAM claim being stated as settled fact inside this document —
see the decision log.

---

## 0. How the two halves relate

How the FPGA track and the camera rig connect is **not yet fully defined** —
it is the main open question the combined-scope framing exists to eventually
answer, not to prematurely settle:

- The FPGA thesis's stereo-matching algorithm choices (SAD/Census family, per
  the FPGA research surveys) are sensor- and rig-agnostic in principle, so
  `../stereo_camera/`'s baseline/error-model literature review is potentially
  directly reusable background for the FPGA thesis's own accuracy/
  disparity-quality bar (an open item, §7).
- `../stereo_camera/`'s working calibration/rectification pipeline
  (Kalibr-based) is a proof of the same rectify-then-match architecture the
  FPGA implementation will also need, on different hardware — useful prior
  art, not code that transfers directly (different language, different
  camera, different resource constraints).
- Whether `../stereo_camera/` results are cited as prior/parallel work for
  context, or actively reused as source material (e.g., its baseline
  literature review folded into the FPGA thesis's own accuracy-bar decision)
  is **not yet decided** — see §7.

---

## 1. Research question

**Provisional — to be finalised around the engineering approach that
[`product-plan.md`](./product-plan.md)'s search selects (Track C / M3).** The
platform (open-toolchain, DSP-less FPGA + passive SWIR stereo) is uncharacterised
enough that a defensible research question can be framed around any of the
candidate approaches in `product-plan.md` §4; the statement below is the current
best formulation, not a commitment.

> On a DSP-less, open-toolchain FPGA (Cologne Chip GateMate A1) fed by a SWIR
> sensor, which stereo-correspondence approach — a **dense disparity map**, a
> **bounded sparse key-point** matcher, or a **direct band-limited plane-sweep /
> disparity-threshold trigger** — best delivers real-time, high-frame-rate
> **close-range detection** (has an object come within a configurable distance
> band), under the streaming/line-buffer constraint the platform's memory
> forces — and how far can the output be reduced (dense map → semi-dense →
> sparse → bare flag) before detection reliability breaks?

The three approaches are the peer **routes** of `product-plan.md` §4.1; output
granularity is part of what the question decides, not a fixed premise.

Supporting sub-questions:

- What is *feasible* at all on this device class — which cost function ×
  density × resolution/disparity-range configurations close timing and fit the
  ~160 KB BRAM budget on the young `nextpnr-himbaechel` flow (Track A/B)?
- Of the feasible ones, which best meets the close-range-detection objective on
  frame rate, detection reliability, and resource cost (Track C)?
- Does the winning configuration hold up end-to-end on live SWIR hardware
  (Track D)?

**Why this is a genuine research contribution, not an implementation exercise.**
The novelty is the platform, not a new algorithm. No prior real-time FPGA stereo
work runs on an open-toolchain, DSP-less part with this little on-chip memory,
and none uses a SWIR sensor — the surveyed designs all use large
Xilinx/Altera devices with hardened DSP arrays and megabytes of BRAM
(`../stereo_camera_fpga/research/summaries/`). So the achievable
feasibility/quality/throughput envelope for stereo correspondence on this
device+sensor combination is genuinely uncharacterised, and selecting the
approach that best serves a concrete objective (proximity detection) from
synthesis-backed measurements is an evaluation result the literature does not
contain. "First open-toolchain stereo bitstream" and "first real-time SWIR FPGA
stereo depth" are citable secondary outcomes regardless of the fps result (§6).

It is **not** abstract algorithm benchmarking, and it is **not** a
characterisation of a cost × aggregation × resolution *trade-off surface* (that
earlier framing is in the §8 decision log). Whichever approach wins the
`product-plan.md` search, the research question is finalised around it before
submission.

### 1.1 Type of contribution

The thesis **broadens** — it carries known stereo-correspondence techniques
(dense Census/SAD, semi-dense seed-and-grow, bounded sparse key-point matching,
and a band-limited plane-sweep / disparity-threshold trigger) onto a DSP-less,
BRAM-starved, open-toolchain FPGA with a SWIR sensor, for a close-range-detection
objective, and reports what transfers, what is feasible, and what wins. It **deepens** where it measures resource / timing
behaviour undocumented for this device class (M2). It does **not** innovate a
new matching algorithm and does not need to — the novelty is carrying real-time
stereo onto this platform at all, and the systematic selection of the best
approach for the goal. The load-bearing contribution rests on M2 + M3
(feasibility + evaluation on an uncharacterised platform), which is also what
the fallback thesis (§5) protects.

The full committee-vocabulary mapping (deepen / broaden / innovate; Shaw's
research-question taxonomy, by track) is **deferred** until the approach is
chosen — see [Appendix B](#appendix-b-contribution-type-mapping-deferred) and
`product-plan.md` §7.

## 2. Target hardware & sensor (decided)

- **FPGA:** Cologne Chip GateMate A1 (CCGM1A1) on an Olimex GateMateA1-EVB.
  20,480 CPEs (8-input-LUT-equivalent, small adder + 2×2 multiplier, no
  hardened DSP/MAC array), small BRAM (40Kb blocks), no hardened ARM core,
  64Mb external PSRAM, VGA-only output, PMOD/UEXT expansion, fully
  open-source toolchain (yosys + nextpnr-himbaechel + openFPGALoader).
- **Sensor:** QDI Systems SWIR (InGaAs) camera, 640×512, 14-bit,
  400–1700nm. `VENDOR-CONFIRMED` (direct vendor contact, 2026-08-06) — real
  ceiling is the LVDS output interface at **700fps at full resolution**
  (the 50fps spec-sheet figure is a nominal rating, not the limit) —
  comfortably above the 600fps target.
  Decided 2026-08-06. Shifts product framing toward SWIR-specific use cases
  (fog/low-light/material penetration) rather than generic visible-light
  depth — see [open item](#7-open-items-not-yet-resolved) on IP/disclosure.

Full technical detail and rationale: see the "Target hardware" and "Founder
context" sections of the FPGA repo's top-level `../stereo_camera_fpga/CLAUDE.md`.

### 2.1 Application scope (folded in 2026-09-04, decision-logged 2026-09-07)

The sensor rides a **moving platform** (pushbroom-stereo regime, Barry &
Tedrake), not a fixed mount — so no static background-disparity model can be
calibrated and online background learning on the FPGA is rejected. Closing
speeds reach **~600 m/s**: at 600 fps that is ~1 m of travel per frame, an
object is in a 6→3 m watch band for only ~3 frames, and 3 m → contact is ~5 ms,
which kills long-window temporal filtering. The system is **purely passive** (no
active illumination, no sensor fusion), and *anything* coming near — terrain
included — is a valid trigger. A textureless close object with no textured rim
in view is an irreducible residual that falls to an **UNKNOWN** state, not to
detection. These constraints are authoritative in
[`product-plan.md`](./product-plan.md) §3.1 / §2.7 and derived in
`../stereo_camera_fpga/design/CLAUDE.md` §1, §4.

## 3. Bandwidth arithmetic (first pass done 2026-08-06; PSRAM part ID confirmed 2026-08-06)

This was the council's one "do this before anything else" item —
completed same-week rather than left until phase 4/17 as the original draft
did.

| Quantity | Value | Status |
|---|---|---|
| Camera raw ingest @ 600fps | 344 MB/s (401 MB/s at the 700fps LVDS ceiling) | `VENDOR-CONFIRMED` (LVDS ceiling, §2) |
| Raw frame size | 573,440 bytes (560 KiB) | arithmetic, not a sourced claim |
| PSRAM part identification (2× LY68S3200SLT, 8-bit combined bus @ 100MHz, SDR only) | — | `VERIFIED` — see confirmation note below |
| PSRAM theoretical ceiling | **100 MB/s** | `VERIFIED` (part ID confirmed; arithmetic from datasheet-stated max clock) |
| PSRAM effective sustained throughput (burst-timing-derived, see below) | **~97.6 MB/s** | `ESTIMATED` (part ID is confirmed, but the AC timing used is still from the LY68L6400 proxy datasheet — see note) |
| PSRAM capacity | 8 MiB total ≈ 14 single-camera frames, ~7 stereo pairs | `VERIFIED` (follows from confirmed part ID) |
| On-chip Block RAM (total, CCGM1A1) | 1,280 Kb = 160 KB (32 blocks × 40Kb; also expressible as 64 × 20Kb blocks) | `VERIFIED` — Olimex GateMateA1-EVB user manual, p.3: "Block RAM Total 1,280 Kb: 20Kb blocks: 64, 40Kb blocks: 32"; cross-confirmed by Cologne Chip's DS1001 datasheet |

**Part-number confirmed 2026-08-06.** The PSRAM identification above
(2× LY68S3200SLT) was flagged `ESTIMATED` earlier the same day by a
council session (`../stereo_camera_fpga/council/council-transcript-2026-08-06_1043.md`), which
caught that a same-day independent research pass over the same board
(`../stereo_camera_fpga/research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`) couldn't
find the part number in the README/manual text it examined, and used a
different stand-in chip instead — a genuine "unvalidated claim treated as
settled" risk. Resolved by going one level deeper than either prior
source: fetched the actual GateMateA1-EVB Rev.C KiCad schematic from
`github.com/OLIMEX/GateMateA1-EVB` (`HARDWARE/GateMateA1-EVB-Rev.C/GateMateA1-EVB_Rev_C.kicad_sch`)
and found two `LY68S3200SLT(SOIC-8_150mil)` symbol instances, board
references **U7** and **U9**, each with `Datasheet` property
`LY68S3200SLT.pdf`, both wired to the shared `PSRAM_DATA[0..7]` /
`PSRAM_CS#` / `PSRAM_SCLK` bus. The original identification was correct
— the contradicting research pass was wrong only because it checked the
user manual/README rather than the schematic source itself. Lesson for
this document's verification legend: `VERIFIED` requires checking the
actual primary source (schematic/BOM/datasheet), not a document that
merely summarizes it.

**On-chip BRAM bounds the streaming mitigation.** CCGM1A1 has only 160 KB
of on-chip Block RAM total (confirmed above from the Olimex manual and
Cologne Chip's DS1001 datasheet) — this is the hard ceiling on how many
rows of line-buffering the streaming architecture (mandated below) can
hold before spilling to the still-bandwidth-constrained PSRAM. Track B's
synthesis sweeps should treat this as a fixed budget line item alongside
CPE count and achievable clock, not an assumed-sufficient resource.

**Effective bandwidth derivation (closes the "not yet done" item below):**
the part ID is now `VERIFIED` (above), but the AC timing used in this
derivation is still `ESTIMATED` — worked from the LY68L6400 datasheet
(same vendor/family/command-set as the confirmed LY68S3200SLT) because the
S3200's own datasheet (Lyontek, via LCSC part C261882) is blocked from
automated fetching by LCSC's bot protection and isn't mirrored on
Lyontek's own site in a form found during this pass. This is a narrower,
lower-stakes gap than the part-ID question above — it affects only the
precision of the ~97.6 MB/s figure, not whether LY68S3200SLT is the right
chip. QPI Fast
Read overhead is 14 clock cycles (2 cmd + 6 address + 6 wait) before data
starts streaming at 1 nibble/clock/chip. The binding constraint isn't page
crossing — it's **tCEM**, the datasheet's 8µs maximum CE#-low pulse width
(the device must be periodically deselected or internal refresh gets
blocked and the memory fails). At 100MHz that's 800 cycles per burst
window: 786 data cycles (= 786 bytes at the combined 8-bit/1-byte-per-clock
rate) plus 14 cycles overhead, then a mandatory ≥50ns CE#-high recovery
(tCPH) before the next burst. Effective throughput = 786 bytes /
[(800 × 10ns) + 50ns] ≈ **97.6 MB/s** — protocol overhead costs only
~2–3% versus the theoretical 100 MB/s figure, not a meaningful cushion.
One consequence worth noting: since 786 bytes never reaches the 1024-byte
page boundary, the datasheet's separate 84MHz page-crossing frequency
derate never actually triggers under this access pattern — the tCEM cap
is the real limiter, not page geometry.

**Not yet in this arithmetic: PSRAM is single-port.** A pipeline that both
writes incoming camera pixels *and* reads them back out for matching
shares the ~97.6 MB/s ceiling across both directions — it is not 97.6 MB/s
per direction. This should be treated as a further-tightening factor on
top of the verdict below, not a separate independent budget.

**Verdict: PSRAM bandwidth (100 MB/s theoretical `VERIFIED`, ~97.6 MB/s
effective `ESTIMATED` pending exact AC timing) is well below even
single-camera raw ingest (344–401 MB/s, `VENDOR-CONFIRMED`)**, before
counting stereo doubling, cost-volume read/write traffic, or the
read/write-sharing point above. Capacity is arguably the tighter
constraint at only ~7 stereo pairs of buffering (`VERIFIED`, follows from
the confirmed part ID and capacity).

**Implication:** a "capture full frames into PSRAM, then process"
architecture cannot hit the fps target. The pipeline must stream
(line-buffer / row-at-a-time processing) rather than materialize full
frames off-chip. This is now a hard architectural constraint on Track A/C
design, not a stretch performance goal — and one bounded on the on-chip
side by the 160 KB BRAM ceiling noted above.

**Resolved (2026-08-06):** both halves of this item are now closed at the
level that matters for Track A/B planning. The derivation *technique* for
effective (not theoretical) sustained PSRAM bandwidth from burst timing is
done and sound; the *part identification* it depends on (2× LY68S3200SLT)
is now `VERIFIED` directly from the board's own KiCad schematic (see
above), not merely assumed. **Still open, narrow scope:** the exact AC
timing figures (tCEM, tCPH, max clock) used in the derivation come from
the LY68L6400 proxy datasheet, not LY68S3200SLT's own datasheet — see the
open item below. This no longer threatens the architectural conclusion or
the part identification, only the third-decimal precision of the ~97.6
MB/s figure.

## 4. Plan structure (research-method rationale)

The **operational** schedule — the ASCII timeline, the M1–M4 milestones, Gate
A/B/C dates, kill criteria, and the kill findings — lives in
[`product-plan.md`](./product-plan.md) §6. This section records only *why* the
work is structured that way, for the advisor conversation.

The original draft (Appendix A) ran everything as one sequential 30-week
pipeline with a single 3-week bring-up phase blocking all downstream work. The
council's unanimous critique: this makes the single highest-uncertainty item
(first hardware bring-up, unproven toolchain) a hard blocking dependency for
everything else, with no defined fallback if it slips.

The restructure (per
`../stereo_camera_fpga/council/council-transcript-2026-08-05_2141.md`) splits it
into two independently-gated parallel tracks plus two sequential ones, mapped in
`product-plan.md` §6 as milestones M1–M4:

- **M1 / Track A — toolchain + hardware bring-up.** Needs real silicon; the
  founder's highest personal-risk item (first time flashing a board). Gated so a
  slip descopes live integration rather than blocking the thesis.
- **M2 / Track B — synthesis-only feasibility sweep.** Needs no working silicon,
  starts day one. This is where the research evidence is generated: it must
  produce **synthesis-backed place-and-route numbers**, not spreadsheet
  arithmetic — the council was explicit that napkin math here is "dressed-up
  engineering," not a defensible research artefact.
- **M3 / Track C — implement & benchmark the feasible set.** Runs against stored
  test vectors, so it does not depend on M1 succeeding. This is the selection
  experiment: M2 narrows the `product-plan.md` §4 candidate menu — the three
  peer routes (dense disparity map / bounded sparse key-point / band-limited
  plane-sweep trigger) and their RTL configs — to the feasible configurations;
  M3 builds and benchmarks the survivors on equal footing against the
  `product-plan.md` §5 criteria, and the approach **and its output granularity**
  are chosen here. This supersedes the 2026-09-03 council's "dense is the thesis
  spine; sparse is a synthesis-only row, off-thesis" verdict (see §8) — that
  verdict rested on the goal wanting a dense depth map, which it does not.
- **M4 / Track D — live integration.** Depends on M1 Gate B and the M3 winner.
  A stretch goal, not load-bearing — per the fallback in §5.

### Write-up
Weeks ~26–30: framed around the research question in §1, finalised around the
M3-selected approach. The feasibility + approach-selection result is the core
contribution regardless of whether M4 lands; live-hardware fps is validation,
not the thing defended.

## 5. Fallback thesis (must be pre-approved by advisor before week 1)

If M1 (Track A) stalls past its Gate A/B kill criteria, or M4 (Track D) live
integration doesn't land in time: the thesis stands on **M2 + M3 (Track B +
Track C) alone** — a synthesis/simulation-only feasibility study and selection
of the best correspondence approach (dense / semi-dense / sparse) for real-time
proximity detection on a DSP-less, BRAM-constrained, open-toolchain device,
benchmarked against stored test vectors. The operational form of this fallback,
and its kill findings, are in [`product-plan.md`](./product-plan.md) §6. This is
the council's most load-bearing structural recommendation (3-of-5 advisors
flagged the missing-fallback gap independently) — the fallback must be defined
and confirmed with the advisor/committee *before* week 1, not invented under
pressure at week 5.

**Status: not yet confirmed with advisor.**

## 6. Device-specific fps angles (post-bring-up scope, not day-one)

Per the council's Expansionist, folded in as "what to try once bring-up
succeeds," not initial scope — every reviewer flagged that these ideas
skip the load-bearing bring-up risk:

- **Census/Rank as natively LUT-friendly:** Hamming-distance cost functions
  (bitwise XOR + popcount) map directly onto GateMate's LUT-heavy,
  DSP-less CPE fabric with no wasted hardened-multiplier silicon — a
  genuine "this device class enables something DSP-rich chips don't
  optimize for" angle, not just "does Census fit."
- **Spatial-tile parallelism:** 20,480 independent small CPEs invites
  tiling the image and running N cost-aggregation engines concurrently
  across rows/regions feeding a shared PSRAM bus in bursts, rather than one
  wide pipeline racing the clock. Unpublished in either literature survey
  — DSP-rich chips never needed this.
- **"First open-toolchain stereo bitstream" as a secondary win condition:**
  citable in the open-EDA community independent of fps outcome — real-time stereo
  on a fully open board and ~$100 of hardware. Worth naming explicitly in the
  write-up regardless of fps result.

## 7. Open items (not yet resolved)

Carried forward from the council session — flagged by peer review, not yet
addressed by any advisor round.

**Combined-scope items** (from the former umbrella scope doc):

- [ ] **How the camera rig feeds the FPGA thesis.** Define precisely how/
      whether `../stereo_camera/` results (baseline literature, calibration
      pipeline, error-model findings) feed into the FPGA thesis, versus being
      prior work mentioned for context only — see §0.
- [ ] **Advisor sign-off on the combined scope** (FPGA-primary,
      camera-rig-supporting). Separate from, and in addition to, the
      fallback-thesis sign-off tracked below.
- [ ] **University's required formal thesis-proposal format/template is not
      yet known.** This document stays in living-planning-doc form until
      that's confirmed, then gets reformatted into whatever's required for
      submission.

**FPGA-track items:**

- [ ] **Proximity-detection quality bar — resolve before M3.** The metric
      definitions and their weighting live in [`product-plan.md`](./product-plan.md)
      §5 (detection reliability at the band edge; the safety-critical
      false-negative rate on close objects; false-positive rate; latency;
      minimum valid-match density). The research-side requirement this item
      tracks: the **minimum fps + detection reliability below which the product
      story collapses** must be written down as an explicit, pre-agreed kill
      criterion *before* M3 benchmarking. The thesis can honestly conclude "this
      part + this sensor can't do real-time proximity detection at spec with any
      candidate approach" — but only if that bar predates the result. The
      2026-09-01 SWIR survey
      (`../stereo_camera_fpga/research/summaries/2026-09-01-swir-stereo-depth-perception.md`)
      adds that matching failure is *spatially structured by material* (specular
      metal, glass, very low-albedo surfaces), so the metric must report where
      detection is unreliable, not just an aggregate rate.
- [ ] **SWIR ground truth / benchmark for Track C.** Track C assumes stored
      KITTI/Middlebury frames, which are visible-light. The 2026-09-01 SWIR
      survey found no SWIR (or NIR/thermal) disparity benchmark with comparable
      ground truth. Options from the literature: (a) evaluate on visible-light
      data and argue the resource/timing results transfer (feasibility and
      resource cost are largely sensor-agnostic; detection *reliability* is
      not), (b) a small SWIR rig with time-synced LiDAR (or simply
      known-distance targets at measured stand-off) for metric ground truth —
      this is lighter for a proximity-detection metric than for dense
      disparity, since only the near-field distance-to-object needs to be
      known, (c) synthetic SWIR. Pick one and record the caveat before Track C
      benchmarking.
- [ ] **Detection range vs. sensing geometry.** A ~12 cm baseline at 600 fps
      gives usable stereo disparity only in the last few metres and ~2–4 frames
      of warning. Whether the stated standoff is reachable at all — a wider
      baseline (a very different mechanical build), a longer focal length, a
      higher windowed-ROI frame rate, or stereo as a last-metres confirm behind
      a monocular looming detector — bounds what any matching route can do, and
      is a legitimate kill finding diagnosable from geometry before M3
      (`product-plan.md` §6 kill findings, `../stereo_camera_fpga/design/CLAUDE.md` §4).
- [ ] **IMU / ego-motion estimation on the platform?** Decides whether Route C's
      two-frame approach-confirm is a motion-compensated predictor (Barry &
      Tedrake) or a plain N-of-M + monotonic-Δd test
      (`../stereo_camera_fpga/design/CLAUDE.md` §7).
- [ ] **UNKNOWN / blind-state output.** `product-plan.md` §2 currently specifies
      no valid/blind companion flag and no large-blind-region conservative
      policy for the textureless-object residual (§2.1 here). Specify both before
      the output form is frozen.
- [ ] **Alarm-consumer reaction time.** At ~600 m/s the loop only closes if the
      downstream effector acts in single-digit ms; otherwise detection must move
      farther out regardless of sensor quality. Needs a figure from the use case
      (`product-plan.md` §2.4, §2.9).
- [ ] **Thesis disclosure vs. startup IP.** A public master's thesis (RTL,
      methodology, results) sits next to a startup's proprietary core,
      sharpened by the SWIR sensor decision shifting product framing toward
      a more specific (and more defensible) use case. Needs one conversation
      with advisor and/or co-founder before write-up, not after.
- [ ] **Advisor/committee sign-off on the fallback thesis** (§5) — the
      fallback only works as a hedge if it's pre-approved, not improvised
      under deadline pressure.
- [ ] **Community outreach** — GateMate/nextpnr-himbaechel forums, Cologne
      Chip reference designs, existing example repos — before sinking Track
      A solo-debugging time into problems that may have known solutions.
- [x] **Effective-vs-theoretical PSRAM bandwidth derivation technique** —
      resolved 2026-08-06: burst-timing math (tCEM-bound, ~2–3% overhead)
      is done and sound, applicable to whichever chip turns out to be
      correct. This item is *only* about the technique, not the specific
      ~97.6 MB/s figure — see the next item for that.
- [x] **Confirm the actual PSRAM part number.** Flagged 2026-08-06 by
      `../stereo_camera_fpga/council/council-transcript-2026-08-06_1043.md`, resolved the same
      day: fetched the GateMateA1-EVB Rev.C KiCad schematic directly from
      `github.com/OLIMEX/GateMateA1-EVB` and found two
      `LY68S3200SLT(SOIC-8_150mil)` instances (board refs **U7**, **U9**),
      confirming §3's original identification was correct — the
      contradicting research pass had only checked the user manual/README,
      not the schematic. §3's part ID is now `VERIFIED`.
- [ ] **Confirm exact AC timing (tCEM, tCPH, max clock) from
      LY68S3200SLT's own datasheet.** Narrower residual of the item above:
      the part is now confirmed, but the ~97.6 MB/s effective-bandwidth
      figure still uses the LY68L6400 proxy datasheet's timing, because
      LCSC (source: Lyontek Inc., part C261882) blocks automated PDF
      fetching and no mirror was found on Lyontek's own site during this
      pass. Fix: download `https://www.lcsc.com/datasheet/C261882.pdf`
      manually (browser, not automated fetch) or request the datasheet
      directly from Lyontek/Olimex, then re-check §3's derivation against
      the real AC timing table. Low priority — does not affect the part
      ID or the architectural conclusion, only the precision of one
      figure.
- [ ] **PSRAM single-port read/write sharing** — §3's arithmetic doesn't
      yet account for the pipeline needing to both write incoming camera
      pixels and read them back for matching over the same ~97.6 MB/s
      channel. Fold this into the streaming-architecture design in Track
      A/C rather than treating read and write as independent budgets.
- [ ] **Rectification: the vertical-misalignment budget `k` must be a signed
      mechanical/optical spec, not a tolerance to design around.** (2026-09-03
      council, `../stereo_camera_fpga/council/council-transcript-2026-09-03_1406.md`.)
      A full per-pixel remap LUT is unusable in *both* memories — in BRAM it is
      ≈ 2.6 MB ≈ 16× the ~160 KB budget; re-read from PSRAM every frame it is
      ≈ 780 MB/s per camera / ~1.5 GB/s per stereo pair, ~15× the ~97.6 MB/s
      ceiling. So rectification must be **computed on the fly** from ~a dozen
      polynomial lens-distortion + rotation coefficients on CPE soft-multipliers,
      fixed at calibration time, and **fused into the matching line buffer**
      (the buffer feeding the cost function *is* the vertical-remap window).
      `k` (raw rows N ± k needed per rectified row) does not shrink with the
      storage choice; realistic free on-chip headroom after the matching window
      `W` and disparity buffers is ~16–32 rows. Get the *real product's* stereo
      head to commit to a worst-case residual (target ≤ ±16 rows, athermal
      mount, alignment fixtured at assembly) held across temperature and
      vibration — by ~week 4. If `k` cannot be bounded small enough to leave
      room for `W` and `D` for *any* of the candidate approaches (including the
      sparse and semi-dense ones, which need fewer rows), "this sensor + this
      part can't stream real-time proximity detection" is a legitimate kill
      finding (needs the minimum-fps/detection bar above defined first). The
      `k`-vs-`W`-vs-`D` BRAM-budget Pareto curve, and
      the sustained multiplies/s the fabric delivers at pixel rate, are
      themselves Track B characterisation results. The Pi rig's own calibration
      numbers are moot — it is throwaway test scaffolding. Full detail:
      `../TODO.md` item 3.
- [ ] **SWIR camera LVDS interface vs. GateMate LVDS-GPIO/SerDes fit** —
      the QDI sensor's LVDS lane count and per-lane bit rate haven't been
      checked against GateMate's DDR-GPIO LVDS timing (no max LVDS toggle
      rate found in the CCGM1A1 datasheet's electrical tables) or against
      whether the die's separate 5.0 Gb/s SerDes block would be needed
      instead. Concrete next step before Track A camera bring-up (Gate B).
      **2026-09-03 council sharpened this:** one 5.0 Gb/s SerDes lane carries
      roughly a *single* SWIR stream (~500 MB/s usable), not two — so two-camera
      ingest needs a second path (DDR-GPIO LVDS, reduced fps/bit-depth, or a
      dual-lane CCGM1A2). And the two sensors must be **line-locked**; if they
      are not, realigning them costs a frame buffer → PSRAM staging → the
      streaming premise collapses before any dense/sparse or rectification
      choice matters. This sits upstream of both.

## 8. Decision log

- **2026-08-05:** First council session — argued for local (SAD/Census)
  matching over SGM to hit fps target; flagged sensor/PSRAM bandwidth as
  unvalidated likely ceiling.
- **2026-08-05:** Follow-up SAD/Census literature survey — partially
  cross-checked the first council's hypothesis. A Belief-Propagation design
  (more expensive than SGM) hit higher fps (1570) than pure SAD (600) by
  shrinking resolution/disparity range — suggesting resolution/disparity
  reduction, not algorithm family, is the dominant fps lever.
- **2026-08-05:** Second council session reviewed the original 5-phase
  draft (Appendix A). Verdict: restructure, don't discard. Recommendations
  applied in this document: parallel Track A/B split, explicit gates and
  kill criteria, fallback thesis requirement, research question reframed as
  the thesis spine, Expansionist's fps angles demoted to post-bring-up
  scope.
- **2026-08-06:** Sensor decided — QDI Systems SWIR camera (§2).
- **2026-08-06:** Bandwidth arithmetic first pass completed (§3) — PSRAM
  bandwidth confirmed as a hard ceiling below raw camera ingest; streaming
  architecture is now a required constraint, not a nice-to-have.
- **2026-08-06:** GateMate A1 research brief completed
  (`../stereo_camera_fpga/research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`) —
  resolved the effective-vs-theoretical PSRAM bandwidth open item
  (~97.6 MB/s effective, confirming rather than softening the 100 MB/s
  theoretical figure); surfaced the previously-unaccounted-for single-port
  read/write-sharing constraint; identified that the QDI SWIR camera's
  LVDS interface (not MIPI/DVP) is the actual camera-connectivity question
  and that its lane count/bit rate vs. GateMate's LVDS-GPIO or SerDes
  capability is still unchecked.
- **2026-08-06:** Council session
  (`../stereo_camera_fpga/council/council-transcript-2026-08-06_1043.md`) reviewed this document
  itself and caught a sourcing-collapse: §3 presented the PSRAM part
  identification (LY68S3200SLT) as confirmed ("sourced from the schematic
  + BOM, not guessed") when a same-day independent research pass on the
  same board couldn't confirm it — the exact "unvalidated claim treated
  as settled" failure this document exists to prevent, recurring inside
  it. Applied the council's fix: added a three-tag verification legend
  (`VERIFIED` / `VENDOR-CONFIRMED` / `ESTIMATED`) applied retroactively
  throughout the document; downgraded the PSRAM part ID and derived
  bandwidth figures to `ESTIMATED`; added the previously-missing ~160 KB
  on-chip BRAM figure as a `VERIFIED` constraint on the streaming
  mitigation; stated explicitly that the streaming-architecture conclusion
  holds regardless of which PSRAM chip is correct; elevated part-number
  confirmation from a buried "residual sub-item" to its own dated, owned
  open item.
- **2026-08-06:** Part-number `ESTIMATED` item resolved same day. Prompted
  by spotting the Olimex user manual's BRAM figure ("1,280 Kb," i.e. the
  same 160 KB already in this document, just in kilobits — no
  discrepancy, just a unit check that led to reading the manual directly),
  fetched the manual and confirmed the BRAM quote on p.3, then went one
  level deeper than either prior source by pulling the actual
  GateMateA1-EVB Rev.C KiCad schematic from
  `github.com/OLIMEX/GateMateA1-EVB`. Found two `LY68S3200SLT` chip
  instances (board refs U7, U9) directly in the schematic — the original
  identification was correct all along; the contradicting research pass
  had only checked the manual/README, not the schematic. Updated §3 and
  §7 accordingly: PSRAM part ID is now `VERIFIED`; only the exact AC
  timing (still sourced from the LY68L6400 proxy datasheet, since LCSC
  blocks automated fetching of LY68S3200SLT's own datasheet) remains open,
  as a narrow, low-stakes residual.
- **2026-08-12:** Created a separate combined-scope umbrella document (and
  the `thesis_proposal_vydar/` repo) to give the two-implementation thesis
  one place stating combined scope: FPGA-primary, camera-rig-supporting;
  primary/supporting split explicitly a best guess, not settled; no files
  copied or moved from either project — the umbrella doc linked out
  instead. (Merged into this file 2026-09-03, below.)
- **2026-09-01:** SWIR + depth-perception literature survey completed
  (`../stereo_camera_fpga/research/summaries/2026-09-01-swir-stereo-depth-perception.md`, 6
  papers; ../TODO.md task 1). Key conclusions: (a) SWIR's risk to the
  Census/SAD costs is *texture starvation* in fog / hot-metal / glass /
  low-light scenes, not spectral mismatch — the two same-band InGaAs
  sensors stay inside Census's monotonic-invariance assumption; (b) every
  prior IR/thermal stereo *system* pairs the local cost with semi-global
  aggregation plus an edge-preserving refinement stage, because the raw
  local cost is texture-starved — pressure against Track C's local /
  no-aggregation configs as the primary bet; (c) no real-time or FPGA
  dense IR/SWIR stereo precedent exists, so there are no comparable
  CPE/fps/accuracy numbers to cite or beat; (d) no SWIR (or NIR/thermal)
  dense-disparity benchmark exists with KITTI/Middlebury status. Reflected
  in §7 (accuracy-bar item updated; SWIR-benchmark item added).
- **2026-09-01:** Added §1.1 (type of contribution) — classifies each Track
  by contribution type (deepen/broaden/innovate; Shaw's taxonomy) to make
  the "characterization, not implementation" argument explicit for the
  advisor.
- **2026-09-03:** Merged the two proposal files in `thesis_proposal_vydar/`
  into this one. The former umbrella scope doc (`thesis-proposal.md`, the
  combined-scope framing) and the detailed FPGA-track plan
  (`thesis-proposal2.md`) were separate files; the umbrella's framing is
  now the header + §0, its three combined-scope open items are folded into
  §7, and its 2026-08-12 creation entry into §8 (above). Relative links to
  the FPGA repo's `research/` and `council/` files were repointed to
  `../stereo_camera_fpga/...` since this file now lives one level out.
  `thesis-proposal2.md` deleted.
- **2026-09-03:** Two council sessions on the sparse-vs-dense choice and on
  rectification (`../stereo_camera_fpga/council/council-transcript-2026-09-03_1156.md`
  and `...-2026-09-03_1406.md`). Verdicts:
  - **Dense (or semi-dense) is the thesis spine.** The sparse key-point front
    end is carried as a synthesis-only characterised row, not a built pipeline —
    its data-dependent tail (keypoint-list scatter/gather, NN match, RANSAC,
    triangulation) is a second RTL subsystem with no dense analogue that every
    surveyed FPGA design offloads to a CPU the A1 lacks; sparse also saves only
    non-binding resources; and SWIR texture starvation gives it too few
    keypoints with too-weak evidence in the target scenes. The real axis is
    "how much aggregation the fabric affords," with sparse as its
    zero-aggregation endpoint and **semi-dense seed-and-grow / ELAS-style
    support points added to the Track C list** as the honest middle (§4).
  - **Rectification.** A full per-pixel LUT is unusable in BRAM *and* in PSRAM;
    compute it on the fly from polynomial coefficients, fixed at calibration
    time, fused into the matching line buffer. The vertical-misalignment budget
    `k` must be a *signed mechanical spec* (target ≤ ±16 rows), not a tolerance;
    resolve by ~week 4. Added to §7.
  - **Blind spots re-surfaced** (added/sharpened in §7): one 5.0 Gb/s SerDes
    lane ≠ two camera streams, and two rolling-shutter sensors must be
    line-locked or a frame buffer (→ PSRAM → streaming collapse) is forced; no
    SWIR stereo dense-disparity benchmark with ground truth; timing closure on
    the young `nextpnr-himbaechel` flow, not CPE/BRAM count, is the real
    feasibility gate; a minimum-fps/accuracy kill criterion must be written down
    before Track C.
  - The 2026-09-01 and 2026-09-02 council sessions (implementation-plan and
    disparity-doc reviews) are still not individually folded into this log.
- **2026-09-03:** Research question re-centred on the real product goal
  (founder direction). The goal was clarified: the product is **real-time,
  high-fps detection of whether the camera is close to an object** —
  proximity/near-object detection — **not** a dense per-pixel depth map, and a
  sparse or low-density output is acceptable (and, given the streaming budget,
  preferable). Consequences applied through §§1, 1.1, 4, 5, 7:
  - §1's research question is no longer "how does removing the DSP/MAC array
    reshape the cost × aggregation × resolution/disparity *trade-off surface*."
    It is now "which correspondence approach — dense, semi-dense, or sparse —
    best delivers real-time proximity detection on this platform." Aggregation
    is one option among several, not a mandatory axis; dropping it entirely is
    a valid answer.
  - The "genuine research contribution" argument now rests on the **novel
    platform** (first real-time stereo on an open-toolchain, DSP-less,
    ~160 KB-BRAM FPGA; first real-time SWIR FPGA stereo) plus the systematic
    feasibility + selection study, not on characterising an abstract surface.
    §1.1 re-pointed accordingly (Tracks B + C are the spine; contribution type
    is *broaden* + *feasibility/evaluation*, not *characterization*).
  - **Supersedes the earlier 2026-09-03 council verdict** that "dense (or
    semi-dense) is the thesis spine" and that the sparse key-point front end is
    "a synthesis-only characterised row, not a built pipeline." Sparse (Config
    5) and semi-dense seed-and-grow (Config 4) are now **built and benchmarked
    Track C configurations on equal footing with dense Census/SAD.** The
    council's *engineering* caveats still hold and are respected in the config
    spec: the built sparse pipeline is **bounded** (hard key-point cap, spatial
    buckets, deterministic LRC + ordering + fixed-epipolar-offset verification,
    sub-pixel) with **no RANSAC / triangulation / variable-length scatter-gather
    tail** — and the proximity-detection goal is what removes the need for that
    tail, since "is something close" needs a bounded set of near matches, not a
    point cloud. The council transcripts themselves are point-in-time snapshots
    and were left unedited.
  - §7's "accuracy/disparity-quality bar" item became the **proximity-detection
    quality bar** (min stand-off distance, false-negative rate for close
    objects, false-positive rate, latency, min match density), and the
    kill-criterion is now "minimum fps + detection reliability below which the
    product story collapses."
- **2026-09-03:** Product re-prioritised as the **organising goal** (founder
  direction) — the previous entry's proximity reframe kept a research question
  as the spine; this one inverts it. The product is now the driver: a
  **high-speed near/far distance-threshold alarm** (boolean or zoned; not a
  distance measurement, not a dense depth map) via passive SWIR stereo on the
  fixed GateMate A1 hardware. Consequences:
  - Created [`product-plan.md`](./product-plan.md) as the **primary driver
    document** — it owns the product goal, the threshold-alarm output spec, the
    open engineering-approach search (its §4), the concrete selection method
    (its §5), and the 30-week milestone / gate / kill-finding schedule (its §6,
    relocated from §4 here as milestones M1–M4).
  - **This document is demoted to the academic artefact** wrapped around the
    chosen approach. Header, line 20, and both CLAUDE.md authority registers
    (top-level and `thesis_proposal_vydar/`) repointed so `product-plan.md` is
    the entry point; `README.md`, `TODO.md`, `stereo_camera_fpga/CLAUDE.md`
    repointed to match; `implementation-plan.md` reframed as "the RTL menu the
    approach search evaluates."
  - **§1 research question and the title are marked provisional**, to be
    finalised around whichever approach M3 selects. The trade-off-surface
    litigation paragraph in §1 was trimmed (history is in this log).
  - **§1.1 shortened** to one paragraph; Shaw's taxonomy table and the
    deepen/broaden/innovate-by-track bullets moved to
    [Appendix B](#appendix-b-contribution-type-mapping-deferred).
  - **§4 keeps only the research-method rationale** for the parallel-track
    structure and points to `product-plan.md` §6 for the schedule; the inline
    Config 1–6 prose was condensed (full menu is `implementation-plan.md` §6,
    framed by `product-plan.md` §4).
  - No bandwidth arithmetic (§3), hardware spec (§2), verification-legend
    definition, decision log (§8), or fallback thesis (§5) moved — those stay
    authoritative here and are referenced from `product-plan.md`.
  - The `presentations/` deck was left untouched (out of scope for this
    restructure; it carries its own stale-framing banner).
- **2026-09-07:** Framing revised to match the proposal-defense deck
  (`../presentations/proposal-defense/deck.md`), which had been built out as a
  staged back-and-forth and now carries the current consensus framing. The deck
  is the source; the planning docs were brought into line with it. Changes:
  - **Output granularity re-opened.** The goal leads as *close-range detection
    within a configurable band*; the output *form* — dense map / sparse /
    semi-dense / bare boolean-or-zoned flag — is an outcome of the approach
    search, not a premise. The 2026-09-03 "not a dense depth map" exclusion is
    dropped: a dense map is one route (Route A). The boolean/zoned spec stays as
    the *minimum* output; the §2.5 (`product-plan.md`) reliability bar binds
    every route. §1 research question and §1.1 updated accordingly.
  - **Three peer routes.** `product-plan.md` §4 and `implementation-plan.md` §6
    are reorganised around Route A (dense disparity map), Route B (bounded sparse
    key-point), Route C (band-limited plane-sweep / disparity-threshold trigger),
    presented without a ranking. `implementation-plan.md` gains **Config 7** for
    Route C, folded in from `../stereo_camera_fpga/design/CLAUDE.md` (Stages 0–4,
    `ESTIMATED` budget).
  - **Application scope folded in** (new §2.1 here; `product-plan.md` §3.1 /
    §2.7): moving platform / no static background model / no online background
    learning, closing speed up to ~600 m/s (kills long-window temporal
    filtering), anything-near-is-valid, purely passive, textureless-object
    **UNKNOWN state**. Corresponding §7 open items added (detection range vs.
    geometry, IMU availability, UNKNOWN-state output, alarm-consumer reaction
    time). This closes the "not yet folded into the planning docs" flag in
    `../stereo_camera_fpga/design/CLAUDE.md` §1 / §7.
  - **Startup / commercial framing de-emphasised** across `product-plan.md` — it
    now reads as the engineering plan for a research project with a concrete
    application goal. §2–§3 arithmetic, the verification legend, the fallback
    thesis (§5) and this log are unchanged.
  - The council transcripts and the earlier decision-log entries above are
    point-in-time records and were left unedited.

---

## Appendix A: original draft proposal (pre-review)

Preserved for reference — this is the 5-phase draft the second council
session reviewed. Superseded by the Track A–D structure in §4.

1. **Weeks 1–3:** De-risk toolchain + first hardware bring-up (trivial
   design through full RTL→bitstream→flash flow) and bring up a
   parallel/DVP camera module to check real achievable sensor frame rate
   early.
2. **Weeks 4–6:** Resource-budget arithmetic — CPE count × achievable clock
   × BRAM Kb → what resolution/disparity-range/cost-function combinations
   are physically feasible on this device.
3. **Weeks 7–16:** Implement and benchmark 2–3 feasible matching-cost/
   aggregation configurations (e.g. pure SAD, AD-Census, possibly a reduced
   -path SGM if resources allow) against stored test-vector frames
   (KITTI/Middlebury).
4. **Weeks 17–24:** Integrate the best-performing configuration with the
   live camera pair end-to-end (capture, rectification, matching, output)
   and validate real fps on hardware.
5. **Weeks 25–30:** Write-up, framed around validating (or refuting) the
   "kill SGM, go local" hypothesis on this specific small/open-toolchain
   device class.

Council's core critique of this structure: phase 1 bundles two
unprecedented risks into one under-scoped bucket with no internal
checkpoints; there is no defined fallback if phase 1 or phase 2 blows the
schedule; phase 2's "arithmetic" risks being napkin math rather than a
defensible research artifact; and live-camera-plus-PSRAM bandwidth (the
likely real ceiling per both literature surveys) isn't validated until
week 17, after 13 weeks are already sunk into algorithm tuning against
idealized in-memory test vectors.

Full reasoning: `../stereo_camera_fpga/council/council-transcript-2026-08-05_2141.md`.

---

## Appendix B: contribution-type mapping (deferred)

Preserved for when the engineering approach is chosen (M3) — not load-bearing in
the current draft. §1.1 carries the short version; this is the full
committee-vocabulary mapping, to be finalised around the selected approach.

**Deepen / broaden / innovate** (common in EU/NL programs):

- The thesis mainly **broadens** — it takes known stereo-correspondence
  techniques (dense Census/SAD, semi-dense seed-and-grow, bounded sparse
  key-point matching, and a band-limited plane-sweep / disparity-threshold
  trigger) and applies them in a genuinely new context: a DSP-less,
  BRAM-starved, open-toolchain FPGA with a SWIR sensor, for a
  close-range-detection objective — and reports what transfers, what is
  feasible, and what wins.
- It **deepens** where it measures resource / timing behaviour of those
  techniques on a device class where that behaviour is undocumented (M2).
- It does **not** innovate a new matching algorithm, and does not need to — the
  novelty is carrying real-time stereo onto this platform at all, and the
  systematic selection of the best approach for the goal.

**Shaw's research-question taxonomy** ("Writing Good Software Engineering
Research Papers", ICSE 2003), by milestone:

| Milestone | Question type | Question |
|---|---|---|
| M1 | Feasibility | Can a non-trivial stereo design be closed and run on a GateMate A1 via the open toolchain at all? |
| M2 | Feasibility / characterisation | Which cost function × density × resolution/disparity configurations close timing and fit the BRAM budget, and at what resource cost? |
| M3 | Evaluation / selection | Of the feasible configurations across the three routes (dense disparity map, bounded sparse key-point, band-limited plane-sweep trigger), which best delivers real-time close-range detection — and at what output granularity, and by how much? |
| M4 | Feasibility (integration) | Does the selected configuration hold up end-to-end on live SWIR hardware? |

The university's "genuine research contribution, not an implementation exercise"
requirement is, in these terms, satisfied by the thesis **resting on M2 + M3
(feasibility + evaluation on an uncharacterised platform)**, with M1 / M4 as
bring-up and validation. A thesis resting on M1 alone would be a pure
implementation result — necessary, not sufficient. The fallback thesis (§5) is
deliberately M2 + M3, so the load-bearing contribution survives losing the
hardware. State this explicitly to the advisor, in whichever vocabulary the
committee uses.
