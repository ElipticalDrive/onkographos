# Gap-Limited Ray Marching

Also called: Distance-Proportional Step with Gap Bound, Terrain Ray Marching with Gap Limiting

## Overview
Gap-Limited Ray Marching is a step-size optimization for heightfield terrain rendering that prevents the ray from overshooting surface features, such as ridgelines, which can cause visual holes. Unlike standard sphere tracing which relies on a global distance field, this method uses a step size proportional to the distance traveled, bounded by half the vertical gap to the surface [S1, unofficial]. It builds on [Step Size and Termination Rules](4132-step-size-and-termination-rules.md) by introducing a geometric constraint specific to heightfields.

## How it works
**Inputs:**
- Ray origin $(o_x, o_y, o_z)$ and direction $(d_x, d_y, d_z)$, where the direction is normalized.
- Heightfield function $H(x, z)$ returning the surface height at any $(x, z)$.
- Parameters: `stepScale` (proportionality constant), `minStep` (minimum step size), `maxSteps` (iteration limit), `tMax` (maximum ray distance), `refineSteps` (bisection iterations).

**Outputs:**
- Intersection parameter $t$ if the surface is hit, or $-1$ if missed.

**Data Structures:**
- No complex structures required; standard scalar variables for ray parameters and heightfield lookups.

**Algorithm Steps:**
1. Initialize $t = 0$.
2. Loop up to `maxSteps`:
   a. Calculate the distance-proportional step: $t_{dist} = t \times \text{stepScale}$.
   b. Calculate the current position: $x = o_x + d_x \times t$, $z = o_z + d_z \times t$.
   c. Calculate the current ray height: $y_{ray} = o_y + d_y \times t$.
   d. Sample the heightfield: $h_{surf} = H(x, z)$.
   e. Calculate the vertical gap: $h = y_{ray} - h_{surf}$.
   f. Calculate the gap-limited step: $t_{gap} = h \times 0.5$.
   g. Determine the step size: $step = \min(t_{dist}, t_{gap})$.
   h. If $step < \text{minStep}$, set $step = \text{minStep}$.
   i. Store previous intersection parameter: $t_{prev} = t$.
   j. Update ray parameter: $t = t + step$.
   k. If $t \ge t_{max}$, return $-1$ (miss).
   l. If $h \le 0$ (surface crossed or hit):
      i. Perform binary search between $t_{prev}$ and $t$ for `refineSteps` iterations:
         - $mid = (t_{prev} + t) \times 0.5$.
         - Calculate $y_{mid} = o_y + d_y \times mid$ and $x_{mid}, z_{mid}$ similarly.
         - $h_{mid} = y_{mid} - H(x_{mid}, z_{mid})$.
         - If $h_{mid} \le 0$, set $t = mid$ (narrow upper bound); else set $t_{prev} = mid$ (narrow lower bound).
      ii. Return $t$ (the refined intersection).

**Pitfalls:**
- Without the gap limiting term ($h \times 0.5$), the ray can step through the surface on steep ridgelines, causing holes in the rendered terrain [S1, unofficial]. The gap step ensures the ray never steps more than half the vertical clearance, preventing overshoot [S1, unofficial].
- Precision issues on flat surfaces may require a sensible `minStep` to prevent infinite loops [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def gap_limited_ray_march(o, d, H, step_scale, min_step, max_steps, t_max, refine_steps):
    t = 0.0
    for _ in range(max_steps):
        # Distance-proportional step
        t_dist = t * step_scale
        
        # Current position
        x = o.x + d.x * t
        z = o.z + d.z * t
        y_ray = o.y + d.y * t
        
        # Vertical gap
        h_surf = H(x, z)
        h = y_ray - h_surf
        
        # Gap-limited step
        t_gap = h * 0.5
        
        # Determine step size
        step = min(t_dist, t_gap)
        if step < min_step:
            step = min_step
            
        # Store previous t for potential refinement
        t_prev = t
        
        # Advance ray
        t = t + step
        
        # Termination check
        if t >= t_max:
            return -1
            
        # Surface crossing check
        if h <= 0:
            # Binary search for precise intersection
            lo = t_prev
            hi = t
            for _ in range(refine_steps):
                mid = (lo + hi) * 0.5
                x_mid = o.x + d.x * mid
                z_mid = o.z + d.z * mid
                y_mid = o.y + d.y * mid
                h_mid = y_mid - H(x_mid, z_mid)
                if h_mid <= 0:
                    hi = mid
                else:
                    lo = mid
            return hi
    return -1
```

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of steps until intersection or termination. Each step involves a heightfield lookup and basic arithmetic.
- **Memory Complexity:** $O(1)$ auxiliary space.
- **Strengths:** Prevents overshooting on ridgelines, allowing larger steps than conservative fixed-step methods while maintaining visual correctness for heightfields [S1, unofficial].
- **Limitations:** Specific to heightfields; not directly applicable to general volumetric or signed distance field rendering without modification. Performance depends heavily on the choice of `stepScale` and terrain roughness.
- **When to use:** Terrain rendering where heightfields are available and ridgeline accuracy is critical.
- **When not to use:** General volume rendering or scenes without a heightfield representation.

## Background
This method is described in the context of a software renderer for procedural terrain generation, specifically within the `theme1` project by mingderwang [S1, unofficial]. It is presented as a practical solution for rendering heightfield-based landscapes without relying on a global distance field, addressing the specific issue of ray overshoot on steep terrain features.

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
