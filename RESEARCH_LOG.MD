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