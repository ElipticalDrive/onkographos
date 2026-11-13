# Screened Poisson Surface Reconstruction

Also called: Poisson Surface Reconstruction, SPSR, Surface Extraction via Screened Poisson Surface Reconstruction

## Overview
Screened Poisson Surface Reconstruction (SPSR) is an optimization-based method that fits a surface directly to a single oriented point cloud without prior training [S2]. It solves a Poisson equation to reconstruct a watertight manifold mesh, distinguishing itself from marching cubes variants (e.g., [Dual Contouring](778-dual-contouring.md), [Surface Nets](921-surface-nets.md)) by guaranteeing watertightness and avoiding the need to search for an ideal level-set threshold [S1, unofficial]. It builds on [Dual Contouring and Surface Nets](6319-dual-contouring-and-surface-nets.md) by offering a global smoothness prior rather than local cell-based extraction.

## How it works
**Inputs:** An oriented point cloud $\{(\mathbf{c}_i, \mathbf{n}_i)\}_{i=1}^N$, where $\mathbf{c}_i$ is the position and $\mathbf{n}_i$ is the unit normal. A scalar field parameter $\lambda$ (often called `pointWeight` or interpolation weight).

**Outputs:** A watertight triangle mesh representing the implicit surface $\phi(\mathbf{x}) = 0$.

**Data Structures:**
1.  **Point Cloud:** Array of $N$ structs, each containing $\mathbf{c}_i \in \mathbb{R}^3$ and $\mathbf{n}_i \in \mathbb{R}^3$.
2.  **Voxel Grid:** A 3D grid of resolution $R \times R \times R$ (typically $R \in [64, 256]$) covering the bounding box of the point cloud. Each voxel stores a scalar value $\phi$ and potentially a gradient.
3.  **Linear System:** A sparse matrix $A$ and vector $b$ representing the discretized screened Poisson equation.

**Steps:**
1.  **Define Bounding Box:** Compute the axis-aligned bounding box (AABB) of the input point cloud. Expand it slightly to ensure boundary conditions are handled correctly.
2.  **Discretize Domain:** Create a uniform voxel grid over the AABB with resolution $R$. Let $V$ be the number of voxels.
3.  **Construct Matrix $A$:** Initialize $A$ as an identity matrix scaled by $\lambda$. Add Laplacian terms for the Poisson operator. For each voxel $j$, the diagonal entry $A_{jj}$ is set to $\lambda + \sum_{k \in \text{neighbors}(j)} w_{jk}$, and off-diagonal entries $A_{jk}$ are set to $-w_{jk}$, where $w_{jk}$ are weights based on voxel adjacency (typically 1 for 6-connected neighbors). [G: Standard 7-point Laplacian stencil].
4.  **Construct Vector $b$:** Initialize $b$ to zero. For each point $i$ in the cloud:
    a. Find the voxel $j$ containing $\mathbf{c}_i$ (or the closest voxel).
    b. Compute the contribution of the normal $\mathbf{n}_i$ to the gradient of $\phi$. The target gradient is $\mathbf{n}_i$.
    c. Distribute the normal information to the voxel centers of the neighboring voxels using trilinear interpolation weights $w_{ij}$.
    d. Update $b_j \leftarrow b_j + \lambda \cdot (\mathbf{n}_i \cdot \nabla \phi_{target})$. [G: In practice, this often involves integrating the normal field over the voxel support. A common approximation is $b_j = \sum_i \lambda \cdot \mathbf{n}_i \cdot \mathbf{v}_{ij}$, where $\mathbf{v}_{ij}$ is the basis function derivative].
5.  **Solve System:** Solve the linear system $A \phi = b$ for the scalar field values $\phi$ at all voxel centers. Use a sparse linear solver (e.g., Conjugate Gradient or Multigrid).
6.  **Extract Mesh:** Apply the Marching Cubes algorithm to the voxel grid scalar field $\phi$ to extract the isosurface $\phi(\mathbf{x}) = 0$.
7.  **Post-processing:** (Optional) Transfer material attributes (albedo, roughness, metallic) from the original point cloud to the mesh vertices using nearest-neighbor or radial-basis-function interpolation [S1, unofficial].

**Parameters:**
*   `pointWeight` ($\lambda$): Controls the interpolation constraint. High values force the surface to pass through the points; low values allow smoothing. Typical values: $10-100$ for dense clouds, lower for sparse [S1, unofficial].
*   `resolution` ($R$): Grid resolution. Typical values: $128-256$ [G].

**Edge Cases and Pitfalls:**
*   **Under-sampling:** In regions where the partition-of-unity condition is not met (e.g., boundaries, rapid variation), Shepard normalization acts as a kernel regression (Nadaraya-Watson estimator) rather than an interpolant. Surfel centers may deviate from the true surface. SPSR with high `pointWeight` produces artifacts here [S1, unofficial].
*   **Sparse/Corrupted Input:** The method struggles with sparse or heavily corrupted inputs [S2].
*   **Mitigation:** Reduce `pointWeight` to relax the interpolation constraint, allowing the Poisson solve to regularize the surface [S1, unofficial].

## Subtopics

### [Poisson Surface Reconstruction for Mesh Export](6401-poisson-surface-reconstruction-for-mesh-export.md)
A variant used to export watertight, colored triangle meshes from 3D Gaussians. It differs from marching cubes by always producing a watertight manifold surface, avoiding level-set threshold search.

## Operating characteristics
*   **Time Complexity:** $O(V \log V)$ or $O(V)$ depending on the solver, where $V$ is the number of voxels. Solving the linear system is the dominant cost.
*   **Memory Complexity:** $O(V)$ for the voxel grid and sparse matrix.
*   **Strengths:** Produces watertight, manifold meshes; handles noisy data well with appropriate $\lambda$; global smoothness prior.
*   **Limitations:** Struggles with sparse inputs; requires tuning of `pointWeight`; may smooth out fine details if $\lambda$ is too low; artifacts in under-sampled regions if $\lambda$ is too high [S1, unofficial][S2].
*   **When to use:** When a watertight mesh is required from a dense, oriented point cloud, and fine detail preservation is less critical than topological correctness.
*   **When not to use:** For sparse point clouds or when preserving sharp features without smoothing is paramount.

## Background
SPSR was introduced by Kazhdan and Hoppe in 2013 [S1, unofficial]. It extends the original Poisson Surface Reconstruction by Hoppe et al. (1992) by adding a "screening" term to the Poisson equation, which allows for better handling of noisy data and varying point densities [S1, unofficial]. It is widely used in computer graphics for surface reconstruction from range scans and point clouds.

## Sources
[S1] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S2] Recent Advances and Trends in Learning‐Based 3D Representations, https://doi.org/10.1111/cgf.70530, fetched 2026-10-26
