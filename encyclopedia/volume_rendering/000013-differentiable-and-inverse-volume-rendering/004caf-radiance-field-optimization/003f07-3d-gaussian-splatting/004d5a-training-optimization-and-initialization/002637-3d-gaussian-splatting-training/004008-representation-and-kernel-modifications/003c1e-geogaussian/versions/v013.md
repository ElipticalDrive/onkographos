# GeoGaussian

## Overview
GeoGaussian is a training optimization and initialization strategy for 3D Gaussian Splatting that addresses geometric inconsistency and aliasing artifacts by modifying the densification process. It solves the problem of Gaussians growing in arbitrary directions during gradient descent, which leads to jagged surfaces and poor normal consistency. It sets itself apart from standard 3DGS by constraining the growth of new primitives to the tangent plane of existing Gaussians and enforcing smoothness of geometric properties between neighboring primitives [S1].

## How it works
GeoGaussian modifies the densification phase of 3D Gaussian Splatting. It does not alter the rasterization kernel or the primitive attributes' storage format, but changes how new Gaussians are spawned and how their geometric attributes are regularized.

**Inputs:**
- Current set of 3D Gaussians $\mathcal{G} = \{G_i\}$, where each $G_i$ is defined by position $\mathbf{p}_i$, covariance $\Sigma_i$, opacity $\alpha_i$, and color $\mathbf{c}_i$.
- Gradient information from the rendering loss.

**Outputs:**
- Updated set of Gaussians $\mathcal{G}'$ with densified primitives.

**Data Structures:**
- Each Gaussian $G_i$ retains the standard 3DGS representation:
  - $\mathbf{p}_i \in \mathbb{R}^3$: Position.
  - $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, parameterized via rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$ such that $\Sigma_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$.
  - $\alpha_i \in [0, 1]$: Opacity.
  - $\mathbf{c}_i$: Color coefficients (e.g., SH coefficients).

**Steps:**
1. **Tangent Plane Projection:** When a Gaussian $G_i$ is selected for densification (based on standard 3DGS criteria such as high gradient magnitude or low opacity), new candidate Gaussians are not placed in the full 3D space. Instead, they are constrained to the tangent plane of $G_i$.
   - The normal vector $\mathbf{n}_i$ of the tangent plane is derived from the orientation of $G_i$. In 3DGS, the orientation is given by the rotation matrix $\mathbf{R}_i$. The normal can be approximated by the third column of $\mathbf{R}_i$ (assuming the local z-axis aligns with the normal) or derived from the eigenvectors of $\Sigma_i$ corresponding to the smallest eigenvalue.
   - New positions $\mathbf{p}_{new}$ are sampled from a distribution centered at $\mathbf{p}_i$ but restricted to the plane defined by $\mathbf{n}_i \cdot (\mathbf{x} - \mathbf{p}_i) = 0$.

2. **Geometric Smoothness Regularization:** To encourage smoothness of geometric properties between neighboring Gaussians, a regularization term is added to the loss function.
   - Let $\mathcal{N}(i)$ be the set of neighbors of Gaussian $i$ (e.g., based on spatial proximity or adjacency in the point cloud).
   - The loss includes a term penalizing the difference in normals or orientations between $G_i$ and its neighbors $G_j \in \mathcal{N}(i)$.
   - A common formulation is:
     $$ \mathcal{L}_{smooth} = \sum_{i} \sum_{j \in \mathcal{N}(i)} \| \mathbf{n}_i - \mathbf{n}_j \|^2 $$
   - This ensures that adjacent Gaussians have consistent orientations, reducing noise and jaggedness in the rendered surface.

3. **Densification:** New Gaussians are added to the scene with initial attributes derived from $G_i$ (position projected to tangent plane, similar covariance structure, and interpolated color/opacity).

**Parameters:**
- Neighbor radius or count for smoothness calculation: Not given in the sources.
- Weight of the smoothness regularization term: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Degenerate Gaussians:** If a Gaussian has a very small scale in one direction, the tangent plane might be ill-defined. [G] A guard should check the condition number of $\Sigma_i$ or the magnitude of the smallest eigenvalue before computing the normal.
- **Boundary Conditions:** At the boundaries of the scene or sparse regions, neighbor sets may be empty. [G] The smoothness term should be skipped or handled gracefully if $\mathcal{N}(i)$ is empty.

## Subtopics
## Operating characteristics
- **Time Complexity:** Densification is performed periodically, not every step. The tangent plane projection adds a small constant overhead per densified Gaussian. The smoothness regularization requires computing neighbor relationships, which can be optimized with spatial hashing or k-d trees. Overall, it adds minimal overhead to the standard 3DGS training loop.
- **Memory Complexity:** No additional memory per Gaussian compared to standard 3DGS. The neighbor graph may require additional storage if precomputed, but this is typically negligible.
- **Performance:** Improves geometric quality and reduces aliasing artifacts, especially on curved surfaces. [S1]
- **Strengths:** Produces smoother surfaces, better normal consistency, and reduces the "floating" artifact common in standard 3DGS.
- **Limitations:** May slow down convergence slightly due to the regularization term. The choice of neighbor definition can affect performance.
- **When to use:** When high-quality geometry and smooth surfaces are required, particularly for scenes with curved surfaces.
- **When not to use:** When training speed is the primary constraint and geometric smoothness is less critical.

## Background
GeoGaussian was proposed in the paper "GeoGaussian: Geometry-aware Gaussian Splatting for Scene Rendering" [41], as cited in [S1]. It is part of the broader effort to improve the geometric fidelity of 3D Gaussian Splatting. The method builds on the observation that unconstrained growth of Gaussians leads to geometric inconsistencies.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
