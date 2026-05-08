# Research Log — BridgeScan Thesis

## 2026-04-19

**Project initialized.**

- Set up Claude Project (BridgeScan) with full project instructions including thesis objective, schedule, scope constraints, and prior work context.
- Confirmed working dataset: DACL10k (decision not yet final).
- Confirmed baseline models from PoC: DeepLabV3+ (WCE) is the strongest baseline (mIoU 0.311, F1 0.447).
- Created thesis GitHub repo (separate from PoC repo).
- Uploaded PoC README and this log to Claude Project files.

**Known open questions carried forward from PoC:**
- Multi-label vs. single-label segmentation strategy
- Input resolution and resizing strategy
- Whether to reintroduce structural component classes
- Dataset: extend DACL10k, add manual labels, or replace

**Next step:** Task 1 — Problem scoping. Confirm target structure type (bridges), define damage typology to detect, and align with relevant inspection standards.

## 2026-04-29

**Architecture and pipeline design decisions — batch processing clarification.**

### Key decision: Post-hoc batch processing, not real-time
- Confirmed the system is an **offline web application**: user uploads drone video → server-side processing → navigable 3D model with damage overlays served to browser.
- **No real-time constraint.** The pipeline prioritizes accuracy over speed because the use case is inspection reporting and intervention planning (decision timescale: weeks), not in-flight decision support.
- **Consequence:** Heavy models (Mask2Former with ConvNeXt-L/EVA-02, fine-tuned SAM2, ensembles, test-time augmentation) are now the primary target, not aspirational. Lightweight models become baselines for comparison.
- Soft constraint: same-day turnaround on commodity hardware (practical without forcing architectural concessions).

### Semantic segmentation survey (v2) — updated for batch pipeline
- Revised survey v2 delivered (`concrete_damage_segmentation_survey_v2.md`).
- Takeaway 2 (architecture) revised: **Mask2Former + ConvNeXt-L/EVA-02-L with morphology-aware per-class losses is the primary model** (target 0.45–0.50+ mIoU on DACL10k, ensemble at 0.48–0.51 mIoU). HrSegNet, SegFormer, and hybrid CNN-transformers are now baselines/ablation references, not primary targets.
- Takeaway 3 (SAM2 + 3DGS) revised: **Fine-tuned SAM2 + 3DGS is the natural pattern for the thesis.** Duan et al. 2025 (JCSHM) and Wang/Spencer 2026 (arXiv:2602.16713) are direct blueprints.
- Accuracy-vs-throughput table reframed for offline use; diffusion methods now listed as usable accuracy-upper-bound references.

### Tentative pipeline architecture (end-to-end)

**1. Data acquisition and preprocessing:**
- Drone video upload to web app.
- Visual-overlap-based keyframe selection (target 60–70% overlap) with hard maximum interval (e.g., every 5 seconds flight time).
- Native-resolution tiled inference: 1024² or 1536² tiles, 25–50% overlap, NIST exact-tile or Flip-n-Slide 8× test-time augmentation.

**2. Multi-class segmentation on keyframes (primary model):**
- **Mask2Former + ConvNeXt-L (or EVA-02-L) backbone.**
- **Multi-label head:** sigmoid + BCE per class (replaces PoC's single-label priority collapse).
- **Morphology-aware per-class losses:**
  - Tubular/elongated (Crack, ACrack, ExposedRebars): Skeleton Recall Loss + Focal-Tversky.
  - Small compact (Cavity, Rockpocket): Focal-Tversky only.
  - Area/diffuse (Spalling, Efflorescence, Rust, etc.): BCE + Dice.
- **Key insight:** Skeleton Recall Loss (Kirchhoff ECCV 2024) is the first multi-class-capable topology-preserving loss — ~90% less overhead than clDice.
- **Dataset strategy:** Train on DACL10k + synth-dacl extensions; pretrain crack class on OmniCrack30k; fine-tune on S2DS/CODEBRIM for multi-class transfer; evaluate drone transfer on MBDD2025.

**3. Optional ensemble (if time permits):**
- Add CrackMamba or Hybrid-Segmentor as a second model for architectural diversity.
- Uniform averaging of predictions at inference (validated baseline; class-specific weighting only if uniform ensemble underperforms).
- **Risk management:** CrackMamba is optional value-add, not load-bearing. If training issues arise, ship with Mask2Former-only.

**4. Video propagation (temporal consistency):**
- **Fine-tune SAM2** with LoRA (CrackSAM-style) or LayerNorm-only (SAC-style) on crack dataset.
- Mask2Former outputs used as prompts to SAM2 on keyframes.
- SAM2 memory-based propagation tracks masks across intermediate frames (~44 FPS on A100).
- **Multi-class handling:** Per-class propagation (run SAM2 N times, once per class) or mask-level identity tracking (treat connected components as instances, propagate, re-attach class labels).
- **Robustness mechanisms:**
  - Confidence-based re-prompting: if SAM2 mask confidence drops, trigger unscheduled keyframe.
  - Periodic re-anchoring: force heavy-model keyframe every K frames to bound propagation drift.

**5. Instance ID assignment and tracking:**
- After Mask2Former outputs on keyframes and SAM2 propagates, assign persistent instance IDs to each connected component. Track these IDs across frames. Output format: per-frame instance masks with (instance_id, class_label, confidence) metadata.

**5. 3D reconstruction and mask projection:**
- **Dual reconstruction strategy:**
  - COLMAP/MVS for measurement-grade geometry (crack width quantification, ±1–4 mm with GCPs).
  - 3D Gaussian Splatting (3DGS) for navigable digital twin visualization.
- **Mask lifting:** Gaussian Grouping, LBG (Lifting By Gaussians), or AVGS to lift temporally consistent 2D masks into 3D semantic field.
- **Critical:** Run segmentation at native resolution; tile rather than downsample. Mask resolution directly affects 3D fidelity for thin cracks.

**6. Web app front-end:**
- Three.js / Gaussian-splatting viewer consumes 3DGS model.
- Damage overlays rendered as per-Gaussian class attributes.
- Queryable semantic digital twin: click on structure → view damage class, quantification, condition state.

### Resolved design questions from PoC
- **Multi-label:** Switch from single-label priority collapse to true multi-label with sigmoid + BCE per class. This alone should recover ~2–5 mIoU.
- **Input resolution:** Native resolution via tiling (1024² or 1536²), not fixed 512² resize. Flip-n-Slide 8× TTA affordable in batch pipeline.
- **Structural components:** Defer for now; focus on 13 damage classes. Components can be auxiliary output if time permits.

### Architectural concepts clarified
- **Hybrid CNN-Transformer:** Three patterns exist (parallel dual-branch, sequential/interleaved, CNN encoder + transformer decoder). Mask2Former with ConvNeXt-L is the third pattern — CNN does dense feature extraction, transformer decoder does set-prediction via learned queries.
- **Mask2Former queries:** Learned embeddings that act as "object detectors." Each query attends to encoder features via cross-attention and outputs (class, mask, confidence). Output is a set of N masks, not dense H×W×C pixel-wise labeling. Natural fit for multi-label (overlapping masks supported natively).
- **SAM2 memory propagation:** Memory encoder compresses (frame + mask) → memory bank stores ~7 recent frames + keyframe → memory attention lets current frame attend to bank via cross-attention. Result: temporal smoothness, occlusion handling, drift bounded by keyframe anchor.
- **Ensembles:** Multiple independently-trained models; predictions averaged at inference (typically uniform averaging across all classes). Architectural diversity (e.g., Mask2Former + CrackMamba) drives ensemble gains.

### Open decisions
- Exact keyframe overlap threshold (60–70% target, needs empirical validation).
- Single-model vs. ensemble: train Mask2Former first, add CrackMamba only if time permits.
- Multi-class propagation strategy: per-class (N× SAM2 calls) vs. mask-level identity tracking (needs implementation testing).
- **Primary architecture choice: Mask2Former vs. HrSegNet vs. CrackMamba (critical decision pending).**

### Architecture selection — open decision requiring empirical validation

**Three candidate architectures under consideration:**

1. **Mask2Former + ConvNeXt-L (or EVA-02-L):**
   - **Advantage:** DACL10k SOTA (0.51 mIoU, dacl-challenge winner). Validated multi-class performance. Well-documented (Detectron2/MMSegmentation). Query-based set-prediction naturally handles multi-label overlaps.
   - **Concern:** Encoder bottleneck at 1/4 resolution. Challenge winner had weak Crack/Cavity classes, though actual per-class metrics are not available. Assumes skeleton losses compensate for resolution loss.
   - **Risk level:** Low (proven recipe, known performance ceiling).

2. **HrSegNet:**
   - **Advantage:** Maintains 1/2 resolution main path (vs. 1/4 for Mask2Former). Explicitly designed for thin cracks. Validated on RCD crack benchmark (82% mIoU, beats HRNet/SegFormer/DeepLabV3+). Lightweight (5-30M params, faster training iteration).
   - **Concern:** No DACL10k multi-class validation. Designed for single-class cracks, needs multi-label adaptation (sigmoid+BCE). Unknown if 1/2-resolution advantage overcomes Mask2Former's query-based decoder on multi-class. RCD dataset is for Road Cracks, might be different from bridge cracks.
   - **Risk level:** Medium (first to validate on DACL10k multi-class, but open code and mature 2023 paper).

3. **CrackMamba:**
   - **Advantage:** SOTA on CrackSeg9k and SewerCrack crack benchmarks. Serpentine scan bakes topology preservation into architecture (no separate skeleton loss needed). VMambaV2 encoder optimized for tubular structures.
   - **Concern:** No DACL10k multi-class validation. Crack-specialist — would require second model for non-crack classes (two-model merging complexity). Newest architecture (2024), less ecosystem maturity than CNNs/Transformers.
   - **Risk level:** Medium-high (newest, Mamba less adopted, no multi-class validation).

**Core tradeoff:**
- **Mask2Former:** Validation breadth (multi-class SOTA) vs. task-specific design (not optimized for cracks).
- **HrSegNet:** Task-specific design (crack-optimized, 1/2 resolution) vs. validation breadth (no multi-class baseline).
- **CrackMamba:** Strongest crack-specific evidence vs. highest architectural complexity (two-model split required).

**Key unknown:** Per-class breakdown of dacl-challenge winner's 0.51 mIoU not published. We know Crack/Cavity were weakest classes across all submissions, but don't know if winner's crack performance was strong or if they won on area classes despite weak cracks.

**Proposed decision strategy (risk-managed):**
1. **Weeks 1-2 of training phase:** Train HrSegNet on DACL10k multi-class with sigmoid+BCE multi-label head + morphology-aware per-class losses (Skeleton Recall + Focal-Tversky). Evaluate on DACL10k val/test, report per-class IoU (especially Crack/Cavity/ExposedRebars).
2. **Decision point:**
   - If HrSegNet mIoU ≥ 0.40 AND Crack IoU ≥ 0.30: Commit to HrSegNet as primary model. Thesis contribution: "Crack-specific architecture matches general-purpose SOTA on multi-class when combined with morphology-aware losses."
   - If HrSegNet mIoU < 0.40 OR multi-class performance weak: Switch to Mask2Former + ConvNeXt-L. Thesis contribution: "Validated that general-purpose SOTA outperforms task-specific architecture on multi-class, confirming challenge winner's choice."
3. **Either outcome is defensible** — the empirical comparison itself is a contribution (first HrSegNet validation on DACL10k multi-class).
4. **CrackMamba reserved as optional ensemble member** if time permits after primary model training.

**Rationale for HrSegNet-first approach:**
- Faster training (5-30M params vs. 100-220M) → find out early if it works, pivot if needed.
- Novel empirical result (HrSegNet on DACL10k multi-class not published).
- De-risks timeline (doesn't lock into underperforming architecture).
- Both outcomes (HrSegNet works / doesn't work) are thesis-defensible research findings.

**Alternative framing if advisor prioritizes DACL10k baseline:** Start with Mask2Former (safest path to 0.45-0.50 mIoU), add HrSegNet as lightweight comparison/ablation if time permits.

**Dependency on thesis success criterion:**
- "Reach competitive DACL10k mIoU + demonstrate end-to-end pipeline" → Mask2Former (safest).
- "Maximize crack-class IoU for structural criticality" → HrSegNet or CrackMamba.
- "Validate crack-specific architectures on multi-class tasks" → HrSegNet-first with Mask2Former fallback (research question).

### Next steps
1. **Complete state-of-the-art survey (Task 2):** Finish international bridge inspection standards comparison document (AASHTO MBEI 2019 + SNBI 2022, German DIN 1076, Italian LG2020, Japan MLIT, Ontario OSIM). Partial search results already gathered; synthesis document pending.
2. **Dataset definition (Task 3):** Finalize DACL10k + auxiliary datasets strategy; acquire synth-dacl, OmniCrack30k, S2DS, CODEBRIM, MBDD2025.
3. **Baseline reproduction:** Reproduce PoC DeepLabV3+ baseline on DACL10k subset with multi-label head to establish improvement delta from PoC's 0.31 mIoU.
4. **Mask2Former training (Task 4):** Implement morphology-aware per-class losses; train on DACL10k; target single-model 0.42–0.46 mIoU.