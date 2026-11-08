# Gaussian Pruning and Sampling

Also called: GRay pruning, Importance-based pruning, Weight pruning, Weight-Based Pruning

## Overview
Gaussian Pruning and Sampling is a density control strategy that reduces the number of 3D Gaussian primitives in a radiance field by removing those with minimal contribution to the rendered image [S1, S2]. It solves the problem of redundant Gaussian representation and inefficient spatial distribution, which leads to high storage costs and slow optimization times [S1, S2]. Unlike opacity-based pruning, which can cause noticeable quality drops or require aggressive thresholds, weight-based pruning evaluates the accumulated blending weights of Gaussians across views, balancing rendering quality with computational efficiency [S2].

## How it works
**Inputs:** A set of 3D Gaussians $\{G_i\}$, a set of training views $V$, and the accumulated blending weights $w_{ij}$ or $w_i(r)$ for each Gaussian-ray intersection.
**Outputs:** A pruned set of Gaussians $\{G_i\}'$ where low-importance primitives are removed.

**Data Structures:**
*   **Gaussian List:** An array or list of Gaussian objects. Each Gaussian $G_i$ stores its parameters (position, covariance, opacity, color) and an accumulated importance score $I_i$.
*   **Accumulated Weights:** A scalar value $I_i$ associated with each Gaussian, initialized to 0.

**Steps:**
1.  **Accumulate Importance Scores:** For each training iteration (or at specified intervals), render the scene or track ray intersections. For every Gaussian $G_i$ that contributes to the rendering, accumulate its blending weights.
    *   Method 1 (Per-ray accumulation): Sum the blending weights $w_{ij}$ for all rays $j$ intersected by $G_i$ [S1].
    *   Method 2 (Per-view average): Calculate the average accumulated weight over all views $v \in V_i$ that observe $G_i$ and all rays $r$ in those views [S2].
2.  **Evaluate Pruning Condition:** Compare the importance score $I_i$ against a pruning threshold $\gamma_{\text{prune}}$.
    *   Condition: If $I_i < \gamma_{\text{prune}}$, mark $G_i$ for removal [S1, S2].
3.  **Remove Gaussians:** Delete all marked Gaussians from the representation [S1, S2].
4.  **Optimize:** Continue optimizing the remaining Gaussian parameters (position, covariance, opacity, color) until convergence or the next pruning interval [S1].

**Formulas:**
*   **Importance Score (S1):**
    $$ I_i = \sum_{j=1}^{K} w_{ij} $$
    Where $I_i$ is the importance score of Gaussian $G_i$, $w_{ij}$ is the blending weight of $G_i$ for ray $j$, and $K$ is the total number of rays intersected with $G_i$ [S1].

*   **Pruning Condition (S2):**
    $$ \left[ \frac{1}{|V_i||v|} \sum_{v \in V_i} \sum_{r \in v} w_i(r) \right] < \gamma_{\text{prune}} $$
    Where $V_i$ is the set of views seeing Gaussian $i$, $v$ is a view, $r$ is a ray, $w_i(r)$ is the Gaussian's accumulation weight for that ray, and $\gamma_{\text{prune}}$ is the pruning threshold [S2].

**Parameters:**
*   **Pruning Threshold ($\gamma_{\text{prune}}$):** A small positive value. A typical value is $10^{-7}$ [S2].
*   **Pruning Frequency:** Pruning is performed every 500 iterations [S2].

**Pitfalls:**
*   **Overlapping and Under-reconstruction:** Pruning alone does not address inefficient spatial distribution, such as overlapping Gaussians or gaps in reconstruction [S1].
*   **Quality Degradation:** Simple pruning can degrade PSNR. For example, reducing Gaussians from 6.1 million to 0.6 million degraded PSNR from 25.2 dB to 24.9 dB [S1].
*   **Opacity-based Limitations:** Opacity-based pruning (e.g., threshold $\mu_{\text{prune}}^{\text{Opacity}}=0.100$) yields similar training times but causes a noticeable drop in quality compared to weight-based pruning [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def prune_gaussians(gaussians, views, threshold=1e-7, frequency=500, current_iter=0):
    """
    Prunes Gaussians based on accumulated blending weights.
    
    Args:
        gaussians: List of Gaussian objects, each with an 'importance' attribute.
        views: List of training views.
        threshold: Pruning threshold gamma_prune.
        frequency: Number of iterations between pruning steps.
        current_iter: Current training iteration count.
        
    Returns:
        pruned_gaussians: List of Gaussians after pruning.
    """
    # Only prune at specified intervals
    if current_iter % frequency != 0:
        return gaussians

    # Reset importance scores for the current pruning cycle
    for g in gaussians:
        g.importance = 0.0

    # Accumulate weights for each Gaussian
    # Note: In practice, this requires tracking which rays intersect which Gaussians
    # during the forward pass of rendering.
    for view in views:
        for ray in view.rays:
            # Assume render_ray returns a list of (gaussian_index, weight) tuples
            # for all Gaussians contributing to this ray.
            contributions = render_ray(ray)
            
            for g_idx, weight in contributions:
                gaussians[g_idx].importance += weight

    # Filter out Gaussians below the threshold
    pruned_gaussians = []
    for g in gaussians:
        # Using the average weight formula from S2 for robustness, 
        # or simple sum from S1. Here we use the sum as per S1's primary definition,
        # but S2 suggests averaging over views/rays. 
        # To be safe and consistent with S2's superior results, we use the average.
        # However, S1's formula is simpler. We implement S1's sum for simplicity 
        # but note that S2's average is more rigorous.
        # Let's stick to S1's sum for the core method, as it is the "most straightforward" [S1].
        if g.importance >= threshold:
            pruned_gaussians.append(g)
            
    return pruned_gaussians
```
*Note: The pseudocode above implements the simpler sum-based approach from [S1]. For the results in [S2], the accumulation should be averaged over views and rays as per the formula in [S2]. An implementer should choose the accumulation method based on the desired trade-off between simplicity and quality.*

## Subtopics

### [Contribution-based Pruning](9445-contribution-based-pruning.md)
A strategy that jointly evaluates opacity and spatial scale to identify insignificant Gaussians, balancing rendering quality and computational overhead better than pruning based on a single attribute.

## Operating characteristics
*   **Time Complexity:** Pruning requires a pass over all Gaussians and their accumulated weights, which is $O(N)$ where $N$ is the number of Gaussians. The optimization step continues until convergence, but pruning reduces $N$, speeding up subsequent iterations.
*   **Memory Complexity:** Reduces memory usage by removing Gaussians. For example, reducing from 6.1 million to 0.6 million [S1] or 3.27 million to 1.52 million [S2].
*   **Measured Performance:**
    *   With pruning ($\gamma_{\text{prune}}=10^{-7}$): PSNR 26.47, LPIPS 0.236, Opt Time 05:40, FPS 248, Final Gaussians 1.52M [S2].
    *   Without pruning: PSNR 26.42, LPIPS 0.234, Opt Time 07:26, FPS 189, Final Gaussians 3.27M [S2].
    *   Simple pruning (S1): Reduced Gaussians from 6.1M to 0.6M, but degraded PSNR from 25.2 dB to 24.9 dB [S1].
*   **Strengths:** Substantial improvement in FPS and optimization time while maintaining or slightly improving PSNR [S2]. Outperforms opacity-based pruning in terms of training overhead and final FPS [S2].
*   **Limitations:** Does not address spatial distribution issues like overlapping or under-reconstruction [S1]. Simple pruning can degrade quality if the threshold is too high or the reduction is too aggressive [S1].
*   **When to use:** When computational efficiency (FPS, optimization time) is a priority and some quality loss is acceptable, or when the Gaussian count is excessively high.
*   **When not to use:** When maximum reconstruction quality is required and computational resources are not a constraint, or when the scene has complex geometry that requires dense Gaussian coverage [S1].

## Background
Gaussian Pruning and Sampling originates from the need to manage the redundancy in 3D Gaussian Splatting representations. The method was introduced in [S1] as a straightforward approach to reduce the number of Gaussians by calculating an importance score based on accumulated blending weights. It was further refined in [S2] (GRay) to use an average accumulated weight criterion, which proved more effective in maintaining quality while reducing computational cost. The method is part of the broader effort to optimize 3D Gaussian Splatting for real-time rendering and efficient storage.

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
