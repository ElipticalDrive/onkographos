# Transparent Voxel Rendering with Refraction

Also called: Transparent Voxel Refraction, Refractive Voxel Accumulation

## Overview
This method extends ray-grid traversal to handle semi-transparent voxels by accumulating color and modulating ray intensity based on material opacity, while simultaneously applying Snell's law to refract the ray direction upon entering or exiting refractive media. It differs from opaque voxel rendering by allowing light to pass through and scatter within the volume, and from pure refraction methods by integrating standard volumetric color accumulation with directional bending, ensuring that total internal reflection (TIR) is handled to prevent infinite recursion.

## How it works
**Inputs and Outputs:**
*   **Input:** Ray origin $O$, ray direction $D$, maximum depth $maxDepth$, chunk bounds, voxel grid data, material properties (opacity, albedo, refractive index), and a boolean flag `enableRefraction`.
*   **Output:** Accumulated color $C_{out}$ and final ray state.

**Data Structures:**
*   **Material:** Contains `opacity` (float, 0.0–1.0), `albedo` (color vector), and `refractIndex` (float).
*   **Ray State:** `rayPos` (current position), `rayDir` (current direction), `invRayDir` (inverse direction for DDA), `lastVoxID` (ID of last processed voxel, 255 for empty), `lastVoxRefract` (refractive index of last medium), `colorMult` (transmittance multiplier, initialized to 1.0), and `colorAdd` (accumulated color, initialized to 0.0).
*   **Constants:** `EPSILON` (small constant, e.g., $10^{-6}$) to prevent self-intersection.

**Algorithm Steps:**
1.  **Initialization:** Set `rayPos = O`, `rayDir = D`, `invRayDir = 1.0 / D`, `colorMult = 1.0`, `colorAdd = 0.0`, `lastVoxID = 255`, `lastVoxRefract = 1.0` (air).
2.  **Traversal Loop:** While `rayPos` is within the voxel grid bounds and ray depth < `maxDepth`:
    a.  **Voxel Lookup:** Identify the current voxel $V$ at `rayPos`.
    b.  **Opaque Voxel Handling:** If $V$ exists and `V.opacity == 1.0`:
        i.  Update `rayPos` by adding `rayDir * (min(lastSideDist) + EPSILON)` to snap to the surface.
        ii. Return `true` (ray terminated).
    c.  **Transparent/Empty Voxel Handling:**
        i.  **If Voxel Exists and is Transparent (`V.opacity < 1.0`):**
            *   If `lastVoxID != V.id`:
                *   Calculate current position relative to chunk entry.
                *   If ray depth < `maxDepth`:
                    *   Add to color: `colorAdd += colorMult * V.material.opacity * V.material.albedo * sunStrength`.
                    *   Update transmittance: `colorMult *= (1.0 - V.material.opacity)`.
                *   **Refraction Check:** If `enableRefraction` and `rayDist > 0.0`:
                    *   Set `refracted = true`.
                    *   Update `rayPos` to the entry point of the voxel.
                    *   Calculate surface normal $N$ (using dot product check against voxel face).
                    *   Compute new direction: $D' = \text{refract}(D, N, \frac{lastVoxRefract}{V.material.refractIndex})$.
                    *   If TIR occurs, $D' = D$ (fallback to original direction).
                    *   Update `rayDir = D'`, `invRayDir = 1.0 / D'`.
                    *   Re-initialize DDA state (`lastSideDist`).
                *   Update `lastVoxID = V.id`, `lastVoxRefract = V.material.refractIndex`.
        ii. **If Voxel is Empty (`V.id == 255` or null):**
            *   If `lastVoxID != 255`:
                *   **Refraction Check:** If `enableRefraction`:
                    *   Set `refracted = true`.
                    *   Update `rayPos` to the exit point of the previous voxel.
                    *   Calculate surface normal $N$ (using `-voxel.normal` of the previous voxel).
                    *   Compute new direction: $D' = \text{refract}(D, N, \frac{lastVoxRefract}{1.0})$.
                    *   If TIR occurs, $D' = D$.
                    *   Update `rayDir = D'`, `invRayDir = 1.0 / D'`.
                    *   Re-initialize DDA state (`lastSideDist`).
                *   Update `lastVoxID = 255`, `lastVoxRefract = 1.0`.
3.  **Final Output:** Return `colorAdd` as the final color.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_transparent_refractive_voxel(ray_origin, ray_dir, max_depth, grid, enable_refraction):
    ray_pos = ray_origin
    ray_dir = normalize(ray_dir)
    inv_ray_dir = 1.0 / ray_dir
    color_mult = 1.0
    color_add = vec3(0.0)
    last_vox_id = 255
    last_vox_refract = 1.0
    ray_dist = 0.0
    
    while ray_pos in grid_bounds and ray_dist < max_depth:
        voxel = grid.get_voxel(ray_pos)
        refracted = False
        
        if voxel is not None and voxel.opacity == 1.0:
            # Opaque: terminate
            ray_pos += ray_dir * (min(last_side_dist) + EPSILON)
            return color_add
        
        if voxel is not None and voxel.opacity < 1.0:
            if last_vox_id != voxel.id:
                if ray_dist < max_depth:
                    color_add += color_mult * voxel.material.opacity * voxel.material.albedo * sun_strength
                    color_mult *= (1.0 - voxel.material.opacity)
                
                if enable_refraction and ray_dist > 0.0:
                    refracted = True
                    ray_pos = get_entry_point(ray_pos, ray_dir)
                    normal = calculate_normal(ray_pos, voxel)
                    eta = last_vox_refract / voxel.material.refract_index
                    ray_dir = refract(ray_dir, normal, eta)
                    if ray_dir is None: # TIR
                        ray_dir = ray_dir # Keep original
                    inv_ray_dir = 1.0 / ray_dir
                    reset_dda_state()
                
                last_vox_id = voxel.id
                last_vox_refract = voxel.material.refract_index
                
        elif last_vox_id != 255:
            # Exiting to empty space
            if enable_refraction:
                refracted = True
                ray_pos = get_exit_point(ray_pos, ray_dir)
                normal = -voxel.normal # Normal of previous voxel
                eta = last_vox_refract / 1.0
                ray_dir = refract(ray_dir, normal, eta)
                if ray_dir is None: # TIR
                    ray_dir = ray_dir
                inv_ray_dir = 1.0 / ray_dir
                reset_dda_state()
            
            last_vox_id = 255
            last_vox_refract = 1.0
            
        ray_dist += step_size
        ray_pos += ray_dir * step_size
        
    return color_add
```

**Parameters:**
*   `EPSILON`: Small constant to prevent precision errors (e.g., $10^{-6}$).
*   `enableRefraction`: Boolean, default `false`.
*   `maxDepth`: Maximum ray traversal depth.

**Edge Cases:**
*   **Total Internal Reflection (TIR):** If the angle of incidence exceeds the critical angle, `refract` returns `None` or invalid direction. The method falls back to the original ray direction to prevent infinite loops and performance degradation [S1, unofficial].
*   **Precision Errors:** `EPSILON` is added to distance calculations to avoid self-intersection when updating `rayPos` after a hit [S1, unofficial].
*   **Redundant Calculations:** Refraction is only applied when transitioning between different voxel types (solid-to-transparent, transparent-to-solid, transparent-to-empty) to avoid redundant computations [S1, unofficial].

## Subtopics

### [Visibility-Based Anti-Aliased Voxel Rendering](16980-visibility-based-anti-aliased-voxel-rendering.md)
Reduces bloating and popping artifacts by converting opaque voxels to semi-transparent ones based on preprocessing visibility, filtering high spatial frequencies through opacity blending rather than naive subdivision.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels traversed, with additional overhead for refraction calculations (dot products, refract function) at each interface.
*   **Memory Complexity:** Constant per ray, dependent on stack depth for recursion (if used) or loop state.
*   **Strengths:** Handles complex refractive materials efficiently within a voxel grid; avoids infinite loops via TIR fallback.
*   **Limitations:** Refraction calculations are more expensive than simple traversal; accuracy depends on normal estimation at voxel boundaries.
*   **When to Use:** Scenes with transparent or refractive objects where performance is critical and full path tracing is too slow.
*   **When Not to Use:** When high-fidelity caustics or complex light paths are required, as this method approximates refraction without full global illumination.

## Background
This method is derived from implementations in real-time voxel rendering engines, specifically the DoonEngine, which integrates DDA traversal with material-based refraction and transparency accumulation [S1, unofficial]. It addresses the limitations of opaque voxel rendering by allowing light to interact with semi-transparent media, a common requirement in voxel-based games and simulations.

## Sources
[S1] frozein/DoonEngine, https://github.com/frozein/DoonEngine, fetched 2026-10-26 (unofficial)
