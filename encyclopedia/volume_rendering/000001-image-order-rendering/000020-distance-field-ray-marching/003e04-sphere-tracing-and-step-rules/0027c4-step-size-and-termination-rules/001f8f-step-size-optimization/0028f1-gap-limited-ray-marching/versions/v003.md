# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization technique for heightfield terrain rendering that prevents ray stalling and surface overshoot on non-distance fields. It combines a distance-proportional step term with a vertical gap bound, ensuring the ray never steps more than half the vertical clearance to the surface, thereby avoiding holes in ridgelines while covering ground efficiently.

## How it works
**Inputs:** Ray origin $\mathbf{o} = (o_x, o_y, o_z)$, ray direction $\mathbf{d} = (d_x, d_y, d_z)$ (normalized), heightfield function $H(x, z)$, maximum march distance $t_{max}$, maximum iterations $maxSteps$, minimum step size $minStep$, step scale multiplier $stepScale$, and refinement iterations $refineSteps$.
**Outputs:** Intersection distance $t$ or $-1$ if no intersection.

**Data Structures:**
*   `heightAtExtended(field, x, z)`: Returns the height $H(x, z)$ from the heightfield, handling boundary conditions (e.g., clamping or wrapping) as defined by the terrain generator.

**Parameters:**
*   $stepScale$: Multiplier for the distance-proportional term. Typical value not specified in sources [S1, unofficial].
*   $minStep$: Minimum step size to prevent infinite loops on flat surfaces or precision issues [S1, unofficial].
*   $maxSteps$: Maximum number of march iterations [S1, unofficial].
*   $refineSteps$: Number of bisection iterations for surface refinement [S1, unofficial].

**Algorithm:**
1.  Initialize $t = 0.0$, $t_{prev} = 0.0$.
2.  Loop for $i$ from $0$ to $maxSteps - 1$:
    a.  Calculate distance-proportional step: $t_{dist} = t \times stepScale$.
    b.  Calculate current position: $x = o_x + d_x \times t$, $z = o_z + d_z \times t$.
    c.  Calculate current height: $y_{ray} = o_y + d_y \times t$.
    d.  Calculate heightfield height: $y_{surf} = H(x, z)$.
    e.  Calculate vertical gap: $h = y_{ray} - y_{surf}$.
    f.  Calculate gap-limited step: $t_{gap} = h \times 0.5$.
    g.  Determine step size: $t_{step} = \min(t_{dist}, t_{gap})$.
    h.  Enforce minimum step: If $t_{step} < minStep$, set $t_{step} = minStep$.
    i.  Update previous distance: $t_{prev} = t$.
    j.  Advance ray: $t = t + t_{step}$.
    k.  Check termination: If $t \ge t_{max}$, return $-1$ [S1, unofficial].
    l.  Check surface crossing: If $h \le 0$ (ray is at or below surface):
        i.  Perform binary search between $t_{prev}$ and $t$ for $refineSteps$ iterations:
            1.  $mid = (t_{prev} + t) \times 0.5$.
            2.  Calculate $y_{mid} = o_y + d_y \times mid$, $x_{mid} = o_x + d_x \times mid$, $z_{mid} = o_z + d_z \times mid$.
            3.  Calculate $h_{mid} = y_{mid} - H(x_{mid}, z_{mid})$.
            4.  If $h_{mid} \le 0$, set $t = mid$ (surface is before mid).
            5.  Else, set $t_{prev} = mid$ (surface is after mid).
        ii. Return $t$ [S1, unofficial].

**Pitfalls:**
*   Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on ridgelines, causing holes [S1, unofficial].
*   The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(maxSteps)$ per ray, with a constant factor for heightfield lookup. Refinement adds $O(refineSteps)$ only upon surface crossing.
*   **Memory Complexity:** $O(1)$ auxiliary memory, assuming the heightfield is accessed via random lookup.
*   **Strengths:** Efficiently covers large empty spaces due to the distance-proportional term. Prevents surface penetration on steep terrain (ridgelines) where standard distance-proportional stepping might fail.
*   **Limitations:** Requires a heightfield representation. Performance degrades if $minStep$ is too small or $maxSteps$ is insufficient for complex terrain. The gap bound is specific to heightfields and not directly applicable to general volumetric or SDF data without adaptation.
*   **When to use:** Heightfield terrain rendering where ridgeline accuracy is critical and empty space skipping is desired.
*   **When not to use:** General volumetric rendering, SDF-based rendering, or scenes without a clear heightfield structure.

## Background
This method is described in the context of a software renderer for procedural terrain generation, specifically within the `theme1` project. It is designed to work with baked heightfields and normals, avoiding the need for finite difference calculations which can cause facet artifacts. The technique is attributed to the implementation details of the `traceTerrain` function in the referenced repository.

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
