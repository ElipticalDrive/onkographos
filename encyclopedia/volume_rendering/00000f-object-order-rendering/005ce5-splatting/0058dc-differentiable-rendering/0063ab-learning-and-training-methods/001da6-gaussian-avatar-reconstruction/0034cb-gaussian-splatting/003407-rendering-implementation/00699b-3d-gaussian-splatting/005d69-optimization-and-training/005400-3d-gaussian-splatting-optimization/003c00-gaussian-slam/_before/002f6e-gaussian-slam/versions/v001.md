# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that utilizes 3D Gaussian Splatting as its geometric representation, distinguished by its sub-map fusion strategy [S1, unofficial]. Unlike monocular online optimizers like MonoGS, it manages map consistency through the merging of localized sub-maps, enabling robust tracking and mapping in larger environments [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Monocular RGB video stream (implied by context of SLAM variants) or RGB-D data.
- **Outputs:** Camera poses, 3D Gaussian map (positions, covariances, opacities, spherical harmonics coefficients).

**Data Structures:**
- **Sub-map:** A localized set of 3D Gaussians representing a specific region of the scene.
- **Keyframe:** A selected frame used for tracking and map initialization/optimization.
- **Gaussian Set:** Inherits the layout from [Gaussian Data Structure](6238-gaussian-data-structure.md), storing per-Gaussian attributes: mean position $\mu$, covariance $\Sigma$ (or rotation $R$ and scale $S$), opacity $\alpha$, and color coefficients $c_{lm}$.

**Steps:**
1. **Initialization:** Initialize the first keyframe and its associated sub-map with a small set of Gaussians derived from initial depth estimates or feature points.
2. **Tracking:** For each new frame, estimate camera pose relative to the current sub-map using photometric or geometric constraints. [G] Standard ICP or bundle adjustment steps are used to refine pose.
3. **Keyframe Selection:** Select new keyframes based on motion or information gain to ensure adequate coverage.
4. **Sub-map Update:** Optimize the Gaussians within the active sub-map using differentiable rendering losses (photometric and depth) to minimize reprojection error.
5. **Sub-map Fusion:** When the camera moves sufficiently far or a new region is observed, merge the current sub-map with existing maps. This involves aligning overlapping Gaussians and resolving conflicts (e.g., via weighted averaging or pruning) to maintain a consistent global map.
6. **Loop Closure (Optional):** Detect loops by comparing current sub-map features with past keyframes. If a loop is detected, re-anchor Gaussians and optimize the global map to correct drift.

**Formulas:**
- Rendering loss: $L = \lambda_{rgb} L_{rgb} + \lambda_{depth} L_{depth}$.
- Sub-map fusion: Gaussians from overlapping regions are merged based on spatial proximity and confidence.

**Parameters:**
- Keyframe interval: Not given in the sources.
- Fusion threshold: Not given in the sources.

**Edge Cases:**
- **Drift:** Accumulated error in pose estimation can lead to misalignment in sub-map fusion.
- **Dynamic Objects:** Static assumption may cause artifacts; [G] Dynamic object removal is not explicitly detailed in the core method but is a known challenge in Gaussian SLAM.

## Subtopics

### MonoGS
A monocular RGB-only SLAM method that performs online optimization using Gaussian Splatting [S1, unofficial]. It differs from Gaussian-SLAM by focusing on real-time, incremental densification without explicit sub-map fusion, relying instead on continuous optimization of a single global map [S1, unofficial]. Choose MonoGS for resource-constrained monocular setups where simplicity of a single map is preferred over the modularity of sub-maps.

## Operating characteristics
- **Time Complexity:** Depends on the number of Gaussians and keyframes. Sub-map fusion adds overhead compared to single-map approaches.
- **Memory Complexity:** Linear with the number of Gaussians. Sub-map management may increase memory usage due to temporary storage of multiple map segments.
- **Strengths:** Robustness to drift through sub-map fusion; modularity allows for localized updates.
- **Limitations:** Complexity of implementation due to fusion logic; potential for inconsistencies during fusion.
- **When to use:** Large-scale environments where single-map optimization becomes intractable or prone to drift.
- **When not to use:** Real-time applications with strict latency constraints where fusion overhead is prohibitive.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S1, unofficial]. It represents a shift from NeRF-based SLAM methods, which struggled with local editing and real-time performance, to 3D Gaussian Splatting, which offers faster rendering and easier local manipulation [S1, unofficial]. The method leverages the efficiency of Gaussian Splatting to enable real-time SLAM with high-quality visual fidelity.

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
