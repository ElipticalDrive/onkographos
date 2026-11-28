# Importance Sampling (Fine)

Also called: Fine Importance Sampling, Fine Ray Importance Sampling

## Overview
This method generates secondary sample positions along a ray by leveraging the probability distribution derived from a prior coarse rendering pass [S1, unofficial]. It is used in hierarchical sampling strategies where coarse points are uniformly sampled and fine points are sampled based on the coarse probability estimation [S2, unofficial].

## How it works
**Inputs:**
- Coarse weights $w_i$ for $i \in \{0, \dots, N_{coarse}-1\}$ [S1, unofficial].
- Number of fine samples to generate, $N_{fine}$ (denoted as $K_f - K_{fd}$ in sources) [S1, unofficial].
- Number of coarse samples, $N_{coarse}$ [S2, unofficial].
- Ray depth or disparity bounds [S1, unofficial].

**Outputs:**
- $N_{fine}$ sample positions (depths or disparities) along the ray [S1, unofficial].

**Data Structures:**
- `weights`: Array of size $N_{coarse}$ [S1, unofficial].
- `pdf`: Array of size $N_{coarse}$ [S1, unofficial].
- `cdf`: Array of size $N_{coarse} + 1$ [S1, unofficial].
- `u`: Array of size $B \times N_{fine}$ containing uniform random numbers in $[0, 1)$ [S1, unofficial].
- `indices`: Array of size $B \times N_{fine}$ containing integer indices into the CDF [S1, unofficial].
- `z_steps`: Array of size $B \times N_{fine}$ containing normalized positions in $[0, 1)$ [S1, unofficial].

**Steps:**

1.  **Regularize Weights:** Add a small epsilon $\epsilon = 10^{-5}$ to all coarse weights to prevent division by zero [S1, unofficial].
    $$ w'_i = w_i + \epsilon $$
2.  **Compute PDF:** Normalize the regularized weights along the ray dimension to form a probability distribution [S1, unofficial].
    $$ \text{pdf}_i = \frac{w'_i}{\sum_{j=0}^{N_{coarse}-1} w'_j} $$
3.  **Compute CDF:** Compute the cumulative sum of the PDF. Prepend a zero to the CDF array to handle the first interval [S1, unofficial].
    $$ \text{cdf}_0 = 0 $$
    $$ \text{cdf}_{i+1} = \text{cdf}_i + \text{pdf}_i \quad \text{for } i \in \{0, \dots, N_{coarse}-1\} $$
4.  **Generate Random Numbers:** Generate $B \times N_{fine}$ uniform random numbers $u_k \in [0, 1)$ [S1, unofficial].
5.  **Find Indices:** Use binary search (`searchsorted`) to find the index $idx_k$ in the CDF corresponding to each $u_k$ [S1, unofficial].
6.  **Clamp Indices:** Ensure indices are valid, clamping to a minimum of 0 [S1, unofficial].
    $$ idx_k = \max(0, idx_k) $$
7.  **Compute Normalized Steps:** Add uniform noise to the index and divide by $N_{coarse}$ to obtain a continuous normalized position $z_k \in [0, 1)$ [S1, unofficial].
    $$ z_k = \frac{idx_k + \text{uniform}(0, 1)}{N_{coarse}} $$
8.  **Map to Depth/Disparity:** Map the normalized step $z_k$ to the actual depth or disparity space using the same transformation logic used in the coarse sampling stage [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def importance_sampling_fine(weights, n_coarse, n_fine, near, far):
    """
    Perform fine importance sampling based on coarse weights.
    
    Args:
        weights: np.ndarray of shape (N_coarse,), coarse rendering weights.
        n_coarse: int, number of coarse samples.
        n_fine: int, number of fine samples to generate.
        near: float, near plane distance.
        far: float, far plane distance.
        
    Returns:
        np.ndarray of shape (N_fine,), sample depths.
    """
    # 1. Regularize weights
    epsilon = 1e-5
    weights_reg = weights + epsilon
    
    # 2. Compute PDF
    pdf = weights_reg / np.sum(weights_reg)
    
    # 3. Compute CDF (prepend zero)
    cdf = np.zeros(n_coarse + 1)
    cdf[1:] = np.cumsum(pdf)
    
    # 4. Generate uniform random numbers
    u = np.random.uniform(0, 1, size=n_fine)
    
    # 5. Find indices in CDF
    indices = np.searchsorted(cdf, u, side='left')
    
    # 6. Clamp indices to valid range [0, n_coarse - 1]
    indices = np.clip(indices, 0, n_coarse - 1)
    
    # 7. Compute normalized z_steps
    noise = np.random.uniform(0, 1, size=n_fine)
    z_steps = (indices + noise) / n_coarse
    
    # 8. Map z_steps to depth space
    depths = near + z_steps * (far - near)
    
    return depths
```

**Parameters:**
- $N_{coarse}$: Number of coarse samples. Value: 64 [S2, unofficial].
- $N_{fine}$: Number of fine samples. Value: 64 [S2, unofficial].
- $\epsilon$: Epsilon for weight regularization. Value: $10^{-5}$ [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Weights:** If all coarse weights are zero, the PDF becomes undefined. The epsilon addition prevents this [S1, unofficial].
- **Index Out of Bounds:** `searchsorted` may return an index equal to the length of the CDF if $u_k$ is very close to 1. Clamping to the maximum valid index ($N_{coarse}-1$) is required to keep $z_k < 1$ [G].
- **Precision:** Floating point precision in cumulative sum can lead to slight inaccuracies in CDF normalization [G].
- **Disparity vs Depth:** The mapping step depends on whether the ray is parameterized by depth or disparity. The source states "using the same logic as coarse sampling," implying consistency with the coarse sampling method's parameterization [S1, unofficial].

## Subtopics

### Inverse Sampling for Fine Rays
The general technique of using inverse transform sampling to generate fine samples based on a probability distribution. Importance Sampling (Fine) is a specific implementation of this technique using coarse weights. [Link](3235-inverse-sampling-for-fine-rays.md)

## Operating characteristics
- Time and memory complexity: Not given in the sources.
- Measured performance: Not given in the sources.
- Strengths and limitations: Not given in the sources.
- When to use: Not given in the sources.

## Background
This method is part of the hierarchical sampling strategy used in volume rendering [S2, unofficial]. It samples fine positions along rays based on the probability distribution defined by coarse rendering weights using inverse transform sampling [S1, unofficial].

## Sources
[S1] sxyu/pixel-nerf, https://github.com/sxyu/pixel-nerf, fetched 2026-10-26 (unofficial)
[S2] [PDF] VolRecon: Volume Rendering of Signed Ray Distance Functions for ..., https://openaccess.thecvf.com/content/CVPR2023/papers/Ren_VolRecon_Volume_Rendering_of_Signed_Ray_Distance_Functions_for_Generalizable_CVPR_2023_paper.pdf, fetched 2026-10-26 (unofficial)
