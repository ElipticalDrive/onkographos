# Gaussian-SLAM

Also called: Yugay et al. 2024 SLAM

## Overview
Gaussian-SLAM is a keyframe-based Simultaneous Localization and Mapping (SLAM) method that uses 3D Gaussian Splatting as its geometric representation [S1, unofficial]. It distinguishes itself from single-map online optimizers like MonoGS by managing map consistency through localized sub-map fusion, enabling robust tracking and mapping in larger environments [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Inputs:** Not given in the sources.
- **Outputs:** Camera poses and a 3D Gaussian map [S1, unofficial].

**Data Structures:**
- **Sub-map:** A localized set of 3D Gaussians representing a specific region of the scene [S1, unofficial].
- **Keyframe:** A selected frame used for tracking and map initialization or optimization [S1, unofficial].
- **Gaussian Set:** Inherits the layout from [Gaussian Data Structure](6238-gaussian-data-structure.md), storing per-Gaussian attributes including mean position, covariance (or rotation and scale), opacity, and color coefficients [S1, unofficial].

**Steps:**
1. **Initialization:** Initialize the first keyframe and its associated sub-map [S1, unofficial].
2. **Tracking:** Estimate camera pose relative to the current sub-map [G].
3. **Keyframe Selection:** Select new keyframes based on motion or information gain [G].
4. **Sub-map Update:** Optimize Gaussians within the active sub-map using differentiable rendering losses to minimize reprojection error [G].
5. **Sub-map Fusion:** Merge the current sub-map with existing maps when the camera moves sufficiently far or a new region is observed [S1, unofficial].
6. **Loop Closure (Optional):** Detect loops by comparing current sub-map features with past keyframes and re-anchor Gaussians to correct drift [G].

**Formulas:**
- Rendering loss: Not given in the sources.
- Sub-map fusion: Gaussians from overlapping regions are merged based on spatial proximity and confidence [S1, unofficial].

**Parameters:**
- Keyframe interval: Not given in the sources.
- Fusion threshold: Not given in the sources.

**Edge Cases:**
- **Drift:** Accumulated error in pose estimation can lead to misalignment during sub-map fusion [G].
- **Dynamic Objects:** The static scene assumption may cause artifacts; dynamic object handling is not explicitly detailed in the core method [G].

## Subtopics

### MonoGS
A monocular RGB-only SLAM method that performs online optimization using Gaussian Splatting [S1, unofficial]. It differs from Gaussian-SLAM by focusing on continuous optimization of a single global map without explicit sub-map fusion [S1, unofficial]. Choose MonoGS for resource-constrained monocular setups where simplicity of a single map is preferred over the modularity of sub-maps [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Not given in the sources.
- **Limitations:** Not given in the sources.
- **When to use:** Not given in the sources.
- **When not to use:** Not given in the sources.

## Background
Gaussian-SLAM was introduced by Yugay et al. in 2024 [S1, unofficial]. It represents a shift from NeRF-based SLAM methods, which struggled with local editing and real-time performance, to 3D Gaussian Splatting, which offers faster rendering and easier local manipulation [S1, unofficial]. The method leverages the efficiency of Gaussian Splatting to enable real-time SLAM with high-quality visual fidelity [S1, unofficial].

## Sources
[S1] Gaussian Splatting — All Variants Cheat Sheet, https://www.chengjianglong.com/ml/3D-Gaussian_Splatting_SOTA_Updated.html, fetched 2026-10-27 (unofficial)
