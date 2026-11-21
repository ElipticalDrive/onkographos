# Compact Box (CB)


## Overview
Compact Box (CB) is a preprocessing optimization for rasterization that reduces the number of Gaussian-tile pairs by tightening the effective support region of 2D Gaussians, eliminating unnecessary pairs where the Gaussian has negligible impact on pixels within a tile [S1, unofficial]. It builds on [SnugBox](3378-snugbox.md) by introducing a scaling factor $\beta$ into the alpha blending equation, which further shrinks the 2D Gaussian footprint and intersection region with tiles compared to standard 3-sigma or precise tile-intersection methods [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** 2D Gaussian parameters (opacity $\sigma_i$, 2D mean $\mu_{i2D}$, 2D covariance $\Sigma_{i2D}$), tile grid configuration, and scaling factor $\beta$.
- **Output:** A pruned set of relevant Gaussian-tile pairs for rasterization.

**Data Structures:**
The method relies on the standard 2D Gaussian representation used in 3D Gaussian Splatting. The covariance matrix $\Sigma_{i2D}$ is typically decomposed into rotation and scaling components, but for the CB calculation, the inverse covariance $\Sigma_{i2D}^{-1}$ is used directly in the Mahalanobis distance calculation. The specific internal layout of the Gaussian data structure is not detailed in the sources, but it must contain $\sigma_i$, $\mu_{i2D}$, and $\Sigma_{i2D}$ [S1, unofficial].

**Steps:**
1.  **Calculate Modified Alpha:** For a given pixel $p$ and the $i$-th Gaussian, compute the alpha value using the modified equation that incorporates the scaling factor $\beta$:
    $$ \alpha_i(p) = \sigma_i \cdot \exp\left(-\frac{\beta}{2} (p - \mu_{i2D})^\top \Sigma_{i2D}^{-1} (p - \mu_{i2D})\right) $$
    where:
    -   $\sigma_i$ is the opacity of the $i$-th Gaussian.
    -   $\mu_{i2D}$ is the 2D mean position of the $i$-th Gaussian.
    -   $\Sigma_{i2D}$ is the 2D covariance matrix of the $i$-th Gaussian.
    -   $\beta$ is the scaling factor controlling the effective support region.
    -   $(p - \mu_{i2D})^\top \Sigma_{i2D}^{-1} (p - \mu_{i2D})$ is the squared Mahalanobis distance [S1, unofficial].

2.  **Determine Effective Support:** Use the modified alpha equation to define the effective support region of the 2D Gaussian. A smaller $\beta$ yields a tighter ellipse around the mean $\mu_{i2D}$, reducing the spatial extent and limiting the number of pixels influenced by the Gaussian [S1, unofficial].

3.  **Integrate with SnugBox:** Integrate the modified alpha equation into the SnugBox implementation. The parameter $\beta$ further reduces the 2D Gaussian footprint and shrinks its intersection region with tiles [S1, unofficial].

4.  **Prune Gaussian-Tile Pairs:** Use the reduced footprint to determine which Gaussian-tile pairs are relevant. Pairs where the Gaussian has negligible impact on pixels within a tile are eliminated. This is typically done using the `accutile` method from Speedy-Splat, which performs precise tile intersection based on the compacted region [S1, unofficial].

**Parameters:**
-   $\beta$: A scaling factor that flexibly controls the effective support region of each 2D Gaussian. A smaller $\beta$ yields a tighter ellipse around the mean, reducing spatial extent and limiting pixels influenced by the Gaussian [S1, unofficial]. Typical values are not given in the sources.

**Edge Cases and Pitfalls:**
-   **Precision:** The calculation involves matrix inversion ($\Sigma_{i2D}^{-1}$) and exponential functions. Numerical stability should be ensured, particularly for degenerate covariances [G].
-   **Boundary Conditions:** The method relies on the precise tile intersection logic of SnugBox. Incorrect handling of tile boundaries or Gaussian positions near tile edges may lead to incorrect pruning [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compact_box_rasterization(gaussians, tiles, beta):
    """
    Perform rasterization with Compact Box optimization.
    
    Args:
        gaussians: List of Gaussian objects, each with attributes:
                   - sigma: opacity
                   - mu_2d: 2D mean position (vector)
                   - Sigma_2d: 2D covariance matrix
        tiles: List of Tile objects, each with attributes:
               - id: tile identifier
               - bounds: bounding box of the tile
        beta: Scaling factor for effective support region.
        
    Returns:
        List of (gaussian, tile) pairs that are relevant for rasterization.
    """
    relevant_pairs = []
    
    # Precompute inverse covariance for each Gaussian
    for g in gaussians:
        try:
            g.Sigma_2d_inv = inverse(g.Sigma_2d)
        except SingularMatrixError:
            # Handle degenerate covariance [G]
            continue
            
    for g in gaussians:
        # Calculate effective support region based on beta
        # The effective support is defined by the region where alpha is significant.
        # In practice, this is used to shrink the bounding box used for tile intersection.
        # The exact shrinkage logic is encapsulated in the SnugBox integration.
        
        # Use SnugBox to find intersecting tiles with the compacted footprint
        # This step relies on the Speedy-Splat accutile method [S1, unofficial]
        intersecting_tiles = snugbox_accutile(g, tiles, beta)
        
        for tile in intersecting_tiles:
            relevant_pairs.append((g, tile))
            
    return relevant_pairs

def compute_alpha_modified(p, g, beta):
    """
    Compute the modified alpha value for a pixel and a Gaussian.
    
    Args:
        p: Pixel position (vector).
        g: Gaussian object.
        beta: Scaling factor.
        
    Returns:
        Alpha value.
    """
    diff = p - g.mu_2d
    mahalanobis_dist = dot(diff, multiply(g.Sigma_2d_inv, diff))
    alpha = g.sigma * exp(-0.5 * beta * mahalanobis_dist)
    return alpha
```

## Subtopics

### SnugBox
A precise algorithm for computing Gaussian-tile bounding box intersections, replacing the overly conservative tile assignment in standard 3D-GS to reduce the number of pixels processed per Gaussian. Choose when you need precise tile intersection without the additional support region tightening of CB. [SnugBox](3378-snugbox.md)

## Operating characteristics
-   **Time Complexity:** Reduces the number of Gaussian-tile pairs, leading to faster rasterization. The exact complexity reduction depends on the scene and the value of $\beta$ [S1, unofficial].
-   **Memory Complexity:** Reduces memory usage associated with storing and processing Gaussian-tile pairs [S1, unofficial].
-   **Measured Performance:** The sources do not provide specific performance metrics (e.g., FPS, training time) for CB alone, but FastGS (which integrates CB) achieves an average training time of around 100 seconds per scene [S1, unofficial].
-   **Strengths:** Further reduces computational redundancy compared to vanilla 3DGS and Speedy-Splat by eliminating Gaussians with negligible impact on tiles [S1, unofficial].
-   **Limitations:** Requires careful tuning of $\beta$ to balance between pruning efficiency and rendering quality. A too-small $\beta$ may lead to visual artifacts due to excessive pruning [S1, unofficial].
-   **When to Use:** Use when rendering efficiency is critical and the scene contains many Gaussians with negligible impact on certain tiles.
-   **When Not to Use:** Avoid if high visual fidelity is paramount and the overhead of tuning $\beta$ is not acceptable.

## Background
Compact Box (CB) was introduced in the paper "FastGS: Training 3D Gaussian Splatting in 100 Seconds" [S1, unofficial]. It builds upon the precise tile-intersection strategy of Speedy-Splat [12] to further reduce the effective 2D Gaussian region and eliminate unnecessary Gaussian-tile pairs. The method addresses the computational redundancy introduced by the 3-sigma rule in vanilla 3DGS [17] and the remaining negligible Gaussians in Speedy-Splat [S1, unofficial].

## Sources
[S1] FastGS: Training 3D Gaussian Splatting in 100 Seconds, https://doi.org/10.48550/arxiv.2511.04283, fetched 2026-10-26 (unofficial)
