# Mesh-based Sampling

Also called: Wan et al. Mesh-based Sampling

## Overview
Mesh-based Sampling localizes surface evaluation by estimating the volume density or Signed Distance Field (SDF) to define an inner and outer mesh boundary, using learned features attached to mesh vertices to estimate radiance [S1]. It sets itself apart from sibling methods by requiring only two samples per ray, significantly improving runtime efficiency while remaining applicable as a post-processing step to any existing NeRF models [S1].

## How it works
**Inputs:**
*   An implicit representation (volume density or SDF) defining the object geometry [S1].
*   Two meshes (inner and outer) derived from the implicit representation [S1].
*   Learned features attached to the vertices of these meshes [S1].
*   A viewing direction vector $\mathbf{d}$ for the ray [S1].

**Outputs:**
*   Estimated radiance for the ray [S1].

**Data Structures:**
*   **Inner Mesh:** A triangular mesh representing the inner boundary of the surface region. Vertices store learned feature vectors.
*   **Outer Mesh:** A triangular mesh representing the outer boundary of the surface region. Vertices store learned feature vectors.
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.

**Steps:**
1.  **Ray-Mesh Intersection:** Compute the intersection points of the viewing ray with both the inner mesh and the outer mesh. Let these intersection points be $\mathbf{p}_{in}$ and $\mathbf{p}_{out}$ respectively [S1].
2.  **Feature Interpolation:** At each intersection point ($\mathbf{p}_{in}$ and $\mathbf{p}_{out}$), interpolate the learned features from the surrounding mesh vertices. This yields feature vectors $\mathbf{f}_{in}$ and $\mathbf{f}_{out}$ [S1].
3.  **Radiance Estimation:** Feed the interpolated features ($\mathbf{f}_{in}, \mathbf{f}_{out}$) and the viewing direction $\mathbf{d}$ into a shallow convolutional network [S1].
4.  **Output:** The network outputs the estimated radiance [S1].

**Parameters:**
*   Number of samples per ray: 2 [S1].

**Edge Cases and Pitfalls:**
*   The method relies on the accuracy of the implicit representation to generate meaningful inner and outer meshes. If the SDF or density estimate is noisy, the mesh boundaries may be ill-defined [S1].
*   The shallow convolutional network must be trained to accept features from two distinct spatial locations and the viewing direction as input [S1].

## Subtopics

### Trajectory Sample Strategy
A query point sampling strategy where points are distributed along a displacement trajectory from a voxel grid to the surface closest point, designed to be more efficient than uniform or Gaussian sampling for mesh deformation tasks [S2, unofficial]. It maintains performance with fewer query points (75k vs 200k) and is more efficient for mesh deformation [S2, unofficial]. In unconstrained chair point cloud generation, Trajectory 75k achieved CD=56.12 and EMD=53.30, outperforming Uniform 75k (CD=56.88, EMD=54.93) and Gaussian 75k (CD=56.46, EMD=53.97) [S2, unofficial].

## Operating characteristics
*   **Efficiency:** Requires only two samples per ray, significantly improving runtime efficiency compared to methods that require dense sampling or multiple evaluations [S1].
*   **Generality:** Can be applied as a post-processing step to any existing NeRF models, making it a versatile addition to existing pipelines [S1].
*   **Performance:** Specific quantitative performance metrics (e.g., FPS, memory usage) for the Wan et al. method are not given in the sources [S1].

## Background
The method was proposed by Wan et al. [WRB* 23] as a way to localize the surface between an inner and outer mesh using volume density or SDF estimates from an implicit representation [S1]. It is cited in recent surveys on learning-based 3D representations as a technique that improves runtime efficiency by minimizing the number of samples per ray [S1].

## Sources
[S1] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
[S2] Neural Volumetric Mesh Generator, https://arxiv.org/abs/2210.03158v1, fetched 2026-10-27 (unofficial)
