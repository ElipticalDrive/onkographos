# Scaffold-GS

Also called: Scaffold-Gaussian Splatting

## Overview
Scaffold-GS addresses the limitations of standard 3D Gaussian Splatting (3DGS) by replacing the direct storage of Gaussian attributes with a structured, voxel-based representation. It initializes a voxel grid and attaches learnable features to each voxel point, determining all Gaussian attributes through interpolated features and lightweight neural networks. This approach distinguishes itself from standard 3DGS and its siblings like Negative Gaussian Splatting by decoupling the geometric scaffold from the primitive attributes, enabling more efficient attribute prediction and potentially better generalization in sparse regions.

## How it works
**Inputs:** Sparse Structure-from-Motion (SfM) point cloud; multi-view images.
**Outputs:** A set of 3D Gaussians defined by parameters derived from the voxel grid.

**Data Structures:**
1.  **Voxel Grid:** A 3D grid where each voxel stores learnable feature vectors. The resolution is determined by the bounding box of the SfM points and a chosen voxel size.
2.  **Gaussian Parameters:** Each Gaussian $i$ is defined by position $\mathbf{p}_i$, covariance $\mathbf{\Sigma}_i$, color $\mathbf{c}_i$, and opacity $\alpha_i$. In Scaffold-GS, these are not stored directly but computed.

**Steps:**
1.  **Voxel Initialization:** Initialize a voxel grid covering the scene extent. Attach learnable feature vectors $\mathbf{f}_v$ to each voxel $v$.
2.  **Gaussian Seeding:** For each Gaussian $i$, determine its position $\mathbf{p}_i$ (typically from SfM or densification). Identify the surrounding voxels $V_i$ that influence $\mathbf{p}_i$.
3.  **Feature Interpolation:** Interpolate the learnable features from the surrounding voxels $V_i$ to obtain a combined feature vector $\mathbf{F}_i$ for the Gaussian. This is typically done via trilinear interpolation or weighted averaging based on distance.
4.  **Attribute Prediction:** Pass $\mathbf{F}_i$ through lightweight neural networks (MLPs) to predict the Gaussian attributes:
    *   $\mathbf{c}_i = \text{MLP}_{color}(\mathbf{F}_i)$
    *   $\alpha_i = \text{MLP}_{opacity}(\mathbf{F}_i)$
    *   $\mathbf{\Sigma}_i$ may be predicted or derived from a base covariance scaled by features.
5.  **Rendering:** Render the scene using standard differentiable rasterization of 3D Gaussians [S1].
6.  **Optimization:** Update the voxel features $\mathbf{f}_v$ and the parameters of the lightweight neural networks using gradient descent to minimize the difference between rendered and ground-truth images.

**Formulas:**
*   Interpolated Feature: $\mathbf{F}_i = \sum_{v \in V_i} w_{i,v} \mathbf{f}_v$, where $w_{i,v}$ are interpolation weights.
*   Attribute Prediction: $\mathbf{A}_i = \text{MLP}(\mathbf{F}_i)$, where $\mathbf{A}_i$ contains $\mathbf{c}_i, \alpha_i, \dots$.

**Parameters:**
*   Voxel grid resolution: Not given in the sources.
*   Neural network architecture: Lightweight MLPs. Specific layer sizes not given in the sources.

**Edge Cases and Pitfalls:**
*   Sparse regions: If SfM points are sparse, voxel coverage may be incomplete. [G] Ensure voxel grid covers the entire scene bounding box, even if voxels are empty initially.
*   Precision: Interpolation weights must be normalized.

## Subtopics

### [Scaffold-GS Integration](12513-scaffold-gs-integration.md)
Applies DART and AGE modules to the Scaffold-GS backbone, targeting sparse initialization regions without modifying the rendering path.

## Operating characteristics
*   **Time Complexity:** Rendering time is similar to 3DGS, but attribute prediction adds a small overhead per Gaussian. Training time may be higher due to neural network forward passes.
*   **Memory Complexity:** Memory usage depends on the voxel grid size and the number of Gaussians. The voxel grid stores features, which may be more compact than storing full attributes for every Gaussian if the grid is sparse.
*   **Strengths:** Efficient attribute prediction; potential for better generalization; structured representation.
*   **Limitations:** Requires careful tuning of voxel grid resolution; neural network overhead.
*   **When to use:** Scenes with sparse SfM initialization or where attribute prediction benefits from structured features.

## Background
Scaffold-GS was proposed in [S1] as an advancement in 3D Gaussian Splatting. It builds on the idea of using neural networks to predict attributes, similar to VDGS, but introduces a voxel grid scaffold to organize the learnable features. This method has influenced subsequent approaches like Octree-GS.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
