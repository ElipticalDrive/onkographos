# Depth-Based Gaussian Pruning

Also called: Depth-adaptive pruning, relative depth pruning

## Overview
Depth-Based Gaussian Pruning is a strategy that varies the opacity pruning threshold based on the relative depth (z-axis value) of 3D Gaussians to preserve points at the far plane in forward-facing scenes [S1]. It sets itself apart from standard fixed-threshold pruning by applying a lower threshold (making pruning harder) to distant primitives, thereby compensating for the sparsity of Gaussians that naturally occurs at the back of the scene [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 3D Gaussians $\mathcal{G} = \{g_1, \dots, g_N\}$, where each $g_k$ has mean position $\mu_k = (x_k, y_k, z_k)$, opacity $\alpha_k$, and other attributes.
*   **Parameters:**
    *   $t_p$: Base pruning threshold. Typical values: $5 \times 10^{-3}$ for real defocus blur datasets, $1 \times 10^{-2}$ for real camera motion blur datasets [S1].
    *   $w_p$: Pruning threshold multiplier based on depth. Typical value: $3$ [S1].
*   **Output:** A subset of Gaussians retained after pruning.

**Data Structures:**
The method operates on the existing Gaussian buffer. No new complex structures are required beyond the standard Gaussian attributes. The relative depth is derived directly from the $z_k$ coordinate.

**Steps:**
1.  **Determine Relative Depth:** For each Gaussian $g_k$, identify its $z_k$ coordinate. In forward-facing scenes, $z_k$ serves as a proxy for relative depth from the viewpoint [S1].
2.  **Calculate Depth Multiplier:** Determine the multiplier $w_k$ for each Gaussian based on its $z_k$. The source specifies that the multiplier is determined by relative depth, with the lowest threshold applied to the farthest point. While the exact interpolation function is not explicitly defined in the provided text, the logic implies a mapping where $w_k$ scales with depth. Let $z_{min}$ and $z_{max}$ be the minimum and maximum $z$ values in the scene (or a relevant subset). A linear interpolation is implied by "determined depending on the relative depth":
    $$ w_k = 1 + (w_p - 1) \cdot \frac{z_k - z_{min}}{z_{max} - z_{min}} $$
    *Note: The source states the lowest threshold is applied to the farthest point. Since the effective threshold is $t_p \cdot w_k$ (or similar scaling), and we want to *preserve* far points, the threshold for far points must be *lower*? Or does the multiplier apply to the opacity? The text says "pruning threshold $t_p$ is scaled by $w_p$... lowest threshold is applied to the farthest point." If the threshold is the minimum opacity required to survive, a "low threshold" means it is easier to survive. Thus, for far points, the required opacity is lower. This implies the multiplier reduces the threshold. However, $w_p=3$ is $>1$. If $t_p$ is scaled by $w_p$, the threshold becomes higher, making pruning *harder* (more points survive). This aligns with "preserve more points located at the far plane". So, the farthest point gets the highest multiplier? The text says "lowest threshold is applied to the farthest point". This is contradictory if "threshold" means the value $t_p \cdot w$. If $w_p=3$, and we want the *lowest* threshold value, we should multiply by a small number. Let's re-read carefully: "pruning threshold $t_p$ is scaled by $w_p$ where $w_p$ is determined depending on the relative depth, and the lowest threshold is applied to the farthest point." If $w_p$ is the multiplier, and the result is the threshold, then the farthest point has the smallest multiplier. But $w_p$ is given as a single value 3. This suggests $w_p$ is the *range* or the *max* multiplier. Let's assume the multiplier $m_k$ ranges from 1 (near) to $w_p$ (far). Then the threshold is $t_p \cdot m_k$. A higher threshold means stricter pruning (higher opacity required). This would prune *more* far points. This contradicts "preserve more points".
    *Alternative Interpretation:* The "threshold" refers to the *allowance* for low opacity. Or, the multiplier is applied inversely. Let's look at the standard 3D-GS pruning: remove if $\alpha < t_p$. If we want to preserve far points, we want to remove *fewer* far points. So we need a *higher* opacity requirement? No, if we raise the requirement, we remove *more*. We need a *lower* opacity requirement. So the threshold $t_{eff}$ should be lower for far points.
    *Re-reading S1:* "prune 3D Gaussians depending on their positions... prune out less 3D Gaussians placed on the far edge... relying on the relative depth. Specifically, the pruning threshold $t_p$ is scaled by $w_p$... and the lowest threshold is applied to the farthest point."
    If the threshold is $T$, and we prune if $\alpha < T$, then a "low threshold" means we prune *less* (easier to survive). So $T_{far} < T_{near}$.
    The text says $t_p$ is scaled by $w_p$. If $w_p=3$, and it's a multiplier, does it mean $T = t_p / w(z)$? Or $T = t_p \cdot (1/w(z))$?
    Given $w_p=3$ is a single parameter, it likely defines the ratio between near and far thresholds.
    Let $T_{near} = t_p$. Let $T_{far} = t_p / w_p$.
    Then $T_k = t_p \cdot \left( 1 + ( \frac{1}{w_p} - 1 ) \frac{z_k - z_{min}}{z_{max} - z_{min}} \right)$.
    This ensures $T_{far}$ is the lowest threshold.
    
3.  **Apply Pruning:** For each Gaussian $g_k$, calculate its effective threshold $T_k$. If $\alpha_k < T_k$, remove $g_k$ from the set [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def depth_based_pruning(gaussians, t_p, w_p):
    """
    gaussians: List of Gaussian objects with attributes .alpha, .z
    t_p: Base pruning threshold (float)
    w_p: Depth multiplier (float, e.g., 3)
    """
    # Find depth range
    z_values = [g.z for g in gaussians]
    if not z_values:
        return []
    
    z_min = min(z_values)
    z_max = max(z_values)
    
    # Handle degenerate case where all z are same
    if z_max == z_min:
        threshold = t_p
        return [g for g in gaussians if g.alpha >= threshold]
    
    retained_gaussians = []
    
    for g in gaussians:
        # Calculate relative depth factor
        # Interpolate multiplier from 1 (near) to 1/w_p (far)
        # This ensures the threshold is lowest at the far plane
        relative_depth = (g.z - z_min) / (z_max - z_min)
        
        # Multiplier for the threshold: starts at 1.0 for near, goes to 1/w_p for far
        # This makes the threshold smaller (easier to survive) for far points
        depth_factor = 1.0 + ( (1.0 / w_p) - 1.0 ) * relative_depth
        
        effective_threshold = t_p * depth_factor
        
        if g.alpha >= effective_threshold:
            retained_gaussians.append(g)
            
    return retained_gaussians
```

**Edge Cases and Pitfalls:**
*   **Non-forward-facing scenes:** The method relies on $z_k$ as relative depth. It is explicitly designed for forward-facing scenes (e.g., DeblurNeRF dataset) [S1]. For general 360-degree scenes, $z$ is not a consistent depth metric across viewpoints, and this method may fail or require viewpoint-dependent depth calculation [S1].
*   **Depth Range Calculation:** $z_{min}$ and $z_{max}$ should likely be computed over the entire scene or a sliding window if the scene geometry changes significantly during training. Using global bounds is standard for static scenes.
*   **Precision:** Ensure floating-point comparisons for $\alpha_k < T_k$ are stable.

## Subtopics

## Operating characteristics
*   **Time Complexity:** $O(N)$ per pruning step, where $N$ is the number of Gaussians. The depth range calculation is one pass, and the filtering is another.
*   **Memory Complexity:** $O(1)$ auxiliary memory (storing min/max z).
*   **Performance:** Improves reconstruction quality in forward-facing scenes by preventing the loss of distant geometry, which is often under-represented due to lower pixel coverage and gradient magnitude [S1].
*   **Limitations:** Not applicable to non-forward-facing or general 3D scenes without modification. Requires the scene to have a well-defined "far" plane relative to the camera orientation.

## Background
Originating from work on Deblurring 3D Gaussian Splatting, this method addresses the issue of Gaussian sparsity at the far end of forward-facing scenes [S1]. It builds on the standard 3D-GS adaptive density control but introduces depth-awareness to the pruning phase [S1]. The approach was validated on datasets with camera motion blur and defocus blur, such as those from the DeblurNeRF benchmark [S1].

## Sources
[S1] Deblurring 3D Gaussian Splatting, https://doi.org/10.1007/978-3-031-73636-0_8, fetched 2026-10-26
