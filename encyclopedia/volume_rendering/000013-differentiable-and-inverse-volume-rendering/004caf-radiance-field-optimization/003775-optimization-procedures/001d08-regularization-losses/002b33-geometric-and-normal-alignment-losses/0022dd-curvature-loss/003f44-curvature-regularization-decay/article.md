# Curvature Regularization Decay

Also called: Curvature weight decay, Hash-resolution-dependent curvature scaling

## Overview
Curvature Regularization Decay is a dynamic weighting strategy for curvature-based geometric losses in differentiable volume rendering, specifically designed to preserve high-frequency details during optimization. It addresses the trade-off between surface smoothness and detail retention by scaling down the curvature regularization strength ($w_{curv}$) as the optimization step size ($\epsilon$) decreases and finer hash grid resolutions are activated [S1]. Unlike static curvature losses, which may overly smooth fine structures, this method adapts the regularization intensity to the current spatial resolution [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Current optimization step size $\epsilon$, current hash grid spacing $s_{curr}$, previous hash grid spacing $s_{prev}$, current curvature loss weight $w_{curv}$.
*   **Output:** Updated curvature loss weight $w_{curv}^{new}$.

**Data Structures:**
*   Hash grid parameters: The method relies on the hierarchical structure of the hash grid (e.g., Mip-NeRF 360 or Instant-NGP style), where resolution is controlled by a base resolution and a number of levels. The spacing $s$ is inversely proportional to the resolution at the current level.

**Steps:**
1.  **Monitor Step Size:** Track the optimization step size $\epsilon$. In Neuralangelo, $\epsilon$ typically decreases over time to refine the SDF [S1].
2.  **Detect Resolution Change:** Identify when the hash grid resolution increases (i.e., when $\epsilon$ decreases below a threshold corresponding to a finer grid level). Let $s_{prev}$ be the spacing of the previous (coarser) grid level and $s_{curr}$ be the spacing of the new (finer) grid level [S1].
3.  **Calculate Scaling Factor:** Compute the ratio of the spacings. The scaling factor is typically the ratio $s_{curr} / s_{prev}$ or a power thereof, reflecting the change in spatial granularity [S1].
4.  **Update Weight:** Scale down the curvature regularization strength. The update rule is:
    $$ w_{curv}^{new} = w_{curv}^{old} \times \frac{s_{curr}}{s_{prev}} $$
    This ensures that as the grid becomes finer ($s_{curr} < s_{prev}$), the regularization strength decreases proportionally [S1].

**Parameters:**
*   **Initial $w_{curv}$:** Not specified in the primary source for the decay method itself, but typically set to a small value (e.g., $10^{-4}$ to $10^{-3}$) in conjunction with the warmup strategy [S1].
*   **Step Size Schedule:** The decay is triggered by the decrease in $\epsilon$. The exact schedule for $\epsilon$ is implementation-dependent (e.g., linear or exponential decay) [S1].

**Edge Cases and Pitfalls:**
*   **Premature Decay:** If the weight decays too quickly before the surface is sufficiently smooth, noise may persist.
*   **Insufficient Decay:** If the weight does not decay sufficiently, fine details may be smoothed out as the optimizer attempts to fit high-frequency noise in the fine grids [S1].
*   **Numerical Stability:** Ensure that the spacing ratio is computed accurately to avoid abrupt jumps in the loss weight [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def update_curvature_weight(w_curv, s_prev, s_curr):
    """
    Update curvature regularization weight based on hash grid spacing change.
    
    Args:
        w_curv (float): Current curvature loss weight.
        s_prev (float): Spacing of the previous (coarser) hash grid level.
        s_curr (float): Spacing of the current (finer) hash grid level.
        
    Returns:
        float: Updated curvature loss weight.
    """
    # Scale down the weight by the ratio of spacings
    # As s_curr < s_prev, the weight decreases
    w_curv_new = w_curv * (s_curr / s_prev)
    return w_curv_new
```

## Subtopics

### Curvature Regularization Warmup
A complementary strategy to prevent topology preservation issues during the initial stages of optimization.
*   **What it is:** A warmup phase where the curvature loss strength is linearly increased from zero to its target value over a short initial period.
*   **Why it is used:** Initializing the SDF as a sphere and applying curvature loss from the start preserves topology by preventing singularities in curvature, which makes forming concave shapes difficult [S1].
*   **When to choose:** Use this in conjunction with Curvature Regularization Decay when the target geometry contains significant concave regions or complex topology [S1].
*   **Link:** [Curvature Regularization Decay](2911-curvature-regularization-decay.md)

## Operating characteristics
*   **Time Complexity:** Negligible overhead. The update involves a simple multiplication and comparison of scalar values per step or per resolution change [S1].
*   **Memory Complexity:** No additional memory required beyond storing the current and previous spacing values [S1].
*   **Performance:** Improves image synthesis quality by preserving fine details that would otherwise be smoothed out by static curvature regularization [S1].
*   **Strengths:** Prevents over-smoothing of high-frequency details; adapts to the multi-resolution nature of hash grids [S1].
*   **Limitations:** Requires careful tuning of the initial weight and the step size schedule; may not be necessary for simple, convex geometries [S1].
*   **When to use:** High-fidelity neural surface reconstruction (e.g., Neuralangelo) where fine details are critical [S1].
*   **When not to use:** Simple scenes with smooth, convex surfaces where detail preservation is not a priority [S1].

## Background
Curvature Regularization Decay was introduced in the context of Neuralangelo, a method for high-fidelity neural surface reconstruction [S1]. It builds on the observation that static curvature regularization, while effective at removing surface noise, can inadvertently smooth out fine details as the optimization progresses to finer hash grid resolutions [S1]. The method is part of a broader set of geometric and normal alignment losses used in differentiable volume rendering to regularize the SDF [S1].

## Sources
[S1] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
