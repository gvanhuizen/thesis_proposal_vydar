# Advisor sync — proposal-defense debrief (2026-09)

**Status:** prep artifact for the post-defense debrief. Point-in-time, like the
`../stereo_camera_fpga/council/` snapshots — **not** a living authoritative doc.
Once items here are agreed, they fold back into `product-plan.md` /
`thesis-proposal.md` (pointers noted per item) and the corresponding §7 / §8
open-item checkboxes get ticked.

**Purpose:** the proposal-defense deck asks the committee three questions
(`../presentations/proposal-defense/deck.md`, "My questions" slide). Two more
sign-offs are needed before week 1 regardless of the defense outcome. And four
quantitative bars must be agreed before M3 — proposed starting values are in
§B so the debrief ratifies numbers rather than starting from a blank page.

---

## A. What changed since the last advisor contact

Context for the debrief — the planning docs were revised 2026-09-07 to match the
deck (`product-plan.md` §8, `thesis-proposal.md` §8 both log it):

- **Goal-first framing.** The organising goal is *real-time close-range
  detection within a configurable band*, on the fixed GateMate A1 + passive SWIR
  stereo hardware. Not a distance measurement.
- **Output granularity is now an open question, not a premise.** Whether the
  system emits a dense vdisparity map, a sparse/semi-dense output, or a bare
  boolean/zoned flag is an outcome of the approach search (scored in
  `product-plan.md` §5), not decided up front. The earlier "explicitly not a
  dense depth map" exclusion is dropped — a dense map is one admissible route.
- **Three peer routes, no ranking:** Route A dense disparity map, Route B
  bounded sparse key-point, Route C band-limited plane-sweep / disparity-threshold
  trigger (`product-plan.md` §4.1; RTL menu Configs 1–7 in
  `implementation-plan.md` §6).
- **Application constraints folded in:** moving platform (no static background
  model), closing speeds to ~600 m/s (kills long-window temporal filtering),
  anything-near-is-a-valid-trigger, purely passive (`product-plan.md` §3.1).
- **`thesis-proposal.md` §1 (research question) and the title are marked
  provisional** — to be finalised around whichever approach M3 selects. Frame
  this to the advisor as *sequencing*, not backsliding: the platform (open
  toolchain, DSP-less, ~160 KB BRAM, SWIR sensor) carries a defensible
  contribution for any of the three routes, so committing the exact question now
  would be premature.
- **Startup / commercial framing de-emphasised** in the docs — reads as a
  research project with a concrete application goal.

---

## B. Sign-offs and questions for the committee

### B1. Fallback-thesis sign-off — *needed before week 1*

**Ask:** explicit pre-approval that the thesis can stand on **M2 + M3 alone** —
a synthesis / simulation-only feasibility study plus systematic selection of the
best correspondence approach (across the three routes) for real-time close-range
detection on a DSP-less, BRAM-constrained, open-toolchain FPGA, benchmarked
against stored test vectors, **with no live-hardware integration**.

**Why it matters:** M1 (toolchain + first-ever board bring-up on a young
`nextpnr-himbaechel` flow) is the highest personal-risk item. If it slips past
its kill gates, M4 (live integration) is descoped to a stretch goal. The council
review flagged the missing-fallback gap independently across 3 of 5 advisors —
it only works as a hedge if pre-approved, not improvised at week 5.

**What the fallback still claims:** feasibility + evaluation on an
uncharacterised platform (the load-bearing contribution). **What it drops:**
end-to-end validation on live SWIR hardware (`thesis-proposal.md` §5, §1.1,
Appendix B).

*Docs:* `thesis-proposal.md` §5 (status: "not yet confirmed with advisor"),
`product-plan.md` §6 "Academic fallback". → tick `thesis-proposal.md` §7 item
"Advisor/committee sign-off on the fallback thesis".

### B2. Combined-scope sign-off — *needed before week 1*

**Ask:** sign-off that the thesis is **FPGA track primary, Pi camera rig
supporting** — one body of work on one 30-week clock.

**Open sub-question to settle in the same conversation:** is `../stereo_camera/`
(the working Pi rig + its baseline / depth-error literature review) cited as
**prior / parallel work for context only**, or **actively reused as source
material** — e.g. its baseline-vs-accuracy literature folded into the FPGA
thesis's own detection-quality-bar decision (B5 below)? Currently undecided
(`thesis-proposal.md` §0).

*Docs:* `thesis-proposal.md` §7 items "How the camera rig feeds the FPGA thesis"
+ "Advisor sign-off on the combined scope".

### B3. Is the contribution enough? *(deck question 1)*

**Ask:** confirm that "feasibility + approach selection on a novel platform, no
new matching algorithm" (broaden + deepen, per `thesis-proposal.md` §1.1) is
sufficient for a master's thesis — with M2 + M3 as the load-bearing spine and
M1 / M4 as bring-up and validation.

**Supporting framing if pushed:** no prior real-time FPGA stereo runs on an
open-toolchain, DSP-less, ~160 KB-BRAM part, and none uses a SWIR sensor; the
achievable feasibility / quality / throughput envelope is genuinely
uncharacterised. "First open-toolchain stereo bitstream" and "first real-time
SWIR FPGA stereo" are citable secondary outcomes regardless of the fps result.

### B4. University proposal format *(deck question 3)*

**Ask:** what is the required formal proposal template, the submission deadline,
and the expected length / structure? The planning docs stay in living-doc form
until this is known, then get reformatted for submission. Also: how is the
supporting camera-rig work credited in the formal document?

*Docs:* `thesis-proposal.md` §7 item "University's required formal
thesis-proposal format/template".

### B5. The detection-reliability bar *(deck question 2)*

**Ask:** ratify or push back on the proposed quantitative bars in §C. These must
be **written down and agreed before M3 benchmarking** — a defensible negative
result ("no candidate approach clears the bar on this part + this sensor") is
only honest if the bar predates the result.

**Note for the committee:** SWIR matching failure is spatially structured by
material (specular metal, glass, very low-albedo surfaces), so the metric
reports *where* detection is unreliable, not only an aggregate rate.

*Docs:* `product-plan.md` §5.2, §2.9; `thesis-proposal.md` §7 item
"Proximity-detection quality bar — resolve before M3".

### B6. Thesis disclosure vs. startup IP — *conversation with advisor and/or co-founder*

**Ask:** agree what is publishable. A public master's thesis (RTL, methodology,
results) sits next to a startup's core. The SWIR sensor decision sharpened the
product framing toward a specific, defensible use case — worth settling the
disclosure boundary *before* write-up, not after.

*Docs:* `thesis-proposal.md` §7 item "Thesis disclosure vs. startup IP".

---

## C. Proposed quantitative bars — `PROPOSED, pending advisor + founder sign-off`

These are **starting proposals with rationale**, not derived requirements. The
latency figure is fairly well constrained by the ~600 m/s geometry; the
detection-reliability figures are structural placeholders for the
advisor / stakeholder to set. Once agreed, fold into `product-plan.md` §2.4,
§2.5, §2.9, §5.2 and tick the §8 rolled-up open items.

### C1. End-to-end latency budget — replaces the §2.4 placeholder

**Proposed:** photon-to-alarm-edge **≤ 3 ms (≈ 2 frames at 600 fps)**, with the
alarm disparity plane `d_alarm` set **≥ 1 watch-band plane early** to absorb it.

**Rationale:** at ~600 m/s the object travels ~1.0 m/frame; 3 m → contact is
~5 ms — "the whole budget" (`../stereo_camera_fpga/design/CLAUDE.md` §4). Stage 2
needs a whole frame before it can emit a cluster, so the latency *floor* is
~1 frame ≈ 1.67 ms; 2 frames leaves headroom for the confirm stage. The
remaining ≥ 2 ms of the 3 m→contact window belongs to the **alarm consumer**,
which must itself act in single-digit ms.

**Hard dependency:** this only closes if the true required standoff is
~last-few-metres. A longer standoff needs a wider baseline / longer focal length /
higher windowed-ROI frame rate, and is the open **detection-range vs. geometry**
question (`product-plan.md` §2.9, §5.5; `thesis-proposal.md` §7). Resolve that
first — it can invalidate this budget and every route at once.

### C2. Minimum sustained frame rate — part of the §5.2 kill line

**Proposed:** sustained **≥ 600 fps at 640×512, full 14-bit**, through the
complete Stage 0–4 pipeline, with timing closed on `nextpnr-himbaechel`.
Documented fallback: **≥ 400 fps** is acceptable *only if* M3 shows detection
reliability (C3) is not degraded by the dropped frames.

**Rationale:** the streaming architecture exists to hold the sensor rate; the
warning window is only 2–4 frames, so dropping frames eats directly into the
safety-critical regime. A config that only closes timing at reduced resolution
or bit-depth that harms detection fails the line regardless of its fps number.

### C3. Detection-reliability floor + close-object false-negative target — the rest of the §5.2 kill line and §2.5

**Proposed (structural placeholders — advisor / stakeholder to set the actual
numbers):**

- **False negative on an object genuinely inside the band, at the band edge:**
  ≤ **1 % per frame**, AND — exploiting the 2–4-frame window — effectively
  **never miss a whole pass** (≤ 0.01 % that every frame of a valid approach is
  missed). Measured on known-distance SWIR targets across the money scenes (C4).
- **Correct in-band / out-of-band call at the band edge:** ≥ **99 %** on
  known-distance targets.
- **False positive (nuisance assertion):** ≤ **1 per minute** of operation as a
  starting figure — weighted far below FN in scoring, and needs the platform's
  actual tolerance.
- **Spatial reporting:** all three reported per material / scene region, not only
  as aggregates (B5 note).

**Rationale:** FN on a close object is the dominant failure (`product-plan.md`
§2.5); it is already the §5.1 criterion-2 hard gate. The per-frame vs.
per-pass split reflects that a real approaching object gives several frames to
catch it, while a single-frame FP does not survive the two-frame Δd confirm.

### C4. Money-scene weighting — resolves the §2.9 item, sets §5.1 criterion 7

**Proposed:** **fog / haze dominant.** Starting split for criterion 7
(SWIR-texture robustness): fog/haze ~**60 %**, low light ~**25 %**, glass
~**15 %**. Glass and low-light treated as *must-not-fail-catastrophically*
rather than co-equal optimisation targets.

**Rationale:** fog / haze penetration is the headline reason SWIR beats
visible-light stereo and the motivating scene for the product. Adjust the split
with the founder / stakeholder.

---

## D. After the debrief

- Record agreed numbers into `product-plan.md` §2.4 / §2.5 / §2.9 / §5.2; tick
  the §8 rolled-up open items and the `thesis-proposal.md` §7 items.
- Log the sign-offs (fallback, combined scope) in `thesis-proposal.md` §8 with
  the date.
- If the committee pushes the contribution framing (B3) or the route-selection
  method (B5), update `thesis-proposal.md` §1 / §1.1 and `product-plan.md` §4–§5
  accordingly.
- Whatever else the defense raises → capture here first, then route to the right
  doc.
