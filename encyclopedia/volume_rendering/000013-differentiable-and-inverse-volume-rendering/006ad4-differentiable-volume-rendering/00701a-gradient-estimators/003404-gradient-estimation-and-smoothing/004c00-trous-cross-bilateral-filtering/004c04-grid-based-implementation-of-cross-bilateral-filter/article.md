# Grid-based Implementation of Cross-Bilateral Filter

Also called: Cross-Bilateral Filter

## Overview
This method implements the cross-bilateral filter on regular 2D or 3D grids to enable real-time spatial filtering of gradients and associated buffers in inverse rendering [S1]. It differs from standard bilateral filters by decomposing the operation into a series of à-trous filters, which allows for fast optimization with a fixed number of non-zero elements per filter pass [S1]. It builds on [À-trous Cross-Bilateral Filtering](13139-trous-cross-bilateral-filtering.md).

## How it works
**Inputs and Outputs:**
*   **Input:** Gradient $\nabla_\theta f_t$ and associated buffers $m_t, v_t$ defined on a regular 2D or 3D grid [S1].
*   **Output:** Filtered gradient and buffers [S1].

**Data Structures:**
The data is stored on a regular grid matching the spatial resolution of the rendered views or volume [S1]. The grid layout follows standard row-major or column-major memory ordering [G].

**Steps:**
1. Initialize the gradient $\nabla_\theta f_t$ and buffers $m_t, v_t$ on the grid [S1].
2. Apply a sequence of à-trous cross-bilateral filters [S1].
3. Each filter in the sequence uses an increasing spatial support but maintains a fixed number of non-zero elements per pass [S1].
4. The filtering follows the implementation strategy of Dammertz et al. [S1].

**Formulas:**
The filter computes a weighted average over neighbors at the current à-trous stride $2^k$ [S1]. The cross-bilateral weight between cell $i$ and neighbor $j$ is $w_{ij} = \exp(-\|p_i - p_j\|^2 / 2\sigma_s^2) \cdot \exp(-\|b_i - b_j\|^2 / 2\sigma_c^2)$ [G], where $p$ denotes spatial coordinates, $b$ denotes the buffer value used for guidance, and $\sigma_s, \sigma_c$ are spatial and cross-bilateral standard deviations [G]. The filtered value at $i$ is $\hat{v}_i = \frac{\sum_j w_{ij} v_j}{\sum_j w_{ij}}$ [G].

**Parameters:**
*   **Number of filter passes:** Not given in the sources [S1].
*   **Support size:** Increases with each pass [S1].
*   **Number of non-zero elements:** Fixed per filter pass [S1].
*   **$\sigma_s, \sigma_c$:** Standard deviations for spatial and cross-bilateral terms [G].

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulators should use double precision to avoid normalization artifacts [G].
*   **Boundaries:** Neighbors outside the grid are clamped or mirrored to prevent boundary bias [G].
*   **Degenerate input:** Zero denominators in normalization are guarded with a small epsilon [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def grid_based_cross_bilateral_filter(grid_gradient, grid_buffers, num_passes, support_radii, sigma_s, sigma_c):
    """
    Applies a series of à-trous cross-bilateral filters to grid data.
    Args:
        grid_gradient: 2D or 3D array of gradients.
        grid_buffers: 2D or 3D array of buffers (m_t, v_t).
        num_passes: Number of filter passes.
        support_radii: List of spatial support radii (strides) for each pass.
        sigma_s: Spatial standard deviation [G].
        sigma_c: Cross-bilateral standard deviation [G].
    Returns:
        Filtered gradient and buffers.
    """
    filtered_gradient = grid_gradient.copy()
    filtered_buffers = grid_buffers.copy()
    
    for k in range(num_passes):
        stride = support_radii[k]
        acc_grad = np.zeros_like(filtered_gradient)
        acc_buf = np.zeros_like(filtered_buffers)
        acc_weight = np.zeros_like(filtered_gradient)
        
        for i in np.ndindex(filtered_gradient.shape):
            spatial_dist_sq = 0.0
            feature_diff_sq = 0.0
            w = 0.0
            
            # Accumulate contributions from neighbors at current stride
            for di in range(-stride, stride + 1, stride):
                for dj in range(-stride, stride + 1, stride):
                    # Handle 3D case by extending loop or using ndenumerate
                    ni = i[0] + di
                    nj = i[1] + dj
                    if len(i) > 2:
                        dk = i[2]
                        nk = ni + dk # Simplified for 2D/3D generalization
                    else:
                        nk = 0
                    
                    if 0 <= ni < filtered_gradient.shape[0] and 0 <= nj < filtered_gradient.shape[1]:
                        spatial_dist_sq = di**2 + dj**2
                        feature_diff_sq = (filtered_buffers[ni, nj] - filtered_buffers[i])**2
                        w = np.exp(-spatial_dist_sq / (2 * sigma_s**2)) * np.exp(-feature_diff_sq / (2 * sigma_c**2))
                        acc_grad[i] += w * filtered_gradient[ni, nj]
                        acc_buf[i] += w * filtered_buffers[ni, nj]
                        acc_weight[i] += w
            
            # Normalize with guard
            acc_weight = np.maximum(acc_weight, 1e-8)
            filtered_gradient[i] = acc_grad[i] / acc_weight[i]
            filtered_buffers[i] = acc_buf[i] / acc_weight[i]
            
    return filtered_gradient, filtered_buffers
```

## Subtopics

### [Bilateral Grid](13150-bilateral-grid.md)
The Bilateral Grid constructs a sparse 5D grid in spatial-luminance-chrominance space to facilitate efficient bilateral filtering via splatting and slicing. It differs from standard bilateral filters by discretizing the feature space into a grid, allowing the filter to be implemented as a sequence of sparse matrix-vector multiplications, which is significantly faster for large kernels. It builds on the Grid-based Implementation of Cross-Bilateral Filter [13144-grid-based-implementation-of-cross-bilateral-filter.md].

### Bistochastization of Bilateral Grid
Computes diagonal matrices $D_n$ and $D_m$ to bistochastize a bilateral grid, ensuring row and column sums are equal [S2, unofficial].
1. Initialize $m = \text{grid.splat}(\text{np.ones}(\text{grid.npixels}))$ and $n = \text{np.ones}(\text{grid.nvertices})$ [S2, unofficial].
2. Iterate $\text{maxiter}$ times: $n = \sqrt{n \cdot m / (\text{grid.blur}(n) + 10^{-10})}$ [S2, unofficial].
3. Correct $m$: $m = n \cdot \text{grid.blur}(n)$ [S2, unofficial].
4. Create diagonal matrices $D_n = \text{diags}(n, 0)$ and $D_m = \text{diags}(m, 0)$ [S2, unofficial].

### Edge Island Filtering
A filtering algorithm to remove invalid color values produced by the bilateral solver in isolated patches ('edge islands') within the masked region, preventing them from being used for supervision [S3].
1. Dilate the reference mask $M_r$ with kernel size 5 to obtain the dilated mask $M_{r_{\text{dilated}}}$ [S3].
2. Find the maximum absolute residual value inside $M_{r_{\text{dilated}}}$ and outside $M_r$: $res_{\max,t} = \max |res_t|$ over $M_{r_{\text{dilated}}} \cap (1 - M_r)$ [S3].
3. Define a mask for out-of-distribution values for view $t$, $M_{r,t}^{ei}$, as pixels in $M_r$ with absolute values higher than $res_{\max,t} \times c_{ei}$, where $c_{ei} \ge 1$ is the filtering threshold [S3].
4. The edge island mask is the union of out-of-distribution masks across all target views: $M_r^{ei} = \bigcup_t M_{r,t}^{ei}$ [S3].

## Operating characteristics
*   **Time Complexity:** Not given in the sources [S1].
*   **Memory Complexity:** Not given in the sources [S1].
*   **Measured Performance:** The method enables real-time performance and faster optimization with low sample budgets [S1].
*   **Scaling:** Not given in the sources [S1].
*   **Strengths:** Fast optimization, real-time performance, low sample budgets [S1].
*   **Limitations:** Not given in the sources [S1].
*   **When to use:** Inverse rendering applications requiring fast spatial filtering of gradients and buffers [S1].
*   **When not to use:** Not given in the sources [S1].

## Background
The method was introduced in "Spatiotemporal Bilateral Gradient Filtering for Inverse Rendering" [S1]. It leverages previous work in real-time denoising [S1].

## Sources
[S1] Spatiotemporal Bilateral Gradient Filtering for Inverse Rendering, https://doi.org/10.1145/3680528.3687606, fetched 2026-10-29
[S2] weiyithu/NerfingMVS, https://github.com/weiyithu/NerfingMVS, fetched 2026-10-27 (unofficial)
[S3] Reference-guided Controllable Inpainting of Neural Radiance Fields, https://doi.org/10.1109/ICCV51070.2023.01633, fetched 2026-10-29
