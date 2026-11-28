# Blending Between Levels (Naive)
Also called: blendNaive

## Overview
Blending Between Levels (Naive) is a continuous volume rendering method that smoothly interpolates values across different resolution levels of an adaptive dataset. It solves the discontinuity problem inherent in selecting a single discrete level of detail by treating virtual cells as transparent and actual cells as opaque, blending interpolants from the selected level and all coarser levels based on interpolated opacity [S1, unofficial]. Unlike its optimized sibling, Blending Between Levels (Fast) [7871-blending-between-levels-fast.md], which uses directional blending logic, this method iterates sequentially from the finest level downwards, making it conceptually simpler but computationally more expensive [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A query point $P$ in 3D space.
*   Output: A blended scalar value $f$.

**Data Structures:**
*   The method assumes access to a function `lerpOnLevel(l, P)` which returns a tuple $(f_l, a_l)$ for a given level $l$ and point $P$.
    *   $f_l$: The trilinearly interpolated value at level $l$.
    *   $a_l$: The interpolated opacity at level $l$, derived from the density of actual vs. virtual cells.

**Algorithm Steps:**
1.  Initialize the accumulator $f = 0$.
2.  Iterate through levels $l$ starting from $0$ (finest) up to $MAX\_LEVEL$ (coarsest).
3.  For each level $l$, compute $(f_l, a_l) = \text{lerpOnLevel}(l, P)$.
4.  If $a_l == 0$, terminate the loop immediately.
5.  Update the accumulator: $f = a_l \cdot f_l + (1 - a_l) \cdot f$.
6.  Return $f$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blendNaive(P):
    f = 0.0
    for level in range(0, MAX_LEVEL + 1):
        f_l, a_l = lerpOnLevel(level, P)
        if a_l == 0:
            break
        f = a_l * f_l + (1 - a_l) * f
    return f
```

**Parameters:**
*   $MAX\_LEVEL$: The maximum depth of the adaptive hierarchy. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The method relies on $a_l$ being exactly 0 to break early. Floating-point inaccuracies in opacity calculation may prevent early termination, causing unnecessary computations on coarser levels [S1, unofficial].
*   **Homogeneous Regions:** In regions where only one level has non-zero opacity, the method reduces to standard trilinear interpolation on that level [S1, unofficial].
*   **Boundaries:** Across boundaries between regions of different resolutions, the method provides a smooth, continuous transition by blending the interpolants of adjoining regions [S1, unofficial].

## Subtopics

### Low-Resolution Part Blending
This variant modifies the opacity handling for low-resolution textures. It samples the texture and normalizes the alpha channel by `_Offsets.w` and `_CurveParams.x`: `from.a = saturate(from.a * _Offsets.w) / (_CurveParams.x + 1e-5f)`. It then squares the alpha twice (power of 4) and multiplies by `_CurveParams.x`: `from.a = square * square * _CurveParams.x` where `square = from.a * from.a` [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of levels, $O(L)$, where $L$ is the number of levels up to $MAX\_LEVEL$. In the worst case (fully opaque throughout), it performs $L$ trilinear interpolations and blending operations.
*   **Memory Complexity:** $O(1)$ auxiliary memory, assuming `lerpOnLevel` accesses the texture data directly.
*   **Performance:** Described as "expensive" compared to optimized variants because it potentially processes all levels from finest to coarsest without early exit optimization logic beyond the opacity check [S1, unofficial].
*   **Strengths:** Adaptive and continuous; handles boundaries smoothly; conceptually simple to implement.
*   **Limitations:** Computational cost increases with hierarchy depth; less efficient than directional blending methods like Blending Between Levels (Fast) [7871-blending-between-levels-fast.md].

## Background
The method originates from research on CPU volume rendering of adaptive mesh refinement (AMR) data [S1, unofficial]. It was proposed as a solution to the discontinuity in level selection, providing a continuous blending function that is also interpolating. The approach treats virtual cells as transparent (opacity 0) and actual cells as opaque (opacity 1), allowing for smooth transitions between different resolution regions [S1, unofficial].

## Sources
[S1] CPU volume rendering of adaptive mesh refinement data, https://www.researchgate.net/publication/321321949_CPU_volume_rendering_of_adaptive_mesh_refinement_data, fetched 2026-10-26 (unofficial)
[S2] dario-zubovic/metaballs, https://github.com/dario-zubovic/metaballs, fetched 2026-10-26 (unofficial)
