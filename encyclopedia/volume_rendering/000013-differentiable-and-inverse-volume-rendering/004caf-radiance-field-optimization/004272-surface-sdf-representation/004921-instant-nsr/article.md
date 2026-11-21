# Instant-NSR

Also called: Neural Surface Reconstruction with Hash

## Overview
Instant-NSR is a neural surface reconstruction method that accelerates training using Hash encoding while improving geometry quality by incorporating a Signed Distance Function (SDF) representation during optimization [S1]. It builds on [Radiance-field optimization](44-radiance-field-optimization.md) and [Surface-SDF Representation](10778-surface-sdf-representation.md) by adapting the NeRF architecture to predict SDF values instead of density, and employs a Truncated SDF (TSDF) formulation to ensure numerical stability when combined with hash grids [S1]. Unlike [NeuS](6060-neus.md), which uses multi-resolution hash grids but may suffer from convergence issues with large temperature parameters, Instant-NSR explicitly truncates the SDF range to prevent gradient explosion [S1].

## How it works
**Inputs:** A set of calibrated images with known camera poses.
**Outputs:** A neural network representing the scene geometry (SDF) and appearance, capable of rendering novel views and extracting a 3D mesh.

**Data Structures:**
1.  **Hash Grid:** A multi-resolution hash table mapping spatial coordinates to feature vectors, identical to the structure in [Instant-NGP](link-to-instant-ngp). It provides the front-end feature encoding for input points [S1].
2.  **Neural Network:** A small MLP that takes the hashed features as input and outputs the SDF value $f(\mathbf{x})$ and color $\mathbf{c}$ [S1]. The architecture follows the SDF-based design of [NeuS](6060-neus.md) rather than the density-based design of [Instant-NGP](link-to-instant-ngp) [S1].

**Algorithm Steps:**

1.  **Sampling:** For each pixel in a training image, cast a ray and sample $N$ points $\mathbf{x}_i$ along the ray segment within the camera frustum.
2.  **Hash Encoding:** Encode each point $\mathbf{x}_i$ using the multi-resolution hash grid to obtain feature vectors [S1].
3.  **SDF Prediction:** Pass the hashed features through the MLP to predict the raw SDF value $f(\mathbf{x}_i)$ and color $\mathbf{c}_i$ [S1].
4.  **TSDF Truncation:** Apply a sigmoid function $\pi(\cdot)$ to the raw SDF output to truncate it to the range $[-1, 1]$. This prevents numerical overflow when the temperature parameter $b$ is large [S1]. The transformation is defined as:
    $$
    \pi(f(\mathbf{x})) = \frac{1 - e^{-bf(\mathbf{x})}}{1 + e^{-bf(\mathbf{x})}}
    $$
    where $b$ is a temperature parameter [S1]. Note: The source text presents this specific formula for $\pi(f(x))$ which maps the real line to $(-1, 1)$, effectively acting as a scaled tanh or a specific sigmoid variant for truncation [S1].
5.  **Probability Distribution:** Compute the cumulative density distribution $\phi_b$ using the truncated SDF value $\pi(f(\mathbf{x}))$. This replaces the standard NeRF/NeuS formulation to ensure stability:
    $$
    \phi_b(f(\mathbf{x})) = \frac{b e^{-b\pi(f(\mathbf{x}))}}{(1 + e^{-b\pi(f(\mathbf{x}))})^2}
    $$
    This $\phi_b$ is used to compute the weight $w(t)$ for volume rendering [S1].
6.  **Rendering:** Render the image using the standard volume rendering equation with the computed weights and colors [S1].
7.  **Loss Computation:** Calculate the total loss as a weighted sum of rendering loss and eikonal loss [S1].
8.  **Optimization:** Update the network parameters (hash grid and MLP weights) using gradient descent [S1].

**Formulas:**

*   **TSDF Mapping:**
    $$
    \pi(f(\mathbf{x})) = \frac{1 - e^{-bf(\mathbf{x})}}{1 + e^{-bf(\mathbf{x})}} \quad \text{[S1]}
    $$
*   **Weight Distribution:**
    $$
    \phi_b(f(\mathbf{x})) = \frac{b e^{-b\pi(f(\mathbf{x}))}}{(1 + e^{-b\pi(f(\mathbf{x}))})^2} \quad \text{[S1]}
    $$
*   **Eikonal Loss:** Enforces that the gradient magnitude of the SDF is 1 everywhere:
    $$
    L_{eik} = \frac{1}{M} \sum_{j=1}^{M} (\|\nabla f(\mathbf{x}_j)\|_2^2 - 1)^2
    $$
    where $M$ is the number of sampled points and $\nabla f(\mathbf{x}_j)$ is the gradient of the SDF at point $\mathbf{x}_j$ [S1].
*   **Total Loss:**
    $$
    L_{total} = \alpha L_{render} + \beta L_{eik}
    $$
    where $L_{render}$ is the rendering loss (e.g., $L_1$ or $L_2$ between rendered and ground truth images) and $\alpha, \beta$ are hyperparameters [S1].

**Parameters:**
*   $b$: Temperature parameter for the TSDF truncation and distribution. Not given in the sources.
*   $\alpha$: Weighting factor for rendering loss. Chosen via cross-validation [S1].
*   $\beta$: Weighting factor for eikonal loss. Chosen via cross-validation [S1].
*   TSDF Range: Values are effectively constrained between -1 and 1 by the sigmoid function $\pi(\cdot)$ [S1].

**Edge Cases and Pitfalls:**
*   **Numerical Instability:** In standard SDF-based methods, increasing $b$ causes $-bf(\mathbf{x})$ to become a large positive number, making $e^{-bf(\mathbf{x})}$ approach infinity, leading to network divergence [S1]. Instant-NSR solves this by using the TSDF truncation $\pi(f(\mathbf{x}))$, which bounds the exponent [S1].
*   **Convergence:** Applying raw SDF representations to hash coding frameworks can cause convergence problems. The TSDF formulation is critical for stable training in this context [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def instant_nsr_forward(ray_origins, ray_directions, hash_grid, mlp, b):
    """
    Forward pass for Instant-NSR.
    
    Args:
        ray_origins: Tensor of shape (H, W, 3), camera origins.
        ray_directions: Tensor of shape (H, W, 3), normalized ray directions.
        hash_grid: Multi-resolution hash grid structure.
        mlp: Neural network predicting SDF and color from hashed features.
        b: Temperature parameter for TSDF truncation.
        
    Returns:
        rendered_image: Tensor of shape (H, W, 3).
        sdf_values: Tensor of sampled SDF values.
    """
    # 1. Sample points along rays (standard volume rendering sampling)
    # Assume sample_points returns points p of shape (N, 3) and depths t of shape (N,)
    points, depths = sample_points_along_rays(ray_origins, ray_directions)
    
    # 2. Hash Encode points
    # hash_encode returns feature vectors for all points
    features = hash_encode(points, hash_grid)
    
    # 3. Predict SDF and Color
    # mlp takes features and returns (sdf, color)
    # sdf_raw shape: (N,), color shape: (N, 3)
    sdf_raw, color = mlp(features)
    
    # 4. TSDF Truncation
    # Apply sigmoid-like truncation to bound SDF to (-1, 1)
    # pi(f) = (1 - exp(-b*f)) / (1 + exp(-b*f))
    exp_term = torch.exp(-b * sdf_raw)
    sdf_truncated = (1 - exp_term) / (1 + exp_term)
    
    # 5. Compute Weight Distribution phi_b
    # phi_b = b * exp(-b * pi(f)) / (1 + exp(-b * pi(f)))^2
    exp_phi = torch.exp(-b * sdf_truncated)
    denominator = (1 + exp_phi) ** 2
    phi_b = (b * exp_phi) / denominator
    
    # 6. Volume Rendering
    # Compute weights w(t) using phi_b and depth intervals
    # This follows the standard NeRF/NeuS volume rendering integration
    # w_i = phi_b(f(x_i)) * exp(-sum(phi_b(f(x_j)) * delta_j))
    # Rendered color = sum(w_i * color_i)
    rendered_image = volume_render(phi_b, depths, color)
    
    return rendered_image, sdf_raw

def compute_loss(rendered_image, gt_image, sdf_raw, points):
    """
    Compute total loss for Instant-NSR.
    
    Args:
        rendered_image: Rendered image tensor.
        gt_image: Ground truth image tensor.
        sdf_raw: Raw SDF values from network.
        points: Sampled 3D points.
        
    Returns:
        total_loss: Scalar loss value.
    """
    # Rendering Loss
    l_render = torch.mean((rendered_image - gt_image) ** 2)
    
    # Eikonal Loss
    # Compute gradient of SDF w.r.t. spatial coordinates
    # grad_sdf shape: (N, 3)
    grad_sdf = compute_gradient(sdf_raw, points)
    
    # L_eik = mean((||grad_sdf||^2 - 1)^2)
    grad_norm_sq = torch.sum(grad_sdf ** 2, dim=1)
    l_eik = torch.mean((grad_norm_sq - 1) ** 2)
    
    # Total Loss
    # alpha and beta are hyperparameters chosen via cross-validation
    alpha = 1.0 # Placeholder, should be tuned
    beta = 1.0  # Placeholder, should be tuned
    total_loss = alpha * l_render + beta * l_eik
    
    return total_loss
```

## Subtopics

### NeuS
Uses SDF and volume rendering but may lack the specific TSDF truncation mechanism for hash grids found in Instant-NSR. Choose when simpler SDF integration is sufficient. [Link](6060-neus.md)

### Surface NeRF
Another SDF-based method. Compare based on specific loss formulations and encoding strategies. [Link](7385-surface-nerf.md)

### VolSDF
One of the earlier methods to introduce SDF into NeRF. Instant-NSR improves upon this by using hash encoding for speed and TSDF for stability. [Link](10715-volsdf.md)

## Operating characteristics
*   **Time Complexity:** Accelerated training compared to standard NeRF due to Hash encoding (similar to [Instant-NGP](link-to-instant-ngp)) [S1].
*   **Memory Complexity:** Depends on the size of the hash grid and MLP, similar to [Instant-NGP](link-to-instant-ngp) [S1].
*   **Performance:** Demonstrates commendable performance in novel-view synthesis and competitive 3D reconstruction results compared to commercial MVS software like Reality Capture in agricultural contexts [S1].
*   **Strengths:** Fast training due to hash encoding; improved geometry quality via SDF; numerical stability via TSDF [S1].
*   **Limitations:** Requires careful tuning of hyperparameters $\alpha$ and $\beta$ via cross-validation [S1].
*   **When to use:** When fast training and high-quality geometry reconstruction are required, particularly in scenarios where standard SDF-NeRF methods suffer from numerical instability [S1].

## Background
Instant-NSR was explored in the context of high-fidelity 3D reconstruction of plants using Neural Radiance Fields [S1]. It builds upon the principles of [Instant-NGP](link-to-instant-ngp) for speed and [NeuS](6060-neus.md) for SDF-based rendering, introducing Truncated SDF (TSDF) to resolve convergence issues inherent in applying SDF representations to hash coding frameworks [S1]. The method was tested on datasets of indoor and outdoor orchards (pitahaya, grape, orange, fig) [S1].

## Sources
[S1] High-fidelity 3D Reconstruction of Plants using Neural Radiance Field, https://arxiv.org/abs/2311.04154v1, fetched 2026-10-27
