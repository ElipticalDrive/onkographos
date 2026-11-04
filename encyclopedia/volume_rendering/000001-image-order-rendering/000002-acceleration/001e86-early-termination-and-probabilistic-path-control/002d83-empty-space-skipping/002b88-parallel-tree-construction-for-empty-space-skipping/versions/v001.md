# Parallel Tree-Construction for Empty-Space Skipping
## Overview
Parallel tree-construction algorithms enable the run-time re-construction of acceleration data structures for empty-space skipping, primarily to accommodate dynamic changes in the transfer function [S1, unofficial]. This approach distinguishes itself from static acceleration structures by allowing the hierarchy to be rebuilt on the GPU at interactive frame rates, ensuring that the skipping logic remains consistent with the current visual parameters [S1, unofficial].
## Shared principles
The core mechanism involves constructing a bounding volume hierarchy (BVH) or similar tree structure in parallel on the GPU. The construction process typically adapts algorithms originally designed for triangle geometry to sparse volumetric data. This adaptation uses the boundaries of non-empty bricks (or voxels) as the bounding geometry for the nodes [S1, unofficial]. The parallel nature of the construction allows for the rapid regeneration of the tree, which is necessary when the transfer function changes, as this alters which regions are considered "empty" (transparent) versus "occupied" (opaque or semi-transparent) [S1, unofficial].
## Subtopics

### [LBVH Adaptation for Sparse Volumes](865-object-order-empty-space-skipping.md)
An adaptation of a linear bounding volume hierarchy (LBVH) construction algorithm. It uses boundaries of non-empty bricks as bounding geometry, allowing construction on the GPU at interactive frame rates for $1024^3$ volumes.

## Background
The technique emerged in recent years to address the limitations of static acceleration structures in dynamic volume rendering scenarios [S1, unofficial]. Key contributions include work by Zellmann et al. [ZHL19], who proposed the LBVH adaptation for sparse data, and subsequent works by ZSL18, ZML19, FW20, DK19, DK20, and ZSL21, which further explored parallel tree construction for this purpose [S1, unofficial]. The primary motivation is to support run-time re-construction of acceleration data structures due to changes in the transfer function [S1, unofficial].
## Sources
[S1] Skalierbares Webbasiertes Volume Rendering großer biomedizinischer Datensätze, https://doi.org/10.34726/hss.2023.106203, fetched 2026-10-26 (unofficial)
