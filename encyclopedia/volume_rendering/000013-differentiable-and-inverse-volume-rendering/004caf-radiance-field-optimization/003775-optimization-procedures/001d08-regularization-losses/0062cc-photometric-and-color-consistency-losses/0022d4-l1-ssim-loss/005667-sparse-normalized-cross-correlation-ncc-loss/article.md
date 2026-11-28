# Sparse Normalized Cross-Correlation (NCC) Loss
Also called: Tile-wise Structure-Aware Loss
## Overview
A local structure-aware loss that provides sparse yet effective supervision to preserve fine details and stabilize training by measuring sparse Normalized Cross-Correlation (NCC) [S1, unofficial]. It sets itself apart from dense photometric losses by operating only on per-tile sparsely sampled pixels, prioritizing challenging regions while skipping well-reconstructed ones to avoid redundant gradient computation [S1, unofficial]. It builds on [L1+SSIM Loss](2890-l1-ssim-loss.md) by replacing or augmenting dense pixel-wise comparisons with a structural consistency metric computed over sparse subsets.

## How it works
**Inputs and Outputs:**
- Input: A set of sparsely sampled pixels per tile, driven by multi-view reconstruction errors. Let $P_{tile}$ be the set of pixel coordinates in a tile. For each pixel $p \in P_{tile}$, inputs are the rendered color $C_r(p)$ and the ground truth color $C_g(p)$.
- Output: A scalar loss value $L_{NCC}$ for the tile.

**Data Structures:**
- Per-tile error map: Maintained online to prioritize per-iteration rasterization and gradient backpropagation on challenging regions [S1, unofficial].
- Sparse pixel set $P_{tile}$: A subset of pixels within a tile, selected based on the error map.

**Steps:**
1. **Sparse Sampling:** Select pixels $P_{tile}$ for each tile based on the online-maintained multi-view error map. Pixels with high reconstruction error are prioritized; well-reconstructed regions are skipped [S1, unofficial].
2. **Compute Local Statistics:** For the set $P_{tile}$, compute the mean and variance of the rendered and ground truth colors.
   - Let $N = |P_{tile}|$.
   - $\mu_r = \frac{1}{N} \sum_{p \in P_{tile}} C_r(p)$
   - $\mu_g = \frac{1}{N} \sum_{p \in P_{tile}} C_g(p)$
   - $\sigma_r^2 = \frac{1}{N} \sum_{p \in P_{tile}} (C_r(p) - \mu_r)^2$
   - $\sigma_g^2 = \frac{1}{N} \sum_{p \in P_{tile}} (C_g(p) - \mu_g)^2$
3. **Compute Covariance:** Calculate the covariance between rendered and ground truth colors over the sparse set.
   - $\sigma_{rg} = \frac{1}{N} \sum_{p \in P_{tile}} (C_r(p) - \mu_r)(C_g(p) - \mu_g)$
4. **Compute NCC:** Calculate the Normalized Cross-Correlation coefficient $\rho$.
   - $\rho = \frac{\sigma_{rg}}{\sqrt{\sigma_r^2 \sigma_g^2 + \epsilon}}$
   where $\epsilon$ is a small constant for numerical stability [G].
5. **Compute Loss:** The loss is typically derived from the NCC score, often as $1 - \rho$ or a similar monotonic transformation to minimize dissimilarity.
   - $L_{NCC} = 1 - \rho$

**Parameters:**
- Tile size: Not given in the sources.
- Sparsity ratio: Not given in the sources.
- $\epsilon$: Not given in the sources.

**Edge Cases and Pitfalls:**
- Degenerate tiles: If a tile has no selected pixels or all selected pixels have zero variance, NCC is undefined. Guard against division by zero with $\epsilon$ [G].
- Sparse bias: If sampling is too sparse, the local statistics may not represent the tile's structure accurately.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sparse_ncc_loss(rendered_colors, ground_truth_colors, sparse_pixel_indices):
    """
    Compute Sparse NCC Loss for a tile.
    
    Args:
        rendered_colors: Array of shape (N, 3) for rendered colors at sparse indices.
        ground_truth_colors: Array of shape (N, 3) for ground truth colors at sparse indices.
        sparse_pixel_indices: List of pixel indices selected for this tile.
        
    Returns:
        Scalar loss value.
    """
    N = len(sparse_pixel_indices)
    if N == 0:
        return 0.0  # No pixels to compute loss for
    
    # Compute means
    mu_r = np.mean(rendered_colors, axis=0)
    mu_g = np.mean(ground_truth_colors, axis=0)
    
    # Compute variances and covariance
    # Using element-wise operations for RGB channels
    diff_r = rendered_colors - mu_r
    diff_g = ground_truth_colors - mu_g
    
    var_r = np.mean(diff_r ** 2, axis=0)
    var_g = np.mean(diff_g ** 2, axis=0)
    cov_rg = np.mean(diff_r * diff_g, axis=0)
    
    # Compute NCC coefficient
    epsilon = 1e-8
    denominator = np.sqrt(var_r * var_g + epsilon)
    ncc = cov_rg / denominator
    
    # Clamp NCC to [-1, 1] for numerical stability
    ncc = np.clip(ncc, -1.0, 1.0)
    
    # Compute loss (1 - NCC)
    loss = 1.0 - np.mean(ncc)
    
    return loss
```

## Subtopics

### [Hybrid Loss Gradient](15696-hybrid-loss-gradient.md)
The Hybrid Loss Gradient is a gradient computation method used in sparse optimization for volume rendering and 3D Gaussian Splatting. It stabilizes training under sparse pixel supervision by combining a pixel-wise Charbonnier $\ell_1$ term with a tile-level sparse Normalized Cross-Correlation (NCC) term. It sets itself apart from dense optimization by maintaining gradient alignment with the full objective through importance sampling, and from pure photometric losses by incorporating structural consistency via NCC. It builds on [Sparse Normalized Cross-Correlation (NCC) Loss](15693-sparse-normalized-cross-correlation-ncc-loss.md).

## Operating characteristics
- **Time Complexity:** Depends on the number of sparse pixels per tile. Significantly faster than dense NCC or SSIM if the sparsity ratio is low.
- **Memory Complexity:** Requires storing per-tile error maps and sparse pixel indices.
- **Measured Performance:** TurboGS delivers on par or superior rendering quality within 100 seconds (up to ~10× training speedup over vanilla 3DGS) [S1, unofficial].
- **Strengths:** Preserves fine details, stabilizes training, avoids redundant gradient computation in well-reconstructed regions.
- **Limitations:** Performance depends on the quality of the error-driven sampling. If sampling is poor, structural details may be lost.
- **When to use:** When training speed is critical and fine detail preservation is important, especially in challenging regions.
- **When not to use:** When uniform supervision is required or when the error map is unreliable.

## Background
Originated in TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization [S1, unofficial]. It is part of a broader effort to accelerate 3D Gaussian Splatting training by combining sparse sampling with structure-aware losses.

## Sources
[S1] TurboGS: Accelerating 3D Gaussian Splatting via Error-Guided Sparse Pixel Sampling and Optimization, https://doi.org/10.48550/arXiv.2606.15924, fetched 2026-10-29 (unofficial)
