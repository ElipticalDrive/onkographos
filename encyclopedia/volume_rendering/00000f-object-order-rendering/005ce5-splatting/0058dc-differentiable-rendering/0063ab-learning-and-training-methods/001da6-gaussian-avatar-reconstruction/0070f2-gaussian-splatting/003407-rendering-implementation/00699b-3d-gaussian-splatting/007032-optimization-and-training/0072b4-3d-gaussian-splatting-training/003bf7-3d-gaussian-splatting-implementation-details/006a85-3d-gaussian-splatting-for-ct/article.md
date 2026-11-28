# 3D Gaussian Splatting for CT

Also called: 3D Gaussian Representation-based CT reconstruction, 3DGR-CT, Sparse-view CT reconstruction with a 3D Gaussian representation

## Overview
3D Gaussian Splatting for CT (3DGR-CT) is a sparse-view Computed Tomography (CT) reconstruction method that represents the volumetric density field as a set of learnable 3D Gaussians, optimized via a differentiable CT projector [S1]. It solves the artifacts and noise inherent in sparse-view acquisitions by replacing implicit neural representations (INRs) with an explicit Gaussian basis, enabling precise recovery of physical CT values required for clinical applications like dose calculation [S1]. Unlike standard 3D Gaussian Splatting which uses 2D splatting for novel view synthesis, this method employs NeRF-style volumetric rendering to align with X-ray physics, and unlike INRs, it avoids spectral bias to better reconstruct sharp boundaries [S1].

## How it works
**Inputs:** Sparse-view projection data $P$.
**Outputs:** A set of 3D Gaussians $\{G_i\}$ defined by parameters $\{\mu_i, \Sigma_i, t_i\}$, representing the reconstructed CT volume.

**Data Structures:**
Each Gaussian $i$ is defined by:
*   $\mu_i \in \mathbb{R}^3$: Center coordinate (normalized to $[0,1]$).
*   $\Sigma_i \in \mathbb{R}^{3 \times 3}$: Covariance matrix, decomposed into scaling vector $s_i \in \mathbb{R}^3$ and rotation quaternion $q_i \in \mathbb{R}^4$ such that $\Sigma_i = R(q_i) S(s_i) S(s_i)^T R(q_i)^T$.
*   $t_i \in \mathbb{R}$: Intensity (CT value), normalized to $[0,1]$.
*   Note: Opacity and Spherical Harmonic coefficients are omitted [S1].

**Numbered Steps:**

1.  **FBP-Image-Guided Initialization:**
    *   Compute the Filtered Backprojection (FBP) reconstruction $I_{FBP}$ from projection data $P$ [S1].
    *   Apply a threshold $\tau$ to exclude void regions: $\bar{I}_{FBP}(X) = I_{FBP}(X) \cdot \mathbb{I}[I_{FBP}(X) > \tau]$, where $\mathbb{I}[\cdot]$ is the indicator function [S1].
    *   Compute the gradient magnitude $\|\nabla \bar{I}_{FBP}\|$ for all voxels. Rank voxels by gradient magnitude.
    *   Select voxel coordinates $V_i$ with medium gradient magnitude (avoiding high-gradient streaking artifacts) as initial Gaussian centers [S1].
    *   For each center $C_j$, calculate the number of neighbors $N(C_j, r)$ within a radius $r$.
    *   Initialize intensity $t_j$ proportional to the FBP intensity at $C_j$:
        $$ t_j = k_t \bar{I}_{FBP}(C_j) $$
        where $k_t$ is a coefficient [S1].
    *   Initialize scaling $\sigma_j$ based on neighbor density:
        $$ \sigma_j = k_\sigma \frac{1}{N(C_j, r)} $$
        where $k_\sigma$ is a coefficient [S1].
    *   Set initial scaling vector $s_j = [\sigma_j, \sigma_j, \sigma_j]$ (isotropic) and rotation quaternion $q_j = [1, 0, 0, 0]$ (identity) [S1].
    *   Normalize center coordinates $\mu_j$ by dividing by the maximum range to map to $[0,1]$. Normalize intensity $t_j$ to $[0,1]$ [S1].

2.  **Differentiable CT Projector (Volumetric Rendering):**
    *   Discretize the volume into a voxel grid.
    *   For a target point $X$ in the volume, compute the contribution of all Gaussians $i$ whose $3\sigma$ confidence interval contains $X$ (i.e., $\|X - \mu_i\| \le 3\sigma_i$) [S1].
    *   The intensity at point $X$ is the sum of contributions:
        $$ I(X | \{G_i\}) = \sum_{i: \|X - \mu_i\| \le 3\sigma_i} t_i \exp \left( -\frac{1}{2} (X - \mu_i)^T \Sigma_i^{-1} (X - \mu_i) \right) $$
        [S1].
    *   Project the 3D volume onto 2D detector planes using the known CT geometry (source and detector positions) to generate rendered projections [S1].

3.  **Two-Stage Optimization:**
    *   **Stage 1 (Low-Resolution):** Sample a grid at $1/2 \times 1/2 \times 1/2$ of the original volume resolution. Optimize Gaussian parameters to minimize the discrepancy between rendered projections and ground truth projections [S1].
    *   **Stage 2 (Full-Resolution):** Sample the full voxel grid. Refine parameters for high-fidelity reconstruction [S1].
    *   **Loss Function:** Minimize projection loss (e.g., MSE) between rendered projections and acquired projections $P$ [S1].
    *   **Activation Functions:** Apply sigmoid activation to intensity $t$ to keep values in $[0,1]$. Apply exponential activation to scaling $s$ to ensure positive values [S1].
    *   **Adaptive Density Control:** During optimization, add or remove Gaussians based on view-space positional gradients and an opacity threshold (adapted from standard GS, though opacity is not stored, the gradient magnitude serves as a proxy for detail requirement) [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def initialize_gaussians(projection_data, threshold_tau, k_sigma, k_t, neighbor_radius):
    """
    Step 1: FBP-image-guided Gaussian initialization.
    """
    # 1.1 Reconstruct initial volume using FBP
    I_fbp = filtered_backprojection(projection_data)
    
    # 1.2 Threshold to remove void
    mask = (I_fbp > threshold_tau).float()
    I_bar_fbp = I_fbp * mask
    
    # 1.3 Compute gradients and rank voxels
    # Gradient magnitude calculation (e.g., using Sobel or finite differences)
    grad_mag = compute_gradient_magnitude(I_bar_fbp)
    
    # Flatten and sort voxels by gradient magnitude
    flat_grad = grad_mag.flatten()
    sorted_indices = torch.argsort(flat_grad)
    
    # Select medium gradient voxels (heuristic: e.g., percentiles 20-80)
    # Avoiding high gradients reduces streaking artifact influence
    num_voxels = len(sorted_indices)
    start_idx = int(num_voxels * 0.2)
    end_idx = int(num_voxels * 0.8)
    selected_flat_indices = sorted_indices[start_idx:end_idx]
    
    # Convert flat indices to 3D coordinates
    voxel_coords_3d = flat_indices_to_3d(selected_flat_indices, I_fbp.shape)
    
    # 1.4 Initialize parameters
    centers = voxel_coords_3d.float()
    
    # Normalize centers to [0, 1]
    max_range = torch.max(centers, dim=0)[0]
    centers_normalized = centers / max_range
    
    # Initialize scaling and rotation
    # Calculate neighbor count for each center to determine sigma
    neighbor_counts = compute_neighbor_counts(centers_normalized, neighbor_radius)
    
    sigmas = k_sigma / neighbor_counts
    scales = torch.stack([sigmas, sigmas, sigmas], dim=-1) # Isotropic
    rotations = torch.ones((len(centers), 4)) # Identity quaternion [1, 0, 0, 0]
    
    # Initialize intensity
    # Sample I_bar_fbp at center locations
    intensities_raw = sample_volume_at_coords(I_bar_fbp, centers)
    intensities = k_t * intensities_raw
    
    # Normalize intensity to [0, 1]
    max_intensity = torch.max(intensities)
    if max_intensity > 0:
        intensities_normalized = intensities / max_intensity
    else:
        intensities_normalized = intensities
        
    return {
        'mu': centers_normalized,
        's': scales,
        'q': rotations,
        't': intensities_normalized
    }

def render_projection(gaussians, camera_params, volume_grid_shape):
    """
    Step 2: Differentiable CT Projector (Volumetric Rendering).
    Uses a 3D grid to integrate Gaussian contributions.
    """
    mu = gaussians['mu']
    s = gaussians['s']
    q = gaussians['q']
    t = gaussians['t']
    
    # Decompose covariance: Sigma = R S S^T R^T
    R = quaternion_to_rotation_matrix(q)
    S = torch.diag_embed(s)
    Sigma = R @ S @ S.transpose(-2, -1) @ R.transpose(-2, -1)
    Sigma_inv = torch.inverse(Sigma)
    
    # Create voxel grid coordinates
    # Shape: (H, W, D, 3)
    grid_coords = create_voxel_grid(volume_grid_shape)
    grid_flat = grid_coords.view(-1, 3)
    
    # Initialize volume intensity
    volume_intensity = torch.zeros(grid_flat.shape[0], device=mu.device)
    
    # Iterate over Gaussians (optimized via CUDA kernel in practice)
    # For each Gaussian, find voxels within 3-sigma distance
    for i in range(len(mu)):
        center = mu[i]
        cov_inv = Sigma_inv[i]
        intensity_val = t[i]
        
        # Compute squared Mahalanobis distance for all voxels
        # dist_sq = (x - c)^T Sigma^-1 (x - c)
        diff = grid_flat - center
        dist_sq = torch.sum(diff @ cov_inv * diff, dim=1)
        
        # Mask for 3-sigma region (approximate: dist_sq <= 9 for isotropic, 
        # but strictly should be eigenvalue dependent. 
        # Source uses ||x - mu|| <= 3*sigma as a simplified bound or 
        # implies the kernel handles the exact Gaussian decay).
        # Note: Source Eq 7 uses ||x - mu|| <= 3*sigma. 
        # Assuming isotropic approximation for the mask logic in the loop for clarity,
        # though the kernel likely uses the exact Mahalanobis distance or bounding box.
        # We follow the source's explicit condition:
        sigma_val = s[i, 0] # Assuming isotropic for the bound check as per source text
        mask = torch.norm(diff, dim=1) <= (3.0 * sigma_val)
        
        # Compute Gaussian contribution
        # G(x) = exp(-0.5 * (x-mu)^T Sigma^-1 (x-mu))
        # Note: The source Eq 4 includes the intensity t_i outside the exp.
        # The exponent uses the full covariance inverse.
        exponent = -0.5 * dist_sq
        gaussian_vals = torch.exp(exponent)
        
        # Add contribution
        volume_intensity[mask] += intensity_val * gaussian_vals[mask]
        
    # Reshape to 3D volume
    volume_3d = volume_intensity.view(volume_grid_shape)
    
    # Project to 2D using CT geometry (ray casting / integration)
    # This step depends on specific CT projector implementation (e.g., Siddons algorithm)
    # which is differentiable.
    projection = ct_projector(volume_3d, camera_params)
    
    return projection

def optimize(gaussians, projection_data, num_stages=2):
    """
    Step 3: Two-Stage Optimization.
    """
    optimizer = torch.optim.Adam(gaussians.values(), lr=0.01)
    
    for stage in range(num_stages):
        if stage == 0:
            # Low-resolution: Downsample volume or grid
            grid_shape = tuple(s // 2 for s in original_volume_shape)
        else:
            grid_shape = original_volume_shape
            
        for iteration in range(num_iterations_per_stage):
            optimizer.zero_grad()
            
            # Render current view
            rendered_proj = render_projection(gaussians, camera_params, grid_shape)
            
            # Compute Loss (Projection Loss)
            # Ground truth is the acquired projection data
            loss = torch.nn.functional.mse_loss(rendered_proj, projection_data)
            
            # Backpropagate
            loss.backward()
            
            # Update parameters
            optimizer.step()
            
            # Apply activation functions to maintain constraints
            # Sigmoid for intensity
            gaussians['t'] = torch.sigmoid(gaussians['t'])
            # Exponential for scaling
            gaussians['s'] = torch.exp(gaussians['s'])
            
            # Adaptive Density Control (simplified)
            # Check gradients and add/remove Gaussians if needed
            gaussians = adaptive_density_control(gaussians)
            
    return gaussians
```

**Parameters:**
*   $\tau$: Threshold for FBP image void exclusion. Value not given in the sources.
*   $k_t, k_\sigma$: Coefficients for intensity and scaling initialization. Values not given in the sources.
*   $r$: Neighbor radius for density estimation. Value not given in the sources.
*   Resolution: Original volume size (e.g., $256 \times 256 \times 256$). Low-res stage uses $1/2 \times 1/2 \times 1/2$ [S1].

**Edge Cases and Pitfalls:**
*   **Affine Approximation Error:** Standard 3D Gaussian Splatting uses an affine approximation to project 3D Gaussians to 2D, which introduces errors unacceptable for precise CT value recovery [S1]. This method avoids this by using volumetric rendering [S1].
*   **PyTorch Slicing Limitations:** Standard PyTorch slicing requires patches of common size, which is inefficient for varying $3\sigma$ confidence intervals. A customized CUDA kernel is required for acceleration [S1].
*   **Spectral Bias:** Unlike INRs, this method does not suffer from spectral bias, allowing better reconstruction of sharp boundaries [S1].

## Operating characteristics
*   **Time Complexity:** Faster than Instant-NGP/INR methods. The customized CUDA kernel achieves a 10x speedup and order-of-magnitude memory reduction compared to standard PyTorch slicing for Gaussian contributions [S1].
*   **Memory Complexity:** Reduced by an order of magnitude compared to naive PyTorch implementations due to the $3\sigma$ bounding box optimization [S1].
*   **Performance:** Outperforms state-of-the-art INR-based methods in reconstruction accuracy and convergence speed on diverse datasets (chest, abdomen, foot, jaw, head, vertebrae) [S1].
*   **Strengths:** Precise CT value recovery (critical for dose calculation), real-time physical simulation capability, robustness to sparse-view artifacts, no spectral bias [S1].
*   **Limitations:** Requires a differentiable CT projector implementation; volumetric rendering is inherently slower than 2D splatting (mitigated by CUDA optimization) [S1].
*   **When to use:** Sparse-view CT reconstruction where radiation dose must be minimized, and precise quantitative CT values are required for clinical applications [S1].
*   **When not to use:** Dense-view CT where standard FBP or iterative reconstruction is sufficient and faster; applications not requiring precise physical intensity values [S1].

## Background
*   **Origin:** Proposed by Yingtai Li et al. in "3DGR-CT: Sparse-view CT reconstruction with a 3D Gaussian representation" [S1].
*   **History:** Inspired by the success of 3D Gaussian Splatting in novel view synthesis (Kerbl et al., 2023) [S1]. It addresses limitations of Implicit Neural Representations (INRs) in CT, such as spectral bias and high computational costs, and adapts 3D Gaussians for volumetric physics rather than 2D appearance [S1].
*   **Usage:** Sparse-view CT reconstruction, enabling real-time physical simulation, surgical planning, and personalized treatment strategies [S1].

## Sources
[S1] 3DGR-CT: Sparse-view CT reconstruction with a 3D Gaussian representation, https://doi.org/10.1016/j.media.2025.103585, fetched 2026-10-26
