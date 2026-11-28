# Compact Box (CB)
## Overview
Compact Box (CB) is a preprocessing optimization for rasterization that reduces the number of Gaussian-tile pairs by tightening the effective support region of 2D Gaussians, eliminating unnecessary pairs where the Gaussian has negligible impact on pixels within a tile [S1, unofficial]. It builds upon the precise tile-intersection strategy of Speedy-Splat to further reduce the effective 2D Gaussian region and eliminate redundant pairs [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** 2D Gaussian parameters (opacity $\sigma_i$, 2D mean $\mu_{i2D}$, 2D covariance $\Sigma_{i2D}$), tile grid configuration, and scaling factor $\beta$ [S1, unofficial].
- **Output:** A pruned set of relevant Gaussian-tile pairs for rasterization [S1, unofficial].

**Data Structures:**
The method relies on the standard 2D Gaussian representation used in 3D Gaussian Splatting [G]. The covariance matrix $\Sigma_{i2D}$ is decomposed into rotation and scaling components, but for the CB calculation, the inverse covariance $\Sigma_{i2D}^{-1}$ is used directly in the Mahalanobis distance calculation [S1, unofficial].

**Steps:**
1. **Calculate Modified Alpha:** For a given pixel $p$ and the $i$-th Gaussian, compute the alpha value using the modified equation that incorporates the scaling factor $\beta$:
   $$ \alpha_i(p) = \sigma_i \cdot \exp\left(-\frac{\beta}{2} (p - \mu_{i2D})^\top \Sigma_{i2D}^{-1} (p - \mu_{i2D})\right) $$
   where $\sigma_i$ is the opacity, $\mu_{i2D}$ is the 2D mean position, $\Sigma_{i2D}$ is the 2D covariance matrix, and $\beta$ is the scaling factor [S1, unofficial]. The term $(p - \mu_{i2D})^\top \Sigma_{i2D}^{-1} (p - \mu_{i2D})$ represents the squared Mahalanobis distance [S1, unofficial].
2. **Determine Effective Support:** Use the modified alpha equation to define the effective support region of the 2D Gaussian [S1, unofficial]. A smaller $\beta$ yields a tighter ellipse around the mean $\mu_{i2D}$, reducing the spatial extent and limiting the number of pixels influenced by the Gaussian [S1, unofficial].
3. **Integrate with SnugBox:** Integrate the modified alpha equation into the SnugBox implementation [S1, unofficial]. The parameter $\beta$ further reduces the 2D Gaussian footprint and shrinks its intersection region with tiles [S1, unofficial].
4. **Prune Gaussian-Tile Pairs:** Use the reduced footprint to determine which Gaussian-tile pairs are relevant [S1, unofficial]. Pairs where the Gaussian has negligible impact on pixels within a tile are eliminated [S1, unofficial]. This is typically done using the `accutile` method from Speedy-Splat, which performs precise tile intersection based on the compacted region [S1, unofficial].

**Parameters:**
- $\beta$: A scaling factor that flexibly controls the effective support region of each 2D Gaussian [S1, unofficial]. A smaller $\beta$ yields a tighter ellipse around the mean, reducing spatial extent and limiting pixels influenced by the Gaussian [S1, unofficial]. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** The calculation involves matrix inversion ($\Sigma_{i2D}^{-1}$) and exponential functions [S1, unofficial]. Numerical stability should be ensured for degenerate covariances [G].
- **Boundary Conditions:** The method relies on the precise tile intersection logic of SnugBox [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np

def compact_box_rasterization(gaussians, tiles, beta):
    """
    Rasterization with Compact Box optimization.
    
    Args:
        gaussians: list of dicts with keys 'sigma', 'mu_2d' (2,), 'Sigma_2d' (2,2)
        tiles: list of dicts with keys 'id', 'min' (2,), 'max' (2,)
        beta: float scaling factor
        
    Returns:
        list of (gaussian_index, tile_index) pairs
    """
    relevant_pairs = []
    for i, g in enumerate(gaussians):
        # Compute inverse covariance for Mahalanobis distance
        try:
            Sigma_inv = np.linalg.inv(g['Sigma_2d'])
        except np.linalg.LinAlgError:
            continue  # Skip degenerate covariance [G]

        # Compute compacted support region
        # Smaller beta tightens the ellipse, reducing spatial extent [S1, unofficial]
        # We approximate the compacted bounding box for tile culling
        eigenvalues = np.linalg.eigvalsh(g['Sigma_2d'])
        radii = np.sqrt(eigenvalues / beta)
        min_pt = g['mu_2d'] - radii
        max_pt = g['mu_2d'] + radii

        # Precise tile intersection is handled by Speedy-Splat's accutile method [S1, unofficial]
        # Here we perform a basic axis-aligned bounding box intersection check
        for j, t in enumerate(tiles):
            if min_pt[0] <= t['max'][0] and max_pt[0] >= t['min'][0] and \
               min_pt[1] <= t['max'][1] and max_pt[1] >= t['min'][1]:
                relevant_pairs.append((i, j))
                
    return relevant_pairs
```

## Subtopics

### SnugBox
A precise algorithm for computing Gaussian-tile bounding box intersections, replacing the overly conservative tile assignment in standard 3D-GS to reduce the number of pixels processed per Gaussian. Choose when you need precise tile intersection without the additional support region tightening of CB. [SnugBox](3378-snugbox.md)

## Operating characteristics
- **Time Complexity:** Reduces the number of Gaussian-tile pairs, leading to faster rasterization [S1, unofficial]. The exact complexity reduction depends on the scene and the value of $\beta$ [S1, unofficial].
- **Memory Complexity:** Reduces memory usage associated with storing and processing Gaussian-tile pairs [S1, unofficial].
- **Measured Performance:** The sources do not provide specific performance metrics for CB alone, but FastGS (which integrates CB) achieves an average training time of around 100 seconds per scene [S1, unofficial].
- **Strengths:** Further reduces computational redundancy compared to vanilla 3DGS and Speedy-Splat by eliminating Gaussians with negligible impact on tiles [S1, unofficial].
- **Limitations:** Requires careful tuning of $\beta$ to balance between pruning efficiency and rendering quality [S1, unofficial]. Typical values are not given in the sources [S1, unofficial].
- **When to Use:** Use when rendering efficiency is critical and the scene contains many Gaussians with negligible impact on certain tiles [S1, unofficial].
- **When Not to Use:** Not given in the sources.

## Background
Compact Box (CB) was introduced in the paper "FastGS: Training 3D Gaussian Splatting in 100 Seconds" [S1, unofficial]. It builds upon the precise tile-intersection strategy of Speedy-Splat to further reduce the effective 2D Gaussian region and eliminate unnecessary Gaussian-tile pairs [S1, unofficial]. The method addresses the computational redundancy introduced by the 3-sigma rule in vanilla 3DGS and the remaining negligible Gaussians in Speedy-Splat [S1, unofficial].

## Sources
[S1] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
