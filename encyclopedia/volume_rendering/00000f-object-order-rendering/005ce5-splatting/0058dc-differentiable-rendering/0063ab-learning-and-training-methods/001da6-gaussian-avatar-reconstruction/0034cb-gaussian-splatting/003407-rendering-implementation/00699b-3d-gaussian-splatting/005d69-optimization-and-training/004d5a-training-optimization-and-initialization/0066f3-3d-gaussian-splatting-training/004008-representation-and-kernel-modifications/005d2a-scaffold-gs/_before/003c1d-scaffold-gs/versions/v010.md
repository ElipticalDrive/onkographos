# Scaffold-GS

Also called: Scaffold-Gaussian Splatting

## Overview
Scaffold-GS is a representation modification for 3D Gaussian Splatting (3DGS) that replaces the direct storage of Gaussian attributes with learnable features on a voxel grid, decoded via lightweight neural networks [S1]. It distinguishes itself from standard 3DGS and siblings like Negative Gaussian Splatting by decoupling attribute storage from direct parameters, enabling more compact and structured representation of 3D shapes and view-dependent appearance [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Sparse 3D point cloud from Structure-from-Motion (SfM); multi-view images.
*   **Output:** Optimized voxel grid features and network weights; rendered images.

**Data Structures:**
1.  **Voxel Grid:** A 3D grid where each voxel point holds learnable feature vectors. The grid resolution is a hyperparameter.
2.  **Lightweight Neural Networks:** Small MLPs or linear layers that map interpolated voxel features to Gaussian attributes (mean, covariance, opacity, color).

**Method Steps:**
1.  **Initialization:** Initialize a voxel grid covering the scene extent. Attach learnable feature vectors to each voxel point [S1].
2.  **Feature Interpolation:** For each Gaussian primitive, determine its position within the voxel grid. Interpolate features from surrounding voxels to obtain a dense feature vector for that Gaussian [S1].
3.  **Attribute Decoding:** Pass the interpolated features through lightweight neural networks to predict all Gaussian attributes: mean $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and color $\mathbf{c}$ [S1].
4.  **Rendering:** Render the Gaussians using the standard differentiable rasterization pipeline of 3DGS [S1].
5.  **Optimization:** Minimize the difference between rendered and ground-truth images using gradient descent, updating voxel features and network weights [S1].

**Formulas:**
Let $F_v$ be the feature vector of voxel $v$. For a Gaussian at position $\mathbf{p}$, the interpolated feature $F_{interp}$ is computed via trilinear interpolation of surrounding voxel features. The attributes $\Theta$ are predicted by a network $N$:
$$ \Theta = N(F_{interp}) $$
where $\Theta = \{\boldsymbol{\mu}, \boldsymbol{\Sigma}, \alpha, \mathbf{c}\}$ [S1].

**Parameters:**
*   Voxel grid resolution: Not given in the sources.
*   Network architecture: Lightweight neural networks (specifics not given in the sources) [S1].

**Edge Cases and Pitfalls:**
*   **Voxel Resolution:** Too coarse leads to loss of detail; too fine increases memory and computation [S1].
*   **Interpolation Artifacts:** Discontinuities at voxel boundaries if not handled properly [G].

## Subtopics

### [Scaffold-GS Integration](12513-scaffold-gs-integration.md)
Applies DART and AGE modules to the Scaffold-GS backbone for improved initialization and densification.

## Operating characteristics
*   **Time Complexity:** Depends on voxel grid size and network inference speed. Generally slower per-frame rendering than standard 3DGS due to network evaluation [S1].
*   **Memory Complexity:** Proportional to voxel grid size and feature dimensionality. Potentially more compact than storing millions of Gaussians directly if the scene is smooth [S1].
*   **Strengths:** Compact representation; structured learning; better generalization for smooth surfaces [S1].
*   **Limitations:** Requires careful tuning of voxel resolution; network inference overhead [S1].
*   **When to Use:** Scenes with smooth geometry and view-dependent effects where compactness is desired [S1].
*   **When Not to Use:** Highly detailed, non-smooth scenes requiring high-frequency detail [S1].

## Background
Scaffold-GS was proposed in "Recent advances in 3D Gaussian splatting" [S1]. It builds on the idea of using neural networks to predict Gaussian attributes, similar to VDGS, but uses a voxel grid scaffold for structured feature storage [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
