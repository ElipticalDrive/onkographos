# Iterative Bisection for Zero-Crossing

Also called: Bisection Zero-Crossing, Zero Crossing Approximation

## Overview
This method finds the precise location along a grid edge where a Signed Distance Function (SDF) crosses a specific isovalue (typically $c=0$), indicating a material boundary between solid and air. It solves the root-finding problem $f(t) = 0$ for $t \in [0,1]$, where $t$ is the relative position along the edge. It is distinguished from [Linear Edge Intersection](868-linear-edge-intersection.md) by using iterative numerical approximation rather than analytic linear interpolation, allowing it to handle non-linear SDF variations at the cost of higher computational overhead.

## How it works
**Inputs and Outputs:**
*   **Input:** Two grid vertices $V_1, V_2$ with known world positions and SDF values $d_1, d_2$. The edge is valid only if one vertex is solid ($d \le c$) and the other is air ($d > c$), indicating a material change.
*   **Output:** A scalar $t \in [0,1]$ representing the relative intersection point.

**Data Structures:**
No complex structures are required. The algorithm operates on scalar values and simple interval bounds.

**Parameters:**
*   Maximum iterations: $N_{max} = 8$ [S1].
*   Sub-interval size threshold: $\epsilon_{size} = 10^{-6}$ [S1].
*   Density value bias: $\epsilon_{val} = 10^{-2}$ [S1].

**Steps:**
1.  Initialize the search interval $[t_{min}, t_{max}] = [0, 1]$.
2.  Loop for $i$ from 1 to $N_{max}$:
    a. Calculate midpoint: $t_{mid} = (t_{min} + t_{max}) / 2$.
    b. Sample the SDF at the corresponding world position to get $d_{mid}$.
    c. Check for termination conditions:
        i. If $|d_{mid}| \le \epsilon_{val}$, return $t_{mid}$ [S1].
        ii. If $(t_{max} - t_{min}) \le \epsilon_{size}$, return $t_{mid}$ [S1].
    d. Determine the new sub-interval:
        i. If $d_{mid}$ and $d_1$ have opposite signs (or $d_{mid} \le c$ vs $d_1 > c$ logic depending on sign convention), set $t_{max} = t_{mid}$.
        ii. Else, set $t_{min} = t_{mid}$.
3.  If the loop completes without early termination, return the last calculated $t_{mid}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def iterative_bisection_zero_crossing(v1_pos, v2_pos, v1_sdf, v2_sdf, isovalue=0.0):
    # Check for material change
    # Solid: sdf <= isovalue, Air: sdf > isovalue
    is_v1_solid = v1_sdf <= isovalue
    is_v2_solid = v2_sdf <= isovalue
    
    if is_v1_solid == is_v2_solid:
        return None  # No zero crossing on this edge

    t_min = 0.0
    t_max = 1.0
    t_mid = 0.5
    
    # Safeguards
    max_iter = 8
    size_thresh = 1e-6
    val_thresh = 1e-2

    for _ in range(max_iter):
        # Calculate midpoint position
        # [G] Linear interpolation of position is standard for sampling location
        mid_pos = v1_pos + t_mid * (v2_pos - v1_pos)
        
        # Sample SDF at midpoint
        mid_sdf = sample_sdf(mid_pos)
        
        # Check value threshold
        if abs(mid_sdf - isovalue) <= val_thresh:
            return t_mid
            
        # Check interval size threshold
        if (t_max - t_min) <= size_thresh:
            return t_mid
            
        # Determine which half contains the root
        # If mid_sdf is on the same side as v1_sdf relative to isovalue,
        # the root is in the right half [t_mid, t_max]
        # Otherwise, it is in the left half [t_min, t_mid]
        
        # Logic: if (mid_sdf - isovalue) and (v1_sdf - isovalue) have same sign
        mid_diff = mid_sdf - isovalue
        v1_diff = v1_sdf - isovalue
        
        if (mid_diff > 0 and v1_diff > 0) or (mid_diff < 0 and v1_diff < 0):
            t_min = t_mid
        else:
            t_max = t_mid
            
        t_mid = (t_min + t_max) / 2
        
    # Return last calculated midpoint if limit breached
    return t_mid
```

**Edge Cases and Pitfalls:**
*   **Multiple Crossings:** The method assumes only one material change per edge. If the grid resolution is insufficient to capture smaller details, multiple zero-crossings on a single edge will be lost, and only the first or dominant one is found [S1].
*   **Precision:** Rounding errors in floating-point arithmetic may prevent the function from ever reaching exactly zero, necessitating the bias threshold [S1].

## Subtopics

### Adaptive Zero-Crossing Sampling
A variant used in ray-marching contexts (e.g., dipole sum reconstruction) rather than grid-edge extraction. It identifies the first zero-crossing along a camera ray by placing 1024 dense samples between near and far limits. If found, it refines the region with 24 sparse samples before the crossing, 48 dense samples around the crossing, and 8 sparse samples after. If no crossing is found, it places 80 uniform samples [S2]. This approach optimizes sample density around the surface for inverse rendering efficiency.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of iterations (constant 8 in this implementation). Each step requires one SDF evaluation.
*   **Memory Complexity:** $O(1)$.
*   **Performance:** The method converges slowly compared to Newton-Raphson but is robust and guaranteed to converge for continuous functions. The limit of 8 steps provides a good trade-off between accuracy and performance for terrain rendering [S1].
*   **Strengths:** Robustness; handles non-linear SDFs better than linear interpolation; simple to implement.
*   **Limitations:** Slower convergence; requires SDF sampling (which may be expensive if the SDF is not analytic); loses sub-grid detail if resolution is low [S1].
*   **When to use:** When the SDF is not analytically differentiable or when robustness against non-linearities is prioritized over speed.
*   **When not to use:** When high-frequency details are expected but grid resolution is low, as multiple crossings will be missed [S1].

## Background
The method originates from volumetric terrain rendering techniques using WebGL, where efficient isosurface extraction from SDF grids was required [S1]. It relies on the bisection method (binary search), described in literature as one of the most effective methods for finding real zeros of continuous functions [S1, citing [24]]. The specific safeguards (8 iterations, $10^{-6}$ size threshold, $10^{-2}$ value bias) were introduced to handle the limitations of computer number representation and to ensure early termination in real-time rendering contexts [S1].

## Sources
[S1] Volumetric Terrain Rendering with WebGL, https://doi.org/10.2352/issn.2470-1173.2018.06.mobmu-136, fetched 2026-10-26
[S2] 3D Reconstruction with Fast Dipole Sums, https://doi.org/10.1145/3687914, fetched 2026-10-28
