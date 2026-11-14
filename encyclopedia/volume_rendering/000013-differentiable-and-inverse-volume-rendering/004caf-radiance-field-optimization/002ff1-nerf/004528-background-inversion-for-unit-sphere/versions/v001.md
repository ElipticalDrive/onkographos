# Background Inversion for Unit Sphere
Also called: Inverted Background Mapping, Sphere Inversion Background

## Overview
This method enables volume rendering of scenes with unbounded or complex backgrounds by mapping points outside a unit sphere into its interior, thereby eliminating the need for foreground masks during optimization [S1, unofficial]. It sets itself apart from sibling methods like [Inverted Sphere Parameterization](5573-inverted-sphere-parameterization.md) by explicitly using a separate density voxel grid and shallow MLP for the inverted background, rather than relying on a single continuous field or point-based representations [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** 3D coordinates $(x, y, z)$ of background points located outside the unit sphere ($r > 1$).
*   **Output:** Rendered color and density values for the background component, integrated with the foreground.

**Data Structures:**
1.  **Inverted Background Density Grid:** A voxel grid storing density values for the mapped background region.
2.  **Feature Grid:** A voxel grid storing per-voxel feature vectors (e.g., for color or appearance).
3.  **Shallow MLP:** A small multi-layer perceptron that maps the inverted coordinates and features to final output properties (color/density).

**Steps:**
1.  **Coordinate Inversion:** For each background point $\mathbf{x} = (x, y, z)$, calculate its distance from the origin $r = \sqrt{x^2 + y^2 + z^2}$. Compute the inverted coordinates $\mathbf{x}' = (x', y', z')$ using the inversion formula:
    $$
    x' = \frac{x}{r^2}, \quad y' = \frac{y}{r^2}, \quad z' = \frac{z}{r^2}
    $$
    This maps points from outside the unit sphere ($r > 1$) to inside the unit sphere ($r' < 1$) [S1, unofficial].
2.  **Grid Sampling:** Sample the density and feature values from the dedicated background voxel grids using the inverted coordinates $\mathbf{x}'$.
3.  **MLP Prediction:** Pass the sampled features and/or coordinates through the shallow MLP to refine the background representation [S1, unofficial].
4.  **Rendering Integration:** Combine the background rendering results with the foreground rendering (typically from the main NeRF++ field) during the final volume rendering pass [S1, unofficial].

**Parameters:**
*   **Unit Sphere Radius:** Fixed at 1.0.
*   **Grid Resolution:** Not given in the sources.
*   **MLP Depth/Width:** Described as "shallow," but specific layer counts or hidden dimensions are not given in the sources.

**Edge Cases and Pitfalls:**
*   **Points at Origin:** The formula involves division by $r^2$. Points exactly at the origin ($r=0$) are undefined. However, the method is specifically for background points *outside* the unit sphere, so $r \ge 1$, avoiding this singularity [S1, unofficial].
*   **Precision:** Floating-point precision in the inversion calculation may introduce minor artifacts for points very close to the unit sphere boundary ($r \approx 1$) [G].
*   **Boundary Continuity:** Discontinuities may arise at the boundary between the foreground field and the inverted background field if not properly blended [G].

## Subtopics

### Background Point Handling
A strategy to incorporate background into point-based radiance fields when standard Multi-View Stereo (MVS) fails on uniform color regions [S2, unofficial]. It represents the background with points extracted from a mesh, keeps track of human and background points separately, and combines them during final rendering [S2, unofficial]. This approach is distinct from the voxel-based inversion method as it relies on explicit point clouds rather than implicit voxel grids [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the voxel grid resolution and MLP size. Sampling from voxel grids is $O(1)$ per point. MLP inference is constant time relative to input size but depends on network depth.
*   **Memory Complexity:** Proportional to the volume of the unit sphere discretized by the voxel grid plus the MLP parameters.
*   **Strengths:** Eliminates the need for clean background masks or foreground segmentation, allowing for more robust training on real-world data with complex backgrounds [S1, unofficial].
*   **Limitations:** Requires additional memory for the separate background grid and MLP. The inversion mapping may distort the background geometry, potentially affecting view consistency if not regularized [G].
*   **When to Use:** When working with unbounded scenes where background segmentation is difficult or unreliable [S1, unofficial].
*   **When Not to Use:** When the background is simple and can be easily masked, or when computational resources are extremely limited [G].

## Background
This method builds on [NeRF++](6198-nerf.md) by addressing its limitation regarding background handling [S1, unofficial]. It was introduced in the context of voxel-based neural surface reconstruction to improve efficiency and accuracy without requiring masks [S1, unofficial]. The technique was also adapted in point-based radiance fields to handle background regions where MVS initialization fails [S2, unofficial].

## Sources
[S1] Voxurf: Voxel-based Efficient and Accurate Neural Surface Reconstruction, https://doi.org/10.48550/arxiv.2208.12697, fetched 2026-10-26 (unofficial)
[S2] Point-Based Radiance Fields for Controllable Human Motion Synthesis, https://doi.org/10.48550/arXiv.2310.03375, fetched 2026-10-29 (unofficial)
