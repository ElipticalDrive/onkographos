# GeoGaussian


## Overview
GeoGaussian is a training optimization and initialization strategy for 3D Gaussian Splatting that improves geometric consistency and surface reconstruction quality. It addresses the issue of noisy or inconsistent normal estimates in standard 3DGS by densifying Gaussians on the tangent plane of existing primitives and enforcing smoothness of geometric properties between neighboring Gaussians [S1]. Unlike standard 3DGS which treats Gaussians as independent entities, GeoGaussian explicitly models the spatial relationship between neighbors to encourage planar consistency [S1].

## How it works
GeoGaussian modifies the standard 3DGS training loop by introducing two key geometric constraints during the densification and optimization phases.

**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\{G_i\}$ with mean $\boldsymbol{\mu}_i$, covariance $\boldsymbol{\Sigma}_i$, opacity $\alpha_i$, and spherical harmonic coefficients $\mathbf{c}_i$, along with a set of input images with known camera poses.
*   **Output:** Optimized Gaussian attributes with improved geometric smoothness and reduced surface noise.

**Data Structures:**
*   **Gaussian Set:** Standard 3DGS representation.
*   **Neighbor List:** For each Gaussian $G_i$, a list of spatially adjacent Gaussians $\mathcal{N}_i$ is maintained or computed based on distance in 3D space.

**Steps:**
1.  **Tangent Plane Densification:** When a Gaussian $G_i$ requires splitting (based on standard 3DGS criteria such as high gradient magnitude or small screen size), new Gaussians are not placed randomly in 3D space. Instead, they are generated on the tangent plane of $G_i$.
    *   The tangent plane is defined by the normal vector $\mathbf{n}_i$ of $G_i$. The normal is derived from the covariance matrix $\boldsymbol{\Sigma}_i$. Specifically, if $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^T \mathbf{R}_i^T$, the normal $\mathbf{n}_i$ corresponds to the direction of the smallest singular value of $\boldsymbol{\Sigma}_i$ (the axis of least extent) [G].
    *   New means $\boldsymbol{\mu}_{new}$ are sampled from a distribution centered at $\boldsymbol{\mu}_i$ but constrained to lie on this tangent plane.
2.  **Geometric Smoothness Regularization:** A loss term is added to the optimization objective to encourage neighboring Gaussians to share similar geometric properties (specifically normals).
    *   Let $\mathbf{n}_i$ be the normal of Gaussian $i$.
    *   The smoothness loss $\mathcal{L}_{geo}$ is defined as:
        $$ \mathcal{L}_{geo} = \sum_{i} \sum_{j \in \mathcal{N}_i} (1 - |\mathbf{n}_i \cdot \mathbf{n}_j|) $$
    *   This term penalizes large angular differences between the normals of adjacent Gaussians, promoting planar surface continuity [S1].

**Parameters:**
*   **Neighbor Radius:** The distance threshold for defining $\mathcal{N}_i$. Not given in the sources.
*   **Regularization Weight ($\lambda_{geo}$):** The weight for $\mathcal{L}_{geo}$. Not given in the sources.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def geo_gaussian_step(gaussians, neighbors, lambda_geo):
    """
    Performs one step of GeoGaussian optimization.
    """
    # 1. Compute normals for all Gaussians
    # Normal is the eigenvector corresponding to the smallest eigenvalue of Sigma
    normals = compute_normals(gaussians.Sigma)
    
    # 2. Calculate Geometric Smoothness Loss
    loss_geo = 0.0
    for i in range(len(gaussians)):
        n_i = normals[i]
        for j in neighbors[i]:
            n_j = normals[j]
            # Penalize difference in dot product (encourages parallel normals)
            loss_geo += (1.0 - abs(dot(n_i, n_j)))
    loss_geo *= lambda_geo
    
    # 3. Standard 3DGS Loss (Reconstruction + SSIM)
    loss_recon = compute_rendering_loss(gaussians)
    
    # 4. Total Loss
    total_loss = loss_recon + loss_geo
    
    # 5. Gradient Update
    update_gaussians(total_loss)
    
    # 6. Densification with Tangent Plane Constraint
    if needs_densification(gaussians):
        for g in gaussians_to_split:
            # Get tangent plane normal
            n_tangent = normals[g.id]
            # Generate new Gaussians on the plane defined by n_tangent
            new_gaussians = generate_on_tangent_plane(g, n_tangent)
            add_gaussians(new_gaussians)
```

## Subtopics

### [Spatio-temporal Gaussian Tracking](9773-spatio-temporal-gaussian-tracking.md)
Spatio-temporal Gaussian Tracking (SGT) is a method for modeling surface dynamics in dynamic scenes by tracking the attribute shift of each Gaussian at time $t$ using a deformation field $D(\mu, t)$, allowing the rendering of deformed Gaussians $G_t$. It sets itself apart from static representations by explicitly parameterizing temporal deformation through a lightweight, decomposed encoding voxel and a set of small MLPs, prioritizing real-time inference speed over large neural network approximations.

### [Progressive training by scales of Gaussians](9871-progressive-training-by-scales-of-gaussians.md)
This method prevents rendering artifacts caused by 3D Gaussians becoming smaller than a single pixel during optimization by enforcing a minimum projected area through covariance regularization. It differs from standard 3D Gaussian Splatting by introducing a scale-dependent low-pass filter that focuses training on low-frequency details in early stages, ensuring coarse scene structure is learned efficiently before fine details are resolved.

## Operating characteristics
*   **Time Complexity:** Adds overhead for computing normals and neighbor interactions. Neighbor computation can be optimized with spatial hashing. The additional loss term is lightweight.
*   **Memory Complexity:** Requires storing neighbor relationships or recomputing them, adding minimal memory overhead compared to standard 3DGS.
*   **Strengths:** Improves surface smoothness, reduces noise in reconstructed geometry, and enhances normal consistency [S1].
*   **Limitations:** May oversmooth fine geometric details if the regularization weight is too high. Requires careful tuning of the neighbor radius and loss weight.
*   **When to use:** When high-quality surface normals and smooth geometry are required, such as in architectural scenes or objects with large planar surfaces.
*   **When not to use:** For scenes with high-frequency noise or complex textures where geometric smoothness is less critical than appearance fidelity.

## Background
GeoGaussian was proposed in the context of recent advances in 3D Gaussian Splatting [S1]. It is cited as reference [41] in the survey "Recent advances in 3D Gaussian splatting" [S1]. The method builds upon the idea of using geometric constraints to improve the quality of Gaussian primitives, similar to how GaussianPro uses plane constraints [S1].

## Sources
[S1] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
