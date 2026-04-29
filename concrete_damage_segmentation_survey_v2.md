# Semantic segmentation for concrete bridge inspection: a 2022–2026 survey

*Revision v2 — adjusted for a post-hoc batch-processing pipeline (drone video uploaded to a web app for offline processing and 3D visualization). Changes are localized to §3 (accuracy-vs-throughput framing), §11 (takeaways 2 and 3). All benchmark numbers, papers, datasets, and methodological findings are unchanged.*

**Bottom line up front.** Concrete damage segmentation has become a genuinely competitive subfield between four architectural families — modernized CNN baselines (HRNet/DeepLabV3+/HrSegNet), plain and hybrid transformers (SegFormer, Mask2Former, ConvNeXt-UPerNet), promptable foundation models (fine-tuned SAM/SAM2), and very recent Mamba/state-space crack models. On DACL10k, the primary benchmark for multi-label bridge damage, the original baseline is **0.42 mIoU** and the best dacl-challenge WACV 2024 entry reached **0.51 mIoU**; everything above roughly 0.45 mIoU should be considered "strong" given the hardness of rare thin classes (Crack, Cavity). The user's current DeepLabV3+/weighted CE baseline at **~0.31 mIoU** is below the baseline but is explainable primarily by (a) the multi-label→single-label collapse, (b) weighted CE being a poor choice for severely imbalanced thin classes, and (c) the decoder's effective 1/4 resolution discarding hairline features. The most consequential trends in 2024–2026 are the shift to **skeleton/topology-aware losses** (clDice, Skeleton Recall Loss), **PEFT fine-tuning of SAM/SAM2** for thin cracks, and the rapid emergence of **SAM2 + 3D Gaussian Splatting** as a practical drone-video-to-3D-damage-map pipeline — the closest published analogue to the thesis pipeline. **With the thesis confirmed as an offline batch pipeline (video uploaded to a web app, processed server-side, results served as a navigable 3D model), compute-per-frame is no longer a binding constraint and the heavy-accuracy branch of the landscape becomes the primary target rather than an aspirational upper bound.** The rest of this survey unpacks those claims.

---

## 1. Recent papers (2022–2026): main priority

### 1.1 Multi-class concrete bridge damage segmentation on DACL10k

**Flotzinger, Rösch, Braml (2024), "dacl10k: Benchmark for Semantic Bridge Damage Segmentation", WACV 2024** (arXiv:2309.00460). Introduces the 9,920-image multi-label bridge dataset with 13 damages + 6 components (19 classes). Baselines (DeepLabV3+, SegFormer, HMA) peak at **0.42 mIoU** on the test split. Coarse crack polygons limit attainable IoU on thin classes. *Code/toolkit: github.com/phiyodr/dacl10k-toolkit.*

**Flotzinger et al. (2024), "dacl-challenge: Semantic Segmentation during Visual Bridge Inspections", WACVW 2024.** 23 teams registered, 8 beat the baseline; best submission reached **0.51 mIoU**. Winning entries used **heavy modern backbones (ConvNeXt-Large, EVA-02-Large) and Mask2Former heads with ensembling**. Crack and Cavity remained the consistently weakest classes, confirming imbalance as the core bottleneck.

**Flotzinger, Deuser, Jaziri, Neumann, Oswald, Ramesh, Braml (2025), "synth-dacl: Does Synthetic Defect Data Enhance Segmentation Accuracy and Robustness for Real-World Bridge Inspections?", GCPR 2025** (arXiv:2506.14255). Three synthetic extensions (15k images total) using procedural fractal generators for cracks and multi-octave noise for cavities. **Directly addresses the DACL10k class imbalance**: best finecrack IoU gains ~1.3 points and Cavity IoU 24.63 % / F1 39.53 % when augmenting training. Directly relevant as thesis pretraining augmentation.

**Flotzinger et al. (2025), "metal-dacl"**, EUROSTRUCT 2025 — steel-bridge sibling dataset; noted for completeness as the same group's ecosystem.

**Hybrid YOLO + SAM for UAV multi-damage (PMC12609025, 2025).** Explicitly trains YOLOv11x-seg on DACL10k and evaluates on a 161-image high-resolution UAV validation set: **mIoU collapses to 0.014** in pure cross-domain transfer, rising to 0.118 after 645 UAV images of fine-tuning. Strong evidence that **DACL10k models do not transfer zero-shot to drone imagery** because of resolution/viewpoint shift — critical finding for the thesis.

**Benz & Rodehorst (2022), S2DS**, GCPR. 743 bridge images, 7 single-target classes (crack, spalling, corrosion, efflorescence, vegetation, control points, background); Hierarchical Multi-Scale Attention reaches ~92 % mIoU. S2DS is the correct auxiliary multi-class benchmark when targeting DACL10k because its taxonomy overlaps tightly. *Code/data: github.com/ben-z-original/s2ds.*

**TSCB-Net — Liu et al. (2024)**, *Structure and Infrastructure Engineering*. PVTv2-B0 encoder + Context Information Refinement; 5.84 M params; **75.84 % mIoU / 85.54 % mF1** on a custom bridge damage set with classes water-erosion, spalling, rebar-corrosion. Beats SegFormer, Segmenter, Mask2Former at equal params.

**Improved DeepLabV3+ lightweight — Nature Sci. Rep. (2025).** MobileNetV3 + CSF-ASPP; 6.97 M params, 52.64 FPS, **75.24 % mIoU** on bridge surface diseases; beats vanilla U-Net, HRNet, PSPNet, SETR, SegFormer, Mask2Former, PIDNet. Key evidence that **tuned CNN baselines still match transformers at a fraction of the cost** for drone-relevant deployment.

**Enhanced Mask2Former for Bridge Cracks (IEEE Access 2025).** Adds RepBlock multi-scale feature fusion on top of Mask2Former; improves continuity on fine cracks that vanilla Mask2Former misses.

**Enstrect — Benz & Rodehorst (2024)**, arXiv:2401.03298. Multi-view 2.5D structural damage on bridge facades using HRNet-OCR (DetectionHMA) vs nnU-Net as CNN baselines — shows **DetectionHMA is the more robust baseline for cracks** while nnU-Net wins for spalling/corrosion. Useful architecture-choice evidence.

**Zhang/Karim/Qin (2023), Multi-task HRNet for bridge element + defect parsing**, TRR — sets the standard multi-task HRNet baseline for bridge component + damage joint parsing. Code: github.com/jingxiaoliu/bridge-damage-segmentation.

### 1.2 Hybrid CNN–Transformer multi-class and crack segmentation

**Hybrid-Segmentor — Goo et al. (2025)**, *Automation in Construction* 170:105960 (arXiv:2409.02866). Dual encoder (ResNet-50 + SegFormer) fused at 5 scales. On a unified 13-dataset crack benchmark: Acc 0.971, F1 0.770, IoU 0.630 — SOTA against U-Net, DeepLabV3+, SwinUNet, SegFormer. *Code + checkpoints: github.com/junegoo94/Hybrid-Segmentor.*

**PSC (Parallel Swin-CNN) — Han et al. (2024)**, *Structural Health Monitoring*. Parallel Swin + CNN encoders with multi-scale pyramid decoder; **+36.57 % F1, +62.38 % IoU** over U-Net/DeepLabV3/PSPNet/DeepCrack on two public concrete crack sets, with 3–41 % fewer parameters.

**AG-TransUNet (2025), MDPI Buildings 15(4), 541.** Adaptive multi-head self-attention and GRU-T gated decoder on TransUNet; +5.63 % F1, +9.07 % IoU on concrete crack dataset vs vanilla TransUNet; includes orthogonal skeleton-based width quantification.

**Ding et al. (2023)**, "Crack detection and quantification for concrete structures using UAV and transformer," *Automation in Construction* 152:104929. Explicitly UAV-captured; hybrid CNN-transformer for pixel segmentation + width quantification.

**DSCformer (arXiv:2411.09371, 2024).** Dynamic Snake Convolution + Transformer specifically for tubular crack geometry; beats crackmer and DTrC-Net.

**CA-SegFormer (2025)**, *Case Studies in Construction Materials*. Coordinate attention in SegFormer decoder; **+2 % IoU over vanilla SegFormer, +4 % over DeepLabV3+** on Concrete3k with half the parameters — a minimal intervention that materially improves thin-class performance.

### 1.3 Crack-specific architectures (Mamba era)

**CrackFormer-II — Liu et al. (IEEE T-ITS 2023).** SegNet-like encoder-decoder with transformer blocks, relative positional embedding, scaling-attention fusion. Still the default transformer crack baseline. *Code: github.com/LouisNUST/CrackFormer-II.*

**CrackMamba — Zuo, Sheng, Shen, Shan (2024)**, *Automation in Construction* 168:105845 (arXiv:2410.19894). VMambaV2 encoder + Snake Scan module (serpentine scan) + Snake Conv VSS block. SOTA on CrackSeg9k and SewerCrack; competitive even on CHASE-DB1 vessels — direct evidence that **vessel-segmentation methodology transfers cleanly to cracks**. *Code: github.com/shengyu27/CrackMamba.*

**SCSegamba — Liu, Jia, Shi, Cheng, Chen (CVPR 2025).** Structure-Aware Visual State Space with gated bottleneck convolution. Lightweight SOTA for thin cracks. *Code: github.com/Karl1109/SCSegamba.*

**HrSegNet — Li, Ma, Liu, Cheng (2023)**, *Automation in Construction* 156:105112 (arXiv:2307.00270). **The single most important paper for the user's 1/4-resolution concern.** Keeps the high-res path at **1/2 original resolution** throughout (vs HRNet's and SegFormer's 1/4) with an auxiliary semantic-guidance path. On RCD: U-Net 76.71 % < DeepLabV3+ 78.29 % < HRNet-W18-OCR 80.90 % < HrSegNet ~82 %. Real-time and explicitly designed for crack-thin structures. *Code: github.com/CHDyshli/HrSegNet4CrackSegmentation.*

**HRCSF — Chu et al. (2024)**, *Computer-Aided Civil and Infrastructure Engineering*. Multi-scale cascaded network with strip pooling, specifically designed for **>4K crack images on RTX 3060**: IoU 90.89 %, Dice 96.28 %, 3.84 FPS at 4K. The only paper explicitly reporting realistic 4K-resolution accuracy/throughput tradeoffs — a direct reference for the inference-throughput section of the thesis.

**HACNet V2 (2025)**, *Expert Systems with Applications*. Full-resolution (never downsamples) dual-branch architecture, HybridASPP. Confirms that **full-resolution networks give the best detail/cost balance** for cracks.

**Segformer++ — Hanisch et al. (2024)** (arXiv:2405.14467). Token-merging strategies that let SegFormer ingest high-resolution input; directly relevant if the user wants to keep a SegFormer backbone.

**EfficientCrackNet (arXiv:2409.18099).** MobileViT + Edge Extraction Module + ULSAM; mIoU 87.10 %, F1 85.88 % on DeepCrack — strong edge-deployment candidate (less relevant for a server-side batch pipeline).

**ConvNeXt-UPerNet / DronePavSeg (2025).** UAV pavement crack dataset; mIoU 79.73 %, Crack-IoU 61.71 %. Outperforms HRNet-FCN by ~0.4–0.7 %.

### 1.4 SAM / SAM2 adaptations (the most active 2023–2026 subfield)

**Ahmadi et al. (2023)**, "Application of Segment Anything Model for Civil Infrastructure Defect Assessment" (arXiv:2304.12600). First SAM-on-concrete paper. Establishes that **zero-shot SAM produces visually plausible but numerically inadequate masks on hairline cracks** because SAM lacks 1-pixel precision and has no crack prior.

**CrackSAM — Ge & Wang (2024)**, *Construction and Building Materials*. PEFT with Adapter + LoRA on ViT encoder (~1 % trainable params); 11k training images + two new UAV+smartphone labelled datasets (Road420, Facade390). Outperforms 12 SOTA segmenters on artificial-noise and out-of-distribution tests. Code/data released.

**Segment Any Crack (SAC) — Hosseini et al. (2025)** (arXiv:2504.14138). Fine-tunes **only SAM's LayerNorm layers** — minimal parameter surgery yet beats CrackSAM on Road420, Facade390, Concrete3k and transfers across concrete/asphalt/ceramic/masonry/steel. Currently the best parameter-efficiency/accuracy point for crack segmentation.

**Guo et al. (2025)**, *Structural Health Monitoring*. SAM + LoRA + dedicated crack head; broad gains across 8 crack datasets.

**SAM-Adapter noisy cracks — Yao et al. (2024)** (arXiv:2410.09409). Mixture-of-Gaussians noise modeling on top of SAM-Adapter. New SOTA cross-domain on Crack500/CFD — **directly handles DACL10k-style coarse/noisy annotation**. *Code: github.com/sky-visionX/CrackSegmentation.*

**Crack-SAM (Rakshitha et al. 2024)**, *J. Infrastructure Preservation and Resilience*. Detectron2 (faster_rcnn_R_101) provides boxes → SAM produces masks with DiceFocalLoss. IoU 0.69 on CFD, 0.59 on Crack500.

**Muturi & Adu-Gyamfi (2025)**, *Transportation Research Record*. Multimodal box + point prompts with cheap ground truths; practical annotation-efficient recipe.

**Crack-EdgeSAM / CrackESS (arXiv:2412.07205, 2024/25).** YOLOv8n prompt generator + ConvLoRA-fine-tuned **EdgeSAM** with DiceFocalLoss, tested on a wall-climbing robot inspecting concrete bridge piers. **>4× faster than CrackSAM** on embedded hardware with comparable accuracy — most relevant for edge deployments; for the thesis's server-side batch pipeline, full-size CrackSAM/SAC is preferable.

**SECrackSeg (MDPI Sensors 2025, 25(9):2642).** Frozen SAM2 backbone + S-Adapter, MSDC multi-scale dilated convolutions, MI-Upsampling, Edge-Aware Attention. Explicitly targets low-data crack regimes.

**Duan et al. (2025), "Defect segmentation and 3D reconstruction in concrete structures using SAM 2 and 3D Gaussian Splatting", *Journal of Civil Structural Health Monitoring* 15:3345–3360.** The closest academic analogue to the thesis pipeline: monocular video → SAM2 mask propagation → COLMAP/3DGS reconstruction → 3D defect visualization with spatial coordinates. **Flag as primary baseline.**

**Integrated SAM + OCR (MDPI Infrastructures 10:348, 2025).** YOLOv8 prompts SAM; local-refinement module for thin cracks; Tesseract OCR for GPS from drone EXIF. IoU 76.69 % on Crack500. End-to-end pavement-inspection pipeline with explicit drone relevance.

### 1.5 Diffusion, Mamba, and foundation-model alternatives

**CrackDiff (MDPI Remote Sensing 16:986, 2024).** First diffusion model for pavement/concrete cracks; multi-task U-Net predicts mask and noise per denoising step; improves curved-crack continuity.

**CrackSegDiff (arXiv:2410.08100, 2024).** Multi-modal (grayscale + depth) diffusion with VMamba U-Net backbone; improves over SegDiff/MedSegDiff.

**CrossDiff (arXiv:2501.12860, 2025).** Cross-encoder/decoder diffusion targeting slender crack morphology; tested on CFD, CrackTree200, DeepCrack, GAPs384, Rissbilder.

Diffusion methods consistently improve topological continuity. **For a batch/offline pipeline the iterative-denoising inference cost is no longer disqualifying** — diffusion methods are usable as an accuracy-upper-bound reference, although none has yet been validated on multi-class DACL10k-style benchmarks and the engineering complexity relative to gain should be weighed carefully given thesis time constraints.

**Zhu et al. (2026), "Autonomous Detection of Concrete Cracks Using Self-Supervised DINOv2"**, *Machine Intelligence Research* 23(1):168–184. Frozen DINOv2 ViT-S/14 features + linear head; 5 epochs of training. Beats ResNet50/101, VGG16, MobileNetV2, DenseNet121, and MoCo v2 on CCiC, Xu, HBC2019, SDNET2018 (F1 0.9346 on HBC2019). Currently the flagship evidence that **DINOv2 features transfer very well to concrete damage**. Classification-level, so a pixel-level DINOv2 crack segmenter is still an open niche.

**Docherty et al. (2024)** (arXiv:2410.19836). Upsampling DINOv2 features for materials segmentation including hairline cracks; shift-average upsampling recovers pixel masks from patch tokens.

**CLIP/open-vocabulary segmentation for concrete damage: essentially unexplored** as of April 2026 — a clear thesis gap to flag.

---

## 2. Foundational pre-2022 work (concise)

**Crack segmentation seminal networks.** CrackNet and CrackNet-II (Zhang 2017/2018) introduced fully-learnable CNNs for 3D asphalt surfaces. **DeepCrack (Liu et al., *Neurocomputing* 2019)** remains the default crack-segmentation comparison baseline with its hierarchical multi-scale deep supervision; its 537-image dataset is a standard benchmark. **FPHBN (Yang et al., T-ITS 2019)** introduced feature-pyramid + hierarchical boosting; still cited for establishing multi-scale aggregation as the core design pattern for thin cracks and releasing Crack500.

**Foundational architectures still used as concrete baselines.** U-Net (Ronneberger 2015) remains the single most-used encoder-decoder on concrete and is the CNN half of most 2024–26 hybrids. DeepLabV3+ (Chen 2018) dominates lightweight CNN baselines for multi-class damage when combined with MobileNet/Xception backbones and tuned ASPP. HRNet (Sun/Wang 2019) is the preferred backbone when thin-feature preservation matters, outperforming U-Net and SegFormer on crack benchmarks in head-to-head studies (HrSegNet, 2023). PSPNet (Zhao 2017) and FCN (Long 2015) round out the ancestor set.

**Multi-class benchmark.** **CODEBRIM (Mundt et al., CVPR 2019)** was the first multi-target multi-class concrete-bridge defect benchmark (cracks, spallation, efflorescence, exposed bars, corrosion stains) with realistic class overlap; 1,590 high-res images with bbox/class-vector annotations, partly UAV-captured. It remains the dominant benchmark in 2024–2026 multi-label bridge-defect work. Zenodo: 10.5281/zenodo.2620293.

**Thin-structure foundations.** HED (Xie & Tu 2015) is the ancestor of side-supervised deep edge detection and the architectural root of DeepCrack/FPHBN. **clDice (Shit et al., CVPR 2021, arXiv:2003.07311)** introduced differentiable soft-skeletonization loss guaranteeing topology preservation for tubular structures — the single most influential loss for cracks. Methodological transfer from retinal-vessel segmentation (U-Net on DRIVE; Frangi vesselness 1998) is a recurring, productive theme: both cracks and vessels are thin, elongated, branching, sparse, and extremely imbalanced.

---

## 3. Architecture families: what the 2024–2026 evidence actually says

**Pure transformers (SegFormer, Mask2Former, SETR, Swin-UNet) are NOT a clear winner over tuned CNNs on concrete damage.** SETR consistently loses; SegFormer and Mask2Former are competitive but almost always require multi-scale fusion or edge modules to match specialized CNNs on thin classes. In the DACL10k 2025 lightweight study, an improved DeepLabV3+ (MobileNetV3 + CSF-ASPP) at 6.97 M params and 52.64 FPS beats vanilla SegFormer and Mask2Former on bridge surface diseases.

**Hybrid CNN-Transformer is the dominant publication trend for crack-focused bridge damage in 2024–2025** (Hybrid-Segmentor, PSC, AG-TransUNet, DSCformer, CA-SegFormer). It outperforms pure CNNs on generalization and pure transformers on thin-crack continuity. This is the architecture family most aligned with a strong thesis baseline.

**SAM/SAM2 with PEFT is the high-leverage 2024–2026 frontier.** Zero-shot SAM is not competitive on thin cracks (Ahmadi 2023; Ge 2024; SAC 2025). However, LoRA/Adapter (~1 % trainable params) or LayerNorm-only tuning (SAC) brings SAM to SOTA on out-of-distribution test sets, often surpassing fully supervised CNNs. **For the thesis's batch-processing pipeline, SAM2's memory-based propagation is a natural fit**: prompt a keyframe, let the memory transformer propagate masklets across the rest of the video. SECrackSeg and Duan et al. 2025 are the validated concrete-specific instances.

**Mamba/state-space crack models (CrackMamba, SCSegamba) are the most recent entrants** (late 2024–2025). They explicitly exploit tubular topology via serpentine/structure-aware scans and achieve competitive accuracy with much lower compute than transformers.

**Diffusion-based segmentation (CrackDiff, CrossDiff, CrackSegDiff)** helps thin-structure continuity. No multi-class DACL10k-style validation exists yet. Batch-mode inference is feasible but engineering complexity is high relative to expected gain.

**DINOv2 as frozen backbone** is the most promising unexplored direction: Zhu et al. 2026 showed perfect crack classification with 5 training epochs; pixel-level concrete segmentation with DINOv2 features is still an open niche worth pursuing.

### Accuracy-vs-compute table (server-side batch inference)

Because the thesis pipeline is offline — video uploaded to a web app, processed server-side, results served as a navigable 3D model — inference throughput is not a hard constraint. The table below reports typical throughput for reference only (e.g., estimating total processing time for a 10-minute 4K drone video). The "fit" column is rewritten for batch/offline use: ★★★ = preferred for the thesis primary model, ★★ = reasonable baseline or lightweight comparison, ★ = reference upper-bound or specialized niche, ✗ = ruled out for other reasons.

| Family | Thin-crack accuracy | Params | Typical FPS | Batch-pipeline fit |
|---|---|---|---|---|
| ConvNeXt-L / EVA-02-L + Mask2Former (+ensemble) | Highest multi-class (dacl-challenge SOTA, ~0.51 mIoU) | 100–220 M × N | 5–15 | ★★★ Primary candidate |
| Fine-tuned SAM/SAM2 (CrackSAM, SAC) | SOTA cross-domain on cracks | ~640 M (frozen) | 1–5 | ★★★ Primary candidate for crack class |
| SAM2-tiny + key-frame propagation | High and temporally consistent | 38.9 M | ~47 FPS (A100) | ★★★ Natural fit for video pipeline |
| Hybrid-Segmentor / PSC / CA-SegFormer | SOTA for cracks | ~60 M | 15–25 | ★★★ Strong primary or ensemble member |
| HrSegNet / HRNet-W48 | High (mIoU 0.73–0.83 on RCD) | 5–30 M | 50–150 | ★★ Lightweight baseline / ablation reference |
| SegFormer-B0/B1 | High once decoder patched (Segformer++, CA-SegFormer) | 4–14 M | 40–100 | ★★ Baseline |
| Improved DeepLabV3+ lightweight | Strong on multi-class (0.75 mIoU bridge surfaces) | 7 M | 50+ | ★★ Baseline, good edge-deployment discussion |
| Diffusion (CrackDiff / CrossDiff) | Top thin-crack continuity | 50–150 M | <1 | ★ Accuracy-upper-bound reference only |
| EdgeSAM / Crack-EdgeSAM | Near-SAM quality | ~40 M | 30–60 | ★ Edge-inference niche, not thesis-relevant |
| DINOv2 ViT-S/14 (frozen + head) | Excellent features | 22 M | 60+ | ★★ Feature extractor — open research niche |

---

## 4. Datasets beyond DACL10k

### 4.1 Multi-class / multi-label concrete damage (complementary to DACL10k)

**CODEBRIM** (Mundt 2019). 1,590 imgs / 8,323 bbox annotations; 6 non-exclusive classes; multi-label. Partly UAV. Multi-target bbox/classification, not pixel-level.

**S2DS** (Benz & Rodehorst 2022). 743 × 1024² bridge patches, 7 single-target classes (background, crack, spalling, corrosion, efflorescence, vegetation, control point). HMA baseline ~92 % mIoU. Best auxiliary multi-class pixel-seg benchmark aligned with DACL10k taxonomy.

**synth-dacl** (Flotzinger 2025). 15,000 synthetic images in three flavors (balanced, crack-focused, cavity-focused), inheriting DACL10k taxonomy. Directly augments DACL10k's weak classes.

**DACL1k** (Flotzinger 2024, *Eng. Applications of AI*). 1,474 imgs; independent cross-dataset evaluation set for DACL10k-trained models.

**MCDS** (Hüthwohl, Lu, Brilakis 2019). 3,617 imgs; image-level multi-label classification (crack, efflorescence, spalling, exposed rebars, scaling, general defect).

**HRCDS** (Mendeley 2024). High-res multi-class concrete damage segmentation (cracks, exposed rebar, corrosion strain, surface spalling).

**Tunnel Defect ViT dataset** (Qin et al. 2024, *CACIE*). 11,781 subway tunnel images; crack, leakage, spalling with compound superpositions — rare **multi-label tunnel** example.

**Dam spillway multi-defect** (Zhou 2023, MDPI Buildings 13:285). 1,711 + 2,218 CSSC augmentation; 5 classes; wall-climbing-robot acquisition.

### 4.2 Bridge-specific detection and segmentation datasets

**CSSC** (Yang 2017). UAV+web, ~1,200 imgs, crack + spalling pixel masks. Seminal UAV bridge dataset.

**ConRebSeg** (Schmidt & Nalpantidis 2025, *Automation in Construction* 171:105990; arXiv:2407.09372). 14,805 imgs, 54,115 instances, exposed rebar focus on shotcrete construction sites. YOLOv8L-seg val mIoU 0.59. *Code/data: github.com/DTU-PAS/ConRebSeg.*

**GYU-DET** (Sci. Data 2025, 12:1101). 11,123 high-res beam-bridge images; 6 detection classes (cracks, spalling, seepage, honeycomb, exposed rebar, holes).

**MBDD2025** (Sci. Data 2025). 14,471 high-res UAV images, 5 defect classes × 6 structure types. **Fully UAV**. Strong cross-domain drone validation dataset.

**Pixel-level UAV bridge (Song 2024)**, *Structural Control and Health Monitoring* 2024:1299095. ~2,500 DJI Phantom 4 RTK images, binary crack, explicit UAV bridge capture.

### 4.3 Crack-only datasets (useful for pretraining)

Crack500 (Yang 2020); CrackTree200 (Zou 2012); CFD (Shi 2016); DeepCrack537 (Liu 2019); SDNET2018 (Dorafshan 2018, classification); GAPs/GAPs384 (Eisenbach 2017); Rissbilder/Eugen-Müller/Volker collections; Masonry Crack (Dais 2021); Concrete Crack Conglomerate (VT Figshare, 10,995 imgs); METU (Özgenel 2019, 40,000 classification patches); Aigle-RN, ESAR, CrackLS315, Stone331, CRKWH100 (CrackFormer benchmarks).

**CrackSeg9k** (Kulkarni et al., ECCV-W 2022). 9,255 × 400² imgs aggregating 10 sub-datasets; binary pixel seg. DeepLabV3 ~77 % mIoU; SAC higher.

**OmniCrack30k** (Benz & Rodehorst, CVPRW 2024). 30,017 images across 20 subsets, cross-material (asphalt/ceramic/concrete/masonry/steel); 22,158/3,277/4,582 splits. nnU-Net 64 % clIoU4px. Contains **UAV75** explicit UAV subset. Currently the best large-scale crack pretraining resource for a DACL10k-bound thesis. *Code: github.com/ben-z-original/omnicrack30k.*

### 4.4 UAV/drone-captured datasets (highest relevance for thesis)

Ranked by explicit drone capture: **MBDD2025** (14,471 imgs, fully UAV) > **HighRPD** (11,696 high-altitude UAV pavement imgs) > **Crack-DA UAV / BCL — Liu 2022** (11,298 imgs, UAV-oriented) > **RDD2022 China_Drone subset** (2,401 × 512² DJI M600 Pro imgs) > **UAV-PDD2023** (2,440 imgs at 30 m altitude) > **Pixel-level UAV bridge (Song 2024)** > **CSSC** > **CODEBRIM** (partial UAV) > **UAV75** > **DACL10k** (partial UAV).

### 4.5 Label-consistency caveats

Label taxonomies differ significantly across datasets. DACL10k distinguishes Crack vs ACrack (alligator), while S2DS and CODEBRIM merge these; OmniCrack30k is binary only. Spalling ↔ Rockpocket confusion is acknowledged by DACL10k authors (v2→v3 split). Corrosion vs Rust vs Efflorescence overlap differently across CODEBRIM, S2DS, and DACL10k. DACL10k crack polygons are **coarse**, OmniCrack30k masks are fine-pixel — naive IoU comparisons mislead. Multi-label (DACL10k, CODEBRIM) vs single-target-per-pixel (S2DS, tunnel datasets) requires different loss functions (sigmoid+BCE vs softmax+CE). UAV-PDD2023 and HighRPD are nadir pavement views unlike close-range DACL10k/CODEBRIM; direct transfer often fails on viewpoint shift (the MDPI 2025 paper's 0.014 mIoU zero-shot result is the clearest evidence).

**Recommended thesis strategy.** Primary training on DACL10k v3 + synth-dacl extensions; pretrain crack class on OmniCrack30k (nnU-Net or SegFormer); fine-tune on CODEBRIM + S2DS for multi-class transfer; use ConRebSeg for exposed-rebar auxiliary; evaluate drone transfer on MBDD2025 + UAV-PDD2023 + CSSC; use DACL1k for cross-domain test.

---

## 5. Multi-label segmentation for overlapping damage classes

The DACL10k specification is explicit: *"Each pixel can be associated with several classes, e.g. a surface can have Rust and Crack."* Collapsing this to single-label by priority discards supervision — papers on the same dataset that report strong results (dacl-challenge winners reaching 0.51 mIoU) treat it as genuine multi-label.

**Standard best practice.** Stack the targets as a (N, C, H, W) binary tensor and train with **sigmoid + BCE** per class instead of softmax + CE (confirmed in the `segmentation_models_pytorch` `MULTILABEL_MODE` convention and PyTorch Forums guidance). Each class gets an independent binary head; pixel losses are summed across classes. This is the correct replacement for the user's weighted-CE baseline on DACL10k.

**Compound multi-label losses.** The most robust practical recipe for severe imbalance in multi-label pixel segmentation is `α·BCE + β·Dice(per-class) + γ·Focal-Tversky(per-class)` with per-class inverse-frequency weighting on BCE and Focal-Tversky α=0.3, β=0.7, γ=4/3 (supported by Nguyen et al., *Engineering Structures* 2023, and carried into multi-label via independent binary heads).

**Asymmetric Loss for multi-label** (Ridnik et al., ICCV 2021, "Asymmetric loss for multi-label classification") uses different γ values for positive and negative samples. Originally for image-level multi-label but transfers cleanly to pixel-level multi-label by applying per pixel, per class; particularly useful when negative (absent-class) pixels dominate — which is the DACL10k regime.

**Class-asymmetric loss / partial-label methods** (Bitter & Willems, PLOS ONE 2022). Specifically designed for heterogeneous multi-class segmentation labels where not all classes are annotated in every image. Directly applicable to DACL10k's partially annotated bridge components.

**Partial-label probabilistic loss** (Bevandic et al., "Multi-domain semantic segmentation with overlapping labels", arXiv:2108.11224). Principled loss for datasets with overlapping class definitions — useful if the thesis plans to combine DACL10k with S2DS/CODEBRIM, which have non-identical taxonomies.

**Topology-preserving multi-label** is the open gap. The user should apply **Skeleton Recall Loss** (Kirchhoff, ECCV 2024) because it is **the first multi-class-capable topology-preserving loss** — see §6.

**Alternatives to the user's label-collapse strategy:**
1. Switch the entire head to sigmoid + BCE per class with per-class inverse-frequency weighting (minimum viable change).
2. Add Dice + Focal-Tversky per class as a compound loss.
3. Use Asymmetric Loss for the positive/negative imbalance.
4. For overlapping classes with inter-class correlation (crack-in-rust), consider a **Mask2Former** style architecture where the output is a set of class-assigned masks rather than per-pixel class — this naturally supports multi-label and was the winning choice in the dacl-challenge.
5. If compute is severely constrained, keep single-label but use **dual-output heads** (main semantic head + auxiliary binary crack head with its own topology loss).

---

## 6. Small / thin structure segmentation

### 6.1 Topology and connectivity-aware losses

**clDice** (Shit et al., CVPR 2021, arXiv:2003.07311). Differentiable soft-skeletonization loss; binary and topology-preserving. `L = (1−α)·Dice + α·clDice`, α≈0.5. The reference citation for thin-structure segmentation; directly applicable to cracks. *Code: github.com/jocpae/clDice.*

**Skeleton Recall Loss — Kirchhoff et al., ECCV 2024** (arXiv:2404.03010). Replaces GPU-intensive soft-skeletonization with precomputed CPU skeletons and skeleton-recall computation: **~90 % less overhead than clDice** and, critically, **the first topology-preserving loss that supports multi-class segmentation**. Validated on concrete cracks (Tomaszkiewicz & Owerko 2023 dataset). *Code: github.com/MIC-DKFZ/Skeleton-Recall.* **Primary loss recommendation for the thesis.**

**Centerline Boundary Dice (cbDice)** (Shi et al., MICCAI 2024, arXiv:2407.01517). Refines clDice with radius and boundary awareness; fixes clDice's bias toward wide branches — important when crack widths range from hairline to millimeters.

**Homotopy Warping Loss** (Hu, NeurIPS 2022). Identifies topologically critical pixels via homotopic warping — cheaper and more targeted than persistent homology.

**Topology-Aware Focal Loss (TAFL)** (Demir et al., CVPRW 2023, arXiv:2304.12223). Focal + Wasserstein over persistence diagrams; jointly handles imbalance and topology. Computationally expensive vs Skeleton Recall.

**Skeleton-Based Loss for Concrete Cracks** (MDPI *Future Transportation* 5(4):177, 2025). Directly adapts clDice to road cracks; tunes λ and iteration count; confirms clDice improves connectivity of fragmented elongated cracks.

### 6.2 Loss functions for severe imbalance (the definitive benchmark)

**Nguyen et al. (2023), "Crack segmentation of imbalanced data: The role of loss functions", *Engineering Structures* 297:116988.** Statistically compares 12 losses across 4 crack benchmarks. **Focal Tversky (α=0.3, β=0.7, γ=4/3) is the single most robust loss for severely imbalanced crack data.** Weighted BCE, Focal, Dice, and compound losses significantly outperform vanilla CE; the gap widens with imbalance. This is the must-cite empirical reference for the loss choice.

**Optimized Hybrid Focal Margin Loss** (Chen et al. 2023, arXiv:2302.04395). Hybrid focal + large-margin softmax + focal Tversky; +0.43 IoU on DeepCrack-DB, +0.44 on PanelCrack under severe imbalance.

**Focal Tversky** (Abraham & Khan, ISBI 2019, arXiv:1810.07842) is the foundational formulation: `FTL = (1 − TI)^γ` with `TI = TP / (TP + α·FP + β·FN)`.

**Lovász-Softmax** (Berman et al., CVPR 2018) is a direct IoU surrogate, used commonly as an auxiliary alongside CE/Dice.

**Boundary DoU Loss** (Sun et al., MICCAI 2023). Adaptive boundary-focused loss; modest benefit for cracks specifically because thin cracks are almost entirely boundary.

**Active Boundary Loss** (AAAI 2022, arXiv:2102.02696). Boundary alignment via distance transforms; pairs well with Lovász for thin objects.

### 6.3 Multi-scale / high-resolution architectures addressing the SegFormer 1/4 problem

The user's SegFormer 1/4-resolution concern is real and well-documented in the SegFormer GitHub issue tracker. Five mitigation paths are published:

1. **HrSegNet** (Li 2023) keeps the main path at 1/2 original resolution and beats HRNet, SegFormer, DDRNet, U-Net on CrackSeg9k/RCD — the direct architectural fix.
2. **HRNet / HRNet-OCR** maintains multi-resolution streams throughout; 80.90 % mIoU on RCD baseline.
3. **Segformer++** (Hanisch 2024) uses token merging to ingest full-resolution input efficiently.
4. **CA-SegFormer** (2025) adds coordinate attention in the SegFormer decoder: +2 % IoU vs vanilla SegFormer on concrete cracks.
5. **Edge-feature-fusion SegFormer** (Nature Sci Rep 2025) adds an Edge Extraction Module between Transformer Block 1 and the decoder to recover detail lost in the 1/4 bottleneck.
6. **DSCformer** adds Dynamic Snake Convolution to a SegFormer branch for tubular geometry.

Empirical head-to-head ranking on CrackSeg9k / RCD: UNet (~76.7 %) ≈ DDRNet < SegFormer vanilla (~77 %) < DeepLabV3+ (78.3 %) < HRNet-OCR (80.9 %) < **HrSegNet (~82 %)** < **CrackMamba / SCSegamba**. For pure thin-crack fidelity, HRNet-family or Mamba-family architectures dominate.

### 6.4 Patch / tiling strategy for 4K drone frames

**NIST Exact Tile-Based Segmentation** (J. Res. NIST 2021). Halo = half the effective receptive field; tiles stride by `(tile_size − halo)`; input tiles overlap, output tiles join exactly at seams with **zero tiling artifacts**. The gold-standard recipe.

**Flip-n-Slide** (ICLR 2024 ML4RS Workshop). Each pixel appears in 8 tiles with 0–75 % overlap + flips. **+15.8 % precision for under-represented classes** — directly relevant for under-represented DACL10k classes like Crack and Cavity. **Because the thesis pipeline is offline, the 8×-inference cost of Flip-n-Slide at test time is acceptable.**

**Hernández-Vázquez et al. (MDPI RS 2024)** statistically test tile sizes 256/512/1024 with 0 % and 12.5 % overlap for road extraction — larger tiles with small overlap typically best for curvilinear features, the closest analogue to cracks.

**Practical recipe for 4K drone video in an offline pipeline:** 1024×1024 tiles (or 1536² if GPU memory permits), 25–50 % overlap using NIST exact-tile inference with halo equal to the network's effective receptive field, Gaussian-weighted blending at seams, Flip-n-Slide (8×) at both training and test time. HRCSF (Chu 2024) demonstrates 4K at 90.89 % IoU and 3.84 FPS on RTX 3060 — a realistic performance target on commodity hardware.

---

## 7. Drone, video, and offline batch processing

**UAV-native segmentation pipelines.**  EfficientSegNet (2025, *Construction and Building Materials*): MobileNetV3 encoder + spatial attention + multi-scale fusion; 80.81 % mIoU at 60.82 FPS on CrackCB. Song et al. (2024, SCHM) validates pixel-level crack segmentation across dark-light, diverse crack, high-roughness, and complex-background drone scenarios.

**Motion blur and preprocessing.** Jung & Kim (2023), *Drones* 7:657 — GAN-based deblurring for UAV bridge imagery with validated downstream segmentation gains. Zhang et al. (2024, *CACIE*) proposes evaluating deblur quality via downstream segmentation accuracy (not only PSNR/SSIM), a useful methodology for the thesis.

**Flight planning.** **ORBIT** (Bartczak, Bassier, Vergauwen 2025, ISPRS XLVIII-2/W11). Open-source UAS flight-path-planning toolkit, field-deployed on concrete canal bridges with DJI Mavic 3 Enterprise under-deck GNSS-denied operation. Code: github.com/ErToBar2/ORBIT. **BIM-based 3D path planning** (Zou et al., 2022/2024) generates candidate viewpoints for photogrammetric reconstruction. **Kim et al. (2025), *Drones* 9:124**: waypoint planning for GNSS-shadowed bridge substructures without hardware modification.

**Video segmentation and temporal consistency.** SAM2 (Ravi et al. 2024, arXiv:2408.00714) is the dominant 2024–2026 video-segmentation foundation model — promptable in the first frame, memory transformer propagates masklets at ~44 FPS, trained on SA-V. *Code: facebookresearch/sam2 (Apache-2.0).* **SSP — Semantic Similarity Propagation** (arXiv:2503.15676, 2025) is a lightweight alternative: image model + pose-aware linear frame-to-frame propagation + temporally consistent knowledge distillation. **Harb et al. (2024, arXiv:2411.04620)** directly compares mono-temporal U-Net vs multi-temporal Swin UNETR on 1,356 × 32 crack sequences: multi-temporal IoU 82.72 % / F1 90.54 % vs mono-temporal IoU 76.69 % / F1 86.18 % at half the trainable parameters. **Temporal information helps crack segmentation even for near-static scenes.**

**Practical offline batch pipeline for the thesis.** Upload drone video → extract keyframes (temporal downsampling based on IMU / visual-overlap heuristics) → run primary segmentation model (ConvNeXt-L + Mask2Former, fine-tuned SAM2, or ensemble) on keyframes with Flip-n-Slide tiling → propagate masks through intermediate frames via SAM2 memory transformer → COLMAP/MVS (for measurement-grade geometry) + 3DGS (for digital-twin visualization) → lift 2D masks into 3D via Gaussian Grouping or LBG → serve to the web app as a navigable 3D model with queryable damage overlays. Duan 2025 and Wang/Spencer 2026 are the closest published embodiments.

---

## 8. Benchmarks, leaderboards, and what counts as "strong"

**DACL10k current state (April 2026).** Original 2024 baseline: 0.42 mIoU (DeepLabV3+ / SegFormer / HMA). dacl-challenge WACV 2024 winner: **0.51 mIoU** (8 of 23 teams beat the baseline; winners used ConvNeXt-Large / EVA-02-Large / Mask2Former ensembles). synth-dacl (2025) mainly improves weak classes (finecrack IoU +1.3 points, Cavity IoU 24.63 %). No published 2025–2026 entry clearly exceeds 0.51 mIoU on the challenge test split that we could verify. **Strong multi-class performance on DACL10k-like data is 0.45–0.55 mIoU; above 0.50 mIoU is SOTA.**

**Contextualizing the user's 0.31 mIoU baseline.** Below the published 0.42 baseline. The most likely causes, in order of impact: (1) collapsing multi-label to single-label by priority discards ~10–20 % of the supervision, which hits rare classes hardest; (2) weighted CE is inferior to Focal-Tversky + Skeleton Recall for DACL10k's imbalance; (3) DeepLabV3+ with a small backbone is outperformed by the challenge-winning ConvNeXt-L + Mask2Former at ~0.50 mIoU; (4) the 1/4-resolution bottleneck in a standard DeepLabV3+ decoder discards hairline features.

**Target range for a thesis with a batch-processing pipeline.** Because inference throughput is not a binding constraint, the thesis should target **the heavy-accuracy range (0.45–0.50+ mIoU on DACL10k test) rather than a lightweight 0.40 mIoU**. Stacked improvements expected to reach this range: sigmoid + BCE multi-label head (+2–5 mIoU); Focal-Tversky + Skeleton Recall (+2–4); ConvNeXt-L or EVA-02 backbone with Mask2Former head (+6–10 vs the current DeepLabV3+ small backbone); synth-dacl auxiliary pretraining (+1–2 on weak classes); Flip-n-Slide tiling at full resolution (+2–3); small 2–3-model ensemble (+1–3). A single-model reference in the 0.42–0.46 range and an ensemble reference at 0.48–0.51 are both realistic defensible targets.

**Other key benchmarks.** S2DS ≈ 92 % mIoU (HMA baseline). OmniCrack30k ≈ 64 % clIoU4px (nnU-Net). CrackSeg9k ≈ 77 % mIoU (DeepLabV3), higher with SAC. CODEBRIM ≈ 72 % balanced accuracy (classification). ConRebSeg val mIoU 0.59 (YOLOv8L-seg). Dam spillway multi-class seg 65–85 % mIoU.

---

## 9. Reviews and survey papers (2022–2026) — high-value starting points

**Top three to read first:**

1. **Cha, Ali, Lewis, Büyüköztürk (2024), "Deep learning-based structural health monitoring", *Automation in Construction* 161:105328.** First holistic review of DL for SHM spanning vibration, vision, and hybrid data; taxonomies DL operations × SHM tasks × data modalities. Critical anchor for Introduction/Background. (sciencedirect.com/science/article/pii/S0926580524000645)
2. **Amirkhani, Allili, Hebbache, Hammouche, Lapointe (2024), "Visual concrete bridge defect classification and detection using deep learning: a systematic review", *IEEE T-ITS* 25(9):10483–10505.** Most directly on-topic survey with ready-made taxonomies, dataset tables, and CODEBRIM/MCDS benchmarks. (ieeexplore.ieee.org/document/10452281)
3. **Deng, Singh, Zhou, Lu, Lee (2022), "Review on computer vision-based crack detection and quantification methodologies for civil structures", *Construction and Building Materials* 356:129238.** Best gap-analysis for segmentation + quantification; its **seven explicit gaps** (data, background, generalization, lightweight, context, semi-supervised, attention/quantification) map directly onto thesis chapters.

**Additional surveys:**
- Nguyen, Tran et al. (2023), "Deep Learning-Based Crack Detection: A Survey", *Int. J. Pavement Research and Technology* 16(4):943–967.
- König, Jenkins, Mannion, Barrie, Morison (2022), "What's Cracking?", arXiv:2202.03714. Widely cited dataset+metric catalog.
- Hamishebahar et al. (2022), *Applied Sciences* 12(3):1374.
- Pan, Ma, Mai, Hu (2025) review of data-driven CV for structural damage (PMC12511193) — best "state of 2025" complement to Cha 2024.
- Jiang et al. (2023), *Sensors* 23(18):7863 — CV-based bridge inspection and monitoring.
- Xu, Xu, Forde, Caballero (2023), *Construction and Building Materials* 402:132596 — ML/DL for concrete and steel bridge SHM algorithm selection.
- Li, Wang, Wang, Li, Vimlund (2022), *J. Traffic and Transportation Engineering* 9(6):945–968 — pixel-level crack detection review (segmentation-only).
- Zhou et al. (2025), *Int. J. Digital Earth* — UAV-based structural monitoring (best UAV-specific survey).
- "Deep Learning for Crack Detection: A Review of Learning Paradigms" (arXiv:2508.10256, 2025) — best framing for foundation-model / SAM direction.
- Ali et al. (2024), *Frontiers in Built Environment* — 85-paper Scopus review of 2022–23 crack detection.

---

## 10. 2D-to-3D integration: 2D masks onto photogrammetry / NeRF / Gaussian Splatting

**Foundational lifting machinery.** Semantic-NeRF (Zhi et al., ICCV 2021) established the paradigm of adding a semantic head to a neural radiance field and showed multi-view aggregation denoises noisy labels. Panoptic NeRF (Fu et al., 3DV 2022) combined 3D bounding primitives + 2D cues on KITTI-360. **Panoptic Lifting** (Siddiqui et al., CVPR 2023) is the most-cited reference architecture for lifting noisy machine-generated 2D panoptic masks to 3D via linear assignment of per-view instance IDs with COLMAP poses.

**Photogrammetry-based bridge pipelines.** **Majidi, Omidalizarandi & Sharifi (2023)**, ISPRS Annals X-4/W1: DeepLabV3+ (Xception + SE) crack segmentation → COLMAP SfM/bundle adjustment → scale bars + GCPs → cracks on 3D point cloud at true scale. **Ni et al. (2024)**, *Structural Control and Health Monitoring* 2024:9988793: UAV flight planning → SfM → YOLOv7 + SIFT matching → damage-to-model correlation; recommends 20 m inspection distance, centimeter model positioning, meter-level damage positioning.

**Depth-aware 2D segmentation.** **Zhang, Wan & Todd (2025)**, *Advanced Engineering Informatics*. RGB-D from SfM point cloud for ROI extraction (F-measure 98.85 %) + improved DeepLabV3+ for damage segmentation (mAP 82.21 %). **Clear evidence that 3D priors improve 2D segmentation** for bridges.

**Hybrid photogrammetry + super-resolution.** Tang et al. (2025), *J. Building Engineering*. YOLO-CSD ROI → U-Net-SR super-resolved crack segmentation → COLMAP SfM + improved CasMVSNet MVS → pier-level 3D mapping. **Demonstrates that super-resolved 2D masks materially improve 3D projection quality for thin cracks.**

**NeRF-based bridge reconstruction.** **Kim & Cha (2024/2025), ABM-Nerfacto**, *Automation in Construction* and *J. Structural Health Monitoring*. Nerfacto with multi-head attention modules (DANN-A, DANN-B) + STRNet with test-time augmentation for pixel-wise crack segmentation, integrated into a 3-span continuous concrete bridge digital twin. **Most directly comparable academic work to the thesis — flag as primary NeRF baseline.**

**Gaussian Splatting for bridge inspection (active 2024–2026 frontier).**
- **Duan et al. (2025)**, *JCSHM* 15:3345–3360 — **the closest published analogue to the thesis**: monocular video → SAM2 propagation → 3DGS → 3D defect visualization.
- **Wang, Nie, Narazaki, Matiki, Spencer Jr. (2026)** (arXiv:2602.16713) — GS-enabled digital twin for 3D damage visualization; explicitly reports **cross-view fusion reduces 2D segmentation errors**, multi-scale reconstruction strategy. Flagship 2026 reference.
- **AVGS (2025)**, *Mechanical Systems and Signal Processing*. Fine-tuned SAM multi-view crack masks + Adaptive View 3DGS with geometric consistency; tested on 3 real engineering structures.
- **Gaussian Grouping** (Ye et al., ECCV 2024, github.com/lkeab/gaussian-grouping) — lifts 2D SAM masks into per-Gaussian identity features via contrastive 3D feature field.
- **Feature-3DGS** (Zhou et al., CVPR 2024) — distills 2D foundation features (CLIP-LSeg, SAM) into a Gaussian feature field via parallel N-dim rasterizer.
- **Lifting By Gaussians (LBG)** (Chacko et al., WACV 2025, arXiv:2502.00173) — **no training required**: lifts SAM/FastSAM masks + DINOv2/CLIP features onto pre-trained 3DGS via max-contributor Gaussian per pixel. 10× faster than contrastive methods. **Strong candidate baseline.**
- **PanSt3R** (arXiv:2506.21348, 2025) — joint 3D geometry + multi-view panoptic segmentation feed-forward (no test-time optimization); shows 3D-aware training improves 2D segmentation.
- **Contrastive Gaussian Clustering** (ECCV 2024) and **MVC-PSU** (AAAI 2025) — mechanisms for handling inconsistent 2D masks across drone frames.

**Best-practice 2D→3D pipeline for bridge inspection (2026).** (1) BIM/cross-section path planning (ORBIT, BIM-driven Zou 2022/2024). (2) Capture with ≤1 mm/px GSD for hairline cracks at 1–2 m distance and ~20 m for overview; 70 %+ overlap; RTK/GCP ground truth. (3) GAN deblur preprocessing. (4) 2D segmentation at **native resolution** (fine-tuned SAM/SAM2 or HrSegNet). (5) Reconstruction split: photogrammetry (COLMAP/MVS) for measurement-grade geometry (±1–4 mm with GCPs) **and** 3DGS for visualization/digital twin. (6) Mask lifting via LBG (fastest, no retraining) or Gaussian Grouping / AVGS (highest quality on fine/inconsistent masks). (7) Close the loop: orthomosaic verification, queryable semantic digital twin. Duan 2025 and Wang 2026 are the closest published embodiments.

**Output-mask resolution matters critically for cracks.** Cracks are 1–5 pixels wide at typical drone distances. 2–4× downsampling can collapse cracks to zero pixels on some Gaussians, producing phantom gaps in the 3D semantic field. Tang 2025 uses U-Net-SR super-resolution precisely because native-resolution masks were insufficient. **Run segmentation at ≥ native camera resolution; tile rather than downsample if memory-bound**; match Gaussian/point density to expected feature scale.

---

## 11. Synthesis: five key takeaways for the thesis

**1. Switch from single-label + weighted CE to true multi-label with a compound loss.** The single highest-leverage change is to abandon the priority-based label collapse and train with sigmoid+BCE per class, combined with Focal-Tversky (for imbalance) and Skeleton Recall Loss (ECCV 2024, for topology preservation on thin classes, multi-class-capable). Nguyen et al. 2023 is the definitive empirical benchmark validating Focal-Tversky; Kirchhoff et al. 2024 is the definitive multi-class topology-preserving loss. This alone should move the DACL10k subset baseline from ~0.31 toward the published ~0.42.

**2. For an offline batch pipeline, the primary model should sit at the heavy-accuracy end of the landscape.** Because inference throughput is not a binding constraint, the thesis should target the architecture family that reached the dacl-challenge SOTA (~0.51 mIoU): **ConvNeXt-Large or EVA-02-Large encoder + Mask2Former head, trained with the compound multi-label loss from Takeaway 1, with a small 2–3-model ensemble at the end**. This is the primary recommendation. HrSegNet, SegFormer-with-edge-fusion, and hybrid CNN-transformer architectures (Hybrid-Segmentor, CA-SegFormer, PSC) remain valuable as **lightweight baselines for ablation studies** and as **ensemble members with architectural diversity**, and deserve discussion in the deployability / edge-inference section of the thesis — but they are not the primary target for peak accuracy. Mask2Former with ConvNeXt/EVA-02 is also the natural architecture for multi-label output: it produces a set of class-assigned masks rather than per-pixel class labels, which cleanly handles overlapping damages. The HrSegNet paper (Li 2023) remains the single best empirical reference for the resolution-vs-accuracy tradeoff and should be cited when justifying mask-resolution choices for the 3D projection step.

**3. Fine-tuned SAM2 + 3DGS is the natural architectural pattern for the thesis's offline video-to-3D pipeline.** Duan et al. 2025 (JCSHM) and Wang/Spencer 2026 (arXiv:2602.16713) are direct blueprints and their pipeline matches the thesis's constraints almost exactly: monocular video → SAM2 keyframe prompting → memory-transformer mask propagation → COLMAP SfM/MVS for metric geometry → 3DGS for the navigable digital twin → Gaussian Grouping / LBG / AVGS to lift 2D masks into 3D. Zero-shot SAM is inadequate on cracks, but LoRA/LayerNorm fine-tuning (CrackSAM, SAC, Crack-EdgeSAM) brings SAM/SAM2 to or above supervised SOTA with ~1 % trainable parameters. A practical architecture for the thesis is a two-stage ensemble: **(a) Mask2Former-based primary model for multi-class DACL10k damages on keyframes** (Takeaway 2) and **(b) fine-tuned SAM2 as a crack-specialist model that also handles video propagation across intermediate frames**. Combining these gives both multi-class coverage and strong thin-structure fidelity, and the SAM2 propagation is the mechanism by which masks become temporally consistent before they are lifted into 3D. The web-app front-end then consumes the 3DGS model directly (via a Three.js / Gaussian-splatting viewer) with damage overlays rendered as per-Gaussian class attributes.

**4. Pretraining and synthetic augmentation can rescue rare classes.** The strongest evidence-based augmentations for DACL10k weak classes are (a) **synth-dacl** for Crack/Cavity class balancing, (b) **OmniCrack30k** pretraining for the Crack class, (c) **ConRebSeg** pretraining for exposed rebars, and (d) **S2DS/CODEBRIM** multi-class fine-tuning with partial-label loss handling (Bitter/Willems or Bevandic formulations). Synthetic defect data specifically for fine-grained DACL10k classes is a validated 2025 approach.

**5. The thesis's 2D mask resolution and output format must be matched to 3D downstream use.** Coarse or downsampled masks destroy 3D fidelity for cracks. Best-practice is native-resolution tiled inference (NIST exact-tile; Flip-n-Slide 8× test-time augmentation — affordable in the offline pipeline; 1024² or 1536² tiles with 25–50 % overlap, Gaussian blending). The 3D pipeline should combine photogrammetry (COLMAP+MVS) for measurement-grade geometry and 3DGS (via Gaussian Grouping, LBG, or AVGS) for visualization-grade semantic digital twin.

## 12. Open problems particularly relevant to a drone-based end-to-end thesis

- **DACL10k → drone-native generalization is essentially unsolved.** Zero-shot transfer of DACL10k-trained models to UAV imagery achieves mIoU 0.014 before domain adaptation (PMC12609025, 2025). A thesis contribution could be a disciplined domain-adaptation protocol from DACL10k (mixed acquisition) to drone-native data using MBDD2025/UAV-PDD2023/CSSC as target domains.
- **Pixel-level DINOv2 for concrete segmentation is unexplored.** Zhu et al. 2026 showed strong classification; no published paper uses upsampled DINOv2 features for pixel-level concrete damage segmentation as of April 2026.
- **Open-vocabulary / CLIP-based segmentation for concrete damage is essentially unexplored**, despite being an active area for general scenes.
- **Multi-class topology-preserving losses are nascent.** Skeleton Recall Loss (ECCV 2024) is the first; no paper has yet combined Skeleton Recall with Asymmetric Loss for multi-label pixel segmentation on DACL10k — this is a low-risk, high-value methodological contribution.
- **End-to-end drone-video-to-3D-damage-map benchmarks do not exist.** Duan 2025 and Wang 2026 show the pattern works but use custom datasets. A thesis could contribute a reproducible drone-video benchmark with 3D ground-truth damage localization.
- **Temporal consistency metrics for crack-propagation video.** Harb 2024 shows multi-temporal helps, but standard metrics like clIoU-over-time or Betti-error-over-time are not yet standardized for thin-structure video segmentation.
- **Output-format standardization for 3D downstream use.** No published work rigorously characterizes how 2D mask resolution/quality degrades 3D projection fidelity for thin cracks as a function of Gaussian/point density — a quantifiable thesis contribution.
- **Small, ailing real-world test sets are underused.** Most published numbers come from either aggregated crack benchmarks or single custom sets; few papers validate on independent ailing bridges. DACL1k + a small UAV set would be a credible external validation protocol.

---

**Key URLs for reproducibility (open code flagged ✅):**

Datasets: dacl10k-toolkit (github.com/phiyodr/dacl10k-toolkit ✅), S2DS (github.com/ben-z-original/s2ds ✅), OmniCrack30k (github.com/ben-z-original/omnicrack30k ✅), ConRebSeg (github.com/DTU-PAS/ConRebSeg ✅), CODEBRIM (zenodo.org/records/2620293), CrackSeg9k (doi.org/10.7910/DVN/EGIEBY), RDD2022 (github.com/sekilab/RoadDamageDetector ✅), UAV-PDD2023 (zenodo.org/records/8429208), synth-dacl (arXiv:2506.14255), Concrete Crack Conglomerate (VT Figshare).

Architectures/losses: Hybrid-Segmentor (github.com/junegoo94/Hybrid-Segmentor ✅), HrSegNet (github.com/CHDyshli/HrSegNet4CrackSegmentation ✅), CrackFormer-II (github.com/LouisNUST/CrackFormer-II ✅), CrackMamba (github.com/shengyu27/CrackMamba ✅), SCSegamba (github.com/Karl1109/SCSegamba ✅), clDice (github.com/jocpae/clDice ✅), Skeleton Recall Loss (github.com/MIC-DKFZ/Skeleton-Recall ✅), Lovász-Softmax (github.com/bermanmaxim/LovaszSoftmax ✅), SAM2 (github.com/facebookresearch/sam2 ✅), bridge-damage-segmentation HRNet (github.com/jingxiaoliu/bridge-damage-segmentation ✅), ORBIT flight planning (github.com/ErToBar2/ORBIT ✅), Gaussian Grouping (github.com/lkeab/gaussian-grouping ✅), Awesome-Crack-Detection (github.com/nantonzhang/Awesome-Crack-Detection).
