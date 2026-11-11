# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization for heightfield terrain rendering that prevents the ray from overshooting ridgelines and creating visual holes [S1, unofficial]. It differs from standard sphere tracing by using a step size proportional to the distance traveled, bounded by half the vertical gap to the surface, rather than the distance field value itself [S1, unofficial]. This approach avoids stalling on non-distance fields while ensuring robust intersection detection [S1, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ and direction $\mathbf{d} = (d_x, d_y, d_z)$, where $\mathbf{d}$ is normalized [S1, unofficial].
- Heightfield function $H(x, z)$ returning terrain height at any $(x, z)$ [S1, unofficial].
- Parameters: `stepScale` (multiplier for distance-proportional term), `minStep` (minimum step size), `maxSteps` (iteration limit), `tMax` (maximum ray distance), `refineSteps` (bisection iterations) [S1, unofficial].

**Outputs:**
- Intersection distance $t$ or $-1$ if no intersection is found [S1, unofficial].

**Data Structures:**
- No complex data structures required; only scalar variables for ray state and heightfield lookups [G].

**Algorithm Steps:**
1. Initialize $t = 0$ and compute initial vertical gap $h = o_y - H(o_x, o_z)$ [S1, unofficial].
2. Loop up to `maxSteps`:
   a. Calculate distance-proportional step: $\Delta t_{dist} = t \times \text{stepScale}$ [S1, unofficial].
   b. Calculate gap-limited step: $\Delta t_{gap} = h \times 0.5$ [S1, unofficial].
   c. Determine step size: $\Delta t = \min(\Delta t_{dist}, \Delta t_{gap})$ [S1, unofficial].
   d. If $\Delta t < \text{minStep}$, set $\Delta t = \text{minStep}$ [S1, unofficial].
   e. Store previous distance: $t_{prev} = t$ [S1, unofficial].
   f. Update distance: $t = t + \Delta t$ [S1, unofficial].
   g. If $t \ge t_{max}$, return $-1$ (miss) [S1, unofficial].
   h. Calculate current position: $x = o_x + d_x \times t$, $z = o_z + d_z \times t$ [S1, unofficial].
   i. Calculate current ray height: $y_{ray} = o_y + d_y \times t$ [S1, unofficial].
   j. Get terrain height: $y_{terr} = H(x, z)$ [S1, unofficial].
   k. Calculate vertical gap: $h = y_{ray} - y_{terr}$ [S1, unofficial].
   l. If $h \le 0$ (surface crossed or hit):
      i. Perform binary search between $t_{prev}$ and $t$ for `refineSteps` iterations:
         - $mid = (t_{prev} + t) \times 0.5$ [S1, unofficial].
         - Calculate $y_{mid} = o_y + d_y \times mid$ and $y_{terr\_mid} = H(o_x + d_x \times mid, o_z + d_z \times mid)$ [S1, unofficial].
         - If $y_{mid} - y_{terr\_mid} \le 0$, set $t = mid$ (hit is in lower half) [S1, unofficial].
         - Else, set $t_{prev} = mid$ (hit is in upper half) [S1, unofficial].
      ii. Return $t$ (intersection distance) [S1, unofficial].

**Pitfalls:**
- Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on ridgelines, causing holes [S1, unofficial]. The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].
- Precision issues on flat surfaces may require `minStep` to prevent infinite loops [S1, unofficial].
- The method assumes a heightfield where $h$ represents vertical clearance above the surface; applying it directly to signed distance fields without adaptation will fail because $h$ may be negative or unrelated to surface proximity [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def trace_terrain(o, d, H, step_scale, min_step, max_steps, t_max, refine_steps):
    t = 0.0
    # Initial gap calculation
    x0 = o[0] + d[0] * t
    z0 = o[2] + d[2] * t
    y0 = o[1] + d[1] * t
    h = y0 - H(x0, z0)
    
    for _ in range(max_steps):
        # Distance-proportional step
        dist_step = max(0.0, t * step_scale)
        
        # Gap-limited step (clamp to non-negative to prevent backward stepping)
        gap_step = max(0.0, h * 0.5)
        
        # Determine step size
        step = min(dist_step, gap_step)
        if step < min_step:
            step = min_step
            
        # Save previous t for refinement
        t_prev = t
        
        # Update t
        t += step
        
        # Check for miss
        if t >= t_max:
            return -1
            
        # Calculate new position and gap
        x_new = o[0] + d[0] * t
        z_new = o[2] + d[2] * t
        y_ray_new = o[1] + d[1] * t
        y_terr_new = H(x_new, z_new)
        h = y_ray_new - y_terr_new
        
        # Check for surface crossing
        if h <= 0:
            # Binary search for precise intersection
            lo = t_prev
            hi = t
            for _ in range(refine_steps):
                mid = (lo + hi) * 0.5
                x_mid = o[0] + d[0] * mid
                z_mid = o[2] + d[2] * mid
                y_ray_mid = o[1] + d[1] * mid
                y_terr_mid = H(x_mid, z_mid)
                if y_ray_mid - y_terr_mid <= 0:
                    hi = mid
                else:
                    lo = mid
            return hi
            
    return -1
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps until intersection or termination. Each step involves a heightfield lookup and basic arithmetic [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [G].
- **Strengths:** Prevents overshooting on ridgelines, covers more ground per step than fixed small steps, robust for heightfields [S1, unofficial].
- **Limitations:** Specific to heightfields; requires a height function $H(x,z)$. Performance depends on `stepScale` and terrain complexity [S1, unofficial].
- **When to use:** Terrain rendering with heightfields where ridgeline accuracy is critical [S1, unofficial].
- **When not to use:** General volumetric data or non-heightfield geometry [G].

## Background
Originated in software-based terrain rendering to address the limitation of standard distance-proportional stepping, which can fail on non-distance fields like heightmaps by introducing a gap-based bound [S1, unofficial].

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
