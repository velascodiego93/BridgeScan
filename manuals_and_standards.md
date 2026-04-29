# Global frameworks for translating concrete bridge pat hologies into damage levels — a survey for automated drone-based inspection

## Executive summary

Across roughly 20 national and international frameworks reviewed, three **families** dominate the translation from quantified pathologies to a structured damage level, and each fits differently with a DACL10k-based drone pipeline that outputs semantic masks, 3D projection, and area/extent quantities.

**Element-based condition-state systems** (AASHTO MBEI CS1–CS4 in the US, Austroads 4–5 states in AU/NZ, Brazilian NBR 9452 0–5 grades, Costa Rican MP-2020 6 levels, Korean KISTEC A–E with fuzzy aggregation) break each element into surface area assigned to discrete states by explicit quantitative defect thresholds (crack width bins, spall size/depth, rebar-exposure category). These systems are the **most natural fit** for pixel-level segmentation, because quantity-in-state is exactly what a projected mask over a 3D element computes.

**Multi-axis scoring/risk systems** (Italian Linee Guida 2020 with G × K₁ × K₂ defect sheets → attention classes CdA; German RI-EBW-PRÜF with S/V/D scores 0–4 → Zustandsnote 1.0–4.0 via the Haardt algorithm; French IQOA 1/2/2E/3/3U; Chinese JTG/T H21 with AHP weights + Dr score and single-index override) decompose each defect into multiple weighted physical dimensions (stability, traffic safety, durability; gravity × extent × intensity). They are also automation-friendly because the catalogues embed numerical thresholds, but they require a **second aggregation layer** and, for Italy, non-visual inputs (hazard, exposure) that a drone cannot provide.

**Reliability/probabilistic assessment frameworks** (ISO 13822, prEN 1990-2 / CEN TC 250 SC10, fib Model Code 2020 Part VII, Swiss SIA 269/2, UK DMRB CS 454) prescribe structural re-analysis with updated material properties, partial factors, or Bayesian updating. A drone pipeline can feed *inputs* (crack maps, spall volumes, effective section loss) but cannot itself produce the output of these frameworks, which is a load-capacity or reliability verdict.

The single most important insight for the thesis is that **DACL10k was engineered against the German RI-EBW-PRÜF Schadensbeispiel-Katalog**, so the taxonomy aligns nearly one-to-one with German and — by extension — AASHTO MBEI defect categories. The Italian Linee Guida is the second-best target because its defect sheets explicitly enumerate every DACL10k class at element level, with G weights that can be applied directly to mask areas. We recommend a **dual-output design**: produce both an AASHTO MBEI-style CS1–CS4 quantity-in-state and an Italian Linee Guida Livello di Difettosità + contribution to CdA-Strutturale, which together satisfy "defensible to civil engineering reviewers" in both Italian and international contexts, while explicitly carrying uncertainty flags for defects (Hollowareas, chloride ingress, cover depth, section loss, settlement, scour) that are fundamentally non-observable from RGB aerial imagery.

---

## Part 1 — United States frameworks

### AASHTO Manual for Bridge Element Inspection (MBEI)

The second edition (2019) with 2022/2024/2025 interim revisions is the current reference, incorporated by federal rule into the 2022 NBIS. It is **paywalled** but reproduced verbatim in many state supplements (WisDOT, VDOT, Caltrans, IDOT). Elements are rated in four condition states — **CS1 Good, CS2 Fair, CS3 Poor, CS4 Severe** — with the inspector distributing the element's total quantity (ft, ft², each) across the four states. CS4 always triggers a **structural review**; CS2/CS3 feed the Appendix D list of feasible actions (do nothing / protect / repair / rehabilitate / replace).

**Quantitative thresholds for concrete (defects 1080, 1090, 1110, 1120, 1130, 1190)** are the most explicit in any national standard:

| Defect | CS2 trigger | CS3 trigger | CS4 trigger |
|---|---|---|---|
| Delamination / Spall / Patched Area (1080) | delamination; spall ≤ 1 in deep OR ≤ 6 in extent; patch sound | spall > 1 in deep OR > 6 in extent; patch unsound | structural review warranted |
| Exposed Rebar (1090) | exposed, **no** measurable section loss | exposed WITH measurable section loss | section loss warranting structural review |
| Efflorescence / Rust Staining (1120) | leaching without rust | heavy build-up WITH rust staining | (CS4 not used) |
| Cracking RC (1130) | unsealed 0.012–0.05 in (0.30–1.27 mm) or moderate map cracking | > 0.05 in (1.27 mm) or heavy map cracking | structural review |
| Cracking PSC (1110) | 0.004–0.009 in (0.10–0.23 mm) | > 0.009 in (0.23 mm) with efflorescence/rust | structural review |
| Abrasion / Wear (1190) | aggregate exposed but secure | aggregate loose / popped out | structural review |
| Settlement (4000), Scour (6000) | arrested with countermeasures | active with no review yet | review warranted |

The changes from the 2010 Guide for CoRe Elements are important: MBEI 2013/2019 standardised to exactly four states across all elements and **added explicit numeric thresholds** where 2010 had only "Minor/Moderate/Severe". The overlap rule — codified most clearly in the **VDOT 2016/2026 Supplement** — is that if the worst defect in a region is in CS3, that region is CS3 regardless of co-occurring CS2 defects. This is the rule the thesis pipeline must implement when fusing DACL10k classes over a projected 3D region.

### AASHTO Manual for Bridge Evaluation (MBE), FHWA BIRM, NBIS 2022, SNBI

The MBE 3rd edition (2018 + interims) is the federally-referenced load-rating document producing a **Rating Factor** for Inventory and Operating levels via LRFR/LFR/ASR. It consumes MBEI condition data through the **condition factor φc** (1.00 good / 0.95 fair / 0.85 poor) and is not itself a defect-classification system. **FHWA BIRM** (2012 baseline, 2022 NBIS-aligned revision) is freely downloadable and remains the best training reference for mapping the full BIRM defect vocabulary (scaling, pop-outs, honeycomb, stains) to DACL10k — it explicitly covers several classes (Weathering, Rockpocket, Graffiti) that MBEI does not.

The **2022 NBIS final rule** (23 CFR 650 Subpart C) and the **Specifications for the National Bridge Inventory (SNBI, FHWA-HIF-22-017, March 2022)** restructure network-level reporting. Component condition ratings on the 0–9 scale are retained for deck, superstructure, substructure, culvert, plus new SNBI items for railings, transitions, approach slab, joints, bearings, scour, channel, and deck wearing surface. The 0–9 bands are qualitative ("5 — Fair: minor section loss, cracking, spalling, or scour; 4 — Poor: advanced section loss"). Bridge Condition Classification (FHWA-HIF-18-023): lowest component rating determines Good (≥ 7) / Fair (5–6) / Poor (≤ 4). The **critical-finding threshold** moved from ≤ 3 to ≤ 2 in the 2022 rule. A drone pipeline cannot directly produce the holistic 0–9 rating but can feed the underlying defect quantities that justify it.

### Sufficiency Rating and Bridge Health Index

The legacy **Sufficiency Rating** (S1 + S2 + S3 − S4, scale 0–100) mixes structural adequacy (max 55%), functional obsolescence (max 30%), essentiality (max 15%) and special reductions. SR ≤ 80 was the historical rehabilitation-eligibility trigger; the metric was deprioritised after the 2017 Performance Management rule. The **California Bridge Health Index**, which became the AASHTOWare BrM standard, uses CS weighting factors **WF1 = 1.00, WF2 = 0.67, WF3 = 0.33, WF4 = 0.00** combined with element Failure Cost: BHI = 100 × Σ(Qᵢ·WFᵢ·FC) / Σ(TEQ·FC). This formula **directly consumes** the CS1–CS4 quantities a drone pipeline would produce, which makes it the most image-ready aggregate in US practice.

### State DOT frameworks

PennDOT (BMS2, Publication 100A), **Caltrans** (Bridge Element Inspection Manual, December 2017; Michael B. Johnson chairs AASHTO T-18), **FDOT** (Bridge Inspection Field Guide, Feb 2016 + Smart Flags), and **VDOT** (2026 Supplement) all adopt the MBEI thresholds verbatim with agency-defined elements (ADEs). **TxDOT** (Bridge Inspection Manual, September 2024) is distinctive for its explicit **25% deck-area threshold**: "repaired areas and/or spalls/delaminations exist. The combined area of distress is more than 25% of the total deck area" triggers a rating downgrade — an image-friendly percentage that a projected mask can compute directly. **NYSDOT** (RC09; 2017/2024) uses a unique **1–7 integer scale per element** (47 elements per span), aggregated into an overall condition < 5.000 = deficient; a rating of 2 is a Red Flag. Thresholds are qualitative and narrative, so NYSDOT is less automation-friendly at the per-element level.

---

## Part 2 — European frameworks

### Italy — Linee Guida MIT/CSLP 2020, DM 204/2022

Approved 17 April 2020 by CSLP, adopted by DM MIT 578/2020 and superseded by DM MIMS 204/2022 (Allegato A). ANSFISA issued operational instructions in September 2022. Chapters on multilevel inspection and classification are authored principally by Prof. Walter Salvatore (Pisa). The guidelines are **free** on mit.gov.it and cslp.mit.gov.it and apply to all bridges with total span > 6 m.

The **six-level approach** (Censimento 0, Ispezione visiva 1, Classificazione del rischio 2, Valutazioni preliminari 3, Valutazione accurata 4, Resilienza 5) is not strictly sequential; critical cases can jump directly from Level 1 to Level 4. Level 2 produces an **Attention Class (CdA)** on five bands — Bassa, Medio-Bassa, Media, Medio-Alta, Alta — computed for four independent risk domains (Strutturale e fondazionale, Sismico, Idraulico, Frane), with the **override rule** that if the structural-foundational Livello di Difettosità is Alto the overall CdA is forced to Alta.

The **defect sheets (Schede difettologiche, Allegato C)** are the heart of the image-computable portion. For each typed defect the inspector records three scalars:
- **G (Gravità)** — integer 1–5 assigned a priori in the scheda; G = 5 defects are highlighted and trigger a Tavolo tecnico, with an optional PS (Pericolo per la Sicurezza statica) flag.
- **K₁ (Estensione)** — 0.2 / 0.5 / 1.0 (linear or areal coverage).
- **K₂ (Intensità)** — 0.2 / 0.5 / 1.0.

The element defectiveness index is Σᵢ GᵢK₁ᵢK₂ᵢ / Σᵢ Gᵢ, banded into Livello di Difettosità Basso–Medio-Alto–Alto. The final CdA per domain is obtained from matrix Tab. 4.10/4.11 combining Pericolosità × Vulnerabilità × Esposizione.

Action thresholds: CdA Bassa → standard cycle; Medio-Bassa/Media → enhanced monitoring; Medio-Alta → Level 3 preliminary evaluations; Alta → Level 4 accurate verification, possible load restriction or closure; any PS flag on a G = 5 defect triggers immediate notification. Results are uploaded to **AINOP** (Archivio Informatico Nazionale delle Opere Pubbliche), with intermediate platforms such as weBridge, BRIDG€, and ispezioneponti.it.

### Germany — DIN 1076 + RI-EBW-PRÜF + SIB-Bauwerke

DIN 1076:1999-11 mandates the inspection regime (Hauptprüfung every 6 yr, Einfache Prüfung every 3 yr, annual Besichtigung, half-yearly Beobachtung) and is **paywalled**. The operational document, **RI-EBW-PRÜF 2017** with continuously updated Schadensbeispiel-Katalog (latest WISAKA 2025), is **free on bast.de**. The **Haardt (1999)** aggregation algorithm is published in a BASt OPUS report.

For each typed defect the inspector picks a Schadensbeispiel and assigns three integer scores:
- **S — Standsicherheit** (structural stability) 0–4.
- **V — Verkehrssicherheit** (traffic safety) 0–4.
- **D — Dauerhaftigkeit** (durability) 0–4.

With ±0.1 inspector adjustments. The Haardt algorithm maps each S/V/D triple to a basis-grade 1.0–4.0 via fixed look-up tables; per-Bauteilgruppe aggregation follows a worst-dominates rule with incremental contributions of 0.1–0.2 per co-occurring defect. Bauteilgruppe → Teilbauwerk → Bauwerk propagate the worst grade upward. **Zustandsnote bands**: 1.0–1.4 sehr gut; 1.5–1.9 gut; 2.0–2.4 befriedigend; 2.5–2.9 ausreichend (measures short/mid-term); 3.0–3.4 nicht ausreichend (urgent repair); 3.5–4.0 ungenügend (immediate action / traffic restriction). S = 4 or V = 4 triggers immediate measures regardless of ZN.

Cracks are binned at roughly 0.2 / 0.3 / 0.4 mm with specific S/V/D presets per orientation and structural zone; cracks in prestressing zones are treated especially severely. The **DACL10k taxonomy was deliberately engineered to match the RI-EBW-PRÜF Schadensbeispiele** (Flotzinger et al., WACV 2024), so every DACL10k class has a direct German counterpart: Crack ↔ Riss, ACrack ↔ Netzrisse/Krakelee, Spalling ↔ Abplatzung, ExposedRebars ↔ freiliegende Bewehrung, Rust ↔ Bewehrungskorrosion, Efflorescence ↔ Ausblühungen, Wetspot ↔ Durchfeuchtung, Rockpocket ↔ Kiesnester, Cavity ↔ Fehlstellen, Hollowareas ↔ Hohlstellen (confirmation by sounding), Weathering ↔ Verwitterung, Restformwork ↔ Schalungsreste, Graffiti ↔ Graffiti.

### Switzerland — SIA 269 series + ASTRA + KUBA

SIA 260–267 (new structures) are complemented by SIA 269 (2011, revised 2024) on existing structures, with SIA 269/2 (concrete) providing rules for updating material characteristics, geometry, and refined resistance models. The SIA approach is **analytical, not primarily a scoring system**: **Erfüllungsgrad n = Rd,act / Ed,act**; n < 1 triggers consideration of remediation, subject to proportionality. The inspection layer is provided by **ASTRA 12002** (2005) and the KUBA handbooks (ASTRA 62011–62016, 2021). **KUBA**, developed by Unit Solutions since 1994, manages > 4300 federal bridges with a classical 5-level Zustandsklassifikation (1 Gut → 5 Alarmierend). SIA standards are paywalled; ASTRA/KUBA handbooks are free on astra.admin.ch. The drone pipeline can support the Zustandsklasse data-entry but **cannot** produce the SIA 269/2 re-analysis, which requires cores and NDT.

### France — ITSEOA 2010 + IQOA + SIAMOA

The ITSEOA 2010 (Instruction Technique pour la Surveillance et l'Entretien des Ouvrages d'Art) governs the inspection regime: annual CEI check, tri-annual IQOA visit, six-yearly Inspection Détaillée Périodique, plus IDI at handover and reinforced/high surveillance for pathological assets. **IQOA** methodology (Sétra/Cerema, main guide 1996, sectoral guides 2005–2014, free on piles.cerema.fr) outputs five classes plus mentions:

| Class | Meaning | Action |
|---|---|---|
| 1 | Bon état apparent | Entretien courant |
| 2 | Défauts mineurs sans risque d'évolution | Entretien spécialisé |
| 2E | As 2 with risk of evolution | Priorité spécialisée |
| 3 | Structure altérée, sans urgence | Réparation planifiée |
| 3U | Structure altérée + **urgence** | Réparation urgente, restrictions possible |
| S (mention) | Insécurité usager | Intervention sécurité |

The global class = max over elements following the priority "*" > 3U > 3 > 2E > 2 > 1. Crack width classes are roughly < 0.2 mm / 0.2–0.3 mm / > 0.3 mm, less codified numerically than Germany. The **2 vs 2E distinction is kinetic** (will the defect evolve?) — not directly readable from a single image, but derivable from repeat drone surveys, which is an interesting angle for the thesis.

### United Kingdom — DMRB CS 450 / CS 454 / CS 470 + BCI

**CS 450** (National Highways, April 2021) replaces BD 63/17 and mandates Safety, General (~2 yr), Principal (6 yr) and Special inspections, with digital defect recording and photography. **CS 454** handles structural assessment (post-BD 21/BA 16/BD 37) and **CS 470** handles risk-based management of sub-standard structures. All three are **free** on standardsforhighways.co.uk.

The **Bridge Condition Indicator (BCI)**, defined by CSS (2002) and ADEPT (2007) and operated via Bridgestation, is the UK's image-friendliest scoring matrix:
- Element Condition Score (ECS) 1–5 from Severity × Extent.
- Severity A–D (HA) or 1–5 (CSS); Extent A–E = <5% / 5–20% / 20–50% / 50–80% / >80% of element area.
- ECI = ECS × ECF (Element Condition Factor, weighted by Element Importance).
- **BCI_av = Σ(ECI·EIF) / Σ EIF**; **BCI_crit = max(ECI) on Very-High-importance elements**.
- **BCI = 100 × (5 − SCS) / 4**; SOGR threshold **BCIcomb = 0.6·BCI_av + 0.4·BCI_crit ≥ 65**.

The Severity × Extent matrix maps directly to segmentation-mask area percentages, making BCI the cleanest image-native scoring scheme in Europe.

### Eurocodes, ISO, fib, ACI, RILEM, PIARC

**EN 1992-1-1 Table 7.1N** is the canonical crack-width reference: wmax = 0.4 mm (X0/XC1; UK NA 0.3 mm), 0.3 mm (XC2/XC3/XC4), 0.3 mm on chloride-exposure classes XD/XS for RC; 0.2 mm + decompression for PSC. EN 1992-2 tightens this to 0.2 mm under frequent combination on prestressed bridges.

**ACI 224R-01 Table 4.1** supplies the benchmark tolerable widths: 0.41 mm dry air, 0.30 mm humid/moist, 0.18 mm deicing chemicals, 0.15 mm seawater, 0.10 mm water-retaining. **ACI 201.1R-08** provides the standard English lexicon for visual concrete defects and maps almost one-to-one to DACL10k classes.

**prEN 1990-2 / CEN TC 250 SC10** (draft toward 2nd-generation Eurocodes) and **ISO 13822:2010** define reliability-based assessment of existing structures, consumer codes for drone-sourced defect data rather than producers of a damage class. **fib** outputs most relevant for the thesis are **Bulletin 17** (2002, Management/Maintenance/Strengthening), **Bulletin 59** (2011, Condition control for RC in corrosive environments — defines initiation vs propagation stages), **Bulletin 80** (2016, Partial factor methods for existing concrete structures), **Bulletin 111** (June 2024, Modelling structural performance of existing concrete structures) and **fib Model Code 2020 Part VII** (published 2023/2024), which unifies new and existing structures with three assessment categories. Note: a "fib Bulletin 243" as cited in the thesis brief does not match the current fib catalogue (highest 2025 numbers are in the 110s); the most likely intended references are Bulletin 80, Bulletin 111, or MC 2020 Part VII — this should be corrected in the thesis.

**RILEM TC 154-EMC** defines electrochemical thresholds used globally: half-cell potential < −350 mV CSE (high corrosion probability), resistivity < 50 Ω·m (high risk), i_corr > 1 µA/cm² (high corrosion rate). These are non-visual and mark the NDE boundary beyond which drone imaging cannot reach.

**PIARC TC 4.2 Report 2022R20EN** "Advancement of Inspection Techniques" (free on piarc.org) and the web-based **PIARC Road Asset Management Manual** (road-asset.piarc.org, TC 3.3) endorse UAV inspection and 3D photogrammetric defect documentation — giving the thesis an explicit multilateral legitimation.

---

## Part 3 — Asian and Australasian frameworks

### Japan — MLIT Periodic Inspection Guidelines + JSCE Maintenance

Triggered by the 2012 Sasago tunnel collapse, a 2013 Road Act amendment and 2014 Ministerial Ordinance mandate **close-visual inspection every 5 years** on all ~730,000 road bridges, with 2019/2024 amendments enabling equivalent remote/sensor-assisted methods. **Judgment categories** are I Good, II Preventive maintenance phase, III Early repair phase, IV Emergency repair phase; **action urgency** uses A, B, C1, C2, M, S, E1, E2 (E1/E2 = emergency). Japan deliberately **does not fix mm-precise crack-width cut-offs** in the ordinance (Jeong et al. 2018; Shirato & Tamakoshi 2013); inspectors use photo-keyed rubrics and JSCE Standard Specifications for Concrete Structures — Maintenance (2001, rev. 2007/2013/2018/2022) as technical baseline. MLIT actively builds AI training data and allows UAV inspections under the 2019/2024 updates.

### South Korea — KISTEC / KALIS + SASMI

The Special Act on Safety Control and Maintenance of Establishments (1995, rev. 2018) mandates a four-tier inspection regime (routine, regular, in-depth, precision safety diagnosis) on all Class 1/2 facilities, administered by KISTEC/KALIS. Condition grades **A Excellent, B Good, C Fair, D Poor, E Critical** are assigned per defect, member, component, and bridge, using **fuzzy aggregation with importance factors**. English-accessible crack-width grade cut-offs (Jeong et al. 2018): ≤ 0.1 mm (A/B), 0.1–0.3, 0.3–0.5, 0.5–1.0, > 1.0 mm (E); PSC is stricter. Grade D triggers use restriction, Grade E triggers immediate suspension. The BMS dataset is structured and already used for ML (Choi et al. 2025, LightGBM 80–96% component-grading accuracy), making it the most data-ready Asian system.

### China — JTG/T H21-2011 + JTG 5120-2021 + JTG 5210-2018

JTG/T H21 produces per-defect damage codes (1–5) and per-member deductions from 100 aggregated by AHP to part-level indices PMCI/PCCI, then bridge-level **Dr (weighted) and BCI (single-index control)**. Bands: Class 1 (95–100 intact), 2 (80–95 routine), 3 (60–80 minor repair), 4 (40–60 major repair), 5 (< 40 dangerous, closure/reconstruction). The **single-index control** (if any key component reaches Class 5, the whole bridge is Class 5) is a distinctive safeguard against dilution and is directly useful for DL pipelines that may over-count minor defects but reliably find severe ones. Exact mm/% thresholds per damage type are tabulated in the standard but not verifiable in English.

### India — IRC SP:35, SP:37, SP:40, SP:52 + IBMS

IRC SP:35-1990 (not revised since) uses a 0–9 per-component rating analogous to FHWA NBI with narrative (not mm-precise) thresholds; IS 456-2000 serviceability limits (0.3 / 0.2 / 0.1 mm by exposure) are imported by reference. The **Indian Bridge Management System** (MoRTH, launched Oct 2016; ~172,000 structures) assigns a Structural Rating Number (SRN 0–9), a Socio-Economic Rating Number (SEBRN), and an aggregate 0–100 score; **< 75 is critical** and triggers detailed investigation. Narrative thresholds make DL mapping coarse.

### Australia / New Zealand — Austroads + TfNSW

The **Austroads Guide to Bridge Technology Part 7** (Edition 2.1, 2024; free on austroads.gov.au) and state manuals (TfNSW, VicRoads, TMR, SA DIT, Main Roads WA, NZTA) adopt an AASHTO-style 4-state (or 5-state) condition system at element level. Representative thresholds from SA DIT RSIM Part 3: crack ≤ 0.3 mm (State 1/2), 0.3–0.5 mm, 0.5–1.0 mm (State 3), > 1.0 mm or active (State 4/5); spalls < 25 mm deep without rebar exposure (State 2), > 25 mm with rebar exposure (State 3). The 2024 Austroads update introduces a "Rosetta Stone" to reconcile condition states with individual defect records, explicitly supporting digital/AI workflows. Per-element **Bridge Health Index** follows AASHTOWare logic.

---

## Part 4 — Latin American frameworks

### Brazil — DNIT 010 + NBR 9452:2023 + SGO

DNIT 010/2004-PRO (updated 2024) and **ABNT NBR 9452:2023** (expanded to concrete, steel, and composite) score each part of the structure on three independent axes — **Estrutural, Funcional, Durabilidade** — from **0 emergencial to 5 excelente**. Critical defects listed in the norm include cracks w > 0.3 mm (per NBR 6118 limits), exposed rebar, concrete disintegration, deflection, and dente-Gerber damage. Grades ≤ 2 on the structural axis trigger short-term intervention; 0 triggers emergency response. The SGO BMS stores the grades and photographs.

### Mexico — SIPUMEX + IMT SIAP

Developed by SCT/SICT in the early 1990s from DANBRO, SIPUMEX scores each of ~6,500 federal bridges on an integer 0–5 scale combining weighted element grades: 0 recently built → 5 extreme danger, with inspection intervals 5 yr → 1 yr correspondingly. IMT publications (PT 348 and newer) summarize the methodology.

### Colombia — SIPUCOL + INVIAS

Regulated by Resolución 1528/2017, SIPUCOL (INVIAS, 1996, DANBRO-derived) scores each element on durabilidad / estabilidad / servicio from the Manual para la Inspección Visual de Puentes y Pontones, which is a heavily photo-illustrated catalogue (free on invias.gov.co). This photographic typology is **one of the best open references in the region** for aligning DACL10k masks with a national defect vocabulary.

### Argentina (SIGMA Puentes, DNV + UNC, 2007+, DANBRO-derived), Chile (MOP-DV MC Vol. 7 — weak, AASHTO-borrowed), Peru (MTC Manual de Puentes 2018 — AASHTO design-focused, supplemented by imported LanammeUCR practice), Costa Rica (**MP-2020/MP-2024** from MOPT — six-level AASHTO-aligned, best and most photo-ready in Central America, free on mopt.go.cr), Uruguay and Ecuador (no structured public inspection framework — borrow from AASHTO) complete the regional survey. **SIAPUC (Chile) could not be verified in any MOP document** and should be treated as non-existent unless the thesis locates a primary source. **Multilaterals (World Bank, IDB, CAF, Infralatam) finance BMS but do not publish their own inspection methodologies.**

---

## Part 5 — Comparative matrix of frameworks

| Framework | Country / issuer | Year | Level | Output | Quantitative thresholds | Co-defect aggregation | Image-automation fit | Open access |
|---|---|---|---|---|---|---|---|---|
| AASHTO MBEI | US / AASHTO | 2019 + interims | Element | CS1–CS4 quantity | Explicit (1 in spall depth; 6 in extent; 0.3/1.3 mm RC cracks; 0.1/0.23 mm PSC) | Worst-defect-controls per region (VDOT rule) | **High** — area-in-state direct | No (paywalled; state copies free) |
| AASHTOWare BHI | US / AASHTO | ongoing | Bridge / network | 0–100 index | WF = 1.00 / 0.67 / 0.33 / 0.00 | Failure-cost weighting | **High** — consumes CS quantities | Algorithm free (FHWA-HRT-15-081) |
| SNBI / NBIS 0–9 | US / FHWA | 2022 | Component | 0–9 holistic | Qualitative bands | Worst component = bridge class | Medium (holistic judgement) | Yes |
| Sufficiency Rating | US / FHWA | legacy | Network | 0–100 | Formula | S1+S2+S3–S4 | Indirect | Yes |
| State DOTs (TxDOT 25%; NYSDOT 1–7; VDOT; Caltrans; PennDOT; FDOT) | US | current | Element + component | Varies | MBEI + extras; TxDOT **25% deck area** | Per-manual | Medium–High | Yes |
| Linee Guida MIT | Italy / CSLP | DM 204/2022 | Multi (element → bridge → network) | CdA 5 bands × 4 domains | G(1–5) × K₁(0.2–1.0) × K₂(0.2–1.0) + PS flag | Σ GK₁K₂ / ΣG; CdA matrices; PS override | **High** for structural branch; non-visual for seismic/hydraulic | Yes (mit.gov.it) |
| RI-EBW-PRÜF + DIN 1076 | Germany / BMV + BASt | 2017 (cat. 2025) | Element → Teilbauwerk → Bauwerk | Zustandsnote 1.0–4.0 | Crack 0.2/0.3/0.4 mm; cover-loss %; spall area | Haardt algorithm (worst + increments) | **Excellent** (DACL10k designed on this) | RI-EBW free; DIN paywalled |
| SIA 269/2 + ASTRA + KUBA | Switzerland | 2024 | Structural + inspection | Erfüllungsgrad n; Zustandsklasse 1–5 | Durability limits from SIA 262 | Worst-class + analytical re-analysis | Low for n (needs NDT); Medium for ZK | ASTRA free; SIA paywalled |
| IQOA + ITSEOA | France / Cerema | 2010 | Element + bridge | 1 / 2 / 2E / 3 / 3U + S | Crack classes ≈ 0.2/0.3 mm; extent qualitative | Worst class with priority order | High for class; needs temporal for 2→2E | Cerema free |
| DMRB CS 450/454/470 + BCI | UK / National Highways | 2020–21 | Element → bridge → network | BCI 0–100; SOGR ≥ 65 | Extent A–E (<5% ... >80%); Severity 1–5 | BCI_av + BCI_crit weighted | **Very high** (image-native matrix) | Yes |
| EN 1992-1-1; EN 1992-2 | CEN | current | Design + serviceability | wmax limit | 0.4/0.3/0.2 mm by exposure | — | High for crack binning | Paywalled |
| prEN 1990-2 / ISO 13822 | CEN / ISO | draft / 2010 | Reliability assessment | Pass/fail per LS | β adjustable | Probabilistic | Indirect | Partial |
| fib MC 2020 + Bull. 59/80/111 | fib | 2020–24 | Assessment | Partial-factor verdict | Electrochemical & mm | Semi-probabilistic | Indirect | Paywalled |
| ACI 224R-01; ACI 201.1R | ACI | 2001/2008 | Design/visual | Tolerable widths / lexicon | **0.41/0.30/0.18/0.15/0.10 mm** | — | High for cracks; High for lexicon | Paywalled |
| MLIT Periodic Inspection | Japan / MLIT | 2014/2019/2024 | Member → bridge | I/II/III/IV + A–E urgency | Narrative (no fixed mm) | Worst-member + escalation | Medium | Primary in Japanese |
| KISTEC / SASMI | Korea | 2018+ | Defect/member/bridge | A/B/C/D/E | Crack bins ≤ 0.1 ... > 1.0 mm | Fuzzy weighted importance factors | **High** (quantitative + structured BMS) | Secondary only |
| JTG/T H21 + JTG 5120/5210 | China / MoT | 2011/2021/2018 | Part + bridge | 1–5 (Dr) with single-index override | Tabulated mm/% (Chinese only) | AHP weighted + worst-key override | High | Chinese primary |
| IRC SP:35 + IBMS | India | 1990 + 2016 | Component + bridge | 0–9 + SEBRN + 0–100 | Narrative; IS 456 wmax | Worst component dominates | Low–medium | Yes |
| Austroads AGBT Pt 7 + TfNSW/SA/TMR | AU + NZ | 2024 | Element | CS 1–4 or 1–5 + BHI | Crack 0.3/0.5/1.0 mm; spall 25 mm | Quantity-weighted → BHI | **High** | Yes |
| NBR 9452:2023 + DNIT 010 | Brazil | 2023 | Part | 0–5 × 3 axes | w > 0.3 mm; NBR 6118 limits | Manager prioritisation | **High** | NBR paid; DNIT free |
| SIPUMEX | Mexico | 1990s+ | Bridge weighted | 0–5 | Narrative | Weighted aggregate | High for grade; per-defect less | Methodology free |
| SIPUCOL | Colombia | 1996 + Res. 1528/2017 | Element | 3 axes + codes + photo catalogue | Field-quantified | Aggregated | **Very High** | Yes |
| SIGMA Puentes | Argentina | 2007 | Risk + condition | CRF + ICF | — | IRF × ICF matrix | Medium | Docs public |
| MP-2020 / MP-2024 | Costa Rica / MOPT + LanammeUCR | 2020/24 | Element | 6 levels | AASHTO-aligned | Element aggregation | **High** | Yes |

---

## Part 6 — DACL10k gap analysis against framework inputs

| DACL10k class | Computable from mask + 3D? | Measurement needed | Frameworks using equivalent category (+ thresholds) | Gap / caveat |
|---|---|---|---|---|
| **Crack** | Area, length, spatial density directly from mask; **width requires calibrated GSD** | Pixel-to-mm conversion, skeleton + orthogonal intercept | AASHTO MBEI 0.30/1.27 mm (RC), 0.10/0.23 mm (PSC); EN 1992-1-1 0.4/0.3/0.2 mm; ACI 224R 0.41/0.30/0.18/0.15/0.10 mm; RI-EBW-PRÜF 0.2/0.3/0.4 mm; KISTEC 0.1/0.3/0.5/1.0 mm; Austroads 0.3/0.5/1.0 mm; NBR 9452 0.3 mm (NBR 6118) | Width measurement needs GSD ≤ 0.1 mm/px for 0.3 mm bins and ≤ 0.05 mm/px for 0.1 mm bins — only achievable with 45 MP FF (P1) at ~1 m standoff or zoom payload (H20T). Orientation/location (bending vs shear) is critical to severity and is model-inferrable but error-prone. |
| **ACrack** (map / alligator) | Pattern recognition and extent direct | Spatial extent | MBEI 1130 "pattern cracking" moderate (CS2) / heavy (CS3); IQOA fissuration diffuse; RI-EBW-PRÜF Netzrisse; JTG/T H21 map cracking code; KISTEC; NBR 9452 | Discrimination from systematic structural cracks is model-dependent. ASR/DEF origin cannot be inferred from image alone. |
| **Spalling** | Area direct; **depth requires stereo / SfM / LiDAR** | Depth sensor | AASHTO MBEI 1080 — **1 in depth, 6 in extent**; Austroads 25 mm depth; NBR 9452 desagregação; ACI 201.1R | Depth is often the CS2→CS3 discriminator; stereo photogrammetry gives depth to ~1 mm. |
| **Efflorescence** | Area, colour distinguishable (white vs rust-tinged) | RGB channel heuristics | MBEI 1120 (CS2 leaching only; CS3 heavy + rust); BIRM; all major systems | Active vs passive not inferrable from single image; repeat surveys help. |
| **ExposedRebars** | Exposure fact, approximate length direct | Section loss requires calipers | MBEI 1090 CS2 (no section loss) vs CS3 (measurable) vs CS4 (review) | Section-loss % is not image-observable and is the CS2→CS3→CS4 discriminator. |
| **Rust** (staining) | Area direct | RGB | MBEI 1120; BIRM; all major systems | Distinguishes active corrosion precursor; can't quantify electrochemical stage without half-cell (RILEM TC 154 thresholds: −350 mV CSE). |
| **Rockpocket** (honeycomb) | Area direct | — | MBEI none (treated as "other" construction defect) or equivalent to 1080; BIRM honeycomb; JTG/T H21; NBR 9452 "nicho"; RI-EBW-PRÜF Kiesnester | Usually a pre-service defect; doesn't deteriorate actively; low G/low D. |
| **Cavity** | Area and depth via 3D | Depth | Treated as severe spall (MBEI 1080 CS3/CS4) or abrasion (1190) | Depth > cover → rebar exposed → triggers 1090. |
| **Hollowareas** (delamination) | **Not reliably detectable from RGB alone** | **Sounding / IR thermography / GPR** | MBEI delamination within 1080 CS2; BIRM; all major systems expect sounding | **Critical limitation**. DACL10k Hollowareas labels come from chalk-marked areas previously identified by sounding; drone DL can localise marks but not detect underlying delamination optically. Thesis must explicitly flag this as NDT-required. |
| **Wetspot** | Area direct | — | MBEI contributes to 1120; BIRM leaching/moisture; all systems | Indicator of active moisture path; precursor to Crack/Efflorescence severity upgrades. |
| **Weathering** | Area direct | — | MBEI 1190 Abrasion/Wear (CS2 aggregate exposed / CS3 loose); BIRM weathering/scaling; ACI 201.1R | Usually cosmetic CS1–CS2; low weight. |
| **Restformwork** | Area direct | — | No MBEI defect; construction anomaly | Exclude from CS computation unless it creates leaching pathway. |
| **Graffiti** | Area direct | — | No structural defect in any framework | Track as cosmetic maintenance; exclude from structural damage score. |

**Aggregate drone-pipeline feasibility summary**: 10 of 13 DACL10k classes are image-native and directly feed at least one framework's quantitative pipeline. **Hollowareas is the principal blocker** for full compliance with any of the standards (it is image-localisable only via proxy cues — texture/staining/chalk outlines — and requires acoustic/IR confirmation). Crack width mm-precision is a secondary blocker solvable by GSD planning (≤ 0.1 mm/px). Rebar section loss (triggers MBEI CS3→CS4, RI-EBW-PRÜF S/D upgrades, Italian G = 5 PS flag) is a third blocker requiring physical access. Chloride content, cover depth, carbonation depth, settlement, scour, and seismic/hydraulic hazard parameters are outside any drone pipeline's envelope.

---

## Part 7 — Academic literature aligned with the thesis (2018–2026)

### Papers linking CV/DL output to official standards

**Çelik, Hoskere & Kessler (2025)**, *CACIE* 39, "From pixels to rating — a semiautomated system linking multi-damage segmentation and condition rating in concrete bridge inspections" — the landmark reference, explicitly framing the gap the thesis addresses. UAV imagery + U-Net/transformer segmentation + SAM foundation-model context, targeting the German Zustandsnote per RI-EBW-PRÜF. **This is the single paper closest to the thesis topic.**

**Cardellicchio, Ruggieri, Nettis, Renò, Uva group (Politecnico di Bari + STIIMA-CNR, 2023–2025)** on the **Italian Linee Guida**: "Physical interpretation of machine learning-based recognition of defects for the risk management of existing bridge heritage" (*Eng. Fail. Anal.* 149, 2023); YOLOv5 for RC bridge defects (SPIE 2023); "Using Attention for Improving Defect Detection in Existing RC Bridges" (*IEEE Access* 13, 2025); Di Mucci et al. (ECCOMAS 2025) on CV-based corrosion classification for seismic assessment of RC piers. These constitute the most direct prior work on automated G/K scoring feeding the Italian CdA.

**Zhang et al. (2024)**, *CACIE* 39(10): 1431–1451, "Deep learning-based automatic classification of three-level surface information in bridge inspection" — hierarchical model structure/component/defect with 96%/92%/81% accuracy.

**Momtaz, Li, Harris & Lattanzi (2023)**, *J. Infrastructure Intelligence and Resilience*, multi-modal deep fusion for bridge condition assessment (text + image). **Wang et al. (2023) BI-FusionNet** in *Eng. Appl. AI* fuses numerical + textual with SENet. **NBI condition-rating prediction** literature using LSTM/GRU on historical NBI data (Liu & Zhang 2020; Fiorillo & Nassif 2020; *Infrastructures* 9(12):221, 2024 — BHI R² > 0.84) is purely tabular and complements image work.

### Foundational CV pipelines

**Cha, Choi & Büyüköztürk (2017)**, *CACIE*, "Deep learning-based crack damage detection using CNNs" — the foundational CNN crack paper. **Cha et al. (2018)**, *CACIE* 33(9): 731–747, Faster R-CNN on 5 damage types, mAP 87.8%. **Dorafshan, Thomas & Maguire (2018)** *Constr. Build. Mater.* 186: 1031–1045, benchmark of deep nets vs edge detectors on SDNET2018 (UAV imagery; minimum reliable width ≈ 0.2 mm).

### Datasets

**SDNET2018** (Dorafshan et al., *Data in Brief* 21, 2018) — 56,000+ bridge deck/wall/pavement images, cracks 0.06–25 mm. **CODEBRIM** (Mundt et al., CVPR 2019) — multi-label concrete bridge defects. **S2DS** (Benz & Rodehorst, DAGM 2022) — 743 images, segmentation. **dacl10k** (Flotzinger, Rösch & Braml, WACV 2024, arXiv 2309.00460) — 9,920 images, 13 damage + 6 component classes, explicitly aligned with RI-EBW-PRÜF. **dacl1k** (Flotzinger et al., *Eng. Appl. AI* 137, 2024) — 1,474 diverse real-world images demonstrating domain gap between academic benchmarks and inspection reality. **OmniCrack30k** (Benz & Rodehorst, CVPRW 2024) — largest crack segmentation benchmark. **HRCDS** (2024 Mendeley) — high-resolution pixel-wise crack/rebar/corrosion/spall.

### Drone GSD and crack-width measurement

**Kim et al. (2017, *Sensors*)** — UAV + ultrasonic rangefinder measures 0.1 mm cracks with ≤ 7.3% length error. **Zollini et al. (2020 *Remote Sens.*; 2022 *ISPRS Archives*)** — explicit UAV flight planning for GSD = 1 mm → minimum detectable width ≈ 3 mm; 3 × GSD heuristic. **Peng et al. (2020 *Adv. Civ. Eng.*; 2021 *Constr. Build. Mater.*)** — UAV + 42 MP DSLR + laser rangefinder achieves > 90% crack-width accuracy. **Chen et al. (2022 *Autom. Constr.*)** — stereo vision with two focal lengths quantifies cracks < 0.2 mm. **Yoon et al. (2023 *Sustainability*)** — VDSR super-resolution recovers accuracy beyond 3 m standoff.

**Rule-of-thumb for the thesis**: to measure a *w* mm crack to ±*e* mm precision, target GSD ≤ min(*w*/5, *e*/2). ±0.1 mm on 0.3 mm cracks needs ~0.05 mm/px (20 px/mm); achievable with DJI M350 + Zenmuse P1 (45 MP FF, 35 mm lens) at ~1 m or 50 mm at ~0.9 m. ±0.2 mm on 0.3–0.5 mm cracks needs ~0.1 mm/px; achievable with M350 + P1 at ~1 m or Mavic 3E zoom at ~1.5 m.

### 3D integration

**Benz & Rodehorst (2024, arXiv 2401.03298)** — Multi-View 3D Instance Segmentation of Structural Anomalies: nnU-Net / DetectionHMA / TopoCrack → point colorisation → instance clustering. **Gao et al. (2024, *Autom. Constr.* 157)** — damage volumetric assessment and digital twin synchronization from LiDAR. **Mafipour et al. (2023, *Autom. Constr.* 156)** — automated geometric digital twinning of bridges from point clouds. **Shi, Kim & Narazaki (2024)** — synthetic 3D point cloud datasets for vision-based bridge condition assessment.

### Gap-analysis / review papers

**Spencer, Hoskere & Narazaki (2019, *Engineering*)** — seminal review of computer-vision SHM. **Koch et al. (2015, *AEI*)** — review on CV-based defect detection. **Zhang & Yuen (2022, *Adv. Mech. Eng.*)** — review of AI-based bridge damage detection. **Iacovino, Turksezer, Giordano & Limongelli (2022, EUROSTRUCT LNCE 200)** "A Survey of Bridge Condition Rating Systems" — **the best cross-national comparative reference** after the present report. **Methodologies for Remote Bridge Inspection Review** (PMC12473642, 2025, IABSE TG 5.9). **"AI-based bridge maintenance management: a comprehensive review"** (*AI Review*, 2025).

### Emerging (2024–2026)

**Darji, Liao & Liao (2025, arXiv 2507.14107)** — LLMs (GPT-4, Claude 3.5) for NDE contour-map interpretation. MDPI *Big Data and Cognitive Computing* 10(1):3 (Dec 2025) "DL-VLM for Bridge Health Diagnosis". **CrackFormer** (Liu et al., ICCV 2021), **CrackMamba-Net** (*Autom. Constr.* 2026).

---

## Part 8 — Frameworks incompatible with image-only inspection

The following frameworks are **effectively incompatible** with a purely image-based pipeline and require structural engineering inputs beyond drone reach, and must be explicitly scoped out or hybridised with NDT in the thesis:

**Swiss SIA 269/2 Überprüfung** produces an Erfüllungsgrad n from updated resistance and action models requiring cores (compressive strength), cover meters, reinforcement sampling, and FEM; the condition-class data entry into KUBA can be automated, but n itself cannot. **Eurocode prEN 1990-2 and ISO 13822** reliability assessments cannot be produced from imagery. **AASHTO MBE load rating** and **UK DMRB CS 454** produce a structural Rating Factor; imagery supports but does not replace the rating. **Italian Linee Guida Level 4** (Valutazione accurata per NTC 2018) is an analytical step; **Levels 0–2 (Difettosità → Vulnerabilità → CdA-Strutturale)** are compatible, but CdA-Sismica/Idraulica/Frane require non-visual hazard/exposure data (seismic ag, TGM, hydraulic regime, landslide maps). **fib Model Code 2020 Part VII** partial-factor verdict is structural. **All frameworks' delamination categories** (MBEI 1080 CS2, RI-EBW-PRÜF Hohlstellen, Austroads delamination, KISTEC hollow sounding, JTG/T H21 void codes) require sounding/IR/GPR confirmation that RGB cannot provide — an unavoidable hybrid step regardless of the chosen target framework.

---

## Part 9 — Recommendations for the thesis pipeline

**Primary recommendation: target AASHTO MBEI CS1–CS4 with a parallel Italian Linee Guida Livello di Difettosità output.** MBEI offers (i) the clearest and most-cited quantitative thresholds in the world (1 in spall depth, 6 in extent, 0.30/1.27 mm RC cracks, 0.10/0.23 mm PSC cracks), (ii) a defensible international standard that reviewers outside Italy will recognise instantly, (iii) direct consumption by AASHTOWare BHI with WF = 1.00 / 0.67 / 0.33 / 0.00, (iv) a clean overlap rule (VDOT 2016/2026: worst defect controls per region) that fuses DACL10k co-occurring classes deterministically. The Italian Linee Guida provides the operational context if the thesis is in Italy and integrates with AINOP, and its G × K₁ × K₂ sheet is essentially a generalisation of the MBEI quantity-in-state calculation. **Producing both outputs from the same DACL10k masks is straightforward** and gives the thesis cross-framework robustness.

**Secondary outputs to compute for robustness checks**: a **German-style Zustandsnote** (continuous 1.0–4.0 via Haardt aggregation on S/V/D triples), which is the target framework DACL10k was engineered for and therefore the most internally consistent sanity check; and a **UK BCI_av / BCI_crit** pair using the Severity × Extent matrix — the most image-native scoring scheme of all, converting directly from segmentation mask area percentages to a 0–100 index.

**Concrete implementation steps per inspection**:
1. Capture drone imagery at **GSD ≤ 0.1 mm/px** (M350 + P1 at ≤ 1 m standoff with 35 mm lens) for crack-width discrimination at the MBEI 0.30/1.27 mm bins; drop to ≤ 0.05 mm/px for PSC-grade 0.10/0.23 mm bins.
2. Segment each frame into DACL10k classes (13 damage + 6 object); project masks onto the SfM/NeRF/Gaussian-Splatting 3D model of the bridge; attribute masks to BIM-level structural elements.
3. For each element, compute: total area; area in each defect class; for cracks, width distribution via skeletonisation + orthogonal intercept with local pixel-to-mm Jacobian; for spalls, depth via stereo/SfM where available.
4. Apply the VDOT-style worst-defect-controls overlap rule to produce per-element CS1–CS4 quantities; in parallel, fill the Italian defect sheets with K₁ (extent 0.2/0.5/1.0) and K₂ (intensity 0.2/0.5/1.0) from area percentages and width bins.
5. Aggregate per-element CS quantities to the bridge-level BHI; aggregate Italian defect sheets to Livello di Difettosità → Vulnerabilità → CdA-Strutturale; optionally compute Zustandsnote and BCI for cross-validation.
6. **Explicitly carry uncertainty flags** for: Hollowareas (needs sounding), rebar section loss (needs calipers), cover depth / chloride / carbonation (needs cores / GPR / half-cell), scour / settlement / seismic / hydraulic (outside imagery envelope). Report these as "NDT required" rather than imputing values.
7. Treat **DACL10k Graffiti and Restformwork** as cosmetic — exclude from structural CS computation — and treat **Weathering and Wetspot** as sub-indicators that can upgrade adjacent Crack/Efflorescence severity rather than as standalone rating drivers.

**Defence framing for civil-engineering reviewers**: the thesis does not claim to replace inspectors or NDT; it claims to automate the **quantification and classification of image-observable surface pathologies into a standards-compliant CS/Difettosità vector**, with explicit NDT handoff for non-visual defects. This is exactly the framing of Çelik, Hoskere & Kessler (2025) and of PIARC 2022R20EN, both of which position UAV-DL as the surface-defect layer within an integrated BMS. The automation compatibility matrix in Part 5 and the DACL10k gap analysis in Part 6 are the core defensive artefacts — they document what the pipeline **can** and **cannot** do against every framework surveyed, which is the standard of rigour civil-engineering reviewers expect.

**Flag for the thesis**: the "fib Bulletin 243" reference in the project brief does not match the current fib catalogue; the most likely intended documents are fib Bulletin 80 (2016, Partial factor methods for existing concrete structures), fib Bulletin 111 (2024, Modelling structural performance of existing concrete structures), or fib Model Code 2020 Part VII (Assessment). Chile's "SIAPUC" could not be verified in any MOP document and should be treated as non-existent unless a primary source is found. Exact mm thresholds in MLIT (Japan), JTG/T H21 (China), KISTEC (Korea) per-member tables, and IRC SP:35 per-class bands are not fully verifiable in English — cite Jeong et al. (2018, *J. Struct. Integrity Maintenance* 3(2):126–135), Cho et al. (2023, *IJCSM* 17), Huang et al. (2022, *Adv. Civ. Eng.*), and Iacovino et al. (2022, EUROSTRUCT LNCE 200) as the canonical English secondary sources.