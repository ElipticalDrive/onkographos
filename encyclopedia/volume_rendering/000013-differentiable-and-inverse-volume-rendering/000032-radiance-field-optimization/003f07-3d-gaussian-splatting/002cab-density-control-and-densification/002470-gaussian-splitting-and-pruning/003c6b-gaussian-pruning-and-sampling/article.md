# Gaussian Pruning and Sampling

Also called: Importance-based pruning

## Overview
Gaussian Pruning and Sampling is a post-processing or intermediate optimization strategy in 3D Gaussian Splatting that reduces the number of Gaussian primitives by removing those with low contribution to the final image. It addresses the inherent redundancy in dense Gaussian representations initialized from point clouds. Unlike [Gaussian Splitting and Pruning](3304-gaussian-splitting-and-pruning.md), which dynamically adjusts density during training based on gradient-based splitting criteria, this method relies on accumulated rendering weights to identify and discard insignificant Gaussians. It is often combined with sampling techniques to mitigate the loss of geometric detail.

## How it works
**Inputs:** A set of 3D Gaussians $\{G_i\}$, a set of training rays $\{R_j\}$, and the rendered blending weights $w_{ij}$ for each Gaussian $i$ along each ray $j$.
**Outputs:** A pruned set of Gaussians $\{G_i'\}$ where $I_i \geq \tau$.

**Data Structures:**
-   **Gaussian List:** Array of Gaussian objects. Each object $G_i$ contains:
    -   `position`: 3D vector $(x, y, z)$
    -   `scale`: 3D vector (or quaternion + scale)
    -   `opacity`: scalar $\alpha_i \in [0, 1]$
    -   `colors`: spherical harmonic coefficients
    -   `importance_score`: scalar $I_i$ (accumulated during forward pass)
-   **Ray Buffer:** Stores the blending weight $w_{ij}$ contributed by Gaussian $i$ to ray $j$.

**Steps:**
1.  **Accumulate Importance Scores:** During the standard differentiable rendering pass, for each ray $j$ intersecting Gaussian $G_i$, accumulate the blending weight $w_{ij}$ into the Gaussian's importance score $I_i$.
    $$ I_i = \sum_{j=1}^{K} w_{ij} $$
    Where:
    -   $I_i$ is the total importance score of Gaussian $G_i$.
    -   $w_{ij}$ is the blending weight of Gaussian $G_i$ for ray $j$.
    -   $K$ is the total number of rays intersected by $G_i$.
    -   Note: $w_{ij}$ is typically derived from the Gaussian's opacity and depth-dependent attenuation during the alpha-blending process [S1].

2.  **Thresholding:** Define a pruning threshold $\tau$. Compare each $I_i$ against $\tau$.
    -   If $I_i < \tau$, mark $G_i$ for removal.
    -   If $I_i \geq \tau$, retain $G_i$.

3.  **Pruning:** Remove all marked Gaussians from the scene representation.

4.  **Re-optimization:** Continue training the remaining Gaussians (optimizing positions, scales, opacities, and colors) until convergence. This step is crucial to recover quality lost during pruning [S1].

5.  **Sampling (Optional):** To address spatial distribution inefficiencies, combine pruning with point cloud sampling techniques (e.g., random, grid, or density-preserved sampling) before or after pruning to ensure uniform coverage [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gaussian_pruning_and_sampling(gaussians, rays, threshold):
    """
    Prunes Gaussians based on accumulated blending weights.
    
    Args:
        gaussians: List of Gaussian objects with attributes:
                   - position, scale, opacity, colors
                   - importance_score (initialized to 0)
        rays: List of Ray objects. Each Ray has a list of intersected Gaussians.
              For each intersection, the renderer computes a blending weight w_ij.
        threshold: Float, the minimum importance score to keep a Gaussian.
        
    Returns:
        pruned_gaussians: List of Gaussian objects that survived pruning.
    """
    # Step 1: Accumulate importance scores
    # During the forward rendering pass, for every ray j and every Gaussian i 
    # intersected by that ray, add the blending weight w_ij to i.importance_score.
    # This is typically done inside the rasterizer/render loop.
    for ray in rays:
        for gaussian in ray.intersected_gaussians:
            # w_ij is computed by the standard 3DGS alpha-blending formula
            w_ij = compute_blending_weight(gaussian, ray) 
            gaussian.importance_score += w_ij

    # Step 2: Filter Gaussians
    pruned_gaussians = []
    for gaussian in gaussians:
        if gaussian.importance_score >= threshold:
            pruned_gaussians.append(gaussian)
        # Else: Gaussian is pruned (removed)

    # Step 3: Re-optimization
    # The remaining Gaussians are optimized using standard 3DGS loss functions
    # (L1, D-SSIM) until convergence. This is handled by the main training loop.
    # [G] Implementation note: Ensure gradients are correctly backpropagated
    # through the remaining Gaussians after pruning.

    return pruned_gaussians
```

**Parameters:**
-   **Threshold ($\tau$):** Not explicitly defined as a fixed value in the source, but described as "a certain threshold." The source notes a reduction from 6.1 million to 0.6 million Gaussians, implying a high pruning ratio [S1].
-   **Sampling Method:** Optional. Options include random, grid, or density-preserved sampling [S1].

**Edge Cases and Pitfalls:**
-   **Quality Degradation:** Pruning alone degrades PSNR (e.g., from 25.2 dB to 24.9 dB in the cited experiment) [S1].
-   **Spatial Inefficiency:** Pruning does not fix "overlapping" or "under-reconstruction" phenomena. It reduces count but may leave inefficient spatial distributions [S1].
-   **Convergence Requirement:** Re-optimization is necessary to recover quality; skipping it leads to suboptimal results [S1].

## Subtopics

### [Contribution-based Pruning](9445-contribution-based-pruning.md)
A variant that jointly evaluates opacity and spatial scale to determine redundancy, balancing rendering quality and computational overhead better than opacity-only or scale-only pruning.

## Operating characteristics
-   **Time Complexity:** $O(N \cdot K)$ for accumulation, where $N$ is the number of Gaussians and $K$ is the average number of rays per Gaussian. Pruning is $O(N)$. Re-optimization cost depends on the number of training steps required for convergence.
-   **Memory Complexity:** $O(N)$ for storing Gaussians. Pruning reduces memory footprint proportionally to the number of removed Gaussians.
-   **Measured Performance:** In the cited study, pruning reduced the Gaussian count from 6.1 million to 0.6 million (approx. 90% reduction) but degraded PSNR from 25.2 dB to 24.9 dB [S1].
-   **Strengths:** Simple to implement; significantly reduces storage and rendering time; effective for reducing redundancy in dense initializations.
-   **Limitations:** Does not improve spatial distribution; causes noticeable quality drop if not followed by re-optimization or sampling; sensitive to threshold selection.
-   **When to Use:** When storage or rendering speed is critical and a small PSNR drop is acceptable; when initializing from very dense point clouds.
-   **When Not to Use:** When high fidelity is required without re-optimization; when spatial distribution is already inefficient (use [Gaussian Splitting and Pruning](3304-gaussian-splitting-and-pruning.md) or [Contribution-based Pruning](9445-contribution-based-pruning.md) instead).

## Background
Originated in the context of 3D Gaussian Splatting (3DGS) optimization, specifically in the "Mini-Splatting" paper [S1]. The method addresses the redundancy inherent in initializing 3DGS from dense point clouds. It builds on the importance calculation concept from earlier works [21] (cited in [S1]).

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
