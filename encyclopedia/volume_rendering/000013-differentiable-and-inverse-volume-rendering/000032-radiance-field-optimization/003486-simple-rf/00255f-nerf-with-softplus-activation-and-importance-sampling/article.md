# NeRF with Softplus Activation and Importance Sampling

Also called: Original NeRF architecture with proposed reparameterization

## Overview
This method adapts the original NeRF architecture [S1, unofficial] to enable stable optimization via a reparameterized volume sampling technique. It replaces the standard ReLU density activation with Softplus to ensure non-zero gradients and employs importance-sampling reparameterization for color estimation [S1, unofficial]. It differs from full NeRF by using uniform weights for samples rather than a second "fine" network for re-weighting [S1, unofficial].

## How it works
**Inputs:**
*   A set of training images with known camera poses.
*   A 3D grid of density knots (typically 256 knots per ray, slightly perturbed from uniform) [S1, unofficial].
*   Hyperparameters: $\beta = 10$ for Softplus, $k$ samples for inference (typically 32 for training, variable for inference) [S1, unofficial].

**Outputs:**
*   Rendered RGB images for novel views.
*   Optimized grid parameters.

**Data Structures:**
*   **Density Grid:** A piecewise linear approximation of volume density $\sigma$ along rays. Constructed by perturbing a uniform grid with 256 knots [S1, unofficial].
*   **Training Batch:** Random subset of 64 training images, with 64 rays sampled per image [S1, unofficial].

**Algorithm Steps:**

1.  **Ray Sampling:** For each ray in the training batch, sample points along the ray segment. The positions are derived from the perturbed uniform grid with 256 knots [S1, unofficial].
2.  **Density Prediction:** Query the network (or grid) for volume density $\sigma$ at each sample point. Apply the Softplus activation function to ensure non-zero gradients:
    $$ \sigma_{final} = \frac{1}{\beta} \ln(1 + e^{\beta \sigma_{raw}}) $$
    where $\beta = 10$ [S1, unofficial].
3.  **Transmittance Calculation:** Compute transmittance $T_i$ for each sample $i$ along the ray:
    $$ T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right) $$
    where $\delta_j$ is the distance between sample $j$ and $j+1$ [S1, unofficial].
4.  **Color Estimation (Importance-Sampling Reparameterization):** Estimate the expected color $C$ using $k$ samples. Unlike full NeRF, this method uses uniform weights for the samples in the color estimate [S1, unofficial]:
    $$ C = \frac{1}{k} \sum_{i=1}^{k} T_i (1 - \exp(-\sigma_i \delta_i)) c_i $$
    where $c_i$ is the predicted color at sample $i$ [S1, unofficial].
5.  **Loss Calculation:** Compute the loss between the rendered color and the ground truth image color.
6.  **Optimization:** Update the grid parameters using gradient descent. The Softplus activation ensures gradients flow even when $\sigma$ is small [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import torch
import torch.nn.functional as F

def softplus_density(sigma_raw, beta=10.0):
    """Apply Softplus activation to density with temperature beta."""
    return (1.0 / beta) * torch.log(1.0 + torch.exp(beta * sigma_raw))

def render_ray_with_importance_sampling(
    sigma_raw, 
    c, 
    deltas, 
    k_samples, 
    beta=10.0
):
    """
    Render a single ray using importance-sampling reparameterization.
    
    Args:
        sigma_raw: Raw density predictions for k_samples points.
        c: Color predictions for k_samples points.
        deltas: Distances between consecutive sample points.
        k_samples: Number of samples to use (k in Equation 10).
        beta: Softplus temperature parameter.
        
    Returns:
        Rendered color for the ray.
    """
    # Apply Softplus activation to ensure non-zero gradients
    sigma = softplus_density(sigma_raw, beta)
    
    # Compute transmittance
    # T_i = exp(-sum_{j=1}^{i-1} sigma_j * delta_j)
    # We compute cumulative sum of sigma * delta
    alpha_deltas = sigma * deltas
    cumulative_alpha = torch.cumsum(alpha_deltas, dim=0)
    
    # Add a small value to avoid log(0)
    cumulative_alpha = F.pad(cumulative_alpha, (1, 0), value=0)
    
    # Transmittance T_i = exp(-cumulative_alpha_{i-1})
    # Note: cumulative_alpha[i] includes sigma[i]*delta[i]
    # So T_i corresponds to exp(-cumulative_alpha[i-1])
    T = torch.exp(-cumulative_alpha[:-1])
    
    # Compute opacity for each sample: alpha_i = T_i * (1 - exp(-sigma_i * delta_i))
    # This is equivalent to T_i * (1 - exp(-alpha_deltas_i))
    opacity = T * (1.0 - torch.exp(-alpha_deltas))
    
    # Color estimation with uniform weights (Equation 10)
    # C = (1/k) * sum_{i=1}^{k} opacity_i * c_i
    color = torch.sum(opacity * c, dim=0) / k_samples
    
    return color

def train_step(batch_images, batch_cameras, grid, optimizer, beta=10.0, k_train=32):
    """
    One training step.
    
    Args:
        batch_images: Tensor of shape (64, H, W, 3)
        batch_cameras: Camera parameters for the 64 rays
        grid: Density grid (256 knots per ray)
        optimizer: Optimizer for the grid
        beta: Softplus temperature
        k_train: Number of samples for training (typically 32)
    """
    optimizer.zero_grad()
    
    # Sample 64 rays per image from 64 random images
    # This part is implementation-specific to the ray sampling strategy
    # Assume we have ray origins, directions, and corresponding grid indices
    
    # For each ray, sample k_train points from the 256-knot grid
    # Perturb the uniform grid slightly as per the source
    ray_samples = sample_rays(batch_cameras, k_train)
    
    # Query grid for raw density and color
    sigma_raw, c = query_grid(grid, ray_samples)
    
    # Compute deltas (distances between samples)
    deltas = compute_deltas(ray_samples)
    
    # Render rays
    rendered_colors = []
    for i in range(len(ray_samples)):
        color = render_ray_with_importance_sampling(
            sigma_raw[i], c[i], deltas[i], k_train, beta
        )
        rendered_colors.append(color)
    
    rendered_colors = torch.stack(rendered_colors)
    
    # Compute loss (e.g., MSE)
    loss = F.mse_loss(rendered_colors, batch_images)
    
    # Backpropagate
    loss.backward()
    optimizer.step()
    
    return loss.item()
```

**Edge Cases and Pitfalls:**
*   **Variance:** The stochastic estimate variance prevents the model from finding the fine optimum, resulting in slightly worse quantitative results than NeRF without hierarchical sampling, despite similar grid density [S1, unofficial].
*   **Noise Artifacts:** Noise artifacts are present in renders when the number of inference samples $k$ is low (e.g., $k=1, 2, 4$). Artifacts only disappear for $k \ge 8$ [S1, unofficial].
*   **Saturation:** Rendering quality metrics (PSNR, SSIM, LPIPS) saturate at approximately 16 samples for the Lego scene [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Rendering speed increases with the number of samples $k$. On an NVIDIA V100, rendering time for the Lego scene (800x800) ranges from 20.18s ($k=1$) to 25.74s ($k=64$) [S1, unofficial].
*   **Memory Complexity:** Memory usage is relatively stable, ranging from 4.189Gb ($k=1$) to 4.209Gb ($k=64$) [S1, unofficial].
*   **Performance:** On the Lego scene (800x800, NVIDIA V100), the model with 32 samples achieved PSNR 30.72, SSIM 0.946, LPIPS 0.0801 [S1, unofficial]. With 1 sample: PSNR 25.67, SSIM 0.870, LPIPS 0.1768 [S1, unofficial]. With 64 samples: PSNR 30.72, SSIM 0.947, LPIPS 0.0794 [S1, unofficial].
*   **Strengths:** Slightly improves rendering time and memory footprint compared to full NeRF [S1, unofficial]. Softplus activation ensures non-zero gradients [S1, unofficial].
*   **Limitations:** Slightly worse quantitative results than NeRF without hierarchical sampling [S1, unofficial]. Noise artifacts at low sample counts [S1, unofficial].

## Background
This method is based on the original NeRF architecture [S1, unofficial]. It was proposed in "Differentiable Rendering With Reparameterized Volume Sampling" [S1, unofficial]. The method uses the original NeRF's architecture and hyperparameters without modifications, except for the output activation of field $\sigma$ and the color estimation technique [S1, unofficial].

## Sources
[S1] Differentiable Rendering With Reparameterized Volume Sampling, https://openreview.net/pdf?id=zHSaBQtj-l, fetched 2026-10-29 (unofficial)
