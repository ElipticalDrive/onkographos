# 3DGS Initialization Policy

Also called: Initialization Policy, OTS Initialization

## Overview
The 3DGS Initialization Policy defines the standard procedure for seeding the 3D Gaussian primitives in 3D Gaussian Splatting (3DGS) and its variants, such as Order-Independent Splatting (OTS). It solves the problem of establishing a valid initial scene representation from sparse input data (typically SfM points) by distributing Gaussians in space with specific opacity and scale properties. It differs from sibling methods like [Volumetric Gaussian Splatting (AVIS)](1875-volumetric-gaussian-splatting-avis.md) or [Direct Gaussian Splatting](3555-direct-gaussian-splatting.md) by relying on a uniform opacity initialization derived from a power-law function of the total Gaussian count, rather than learned or geometrically constrained initial weights [S1].

## How it works
**Inputs:**
*   Sparse point cloud from Structure-from-Motion (SfM) or a unit cube bounding box (for synthetic scenes).
*   Total number of Gaussians to initialize, $N$.
*   Scene bounding box (typically a unit cube centered at the origin for synthetic data).

**Outputs:**
*   A set of $N$ 3D Gaussians, each defined by position $\mathbf{x}$, covariance $\Sigma$ (or scale/quaternion), opacity $\alpha$, and spherical harmonics coefficients.

**Data Structures:**
*   **Gaussian Set:** An array of $N$ elements. Each element stores:
    *   $\mathbf{x} \in \mathbb{R}^3$: Position.
    *   $\mathbf{s} \in \mathbb{R}^3$: Scale (log-space or linear).
    *   $\mathbf{q} \in \mathbb{R}^4$: Rotation (quaternion).
    *   $\alpha \in [0, 1]$: Opacity.
    *   $\mathbf{c} \in \mathbb{R}^k$: Color/SH coefficients.

**Steps:**
1.  **Position Initialization:**
    *   For synthetic scenes (e.g., Nerf-Synthetic): Sample $N$ positions uniformly at random within a unit cube centered at the scene origin [S1].
    *   For real-world scenes: Typically, positions are initialized from the SfM point cloud, often with jitter or random sampling if $N$ exceeds the point count, though the specific real-world initialization strategy is not detailed in the primary source beyond the synthetic case [S1].

2.  **Opacity Initialization:**
    *   Compute a single scalar value $V$ used as the initial opacity for all Gaussians. This value is derived from the total number of Gaussians $N$ using a power function [S1]:
        $$ V = 2 \cdot N^{\beta} $$
    *   The exponent $\beta$ depends on the rendering method:
        *   $\beta = 0.35$ for standard 3DGS [S1].
        *   $\beta = 0.55$ for OTS-based methods [S1].
    *   Set $\alpha_i = V$ for all $i \in \{1, \dots, N\}$. Note: In practice, opacity is often clamped to $[0, 1]$ or processed through a sigmoid during rendering, but the initialization value is set to this computed $V$ [S1].

3.  **Scale and Rotation Initialization:**
    *   The sources do not specify the exact initial scale and rotation values for the general policy, only that positions and opacity are initialized as described [S1]. [G] Standard 3DGS implementations typically initialize scales to a small random value or derived from the average distance to nearest neighbors, and rotations to identity. This detail is not provided in the cited sources.

4.  **Color Initialization:**
    *   The sources do not specify the initial color/SH coefficients [S1]. [G] Typically initialized to white or random colors.

**Parameters:**
*   $N$: Total number of Gaussians. Typical values mentioned are 4k, 12k, 36k, 100k, 500k, 1m, and 5m [S1].
*   $\beta$: Exponent for opacity scaling. 0.35 for 3DGS, 0.55 for OTS [S1].

**Edge Cases and Pitfalls:**
*   **Opacity Clamping:** The computed value $V = 2 \cdot N^{\beta}$ may exceed 1.0 for large $N$. The implementation must handle this, either by clamping $\alpha$ to 1.0 or by relying on the rendering pipeline's sigmoid activation [S1].
*   **Scene Bounding:** For real-world scenes, ensuring Gaussians are initialized within the relevant scene volume is critical to avoid empty space initialization [S1].

## Subtopics

### Geometric Correction via Neural Gaussians
A strategy to maximize overlap between cells and emitted neural Gaussians by dynamically aligning cell geometry with Gaussian properties [S2]. It involves: 1. Dynamically defining the cell's quaternion as a weighted average of the quaternions associated with its emitted Gaussians [S2]. 2. Periodically updating cell centers using $\mathbf{x}_n = \frac{\sum_{k=1}^K \boldsymbol{\mu}_{n,k}}{K}$, where $\mathbf{x}_n$ is the cell center, $K$ is the number of Gaussians in the cell, and $\boldsymbol{\mu}_{n,k}$ is the mean of the k-th Gaussian in cell n [S2]. 3. Learning the three-dimensional covariances and positions of first-layer cells in accordance with the updates to the second-layer Gaussians, leveraging the differentiability of cell scales [S2].

### Cell Position Reset
Following adaptive density control, new cells are grown at Gaussians with high view-space positional gradients. To decouple existing cells from newly introduced ones, the offsets of the significant Gaussians are reset to zero after cell growth, i.e., their positions are reinitialized to the centers of the corresponding cells [S2].

## Operating characteristics
*   **Time Complexity:** Initialization is $O(N)$ for sampling positions and setting opacity [S1].
*   **Memory Complexity:** $O(N)$ to store the initial Gaussian parameters [S1].
*   **Performance:** The policy is designed to provide a stable starting point for optimization. The use of a power-law opacity initialization helps balance the contribution of Gaussians during the early stages of training [S1].
*   **Strengths:** Simple to implement; provides a consistent baseline for comparing different splatting techniques (e.g., 3DGS vs. OTS) by controlling the initial opacity scaling [S1].
*   **Limitations:** Does not account for scene geometry during initialization (except for position sampling); relies on subsequent densification and pruning to refine the representation [S1].

## Background
The initialization policy was introduced in the context of evaluating whether 3D Gaussian Splatting requires accurate volumetric rendering, specifically comparing standard 3DGS with Order-Independent Splatting (OTS) [S1]. The policy was adapted from the original 3DGS codebase's training procedure for Nerf-Synthetic scenes [S1]. It has been used in experiments involving real-world scenes with varying numbers of Gaussians (100k to 5m) to evaluate the impact of densification and initialization strategies [S1].

## Sources
[S1] Does 3D Gaussian Splatting Need Accurate Volumetric Rendering?, https://doi.org/10.1111/cgf.70032, fetched 2026-10-26
[S2] Duplex-GS: Proxy-Guided Weighted Blending for Real-Time Order-Independent Gaussian Splatting, https://doi.org/10.1109/tcsvt.2026.3666188, fetched 2026-10-30
