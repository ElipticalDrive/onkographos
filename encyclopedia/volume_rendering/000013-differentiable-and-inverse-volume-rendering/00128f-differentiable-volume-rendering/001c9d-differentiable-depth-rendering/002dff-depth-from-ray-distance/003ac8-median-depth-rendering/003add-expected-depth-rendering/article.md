# Expected Depth Rendering

Also called: Depth Renderer (Expected)

## Overview
Expected Depth Rendering computes the expected depth of a ray by calculating the weighted mean of sample depths [S1, unofficial]. It serves as a differentiable depth estimation method, distinct from Median Depth Rendering which uses the median of weighted depths [8815-median-depth-rendering.md]. While Median Depth Rendering is robust to outliers, Expected Depth Rendering provides a smooth, differentiable estimate of the center of mass of the volume along the ray, making it suitable for optimization tasks where gradient continuity is prioritized over robustness [S1, unofficial].

## How it works
**Inputs:**
- Ray samples: A set of $N$ samples along a ray, indexed $k=1 \dots N$.
- Sample depths: $d_k$, the distance of the $k$-th sample from the camera origin.
- Sample opacities: $o_k$, the opacity contribution of the $k$-th sample.
- Transmittance: $T_k$, the accumulated transmittance up to the $k$-th sample.
- Gaussian contribution (for Gaussian Splatting variants): $\mathcal{G}_k^{\text{D}}(\mathbf{x})$, the 2D Gaussian projection weight for pixel $\mathbf{x}$.

**Outputs:**
- Expected depth $D$, a scalar value representing the mean depth along the ray.

**Data Structures:**
- Arrays for $d_k$, $o_k$, and $T_k$ of size $N$.
- $T_k$ is typically computed as $T_k = \prod_{j=1}^{k-1} (1 - o_j)$, with $T_1 = 1$.

**Steps:**
1. **Compute Transmittance:** Calculate the transmittance $T_k$ for each sample. For standard volume rendering, $T_k = \prod_{j=1}^{k-1} (1 - o_j)$. For Gaussian-based methods, the weight may include the Gaussian projection term [S2].
2. **Compute Weights:** Determine the weight $w_k$ for each sample.
   - For standard volume rendering: $w_k = o_k T_k$.
   - For Gaussian Depth Rendering: $w_k = o_k \mathcal{G}_k^{\text{D}}(\mathbf{x}) T_k$ [S2].
3. **Compute Expected Depth:** Calculate the weighted sum of depths normalized by the sum of weights.
   $$ D = \frac{\sum_{k=1}^N w_k d_k}{\sum_{k=1}^N w_k} $$
   If $\sum w_k \approx 0$, return a default depth (e.g., far plane) [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def expected_depth_rendering(depths, opacities, transmittances, gaussian_weights=None):
    """
    Computes the expected depth from ray samples.
    
    Args:
        depths (Tensor): [N] depths d_k.
        opacities (Tensor): [N] opacities o_k.
        transmittances (Tensor): [N] transmittances T_k.
        gaussian_weights (Tensor, optional): [N] Gaussian projection weights G_k^D(x).
        
    Returns:
        Tensor: Expected depth scalar.
    """
    N = depths.shape[0]
    
    # Compute weights
    if gaussian_weights is not None:
        weights = opacities * gaussian_weights * transmittances
    else:
        weights = opacities * transmittances
        
    # Sum of weights
    sum_weights = torch.sum(weights)
    
    # Handle empty or transparent rays
    if sum_weights < 1e-6:
        return torch.tensor(0.0) # Or far plane depth [G]
        
    # Weighted mean
    expected_depth = torch.sum(weights * depths) / sum_weights
    
    return expected_depth
```

**Parameters:**
- `method`: 'expected' indicates the use of weighted mean depth [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Transparency:** If all opacities are zero, the denominator is zero. A fallback value must be chosen [G].
- **Precision:** Summing small weights can lead to numerical instability. Use Kahan summation or higher precision if necessary [G].
- **Jensen's Inequality Gap:** In Gaussian Splatting contexts, fitting volumetric density (phasors or raw quads) does not guarantee consistency with the rendered mean depth [S2]. This is because sine/cosine functions in phase encoding oscillate, preventing equality in Jensen's inequality. The rendered mean depth diverges from depth derived from raw quads if multiple density peaks exist along a ray but only one surface exists [S2]. The gap reduces only when all Gaussians are clustered at the same z-position [S2].

## Subtopics

### Mean Gaussian Depth Rendering
A variant of Expected Depth Rendering specifically for 3D Gaussian Splatting. It computes the mean depth $d(\mathbf{x})$ for a pixel $\mathbf{x}$ as the weighted sum of Gaussian depths $d_k$, where weights are the product of opacity $o_k$, 2D Gaussian contribution $\mathcal{G}_k^{\text{D}}(\mathbf{x})$, and transmittance $T_k$ [S2]. It is chosen when differentiable depth is required from Gaussian primitives. A key pitfall is that optimizing volumetric density (phasors/quads) does not ensure the rendered mean depth matches the depth derived from raw quads due to oscillations in phase encoding [S2]. Consistency is only guaranteed when Gaussians are clustered at the same z-position [S2].

## Operating characteristics
**Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples. The computation involves a single pass to compute weights and a second pass (or accumulation) for the weighted sum.
**Memory Complexity:** $O(N)$ to store sample attributes.
**Performance:** Fast and differentiable. Suitable for real-time or near-real-time applications if $N$ is small.
**Strengths:** Smooth gradients, differentiable, easy to implement.
**Limitations:** Sensitive to outliers (unlike Median Depth). In Gaussian Splatting, inconsistent with raw quad depth if density peaks are distributed [S2].

## Background
Expected Depth Rendering is a standard technique in differentiable volume rendering. It is implemented in frameworks like GanERF [S1, unofficial]. The method is derived from the fundamental principles of volume rendering, where the expected value of the depth is the first moment of the depth distribution weighted by the alpha compositing weights.

## Sources
[S1] barbararoessle/ganerf, https://github.com/barbararoessle/ganerf, fetched 2026-10-27 (unofficial)
[S2] Time of the Flight of the Gaussians: Optimizing Depth Indirectly in Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52734.2025.01958, fetched 2026-10-26
