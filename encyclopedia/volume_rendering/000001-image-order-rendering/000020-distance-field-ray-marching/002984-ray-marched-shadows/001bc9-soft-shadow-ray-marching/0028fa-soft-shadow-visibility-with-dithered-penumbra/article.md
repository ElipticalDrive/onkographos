# Soft Shadow Visibility with Dithered Penumbra

Also called: Dithered Penumbra Shadow Visibility

## Overview
Calculates soft shadow visibility (0..1) towards a directional light (e.g., the sun) by marching along the light direction, using a dithered rotation of the light vector per pixel to create a penumbra that averages out with supersampling [S1, unofficial]. It sets itself apart from standard soft shadow ray marching by normalizing the penumbra calculation by $t \cdot \text{sunY}$ rather than distance alone, preventing false heavy shadows on flat ground where clearance is proportional to distance [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Height field function `heightAtExtended(x, z)`, sun direction vector $\vec{S} = (dx, dy, dz)$, pixel coordinates $(ox, oz)$, camera origin $oy$, and parameters `shadowMaxDistance`, `shadowSoftness`, `shadowMinStep`, `shadowSteps`.
*   **Output:** A single float value in $[0, 1]$ representing shadow visibility.

**Data Structures:**
No complex data structures are required beyond the height field access function. The algorithm maintains scalar state variables for the ray march.

**Parameters:**
*   `shadowMaxDistance`: Maximum distance to check for shadows.
*   `shadowSoftness`: Parameter controlling shadow softness.
*   `shadowMinStep`: Minimum step size for shadow ray marching.
*   `shadowSteps`: Number of shadow ray march iterations.
*   `softness`: Derived parameter, calculated as $6 \times \text{shadowSoftness} + 2$ [S1, unofficial].
*   `sunY`: Vertical component of the sun direction, clamped to $\ge 0.12$ [S1, unofficial].

**Steps:**
1.  Initialize ray distance $t = 0$, clearance $h = 0$, and visibility $V = 1$.
2.  Loop $i$ from $0$ to `shadowSteps` - 1:
    a. Calculate distance-based step: $d_{step} = t \times 0.15$.
    b. Calculate gap-based step: $g_{step} = h > 0 ? h \times 0.5 : \text{shadowMinStep}$.
    c. Determine step size: $\Delta t = \min(d_{step}, g_{step})$.
    d. Advance ray: $t \leftarrow t + \max(\Delta t, \text{shadowMinStep})$.
    e. If $t > \text{shadowMaxDistance}$, break loop.
    f. Calculate new clearance: $h = oy + dy \times t - \text{heightAtExtended}(ox + dx \times t, oz + dz \times t)$.
    g. If $h < 0$ (terrain hit):
        i. Return $\text{clamp01}(V \times \max(0, 1 + h / (\text{shadowMinStep} \times 1.5)))$.
    h. Calculate visibility ratio: $R = (\text{softness} \times h) / (t \times \text{sunY})$.
    i. Update visibility: $V = \min(V, R)$.
3.  Return $\text{clamp01}(V)$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def soft_shadow_visibility(ox, oz, oy, dx, dy, dz, 
                           shadow_max_distance, shadow_softness, 
                           shadow_min_step, shadow_steps, 
                           height_at_extended):
    # Derived parameters
    softness = 6 * shadow_softness + 2
    sun_y = max(dy, 0.12)
    
    t = 0.0
    h = 0.0
    visibility = 1.0
    
    for _ in range(shadow_steps):
        # Step size calculation
        distance_step = t * 0.15
        gap_step = h * 0.5 if h > 0 else shadow_min_step
        step = min(distance_step, gap_step)
        
        # Advance ray
        t += max(step, shadow_min_step)
        
        # Boundary check
        if t > shadow_max_distance:
            break
            
        # Calculate clearance
        # Note: ox, oz are horizontal offsets from origin; 
        # assuming ox, oz represent the horizontal position of the ray origin
        # and dx, dz are horizontal components of sun direction.
        # The height field is queried at the projected horizontal position.
        hx = ox + dx * t
        hz = oz + dz * t
        terrain_height = height_at_extended(hx, hz)
        h = oy + dy * t - terrain_height
        
        # Terrain hit detection
        if h < 0:
            # Clamp visibility contribution from the hit
            contribution = max(0.0, 1.0 + h / (shadow_min_step * 1.5))
            return max(0.0, min(1.0, visibility * contribution))
            
        # Visibility update
        # Formula: visibility = min(visibility, (softness * h) / (t * sunY))
        ratio = (softness * h) / (t * sun_y)
        visibility = min(visibility, ratio)
        
    return max(0.0, min(1.0, visibility))
```

**Edge Cases and Pitfalls:**
*   **Flat Ground:** Using a simple clearance/distance ratio fails on flat ground because clearance is proportional to distance, causing false heavy shadows. Normalizing by $t \cdot \text{sunY}$ ensures flat ground is fully lit [S1, unofficial].
*   **Ridge Shoulders:** Without gap limiting, the shadow ray can skip over ridge shoulders, creating hard stair-step shadow edges aligned with the heightfield grid [S1, unofficial].
*   **Sun Direction:** The sun's vertical component (`sunY`) must be clamped to a minimum value (0.12) to prevent division by zero or extreme values when the sun is near the horizon [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is `shadowSteps`. The number of steps is fixed per pixel.
*   **Memory Complexity:** $O(1)$ auxiliary space.
*   **Performance:** In the source implementation (M-series Mac, 6 workers, 1536² heightfield), rendering time scales with resolution. For 1920×1080 high quality, 18.7M rays were processed in 34.0 seconds [S1, unofficial]. Generation is a fixed cost and does not scale with image size [S1, unofficial].
*   **Strengths:** Efficient soft shadow calculation without expensive supersampling or kernel sampling. Handles flat terrain correctly.
*   **Limitations:** Fixed number of steps may not adapt to complex terrain features at varying distances. Dithering requires multiple samples or temporal accumulation to fully average out artifacts.

## Background
This method is part of a procedural terrain rendering pipeline described in the `theme1` repository by mingderwang [S1, unofficial]. It is designed for real-time or near-real-time rendering of large-scale terrain with volumetric clouds and water, leveraging distance-proportional ray marching for terrain intersection [S1, unofficial].

## Sources
[S1] theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
