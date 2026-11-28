# Depth-Based Gaussian Pruning

Also called: Depth-adaptive pruning, relative depth pruning

## Overview
Depth-Based Gaussian Pruning is a strategy that varies the pruning threshold based on the relative depth (z-axis value) of 3D Gaussians to preserve points at the far plane in forward-facing scenes [S1]. It distinguishes itself from standard uniform pruning by applying a lower threshold (more aggressive pruning) to near-plane Gaussians and a higher threshold (more conservative pruning) to far-plane Gaussians, thereby compensating for the sparsity of primitives at the scene's depth extremes [S1].

## How it works
**Inputs:**
*   Set of 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_i$ has position $\mathbf{p}_i = (x_i, y_i, z_i)$ and opacity $\alpha_i$.
*   Base pruning threshold $t_p$.
*   Depth-based multiplier $w_p$.

**Outputs:**
*   Pruned set of 3D Gaussians $\mathcal{G}' \subset \mathcal{G}$.

**Data Structures:**
*   The Gaussian attributes are stored in contiguous arrays (e.g., Structure of Arrays) for efficient access during the pruning pass. The position array contains the z-coordinate $z_i$ for each Gaussian.

**Steps:**
1.  **Calculate Relative Depth:** For each Gaussian $g_i$, determine its relative depth. In forward-facing scenes, this is defined as the z-axis value of the Gaussian's position [S1]. Let $z_i$ be this value.
2.  **Determine Depth Multiplier:** Calculate the multiplier $w(z_i)$ based on the relative depth $z_i$. The method specifies that the multiplier is determined depending on the relative depth, such that the lowest threshold is applied to the farthest point [S1]. While the exact functional form $w(z)$ is not explicitly defined in the source, the implementation implies a monotonic relationship where $w(z)$ increases or decreases to adjust the threshold. The source states $w_p$ is the multiplier value set to 3, and the lowest threshold is applied to the farthest point [S1]. This implies the effective threshold $t_{eff}$ is scaled by $w_p$ relative to the base $t_p$. Specifically, the text says "the pruning threshold $t_p$ is scaled by $w_p$ where $w_p$ is determined depending on the relative depth, and the lowest threshold is applied to the farthest point" [S1]. This suggests the effective threshold $t_{eff}(z_i)$ is derived from $t_p$ and $w_p$. A common interpretation of "scaled by $w_p$" with "lowest threshold at farthest" is that the threshold decreases as depth increases. However, the text says "lowest threshold is applied to the farthest point" which means $t_{eff}$ is smallest for large $z$. The multiplier $w_p=3$ is given. It is likely $t_{eff}(z_i) = t_p \cdot f(z_i)$ where $f(z_i)$ ranges such that the minimum is $t_p/w_p$ or similar. Given the ambiguity, we define the effective threshold $t_{eff}(z_i)$ as a function that yields the lowest value for the maximum $z_i$.
    *   *Clarification [G]:* The source states "lowest threshold is applied to the farthest point". If $t_p$ is the base, and $w_p=3$ is the multiplier, and the threshold is *lowest* at the far end, the effective threshold might be $t_{eff}(z_i) = t_p / (1 + w_p \cdot \frac{z_i - z_{min}}{z_{max} - z_{min}})$ or similar scaling. However, without the exact formula, we state the logic: The threshold is adjusted per Gaussian based on its z-depth.
3.  **Evaluate Pruning Condition:** For each Gaussian $g_i$, compare its opacity $\alpha_i$ against the depth-dependent effective threshold $t_{eff}(z_i)$.
4.  **Prune:** If $\alpha_i < t_{eff}(z_i)$, remove $g_i$ from the set.
5.  **Compaction:** Remove the pruned Gaussians from the data structure to maintain contiguous memory access for the rasterizer [Parent].

**Parameters:**
*   $t_p$: Base 3D Gaussian pruning threshold.
    *   Value: $5 \times 10^{-3}$ for real defocus blur dataset [S1].
    *   Value: $1 \times 10^{-2}$ for real camera motion blur dataset [S1].
*   $w_p$: Pruning threshold multiplier based on depth.
    *   Value: 3 [S1].

**Edge Cases and Pitfalls:**
*   **Forward-Facing Assumption:** This method relies on the z-axis value representing relative depth from any viewpoint. It is not suitable for non-forward-facing or omnidirectional scenes where z-depth does not correlate with distance from the camera center [S1].
*   **Threshold Selection:** The choice of $t_p$ significantly impacts the final point cloud density. Too high a threshold removes too many points, leading to reconstruction artifacts; too low retains noise [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_based_pruning(gaussians, t_p, w_p):
    """
    Prune Gaussians based on depth-dependent threshold.
    
    Args:
        gaussians: List of Gaussian objects with attributes .z (position z) and .alpha (opacity).
        t_p: Base pruning threshold.
        w_p: Depth multiplier (set to 3).
        
    Returns:
        List of pruned Gaussians.
    """
    pruned_gaussians = []
    
    # Determine min and max z for normalization if needed, 
    # though source implies direct depth dependence.
    # Source states: "lowest threshold is applied to the farthest point"
    # and "tp is scaled by wp where wp is determined depending on the relative depth".
    # We assume a linear scaling from near to far.
    
    z_values = [g.z for g in gaussians]
    if not z_values:
        return []
        
    z_min = min(z_values)
    z_max = max(z_values)
    z_range = z_max - z_min if z_max != z_min else 1.0
    
    for g in gaussians:
        # Calculate relative depth factor (0 at near, 1 at far)
        # [G] Assuming linear interpolation for relative depth factor
        relative_depth = (g.z - z_min) / z_range
        
        # Calculate effective threshold
        # Source: "lowest threshold is applied to the farthest point"
        # Source: "tp is scaled by wp"
        # Interpretation: The threshold is scaled down at the far end.
        # A simple model: t_eff = t_p / (1 + w_p * relative_depth)
        # This ensures t_eff is t_p at near (relative_depth=0) and t_p/(1+w_p) at far.
        # However, the text says "scaled by wp". Another interpretation:
        # t_eff = t_p * (1 - relative_depth * (1 - 1/w_p)) ?
        # Given the ambiguity, we implement the logic: 
        # The threshold varies such that it is lowest at the farthest point.
        # Let's use the explicit multiplier w_p=3.
        # If w_p is a multiplier, maybe t_eff = t_p / w_p at far?
        # Let's stick to the most direct reading: 
        # "pruning threshold tp is scaled by wp ... lowest threshold is applied to the farthest point"
        # This implies the scaling factor depends on depth.
        # Let's define a scaling factor s(z) such that s(near) = 1 and s(far) = 1/w_p (or similar).
        # Without exact formula, we note the dependency.
        
        # Implementation note: The source does not give the exact function w(z).
        # We will use a placeholder comment for the exact scaling function.
        # [G] Common practice in such papers is linear interpolation of the threshold.
        # Let's assume t_eff(z) = t_p * (1 - relative_depth * (1 - 1/w_p))
        # This gives t_p at near, and t_p/w_p at far.
        
        scaling_factor = 1.0 - relative_depth * (1.0 - 1.0 / w_p)
        t_eff = t_p * scaling_factor
        
        if g.alpha >= t_eff:
            pruned_gaussians.append(g)
            
    return pruned_gaussians
```

## Subtopics

### [Advance Floater Pruning](17398-advance-floater-pruning.md)
Advance Floater Pruning is an explicit and adaptive operator designed to identify and eliminate undesired floating artifacts (misplaced Gaussians) in 3D Gaussian Splatting (3DGS) representations, particularly in sparse-view synthesis settings.

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of Gaussians, as it requires a single pass to evaluate opacity against the depth-dependent threshold.
*   **Memory Complexity:** $O(N)$ for storing the Gaussian attributes.
*   **Strengths:** Effectively preserves geometric detail at the far plane of forward-facing scenes, which is often under-represented due to sparsity [S1].
*   **Limitations:** Only applicable to forward-facing scenes where z-depth correlates with camera distance [S1]. Not robust to scenes with significant depth variation in the x-y plane that does not align with the z-axis.

## Background
Originated in the context of Deblurring 3D Gaussian Splatting to address the sparsity of Gaussians at the far end of forward-facing scenes in the DeblurNeRF dataset [S1]. It was introduced to improve reconstruction quality in scenarios involving camera motion blur and defocus blur by ensuring sufficient point coverage at depth [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
