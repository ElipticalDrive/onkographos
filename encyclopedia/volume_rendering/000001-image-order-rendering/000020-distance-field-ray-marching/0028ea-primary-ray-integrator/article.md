# Primary Ray Integrator

Also called: Composite Ray Marcher, renderRay

## Overview
The Primary Ray Integrator is a single-function ray integrator that resolves a complete ray by compositing terrain, water, sky, cloud volume, and aerial perspective [S1, unofficial]. It serves as the core rendering loop for distance-field ray marching in terrain and atmospheric scenes, handling the sequential evaluation of distinct environmental layers [S1, unofficial]. It sets itself apart from sibling methods by explicitly managing the compositing order of heterogeneous scene elements (solid surfaces vs. volumetric clouds) and handling secondary ray recursion to prevent infinite loops [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   `ctx`: A `RenderContext` object containing scratch buffers, theme parameters, and camera state [S1, unofficial].
    *   `ox, oy, oz`: The ray origin coordinates [S1, unofficial].
    *   `dx, dy, dz`: The normalized ray direction vector components [S1, unofficial].
    *   `tMax`: The maximum ray distance for traversal [S1, unofficial].
    *   `depth`: An integer recursion depth counter. Primary rays pass `0`; secondary rays (reflections/refractions) pass `1` [S1, unofficial].
*   **Outputs:**
    *   `result`: An `Rgb` color value representing the final pixel color [S1, unofficial].

**Data Structures:**
*   `RenderScratch`: A struct used for intermediate calculations, initialized to zero upon creation [S1, unofficial]. It contains:
    *   `result` (`Rgb`): Accumulated color.
    *   `secondary` (`Rgb`): Color for secondary ray contributions.
    *   `surface` (`SurfaceShade`): Surface shading data (albedo, normal, etc.) [S1, unofficial].

**Algorithm Steps:**

1.  **Secondary Ray Guard:** If `depth > 0`, disable water and cloud passes to prevent infinite recursion [S1, unofficial].
2.  **Water Plane Intersection:** If `depth == 0`, test for intersection with the water plane [S1, unofficial]. Compute `limit` as the minimum of the water hit distance and `tMax`, or `tMax` if no water hit [S1, unofficial].
3.  **Terrain/Water/Sky Trace:** Trace the ray from the origin along the direction up to `limit` [S1, unofficial]. Determine the `hitKind` (`KIND_TERRAIN`, `KIND_WATER`, `KIND_SKY`) and `distance` to the hit [S1, unofficial].
4.  **Surface Shading:**
    *   **If `KIND_TERRAIN`:** Compute hit position $\mathbf{p} = \mathbf{o} + \mathbf{d} \cdot t_{\text{terrain}}$ [S1, unofficial]. Call `shadeTerrain` with the hit position and a `ditherAngle` [S1, unofficial]. Copy the resulting surface color to `result` [S1, unofficial].
    *   **If `KIND_WATER`:** Compute hit position $\mathbf{p} = \mathbf{o} + \mathbf{d} \cdot t_{\text{water}}$ [S1, unofficial]. Call `shadeWater` with `ctx.waterShade` parameters [S1, unofficial]. Copy the resulting surface color to `result` [S1, unofficial].
    *   **If `KIND_SKY` or no hit:** Call `skyRadiance` with the ray direction to fill `result` [S1, unofficial].
5.  **Atmospheric Compositing:**
    *   **If `KIND_SKY`:** Apply sky haze using the vertical component of the direction vector, $d_y$ [S1, unofficial].
    *   **Else (Terrain/Water hit):** Compute the hit Y coordinate $y_{\text{hit}} = o_y + d_y \cdot \text{distance}$ [S1, unofficial]. Apply aerial perspective using `distance`, $y_{\text{hit}}$, and the ray direction to composite the atmosphere over the surface color [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def renderRay(ctx, ox, oy, oz, dx, dy, dz, tMax, depth):
    # Initialize scratch buffers to zero
    result = [0.0, 0.0, 0.0]
    secondary = [0.0, 0.0, 0.0]
    surface = {"color": [0.0, 0.0, 0.0], "normal": [0.0, 0.0, 0.0]}

    # Step 1: Secondary Ray Guard
    disable_water = depth > 0
    disable_cloud = depth > 0

    # Step 2: Water Plane Intersection
    limit = tMax
    if not disable_water:
        # Intersect ray with water plane [G]
        water_hit_dist = intersect_water_plane(ox, oy, oz, dx, dy, dz)
        if water_hit_dist is not None and water_hit_dist < tMax:
            limit = water_hit_dist

    # Step 3: Trace ray up to limit
    # Sphere tracing implementation from parent method [G]
    hit_kind, distance = trace_ray(ox, oy, oz, dx, dy, dz, limit)

    # Step 4: Surface Shading
    if hit_kind == "KIND_TERRAIN":
        px, py, pz = ox + dx * distance, oy + dy * distance, oz + dz * distance
        # Theme-specific terrain shading [G]
        shadeTerrain(px, py, pz, surface, ctx.ditherAngle)
        result = surface["color"]
    elif hit_kind == "KIND_WATER":
        px, py, pz = ox + dx * distance, oy + dy * distance, oz + dz * distance
        # Theme-specific water shading [G]
        shadeWater(px, py, pz, surface, ctx.waterShade)
        result = surface["color"]
    else:
        # Sky radiance calculation [G]
        result = skyRadiance(dx, dy, dz)

    # Step 5: Atmospheric Compositing
    if hit_kind == "KIND_SKY":
        # Sky haze computation [G]
        haze_factor = compute_haze(dy)
        result = composite_haze(result, haze_factor)
    else:
        y_hit = oy + dy * distance
        # Aerial perspective compositing [G]
        result = apply_aerial_perspective(result, distance, y_hit, dx, dy, dz)

    return result
```

**Parameters:**
*   `ditherAngle`: Used in terrain shading to reduce banding artifacts [S1, unofficial].
*   `ctx.waterShade`: Configuration object for water rendering properties [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Infinite Recursion:** Secondary rays (water reflections/refractions) calling back into the primary integrator can cause infinite loops. This is handled by checking `depth > 0` and disabling water/cloud passes for recursive calls [S1, unofficial].
*   **Boundary Conditions:** If `tMax` is not respected, rays may escape the scene bounds without proper termination [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Handles multiple scene types (terrain, water, sky, clouds) in a single pass; robust against recursion issues via depth checking [S1, unofficial].
*   **Limitations:** Limited to specific scene compositions defined in the integrator; does not handle complex volumetric scattering beyond simple aerial perspective and cloud compositing [S1, unofficial].
*   **When to use:** Not given in the sources.

## Background
The Primary Ray Integrator is described in the context of the `theme1` project by mingderwang [S1, unofficial]. It is part of a larger system that includes CLI tools for rendering, theme validation, and batch processing [S1, unofficial]. The method builds on the principles of distance-field ray marching, specifically sphere tracing, as detailed in the parent article [S1, unofficial].

## Sources
[S1] mingderwang/theme1, https://github.com/mingderwang/theme1, fetched 2026-10-27 (unofficial)
