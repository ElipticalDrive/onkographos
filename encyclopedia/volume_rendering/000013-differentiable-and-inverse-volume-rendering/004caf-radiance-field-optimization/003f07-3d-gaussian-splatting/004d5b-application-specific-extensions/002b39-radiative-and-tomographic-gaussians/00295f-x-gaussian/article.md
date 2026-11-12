# X-Gaussian

Also called: Radiative Gaussian Splatting for X-Ray, X-Ray Gaussian Splatting

## Overview
X-Gaussian is a 3D Gaussian Splatting (3DGS) variant designed for efficient X-ray novel view synthesis (NVS) and sparse-view computed tomography (CT) reconstruction [S1]. It replaces the standard view-dependent RGB color model with a view-independent radiative intensity model, enabling the generation of synthetic X-ray projections that serve as high-quality priors for learning-free CT reconstruction algorithms like FDK, SART, and ASD-POCS [S1]. Unlike standard 3DGS, which optimizes for visual appearance in visible light, X-Gaussian optimizes for physical radiation attenuation, resulting in faster convergence, cleaner backgrounds, and significantly higher inference speeds compared to NeRF-based X-ray methods [S1].

## How it works
X-Gaussian retains the core 3DGS primitive representation: a set of $M$ anisotropic 3D Gaussian kernels $\mathcal{G} = \{G_i\}_{i=1}^M$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md), [1875-volumetric-gaussian-splatting-avis.md]. Each kernel $i$ is defined by:
*   $\mathbf{p}_i \in \mathbb{R}^3$: Position (mean) [S1].
*   $\mathbf{R}_i \in \mathbb{R}^{3 \times 3}$: Rotation matrix [S1].
*   $\mathbf{S}_i \in \mathbb{R}^{3 \times 3}$: Scale matrix [S1].
*   The covariance matrix is $\mathbf{\Sigma}_i = \mathbf{R}_i \mathbf{S}_i \mathbf{S}_i^\top \mathbf{R}_i^\top$ [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).

The rendering pipeline uses Differentiable Radiative Rasterization (DRR) [S1]. Geometric projection remains consistent with standard 3DGS: 3D Gaussians are transformed into ray space, and the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ is computed using the Jacobian $J$ of the affine approximation of the projection and the rotation part $W$ of the extrinsic matrix:
$$ \mathbf{\hat{\Sigma}}_i = J W \mathbf{\Sigma}_i W^T J^T $$
[Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md)

However, the attribute representation diverges from standard 3DGS. Instead of view-dependent color $\mathbf{c}_i(\mathbf{d})$ [G], X-Gaussian uses a view-independent scalar attribute representing radiation intensity (or radiodensity) [S1].

### Inputs and Outputs
*   **Inputs**: X-ray projection images from various angles; X-ray scanner parameters (intrinsic and extrinsic matrices) [S1].
*   **Outputs**: A set of 3D Gaussians $\mathcal{G}$ with radiative intensity attributes; synthetic X-ray projections for novel views [S1].

### Data Structures
*   **Gaussian Point Cloud**: List of $M$ Gaussians. Each Gaussian stores $\mathbf{p}_i, \mathbf{R}_i, \mathbf{S}_i$ and a scalar radiative intensity value $\alpha_i$ (or similar radiative attribute) [S1].
*   **Scanner Parameters**: Intrinsic matrix $\mathbf{K}$ and extrinsic matrix $\mathbf{E}$ defining the X-ray source and detector geometry [S1].

### Numbered Steps

1.  **Initialization (ACUI Strategy)**:
    *   Compute the extrinsic and intrinsic matrices from the X-ray scanner parameters [S1].
    *   Define a cuboid volume enclosing the scanned object [S1].
    *   Uniformly sample center points $\mathbf{p}_i$ for the 3D Gaussians within this cuboid [S1].
    *   Initialize rotation $\mathbf{R}_i$ and scale $\mathbf{S}_i$ (standard 3DGS initialization practices apply, though specific initial values are not detailed in the source, [G] assumes standard isotropic initialization or random rotation) [S1].
    *   Initialize radiative intensity attributes $\alpha_i$ [S1].

2.  **Differentiable Radiative Rasterization (DRR)**:
    *   For each training view, project the 3D Gaussians to 2D screen space using the camera matrices [S1].
    *   Compute the projected 2D covariance $\mathbf{\hat{\Sigma}}_i$ for each Gaussian [Radiative Gaussian Splatting](3040-radiative-gaussian-splatting.md).
    *   Sort Gaussians by depth [G] (standard alpha blending requirement) [G].
    *   For each pixel, accumulate the radiative intensity from front to back. The contribution of Gaussian $i$ to pixel $j$ is determined by its 2D Gaussian density at the pixel location and its radiative attribute $\alpha_i$ [S1].
    *   The rendered intensity $I_j$ for pixel $j$ is computed as:
        $$ I_j = \sum_{i \in \mathcal{G}} \alpha_i \cdot c_i(\mathbf{x}_j) $$
        where $c_i(\mathbf{x}_j)$ is the 2D Gaussian kernel value evaluated at pixel coordinate $\mathbf{x}_j$ [G]. Note: Standard 3DGS alpha blending is modified to model radiative transmission/attenuation rather than opacity blending, but the source describes it as "radiative rasterization" without providing the specific attenuation formula (e.g., Beer-Lambert law integration). [G] assumes a standard volumetric rendering integral adapted for Gaussians, where the scalar attribute represents the density contributing to attenuation.
    *   This step is implemented via a GPU-friendly CUDA kernel [S1].

3.  **Optimization**:
    *   Compute the loss between the rendered X-ray projections and the ground-truth X-ray images [S1].
    *   Exclude the influence of view direction when fitting radiation intensity, as X-ray attenuation is view-independent for a given ray path [S1].
    *   Update $\mathbf{p}_i, \mathbf{R}_i, \mathbf{S}_i, \alpha_i$ using gradient descent [S1].
    *   Perform densification/culling of Gaussians based on gradient magnitude (standard 3DGS strategy) [G] to avoid redundant Gaussians [S1].

### Parameters
*   **Training Iterations**: Convergence is analyzed at 1000, 20000 iterations, and 20s, 60s, 180s of training time [S1].
*   **Number of Views**: Performance improves as the number of training views increases, leading to a decrease in mean covariance (coarse-to-fine adaptation) [S1].

### Edge Cases and Pitfalls
*   **Implementation Complexity**: The method requires CUDA (C++) implementation for the rasterization kernel, making it harder to debug and less interpretable than PyTorch-based methods [S1].
*   **Degenerate Input**: Requires accurate scanner parameters for initialization; incorrect intrinsic/extrinsic matrices will lead to misaligned Gaussians [S1].
*   **Precision**: The method avoids noisy point clouds and redundant Gaussians better than standard 3DGS, but precision depends on the number of training views [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import numpy as np

def initialize_gaussians_acui(scanner_params, cuboid_bounds, num_gaussians):
    """
    Initialize 3D Gaussians using the ACUI strategy.
    :param scanner_params: Dictionary containing intrinsic (K) and extrinsic (E) matrices.
    :param cuboid_bounds: Tuple (min_x, max_x, min_y, max_y, min_z, max_z) defining the volume.
    :param num_gaussians: Number of Gaussians to initialize.
    :return: Dictionary of Gaussian attributes.
    """
    # Extract matrices
    K = scanner_params['intrinsic']
    E = scanner_params['extrinsic']
    
    # Uniform sampling within cuboid
    min_x, max_x, min_y, max_y, min_z, max_z = cuboid_bounds
    positions = np.random.uniform(
        low=[min_x, min_y, min_z], 
        high=[max_x, max_y, max_z], 
        size=(num_gaussians, 3)
    )
    
    # Initialize rotation (identity or random) and scale (small isotropic)
    rotations = np.tile(np.eye(3), (num_gaussians, 1, 1)) # Shape: (M, 3, 3)
    scales = np.ones((num_gaussians, 3)) * 0.1 # [G] Default small scale
    
    # Initialize radiative intensity (scalar)
    intensities = np.ones(num_gaussians)
    
    return {
        'positions': torch.tensor(positions, dtype=torch.float32),
        'rotations': torch.tensor(rotations, dtype=torch.float32),
        'scales': torch.tensor(scales, dtype=torch.float32),
        'intensities': torch.tensor(intensities, dtype=torch.float32),
        'K': K,
        'E': E
    }

def compute_covariance(rotations, scales):
    """
    Compute 3D covariance matrix from rotation and scale.
    Sigma = R * S * S^T * R^T
    """
    # rotations: (M, 3, 3), scales: (M, 3)
    S = torch.diag_embed(scales) # (M, 3, 3)
    S_S_T = torch.bmm(S, S.transpose(1, 2)) # (M, 3, 3)
    Sigma = torch.bmm(rotations, torch.bmm(S_S_T, rotations.transpose(1, 2))) # (M, 3, 3)
    return Sigma

def project_gaussians(positions, rotations, scales, K, E):
    """
    Project 3D Gaussians to 2D screen space.
    Returns projected 2D covariance and screen positions.
    """
    # Transform positions to camera space
    # E is usually [R|t], so position_cam = R * position + t
    R_cam = E[:, :3, :3]
    t_cam = E[:, :3, 3]
    positions_cam = torch.bmm(R_cam, positions.unsqueeze(2)).squeeze(2) + t_cam
    
    # Compute 3D covariance in world space
    Sigma_3d = compute_covariance(rotations, scales)
    
    # Transform covariance to camera space
    Sigma_cam = torch.bmm(R_cam, torch.bmm(Sigma_3d, R_cam.transpose(1, 2)))
    
    # Project to 2D using Jacobian J and intrinsic K
    # Standard 3DGS projection: Sigma_2d = J * W * Sigma_3d * W^T * J^T
    # Here W is part of extrinsic, J is from projection
    # For simplicity, assuming standard perspective projection logic
    # [G] Detailed Jacobian computation omitted as per standard 3DGS implementation
    # This step is typically handled by the CUDA kernel
    pass

def differentiable_radiative_rasterization(gaussians, target_projections):
    """
    Render X-ray projections using DRR and compute loss.
    """
    # 1. Project Gaussians
    # 2. Sort by depth
    # 3. Accumulate radiative intensity
    # 4. Compute loss against target_projections
    
    # Placeholder for CUDA kernel call
    # rendered_projections = cuda_drr_kernel(gaussians)
    
    # Loss computation (e.g., L1 or L2)
    # loss = torch.mean((rendered_projections - target_projections) ** 2)
    pass

def train_x_gaussian(scanner_params, cuboid_bounds, training_data, num_iterations=20000):
    """
    Main training loop for X-Gaussian.
    """
    # Initialize
    gaussians = initialize_gaussians_acui(scanner_params, cuboid_bounds, num_gaussians=10000)
    
    for i in range(num_iterations):
        # Sample a random view from training_data
        view = sample_view(training_data)
        
        # Forward pass: Render
        # rendered = differentiable_radiative_rasterization(gaussians, view)
        
        # Backward pass: Update
        # loss = compute_loss(rendered, view.target)
        # loss.backward()
        
        # Densification/Culling (Standard 3DGS strategy)
        # if i % 100 == 0:
        #     densify_and_cull(gaussians)
        
        # Optimizer step
        # optimizer.step()
        pass
```

## Operating characteristics
*   **Time Complexity**: Inference is 73× faster than SOTA NeRF-based methods and 192× faster than RGB NeRF-based methods like TensoRF [S1]. Training time is less than 15% of that required by comparable NeRF methods [S1].
*   **Memory Complexity**: Not explicitly given, but the method avoids redundant Gaussians, leading to a more compact representation than standard 3DGS [S1].
*   **Performance**: Outperforms SOTA methods by over 6.5 dB in PSNR [S1]. When used to create novel-view projections for CT reconstruction (FDK, SART, ASD-POCS), it yields PSNR improvements of 15.19, 13.01, and 13.53 dB respectively, which are 1.32, 2.41, and 2.65 dB higher than using NAF [S1].
*   **Strengths**: Faster convergence (basic shape formed by 1000th iteration), cleaner backgrounds, clearer structural details (e.g., ribs, blood vessels) even with sparse views, and no need for SfM initialization [S1].
*   **Limitations**: Requires CUDA implementation, making debugging and interpretation harder than Python-based PyTorch methods [S1]. Performance depends on the number of training views; more views lead to finer covariance and better representation of small structures like tumors [S1].

## Background
X-Gaussian was proposed as the first 3DGS-based framework for X-ray novel view synthesis [S1]. It addresses the limitations of NeRF-based X-ray methods, which suffer from slow training and inference speeds, and standard 3DGS, which is designed for visible light and view-dependent color [S1]. The method leverages the physical properties of X-ray imaging, where radiation intensity is view-independent, to simplify the optimization problem and improve efficiency [S1]. It is particularly useful for sparse-view CT reconstruction, where generating high-quality synthetic projections can significantly enhance the performance of learning-free reconstruction algorithms [S1].

## Sources
[S1] Radiative Gaussian Splatting for Efficient X-Ray Novel View Synthesis, https://doi.org/10.1007/978-3-031-73232-4_16, fetched 2026-10-26
