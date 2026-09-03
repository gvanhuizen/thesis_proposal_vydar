# Implementation Plan: GateMate A1 Stereo Matching — Specs, Trade-offs, and What to Build

**Status:** draft synthesis — consolidates everything decided/researched to date
into one implementation direction. Not yet reviewed by advisor or council.
**Last updated:** 2026-09-01

## Purpose & how to read this

This document is the synthesis step between the research and the RTL. It does
four things, in order:

1. Consolidates the GateMate A1 / Olimex EVB / SWIR-sensor specs into one place
   (§2) — currently spread across `CLAUDE.md`, `thesis-proposal.md` §2/§3, and
   `research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`.
2. States the pros and cons of this device versus the FPGAs used in the surveyed
   literature (§3–§4).
3. Walks the logical chain from each hardware constraint to which stereo-matching
   techniques are viable on it (§5).
4. Lands on a **justified shortlist of configurations to implement**, with the
   reasoning for the picks and the rejections (§6–§7).

It is written to be read start-to-finish on its own. Two companion files go
deeper on specific parts and are not required reading here:

- `thesis_proposal/hardware-and-technique-comparison.md` — the neutral reference
  matrix: full device table, a 13-technique catalogue with per-technique "fit"
  verdicts, an exhaustive constraints/gaps list. This document is the opinionated
  layer on top of it.
- `thesis_proposal/thesis-proposal.md` — the research-process plan (Track A–D,
  gates, kill criteria, fallback thesis, open items). The shortlist in §6 here
  **refines** that document's Track C candidate list; it does not replace the
  Track structure or the schedule.

**Verification tags.** Following `thesis-proposal.md`'s convention, any number
that gates a design decision carries one of: `VERIFIED` (primary source + date),
`VENDOR-CONFIRMED` (direct vendor contact + date), or `ESTIMATED` (proxy/derived,
unconfirmed). Untagged numbers are plain arithmetic or qualitative.

---

## 2. GateMate A1 — consolidated spec sheet

### 2.1 FPGA fabric (Cologne Chip CCGM1A1)

| Resource | Figure | Notes |
|---|---|---|
| Logic elements | **20,480 CPEs**, arranged 160×128 `VERIFIED` | Each CPE = dual 4-input LUT-2 tree / one 8-input LUT-2 tree / 4-input MUX / 1–2-bit full adder / **2×2-bit multiplier**, selectable. |
| Flip-flops | **40,960** (2 per CPE) `VERIFIED` | |
| Fast arithmetic | dedicated carry & propagation lines between adjacent CPEs `VERIFIED` | Cheap ripple-carry chains; no separate carry fabric to fight for. |
| Hard multiply | **2×2-bit per CPE — nothing larger** `VERIFIED` | No hardened DSP/MAC block anywhere on the die. An 8-bit absolute-difference accumulator or a Hamming popcount tree is built entirely from chained CPEs. |
| Block RAM | **32 × 40 Kb = 1,310,720 bits ≈ 160 KB total** `VERIFIED` | Dual-port; 1–40-bit (TDP) or up to 80-bit (SDP) words; per-port ECC; two adjacent blocks cascade into a 64K×1 array. This is the entire on-chip memory budget. |
| PLLs | **4**, DCO 500–2500 MHz (1250–2500 MHz in speed mode) `VERIFIED` | Derive pixel/system clocks from these. |
| Core modes | 0.9 V low-power / 1.0 V economy / 1.1 V speed `VERIFIED` | Board-supported power-vs-fmax trade axis. |
| GPIO | **162** in 9 banks; each pair single-ended or **LVDS** (2.5 V nominal, to 1.8 V; on-die 100 Ω differential termination); **DDR-capable** (2 FFs per pad); MIPI D-PHY electrically compatible `VERIFIED` | **No hardened CSI-2 / MIPI receiver** — D-PHY compatibility is electrical only. No max LVDS toggle rate stated in the datasheet's electrical tables. |
| SerDes | separate dedicated **5.0 Gb/s** transceiver block `VERIFIED` | Distinct from the GPIO fabric; the fallback camera receiver if DDR-GPIO deserialisation is too slow. |
| Process | GlobalFoundries **28 nm**, EU-funded `VERIFIED` | "European 28 nm FPGA with an open-source toolchain." |

### 2.2 Board — Olimex GateMateA1-EVB

- **PSRAM:** 2× **LY68S3200SLT** (32 Mbit each, SOIC-8, board refs U7/U9),
  wired in parallel for a combined **8-bit bus at 100 MHz, SDR only — no DDR
  mode** `VERIFIED` (Olimex Rev.C KiCad schematic, 2026-08-06).
  - Theoretical ceiling: 8 bit × 100 MHz = **100 MB/s** `VERIFIED`.
  - Effective sustained: **~97.6 MB/s** `ESTIMATED` — burst-timing derivation
    (14-cycle QPI read overhead, tCEM 8 µs max CE#-low window, ≥50 ns tCPH
    recovery) from the LY68L6400 proxy datasheet; protocol overhead costs only
    ~2–3 %, not a meaningful cushion. Exact AC timing from the S3200's own
    datasheet is still open (low stakes).
  - **Single-port:** one shared bus for reads and writes — no concurrent
    read/write, no per-direction budget. This is why PSRAM cannot serve as a
    frame-staging buffer at ingest rate (§2.5); it does *not* constrain a
    streaming datapath, which bypasses PSRAM entirely.
  - Capacity: **8 MiB total ≈ 14 single-camera frames ≈ 7 stereo pairs**
    `VERIFIED`, before any multi-frame buffering.
- **RP2040:** USB-C → JTAG/SPI configuration bridge only. **Not** a compute
  co-processor and not a usable data path.
- **Output:** VGA only (no HDMI/DVI; HDMI demos in the community use an external
  TMDS IC).
- **Expansion:** one PMOD + one UEXT (both level-shifted); 2 MB configuration
  flash. **No camera connector on the base board** — sensor integration is an
  expansion-board / flywire problem.

### 2.3 Toolchain

Fully open: `yosys` (synthesis, Verilog/SystemVerilog/VHDL) → `nextpnr-himbaechel`
(place & route, GateMate backend) → `gmpack` (bitstream) → `openFPGALoader`
(programming). Bundled in YosysHQ's OSS CAD Suite, ~25 MB install. Amaranth /
SpinalHDL / Silice front ends also target it. **No C++ HLS front end.**

Maturity signal is mixed: install and basic bring-up reported "very nice and
simple," multiple RISC-V soft cores and video-timing demos work end-to-end — but
**timing closure is explicitly unverified** by the one hands-on reviewer who
looked, there are **no public fmax/utilisation numbers for any real GateMate
design**, and the `nextpnr-himbaechel` GateMate backend is young enough to have
no public bug corpus.

### 2.4 Sensor (decided 2026-08-06)

QDI Systems SWIR (InGaAs) camera, **640×512, 14-bit, 400–1700 nm**. The 50 fps
spec-sheet figure is nominal; the real ceiling is the LVDS output interface,
rated **700 fps at full resolution** `VENDOR-CONFIRMED` (direct vendor contact,
2026-08-06) — comfortably above the 600 fps product target. Interface is
Camera-Link-style LVDS (clock + parallel data lanes), **not** MIPI CSI-2, so the
`camIO` open-hardware extension board does not apply and no off-the-shelf adapter
exists. Lane count and per-lane bit rate versus GateMate's DDR-GPIO limit are
still unchecked (open item).

### 2.5 Bandwidth arithmetic — the single most shaping fact

| Quantity | Value | Status |
|---|---|---|
| Camera raw ingest @ 600 fps | **344 MB/s** (401 MB/s at the 700 fps ceiling) | `VENDOR-CONFIRMED` (LVDS ceiling) |
| Raw frame size | 573,440 bytes (560 KiB) | arithmetic |
| PSRAM effective throughput | **~97.6 MB/s**, single-port | `ESTIMATED` |
| PSRAM capacity | ~7 stereo pairs | `VERIFIED` |
| On-chip BRAM | 160 KB total | `VERIFIED` |

**Why streaming is forced — a reductio, not a description of the pipeline.**
Take the obvious "capture full frames into PSRAM, then process" design: it
writes incoming pixels to PSRAM at 344–401 MB/s per camera and reads them back
for matching at a comparable rate — on one single-port bus that tops out near
~97.6 MB/s. That is 7–8× over budget before stereo doubling. So frame staging
in PSRAM is impossible, and the pipeline **must stream**: the camera's LVDS
lanes feed the FPGA fabric directly, pixels flow through on-chip line buffers
(out of the 160 KB BRAM pool) into the matching datapath, and disparity comes
out row-by-row. Hard architectural constraint, not a performance goal.

**What the streaming pipeline does with PSRAM — almost nothing.** In the ideal
streaming design the camera→match→output datapath never touches PSRAM, so the
344–401 MB/s ingest figure and the ~97.6 MB/s PSRAM ceiling are on different
paths and never compete. PSRAM's residual role:

- **Rectification (the main one — unsized, see §8).** If the two cameras are
  aligned well enough that epipolar correction only shifts pixels a few rows,
  rectification stays in on-chip line buffers. If not, rectified row *N* needs
  raw rows *N ± k*, and once *k* exceeds the on-chip line-buffer budget a frame
  buffer in PSRAM is required — which puts an ingest-rate write plus a warped
  read-back onto the single-port bus and re-creates the exact bottleneck the
  streaming design exists to avoid.
- **Output / display buffer** if VGA frame rate ≠ capture rate (it won't match
  — nothing displays 600 fps).
- **Track C test-vector playback** — stored KITTI/Middlebury frames sit in
  flash or PSRAM and are streamed in to mimic the camera; read-mostly, lighter.

**Capacity may bite before bandwidth.** 8 MiB ≈ 7 stereo pairs `VERIFIED` — so
any residual PSRAM use above works against a tight capacity budget too.

---

## 3. The literature baseline — what we are comparing against

The two surveys in `research/summaries/` cover 14 papers. Every one runs on a
device from this list:

| Device family | Hard DSP/MAC | On-chip BRAM | Hard CPU | Toolchain |
|---|---|---|---|---|
| Xilinx Virtex-7 | ~2,800 DSP48E1 | tens of Mb | none | Vivado (proprietary) |
| Xilinx Virtex-5 | ~192 DSP48E | several Mb | none | ISE (proprietary) |
| Xilinx Zynq UltraScale+ | ~2,520 DSP slices | tens of Mb | quad A53 + dual R5 | Vivado (proprietary) |
| AMD Versal ACAP | AI Engine array (very large) | tens of Mb | ARM cores | Vitis (proprietary) |
| Altera Stratix IV | ~1,024 18×18 DSP | tens of Mb | none | Quartus (proprietary) |
| Altera Stratix V | ~3,926 18×18 DSP | tens of Mb | none | Quartus (proprietary) |
| **Cologne Chip GateMate A1** | **2×2-bit / CPE** | **160 KB total** | **none** | **open (yosys/nextpnr)** |

Headline results worth carrying forward (see the surveys for the full set):

| Design | Cost fn × aggregation | Device | Res / disp | fps | Resource note |
|---|---|---|---|---|---|
| Ambrosch 2009 (S2) | SAD, no aggregation | FPGA (2009-era) | 450×375 / d≤150 | **600** | zero cross-pixel dependency |
| Papadimitriou 2013 (S3) | AD+Census, cross-based + Belief Propagation | Virtex-5 | 400×320 | **1570** | global method, beats pure SAD by shrinking the frame |
| Shrivastava 2020 (P10) | Census, dependency-relaxed SGM + N PUs | Virtex-7 | 1280×960 / d64 | 322 | fps ~linear in PU count; **+0.12 disp err, +1.96 % bad-pixel per PU** |
| Ma 2022 (P8) | SGM + sub-pixel + 5-dir occlusion fill | Stratix IV | 640×480 | 320 | **5.6K LUT, 1.459 W** (post-proc only) |
| Zhao 2020 FP-Stereo (S5) | SAD/ZSAD/Census/Rank swappable, SGM | Xilinx | 1242×375 | 161 | 2× faster, 30 % less resource, 40 % less energy vs. xfOpenCV SGM |

**Two facts to take from this table.** (1) The highest-fps results (S3 1570, S2
600) got there by **shrinking resolution and disparity range**, not by having
more silicon — encouraging, because it means the fps target does not require
Xilinx/Altera-class resources. (2) The only concrete resource numbers in either
survey are LUT/BRAM counts on other vendors' fabrics. A GateMate CPE is not a LUT
(it bundles LUT + adder + 2×2 multiplier + 2 FFs), so **none of these numbers
transfer** — which is exactly why the thesis treats synthesis-backed
characterisation (Track B) as the experiment, not a preliminary.

---

## 4. Pros and cons of GateMate A1 versus the literature devices

### 4.1 Cons — where this device is harder to build on

- **No hardened DSP/MAC.** Every surveyed device has hundreds-to-thousands of
  DSP slices; GateMate's largest hard multiply is 2×2-bit. Any wide
  multiplication is chained CPEs competing with the rest of the design.
- **160 KB of on-chip BRAM, total.** Smaller than a *single* surveyed design's
  BRAM count implies (Liang 2024's 101 BRAM blocks ≈ 3.6 Mb ≈ 450 KB — ~3× this
  device's entire budget). Line buffers, cost volumes, aggregation memories all
  come out of this one pool.
- **PSRAM is small and slow, and its single port cannot stage frames.**
  ~97.6 MB/s effective, single-port, ~7 stereo pairs of capacity. This does not
  bottleneck a streaming datapath — which bypasses PSRAM (§2.5) — but it rules
  out any frame-staging design outright and squeezes whatever residual PSRAM use
  rectification turns out to need. No surveyed paper had to design around a
  memory subsystem this constrained relative to its sensor.
- **No hardened CPU.** Zynq designs offload control-plane work (rectification
  parameters, sensor configuration, output framing) to ARM cores. Here it must
  live in fabric FSMs or a RISC-V soft core, spending CPEs.
- **VGA-only egress, no native CSI/MIPI receiver.** Output and camera ingress
  both need custom interfacing.
- **Immature toolchain on this exact part.** Timing closure unverified in
  practice; no public fmax/utilisation data to sanity-check against; young
  `nextpnr-himbaechel` backend; no stereo-vision or image-pipeline project in any
  GateMate community directory.

### 4.2 Pros — where this device is genuinely favourable

- **Fully open toolchain.** Source of the thesis's novelty and a citable
  secondary result ("first open-toolchain stereo bitstream"); also reproducible
  end-to-end and ~$100 of hardware versus a multi-thousand-dollar Xilinx eval
  kit.
- **LUT-heavy, DSP-less fabric is a real *fit* for Hamming-distance cost
  functions.** Census/Rank matching is XOR + popcount — pure LUT logic, no
  multiply. A DSP-rich device spends hardened multiplier silicon it can't avoid
  on something that never needed a multiply; GateMate does not. This is the one
  place the device's shape is an advantage rather than a tax.
- **20,480 independent small CPEs invite spatial-tile parallelism.** Running N
  cost-aggregation engines across image tiles is unexploited in the surveyed
  literature. (Why it's unexploited isn't established — could be that DSP-rich
  pipelines already hit target fps, could be that tiling's arbitration/stitching
  complexity wasn't worth it. Treat it as a candidate scaling lever, not a
  default — see §6.)
- **Three core voltage/performance modes** give a board-supported power-vs-fps
  axis the literature devices didn't expose the same way.
- **Native LVDS + DDR-capable GPIO with on-die termination** can likely receive
  the SWIR camera's LVDS lanes directly, no adapter board, if the per-lane rate
  is within DDR-GPIO limits (unchecked).
- **The fps target does not need big-FPGA resources.** Per §3, the literature's
  top fps came from small frames and small disparity ranges, both of which this
  device can do.

---

## 5. From constraints to technique choices

Each hardware fact drives a specific algorithmic consequence:

- **No DSP/MAC + LUT-heavy fabric →** favour **Census/Rank** (Hamming distance,
  LUT-native) and **plain SAD** (subtract + accumulate is fine on the carry
  chains). Avoid cost functions that pull in wide multiplication. **CNN /
  learned matching is ruled out** — its 2D/3D convolution stages depend on a
  hardened MAC array (DSP slices or an AI Engine tile array) this device does not
  have.
- **160 KB BRAM + PSRAM ceiling →** a **streaming / line-buffer datapath is
  mandatory**. This bounds (resolution × disparity range × window size) to what
  fits in on-chip line buffers. It disfavours full cost-volume materialisation,
  makes 4ppc line-widening expensive (4× parallel BRAM ports out of one small
  pool), and makes **Belief Propagation a tight fit** — its per-node memory
  scales with disparity range × iteration count, exactly what the BRAM ceiling
  bites. BP is not ruled out (S3 hit 1570 fps with it), but only at small
  resolution/disparity/iteration counts, and only with Track B numbers first.
- **Serial cross-pixel dependency chains are the classic throughput cap, and
  this is a young PnR with unproven timing closure →** strongly favour designs
  with **no serial chain anywhere** (fixed-window block matching). If SGM is
  attempted at all: prefer **dependency-relaxation + PU replication** (generic
  logic, more copies of simpler datapath) over **comparator-tree min-search**
  (a lot of chained combinational CPE logic at D≈64–128, the kind of structure
  most likely to miss timing on an immature toolchain); and prefer **dual-path
  (H+V)** over 8-path, which removes the width-scaling diagonal buffers.
- **PSRAM cannot stage frames + forced streaming →** the whole
  camera→match→output datapath must fit on-chip; there is no "spill the cost
  volume to DRAM" escape hatch the literature designs lean on. How a
  cost-function / aggregation choice behaves under that hard on-chip ceiling is
  itself part of the trade-off surface the research question asks about. Keep
  the per-row working set small. (Residual PSRAM traffic — rectification,
  output, test vectors — is a separate budget; see §2.5.)
- **No hardened CPU →** the control plane is small fabric FSMs or a RISC-V soft
  core. Working GateMate soft-core ports (FemtoRV, NEORV32, LiteX VexRiscv/Serv)
  exist and double as a low-risk Track A Gate A bring-up vehicle.
- **Post-processing is cheap and buys accuracy →** sub-pixel interpolation
  (LUT + Newton division, no hardware divider), left-right consistency check, and
  occlusion filling together cost ~5.6K LUT (Ma 2022) and add multi-% accuracy.
  Budget them as **optional parallel pipeline stages**; they're needed once the
  accuracy/quality bar (open item) is defined.
- **Track B is a synthesis sweep →** all candidate RTL must be **parameterised**
  (resolution, disparity range, window size as synthesis-time generics), so one
  codebase elaborates every point in the sweep.

---

## 6. Implementation plan — the justified shortlist

### 6.1 Commitments shared by every candidate

- **Streaming line-buffer datapath.** The camera→match→output path holds its
  working set in on-chip BRAM and does not stage frames in PSRAM. (Rectification
  is the one stage that may still need a PSRAM frame buffer — unresolved, §8.)
- **Parameterised RTL.** Resolution / disparity range / window size as generics.
- **Fixed BRAM budget.** 160 KB is a hard line item in every Track B sweep,
  alongside CPE count and achievable clock.
- **Post-processing as optional parallel stages**, switched in once the accuracy
  bar exists.
- **Start small, let Track B set the ceiling.** Initial target: native 640×512
  (or downscaled), disparity range ≈64. These are starting points for the sweep,
  not claims.

### 6.2 The configurations

Each is a point on the (cost function × aggregation strategy × resolution/
disparity) surface from the research question. They are ordered by ascending
build risk.

**Config 1 — Census + fixed-window block matching, streaming. *(primary, lowest risk)***
- *What:* Census transform per pixel, Hamming-distance cost, fixed W×W support
  window, winner-take-all disparity, no path aggregation.
- *Why on the list:* no serial cross-pixel dependency anywhere; Hamming is
  LUT-native (the one place the device's shape helps); smallest timing-closure
  risk of any real matcher; directly tests "does GateMate's natural strength
  actually pay off in synthesised fmax/CPE numbers."
- *Trade-off point tested:* robust cost function, cheapest aggregation.
- *Main risk:* fixed-window edge fattening hurts accuracy at depth
  discontinuities — mitigated later by Config 3's aggregation or by
  post-processing.

**Config 2 — Pure SAD + incremental moving-sum, streaming. *(baseline)***
- *What:* SAD over a fixed window with the box-filter O(1) window-sum update
  (Ambrosch 2009), winner-take-all, no aggregation.
- *Why on the list:* a direct hardware blueprint exists; simplest possible
  arithmetic (subtract, abs, add on the carry chains); the literature's canonical
  high-fps local matcher. It is the control point for whether Census's extra
  per-pixel hardware (bit-encoding + popcount) is worth it *on this fabric* —
  a question no surveyed paper answers for a DSP-less device.
- *Trade-off point tested:* cheapest cost function, cheapest aggregation.
- *Main risk:* radiometric fragility (two SWIR cameras with any gain/offset
  mismatch) — which is precisely what Config 1 buys back.

**Config 3 — AD-Census + cross-based aggregation, streaming. *(accuracy-leaning stretch)***
- *What:* AD-Census fused cost (independently capped AD and Census terms) +
  cross-based variable-support aggregation (four arm-length registers per pixel),
  winner-take-all.
- *Why on the list:* isolates the **aggregation axis** — same LUT-native cost
  family as Config 1, plus a per-pixel adaptive window that fixes edge fattening
  for very little memory (registers, not buffers). Measures the accuracy gain and
  the CPE/BRAM cost of moving one axis while holding the other fixed — the exact
  separable-axes experiment the research question is built around.
- *Trade-off point tested:* moderate cost function, adaptive aggregation.
- *Main risk:* cross-based arm computation adds a data-dependent stage; needs a
  Track B synthesis run to confirm it stays cheap on CPEs.

**Config 4 (optional) — dual-path SGM via dependency-relaxation. *(only if Track B shows headroom)***
- *What:* horizontal + vertical path aggregation only, recursion reading the
  predecessor n pixels back (n = 4 or 8) to break the serial chain, datapath
  replicated across n PUs.
- *Why maybe:* tests the aggregation axis at its expensive end without a wide
  comparator tree. Dependency-relaxation is the SGM parallelism lever that suits
  a device with no comparator-friendly hard primitive.
- *Why optional:* only worth the build effort if Config 1–3 leave CPE/BRAM
  headroom and the accuracy bar demands stronger smoothing. Carries a real,
  quantified accuracy cost (+0.12 disp error, +1.96 % bad-pixel per PU).

### 6.3 What to build first (Track A, Gate A)

Do **not** bet Gate A on a full pipeline. Two honest "non-trivial but bounded"
options:

1. The **Census-transform + Hamming block** from Config 1, fed a stored test
   pattern, result to VGA — exercises real arithmetic, real line buffers, and the
   full synth→P&R→flash→observe loop without the disparity search or the camera.
2. A **RISC-V soft-core port** (NEORV32 / FemtoRV) — de-risks the toolchain
   against a design known to close elsewhere, and gives the control plane a home.

Option 1 is preferred if the goal is to surface pipeline-specific toolchain
problems early; Option 2 if the goal is the lowest-risk possible Gate A pass.

---

## 7. Why this and not that — explicit rejections

Each traces to a constraint in §5:

- **CNN / learned stereo matching** — needs a hardened MAC array for the 2D/3D
  convolution stages. GateMate has a 2×2-bit multiplier per CPE and nothing
  larger. Not a tuning problem; structurally out.
- **8-direction SGM** — the four diagonal paths need buffers that scale with
  image width, out of a 160 KB pool, and add serial recursion chains on an
  unproven-timing toolchain. Dual-path (Config 4) is the only SGM form
  considered, and only optionally.
- **Comparator-tree min-search as the primary SGM speed-up** — a wide balanced
  combinational reduction over D≈64–128 disparities is a lot of chained CPE logic
  and the structure most likely to miss timing on a young PnR. Dependency-
  relaxation is the preferred lever if SGM is attempted at all.
- **FP-Stereo / HLS code reuse** — no C++ HLS front end for this toolchain. The
  *methodology* (vary cost function, hold aggregation fixed, sweep resource/fmax)
  is the right template for Track B; the *code* must be re-implemented in RTL.
- **"Capture full frames to PSRAM, then process"** — ruled out by §2.5. Every
  candidate above is streaming by construction.

---

## 8. Open dependencies that gate this plan

These must resolve before the shortlist can be finalised or ranked. The
authoritative list with owners and dates is `thesis-proposal.md` §7 — not
duplicated here, only the ones that bear directly on §6:

- **Accuracy / disparity-quality bar is undefined.** Track C cannot rank Config
  1–4 against each other without a depth-quality metric (e.g. bad-pixel-% versus
  KITTI/Middlebury ground truth at a stated threshold) to weigh against fps.
- **SWIR camera LVDS interface vs. GateMate DDR-GPIO / SerDes fit is unchecked.**
  Gates the Config choice only indirectly, but gates Track A Gate B directly.
- **Rectification buffer is unsized.** Whether epipolar rectification fits in
  on-chip line buffers or needs a full frame buffer in PSRAM depends on the
  stereo rig's mechanical alignment (vertical correction budget, in rows) versus
  the BRAM left after the matching core. If it needs PSRAM it re-introduces an
  ingest-rate write plus read-back on the single-port bus (§2.5) and becomes a
  first-order constraint on the whole design, not a residual. Size this before
  committing Track D's integration architecture — ideally alongside the
  SWIR-camera interface check above.
- **Exact PSRAM AC timing** still from a proxy datasheet — affects the precision
  of ~97.6 MB/s, not the streaming conclusion.
- **Track B must supply real fmax / CPE / BRAM numbers per config** before any of
  §6.2 is more than a hypothesis. The whole point of the shortlist is to be the
  input to that sweep, not its conclusion.
- **No dataset-plumbing path for Track C** — how stored KITTI/Middlebury frames
  reach the device (UART? flash? streamed to mimic the camera) is unspecified and
  owned by neither Track B nor Track C.

---

## Sources

- `research/summaries/2026-08-05-fpga-stereo-vision-pipelines.md`
- `research/summaries/2026-08-05-sad-census-stereo-matching.md`
- `research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`
- `research/stereo-vision-techniques-explained.md`
- `research/top-5-papers-to-read.md`
- `thesis_proposal/hardware-and-technique-comparison.md`
- `thesis_proposal/thesis-proposal.md`
- `council/council-transcript-2026-08-05_1752.md`,
  `council/council-transcript-2026-08-05_2141.md`,
  `council/council-transcript-2026-08-06_1043.md`
- Repo top-level `CLAUDE.md`
