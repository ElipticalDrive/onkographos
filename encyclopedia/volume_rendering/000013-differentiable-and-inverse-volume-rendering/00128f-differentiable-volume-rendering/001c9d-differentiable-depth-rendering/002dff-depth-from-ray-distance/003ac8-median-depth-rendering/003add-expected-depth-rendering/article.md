# Expected Depth Rendering

Also called: Expected Depth Calculation, Weighted Mean Depth

## Overview
Expected Depth Rendering computes the geometric center of mass of a ray's interaction with a volume or Gaussian mixture by calculating the weighted average of sample depths, normalized by the accumulated transmittance weight [S1, S4]. It provides a continuous, differentiable depth signal that is numerically stable for downstream tasks such as pseudo-normal estimation and ray-tracing initialization [S1]. Unlike Median Depth Rendering, which selects a discrete sample index based on a cumulative opacity threshold, Expected Depth Rendering integrates contributions from all samples along the ray, resulting in smoother gradients and superior reconstruction quality for inverse rendering pipelines [S1][S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `z`: Tensor of depth samples (or sample positions $t_i$) along each ray. Shape: $(N_{rays}, N_{samples})$ [S1, S4].
    *   `w`: Tensor of per-sample weights. For Gaussian Splatting, $w_i = T_i g_i \alpha_i$ [S1]. For volumetric rendering, $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S4].
    *   `T`: Transmittance tensor, where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$ or the product of $(1-\alpha_j)$ for Gaussians [S1, S4].
    *   `ray_indices`: Optional packed indices mapping samples to rays (used in sparse implementations) [S2, unofficial].
    *   `num_rays`: Total number of rays (used in sparse implementations) [S2, unofficial].
*   **Outputs:**
    *   `depth`: Tensor of expected depth values for each ray. Shape: $(N_{rays})$ [S1, S4].

**Data Structures:**
*   **Samples:** Stored as 1D or 2D tensors. In packed formats (e.g., `nerfacc`), samples are flattened into a 1D array with a corresponding `ray_indices` array indicating which ray each sample belongs to [S2, unofficial].
*   **Weights:** Must be non-negative. In Gaussian splatting, this combines the Gaussian response $g_i$, opacity $\alpha_i$, and transmittance $T_i$ [S1].

**Steps:**
1.  **Compute Sample Midpoints/Positions:** Determine the depth value $z_i$ for each sample. If only start/end frustum bounds are available, compute midpoints:
    $$ z_i = \frac{\text{starts}_i + \text{ends}_i}{2} $$ [S2, unofficial].
2.  **Compute Weights:** Calculate the weight $w_i$ for each sample.
    *   For Gaussian mixtures: $w_i = T_i g_i \alpha_i$ [S1].
    *   For volumetric density: $w_i = T_i (1 - \exp(-\sigma_i \delta_i))$ [S4].
3.  **Accumulate Weights:** Compute the total accumulated weight $W$ for each ray.
    $$ W = \sum_{i \in N} w_i $$ [S1].
4.  **Compute Expected Depth:** Calculate the weighted sum of depths and normalize by the total weight.
    $$ z_E = \frac{\sum_{i \in N} w_i z_i}{W} $$ [S1].
    *   Note: In volumetric rendering, this is equivalent to $\hat{D}(r) = \sum_{i=1}^{N} T_i (1 - \exp(-\sigma_i \delta_i)) t_i$ [S4].
5.  **Numerical Stability:** Add a small epsilon $\epsilon$ to the denominator to prevent division by zero in empty regions.
    $$ z_E = \frac{\sum_{i \in N} w_i z_i}{W + \epsilon} $$ [S1, G].
6.  **Clipping (Optional):** Clip the resulting depth to the range $[\min(z), \max(z)]$ to handle boundary artifacts [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def expected_depth_rendering(z, weights, eps=1e-7):
    """
    Compute expected depth from sample depths and weights.
    
    Args:
        z: Tensor of shape (N_rays, N_samples) containing sample depths.
        weights: Tensor of shape (N_rays, N_samples) containing per-sample weights.
        eps: Small constant for numerical stability.
        
    Returns:
        depth: Tensor of shape (N_rays) containing expected depth.
    """
    # Step 1: Compute weighted sum of depths
    weighted_depth_sum = torch.sum(weights * z, dim=-1)
    
    # Step 2: Compute total accumulated weight
    total_weight = torch.sum(weights, dim=-1)
    
    # Step 3: Normalize with epsilon for stability
    depth = weighted_depth_sum / (total_weight + eps)
    
    # Step 4: Clip to valid range (optional, depends on implementation)
    # depth = torch.clamp(depth, min=z.min(dim=-1).values, max=z.max(dim=-1).values)
    
    return depth

def expected_depth_rendering_packed(z, weights, ray_indices, num_rays, eps=1e-7):
    """
    Compute expected depth for packed ray samples (e.g., using nerfacc).
    
    Args:
        z: 1D Tensor of sample depths.
        weights: 1D Tensor of sample weights.
        ray_indices: 1D Tensor mapping each sample to a ray index.
        num_rays: Total number of rays.
        eps: Small constant for numerical stability.
        
    Returns:
        depth: Tensor of shape (num_rays) containing expected depth.
    """
    # Use accumulation function similar to nerfacc.accumulate_along_rays
    # depth = accumulate_along_rays(weights, ray_indices, z, num_rays)
    # total_weight = accumulate_along_rays(weights, ray_indices, None, num_rays)
    
    # Manual implementation for clarity:
    depth_sum = torch.zeros(num_rays, device=z.device)
    weight_sum = torch.zeros(num_rays, device=z.device)
    
    # Scatter add operations
    depth_sum.scatter_add_(0, ray_indices, weights * z)
    weight_sum.scatter_add_(0, ray_indices, weights)
    
    # Normalize
    depth = depth_sum / (weight_sum + eps)
    
    return depth
```

**Parameters:**
*   `eps`: Small constant for numerical stability. Default value is typically $10^{-7}$ or similar, though not explicitly stated in sources, it is standard practice [S1, G].
*   `method`: Strategy for computing depth. 'expected' denotes the weighted mean approach [S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Rays:** If a ray hits no geometry, $W=0$. Division by zero is prevented by adding $\epsilon$ [S1].
*   **Precision:** Expected depth is sensitive to the distribution of weights. If weights are very small or noisy, the depth estimate may be unstable [S1].
*   **Boundary Conditions:** Depth values at the start/end of the ray may be biased if the volume extends beyond the sampled interval. Clipping to $[z_{min}, z_{max}]$ can mitigate this [S2, unofficial].

## Subtopics

### [Mean Gaussian Depth Rendering](9666-mean-gaussian-depth-rendering.md)
Computes the expected depth of a ray by calculating the weighted mean of sample depths, where weights are derived from the Gaussian's opacity, 2D contribution, and transmittance. It differs from Mode Depth Rendering, which selects the single Gaussian with the highest weight, by integrating contributions from all Gaussians along the ray to produce a continuous depth value. This method builds on [Expected Depth Rendering](8816-expected-depth-rendering.md).

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The computation involves a single pass to compute weights and a second pass (or fused operation) for the weighted sum and normalization [S1, S4].
*   **Memory Complexity:** $O(N)$ per ray to store sample depths and weights. In packed formats, memory is proportional to the total number of samples across all rays [S2, unofficial].
*   **Performance:** Expected depth is computationally efficient and differentiable, making it suitable for gradient-based optimization in neural rendering [S1]. It is generally faster than median depth if the median requires sorting or binary search, though both are $O(N)$ with efficient implementations [S3, unofficial].
*   **Strengths:** Provides a smooth, continuous depth signal that is robust to noise and suitable for normal estimation [S1]. It integrates information from the entire ray, reducing artifacts from sparse sampling [S1].
*   **Limitations:** Can be biased towards regions with high opacity/weight, potentially missing thin structures if weights are not well-calibrated [S1]. It does not provide a discrete surface location like median depth [S3, unofficial].
*   **When to Use:** When a continuous depth signal is needed for gradient-based optimization, pseudo-normal estimation, or ray-tracing initialization [S1]. Preferred over inverse-depth variants for overall reconstruction quality [S1].
*   **When Not to Use:** When a discrete surface location is required, or when the scene has significant transparency and a specific "first hit" is needed (consider Median Depth) [S3, unofficial].

## Background
Expected Depth Rendering originates from the foundational work on volumetric rendering and neural radiance fields (NeRF). It is a standard technique for computing depth from volume density and transmittance, analogous to color rendering but weighting samples by their depth position [S4]. The method has been widely adopted in 3D Gaussian Splatting variants for geometry supervision, where it provides a stable geometric signal for pseudo-normal estimation and ray-tracing initialization [S1]. It is often compared with Median Depth Rendering, which uses a threshold-based accumulation point [S3, unofficial].

## Sources
[S1] VisibilitySplat: Ray-Traced Occlusion for Relighting with 3D Gaussian Splatting, https://doi.org/10.24132/csrn.2026-e17, fetched 2026-10-26
[S2] autonomousvision/sdfstudio, https://github.com/autonomousvision/sdfstudio, fetched 2026-10-26 (unofficial)
[S3] google/nerfies, https://github.com/google/nerfies, fetched 2026-10-26 (unofficial)
[S4] 360FusionNeRF: Panoramic Neural Radiance Fields with Joint Guidance, https://doi.org/10.1109/iros55552.2023.10341346, fetched 2026-10-26
[S5] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
