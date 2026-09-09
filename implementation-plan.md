# Implementation Plan: GateMate A1 Stereo Matching — Specs, Trade-offs, and What to Build

**Status:** draft synthesis — consolidates everything decided/researched to date
into one implementation direction. Not yet reviewed by advisor or council.
**Build target as of 2026-09-09: Config 7** (Route C — band-limited
disparity-range sweep). Configs 1–6 are retained below as the alternatives
considered and set aside; see the 2026-09-09 note and `product-plan.md` §4 / §8.
**Last updated:** 2026-09-09

> **2026-09-03 — goal re-centred (founder direction).** The goal is
> **real-time, high-fps detection of whether the camera is close to an object**
> (close-range / near-object detection), **not** a dense depth map — a sparse or
> low-density output is acceptable and, given the bandwidth budget, preferable.
> This changes the shortlist: the **density axis (dense → semi-dense → sparse)**
> is now first-class, and **semi-dense seed-and-grow** and a **bounded sparse
> key-point** pipeline join Census/SAD as *built and benchmarked* configs, not
> synthesis-only rows. See `thesis-proposal.md` §1 and its decision log for the
> full reframe. The dense-matching detail below is still accurate for the dense
> configs; read "the trade-off surface the research question asks about" as "the
> feasibility + approach-selection question §1 now asks."
>
> **2026-09-07 — organised into three peer routes; Config 7 added.** Matching the
> proposal-defense deck (`../presentations/proposal-defense/deck.md`), the menu
> below was grouped into three **peer routes, not a ranking**: **Route A** a
> dense disparity map (Configs 1–4, 6), **Route B** a bounded sparse key-point
> matcher (Config 5), **Route C** a direct band-limited plane-sweep /
> disparity-threshold trigger (**Config 7**, folded in from
> `../stereo_camera_fpga/design/CLAUDE.md`).
>
> **2026-09-09 — approach decided: Config 7 (Route C).** The engineering approach
> is decided (founder direction, `product-plan.md` §4 / §8): a **band-limited
> disparity-range sweep** — cost at the `K` watch-band planes only, gate / reduce
> / confirm, boolean-or-zoned readout; no per-pixel disparity map, no sparse
> key-point set. **Config 7 is the build target.** Configs 1–6 stay below as the
> record of the alternatives weighed and as M2 comparison points if a synthesis
> result forces a rethink; they are no longer candidates for selection. The
> output granularity is settled (boolean/zoned); `K`, boolean-vs-zoned, and
> whether a key-point pre-gate earns its place are what M2/M3 fix
> (`product-plan.md` §5).
>
> **Where this sits.** [`product-plan.md`](./product-plan.md) is the primary
> driver: it owns the goal, the output spec, and the decided engineering approach
> (§4). **This document carries the RTL detail** — the configurations in §6, with
> the constraint→technique reasoning (§5) and the rejections (§7). The build
> target (Config 7) is set by `product-plan.md` §4.

## Purpose & how to read this

This document is the synthesis step between the research and the RTL. It does
four things, in order:

1. Consolidates the GateMate A1 / Olimex EVB / SWIR-sensor specs into one place
   (§2) — currently spread across `../stereo_camera_fpga/CLAUDE.md`,
   `thesis-proposal.md` §2/§3, and
   `../stereo_camera_fpga/research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`.
2. States the pros and cons of this device versus the FPGAs used in the surveyed
   literature (§3–§4).
3. Walks the logical chain from each hardware constraint to which stereo-matching
   techniques are viable on it (§5).
4. Lands on the **decided configuration to implement** (Config 7, the
   disparity-range sweep), with the alternatives that were weighed and the
   rejections (§6–§7).

It is written to be read start-to-finish on its own. Two companion files go
deeper on specific parts and are not required reading here:

- `../stereo_camera_fpga/research/synthesis/hardware-and-technique-comparison.md`
  — the neutral reference matrix: full device table, a 13-technique catalogue
  with per-technique "fit" verdicts, an exhaustive constraints/gaps list. This
  document is the opinionated layer on top of it.
- `thesis-proposal.md` (this folder) — the research-process plan (Track A–D,
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

The two surveys in `../stereo_camera_fpga/research/summaries/` cover 14 papers.
Every one runs on a device from this list:

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
transfer** — which is exactly why the thesis needs its own synthesis-backed
feasibility numbers (Track B) before any approach can be ranked (Track C), not
just a lit-review estimate.

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
  volume to DRAM" escape hatch the literature designs lean on. This favours
  approaches that emit *less* over a full dense map, and — with the 2026-09-09
  decision — is what makes the **band-limited plane-sweep** the approach: it never
  materialises a cost volume at all. Keep the per-row working set small. (Residual
  PSRAM traffic — rectification, output, test vectors — is a separate budget; see
  §2.5.)
- **Close-range goal → density is not fixed at "dense," and the decided approach
  emits the least of any.** "Is the camera close to an object" needs a reliable
  near-field match signal, not a per-pixel map. The **direct band-limited
  plane-sweep** (Config 7 / Route C, decided 2026-09-09) evaluates the Hamming
  cost only at the `K` planes of the watch band, gates + reduces + confirms, and
  emits a boolean/zoned readout — never a cost volume or a map. (The sparse
  key-point and semi-dense seed-and-grow forms that were also considered under
  this heading are recorded in §6.2 as set aside.)
- **Moving platform + no static background model → the false-positive defence
  must come from the current frame(s), not a calibrated background.** `d_bg`
  cannot be learned (the platform moves), and online background learning on the
  FPGA is rejected (`product-plan.md` §3.1). So a fixed disparity threshold is
  not by itself "closer than D": Config 7 leans on a **per-frame far-prior**
  (Delaunay-free ELAS envelope from this frame's own confident pixels), **spatial
  coherence** (U/V-disparity runs), and a **two-frame Δd** approach test — all
  frame-local, all cheap. The ~600 m/s regime collapses any long-window temporal
  filter to that 2-of-3 / fast-path Δd form.
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

## 6. Implementation plan — the decided configuration and the alternatives

**Build target (2026-09-09): Config 7** — the band-limited plane-sweep /
disparity-threshold trigger (§6.2). Configs 1–6 below are the alternatives that
were weighed and set aside; they are kept for the record and as M2 comparison
points, not as candidates for selection.

### 6.1 Commitments shared by every configuration

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

**Config 7 (Route C) is the decided approach** — see the 2026-09-09 note above and
`product-plan.md` §4. Configs 1–6 are the alternatives that were considered along
the **density axis** (dense → semi-dense → sparse) and set aside; they remain
useful as the reasoning trail and as M2 comparison points. Ordered by ascending
build risk.

| Route | What it commits to | Configs | Status |
|---|---|---|---|
| **A — dense disparity map** | one disparity per pixel (optionally reduced to semi-dense); for a band decision, run the dense datapath as a single/few-plane occupancy test | 1, 2, 3, 4, (6) | considered, set aside |
| **B — bounded sparse key-point** | a short, capped list of matched near points; no map | 5 | considered, set aside (kept as an additive fast-path option, not the front end) |
| **C — direct range-threshold plane-sweep** | a boolean/zoned occupancy readout; never `argmin`, never a map | **7** | **DECIDED — build target** |

The work started aimed at Route A; the close-range goal relaxes the expensive
parts (`product-plan.md` §2.8), and the 2026-09-09 decision took that to its
conclusion — the range sweep delivers the decision at the smallest footprint. `K`,
boolean-vs-zoned and the key-point-pre-gate question are what M2/M3 now settle.

**Config 1 — Census + fixed-window block matching, streaming. *(Route A — considered, set aside 2026-09-09; its Census/Hamming datapath is reused by Config 7)***
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

**Config 2 — Pure SAD + incremental moving-sum, streaming. *(Route A — considered, set aside 2026-09-09)***
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

**Config 3 — AD-Census + cross-based aggregation, streaming. *(Route A — considered, set aside 2026-09-09)***
- *What:* AD-Census fused cost (independently capped AD and Census terms) +
  cross-based variable-support aggregation (four arm-length registers per pixel),
  winner-take-all.
- *Why on the list:* tests whether adaptive aggregation is worth its cost for
  the proximity goal, or whether it can be dropped. Same LUT-native cost family
  as Config 1, plus a per-pixel adaptive window that fixes edge fattening for
  very little memory (registers, not buffers). Measures the detection-quality
  gain and the CPE/BRAM cost of adding aggregation while holding the cost
  function fixed.
- *Trade-off point tested:* moderate cost function, adaptive aggregation.
- *Main risk:* cross-based arm computation adds a data-dependent stage; needs a
  Track B synthesis run to confirm it stays cheap on CPEs.

**Config 4 — semi-dense seed-and-grow / ELAS-style, streaming. *(Route A — considered, set aside 2026-09-09)***
- *What:* Census/Hamming confident "support points" along the epipolar row →
  guided disparity fill to neighbours along image gradients, the growth bounded
  entirely on fabric (fixed fan-out, fixed pass count — no ARM core).
- *Why on the list:* the honest middle of the density axis under SWIR texture
  starvation — reintroduces a smoothness prior at a fraction of dense cost, and
  emits far less than a full map. A genuine contribution: every surveyed
  seed-and-grow system used an ARM core to do the growing.
- *Trade-off point tested:* confident-seed cost, cheap propagation, semi-dense
  output — enough surface coverage to answer "is something close" robustly.
- *Main risk:* bounding the growth on fabric without the data-dependent tail;
  needs a Track B run to confirm the fan-out logic stays cheap.

**Config 5 — bounded sparse key-point correspondence, streaming. *(Route B — considered, set aside 2026-09-09 as a front end; the detector is kept as a possible additive fast-path pre-gate inside Config 7)***
- *What:* single-scale FAST/Harris detector → binary descriptor (BRIEF) or
  reused Census word → 1-D along-row Hamming search. **Bounded**: hard
  key-point cap, spatial buckets, top-K by response, so all buffers and cycle
  counts are static. Verification = left-right consistency + ordering +
  fixed epipolar-row-offset threshold (from calibration) + parabola sub-pixel.
  **No RANSAC, no triangulation, no variable-length scatter/gather** — the
  proximity goal ("is there a close object") needs a bounded set of near
  matches, not a point cloud, so the CPU-bound irregular tail every surveyed
  sparse FPGA system offloads is simply not built.
- *Why on the list:* directly aligned with the goal, lowest output bandwidth,
  and it frees the aggregation stage + dense-map write traffic — spend the
  freed CPE/BRAM on wider disparity search or more spatial tiles. The Census
  datapath from Config 1 is reused almost unchanged as the descriptor + matcher.
- *Trade-off point tested:* no aggregation, sparse output — the cheapest point
  on the density axis, and the question is whether SWIR key-point density in
  the target (fog/glass/low-light) scenes is enough for a reliable "close?"
  decision.
- *Main risk:* SWIR texture starvation → too few / too-weak key-points in
  exactly the scenes the work targets. Config 4 is the hedge if so.
- *Related use — keypoint as a pre-gate, not a standalone matcher:* the same
  detector can gate what enters a dense few-plane trigger chain (only key-point
  pixels run the plane-sweep + FP filters) instead of producing the output
  itself. **Ups:** removes most textureless-FP machinery by construction, can
  shrink the K-plane Hamming bank, stays coreless under a fixed key-point cap.
  **Downs:** motion blur (600 m/s ego-motion attenuates corner content — Census
  degrades gracefully, a blurred corner does not survive) and the thin-statistics
  follow-on (2–4 frames of warning; a handful of key-points per object starves
  the spatial/temporal FP tests that a dense vote field keeps fed). Not a sole
  front-end on FN grounds; viable as an additive fast-path channel or a
  low-key-point exception path. *Cap-free form:* keep the full raster with a
  per-pixel valid / annotation bit (no grid buckets, no top-K, no compaction) —
  removes all list-sequencing, but forgoes the K-plane-bank shrink (no cap ⇒ bank
  sized for peak ⇒ power saving only, not CPE/fmax), and fits the additive-channel
  role, not the input-veto one. Full ups/downs in
  `../stereo_camera_fpga/design/CLAUDE.md` (keypoint pre-gate variant).

**Config 6 — dual-path SGM via dependency-relaxation. *(Route A — considered, set aside 2026-09-09)***
- *What:* horizontal + vertical path aggregation only, recursion reading the
  predecessor n pixels back (n = 4 or 8) to break the serial chain, datapath
  replicated across n PUs.
- *Why maybe:* tests the aggregation axis at its expensive end without a wide
  comparator tree. Dependency-relaxation is the SGM parallelism lever that suits
  a device with no comparator-friendly hard primitive.
- *Why optional:* only worth the build effort if Config 1–5 leave CPE/BRAM
  headroom and the detection-reliability bar demands stronger smoothing.
  Carries a real, quantified accuracy cost (+0.12 disp error, +1.96 % bad-pixel
  per PU).

**Config 7 — band-limited plane-sweep / disparity-threshold trigger, streaming. *(Route C — DECIDED 2026-09-09, the build target; timing-closure risk on the `K`-plane bank is the open question M2 resolves)***

Folded in 2026-09-07 from a 2026-09-04 design session
(`../stereo_camera_fpga/design/CLAUDE.md`, where the per-stage reasoning and the
two feeding surveys live). Established terminology: **band-limited plane-sweep
stereo with a disparity-threshold occupancy readout** — a degenerate plane-sweep
that tests one or a few disparity hypotheses, decides, and does **not** `argmin`
and does **not** emit a depth map.

- *What — a streaming, line-buffered pipeline, no frame buffer anywhere*
  (PSRAM carries only the rectification coefficients):
  - **Stage 0 — rectification** on the fly from ~a dozen polynomial
    lens-distortion + rotation-homography coefficients, bilinear-sampled; the
    vertical-remap window is fused into the Stage 1 line buffer (no per-pixel
    LUT — it is ~16× BRAM / ~15× PSRAM).
  - **Stage 1 — banded plane-sweep + per-pixel pre-filter.** Census-transform
    both images; compute the Hamming cost at the `K ≈ 30–50` planes of the watch
    band `d_watch..d_max`; emit the winning plane index only for pixels that pass
    three orthogonal rejects — a **texture / squared-gradient gate** (Konolige
    SVS), a **curve-shape reject** (3-tap curvature / peak-ratio / second-minimum
    — the one that catches textureless; LRC alone misses it), and **LRC /
    uniqueness** (Di Stefano single-pass).
  - **Stage 1.5 — per-frame far-prior.** A coarse ~40×32-cell disparity envelope
    from *this* frame's own confident pixels; a near-band pixel is promoted only
    if the local coarse prior is also near-band, or the match is high-confidence
    and spatially coherent. Replaces the calibrated background model a moving
    platform cannot have.
  - **Stage 2 — spatial reduction** (recommended: U/V-disparity histograms,
    Oleynikova / Irki): per-pixel stream → ≤ ~16 cluster records
    `{x̄, ȳ, area, d̄, bbox}` with no connected-component labelling.
  - **Stage 3 — two-frame Δd confirm.** At 600 fps an object is in-band ~2–4
    frames, so the long-window approach tracker collapses to a frame-to-frame
    disparity-jump test: associate to the `t−1` cluster within a position gate,
    flag `Δd ≥ Δd_min` + area-consistency, 2-of-3 hit counter, plus a
    single-frame-pair **fast path** for `Δd ≥ Δd_fast`.
  - **Stage 4 — alarm decision.** Boolean (any confirmed cluster with
    `d̄ ≥ d_alarm`, `d_alarm` set early to absorb latency) or zoned; hysteresis;
    GPIO/PMOD/UART readout. Any VGA/debug overlay is strictly downstream and
    never gates the alarm.
- *Why on the list:* the most goal-aligned route — smallest on-chip footprint of
  any candidate, disparity search already collapsed to `K` planes, no cost
  volume, no dense-map writes. The false-positive machinery (Stages 1.5–4) is
  ~300 CPE / ~1 BRAM block — nearly free; the cost is all in the Stage 1 matcher
  and the Stage 0 rectification band, paid by every route.
- *Trade-off point tested:* no aggregation, no map — the extreme low end of the
  density axis, with the FP defence moved entirely into frame-local priors +
  coherence + Δd.
- *ESTIMATED budget* (napkin arithmetic vs. the A1, **not** place-and-route;
  pixel rate ≈ 197 Mpix/s):

  | Stage | CPE | BRAM (of 32) | Mult | PSRAM |
  |---|---|---|---|---|
  | 0 Rectify | ~450 | ~9–10 (±16-row band) | ~8/pix | coeffs |
  | 1 Match + pre-filter | ~3,000–3,500 | ~3–4 (or folded into St.0) | 0 | none |
  | 1.5 Per-frame far-prior | ~150 | ~1–2 | 0 | none |
  | 2 Spatial reduction (U/V) | ~100 | < 1 | 0 | none |
  | 3 Two-frame Δd confirm | ~100 | 0 | 0 | none |
  | 4 Alarm decision | ~50–150 | 0 | 0 | none |
  | **Total** | **~4,000–4,500 (~20–22 %)** | **~15–18 (~half)** | St. 0 only | none in datapath |

- *Main risk:* the `K`-plane Hamming bank must run **combinational at pixel
  rate** — this is **the** timing-closure gate on the young `nextpnr-himbaechel`
  flow. Levers: shrink `K` (narrower watch band) or time-multiplex the Hamming
  units. Secondary: the Stage 0 ±16-row rectification band (~9–10 BRAM blocks) vs.
  everything else — needs a signed mechanical misalignment spec (`../TODO.md`
  item 3). Residual: a textureless close object with no textured rim → UNKNOWN
  state, not detection (`product-plan.md` §2.7).
- *Relationship to the other routes:* Config 7 is the detailed form of the "dense
  few-plane occupancy test" noted for Route A; a **key-point pre-gate** (Config 5
  detector gating which pixels enter the Stage 1 sweep) is a studied variant —
  viable as an additive fast-path channel, not as the sole front end on
  false-negative grounds (`../stereo_camera_fpga/design/CLAUDE.md`, variant
  section).

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
  unproven-timing toolchain. Dual-path (Config 6) is the only SGM form
  considered, and only optionally.
- **Full sparse-SLAM front end** (unbounded key-point list, brute-force /
  approximate NN over descriptor sets, RANSAC essential-matrix fit, Delaunay
  triangulation) — the data-dependent irregular tail every surveyed FPGA design
  offloads to a hardened CPU the A1 lacks. Config 5 keeps the *bounded* sparse
  matcher and drops this tail; the close-range goal does not need it.
- **Static background-disparity model / online background learning** (for
  false-positive suppression) — the platform moves, so `d_bg` cannot be
  calibrated, and an on-FPGA learner is stateful, memory-hungry, and can learn
  away a real intruder (`product-plan.md` §3.1). Config 7's FP defence is
  frame-local instead (per-frame far-prior + spatial coherence + two-frame Δd).
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

These must resolve before Config 7 can be parameterised and confirmed. The
authoritative open-items list with owners and dates is `thesis-proposal.md` §7;
the evaluation criteria are [`product-plan.md`](./product-plan.md) §5. Only the
items bearing directly on §6:

- **Close-range-detection quality bar is undefined.** M3 cannot confirm Config 7
  (or fix `K` / boolean-vs-zoned) without the detection metrics and weighting in
  `product-plan.md` §5 (detection reliability at the band edge, false-negative
  rate for close objects, false-positive rate, latency, min valid-match
  density). It is **not** dense bad-pixel-% versus KITTI/Middlebury. See
  `thesis-proposal.md` §7 for the research-side kill-criterion requirement.
- **Detection range vs. sensing geometry.** A ~12 cm baseline at 600 fps yields
  usable disparity only in the last few metres and ~2–4 frames of warning; if the
  required standoff needs a wider baseline / longer focal length / higher
  windowed-ROI frame rate, that is a geometry constraint bounding *every* config,
  diagnosable before M3 (`product-plan.md` §6 kill findings,
  `../stereo_camera_fpga/design/CLAUDE.md` §4).
- **IMU / ego-motion estimation on the platform?** Sets Config 7 Stage 3 —
  motion-compensated prediction (Barry & Tedrake) vs. plain N-of-M + monotonic-Δd
  (`../stereo_camera_fpga/design/CLAUDE.md` §7).
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

(paths relative to this folder; the FPGA-repo files are in the sibling
`../stereo_camera_fpga/`)

- `../stereo_camera_fpga/research/summaries/2026-08-05-fpga-stereo-vision-pipelines.md`
- `../stereo_camera_fpga/research/summaries/2026-08-05-sad-census-stereo-matching.md`
- `../stereo_camera_fpga/research/summaries/2026-08-06-gatemate-a1-fpga-overview.md`
- `../stereo_camera_fpga/research/summaries/2026-09-01-swir-stereo-depth-perception.md`
- `../stereo_camera_fpga/research/summaries/2026-09-04-plane-sweep-disparity-threshold-trigger.md`
  (Config 7)
- `../stereo_camera_fpga/research/summaries/2026-09-04-textureless-region-false-positive-suppression.md`
  (Config 7 pre-filters)
- `../stereo_camera_fpga/research/synthesis/stereo-vision-techniques-explained.md`
- `../stereo_camera_fpga/research/synthesis/top-5-papers-to-read.md`
- `../stereo_camera_fpga/research/synthesis/hardware-and-technique-comparison.md`
- `../stereo_camera_fpga/research/synthesis/dense-vs-keypoint-for-threshold-alarm.md`
- `../stereo_camera_fpga/design/CLAUDE.md` — Config 7 per-stage design + budget
- `thesis-proposal.md` (this folder)
- `../stereo_camera_fpga/council/council-transcript-2026-08-05_1752.md`,
  `../stereo_camera_fpga/council/council-transcript-2026-08-05_2141.md`,
  `../stereo_camera_fpga/council/council-transcript-2026-08-06_1043.md`
- `../stereo_camera_fpga/CLAUDE.md` and the `thesis/` top-level `CLAUDE.md`
