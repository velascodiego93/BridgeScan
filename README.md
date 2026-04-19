# BridgeScan — Automated Damage Detection & 3D Assessment

Master's thesis project in Applied Data Science.

## Objective

Development of an end-to-end system for automated structural inspection of concrete bridges using drone video. The system detects and semantically segments surface damage (cracks, spalling, exposed rebars, etc.), quantifies each damage type from the resulting masks, constructs a 3D model of the structure using photogrammetry or similar techniques, and projects the 2D masks onto the 3D model. A global damage level is then calculated using existing structural inspection standards, enabling near-real-time assessment without requiring human inspectors on site.

## Status

Work in progress. Currently in the problem scoping and state of the art survey phase.

## Prior work

A proof of concept was developed as part of a prior course, training and evaluating three semantic segmentation models (UNet, DeepLabV3+, SegFormer-B0) on the DACL10k dataset. The thesis builds directly on those results. See `poc_2024_readme.md` for details.

## Repository structure

_To be defined as the project evolves._
