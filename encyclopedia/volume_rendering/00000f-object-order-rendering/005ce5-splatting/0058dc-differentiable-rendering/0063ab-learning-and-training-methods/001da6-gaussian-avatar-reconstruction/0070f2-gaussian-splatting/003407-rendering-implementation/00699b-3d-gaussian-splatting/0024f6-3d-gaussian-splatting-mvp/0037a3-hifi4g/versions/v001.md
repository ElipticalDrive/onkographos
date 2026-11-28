# HiFi4G

Also called: HiFi4G

## Overview
HiFi4G is a 3D Gaussian Splatting variant designed for high-fidelity reconstruction of dynamic humans from monocular video. It combines standard 3D Gaussian Splatting with a dual-graph mechanism to enable non-rigid tracking, ensuring spatial-temporal consistency while maintaining a compact representation. It distinguishes itself from static 3DGS by explicitly modeling temporal deformation and from other dynamic methods by its specific dual-graph topology for tracking.

## How it works
HiFi4G builds on [3D Gaussian Splatting Rendering (Software)](3410-3d-gaussian-splatting-rendering-software.md) by adding a temporal tracking layer.

**Inputs:**
- Monocular video sequence (typically 50–100 frames) [S1][S2, unofficial].
- Camera poses for each frame [S1][S2, unofficial].

**Outputs:**
- A compact set of 3D Gaussians representing the canonical space and their temporal deformations [S1][S2, unofficial].

**Data Structures:**
- **Tri-plane Representation:** Factorizes the canonical space to reconstruct the person and scene [S1][S2, unofficial].
- **Dual-Graph Mechanism:** A graph structure used for non-rigid tracking of the Gaussians across frames [S1][S2, unofficial].

**Steps:**
1. Initialize 3D Gaussians in a canonical space using the tri-plane representation [S1][S2, unofficial].
2. Construct a dual-graph mechanism to model the non-rigid deformation of the Gaussians [S1][S2, unofficial].
3. Optimize the Gaussian parameters and graph connections jointly to track the subject across the video frames [S1][S2, unofficial].
4. Render frames by transforming the canonical Gaussians according to the tracked deformation for each time step [S1][S2, unofficial].

**Parameters:**
- Video length: 50–100 frames [S1][S2, unofficial].
- Reconstruction time: ~30 minutes [S1][S2, unofficial].

**Edge Cases:**
- Monocular input limits depth accuracy, relying on the tri-plane and graph constraints for consistency [S1][S2, unofficial].

## Subtopics

## Operating characteristics
- **Speed:** Reconstruction takes approximately 30 minutes for 50–100 frames [S1][S2, unofficial].
- **Compactness:** Described as "more compact" than standard 3DGS due to the factorization and tracking mechanism [S1][S2, unofficial].
- **Consistency:** Successfully preserves spatial-temporal consistency [S1][S2, unofficial].
- **Hardware:** Consumer-level devices are sufficient for rendering [S1][S2, unofficial].

## Background
HiFi4G was introduced in reference [208] (cited in [S1][S2, unofficial]). It addresses the challenge of reconstructing dynamic humans from limited monocular input by leveraging the compactness of 3DGS and the tracking capabilities of a dual-graph structure. It is part of the broader family of 3D Gaussian Splatting advancements, alongside methods like GPS-Gaussian [S1][S2, unofficial] and GART [S1][S2, unofficial].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
[S2] PDF Recent advances in 3D Gaussian splatting - Springer, https://link.springer.com/content/pdf/10.1007/s41095-024-0436-y.pdf, fetched 2026-10-29 (unofficial)
