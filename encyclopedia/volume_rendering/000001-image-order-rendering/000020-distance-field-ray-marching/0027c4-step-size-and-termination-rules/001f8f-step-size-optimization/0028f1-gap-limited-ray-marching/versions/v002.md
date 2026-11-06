# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization method for heightfield terrain rendering that prevents ray stalling on non-distance fields by combining a distance-proportional step with a vertical gap bound [S1, unofficial]. Unlike sibling methods that rely on local surface derivatives or global distance fields, this approach ensures the ray never overshoots ridgelines by capping the step size at half the vertical clearance to the surface, allowing for larger steps in open space while maintaining geometric correctness [S1, unofficial]. It is specifically designed for heightfield-based terrain where the vertical gap provides a reliable conservative bound for step size [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Ray origin $\mathbf{o} = (o_x, o_y, o_z)$, ray direction $\mathbf{d} = (d_x, d_y, d_z)$ (normalized), heightfield function $H(x, z)$, maximum march distance $t_{max}$, maximum iteration count $maxSteps$, minimum step size $minStep$, step scale factor $stepScale$, and refinement iteration count $refineSteps$.
*   **Outputs:** Intersection distance $t$ (or $-1$ if no intersection), or refined intersection point via binary search.

**Data Structures:**
*   Heightfield $H$: A 2D array or function mapping $(x, z)$ coordinates to height $y$.
*   Ray state: Current distance $t$, previous distance $t_{prev}$.

**Parameters:**
*   $stepScale$: Multiplier for the distance-proportional term. Typical value not specified in sources [S1, unofficial].
*   $minStep$: Minimum step size to prevent infinite loops on flat surfaces or precision issues [S1, unofficial].
*   $maxSteps$: Maximum number of march iterations [S1, unofficial].
*   $refineSteps$: Number of bisection iterations for surface refinement [S1, unofficial].

**Algorithm Steps:**
1.  Initialize $t = 0.0$, $t_{prev} = 0.0$.
2.  Loop up to $maxSteps$:
    a.  Calculate distance-proportional step: $d_{step} = t \times stepScale$.
    b.  Calculate current ray position: $x = o_x + d_x \times t$, $z = o_z + d_z \times t$.
    c.  Calculate current ray height: $y_{ray} = o_y + d_y \times t$.
    d.  Sample heightfield: $h_{surf} = H(x, z)$.
    e.  Calculate vertical gap: $h = y_{ray} - h_{surf}$.
    f.  Calculate gap-limited step: $g_{step} = h \times 0.5$.
    g.  Determine step size: $\Delta s = \min(d_{step}, g_{step})$.
    h.  Enforce minimum step: If $\Delta s < minStep$, set $\Delta s = minStep$.
    i.  Update previous distance: $t_{prev} = t$.
    j.  Update distance: $t = t + \Delta s$.
    k.  Check termination: If $t \ge t_{max}$, return $-1$ (miss).
    l.  Check surface crossing: If $h \le 0$ (ray is at or below surface):
        i.  Perform binary search between $t_{prev}$ and $t$ for $refineSteps$ iterations:
            1.  $mid = (t_{prev} + t) \times 0.5$.
            2.  Calculate $y_{mid} = o_y + d_y \times mid$, $x_{mid} = o_x + d_x \times mid$, $z_{mid} = o_z + d_z \times mid$.
            3.  $h_{mid} = y_{mid} - H(x_{mid}, z_{mid})$.
            4.  If $h_{mid} \le 0$, set $t = mid$ (surface is below or at mid).
            5.  Else, set $t_{prev} = mid$ (surface is above mid).
        ii. Return $t$ (refined intersection distance).

**Pitfalls:**
*   Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on ridgelines, causing holes in the rendered terrain [S1, unofficial].
*   The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].
*   Precision issues on flat surfaces may require a non-zero $minStep$ to avoid infinite loops [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(maxSteps)$ per ray, with an additional $O(refineSteps)$ for binary search refinement upon surface crossing.
*   **Memory Complexity:** $O(1)$ auxiliary memory, assuming the heightfield is accessed via lookup.
*   **Strengths:** Efficiently covers large distances in open terrain due to distance-proportional stepping; robust against overshooting ridgelines due to gap limiting; simple to implement.
*   **Limitations:** Specific to heightfield terrain; requires a heightfield function; step size is not adaptive to local surface curvature beyond the vertical gap; performance depends on $stepScale$ and terrain complexity.
*   **When to use:** Heightfield terrain rendering where ridgeline accuracy is critical and large open areas exist.
*   **When not to use:** Non-heightfield volumes, scenes with complex geometry not representable as a heightfield, or when high-frequency surface details require smaller, adaptive steps.

## Background
Originating from the `theme1` project by mingderwang, this method is described in an unofficial GitHub repository [S1, unofficial]. It was developed to address the inefficiency and inaccuracies of constant step size ray marching on terrain, specifically avoiding stalls on non-distance fields and preventing holes on ridgelines [S1, unofficial]. The technique is part of a software renderer that bakes terrain data for efficient CPU-based rendering.

## Sources
[S1] theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
