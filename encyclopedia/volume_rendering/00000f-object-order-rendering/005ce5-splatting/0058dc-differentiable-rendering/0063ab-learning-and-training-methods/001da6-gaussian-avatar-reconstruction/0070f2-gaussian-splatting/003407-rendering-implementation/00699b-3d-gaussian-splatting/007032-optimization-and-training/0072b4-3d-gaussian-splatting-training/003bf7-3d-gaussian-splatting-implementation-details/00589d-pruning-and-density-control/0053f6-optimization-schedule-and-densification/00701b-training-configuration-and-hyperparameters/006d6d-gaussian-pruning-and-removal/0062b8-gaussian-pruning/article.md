# Gaussian Pruning

Also called: Dominant Gaussian Retention, TopK Pruning

## Overview
Gaussian Pruning is a post-densification strategy that reduces the set of 3D Gaussians $\mathcal{G}$ by retaining only those "dominant" primitives that contribute most significantly to the final pixel color for each ray, while removing "auxiliary" Gaussians that lack real geometric meaning [S1]. It sets itself apart from sibling methods like [Gaussian Splitting](link) or [Random Pruning](link) by using a per-ray contribution metric derived from the alpha-blending process to identify redundancy, rather than relying on spatial density or random selection [S1]. This approach dramatically reduces Gaussian count and storage overhead while maintaining rendering quality comparable to the full set [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** The current set of 3D Gaussians $\mathcal{G}$ after densification steps, the camera parameters for all training views, and the hyperparameter $K$ (number of dominant Gaussians per ray).
*   **Output:** A pruned subset $\mathcal{G}' \subset \mathcal{G}$ containing only retained Gaussians.

**Data Structures:**
*   $\mathcal{G}$: List of Gaussians. Each Gaussian $g_i$ contains mean $\mu_i$, covariance $\Sigma_i$ (or rotation $R_i$ and scale $S_i$), opacity $\alpha_i$, and spherical harmonic coefficients for color.
*   $N_p$: The set of Gaussians overlapping pixel $p$ (or ray $r$), sorted by depth (distance from camera) in ascending order.

**Parameters:**
*   $K$: The number of top-weight Gaussians to retain per pixel as dominant. Typical value: 1 [S1]. Smaller $K$ enforces stricter filtering (more removal, potential quality degradation); larger $K$ retains more Gaussians (higher quality) [S1].

**Steps:**
1.  **Weight Calculation:** For each pixel $p$ in all training images, identify the set of overlapping Gaussians $N_p$. For each Gaussian $g_i \in N_p$, calculate its contribution weight $w_i^p$ to the pixel color using the alpha-blending formula [S1]:
    $$
    w_i^p = \alpha_i \prod_{j=1}^{i-1} (1 - \alpha_j)
    $$
    where $\alpha_i$ is the opacity of $g_i$, and the product is over all Gaussians $g_j \in N_p$ that precede $g_i$ in depth order [S1].

2.  **Ranking:** For each pixel $p$, sort the weights of all Gaussians in $N_p$ in ascending order. Identify the threshold weight corresponding to the $K$-th largest weight [S1]. This is equivalent to finding the $K$-th element in the sorted list of negative weights, or simply the $K$-th largest value in the list of positive weights [S1].

3.  **Selection:** A Gaussian $g$ is marked for retention if it is identified as one of the top-$K$ dominant Gaussians for at least one pixel $p$ across all training images [S1]. Formally, $g$ is retained if:
    $$
    \exists p \text{ such that } w_g^p \geq \text{sort}(\{-w_j^p \mid g_j \in N_p\})[K]
    $$
    where $\text{sort}(\dots)[K]$ returns the $K$-th largest weight among the overlapping Gaussians for pixel $p$ [S1].

4.  **Removal:** All Gaussians not marked for retention are deleted from $\mathcal{G}$ [S1].

5.  **Post-Pruning Optimization:** Continue optimizing the colors (and potentially other parameters) of the retained dominant Gaussians. This allows them to gradually absorb the rendering workload previously handled by the deleted auxiliary Gaussians [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_pruning(Gaussians, Images, K=1):
    """
    Prune Gaussians based on dominant contribution per ray.
    
    Args:
        Gaussians: List of Gaussian objects, each with attributes:
                   - id: unique identifier
                   - alpha: opacity scalar
                   - depth: distance from camera (pre-computed or computed per view)
        Images: List of image objects, each with:
                - width, height
                - rays: list of ray objects, each with:
                       - pixel_coords: (u, v)
                       - overlapping_gaussians: list of Gaussian IDs sorted by depth
        K: int, number of dominant Gaussians to keep per ray.
        
    Returns:
        Pruned list of Gaussian IDs.
    """
    retained_ids = set()
    
    # Step 1 & 2: Calculate weights and determine dominant Gaussians per pixel
    for image in Images:
        for ray in image.rays:
            overlapping_gaussians = ray.overlapping_gaussians
            if not overlapping_gaussians:
                continue
                
            # Calculate contribution weight for each overlapping Gaussian
            # w_i = alpha_i * product_{j < i} (1 - alpha_j)
            weights = []
            accumulated_transmittance = 1.0
            
            for g_id in overlapping_gaussians:
                g = Gaussians[g_id]
                weight = g.alpha * accumulated_transmittance
                weights.append((weight, g_id))
                accumulated_transmittance *= (1.0 - g.alpha)
            
            # Step 3: Identify top-K dominant Gaussians
            # Sort weights in descending order to get top-K
            # Note: The source uses sort({-weight})[K] which is equivalent to 
            # taking the K-th largest positive weight.
            weights.sort(key=lambda x: x[0], reverse=True)
            
            # Retain the top K Gaussians for this ray
            for i in range(min(K, len(weights))):
                retained_ids.add(weights[i][1])
                
    # Step 4: Return the set of retained Gaussian IDs
    return list(retained_ids)
```

**Edge Cases and Pitfalls:**
*   **Precision:** The product term $\prod (1 - \alpha_j)$ can underflow to zero if many opaque Gaussians precede a Gaussian in the depth order. This results in a weight of zero, potentially causing a valid Gaussian to be pruned if it is not dominant in other rays [S1]. Implementers should use log-space arithmetic or check for transmittance near zero [G].
*   **Boundary Conditions:** If a ray overlaps with fewer than $K$ Gaussians, all overlapping Gaussians are retained [S1].
*   **Degenerate Input:** If all Gaussians have zero opacity, weights are zero, and the selection logic may behave unpredictably depending on the sorting implementation. However, zero-opacity Gaussians are typically removed during initialization or other pruning steps [G].

## Operating characteristics
*   **Time Complexity:** The pruning step requires iterating over all pixels and their overlapping Gaussians. For each pixel, sorting the overlapping Gaussians takes $O(N_p \log N_p)$, where $N_p$ is the number of overlapping Gaussians per pixel. The total time is proportional to the sum of $N_p \log N_p$ over all pixels. This is performed once per pruning interval (e.g., every few thousand iterations) [S1].
*   **Memory Complexity:** Requires storing the weights for all overlapping Gaussians during the pass. This is $O(\sum N_p)$, which is bounded by the total number of Gaussian-ray intersections [S1].
*   **Measured Performance:** On the Mip-NeRF360 dataset with $K=1$, the method achieves SSIM 0.817, PSNR 27.38, LPIPS 0.216, Training Time 19m41s, FPS 218, and Storage 98MB [S1]. Increasing $K$ to 2 yields SSIM 0.818, PSNR 27.43, LPIPS 0.210, Training Time 20m44s, FPS 185, and Storage 123MB [S1]. Increasing $K$ to 3 yields SSIM 0.818, PSNR 27.45, LPIPS 0.209, Training Time 21m07s, FPS 174, and Storage 137MB [S1].
*   **Strengths:** Dramatically reduces Gaussian count and storage overhead (e.g., from 742MB to 98MB in ablation studies) while maintaining nearly identical rendering quality to vanilla 3DGS [S1]. Enables real-time rendering for large-scale scenes [S1].
*   **Limitations:** Requires a full forward pass through all training images to compute weights, which can be memory-intensive for very large scenes if not optimized [S1]. The quality degradation from aggressive pruning (small $K$) is non-linear; $K=1$ is recommended as a balance [S1].
*   **When to Use:** Use when storage or rendering speed is a bottleneck, particularly for large-scale scenes. It is most effective after densification has created redundant auxiliary Gaussians [S1].
*   **When Not to Use:** Avoid if the scene has very few overlapping Gaussians per ray (e.g., sparse scenes), as the pruning benefit is minimal. Also avoid if strict quality preservation is required without any trade-off, as aggressive pruning (low $K$) may slightly degrade metrics [S1].

## Background
Gaussian Pruning was introduced in the context of streamlining 3D Gaussian Splatting for large-scale high-resolution scene representation. It draws inspiration from NeRF importance sampling strategies [S1]. The method addresses the redundancy introduced by densification processes (like splitting), which can generate many auxiliary Gaussians that lack clear geometric meaning [S1]. By retaining only the dominant contributors per ray, the method simplifies the scene representation, reducing computational load and storage requirements while preserving visual fidelity [S1]. It is typically applied after selective densification and before sparse order increment for spherical harmonics [S1].

## Sources
[S1] EfficientGS: Streamlining Gaussian Splatting for Large-Scale High-Resolution Scene Representation, https://doi.org/10.1109/mmul.2025.3543224, fetched 2026-10-30
