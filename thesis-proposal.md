# Thesis Proposal: Real-Time Stereo Depth

**Status:** draft — not yet submitted to advisor. Combined-scope framing and
the detailed FPGA-track plan merged into this one file on 2026-09-03 (see
decision log); FPGA-track structure was restructured per council review on
2026-08-05.
**Last updated:** 2026-09-03
**Clock:** 30 weeks, shared with startup product development

This is the single living planning document for the thesis. The thesis is
built from two implementations tracked as separate git repos on disk,
siblings of this folder:

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

> How does removing a hardened DSP/MAC array and abundant fast BRAM reshape
> the achievable trade-off surface (cost function × aggregation strategy ×
> resolution/disparity range) for real-time stereo matching?

This is the thesis's spine (per the council's First Principles Thinker,
adopted in synthesis). It is **not** "which matching algorithm is fastest" —
that's benchmarking, already partially answered by existing literature
(see `../stereo_camera_fpga/research/summaries/`), and doesn't by itself satisfy the university's
requirement for genuine research contribution.

Framing it this way means the resource-characterization work (Track B below)
*is* the experiment, not a preliminary step before the "real" implementation
work.

### 1.1 Type of contribution

The question above is deliberately framed to make the thesis a
*characterization* study rather than a *build* exercise. Two vocabularies a
committee is likely to use, and where each Track sits in them:

**Deepen / broaden / innovate** (common in EU/NL programs):

- Track B **deepens** — takes known stereo-matching techniques and
  characterizes their resource/timing behavior more precisely, on a device
  class where that behavior is undocumented.
- Track C **broadens** — applies those techniques in a new context
  (DSP-less, BRAM-starved, open-toolchain) and reports what transfers.
- No Track **innovates** a new matching algorithm, and none needs to — the
  contribution is the trade-off surface, not a new cost function.

**Shaw's research-question taxonomy** ("Writing Good Software Engineering
Research Papers", ICSE 2003), by Track:

| Track | Question type | Question |
|---|---|---|
| A | Feasibility | Can a non-trivial stereo design be closed and run on a GateMate A1 via the open toolchain at all? |
| B | Characterization / generalization | How does the cost × aggregation × resolution trade-off surface change once the hardened DSP/MAC array and fast BRAM are removed? |
| C | Evaluation / selection | Of the feasible configurations, which wins on fps / accuracy / resource, and by how much? |
| D | Feasibility (integration) | Does the best configuration hold up end-to-end on live hardware? |

The university's "genuine research contribution, not an implementation
exercise" requirement is, in these terms, a requirement that the thesis
**rest on Track B (characterization)**, with A / C / D as support and
validation. A thesis resting on Track A alone would be a
feasibility/implementation result — necessary, not sufficient. The fallback
thesis (§5) is deliberately Track B + C, i.e. characterization + evaluation,
so the load-bearing contribution survives losing the hardware. State this
explicitly to the advisor, in whichever of the two vocabularies the
committee uses.

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

## 4. Plan structure

The original draft ran everything as one sequential 30-week pipeline with a
single 3-week bring-up phase blocking all downstream work. The council's
unanimous critique: this makes the single highest-uncertainty item (first
hardware bring-up, unproven toolchain) a hard blocking dependency for
everything else, with no defined fallback if it slips.

Restructured into two independently-gated parallel tracks plus two
sequential tracks that depend on both:

```
Week:        1    2    3    4    5    6  ...  16       24       30
Track A:  [--- bring-up ---][Gate A][-- camera bring-up --][Gate B]
Track B:  [-- synthesis-only resource characterization --][Gate C]
Track C:                          [-- implement & benchmark configs --]
Track D:                                      [-- live integration --][write-up]
```

### Track A — Toolchain + hardware bring-up
*Needs real silicon. Founder's highest personal-risk item (never flashed
hardware before, unproven toolchain on this device).*

- **Gate A (target: end of week 4–5, not week 3):** a *non-trivial* design
  (not blinky) synthesized via yosys/nextpnr-himbaechel, placed & routed,
  flashed, and producing observable output on real hardware. JTAG/programmer
  flow confirmed repeatable.
- **Gate B (target: end of week 8):** SWIR camera streams pixels into BRAM,
  readable back out over UART/VGA, at a measured (not assumed) sustained
  rate.
- **Kill criterion:** if Gate A isn't hit by week 6, or Gate B isn't hit by
  week 9, live-camera integration (Track D) is descoped to a stretch goal
  and the thesis runs on the fallback described in §5.
- Before burning solo weeks rediscovering toolchain quirks: check for
  GateMate/nextpnr-himbaechel community prior art (forums, Cologne Chip
  reference designs, existing example repos) — flagged as a blind spot no
  one addressed in the council review.

### Track B — Synthesis-only resource characterization
*Needs no working silicon — can start day one, runs in parallel with Track A.*

- This is the thesis's actual experiment (§1). Must produce
  **synthesis-backed (place-and-route) numbers**, not spreadsheet
  arithmetic — the council was explicit that napkin math here is "dressed-up
  engineering," not a defensible research artifact.
- Sweep candidate configurations (cost function × aggregation strategy ×
  resolution/disparity range) through yosys/nextpnr and record CPE
  utilization, achievable clock, BRAM usage for each.
- **Gate C (target: end of week 10):** enough synthesis sweeps done to
  identify 2–3 feasible configurations to carry into Track C.

### Track C — Implement & benchmark feasible configurations
*Depends on Track B's Gate C. Does not require Track A to have succeeded —
can run against stored test-vector frames (KITTI/Middlebury) even if live
bring-up has stalled.*

- Weeks ~10–20: implement and benchmark the 2–3 configurations Track B
  identified as feasible, against stored test frames, for controlled
  fps/accuracy/resource comparison.
- Candidate configurations, informed by the SAD/Census survey and the
  Expansionist's device-fit argument (§6): pure SAD, AD-Census, possibly a
  reduced-path SGM if Track B's resource numbers allow.

### Track D — Live integration & real-hardware validation
*Depends on both Track A Gate B and Track C's best-performing
configuration. Stretch goal, not the thesis's load-bearing requirement —
per the fallback in §5.*

- Weeks ~20–26: integrate the best-performing configuration with the live
  SWIR camera pair end-to-end (capture, rectification, matching, output)
  and validate real fps on hardware, honoring the streaming-architecture
  constraint from §3.

### Write-up
- Weeks ~26–30: framed around the research question in §1 — the
  trade-off-surface characterization is the core result regardless of
  whether Track D lands; live-hardware fps is presented as validation/
  upper-bound confirmation, not as the thing being defended.

## 5. Fallback thesis (must be pre-approved by advisor before week 1)

If Track A stalls past its Gate A/B kill criteria, or Track D's live
integration doesn't land in time: the thesis stands on **Track B + Track C
alone** — a synthesis/simulation-only characterization of the cost
function × aggregation × resolution/disparity-range trade-off surface on a
DSP-less, BRAM-constrained, open-toolchain device, benchmarked against
stored test vectors. This is the council's most load-bearing structural
recommendation (3-of-5 advisors flagged the missing-fallback gap
independently) — the fallback must be defined and confirmed with the
advisor/committee *before* week 1, not invented under pressure at week 5.

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
  citable in the FOSS-EDA community independent of fps outcome, and doubles
  as startup narrative ("depth on a fully-open board" vs. a $3k Xilinx eval
  kit). Worth naming explicitly in the write-up regardless of fps result.

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

- [ ] **Accuracy/disparity-quality bar.** 600fps is trivially achievable by
      degrading resolution/disparity range into meaninglessness. Define what
      depth-quality metric (e.g. bad-pixel-percentage against KITTI/
      Middlebury ground truth, at what threshold) the thesis defends
      alongside fps, before Track C benchmarking starts. The 2026-09-01 SWIR
      survey (`../stereo_camera_fpga/research/summaries/2026-09-01-swir-stereo-depth-perception.md`)
      adds two constraints: IR/cross-spectral stereo work shows matching
      failure is *spatially structured by material* (specular metal, glass,
      very low-albedo surfaces), so the metric must report an invalid-pixel
      fraction, not only an error over valid pixels; and KITTI/Middlebury
      ground truth is visible-light — see the SWIR-benchmark item below.
- [ ] **SWIR ground truth / benchmark for Track C.** Track C assumes stored
      KITTI/Middlebury frames, which are visible-light. The 2026-09-01 SWIR
      survey found no SWIR (or NIR/thermal) dense-disparity benchmark with
      comparable ground truth. Options from the literature: (a) evaluate on
      visible-light data and argue the resource/timing surface transfers
      (the cost × aggregation surface is largely sensor-agnostic; disparity
      *quality* is not), (b) a small SWIR rig with time-synced LiDAR for
      sparse metric ground truth, (c) synthetic SWIR. Pick one and record
      the caveat before Track C benchmarking.
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
- [ ] **SWIR camera LVDS interface vs. GateMate LVDS-GPIO/SerDes fit** —
      the QDI sensor's LVDS lane count and per-lane bit rate haven't been
      checked against GateMate's DDR-GPIO LVDS timing (no max LVDS toggle
      rate found in the CCGM1A1 datasheet's electrical tables) or against
      whether the die's separate 5.0 Gb/s SerDes block would be needed
      instead. Concrete next step before Track A camera bring-up (Gate B).

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
