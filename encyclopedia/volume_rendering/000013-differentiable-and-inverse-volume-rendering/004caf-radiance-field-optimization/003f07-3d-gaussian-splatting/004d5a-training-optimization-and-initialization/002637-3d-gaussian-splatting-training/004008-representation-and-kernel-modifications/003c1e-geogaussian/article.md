# GeoGaussian


## Overview
GeoGaussian is a training optimization and initialization strategy for 3D Gaussian Splatting that addresses geometric inconsistency and poor surface reconstruction by modifying the densification process. It solves the problem of noisy, non-smooth geometry often produced by standard 3DGS densification, which tends to create scattered, unstructured primitives. It sets itself apart from siblings like Negative Gaussian Splatting [9152-negative-gaussian-splatting.md] (which allows negative opacity) and Scaffold-GS [9186-scaffold-gs.md] (which uses latent features) by strictly constraining the spatial growth of Gaussians to the tangent plane of existing primitives and enforcing geometric smoothness between neighbors, thereby improving surface fidelity without altering the fundamental alpha-compositing pipeline.

## How it works
GeoGaussian modifies the standard 3D Gaussian Splatting training loop, specifically the densification and regularization phases. It operates within the differentiable rendering framework defined in [Representation and Kernel Modifications](10145-representation-and-kernel-modifications.md).

**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$, where each $G_i$ is defined by position $\mathbf{p}_i$, covariance $\Sigma_i$ (parameterized by rotation $\mathbf{R}_i$ and scale $\mathbf{s}_i$), opacity $\alpha_i$, and spherical harmonic coefficients for color.
*   **Output:** An updated set of Gaussians with improved geometric coherence and smoother surface representation.

**Data Structures:**
The method relies on the standard 3DGS attribute storage:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Center position.
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix (often stored as a quaternion $\mathbf{q}_i$).
*   $\mathbf{s}_i \in \mathbb{R}^3$: Scale vector (log-space for optimization stability).
*   $\alpha_i \in [0, 1]$: Opacity.
*   $\mathbf{c}_i$: Color coefficients.

**Numbered Steps:**

1.  **Tangent Plane Densification:**
    When a Gaussian $G_i$ requires splitting or cloning (based on standard 3DGS criteria such as high gradient magnitude or low opacity), GeoGaussian does not spawn new Gaussians in the full 3D volume. Instead, it restricts the placement of new Gaussians to the tangent plane of $G_i$.
    *   Compute the normal vector $\mathbf{n}_i$ of $G_i$ from its covariance $\Sigma_i$. The normal is the eigenvector corresponding to the smallest eigenvalue of $\Sigma_i$.
    *   Generate new candidate positions $\mathbf{p}_{new}$ by perturbing $\mathbf{p}_i$ along the basis vectors of the tangent plane (orthogonal to $\mathbf{n}_i$).
    *   This ensures that new primitives lie on the local surface approximation defined by $G_i$, reducing volumetric noise.

2.  **Geometric Smoothness Regularization:**
    To encourage consistency between neighboring Gaussians, GeoGaussian introduces a regularization term that penalizes large differences in geometric properties (position, normal, scale) between adjacent Gaussians.
    *   Identify neighboring Gaussians $\{G_j\}$ for each $G_i$ based on spatial proximity or overlap in the image plane.
    *   Compute the difference in normals $\Delta \mathbf{n}_{ij} = \mathbf{n}_i - \mathbf{n}_j$ and positions $\Delta \mathbf{p}_{ij} = \mathbf{p}_i - \mathbf{p}_j$.
    *   Add a loss term $\mathcal{L}_{geo}$ to the total training loss:
        $$ \mathcal{L}_{geo} = \sum_{i} \sum_{j \in \mathcal{N}(i)} \left( \| \mathbf{n}_i - \mathbf{n}_j \|_2^2 + \lambda_p \| \mathbf{p}_i - \mathbf{p}_j \|_2^2 \right) $$
        where $\mathcal{N}(i)$ is the set of neighbors of $G_i$, and $\lambda_p$ is a weighting hyperparameter.
    *   This term encourages the normals and positions of nearby Gaussians to be similar, promoting a smooth surface.

3.  **Optimization Update:**
    Perform standard gradient descent on the combined loss $\mathcal{L}_{total} = \mathcal{L}_{render} + \lambda_{geo} \mathcal{L}_{geo}$, where $\mathcal{L}_{render}$ is the standard photometric loss. The gradients from $\mathcal{L}_{geo}$ update the position, rotation, and scale parameters of the Gaussians.

**Parameters:**
*   $\lambda_{geo}$: Weight for the geometric smoothness loss. Not given in the sources.
*   $\lambda_p$: Weight for position consistency in the smoothness term. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Degenerate Covariance:** If a Gaussian becomes extremely thin (planar), the normal estimation from the smallest eigenvalue eigenvector may become unstable. [G] A minimum eigenvalue threshold or regularization on the covariance matrix may be required to ensure stable normal computation.
*   **Neighbor Definition:** The definition of "neighboring" Gaussians is critical. In standard 3DGS, neighbors are often determined by spatial hashing or k-d trees. The specific algorithm for identifying neighbors is not detailed in the sources [S1].

## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
*   **Time Complexity:** The addition of neighbor search and geometric smoothness regularization increases the per-step computational cost compared to standard 3DGS. Neighbor search typically scales with $O(N \log N)$ or $O(N)$ with spatial hashing. The regularization loss calculation scales with the number of edges in the neighbor graph.
*   **Memory Complexity:** No significant additional memory overhead beyond storing neighbor relationships, which is typically sparse.
*   **Strengths:** Produces smoother, more coherent surface geometry; reduces noise and artifacts in reconstructed surfaces; improves normal map quality.
*   **Limitations:** May converge slower due to the additional regularization constraints; performance depends heavily on the definition of neighbors and the choice of regularization weights.
*   **When to use:** When high-quality surface reconstruction and smooth normals are required, such as in architectural scenes or objects with smooth surfaces.
*   **When not to use:** For scenes with highly complex, non-surface-like geometry (e.g., foliage, hair) where the tangent plane assumption may be less valid, or when computational efficiency is the primary constraint.

## Background
GeoGaussian was proposed in the context of recent advances in 3D Gaussian Splatting, specifically addressing the limitations of the original densification strategy which could lead to geometric inconsistencies. It is cited alongside other methods like GaussianPro [32] and RadSplat [42] in surveys of 3DGS improvements [S1]. The method emphasizes the importance of geometric priors in differentiable rendering.

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
