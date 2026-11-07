# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization for distance-field ray marching on heightfield terrain that prevents the ray from overshooting surface features like ridgelines. Unlike standard distance-field marching which relies on a global Lipschitz constant, this method uses a step size proportional to the distance traveled, bounded by half the vertical gap to the surface. This ensures the ray never steps through the surface, eliminating holes in terrain rendering while maintaining efficiency on flat areas.

## How it works
**Inputs:**
- Ray origin $(o_x, o_y, o_z)$ and direction $(d_x, d_y, d_z)$, where $d$ is normalized.
- Heightfield function $H(x, z)$ returning surface height at horizontal coordinates.
- Parameters: `stepScale` (dimensionless multiplier), `minStep` (minimum step size), `maxSteps` (iteration limit), `tMax` (maximum ray distance), `refineSteps` (bisection iterations).

**Outputs:**
- Intersection distance $t_{hit}$ or $-1$ if no intersection is found within `tMax`.

**Data Structures:**
- No complex structures required. Standard floating-point variables for ray parameters and height values.

**Algorithm Steps:**
1. Initialize $t = 0$.
2. Initialize `prevT` = 0.
3. Loop for $i$ from 0 to `maxSteps` - 1:
    a. Calculate distance-proportional step: $d_{step} = t \times \text{stepScale}$.
    b. Calculate current horizontal position: $x = o_x + d_x \times t$, $z = o_z + d_z \times t$.
    c. Calculate current ray height: $y_{ray} = o_y + d_y \times t$.
    d. Sample heightfield: $h_{surf} = H(x, z)$.
    e. Calculate vertical gap: $h = y_{ray} - h_{surf}$.
    f. Calculate gap-limited step: $g_{step} = h \times 0.5$.
    g. Determine step size: $step = \min(d_{step}, g_{step})$.
    h. Enforce minimum step: if $step < \text{minStep}$, set $step = \text{minStep}$.
    i. Update previous distance: `prevT` = $t$.
    j. Advance ray: $t = t + step$.
    k. Check termination: if $t \ge \text{tMax}$, return $-1$.
    l. Check surface crossing: if $h \le 0$ (ray is at or below surface):
        i. Perform binary search between `prevT` and $t$ for `refineSteps` iterations:
            1. $mid = (\text{prevT} + t) \times 0.5$.
            2. Calculate ray height at $mid$: $y_{mid} = o_y + d_y \times mid$.
            3. Calculate horizontal position at $mid$: $x_{mid} = o_x + d_x \times mid$, $z_{mid} = o_z + d_z \times mid$.
            4. Sample heightfield: $h_{mid} = H(x_{mid}, z_{mid})$.
            5. If $y_{mid} - h_{mid} \le 0$, set $t = mid$ (surface crossed before mid).
            6. Else, set `prevT` = $mid$ (surface crossed after mid).
        ii. Return $t$ as the intersection distance.
4. If loop completes without intersection, return $-1$.

**Pitfalls:**
- Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on steep ridgelines, causing holes in the rendered terrain [S1, unofficial].
- The `stepScale` parameter must be chosen carefully; if too large, the distance-proportional step may exceed the gap limit unnecessarily, though the gap limit acts as a hard cap. If too small, performance degrades on flat terrain.
- Precision issues on flat surfaces or at very small distances can cause infinite loops if `minStep` is not enforced.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps per ray, typically bounded by `maxSteps`. The binary search adds $O(\log(\text{precision}))$ overhead per intersection.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Prevents overshooting on non-distance-field terrain (heightfields) where Lipschitz constants are not uniform or known. Efficient on flat terrain due to distance-proportional stepping.
- **Limitations:** Requires heightfield sampling at every step. Performance depends on terrain roughness; very rough terrain may require many small steps.
- **When to use:** Rendering heightfield-based terrain where surface continuity is critical and standard distance-field marching fails due to lack of a global Lipschitz bound.
- **When not to use:** When a true distance field is available and efficiently computable, as standard distance-field marching is generally more robust and simpler.

## Background
This method is derived from techniques used in software-based terrain rendering, specifically in the `theme1` project by mingderwang, which implements a no-GPU renderer for procedural terrain [S1, unofficial]. It addresses the specific challenge of ray marching over heightfields without relying on a global distance bound, which is often unavailable or loose for complex terrain.

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
