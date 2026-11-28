# Direct Depth Supervision

Also called: Direct supervision, L1 depth loss, MSE depth loss

## Overview
Direct Depth Supervision is a regularization technique in differentiable volume rendering that minimizes the pixel-wise discrepancy between the depth map rendered by a neural radiance field and a provided ground truth depth prior [S4]. It directly penalizes errors in the expected depth value, distinguishing it from indirect geometric methods that enforce consistency through other signals [S4].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A batch of $N_r$ camera rays $\{r_i\}_{i=1}^{N_r}$, where each ray $r_i$ is defined by origin $o_i$ and direction $d_i$ [G]. Ground truth depth values $\{D(r_i)\}_{i=1}^{N_r}$ corresponding to these rays [S4].
*   **Outputs:** A scalar loss value $\mathcal{L}_d$ used for backpropagation to update network parameters $\theta$ (density $\sigma$ and color $\mathbf{c}$) [G].

**Data Structures:**
*   **Rendered Depth:** For each ray $r_i$, the rendered depth $\hat{D}(r_i)$ is computed as the expected distance along the ray using the standard volume rendering equation [G]:
    $$ \hat{D}(r_i) = \sum_{k=1}^N w_{i,k} t_{i,k} $$
    where $t_{i,k}$ is the sample depth along ray $i$ at index $k$, and $w_{i,k}$ is the rendering weight:
    $$ w_{i,k} = T_{i,k} (1 - \exp(-\sigma_{i,k} \delta_{i,k})) $$
    with accumulated transmittance $T_{i,k} = \prod_{j=1}^{k-1} (1 - \rho_{i,j})$ and opacity $\rho_{i,k} = 1 - \exp(-\sigma_{i,k} \delta_{i,k})$ [G].

**Algorithm Steps:**
1.  Sample a batch of $N_r$ rays from the training camera poses [S4].
2.  For each ray $r_i$, query the neural network for density $\sigma$ and color $\mathbf{c}$ at sample points $t_{i,k}$ [G].
3.  Compute rendering weights $w_{i,k}$ and accumulated transmittance $T_{i,k}$ [G].
4.  Calculate the rendered depth $\hat{D}(r_i)$ for each ray [G].
5.  Retrieve the ground truth depth $D(r_i)$ for each ray from the depth prior [S4].
6.  Compute the loss using either Mean Squared Error (MSE) or L1 norm [S4].

**Formulas:**
The loss function $\mathcal{L}_d$ is defined as:

For MSE Loss:
$$ \mathcal{L}_d^{\text{MSE}} = \frac{1}{N_r} \sum_{i=1}^{N_r} \| D(r_i) - \hat{D}(r_i) \|^2 \quad [S4] $$

For L1 Loss:
$$ \mathcal{L}_d^{\text{L1}} = \frac{1}{N_r} \sum_{i=1}^{N_r} | D(r_i) - \hat{D}(r_i) | \quad [S4] $$

where $D(r_i)$ is the ground truth depth and $\hat{D}(r_i)$ is the rendered depth for ray $r_i$ [S4]. Variations exist: some implementations use the L1 norm over sparse LiDAR projections $\|D - D_{\text{LiDAR}}\|_1$ [S1, unofficial], while others normalize by the number of pixels with valid depth $S_1$ as $\frac{1}{S_1} \sum |\hat{D}_s - D_s|$ [S2, unofficial].

**Parameters:**
*   $N_r$: Batch size of rays. Typical values are not specified in the sources.
*   Loss Type: Either MSE or L1. Both are included in experimental setups [S4].

**Edge Cases and Pitfalls:**
*   **Depth Prior Quality:** The method relies on the accuracy of the ground truth depth prior [G].
*   **Ambiguity:** In regions with low texture or transparency, the rendered depth may be ambiguous, leading to high loss values even if the color is correct [G].
*   **Precision:** Standard floating-point precision is assumed for the summation in the depth calculation [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def direct_depth_supervision(rays, gt_depths, near, far, num_samples):
    """
    Computes direct depth supervision loss.
    
    Args:
        rays: List of tuples (origin, direction).
        gt_depths: Ground truth depth values for each ray.
        near: Near plane distance.
        far: Far plane distance.
        num_samples: Number of samples per ray.
        
    Returns:
        loss: Scalar loss value.
    """
    rendered_depths = []
    
    for (origin, direction), gt in zip(rays, gt_depths):
        # Sample points along the ray
        t_vals = [near + (far - near) * (i + 0.5) / num_samples for i in range(num_samples)]
        
        # Query network for density (standard MLP forward pass)
        sigma_vals = [network_forward(origin + t * direction) for t in t_vals]
        
        # Compute rendering weights and transmittance
        weights = []
        transmittance = 1.0
        for i, sigma in enumerate(sigma_vals):
            delta = t_vals[i+1] - t_vals[i] if i < len(t_vals) - 1 else (far - t_vals[-1])
            opacity = 1.0 - math.exp(-sigma * delta)
            weight = transmittance * opacity
            weights.append(weight)
            transmittance *= (1.0 - opacity)
            
        # Compute rendered depth
        rendered_depth = sum(w * t for w, t in zip(weights, t_vals))
        rendered_depths.append(rendered_depth)
        
    # Compute L1 loss (MSE can be used similarly)
    loss = sum(abs(gt - pred) for gt, pred in zip(gt_depths, rendered_depths)) / len(rays)
    
    return loss

def network_forward(point):
    # Standard 2-layer MLP forward pass for density
    # In practice, this is the trained neural network evaluating density at point
    return 0.0  # Placeholder for actual network evaluation
```

## Subtopics

### Gradient-Aware Depth Supervision Loss
Extends direct depth supervision by adding gradient terms to the L1 loss on inverse depths: $\mathcal{L}_{\text{depth}} = \alpha |D_{\text{pred}} - D_{\text{gt}}| + \beta |\partial_x D_{\text{pred}} - \partial_x D_{\text{gt}}| + \beta |\partial_y D_{\text{pred}} - \partial_y D_{\text{gt}}|$ [S5]. The weights are $\alpha=20$ and $\beta=20$ [S5]. Choose this when preserving local depth structure and edges is critical [S5].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of rays $N_r$ and samples per ray $N$ [G].
*   **Memory Complexity:** Linear with respect to $N_r \times N$ to store intermediate values (weights, transmittances) for backpropagation [G].
*   **Strengths:** Directly enforces geometric accuracy; simple to implement; differentiable [G].
*   **Limitations:** Sensitive to the quality of the depth prior; may struggle with sparse or noisy depth data if not regularized [G].
*   **When to use:** When high-quality depth priors (e.g., from LiDAR or stereo matching) are available and geometric accuracy is prioritized over photometric consistency [G].
*   **When not to use:** When depth priors are unreliable or absent [G].

## Background
Direct Depth Supervision was introduced as part of the broader effort to integrate depth priors into neural radiance fields for outdoor scenes [S4]. The method leverages the differentiability of volume rendering to backpropagate gradients from depth errors to the network parameters, enabling joint optimization of color and geometry [G]. It is a foundational technique in depth-supervised neural radiance field frameworks [S4].

## Sources
[S1] DrivingVoxels: Compositional Sparse Voxel Rasterization for Dynamic Driving Scene Reconstruction, https://doi.org/10.48550/arxiv.2606.23031, fetched 2026-10-26 (unofficial)
[S2] VolRecon: Volume Rendering of Signed Ray Distance Functions for Generalizable Depth Estimation, https://openaccess.thecvf.com/content/CVPR2023/papers/Ren_VolRecon_Volume_Rendering_of_Signed_Ray_Distance_Functions_for_Generalizable_CVPR_2023_paper.pdf, fetched 2026-10-26 (unofficial)
[S3] FIRe: Fast Inverse Rendering Using Directional and Signed Distance Functions, https://www.youtube.com/watch?v=Qj0ymDlRYZs, fetched 2026-10-30
[S4] Digging into Depth Priors for Outdoor Neural Radiance Fields, https://doi.org/10.1145/3581783.3612306, fetched 2026-10-30
[S5] DepthSplat: Connecting Gaussian Splatting and Depth, https://doi.org/10.1109/cvpr52734.2025.01534, fetched 2026-10-26
