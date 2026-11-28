# Gaussian Piling

Also called: Focus Gaussian, GaussianPile, Slice-based Volumetric Reconstruction

## Overview
Gaussian Piling is a slice-based volumetric reconstruction method that unifies 3D Gaussian Splatting with an imaging system-aware focus model to compress aggressively while preserving internal structure [S1, unofficial]. Unlike standard 3D Gaussian Splatting, which discards internal volumetric information and produces incoherent 3D structures with floating artifacts for slice-based modalities, Gaussian Piling models the finite focal zone to preserve high-frequency internal volumetric detail [S1, unofficial]. It achieves this by enforcing 3D-to-2D consistency through axial re-parameterization and opacity modulation, preventing ghosting artifacts across slices [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ 3D Gaussian primitives $\{G_i\}_{i=1}^N$ defined in world space, each with center $\boldsymbol{\mu}$, covariance $\boldsymbol{\Sigma}$, opacity $\alpha$, and scale/orientation parameters. The imaging system's axial resolution $\sigma_z$ (standard deviation of the point spread function in the elevational direction).
*   **Outputs:** Rendered 2D slice images $I(p)$ and/or a 3D voxel volume $\sigma(\mathbf{x})$.

**Data Structures:**
Each Gaussian primitive $i$ stores:
*   Position $\boldsymbol{\mu}_i \in \mathbb{R}^3$ (world space).
*   Scale $s_{i,x}, s_{i,y}, s_{i,z} \in \mathbb{R}^+$ and Rotation $q_i$ (unit quaternion), combined to form covariance $\boldsymbol{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [S1, unofficial].
*   Opacity $\alpha_i \in [0, 1]$.
*   (Note: Spherical Harmonics coefficients are discarded in this method, reducing parameters per primitive by ~40% [S1, unofficial].)

**Rendering Pipeline (Slice-based):**
The rendering process consists of three stages for a given slice at camera depth $z_c=0$:

1.  **Scan:** Project Gaussians onto the slice plane.
2.  **Axial Re-parameterization and Opacity Modulation:** Transform the camera-space Gaussian into a "Focus Gaussian" that accounts for the finite slice thickness and attenuate opacity for out-of-focus primitives.
3.  **Screen-space Projection and Additive Rasterization:** Compute 2D marginal distributions and accumulate contributions additively.

**Step-by-Step Derivation:**

1.  **Coordinate Transformation:** Convert world-space parameters to camera space $(\cdot)_c$ using the camera extrinsics. Let $\boldsymbol{\mu}_c$ and $\boldsymbol{\Sigma}_c$ be the position and covariance in camera coordinates.

2.  **Axial Re-parameterization:**
    Compute the effective inverse covariance matrix $\boldsymbol{\Sigma}_e^{-1}$ by adding the axial resolution term to the camera-space inverse covariance [S1, unofficial]:
    $$ \boldsymbol{\Sigma}_e^{-1} = \boldsymbol{\Sigma}_c^{-1} + \frac{\mathbf{e}_3 \mathbf{e}_3^\top}{\sigma_z^2} $$
    where $\mathbf{e}_3 = [0, 0, 1]^\top$ and $\sigma_z$ is the axial resolution (PSF standard deviation).
    Compute the effective mean $\boldsymbol{\mu}_e$:
    $$ \boldsymbol{\mu}_e = \boldsymbol{\Sigma}_e \boldsymbol{\Sigma}_c^{-1} \boldsymbol{\mu}_c $$
    This operation shrinks the axial support without disturbing the lateral structure, producing an effective Gaussian $G_e \sim \mathcal{G}(\boldsymbol{\mu}_e, \boldsymbol{\Sigma}_e)$ [S1, unofficial].

3.  **Opacity Modulation:**
    Calculate the rendered opacity factor $\text{opacity}_r$ based on the change in Mahalanobis distance between the original and effective Gaussians [S1, unofficial]:
    $$ \text{opacity}_r = \exp\left(-\frac{1}{2}(\boldsymbol{\mu}_c^\top \boldsymbol{\Sigma}_c^{-1} \boldsymbol{\mu}_c - \boldsymbol{\mu}_e^\top \boldsymbol{\Sigma}_e^{-1} \boldsymbol{\mu}_e)\right) $$
    This factor downweights primitives far from the focal vicinity, preventing ghosting artifacts [S1, unofficial]. Primitives with $\text{opacity}_r < \tau$ (default $\tau=0.02$) are culled [S1, unofficial].

4.  **Screen-space Projection:**
    Extract the 2D covariance $\boldsymbol{\Sigma}_{2d}$ and mean $\boldsymbol{\mu}_{2d}$ from the effective Gaussian by marginalizing over the $(x_c, y_c)$ plane [S1, unofficial]:
    $$ \boldsymbol{\Sigma}_{2d} = \begin{bmatrix} \Sigma_e[0,0] & \Sigma_e[1,0] \\ \Sigma_e[1,0] & \Sigma_e[1,1] \end{bmatrix}, \quad \boldsymbol{\mu}_{2d} = [\mu_e[0], \mu_e[1]]^\top $$
    Normalize the final rendered intensity $\tilde{\alpha}$ using the determinant of the 2D covariance to preserve total energy [S1, unofficial]:
    $$ \tilde{\alpha} = \alpha \cdot \text{opacity}_r / \sqrt{\det(\boldsymbol{\Sigma}_{2d})} $$

5.  **Additive Rasterization:**
    For each pixel $p$, sum the contributions of overlapping Gaussians $i \in \mathcal{N}(p)$ [S1, unofficial]:
    $$ I(p) = \sum_{i \in \mathcal{N}(p)} \tilde{\alpha}_i \exp\left(-\frac{1}{2}\mathbf{d}_i^\top \boldsymbol{\Sigma}_{2d,i}^{-1} \mathbf{d}_i\right) $$
    where $\mathbf{d}_i = \boldsymbol{\mu}_{2d,i} - p$ is the vector from the 2D Gaussian center to the pixel. Note that standard alpha blending is replaced by additive accumulation because slice imaging intensity originates from the integral of opacities along the projection line without occlusion [S1, unofficial].

**Differentiable Voxelization (Optional 3D Output):**
To generate a 3D volume, a differentiable voxelizer partitions space into uniform $8^3$ tiles. It culls Gaussians per tile, retaining only those whose spatial support (3-$\sigma$ ellipsoid) intersects the tile. Voxel intensity $\sigma(\mathbf{x})$ is computed by summing world-space Gaussians [S1, unofficial]:
$$ \sigma(\mathbf{x}) = \sum_{i=1}^M g_i^3(\mathbf{x}; \boldsymbol{\mu}_i, \boldsymbol{\Sigma}_i) $$
where $g_i^3$ is the original unnormalized Gaussian function [S1, unofficial]:
$$ p(\mathbf{x}) = \exp\left(-\frac{1}{2}(\mathbf{x} - \boldsymbol{\mu})^\top \boldsymbol{\Sigma}^{-1}(\mathbf{x} - \boldsymbol{\mu})\right) $$

**Optimization:**
*   **Loss Function:** $\mathcal{L} = \mathcal{L}_1 + \lambda \mathcal{L}_{ssim}$, where $\mathcal{L}_1$ is L1 loss and $\mathcal{L}_{ssim}$ is D-SSIM loss [S1, unofficial].
*   **Gradients:** Full gradient propagation is implemented in CUDA. The gradient w.r.t. opacity $\alpha$ is:
    $$ \frac{\partial \mathcal{L}}{\partial \alpha} = \frac{\partial \mathcal{L}}{\partial \tilde{\alpha}} \cdot \frac{\text{opacity}_r}{\sqrt{\det(\boldsymbol{\Sigma}_{2d})}} $$
    The gradient w.r.t. effective covariance $\boldsymbol{\Sigma}_e$ couples the determinant term and the conic term [S1, unofficial].
*   **Densification/Pruning:** Uses tile-wise focus radii and opacity scores. Adaptive control runs with a gradient threshold of 0.00005 [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_piling_render(gaussians, camera_extrinsics, sigma_z, image_width, image_height):
    """
    Renders 2D slices using Gaussian Piling.
    
    Args:
        gaussians: List of Gaussian primitives. Each has:
            - mu_world: (3,) array, position in world space.
            - scale: (3,) array, anisotropic scaling.
            - rot: (4,) array, unit quaternion.
            - alpha: float, opacity.
        camera_extrinsics: Camera-to-world matrix (4x4).
        sigma_z: float, axial resolution of the imaging system.
        image_width, image_height: int, dimensions of the output slice.
        
    Returns:
        rendered_image: (H, W) array.
    """
    # 1. Transform Gaussians to Camera Space
    cam_gaussians = []
    for g in gaussians:
        # Transform position
        mu_world = g.mu_world
        mu_cam = transform_point(camera_extrinsics, mu_world)
        
        # Transform covariance
        R_world = quaternion_to_rotation_matrix(g.rot)
        S = diag(g.scale)
        Sigma_world = R_world @ S @ S.T @ R_world.T
        
        # Camera rotation matrix (inverse of extrinsic rotation)
        R_cam_to_world = camera_extrinsics[:3, :3]
        R_world_to_cam = R_cam_to_world.T
        
        Sigma_cam = R_world_to_cam @ Sigma_world @ R_world_to_cam.T
        
        cam_gaussians.append({
            'mu_c': mu_cam,
            'Sigma_c': Sigma_cam,
            'alpha': g.alpha
        })
    
    # 2. Axial Re-parameterization and Opacity Modulation
    focus_gaussians = []
    e3 = np.array([0, 0, 1])
    
    for g in cam_gaussians:
        mu_c = g['mu_c']
        Sigma_c = g['Sigma_c']
        alpha = g['alpha']
        
        # Compute effective inverse covariance
        # Sigma_e_inv = Sigma_c_inv + (e3 @ e3.T) / sigma_z^2
        Sigma_c_inv = np.linalg.inv(Sigma_c)
        axial_term = (e3.reshape(-1, 1) @ e3.reshape(1, -1)) / (sigma_z ** 2)
        Sigma_e_inv = Sigma_c_inv + axial_term
        
        # Compute effective covariance and mean
        Sigma_e = np.linalg.inv(Sigma_e_inv)
        mu_e = Sigma_e @ Sigma_c_inv @ mu_c
        
        # Opacity modulation
        # opacity_r = exp(-0.5 * (mu_c^T Sigma_c_inv mu_c - mu_e^T Sigma_e_inv mu_e))
        mahal_c = mu_c.T @ Sigma_c_inv @ mu_c
        mahal_e = mu_e.T @ Sigma_e_inv @ mu_e
        opacity_r = np.exp(-0.5 * (mahal_c - mahal_e))
        
        # Cull if opacity is too low
        if opacity_r < 0.02: # Default tau
            continue
            
        # Screen-space projection (Marginalization)
        # Sigma_2d is the top-left 2x2 block of Sigma_e
        Sigma_2d = Sigma_e[:2, :2]
        mu_2d = mu_e[:2]
        
        # Normalize intensity
        det_Sigma_2d = np.linalg.det(Sigma_2d)
        if det_Sigma_2d <= 0:
            continue # Degenerate case
        alpha_tilde = alpha * opacity_r / np.sqrt(det_Sigma_2d)
        
        focus_gaussians.append({
            'mu_2d': mu_2d,
            'Sigma_2d': Sigma_2d,
            'alpha_tilde': alpha_tilde
        })
        
    # 3. Additive Rasterization
    image = np.zeros((image_height, image_width))
    
    for g in focus_gaussians:
        mu_2d = g['mu_2d']
        Sigma_2d = g['Sigma_2d']
        alpha_tilde = g['alpha_tilde']
        
        # Compute bounding box for this Gaussian in screen space
        # Approximate using 3-sigma extent
        eigenvalues, eigenvectors = np.linalg.eigh(Sigma_2d)
        max_extent = 3.0 * np.sqrt(np.max(eigenvalues))
        
        min_x = max(0, int(mu_2d[0] - max_extent))
        max_x = min(image_width, int(mu_2d[0] + max_extent))
        min_y = max(0, int(mu_2d[1] - max_extent))
        max_y = min(image_height, int(mu_2d[1] + max_extent))
        
        Sigma_2d_inv = np.linalg.inv(Sigma_2d)
        
        for y in range(min_y, max_y):
            for x in range(min_x, max_x):
                pixel_pos = np.array([x, y])
                d = mu_2d - pixel_pos
                exponent = -0.5 * (d.T @ Sigma_2d_inv @ d)
                contribution = alpha_tilde * np.exp(exponent)
                image[y, x] += contribution
                
    return image
```

**Parameters:**
*   $\sigma_z$: Axial resolution. Estimated theoretically, numerically, or via experimental calibration. If unknown, assumed equal to scanning step $\delta_z$ satisfying Nyquist criterion ($\sigma_z \approx \delta_z$) [S1, unofficial].
*   $\tau$: Opacity minimal threshold for culling. Default: 0.02 [S1, unofficial].
*   $\lambda$: Weight balancing perceptual (SSIM) and pixel-wise (L1) fidelity. Default: 0.2 [S1, unofficial].
*   Learning Rates: Position 0.0006, Opacity 0.02, Scale 0.002, Rotation 0.001. Exponentially decay to 0.1 of initial values [S1, unofficial].
*   $M$: Initial number of Gaussians. Default: 1000k [S1, unofficial].
*   Gradient threshold for adaptive control: 0.00005 [S1, unofficial].

**Pitfalls:**
*   **Ghosting Artifacts:** Standard 3DGS produces floating artifacts across slices because it discards internal volumetric information. Gaussian Piling prevents this by downweighting primitives far from the focal vicinity via opacity modulation [S1, unofficial].
*   **Degenerate Covariance:** If $\det(\boldsymbol{\Sigma}_{2d}) \le 0$, the Gaussian is degenerate and should be skipped to avoid division by zero or invalid projections [G].
*   **Boundary Conditions:** Ensure bounding box calculations for rasterization respect image boundaries to avoid out-of-bounds memory access [G].

## Operating characteristics
*   **Time Complexity:** The method delivers high-quality results in as few as 3 minutes, up to 11× faster than NeRF-based approaches [S1, unofficial]. The differentiable voxelizer achieves > 100 FPS for real-time query performance [S1, unofficial].
*   **Memory Complexity:** Quantization and compression achieve a 16× reduction over voxel-based storage while retaining full 2D rendering fidelity and volumetric consistency [S1, unofficial].
*   **Strengths:** Preserves high-frequency internal volumetric detail; enforces 3D-to-2D consistency; discards spherical harmonics to reduce parameters per primitive by ~40% and improve geometric fidelity [S1, unofficial].
*   **Limitations:** Designed specifically for slice-based volumetric imaging (e.g., microscopy, ultrasound); not directly applicable to standard multi-view surface reconstruction without modification [S1, unofficial].
*   **When to use:** When reconstructing slice-based volumetric datasets (medical imaging, microscopy) where internal structure preservation and compression are critical.
*   **When not to use:** For standard surface rendering from multi-view images where internal volume is irrelevant.

## Background
Gaussian Piling (GaussianPile) was introduced to bridge the gap between the high compression ratio of Implicit Neural Representations (INRs) and the fast fitting of 3D Gaussian Splatting (3DGS) for slice-based volumetric imaging [S1, unofficial]. Standard 3DGS is fundamentally designed to model surface appearance from multi-view images and discards internal volumetric information, making it unsuitable for scientific and medical imaging where complete internal structure is paramount [S1, unofficial]. The method unifies 3D Gaussian splatting with an imaging system-aware focus model, incorporating the physical properties of the imaging system (specifically the point spread function) into the rendering pipeline [S1, unofficial]. It is implemented in PyTorch and CUDA, trained with the Adam optimizer for 30k iterations [S1, unofficial].

## Sources
[S1] GaussianPile: A Unified Sparse Gaussian Splatting Framework for Slice-based Volumetric Reconstruction, https://doi.org/10.48550/arXiv.2603.20611, fetched 2026-10-27 (unofficial)
