# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization technique for heightfield terrain rendering that prevents ray stalling and surface penetration on non-distance fields. Unlike standard sphere tracing which relies on a global Lipschitz constant, this method uses a step size proportional to the accumulated distance traveled, bounded by half the vertical gap to the surface. This ensures the ray never overshoots ridgelines while covering more ground per step than fixed-step methods.

## How it works
**Inputs:**
- Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ and normalized direction $\mathbf{d} = (d_x, d_y, d_z)$.
- Heightfield function $H(x, z)$ returning surface height at world coordinates.
- Parameters: `stepScale` (dimensionless multiplier), `minStep` (float), `maxSteps` (int), `refineSteps` (int), `tMax` (float).

**Outputs:**
- Intersection distance $t$ or $-1$ if no intersection is found within `tMax`.

**Data Structures:**
- No persistent data structures; state is held in local variables during the trace.

**Algorithm:**
1. Initialize $t = 0$.
2. Loop $i$ from $0$ to `maxSteps` - 1:
   a. Calculate the distance-proportional step: $t_{step} = t \times \text{stepScale}$.
   b. Calculate the current world position: $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
   c. Calculate the vertical gap to the surface: $h = p_y - H(p_x, p_z)$.
   d. Calculate the gap-limited step: $g_{step} = h \times 0.5$.
   e. Determine the step size: $\Delta t = \min(t_{step}, g_{step})$.
   f. If $\Delta t < \text{minStep}$, set $\Delta t = \text{minStep}$. [G] (Guard against division by zero or infinite loops on flat surfaces).
   g. Store previous distance: $t_{prev} = t$.
   h. Update distance: $t = t + \Delta t$.
   i. If $t \ge t_{\text{Max}}$, return $-1$.
   j. Recalculate position: $\mathbf{p} = \mathbf{o} + t\mathbf{d}$.
   k. Recalculate gap: $h = p_y - H(p_x, p_z)$.
   l. If $h \le 0$, the surface has been crossed. Perform binary search between $t_{prev}$ and $t$ for `refineSteps` iterations:
      i. $mid = (t_{prev} + t) \times 0.5$.
      ii. Calculate $h_{mid} = (o_y + mid \cdot d_y) - H(o_x + mid \cdot d_x, o_z + mid \cdot d_z)$.
      iii. If $h_{mid} \le 0$, set $t = mid$ (narrow upper bound).
      iv. Else, set $t_{prev} = mid$ (narrow lower bound).
      v. Return $t$.
3. Return $-1$ (miss).

**Parameters:**
- `stepScale`: Typical value not specified in sources. Controls the aggressiveness of the distance-proportional term.
- `minStep`: Prevents stalling on flat terrain or precision errors. Value not specified.
- `maxSteps`: Limit to prevent infinite loops. Value not specified.
- `refineSteps`: Number of bisection iterations for precision. Value not specified.

**Pitfalls:**
- Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on ridgelines, causing visual holes [S1, unofficial].
- The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps until convergence or termination. The binary search adds a constant overhead $O(\log(\text{precision}))$ upon intersection.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Avoids stalling on non-distance fields (heightmaps) where Lipschitz constants are not uniform or known. Covers more ground per step than fixed-step methods. Prevents surface penetration on complex terrain features like ridgelines.
- **Limitations:** Requires a heightfield representation. Not directly applicable to general volumetric or implicit surface rendering without adaptation. Performance depends heavily on the choice of `stepScale` and terrain roughness.
- **When to use:** Terrain rendering where heightmaps are used and standard sphere tracing is inefficient or unsafe due to varying terrain gradients.

## Background
Originating from practical implementations of terrain rendering in software ray tracers, this method addresses the specific challenge of marching rays over heightfields. Standard sphere tracing requires a Signed Distance Function (SDF) with a known Lipschitz constant. Heightfields are not SDFs, and their local gradient can vary wildly, making fixed or purely distance-proportional steps unsafe. The gap-limiting technique was developed to provide a safe, adaptive step size that scales with distance but is strictly bounded by the local vertical clearance to the surface.

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
