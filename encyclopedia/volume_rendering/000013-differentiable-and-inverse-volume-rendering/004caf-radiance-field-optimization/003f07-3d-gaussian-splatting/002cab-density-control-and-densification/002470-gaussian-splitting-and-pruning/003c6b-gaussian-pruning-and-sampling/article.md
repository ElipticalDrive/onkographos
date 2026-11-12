# Gaussian Pruning and Sampling

Also called: GRay pruning, Importance-based pruning, Weight pruning, Weight-Based Pruning

## Overview
Gaussian Pruning and Sampling is a density control strategy that reduces the number of 3D Gaussian primitives in a radiance field by removing those with minimal contribution to the rendered image [S1, S2]. It solves the problem of redundant Gaussian representation and inefficient spatial distribution, which leads to high storage costs and slow optimization times [S1, S2]. Unlike opacity-based pruning, which can cause noticeable quality drops or require aggressive thresholds, weight-based pruning evaluates the accumulated blending weights of Gaussians across views, balancing rendering quality with computational efficiency [S2].

## How it works
**Inputs:** A set of 3D Gaussians $\{G_i\}$, a set of training views $V$, and the accumulated blending weights $w_{ij}$ or $w_i(r)$ for each Gaussian-ray intersection [S1, S2].
**Outputs:** A pruned set of Gaussians $\{G_i\}'$ where low-importance primitives are removed [S1, S2].

**Data Structures:**
*   **Gaussian List:** An array or list of Gaussian objects storing parameters (position, covariance, opacity, color) and an accumulated importance score $I_i$ [G].
*   **Accumulated Weights:** A scalar value $I_i$ associated with each Gaussian, initialized to 0 before each pruning cycle [G].

**Steps:**
1.  **Accumulate Importance Scores:** At specified intervals, track ray intersections during the forward rendering pass and sum the blending weights for each Gaussian [S1, S2].
2.  **Evaluate Pruning Condition:** Compare the accumulated importance score $I_i$ against a pruning threshold $\gamma_{\text{prune}}$ [S1, S2].
3.  **Remove Gaussians:** Delete all Gaussians where $I_i < \gamma_{\text{prune}}$ from the representation [S1, S2].
4.  **Optimize:** Continue optimizing the remaining Gaussian parameters until convergence or the next pruning interval [S1].

**Formulas:**
*   **Importance Score (S1):**
    $$ I_i = \sum_{j=1}^{K} w_{ij} $$
    Where $I_i$ is the importance score of Gaussian $G_i$, $w_{ij}$ is the blending weight of $G_i$ for ray $j$, and $K$ is the total number of rays intersected with $G_i$ [S1].
*   **Pruning Condition (S2):**
    $$ \left[ \frac{1}{|V_i| \cdot |v|} \sum_{v \in V_i} \sum_{r \in v} w_i(r) \right] < \gamma_{\text{prune}} $$
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
def prune_gaussians(gaussians, accumulated_contributions, current_iter, frequency=500, threshold=1e-7):
    if current_iter % frequency != 0:
        return gaussians
    for g in gaussians:
        g.importance = 0.0
    # Sources do not specify the exact intersection tracking mechanism during rendering [G].
    # Accumulate blending weights for each Gaussian across all rays.
    for g_idx, weight in accumulated_contributions:
        gaussians[g_idx].importance += weight
    pruned_gaussians = [g for g in gaussians if g.importance >= threshold]
    return pruned_gaussians
```

## Subtopics

### [Contribution-based Pruning](9445-contribution-based-pruning.md)
A strategy that jointly evaluates opacity and spatial scale to identify insignificant Gaussians, balancing rendering quality and computational overhead better than pruning based on a single attribute.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Measured Performance:**
    *   With pruning ($\gamma_{\text{prune}}=10^{-7}$): PSNR 26.47, LPIPS 0.236, Opt Time 05:40, FPS 248, Final Gaussians 1.52M [S2].
    *   Without pruning: PSNR 26.42, LPIPS 0.234, Opt Time 07:26, FPS 189, Final Gaussians 3.27M [S2].
    *   Simple pruning (S1): Reduced Gaussians from 6.1M to 0.6M, but degraded PSNR from 25.2 dB to 24.9 dB [S1].
*   **Strengths:** Substantial improvement in FPS and optimization time while maintaining or slightly improving PSNR [S2]. Outperforms opacity-based pruning in terms of training overhead and final FPS [S2].
*   **Limitations:** Does not address spatial distribution issues like overlapping or under-reconstruction [S1]. Simple pruning can degrade quality if the threshold is too high or the reduction is too aggressive [S1].
*   **When to use:** When computational efficiency (FPS, optimization time) is a priority and some quality loss is acceptable, or when the Gaussian count is excessively high [S1, S2].
*   **When not to use:** When maximum reconstruction quality is required and computational resources are not a constraint, or when the scene has complex geometry that requires dense Gaussian coverage [S1].

## Background
Gaussian Pruning and Sampling originates from the need to manage the redundancy in 3D Gaussian Splatting representations [S1]. The method was introduced in [S1] as a straightforward approach to reduce the number of Gaussians by calculating an importance score based on accumulated blending weights. It was further refined in [S2] (GRay) to use an average accumulated weight criterion, which proved more effective in maintaining quality while reducing computational cost [S2].

## Sources
[S1] Mini-Splatting: Representing Scenes with a Constrained Number of Gaussians, https://doi.org/10.1007/978-3-031-72980-5_10, fetched 2026-10-26
[S2] GRay: Ray Tracing 3D Gaussians Near the Speed of Splats, https://doi.org/10.1145/3804496, fetched 2026-10-29
