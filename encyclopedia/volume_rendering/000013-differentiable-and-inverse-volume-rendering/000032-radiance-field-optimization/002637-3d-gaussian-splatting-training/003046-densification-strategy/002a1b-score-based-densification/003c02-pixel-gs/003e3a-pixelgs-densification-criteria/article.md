# PixelGS Densification Criteria

Also called: Pixel-aware and Depth-scaled Gradient Densification

## Overview
PixelGS Densification Criteria modifies the gradient-based densification strategy of 3D Gaussian Splatting to account for the spatial footprint of each Gaussian primitive. It scales the positional gradient by the number of pixels a Gaussian influences and applies a depth-dependent scaling factor to encourage densification in background regions while suppressing it in the foreground. This approach yields state-of-the-art rendering quality compared to standard 3DGS, though it typically results in a higher number of Gaussian primitives [S1].

## How it works
**Inputs:**
- Set of 3D Gaussians $\{g_k\}$.
- For each Gaussian $k$ and view $\nu$:
  - $\text{count}(g_k^\nu)$: The number of pixels influenced by Gaussian $k$ in view $\nu$.
  - $\text{depth}(g_k^\nu)$: The image depth of the rendered splat for Gaussian $k$ in view $\nu$.
  - $\left\| \frac{\partial \mathcal{L}(I_\nu, f'_\nu)}{\partial \mu_k^\nu} \right\|_2$: The 2D positional gradient norm of the loss $\mathcal{L}$ with respect to the mean $\mu_k$ of Gaussian $k$ in view $\nu$.
- Hyperparameters:
  - $\gamma$: Depth scaling hyperparameter (default 0.37) [S1].
  - $e_{\text{scene}}$: The extent of the scene [S1].
  - $N_{\text{views}}$: Total number of training views.

**Outputs:**
- $\tau'_k$: The refined gradient value for Gaussian $k$, used to determine if densification (splitting or cloning) should occur.

**Data Structures:**
- No specific complex data structures are required beyond standard per-Gaussian accumulators for the gradient norm and pixel count across views.

**Steps:**
1. For each Gaussian $k$ and each view $\nu \in \{1, \dots, N_{\text{views}}\}$:
   a. Compute the depth-scaling factor $f_k^\nu$.
   b. Accumulate the weighted gradient contribution.
2. Compute the final refined gradient $\tau'_k$ by averaging the weighted contributions normalized by the total pixel count.
3. Compare $\tau'_k$ against a threshold to decide on densification.

**Formulas:**

The depth-scaling factor $f_k^\nu$ for Gaussian $k$ in view $\nu$ is defined as [S1]:

$$
f_k^\nu = \text{clip} \left( \frac{\text{depth}(g_k^\nu)}{\gamma \cdot e_{\text{scene}}}, [0, 1] \right)
$$

where:
- $\text{depth}(g_k^\nu)$ is the image depth of the rendered splat.
- $\gamma$ is the hyperparameter (default 0.37).
- $e_{\text{scene}}$ is the scene extent.
- $\text{clip}(x, [a, b])$ clamps $x$ to the range $[a, b]$.

The refined gradient $\tau'_k$ is calculated as [S1]:

$$
\tau'_k = \frac{\sum_{\nu=1}^{N_{\text{views}}} f_k^\nu \cdot \text{count}(g_k^\nu) \cdot \left\| \frac{\partial \mathcal{L}(I_\nu, f'_\nu)}{\partial \mu_k^\nu} \right\|_2}{\sum_{\nu=1}^{N_{\text{views}}} \text{count}(g_k^\nu)}
$$

where:
- $\text{count}(g_k^\nu)$ is the number of pixels influenced by Gaussian $k$ in view $\nu$.
- $\left\| \frac{\partial \mathcal{L}(I_\nu, f'_\nu)}{\partial \mu_k^\nu} \right\|_2$ is the 2D positional gradient norm.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_pixelgs_densification_criteria(gaussians, views, gamma=0.37, scene_extent=None):
    """
    Computes the refined gradient tau'_k for each Gaussian k using PixelGS criteria.
    
    Args:
        gaussians: List of Gaussian objects, each having an id 'k'.
        views: List of view objects, each having an id 'nu'.
        gamma: Depth scaling hyperparameter.
        scene_extent: The extent of the scene. If None, it must be provided externally or assumed known.
        
    Returns:
        dict: Mapping of Gaussian id 'k' to refined gradient tau'_k.
    """
    # Initialize accumulators for each Gaussian
    # weighted_gradient_sum[k] = sum over views of (f * count * grad_norm)
    # pixel_count_sum[k] = sum over views of count
    weighted_gradient_sum = {}
    pixel_count_sum = {}
    
    for k in [g.id for g in gaussians]:
        weighted_gradient_sum[k] = 0.0
        pixel_count_sum[k] = 0.0

    for g in gaussians:
        k = g.id
        for nu in views:
            # Retrieve pre-computed or compute-on-the-fly metrics for Gaussian k in view nu
            # Note: In a real implementation, these values are typically available from the rasterization step.
            count = get_pixel_count(g, nu)
            depth = get_rendered_depth(g, nu)
            grad_norm = get_positional_gradient_norm(g, nu)
            
            # Step 1: Compute depth-scaling factor f_k^nu
            # f_k^nu = clip(depth / (gamma * scene_extent), 0, 1)
            if scene_extent is None:
                raise ValueError("scene_extent must be provided to compute depth scaling.")
            
            depth_scale = depth / (gamma * scene_extent)
            f_k_nu = max(0.0, min(1.0, depth_scale))
            
            # Step 2: Accumulate weighted gradient
            # Contribution = f_k^nu * count * grad_norm
            contribution = f_k_nu * count * grad_norm
            weighted_gradient_sum[k] += contribution
            
            # Accumulate pixel count for normalization
            pixel_count_sum[k] += count

    # Step 3: Compute final refined gradient tau'_k
    results = {}
    for k in weighted_gradient_sum:
        total_count = pixel_count_sum[k]
        if total_count == 0:
            # Edge case: Gaussian not visible in any view
            results[k] = 0.0
        else:
            results[k] = weighted_gradient_sum[k] / total_count
            
    return results
```

**Edge Cases and Pitfalls:**
- **Zero Visibility:** If a Gaussian is not visible in any view, $\text{count}(g_k^\nu)$ is 0 for all $\nu$. The denominator becomes 0. The implementation must handle this division by zero, typically by assigning a gradient of 0 or skipping densification for that Gaussian [G].
- **Depth Scaling Range:** The depth scaling factor is clamped to $[0, 1]$. If $\text{depth}(g_k^\nu)$ is very small (foreground), $f_k^\nu$ approaches 0, suppressing densification. If it is large (background), $f_k^\nu$ approaches 1, encouraging densification. Incorrect scene extent $e_{\text{scene}}$ will distort this balance [S1].
- **Hyperparameter Sensitivity:** The value of $\gamma$ (default 0.37) directly controls the depth threshold. A wrong value may lead to over-densification in the background or under-densification in the foreground [S1].

## Subtopics

### [Pixel-GS](9189-pixel-gs.md)
The parent method that uses this criteria to scale densification by projected area. Choose this when you need the specific pixel-aware weighting logic described here.

### [3D Gaussian Splatting Training](9188-3d-gaussian-splatting-training.md)
The base training loop. PixelGS builds on this by modifying the densification step.

## Operating characteristics
- **Time Complexity:** The computation adds a linear pass over the Gaussians and views to accumulate the weighted gradients and pixel counts. This is negligible compared to the rasterization and gradient computation steps of the main training loop.
- **Memory Complexity:** Requires storing two accumulators per Gaussian (weighted gradient sum and pixel count sum) across views. This is minimal additional memory overhead [S1].
- **Performance:** PixelGS yields state-of-the-art rendering quality compared to standard 3DGS and other densification strategies [S1].
- **Trade-offs:** The primary cost is a considerably higher number of Gaussian primitives. The method encourages densification in background regions, which increases the total primitive count compared to methods that do not use depth scaling [S1].
- **When to use:** Use when rendering quality is the primary metric and increased memory/compute for a larger number of Gaussians is acceptable.
- **When not to use:** Use when memory or rendering speed is constrained, as the increased number of primitives will increase memory usage and potentially slow down rendering.

## Background
PixelGS Densification Criteria was proposed by Zhang et al. in the context of PixelGS [S1]. It builds upon the adaptive density control of 3D Gaussian Splatting and incorporates ideas similar to Bulo et al. (2024) regarding pixel-weighted gradients. The method was introduced to improve rendering quality by better handling the distribution of Gaussians in depth, specifically addressing the tendency of standard 3DGS to under-densify background regions [S1].

## Sources
[S1] Improving Adaptive Density Control for 3D Gaussian Splatting, https://doi.org/10.5220/0013308500003912, fetched 2026-10-30
