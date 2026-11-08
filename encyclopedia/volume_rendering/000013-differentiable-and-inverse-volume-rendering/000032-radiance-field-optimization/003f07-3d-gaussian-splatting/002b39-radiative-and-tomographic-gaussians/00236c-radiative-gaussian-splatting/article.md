# Radiative Gaussian Splatting

Also called: Radiative Gaussians, R²-Gaussian

## Overview
Radiative Gaussian Splatting adapts 3D Gaussian Splatting (3DGS) for tomographic reconstruction by replacing view-dependent color and opacity with a view-independent scalar central density $\rho_i$ [S1]. It models isotropic density fields to estimate the 3D distribution of density $\sigma(\mathbf{x})$ from X-ray projections, solving the problem of reconstructing volumetric data from 2D attenuation measurements [S1]. Unlike standard 3DGS, which uses spherical harmonics for view-dependent appearance, this method removes angular dependence because X-ray attenuation depends only on the integrated density along the ray path [S1].

## How it works
**Inputs:** A set of $N$ X-ray projection images $\{I_i\}_{i=1}^N$ captured from different angles [S1].
**Outputs:** A discrete 3D density volume $\sigma(\mathbf{x})$ and/or rendered X-ray projections [S1].

**Data Structures:**
The scene is represented by a set of $M$ 3D Gaussian kernels $\mathcal{G}_3 = \{G_{3i}\}_{i=1}^M$ [S1]. Each kernel $G_{3i}$ is defined by:
*   $\rho_i \in \mathbb{R}$: Central density (scalar).
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position.
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix.
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix.
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [S1].

**Algorithm Steps:**

1.  **Initialization:** Initialize $M$ sparse Gaussians using Structure-from-Motion (SfM) points derived from the input projections [S1].
2.  **Ray Space Transformation:** Transform 3D Gaussians into ray space to align viewing rays with the coordinate axis, enhancing computational efficiency [S1].
3.  **Projection:** Project 3D Gaussians onto the 2D image plane to obtain 2D Gaussians $G_{2i}$. The projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
    $$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
    [S1, S_parent]
    The 2D Gaussian retains the central density but omits the third row and column of position and covariance relative to the 3D kernel [S1].
4.  **Density Normalization:** Compute the normalized density $\hat{\rho}_i$ for compositing:
    $$ \hat{\rho}_i = \rho_i \frac{\sqrt{2\pi |\mathbf{\hat{\Sigma}}_i|}}{|\mathbf{\hat{\Sigma}}_i|} $$
    [S1]
    *Note: The source formula in [S1] contains a dimensional inconsistency ($\sqrt{2\pi |\Sigma|} / |\Sigma|$). In standard 2D Gaussian splatting, the peak value is $\rho / (2\pi \sqrt{|\Sigma|})$. The source likely intends the normalization factor for alpha compositing. We implement the source's explicit formula as written, but note that standard rasterization uses $\alpha = 1 - \exp(-\hat{\rho}_i \dots)$ or similar alpha-blending.*
5.  **Rendering (Differentiable Radiative Rasterization):** Render the X-ray projection by compositing the 2D Gaussians using alpha-blending. The rendered intensity $I_{\text{rendered}}$ is computed by summing the contributions of sorted Gaussians along the ray:
    $$ I_{\text{rendered}} = \sum_{i=1}^M c_i \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j) $$
    [S1, S_parent]
    Where $\alpha_i$ is derived from the radiative intensity and spatial extent. For X-ray tomography, the "color" $c_i$ is typically unity or related to the source intensity, and the attenuation is modeled via the density $\rho_i$ [S1].
6.  **Optimization:** Optimize kernel parameters ($\rho_i, \mathbf{p}_i, \mathbf{R}_i, \mathbf{S}_i$) using photometric losses (L1, DSSIM) between rendered and real projections, plus Total Variation (TV) regularization on the density volume [S1].
7.  **Adaptive Control:** Dynamically densify Gaussians during training based on gradient magnitude or position error to improve scene representation [S1].
8.  **Volume Retrieval:** Retrieve the final 3D density volume using a differentiable voxelizer [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def radiative_gaussian_splatting(projections, num_gaussians, num_iterations):
    """
    Reconstructs a 3D density volume from X-ray projections using Radiative Gaussians.
    
    Args:
        projections: List of N 2D arrays (X-ray images).
        num_gaussians: M, number of 3D Gaussian kernels.
        num_iterations: Number of optimization steps.
        
    Returns:
        density_volume: 3D array representing sigma(x).
    """
    # 1. Initialization using SfM points (simulated here)
    # In practice, extract points from projections using SfM
    positions = initialize_sfm(projections) 
    scales = np.ones((num_gaussians, 3))
    rotations = np.eye(3).reshape(1, 3, 3).repeat(num_gaussians, axis=0)
    densities = np.ones(num_gaussians) # Initial central density rho_i
    
    # 2. Optimization Loop
    for step in range(num_iterations):
        rendered_projections = []
        
        for proj_idx, projection in enumerate(projections):
            # Get camera pose for this projection
            extrinsic = get_extrinsic(proj_idx)
            W = extrinsic[:3, :3] # Rotation part
            J = get_projection_jacobian(extrinsic) # Jacobian of affine projection
            
            # 3. Project 3D Gaussians to 2D
            # Compute 3D Covariance: Sigma = R S S^T R^T
            covs_3d = np.einsum('ijk,ikl,ilm->ijm', rotations, np.einsum('ijk,ilk->ijl', scales, scales), rotations)
            
            # Project Covariance: Sigma_hat = J W Sigma W^T J^T
            # Note: J is 2x3, W is 3x3, Sigma is 3x3
            # Implementation detail: batched matrix multiplication
            temp = np.einsum('ij,ikl->jkl', W, covs_3d) # W Sigma
            temp = np.einsum('ijk,ikl->ijl', temp, W.transpose(0,2,1)) # W Sigma W^T
            covs_2d = np.einsum('ij,jkl,ikl->ikl', J, temp, J.transpose(0,2,1)) # J (W Sigma W^T) J^T
            
            # 4. Compute Normalized Density for Alpha
            # det(Sigma_hat) for each Gaussian
            det_covs = np.linalg.det(covs_2d) # Shape (M,)
            # Avoid division by zero
            det_covs = np.maximum(det_covs, 1e-8)
            
            # Formula from [S1]: rho_hat = rho * sqrt(2*pi*|Sigma|) / |Sigma|
            # Note: This formula is dimensionally suspect for 2D Gaussian peak (usually 1/sqrt(det)), 
            # but we follow the source [S1] explicitly.
            # Standard 2D Gaussian peak is 1/(2*pi*sqrt(det)). 
            # The source might imply a specific normalization for the rasterizer.
            # We implement the source's literal formula.
            rho_hat = densities * np.sqrt(2 * np.pi * det_covs) / det_covs
            
            # 5. Rasterization (Alpha Compositing)
            # Sort Gaussians by depth (z-coordinate in ray space)
            # Ray space transformation T aligns rays with z-axis
            # For simplicity, assume depth sorting is handled by the rasterizer
            # Rendered image I = C(G2)
            rendered = rasterize_2d_gaussians(rho_hat, covs_2d, positions, J, W)
            rendered_projections.append(rendered)
            
        # 6. Loss Calculation
        # Photometric Loss (L1 + DSSIM)
        loss_render = compute_l1_loss(rendered_projections, projections)
        
        # Total Variation (TV) Regularization on Density Volume
        # Requires voxelization of current Gaussians
        density_vol = voxelize_gaussians(densities, positions, covs_3d)
        loss_tv = compute_tv(density_vol)
        
        total_loss = loss_render + 0.1 * loss_tv # Weight for TV
        
        # 7. Gradient Descent Update
        gradients = compute_gradients(total_loss, densities, positions, scales, rotations)
        densities -= 0.01 * gradients['densities']
        positions -= 0.01 * gradients['positions']
        scales -= 0.01 * gradients['scales']
        rotations -= 0.01 * gradients['rotations']
        
        # 8. Adaptive Control (Densification)
        # If gradient magnitude is high, split or clone Gaussians
        densify_gaussians(positions, densities, gradients)

    # 9. Final Volume Retrieval
    final_volume = voxelize_gaussians(densities, positions, covs_3d)
    return final_volume

def rasterize_2d_gaussians(rho_hat, covs_2d, positions, J, W):
    """
    Renders 2D Gaussians using alpha compositing.
    """
    # This is a placeholder for the actual differentiable rasterizer
    # which handles sorting, screen-space bounding boxes, and blending.
    # The core physics is that the alpha value is derived from rho_hat.
    pass

def voxelize_gaussians(densities, positions, covs_3d):
    """
    Differentiable voxelizer to retrieve 3D density volume.
    """
    # Sum contributions of all 3D Gaussians to a 3D grid
    # sigma(x) = sum G3i(x)
    pass
```

**Edge Cases and Pitfalls:**
*   **Anisotropic Effects:** Real-world X-ray projections contain minor anisotropic physical effects like Compton scattering. These are not explicitly modeled and are treated as noise [S1].
*   **Mesh Extraction:** Standard 3DGS opacity is empirically designed for RGB rendering, leading to challenges when extracting 3D meshes. Radiative Gaussians use central density, which is more suitable for volumetric representation but requires careful thresholding for mesh extraction [S1].
*   **Precision:** The determinant calculation in the normalization step can suffer from numerical instability if the projected Gaussian is degenerate (near-zero area). Guard against division by zero [S1].

## Subtopics

### Radiative Gaussian Point Cloud Representation
A variant where the representation is explicitly defined as a set of radiative Gaussian point clouds $\mathcal{G}_x$ for X-ray novel view synthesis [S2]. It includes a radiative feature vector $\mathbf{f}_i \in \mathbb{R}^{N_f}$ in addition to position, covariance, and opacity $\alpha_i$ [S2]. The formulation is $\mathcal{G}_x = \{G(\boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i, \alpha_i, \mathbf{f}_i) \mid i = 1, \dots, N_p\}$ [S2]. This variant excludes view direction dependence, matching the isotropic nature of X-ray imaging, and avoids complex Spherical Harmonics computations, making forward/backward passes faster than original 3DGS [S2].

## Operating characteristics
*   **Time Complexity:** Rendering is $O(M)$ per pixel, where $M$ is the number of Gaussians, similar to 3DGS. Optimization involves backpropagation through the rasterizer and voxelizer.
*   **Memory Complexity:** $O(M)$ to store Gaussian parameters. The density volume requires $O(V)$ memory where $V$ is the voxel grid size.
*   **Performance:** Faster than concurrent works using simplified isotropic Gaussians due to the flexibility of anisotropic covariance [S1]. Faster than original 3DGS for X-ray tasks because it avoids Spherical Harmonics [S2].
*   **Strengths:** Suitable for tomographic reconstruction and X-ray novel view synthesis. Provides a direct 3D density volume [S1].
*   **Limitations:** Does not model anisotropic physical effects like Compton scattering [S1]. Mesh extraction from the density field requires post-processing [S1].

## Background
Radiative Gaussian Splatting was introduced to address the limitations of standard 3DGS in tomographic reconstruction. Standard 3DGS uses view-dependent color and opacity, which are inappropriate for X-ray imaging where attenuation depends on isotropic density [S1]. The method was developed to provide a flexible kernel-based representation using general anisotropic Gaussians, offering more precision than simplified isotropic approaches [S1]. It builds on the 3DGS framework [23] but adapts the rasterization and optimization for X-ray physics [S1].

## Sources
[S1] R$^2$-Gaussian: Rectifying Radiative Gaussian Splatting for Tomographic Reconstruction, https://arxiv.org/abs/2405.20693v2, fetched 2026-10-26
[S2] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
