# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization technique for heightfield terrain rendering that prevents ray stalling and surface overshooting on non-distance fields [S1, unofficial]. Unlike standard sphere tracing which relies on a global Lipschitz constant, this method bounds the step size by half the vertical gap to the surface, ensuring the ray never penetrates ridgelines while allowing larger steps in open space [S1, unofficial]. It builds on [Step Size Optimization](2054-step-size-optimization.md) by combining distance-proportional growth with a local geometric constraint.

## How it works
**Inputs:** Ray origin $\mathbf{o} = (o_x, o_y, o_z)$, normalized ray direction $\mathbf{d} = (d_x, d_y, d_z)$, heightfield function $H(x, z)$, maximum distance $t_{max}$, maximum iterations $N_{max}$, step scale factor $s$ (typically $0 < s \le 1$), minimum step size $\delta_{min}$, and refinement iterations $N_{refine}$ [S1, unofficial].

**Outputs:** Intersection distance $t_{hit}$ or miss indicator [S1, unofficial].

**Data Structures:**
*   `heightAtExtended(field, x, z)`: Returns the height $H(x, z)$ at the given world coordinates. For bilinear interpolation, this involves sampling the four surrounding grid points and interpolating [S1, unofficial].

**Algorithm Steps:**
1.  Initialize $t = 0.0$, $t_{prev} = 0.0$.
2.  Loop for $i$ from $0$ to $N_{max} - 1$:
    a.  If $t \ge t_{max}$, return miss [S1, unofficial].
    b.  Calculate current position:
        $$x = o_x + d_x \cdot t$$
        $$z = o_z + d_z \cdot t$$
        $$y_{ray} = o_y + d_y \cdot t$$
    c.  Calculate surface height at current $x, z$:
        $$h_{surf} = H(x, z)$$
    d.  Calculate vertical gap:
        $$h = y_{ray} - h_{surf}$$
    e.  Check for surface hit:
        If $h \le 0$, the ray has crossed or hit the surface. Perform binary search between $t_{prev}$ and $t$ for $N_{refine}$ iterations to refine the intersection:
        $$lo = t_{prev}, hi = t$$
        For $j$ from $0$ to $N_{refine} - 1$:
        $$mid = (lo + hi) \cdot 0.5$$
        $$x_{mid} = o_x + d_x \cdot mid$$
        $$z_{mid} = o_z + d_z \cdot mid$$
        $$y_{mid} = o_y + d_y \cdot mid$$
        $$h_{mid} = y_{mid} - H(x_{mid}, z_{mid})$$
        If $h_{mid} \le 0$, $hi = mid$; else $lo = mid$
        Return $hi$ as the intersection distance [S1, unofficial].
    f.  Calculate step size:
        $$t_{dist} = t \cdot s$$
        $$t_{gap} = h \cdot 0.5$$
        $$\Delta t = \min(t_{dist}, t_{gap})$$
    g.  Apply minimum step clamp:
        If $\Delta t < \delta_{min}$, $\Delta t = \delta_{min}$ [S1, unofficial].
    h.  Update previous distance:
        $$t_{prev} = t$$
    i.  Advance ray:
        $$t = t + \Delta t$$
3.  If loop completes without hit, return miss [S1, unofficial].

**Parameters:**
*   $s$: Step scale factor. Determines how fast the distance-proportional term grows. No typical value specified in sources [S1, unofficial].
*   $\delta_{min}$: Minimum step size to prevent infinite loops on flat surfaces or precision issues [S1, unofficial].
*   $N_{max}$: Maximum number of march iterations [S1, unofficial].
*   $N_{refine}$: Number of bisection iterations for precision [S1, unofficial].

**Pitfalls:**
*   Without the gap limiting term ($h \cdot 0.5$), the ray can step through the surface on ridgelines, causing holes [S1, unofficial].
*   The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].
*   Precision issues on flat surfaces may require $\delta_{min}$ to avoid stalling [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of steps, which is typically much less than fixed-step marching for large terrains due to distance-proportional growth. Refinement adds $O(N_{refine})$ per hit.
*   **Memory Complexity:** $O(1)$ auxiliary space, assuming the heightfield is accessed via random lookup.
*   **Strengths:** Prevents surface penetration on complex terrain (ridgelines); efficient for large open areas due to distance-proportional steps; simple to implement [S1, unofficial].
*   **Limitations:** Specific to heightfields; requires a height function; may stall if $s$ is too small or $\delta_{min}$ is too large relative to terrain features [S1, unofficial].
*   **When to use:** Terrain rendering where heightfields are available and ridgeline accuracy is critical [S1, unofficial].
*   **When not to use:** General SDFs or volumetric data where heightfield structure is not present [S1, unofficial].

## Background
Originating from the `theme1` renderer by mingderwang, this method was developed to efficiently render heightfield-based terrain without relying on distance fields [S1, unofficial]. It addresses the specific challenge of ray marching over terrain where the surface is not a distance field, preventing the common issue of rays stepping over ridgelines [S1, unofficial].

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
